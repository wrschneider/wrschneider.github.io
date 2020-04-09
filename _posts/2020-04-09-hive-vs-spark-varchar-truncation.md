---
layout: post
title: Weird Hive and Spark SQL discrepancy with varchar truncation
description: Hive and Spark SQL may return different results on underlying Parquet when string length exceeds the defined VARCHAR length
---

I found an edge case where Hive SQL and Spark SQL will produce different results on a basic
`SELECT col FROM table` query.

Let's say you have DDL on an underlying Parquet file with a VARCHAR column.

```
-- Hive DDL
CREATE TABLE truncate_demo (
   col VARCHAR(10)
) STORED AS 'PARQUET' LOCATION ....
```

Now, let's suppose the Parquet file has a value that overflows the specified length
in the above DDL. For example, the value `"foo bar baz"` is 11 characters.  

If you run a query `select col, length(col) from truncate_demo` in Hive directly
through Beeline, it will show a length of `10` and the string truncated to 10
characters.

If you run the same query through Spark SQL it will show the full length of the
string (11) and display the string without truncation.

I found this discrepancy unexpected.

Note that the same does not apply to flat files/CSV with `STORED AS 'TEXTFILE'`;
in that case, both Spark and Hive will truncate equally.
