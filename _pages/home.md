---
layout: page
permalink: home/
---
# DINAMITE toolkit

DINAMITE toolkit is an ecosystem of tools for program analysis revolving 
around a powerful instrumentation pass written for the LLVM compiler 
infrastructure.

The tools can be used to get debug information rich traces about function executions,
memory allocations and memory accesses.

The typical workflow for using DINAMITE goes as follows:

1. Build the target program with clang/clang++ and link it with `libinstrumentation.so`
2. Choose a logging library flavour and build it.
    We currently provide three example logging libraries: binary filesystem output,
    text file system output and binary TCP output.
    The logging library is what emits logs of instrumented events. For more details
    see our page on logging libraries *link*.
3. Execute the instrumented binary (make sure the built `libinstrumentation.so` is
in the library path on your system.

The output of this process will vary depending on what kind of logging library you
are using. The most common case for us is using the binary filesystem output library.
We store our logs in a filesystem, and then process them with our *link* binary trace
toolkit.

You don't need all the access information? Head over to our Instrumentation Filtering *link*
page, where you can learn how to selectively instrument only the things you're interested in.

If the logs are getting too big for your storage system, you can use the binary TCP output
to send them into an instance of Spark Streaming (or your stream processing engine of choice,
with some fiddling) and process them online without having to store massive traces.

In our *link* blog we will write about case studies, new tools we develop and interesting
findings.

