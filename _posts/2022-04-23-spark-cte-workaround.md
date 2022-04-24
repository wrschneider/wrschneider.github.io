---
layout: post
title: Workaround for Spark lack of recursive CTE support
description: How to use Scala code to make up for Spark not supporting recursive CTEs
---

### Problem

Spark SQL support is robust enough that many queries can be copy-pasted from a database and will run on Spark with only minor modifications.  One notable exception is recursive CTEs (common table expressions), used to unroll parent-child relationships.

For example, this will not work on Spark (as of Spark 3.1):

```
with recursive_cte as (
    select child, parent as ancestor, column2, column3, ...
    from my_table
    
    union all

    select t.child, p.parent as ancestor, p.column2, p.column3
    from my_table t
    join recursive_cte p on t.parent = p.child
)
```

Assuming `my_table` contains a list of parent-child relationships, this recursive CTE would return a list of parent-child relationships exploded to include every ancestor for a given record rather than just the immediate parent.  The base case before the `union all` lists the immediate parent-child relationships and then the join after the `union all` goes one level down the hierarchy, with the recursion unrolling all possible relationships (i.e., transitive closure).

For example if you start with

|child|parent|column2|column3|
|-----|------|-------|-------|
|1|null|A|X|
| 2 | 1 |B|Y|
| 3 | 2 |C|Z|

you would end up with 

| child | ancestor |
|-----|------|
| 1 | null |A|X|
| 2 | 1 |B|Y|
| 2 | null | A|X|
| 3 | 2 |C|Z|
| 3 | 1 |B|Y|
| 3 | null || A|X|

Each non-root record is duplicated for every ancestor it has in the tree, inheriting the other columns (column2, column3) from ancestors.

### Workaround

For small lookups that can fit in memory, you can `collect` the input dataframe, unroll the relationships in the driver with Scala (or Python) code, and convert the results back into a dataframe.

Example of what this could look like in Scala:

```scala
// assumes input has two columns: (child, parent)
def unrollParentChildRelationships[T: TypeTag](input: DataFrame): DataFrame = {
val inputMap = input
    .collect()
    .map(r => r.getAs[T](0) -> Option(r.getAs[T](1)))
    .toMap

@tailrec
def recurse(
    remaining: Map[T, Option[T]],
    finished: Map[T, List[T]]
): Map[T, List[T]] = {
    if (remaining.isEmpty) {
        finished
    } else {
        val first = remaining.find { case (_, parent) =>
            parent.forall(finished.contains) // like parent.isEmpty || finished.contains(parent.get)
        }.get
        val id = first._1
        val parent = first._2
        val finalAncestors = parent.map(p => (p :: finished(p))).getOrElse(parent.toList)
        recurse(remaining - id, finished + (id -> finalAncestors))
    }
}

import input.sparkSession.implicits._
recurse(inputMap, Map()).toSeq
    // turn map of child_id -> [ancestors list] into a flat list of (child_id, ancestor) tuples
    .flatMap { case (id, ancestors) => ancestors.map((id, _)) }
    .toDF("child_id", "ancestor_id")
}

// usage
val input = spark.sql("select child_id, parent_id from my_table")
unrollParentChildRelationships[Int](input).createOrReplaceTempView("ancestors")

spark.sql("""
select child_id, parent_id as ancestor_id, column2, column3
from my_table

union all

select a.child_id, a.ancestor_id, t.column2, t.column3
from ancestors a
join my_table t on a.ancestor_id = t.child_id
""")
```

Explanation:

* First, collect the initial dataframe of parent/child relationships and turn it into a Map of child->parent tuples for easier lookup.  (Parent is an `Option` because it might be null.)  Note that the Scala method only deals with the parent/child IDs for simplicity and reuse. The full result set will be created later.
* Next, the tail-recursive `recurse` method is the functional programming / immutable-everything equivalent of a loop:
  * while the initial map is not empty, find the first parent/child entry with a null parent (root) or where the parent has already been processed.
  * once you find that entry, calculate the list of ancestors for the current child, by concatenating the immediate parent, with all of the ancestors for the parent in the list of nodes already processed.  For root nodes, the ancestor list is an empty list, and for entries whose parent is a root, the ancestor list is a single element.
  * The final output of `recurse` is a Map of child to list of ancestors
* Expand the map of child->(ancestor list) into a flat list of child/ancestor tuples
* Turn the final list of tuples back into a dataframe and register it as a temp view.
* The full result set is created with a similar `union all` query but without recursive CTE.  The recursive results are available within the `ancestors` temp view.

There is a bunch of Scala trickery in the above, mostly using (abusing?) Options like collections:

*  `forall` on an Option returns `true` (vacuously) if the Option is None, so you can replace a conditional like `if (option.empty || condition(option.get))` with `option.forall(condition)`
* `map` on an Option combined with `getOrElse`: instead of `if (!option.isEmpty) something(option.get) else foo`, you can replace that with `option.map(something).getOrElse(foo)`.

In both cases, the functional patterns with `forall`, `map` etc. are more succinct and have fewer conditional branches in code, but may be harder to understand because they're patterns typically used on collections, on a single optional value.
