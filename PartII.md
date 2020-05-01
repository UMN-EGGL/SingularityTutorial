# Singularity Tutorial -- Part II 
In Part I, we peeled back what containers were "under the hood", and build a container using the `--sandbox` option to prove that it was just a filesystem.
In this part, we will be building our own container using [singularity definition files](https://sylabs.io/guides/3.0/user-guide/definition_files.html).

Overall, there are 2 major steps in getting a container running. The first is building, the second in running.
If you are planning on running containers on your HPC (MSI), you will need to *build* them somewhere else, since building requires admin/root access.
To get going, start with a fresh text file called `singularity.def` and open your browser to `https://cloud.sylabs.io/home`.

```
$ touch singularity.def
```

## Building a container image from a definition file
The easiest way to build an image is by modifying an existing one.
In fact, 99.99% of the time, building a new image will require you to start from a **base** image.
This is sort of a chicken / egg situation, which is common in computing.
The process is called a Bootstrap, a la pick yourself up by your own bootstraps.
Mostly, it is just convenient since you would want to start from a working linux filesytem anyways.

The first thing in your definition file is a Header section that definies how to boostrap your base image.
Singularity offers several options (including docker files) to bootstrap from, however, to simplify things, lets focus on bootstrapping from images in the official Singularity repository.
Pop open your browser to `https://cloud.sylabs.io/home`.
In the search bar, enter `ubuntu`.
The way things are organized is that container images are assigned simple names, like `ubuntu`.
Be warned ... *anyone* can upload images to this repository, so make sure you trust your base image.
You can see who owns the image from the full path of the image: `library://library/default/ubuntu`.
By clicking the user `library` you can see it is the "Library-wide default user".
This is contrast to one of the other `ubuntu` images in the repository.
For example, `library://dtrudg/linux/ubuntu`.
I have no idea who `dtrudg` is, so I would be weary before I download and run any images from them.
The same idea goes for downloading and running code from github.

Ok, so we want to use `library://library/default/ubuntu` as our base image.
How do we get this into our definition file?
The header section of the file has some shortcuts to define bootstrap/base images.

```
# In: singularity.def
Bootstrap: library                                                              
From: ubuntu:20.04

```

Here, there are two directives. 
`Bootstrap: library` says we want to bootstrap out base image from `library` which is shorthand to the singularity container library.
`From: ubuntu:20.04` is some more shorthand.
This protects you from the problem described above.
This defaults to a longer, full path to a library image.
It is the equivalent to: `From: library/default/ubuntu:20.04`.
So what is the `20:04`?
That specifies an image "tag".
The owner of that container image can upload updated versions of their images.
This helps keep images updated, as well as providing historical images for reproducibility.
The conventional tags for ubuntu correspond to `year:month` of that release.
So with `20:04` we are using an image from April 2020.

Our definition file could have been:
```
# In: singularity.def
Bootstrap: library                                                              
From: library/default/ubuntu:20.04

```

This is all we need to build a container. 
Let's use the same command from part 1 to build a container using `--sandbox` so we can have a look around.

```
$ sudo singularity build --sandbox base_image singularity.def
[sudo] password for rob: 
INFO:    Starting build...
INFO:    Creating sandbox directory...
INFO:    Build complete: base_image
```
We need `sudo` here because in order to modify the filesystem inside the image, we need to be an admin.
The output of this is a directory that contains our container image filesystem.
Let's take a peek inside:

```
$ ls base_image 
bin   dev          etc   lib    lib64   media  opt   root  sbin         srv  tmp  var
boot  environment  home  lib32  libx32  mnt    proc  run   singularity  sys  usr
```

To run this container interactively, use the `singularity shell` command.

```
$ singularity shell base_image 
Singularity> ls /
bin   dev          etc   lib    lib64   media  opt   root  sbin         srv  tmp  var
boot  environment  home  lib32  libx32  mnt    proc  run   singularity  sys  usr
```
The prompt `Singularity>` lets you know you are in a container.
We can list the root directory `/` and see it is the same as we saw before. 
Let's explore

```
Singularity> whoami
rob
Singularity> pwd
/home/rob/Codes/SingularityTutorial
Singularity> ls
PartI.md  PartII.md  base_image  singularity.def
Singularity> ls ~
 vmware                   Desktop      Music      Videos                 
 Codes                    Documents    Pictures  'VirtualBox VMs'   
 CytoscapeConfiguration   Downloads    Project    snap
 Cytoscape_v3.7.1         Lightworks   R          igv               
```

While you are in the container, you are ... you. 
This might not be surprising, but it is different than how other container systems work.
Another thing to notice is that we have access to the directory where we started the container from.
We also access to our home directory (yours will look different than mine, which is the point!). 

So this sort of goes against the pure philosophy of what containers do and how they should work.
However, this is extremely pragmatic.
It allows you to access data outside your container, and in the most likely places.

We also get the benefit of being in a container. 
We have access to all the programs that were built when we build the image.
Lets re-build our container to contain a new program!

```                                                                             
# In: singularity.def                                                           
Bootstrap: library                                                                                      
From: ubuntu:19.10                                                                                      

%post
    apt-get update && apt-get upgrade --yes    
    apt-get install curl wget \                                                 
    openjdk-8-jre \                                                             
    gcc zlib1g-dev libbz2-dev liblzma-dev \                                     
    build-essential \                                                           
    unzip --yes                                                                 
                                                                                
    # install bcftools (system wide)                                            
    wget https://github.com/samtools/bcftools/releases/download/1.9/bcftools-1.9.tar.bz2
    tar xvvf bcftools-1.9.tar.bz2                                               
    cd bcftools-1.9                                                             
    ./configure                                                                 
    make                                                                        
    make install                                                                                
``` 
