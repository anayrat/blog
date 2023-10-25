+++
title = "PostgreSQL et updates heap-only-tuples - partie 3"
date = 2018-11-26T08:00:00+01:00
draft = false
summary = "Impact sur les performances"

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

1. [Fonctionnement du MVCC et update *heap-only-tuples*][1]
2. [Quand le moteur ne fait pas d'update *heap-only-tuple* et présentation de la nouveauté de la version 11][2]
3. [Impact sur les performances][3]

**Cette fonctionnalité a été désactivée en 11.1 car elle pouvait conduire à des
crash d'instance[^4]. J'ai tout de même choisi de publier ces articles car ils permettent
de comprendre le mécanisme des updates HOT et le gain que pourrait apporter cette
fonctionnalité.**

Je remercie au passage Guillaume Lelarge pour la relecture de cet article ;).


# Impact sur les performances

Voici un test assez simple pour mettre en évidence l'intérêt de cette fonctionnalité.
On pourrait s'attendre à des gains en performances car le moteur évite de mettre
à jour les index, ainsi qu'en matière de taille d'index, comme vu précédemment,
on évite la fragmentation.

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

Puis ce test pgbench :

```sql
\set id  random(1, 100000)
\set id2  random(1, 100000)

UPDATE t5 SET c1 = '{"valeur": ":id", "prenom": "guillaume"}' WHERE c2=2;
UPDATE t5 SET c1 = '{"valeur": ":id2", "prenom": "adrien"}' WHERE c2=1;
```

Qu'on exécute pendant 60 secondes, avec `recheck_on_update=on` (par défaut) :

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

SELECT * FROM pg_stat_user_tables WHERE relname = 't5';
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

Et maintenant avec `recheck_on_update=off`. Donc même jeu de donnée que
précédemment mais cette fois l'index est créé avec cet ordre :
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

L'écart de performance est assez impressionnant de même que la taille des tables
et index.

J'ai refait le premier test en désactivant l'autovacuum et voici le résultat :

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

Puis le second test :

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

A nouveau, l'écart de performance est important, il en est de même pour la taille
des tables et index. On note également l'importance de laisser l'autovacuum activé.

Pourquoi avons-nous un tel écart de taille sur les index et la table ?

Pour les index, c'est dû au mécanisme expliqué plus haut. Le moteur a pu chaîner
les enregistrements en évitant de mettre à jour l'index. L'index a quand même
légèrement augmenté de taille, il arrive que le moteur ne peut pas faire de HOT,
par exemple quand il n'y a plus de place dans le bloc.

Pour ce qui est de la taille de la table, lors du test avec autovacuum activé,
l'autovacuum avait plus de difficultés à passer sur la table avec le HOT désactivé.
L'index grossissant, cela engendrait plus de "travail".
Lors du test sans autovacuum, l'écart s'explique par le fait que même un simple
SELECT peut nettoyer des blocs[^5].

Rappelons que cette fonctionnalité a été retirée avec la version 11.1. J'avais écrit
ces articles peu après la sortie de la version 11.0 et j'ai tout de même choisit
de les publier afin de présenter le fonctionnement des UPDATES HOT. Espérons que
cette fonctionnalité sera corrigée dans les versions à venir.

[1]: https://blog.anayrat.info/2018/11/12/postgresql-et-updates-heap-only-tuples-partie-1/
[2]: https://blog.anayrat.info/2018/11/19/postgresql-et-updates-heap-only-tuples-partie-2/
[3]: https://blog.anayrat.info/2018/11/26/postgresql-et-updates-heap-only-tuples-partie-3/

[^4]: [Disable recheck_on_update optimization to avoid crashes](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=05f84605dbeb9cf8279a157234b24bbb706c5256)
[^5]: [README.HOT](https://github.com/postgres/postgres/blob/master/src/backend/access/heap/README.HOT#L251)
