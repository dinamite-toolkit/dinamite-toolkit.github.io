---
layout: page
permalink: /
---
# DINAMITE toolkit

DINAMITE is a unique set of tools that deliver ***program comprehension and performance insight.***

* Have you ever felt frustrated when debugging a performance issue in
a large multithreaded application?
* Have you ever fumed over
limitations of performance tools like `perf`?
* Have you ever wanted
to get more insight into a large codebase written by other people and
understand not only where it spends CPU cycles, but how it actually
works?

***If you answered "yes" to any of these questions, DINAMITE is for you.***

* DINAMITE automatically instruments C/C++ programs to **log names and
timestamps of all or selected functions and/or variables**
into a log file, as the program executes.
* As you run the program,
it produces the execution log.
* You can analyze the logs using
sophisticated and [visually rich tools](/trace-analysis/). Or just use
`grep` ;).

**DINAMITE LLVM compiler is available on [GitHub](https://github.com/dinamite-toolkit/dinamite)**

If you want to try out DINAMITE, click on [Quick Start](/quickstart/)
and follow the instructions. If you want to use it with a large
complex code base, take a look at [Technical
Articles](/tech-articles/index/).  **If you need help using DINAMITE please
[get in touch](/contact/)**


**A typical DINAMITE workflow goes as follows:**

1. Build the target program with clang/clang++ and link it with `libinstrumentation.so`
2. Choose a logging library flavour and build it.
    We currently provide three example logging libraries: binary filesystem output,
    text file system output and binary TCP output.
    The logging library emits logs of instrumented events. For more details
    see our page on [Logging libraries](user-guide/#logging-libraries)
3. Execute the instrumented binary. The log files will be created in your file
system or streamed over the network.
4. Analyze traces using [our tools](/trace-analysis/), 3rd party tools or Unix data analytics tools.

**To get started** with a simple example, read the [Quick
Start](/quickstart/) guide.  For examples of using DINAMITE with
complex systems, from integration into the build system to
sophisticated trace analysis, check out [Technical
Articles](/tech-articles/index/).


