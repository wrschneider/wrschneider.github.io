---
layout: post
title: Foo/FooImpl pairs - stop doing it!
---
I was wondering why so many people are still creating interfaces for their class, instead of just creating the class, 
when there's only a single implementation.

I came across [this blog post from Adam Bien](http://www.adam-bien.com/roller/abien/entry/service_s_new_serviceimpl_why) on the topic, which 
explains the drawbacks: it creates redundancy (breaking DRY), and makes it harder to navigate through the IDE.  Hitting F3 in Eclipse 
or F12 in Visual Studio on a method call takes me to the interface definition, when I really want to get to the method body.  That breaks my flow
and makes me do extra work to go find the actual implementation class.  This seems like a minor annoyance, but considering that you spend most of 
your time reading code (Uncle Bob claims the ratio is 10-to-1) the impact on productivity is not insignificant.

Martin Fowler also agrees, in his post about [interface-implementation pairs](http://martinfowler.com/bliki/InterfaceImplementationPair.html).

In the Java universe, it seems like a holdover from the early Spring 2.x and EJB 2.x days, pre-CGLIB, where interfaces were a necessary evil for 
dynamic proxies and dependency injection.  Before we had good mocking frameworks, we might have a different concrete implementation for testing.  Also, 
EJB sucked so we had interfaces as a survival strategy to allow us to swap out POJO and EJB implementations.  In C#, perhaps the practice may have 
evolved from the C/C++ practice of header files, as Martin Fowler's article suggests. 

Today, those constraints are gone.  Also, test implementations are often created dynamically through frameworks like EasyMock or Mockito.  In 
other words, the codebase has a Foo interface with FooImpl as the only implementation - there is no FooTest or OtherFoo.  Interfaces are *not* required
for mocks - EasyMock and Mockito can both mock classes without interfaces.  So can Moq in C#, although you need to mark the methods `virtual`, 
given the opposite default from Java, where all non-`final` methods can be overridden.  [Microsoft's example on mocking 
DbContext](https://msdn.microsoft.com/en-us/data/dn314429.aspx) illustrates this use of the `virtual` keyword in C#.

Foo/FooImpl pairs today remind me of this quote from _The Fountainhead_:

> The famous flutings on the famous columns – what are they there for? To hide the joints in wood – when columns were made of wood, 
> only these aren’t, they’re marble. The triglyphs, what are they? Wood. Wooden beams, the way they had to be laid when people began to 
> build wooden shacks. Your Greeks took marble and they made copies of their wooden structures out of it, because others had done it that 
> way. Then your masters of the Renaissance came along and made copies in plaster of copies in marble of copies in wood. Now here we are, 
> making copies in steel and concrete of copies in plaster of copies in marble of copies in wood. Why?

Foo/FooImpl pairs are now like copies in steel of copies in wood - they've outlived their purpose, but we keep using them 
out of habit.  And yet they seem to have taken on a life of their own, as a symptom of over-applying the
[Dependency Inversion Principle](https://en.wikipedia.org/wiki/Dependency_inversion_principle) at each layer of method calls, rather than 
only at architecture boundaries.  