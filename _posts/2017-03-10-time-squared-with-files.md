---
layout: post
title: Viewing DINAMITE traces with TimeSquared.
excerpt_separator: <!--more-->
---

In this blog post we describe how to view DINAMITE traces using [TimeSquared](https://auggy.bitbucket.io/timesquared).
<!--more-->

In the [previous post](/2017/03/10/querying-dinamite-traces) we
described how to query DINAMITE traces using a SQL database.

** Pre-requisite:** You will need to set up the database as described
in that post and try the example queries, because we will use the
database to generate "interesting" portions of the trace for viewing
with TimeSquared.

Suppose that after reading the [previouspost](/2017/03/10/querying-dinamite-traces)
you want to generate a portion to the trace corresponding to the time when a function of
interest took a particularly long time to run. Suppose that you found,
using the query `SELECT * FROM outliers ORDER BY duration DESC;` that
the ID of your "outlier" function invocation equals to 4096.

Issue the following query to record the portion of the trace
concurrent with the outlier function invocation to a file, like so:

   ```
   sql> COPY WITH lowerboundtime AS
   (SELECT time FROM trace WHERE id=4096 AND dir=0),
   upperboundtime AS
   (SELECT time FROM trace WHERE id=4096 and dir=1)
   SELECT dir, func, tid, time FROM trace WHERE
   time>= (SELECT time FROM lowerboundtime)
   AND time <= (SELECT time FROM upperboundtime)
   INTO '/full/path/to/output/file'
   USING DELIMITERS ' ';
   ```

Note the single quotes around the **full** file path, and the mention
of delimiters.

Suppose you named your output file `query_result.txt`. Now, fire up [TimeSquared](https://auggy.bitbucket.io/timesquared) in the browser, click on "select file", choose `query_result.txt` and enjoy your trace being rendered!

Once rendering is complete, you will see a succession of callstacks
over time, for each thread. Time flows on the horizontal axis, threads
and callstacks are along the vertical axis, like so:

![TimeSquared]({{ site.url }}/assets/timesquared.png)

If you hover with the mouse over a coloured strip in the callstack,
TimeSquared will show you the name of the function and some
information about it.

TimeSquared is work in progress. We are working on building an
interface to effectively navigate very large traces (at the moment
TimeSquared cannot show too many events -- and even if it could, you'd
hardly want to look at all of them), and to query traces from within
the browser. We will report on these features in our future blog
posts.