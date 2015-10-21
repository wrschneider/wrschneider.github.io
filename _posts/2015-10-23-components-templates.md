---
layout: post
title: React.js and Components vs. Templates
disqus: false
---

"React is all about modular, composable components," according to the [tutorial](https://facebook.github.io/react/docs/tutorial.html).  [Riot.js](http://riotjs.com)
emulates the React philosophy that "templates separate technologies, not concerns," so they "focus on reusable components instead of templates."  As far as I can tell,
[Angular 2.0](http://angular.io) embraces components as well.

Components are great, but what about building an application?  Within an application, only a handful of things may be generic, reusable components.  Much of the UI in a custom application
is, well, custom.  The limitations of React's JSX syntax, in particular the [difficulty of doing conditionals or loops in-line](https://facebook.github.io/react/tips/if-else-in-JSX.html),
pushes you to slice up your pages into granular components (e.g., `CommentList`, `Comment`, etc.)  This is fine if those components are reusable, but if they're one-offs, it breaks
the flow and makes the code harder to read.  I've worked on projects where the team moved the opposite direction and consolidated their JS into *less* granular components, because the 
context-switch between multiple files to complete a single task created too much friction.

If React embraced conditionals and loops in JSX, it might be possible to treat a custom panel as a single monolithic component, like you
could in Angular 1.x (not sure about 2.x), Knockout or Ember.  There might be some challenges in isolating changes to portions of the panel (for example, to recalculate computed 
properties), because `render` re-renders the whole component, and React's solution is to split things up into more granular components whose state can be set and which can be re-rendered 
independently.  I found [ways to work around this in Riot.js]({% post_url 2015-10-09-stupid-riotjs-tricks %}), by introducing inline 
component demarcations to isolate update events, but I admit this is a crude POC that might not always work well in practice. 

Then again, this all might be moot when we can stop supporting IE8 and overload `Object.defineProperty` to transparently watch objects for changes, like [vue.js](http://vuejs.org) does,
 or take advantage of `Object.observe` when browsers support it natively.  When old versions of IE finally die, maybe the JS world will change yet again.