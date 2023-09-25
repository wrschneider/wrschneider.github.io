---
layout: post
title: More Spark workarounds for recursive CTEs
description: Spark does not have support for recursive CTEs, so some workarounds are necessary
---

I previously wrote about the [lack of recursive CTEs in Spark SQL]({{ site.baseurl }}{% link 2022-04-23-spark-cte-workaround.md %}) for parent/child hierarchies.

This may be addressed in a [future update to Spark](https://github.com/apache/spark/pull/40744).

In the meantime there are workarounds:

- Pull the parent/child lookup into an in-memory collection, and unroll the hierarchy in regular Scala or Python code
- Use Scala or Python recursion to build up the equivalent recursive joins/unions in Spark SQL

My [previous article]({{ site.baseurl }}{% link 2022-04-23-spark-cte-workaround.md %}) gave an example of how to unroll a hierarchy in Scala.

I found a more succinct way to do this with the help of Github co-pilot:

```scala
@tailrec
def recurseMap[T](childToParentMap: Map[T, T], currentMap: Map[T, Set[T]]): Map[T, Set[T]] = {
  val nextMap = currentMap.map { case (child, parents) =>
    val nextParents = parents ++ parents.flatMap(childToParentMap.get)
    child -> nextParents
  }
  if (nextMap == currentMap) {
    nextMap
  } else {
    recurseMap(childToParentMap, nextMap)
  }
}

val childToParentMap = baseDf.select("id", "parent_id")
    .collect().map(r => r.getAs[String](0) -> r.getAs[String](1)).toMap

val result = recurseMap(childToParentMap, childToParentMap.mapValues(Set(_)))
result.toSeq.flatMap { case (child, parents) =>
    parents.map(parent => (child, parent))
}.toDF("id", "parent_id")
```

The usage pattern is exactly as it was before; you get a parent-child lookup into memory
from a DataFrame using `collect`, recurse in memory, and turn the data structure back into a DataFrame with `toDF`.

The recursion keeps track of a `Map` of elements to a `Set` of ancestors.  On each
execution, the method adds the next level of parents to each ancestor in the list.  The
`Set` ensures that no elements are added twice.  If there are no more to add, and the
iteration stops.

Again, this only works if the parent/child lookup is small enough to fit in memory.  If 
you would have `broadcast` this lookup for joins, `collect` should also work.

There is another option which uses recursion with Spark DataFrame operations:

```scala
  @tailrec
  def recurseAncestors(baseDf: DataFrame, currentDf: DataFrame, prevLevel: DataFrame): DataFrame = {
    val nextLevel = prevLevel.as("_prev_level")
      .join(baseDf.as("_base_level"), expr("_prev_level.parent_id = _base_level.id"), "inner")
      .select("_prev_level.id", "_base_level.parent_id")
      .filter("parent_id is not null")
    // TODO: check for cycles
    // TODO: cache/persist/checkpoint nextLevel to avoid re-execution for count
    if (nextLevel.count() > 0) {
      recurseAncestors(baseDf, currentDf.union(nextLevel), nextLevel)
    } else {
      currentDf
    }
  }

  def recurseAncestors(baseDf: DataFrame): DataFrame = recurseAncestors(baseDf, baseDf, baseDf)

  val parentChild = spark.sql("select id, parent_id from ...")
  val ancestors = recurseAncestors(parentChild)
```

Assuming you start with a base DataFrame with two columns, `id` and `parent_id`.
What this will do is, create the next level of recursion for the UNION, by 
joining the previous level to the base looup, checking if there are any new rows
to add, then unioning the next level.

More explanation:
* `currentDf` and `prevLevel` both start out with contents of `baseDf` (lookup of direct parent/child relationships)
* In th first iteration, `nextLevel` is a self-join on `baseDf`, and then unioned to `baseDf` 
* At the start of the second iteration, `currentDf` is equivalent to `baseDf.union(baseDf.join(baseDf))`.  `nextLevel` is `baseDf.join(baseDf.join(baseDf...))`, two layers of self-joining.
* At the start of the third iteration, `currentDf` is `baseDf.union(baseDf.join(baseDf)).union(baseDf.join(baseDf.join(baseDf...)))`
* Recursion stops when `nextLevel` returns no rows, because the number of joins exceeds the depth of the hierarchy

The DataFrame for the final result, is equivalent to what a recursive CTE would have done:
```sql
baseDF
union all
baseDF join baseDF
union all
baseDF join baseDF join baseDF .... 
```
The downside of this is, with Spark dataframes being lazy, scans on underlying tables 
will be re-executed and joins will be re-executed multiple times.

In particular each level added to the `union` will be executed independently, to 
check whether any new rows are added and break out of the recursion.

So this is only a good solution for hierarchies that are wide but not deep -- too many
records to fit into memory, but not too many levels of recursion. 

Also, this could be worked around with caching, persistence or checkpoints, to prevent
re-execution of intermediate results for checking counts.  It looks like the 
[PR for adding Recursive CTE support to Spark](https://github.com/apache/spark/pull/40744) does exactly that and would make the caching mechanism configurable.