---
layout: post
title: Query cache without L2 cache
---

(I'm writing about NHibernate, but this applies equally to JPA in the Java universe. I'm not sure about Entity Framework.)

Turning on the NHibernate query cache without the L2 cache can actually make performance worse, rather than better.

Here's why: the query cache contains identifiers of the objects in the result set, not the objects themselves.  Object caching is separate, in an L1 or L2 cache.  Whenever 
NHibernate runs a query, NHibernate tries to fetch each object in the result set from the L1/L2 cache, 
and constructs new objects from the query results if necessary.  When NHibernate has a query cache hit, it skips running the query, takes the list of object IDs from the 
query cache, and then tries to retrieve or construct the objects as usual.  The problem is, if those objects are not in the L2 cache, each individual object is fetched 
from the database in its own round trip. In the logs, it will look similar to the N+1 selects problem.  

Here's an extreme example of a query that loads a full table (e.g., a lookup) with NHibernate fluent interface:

```
session.Query<MyLookup>().Cacheable().ToList()
```

The `Cacheable` method will cache the query results in the query cache.  But that only caches identifiers.  So the first time everything is OK.   But when you run this a second 
time, the identifiers are retrieved from the query cache and, if `MyLookup` isn't cached in the L2 cache, there's a round trip for each row.  

The right answer in this case is to either use the L2 cache (a good option if the objects are read-only) or avoid the query cache.  Turning on the query cache without understanding
how it interacts with the L2 cache risks turning a simple "select * from table" into "select * from table where id = ?", N times.