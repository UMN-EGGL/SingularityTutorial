# Singularity Tutorial -- Part II -- Part I
In Part I, we peeled back what containers were "under the hood", and build a container using the `--sandbox` option to prove that it was just a filesystem.
In this part, we will be building our own container using [singularity definition files](https://sylabs.io/guides/3.0/user-guide/definition_files.html).

Overall, there are 2 major steps in getting a container running. The first is building, the second in running.
If you are planning on running containers on your HPC (MSI), you will need to *build* them somewhere else, since building requires admin/root access.
To get going, start with a fresh text file called `Singularity.def` and open your browser to `https://cloud.sylabs.io/home`.

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



