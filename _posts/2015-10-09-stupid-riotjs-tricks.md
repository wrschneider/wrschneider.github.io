---
layout: post
title: Stupid riot.js tricks
---

I made a [simple Riot.js application](https://github.com/wrschneider/random-list-riot) to try it out.  My demo is as simple as it gets: it
lets you add and remove items from a list, with a link to display an item from the list selected at random.

My initial reaction is mostly positive.  It emulates many of the popular aspects of React.js, like component-orientation and automatic refreshes
via virtual DOM, but with much better readability.

I designed my app as a single custom tag (component), since it's all custom, and the two main sections (list management and random item selection) are both 
tightly coupled to the same state (the list).  There is nothing here that I see as reusable.  That approach worked great, until I wanted to add 
an animation when you select a random item from the list.  

My first attempt was to make a new `highlight` custom tag, which wraps the inner content and listens to the Riot.js `updated` event:

```html
<highlight><span>{foo}</span></highlight>
 
<higlight>
  <yield/>
  <script>
  // script tag only for syntax highlight
  this.on('updated', function() {
    $(this.root).effect('highlight', 500)
  })
  </script>
</highlight>
```

Two problems with that:

* the `updated` event will fire on *any* update within the custom tag.  So updates because the list changed on add/remove will cascade to the 
nested `highlight` tag.

* a new custom tag creates a new nested binding context, so an expression like `{foo}` that worked before now needs to be `{parent.foo}`.   That 
reminds me a little bit about how Knockout.js binding contexts work.  Similarly, bindings for form inputs are pushed down to the child tag as well.

Now for the "stupid tricks" part to address these issues:

* I introduced a new `inline-tag` that serves no useful purpose except to isolate update events.  That saved me from having to introduce 
one-off custom tags like `list-management`, which would break the flow of the page and make readability worse rather than better.  (I would
feel differently if the components were reusable, or if the page was large enough that breaking it into smaller pieces improved readability.)

* I introduced a convention to qualify all references in template expressions with a `ctx` placeholder, and then assign `ctx` 
appropriately in each tag handler.  In the main tag, I assigned `ctx = this`.  In the placeholder tags where I want to automatically refer to 
the parent's context, I assigned `ctx = this.parent.ctx`.   That way, I can refer to `{ctx.foo}` in an expression regardless if it's wrapped
with `inline-tag` or `highlight`.  

* For form inputs, which are bound on the current tag context, I added an option to export the bindings to the parent.  

Purists will probably tell me that I should have made more custom tags rather than coming up with hacks to avoid them.  Also, anyone who has
spent more than the few hours that I have on Riot will probably tell me that there was a better way to handle animation effects.  Still, 
I got it to work, and I feel like the end result is easy to read and understand.
