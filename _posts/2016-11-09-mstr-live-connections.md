---
layout: post
title: Microstrategy Visual Insights live connection issues
---

Microstrategy's data import feature is promising, because it can shorten the time to insight.  Instead of architecting a project schema up front (attributes, facts, metrics), you can import tables directly 
into a dataset (Intelligent Cube).  

The new "Connect Live" capability, introduced in MSTR 10, is especially interesting because it gives you the best of both worlds: rapid development by going straight from DB to dashboard, while generating 
queries to push filters and aggregation into the DB just like the traditional ROLAP model.  Because the data is fetched on demand, you don't have to pre-filter the dataset for the use case to keep it under
the in-memory size limit -- filters can be specified on the dashboard directly and those get translated into `WHERE` clauses in SQL.  Likewise you don't have to worry about refreshing the intelligent cube 
when your underlying data changes.

Of course, there's a catch.

I ran into a number of issues with Connect Live:  

* The most troubling one was that live connections would add improper cross joins and [give me *incorrect answers*, when in-memory datasets gave the correct result](https://community.microstrategy.com/t5/Reporting-Dashboards-and/VI-live-connection-giving-wrong-answer-with-cross-join/m-p/313938/).
* Although it looks like you can just "pick tables" and start using them in a dashboard, you often have to [clean up attribute relationships to avoid errors like "SQL Engine 
got an exception from DFC: %1"](https://community.microstrategy.com/t5/Reporting-Dashboards-and/Visual-Insights-live-connection-SQL-Engine-got-an-Exception-from/m-p/313621).  These attribute relationships
can also cause [unexpected cross-joins and wrong answers](https://community.microstrategy.com/t5/Reporting-Dashboards-and/Unexpected-cross-join-with-live-connect-datasource-and-date/m-p/313811).
* Sometimes I was seeing an extra [`select distinct` pass on some dimensions](https://community.microstrategy.com/t5/Reporting-Dashboards-and/Expensive-quot-select-distinct-quot-pass-with-live-connections/m-p/313810) without any filters, which could be expensive.
* There is no "View SQL" option with dashboards, so you have to monitor generated SQL on the DB server itself (assuming you're on Desktop).

Given all these traps with live connections it may be counterproductive to use them for anything non-trivial, like discovery with simple filters and roll-ups.  When it was all said and done, dealing with 
the above issues on a small model might have cost me more time than modeling a classic project schema.

In-memory datasets are an option and tended to work more reliably, but only if your dataset is small, or your use cases can be supported by pre-filtering at the dataset/intelligent cube level.  Also, it is not obvious to an end user that changing your Data Access Type might change your results or break your dashboards -- the only difference between live/in memory *should* be performance and operational characteristics.