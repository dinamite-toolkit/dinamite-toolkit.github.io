---
layout: post
title: Discovering lock contention in Facebook's RocksDB
excerpt_separator: <!--more-->
---

This post describes how we discovered a source of scaling problems for
the multireadrandom benchmark in RocksDB.
<!--more-->

A while ago, we ran some scaling tests on a set of RocksDB benchmarks. We
noticed that some of the benchmarks didn't scale well as we increased the number
of threads. We decided to instrument the runs and use
[FlowViz](/2016/12/20/interactive-execution-flow-WT/) to visualize
what is going on with multithreaded performance. The discovery helped open two
issues in RocksDB's repository:
[1809](https://github.com/facebook/rocksdb/issues/1809) and
[1811](https://github.com/facebook/rocksdb/issues/1811).

Here's what we did:

*(This tutorial was tested on [DINAMITE's docker
container](https://github.com/dinamite-toolkit/dinamite-compiler-docker). If any of the
instructions do not work in your environment, please [contact us](/contact/).)*

## Setting up RocksDB

Configuring RocksDB to use DINAMITE's compiler is straightforward.
The following is a set of environment variables that need to be set in order to
do this. Save it into a file, adjust the paths to point to your DINAMITE
installation path, and source before running `./configure`

```
############################################################
## source this file before building RocksDB with DINAMITE ##
############################################################

export DIN_FILTERS=/dinamite_workspace/rocksdb/rocksdb_filters.json
export DINAMITE_TRACE_PREFIX=<path_to_trace_output_dir>
export DIN_MAPS=<path_to_json_maps_output_dir>
export EXTRA_CFLAGS="-O0 -g -v -Xclang -load -Xclang
${LLVM_SOURCE}/Release+Asserts/lib/AccessInstrument.so"
export EXTRA_CXXFLAGS=$EXTRA_CFLAGS
export EXTRA_LDFLAGS="-L${INST_LIB} -linstrumentation"
```
Make sure that `$LLVM_SOURCE` points to the root of your LLVM build, and that
`$INST_LIB` points to the `library/` subdirectory of DINAMITE compiler's root.

The contents of `rocksdb_filters.json` are as follows:

```
{
    "blacklist" : {
        "function_filters" : [
            "*atomic*",
            "*SpinMutex*"
        ],
        "file_filters" : [ ]
        },
    "whitelist": {
        "file_filters" : [ ],
        "function_filters" : {
            "*" : {
                "events" : [ "function" ]
            },
            "*SpinMutex*" : {
                "events" : [ "function" ]
            },
            "*InstrumentedMutex*" : {
                "events" : [ "function" ]
            }
        }
    }
}
```

Run `./configure` in RocksDB root, and then run `make release`. 

After the build is done, you should have a `db_bench` executable in RocksDB
root.

## Running the instrumented benchmark

The following is the script we used to obtain our results:

```
#!/bin/bash
BENCH=$1
THREADS=$2
./db_bench --db=<path_to_your_db_storage> --num_levels=6 --key_size=20 --prefix_size=20 --keys_per_prefix=0 --value_size=100 --cache_size=17179869184 --cache_numshardbits=6 --compression_type=none --compression_ratio=1 --min_level_to_compress=-1 --disable_seek_compaction=1 --hard_rate_limit=2 --write_buffer_size=134217728 --max_write_buffer_number=2 --level0_file_num_compaction_trigger=8 --target_file_size_base=134217728 --max_bytes_for_level_base=1073741824 --disable_wal=0 --wal_dir=/tmpfs/rocksdb/wal --sync=0 --disable_data_sync=1 --verify_checksum=1 --delete_obsolete_files_period_micros=314572800 --max_background_compactions=4 --max_background_flushes=0 --level0_slowdown_writes_trigger=16 --level0_stop_writes_trigger=24 --statistics=0 --stats_per_interval=0 --stats_interval=1048576 --histogram=0 --use_plain_table=1 --open_files=-1 --mmap_read=1 --mmap_write=0 --memtablerep=prefix_hash --bloom_bits=10 --bloom_locality=1 --benchmarks=${BENCH} --use_existing_db=1 --num=524880 --threads=${THREADS}
```

Make sure you fix the `--db` argument to point to where your database will be
stored. In our case, the first argument (`$BENCH`) was set to `multireadrandom`
and the number of threads varied from 1 to 8 in powers of 2.

## Results

The output logs were processed and visualized with FlowViz. The guide on how to
use this tool is available [here](/2016/12/20/interactive-execution-flow-WT/)
and will be omitted from this post for brevity.

Here is the FlowViz output for runs with 1 and 8 threads:

1 thread:

![RocksDB 1 thread run](/assets/rocksdb-1thread.png)

8 threads:

![RocksDB 8 threads run](/assets/rocksdb-8threads.png)

From these, it is immediately obvious that the amount of time spent performing
locking operations increases with the number of threads.

These results were used to infer inefficiencies in workloads using `MultiGet()`
and led to opening of the two aforementioned issues on the RocksDB tracker.


