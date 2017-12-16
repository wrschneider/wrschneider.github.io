--- 
layout: post
title: Learning Scala for Spark, and the apply method
---

Sometimes in Spark you will see code like 

```
val df1 = ...
val df2 = ...
val df3 = df1.join(df2, df1("col") === df2("col"))
```

It is a little odd at first to use `DataFrame` objects like methods.  

What's going on here?  

In Scala, objects have an `apply` method, which allows any object to be invoked like a method.  `obj(foo)` is equivalent to `obj.apply(foo)`.  DataFrame's `apply` method
is the same as `col`, so `df("col")` is equivalent to `df.col("col")`.  

This is also related to why you can create instances of case classes without `new` -- a case class defines a companion object with the same name, and that 
companion object has an `apply` method that returns `new ClassName()`.  

Personally I haven't learned to like Scala's `apply` feature, because it's not entirely obvious what `obj(foo)` is supposed to do.  But in this case,
it makes sense to have shortcuts like that when I'm thinking of Scala as a DSL for Spark.