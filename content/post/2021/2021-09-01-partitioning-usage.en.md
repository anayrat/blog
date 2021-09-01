+++
title = "Partitioning use cases with PostgreSQL"
date = 2021-09-01T09:00:00+02:00
draft = false

summary = "Different use cases of native partioning under PostgreSQL"

# Tags and categories
# For example, use `tags = []` for no tags, or the form `tags = ["A Tag", "Another Tag"]` for one or more tags.
tags = ["postgres","partitioning"]
categories = ["Postgres"]

+++

After a short break, I'm back to writing technical articles about Postgres. This is also an opportunity for me to announce my change of activity. Since 2021, I'm a freelancer to give companies the opportunity to benefit from my experience on Postgres.

{{% toc %}}

# History of partitioning in PostgreSQL

For a long time, PostgreSQL has made it possible to partition tables by using table inheritance. However, this method was rather difficult to implement: it implied setting up triggers to redirect writes (less efficient than native partitioning), and the planning time could increase significantly if there were more than a hundred partitions...


The native partitioning came with version 10. Since this version, postgres is able (among other things) to send writes to the right partition, perform partition pruning to only read selected partitions, use algorithms exploiting partitioning, etc. It thus offers better performance and easier maintenance.

Since Postgres 10 we can:

  * Partition:
    * By list
    * By hash
    * By intervals
  * Do multi-level partitioning
  * Partition on multiple columns
  * Use primary and foreign keys

All these features are interesting, but we can ask ourselves a very simple question: when to implement partitioning?

I will introduce you to several use cases that I have encountered. But first, here are some common mistakes about partitioning.


# Common mistakes

## "Partitioning is necessary as soon as the size is important"

First, what is a "large" size?

Some will say that it is more than several hundred GB, others more than a terabyte, others still more than a petabyte...

There is no real answer to this question, and generally it will depend on the type of workload: ratio INSERT/UPDATE/DELETE, type of SELECT (OLTP, OLAP...).
It will also depend on the hardware. 10 years ago, when servers only had a few GB of RAM with mechanical disks, it was likely that a database of a few hundred GB would be perceived as a large database.
Now it is not uncommon to see servers with over a terabyte of RAM, NVMe drives.

Thus, a database of a few hundred GB is no longer considered a large database. But rather as a modest database.

A little story, to reassure himself, a customer asked me if Postgres was already used for "large" volumes. We were talking about a 40 GB database on a server with 64 GB of RAM. All reads were done from the cache... :). I was able to reassure him about the size of his database which was relatively modest.

It may be superfluous to partition a database of a few TB as it may be necessary to partition one of a few hundred GB. For example, if the activity is just adding rows to tables and the queries are simple `WHERE column = 4` that return a few rows. A simple Btree will do the job. And if the query returns many rows, it is possible to use BRIN indexes or bloom filters.

> BRIN indexes provide similar benefits to horizontal partitioning or sharding but without needing to explicitly declare partitions.[^1]

## "Partitioning is necessary to spread data over several disks"

The idea would be to create partitions and tablespaces on different disks in order to spread the input/output operations.

For PostgreSQL, a tablespace is nothing more or less than a path to a directory. It is quite possible to manage the storage at the operating system level and to aggregate several disks (in RAID10) for example.
Then, it is just a matter of storing the table on the volume created. Thus, I/O can be spread over a set of disks.

In this case, it is not necessary to implement partitioning. However, we will see a case where it might make sense.

Now we will look at "legitimate" use cases of partitioning.

# Partitioning uses cases

## Partitioning to manage retention

Due to the MVCC model, massive deletion leads to bloat in the tables.

A possible use case is to partition by date. Deleting the old data is the same as dropping a complete partition. The operation will be fast and the tables will not be bloated.



## Partitioning to control index bloat

Adding and modifying data bloats indexes over time. To put it simply, you can't recover the free space in a block until it is empty. Over time, index splits create "empty" space in the index and the only way to recover this space is to rebuild the index.

This is called "bloat". There have been many improvements on the last versions of Postgres:


* Version 12, we can read in [Releases Notes](https://www.postgresql.org/docs/12/release-12.html):
> Improve performance and space utilization of btree indexes with many duplicates (Peter Geoghegan, Heikki Linnakangas)
>
> Previously, duplicate index entries were stored unordered within their duplicate groups. This caused overhead during index inserts, wasted space due to excessive page splits, and it reduced VACUUM's ability to recycle entire pages. Duplicate index entries are now sorted in heap-storage order.


* Version 13, we can read in [Releases Notes](https://www.postgresql.org/docs/13/release-13.html):
> More efficiently store duplicates in B-tree indexes (Anastasia Lubennikova, Peter Geoghegan)
>
> This allows efficient B-tree indexing of low-cardinality columns by storing duplicate keys only once. Users upgrading with pg_upgrade will need to use REINDEX to make an existing index use this feature.


* Version 14, we can read in [Releases Notes](https://www.postgresql.org/docs/14/release-14.html):
> Allow btree index additions to remove expired index entries to prevent page splits (Peter Geoghegan)
>
> This is particularly helpful for reducing index bloat on tables whose indexed columns are frequently updated.

To control the bloat, we could rebuild the index regularly (thanks to `REINDEX CONCURRENTLY` which arrived in version 12). This solution would be cumbersome, because the whole index would have to be rebuilt regularly.

If most of the modifications are made on recent data, for example: log table, customer orders, appointments... We could imagine a partitioning by month. Thus, at the beginning of each month, we start with a "new" table and we can reindex the previous table to remove the bloat.

We can also take advantage of this to make a `CLUSTER` on the table to have a good correlation of the data with the storage.

## Partitioning for low cardinality

Gradually we will see more complicated use cases :)

Let's take an example: an order table with a delivery state, after a few years 99% of the orders are delivered (we hope!) and very few are in the process of payment or delivery.

Let's imagine that we want to retrieve 100 orders in progress. We will create an index on the state and use it to retrieve the records.
If we are a bit clever, we can create a partial index on this particular state. The problem is that this index will bloat quite quickly as the orders are delivered.

In this case we could do a partitioning on the state. Thus, retrieving 100 orders in the process of being delivered is equivalent to reading 100 records from the partition.

## Partitioning to get more accurate statistics

To determine the best execution plan, Postgres makes decisions based on statistics. They are obtained from a sample of the table (the `default_statistic_target` which is 100 by default).

By default, postgres will collect 300 x `default_statistic_target` rows, that is 30 000 rows. With a table of several hundred million rows, this sample is sometimes too small.

We can drastically increase the sample size, but this approach has some drawbacks:

* It increases the planning time
* It makes the `ANALYZE` more heavy.
* Sometimes it is not enough if the data are not well distributed. For example, if you take a few hundred thousand rows from a table with several hundred million rows, you may miss the rows that are in delivery status.

With partitioning, we could have the same sample but per partition, which allows us to increase the accuracy.

This would also be useful when we have correlated data between columns. I will take the example of orders. We have a whole year's worth of orders: all the orders that are more than one month old are delivered, those of the last month are 90% delivered (10% are in progress).

Intuitively, if I look for an order in progress more than 6 months ago, I should not get any result. On the other hand, if I search for orders in progress for the last month, I should get 10% of the table. But postgres doesn't know that, for it, the orders in progress are spread over the whole table.

With a partitioning by date, it can estimate that there are no orders in progress for deliveries of more than one month. This approach is mainly used to reduce an estimation error in an execution plan.

Here is an example with this order table, `orders_p` is the month partitioned version of the `orders` table. The data is identical in both tables.

We can notice that the estimation is much better in the case where the table is partitioned, postgres having statistics per partitions.

{{< highlight sql "linenos=table,hl_lines=3 6 10 19 22 25" >}}
EXPLAIN ANALYZE
SELECT *
FROM orders_p
WHERE
    state = 'pending'
    AND c1 BETWEEN '2021-01-01' AND '2021-01-31';

                                                  QUERY PLAN                                                  
--------------------------------------------------------------------------------------------------------------
 Index Scan using orders_13_state_idx on orders_13  (cost=0.42..4.45 rows=1 width=12) (actual rows=0 loops=1)
   Index Cond: (state = 'pending'::text)
   Filter: ((c1 >= '2021-01-01'::date) AND (c1 <= '2021-01-31'::date))
 Planning Time: 0.120 ms
 Execution Time: 0.059 ms
(5 rows)

EXPLAIN ANALYZE
SELECT *
FROM orders
WHERE
    state = 'pending'
    AND c1 BETWEEN '2021-01-01' AND '2021-01-31';
                                                  QUERY PLAN                                                   
---------------------------------------------------------------------------------------------------------------
 Index Scan using orders_state_idx on orders  (cost=0.44..13168.25 rows=3978 width=12) (actual rows=0 loops=1)
   Index Cond: (state = 'pending'::text)
   Filter: ((c1 >= '2021-01-01'::date) AND (c1 <= '2021-01-31'::date))
   Rows Removed by Filter: 100161
 Planning Time: 0.188 ms
 Execution Time: 141.571 ms
(6 rows)
{{< / highlight >}}

Now let's take the same query over the last month:

{{< highlight sql "linenos=table,hl_lines=3 6 10 19 22 26" >}}
EXPLAIN ANALYZE
SELECT *
FROM orders_p
WHERE
    state = 'pending'
    AND c1 BETWEEN '2021-07-01' AND '2021-07-31';

                                                       QUERY PLAN                                                        
-------------------------------------------------------------------------------------------------------------------------
 Index Scan using orders_19_state_idx on orders_19  (cost=0.43..2417.50 rows=19215 width=12) (actual rows=20931 loops=1)
   Index Cond: (state = 'pending'::text)
   Filter: ((c1 >= '2021-07-01'::date) AND (c1 <= '2021-07-31'::date))
 Planning Time: 0.297 ms
 Execution Time: 32.618 ms
(5 rows)

EXPLAIN ANALYZE
SELECT *
FROM orders
WHERE
    state = 'pending'
    AND c1 BETWEEN '2021-07-01' AND '2021-07-31';

                                                     QUERY PLAN                                                     
--------------------------------------------------------------------------------------------------------------------
 Index Scan using orders_state_idx on orders  (cost=0.44..13168.25 rows=15008 width=12) (actual rows=20931 loops=1)
   Index Cond: (state = 'pending'::text)
   Filter: ((c1 >= '2021-07-01'::date) AND (c1 <= '2021-07-31'::date))
   Rows Removed by Filter: 79230
 Planning Time: 0.173 ms
 Execution Time: 146.326 ms
(6 rows)
{{< / highlight >}}

Here again we can see that the estimation is better.

## partitionwise join & partitionwise aggregate

Another interest of partitioning is to benefit from better algorithms for joins and aggregation.

The `partitionwise aggregate` allows to do an aggregation or a grouping partition per partition. An example is better than a long speech:

{{< highlight sql "linenos=table,hl_lines=4 16 21" >}}
explain (analyze,timing off) select count(*), c1 from orders_p group by c1;
                                                  QUERY PLAN                                                  
--------------------------------------------------------------------------------------------------------------
 HashAggregate  (cost=508361.80..508365.45 rows=365 width=12) (actual rows=365 loops=1)
   Group Key: orders_01.c1
   ->  Append  (cost=0.00..408317.35 rows=20008890 width=4) (actual rows=20000000 loops=1)
         ->  Seq Scan on orders_01  (cost=0.00..22.70 rows=1270 width=4) (actual rows=0 loops=1)
         ->  Seq Scan on orders_02  (cost=0.00..22.70 rows=1270 width=4) (actual rows=0 loops=1)
[...]
         ->  Seq Scan on orders_19  (cost=0.00..45308.04 rows=2941004 width=4) (actual rows=2941004 loops=1)
         ->  Seq Scan on orders_20  (cost=0.00..131708.21 rows=8549421 width=4) (actual rows=8549421 loops=1)
 Planning Time: 0.576 ms
 Execution Time: 5273.217 ms
(25 rows)

set enable_partitionwise_aggregate to on;

explain (analyze,timing off) select count(*), c1 from orders_p group by c1;
                                                  QUERY PLAN                                                  
--------------------------------------------------------------------------------------------------------------
 Append  (cost=29.05..408343.83 rows=1765 width=12) (actual rows=365 loops=1)
   ->  HashAggregate  (cost=29.05..31.05 rows=200 width=12) (actual rows=0 loops=1)
         Group Key: orders_01.c1
         ->  Seq Scan on orders_01  (cost=0.00..22.70 rows=1270 width=4) (actual rows=0 loops=1)
   ->  HashAggregate  (cost=29.05..31.05 rows=200 width=12) (actual rows=0 loops=1)
         Group Key: orders_02.c1
         ->  Seq Scan on orders_02  (cost=0.00..22.70 rows=1270 width=4) (actual rows=0 loops=1)
[...]
   ->  HashAggregate  (cost=60013.06..60013.37 rows=31 width=12) (actual rows=31 loops=1)
         Group Key: orders_19.c1
         ->  Seq Scan on orders_19  (cost=0.00..45308.04 rows=2941004 width=4) (actual rows=2941004 loops=1)
   ->  HashAggregate  (cost=174455.32..174455.55 rows=24 width=12) (actual rows=24 loops=1)
         Group Key: orders_20.c1
         ->  Seq Scan on orders_20  (cost=0.00..131708.21 rows=8549421 width=4) (actual rows=8549421 loops=1)
 Planning Time: 1.461 ms
 Execution Time: 4669.315 ms
(63 rows)
{{< / highlight >}}

In the first case the aggregation is done once for all the tables, while in the second example the aggregation is done per partition.
We can also notice that the total cost is lower in the plan with aggregation per partition.

The `partitionwise join` performs on the same principle, we do a partition per partition join. It is useful to join two partitioned tables.


## Storage tiering

Finally, another use case would be to store a part of the table on a different storage:

We can store a partitioned table in different tablespaces. For example recent data on a fast tablespace on NVMe SSD.
Then, the more rarely accessed data on another tablespace, with less expensive mechanical disks.

This approach can also make sense in the cloud era where storage is very expensive.

# Conclusion

I think I've covered the main use cases that came to my mind.

Obviously, the implementation of partitioning implies a bigger complexity (management of partitions...)
and limitations that will have to be studied upstream.

[^1]: "BRIN indexes provide similar benefits to horizontal partitioning or sharding but without needing to explicitly declare partitions." - <https://en.wikipedia.org/wiki/Block_Range_Index>

