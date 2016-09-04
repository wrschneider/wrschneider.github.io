---
layout: post
title: Is there a good reason to use SSAS cubes anymore?
---

After using SSAS cubes on a project recently, I'm not sure I would do it again. Some of my observations:

* The performance benefits in SSAS come from compressed, columnar storage with batch computation; this reduces I/O and limits the need for maintaining various pre-built aggregate tables and covering indexes for reporting queries.  Column stores are now available in mainstream relational databases: Oracle and SQL Server have column stores, and there is an open-source column store extension for PostgreSQL (citus_fdw).  There's also MonetDB, and Greenplum was recently open-sourced too.

* SQL and database design knowledge are more ubiquitous than MDX and cube design.  For anything non-trivial, the learning curve of cube design and MDX is going to eclipse the benefits.  It doesn't help that MDX sort of looks like SQL but is fundamentally different.

* Analytics tools (Tableau, Qlik, MicroStrategy, etc.) support cubes but tend to treat them as second-class.  Also these tools often provide their own in-memory OLAP engines with columnar storage, so what a cube gives you is duplicative.

Two other sources worth mentioning:

* ["Cubes are 1980's technology"](https://www.interworks.com/blog/dmurray/2014/05/19/3-reasons-you-should-kill-your-data-cubes)
* ["As a result of columnstore performance, the customer retired their SSAS infrastructure."](https://blogs.msdn.microsoft.com/jimmymay/2014/05/26/columnstore-case-study-2-columnstore-faster-than-ssas-cube-at-devcon-security/)

The exception case might be, if you're planning to commit to an all-Microsoft stack with Excel pivot tables or SSRS as your front-end, cubes might still make sense.  But for custom webapps or a standalone analytics tool, cubes are probably not the right answer anymore.