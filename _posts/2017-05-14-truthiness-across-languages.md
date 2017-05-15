---
layout: post
title: Opinions on truthiness across languages
--- 

Different languages have different opinions about what to treat as "truthy" or "falsy" when using a non-boolean object as an expression inside an `if` statement.

I looked at Python, Groovy, Javascript and Ruby to compare their differences.

|---
| Value | Python | Groovy | Javascript | Ruby |
|-------|--------|--------|------------|------|
| null / nil | False | False | False | False |
| Zero (0)   | False | False | False | True |
| Empty String ("") | False | False | False | True |
| Empty list / dict | False | False | True | True |
|---

My observations and personal opinions on language design:

* Python treats zero, empty strings and collections all as 'falsy'.  Personally, I find this the most intuitive convention.  
    * Treatment of zero and null as falsy has historical precedent, from C. 
    * Treatment of empty strings and collections is a nice convenience, given the number of times I've written conditionals like `if (foo != null and !foo.empty())`.  It's usually the exception that I want to distinguish between null and empty in a conditional.  So it's nice that `if (foo)` handles the common case, then I can write `if (not foo is None)` when I really do want to distinguish null.  
    * Treatment of empty string as similar to null feels familiar from my Oracle experience.  
* Groovy is inspired by Python and adopts similar conventions for truthiness. 
* Ruby takes a different opinion that all values are truthy except `nil` (and `false`, of course).  While it's not my personal preference, it's defensible and 
self-consistent.
* Javascript can reliably be expected to deliver a WTF.  Javascript treats zero and empty strings as falsy, but empty collections as truthy.    
To me, it's hard to understand why strings and collections ought to behave differently; the Python behavior makes much more sense.   But wait, it gets even better: 
check out this [link on StackOverflow](http://stackoverflow.com/questions/5491605/empty-arrays-seem-to-equal-true-and-false-at-the-same-time).