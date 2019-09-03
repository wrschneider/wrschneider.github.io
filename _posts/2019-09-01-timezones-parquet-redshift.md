---
layout: post
title: Timestamp and timezone confusion with Spark, Parquet and Redshift
description: moving to big data technologies (Spark, Parquet, Redshift, AWS) can introduce discrepancies in how times are represented
---

Most of the time developers don't give much thought to timezones.  In a data-center based application, ETL, DB, and applications
all run in the same timezone, so there are limited opportunities for discrepancies.

When you move to the cloud and big data, especially when you migrate components incrementally, there are suddenly lots of
places where things can go wrong.

Here's an example of an issue I ran into when migrating a front-end reporting application to AWS and Redshift incrementally.

- A Spark data pipeline runs in existing data center, and generates Parquet
- Parquet is copied to AWS S3
- Parquet is then loaded to Redshift via `COPY`
- *Problem*: some dates in the application are now off by a day, compared with Parquet imported into a legacy DB via JDBC

Digging deeper it turns out the problem is something like this:
- The original source of truth is a flat file with date-time strings with no particular timezone, like "2019-01-01 17:00".  That's
5 o'clock *somewhere* (sorry, couldn't resist) but the instant in time is ambiguous.  Often times we don't really
care about the moment in time, as long as we can print back the same string we started with, so local time semantics makes sense.

- Spark parses that flat file into a `DataFrame`, and the time becomes a timestamp field.  But a timestamp field is like
a UNIX timestamp and has to represent a single moment in time.  So Spark interprets the text in the current JVM's timezone
context, which is Eastern time in this case.  So the "17:00" in the string is interpreted as 17:00 EST/EDT.  That DataFrame is then
written to Parquet.    

- Redshift loads the timestamp from Parquet file into a `TIMESTAMP` column.  A `TIMESTAMP` is like a date-time string, in
that it has no timezone info and does not correspond to a specific moment in time.  The problem here is that Redshift `COPY`
interprets timestamps in Parquet as literal moments in time, and then "formats" the value into the `TIMESTAMP` column in UTC.  

So what happens here is "17:00" in the original text file becomes "22:00" in Redshift.  This is an even bigger problem
if the time was close to midnight; 23:00 becomes 04:00 *the next day* and so `trunc(datetime)` or aggregates that group by
date, month or quarter won't match either.

Before proposing a solution or workaround, let's take a step back and try to understand root cause here.

The key here is to recognize that specific moments in time ("instants"), and a string like "17:00" ("local time")
are *two distinct concepts*.  They can be converted: instant = local time + timezone.  But often, what happens is
a local time is converted to an instant, by implicitly adding the current system timezone.

Some examples:

||Instant|local time|
|-----|------|------|
|Literal|2019-01-01 17:00-05:00 (ISO with offset)<br/>1546380000 (UNIX timestamp)|2019-01-01 17:00 (no timezone)|
|SQL type|TIMESTAMP WITH TIMEZONE|TIMESTAMP [WITHOUT TIMEZONE]<br/>|
|Java type|java.sql.Timestamp|java.time.LocalDateTime|
|[Parquet type](https://github.com/apache/parquet-format/blob/master/LogicalTypes.md)|Timestamp with isAdjustedToUTC=true|Timestamp with isAdjustedToUTC=false|

The challenge is between Spark and Redshift:

- Redshift `COPY` from Parquet into `TIMESTAMP` columns treats timestamps in Parquet as if they were UTC, even if they
are intended to represent local times.  So if you want to see the value "17:00" in a Redshift `TIMESTAMP` column,
you need to load it with 17:00 *UTC* from Parquet.  Technically, [according to Parquet documentation](https://github.com/apache/parquet-format/blob/master/LogicalTypes.md), this is correct: the integer value
to represent 17:00 as a local time is equivalent to the value that would represent the instant 17:00 UTC.

- Spark does not support a distinction between local times and instants in DataFrames.  So we have no way to parse a time
from a CSV without implicitly converting it to an instant, using
the current Spark session timezone.  Spark will then generate Parquet with either `INT96` or `TIME_MILLIS` Parquet types, both
of which assume UTC normalization (instant semantics).

The only workaround I could come up with was forcibly converting the instant in the DataFrame parsed in the current
Spark timezone to the same local time in UTC, i.e., converting 17:00 EST to 17:00 UTC.  We can do this with the Spark helper
`from_utc_timestamp`, which has the net effect of adding the specified timezone offset to the input.  

But ideally, I wouldn't have to do this, because Spark should be able to natively support local time semantics explicitly.
