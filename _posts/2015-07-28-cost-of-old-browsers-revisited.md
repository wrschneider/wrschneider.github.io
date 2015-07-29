---
layout: post
title: Cost of supporting old browsers, revisited
---

As I was starting to pick up blogging again, I came across a post I wrote over 10 years ago about
 [having to put up with older browsers](/2005/06/04/cost-of-supporting-old-browsers.html).   It was kind of funny to see how back then, I was 
 excited about IE _6_ and this new thing called AJAX.  Also, that back then, it seemed like a novelty to prioritize support for 
 different operating systems and browsers based on real usage metrics. 
 
Some things haven't changed: old browsers are still a pain, and there's still a sizeable number of users in various industries stuck 
on Windows XP and therefore IE8 -- the 
[US Navy is one of the more public examples](http://arstechnica.com/information-technology/2015/06/navy-re-ups-with-microsoft-for-more-windows-xp-support/).

But a number of things are different this time around.

* The pain from IE8 is both more subtle and insidious.  Rather than being obviously broken up front, IE8 seems to work well enough, 
until it kills you in rendering and JS execution performance.  Those performance issues may not become obvious until your 
app or feature already looks "done".  And once you realize you have a problem, it's not a matter of tweaking CSS or putting up with square 
corners instead of round -- you need to re-think your UI design approach.

* On the other hand, Chrome is fast, and can be installed with user-level permissions.  

That makes me wonder, between the pain of IE8 and a viable alternative, is the industry now at a point where we can kick IE8 to the curb and say,
if you don't have the latest IE, just go install Chrome?

