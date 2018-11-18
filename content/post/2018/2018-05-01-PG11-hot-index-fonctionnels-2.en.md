+++
title = "PostgreSQL and heap-only-tuples updates - part 2"
date = 2018-11-19T08:00:00+01:00
draft = false
summary = "When postgres do not use *heap-only-tuple* updates and introduction to the new feature in v11"

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

The previous article showed how heap-only-tuple UPDATE works. In this one, we will
see when Postgres does not perform heap-only-tuple UPDATE.
This will allow us to approach the functionality that should have been available
in version 11.


# Cases with an index on an updated column

Let's repeat the previous example and add an indexed column:

```sql
ALTER TABLE t3 ADD COLUMN c3 int;
CREATE INDEX ON t3(c3);
```

Previous UPDATE were on a non-indexed column. What happens if the update is on c3?


Content of the table and indexes before UPDATE:

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

SELECT * FROM  bt_page_items(get_raw_page('t3_c1_idx',1));              
 itemoffset | ctid  | itemlen | nulls | vars |          data
------------+-------+---------+-------+------+-------------------------
          1 | (0,1) |      16 | f     | f    | 01 00 00 00 00 00 00 00
          2 | (0,2) |      16 | f     | f    | 02 00 00 00 00 00 00 00
(2 rows)

SELECT * FROM  bt_page_items(get_raw_page('t3_c3_idx',1));
 itemoffset | ctid  | itemlen | nulls | vars | data
------------+-------+---------+-------+------+------
          1 | (0,1) |      16 | t     | f    |
          2 | (0,2) |      16 | t     | f    |
(2 rows)
```

No change on the table because the column c3 contains only nulls, we can see this
by looking at the index `t3_c3_idx` where `nulls` is  *true* on each line.

```sql
UPDATE t3 SET c3 = 7 WHERE c1=1;
SELECT * FROM  bt_page_items(get_raw_page('t3_c3_idx',1));
 itemoffset | ctid  | itemlen | nulls | vars |          data
------------+-------+---------+-------+------+-------------------------
          1 | (0,3) |      16 | f     | f    | 07 00 00 00 00 00 00 00
          2 | (0,1) |      16 | t     | f    |
          3 | (0,2) |      16 | t     | f    |
(3 rows)

SELECT lp,lp_flags,t_data,t_ctid FROM  heap_page_items(get_raw_page('t3',0));
 lp | lp_flags |           t_data           | t_ctid
----+----------+----------------------------+--------
  1 |        2 |                            |
  2 |        1 | \x0200000002000000         | (0,2)
  3 |        1 | \x010000000600000007000000 | (0,3)
  4 |        0 |                            |
  5 |        1 | \x0100000006000000         | (0,3)
(5 rows)

SELECT * FROM  bt_page_items(get_raw_page('t3_c1_idx',1));              
 itemoffset | ctid  | itemlen | nulls | vars |          data
------------+-------+---------+-------+------+-------------------------
          1 | (0,3) |      16 | f     | f    | 01 00 00 00 00 00 00 00
          2 | (0,1) |      16 | f     | f    | 01 00 00 00 00 00 00 00
          3 | (0,2) |      16 | f     | f    | 02 00 00 00 00 00 00 00
(3 rows)
```

We notice the new entry in the index for c3. The table does contain a new record
On the other hand, the `t3_c1_idx` index has also been updated. This results in
the addition of a third entry, even if the value in column c1 has not changed.

After a vacuum:

```sql
VACUUM t3;
SELECT * FROM  bt_page_items(get_raw_page('t3_c1_idx',1));
 itemoffset | ctid  | itemlen | nulls | vars |          data
------------+-------+---------+-------+------+-------------------------
          1 | (0,3) |      16 | f     | f    | 01 00 00 00 00 00 00 00
          2 | (0,2) |      16 | f     | f    | 02 00 00 00 00 00 00 00
(2 rows)

SELECT lp,lp_flags,t_data,t_ctid FROM  heap_page_items(get_raw_page('t3',0));
 lp | lp_flags |           t_data           | t_ctid
----+----------+----------------------------+--------
  1 |        0 |                            |
  2 |        1 | \x0200000002000000         | (0,2)
  3 |        1 | \x010000000600000007000000 | (0,3)
  4 |        0 |                            |
  5 |        0 |                            |
(5 rows)

SELECT * FROM  bt_page_items(get_raw_page('t3_c3_idx',1));
 itemoffset | ctid  | itemlen | nulls | vars |          data
------------+-------+---------+-------+------+-------------------------
          1 | (0,3) |      16 | f     | f    | 07 00 00 00 00 00 00 00
          2 | (0,2) |      16 | t     | f    |
(2 rows)
```

Postgres cleaned the index and the table. The first line of the table no longer
has the flag `REDIRECT`.



# New in version: 11 heap-only-tuple (HOT) with functional indexes


When a functional index is applied to the modified column, it may happen that
the result of the expression remains unchanged despite the updating of the column.
The key in the index would therefore remain unchanged.

Let's take an example: a functional index on a *specific key* of a JSON object.

```SQL
CREATE TABLE t4 (c1 jsonb, c2 int,c3 int);
CREATE INDEX ON t4 ((c1->>'prenom')) ;
CREATE INDEX ON t4 (c2);
INSERT INTO t4 VALUES ('{ "prenom":"adrien" , "ville" : "valence"}'::jsonb,1,1);
INSERT INTO t4 VALUES ('{ "prenom":"guillaume" , "ville" : "lille"}'::jsonb,2,2);


-- change that does not concern the first name, we change only the city
UPDATE t4 SET c1 = '{"ville": "valence (#soleil)", "prenom": "guillaume"}' WHERE c2=2;
SELECT pg_stat_get_xact_tuples_hot_updated('t4'::regclass);
 pg_stat_get_xact_tuples_hot_updated
-------------------------------------
                                   0
(1 row)

UPDATE t4 SET c1 = '{"ville": "nantes", "prenom": "guillaume"}' WHERE c2=2;
SELECT pg_stat_get_xact_tuples_hot_updated('t4'::regclass);
 pg_stat_get_xact_tuples_hot_updated
-------------------------------------
                                   0
(1 row)
```

The function `pg_stat_get_get_xact_tuples_hot_updated` indicates the number of
lines  updated by the HOT mechanism.

The two UPDATE only modified the "city" key and not the "first name" key.
This does not lead to a modification of the index because it only indexes the "first name" key.

Postgres could not make a HOT. Indeed, for him, the UPDATE is on the column and
the index must be updated.

With version 11, postgres is able to see that the result of the expression does
not change. Let's do the same test on version 11 :


```SQL
CREATE TABLE t4 (c1 jsonb, c2 int,c3 int);
-- CREATE INDEX ON t4 ((c1->>'prenom'))  WITH (recheck_on_update='false');
CREATE INDEX ON t4 ((c1->>'prenom')) ;
CREATE INDEX ON t4 (c2);
INSERT INTO t4 VALUES ('{ "prenom":"adrien" , "ville" : "valence"}'::jsonb,1,1);
INSERT INTO t4 VALUES ('{ "prenom":"guillaume" , "ville" : "lille"}'::jsonb,2,2);


-- changement qui ne porte pas sur prenom
UPDATE t4 SET c1 = '{"ville": "valence (#soleil)", "prenom": "guillaume"}' WHERE c2=2;
SELECT pg_stat_get_xact_tuples_hot_updated('t4'::regclass);
 pg_stat_get_xact_tuples_hot_updated
-------------------------------------
                                   1
(1 row)

UPDATE t4 SET c1 = '{"ville": "nantes", "prenom": "guillaume"}' WHERE c2=2;
SELECT pg_stat_get_xact_tuples_hot_updated('t4'::regclass);
 pg_stat_get_xact_tuples_hot_updated
-------------------------------------
                                   2
(1 row)
```

This time, Postgres used the HOT mechanism correctly. This can be verified by
looking at the physical content of the index with pageinspect:

Version 10 :
```sql
SELECT * FROM  bt_page_items(get_raw_page('t4_expr_idx',1));
itemoffset | ctid  | itemlen | nulls | vars |                      data                       
------------+-------+---------+-------+------+-------------------------------------------------
         1 | (0,1) |      16 | f     | t    | 0f 61 64 72 69 65 6e 00
         2 | (0,4) |      24 | f     | t    | 15 67 75 69 6c 6c 61 75 6d 65 00 00 00 00 00 00
         3 | (0,3) |      24 | f     | t    | 15 67 75 69 6c 6c 61 75 6d 65 00 00 00 00 00 00
         4 | (0,2) |      24 | f     | t    | 15 67 75 69 6c 6c 61 75 6d 65 00 00 00 00 00 00
(4 rows)
```

Version 11 :

```sql
SELECT * FROM  bt_page_items(get_raw_page('t4_expr_idx',1));
itemoffset | ctid  | itemlen | nulls | vars |                      data
------------+-------+---------+-------+------+-------------------------------------------------
         1 | (0,1) |      16 | f     | t    | 0f 61 64 72 69 65 6e 00
         2 | (0,2) |      24 | f     | t    | 15 67 75 69 6c 6c 61 75 6d 65 00 00 00 00 00 00
(2 rows)
```

This behavior can be controlled by a new option when creating the index: `recheck_on_update`.

On by default, postgres checks the result of the expression to perform a HOT UPDATE.
It can be set to `off` if there is a good chance that the result of the expression
will change during an UPDATE. This avoids executing the expression unnecessarily.

Also notes that postgres avoids the evaluation of the expression if its cost is higher than 1000.

In the third and last article, we will see a more concrete case to measure
impact in terms of  performance and volumetry.

[1]: https://blog.anayrat.info/en/2018/11/12/postgresql-and-heap-only-tuples-updates-part-1/
[2]: https://blog.anayrat.info/en/2018/11/19/postgresql-and-heap-only-tuples-updates-part-2/
[3]:

[^4]: [Disable recheck_on_update optimization to avoid crashes](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=05f84605dbeb9cf8279a157234b24bbb706c5256)
