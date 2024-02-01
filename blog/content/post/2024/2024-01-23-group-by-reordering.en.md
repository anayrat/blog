---
# Documentation: https://wowchemy.com/docs/managing-content/

title: "GROUP BY reordering"
subtitle: "New feature in PostgreSQL 17: GROUP BY reordering"
summary: "New feature in PostgreSQL 17: GROUP BY reordering"
authors: []
tags: ['PostgreSQL 17','News']
categories: ['Postgres']
date: 2024-01-26T10:00:00+01:00
lastmod: 2024-01-29T10:00:00+01:00
featured: false

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image:
  caption: ""
  focal_point: ""
  preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---

A commit caught my attention during my technical watch:

```diff
commit 0452b461bc405e6d35d8a14c02813c15e28ae516
Author:     Alexander Korotkov <akorotkov@postgresql.org>
AuthorDate: Sun Jan 21 22:21:36 2024 +0200
Commit:     Alexander Korotkov <akorotkov@postgresql.org>
CommitDate: Sun Jan 21 22:21:36 2024 +0200

    Explore alternative orderings of group-by pathkeys during optimization.

    When evaluating a query with a multi-column GROUP BY clause, we can minimize
    sort operations or avoid them if we synchronize the order of GROUP BY clauses
    with the ORDER BY sort clause or sort order, which comes from the underlying
    query tree. Grouping does not imply any ordering, so we can compare
    the keys in arbitrary order, and a Hash Agg leverages this. But for Group Agg,
    we simply compared keys in the order specified in the query. This commit
    explores alternative ordering of the keys, trying to find a cheaper one.

    The ordering of group keys may interact with other parts of the query, some of
    which may not be known while planning the grouping. For example, there may be
    an explicit ORDER BY clause or some other ordering-dependent operation higher up
    in the query, and using the same ordering may allow using either incremental
    sort or even eliminating the sort entirely.

    The patch always keeps the ordering specified in the query, assuming the user
    might have additional insights.

    This introduces a new GUC enable_group_by_reordering so that the optimization
    may be disabled if needed.

    Discussion: https://postgr.es/m/7c79e6a5-8597-74e8-0671-1c39d124c9d6%40sigaev.ru
    Author: Andrei Lepikhov, Teodor Sigaev
    Reviewed-by: Tomas Vondra, Claudio Freire, Gavin Flower, Dmitry Dolgov
    Reviewed-by: Robert Haas, Pavel Borisov, David Rowley, Zhihong Yu
    Reviewed-by: Tom Lane, Alexander Korotkov, Richard Guo, Alena Rybakina
```


We notice the explicit commit message mentioning peoples involved (12 reviewers!) as well as the link to the discussion :
[POC: GROUP BY optimization](https://www.postgresql.org/message-id/flat/7c79e6a5-8597-74e8-0671-1c39d124c9d6%40sigaev.ru)

Work began in 2018! It took 5 and a half years to reach this commit. It's the result of many exchanges to reach a consensus taking into consideration the multiple ideas.

To test this patch, I compiled Postgres from source. Let's take a simple example:

```SQL
create table t1 (c1 int, c2 int, c3 int);
create index on t1 (c1,c2);
insert into t1 SELECT i%100, i%1000, i from generate_series(1,10_000_000) i;
vacuum analyze t1;
```

We got a table of 422MB and an index of 66MB.

You may see, that I've used a Postgres 16 new feature by using underscores in the *generate_series*[^1].

[^1]: A little story: to add this feature, the author made the SQL standard evolve: [Grouping digits in SQL](http://peter.eisentraut.org/blog/2023/09/20/grouping-digits-in-sql) .

If we do a `GROUP BY` on *c1,c2*, we get this plan:

```
explain (settings, analyze,buffers) select count(*),c1,c2 from t1 group by c1,c2 order by c1,c2;
                                                                    QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------------------
 GroupAggregate  (cost=0.43..235342.27 rows=100000 width=16) (actual time=3.605..3501.887 rows=1000 loops=1)
   Group Key: c1, c2
   Buffers: shared hit=10447
   ->  Index Only Scan using t1_c1_c2_idx on t1  (cost=0.43..159340.96 rows=10000175 width=8) (actual time=0.070..1900.730 rows=10000000 loops=1)
         Heap Fetches: 0
         Buffers: shared hit=10447
 Settings: enable_group_by_reordering = 'off', random_page_cost = '1.1', max_parallel_workers_per_gather = '0'
 Planning:
   Buffers: shared hit=1
 Planning Time: 0.191 ms
 Execution Time: 3502.036 ms
```

*You'll notice that I've disabled this feature (enable_group_by_reordering=off). I've also disabled parallelization for clarity*.

We have the expected plan, Postgres reads 10447 blocks with an *Index Only Scan* path. It uses the plan that manipulates the fewest blocks possible.

However, if we change the group by order:

```
explain (settings, analyze,buffers) select count(*),c1,c2 from t1 group by c2,c1 order by c1,c2;
                                                          QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------
 Sort  (cost=602422.32..602672.32 rows=100000 width=16) (actual time=5393.446..5393.503 rows=1000 loops=1)
   Sort Key: c1, c2
   Sort Method: quicksort  Memory: 79kB
   Buffers: shared hit=96 read=53959
   ->  HashAggregate  (cost=514992.50..594117.50 rows=100000 width=16) (actual time=5392.351..5392.894 rows=1000 loops=1)
         Group Key: c2, c1
         Batches: 1  Memory Usage: 3217kB
         Buffers: shared hit=96 read=53959
         ->  Seq Scan on t1  (cost=0.00..154055.00 rows=10000000 width=8) (actual time=0.033..1186.171 rows=10000000 loops=1)
               Buffers: shared hit=96 read=53959
 Settings: enable_group_by_reordering = 'off', random_page_cost = '1.1', max_parallel_workers_per_gather = '0'
 Planning:
   Buffers: shared hit=1
 Planning Time: 0.189 ms
 Execution Time: 5394.566 ms
```


In this case, Postgres will read the entire table (*seqscan*) and sort it, although the result is identical.

It reads 422MB of data versus 80MB, i.e. 5 times more. The result can be disastrous, given storage performance.
In this case, we're lucky: my instance is on a ramdisk, so the query isn't much slower.
With mechanical storage or cloud disks, response times can increase drastically.


The order of columns in a group by is important, and is a fairly simple optimization as long as you know the database schema and are able to control the queries executed on the server.

Unfortunately, with ORMs, this understanding can be lost or corrections missed.
That's where this feature comes in. Let's see what it does:


```
postgres=# set enable_group_by_reordering to on;
SET
postgres=# explain (settings, analyze,buffers) select count(*),c1,c2 from t1 group by c2,c1 order by c1,c2;
                                                                    QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------------------
 GroupAggregate  (cost=0.43..235338.33 rows=100000 width=16) (actual time=4.502..3658.168 rows=1000 loops=1)
   Group Key: c1, c2
   Buffers: shared hit=10447
   ->  Index Only Scan using t1_c1_c2_idx on t1  (cost=0.43..159338.33 rows=10000000 width=8) (actual time=0.081..1923.553 rows=10000000 loops=1)
         Heap Fetches: 0
         Buffers: shared hit=10447
 Settings: random_page_cost = '1.1', max_parallel_workers_per_gather = '0'
 Planning:
   Buffers: shared hit=1
 Planning Time: 0.230 ms
 Execution Time: 3658.337 ms
```

We got back the first plan, much more performant.


This is a simple feature that should improve execution times for certain queries whose group by clause has not been optimized.

It's a recurring demand to improve the planner in order to handle apparently simple cases. Bear in mind that the risk is to increase the number of
operations in the planner. But we want this step to be as quick as possible: "we don't want to increase the complexity of the planner when we can correct the query".

However, this type of optimization can be accepted if we know that it won't be costly for the planner.

Let's just hope this feature won't be removed by the time Postgres 17 is released :)


