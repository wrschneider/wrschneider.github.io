---
layout: post
title: Node.js with SQL Server
---

Yes, you can use SQL Server with Node.js.  I added a [Gist](https://gist.github.com/wrschneider/edaa6cbb4ff39d4bcfa8.js) to show how it could work.  I didn't 
get to connection pooling or transactions, just enough to get a general feel.

I'm not convinced that SQL (or an RDBMS in general) is the best persistent store for Node. 

Some specific observations:

* **Support for Node with SQL Server seems immature and fragmented.**  There are many different drivers to choose from, no ADO.NET- or JDBC-like standard 
interface to wrap all those drivers.  So if you run into issues with a driver and need to change to a different one, you may be rewriting code. I ended up picking 
[node-sqlserver-unofficial](https://www.npmjs.com/package/node-sqlserver-unofficial), which is a precompiled binary version of Microsoft's 
[msnodesql](https://www.npmjs.com/package/msnodesql) driver, to get support for Windows integrated/trusted authentication, i.e., single sign-on to DB without a username and 
password in connect string.  Yet, Microsoft's says their driver is "not production ready" and doesn't look like it can be cleanly installed via npm.  

[Tedious](http://pekim.github.io/tedious/) looks like a better solution if you can live without Windows integrated authentication.  It is pure Javascript and appears to be more stable 
and widely supported.  

* **Async callback hell.**  Sequential operations, like inserting a set of rows in order, become painful when each individual operation is 
asynchronous.  I tamed this by replacing callbacks with promises (`Promise.denodify` is helpful), but it's still harder to read and single-step than a corresponding synchronous
version.  This is a bigger issue with an RDBMS than with a document store like MongoDB or a key-value store like Redis – in a relational model, an object that contains an array turns 
into multiple rows that need to be inserted in order (insert parent first, then child), while in a document or key-value store, the same object would be persisted in a single operation.  

With these observations in mind, it seems like SQL Server isn't the best option.  It *can* work, especially if you can use Tedious and live without integrated 
authentication.  PostgreSQL support seems more promising and production-stable, plus it has native JSON support.  With native JSON, you can slam a composite object into a single column
and query its components, without having to deal with parent-child relationships.