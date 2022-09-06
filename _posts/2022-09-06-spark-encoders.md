---
layout: post
title: Spark encoders, implicits and custom encoders
description: Understanding how Spark converts rows into Scala case classes
tags: spark
---

One of the nice things about Spark SQL is that you can reference datasets as if they were like statically-typed collections
of Scala case classes.  However, Spark datasets do not natively store case class instances; Spark has its own internal format
for representing rows in datasets.  Conversion happens on demand in something called an `encoder`.  When you write code
like this:

```
case class Foo(a: Int, b: String)

spark.sql("select a, b from table").as[Foo].map(foo => Foo(foo.a + 1, foo.b))
```

the call to `map` will trigger Spark to convert its internal rows into instances of case classes, using an encoder.

The above call will actually fail to compile unless you import `spark.implicits._`.  One of the implicits defined is an encoder that 
can convert any Scala `Product` type (such as a case class) into internal rows and back, in addition to primitive types like
ints and strings.  Once those encoders are in scope, the above code will compile and work.

The same import for the same encoders are required if you want to build a dataset from a collection of case classes, e.g.,

```
Seq(Foo(1, "x")).toDS
```

Some points of caution:

- Spark Dataframes are alias for `Dataset[Row]` but this `Row` type (`org.apache.spark.sql.Row`) is *not the same* as the Spark internal row representation

- `df.map((r: Row) => r)` will not work even if you import `spark.implicits._`.  This is because you need the `RowEncoder` that translates Spark internal rows to and from `Row`, but this encoder needs the dataframe's schema.  You would have to provide the
`RowEncoder` explicitly like, `df.map((r: Row) => r)(df.encoder)`

- The `udf` function does not allow you to specify explicit encoders, and a UDF like `udf { (r: Row) => ... }` must work some other way.  It looks like instead, 
[when there is no encoder available, Spark UDFs will try to explicitly convert](https://github.com/apache/spark/blob/295e98d29b34e2b472c375608b8782c3b9189444/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/expressions/ScalaUDF.scala#L168) 
the `StructType` of the argument in the row, to a Scala `Row` object.

- Because the encoders for case classes work on reflection and are not aware of the runtime schema of the underlying dataset,
they will fail if there are missing fields.

For example, this will break with runtime error:

```
case class Foo(a: Int, b: String)

spark.sql("select a from table").as[Foo].first()
```

Now, what happens if you want to have objects in a Dataset that are not covered by an existing encoder, for example a class 
that isn't a case class, or a Java object that doesn't follow bean conventions? The
[HAPI FHIR Java classes](https://hapifhir.io/hapi-fhir/docs/appendix/javadocs.html) are one such 
example.  In this case, it is possible to write your own encoder.

Example of non-conforming class definition (intentionally bad code for illustration purposes):

```
class Foo {
  var a: Int = 0
  var b: String = ""

  def setA(a: Int): Unit = {
    this.a = a
  }
  def getA: Int = this.a
  def getB: String = this.b
  def setB_misnamed(b: String): Unit = {
    this.b = b
  }
}
```

This won't work:

```
spark.sql("select 1 as a, 'x' as b").as[Foo]
```

You can fix this by making your encoder, using [RowEncoder](https://github.com/apache/spark/blob/master/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/encoders/RowEncoder.scala) as a guide:

```
def serializerFor(inputObject: Expression) = CreateNamedStruct(
    Seq(
        Literal("a"),
        SerializerBuildHelper.createSerializerForInteger(Invoke(inputObject, "getA", IntegerType)),
        Literal("b"),
        SerializerBuildHelper.createSerializerForString(Invoke(inputObject, "getB", IntegerType))
    )
)

def deserializerFor(inputObject: Expression): Expression = {
  val instance = NewInstance(classOf[Foo], Seq(), ObjectType(classOf[Foo]))
  InitializeJavaBean(instance, Map(
    "setA" -> GetStructField(inputObject, 0),
    "setB_misnamed" -> DeserializerBuildHelper.createDeserializerForString(GetStructField(inputObject, 1), true)
  ))
}

val serializer = serializerFor(BoundReference(0, ObjectType(classOf[Foo]), true))
val deserializer = deserializerFor(GetColumnByOrdinal(0, serializer.dataType))

implicit val encoder = new ExpressionEncoder[Foo] (
  objSerializer = serializer,
  objDeserializer = deserializer,
  ClassTag(classOf[Foo])
)
```

Notes:

* The encoder contains both a serializer and deserializer.  The serializer turns Scala objects into Spark internal objects, 
while the deserializer turns Spark internal objects into Scala objects.
* You can either pass `encoder` as an explicit argument (`df.as[Foo](encoder)`), or define it as implicit.
* `Invoke`, `NewInstance`, `InitializeJavaBean` etc. are *expressions* that result in generated code, that will compile and then 
run on each individual worker node as part of a query plan.  Effectively, you are building an expression tree that will drive
code generation.  Similarly, the `inputObject` is not the actual input data, but rather an expression that will result in generated code that evaluates to the input data.
