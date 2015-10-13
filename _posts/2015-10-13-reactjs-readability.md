---
layout: post
title: React.js, JSX and readability
---

React.js is the new hotness in front-end JS frameworks, although I am having a hard time getting excited about it.  I'm reacting (pun intended) mostly to readability 
concerns with JSX.  I also have issues about slicing everything up into granular components, but that's a topic for another day.

[One presentation](https://docs.google.com/presentation/d/1afMLTCpRxhJpurQ97VBHCZkLbR1TEsRnd3yyxuSQ5YY/edit#slide=id.g380053cce_1205) about React makes the statement: 

> "If you’re going to hate on React for some reason, make it something other than JSX"

I believe JSX is a valid concern, though, because React.js's decision to eschew HTML templating has a real impact on readability -- which the same presentation acknowledges.  It 
may be a valid point that "templates separate technologies, not concerns," but that does not negate the value of templates for HTML generation.  I feel like React.js and its fans
have dismissed those readability concerns too quickly.  

Here is a simple example to illustrate.  In Angular (and Knockout, Ember, etc.) you can something like this:

```html
<ul>
  <li ng-foreach="item in list">
  {% raw %}{{ item }}{% endraw %}
  </li>
</ul>
```

By comparison, in React, you have to intermingle Javascript code and JSX elements.  You can minimize the damage by using functional patterns:

```
return (
  <ul>
  {
   list.map(function(item) { 
     return <li> {item} </li>
   })
  }
  </ul>
)
```

But it's still hard to read, in part because of the back and forth between Javascript and HTML syntax.   Indeed, Bob Martin's Clean Code lists this mixing of languages as a 
smell.  The alternative -- breaking the flow and creating a separate component to render the list item -- is not much better, unless the component is reusable.  

Sure, JSX is just shorthand for Javascript, so you don't have to use it.  But I don't see how the alternative of constructing HTML elements as Javascript objects is an *improvement* in 
readability. For better or for worse, HTML is the default markup language for the web, and that's what most people are used to working with.  

In some ways the JSX syntax and the arguments that go with it remind me of the early days of JSP -- JSPs are just Java!  They're optional! They just compile to servlets!  And JSP 
readability was such a disaster, that this was addresed over time by moving closer to a domain-specific templating language with JSTL tags and EL for loops and conditionals.  Despite 
all JSP's problems, though, it was better than the other options like generating your HTML with `out.println` or instantiating Java objects for each DOM element.  So
forgive me for thinking that JSX feels like a step backwards.

This does not take away from the other cool ideas in React -- that it's a simple, view-only framework, and that you can re-rendering everything with good performance through 
virtual DOM diffing.  Still, readability matters, and there's at least one framework out there ([Riot.js](http://riotjs.com)) that shows that the good ideas in React are not 
mutually exclusive with readability.  Besides, if JSX just compiles to Javascript, why is it too much to ask for a `<foreach>` tag that hides the `list.map(function(item)...` 
syntax?  I'm even OK with having JSX/HTML in the same file as the rest of the component Javascript, like Riot.js does, as long as you don't have to jump back and forth between 
the angle brackets and curlies/`function`. 

And yes, I am aware of [React Templates](http://wix.github.io/react-templates/).  The fact that it's a separate library is not a concern *per se*, but it seems to be anathema to 
React's position on templating.

