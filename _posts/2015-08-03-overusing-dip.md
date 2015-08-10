---
layout: post
title: Overusing dependency inversion principle
---

A few days ago I wrote about [overusing interface-implementation pairs](/2015/07/27/foo-fooimpl-pairs.html).

This is a symptom of the deeper problem of over-applying the [Dependency Inversion Principle (DIP)](https://en.wikipedia.org/wiki/Dependency_inversion_principle), which is the 
"D" in SOLID.  

Uncle Bob's article on [when to mock](https://blog.8thlight.com/uncle-bob/2014/05/10/WhenToMock.html) is a good guideline for when to apply DIP as well -- only at the seams.  In a typical 
webapp, the seams are HTTP, the database, and calls to external APIs or shared libraries.  Method calls contained entirely within the webapp are usually not seams.  When you find 
yourself complaining about IDE navigation -- that F3/F12 takes you to an interface definition rather than the method implementation you were expecting, and the interface mirrors
the implementation one-to-one -- it's a sign that you are treating something as a seam when it really isn't.

How I translate this in practice, in typical webapp development:

* Limit the logic in methods that are dependent on HTTP requests/responses.  The goal should be that you can test meaningful business logic directly, without requiring mock HTTP 
requests or responses.  Note that you don't have to always pull this logic into a separate class, until your controller class gets large
enough that it needs to be split up.  

* If you do split out logic into a separate service class, it's OK for your controller to directly reference that class.  The seam is the controller's HTTP interface, and
the controller-service handoff is a regular method call, not another seam.   Besides, there is little value in unit testing controllers with a mock 
service, if all the business logic is extracted to the service.  You can test the controller with a mock HTTP request and let it call the real service (perhaps mocking the DB at the 
other end), or do an end-to-end integration test.

* Wrap your database access in a thin data-access layer so you can mock DB access in controller or service unit tests.  ORM tools like Entity Framework, JPA etc. are already wrappers 
around the database and you may not need to add additional layers -- in other words, you don't need a wrapper around a wrapper.  But, in some cases, introducing another layer may make 
sense, for example, to encapsulate ORM operations more complex than basic CRUD.

* Don't feel obligated to add layers for the sake of adding layers.  You might not always need a controller, service, repository *and* ORM.  The ORM might serve 
as a repository, and your service could call ORM directly.  Heck, your controller could even call ORM directly, in simple cases when there is no other logic.

* Calls to domain classes are also not seams.  And if you have business logic that depends entirely on properties of a domain class, it can go in the domain class itself.  Domain
objects are objects, after all, and there's no rule that says they must be "anemic" with only getters and setters.
