---
layout: post
title: Rant - Java 8 streams are too little, too late
---

I was looking at some code the other day to convert a comma-delimited string like `"1,2,3,4"` into a list of integers.

The code was on Java 7, so it's still written like it's 1999: with a loop, parsing ints and appending to the result inside the loop.  (Hey, at least we have generics now.)

Collection manipulation like this is one of both the most common and tedious operations in application code.  So I asked what this would look like on Java 8, and I came back disappointed.  The best you can do on Java 8 is something like

    Stream.of(s.split(",")).map(Integer::parseInt).collect(Collectors.toList()) 

That's better than a loop but still a lot of syntax bloat compared with other modern languages.  The equivalent in C# would have been much less verbose: 

    s.Split(",").Select(int.Parse).ToList()

The magic of LINQ and extension methods eliminates the syntax bloat of converting the array to a stream first, and the conversion to a list is more concise (`ToList()` vs. `collect(Collectors.toList())`)

Javascript is even better because their arrays natively support map/reduce operations:

    s.split(",").map(Number)

And Python supports both list comprehensions (`[int(x) for x in s.split(',')]`) and functional style (`map(int, s.split(','))`).

Main takeaway is that Java 8 underdelivers in terms of developer convenience for common collection manipulations, when compared to nearly every other modern language.