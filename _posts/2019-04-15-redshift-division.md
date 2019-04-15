---
layout: post
title: How I used middle-school arithmetic to solve a Redshift migration issue
description: Redshift truncates the result of division to a number of decimal places, making some migrations from other technologies more complex
---

One of the most annoying issues I encountered while migrating from Oracle
to Redshift could be solved with middle-school arithmetic.

No, seriously.

Redshift has a bunch of [complex rules](https://docs.aws.amazon.com/redshift/latest/dg/r_numeric_computations201.html)
governing the precision and scale of results of basic arithmetic operations.

The one for multiplication makes some intuitive sense; `xx.xx` * `yy.yy` = `zzzz.zzzz`.  

The one for division is really hard to understand.  I even resorted to [making a calculator](https://jsfiddle.net/wrschneider/4cjs3a0t/7/) to experiment with different
scenarios.   I'm not sure why it works the way that it does, but at least it's documented.  

One consequence of the Redshift behavior is that when you calculate `A/B`,
the more decimal places in `B`, the *fewer* in the result.  (This of course
assumes any `integer` arguments are casted to `decimal` to avoid integer
division.)   So if `A` and `B` are both decimals, if you want to get more
decimal places in `A/B`, you may have to round `B` first, sacrificing some
accuracy.

Here's where the math comes in.  My specific scenario was the result of a
nested division, `A/(B/C)`.  In my case, A was `decimal(19,2)` (dollars and cents),
and B and C were `bigint`.  If I cast B and C to `decimal(19,0)`, then
`B/C` would be `decimal(38,19)` and `A/(B/C)` would fall back to `(38,4)` -- only
four decimal places.  If I want more decimal places in the result, I have to round
`B/C`, which will get me closer to matching Oracle but with some error a few
more decimal places out.

But what if you change the formula to `A * (C/B)`, which is exactly the same
thing?

The intermediate result is the same `(38,19)` but because you're multiplying,
you now get a result of `(38,21)` -- with multiplication, the two
operand scales are added, and the precision maxes out at 38.  Further if you
run into an overflow issue, you can round the intermediate result, but we'll
have better understanding of the impact on the accuracy of the final result.
