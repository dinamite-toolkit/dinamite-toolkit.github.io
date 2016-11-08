---
layout: page
permalink: quickstart/
---
# Quickstart

## Setting up

If you want to try DINAMITE out quickly, your best option is to grab our Docker.io container recipes at *link here*.

To build the Docker container, clone the repository locally. The final image will be around 5GB, and traces can get
big, so make sure you have some free space on your machine (in our experience ~20GB free space is usually enough for
basic use cases).

Once you have the repository in a local directory, navigate to it and run the following:

```
$ ./get_tarballs.sh     ## download LLVM sources
$ docker build .        ## build docker in the current dir
```

Sit back and relax while docker downloads a base image and builds LLVM with DINAMITE in it.
When the setup is done, we provide a script called `run_container.sh` that makes a mount directory
in the current location, and runs the container with it mounted to `/dinamite_workspace/`.

The script is provided as a shortcut for those not familiar with Docker. Feel free to adapt it to your own style / use case.

## Using DINAMITE

In order to compile C/C++ projects with DINAMITE, you need to change your compiler invocation somewhat.

The exact procedure will vary between build systems, but in most cases, you will have an environment
variable containing your compiler invocation command (something like `CC=gcc`).
Our Docker container comes with a bashrc files which sets up the following variables that usually
directly translate into variables used by build systems:

- `DIN_CC` - clang with options for invoking DINAMITE compiler pass
- `DIN_CXX` - clang++ with options for invoking DINAMITE compiler pass
- `DIN_LDFLAGS` - linker flags, setting the path to libinstrumentation.so and adding it as a build flag
- `LD_LIBRARY_PATH` - this is needed to be able to run DINAMITE. If you prefer not setting this variable, 
    an alternative would be copying libinstrumentation.so to a system-wide shared object path

With these environment variables at your disposal, building a single source file boils down to:

```
${DIN_CC} ${DIN_LDFLAGS} main.c
```
## DINAMITE output

The default build setup in our Docker container builds DINAMITE with a binary format logging library.
This library will get linked with the instrumented executable at runtime, and will output one file per
thread of execution. These files will contain logs made up of plain C structures (defined in 
${INST_LIB}/binaryinstrumentation.h) reflecting the sequence of events in the execution.

To process these logs into a human readable format, or perform any filtering or analysis on them,
you can use our binary trace analysis toolkit (*link_here*).

The binary trace analysis toolkit provides a harness for writing plugins to process and analyze DINAMITE's
default binary output. It was designed to be easily extensible, and comes with a couple of example plugins
that can be used out of the box.
