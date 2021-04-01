---
layout: post
title: SQL Server, JDBC and compiled query plans
description: Sometimes SQL server generates a bad query plan without knowledge of parameters
--- 

I had a query like this

```sql
select ... from big_table where key in (select key from other_table where id between @p1 and @p2)
```

I had noticed it was performing badly, and the plan didn't make any sense.  The values for `@p1` and `@p2`
typically return a small number (200-500) rows from the subquery, which in turn retrieves a small fraction of the overall
rows in `big_table`.  Yet the estimate was off by an order of magnitude, showing that the subquery was expected
to return about 1000x the number of rows it would in practice.  

I noticed if I ran `DBCC FREEPROCCACHE` or `sp_recompile`, I could force the query to be recompiled with the plan
I expected.  So I assumed the issue was parameter sniffing, and maybe somehow the query compiled originally with
values that triggered a different plan.  But it turned out in this case, it was exactly the opposite--my problem
was that parameter sniffing was *not* happening.  

I was able to show this with a test case to reproduce my problem within SSMS.

If I ran the query with `sp_executesql` multiple times, with different parameter values, the parameters on the
first execution would be "sniffed" as I expected and subsequent executions would use the same plan with frozen estimates.

If I ran the query wuth `sp_preparesql` instead, then the query plan is compiled with *no* known parameter values, similar
to what I get with `OPTIMIZE FOR UNKNOWN`.

After I found [this post](https://www.brentozar.com/archive/2018/03/sp_prepare-isnt-good-sp_executesql-performance/), I
believe that the JDBC driver must be using `sp_preparesql` under the hood, and there is likely a setting I could tweak to
resolve the issue and get the expected plan.
