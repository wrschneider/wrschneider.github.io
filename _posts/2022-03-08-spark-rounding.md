---
layout: post
title: Spark arithmetic discrepancies with SQL Server
description: How to solve arithmetic discrepancies between SQL Server and Spark
---

I ran into a number of numeric discrepancies migrating an ETL process from Microsoft SQL Server to Apache Spark.  Some of the same principles may apply to any relational database.

The discrepancies I found fall into a few patterns:

* **Floating point arithmetic**: Floating point calculations should never be expected to match exactly.  Even a [`SUM` on SQL Server can return different results on the same values](https://dba.stackexchange.com/questions/233513/non-deterministic-sum-of-floats).  So floating point comparisons should always be treated as approximate.
  * Pro tip #1: Never use floats for currency or any value you expect to match exactly.  (If I had a nickel for every time I saw someone misuse floating types for currency, I would have either 99 cents, or $1.01.)
  * Pro tip #2: Be aware of floating point calculations creeping in.  Statistical functions like `STDEV`, `VAR`, `PERCENT_RANK` and their Spark counterparts return SQL `float`/double precision, and that may not be obvious if the result is implicitly cast on insert into a decimal column.

* **Decimal truncation vs. rounding**: Both Spark and SQL Server follow similar rules on how to treat decimal precision and scale after arithmetic operations (see [here for SQL Server](https://docs.microsoft.com/en-us/sql/t-sql/data-types/precision-scale-and-length-transact-sql?view=sql-server-ver15) and [here for Spark](https://github.com/apache/spark/blob/master/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/analysis/DecimalPrecision.scala)).  One key difference, though, is that SQL Server will _truncate_ decimals to the number of places, while Spark will _round_.  So even if you hold everything else constant with data types, it's possible a division result can mismatch.
  
  You can work around this with a Spark UDF:  
  
  ```scala
  val divide = udf((x: BigDecimal, y: BigDecimal, scale: Int) => x.divide(y, scale, RoundingMode.FLOOR))
  spark.udf.register("divide", divide)
  spark.sql("select divide(foo, bar, 6) as ratio from table")
  ```  
  
  Also note that SQL Server is not necessarily correct, but that it may be convenient to eliminate a source of discrepancies for validation purposes.

* **Implicit casting on INSERT**: In SQL Server (or any database), if you `CREATE TABLE` and then `INSERT` into it, values will be implicitly casted to the defined schema types on insert.  The typical Spark pattern is more similar to a `CREATE TABLE AS SELECT` or `SELECT INTO` where the resulting schema inherits from the results of the query, following the above rules for arithmetic operations.  This can cause discrepancies further downstream when these values are used in other calculations.  The solution in this case is to make the casts explicit.

* **Different promotion rules on SUM**: Spark increases the [precision of decimals by 10 on SUM](https://github.com/apache/spark/blob/02d3b3b452892779b3f0df7018a9574fde02afee/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/optimizer/Optimizer.scala#L1876), while SQL server promotes decimals to precision 38 on SUM. This difference can cause discrepancies further downstream with division operations -- greater precision in the numerator may cause SQL Server to cap scale at 6 decimal places, while Spark might go out to more decimal places.  Again, SQL is not necessarily more correct, but the discrepancy can be controlled with explicit casts to simplify validation.

Other than with floating points, it should be possible to get exact matches for decimal columns, when explicit casts or truncations are applied in the right places.