---
layout: post
title: Spark mistakes I made
description: I made a bunch of naive mistakes with Spark that I didn't realize until running on larger datasets
---

I built a Spark process to extract a SQL Server Database to Parquet files on S3, using an EMR cluster.  I am using as much
parallelism as possible, extracting both multiple tables at a time and splitting tables up into partitions to be extracted
in parallel.  My goal is to size the EMR cluster and number of total parallel threads to the point where I saturate the
SQL Server.

At its core, this process is almost the simplest Spark job you can get:

* [`spark.read.jdbc` with partitioning](https://spark.apache.org/docs/latest/sql-data-sources-jdbc.html)
* `spark.write.parquet` to S3
* the above are wrapped in `Future`, called with `Future.traverse` and `Await.result`, allowing for multiple threads on the
Spark driver to invoke jobs.

I had it all working but then it didn't perform as well as I expected on larger databases.  It turned out I made a bunch of
naive mistakes, which I'm capturing here for posterity.

* _I mis-used `repartition`._  I wanted to limit the number of threads for a single table, to limit the number of times
SQL Server has to scan the same table.  So I figured my source dataframe would keep a small number of partitions (say, 8)
and I could repartition to a larger number on write.  What I forgot is, this results not in simply splitting the source
dataframes into smaller pieces, but shuffles them all over the network within the Spark cluster.  So I got rid of this and
replaced with the `spark.sql.files.maxRecordsPerFile` setting to keep the generated files under a certain size.

* _I forgot that `DataFrame` is lazy and thus a leaky abstraction._  At the last minute I added support to validate the
rows in the `DataFrame` by calling `df.count` and comparing against a `select count(1) from table` query.  I forgot that
this would result in querying the database again to fetch the rows in the `DataFrame`.  It was even worse, because I was
calling `df.count` _after_ repartition, so the shuffle above was happening twice too.  I ended up fixing this by doing a
count off the generated Parquet instead, which is much faster.

* I didn't size my EMR cluster properly.  I overlooked the basic math to realize that I was trying to run 32 tasks
in parallel (4 tables * 8 partitions per table) but only giving my cluster 24 cores.

* I copy-pasted settings for `spark.executor.memory`, `spark.driver.memory` etc. without understanding their impact fully.
What I didn't understand was that this was also causing apparent single-threading in my driver -- only one table was
running at a time.  The issue turned out to be that, I was setting `spark.executor.cores` to 4. Each EMR node had 8 cores,
but the `spark.executor.memory` was so high that only one executor could go on a node and 4 cores were going to waste.
Further, the driver was taking up its own EMR core node -- it doesn't necessarily run on the EMR master node.  
\
It turned out that, removing these settings altogether and accepting the defaults was actually better.  I ended up with
two executors per node, all the cores getting used, and the driver running on the same node with two other executors.

* On the SQL side, I used `abs(binary_checksum(*)) % ${numPartitions}` as my partitioning column.  It worked, but I
neglected the CPU impact on the SQL Server.  The issue here is that the CPU overhead scales with the number of columns and
`binary_checksum(*)` is much more expensive than `binary_checksum(column)`.  So, I fixed this by picking individual
columns with high cardinality and low skew.  There were a few exception cases where I had to use a combination of a
few columns together.

I still haven't figured out the optimal balance of EMR to SQL concurrency, but my rule of thumb so far is, size the
EMR cluster to have at least as many cores as the SQL instance, and then set the concurrency (tables, partitions) based on
the EMR size.  I haven't figured out how much harder I can push SQL -- with basic `select *` queries I expect I/O or network
to be the bottleneck rather than CPU.  Nor have I figured out the optimal tradeoff of more tables in parallel vs. more
partitions per table, especially with unevenly sized tables.
