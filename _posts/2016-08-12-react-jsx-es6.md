---
layout: post
title: React, ES6 and JSX readability
---

Previously I complained about [React and JSX readability]({% post_url 2015-10-13-reactjs-readability %}). I was unhappy with something like this:

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

But if you rewrite it with arrow functions, it's not as bad:

```
return (
  <ul>
  {list.map(item =>  
     <li> {item} </li>
  )}
  </ul>
)
```

