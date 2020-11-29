---
layout: post
title: Another look back at C
description: I used C for the first time in a while and had some thoughts on immutability and garbage collection
---

I just worked on C code again for the first time in a long while.  I've worked almost exclusively in some kind of managed
runtime environment -- Java, C#, Python, or Javascript in a browser.  So working on C again was like going through a bit of a time warp.  I spent a while trying to figure out how to connect the old stuff I used to know with the new.  Like, are gcc, gdb etc. all still there?  And how do I get them to work with VS.Code, and on MacOS?

Once the logistics were out of the way, I had some observations about the differences between working in C and
working in a managed environment, and how I had missed some of the bigger themes when I first learned Java several decades
back.

**Garbage collection is more than a convenience; it changes your programming patterns for the better.**  When I 
switched from C/C++ to Java,
my younger and more naive self thought, great, now I don't have to worry about where to `free` things that I `malloc`'ed 
somewhere else.  The deeper point, which I didn't fully appreciate at the time, is the extent to which *garbage collection
is a prerequisite for functional programming patterns* like pure functions, filter/map/reduce etc.

Most of the C code I was working with, had subroutines that would "return" results via mutating blocks of memory
allocated and passed in by the caller, rather than by constructing and returning new objects.  Often these are arrays
allocated on the *caller's stack*, which avoids concerns about whose responsibility it is to free memory and when.

When you can allocate new arrays or objects without worrying about who has to free them, though, you can write pure
functions that take immutable arguments and return new objects.  Aside from the benefit to readability that comes from
returning things by, well, returning them, you get additional benefits of function composition and thread safety.  Without
garbage collection, you can't write a pure function, because even returning a simple string requires your function to allocate
memory on the heap and the caller has to worry about when to `free` it.

**Exceptions are a bigger deal than replacing magic flag codes with more meaningful names**.  Without exceptions, code ends up
littered with boilerplate conditional checks. You have to explicitly check every possible null-pointer or index-out-of-bounds
condition explicitly to avoid the whole program from crashing.  With exceptions, you can add a `catch` somewhere and a runtime
exception from ten layers deep will percolate back to your catch, giving you a chance to log and recover but not crash the
whole program.  Without exceptions you'd have to deal with each one of those levels.

This is also a complaint I have about Go.  Go has a unique system of error types that solves for separating errors 
from normal return values.  But Go does not have a good answer for exception propagation.  Exceptions have to be propagated explicitly with lots of `if err != nil { return nil, err }` boilerplate.