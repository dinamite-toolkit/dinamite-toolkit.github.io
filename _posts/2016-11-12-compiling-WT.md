---
layout: post
title: Compiling the WiredTiger key-value store using DINAMITE
excerpt_separator: <!--more-->
---

In this blog post we describe how we compiled with DINAMITE a commercial
key-value store WiredTiger, which uses `autoconf`, `automake` and `libtool`
for its build system.
<!--more-->

## Pre-requisites.

We assume that you have already installed DINAMITE and LLVM. If not,
follow the steps described in our
[earlierpost](/2016/11/11/installing-DINAMITE).  If you completed
those steps, your `$LLVM_SOURCE`, `INST_LIB` and other environment
variables would be set to point to the location of LLVM 3.5 sources
and the instrumentation library your system.

## Configuring the build

Instructions for building WiredTiger with ''normal'' compilers can be
found
[here:](http://source.wiredtiger.com/2.8.0/build-posix.html). You will
follow these steps with some modifications.

Clone the WiredTiger tree and run `autogen.sh` as detailed in
[these instructions](http://source.wiredtiger.com/2.8.0/build-posix.html).
After you have done so, you need to change how you invoke the
`configure` command. Instead of invoking it as shown in the build
 instructions for ''normal'' compilers, you need to pass it a few
environmental variables, like so:

```shell
INST_LIB=$LLVM_SOURCE/projects/dinamite/library
LD_LIBRARY_PATH=$INST_LIB LDFLAGS="-L$INST_LIB"
LIBS="-linstrumentation -lpthread" CFLAGS="-O0 -g -Xclang -load -Xclang
$LLVM_SOURCE/Release+Asserts/lib/AccessInstrument.so" CC="clang"
../configure
```

If the `clang 3.5` (that you installed with DINAMITE) is not the
default `clang` on your system, explicitly point the command to the
`clang` you installed by modifying the `CC`,  `LD_LIBRARY_PATH` and `LDFLAGS`
variables in the above command like so:

```
CC="~/Work/DINAMITE/LLVM/build/bin/clang"
LD_LIRBARY_PATH=$INST_LIB:${HOME}/Work/DINAMITE/LLVM/build/lib
LDFLAGS="-L$INST_LIB -L${HOME}/Work/DINAMITE/LLVM/build/lib"
```

**Important**: Some systems won't like the fact that you are setting
   the `INST_LIB` variable and attempting to use it on the same
   command line.  In that case, just replace `INST_LIB` with the
   actual path to which it corresponds.

**Important**: If you are building on an OS X, you should use the
    `DYLD_LIBRARY_PATH` instead of the `LD_LIBRARY_PATH`.

## Fixing the libtool
You are almost ready to build. Before you do, you need to make a
small change to `libtool`, which is located in the build_posix
directory of your WiredTiger tree.  Open the libtool file in the
editor and search for code that looks sort of like this:

```shell
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

Instead of the `func_append deplibs " $arg"` line, add the following
code snippet:

```shell
if [ "$arg" != "-load" ]; then
      func_append deplibs " $arg"
else
      func_append compile_command " $arg"
      func_append finalize_command " $arg"
fi
```

In some versions of libtool, you would see a slightly different
syntax, so you'd need to replace

```shell
deplibs+=" $arg"
```

with:

```shell
if [ "$arg" != "-load" ]; then
      deplibs+=" $arg"
else
      compile_command+=" $arg"
      finalize_command+=" $arg"
fi
```

What is happening here is that libtool is trying to be smart and
interpret the `-load` option to `clang` as a directive to link to
`liboad.so`.  This is a known issue with libtool. By inserting this
if-statement, you are instructing the libtool to not interpret this
option as such.

## Modifying and invoking the build command

Instead of just typing `make`, you will need to pass it a few
environment variables, like so:

```
INST_LIB=$LLVM_SOURCE/projects/dinamite/library
DIN_MAPS="/where/to/put//map/files/"
DIN_FILTERS="/full/path/to/function_filter.json" make
```

`INST_LIB` tells the compiler where the instrumentation library
lives. We already used this variable during configuration. It is set
after you successfully [installed DINAMITE](/2016/11/11/installing-DINAMITE).

`DIN_MAPS` tells DINAMITE where to put its "map" files. As DINAMITE
instruments the program it produces dictionary files that are
later used to covert binary traces to text. By setting DIN_MAPS
you are telling DINAMITE where to place these files. <b> You must
specify an absolute path for DIN_MAPS if you want to correctly
decode your traces.</b>

`DIN_FILTERS` variable tells the instrumentation pass the
location of the instrumentation filter file.  That file controls
what to instrument. If you don't provide the filters file, the
compiler will insturment all function calls and memory accesses in
your program, which is probably not what you want from the beginning.

Here is the contents of a ```function_filter.json``` file with
reasonable defaults. With this file, DINAMITE will instrument all
functions whose critical path is at least 40 lines of code (LOC),
where LOC does not include blank lines or comments and LOC for
loops are computed with the assumption that they execute once.

**Note that if you encountered problems during installation, where you
  had to set your `COMPILER_PATH` variable to properly compile the
  code with DINAMITE (see our [installation
  post](/2016/11/11/installing-DINAMITE)), you will encounter similar
  errors here. To work around, set your `COMPILER_PATH` variable as
  explained in the [installation
  post](/2016/11/11/installing-DINAMITE).**

```
{
    "minimum_function_size" : 40,
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


Copy the above into function_filters.json file and set `DIN_FILTERS`
to point to its location.

Now wait till the program compiles!


## Run the compiled program

If you run a command that uses the instrumented WiredTiger library,
you need to provide the path for the instrumentation library. For
example, suppose you run wtperf:

```shell
LD_LIBRARY_PATH=$LLVM_SOURCE/projects/dinamite/library ./wtperf
```

Or, if using OS X:

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

