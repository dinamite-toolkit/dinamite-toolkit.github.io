---
layout: page
permalink: user-guide/
---
# User Guide

This page is intended to help users leverage all the features of DINAMITE.

## Contents

#### [Instrumentation filtering](#instrumentation-filtering)

#### [Log format](#log-format)

#### [String maps](#string-maps)

#### [Logging libraries](#logging-libraries)

#### [Allocator definitions](#alloc-defs)

#### [Binary trace parsing framework](#trace-parsing)


<hr>

# <a name="instrumentation-filtering"></a> Instrumentation filtering 

By default DINAMITE will instrument every memory access in your program. However, often you only want to instrument parts of the code, and not all memory accesses, but simply function entry/exit timestamps. 

In order to instrument only parts of your code, and specific events, DINAMITE supports function filtering.
To leverage this, you need to provide a filter file. The filter basically works as a configuration for functions that are allowed to be instrumented.
It is stored in JSON format and looks something like this:

```
{
    "minimum_function_size" : 10,
    "function_size_metric" : "LOC_PATH",
    "check_small_function_loops" : true,

    "blacklist" : {
        "function_filters" : [
            "function_that_wont_be_instrumented1",
            "function_that_wont_be_instrumented2"
        ],
        "file_filters" : [
            "wont_be_instrumented.c"
        ]
    },
    "whitelist": {
        "file_filters" : [
        ],
        "function_filters" : {
            "*" : {
                "events" : [
                    "function",
                    "alloc"
                ]

            },
            "function_with_an_interesting_access_pattern" : {
                "events" : [
                    "access",
                    "function"
                ]
            },
            "*lock*" : {
                "events" : [
                    "function"
                ],
                "arguments" : "*"
            },
            "interesting_second_argument" : {
                "events" : [
                    "function"
                ],
                "arguments" : [ 1 ]
            }
        }
    }
}
```

The top level objects in the filter JSON specification are:

- `minimum_function_size` (integer) - Anything below this size (in specified units, see next) will **not** be instrumented.
- `function_size_metric` - Can have three different values: `IR`, `LOC` and `LOC_PATH`. This tells the compiler what metric we would like to use for size-based
    filtering. Respectively, the three metrics are: number of LLVM IR instructions, total number of lines of code in a function and
    number of lines of code in the longest execution path through the function.
- `check_small_function_loops` - if set to true, it will **enable** instrumentation of functions smaller than the specified size which **contain a loop**
- `blacklist` - list of files and functions which will be ignored when instrumenting
- `whitelist` - list of files and functions which will get instrumented, along with more detailed filtering of functions

Any field that calls for naming an entity in the code allows for simple globbing (using `*` as a wildcard for parts of the string)

## Whitelisting and blacklisting

Whitelist and blacklist entries have a priority ordering as follows:

- Everything that is in the blacklist will **automatically** get ignored.
- If the whitelists are empty, everything is instrumented.
- As soon as anything is specified in the whitelist, everything else is ignored
- Functions that are smaller than the specified minimum size will get ignored, unless they match anything other than `"*"` in the whitelist.

## Whitelist function entry format

The `whitelist->function_filters` object is keyed with function names. Its contents are an array of `events` which can contain the strings
`"access"`, `"function"` and `"alloc"`, telling DINAMITE to instrument memory access, function and allocation events in the specified function.
When instrumenting a function one can also specify arguments to be logged. In practice, we found this to be very helpful when logging lock behaviour
in multithreaded programs. Locking routines usually accept lock pointers as arguments, and with argument logging one can get all the pointer values
that get passed.

## Using filters in DINAMITE

In order to tell DINAMITE about the function filters, at the time you compile your program with DINAMITE, you will need to set the environment variable `DIN_FILTERS` to point to the JSON document, like so:

`DIN_FILTERS="/path/to/function_filter.json" make -j 4`

If you don't set `DIN_FILTERS`, DINAMITE assumes that you want to instrument all the events in your program. **It is safest to provide the full path of the `function_filters.json` file.** If you are building a complex project the build might be performed in multiple directories. If only the top-level directory has the filters file, the files in the other directories will not be filtered correctly.

With C++ code, function names are typically mangled. To find a real function name, run compilation once. The filtering tool will output `function_sizes.json` which will contain the names of all encountered functions and their respective sizes (defaults to LOC_PATH metric). You can use these to fill your filter list properly.

# <a name="log-format"></a> Log format

DINAMITE's log format is not enforced. If you don't like the way it is,
writing a new logging library takes very little effort! If you do that,
we'd love to hear how you did it and why!

The default implementation in the binary logging library differentiates
between three types of events: function, allocation and access.
This is, in fact, all the event types that are available from DINAMITE.
When you instrument a program, the compiler adds calls to external logging
functions in the appropriate places and passes them all the relevant information.

(You can follow along by opening `<dinamite_root>/library/binaryinstrumentation.h` )

All three log event types are encased in a union which contains a field to identify the event type.

All three log event types also have a shared field called `thread_id` which, as you've
already guessed, contains the ID of the thread the event happened on. This is the only field
that doesn't come from DINAMITE's compiler (not available at compile time). The implementation
in `binaryinstrumentation.h` can be used for reference if you want to make a new 
thread-aware logging implementation..

The following field descriptions will omit the `thread_id` field, so don't get confused, it's still
there.

## Function events

These events contain the type of event (`fn_event_type`) which can have a value of either `FN_BEGIN` or `FN_END` (defined in the same file) and the ID of the function (see string maps).

## Allocation events

Allocation events are described with the code location of where the allocation
happened and information about the type, size and number of allocated elements.
They also contain the address of the beginning of the allocated memory region.

The field `size` is expressed in bytes.

The `type` and `file` fields are string IDs, which will correspond with the emitted
JSON string maps.

## Access events

Access events each describe a single memory access.

They contain the pointer to the accessed memory location (`ptr`), value - encoded as a union
of all possible primitive types, type of access ('r' for reads, 'w' for writes, and 'a' for
special argument logging events), three fields for code location, type of accessed data and
the name of the variable (both corresponding to the string maps).

Variable names can pertain to single scalar variables, in which case, their name in the maps
will be whatever is visible to LLVM in its IR. Besides that, they can be fields of a complex
data type (classes or structs), in which case, their name will be of the format "ClassName.fieldName".

In our experience, this is the most meaningful way to encode variable name information.

# <a name="string-maps"></a> String maps

In order to keep the size of logs as small as possible, DINAMITE encodes all strings encountered
while instrumenting as integer IDs. All strings fall into one of the 4 categories: type,
variable, function and file names.
When compiling, DINAMITE stores these string to integer mappings in 4 JSON files.

The default location of these files is the current build directory. This, however,
means that build systems in which the compiler is invoked from different locations
will store maps in multiple directories. **This is incorrect behaviour**, because
DINAMITE relies on reading the state of these maps from the previous invocations.
To avoid this problem, when invoking the build, provide the path in which to store
string maps as `$DIN_MAPS` like so:

```
DIN_MAPS=/path/to/maps make
```

After the build, `/path/to/maps` will contain 4 files: `map_sources.json`, `map_functions.json`,
`map_types.json` and `map_variables.json`, each containing string to integer ID mappings
for its respective string category.

# <a name="logging-libraries"></a> Logging libraries

DINAMITE's compiler instruments events in the code it compiles with function calls.
These function calls are directed at external functions which will, in the end,
be located in `libinstrumentation.so`.

This design lets us keep the instrumenting compiler decoupled from the actual logging
implementation, and makes implementing new output libraries very easy.

If you want to make a new logging library (example: send all events into a database),
all you have to do is implement a handful of functions.
In the root of our compiler project repo, you will find a `library/` directory. This directory
contains a couple of default logging library implementations, which can be used as is,
or as templates for your own implementations.

When running instrumented binaries, you have to make sure that `libinstrumentation.so` is built
(go to `library/` and run `make binary`, for example) and that it is in your
library path for the run.

## Making a new logging library

If you want to make a new logging library, you need to add an entry to the Makefile in `library/`.
Let's assume the code for your library is in `fooinstrumentation.c`. You would add someting like this
to the make file:

```
foo: fooinstrumentation.o dinamite_time.o
	make bitcode
	$(CC) -shared -o libinstrumentation.so $^
```

You would build this logging library by invoking

```
make foo
```

In the actual implementation, you have to worry about implementing the following functions:

{% highlight c %}
void logInit(int functionId)

void logExit(int functionId)

void logFnBegin(int functionId)

void logFnEnd(int functionId)

void logAlloc(void *addr, uint64_t size, uint64_t num, int type, int file, int line, int col)

void logAccessPtr(void *ptr, void *value, int type, int file, int line, int col, int typeId, int varId)

void logAccessStaticString(void *ptr, void *value, int type, int file, int line, int col, int typeId, int varId)

void logAccessI8(void *ptr, uint8_t value, int type, int file, int line, int col, int typeId, int varId)

void logAccessI16(void *ptr, uint16_t value, int type, int file, int line, int col, int typeId, int varId)

void logAccessI32(void *ptr, uint32_t value, int type, int file, int line, int col, int typeId, int varId)

void logAccessI64(void *ptr, uint64_t value, int type, int file, int line, int col, int typeId, int varId)
{% endhighlight %}

`logInit` and `logExit` will get called at the start of `main()` (or global ctors in C++) and end of `main()` (or before calls to `exit()`), respectively.

`logFnBegin` and `logFnEnd` are called at the start and end of the function with the given ID.

`logAlloc` is called at every allocation.

`logAccess*` are called when instrumenting appropriate memory accesses.


# <a name="alloc-defs"></a> Allocator definitions

Different programs use different memory allocation libraries. In order to be able to recognize
and instrument allocations in your software, DINAMITE uses a special configuration file.

By default, DINAMITE will look for this configuration as `alloc.in` in the current directory.
You can also set an environment variable, called `ALLOC_IN`, to contain the path to the file
with allocator configuration.

An example configuration file is provided in DINAMITE compiler's repo root, as `alloc.in`


The format looks like this: 

```
# func                number   size   addr
#
malloc                 -1       0    -1
calloc                 0       1    -1
```

Lines prefixed with "#" are comments.

Definitions are formated as the name of allocator function, followed by three indices:

- The first index refers to the argument containing the number of elements to allocate (think calloc).
- The second index is the argument denoting the size of a single element (or the whole allocation, in case there's no count argument)
- The third argument is the index of the returned address. `-1` is specified when the allocator function returns the
    allocated address.

The above example shows configurations for standard libc allocator functions.



# <a name="trace-parsing"></a> Binary trace parsing framework

## Building

To build this, you must set `$INST_LIB` to point to the location of your `binaryinstrumentation.h` file and then run `make`. For example, assuming your DINAMITE LLVM pass lives in the $DINAMITE directory:

    INST_LIB=$DINAMITE/dinamite/library make

## Using

The build will produce a binary called `trace_parser`. If you run it without the arguments, it will print the list of available plugins. For example if you just want to print the binary trace generated by running a test in the $DINAMITE directory, you run the trace parser as follows:

    ./trace_parser -p print -m $DINAMITE $DINAMITE/trace.bin

The `-m` argument points to the location of the [string map files](#string-maps) 

## Adding new plugins

In order to add a new plugin and register it with the tool, you need to add a new header file in `src/`.
The header file should include `traceplugin.h` and contain a class definition for the new plugin that inherits TracePlugin.

The general template should look something like this:

```
#include "traceplugin.h"

class NewPlugin : TracePlugin {
    public:
        PrintPlugin() : TracePlugin("plugin_name") { // change plugin_name into whatever you want to use to invoke your plugin
        }

        void processLog(logentry *log) {
            // your processing code goes here
        }

        void finalize() {
            // this gets called when we go through the entire log file
        }

        void passArgs(char *args) {

        }

};

static NewPlugin registerMyPlugin;  // this line is necessary for registering the plugin properly.

```

You also need to include the new header file in `main.cpp` for it to get built.

## Name maps

Our LLVM instrumentation tool emits JSON files with dictionaries that map numerical IDs to strings, used when tagging logs with variable, file, function and type names. This trace parser supports loading those maps and using them directly in any new plugin.

Your plugin will inherit the property `NameMaps *nmaps` from the `TracePlugin` base class. `NameMaps` provides the following functions each returning a **pointer** to std::string containing the name (to avoid needless copying on each read log):

```
string *getVariable(int idx)
string *getType(int idx)
string *getFile(int idx)
string *getFunction(int idx)

```

To properly load the maps, put the JSON files in the same directory and pass the path to it to the `-m` option parameter when running the tool.

## Plugin arguments

You can use `-a <argument_string>` option to pass arguments to the invoked plugin. This is used for setting up parameters, and your plugin should handle the behaviour in the `void passArgs(char *)` method.

If your argument string has spaces in it, you must wrap it in quotes (`"arg string"`)


