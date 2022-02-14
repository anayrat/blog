+++
title = "PostgreSQL: TOAST compression and toast_tuple_target"
date = 2022-02-14T09:00:00+02:00
draft = false

summary = "Effects of TOAST and application of toast_tuple_target with JSONB"

# Tags and categories
# For example, use `tags = []` for no tags, or the form `tags = ["A Tag", "Another Tag"]` for one or more tags.
tags = ["postgres","toast", "jsonb"]
categories = ["Postgres"]

+++

Some reminders about TOAST and a new features appeared in PostgreSQL 11.


{{% toc %}}

# What is the TOAST ?

Have you ever wondered how Postgres stores rows that exceed the size of a block? As a reminder, the default block size is 8KB.

Postgres uses a mechanism called TOAST for [The Oversized-Attribute Storage Technique](https://www.postgresql.org/docs/current/storage-toast.html).

When a record becomes too big to be stored in a block, Postgres will store it " aside", in a toast table. The record will be split into chunks,
so the main table (called heap) will contain a pointer (chunk_id) pointing to the right chunk in the toast table.

This chunk will be stored on several rows, for one chunk_id we can have several rows in this toast table. Thus, this toast table is made up of 3 columns:

* *chunk_id* : Number of the chunk referenced in the heap
* *chunk_seq* : Number of each segment of a chunk
* *chunk_data* : Data part of each segment

The reality is a bit more complex, in fact Postgres will try to avoid storing the data in the toast table.
If the row exceeds `TOAST_TUPLE_THRESHOLD` (2Kb), it will try to compress the columns to try to fit the row into the block.
More precisely, the size has to be smaller than `TOAST_TUPLE_TARGET` (2Kb by default, we'll talk about that later).

If we are lucky, the compressed line will fit in the heap.
If not, it will try to compress the columns, from the biggest to the smallest, and store them in the toast part until the remaining columns fit in a row of the heap.[^toastpass]

[^toastpass]: See [heap_toast_insert_or_update](https://git.postgresql.org/gitweb/?p=postgresql.git;a=blob;f=src/backend/access/heap/heaptoast.c;h=55bbe1d584760a849960871296dfbdd7447b2b67;hb=refs/heads/REL_14_STABLE#l160)


Note also that if the compression gain is too small, it considers that it is useless to spend resources trying to compress.
It therefore stores the data without compression. [^toastcompress]

[^toastcompress]: Two compression algorithms are supported : *pglz* (historical and integrated in Postgres) and *lz4* (since Postgres 14).

    * For *pglz*, see [PGLZ_Strategy](https://git.postgresql.org/gitweb/?p=postgresql.git;a=blob;f=src/include/common/pg_lzcompress.h;h=3e53fbe97bd0a10e3fbf7ed4396924084f657868;hb=refs/heads/REL_14_STABLE#l25)
and [strategy_default_data](https://git.postgresql.org/gitweb/?p=postgresql.git;a=blob;f=src/common/pg_lzcompress.c;h=a30a2c2eb83a71725754d8dd680621a02e7557e9;hb=refs/heads/REL_14_STABLE#l223)
    * For *lz4*, it is an external library. See [LZ4_compress_default](https://github.com/lz4/lz4/blob/dev/lib/lz4.h#L145).

Have you ever paid attention to the "Storage" column when you display the characteristics of a table with the meta command `\d+ table`?

```
stackoverflow=# \d+ posts
                   Table "public.posts"
    Column     |  Type   | Collation | Nullable | Default | Storage  |
---------------+---------+-----------+----------+---------+----------+
 id            | integer |           | not null |         | plain    |
 posttypeid    | integer |           | not null |         | plain    |
 score         | integer |           |          |         | plain    |
 viewcount     | integer |           |          |         | plain    |
 body          | text    |           |          |         | extended |
```


In this example, the column takes the value *plain* or *extended*. In fact, there are 4 possible values depending on the type of data:

* *plain* : the column is stored in the heap only and without compression.
* *extended* : the column can be compressed and stored in the toast if necessary.
* *external* : the column can be stored in the toast but only without compression. Sometimes this mode can be used to gain in performance (avoids compression/decompression)  at the cost of a higher consumption of disk space.
* *main*: The column is stored in the heap only but unlike the *plain* mode, compression is allowed.

At first thought, we may think that the advantage is mainly on the opportunity to store
rows exceeding the size of a block and compressing the data to save disk space.

There is another benefit: when a row is updated, if the "toasted" columns are not modified, Postgres does not need to update the toast table.
This avoids having to decompress and recompress the toast and write all this in transaction logs.

We will see that another advantage is that Postgres can avoid reading the toast if it is not necessary.

# Example with the JSONB

To study this, we will use the JSONB type. In general, I don't recommend the usage of this type:

* You lose the advantages of having a schema:
  * type checking
  * integrity constraints
  * no foreign keys
* Writing queries becomes more complex
* No statistics on the keys of a json field
* Loss of storage efficiency as we store the keys for each row
* No partial update of JSONB. If you change a key you have to *detoast* and *toast* the whole JSONB
* No partial *detoast* : if we want to read only one key of the JSONB, we will have to *detoast* the whole JSONB [^1]

[^1]: See this slides from Oleg Bartunov and Nikita Glukhov : [json or not json that is the question](http://www.sai.msu.su/~megera/postgres/talks/jsonb-nizhny-2021.pdf)

However, there are a few exceptions where JSON can be useful:

* When we don't need to search multiple fields and retrieve the json via another column. (we do not need statistics on json's keys).
* And, when it would be very difficult to fit the json into a relational schema. Some cases would involve having a lot of columns and most of them at `NULL`.

For example, to store product features where a normalized version would imply the use of a lot of columns, most of which would be `NULL`.
Let's say you are storing products, a television would have specific characteristics (screen type, size etc).
A washing machine would also have other specific characteristics (spin speed, weight accepted...).

We could thus consider having "normal" columns including the model, its price, its reference etc, and a column containing all the characteristics.
We would access the row via the reference and thus we would recover all the characteristics of the product stored in the json.

I will reuse the Stackoverflow posts table by moving some columns into a jsonb column (*jsonfield* column in this example):

```
\d posts
                            Unlogged table "public.posts"
        Column         |            Type             | Collation | Nullable | Default
-----------------------+-----------------------------+-----------+----------+---------
 id                    | integer                     |           | not null |
 posttypeid            | integer                     |           | not null |
 acceptedanswerid      | integer                     |           |          |
 parentid              | integer                     |           |          |
 creationdate          | timestamp without time zone |           | not null |
 score                 | integer                     |           |          |
 viewcount             | integer                     |           |          |
 body                  | text                        |           |          |
 owneruserid           | integer                     |           |          |
 lasteditoruserid      | integer                     |           |          |
 lasteditordisplayname | text                        |           |          |
 lasteditdate          | timestamp without time zone |           |          |
 lastactivitydate      | timestamp without time zone |           |          |
 title                 | text                        |           |          |
 tags                  | text                        |           |          |
 answercount           | integer                     |           |          |
 commentcount          | integer                     |           |          |
 favoritecount         | integer                     |           |          |
 closeddate            | timestamp without time zone |           |          |
 communityowneddate    | timestamp without time zone |           |          |
 jsonfield             | jsonb                       |           |          |
```

Here is a simple aggregate:

```sql
SELECT
  avg(viewcount),
  avg(answercount),
  avg(commentcount),
  avg(favoritecount)
FROM posts;
```


```
                                                          QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=10265135.77..10265135.78 rows=1 width=128) (actual time=170221.557..170221.558 rows=1 loops=1)
   Buffers: shared hit=1 read=9186137
   I/O Timings: read=138022.290
   ->  Seq Scan on posts  (cost=0.00..9725636.88 rows=53949888 width=16) (actual time=0.014..153665.913 rows=53949886 loops=1)
         Buffers: shared hit=1 read=9186137
         I/O Timings: read=138022.290
 Planning Time: 0.240 ms
 Execution Time: 170221.627 ms
(8 rows)
```

The query reads 70 GB and takes about 2min 50s to execute.

Now the same query, but this time using the keys present in the json.


```sql
SELECT
  avg((jsonfield ->> 'ViewCount')::int),
  avg((jsonfield ->> 'AnswerCount')::int),
  avg((jsonfield ->> 'CommentCount')::int),
  avg((jsonfield ->> 'FavoriteCount')::int)
FROM posts;
```


```
                           QUERY PLAN
------------------------------------------------------------------------------
 Aggregate  (cost=11883632.41..11883632.42 rows=1 width=128)
            (actual time=520917.028..520917.030 rows=1 loops=1)
   Buffers: shared hit=241116554 read=13625756
   ->  Seq Scan on posts  (cost=0.00..9725636.88 rows=53949888 width=570)
                          (actual time=0.972..70569.365 rows=53949886 loops=1)
         Buffers: shared read=9186138
 Planning Time: 0.118 ms
 Execution Time: 520945.395 ms
(10 rows)
```

The query takes about 8min 40s to execute. However, the number of blocks read seems a bit crazy:

The Seq Scan indicates 70Gb as before. But, the parent node indicates more than 1.9 TB read!

Here is the size of the table with the default setting.
You should know that for some records, Postgres will either compress the row in the heap (inline compression)
or compress it and place it in the toast.

```
SELECT
  pg_size_pretty(pg_relation_size(oid)) table_size,
  pg_size_pretty(pg_relation_size(reltoastrelid)) toast_size
FROM pg_class
WHERE relname = 'posts';

 table_size | toast_size
------------+-----------
 70 GB      | 33 GB
(1 row)
```

How to explain the 1.9 TB read?

By curiosity, I made the same query, but with a single aggregation and I get about 538 GB.

There are several questions:

1. How do I know if Postgres will read the toast?
2. Why such a difference in execution time between the "standard column" version and jsonb field?
3. What do counters in the `Aggregate` node correspond to?

To answer the first question, just read the `pg_statio_user_tables` view.

Before executing the query :

```
select relid,schemaname,relname,heap_blks_read,heap_blks_hit,toast_blks_read,toast_blks_hit from pg_statio_all_tables where relname in ('posts','pg_toast_26180851');
  relid   | schemaname |      relname      | heap_blks_read | heap_blks_hit | toast_blks_read | toast_blks_hit
----------+------------+-------------------+----------------+---------------+-----------------+----------------
 26180851 | public     | posts             |      422018238 |      87673549 |       129785076 |      628153337
 26180854 | pg_toast   | pg_toast_26180851 |      129785076 |     628153337 |                 |
(2 rows)
```

After :

```
  relid   | schemaname |      relname      | heap_blks_read | heap_blks_hit | toast_blks_read | toast_blks_hit
----------+------------+-------------------+----------------+---------------+-----------------+----------------
 26180851 | public     | posts             |      431204376 |      87673549 |       134156898 |      686299551
 26180854 | pg_toast   | pg_toast_26180851 |      134156898 |     686299551 |                 |
(2 rows)
```

Which give us :

```sql
SELECT
pg_size_pretty(
    ((431204376 + 87673549) - (422018238 + 87673549) ) * 8*1024::bigint
) heap_buffers,
pg_size_pretty(
    ((134156898 + 686299551) - (129785076 + 628153337) ) * 8*1024::bigint
) toast_buffers;

 heap_buffers | toast_buffers
--------------+---------------
 70 GB        | 477 GB
(1 row)
```


Postgres reads the toast. However, counters suggest that Postgres will read the toast several times.

If I do the same calculation, but this time aggregating only on one field, I get 119 GB (~ 477 GB / 4)
I guess Postgres reads the toast for each function.

Then, the difference in execution time is due to several reasons:

* Postgres will have to read and *detoast* (decompress) the toast
* Doing additional operations on the jsonb to access the value

With the first query, Postgres did not have to read the toast. On one hand, it has less data to read, on the other hand,
it does not have to manipulate the json to identify the key and extract the value to be calculated.

Finally, counters of the aggregate node must correspond to the decompressed data for each function that will read the json.
Indeed, if we take the aggregate minus the *seqscan* of the table, so that the *toast* part, we have:

* 468 GB for a single field
* 936 GB, double for two fields
* 1873 GB for the 4 fields (so about 4 x 468 GB)

This explains why the value is so high.


# Advanced settings

Now, we will encourage Postgres to put the maximum amount of data in the toast thanks to the *toast_tuple_target* option that was introduced with Postgres version 11.

This option allows you to control the threshold at which data is stored in the *toast*.

Moreover, being under Postgres 14, I took the opportunity to use the lz4 compression algorithm (parameter *default_toast_compression*).
This algorithm offers a similar compression ratio to pglz, but it is much faster (See [What is the new LZ4 TOAST compression in PostgreSQL 14, and how fast is it?](https://www.postgresql.fastware.com/blog/what-is-the-new-lz4-toast-compression-in-postgresql-14)).


```sql
CREATE TABLE posts_toast
  WITH (toast_tuple_target = 128) AS
    SELECT *
    FROM posts;
```

Here is the size of both table and toast table:

```
SELECT
  pg_size_pretty(pg_relation_size(oid)) table_size,
  pg_size_pretty(pg_relation_size(reltoastrelid)) toast_size
FROM pg_class
WHERE relname = 'posts_toast';

 table_size | toast_size
------------+------------
 59 GB      | 52 GB
```

In total, the table with the toast is roughly the same size.
In the example with the first table, you should remember that the engine also compresses the data in the heap.

Let's play our aggregation query again:


```sql
SELECT
  avg(viewcount),
  avg(answercount),
  avg(commentcount),
  avg(favoritecount)
FROM posts_toast;
```

This time the query reads 59 GB and takes 2min 17 seconds.
We have saved about 20% of execution time on this example.

We could save a lot more if the part stored in toast was bigger. The volume of data to read in the heap would be much smaller.

By curiosity, I also executed the query that aggregates the data from the json field. I get an execution time of 7min 17s.

# Conclusion

Summary in a few numbers:

* Standard aggregation, standard storage: 2min 50s
* Aggregation type jsonb, standard storage: 8min 40s
* Standard aggregation, storage with *toast_tuple_target* = 128 : 2min 17s
* Aggregation type jsonb, storage with *toast_tuple_target* = 128 : 7min 17s

We can see that using JSON is much more expensive than using standard types.
Postgres has to do more operations to access the value of a json key.

Moreover, it has to decompress the data in the toast to access it. However, we can also play with the `toast_tuple_target` parameter to push more information in the toast.
Thus, in some cases, this can reduce the amount of data read by avoiding reading the toast.

# Bonus

As usual in Postgres, everything evolves with each version. TOAST is not escaping this rule.
Thus, some new features could appear in the next versions:

1. A first patch has been proposed to have more statistics on the toast : [pg_stat_toast](https://commitfest.postgresql.org/37/3457/).
The idea is to have statistics on compression: compression gain, inline or separate storage in the toast...
2. A second patch called [Pluggable toaster](https://commitfest.postgresql.org/37/3490/). This one is much more important. It suggests extending the *"toaster "*.
The idea would be to be able to propose different *"toaster "* depending on the type (especially JSONB).
