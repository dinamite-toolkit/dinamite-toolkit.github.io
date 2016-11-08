---
layout: page
permalink: filtering/
---
#Instrumentation filtering

By default DINAMITE will instrument every memory access in your program. However, often you only want to instrument parts of the code, and not all memory accesses, but simply function entry/exit timestamps. 

In order to instrument only parts of your code, and specific events, DINAMITE supports function filtering.
To leverage this, you need to provide a filter file. The filter basically works as a white list for functions that are allowed to be instrumented.
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

DIN_FILTERS="/path/to/function_filter.json" make -j 4

If you don't set `DIN_FILTERS`, DINAMITE assumes that you want to instrument all the events in your program. **It is safest to provide the full path of the `function_filters.json` file.** If you are building a complex project the build might be performed in multiple directories. If only the top-level directory has the filters file, the files in the other directories will not be filtered correctly.

With C++ code, function names are typically mangled. To find a real function name, run compilation once. The filtering tool will output `function_sizes.json` which will contain the names of all encountered functions and their respective sizes (defaults to LOC_PATH metric). You can use these to fill your filter list properly.
