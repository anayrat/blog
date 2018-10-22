+++
title = "PostgreSQL and heap-only-tuples updates - part 3"
date = 2018-04-25T14:09:22+02:00
draft = true
summary = "Fonctionnement des updates heap-only-tuple"

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

Voici une série d'articles qui va porter sur une nouveauté de la version 11.

Durant le développement de cette version, une fonctionnalité a attiré mon attention.
On peut la retrouver dans les releases notes : <https://www.postgresql.org/docs/11/static/release-11.html>


> Allow heap-only-tuple (HOT) updates for expression indexes when the values of the expressions are unchanged (Konstantin Knizhnik)

J'avoue que ce n'est pas très explicite et cette fonctionnalité nécessite quelques
connaissances sur le fonctionnement du moteur que je vais essayer d'expliquer à travers
plusieurs articles :

1. [Fonctionnement du MVCC et update *heap-only-tuples*](toto)
2. [Quand le moteur ne fait pas d'update *heap-only-tuple* et présentation de la nouveauté de la version 11](toto)
3. [Impact sur les performances](toto)

# Impact on performance

Here is a simple test to demonstrate the benefits of this feature.
We could expect performance gains because postgres avoids updating the indexes. As well as in terms of index size, as seen above, fragmentation is avoided.

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

That we execute for 60 seconds:

With `recheck_on_update=on` (default):

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

select * from pg_stat_user_tables where relname = 't5';
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

With `recheck_on_update=off`:

Same data set as before but this time the index is created with this order:
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
FIXME : rappeler les chiffres des deux tests.

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

FIXME : rappeler les chiffres des deux tests.

Once again, the performance gap is significant, as is the size of tables and
indexes. We also note the importance of leaving the autovacuum activated.

Why do we have such a large difference on the indexes and the table?

For the indexes this is due to the mechanism explained above, postgres was
able to chain the records by avoiding updating the index. The index has
nevertheless slightly increased in size, it may happen that postgres cannot make a HOT.
For example, when there is no more space in the block.

As for the size of the table. During the test with autovacuum activated, the
autovacuum had more difficulty to pass on the table with the HOT disabled.
The index growing, it resulted in more "work".
During the test without autovacuum, the difference is explained FIXED: finish explanation



# Cas où l'expression est coûteuse

FIXME : est-ce que je vais aussi loin?

Et dans le cas d'un index fonctionnel? Il est trop coûteux de verifier à chaque mise à jour si l'index doit être mis à jour.
Par exemple, un index fonctionnel qui n'indexe que certaines clés d'un objet JSON :

```SQL
BEGIN;
DROP TABLE IF EXISTS t4;

CREATE TABLE t4 (c1 jsonb, c2 int,c3 int);
-- CREATE INDEX ON t4 ((c1->>'prenom'))  WITH (recheck_on_update='false');
CREATE INDEX ON t4 ((c1->>'prenom')) ;
CREATE INDEX ON t4 (c2);
INSERT INTO t4 VALUES ('{ "prenom":"adrien" , "ville" : "valence"}'::jsonb,1,1);
INSERT INTO t4 VALUES ('{ "prenom":"guillaume" , "ville" : "lille"}'::jsonb,2,2);


-- changement qui ne porte pas sur prenom
UPDATE t4 SET c1 = '{"ville": "valence (#soleil)", "prenom": "guillaume"}' WHERE c2=2;
SELECT pg_stat_get_xact_tuples_hot_updated('t4'::regclass);
UPDATE t4 SET c1 = '{"ville": "nantes", "prenom": "guillaume"}' WHERE c2=2;
SELECT pg_stat_get_xact_tuples_hot_updated('t4'::regclass);

ROLLBACK;

\timing
CREATE OR REPLACE FUNCTION upper_ville_field(obj jsonb)
RETURNS jsonb
AS $$
SELECT pg_sleep(5);
SELECT jsonb_set($1, '{"prenom"}',a::jsonb, true) FROM (SELECT to_jsonb(upper($1->>'prenom'))) a(a);
$$
LANGUAGE SQL IMMUTABLE COST 1100;

BEGIN;
CREATE TABLE t4 (c1 jsonb, c2 int,c3 int);
CREATE INDEX ON t4 ((upper_ville_field(c1)->>'prenom')) ;
-- CREATE INDEX ON t4 ((upper_ville_field(c1)->>'prenom')) WITH (recheck_on_update=œ'true');;

CREATE INDEX ON t4 (c2);
INSERT INTO t4 VALUES ('{ "prenom":"adrien" , "ville" : "valence"}'::jsonb,1,1);
INSERT INTO t4 VALUES ('{ "prenom":"guillaume" , "ville" : "lille"}'::jsonb,2,2);
SELECT * FROM t4;

-- changement qui ne porte pas sur prenom
UPDATE t4 SET c1 = '{"ville": "valence (#soleil)", "prenom": "guillaume"}' WHERE c2=2;
SELECT * FROM t4;
SELECT pg_stat_get_xact_tuples_hot_updated('t4'::regclass);
UPDATE t4 SET c1 = '{"ville": "nantes", "prenom": "guillaume"}' WHERE c2=2;
SELECT pg_stat_get_xact_tuples_hot_updated('t4'::regclass);


-- changement qui ne porte pas sur prenom
UPDATE t4 SET c1 = '{"ville": "lillgegre", "prenom": "guillaume"}' WHERE c2=2;
```
