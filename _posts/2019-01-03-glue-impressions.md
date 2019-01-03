---
layout: post
title: First impressions of AWS Glue
---

I've worked with both old-school ETL tools (Informatica, SSIS), and more recently worked with Spark.  My takeaway 
is that AWS Glue is a mash-up of both concepts in a single tool.

The strength of Spark is in transformation -- the "T" in ETL.  Traditional ETL tools or SQL-based tranformation in an ELT
process works well enough for set operations (filters, joins, aggregations, pivot/unpivot) but struggles with more complex
enrichments or measure calculations.  When you have business logic that is based on relationships between 
child objects related to some root object (customer-order-product; student-class-assignment; patients-claim-procedure), these
can be difficult to express efficiently with set operations.  In SQL logic, to avoid cartesian join issues you end up doing 
multiple passes over the same tables.  In Spark, you can [pack the nested child objects](http://wrschneider.github.io/2017/03/30/postgresql-json-spark.html) into structures within the root object itself, then write Scala or Python code
that calculates measures on the root object with all the associated child objects in memory, in parallel.  As a bonus, 
in-memory business logic can be unit-tested outside Spark.

Conversely, ETL tools are good at making it easy to do simple cleansing or mapping transformations on data -- for example,
discovering schemas from files or tables, then providing simple transformations to rename or reorder columns through drag-and-drop,
or filter out rows.  These tools don't support the more complex logic, and worse, they tend not to have good support
for abstractions or reuse (e.g., for each column apply same transformation).  Mappings can be brittle and hard to maintain, 
and need to be explicitly updated when columns are added.  

AWS Glue seems to combine both together in one place, and the best part is you can pick and choose what elements of it you
want to use.  The Glue catalog and the ETL jobs are mutually independent; you can use them together or separately. 

* The Glue catalog plays the role of source/target definitions in an ETL tool.  Like with ETL tools, it can be defined 
explicitly, or it can discover from DB schema or files.  This can also be a place to manage DB connections (including 
credentials) across multiple ETL jobs.

* A Glue job is like a serverless Spark job.  Glue can generate code for a job automatically from a source, target
and drag-and-drop mapping.   Analogous to an Informatica mapping or SSIS data flow task, common transformations (splitting or 
joining fields etc.) are point-and-click or at least simplified.  But if you want more complex logic, you can write whatever
code you want.  You can manipulate DynamicFrames just like DataFrames.  You can treat Glue like Lambda-for-EMR,
and ignore the catalog and mapping capability.  You can ignore the catalog for individual tables, and still use the
connection abstractions for DB targets (`from_jdbc_conf`).  

* You can go the other direction and use the Glue catalog with EMR as the Hive metastore.  This is useful to share the 
same catalog across multiple EMR clusters.  