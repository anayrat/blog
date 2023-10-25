+++
title = "Cas d'usages du partitionnement natif dans PostgreSQL"
date = 2021-09-01T09:00:00+02:00
draft = false

summary = "Différents cas d'usages du partitionnement natif sous PostgreSQL"
authors = ['adrien']

# Tags and categories
# For example, use `tags = []` for no tags, or the form `tags = ["A Tag", "Another Tag"]` for one or more tags.
tags = ["postgres","partitionnement"]
categories = ["Postgres"]
show_related = true

+++

Après une période d'inactivité, je reprends l'écriture d'articles techniques sur Postgres. C'est aussi pour moi l'occasion de vous annoncer mon changement d'activité. Depuis courant 2021 je suis passé freelance pour permettre aux entreprises de bénéficier de mon expérience sur Postgres.

{{% toc %}}

# Histoire du partitionnement dans PostgreSQL

PostgreSQL permet depuis très longtemps de partitionner des tables en exploitant l'héritage de table. Toutefois, cette méthode était assez lourde à mettre en oeuvre : elle impliquait de mettre en place soi-même des triggers pour rediriger les écritures (moins performant que le partitionnement natif), le temps de planification pouvait augmenter fortement au-delà d'une centaine de partitions...


Le partitionnement natif est arrivé avec la version 10. C'est depuis cette version que le moteur est capable (entre autres) de diriger lui-même les écritures vers les bonnes tables, lire seulement les tables concernées, d'utiliser des algorithmes exploitant le partitionnement etc.
Il offre ainsi de meilleures performances et une facilité d'exploitation. On peut entre autres :

  * Partitionner :
    * Par liste
    * Par hashage
    * Par intervalles
  * Faire des partitionnements à plusieurs niveaux
  * Partitionner sur plusieurs colonnes
  * Utiliser des clés primaires et clés étrangères

Toutes ces fonctionnalités sont intéressantes, mais on en vient à se poser une question toute bête : quand mettre en oeuvre le partitionnement?

Je vais vous présenter plusieurs cas d'usages que j'ai pu rencontrer. Mais avant, voici quelques erreurs courantes sur le partitionnement.


# Erreurs courantes

## "Il faut partitionner dès que la volumétrie est importante"

Déjà, qu'est-ce qu'une volumétrie "importante"?

Certains diront que c'est au-delà de plusieurs centaines de Go, d'autres au-delà du téraoctet, d'autres encore au-delà du pétaoctet...

Il n'existe pas vraiment de réponse à cette question et globalement ça va dépendre du type d'activité : ratio INSERT/UPDATE/DELETE, type de SELECT (OLTP, OLAP...).
Ca dépendra également du matériel. Il y a 10 ans, quand les serveurs n'avaient que quelques Go de RAM avec des disques mécaniques, il était probable qu'une base de quelques centaines de Go soit perçue comme une grosse base.
Maintenant il n'est pas rare de voir des serveurs avec plus d'un téraoctet de RAM, des disques NVMe.

Ainsi, une base de quelques centaines de Go n'est plus considérée comme une grosse base. Mais plutôt comme une base de taille modeste.

Petite anecdote, pour se rassurer, un client m'a questionné si Postgres était déjà utilisé pour des volumétries "importantes". On parlait alors d'une base d'une quarantaine de Go sur un serveur qui disposait de 64Go de RAM. Toutes les lectures se faisaient depuis le cache... :). J'ai pu le rassurer sur la taille de sa base qui était relativement modeste.

Il peut tout à fait être superflu de partitionner une base de quelques To comme il peut être nécessaire de partitionner une base de quelques centaines de Go. Par exemple, si l'activité consiste juste à ajouter des lignes à des tables et que les requêtes se résument à de simple `WHERE colonne = 4` qui retournent quelques lignes. Un simple Btree fera l'affaire. Et si la requête retourne un nombre assez important de lignes, il est possible d'utiliser les index BRIN ou les bloom filter.

Les index BRIN présentent des bénéfices proches du partitionnement ou sharding en évitant la complexité de mise en oeuvre[^1].


## "Il faut partitionner pour répartir les données sur plusieurs disques"

L'idée serait de créer des partitions et des tablespaces sur différents disques afin de répartir les opérations d'entrées/sorties.

Pour PostgreSQL, un tablespace n'est ni plus, ni moins qu'un chemin vers un répertoire. Il est tout à fait possible
de gérer le stockage au niveau du système d'exploitation et d'agréger plusieurs disques (en RAID10) par exemple.
Ensuite, il suffit de stocker la table sur le volume créé. Ainsi, on peut répartir les I/O sur un ensemble de disques.

Dans ce cas, il n'est donc pas nécessaire de mettre en oeuvre le partitionnement. Toutefois, nous verrons un cas où il pourrait avoir du sens.

Maintenant nous allons nous intéresser à des cas d'usage "légitimes" du partitionnement.

# Cas d'usages du partitionnement

## Partitionner pour gérer la rétention

A cause du modèle MVCC, la suppression massive de données entraine de la fragmentation dans les tables.

Un cas d'usage possible est de partitionner par date. Supprimer les anciennes données revient à supprimer une partition complète. L'opération sera rapide et les tables ne seront pas fragmentées


## Partitionner pour contrôler la fragmentation des index

L'ajout et modification de données dans une table fragmente les index au fil du temps. Pour faire simple, on ne peut pas récupérer l'espace libre dans un bloc tant qu'il n'est pas vide. Avec le temps les splits d'index créent du "vide" dans ce dernier et le seul moyen de récupérer cet espace est de reconstruire l'index.

On appelle cela le "bloat". Il y a eu de nombreuses améliorations sur les dernières versions de Postgres:

* Version 12, on peut lire dans les [Releases Notes](https://www.postgresql.org/docs/12/release-12.html):
> Improve performance and space utilization of btree indexes with many duplicates (Peter Geoghegan, Heikki Linnakangas)
>
> Previously, duplicate index entries were stored unordered within their duplicate groups. This caused overhead during index inserts, wasted space due to excessive page splits, and it reduced VACUUM's ability to recycle entire pages. Duplicate index entries are now sorted in heap-storage order.


* Version 13, on peut lire dans les [Releases Notes](https://www.postgresql.org/docs/13/release-13.html):
> More efficiently store duplicates in B-tree indexes (Anastasia Lubennikova, Peter Geoghegan)
>
> This allows efficient B-tree indexing of low-cardinality columns by storing duplicate keys only once. Users upgrading with pg_upgrade will need to use REINDEX to make an existing index use this feature.


* Version 14, on peut lire dans les [Releases Notes](https://www.postgresql.org/docs/14/release-14.html):
> Allow btree index additions to remove expired index entries to prevent page splits (Peter Geoghegan)
>
> This is particularly helpful for reducing index bloat on tables whose indexed columns are frequently updated.


Pour contrôler le bloat, on pourrait reconstruire l'index à intervalles réguliers (merci `REINDEX CONCURRENTLY` arrivé en version 12). Cette solution serait contraignante, car il faudrait régulièrement reconstruire l'intégralité de l'index.

Si la majorité des modifications sont faites sur les données récentes, par exemple: table de logs, commandes clients, rendez-vous... On pourrait imaginer un partitionnement par mois. Ainsi, à chaque début de mois on part sur une table "neuve" et on peut ré-indexer la précédente table pour supprimer le bloat.

On peut aussi en profiter pour faire un `CLUSTER` sur la table pour avoir une bonne corrélation des données avec le stockage.

## Partitionner pour faciliter l'exécution de requête lorsque la cardinalité est faible

Petit à petit on va voir des cas d'usages un peu plus compliqués :)

Prenons un exemple : une table de commande comprenant un statut de livraison, au bout de quelques années 99% des commandes sont livrées (on l'espère!) et très peu en cours de paiement ou livraison.

Imaginons qu'on souhaite récupérer 100 commandes en cours de livraison. On va créer un index sur le statut et l'utiliser pour récupérer les enregistrements.
En étant un peu astucieux, on peut créer un index partiel sur ce statut particulier. Problème, cet index va se fragmenter assez vite au fur et à mesure que les commandes seront livrées.

Dans ce cas on pourrait faire un partitionnement sur le statut. Ainsi, récupérer 100 commandes en cours de livraison revient à lire 100 enregistrements de la partition.

## Partitionner pour obtenir de meilleures statistiques

Pour déterminer le meilleur plan d'exécution, Postgres prend des décisions à partir des statistiques d'une table. Ces statistiques sont obtenues à partir d'un échantillon de la table (le `default_statistic_target` qui vaut 100 par défaut).

Par défaut le moteur va collecter 300 x `default_statistic_target` lignes, soit 30 000 lignes. Avec une table de plusieurs centaines de millions de lignes, cet échantillon est parfois trop petit.

On peut augmenter de manière drastique la taille de l'échantillon, mais cette approche présente quelques inconvénients:

* Ca alourdis le temps de planification
* Ca alourdis le `ANALYZE`
* Parfois ce n'est pas suffisant si les données sont mal réparties. Par exemple si on prend quelques centaines de milliers de lignes sur une table qui comprend plusieurs centaines de millions, on peut rater les lignes dont le statut est en livraison.

Avec le partitionnement on pourrait avoir un même échantillon, mais par partition, ce qui permet de gagner en précision.

Ce serait également utile quand on a des données corrélées entre colonnes. Je vais reprendre l'exemple des commandes. On a une année entière de commandes: toutes les commandes qui ont plus d'un mois sont livrées, celles du dernier mois sont livrées à 90% (10% sont en cours de livraison).

Intuitivement, si je cherche une commande en cours de livraison il y a plus de 6 mois je ne devrais pas avoir de résultat. Inversement, si je cherche des commandes en cours de livraison sur le dernier mois, je devrais obtenir 10% de la table. Or, le moteur ne le sait pas, pour lui les commandes en cours de livraison sont réparties sur toute la table.

Avec un partitionnement par date, il peut estimer qu'il n'y a pas de commande en cours de livraisons de plus d'un mois. Ce type d'approche permet surtout de réduire une erreur d'estimation dans un plan d'exécution.

Voici un exemple avec cette table de commandes, `orders_p` est la version partitionnée par mois de la table `orders`. Les données étant identiques dans les deux tables.

On peut remarquer que l'estimation est bien meilleure dans le cas où la table est partitionnée, le moteur ayant des statistiques par partition.


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


Maintenant prenons la même requête sur le dernier mois:

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

Ici aussi on peut remarquer que l'estimation est meilleure.

## partitionwise join & partitionwise aggregate

Un autre intérêt du partitionnement est de bénéficier de meilleurs algorithmes pour les jointures et agrégation.

Le `partitionwise aggregate` permet de faire une agregation ou un regroupement partition par partition. Un exemple vaut mieux qu'un long discours:


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

Dans le premier cas l'agrégation se fait une fois pour toutes les tables, alors que dans le second exemple, on fait l'agrégation par partition.
On peut également remarquer que le coût total est inférieur dans le plan avec agrégation par partition.

Le `partitionwise join` fonctionne sur le même principe, on fait une jointure partition par partition. C'est utile pour joindre deux tables partitionnées.


## Stockage avec tiering

Enfin, un autre cas d'usage serait de vouloir stocker une partie de la table sur un stockage différent:

On peut stocker une table partitionnée dans des tablespaces différents. Par exemple les données récentes sur un tablespace rapide sur SSD NVMe.
Puis les données plus rarement accédées sur un autre tablespace, avec des disques mécaniques moins couteux.

Cette approche peut aussi avoir du sens à l'heure du cloud où le stockage est très onéreux.

# Conclusion

Voilà, je pense avoir fait le tour des principaux cas d'usages qui me venaient en tête.

Evidemment, la mise en oeuvre du partitionnement implique une plus grande complexité (gestion des partitions...)
et des limitations qu'il faudra étudier en amont.

[^1]: "BRIN indexes provide similar benefits to horizontal partitioning or sharding but without needing to explicitly declare partitions." - <https://en.wikipedia.org/wiki/Block_Range_Index>

