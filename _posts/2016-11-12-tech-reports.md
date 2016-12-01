---
layout: post
title: Compiling the WiredTiger key-value store using DINAMITE
---


## Environment

You will need LLVM 3.5 is needed to build and use DINAMITE.
DINAMITE can be built only within the LLVM's source tree.

A Docker container with LLVM and DINAMITE can be found [here](https://bitbucket.org/datamancers/dinamite-compiler-docker). Use the Dockerfile-wt docker file to prepare for building WiredTiger.

If you don't like Docker or would like to install LLVM 3.5 and DINAMITE natively
for other reasons, follow the steps to download the code outlined in the Dockerfile-wt found [here](https://bitbucket.org/datamancers/dinamite-compiler-docker).
Modify the steps for your OS and and then follow the build instructions on the
[LLVM "Getting Started" website](http://llvm.org/docs/GettingStarted.html#local-llvm-configuration).

### MacOS note:

We are investigating a strange anomaly on OS X, where the main compiler pass would only build with the built-in system clang or g++, while the instrumented programs would only build with the clang included in the LLVM distribution. To work around it, do the following:

To build LLVM 3.5.0 natively on a MacOS, build and install clang as part of building the LLVM using cmake as explained on the LLVM's "Getting started" page:
 
    cmake -G "Unix Makefiles" -DCMAKE_INSTALL_PREFIX=/usr/local

Then, in the top-level directory of the LLVM source, run:

    CXX=g++ ./configure

This way, the instrumentation pass itself will be built using g++, but the instrumented programs will be built using the clang installed in `/usr/local`, provided that it is the default version that your system finds. If not, put `/usr/local` as the first item on your path. 

# Building the WiredTiger library

Instructions for building WiredTiger with ''normal'' compilers can be found [here:](http://source.wiredtiger.com/2.8.0/build-posix.html). We assume you 
building on a Posix system and that you are using a [Docker container](https://bitbucket.org/datamancers/dinamite-compiler-docker).

We will refer to the root of your LLVM installation using the $LLVM variable.
If you are using the Docker container, the root of your installation will be
at /root/dinamite.
    
 1. Build the instrumentation pass and the instrumentation library.
 We will build the version of the library that produces logs in a binary format,
 because this is a lot more efficient than generating text traces.
	
	    cd $LLVM/llvm-3.5.0.src/projects/dinamite/library
	    make binary
            cd ../
            make

 2. Next, you need to clone the WiredTiger tree and run autogen.sh as detailed in [these instructions](http://source.wiredtiger.com/2.8.0/build-posix.html). 
    After you have done so, you need to change how you invoke the ''configure'' command. Instead of invoking it as shown in the build instructions for
	''normal'' compilers, you need to pass it a few environmental variables, like so: 
	
	    INST_LIB=$LLVM/llvm-3.5.0.src/projects/dinamite/library
	    LD_LIBRARY_PATH=$INST_LIB LDFLAGS="-L$INST_LIB" LIBS="-linstrumentation"
	    CFLAGS="-O0 -g -Xclang -load -Xclang  LLVM/llvm-3.5.0.src/Release+Asserts/lib/AccessInstrument.so"
	    CC="clang" ../configure

    **Important**: Some systems won't like the fact that you are setting the
    INST_LIB variable and attempting to use it on the same command line.
    In that case, just replace $INST_LIB with the actual path to which it
    corresponds.

    **Important**: If you are building on an OS X, you should use the
    ''DYLD_LIBRARY_PATH'' instead of the ''LD_LIBRARY_PATH''.

    **Important**: If you have your clang compiler installed somewhere other
    than the default system path, like `/usr/local`, you need to add your
     installation path to your path variable and the path to LLVM libraries to
     the LD_LIBRARY_PATH variable in the above configuration script. Otherwise,
     your configure script will complain that it can't find the compiler or the
     compilation will fail later because of linkage errors.

  3. You are almost ready to build. Before you do, you need to make a small change
  to libtool, which is located in the build_posix directory of your WiredTiger tree.
  Open the libtool file in the editor and search for code that looks sort of
  like this:
  
           elif test X-lc_r = "X$arg"; then
	             case $host in *-*-openbsd* | *-*-freebsd* | *-*-dragonfly* | *-*-bitrig*)
		    # Do not include libc_r directly, use -pthread flag.
		    continue  
            ;;
          esac
        fi
        func_append deplibs " $arg"
        continue
        ;;

   Instead of the "func_append" line, add the following code snippet:

     	if [ "$arg" != "-load" ]; then
                func_append deplibs " $arg"
        else
                func_append compile_command " $arg"
                func_append finalize_command " $arg"
        fi

    What is happening here is that libtool is trying to be smart and
   	interpret the -load option to clang as a directive to link to liboad.so.
	By inserting this if-statement, you are instructing the libtool to not
	interpret this option as such.

 4. Now invoke the make command as follows:
	
	    INST_LIB=/root/dinamite/llvm-3.5.0.src/projects/dinamite/library make
        
 6. If you run a command that uses the instrumented WiredTiger library, you need to provide the path for the instrumentation library. For example, suppose you run wtperf:
 
        LD_LIBRARY_PATH=/root/dinamite/llvm-3.5.0.src/projects/dinamite/library ./wtperf

    Or, if using a MacOS:

        DYLD_LIBRARY_PATH=/root/dinamite/llvm-3.5.0.src/projects/dinamite/library ./wtperf


## Controlling the instrumentation

The DINAMITE compiler can perform different kinds of instrumentations. For example, it instrument every memory access, or just function entry/exit timestamps, or both. It can also use different formats for the traces. 

### Controlling the trace format.

A text trace will produce a text output, but will slow down the execution many times more than the binary output format. To control the format of the instrumentation trace, compile the instrumentation library in the `INST_LIB` directory:

    cd $INST_LIB
    make text 

or:

    cd $INST_LIB
    make binary 

For testing purposes, if you want the binary instrumented, but not producing any output:

    cd $INST_LIB
    make null

This command will produce `libinstrumentation.so' that is linked into the instrumented binary as you saw from the compilation examples above.

### Controlling what is instrumented.

By default DINAMITE will instrument every memory access in your program. However, often you only want to instrument parts of the code, and not all memory accesses, but simply function entry/exit timestamps. 

In order to instrument only parts of your code, and specific events, DINAMITE supports function filtering.
To leverage this, you need to provide a filter file. The filter basically works as a white list for functions that are allowed to be instrumented.
It is stored in JSON format and looks something like this:
```
{
    "function_filters" : {
            "foo" : {
                "events" : [
                        "function"
                    ]
            },
            "bar" : {
                "events" : [
                        "alloc"
                    ]
            },
            "baz" : {
                "events" : [
                        "access", "function"
                    ]
            }

    }
}
```
`foo`, `bar` and `baz` are function names in your code. 

You will need to populate the `function_filters` with a dictionary of function names mapped to lists of events to instrument. Events can be either `alloc`, `access` or `function`.

In order to tell DINAMITE about the function filters, at the time you compile your program with DINAMITE, you will need to set the environment variable `DIN_FILTERS` to point to the JSON document, like so:

    DIN_FILTERS="/path/to/function_filter.json" INST_LIB=/root/dinamite/llvm-3.5.0.src/projects/dinamite/library make -j 4

If you don't set `DIN_FILTERS`, DINAMITE assumes that you want to instrument all the events in your program. **It is safest to provide the full path of the `function_filters.json` file.** If you are building a complex project the build might be performed in multiple directories. If only the top-level directory has the filters file, the files in the other directories will not be filtered correctly.

With C++ code, function names are typically mangled. To find a real function name, run compilation once with `DIN_FILTERS` set. The filtering tool will output `functions.out` which will contain the names of all encountered functions. You can use these to fill your filter list properly.

#### Instrumenting argument values 

In order to instrument the values of specific (or all) arguments to certain functions, add the following into the filters JSON document:

```
            "function_name" : {
                "events" : [
                        "function"
                    ],
                "arguments" : [ 0, 1, ...]
            }
```

It is important to enable function event filtering for the given function, as without that argument printing will be disabled.
Then, to enable instrumenting a subset of arguments, you add a new field to the function filter object, called `arguments` and set it to an array of integers (zero-indexed) that enumerate the arguments you wish to instrument. Alternatively, you can set the value of `"arguments"` to a string value of `"*"` (`"arguments" : "*"`). This will tell DINAMITE to instrument all the arguments of the given function.
