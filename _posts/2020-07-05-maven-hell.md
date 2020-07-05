---
layout: post
title: Maven dependency conflict resolution is annoying
description: Maven's nearest-wins strategy for resolving transitive dependencies is usually the wrong answer
---

With Spark development, I am frequently running into dependency conflicts because of
[Maven's "nearest wins" strategy](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html)
for resolving transitive dependencies.

The scenario I have frequently is something like this

* one of the Spark libraries depends on `jackson-databind` version 2.6.7
* `jackson-databind` depends on `jackson-core` 2.6.7
* Some other dependency in my POM depends on an older version of `jackson-core` (say, 2.3)
* Maven resolves the conflict by giving preference to the "nearest", or more direct, dependency -- so the
older version of `jackson-core` wins because it's one level closer to the root of the dependency graph
* `jackson-databind` now breaks against the old version of `jackson-core`

For any one given library, this can be solved easily in the POM by either excluding
offending transitive dependencies, or by explicitly specifying version numbers.

But this is a huge hassle at scale, and it's super-frustrating that Maven does not
implement an option for a "newest wins" strategy instead.  By not having such a feature,
every single Maven project out there is effectively implementing a one-off "newest wins"
on any library that encounters a conflict. 
