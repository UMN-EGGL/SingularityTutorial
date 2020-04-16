# Singularity Tutorial
This repository contains a brief tutorial on what singularity is and why you
would want to use it. To get started `clone` this repository. We will be
building and running a singularity image together.

## Setting the stage: what happens when you run a "little" script
Lets say you log into MSI to run a python script.
In order to stay organized and to be re-producible, you have everything you need for you analysis in one script.
For convenience, you wrote the script on your laptop and now want to run it on a big data set.

```
$ python myscript.py
```

When you run this program **a lot** of things are happening.
Python is a program that accepts valid python code.
It's an interpreted language, which means there is a binary (or executable) that takes your script and turns it into machine executable code on the fly.
Pretty cool, except that means that for each working python script, you need a matching, compatible python executable.
Maybe you took note of the version of Python you had running on your laptop.
But even that can change!
Typically, there are "default" versions of python that are distributed by the [Python foundation](https://www.python.org/downloads/).
And most systems have *multiple* versions of python on them!
My machine running Ubuntu comes with at least two system installed versions of Python.

```
$ /bin/python --version
Python 2.7.16

$ /bin/python3 --version
Python 3.7.3
```

But did you know there are also versions that are distributed by third parties?
Maybe you are using them and don't even know it!
If you are using conda virtual environments, you are using a python binary that is distributed by [Continuum Analytics](https://www.anaconda.com/distribution/).

```
$ which python
/home/rob/.conda/bin/python

$ /home/rob/.conda/bin/python --version
Python 3.7.4
```

Who are they?
Why do they have their own versions of python?
What are the differences?

This doesn't stop at different versions of Python. If your script uses any libraries, those too are probably going to slightly differ between systems.
When you import a library in python, the interpreter looks in a couple of different places for those packages.

```
$ ipython
Python 3.7.3 | packaged by conda-forge | (default, Dec  6 2019, 08:54:18)
Type 'copyright', 'credits' or 'license' for more information
IPython 7.7.0 -- An enhanced Interactive Python. Type '?' for help.

In [1]: import sys

In [2]: sys.path
Out[2]:
['/home/rob/.conda/bin',
 '/home/rob/.conda/lib/python37.zip',
 '/home/rob/.conda/lib/python3.7',
 '/home/rob/.conda/lib/python3.7/lib-dynload',
 '',
 '/home/rob/.local/lib/python3.7/site-packages',
 '/home/rob/.conda/lib/python3.7/site-packages',
 '/home/rob/Codes/Minus80',
 '/scratch/EGGLImputationPipeline/WatchDog',
 '/home/rob/.conda/lib/python3.7/site-packages/IPython/extensions',
 '/home/rob/.ipython']
```

And just to drive this point home, many python packages provide convenient ways to access underlying libraries written in C or C++.
For instance, if you are using `numpy`, there are links to low-level (and fast!) numerical computing libraries that aren't written or affiliated with Python.
So, if reproducibility is so important, why not create your work environment in a viurtual machine and distribute that?
VMs are close to solving this problem, they check off a lot of the boxes for what we need.
However, VM images tend to be very big, and while they are running they inefficient (everything needs to go through a virtualization layer), so they are slow.
What if we could have something in the middle?
A computing environment that was running natively on the computer your were on, just somehow partitioned off on its own?
What would that take?

## Enter, the containers!
Ok, so when you open a terminal and start entering commands, what is happening?
Just like we saw above, when you type in a command, the terminal is doing some behind the scenes work to make things more human compatible.
For instance, you'd much rather just write `python` than having to remember each time that your defualt python is in `/home/rob/.conda/bin/python`.
The terminal knows that you want to start `python`, so it starts the first one that it finds in the `PATH` `env` variable.
There are a lot of `env` variables!

Another one that gets used all the time is `PWD`.
When you execute the command `ls`, a program gets run.
It's the `ls` program, and the terminal resolves it to `/bin/ls`.
If you read the manual page for `ls`, you see something strange.

```
/bin/ls --help
Usage: /bin/ls [OPTION]... [FILE]...
List information about the FILEs (the current directory by default).
Sort entries alphabetically if none of -cftuvSUX nor --sort is specified.

Mandatory arguments to long options are mandatory for short options too.
  -a, --all                  do not ignore entries starting with .

[... truncated ...]
```
`ls` **requires** a directory to list files from.
This is the `[FILE]` portion of that usage string (in linux directories are types of files).
When you run `ls` without its required argument, the `env` variable `PWD` gets substituted in for you.

`PWD` is the path of the directory you are currently in.
```
$ echo $PWD
```
So, the following two commands are equivalent:
```
$ ls
 Codes                    Cytoscape_v3.7.1   Documents   Music      Project   Videos            vmware
 CytoscapeConfiguration   Desktop            Downloads   Pictures   R         snap
$ ls $PWD
 Codes                    Cytoscape_v3.7.1   Documents   Music      Project   Videos            vmware
 CytoscapeConfiguration   Desktop            Downloads   Pictures   R         snap
```

Furthermore, when you run the `cd` command, you are not doing anything special.
The `cd` command primarily updates the `PWD` variable in your shell.
So, `$ cd ..` just removes the trailing directory from your current path.
Essentially, just turning `/home/rob/` into `/home`.
Now, if you run `ls` after doing a `cd`, you get the listing of the directory you just changed to.

This all works because the filesystem in Linux is arranged as a tree, with the root of the tree being `/`.
So when a program gets installed in your home directory, it can make big assumptions about the layout of other things on the system.
Like, it can expect that `/tmp` would be a good place to put temporary files (stuff in `/tmp` conventionally gets routinely wiped).

Likewise, the terminal can expect that things in `/bin` are executable programs.
If you say, "run MyCoolProgram", the terminal can assume that if it finds a file called `/bin/MyCoolProgam` (in your `$PATH`), that is what you wanted.

Whoah woah woah, hold up. I know what you are thinking, "I thought this was a tutorial on containers?".
Bear with me.
What if you could just tell the terminal,
    "hey, I know you are assuming that everything that I am doing is relative to `/`.
     But I want you to change `/` to be `/home/rob/nestedFilesystem/`."
And, what if inside of `/home/rob/nestedFilesystem/` there were directories for `/bin` and `/home` and `/tmp`?
What if you had a complete and trimmed down filesystem in there that was totally catered to a specific task?
By telling the terminal to change its `/` to `/home/rob/nestedFilesystem/` you have the benefit of the terminal being managed by
    the host OS (no virtualization) as well as a sort of nested compartmentalized system where everthing inside that nested directory is relative to its new root directory.

There is a actually a linux command, `chroot`, that accomplishes this.
For our purposes, this is a pretty good container.
We can put everything we need for a specific analysis in a directory and just tell the terminal to ignore everything else.
Well, it turns out, that while the filesystem is an important piece of implementing a container, it doesn't get you 100% of the way there.
[Here](https://ericchiang.github.io/post/containers-from-scratch/) is a good post of the other things that need to be changed in order to have fully isolated containers.
And, as we all know, if you need to do more than a couple of things in computing, you just write a program to do all the boring stuff for you!

## Singularity
Hopefully, you have singularity installed on your system:

```
$ singularity --help

Linux container platform optimized for High Performance Computing (HPC) and
Enterprise Performance Computing (EPC)

Usage:
  singularity [global options...]

Description:
  Singularity containers provide an application virtualization layer enabling
  mobility of compute via both application and environment portability. With
  Singularity one is capable of building a root file system that runs on any
  other Linux system where Singularity is installed.

[... truncated ...]
```

Hey, look at that! They talk about "root file systems" in the description ;)

Let's download a container image and prove that its just a filesystem with some other bells and whistles.

```
$ singularity pull ubuntu:19.04
$ ls
ubuntu_latest.sif
```

This downloads an `.sif` file, which is an "image" from the sylabs.io online repository. 
A "container" is an "image" thas is running.
As it stands, this image is packaged up and read-only. 
It's main purpose is as a base image which we will talk about in a sec.

We can create a duplicate image from it, with some additional flags to let singularity know we want to play around.

```
$ singularity build --sandbox lets_play ubuntu_latest.sif
$ ls
INFO:    Starting build...
INFO:    Creating sandbox directory...
INFO:    Build complete: lets_play
$ ls                                                                                                                                                                                                        !10058
lets_play  ubuntu_latest.sif
```
While the `.sif` file is a nice packaged "image", we built a new image called `lets_play`.
We also enabled `--sandbox` mode which tells singularity not to package everything up.
We end up with a new "image" called `lets_play`, which is ... you guessed it!
A root directory!
