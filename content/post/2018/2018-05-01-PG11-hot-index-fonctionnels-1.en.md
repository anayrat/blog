+++
title = "PostgreSQL and heap-only-tuples updates - part 1"
date = 2018-11-12T08:00:00+01:00
draft = false
summary = "How MVCC works and *heap-only-tuples* updates"

# Tags and categories
# For example, use `tags = []` for no tags, or the form `tags = ["A Tag", "Another Tag"]` for one or more tags.
tags = ["postgres","index","heap-only-tuple"]
categories = ["Postgres"]

# Featured image
# Place your image in the `static/img/` folder and reference its filename below, e.g. `image = "example.jpg"`.
# Use `caption` to display an image caption.
#   Markdown linking is allowed, e.g. `caption = "[Image credit](http://example.org)"`.
# SET `preview` to `false` to disable the thumbnail in listings.
[header]
image = ""
caption = ""
preview = true

+++

Here is a series of articles that will focus on a new feature in version 11.

During the development of this version, a feature caught my attention.
It can be found in releases notes : <https://www.postgresql.org/docs/11/static/release-11.html>


> Allow heap-only-tuple (HOT) updates for expression indexes when the values of the expressions are unchanged (Konstantin Knizhnik)

I admit that this is not very explicit and this feature requires some
knowledge about how postgres works, that I will try to explain through several articles:

1. [How MVCC works and *heap-only-tuples* updates][1]
2. [When postgres do not use *heap-only-tuple* updates and introduction to the new feature in v11][2]
3. Impact on performances

**This feature was disabled in 11.1 because it could lead to instance crashes[^4].
I chose to publish these articles because they help to understand the mechanism
of HOT updates and the benefits that this feature could bring.**

I thank Guillaume Lelarge for his review of this article ;).


# How MVCC works

Due to [MVCC](https://en.wikipedia.org/wiki/Multiversion_concurrency_control),
postgres does not update a line directly: it duplicates
the line and provides visibility information.

Why does it work like this?

There is a critical component to consider when working with a
RDBMS: concurrency.

The line you are editing may be used by a previous transaction.
An ongoing backup, for example :)

To this end, RDBMS have adopted different techniques:

  * Modify the line and store previous versions on another
  location. This is what oracle does with undo logs, for example.
  * Duplicate the line and store visibility information to
  know which line is visible by which transaction. This requires
  to have a cleaning mechanism for lines that are no longer visible to
  no one. This is the implementation in Postgres and the vacuum is responsible
  to perform this cleaning.


Let's take a very simple table and watch its content evolve using the pageinspect extension:

```sql
CREATE TABLE t2(c1 int);
INSERT INTO t2 VALUES (1);
SELECT lp,t_data FROM  heap_page_items(get_raw_page('t2',0));
 lp |   t_data   
----+------------
  1 | \x01000000
(1 row)

UPDATE t2 SET c1 = 2 WHERE c1 = 1;
SELECT lp,t_data FROM  heap_page_items(get_raw_page('t2',0));
 lp |   t_data   
----+------------
  1 | \x01000000
  2 | \x02000000
(2 rows)

VACUUM t2;
SELECT lp,t_data FROM  heap_page_items(get_raw_page('t2',0));
 lp |   t_data   
----+------------
  1 |
  2 | \x02000000
(2 rows)
```

We can see that the engine duplicated the line and the vacuum cleaned
location for future use.

# heap-only-tuple mechanism

Let's take another case, a little more complicated, a table with two columns and
an index on one of the two columns:

```sql
CREATE TABLE t3(c1 int,c2 int);
CREATE INDEX ON t3(c1);
INSERT INTO t3(c1,c2) VALUES (1,1);
INSERT INTO t3(c1,c2) VALUES (2,2);
SELECT ctid,* FROM t3;
 ctid  | c1 | c2
-------+----+----
 (0,1) |  1 |  1
 (0,2) |  2 |  2
(2 rows)
```

In the same way it is possible to read table blocks, it is possible
to read blocks with pageinspect:


```sql
SELECT * FROM  bt_page_items(get_raw_page('t3_c1_idx',1));
 itemoffset | ctid  | itemlen | nulls | vars |          data           
------------+-------+---------+-------+------+-------------------------
          1 | (0,1) |      16 | f     | f    | 01 00 00 00 00 00 00 00
          2 | (0,2) |      16 | f     | f    | 02 00 00 00 00 00 00 00
(2 rows)
```

So far, it's pretty simple, my table contains two records and the index also
contains two records pointing to the corresponding blocks in the table (ctid column).

If I update column c1, with the new value 3 for example, the index will have to be updated.

Now, if I update the column c2. Will the index on c1 be updated?

At first glance, we might say no because c1 is unchanged.

But because of the MVCC model presented above, in theory, the answer will be yes:
we have just seen that postgres will duplicate the line, so its physical location
will be different (the next ctid will be (0.3)).

Let's check it out:

```sql
UPDATE t3 SET c2 = 3 WHERE c1=1;
SELECT lp,t_data,t_ctid FROM  heap_page_items(get_raw_page('t3',0));
 lp |       t_data       | t_ctid
----+--------------------+--------
  1 | \x0100000001000000 | (0,3)
  2 | \x0200000002000000 | (0,2)
  3 | \x0100000003000000 | (0,3)
(3 rows)


SELECT * FROM  bt_page_items(get_raw_page('t3_c1_idx',1));
 itemoffset | ctid  | itemlen | nulls | vars |          data           
------------+-------+---------+-------+------+-------------------------
          1 | (0,1) |      16 | f     | f    | 01 00 00 00 00 00 00 00
          2 | (0,2) |      16 | f     | f    | 02 00 00 00 00 00 00 00
(2 rows)
```

Reading the table block confirms that the line has been duplicated.
By looking carefully at the field `t_data`, we can distinguish the 1 from column
c1 and the 3 from column c2.

If you read index block, you can see that its content has not changed! If I search
for line `WHERE c1 = 1`, the index points me towards the record (0,1) which
corresponds to the old line!

What happened?

In fact, we have just revealed a rather special mechanism called *heap-only-tuple*
alias HOT. When a column is updated, no index points to that column and the
record can be inserted in the same block, Postgres will only make a pointer
between the old and the new record.

This allows postgres to avoid having to update the index. With all that implies:

  * Read/write operations avoided
  * Reduce index fragmentation and therefore of its size (it is
    difficult to reuse old index locations)

If you look at table block, the column `t_ctid` of the first line points to (0.3).
If the line was updated again, the first line of the table would point to the
line (0.3) and the line (0.3) would point to (0.4), forming what is called a chain.
A vacuum would clean up the free spaces but would always keep the first line
that would point to the last recording.


A line is changed and the index still does not change:

```sql
UPDATE t3 SET c2 = 4 WHERE c1=1;
SELECT * FROM  bt_page_items(get_raw_page('t3_c1_idx',1));
 itemoffset | ctid  | itemlen | nulls | vars |          data           
------------+-------+---------+-------+------+-------------------------
          1 | (0,1) |      16 | f     | f    | 01 00 00 00 00 00 00 00
          2 | (0,2) |      16 | f     | f    | 02 00 00 00 00 00 00 00
(2 rows)

SELECT lp,t_data,t_ctid FROM  heap_page_items(get_raw_page('t3',0));
 lp |       t_data       | t_ctid
----+--------------------+--------
  1 | \x0100000001000000 | (0,3)
  2 | \x0200000002000000 | (0,2)
  3 | \x0100000003000000 | (0,4)
  4 | \x0100000004000000 | (0,4)
(4 rows)
```

Vacuum cleans available spaces:

```sql
VACUUM t3;
SELECT lp,t_data,t_ctid FROM  heap_page_items(get_raw_page('t3',0));
 lp |       t_data       | t_ctid
----+--------------------+--------
  1 |                    |
  2 | \x0200000002000000 | (0,2)
  3 |                    |
  4 | \x0100000004000000 | (0,4)
(4 rows)
```

An update will reuse the second location and the index remains unchanged.
Look at the value of the column `t_ctid` to reconstruct the chain:

```sql
UPDATE t3 SET c2 = 5 WHERE c1=1;
SELECT lp,t_data,t_ctid FROM  heap_page_items(get_raw_page('t3',0));
 lp |       t_data       | t_ctid
----+--------------------+--------
  1 |                    |
  2 | \x0200000002000000 | (0,2)
  3 | \x0100000005000000 | (0,3)
  4 | \x0100000004000000 | (0,3)
(4 rows)


SELECT * FROM  bt_page_items(get_raw_page('t3_c1_idx',1));
 itemoffset | ctid  | itemlen | nulls | vars |          data           
------------+-------+---------+-------+------+-------------------------
          1 | (0,1) |      16 | f     | f    | 01 00 00 00 00 00 00 00
          2 | (0,2) |      16 | f     | f    | 02 00 00 00 00 00 00 00
(2 rows)
```

Uh, the first line is empty and postgres reused the third location?

In fact, an information does not appear in pageinspect. Let's go read the block directly with
[pg_filedump](https://wiki.postgresql.org/wiki/Pg_filedump) :

Note: You must first request a `CHECKPOINT` otherwise the block may not yet be
written to disk.

```
pg_filedump  11/main/base/16606/8890510

*******************************************************************
* PostgreSQL File/Block Formatted Dump Utility - Version 10.1
*
* File: 11/main/base/16606/8890510
* Options used: None
*
* Dump created on: Sun Sep  2 13:09:53 2018
*******************************************************************

Block    0 ********************************************************
<Header> -----
 Block Offset: 0x00000000         Offsets: Lower      40 (0x0028)
 Block: Size 8192  Version    4            Upper    8096 (0x1fa0)
 LSN:  logid     52 recoff 0xc39ea148      Special  8192 (0x2000)
 Items:    4                      Free Space: 8056
 Checksum: 0x0000  Prune XID: 0x0000168b  Flags: 0x0001 (HAS_FREE_LINES)
 Length (including item array): 40

<Data> ------
 Item   1 -- Length:    0  Offset:    4 (0x0004)  Flags: REDIRECT
 Item   2 -- Length:   32  Offset: 8160 (0x1fe0)  Flags: NORMAL
 Item   3 -- Length:   32  Offset: 8096 (0x1fa0)  Flags: NORMAL
 Item   4 -- Length:   32  Offset: 8128 (0x1fc0)  Flags: NORMAL
```

The first line contains `Flags: REDIRECT`. This indicates that this line corresponds
to a HOT redirection. This is documented in `src/include/storage/itemid.h` :

```c
/*
 * lp_flags has these possible states.  An UNUSED line pointer is available     
 * for immediate re-use, the other states are not.                              
 */                                                                             
#define LP_UNUSED       0       /* unused (should always have lp_len=0) */      
#define LP_NORMAL       1       /* used (should always have lp_len>0) */        
#define LP_REDIRECT     2       /* HOT redirect (should have lp_len=0) */       
#define LP_DEAD         3       /* dead, may or may not have storage */   
```

In fact, it is possible to see it with pageinspect by displaying the column `lp_flags`:

```sql
SELECT lp,lp_flags,t_data,t_ctid FROM  heap_page_items(get_raw_page('t3',0));
 lp | lp_flags |       t_data       | t_ctid
----+----------+--------------------+--------
  1 |        2 |                    |
  2 |        1 | \x0200000002000000 | (0,2)
  3 |        1 | \x0100000005000000 | (0,3)
  4 |        1 | \x0100000004000000 | (0,3)
(4 rows)
```

If we do an update again, then a vacuum, followed by a CHECKPOINT to write the block to disk:

```sql
SELECT lp,lp_flags,t_data,t_ctid FROM  heap_page_items(get_raw_page('t3',0));
 lp | lp_flags |       t_data       | t_ctid
----+----------+--------------------+--------
  1 |        2 |                    |
  2 |        1 | \x0200000002000000 | (0,2)
  3 |        0 |                    |
  4 |        0 |                    |
  5 |        1 | \x0100000006000000 | (0,5)
(5 rows)

CHECKPOINT;

pg_filedump  11/main/base/16606/8890510

*******************************************************************
* PostgreSQL File/Block Formatted Dump Utility - Version 10.1
*
* File: 11/main/base/16606/8890510
* Options used: None
*
* Dump created on: Sun Sep  2 13:16:12 2018
*******************************************************************

Block    0 ********************************************************
<Header> -----
 Block Offset: 0x00000000         Offsets: Lower      44 (0x002c)
 Block: Size 8192  Version    4            Upper    8128 (0x1fc0)
 LSN:  logid     52 recoff 0xc39ea308      Special  8192 (0x2000)
 Items:    5                      Free Space: 8084
 Checksum: 0x0000  Prune XID: 0x00000000  Flags: 0x0005 (HAS_FREE_LINES|ALL_VISIBLE)
 Length (including item array): 44

<Data> ------
 Item   1 -- Length:    0  Offset:    5 (0x0005)  Flags: REDIRECT
 Item   2 -- Length:   32  Offset: 8160 (0x1fe0)  Flags: NORMAL
 Item   3 -- Length:    0  Offset:    0 (0x0000)  Flags: UNUSED
 Item   4 -- Length:    0  Offset:    0 (0x0000)  Flags: UNUSED
 Item   5 -- Length:   32  Offset: 8128 (0x1fc0)  Flags: NORMAL


*** End of File Encountered. Last Block Read: 0 ***
```

Postgres has kept the first line (flag `REDIRECT`) and writes a new line at location 5.

However, there are a few cases where postgres cannot use this mechanism:


  * When there is no more space in the block and he has to write another block.
  We can deduce that the fragmentation of the table is useful here to benefit of HOT.
  * When an index exists on the updated column. In this case postgres must update
  the index. Postgres can detect if there has been a change by performing a binary
  comparison between the new value and the previous one [^5].

In the next article we will see an example where postgres cannot use HOT mechanism.
Then, the new feature of version 11 where postgres can use this mechanism.

[1]: https://blog.anayrat.info/en/2018/11/12/postgresql-and-heap-only-tuples-updates-part-1/
[2]: https://blog.anayrat.info/en/2018/11/19/postgresql-and-heap-only-tuples-updates-part-2/
[3]:

[^4]: [Disable recheck_on_update optimization to avoid crashes](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=05f84605dbeb9cf8279a157234b24bbb706c5256)
[^5]: [README.HOT](https://git.postgresql.org/gitweb/?p=postgresql.git;a=blob;f=src/backend/access/heap/README.HOT;h=4cf3c3a0d4c2db96a57e73e46fdd7463db439f79;hb=HEAD#l128)
