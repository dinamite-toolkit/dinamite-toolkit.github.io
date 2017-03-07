---
layout: post
title: Building Facebook's RocksDB using DINAMITE
excerpt_separator: <!--more-->
---

In this blog post we describe how we use DINAMITE to compiled RocksDB, an open
source persistent key-value store developed by Facebook.
<!--more-->

## Environment

You will need LLVM 3.5 is needed to build and use DINAMITE.
DINAMITE can be built only within the LLVM's source tree.

A Docker container with LLVM and DINAMITE can be found
[here](https://github.com/dinamite-toolkit/dinamite-compiler-docker.git).
Use the Dockerfile-wt docker file to prepare for building WiredTiger.

If you don't like Docker or would like to install LLVM 3.5 and
DINAMITE natively for other reasons, follow the steps to download the
code outlined in the Dockerfile-wt found
[here](https://github.com/dinamite-toolkit/dinamite-compiler-docker.git).
Modify the steps for your OS and and then follow the build
instructions on the [LLVM "Getting Started"
website](http://llvm.org/docs/GettingStarted.html#local-llvm-configuration).

### MacOS note:

We are investigating a strange anomaly on OS X, where the main
compiler pass would only build with the built-in system clang or g++,
while the instrumented programs would only build with the clang
included in the LLVM distribution. To work around it, do the
following:

To build LLVM 3.5.0 natively on a MacOS, build and install clang as
part of building the LLVM using cmake as explained on the LLVM's
"Getting started" page:
 
    cmake -G "Unix Makefiles" -DCMAKE_INSTALL_PREFIX=/usr/local

Then, in the top-level directory of the LLVM source, run:

    CXX=g++ ./configure

This way, the instrumentation pass itself will be built using g++, but
the instrumented programs will be built using the clang installed in
`/usr/local`, provided that it is the default version that your system
finds. If not, put `/usr/local` as the first item on your path.


# Building the RocksDB

Here is the [RocksDB Github Repo](https://github.com/facebook/rocksdb/) of
RocksDB. If you have your `gcc` correctly set up, after cloning in the repo, a
simple `make` instruction can build a ''normal'' version of RocksDB.

In this article, we will refer to the root of your LLVM installation using the
`$LLVM_SOURCE variable`. If you are using the [Docker container](https://github.com/dinamite-toolkit/dinamite-compiler-docker.git), this variable is
already set up for you; it would point to `/root/dinamite/llvm-3.5.0.src`. If you
built the LLVM from sources, set `$LLVM_SOURCE` to wherever your compiled sources are.

 1. If you are using the [Docker container](https://github.com/dinamite-toolkit/dinamite-compiler-docker.git)
 the compiler pass is already included and built, but if you built your own version
 of LLVM from sources, download the instrumentation pass and the instrumentation
 library as follows:

    ```
    cd $LLVM_SOURCE/projects
    git clone https://github.com/dinamite-toolkit/dinamite.git
    ```
 2. Build the instrumentation pass and the instrumentation library.
 We will build the version of the library that produces logs in a binary format,
 because this is a lot more efficient than generating text traces. But if you don't
 want to deal with converting binary traces to text just yet, replace `make binary`
 command below with `make text`. **If you are using Docker, the library is already
 built for you**, so you may skip this step, unless you want to change the format
 of the traces.
 
    ```
     cd $LLVM_SOURCE/projects/dinamite/library
     make binary
     cd ../
     make
    ```

 3. Clone the [RocksDB Github Repo](https://github.com/facebook/rocksdb/) to
 your working directory (We will assume $WORKING_DIR to be your working
 directory in later article). And do:
    ```shell
     mkdir $WORKING_DIR/din_maps
     mkdir $WORKING_DIR/din_traces
     cd $WORKING_DIR/rocksdb
    ```
 
 4. Set up environment variables for building RocksDB with DINAMITE.

    ```shell
    # Variables for DINAMITE build:
    export PATH="LLVM_SOURCE/../build/bin:$PATH"
    export DIN_FILTERS="$LLVM_SOURCE/projects/dinamite/function_filter.json" 
    export DIN_MAPS="$WORKING_DIR/din_maps"
    export ALLOC_IN="$LLVM_SOURCE/projects/dinamite"
    export INST_LIB="$LLVM_SOURCE/projects/dinamite/library"

    # variables for loading DINAMITE pass on RocksDB:
    export CXX=clang++
    export EXTRA_CFLAGS="-O3 -g -v -Xclang -load -Xclang $LLVM_SOURCE/Release+Asserts/lib/AccessInstrument.so"
    export EXTRA_CXXFLAGS=$EXTRA_CFLAGS
    export EXTRA_LDFLAGS="-L$LLVM_SOURCE/projects/dinamite/library/ -linstrumentation"

    # Set up variables for runtime, can be set up later before running
    export DINAMITE_TRACE_PREFIX=$WORKING_DIR/din_traces
    export LD_LIBRARY_PATH=$INST_LIB
    ```

    *Note*: If you are running on MacOS, please to the following changes:
    
    ```shell
    export EXTRA_CFLAGS="-O3 -g -v -Xclang -load -Xclang $LLVM_SOURCE/Release+Asserts/lib/AccessInstrument.dylib"
    export DYLD_LIBRARY_PATH=$INST_LIB
    ```

    DINAMITE build Variables explanation:

        * `PATH` change is to make sure the correct clang++ is found
        * `DIN_FILTERS` is a configuration file in json format indicating what get filtered out for injecting instrumentation code
        * `DIN_MAPS` is the place where storing id to name maps, important for processing later traces        
        * `ALLOC_IN` is a directory contains alloc.in, which let you specify your allocation function
        * `INST_LIB` is the variable telling the runtime library DINAMITE uses for writing traces

 5. Build the RocksDB library and some sample applications.

    ```shell
    make static_lib db_bench -j4 # assume your machine has 4 cores
    cd examples
    make clean
    make -j4
    cd ..
    ```
 
 6. Now run the simple example RocksDB program to get some traces!

    ```shell
    cd examples
    ./simple_example
    ```

If you run into trouble, get in touch. If you'd like to find out how to control what's instrumented, take a look at the
[User Guide](/user-guide/). If you'd like to know **what to do with the traces** now that you have them,
take a look at the other [Technical Articles](/tech-articles/)

