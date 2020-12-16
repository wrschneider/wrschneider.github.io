---
layout: post
title: Code reuse - REST API vs. embedded library
description: Pros and cons of reusing code via REST API vs. distributing libraries
---

There are two popular ways of packaging code for reuse (besides copy-paste):

* A REST API that you can invoke through HTTPS (using "REST" loosely, in the sense of "modern and JSON-based")
* A library that you can embed in your runtime as a dependency (JAR, nuget, npm etc.)

Each approach has different pros and cons and the right approach depends on how you want to weigh those.

REST APIs are better because:

* They can be consumed from any technology stack
* Callers can automatically get the latest version
* Complex runtime dependencies within the API are abstracted from the caller, via process isolation
* Onwers can offer more fine-grained control over intellectual property (no option to decompile and tweak)

On the other hand, embedded libraries are better because:

* Clients can keep using old versions indefinitely, without any overhead to the API maintainer.
* No latency inflation from HTTPS round-trip times (network latency, SSL/TLS overhead etc.)
* No availability concerns; a consumer is not impacted by someone else's downtime or maintenance windows

My general rule of thumb is: *Prefer embedding for pure functions*.  To give an extreme example, you wouldn't
expose something like `String.length` as a REST API.  In other cases, prefer APIs.

ML models (e.g., Tensorflow) are like pure functions; a model gives an output or a prediction for an input, with
no side effects.  So, if you have a data pipeline applying a model repeatedly to lots of data, bring the
model to the data.  Don't invoke the model as an API call for each individual call to the model.

The harder scenario is one that could go either way: a pure function, but with moderately complex logic and need to support
multiple technologies, such that maintaining multiple libraries (a C# version and Java version for example) is too
difficult.  An API might be a better solution in this case, but it's nice to make libraries available for situations
where latency is an issue as well.
