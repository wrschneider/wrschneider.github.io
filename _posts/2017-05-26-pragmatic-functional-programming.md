---
layout: post
title: Pragmatic Functional Programming
---

I am not passionate about functional programming for its own sake.  I am passionate about readability and clean code, and functional programming is a tool 
to help get there.  I take a pragmatic approach: functional programming should be a tool in your toolbox, and you should be ready to
use when it makes your life easier.  At the same time, I don't feel a rush to Pure Function and Immutable All The Things.  I like languages that
support functional programming but don't strictly always require it.

The patterns I use most frequently fall in the filter-map-reduce category (it's not just for big data!)  One way of looking at this as 
applying the DRY principle to common combinations of loops and conditionals.  For example, you may find yourself copy-pasting loops like this frequently (C#)

```c#
var newList = new List<T>();
foreach (var row in rows) {
   if (condition(row)) {
     newList.add(transform(row));
   }
}
```

and the only parts that change are the guts of what you're doing to each element or the test that you're running on each element.  Rewriting 
with filter-map-reduce patterns looks like this:

```c#
var newList = rows.Where(row => condition(row)).Select(row => transform(row))
```

There are a number of benefits of writing your code like this:

* Improved readability - function composition allows your code to be DRY by replacing the common loop boilerplate with standard utility methods, passing the 
unique bits of code as an argument.
* Thread-safety - the rewritten code has no mutable state of its own--no appending to a list, no loop index variable.  Mutable state is still there *somewhere*, but it's Not Your Problem (tm).  So you get thread-safety for free.   
* Flexible evaluation - writing code in a more declarative style with pure functions allows more flexibility about when and how to evaluate.  `Select` and `Where` 
*could* be implemented with a loop, but they don't *have* to be.  Two interesting consequences of this is:
  * code to operate on a Spark RDD in parallel on a cluster doesn't look that different than the single-threaded equivalent.  
  * frameworks like Entity Framework can translate certain cases into SQL queries, so the same code written against regular in-memory collections can be turned into database operations depending on the type of the underlying object (`IQueryable` vs. `IEnumerable`)

*Special Cases for Reduce Operations* 

There are some obvious applications for reduce operations like sum, count, min, max, etc.

There are some other useful special cases.  For example: "does an item exist in a list" could be written like

```c#
var found = false;
foreach (var row in rows) 
{
  if (condition(row)) 
  {
    found = true;
    break;
  }
}
```

This could be replaced with

```c#
var found = rows.Any(row => condition(row));
```

Another example is bucketing list elements into a dictionary of some key to a sub-lists of items matching that key.  

```c#
result = new Dictionary
for each item in list
{
  key = expr(item)
  if key not in result
  {
    sublist = new List
    result[key] = sublist
  }
  result[key].Add(item)
}
```
  
becomes:

```
list.GroupBy(item => expr(item))
```

In both cases there's a lot less boilerplate code.
In practice I don't find that I write many of my own reducer operations, since there are usually library functions for the common cases. 

*Anti-pattern: Replace Loops with Foreach*

A common mistake when first adopting functional patterns is to replace the loop itself with `foreach` but not consider the body of the loop.

```c#
rows.ForEach(row => newList.add(transform(row)))
```

This misses the point--you still have mutable state and avoidable boilerplate in your code.  The `Select` version is better.

*Recursion vs. loops*

Some pure-functional languages like Haskell have no concept of a loop.  The equivalent of a loop requires tail recursion.  Mutable state is still 
there -- it's just moved from your 
code to the call stack, where it's managed by the compiler or interpreter.  So you would write an "iterative" version of `factorial` like this (using ES6
 to mix things up): 

```
const factorial = function(n) {
  const facHelper = function(n, res) {
     if (n <= 1) return res;
     else return facHelper(n-1, res*n); 
   }
   facHelper(n, 1);
}
```

This is equivalent in time/space complexity to a version written with a `for` loop (assuming language supports tail-call optimization).  

I'm not convinced this is an actual improvement, though; it feels like immutability for the sake of immutability.  By the time you get to this point, where 
you aren't able to use a reducer from a standard library, you may be better off sticking with imperative style and using a loop. 

On the other hand, recursion is general is great when it's a  natural fit for a problem and provides a more elegant solution than looping.  Recursion is best for problems like tree traversal or other forms of nesting that are harder to do with loops. 

*Pattern: Optional Types (replace conditionals with composition)*

Optional types are useful when you are trying to extract something from nested structures where any level could be null.  Optional types effectively replace 
null references with actual objects which accept a callback to pass the contained value if present.  The wrapper object behaves sort of like a single-element
collection.  Option types are monads, but you don't need to understand that to use it.

Example of what this might look like

```
if (x != null) {
    if (x.y != null) {
      output = x.y.z;
    }
}
```

replacing `x` and the nested elements with optional types would allow it to be traversed without null checks, like this:

``` 
output = x.flatMap(x => x.y).flatMap(y => y.z)
```

Again, the main benefit is readability.

Honestly, I'm skeptical of the utility of option types in languages that don't have algebraic types and pattern matching.  The chain of `flatMap` calls 
for safe derefence is an improvement over nested conditionals, but it would be even better for the language to attack that problem directly with the Elvis operator.  Or in Javascript and Python you could do `x && x.y && x.y.z`.

Also, I'm less convinced of the argument that eliminating null references (and null pointer exceptions) is hugely valuable by itself.  The optional wrapper is always not-null, but what you do to handle null or missing values is up to your code.  There's a risk that silently handling an unexpected null in your code (i.e., returning a nonsense value) might actually be harder to troubleshoot and fix than a loud, obvious null pointer exception--like a variation on catching and swallowing a runtime exception.