---
layout: post
title: Migrating DATE columns from Oracle to Redshift
---

Oracle's `DATE` type is both a date and a time down to the second level.

Redshift gives you a choice between a true `DATE` that is only a date, with no
time, and a `TIMESTAMP` that goes down to a microsecond.

When migrating an Oracle database to Redshift, you can simply convert all the
`DATE` columns to `TIMESTAMP`.  (Keeping them as `DATE` columns will syntactically
work, but possibly lose information.)  For the Oracle columns which are truly
dates, a `TIMESTAMP` column would be wider than they need to be, which
breaks Redshift best practices.

So it's good to introspect the actual column data and determine which columns
should remain `DATE` and which should be replaced with `TIMESTAMP`.  (It's
better to design your Oracle schema to limit `DATE` types to columns
where time doesn't matter, to avoid this issue in the first place.)

I wrote a script in PL/SQL to introspect data to determine which `DATE`
columns are actually datetimes / timestamps and which are really dates.
You could use this to improve your Oracle schema even if you're not migrating
to Redshift, to change `DATE` columns to `TIMESTAMP` where you care about time.
That would make the schema more self-descriptive - a column like
`event_dttm DATE` should be an anti-pattern!

There are at least two things that I don't like about this script and I would do
better if I were spending more time on it -- see if you can spot them:

```sql
declare
    cursor c is (
        select * from all_tab_columns where owner = 'YOUR_SCHEMA' and data_type='DATE'
    );
    x int;
begin
    for r in c
    loop
        execute immediate
            'select case when exists ' ||
            '   (select 1 from ' ||  r.owner || '.' || r.table_name ||
            '       where ' || r.column_name || ' <> trunc(' || r.column_name || ')' ||
            '   ) then 1 else 0 end as foo from dual'
            into x;
        if x > 0 then
            dbms_output.put_line(r.table_name || '.' || r.column_name || ' should be timestamp');
        else
            dbms_output.put_line(r.table_name || '.' || r.column_name || ' is OK as a date');
        end if;
    end loop;
end;
/
```

What I would do better next time

* Be more intelligent about inferring intent; even if `col <> trunc(col)`
for a given date column, maybe all the time components are still identical.
(Perhaps due to migrating from another environment in a different timezone)

* group columns together to scan each table only once, if there are multiple
date columns on the same table

* make this a table-valued function so you can `SELECT` the output as discrete
fields rather than parsing it from `dbms_output` (that would also make it
reusable and un-hard-code the schema name)

* do it in Spark instead of PL/SQL, so we can do parallel scans on each table
and generate dynamic SQL more cleanly; although that might actually perform
worse when taking into account pulling the data over the wire and needing
a Spark cluster to run it on (or spilling to disk if running locally and/or
without enough memory)
