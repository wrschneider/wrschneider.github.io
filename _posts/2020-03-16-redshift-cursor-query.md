---
layout: post
title: See Redshift queries behind cursor fetch
description: Redshift queries if using UseDeclareFetch will not show up in STL_QUERY
---

By default, the Redshift ODBC/JDBC drivers will fetch all result rows from a query.
If your result sets are large, you may have ended up using the `UseDeclareFetch` and
`Fetch` parameters.  But if you do this, you won't see your actual queries in
the `STL_QUERY` table or Redshift console.  Instead you will see that the
actual long-running query looks like

```sql
fetch xxx in "SQL_CUR4";
```

So you will have to do some extra work to see where the actual query came
from.

The underlying SQL query will actually be in `STL_UTILITYTEXT`, to open the
cursor.  You can get the SQL like this:

```sql
select ut.* from stl_utilitytext ut
  join stl_query q on q.xid = ut.xid and q.pid = ut.pid
 where query=? and text not like 'begin;%' and text not like 'close "%'
 order by sequence;
 ```

 It will take some editing to reformat the SQL for copy-paste.

 Note that many of the online forums mention `STV_ACTIVE_CURSORS` but this is not
 that helpful if you are looking for queries that have already completed.
