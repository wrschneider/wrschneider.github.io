---
layout: post
title: Spark UDFs to help migrate from other SQL dialects
---

I found it helpful to create Spark UDFs to make it easier to migrate logic in SQL from another database like SQL Server.

SQL Server defines several string functions like `LEN`, `REPLACE` and `CHARINDEX`, which are not available in Spark by default.  Fortunately these are easy to implement in Spark with UDFs:  

```
spark.udf.register("len", (s: String) => s.length())
spark.udf.register("replace", (orig: String, toReplace: String, replaceString: String) => orig.replace(toReplace, replaceString))
spark.udf.register("charindex", (substring: String, str: String, startPos: Int) => str.indexOf(substring, startPos - 1) + 1)
```

These are all thin wrappers around native Scala string functions.  These will now be available for use in Spark SQL queries: 

```
select len('foo bar') as len_test,
     replace('foo bar baz', 'bar', 'quux') as replace_test,
     charindex('.', '1.2.3', 3) as idx_should_be_4,
     charindex('.', '1.2.3', 0) as idx_should_be_2,
     charindex('@', '1.2.3', 0) as idx_should_be_0  
```

This makes it a little easier to copy-paste queries from SQL Server to Spark, if the syntax is otherwise standard.

The one downside is that Spark UDFs are functions, not methods, and as such 
[do not allow for default argument values](https://stackoverflow.com/questions/25234682/in-scala-can-you-make-an-anonymous-function-have-a-default-argument).  So 
you would have to explicitly add a `0` as the third argument for `CHARINDEX` wherever it's missing.