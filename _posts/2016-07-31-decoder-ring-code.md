---
layout: post
title: Good-enough secret message encoder on JSFiddle
---

One of the activities at my daughter's 9th birthday party was decoding secret messages (e.g., "be sure to drink your Ovaltine") using 
this [decoder ring my wife found on Pinterest](http://dabblesandbabbles.com/printable-secret-decoder-wheel/).  

Encoding several messages by hand was time-consuming and error-prone, so naturally my impulse was to [write some code to do it](https://jsfiddle.net/ut3pw9zj/). 
The code itself is trivial although it was an excuse to try out some ES6 features like the `...` spread operator to turn a string into a character
array, which then allowed me to use a pure-functional map-reduce style with no mutable state.  (Incidentally, kids decoding the messages in teams realized that they could split up
the words and work in parallel.  None of them recognized this as a "map" function, though.)

The more interesting takeaway is the value of "good enough."  The key constraint was that, the original task was probably going to take less than an hour,
so there's a tight bound on how much time I could spend before it turned counterproductive.  (See [xkcd's take on this](https://xkcd.com/1205/).)  So the HTML form interface was crude but good enough.
There is no error handling.  It only encodes messages and doesn't decode, because that's all I needed.  JSFiddle was fastest way to get up and running with something, with the constraint that I wanted something that could be accessed via URL.  I could have done this even faster with command-line Python,  but that wouldn't have been "good enough."  

Also, for something this trivial, I find JSFiddle's interface helpful because you can see the Javascript, HTML and end result all at the same time instead of flipping between windows.  As an added bonus, if my daughter wants to code more messages with this later, she'll see the code too.