+++
title = "PostgreSQL and heap-only-tuples updates - part 3"
date = 2018-11-26T08:00:00+01:00
draft = false
summary = "Impact on performances"

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
3. [Impact on performances][3]

**This feature was disabled in 11.1 because it could lead to instance crashes[^4].
I chose to publish these articles because they help to understand the mechanism
of HOT updates and the benefits that this feature could bring.**

I thank Guillaume Lelarge for his review of this article ;).

# Impact on performance

Here is a simple test to demonstrate the benefits of this feature.
We could expect performance gains because postgres avoids updating the indexes,
as well as in terms of index size, as seen above, fragmentation is avoided.

```sql
CREATE TABLE t5 (c1 jsonb, c2 int,c3 int);
CREATE INDEX ON t5 ((c1->>'prenom')) ;
CREATE INDEX ON t5 (c2);
INSERT INTO t5 VALUES ('{ "prenom":"adrien" , "valeur" : "1"}'::jsonb,1,1);
INSERT INTO t5 VALUES ('{ "prenom":"guillaume" , "valeur" : "2"}'::jsonb,2,2);
\dt+ t5
                   List of relations
 Schema | Name | Type  |  Owner   | Size  | Description
--------+------+-------+----------+-------+-------------
 public | t5   | table | postgres | 16 kB |
(1 row)

\di+ t5*
                           List of relations
 Schema |    Name     | Type  |  Owner   | Table | Size  | Description
--------+-------------+-------+----------+-------+-------+-------------
 public | t5_c2_idx   | index | postgres | t5    | 16 kB |
 public | t5_expr_idx | index | postgres | t5    | 16 kB |
(2 rows)
```

Then this pgbench test:

```sql
\set id  random(1, 100000)
\set id2  random(1, 100000)

UPDATE t5 SET c1 = '{"valeur": ":id", "prenom": "guillaume"}' WHERE c2=2;
UPDATE t5 SET c1 = '{"valeur": ":id2", "prenom": "adrien"}' WHERE c2=1;
```

That we execute for 60 seconds with `recheck_on_update=on` (default):

```
pgbench -f test.sql -n -c6 -T 120
transaction type: test.sql
scaling factor: 1
query mode: simple
number of clients: 6
number of threads: 1
duration: 120 s
number of transactions actually processed: 2743163
latency average = 0.262 ms
tps = 22859.646914 (including connections establishing)
tps = 22859.938191 (excluding connections establishing)

 \dt+ t5*
                    List of relations
 Schema | Name | Type  |  Owner   |  Size  | Description
--------+------+-------+----------+--------+-------------
 public | t5   | table | postgres | 376 kB |
(1 row)
\di+ t5*
                           List of relations
 Schema |    Name     | Type  |  Owner   | Table | Size  | Description
--------+-------------+-------+----------+-------+-------+-------------
 public | t5_c2_idx   | index | postgres | t5    | 16 kB |
 public | t5_expr_idx | index | postgres | t5    | 32 kB |
(2 rows)

SELECT * from pg_stat_user_tables where relname = 't5';
-[ RECORD 1 ]-------+------------------------------
relid               | 8890622
schemaname          | public
relname             | t5
seq_scan            | 4
seq_tup_read        | 0
idx_scan            | 7999055
idx_tup_fetch       | 7999055
n_tup_ins           | 4
n_tup_upd           | 7999055
n_tup_del           | 0
n_tup_hot_upd       | 7998236
n_live_tup          | 2
n_dead_tup          | 0
n_mod_since_analyze | 0
last_vacuum         |
last_autovacuum     | 2018-09-19 06:29:37.690575+00
last_analyze        |
last_autoanalyze    | 2018-09-19 06:29:37.719911+00
vacuum_count        | 0
autovacuum_count    | 5
analyze_count       | 0
autoanalyze_count   | 5
```

Now with `recheck_on_update=off`. So, why the same data set as before but this
time the index is created with this order:
`CREATE INDEX ON t5 ((c1->>'prenom')) WITH (recheck_on_update=off);`


```
pgbench -f test.sql -n -c6 -T 120
transaction type: test.sql
scaling factor: 1
query mode: simple
number of clients: 6
number of threads: 1
duration: 120 s
number of transactions actually processed: 1065688
latency average = 0.676 ms
tps = 8880.679565 (including connections establishing)
tps = 8880.796478 (excluding connections establishing)

\dt+ t5
                    List of relations
 Schema | Name | Type  |  Owner   |  Size   | Description
--------+------+-------+----------+---------+-------------
 public | t5   | table | postgres | 9496 kB |
(1 row)

\di+ t5*
                           List of relations
 Schema |    Name     | Type  |  Owner   | Table |  Size  | Description
--------+-------------+-------+----------+-------+--------+-------------
 public | t5_c2_idx   | index | postgres | t5    | 768 kB |
 public | t5_expr_idx | index | postgres | t5    | 58 MB  |
(2 rows)

select * from pg_stat_user_tables where relname = 't5';
-[ RECORD 1 ]-------+------------------------------
relid               | 8890635
schemaname          | public
relname             | t5
seq_scan            | 2
seq_tup_read        | 0
idx_scan            | 2131376
idx_tup_fetch       | 2131376
n_tup_ins           | 2
n_tup_upd           | 2131376
n_tup_del           | 0
n_tup_hot_upd       | 19
n_live_tup          | 2
n_dead_tup          | 0
n_mod_since_analyze | 0
last_vacuum         |
last_autovacuum     | 2018-09-19 06:34:42.045905+00
last_analyze        |
last_autoanalyze    | 2018-09-19 06:34:42.251183+00
vacuum_count        | 0
autovacuum_count    | 3
analyze_count       | 0
autoanalyze_count   | 3
```
| recheck_on_update | on     | off     | Gain   |
|------------------:|--------|---------|--------|
|               TPS | 22859  | 8880    | 157%   |
|           t5 size | 376 kB | 9496 kB | -96%   |
|    t5_c2_idx size | 16 kB  | 768 kB  | -98%   |
|  t5_expr_idx size | 32 kB  | 58 MB   | -99.9% |

The performance difference is quite impressive, as well as the size of the tables and indexes.

I did the first test again by disabling the autovacuum and here is the result:

```
pgbench -f test.sql -n -c6 -T 120
transaction type: test.sql
scaling factor: 1
query mode: simple
number of clients: 6
number of threads: 1
duration: 120 s
number of transactions actually processed: 2752479
latency average = 0.262 ms
tps = 22937.271749 (including connections establishing)
tps = 22937.545872 (excluding connections establishing)


select * from pg_stat_user_tables where relname = 't5';
-[ RECORD 1 ]-------+--------
relid               | 8890643
schemaname          | public
relname             | t5
seq_scan            | 2
seq_tup_read        | 0
idx_scan            | 5504958
idx_tup_fetch       | 5504958
n_tup_ins           | 2
n_tup_upd           | 5504958
n_tup_del           | 0
n_tup_hot_upd       | 5504258
n_live_tup          | 2
n_dead_tup          | 2416
n_mod_since_analyze | 5504960
last_vacuum         |
last_autovacuum     |
last_analyze        |
last_autoanalyze    |
vacuum_count        | 0
autovacuum_count    | 0
analyze_count       | 0
autoanalyze_count   | 0

\di+ t5*
List of relations
-[ RECORD 1 ]------------
Schema      | public
Name        | t5_c2_idx
Type        | index
Owner       | postgres
Table       | t5
Size        | 16 kB
Description |
-[ RECORD 2 ]------------
Schema      | public
Name        | t5_expr_idx
Type        | index
Owner       | postgres
Table       | t5
Size        | 40 kB
Description |

\dt+ t5
List of relations
-[ RECORD 1 ]---------
Schema      | public
Name        | t5
Type        | table
Owner       | postgres
Size        | 1080 kB
Description |
```

Then the second test:

```
pgbench -f test.sql -n -c6 -T 120
transaction type: test.sql
scaling factor: 1
query mode: simple
number of clients: 6
number of threads: 1
duration: 120 s
number of transactions actually processed: 881434
latency average = 0.817 ms
tps = 7345.208875 (including connections establishing)
tps = 7345.304797 (excluding connections establishing)

select * from pg_stat_user_tables where relname = 't5';
-[ RECORD 1 ]-------+--------
relid               | 8890651
schemaname          | public
relname             | t5
seq_scan            | 2
seq_tup_read        | 0
idx_scan            | 1762868
idx_tup_fetch       | 1762868
n_tup_ins           | 2
n_tup_upd           | 1762868
n_tup_del           | 0
n_tup_hot_upd       | 23
n_live_tup          | 2
n_dead_tup          | 1762845
n_mod_since_analyze | 1762870
last_vacuum         |
last_autovacuum     |
last_analyze        |
last_autoanalyze    |
vacuum_count        | 0
autovacuum_count    | 0
analyze_count       | 0
autoanalyze_count   | 0

\di+ t5*
List of relations
-[ RECORD 1 ]------------
Schema      | public
Name        | t5_c2_idx
Type        | index
Owner       | postgres
Table       | t5
Size        | 600 kB
Description |
-[ RECORD 2 ]------------
Schema      | public
Name        | t5_expr_idx
Type        | index
Owner       | postgres
Table       | t5
Size        | 56 MB
Description |

\dt+ t5*
List of relations
-[ RECORD 1 ]---------
Schema      | public
Name        | t5
Type        | table
Owner       | postgres
Size        | 55 MB
Description |
```

| recheck_on_update | on      | off    | Gain   |
|------------------:|---------|--------|--------|
|               TPS | 22937   | 7345   | 212%   |
|           t5 size | 1080 kB | 55 MB  | -98%   |
|    t5_c2_idx size | 16 kB   | 600 kB | -97%   |
|  t5_expr_idx size | 40 kB   | 56 MB  | -99.9% |

Once again, the performance gap is significant, as is the size of tables and
indexes. We also note the importance of leaving the autovacuum activated.

Why do we have such a large difference on the indexes and the table?

For the indexes, this is due to the mechanism explained above. Postgres was
able to chain the records by avoiding updating the index. The index has
nevertheless slightly increased in size, it may happen that postgres cannot make a HOT.
For example when there is no more space in the block.

As for the size of the table, during the test with autovacuum activated, the
autovacuum had more difficulty to pass on the table with the HOT disabled.
The index growing, it resulted in more "work".
During the test without autovacuum, the difference is explained by the fact that
even a simple SELECT can prune blocs[^5].

Remember that this feature was removed with version 11.1. I wrote these articles shortly after the release of version 11.0 and I still choose to publish them in order to explain how UPDATES HOT works. Hopefully this feature will be fixed in future versions.

[1]: https://blog.anayrat.info/en/2018/11/12/postgresql-and-heap-only-tuples-updates-part-1/
[2]: https://blog.anayrat.info/en/2018/11/19/postgresql-and-heap-only-tuples-updates-part-2/
[3]: https://blog.anayrat.info/en/2018/11/26/postgresql-and-heap-only-tuples-updates-part-3/

[^4]: [Disable recheck_on_update optimization to avoid crashes](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=05f84605dbeb9cf8279a157234b24bbb706c5256)
[^5]: [README.HOT](https://github.com/postgres/postgres/blob/master/src/backend/access/heap/README.HOT#L251)
