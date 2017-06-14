---
layout: post
title: Compiling Facebook's RocksDB using DINAMITE
excerpt_separator: <!--more-->
---

In this blog post we describe how we use DINAMITE to compiled RocksDB, an open
source persistent key-value store developed by Facebook.
<!--more-->


# Building RocksDB

Here is the [RocksDB Github Repo](https://github.com/facebook/rocksdb/) of
RocksDB. If you have your `gcc` correctly set up, a
simple `make static_lib db_bench` instruction in the root of the repo can build a ''normal'' version of RocksDB static library and some db_bench sample applications.

## Build and Run With Docker

 1. A Docker container with LLVM and DINAMITE can be found [here](https://github.com/dinamite-toolkit/dinamite-compiler-docker.git). Run and setup the docker as suggested in the repo.

 2. In the docker command line, setting the following environment variables:

    ```shell
    export WORKING_DIR=/root/dinamite
    export LLVM_SOURCE=$WORKING_DIR/llvm-3.5.0.src
    export LLVM_BUILD=$WORKING_DIR/build
    ```

 3. Clone the RocksDB.

    ```shell
    cd $WORKING_DIR
    git clone https://github.com/facebook/rocksdb.git
    cd rocksdb

    ```

 4. Set up environment variables for building RocksDB with DINAMITE

    ```shell
    # Variables for DINAMITE build:
    export PATH="$LLVM_BUILD/bin:$PATH"
    export DIN_FILTERS="$LLVM_SOURCE/projects/dinamite/function_filter.json" # default filter
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

    # Create directories if not exist:
    mkdir $DIN_MAPS
    mkdir $DINAMITE_TRACE_PREFIX
    ```

    **Note**: If you are running on MacOS, please do the following changes:
    
    ```shell
    export EXTRA_CFLAGS="-O3 -g -v -Xclang -load -Xclang $LLVM_SOURCE/Release+Asserts/lib/AccessInstrument.dylib"
    export DYLD_LIBRARY_PATH=$INST_LIB
    ```
 
 5. Build the RocksDB library and some sample applications.

    ```shell
    make clean
    make static_lib db_bench
    cd examples
    make clean
    make
    cd ..
    ```
 
 6. Now run the simple example RocksDB program to get some traces!

    ```shell
    cd examples
    ./simple_example
    ```


## Build and Run in Your Own Environment

If you don't like docker (don't worry, we have the same feeling), you can definitely build DINAMITE instrumented RocksDB in your own environment.

 1. Basically you need to (1) build your LLVM; (2) Place DINAMITE pass to the correct place; (3) Build DINAMITE pass and its runtime library.

    You can copy the commands in the [Dockerfile](https://github.com/dinamite-toolkit/dinamite-compiler-docker/blob/master/Dockerfile) to apply them appropriately to your system. Details information is provided in the section *Environment* in another article [Compiling the WiredTiger key-value store using DINAMITE](https://dinamite-toolkit.github.io/2016/11/12/compiling-WT/).

 2. Make sure your are assigning those environment variables correctly:

    ```shell
    export WORKING_DIR="Your working directory, which will later contain rocksdb repo"
    export LLVM_SOURCE="Your LLVM source base directory"
    export LLVM_BUILD="The directory contains your LLVM build. This directory is supposed to have a directory called bin, which contains the clang/clang++ binary"
    ```

 3. Do the same things starting from (including) Step 3 in section *Build and Run With Docker*.


# Appendix

## DINAMITE Variables Explanation:

    * `DIN_FILTERS` is a configuration file in json format indicating what get filtered out for injecting instrumentation code
    * `DIN_MAPS` is the place where storing id to name maps, important for processing later traces.
    * `ALLOC_IN` is a directory contains alloc.in, which let you specify your allocation function
    * `INST_LIB` is the variable telling the runtime library DINAMITE uses for writing traces



If you run into trouble, get in touch. If you'd like to find out how to control what's instrumented, take a look at the
[User Guide](/user-guide/). If you'd like to know **what to do with the traces** now that you have them,
take a look at the other [Technical Articles](/tech-articles/)

