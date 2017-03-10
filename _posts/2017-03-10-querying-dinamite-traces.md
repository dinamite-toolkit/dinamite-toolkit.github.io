---
layout: post
title: Querying DINAMITE traces using SQL
excerpt_separator: <!--more-->
---

In this blog post we describe how to query DINAMITE traces using SQL.
<!--more-->

## Setting up the database and importing traces

If you have read our [earlier
post](https://dinamite-toolkit.github.io/2016/12/02/visualize-execution-flow-WT),
you know how to process the binary trace files generate by DINAMITE
using the `process-logs.py` script from the binary trace toolkit.

This script will produce a `trace.sql` file with the entire trace in a
human-readable format with delimiters, and a few SQL commands allowing
you to import this file directly into the database and query it from
there.

If for some reason you don't want to use a database, you can still
examine this file and query it using any other tools you like, because
it is just plain text. If you are a brave soul willing to embrace
latest database technologies, we will tell you how to set up a free
database MonetDB and use it to query the trace.

1. Download and install MonetDB for your platform as explained [here](https://www.monetdb.org/Downloads).

2. Start the database daemon and create the database called 'dinamite' like so:
   ```
   monetdbd create monetdb
   monetdbd start monetdb
   monetdb create dinamite
   monetdb release dinamite
   ```

3. Create a file `.monetdb` in your home directory with the following
content. This is needed, so you don't have to type the username and
password every time. Note, we are using the default password for the
database that will contain your traces. If you want to protect your
trace with a stronger password, change it accordingly.

   ```
   user=monetdb
   password=monetdb
   ```

4. Import the `trace.sql` file created by `process-logs.py` into the database:

   ```
   mclient -d dinamite < trace.sql
   ```

You are good to go! Now you can query your traces using SQL. For those
not fluent in SQL we provide a few example queries to get you started.

## Querying the traces

The database you created in the preceding steps will have three
tables: `trace`,`avg_stdev` and `outliers`. The `trace` table has raw
trace records softed by time. The `avg_stdev` table has average
duration and the standard deviation of durations for each
function. The table `outliers` has functions whose duration was more
than two standard deviations above the average.

The tables have the following schema:

   ```
   CREATE TABLE "sys"."trace" (
    "id"       INTEGER,
    "dir"      INTEGER,
    "func"     VARCHAR(255),
    "tid"      INTEGER,
    "time"     BIGINT,
    "duration" BIGINT
   );
   ```

where `id` is the unique identifier for each invocation of a function,
`dir` is an indicator whether the given record is a function entry
(dir=0) or exit point (dir=1), `func` is the function name, `tid` is
the thread id, `time` is the timestamp (in nanoseconds, by default),
and `duration` is the duration of the corresponding function.

   ```
   CREATE TABLE "sys"."avg_stdev" (
    "func"  VARCHAR(255),
    "stdev" DOUBLE,
    "avg"   DOUBLE
   );
   ```

where `func` is the function name, `avg` is its average duration
across all threads and `stdev` is the standard deviation.

   ```
   CREATE TABLE "sys"."outliers" (
    "id"       INTEGER,
    "func"     VARCHAR(255),
    "time"     BIGINT,
    "duration" BIGINT
   );
   ```

where `id` is the unique ID for each function invocation, `func` is
the function name, `time` is the timestamp of the function's entry
record and `duration` is the duration.

You can examine the schema yourself by opening a shell into the
database, like so:

   ```
   mclient -d dinamite
   ```

and issuing a command '\d'. Running this command without arguments
will list all database tables. Running this command with a "table"
argument will give you the table's schema, for example:

   ```
   sql> \d trace
   ```

Suppose now you wanted to list all "outlier" functions. You could do
that with the following SQL query:

   ```
   sql> SELECT * from outliers;
   ```


To be continued...
