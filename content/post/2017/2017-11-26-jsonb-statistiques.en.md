+++
title = "PostgreSQL - JSONB and Statistics"
date = 2017-11-26T21:11:38+01:00
draft = false

# Tags and categories
# For example, use `tags = []` for no tags, or the form `tags = ["A Tag", "Another Tag"]` for one or more tags.
tags = ["postgres","JSONB","statistics"]
categories = ["Postgres"]

# Featured image
# Place your image in the `static/img/` folder and reference its filename below, e.g. `image = "example.jpg"`.
# Use `caption` to display an image caption.
#   Markdown linking is allowed, e.g. `caption = "[Image credit](http://example.org)"`.
# Set `preview` to `false` to disable the thumbnail in listings.
[header]
image = ""
caption = ""
preview = true

+++

{{% toc %}}

# Statistics, cardinality, selectivity

SQL is a declarative language. It is a language where the user asks what he wants. Without specifying how the computer should proceed to get the results.

It is the DBMS that must find "how" to perform the operation by ensuring:

   * Return the right result
   * Ideally, as soon as possible

"As soon as possible" means:

   * Minimize disk access
   * Give priority to sequential readings (especially important for mechanical disks)
   * Reduce the number of CPU operations
   * Reduce memory footprint

To do this, a DBMS has an optimizer whose role is to find the best *execution plan*.

PostgreSQL has an optimizer based on a cost mechanism.
Without going into details, each operation has a unit cost (reading a sequential block, CPU processing of a record ...).
Postgres calculates the cost of several execution plans (if the query is simple) and chooses the least expensive.

How can postgres estimate the cost of a plan? By estimating the cost of each node of the plan based on statistics.
PostgreSQL analyzes tables to obtain a statistical sample (this operation is normally performed by the *autovacuum* daemon).

Some words of vocabulary:

*Cardinality*: In set theory, it is the number of elements in a set. In databases, it will be the number of rows in a table or after applying a predicate.

*Selectivity*: Fraction of records returned after applying a predicate. For example, a table containing people and about one third of them are children. The selectivity of the predicate `person = 'child'` will be 0.33.

If this table contains 300 people (this is the cardinality of the "people" set), we can estimate the number of children because we know that the predicate `person = 'child'` is 0.33:

300 * 0.33 = 99

These estimates can be obtained with `EXPLAIN` which displays the execution plan.

Example (simplified):

```SQL
explain (analyze, timing off) select * from t1 WHERE c1=1;
                                  QUERY PLAN
------------------------------------------------------------------------------
 Seq Scan on t1  (cost=0.00..5.75 rows=100 ...) (actual rows=100 ...)
   Filter: (c1 = 1)
   Rows Removed by Filter: 200
```

*(cost=0.00..5.75 rows=100 ...)* : Indicates the estimated cost and the estimated number of records (*rows*).

*(actual rows=100 ...)* : Indicates the number of records obtained.

PostgreSQL documentation provides examples of estimation calculations : [Row Estimation Examples](https://www.postgresql.org/docs/current/static/row-estimation-examples.html)

It is quite easy to understand how to obtain estimates from scalar data types.

How are things going for particular types? For example JSON?

# Search on JSONB

## Dataset

As in previous articles, I used the stackoverflow dataset. I created a new table by aggregating data from multiple tables into a JSON object:

```SQL
CREATE TABLE json_stack AS
SELECT t.post_id,
       row_to_json(t,
                   TRUE)::jsonb json
FROM
  (SELECT posts.id post_id,
          posts.owneruserid,
          users.id,
          title,
          tags,
          BODY,
          displayname,
          websiteurl,
          LOCATION,
          aboutme,
          age
   FROM posts
   JOIN users ON posts.owneruserid = users.id) t;
```

The processing is quite long because the two tables involved total nearly 40GB.

So I get a 40GB table that looks like this:

```SQL
 \dt+ json_stack
                      List of relations
 Schema |    Name    | Type  |  Owner   | Size  | Description
--------+------------+-------+----------+-------+-------------
 public | json_stack | table | postgres | 40 GB |
(1 row)


\d json_stack
             Table "public.json_stack"
 Column  |  Type   | Collation | Nullable | Default
---------+---------+-----------+----------+---------
 post_id | integer |           |          |
 json    | jsonb   |           |          |


select post_id,jsonb_pretty(json) from json_stack
    where json_displayname(json) = 'anayrat' limit 1;
 post_id  |
----------+-----------------------------------------------------------------------------------------
 26653490 | {
          |     "id": 4197886,
          |     "age": null,
          |     "body": "<p>I have an issue with date filter. I follow [...]
          |     "tags": "<java><logstash>",
          |     "title": "Logstash date filter failed parsing",
          |     "aboutme": "<p>Sysadmin, Postgres DBA</p>\n",
          |     "post_id": 26653490,
          |     "location": "Valence",
          |     "websiteurl": "https://blog.anayrat.info",
          |     "displayname": "anayrat",
          |     "owneruserid": 4197886
          | }
```

## Operators and indexing for JSONB

PostgreSQL provides several operators for querying JSONB [^1]. We will use the operator `@>`.

[^1]: <https://www.postgresql.org/docs/current/static/functions-json.html>

It is also possible to index JSONB using GIN indexes:

```SQL
create index ON json_stack using gin (json );
```

Finally, here is an example of query:

```SQL
explain (analyze,buffers)  select * from json_stack
    where json @>  '{"displayname":"anayrat"}'::jsonb;
                                  QUERY PLAN
---------------------------------------------------------------------------------------
 Bitmap Heap Scan on json_stack
                (cost=286.95..33866.98 rows=33283 width=1011)
                (actual time=0.099..0.102 rows=2 loops=1)
   Recheck Cond: (json @> '{"displayname": "anayrat"}'::jsonb)
   Heap Blocks: exact=2
   Buffers: shared hit=17
   ->  Bitmap Index Scan on json_stack_json_idx
                          (cost=0.00..278.62 rows=33283 width=0)
                          (actual time=0.092..0.092 rows=2 loops=1)
         Index Cond: (json @> '{"displayname": "anayrat"}'::jsonb)
         Buffers: shared hit=15
 Planning time: 0.088 ms
 Execution time: 0.121 ms
(9 rows)
```

Reading this plan we see that postgres is completely wrong.
He estimates getting 33,283 lines, but the query returns only two rows. The error factor is around 15,000!

## Selectivity on JSONB

What is the cardinality of the table? The information is contained in the system catalog:


```SQL
select reltuples from pg_class where relname = 'json_stack';
  reltuples
-------------
 3.32833e+07
```

What is the estimated selectivity?

```SQL
select 33283 / 3.32833e+07;
        ?column?
------------------------
 0.00099999098647069251
```

Arround 0.001.

## Diving in the code

I had fun taking out the debugger GDB to find out where this number could come from. I ended up arriving in this function:

```C
[...]
79 /*
80  *  contsel -- How likely is a box to contain (be contained by) a given box?
81  *
82  * This is a tighter constraint than "overlap", so produce a smaller
83  * estimate than areasel does.
84  */
85
86 Datum
87 contsel(PG_FUNCTION_ARGS)
88 {
89     PG_RETURN_FLOAT8(0.001);
90 }
[...]
```

The selectivity depends on the type of the operator. Let's look in the system catalog:


```sql
select oprname,typname,oprrest from pg_operator op
    join pg_type typ ON op.oprleft= typ.oid where oprname = '@>';
 oprname | typname  |   oprrest
---------+----------+--------------
 @>      | polygon  | contsel
 @>      | box      | contsel
 @>      | box      | contsel
 @>      | path     | -
 @>      | polygon  | contsel
 @>      | circle   | contsel
 @>      | _aclitem | -
 @>      | circle   | contsel
 @>      | anyarray | arraycontsel
 @>      | tsquery  | contsel
 @>      | anyrange | rangesel
 @>      | anyrange | rangesel
 @>      | jsonb    | contsel
```

There are several types, in fact the operator `@>` means (roughly): "Does the object on the left contain the right element?".
It is used for different types: geometry, array ...

In our case, does the left JSONB object contain the `''{" displayname ":" anayrat "}''` element?

A JSON object is a special type. Determining the selectivity of an element would be quite complex. The comment is quite explicit:

```C
 25 /*
 26  *  Selectivity functions for geometric operators.  These are bogus -- unless
 27  *  we know the actual key distribution in the index, we can't make a good
 28  *  prediction of the selectivity of these operators.
 29  *
 30  *  Note: the values used here may look unreasonably small.  Perhaps they
 31  *  are.  For now, we want to make sure that the optimizer will make use
 32  *  of a geometric index if one is available, so the selectivity had better
 33  *  be fairly small.
[...]
```

It is therefore not possible (currently) to determine the selectivity of JSONB objects.

But all is not lost :wink:

# Functional indexes

PostgreSQL permits to creates so-called *functional* indexes. We create an index on a fonction.

You're going to say, "Yes, but we do not need it." In your example, postgres is already using an index.

That's right, the difference is that postgres collects statistics about this index.
As if the result of the function was a new column.

## Creating the function and the index

It is very simple :

```plpgsql
CREATE or replace FUNCTION json_displayname (jsonb )
RETURNS text
AS $$
select $1->>'displayname'
$$
LANGUAGE SQL IMMUTABLE PARALLEL SAFE
;

create index ON json_stack (json_displayname(json));
```

## Search using a function

To use the index we just created, use it in the query:

```SQL
explain (analyze,verbose,buffers) select * from json_stack
        where json_displayname(json) = 'anayrat';
                        QUERY PLAN
----------------------------------------------------------------------------
 Index Scan using json_stack_json_displayname_idx on public.json_stack
            (cost=0.56..371.70 rows=363 width=1011)
            (actual time=0.021..0.023 rows=2 loops=1)
   Output: post_id, json
   Index Cond: ((json_stack.json ->> 'displayname'::text) = 'anayrat'::text)
   Buffers: shared hit=7
 Planning time: 0.107 ms
 Execution time: 0.037 ms
(6 rows)
```

This time postgres estimates to get 363 rows, which is much closer to the final result (2).


## Another example and selectivity calculation

This time we will search on the "age" field of the JSON object:


```SQL
explain (analyze,buffers)  select * from json_stack
      where json @>  '{"age":27}'::jsonb;
                      QUERY PLAN
---------------------------------------------------------------------
 Bitmap Heap Scan on json_stack
        (cost=286.95..33866.98 rows=33283 width=1011)
        (actual time=667.411..12723.906 rows=804630 loops=1)
   Recheck Cond: (json @> '{"age": 27}'::jsonb)
   Rows Removed by Index Recheck: 2211190
   Heap Blocks: exact=391448 lossy=344083
   Buffers: shared hit=576350 read=881510
   I/O Timings: read=2947.458
   ->  Bitmap Index Scan on json_stack_json_idx
        (cost=0.00..278.62 rows=33283 width=0)
        (actual time=562.648..562.648 rows=804644 loops=1)
         Index Cond: (json @> '{"age": 27}'::jsonb)
         Buffers: shared hit=9612 read=5140
         I/O Timings: read=11.195
 Planning time: 0.073 ms
 Execution time: 12809.392 ms
(12 lignes)

set work_mem = '100MB';

explain (analyze,buffers)  select * from json_stack
      where json @>  '{"age":27}'::jsonb;
                      QUERY PLAN
---------------------------------------------------------------------
 Bitmap Heap Scan on json_stack
        (cost=286.95..33866.98 rows=33283 width=1011)
        (actual time=748.968..5720.628 rows=804630 loops=1)
   Recheck Cond: (json @> '{"age": 27}'::jsonb)
   Rows Removed by Index Recheck: 14
   Heap Blocks: exact=735531
   Buffers: shared hit=123417 read=780542
   I/O Timings: read=1550.124
   ->  Bitmap Index Scan on json_stack_json_idx
        (cost=0.00..278.62 rows=33283 width=0)
        (actual time=545.553..545.553 rows=804644 loops=1)
         Index Cond: (json @> '{"age": 27}'::jsonb)
         Buffers: shared hit=9612 read=5140
         I/O Timings: read=11.265
 Planning time: 0.079 ms
 Execution time: 5796.219 ms
(12 lignes)
```

In this example we see that postgres still estimates 33,283 records.
Out he gets 804 644. This time he is too much optimistic.

P.S: In my example you will see that I run the same query by modifying `work_mem`. This is to prevent the bitmap from being `lossy` [^2]

[^2]: A bitmap node becomes lossy when postgres can not make a bitmap of all tuples. It thus passes in so-called "lossy" mode where the bitmap is no longer on the tuple but for the entire block. This requires reading more blocks and doing a "recheck" which consists in filtering obtained tuples.

As seen above we can create a function:

```SQL
CREATE or replace FUNCTION json_age (jsonb )
RETURNS text
AS $$
select $1->>'age'
$$
LANGUAGE SQL IMMUTABLE PARALLEL SAFE
;

create index ON json_stack (json_age(json));
```

Again the estimate is much better:

```SQL
explain (analyze,buffers)   select * from json_stack
      where json_age(json) = '27';
                         QUERY PLAN
------------------------------------------------------------------------
 Index Scan using json_stack_json_age_idx on json_stack
  (cost=0.56..733177.05 rows=799908 width=1011)
  (actual time=0.042..2355.179 rows=804630 loops=1)
   Index Cond: ((json ->> 'age'::text) = '27'::text)
   Buffers: shared read=737720
   I/O Timings: read=1431.275
 Planning time: 0.087 ms
 Execution time: 2410.269 ms
```

Postgres estimates to get 799,908 records. we will check it.

As I said, Postgres has statistics information based on a sample of data.
This information is stored in a readable system catalog with the `pg_stats` view.
With a functional index, Postgres sees it as a new column.

```psql
schemaname             | public
tablename              | json_stack_json_age_idx
attname                | json_age
[...]
most_common_vals       | {28,27,29,31,26,30,32,25,33,34,36,24,[...]}
most_common_freqs      | {0.0248,0.0240333,0.0237333,0.0236333,0.0234,0.0229333,[...]}
[...]
```

The column *most_common_vals* contains the most common values and the column *most_common_freqs* the corresponding selectivity.

So for `age = 27` we have a selectivity of 0.0240333.

Then we just have to multiply the selectivity by the cardinality of the table:

```SQL
select n_live_tup from pg_stat_all_tables where relname ='json_stack';
 n_live_tup
------------
   33283258


select 0.0240333 * 33283258;
    ?column?
----------------
 799906.5244914
```

Okay, estimate is much better. But is it serious if postgres is wrong?
In the two queries above we see that postgres uses an index and that the result is obtained quickly.

# Consequences of a bad estimate

How can a bad estimate be a problem?

When that leads to the choice of a bad plan.

For example, this aggregation query that counts the number of posts by age:

```SQL
explain (analyze,buffers)  select json->'age',count(json->'age')
                          from json_stack group by json->'age' ;
                             QUERY PLAN
--------------------------------------------------------------------------------------
 Finalize GroupAggregate
  (cost=10067631.49..14135810.84 rows=33283256 width=40)
  (actual time=364151.518..411524.862 rows=86 loops=1)
   Group Key: ((json -> 'age'::text))
   Buffers: shared hit=1949354 read=1723941, temp read=1403174 written=1403189
   I/O Timings: read=155401.828
   ->  Gather Merge
        (cost=10067631.49..13581089.91 rows=27736046 width=40)
        (actual time=364151.056..411524.589 rows=256 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         Buffers: shared hit=1949354 read=1723941, temp read=1403174 written=1403189
         I/O Timings: read=155401.828
         ->  Partial GroupAggregate
            (cost=10066631.46..10378661.98 rows=13868023 width=40)
            (actual time=363797.836..409187.566 rows=85 loops=3)
               Group Key: ((json -> 'age'::text))
               Buffers: shared hit=5843962 read=5177836,
                        temp read=4212551 written=4212596
               I/O Timings: read=478460.123
               ->  Sort
                  (cost=10066631.46..10101301.52 rows=13868023 width=1042)
                  (actual time=299775.029..404358.743 rows=11094533 loops=3)
                     Sort Key: ((json -> 'age'::text))
                     Sort Method: external merge  Disk: 11225392kB
                     Buffers: shared hit=5843962 read=5177836,
                              temp read=4212551 written=4212596
                     I/O Timings: read=478460.123
                     ->  Parallel Seq Scan on json_stack
                        (cost=0.00..4791997.29 rows=13868023 width=1042)
                        (actual time=0.684..202361.133 rows=11094533 loops=3)
                           Buffers: shared hit=5843864 read=5177836
                           I/O Timings: read=478460.123
 Planning time: 0.080 ms
 Execution time: 411688.165 ms
```

Postgres expects to get 33,283,256 records instead of 86. It also performed a very expensive sort since it generated more than 33GB (11GB * 3 loops) of temporary files.

The same query using the *json_age* function:


```SQL
explain (analyze,buffers)   select json_age(json),count(json_age(json))
                              from json_stack group by json_age(json);
                                             QUERY PLAN
--------------------------------------------------------------------------------------
 Finalize GroupAggregate
  (cost=4897031.22..4897033.50 rows=83 width=40)
  (actual time=153985.585..153985.667 rows=86 loops=1)
   Group Key: ((json ->> 'age'::text))
   Buffers: shared hit=1938334 read=1736761
   I/O Timings: read=106883.908
   ->  Sort
      (cost=4897031.22..4897031.64 rows=166 width=40)
      (actual time=153985.581..153985.598 rows=256 loops=1)
         Sort Key: ((json ->> 'age'::text))
         Sort Method: quicksort  Memory: 37kB
         Buffers: shared hit=1938334 read=1736761
         I/O Timings: read=106883.908
         ->  Gather
            (cost=4897007.46..4897025.10 rows=166 width=40)
            (actual time=153985.264..153985.360 rows=256 loops=1)
               Workers Planned: 2
               Workers Launched: 2
               Buffers: shared hit=1938334 read=1736761
               I/O Timings: read=106883.908
               ->  Partial HashAggregate
                  (cost=4896007.46..4896008.50 rows=83 width=40)
                  (actual time=153976.620..153976.635 rows=85 loops=3)
                     Group Key: (json ->> 'age'::text)
                     Buffers: shared hit=5811206 read=5210494
                     I/O Timings: read=320684.515
                     ->  Parallel Seq Scan on json_stack
                     (cost=0.00..4791997.29 rows=13868023 width=1042)
                     (actual time=0.090..148691.566 rows=11094533 loops=3)
                           Buffers: shared hit=5811206 read=5210494
                           I/O Timings: read=320684.515
 Planning time: 0.118 ms
 Execution time: 154086.685 ms

```

Here postgres sorts later on a lot less lines. The execution time is significantly reduced and we save especially 33GB of temporary files.

# Last word

Statistics are essential for choosing the best execution plan. Currently Postgres has advanced features for JSON [^4]
Unfortunately there is no possibility to add statistics on the JSONB type. Note that PostgreSQL 10 provides the infrastructure to [extend statistics](https://www.postgresql.org/docs/current/static/sql-createstatistics.html). Hopefully in the future it will be possible to extend them for special types.

In the meantime, it is possible to work around this limitation by using functional indexes.

[^4]: The [documentation](https://www.postgresql.org/docs/current/static/functions-json.html) is very complete and provides many examples: use of operators, indexing.
