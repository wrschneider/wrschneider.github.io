---
layout: post
title: Notebooks in production vs. JARs and Python Wheels
description: debate over whether you should run Databricks notebooks directly in production pipelines or rely on built artifacts
---

Most cloud-native frameworks for Apache Spark (Databricks, Microsoft Fabric, AWS EMR, etc.) provide a way to not only work
on notebooks interactively, but to automate running those notebooks in production.

Is that a good idea, or are you better off building an artifact (JAR for Java/Scala code or a Python wheel) in a CI/CD process, 
deploying that and running it in production?  (Either way, you can have Git integration and version control.)

As usual, it depends on what you want to optimize for.

Notebooks are great for getting something up and running quickly.  They're also good for troubleshooting and collaboration
on a specific dataset.  The shortest path to get something running in production is to work on a notebook, and when you're happy with it,
set up an automated job to run the notebook.

On the other hand, you _don't_ always want to run exactly the same code you wrote in an notebook in production.  Notebooks are
great because you can interactively build your results one step at a time and inspect each intermediate step along the way.  That
means your notebooks tend to be optimized for interaction and collaboration, rather than performance and scaling.

Also, notebooks can only run if there's a cluster up and running.  If much of your code is plain Spark, you may be better off 
developing and testing code locally, and building an artifact (`jar` or `whl`) with a CI/CD process.

You can write unit tests that use [PySpark sessions in local mode](https://spark.apache.org/docs/latest/api/python/getting_started/testing_pyspark.html) (or similar in Scala) and run these locally, or in your CI/CD process (e.g., Jenkins, Github Actions) without 
a running cluster (and paying for one). Unit tests are especially useful if you synthesize data to test all the potential scenarios
that may appear over time, rather than only covering those present in your development dataset.

Finally, while the notebook developer experience has improved (auto-complete, etc.), being productive in a single notebook doesn't 
translate to being able to manage many notebooks as an integrated whole.  It's not as easy to find and manage reusable utility code, 
or do a refactoring that affect multiple files across a bigger codebase.

I wouldn't call notebooks "untestable, unmaintainable trash" as [one comment on a Reddit r/dataengineering thread says](https://www.reddit.com/r/dataengineering/comments/yyzzfb/comment/ix4mmtk/?utm_source=share&utm_medium=web3x&utm_name=web3xcss&utm_term=1&utm_content=share_button), but there are certainly advantages to developing and testing code outside a cluster, and to take advantage of IDE tooling for refactoring.

I do think it is reasonable to use notebooks in production as a top-level job driver, as long as the total lines
of code in notebooks stays small enough to be manageable (for some definition of "small enough").  I could see a case for notebooks
that let you pass parameters to some other Python or Scala code, and visualize the results, with much of the underlying code 
called from the notebook is deployed to the cluster via CI/CD.