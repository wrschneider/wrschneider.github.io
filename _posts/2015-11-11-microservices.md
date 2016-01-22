---
layout: post
title: Microservice first?
---

One argument I've heard for building microservices from the beginning is that, teams often lack discipline to enforce modularity and separation of 
concerns.  A monolith usually turns into a "big ball of mud," which is a problem.  Starting with microservices forces teams to think about 
their module boundaries up front.  

There are two big catches though.  The first is, splitting an application into microservices isn't free.  There is some loss of development efficiency from context-switching 
between different repos, different builds and deployments, maybe even different teams.  Turning in-process local method calls into remote calls over HTTP adds additional concerns 
like authentication, serialization, reliability, etc.  Those penalties could be offset by the benefits of loose coupling: microservices and their consumers can change 
independently, on their own time scales, and leveraged by other consumers outside the original application scope.

Which brings me to the second catch: early on in a project, module boundaries tend to be unclear and fluid, yet the benefits are predicated on getting the module boundaries in the right
place.  If they're in the wrong place, you won't get the benefits, and you'll suffer from the context-switching of changing both the services and their consumers at the 
same time.  (Think about the Single Responsibility Principle in reverse -- just like you would limit the number of reasons to touch a piece of code, you should limit the 
number of pieces of code to touch for one reason.)  Plus, once you realize that you didn't get the boundaries right the first time, it's harder to refactor.   You still have the 
same big ball of mud, but with a lot more moving parts.   

In other words, loose coupling comes from getting the right boundaries, not from the technology.  You can't just pull some chunk of code out into another process and 
call it via HTTP endpoint and say you're loosely coupled.  You should think of *loose coupling as the cause, not the effect, of microservices.*

When I was searching around for material to support my opinion, I was relieved to find that [Martin Fowler took a similar position](http://martinfowler.com/bliki/MonolithFirst.html),
perhaps more articulately: "microservices ... only work well if you come up with good, stable boundaries between the 
services ... even experienced architects working in familiar domains have great difficulty getting boundaries right at the beginning."  [Another linked 
article](http://samnewman.io/blog/2015/04/07/microservices-for-greenfield/)  concurs that because "getting service boundaries wrong can be expensive" and because it becomes 
"easier to find those stable boundaries" later in the lifecycle after a team has a chance to get a clearer understanding of their domain, it is better to 
"be cautious [and] only split around those boundaries that are very clear at the beginning."

Which makes me wonder, if the lack of discipline around modularity is really a lack of clarity.  And if discipline is the primary issue, will the same team be disciplined 
enough to take on the harder task of refactoring boundaries between independent microservices as their understanding of the domain evolves?

On a separate note, it's debatable whether "singleton consumer" - a service with a single consumer - should be considered a smell, because it depends on a number 
of other factors.  It might be a sign that the service and consumer are tightly coupled, despite an HTTP endpoint creating the appearance of loose coupling.  But it could
also mean that the boundaries are correct, and no one else has the need to call your service yet.  Still, in the latter case, it could be a sign of missed 
opportunity - would you have been better off spending more time on features or user experience instead of the microservices overhead?