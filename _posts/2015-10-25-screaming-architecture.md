--- 
layout: post
title: Screaming Architecture translated to practice
---

I like Uncle Bob's post on [Screaming Architecture](https://blog.8thlight.com/uncle-bob/2011/09/30/Screaming-Architecture.html).

On a small scale, I can visualize how code should look like what it does: meaningful identifiers, functions that do one thing, etc.  On a larger
scale, the challenge is getting the right balance between embracing frameworks and keeping them at arms' length.  Tight coupling to frameworks
can add friction by obscuring your intent.  But adding too many extra layers to hide the frameworks can add even more friction, especially if the
extra layers prevent you from using the frameworks idiomatically.

So in practice, if I'm writing an ASP.NET application with Entity Framework and Angular, my application is going to look like I'm using all those
frameworks, but I'm going to use them in a way that my domain logic is still clear and at least somewhat framework-agnostic.  The sweet spot I've 
found with both Spring and ASP.NET is something like:

* Keep controllers thin, with business logic handled elsewhere.  That way you can invoke business logic directly for test purposes.
* Don't be afraid to put domain logic directly in domain objects, when the logic is local to a single entity.  The whole point of ORM is to map
real domain objects -- if the domain object only represents a row in a table, we could just use hashes.
* ORM is already an abstraction layer around the database so it's often not necessary to put another layer around the ORM.  
* On the client side, presentation code is likely coupled to a framework, but there may be some logic that can be extracted and invoked 
independently.  That can be isolated the same way as you would separate a service from a controller on the server.
