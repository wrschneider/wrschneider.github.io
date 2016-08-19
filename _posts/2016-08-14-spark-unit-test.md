---
layout: post
title: Unit testing Spark Dataframe transformations
---

Parts of Spark jobs that do pure-functional transformations on dataframes or RDDs – independent of I/O – are ideal candidates for unit testing.

I demonstrated this in a [trivial project here](https://github.com/wrschneider/spark-test).  This project has two main classes: one that takes a dataframe and generates a new dataframe with an additional column; and another that tests that transformation.  This uses another library, which provides a trait (SharedSparkContext) for setting up a default local-testing SparkContext.  Spark will then run in local mode on a workstation without any Hadoop ecosystem dependencies.

The main code for creating a dataframe programmatically looks like this:

```
val sqlContext = new SQLContext(sc)
    val rows = Seq[Row](
        Row(1),
        Row(2),
        Row(3)
        ).asJava
    val schema = StructType(Seq(StructField("a", IntegerType)))
   
    val df = sqlContext.createDataFrame(rows, schema)
```

(There is probably a better way to construct the input row list.)

The test can be run through sbt or through Eclipse with Scala IDE. 

To get up and running from a clean Eclipse environment (assuming you already have Eclipse and a JDK installed):

* install Scala
* install sbt
* install Scala-IDE for Eclipse (should include scalatest support)
* The project already includes a plugins.sbt file with dependency on sbteclipse plugin, so that should get picked up when you run "sbt eclipse" for the first time.
* "sbt test" will run tests, or you can run them through Eclipse with "Run As..." (Ctrl-F11)
