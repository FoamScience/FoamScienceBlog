---
title: Object Registration in OpenFOAM
abstract: "This article introduces you to how amazing OpenFOAM is in
   keeping track of directly and indirectly managed variables in a solver
   using its objectRegistry and regIOobject classes."
header_image: /assets/images/ionic2-banner.png
cover: /assets/images/ionic2-banner.png
date: 2019/09/12
categories:
  - coding
tags:
  - OpenFOAM
  - Programming
  - Cpp
---

Being aware of all the important variables during the simulation is a nice feature to have in any CFD software. I've even seen some software developers "brag" about how many "variables" their simulators efficiently manage at run time.  Today, we'll discover how OpenFOAM solvers **keep track** of model data used across libraries.

## Know the difference between Object Registration and Run-Time Selection Table

> By "Object registration" I don't mean "Model registration" (The mechanism allowing for reporting possible models if the user input is unexpected, eg. if using a non-existent Boundary Condition Type). I refer instead to the mechanism allowing the program to retrieve references to all sorts of "data" objects (volScalarFields ... etc) that are under the solver's direct or indirect control. Read this section to make sense of the difference.

Traditional C++ programming will probably employ "Factory-like" design to create new objects: A virtual base class will have a `static create()` (or a `static New()` in the case of OpenFOAM classes) to create a pointer (of some kind) to the newly created model.

This traditional technique is used when the programmer wants to give users the ability to "**select**" a model from a set of pre-defined ones (Child classes of the base class). 

It's common in OpenFOAM to give the virtual base class (usually belonging to the `Foam` namespace) a static `New()` method returning an `autoPtr` to the newly created instance of the model class (`autoPtr` is used so that the created object couldn't be referred to by more than _one_ pointer). Of course, "templates" may complicate things a bit more, but nothing we can't live with.

This `New()` method would typically read the model's name from a dictionary, check if such model is "registered" to the RunTime selection Table, then construct an object of the appropriate type. This process is referred to as "selection of a model".

> Note that if another library wants to provide a child for such base class, it has to statically link the base class's library (at compile time). That is to say, if a library `transportModels.so` has a `phase` base class which already has some child classes (say `incompressiblePhase` and `gasPhase`) and you want to add a phase object for your specific solver, you can build a new library `myPhaseModels.so` which must statically link `transportModels.so` so it can be compiled. It's that simple, you just create your new `compositionalPhase` class (inheriting from `phase`) and it will show up in the selection table when you load the library (in `controlDict` of your case).

Older C++ code would contain overloaded versions of `New()` (each returning a different type) but OpenFOAM uses templates and a collection of Macros to construct a `RunTimeSelectionTable` which `New` uses to check for existence and select the model.

But that's not the goal of this post. Let's instead dive into the registration of model data. For example, consider the case of a library trying to access a field that is created and managed by another library! To illustrate, say we have some library `transportModels.so` whose basic `transportModel` class has a member `volScalarField rho`. Suppose we're working on another library and we need a (maybe `const`) reference to this field without actually statically linking `transportModels.so` (Say, this would result in a **cyclic dependency** between the two libraries). How would one get a reference to such field?

That's exactly the situation we need object registration for.

## OpenFOAM objectRegistry and regIOobject

OpenFOAM solvers keep a "hierarchical database", logging objects at different levels:
- The master registry is always the Time object (`runTime`).
- The second-level in the database logs mesh regions and the global `controlDict`.
- The lower levels  register "sub-objects" (eg. Each mesh region would have its own `fvSchemes`, `fvSolution`, mesh data, the fields associated to the mesh region - created in `createFields.H` ... etc) 

Note that each object in the database "can be" a database of registered objects allowing for the hierarchical structure; and by "mesh regions" I mean different mesh objects (for example, a mesh for fluids and another for solid regions in fluid-solid interaction simulations).

> If two "Time instances" are used in a solver, we would have two independent databases; and it makes complete sense!

For illustration, standard solvers databases would look like this:

```bash
* runTime         # objectRegistry
 |--> controlDict # regIOobject (can't have sub-entries attached to it)
 |--> mesh1       # objectRegistry (a database)
    |--> points, owner, neighbour, cellZones ...
    |--> fvSchemes, fvSolution
    |--> U, p
 |--> mesh2
    |--> points, owner, neighbour, cellZones ...
    |--> fvSchemes, fvSolution
    |--> U, p
```


> This Hierarchy is kept in memory, but the same class handles how it's written to disk.

The database class is called `objectRegistry` (so it's a database and an entry in a database at the same time), and in order for objects to auto-register to a database, they must inherit from `regIOobject` class.

The inheritance from `regIOobject` promotes the object into an auto-registered object and it requires to implement the pure virtual member function `writeData()` (not `write()`) so that when the database issues the order of writing to disk to all its child entries, the child will be able to write itself properly :

```cpp
virtual bool writeData(Ostream&) const;
```

An example of a simple, non-standard class which inherits from `regIOobject` is the basic `well` class in my [Reservoir Simulation toolkit](https://github.com/FoamScience/OpenRSR/blob/master/libs/wellModels/wells/well/well.H). A well can then be written to disk as a regular dictionary.

## Cross library interaction in OpenFOAM

Now that we all have a good understanding of the hierarchical registration in OpenFOAM, let's dive into how to use such mechanism for our benefit.

In practice, when writing OpenFOAM libraries, it's common to pass a mesh reference to the base model's constructor (You can do very little without a reference to the mesh, really). Thus, the mesh is the number-one database to log our objects to: Which database to use is decided at the object's construction time (by calling `regIOobject`'s  constructor with the right third parameter). 

In standard solvers, the mesh is created by passing an `IOobject` to the constructor of the parent class `polyMesh` which passes it to `objectRegistry`'s  constructor:

```cpp
// fvMesh is an objectRegistry
Foam::fvMesh mesh
(
    Foam::IOobject
    (
        Foam::fvMesh::defaultRegion,
        runTime.timeName(),
        runTime,    // That's the database we branch into
        Foam::IOobject::MUST_READ
    )
);
```

> And you can verify that the `Time` class is not registered to anything.


Now, say we registered a `volScalarField` with the name of "fName" to the previous mesh object  (See `createFields.H` file of any standard solver for examples). How can we refer to it in a Boundary Condition Type we're adding?

Well, that's easy, we just use `fvMesh`'s interface:

```cpp
// Const-ref to the field named fName
const volScalarField& fN = mesh.lookupObject("fName");
// Ref to the field named fName
volScalarField& fN = mesh.lookupObjectRef("fName");
```

Well, for a more sophisticated way, we may give the user the option of selecting the field's name:
```cpp
// Read the name if provided
word theName = someDict.lookupOrDefault<word>("fieldName","fName");
// Const-ref to the field named fName
const volScalarField& fN = mesh.lookupObject(theName);
```

This particular trick may complicate things for beginners with all the "**didn't find a field in the database**" Fatal Errors, but it's a decent way to program things in OpenFOAM. Anyway, I hope this article clarified at least what the third argument to `IOobject`'s constructor is meant to do :smile:

If you have any suggestions or comments on this matter, don't hesitate, fire at me bellow.

