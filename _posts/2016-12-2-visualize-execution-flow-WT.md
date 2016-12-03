---
layout: post
title: Visualizing the execution flow using DINAMITE traces
---

In this post, we explain to obtain a high-level summary of the execution using
DINAMITE trace analysis tools.

<!--more-->

After [compiling WiredTiger with DINAMITE](https://dinamite-toolkit.github.io/2016/11/12/compiling-WT/) we ran it with the LevelDB dbbench workload called `readseq`.
This workload sequentially iterates over the btree. After obtaining the traces
we went over the following sequence of steps to visualize the execution.

1. Download the binary trace toolkit, which contains loads of useful scripts for
processing and visualizing the traces, and build it:

   ```
   % git clone https://github.com/dinamite-toolkit/dinamite-binary-trace-parser.git
   % cd dinamite-binary-trace-parser
   % INST_LIB=$LLVM_SOURCE/projects/dinamite/library make
   ```
2. For convenience, put the traces as well as all the `map_*` files generated
during the DINAMITE compilation into the working directory, where you will process
the traces. Assuming your traces were placed into `/tmp` and you compiled the
program in `$BUILD_DIR` directory, this will do the trick:

   ```
   % mv /tmp/trace.bin.* .
   % cp $BUILD_DIR/map_* .
   ```

   DINAMITE generates a trace file per thread, so you will have as many `trace.bin.*`
   files as there were threads during your execution. The files will be numbered
   sequentially. These numbers are generated by the logging library in the order that
   threads invoke the library (i.e., the first thread that calls the instrumentation
   library will get the id of '1', the second thread will get the id of '2', etc.).
   They have nothing to do with `pthread` thread ids or with application-specific thread
   ids.

3. Now you will need to run a Python script located in the DINAMITE binary trace
parser, which you downloaded in Step 1. This script needs to know where the binary
trace toolkit lives, so you need to set the environment variable
`$DINAMITE_BINTRACE_TOOLKIT` to indicate its location as you invoke the script:

   ```
   % $DINAMITE_BINTRACE_TOOLKIT=/home/dinamite-binary-trace-parser $DINAMITE_BINTRACE_TOOLKIT/do-all.sh trace.bin.*
   ```

   Now sit back and relax or go get a coffee while the files are being processed.
   This might take a while depending on your traces sizes.

4. Once the traces are processed, your directory will have a bunch of new files.
For each binary `trace.bin.*` file there will be a corresponding `trace.bin.*.txt`.
This file contains the text version of the trace. You can peek into those files to
see all function names and timestamps. For example, you would see records like this:

   ```
   --> __wt_btcur_next 2 334888040634441
   <-- __wt_btcur_next 2 334890966319815
   ```

   The "-->" or "<--" arrows at the beginning of the line indicate whether this is
   the timestamp for entering or exiting the function: "-->" is for entering,
   "<--" is for exiting. The next word is the function name. The number after that
   is the thread id (redundant with the name of the file). The huge number
   following the thread id is the timestamp obtained with `clock_gettime` on
   Linux and `mach_absolute_time` on OS X (these calls are just as fast as reading
   the system time directly from the `rdtsc` register.

   In addition to the text traces, you will also have `trace.bin.*.summary`,
   for each trace file.  Summary files tell you what functions were executed by
   the corresponding thread, how many times each function was called, and how
   much time it took to execute on average and in total.

   But the best part is the visual summaries of the traces, which will be placed
   in the PNG files.

5. For each trace file there will be a corresponding PNG with a `enter_exit.0.0%.png`
extension. This file contains a diagram visualizing the execution as a state machine,
where each state is either a function entry or a function exit, and edges are annoted
with the numbers showing how many times a particular state transition has occurred.

   0.0% in the file's name indicates that there was no performance-related
   filtering applied to the execution flow diagram: functions that contributed at
   least 0.0% to the execution time were all included in the diagram. For many
   real-world applications this would make the diagram large and difficult to look
   at, so we are going to invoke a script to filter out all the functions that
   contributed less than 3%:

   ```
   % $DINAMITE_BINTRACE_TOOLKIT/process_logs.py -p 3.0 trace.bin.*.txt
   ```

   Here is one of the resulting diagrams that we obtained for the WiredTiger trace
   (running the LevelDB benchmark) compiled using the default function filter
   file from the [previous post](https://dinamite-toolkit.github.io/2016/11/12/compiling-WT/).

   ![WiredTiger execution flow diagram]({{ site.url }}/assets/trace.png)

   Each function state is colour coded depending on how much time this function
   contributed to the execution (more saturated colours: more time spent in
   this function). Percentages inside the circles representing the functions
   show much time the function contributed to the execution.
   Note that these percentages are *cumulative*. For example, function `__clsm_next`, which
   takes 98% of the execution time, is reported to call `__wt_btcur_next`, which
   takes 80% of the execution time: the execution time of `__wt_btcur_next` is counted
   toward the execution time of its parent function `__clsm_next`.

   The execution flow diagram reveals some interesting
   information about how the program works and runs. Even
   a developer entirely unfamiliar with the WiredTiger codebase can intuit that
   this program iterates over a Btree using a database cursor (repeated calls to
   `__wt_btcur_next` on the left side of the diagram). From the numbers on the
   execution flow chart edges we infer that the program iterated over the cursor
   in the vicinity of 200K times. We observe that roughly 2200 times the progam
   calls `__wt_page_in_func`, which probably indicates that about 1% of the
   time the needed memory page was not in the page cache, so we had to "page it
   in" from disk.

   For comparison's sake, here is the output that we get from profiling
   the same execution using the samping profiler `perf`:

   ![WiredTiger execution flow diagram]({{ site.url }}/assets/WT-leveldb.png)

   We can see that the top-ranking function `__wt_btcur_next` takes around 85%
   of execution time according to `perf`: pretty close to the time reported by the
   execution flow diagram. However, because the execution flow diagram was obtained
   running *instrumented* code, it ran slower than the uninstrumented program
   sampled by `perf`, so the percentages reported by `perf` and by the execution
   flow diagram may not be the same.

   Further, `perf` reports only the CPU time, while the execution flow diagram
   measures the total time. Note, for example, the function `__wt_page_in_func`.
   According to `perf`, it takes 0.08% of the CPU time, but the execution flow
   diagram reports that it takes 40% of the execution time. This function reads
   from disk the database pages that are not found in the cache: it does I/O, so it's
   not surprising that its CPU time is much smaller than its actual execution time.


