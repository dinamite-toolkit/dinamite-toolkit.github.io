---
layout: post
title: Compiling the WiredTiger key-value store using DINAMITE
excerpt_separator: <!--more-->
---

In this blog post we describe how we compiled with DINAMITE a commercial
key-value store WiredTiger, which uses `autoconf`, `automake` and `libtool`
for its build system.
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

This way, the instrumentation pass itself will be built using g++, but the instrumented programs will be built using the clang installed in `/usr/local`, provided that it is the default version that your system finds. If not, put `/usr/local` as the first item on your path. 

# Building the WiredTiger library

Instructions for building WiredTiger with ''normal'' compilers can be found
[here:](http://source.wiredtiger.com/2.8.0/build-posix.html).

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
 1. Build the instrumentation pass and the instrumentation library.
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

 2. Next, you need to clone the WiredTiger tree and run autogen.sh as detailed in [these instructions](http://source.wiredtiger.com/2.8.0/build-posix.html). 
    After you have done so, you need to change how you invoke the ''configure'' command. Instead of invoking it as shown in the build instructions for
	''normal'' compilers, you need to pass it a few environmental variables, like so: 
	
	    INST_LIB=$LLVM_SOURCE/projects/dinamite/library
	    LD_LIBRARY_PATH=$INST_LIB LDFLAGS="-L$INST_LIB" LIBS="-linstrumentation"
	    CFLAGS="-O0 -g -Xclang -load -Xclang  $LLVM_SOURCE/Release+Asserts/lib/AccessInstrument.so" CC="clang" ../configure

    **Important**: The preceding command tells the `configure` command
    to use the `clang` compiler. If you have more than one version of
    clang installed, make sure that you point the configure to the
    **version of clang that comes with the DINAMITE installation**
    (currently 3.5.0). Otherwise you won't be able to configure or
    build the programs.

    ** Important**: If you have your clang compiler installed
    somewhere other than the default system path, like `/usr/local`,
    you need to add your installation path to your path variable. You
    also need to add the path to LLVM libraries to the LD_LIBRARY_PATH
    variable in the above configuration script. Otherwise, your
    configure script will complain that it can't find the compiler or
    the compilation will fail later because of linkage errors.

    **Important**: Some systems won't like the fact that you are setting the
    INST_LIB variable and attempting to use it on the same command line.
    In that case, just replace $INST_LIB with the actual path to which it
    corresponds.

    **Important**: If you are building on an OS X, you should use the
    ''DYLD_LIBRARY_PATH'' instead of the ''LD_LIBRARY_PATH''.

 3. You are almost ready to build. Before you do, you need to make a small change
    to libtool, which is located in the build_posix directory of your WiredTiger tree.
    Open the libtool file in the editor and search for code that looks sort of
    like this:

    ```
      elif test X-lc_r = "X$arg"; then
	   	case $host in
		*-*-openbsd* | *-*-freebsd* | *-*-dragonfly* | *-*-bitrig*)
		    # Do not include libc_r directly, use -pthread flag.
		    continue  
            	    ;;
          	esac
      fi
      func_append deplibs " $arg"
      continue
      ;;
    ```

    Instead of the "func_append" line, add the following code snippet:

    ```
     if [ "$arg" != "-load" ]; then
           func_append deplibs " $arg"
     else
           func_append compile_command " $arg"
           func_append finalize_command " $arg"
     fi
    ```

     What is happening here is that libtool is trying to be smart and
   	interpret the -load option to clang as a directive to link to liboad.so.
	By inserting this if-statement, you are instructing the libtool to not
	interpret this option as such.

4. Now invoke the make command as follows:

    ```
    INST_LIB=$LLVM_SOURCE/projects/dinamite/library DIN_FILTERS="/full/path/to/function_filter.json" make
    ```
    
    The INST_LIB variable tells the compiler where the instrumentation library lives.
    If you are using Docker, it's already properly set in your environment.
    The DIN_FILTERS variable tells the instrumentation pass what to instrument. If you
    don't provide the filters file, the compiler will insturment all function calls
    and memory accesses in your program, which may not be what you want.

    Here is the
    contents of a ```function_filter.json``` file with reasonable defaults. With this
    file, DINAMITE will instrument all functions whose critical path is at least 40
    lines of code (LOC), where LOC does not include blank lines or comments and LOC
    for loops are computed with the assumption that they execute once.

    ```
     {
         "minimum_function_size" : 50,
         "check_small_function_loops" : true,
         "function_size_metric" : "LOC_PATH",
         "whitelist": {
              "function_filters" : {
                   "*" : {
                        "events" : [ "function" ]
                   }
              }
          }
     }
    ```
6. Next, run your program! If you run a command that uses the instrumented WiredTiger library, you need to provide the path for the instrumentation library. For example, suppose you run wtperf:

        ```
        LD_LIBRARY_PATH=$LLVM_SOURCE/projects/dinamite/library ./wtperf
        ```

        Or, if using a MacOS:

        ```
        DYLD_LIBRARY_PATH=$LLVM_SOURCE/projects/dinamite/library ./wtperf
        ```

        To tell the instrumentation library where to put the resulting performance traces,
        that will be generated when your program runs, you can set the DINAMITE_TRACE_PREIX
        variable to the desired location:

        ```
        DINAMITE_TRACE_PREFIX=/path/to/traces DYLD_LIBRARY_PATH=$LLVM_SOURCE/projects/dinamite/library ./wtperf
        ```

That's it! Now watch your program run and generate traces.

If you run into trouble, get in touch. If you'd like to find out how to control what's instrumented, take a look at the
[User Guide](/user-guide/). If you'd like to know **what to do with the traces** now that you have them,
take a look at the other [Technical Articles](/tech-articles/)

