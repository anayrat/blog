---
title: Index BRIN – Corrélation
author: Adrien Nayrat
type: post
date: 2016-04-20T20:57:31+00:00
aliases: /2016/04/20/index-brin-correlation/
categories:
  - Linux
  - Postgres
tags:
  - brin
  - index
  - postgres

---
La version 9.5 de PostgreSQL sortie en Janvier 2016 propose un nouveau type d'index : les Index BRIN pour Bloc Range INdex. Ces derniers sont recommandés pour les tables volumineuses et corrélées avec leur emplacement. J'ai décidé de consacrer une série d'article sur ces index :

* [Index BRIN - Principe][1]
* [Index BRIN - Fonctionnement][2]
* [Index BRIN - Corrélation][3]
* [Index BRIN - Performances][4]

Pour information, je serai présent au [PGDay France][5] à Lille le mardi 31 mai pour présenter cet index. Il y aura également plein d'autres [conférences intéressantes][6]!

Dans ce troisième article nous verrons pourquoi la corrélation des données avec leur emplacement est importante pour les index BRIN.

<!--more-->

# Données corrélées

Ces premiers exemples étaient volontairement simples voire simplistes afin de faciliter la compréhension. Les enregistrements avaient une particularité importante : les valeurs étaient croissantes.

Ce qui signifie qu'il y a une forte corrélation entre les enregistrements et leur emplacement.

Pour information, le moteur stocke des statistiques sur les tables afin de choisir le meilleur plan d'exécution. Lançons un ANALYSE sur notre table brin\_demo et consultons la vue pg\_stats, notamment la colonne « correlation » :

```SQL
SELECT tablename,attname,correlation from pg_stats where tablename='brin_demo';
 tablename | attname | correlation
-----------+---------+-------------
 brin_demo | c1 | 1
 ```

Essayons avec des données aléatoires :

```SQL
CREATE TABLE brin_random (c1 INT);
INSERT INTO brin_random SELECT trunc(random() * 90 + 1) AS i FROM generate_series(1,100000);
CREATE INDEX brin_random_brin_idx_16 ON brin_random USING brin (c1) WITH (pages_per_range = 16);
ANALYZE brin_random;
SELECT tablename,attname,correlation from pg_stats where tablename='brin_random';
 tablename | attname | correlation
-------------+---------+-------------
 brin_random | c1 | 0.0063248
 ```

=> La vue pg_stats nous indique que les données ne sont pas corrélées avec leur emplacement.

Que contient notre index?

```SQL
SELECT * FROM brin_page_items(get_raw_page('brin_random_brin_idx_16', 2), 'brin_random_brin_idx_16');
 itemoffset | blknum | attnum | allnulls | hasnulls | placeholder | value
------------+--------+--------+----------+----------+-------------+-----------
 1 | 0 | 1 | f | f | f | {1 .. 90}
 2 | 16 | 1 | f | f | f | {1 .. 90}
 3 | 32 | 1 | f | f | f | {1 .. 90}
 4 | 48 | 1 | f | f | f | {1 .. 90}
 5 | 64 | 1 | f | f | f | {1 .. 90}
 6 | 80 | 1 | f | f | f | {1 .. 90}
 7 | 96 | 1 | f | f | f | {1 .. 90}
 8 | 112 | 1 | f | f | f | {1 .. 90}
 9 | 128 | 1 | f | f | f | {1 .. 90}
 10 | 144 | 1 | f | f | f | {1 .. 90}
 11 | 160 | 1 | f | f | f | {1 .. 90}
 12 | 176 | 1 | f | f | f | {1 .. 90}
 13 | 192 | 1 | f | f | f | {1 .. 90}
 14 | 208 | 1 | f | f | f | {1 .. 90}
 15 | 224 | 1 | f | f | f | {1 .. 90}
 16 | 240 | 1 | f | f | f | {1 .. 90}
 17 | 256 | 1 | f | f | f | {1 .. 90}
 18 | 272 | 1 | f | f | f | {1 .. 90}
 19 | 288 | 1 | f | f | f | {1 .. 90}
 20 | 304 | 1 | f | f | f | {1 .. 90}
 21 | 320 | 1 | f | f | f | {1 .. 90}
 22 | 336 | 1 | f | f | f | {1 .. 90}
 23 | 352 | 1 | f | f | f | {1 .. 90}
 24 | 368 | 1 | f | f | f | {1 .. 90}
 25 | 384 | 1 | f | f | f | {1 .. 90}
 26 | 400 | 1 | f | f | f | {1 .. 90}
 27 | 416 | 1 | f | f | f | {1 .. 90}
 28 | 432 | 1 | f | f | f | {1 .. 90}
(28 lignes)
```

L'index nous indique que tous les blocs contiennent des valeurs comprises entre 1 et 90.

Que donne une recherche des valeurs comprises entre 10 et 20?

```SQL
EXPLAIN (ANALYZE,BUFFERS,VERBOSE) SELECT c1 FROM brin_random WHERE c1> 10 AND c1<20;
 QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on public.brin_random (cost=118.46..717.27 rows=10387 width=4) (actual time=0.068..10.241 rows=10240 loops=1)
 Output: c1
 Recheck Cond: ((brin_random.c1 > 10) AND (brin_random.c1 < 20))
 Rows Removed by Index Recheck: 89760
 Heap Blocks: lossy=443
 Buffers: shared hit=445
 -> Bitmap Index Scan on brin_random_brin_idx_16 (cost=0.00..115.87 rows=10387 width=0) (actual time=0.052..0.052 rows=4480 loops=1)
 Index Cond: ((brin_random.c1 > 10) AND (brin_random.c1 < 20))
 Buffers: shared hit=2
 ```

=> Le moteur lit l'intégralité de la table ainsi que 2 blocs d'index.

Il est légitime de se demander quel est l'intérêt d'utiliser l'index si on sait que les données ne sont pas corrélées. Et qu'au final le moteur sera contraint de lire toute la table.

Allons faire un tour dans le code source. [Plus précisément à la ligne 7568 du fichier src/backend/utils/adt/selfuncs.c](http://doxygen.postgresql.org/index__selfuncs_8h.html#aa732367fc3b041ae0a0c5a377e2b1027) :

```c
/*
 * BRIN indexes are always read in full; use that as startup cost.
 *
 * XXX maybe only include revmap pages here?
 */
 *indexStartupCost = spc_seq_page_cost * numPages * loop_count;

 /*
 * To read a BRIN index there might be a bit of back and forth over
 * regular pages, as revmap might point to them out of sequential order;
 * calculate this as reading the whole index in random order.
 */
 *indexTotalCost = spc_random_page_cost * numPages * loop_count;

 *indexSelectivity =
 clauselist_selectivity(root, indexQuals,
 path->indexinfo->rel->relid,
 JOIN_INNER, NULL);
 *indexCorrelation = 1;

 /*
 * Add on index qual eval costs, much as in genericcostestimate.
 */
 qual_arg_cost = other_operands_eval_cost(root, qinfos) +
 orderby_operands_eval_cost(root, path);
 qual_op_cost = cpu_operator_cost *
 (list_length(indexQuals) + list_length(indexOrderBys));

 *indexStartupCost += qual_arg_cost;
 *indexTotalCost += qual_arg_cost;
 *indexTotalCost += (numTuples * *indexSelectivity) * (cpu_index_tuple_cost + qual_op_cost);
 ```

On peut voir « *indexCorrelation = 1; ». En réalité le moteur ignore la corrélation... pour le moment. Une discussion est en cours pour prendre en compte la corrélation dans le coût de l'index : <http://www.postgresql.org/message-id/20151116135239.GV614468@alvherre.pgsql>

Trions notre table en utilisant l'ordre CLUSTER et un index b-tree :

```SQL
CREATE INDEX brin_random_btree_idx ON brin_random USING btree (c1);
CLUSTER brin_random USING brin_random_btree_idx;
DROP INDEX brin_random_btree_idx ;
ANALYZE brin_random ;
```

Vérifions la corrélation :

```SQL
SELECT tablename,correlation FROM pg_stats WHERE tablename='brin_random';
 tablename | correlation
-------------+-------------
 brin_random | 1
(1 ligne)
```

Rejouons notre requête :

```SQL
EXPLAIN (ANALYZE,BUFFERS,VERBOSE) SELECT c1 FROM brin_random WHERE c1> 10 AND c1<20;
 QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on public.brin_random (cost=115.08..708.94 rows=10057 width=4) (actual time=0.113..3.166 rows=9999 loops=1)
 Output: c1
 Recheck Cond: ((brin_random.c1 > 10) AND (brin_random.c1 < 20))
 Rows Removed by Index Recheck: 849
 Heap Blocks: lossy=48
 Buffers: shared hit=50
 -> Bitmap Index Scan on brin_random_brin_idx_16 (cost=0.00..112.57 rows=10057 width=0) (actual time=0.053..0.053 rows=480 loops=1)
 Index Cond: ((brin_random.c1 > 10) AND (brin_random.c1 < 20))
 Buffers: shared hit=2
 Planning time: 0.086 ms
 Execution time: 3.849 ms
 ```

Cette fois le moteur a lu moins de blocs, regardons ce que contient notre index :

```SQL
SELECT * FROM brin_page_items(get_raw_page('brin_random_brin_idx_16', 2), 'brin_random_brin_idx_16');
 itemoffset | blknum | attnum | allnulls | hasnulls | placeholder | value
------------+--------+--------+----------+----------+-------------+------------
 1 | 0 | 1 | f | f | f | {1 .. 4}
 2 | 16 | 1 | f | f | f | {4 .. 7}
 3 | 32 | 1 | f | f | f | {7 .. 10}
 4 | 48 | 1 | f | f | f | {10 .. 14}
 5 | 64 | 1 | f | f | f | {14 .. 17}
 6 | 80 | 1 | f | f | f | {17 .. 20}
 7 | 96 | 1 | f | f | f | {20 .. 23}
 8 | 112 | 1 | f | f | f | {23 .. 27}
 9 | 128 | 1 | f | f | f | {27 .. 30}
 10 | 144 | 1 | f | f | f | {30 .. 33}
 11 | 160 | 1 | f | f | f | {33 .. 36}
 12 | 176 | 1 | f | f | f | {36 .. 39}
 13 | 192 | 1 | f | f | f | {39 .. 43}
 14 | 208 | 1 | f | f | f | {43 .. 46}
 15 | 224 | 1 | f | f | f | {46 .. 49}
 16 | 240 | 1 | f | f | f | {49 .. 52}
 17 | 256 | 1 | f | f | f | {52 .. 56}
 18 | 272 | 1 | f | f | f | {56 .. 59}
 19 | 288 | 1 | f | f | f | {59 .. 62}
 20 | 304 | 1 | f | f | f | {62 .. 65}
 21 | 320 | 1 | f | f | f | {65 .. 69}
 22 | 336 | 1 | f | f | f | {69 .. 72}
 23 | 352 | 1 | f | f | f | {72 .. 75}
 24 | 368 | 1 | f | f | f | {75 .. 78}
 25 | 384 | 1 | f | f | f | {78 .. 82}
 26 | 400 | 1 | f | f | f | {82 .. 85}
 27 | 416 | 1 | f | f | f | {85 .. 88}
 28 | 432 | 1 | f | f | f | {88 .. 90}
 ```

Cette fois le moteur savait qu'il n'avait à parcourir que les blocs de 48 à 96.

[1]: http://blog.anayrat.info/2016/04/19/index-brin-principe/
[2]: http://blog.anayrat.info/2016/04/20/index-brin-fonctionnement/
[3]: http://blog.anayrat.info/2016/04/20/index-brin-correlation/
[4]: http://blog.anayrat.info/2016/04/21/index-brin-performances/
[5]: http://www.pgday.fr/index.html
[6]: http://www.pgday.fr/programme.html
