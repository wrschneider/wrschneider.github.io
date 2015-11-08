---
layout: post
title: Anonymous types, dynamic keyword and other reasons I like C#
---

A few years back I wrote about how I was starting to [like C# better than Java]({% post_url 2012-04-20-c-envy %}).  

Even after using Java 8, I still think C# is easier to work with, and provides more syntax conveniences for day-to-day webapp development.

Two such conveniences in C# are anonymous types and the `dynamic` keyword.  If you want to define an object whose only purpose is to define a structure to serialize to JSON, in 
C# you can do something like 

```
return Json(new {foo = "bar", baz = new[] {"asdf", "zxcv"});
```

This is more concise than defining a nested list/map structure.  Not quite as concise as Javascript or Python, but close.

Similarly, you can declare a variable `dynamic`, which turns off compile-time type checking and does dynamic dispatch at runtime similar to Groovy.  This allows you to dereference 
a structure like the one above:

```
dynamic x = new {foo = "bar", baz = new[] {"asdf", "zxcv"}
Debug.Assert(x.foo == "bar");
Debug.Assert(x.baz[0] == "asdf");
```

With `dynamic` you get the best of both worlds -- static compile-time type checking and IntelliSense most of the time, with the ability to selectively turn it off.  

My other complaints about Java's lack of syntax conveniences are mainly still valid:

* Java 8 now provides object initializers, but double-brace initialization 
[looks like a hack and some people discourage its use](http://stackoverflow.com/questions/9108531/is-there-an-object-initializers-in-java).  Java 8 collection initializers
seem to be just a [special case of double-brace initialization](http://stackoverflow.com/questions/6665049/what-are-the-cons-of-using-in-place-collection-initializers-in-java).

* Java still doesn't have a `var` keyword to automatically declare the compile-time type of a variable the same as the right-hand side of its assignment expression.

* You still have to remember that the Java equality (`==`) operator only tests for instance/reference equality, which means the 99% of the time you care about value equality, 
you have to remember to use `equals`.  So you still have to waste time and brain cells remembering if some variables are defined as `int` or `Integer` to make 
choose between `x == y` and `x.equals(y)`.  (It's actually even worse -- if `x` and `y` are both `Integer`, `x == y` might not break obviously [until the values 
get above a certain threshold](http://stackoverflow.com/questions/1700081/why-does-128-128-return-false-but-127-127-return-true-in-this-code).
