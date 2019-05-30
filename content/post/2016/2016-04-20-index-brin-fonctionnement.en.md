---
title: BRIN Indexes – Operation
author: Adrien Nayrat
type: post
date: 2016-04-20T06:00:40+00:00
draft: false
categories:
  - Linux
  - Postgres
tags:
  - brin
  - index
  - postgres

---
PostgreSQL 9.5 released in january 2016 brings a new kind of index: BRIN indexes for Bloc Range INdex. They are recommanded when tables are very big and correlated with their physical location. I decided to devote a series of articles on these indexes:


  * [BRIN Indexes - Overview][1]
  * [BRIN Indexes - Operation][2]
  * [BRIN Indexes - Correlation][3]
  * [BRIN Indexes - Performances][4]

In this second article we will see how BRIN indexes works.

<!--more-->

# Operation

Index will contain the minimal and maximal value [^7] of an attribute for a range of blocks.

Range's size is 128 blocks (128 x 8KB => 1MB)


Take an example, a 100 000 lines table composed by a column "id" incremented by 1. Basically, the table will be stored like this (one bloc can contain several tuples) :

| Bloc 	| id    	|
|------	|-------	|
| 0    	| 1     	|
| 0    	| 2     	|
| …    	| …     	|
| 1    	| 227   	|
| …    	| …     	|
| 128  	| 28929 	|
| …    	| …     	|
| 255  	| 57856 	|
| 256  	| 57857 	|
| …    	| …     	|

```SQL
CREATE TABLE t1 (c1) AS (SELECT * FROM generate_series(1,100000));
SELECT ctid,c1 from t1;
```

If we search for value between 28929 et 57856, PostgreSQL will have to read the whole table. It do not know it is not necessary to read before block 128 and after block 255.

First reaction will be to create a B-tree index. Without going into details, this kind of index allows a more efficient read of the table. It will contain each location of each value of the table.

Basically, omitting tree view, the index would contain:

| id    	| Location 	|
|-------	|----------	|
| 1     	| 0        	|
| 2     	| 0        	|
| …     	|          	|
| 227   	| 1        	|
| …     	| …        	|
| 57857 	| 256      	|

Intuitively we can already deduce that if our table is large, the index will also be large.

BRIN index will contain theses rows:

| Range (128 blocks) 	| min   	| max    	| allnulls 	| hasnulls 	|
|-------------------	|-------	|--------	|----------	|----------	|
| 1                 	| 1     	| 28928  	| false   	| false    	|
| 2                 	| 28929 	| 57856  	| false    	| false    	|
| 3                 	| 57857 	| 86784  	| false    	| false    	|
| 4                 	| 86785 	| 115712 	| false    	| false    	|

```SQL
create index ON t1 using brin(c1) ;
create extension pageinspect;
SELECT * FROM brin_page_items(get_raw_page('t1_c1_idx', 2), 't1_c1_idx');
```

So looking for the values ​​between 28929 and 57856 Postgres knows that it will have to go through blocks 128 to 255.

Comparing to a B-tree, we could represent in 4 rows what would have taken more than 100 000 rows in a B-tree. Or course, reality is much more complex, however, this simplification already provides an overview of the compactness of this index.

## Influence of the size of the range

Default range size is 128 blocks, which corresponds  to 1MB (1 block is 8KB). We will test with differents range size thanks to the _pages\_per\_range_ option.



Let's take a bigger data set, with 100 million lines:

```SQL
CREATE TABLE brin_large (c1 INT);
INSERT INTO brin_large SELECT * FROM generate_series(1,100000000);
```

Table size is arround 3.5GB:


```SQL
\dt+ brin_large
                       List OF relations
 Schema |    Name    | TYPE  |  Owner   |  SIZE   | Description
--------+------------+-------+----------+---------+-------------
 public | brin_large | TABLE | postgres | 3458 MB |
```

Enable psql "\timing" option to measure index creation time. Let's start with BRIN with a different _pages\_per\_range_:

```SQL
CREATE INDEX brin_large_brin_idx ON brin_large USING brin (c1);
CREATE INDEX brin_large_brin_idx_8 ON brin_large USING brin (c1) WITH (pages_per_range = 8);
CREATE INDEX brin_large_brin_idx_16 ON brin_large USING brin (c1) WITH (pages_per_range = 16);
CREATE INDEX brin_large_brin_idx_32 ON brin_large USING brin (c1) WITH (pages_per_range = 32);
CREATE INDEX brin_large_brin_idx_64 ON brin_large USING brin (c1) WITH (pages_per_range = 64);
```

Also create a b-tree:

```SQL
CREATE INDEX brin_large_btree_idx ON brin_large USING btree (c1);
```

Compare size:


```SQL
\di+ brin_large*
                                    List OF relations
 Schema |          Name          | TYPE  |  Owner   |   TABLE    |  SIZE   | Description
--------+------------------------+-------+----------+------------+---------+-------------
 public | brin_large_brin_idx    | INDEX | postgres | brin_large | 128 kB  |
 public | brin_large_brin_idx_16 | INDEX | postgres | brin_large | 744 kB  |
 public | brin_large_brin_idx_32 | INDEX | postgres | brin_large | 392 kB  |
 public | brin_large_brin_idx_64 | INDEX | postgres | brin_large | 216 kB  |
 public | brin_large_brin_idx_8  | INDEX | postgres | brin_large | 1448 kB |
 public | brin_large_btree_idx   | INDEX | postgres | brin_large | 2142 MB |
```

Here are the results on the  index creation times and their sizes:

![Duration](/img/2016/duration.png)

Index creation is faster. The gain is arround x10 between two indexes type. Tested with a pretty basic configuration (laptop, mechanical HDD).  I advise you to conduct your own tests with your material.

FYI I used a maintenance\_work\_mem at 1GB. Despite this high value, sort did not fit in RAM so it generates temps files for B-tree index.

![size](/img/2016/size.png)

For index size, difference is much more important, I used a logarithmic scale to represent gap. BRIN with a default _pages\_per\_range_ is 128KB while B-tree is more than 2GB!

What about queries?


Let's try this query which retrieves the values between 10 and 2000. To study in detail what Postgres does we will use EXPLAIN with the options (ANALYZE, VERBOSE, BUFFERS).

Finally, to facilitate the analysis we will use a smaller data set:

```SQL
CREATE TABLE brin_demo (c1 INT);
INSERT INTO brin_demo SELECT * FROM generate_series(1,100000);
```

```SQL
EXPLAIN (ANALYZE,BUFFERS,VERBOSE) SELECT c1 FROM brin_demo WHERE c1> 1 AND c1<2000;
 QUERY PLAN
-------------------------------------------------------------------------------------------------------------------
 Seq Scan on public.brin_demo (cost=0.00..2137.47 rows=565 width=4) (actual time=0.010..11.311 rows=1998 loops=1)
 Output: c1
 Filter: ((brin_demo.c1 > 1) AND (brin_demo.c1 < 2000))
 Rows Removed by Filter: 98002
 Buffers: shared hit=443
 Planning time: 0.044 ms
 Execution time: 11.412 ms
 ```

 Without index Postgres read the entire table (seq scan) and reads 443 blocks.

 The same query with a BRIN index:

```SQL
CREATE INDEX brin_demo_brin_idx ON brin_demo USING brin (c1);
EXPLAIN (ANALYZE,BUFFERS,VERBOSE) SELECT c1 FROM brin_demo WHERE c1> 1 AND c1<2000;
 QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on public.brin_demo (cost=17.12..488.71 rows=500 width=4) (actual time=0.034..3.483 rows=1998 loops=1)
 Output: c1
 Recheck Cond: ((brin_demo.c1 > 1) AND (brin_demo.c1 < 2000))
 Rows Removed by Index Recheck: 26930
 Heap Blocks: lossy=128
 Buffers: shared hit=130
 -> Bitmap Index Scan on brin_demo_brin_idx (cost=0.00..17.00 rows=500 width=0) (actual time=0.022..0.022 rows=1280 loops=1)
 Index Cond: ((brin_demo.c1 > 1) AND (brin_demo.c1 < 2000))
 Buffers: shared hit=2
 Planning time: 0.074 ms
 Execution time: 3.623 ms
 ```

Postgres reads 2 index blocks then 128 blocks from table.


Let's try with a smaller pages\_per\_range, for example:

```SQL
CREATE INDEX brin_demo_brin_idx_16 ON brin_demo USING brin (c1) WITH (pages_per_range = 16);
EXPLAIN (ANALYZE,BUFFERS,VERBOSE) SELECT c1 FROM brin_demo WHERE c1> 10 AND c1<2000;
 QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on public.brin_demo (cost=17.12..488.71 rows=500 width=4) (actual time=0.053..0.727 rows=1989 loops=1)
 Output: c1
 Recheck Cond: ((brin_demo.c1 > 10) AND (brin_demo.c1 < 2000))
 Rows Removed by Index Recheck: 1627
 Heap Blocks: lossy=16
 Buffers: shared hit=18
 -> Bitmap Index Scan on brin_demo_brin_idx_16 (cost=0.00..17.00 rows=500 width=0) (actual time=0.033..0.033 rows=160 loops=1)
 Index Cond: ((brin_demo.c1 > 10) AND (brin_demo.c1 < 2000))
 Buffers: shared hit=2
 Planning time: 0.114 ms
 Execution time: 0.852 ms
 ```

Again Postgres reads 2 blocks from the index, however it will only read 16 blocks from the table.

An index with a smaller pages\_per\_range will be more selective and will allow you to read fewer blocks.

Use the pageinspect extension to observe the contents of the indexes:

```SQL
CREATE extension pageinspect;
SELECT * FROM brin_page_items(get_raw_page('brin_demo_brin_idx_16', 2),'brin_demo_brin_idx_16');
 itemoffset | blknum | attnum | allnulls | hasnulls | placeholder | value
------------+--------+--------+----------+----------+-------------+-------------------
 1 | 0 | 1 | f | f | f | {1 .. 3616}
 2 | 16 | 1 | f | f | f | {3617 .. 7232}
 3 | 32 | 1 | f | f | f | {7233 .. 10848}
 4 | 48 | 1 | f | f | f | {10849 .. 14464}
 5 | 64 | 1 | f | f | f | {14465 .. 18080}
 6 | 80 | 1 | f | f | f | {18081 .. 21696}
 7 | 96 | 1 | f | f | f | {21697 .. 25312}
 8 | 112 | 1 | f | f | f | {25313 .. 28928}
 9 | 128 | 1 | f | f | f | {28929 .. 32544}
 10 | 144 | 1 | f | f | f | {32545 .. 36160}
 11 | 160 | 1 | f | f | f | {36161 .. 39776}
 12 | 176 | 1 | f | f | f | {39777 .. 43392}
 13 | 192 | 1 | f | f | f | {43393 .. 47008}
 14 | 208 | 1 | f | f | f | {47009 .. 50624}
 15 | 224 | 1 | f | f | f | {50625 .. 54240}
 16 | 240 | 1 | f | f | f | {54241 .. 57856}
 17 | 256 | 1 | f | f | f | {57857 .. 61472}
 18 | 272 | 1 | f | f | f | {61473 .. 65088}
 19 | 288 | 1 | f | f | f | {65089 .. 68704}
 20 | 304 | 1 | f | f | f | {68705 .. 72320}
 21 | 320 | 1 | f | f | f | {72321 .. 75936}
 22 | 336 | 1 | f | f | f | {75937 .. 79552}
 23 | 352 | 1 | f | f | f | {79553 .. 83168}
 24 | 368 | 1 | f | f | f | {83169 .. 86784}
 25 | 384 | 1 | f | f | f | {86785 .. 90400}
 26 | 400 | 1 | f | f | f | {90401 .. 94016}
 27 | 416 | 1 | f | f | f | {94017 .. 97632}
 28 | 432 | 1 | f | f | f | {97633 .. 100000}
(28 lignes)

SELECT * FROM brin_page_items(get_raw_page('brin_demo_brin_idx', 2),'brin_demo_brin_idx');
 itemoffset | blknum | attnum | allnulls | hasnulls | placeholder | value
------------+--------+--------+----------+----------+-------------+-------------------
 1 | 0 | 1 | f | f | f | {1 .. 28928}
 2 | 128 | 1 | f | f | f | {28929 .. 57856}
 3 | 256 | 1 | f | f | f | {57857 .. 86784}
 4 | 384 | 1 | f | f | f | {86785 .. 100000}
(4 lignes)
```


With the index brin\_demo\_brin\_idx\_16 we note that the values we are interested in are present in the first set of blocks (0 to 15). On the other hand, with the index brin\_demo\_brin_idx, this one is less selective. The values that interest us are included in blocks 0 to 127 which explains why there are more blocks read in the first example.

[1]: https://blog.anayrat.info/en/2016/04/19/brin-indexes-overview/
[2]: https://blog.anayrat.info/en/2016/04/20/brin-indexes-operation/
[3]: https://blog.anayrat.info/en/2016/04/20/brin-indexes-correlation/
[4]: https://blog.anayrat.info/en/2016/04/21/brin-indexes-performances/
[5]: http://www.pgday.fr/index.html
[6]: http://www.pgday.fr/programme.html
[^7]: Index also contains two Booleans that indicate whether the set contains nulls (hasnulls) or contains *only* nulls (allnulls)
