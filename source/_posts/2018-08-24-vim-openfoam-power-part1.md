---
title: Customize VIM to work efficiently with OpenFOAM - Part 1
abstract: "Working with OpenFOAM case files gets really tedious if the text editor is not 	
	selected properly; and with VIM I still feel like missing some features, that's why 	
	customizing VIM for such tasks is the best path to take"
header_image: /assets/images/ionic2-banner.png
cover: https://s2.ax1x.com/2019/09/18/n7E8Wd.png
date: 2018/08/27
categories:
  - productivity
tags:
  - OpenFOAM
  - Vim
---

Today, I'm starting new (short) post series suggesting tricks to customize the **VIM** text editor so it can be used to edit OpenFOAM cases more efficiently. Of course, making VIM work optimally with every single piece of software you have installed through scripting is a bit of "over-kill"; but having it configured properly is crucial.

> These series of posts are for those who are ready to write some lines of VIML code to add interesting functionality to VIM (OpenFOAM-wise).


## VIM Plugins for OpenFOAM

We can easily customize our VIM installation using our own scripts, sourced in the `.vimrc` file. The versatility of VIM allows for sharing these scripts between users, we then call the scripts: *Plug-ins*.

These plugins differ in both their goal and their implementation:
- General purpose plugins. Eg. for handling text formatting, VIM configuration.
- File-Type specific plugins. Eg. those which operate only on C source/header files.
- Syntax highlighting. Eg. highlight OpenFOAM keywords in a OpenFOAM dictionary.
- Compiler plugins. Eg. interpret a python script from VIM command line and jump to errors.

As OpenFOAM users, we'll naturally be interested in any C++ related plugins in addition to few OpenFOAM-specific ones.

### Syntax Highlighting of OPENFOAM dictionaries in VIM

It's always good to have keyword-specific highlighting of a file; It helps you instantly catch misspelled keywords, it makes understanding the file easier; and with appropriate color themes, it makes you feel comfortable while editing the file.

There is already a VIM plugin that takes care of OpenFOAM keywords highlighting (Vim-OpenFOAM-syntax). It's true that the plugin does some weird stuff (eg. loading weird files into the buffer list) but, in general, it's good for the job of highlighting OpenFOAM syntax. To install it on your system, follow the instructions at [Vim-OpenFoam-syntax](https://github.com/effi/vim-OpenFoam-syntax) on github, or the fork for manual installation (not that scary) [Vim-Extension-OpenFOAM](https://bitbucket.org/shor-ty/vimextensionopenfoam) on bitbucket.

While programming, I usually use a couple of VIM plugins:
- *c-support*: to turn VIM into a C/C++ IDE (code snippets ... etc).
- *taglist*: to browse source code efficiently

## Your own VIM scripts

In this blog post, we'll write a very small VIM script, called `customFoam.vim`,  which is capable of doing a couple of things:

- It needs to be aware of the current file's FileType.
- When the file is of type `foam`, VIM should set a couple of things up:
  * Changing the path variable for the `:find` command so it includes our case's directory.
  * Changing VIM's current directory to case directory.
- At this moment, that's all, but we'll add some stuff in the upcoming blog posts.

### First steps: Determining file types

There are two common ways to determine a file type for a file:

- Use the extension in its same; I call it the `lazy Windows way`: for example ,files that have names ending with `.html` are considered HTML files no matter what their content is.
- Take a look at the file's content then decide; This is the Unix way, which we'll be using here: It just means we'll search the content of a file for the line that says `FoamFile`.


We are not the first ones who try to detect `foam` file type in VIM; the *vim-OpenFoam-syntax* plugin already does that. Actually, it assigns different file types for different `foam` files. For example, it sets: `ft="foam256_thermodynamicProperties"` for `thermodynamicProperties` dictionaries, but in general, its sets the file type to `foam256_general`.

Thus, if you have the plugin installed, you already have the required means to detect `foam` file type. However, for the sake of leaning VIM, we'll illustrate a very quick way to do it (but not that efficient!).

All what we have to do is to write a small script -which contains only one function- searching the first few lines of a file for the keyword `FoamFile`. 
This tells VIM that the file is a `OpenFOAM Dictionary File`:

~~~vim
" Filename: foamFT.vim

" No debug stuff
" Should check if the file type is already set
" But it's OK; just set it again

function! SetFoamType()
    " loop through the first 10 lines
    " FoamFile is at line 8 usually
    for nL in range(1,10)
    " match 'FoamFile'
        if (getline(nL) =~ 'FoamFile')
            setfiletype foam
            " if file type is set, leave the loop
            break
        endif
    endfor
endfunction

" That's it!
" Now, when to call it?
augroup FOAMFTautocmds
autocmd!
autocmd BufRead * call SetFoamType()
augroup End
~~~


To check whether the script works or not; Just execute the ex-command `:set ft` while an OpenFOAM dictionary is loaded. If it works, the output should say *filetype=foam*.

### Now we know the filetype, then what?

Well, **VIM** knows *what* we are editing, but it doesn't know *where* we are editing it. In fact, it does know the exact path to the current file (try running the ex-command `:pwd`): VIM considers the startup-directory as your current directory; and to change it, guess what, we have to use the `:cd` (change directory!) command.

If you invoke VIM from the case directory, you'll have the advantage of file names auto-completion in command-line mode; that is, when you type `:e c<TAB>`, VIM automatically completes *constant* for you. But, if, for example, you invoke it from the system directory of a case, it won't know about the files in the other directories (*0*, *constant* ... etc).


There's also the super-useful command `:find` which finds the specified file in the path (could include multiple -independent- directories) and then edit it. The desired work-flow can be summarized as follows: While editing `blockMeshDict` in *system* directory of a case; type `:find t<TAB>`, which instantly expands to `:find transportProperties` (Note that this file is located under *constant* directory), then, hitting *ENTER* switches to that file to edit it.


~~~vim
" Filename: customFoam.vim

function! FOAMSetPathToCaseDir()
    " IF you care only for files in
    " system, 0, and constant, use
    " let caseDir = expand('%:p:h:h')
    " But, I'm a regexp lover, so,
    " Make Vim remember everything in the path until case name.
    " delete everything else!
    let caseDir = expand('%:p:s?\(\/.*run\/[a-zA-Z1-9\. ]*\)\/.*?\1?')
    " set path to include all subdirs of caseDir.
    " you can also use '=+' instead of '=' to add the caseDir to
    " the default path; but I like it this way.
    exe 'set path='.caseDir.'/**'
    " I want to run blockMesh, solvers and other tools
    " Directly in VIM, so change dir to casedir
    exe 'cd '.caseDir
endfunction

" This is just incredible

augroup FOAMautocmds
autocmd!
autocmd FileType foam* call FOAMSetPathToCaseDir()
augroup End
~~~


Let's go through that script line-by-line, shall we?
- `function! FOAMSetPathToCaseDir()` overwrites any function defined with this name (Remember, user-defined function names start with a capital letter). `endfunction` denotes the end of function block.

- All lines preceded with ( `"` ) are considered comments.

- The line `let caseDir = expand('%:p:s?\(\/.*run\/[a-zA-Z1-9\. ]*\)\/.*?\1?')` needs deeper explanation: 
  1. The standard function `let` defines Vim Variables (In this case, variables are local to the user-defined function).
  2. *caseDir* is the name of our variable (a string). It is local to the function: To use a global variable inside a function, name it `g:var`, to make it available only in this script, use `s:var`, and, in fact, to use a local variable in a function, it should be called `l:var`, but this is tolerated.
  3. The standard function `expand` manipulates filenames (shortcut *%*) and paths to them, with modifiers:
    4. `:p` prints the whole absolute path to the file, eg. while editing *blockMeshDict*,
       the ex-command `echo expand('%:p')` should display something like
      `/home/path-to-case/system/blockMeshDict`.
    5. `:h` removes the last entry in a path, eg. `echo expand('%:p:h')` would display
      `/home/path-to-case/system`
    6. `:s?pattern?string?` acts like the substitute command. *pattern* is a regular expression, and *string* accepts **back-references**; All the power one needs in one command.
    7. So, the regexp `\(\/.*run\/[a-zA-Z1-9\. ]*\)\/.*` remembers everything from the start of filename to "run" (hopefully, your OpenFOAM cases directory) to the first directory (that's the case dir. at least for me). Case Name may contain alphabet characters, digits, dots and spaces (The last two are really bad options). The rest of the path is forgotten (eg. /system/blockMeshDict).



This way, we can find our path to the case no matter what case file we are editing!



- In the line `exe 'set path='.caseDir.'/**'` , the command `exe` (short for `execute`) executes the following string as an ex-command. Concatenation in VIML is done with a dot,  and because our *caseDir* variable is a string, and we are still inside the same function, it can be concatenated with other strings. The /** part expands to all sub-directories, to the last one of them, no matter how many levels of directories the case have.

- The path variable is set so the `:find` command can help us a bit.

- I like to change the directory to the case dir. so I can run commands directly from within VIM without worrying about which files I'm editing, or the start-up directory. For that, we simply use the `:cd` ex-command (VIM's command).



The function is completed, now, we have to find a way to automate calling it: `auto-commands`. Simply, we group all of our autocmds in a group called `FOAMautocmds` (Good practice but optional) .



- `autocmd!` overwrites previous autocmds of this group (often abbriviated as `au!` ).

- We want to call the function when we open a file of type `foam*` (including `foam`, `foam_general` ... etc), so we write: `autocmd FileType foam* call FOAMSetPathToCaseDir()`
That's it, source the script in your `.vimrc` file and you are good to test it out:

- From the shell (from /../constant/polyMesh in an arbitrary OpenFOAM case, where the mesh is present) run `vim boundary`
- Now, in Vim, type `:fin tr<TAB>` (`fin` is a short form for `find`) then hit ENTER.
- Check both the path and the working directory (using `:set path` and `:pwd` respectively).
- You can issue `:!blockMesh` while editing the file to rebuild the mesh.


That's it for now, the next post will explain ways to get the most of VIM compilers to run `blockMesh` and OpenFOAM solvers on cases right from inside VIM and browse errors.
