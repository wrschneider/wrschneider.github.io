---
layout: post
title: Multiple teams, one codebase
---

When multiple (small, agile) teams are working on the same codebase, it can be
tempting to create a branch for each team so they can work  in isolation
without impacting each other.   Don't do it.  Teams working in isolated
branches may appear to make faster progress, but it is an illusion--in
reality, work in an isolated branch can't be delivered without getting through
some big scary merge in the future.  The hard work of integration is just
being kicked down the road and gets harder the longer it waits.  There is no
substitute for coordination and communication.

The whole point of continuous integration is the recognition is that you can't
call your stuff done until it works with everyone else's stuff.  That's why
feature branches should be short-lived, with frequent merges into the main
development trunk.  If your work breaks something else, you want to know about
it and correct it as soon as possible.  That gives you more flexibility  about
when you can release, because you will always have your main trunk in a
working state.  These principles are no different whether feature branches are
created by people on the same team or on different teams.

Of course it's better if a large project can be subdivided into modules with
separate builds/repos, but this does not always make sense.  This only works
if you can divide your repo  along clear domain boundaries, which assumes you
know your domain well enough to get those boundaries right ([which is the
problem with doing microservices too eagerly]({% post_url
2015-11-11-microservices %}));  *and* your teams are aligned with those
module boundaries.  In many cases, teams and their backlogs are fluid such
that there is no clear mapping between teams and repos.

Modularity within a repo *is* a good thing, and is easier to refactor as 
understanding of the domain evolves.  That kind of modularity, along with SOLID
principles like open-closed, make conflicts less likely because developers
working on different stories should generally be touching different files even
if they're in the same repo.