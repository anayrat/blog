---
title: BRIN Indexes – Overview
authors: ['adrien']
type: post
date: 2016-04-19T06:00:21+00:00
draft: false
categories:
  - Linux
  - Postgres
tags:
  - brin
  - index
  - postgres
show_related: true

---
PostgreSQL 9.5 released in january 2016 brings a new kind of index: BRIN indexes for Bloc Range INdex. They are recommanded when tables are very big and correlated with their physical location. I decided to devote a series of articles on these indexes:


  * [BRIN Indexes - Overview][1]
  * [BRIN Indexes - Operation][2]
  * [BRIN Indexes - Correlation][3]
  * [BRIN Indexes - Performances][4]

<!--more-->

# Introduction

PostgreSQL provides several kind of access method: B-Tree, GIN, GiST, SP-GiST, Hash [^8]

Discissions started in 2008: [Segment Exclusion](http://www.postgresql.org/message-id/1199296574.7260.149.camel@ebony.site)

Followed by this RFC which suggest "[Minmax indexes](https://www.postgresql.org/message-id/20130614222805.GZ5491@eldon.alvh.no-ip.org)" renamed later with BRIN.

At physical level, a table is composed of 8Ko blocks which contains tuples[^9]. BRIN idea is to store the maximal and minimal value of an attribute from a bloc range[^10]. Then, it is possible to exclude a range of blocks if you  search a value which is not stored in the range.



Note : BRIN indexes are similar to _storage indexes_  in Oracle ([Exadata Storage Indexes][7]).

Here are supported types: [Built-in Operator Classes](http://www.postgresql.org/docs/current/static/brin-builtin-opclasses.html#BRIN-BUILTIN-OPCLASSES-TABLE)

```
abstime
bigint
bit
bit varying
box
bytea
character
"char"
date
double precision
inet
inet
integer
interval
macaddr
name
numeric
pg_lsn
oid
any range type
real
reltime
smallint
text
tid
timestamp without time zone
timestamp with time zone
time without time zone
time with time zone
uuid
```


 [1]: https://blog.anayrat.info/en/2016/04/19/brin-indexes-overview/
 [2]: https://blog.anayrat.info/en/2016/04/20/brin-indexes-operation/
 [3]: https://blog.anayrat.info/en/2016/04/20/brin-indexes-correlation/
 [4]: https://blog.anayrat.info/en/2016/04/21/brin-indexes-performances/
 [7]: https://en.wikipedia.org/wiki/Block_Range_Index#Exadata_Storage_Indexe

[^8]: `SELECT amname FROM pg_am;` or <http://www.postgresql.org/docs/current/static/indexes-types.html>
[^9]: If a tuple do not fit in a bloc, it is stored separately in TOAST (The Oversized-Attribute Storage Technique). More precisely, if a tuple is more than 2Ko.
[^10]: The index also contains two boolean which indicate if the range contains NULL values (hasnulls) or if it contains **only** NULL values (allnulls).
