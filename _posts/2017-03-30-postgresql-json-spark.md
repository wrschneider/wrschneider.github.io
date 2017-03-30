---
layout: post
title: Comparing PostgreSQL json_agg and Spark collect_list
---

In PostgreSQL, you can convert child records to look like a nested collection of objects on the parent record.  This is useful if you want to convert 
a relational-style parent-child model into a document style, with the child records represented as a composite within the parent document. 

JSON functions allow conversion of result set rows to and from JSON.  The `json_agg` function aggregates a list of records into a JSON array of objects.  An example of what this would look like:

```
select parent.id, parent.name, json_agg(child.*) as nested_child
from parent 
join child on parent.id = child.parent_id
group by parent.id, parent.name
```

The resulting JSON column could be stored in a `jsonb` column.

You can do the same thing in Spark SQL as well with the `struct` and `collect_list` functions:

```
select parent.id, parent.name, collect_list(struct(child.*)) as nested_child
from parent 
join child on parent.id = child.parent_id
group by parent.id, parent.name
```

A key difference between Spark arrays/structs and PostgreSQL JSON: Spark SQL is a two-step process.  First  `struct` converts a list of fields into a single `struct` object on each child record.  Then `collect_list` aggregates the structs from the child records into an array per parent record in the `group by`.  PostgreSQL `json_agg` is a single step.

The similarity is that the resulting dataframe can be stored with the nested array of structs in Parquet or Avro format.


