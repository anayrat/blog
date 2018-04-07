+++
title = "Les évolutions de PostgreSQL pour le traitement des fortes volumétries"
date = 2018-03-27T09:11:00+02:00
draft = true
summary = "Depuis quelques années PostgreSQL a vue de nombreuses améliorations pour le traitement des grosses volumétries."

# Tags and categories
# For example, use `tags = []` for no tags, or the form `tags = ["A Tag", "Another Tag"]` for one or more tags.
tags = ["postgres","JIT","Index","BRIN","VACUUM","parallélisme" ]
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

# Introduction

Depuis quelques années, PostgreSQL s'est doté de nombreuses améliorations pour le
traitement des grosses volumétries.

Ce premier article va tenter de les lister ces améliorations nous verront qu'elles peuvent être de différents ordres :

  * Parallélisation
  * Amélioration intrinsèque du traitement des requêtes
  * Partitionnement
  * Techniques d'indexation
  * Tâches de maintenance
  * Ordres SQL

# SQL

## TABLESAMPLE (9.5)

L'ordre `TABLESAMPLE` (apparu avec la verison 9.5) permet de faire une requête
sur un échantillon. Cela permet d'avoir un aperçu du résultat. Comme en
statistique, plus l'échantillon sera important, plus le résultat sera proche du
résultat réel.

## GROUPING SETS (9.5)

Toujours avec la version 9.5, PostgreSQL dispose de nouvelles clauses permettant
de faire des aggrégations multiples : `GROUPING SETS`, `ROLLUP`, `CUBE`.

A noter que la version 10 apporte des gains très significatifs grâce à l'amélioration
des fonction de hashage.

## Heritage sur les foreign tables (9.5)

Depuis la version 9.5 il est possible des foreign tables comme tables enfant.
Il est ainsi possible de distribuer les données sur différents serveurs et d'y
accéder depuis une seule instance. Ca s'apparente à du sharding.

# Parallélisation

En effet, PostgreSQL étant multi-processus,
le traitement d'une requête ne se faisait que sur un coeur. On retrouve de nombreux
posts où les utilisateurs se plaignent de ce fonctionnement :

  * [Query parallelization for single connection in Postgres](https://stackoverflow.com/questions/32629988/query-parallelization-for-single-connection-in-postgres?rq=1&utm_medium=organic&utm_source=google_rich_qa&utm_campaign=google_rich_qa)
  * [Any way to use >1 Core in PostgreSQL for a single Connection/Query?](https://stackoverflow.com/questions/18268130/any-way-to-use-1-core-in-postgresql-for-a-single-connection-query)

Depuis la version 9.6, le moteur est capable de mobiliser plusieurs processus
pour le traitemetn d'une requête. Cette avancée majeure a été l'aboutissement de
plusieurs années de travail. Il a fallu mettre en place toute une infrastructure
pour permettre au moteur d'utiliser plusieurs processeurs. Voir cet article de
Robert Haas : [Parallelism Progress](http://rhaas.blogspot.fr/2013/10/parallelism-progress.html)

## Parcours séquentiel (9.6)

Depuis la version 9.6, PostgreSQL est capable d'utiliser plusieurs processus pour
paralléliser l'opération de lecture d'une table. Cette opération correspond au
noued `Parallel Seq Scan`. Voici un exemple de plan d'exécution :

```psql
```

Naïvement on pourrait penser que le moteur partage la table en autant de parties
qu'il y a de worker. Cette approche présenterait plusieurs inconvénients:
  * Il faudrait arrivier à partager la table de manière équitable
  * Rien ne garantie qu'un worker finisse avant les autres (et se retrouve inexploité)
  * Le point le plus important : on aurait des lectures aléatoire plutôt que séquentielles

Le moteur utilise donc un compteur et chaque worker réserve le bloc qu'il va traiter.
Ainsi la lecture se fait de manière séquentielle et tous les workers sont exploités.


## Parcours d'index (10)

La version 10 a permis d'etendre la parallélisation aux parcours d'index.

Ainsi le moteur est maintenant capable d'utiliser plusieurs processus pour ces types de parcours :

  * Index Scan
  * Bitmap heap scan
  * Index Only Scan

## Jointures (9.6)

En plus de la parallisation des parcours séquentiel, la version 9.6 apportait la
possibilité de paralléliser les opérations de jointure aux noeuds suivants:

  * nested-loop
  * hash join

La version 10 a permis d'étendre la parallélisation au noeud de type *merge join.*

Enfin, la version 11 apporte un gros changement avec le *parallel hash join*.
Avec les versions précédentes, chaque worker devait construire sa propre table
de hashage. Il y avait une grosse perte d'efficacité :
  * Plusieurs worker faisaient en réalité la même opération
  * Une même table de hashage existait plusieurs fois en mémoire

Le *parallel hash join* permet aux worker de paralléliser la création de cette
table de la hashage et partager une seule table de hashage.


## Aggrégation (9.6)

Toujours avec la version 9.6, le moteur est capable d'utiliser plusieurs workers
pour réalisée des opérations d'aggrégation (`COUNT`, `SUM`...).

En réalité, chaque worker fait une aggrégation partielle *Partial Aggregate*, puis,
un noeud parent se charge de faire l'aggrégation finale *Finalize Aggregate*.


## Union d'ensembles (11)

La version 11 apporte la possibilité de paralléliser les opérations d'union d'ensemble (noeud *append*).
Par exemple lors de l'utilisation d'un `UNION` ou lorsque des tables sont héritées.

# Méthodes d'accès

On confond souvent index et méthodes d'accès en employant régulièrement le terme "index".

En réalité, au sens large, dans le domaine des bases de données, un index est une méthode d'accès.

## Index BRIN (9.5)

Depuis la version 9.5, PostgreSQL propose un type d'index particulier : BRIN pour Bloc Range INdexes.

Ce type d'index contient le résumé d'un ensemble de blocs. Ils sont donc très compacts et tiennent facilement en mémoire.

Ils sont particulièrement adapté aux fortes volumétries avec des requêtes manipulant
un gros volume de données. Attention, il est très important qu'il y ait une forte corrélation entre les données et leur emplacement.

J'avais présenté le fonctionnement de ce type d'index lors du PGDay France 2016 à Lille : [Index BRIN - Fonctionnement et usages possibles](http://localhost:1313/talk/2016/05/31/index-brin---fonctionnement-et-usages-possibles/)

## BLOOM filters (9.6)

Depuis la version 9.6 il est possible d'utiliser des [filtres de Bloom](https://fr.wikipedia.org/wiki/Filtre_de_Bloom). Sans
rentrer dans les détail, ce type de structure de donnée permet d'affirmer avec
certitude que l'information recherchée ne se trouve pas un ensemble. Inversement,
l'information *peut* (avec une certaine probabilité) se trouver dans un autre ensemble.

L'intérêt des filtres bloom, c'est qu'ils sont très compact et permettent de répondre
à des recherches multi-colonnes à partir d'un seul index.

# Partionnement

La partionnement existait sous forme d'héritage de table. Cependant cette approche
avait l'inconvénient de nécessiter la mise en place de trigger pour router les
écritures dans bonnes tables. L'impact sur les performances était important.

La version 10 intègre une gestion native du partionnement. Ainsi, il n'est plus
nécessaire de mettre en place des triggers. Cela facilite grandement les opérations
de maintenance et les performances sont améliorées.

PostgreSQL supporte le partionnement par :

  * Liste - `LIST` (10)
  * Intervalle - `RANGE` (10)
  * Hashage - `HASH` (11)

La version 11 apporte également une gestion plus facilitée des index. La création
d'un index sur une table partitionnées entraine la création sur toutes les partitions.
De même, il est possible d'avoir un index unique et une clé primaire (à condition
qu'elle porte sur la clé de partitionnement). Il est également possible d'avoir
une clé étrangère sur une table partitionnée.


# Amélioration intrinsèques

## Fonctions de hashage

Les fonctions de hashage ont été améliorées dans la version 10. Ainsi, les opérations
d'aggrégat ()`GROUP BY`, `GROUPING SETS`, `CUBE`...) ansi que les noeuds type *hash join*
bénéficient de ces améliorations. FIXME : a vérifier pour les noeuds.

## Tris avec abbreviated keys (9.5)

## Just-In-time



# Taches de maintenance

## VACUUM FREEZE (9.6)

Avant la version 9.6, un `VACUUM FREEZE` entrainait la lecture de l'intégralité
de la table. Même si des lignes avaient déjà été "freezée". La version 9.6 ajoute
une information supplémentaire dans la *visibility map* afin de savoir si un bloc
a déjà été freezé. Cette information permet au moteur de sauter les blocs déjà freezé.

## vacuum qui évite de parcourir les index (11)

Lors d'une opération de vacuum simple (où le moteur va nettoyer les lignes périmée).
Le moteur était capable de sauter les blocs où il savait qu'il n'y avait aucune ligne
à traiter. Néanmoins il devait quand même parcourir l'index, ce qui peut s'avérer couteux
avec une table volumineuse. La version 11 permet d'éviter de phénomène. FIXME

## Création index parallisée (11)

# Tri

## Abbreviated keys

## Parallel sort

# Pushdown dans les FDW

tri, jointure, aggregation,

# Futur

## pluggable storage (avec compression)

  * stockage colonne
  * compression

## partition pruning


## vectorisation (à creuser)
## FDW et exécution asynchrone (à creuser)
