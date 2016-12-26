---
layout: post
title: SSIS data flow vs. insert-select
---

To transform data within a single SQL Server, with source and target data in the same database, it is probably faster to use an `INSERT` statement than a SSIS data flow task.

I compared performance of both on a table with about 23 million rows, taking about 3 GB.  On a small-to-medium size development server, an `INSERT` statement took about 30 seconds, 
and a comparable SSIS data flow task took over two minutes, roughly a 4x difference.  The data flow task did no transformation -- only a source, directly connected to target.  I was using the
 SQL Server native client, and was running `dtexec` on the same server as the database so the cost of pulling rows over the network should not have been a factor.  I was also turning off all progress logging from `dtexec` to  avoid inflating the runtime from overhead of writing to the console.  (Just to be clear, I was *not* running the package from Visual Studio.)

To maximize performance I was using the `TABLOCK` hint with the `INSERT` statement (`insert into table with (tablock)`) and have my database in simple recovery mode, to limit logging overhead. For 
this kind of batch operation I don't care about point-in-time recovery; if the process fails I would just start it over.  This is similar to what I would have done in Oracle with `insert /*+ append */`.

It seems like the main reason to use a data flow task is to load and transform data from *different* locations.