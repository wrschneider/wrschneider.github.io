---
layout: post
title: Measuring AWS Redshift Query Compile Latency
---

AWS is transparent that Redshift's distributed architecture entails a [fixed cost every time a new query is issued](http://docs.aws.amazon.com/redshift/latest/dg/c-query-performance.html).  The documentation says the impact "might be especially noticeable when you run one-off (ad hoc) queries."

I went deeper to try to quantify exactly what "noticeable" means.

To isolate the impacts of data cache hits/misses from query compilation, I ran a bunch of queries on empty tables so there is no data to load or cache. Each query was 
slightly modified to trigger a recompilation, by changing the columns or aggregate functions.

I found that the compile latency scales with the complexity of the query.  

* Simple query: usually between 1-1.5 sec, with an outlier around 3 seconds.  Example of a simple query:

``` 
select sum(a1) from foo where a2 = 1;
select sum(a2) from foo where a3 = 1;
-- etc.
```

* More complex query with more conditions, and group-by: usually around 2-3 seconds.  Example of a query in this category:

```
select a8, a9, sum(a1), sum(a2) 
from foo
where foo.a3 > 10 and foo.a4 < foo.a5
group by a8, a9;
```

* Even more complex, with joins and group-by: average around 5 seconds, ranging between 3-7 seconds. Example query:

```
select s, s2, count(a6), sum(a7) 
from foo
join bar on bar.a = foo.a6
join baz on baz.b = foo.a7
where foo.a3 = 1 and baz.s2 is not null
group by s, s2;
```
