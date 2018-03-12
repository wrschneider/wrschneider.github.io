---
layout: post
title: MSTR FFSQL vs. Live Connection
---

MicroStrategy offers two different ways to connect to databases with ad hoc SQL, bypassing the managed schema (metrics and attributes):

* Free-Form SQL Reports (FFSQL)
* Add External Data / Connect Live

### Summarizing the main differences

* FFSQL reports can only be created through the Developer tool, while "Connect Live" is available through the web.  The latter has a
friendlier user experience for analysts who are good with SQL but not as familiar with MSTR project development.

* FFSQL reports can include prompts, just like regular reports except the SQL is hard-coded instead of generated 
automatically from metrics and attributes.  "Connect Live" SQL is not prompted.  

The "Add External Data" abstraction is fundamentally different than the FFSQL Report.  Your SQL query needs to be written in a way that can support the 
dataset as either a live connection *or* an in-memory dataset, which means the query looks more like an extract than a report SQL.  Filtering and aggregation would 
be performed as a second step, outside of your query, like a View Filter or dynamic aggregation.  

"Connect Live" will *usually* push filters from your Dashboard (or Dossier) to the SQL query, as a wrapper around the query you provided.  The final SQL will look something like:

```
select a11.attribute_column, sum(a11.metric_column)
from (
   -- your datasource SQL here -- 
   select attribute_column, metric_column, filtered_column, ... 
   from my_table 
   ... 
   ) a11
where a11.filtered_column = x
group by a11.attribute_column
```

The SQL you provide is wrapped into a subquery, with the filtering and aggregation for the visualization in the outer query.  Your query does not have the filter condition on it; it is supplied
later by the visualization.

There are some use cases where this breaks down and MSTR loads your whole query result set into memory first before it filters and aggregates; I don't have a clear picture yet of what exact use case causes that. 

### Impact on row-level security 

You might add granular user-level security to a FFSQL report by incorporating the system prompt directly into the SQL:

```
select ... from fact_table 
where attribute_column in (select distinct attribute_column from user_attribute_security where user = ?[user login])
```

In Connect Live, because prompts are not part of the SQL, you have to approach the problem differently.  You would instead make sure the username column is reflected in the result set, and map it to a Project Attribute 
with a security filter that qualifies on the username attribute.

```
select ... from fact_table 
join user_attribute_security on attribute...
```

This would cause the inner query to return the username as a column, and the outer query wrapper would include a filter on the username.  
Note that this approach will only work well with Live datasets; it will break down with scale issues for In-memory.  This is a fundamental architecture issue not limited to MSTR; Tableau has 
["its own complications around row duplication and performance."](https://onlinehelp.tableau.com/current/pro/desktop/en-us/publish_userfilters.html).
