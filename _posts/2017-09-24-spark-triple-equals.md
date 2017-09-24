---
layout: post
title: Learning Scala for Spark, or, what's up with that triple equals?
---

I began to learn Scala specifically to work with Spark.  The sheer number of language features in Scala can be overwhelming, so, I find it useful to learn Scala features one by one, in context of specific use cases.  In a sense I'm treating Scala like a DSL for writing Spark jobs.

Let's pick apart a simple fragment of Spark-Scala code: `dataFrame.filter($"age" === 21)`.

There are a few things going on here:

* The `$"age"` creates a Spark `Column` object referencing the column named `age` within in a dataframe.  The `$` operator is defined in an implicit class [`StringToColumn`](https://spark.apache.org/docs/2.1.1/api/java/index.html?org/apache/spark/sql/SQLImplicits.StringToColumn.html).  Implicit classes are a similar concept to C# extension methods or mixins in other dynamic languages.  The `$` operator is like a method added on to the `StringContext` class.

* The triple equals operator `===` is normally the Scala type-safe equals operator, analogous to the one in Javascript.  Spark overrides this with a method in `Column` to create a new `Column` object that compares the `Column` to the left with the object on the right, returning a boolean.  Because [double-equals (`==`) cannot be overridden](https://stackoverflow.com/questions/7681183/how-can-i-define-a-custom-equality-operation-that-will-be-used-by-immutable-set), Spark *must* use the triple equals. 

* The `dataFrame.filter` method takes an argument of `Column`, which defines the comparison to apply to the rows in the `DataFrame`.  Only rows that match the condition will be included in the resulting `DataFrame`.  

Note that the actual comparison is not performed when the above line of code executes!  Spark methods like `filter` and `select` -- including the `Column` objects passed in--are lazy.  You can think of a DataFrame like a query builder pattern, where each call builds up a plan for what Spark will do later when a call like `show` or `write` is called.  It's similar in concept to something like `IQueryable` in LINQ, where `foo.Where(row => row.Age == 21)` builds up a plan and an expression tree that is later translated to SQL when rows must be fetched, e.g., when `ToList()` is called.