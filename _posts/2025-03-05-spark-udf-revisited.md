---
layout: post
title: Spark UDFs Revisited
date: 2025-03-05
categories: spark big-data
description: revisiting the thought process on when to use or not use Spark UDFs
tags: [spark, udf, big-data, data-processing]
author: wschnei2
---

A while back I wrote about [some of the caveats of using UDFs](./2023-05-23-spark-udf-issues.md).

Beyond the concerns about null handling and performance, there are a few additional points to keep in mind:

* Not everyone is familiar with Spark array functions like [filter](https://spark.apache.org/docs/latest/api/scala/org/apache/spark/sql/functions$.html#filter(column:org.apache.spark.sql.Column,f:org.apache.spark.sql.Column=%3Eorg.apache.spark.sql.Column):org.apache.spark.sql.Column)
and [transform](https://spark.apache.org/docs/latest/api/scala/org/apache/spark/sql/functions$.html#transform(column:org.apache.spark.sql.Column,f:(org.apache.spark.sql.Column,org.apache.spark.sql.Column)=%3Eorg.apache.spark.sql.Column):org.apache.spark.sql.Column).  Some UDFs may be replaced with these 
native Spark SQL functions.

* There is now less of a penalty for using UDFs in Python, because of [vectorized UDFs with Pandas](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/api/pyspark.sql.functions.pandas_udf.html).  Before, Python UDFs would perform worse than UDFs in Scala or Java, because of the 
overhead of serializing from the JVM to the Python interpreter.

* On the other hand, if you're using Databricks, UDFs will prevent a given Spark SQL job from using
[Photon Acceleration](https://learn.microsoft.com/en-us/azure/databricks/compute/photon#limitations).

### When you want to use UDFs anyway

There are some times when should still use UDFs.  My current rule of thumb is, use UDFs _if and only if_ there isn't a straightforward
solution in Spark SQL (or Dataframe operations), after taking full account of what you can do with window functions and array operators like `filter`
and `transform`.

Some examples:

* Running batch inference on a dataframe, perhaps with the the [MLFlow Spark API](https://mlflow.org/docs/latest/api_reference/python_api/mlflow.spark.html?highlight=spark).

* Business logic that operates on a group of records at a time, where you might need to make multiple passes over the records in a group or where
processing one record in a group requires tracking state over how the other records in the group have been processed.  
  
Some examples of complex business logic that justifies a UDF:

* Pairing up rows in a transaction ledger where positive/negative amounts cancel each other out.  Doing this in Python, Scala or Java code
on a list of objects is straightforward.  (I could imagine it as a Leetcode-style interview question.) Doing this in SQL would be tricky, because you have to
keep track of whether a given record was already paired with another to prevent using it again.

* Clinical quality measures, where you have different measures with different criteria that would translate to combinations of `exists`/`not exists`
logic.  This is unwieldy to do in SQL alone and might
end up with more reads of source data than you'd like.  The code becomes simpler and more readable, if instead you use a UDF that takes a `PatientInfo`
object that contains a full record for the patient (lists of encounters, diagnosis codes, procedure codes etc.) and can iterate over the collections multiple
times. (The `PatientInfo` object would be analogous to a FHIR bundle.)

I would _not_ use UDFs when SQL window functions would have been straightforward. For example, comparing a row to a subtotal within a group of rows, is
exactly why SQL window functions exist.