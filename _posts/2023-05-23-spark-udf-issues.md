---
layout: post
title: Spark UDFs are good except when they're not
description: Spark UDFs are a useful tool, but can create unintentional performance and maintenance issues
---

I am a fan of using Spark UDFs and/or `map` functions for complex business logic 
where using the [full power of Scala or Java gives better readability or performance](https://opensource.optum.com/blog/2022/02/08/spark-part2-refactoring)
than relying on SQL set operations alone.

That said, I have run into two challenges with Spark UDFs:

### Extra work to handle nulls

Many SQL operations will deal with null or missing values gracefully, for example

```sql
case when x > y then 1 else 0 end as x_bigger_flag
```

will give `0` if either input is null.

On the other hand, if you write similar code in Scala:

```scala
val xIsBiggerFlag = if (x > y) 1 else 0
```

you will get a null pointer exception if either input is null, and have to check
for null explicitly:

```scala
val xIsBiggerFlag = if (x != null && y != null && x > y) 1 else 0
```

Even if you use `Option` types indiomatically, you still have to do extra work to 
replicate what SQL would have done for free:

```scala
(xOpt, yOpt) match {
   case (Some(x), Some(y)) => if (x > y) else 0
   case _ => 0
}
```

So for simple operations, UDFs are _less_ maintainable than their SQL counterparts.
For this reason alone, UDFs should only be used for sufficiently complex logic
where the overhead of null handling is insignificant relative to the overall
improvement in simplicity and readability.

### Performance with reading wide tables

It is well understood that [Python UDFs are a performance problem](https://medium.com/quantumblack/spark-udf-deep-insights-in-performance-f0a95a4d8c62), but even with
Scala UDFs you lose some ability for Spark to optimize.

There are several articles that mention how Spark UDFs are a black
box for optimization, mostly focused on the loss of predicate pushdown to Parquet ([example](https://jaceklaskowski.gitbooks.io/mastering-spark-sql/content/spark-sql-udfs-blackbox.html)) for filtering operations.

There is another important issue beyond filters: what columns are read from 
Parquet in the first place.  When you do something like

```scala
spark.read.parquet(file).where("condition").select("foo")
```

Spark only needs to read the column for `foo` from disk.

But when you replace this with

```scala
case class MyTable(
    foo: String,
    bar: String,
    baz: String,
    ....
)

spark.read.parquet(file).where("condition").as[MyTable].map(_.foo)
```

Spark loses the ability to determine that you are only consuming one column from your
dataset, and will not be able to do any optimization when reading Parquet. Even if 
it can do predicate pushdown with the filter, the object deserialization from Parquet to
Scala will use 

If your table is wide and has many fields, the difference could become significant.

One workaround is to define case classes specific to your UDF inputs and
outputs, rather than to the full underlying table.  For example

```scala
case class FooOnly(foo: String)

spark.read.parquet(file)
  .select("foo")
  .where("condition")
  .as[FooOnly]
  .map(_.foo)
```

