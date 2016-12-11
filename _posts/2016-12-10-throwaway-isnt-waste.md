--- 
layout: post
title: Throwaway code isn't always wasteful
---

In a lean/agile environment, you avoid [Big Design Up Front](https://en.wikipedia.org/wiki/Big_Design_Up_Front) (BDUF) in favor of getting something small and simple to work first, then iterating as you learn more.

This approach means you end up revisiting decisions and assumptions made earlier, in light of new information.  The downside of this is you might end up backtracking and throwing out some code as assumptions change.  This can look wasteful--if you spent more time up front talking about requirements and design, couldn't you get it right the first time?

The problem with this line of thinking is, lines of code written and later deleted aren't the best indicator of efficiency.  It's time and cost--how quickly is value delivered with how much effort.  

One challenge of BDUF is it presumes you know exactly what you want.  Often you don't really know until you have something concrete as a baseline.  It may be more efficient to get something 
working first, even if it's wrong, rather than to try to figure it all out on paper.  Once you see something taking shape, you may get more clarity on how to define what you really do want.  If you try to get everything perfect on paper first, there's a good chance you'll be wrong anyway, and it will take you longer to get to that point of clarity.

Another challege is edge cases.  If you don't define all your edge cases up front, you may have to rip up some of your code to account for them later.  The other alternative, though, might result in more 
waste, in the form of edge cases that never happen in practice.  Once I worked on a team that spent a significant amount of effort on edge cases that were purely hypothetical--if we had built the "happy path"
first, we would have had a better idea of which edge cases were worth the effort.  

Finally, there's premature generalization.  If you build something simple first, with limited scope, you might end up rewriting it later to refactor as more use cases are 
introduced.  Throwing that first iteration out might actually be less wasteful than trying to account for the future use cases too soon, though.  If you build in extra layers of abstraction
to account for hypothetical future use cases, building your first use cases is harder.  Later, you may realize that your actual use cases are different than the ones you anticipated, and you're stuck
with the maintenance costs of the extra layers of abstraction.  I've learned this the hard way after building generic platforms that only got used in a single use case.

Of course, this all assumes a typical webapp context, where the cost of rework is lower than the cost of avoiding said rework, and iterating with real users is cheap and 
low-risk.  In a different context, when lives are at stake and failure cases can be objectively defined (embedded software in a car to control acceleration/braking) you would need a different approach.