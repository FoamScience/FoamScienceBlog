---
title: Customize VIM to work efficiently with OpenFOAM - Part 2
abstract: "Working in the right directory and easier file navigation aren't productive enough, 
	we need a way to run OpenFOAM commands right from inside VIM while editing files"
header_image: /assets/images/ionic2-banner.png
cover: https://s2.ax1x.com/2019/09/18/n7E8Wd.png
date: 2018/08/28
categories:
  - productivity
tags:
  - OpenFOAM
  - Vim
---

We've seen in [the previous part](/vim-openfoam-power-part1) some ways to improve our interaction with OpenFOAM case directories and files. In this new post, we want to implement something that facilitates running `blockMesh`, or the case solver, right from inside VIM; without going back to the shell. Of course, our implementation should catch errors in the quick fix list (if there are any),  then go to the concerned file, and put the cursor on the error line.

For this, we'll use two main VIM facilities: *VIM compilers* and *QuickFix* list.
The compilers are used to automatically run a shell command on a file; For example:

- Running a `gcc` (or `make` if there is a *makeFile*) command on a `*.cpp` file would compile it to an executable, 
- Running a `pdflatex` (or better `latexmk`) command on a *Latex file* would compile a PDF version of the document.
- Running a `blockMesh` command on a *blockMeshDict* would build the mesh and check for errors in `blockMesh`'s output.

In fact, we can run as many commands as we want on the file using VIM compilers. Think: Generate the mesh, check its validity, set fields and run appropriate solvers (In my opinion, this kind of work-flows beats GUIs to death).

So, let's get to work: We need to know how VIM's default compiler plugins are set, and the best way to learn is by looking at examples. On Unix systems, the default compilers should be in `/usr/share/vim/vim74/compiler` directory, you can use VIM's ex-command `:compile` to find out where  they are on your system.

## Studying an example VIM compiler

First of all, a VIM compiler plugin is nothing more than a short `VimL` script, configuring two main things: `errorformat` (scanf-style strings, separated by commas, showing the format of error lines) and `makeprg` (make program, the shell executable to run).

As an example, let's go through the *ant-compiler* line by line:

~~~vim
" Vim Compiler File
" Compiler:	ant
" Maintainer:	Johannes Zellner <johannes@zellner.org>
" Last Change:	Mi, 13 Apr 2005 22:50:07 CEST
~~~

The header, just for information, then we have some checks to perform:

~~~vim
if exists("current_compiler")
    finish
endif
let current_compiler = "ant"

if exists(":CompilerSet") != 2		" older Vim always used :setlocal
  command -nargs=* CompilerSet setlocal <args>
endif
~~~

The first if structure checks if the compiler is already set (if it's the case, vim will leave the script). If not, it
declares `current_compiler` as *"ant"* (so, the next time the script executes, it will be finished in the first if structure). The second if statement sets compatibility stuff for older VIM versions.

All the `cpo`-related stuff are set to configure Vi-compatibility options, we don't need to learn about this topic, just don't touch those lines!

The line `CompilerSet makeprg=ant` sets the ant program as the make program (when you execute the ex-command `:make`, VIM runs `ant` in a shell, and watches for its output).

`CompilerSet errorformat=\ %#[%.%#]\ %#%f:%l:%v:%*\\d:%*\\d:\%t%[%^:]%#:%m` shows VIM how to find the error line and what to extract from it (explained in more details in the next section).

~~~vim
let s:cpo_save = &cpo
set cpo&vim

CompilerSet makeprg=ant
" first  line:
"     ant with jikes +E, which assumes  the following
"     two property lines in your 'build.xml':
"
"         <property name = "build.compiler"       value = "jikes"/>
"         <property name = "build.compiler.emacs" value = "true"/>
"
" second line:
"     ant with javac
"
" note that this will work also for tasks like [wtkbuild]
"
CompilerSet errorformat=\ %#[%.%#]\ %#%f:%l:%v:%*\\d:%*\\d:\ %t%[%^:]%#:%m,
    \%A\ %#[%.%#]\ %f:%l:\ %m,%-Z\ %#[%.%#]\ %p^,%C\ %#[%.%#]\ %#%m

" ,%-C%.%#

let &cpo = s:cpo_save
unlet s:cpo_save
~~~


### Using errorformats to catch OpenFOAM Fatal Errors

These *errorformats* are similar to regular expressions. Assume a program outputting errors in the following format:

`Error: something-is-wrong in /path/to/file at line 25`

To match this error line, we use a single scanf-style string, that may contain special items (these are the most important ones):
- `%f` matches a file name (a string), eg. */path/to/file*
- `%l` matches a line number (digits), eg. *25*
- `%m` matches an error message, eg. *something-is-wrong*
- `%*{conv}` converts stuff from regexp to scanf string, eg. %*[a-z] matches any number of lowercase alphabets.

Thus, the example line would be matched with `\Error: %m in %f at line %l`

## Building a compiler plugin for blockMesh

The goal of this section is to build a blockMesh-compiler for VIM which can be used to browse errors when the tool fails. For this, we need to create our custom-compilers directory `$HOME/.vim/compiler`.

Copy the `ant.vim` compiler to that directory and rename it as `blockMesh.vim`, on Unix systems, one can say:

~~~bash
cp /usr/share/vim/vim74/compiler/ant.vim ~/.vim/compiler/blockMesh.vim
~~~

Modify the header so it suits your preferences, and then start building the custom VIM compiler:
- Hopefully, you are editing the file with VIM.
- Change all occurrences of "*ant*" to "*blockMesh*" in the file `:%s/ant/blockMesh/g`
- Time to set the "errorformat": We know that OpenFOAM displays errors in multiple
lines (if you write "hxe" instead of "hex" in a *blockMeshDict*):

~~~text
--> FOAM FATAL IO ERROR:
CellShape has unknown model on line 45 the word 'hxe'
file: /home/elwardi/OpenFOAM/elwardi-4.1/run/movingCone/system/blockMeshDict.blocks at line 45.
~~~

Note that this is NOT a general FOAM ERROR; some other IO errors specify a range of lines; others suggest solutions before stating the error-line.

The most important line is probably the one stating the filename, and the exact line to jump to (these will be stored in `%f` and `%l` respectively).However, there are some problems in the filename: notice the ending `.blocks` appended to it, in fact, things could be messier: `.ddtschemes.default`.


How many dots are there in the filename? The real problem is that the `path/to/case` always contains a dot character (frrom the OpenFOAM version, eg. user-4.1), so matching from the start of the path to the first dot character won't work!

Instead, we do notice that versions are expressed in digits, and that case names usually contain no dots (hopefully); so, matching the path up to the first dot character preceding an alphabet character is a good idea:
`CompilerSet errorformat=\file:\ %f\.%[a-z]%*[a-z\ ]\ line\ %l\.`

- `%f` matches the string "file: " at the start of line, then starts storing the filename up to the first dot that falls immediately before a lowercase alphabet character (hopefully, that's the keyword `.blocks`, or `.fvSchemes` ...). Then we'll have  some lowercase characters and spaces until we reach the last line before "line" string,  then `%l` catches the line number (in case there are two, it catches the last one!).

Now, what to tell the user? Well, It's good to have a message at the status line saying "*CellShape has unknown model ...*", so, we should use a complex multi-line *errorformat*! But, still we'll have to deal with the problem of how many lines are there between the line that tells our message and the line that spells out the error line. 

This is actually troublesome, so, I've decided to approach the situation differently: We leave our filename and error-line alone, and add another error that captures only the message (using a simple multi-line error-format).

~~~vim
CompilerSet errorformat=\file:\ %f\.%[a-z]%*[a-z\ ]\ line\ %l\.
                        \%E-->\ %*[A-Z:\ ],%Z%m
~~~

- `%E` denotes the start of a multi-line error.
- Then we find the line saying `Fatal IO Error` (starts with –> then there are some uppercase letters,semicolons, and spaces).
- `%Z` denotes the end of multi-line error (which is only one line here!) then we capture everything in the second line as a `%m` (an error message).

The compiler plugin is now complete:

~~~vim
" Vim Compiler File
" Compiler: blockMesh
" Fadeli Mohammed Elwardi <foamscience.github.io>

if exists("current_compiler")
finish
endif

let current_compiler = "blockMesh"

if exists(":CompilerSet") != 2
" older Vim always used :setlocal
command -nargs=* CompilerSet setlocal <args>
endif

let s:cpo_save = &cpo
set cpo&vim

CompilerSet makeprg=blockMesh
CompilerSet errorformat=
                 \\file:\ %f\.%[a-z]%*[a-z\ ]\ line\ %l\.,
                 \%E->\ %*[A-Z:\ ],%Z%m

let &cpo = s:cpo_save
unlet s:cpo_save
~~~


And you should be ready to test it with a sample (erroneous) `blockMeshDict`:
1. Make something of importance go wrong in a `blockMeshDict` (the last ; in a block is not that important).
2. While editing `blockMeshDict`, run `:compile blockMesh` to set the correct compiler (We'll improve on this).
3. Hopefully your working directory is set by our previous `customFoam` script, if it is the case, run the ex-command `:make` and check that beautiful (and useful) error message!
4. To automatically jump to the error line, type `:cn` (short of cnext).

> The error format can't handle "from line 15 to line 18." lines, because no file path will be output in such lines!


Now that VIM knows how to handle `blockMesh`, let's introduce it to the other FOAM applications (we'll be doing just solvers for illustration). Wait, should we repeat the work done previously tens of times? Absolutely not.

The goal of this blog post is to achieve this work-flow:
1. When I read in a FOAM File, VIM changes its working dir. to case dir. (already implemented in `customFoam.vim`, see [this earlier post](/vim-openfoam-power-part1))
2. VIM looks for the suitable OpenFOAM solver for the current case (mentioned in `controlDict`).
3. VIM sets the right solver as the `makeprg` automatically.
4. When I run `:make` (Or press a shortcut to it), VIM will filter the output to show me the errors if there are any.

One more thing, I want my own solvers to be involved, so, I won't generate a list of standard solvers and compare anything to its content. That's one lousy approach to take!

Instead, I'm going to define two functions: The first one, in `customFoam.vim`, which searches and stores the solver name, and the second sets the option `makeprg` dynamically in a general compiler plugin called `foam.vim`.

## Use FOAMGetApplication to get solver name

From any file in the case dir., we can tell VIM what solver is to be used with the current case because it's mentioned in the `controlDict` file. Actually, we have two options here:
- Use VIM to take a look at `controlDict`, and store a string representing the solver name.
- Or, do it the FOAM way (use run functions: `getApplication`), which needs a working Installation of OpenFOAM (but that's Ok, you won't be reading this guide otherwise). This method is heavily used in `.Allrun` shell scripts.

In order to learn how VIM interacts with the system, we'll use the second option: We'll add a `FOAMGetApplication()` function to our `customFoam.vim`

~~~vim
function! FOAMGetApplication()
    " Source run functions and execute getApplication
    let cmd=". $WM_PROJECT_DIR/bin/tools/RunFunctions && getApplication"
    let foamApp=system(cmd)[:-2]
    echomsg 'This case is set to be simulated with: '.foamApp
    return foamApp
endfunction
~~~

- `cmd` is a string, representing the shell command that sources run functions and then executes `getApplication` in the current case dir.
- `foamApp` is the output of the previous command (solver name), the `[:-2]` part deletes the last two characters from the output of the command (by default, It will output `icoFoam @`, the last two characters are not needed).
- `echomsg` writes messages to the status line, and keeps them in messages list (run `:messages`).

## The general VIM compiler plugin for OpenFOAM

Copy `blockMesh.vim` to `foam.vim`.
- Don't forget `let current_compiler = "foam"`
- Declare a script variable, storing the solver name
`let s:foamApp = FOAMGetApplication()`
- Instead of `CompilerSet`, define a new function setting the compiler to the solver name, then call it.

~~~vim
function! FOAMSetCompiler()
    exe 'CompilerSet makeprg='.s:foamApp
endfunction
call FOAMSetCompiler()
~~~

- For the file line in the error format, use
`\\file:\ %f\.%*[a-zA-Z]%*[a-zA-Z\ .]\ line\ %l%*[.a-zA-Z0-9\ ]`
So we can cover stuff like "from line 15 to line 20." and be more dynamic with the number of dots in the filename (handle things like `0/U.boundaryField.inlet`). Done!! We are ready to go!

## Final tweaks: VIM's auto-commands

We only need to tell VIM to set the compiler to "foam" whenever a Foam File is opened; and to set the compiler to `blockMesh` if the entered buffer is named `blockMeshDict`.

~~~vim
augroup FOAMautocmds
autocmd!
autocmd FileType foam* call FOAMSetPathToCaseDir()
autocmd FileType foam* compile foam
autocmd BufEnter *blockMeshDict compile blockMesh
augroup End
~~~

> *FileType* autocmds will *always* be executed before *BufEnter* ones, so `blockMeshDict` files will always have `blockMesh` as their compiler.


The complete configuration files can be downloaded from here: <a href="https://drive.google.com/uc?id=1Sf-6Pm8twy8dM6YIcWZ8lMFaem2S2F0O&export=download" class="btn">VIM Customization</a> 
