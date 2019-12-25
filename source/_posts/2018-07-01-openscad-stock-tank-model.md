---
title: Stock-Tank mesh construction using OpenSCAD
abstract: "Constructing OpenFOAM meshes with blockMesh utility can get really tricky, that's 
	why I prefer to create geometry STL models with OpenSCAD first and then mesh them."
header_image: /assets/images/ionic2-banner.png
cover: https://s2.ax1x.com/2019/09/18/n7E8Wd.png
date: 2018/07/01
categories:
  - openscad
tags:
  - OpenSCAD
  - Meshing
---

This post is part of a longer collection:
* [OpenSCAD 101: Basics of the OpenSCAD language](/openscad-basics/)
* [OpenSCAD Transformations & Boolean Ops](/openscad-transformations-boolean-operations/)
* [OpenSCAD CFD-oriented Usage Examples](/openscad-cfd-usage-examples/)

It features a very simple, quick and efficient way of constructing a (2D) stock-tank geometry and similar shapes using **OpenSCAD**.



The main reason I present this way of thinking here is that I don't think `blockMesh` is quite the right tool for such tasks; even when used with a powerful macro language, like *m4*.

Well, there are a couple of things I expect you to be proficient in:
* A decent text editor should be used (I'm using VIM) to edit OpenSCAD scripts:  It's good to have something that allows for quick basic calculations to be inserted,  or at least, provide access to the shell.
* Basic Knowledge of some interpreted languages (Python, Lua, Ruby ... etc, pick the one you like, I usually use Python): They are great to test ideas quickly.

As an example mesh, we will be creating the one presented in the following
figure:

![Sketch of desired OpenSCAD model](/assets/img/OpenSCAD/tank-model-sketch.png)

Of course, model dimensions don't matter because the goal is to create a parametric one.

# A one time deal: Hard code everything in OpenSCAD scripts!

Constructing the stock-tank model point-by-point is not as hard as you might think:

1. Create **points** and **paths** vectors (if necessary) using your text editor.
2. Run `linear_extrude` on resulting polygon.

The first step strongly depends on used text editor; In VIM,  I have this awesome vim command defined in my *.vimrc* file:

~~~vim
command! -nargs=+ Calc :read !python -c "from math import *;
            \ import numpy as np;
            \ np.set_printoptions(precision=4);
            \ <args>"
~~~

Which allows me to run Python code directly via VIM (imports are kind of important here).

If, for example, I need a set of points to define an "arc" of **radius = 1** between `[10,1]` and `[12,1]` points,
I only need to issue the following VIM command while editing the SCAD script:


~~~vim
:Calc l=np.linspace(10,12,10); m=np.mat([l,1+np.sqrt(1-(l-11)**2)]); print(m.transpose().tolist())
~~~

Which will insert all points into the current file (actually, the print command is the one responsible for this). `linear_extrude`'s default `paths` vector is usually fine.

## Is it a bad idea to do things this way?


Yes, it is. These tricks may work if one only needs to prototype something, or if one desires to check whether an idea is good enough to be implemented!  But they fail badly in real situations, where users absolutely have to create a parametric model.

Fortunately, similar tricks often suggest possible approaches to the task at hand: Notice that we used Python's **list comprehensions** to create *numpy* vectors; So, why not using **OpenSCAD list comprehensions** to create OpenSCAD vectors?

# Parametric design of a stock tank

## Better than hard-coding points, but still not an optimal approach

Instead of relying on the power of a text editor, it might be a good idea to use list comprehensions to build needed points. Here's a complete OpenSCAD script explaining the situation:

~~~clike
// Points of the lower arc:
// half-a-circle, radius = 1, between [10,1] and [12,1]
// which means center is [11,1]
list1 = [ for (i = [10 : 0.1 : 12]) [i, 1+sqrt(1-pow((i-11),2))] ];
// Points of the upper arc
// half-a-circle, radius = 2, between [9,1] and [13,1]
// which means center is [11,1]
list2 = [ for (i = [13 : -0.1 : 9]) [i, 1+sqrt(4-pow((i-11),2))] ];
// Repeating code, --> consider a function

// polygon points, just some concatenated lists
points = concat(
    [[0,0], [10,0]],
    list1,
    [[12, -3], [13, -3]],
    list2,
    [[8,1], [8,8], [4,8], [4,4], [3,4], [3,6], [0,6]]
    );  

// Default behavior of polygon module
paths = [ for (i = [0 : 1 : len(points)-1]) i ];

// The actual 3D model
linear_extrude(2,[0,0]) polygon(points, [paths]);
~~~


This is certainly a better approach, but we still need to do some adjustments for easier parameterization: Let's define an (object) module to create the boxes-like portion of our model.

## Using modules to construct parametric geometries


~~~clike
/*
    Create an object model called tank.
    Parameters:
    - w: Base width, to interface with the "arc-y region", default=1.
    - tbh: Total Box height, defaults to 10.
    - pbl: A list of partial box lengths, defaults to [6,4]
    - pbw: A list of partial box "widths", default = [3,1]
    - st: Translate partial boxes in x-direction, defaults to [0,3]
*/
module tank (w=1, tbh=10, pbl=[6,4],pbw=[3,1],st=[0,3]) {
        // create a parent cube, and subtruct all following objects
        // You can also build this region box-by-box    
        difference () {
        
        	// parent cube
        	cube([tbh,tbh,2]);
        
        	// A special box, allows for better connection with the arc.
        	translate([tbh-2*w,w,-0.01]) cube([2.001,tbh+0.01,3]);
        
        	// A set of helper boxes to create desired shape.
        	for (i=[0:len(pbl)-1]) translate([st[i]-0.01,pbl[i]-0.01,-0.01]) 
            	cube([pbw[i]+0.001,tbh,3]);
        }   
}
~~~

We can also define our own module to create arcs:

~~~clike
/*
    Create module named arc.
    Parameters:
    - r: inner radius, default=1.
    - w: arc width, defaults to 1.
    - ext: vertical extension length, defaults  to 3.
*/
module arc (r=1, w=1, ext=3) {
    ar = r+w;
    
    // points on arcs
    list1 = [ for (i = [-r : 0.1 : r])
        [i, sqrt(pow(r,2)-pow(i,2))] ];
    list2 = [ for (i = [ar : -0.1 : -ar])
        [i, sqrt(pow(ar,2)-pow(i,2))] ];

    points = concat(
        [[-ar,0]],
        list1,
        [[r, -ext], [ar, -ext]],
        list2
        );

    // Default behavior of polygon, but let's
    // specify it.
    paths = [ for (i = [0 : 1 : len(points)-1]) i ];
    linear_extrude(2) polygon(points, [paths]);
}
~~~

To make sense of module arguments, refer to the following figure:

![Partially parametric OpenSCAD model](/assets/img/OpenSCAD/tank-model-boxes.png)

Now, creating basic models is as simple as issuing:

~~~clike
union() {
    tank();
    translate([11,1]) arc();
}
~~~

More complicated models can be easily derived by overwriting default parameter values:

~~~clike
union() {
    tank(tbh=15, pbl=[6,4,10],pbw=[3,1,5],st=[0,3,4]);
    translate([16,1]) arc();
    translate([-2,6,2]) rotate([0,180,0]) arc(r=2);
}
~~~

Which results in:


![Final OpenSCAD parametric model](/assets/img/OpenSCAD/tank-model.png)

> There may be simpler ways to achieve these results, but this approach is most useful for users who want to practice creating 3D geometry models with **OpenSCAD**.


