+++
title = "Les évolutions de PostgreSQL pour le traitement des fortes volumétries"
date = 2018-03-27T09:11:00+02:00
draft = true
summary = "Depuis quelques années PostgreSQL a vue de nombreuses améliorations pour le traitement des grosses volumétries."

# Tags and categories
# For example, use `tags = []` for no tags, or the form `tags = ["A Tag", "Another Tag"]` for one or more tags.
tags = ["postgres","JIT","Index","BRIN","VACUUM","parallélisme", "Bloom" ]
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

Ce premier article va tenter de les lister, nous verront qu'elles peuvent être de différents ordres :

  * Parallélisation
  * Amélioration intrinsèque du traitement des requêtes
  * Partitionnement
  * Méthodes d'accès
  * Tâches de maintenance
  * Ordres SQL

Afin de conserver de la clarté, l'explication de chaque fonctionnalité restera succincte.

Note : cet article a été écrit durant la phase de développement de la version 11.
J'ai intégré des nouveautés de la version 11. Tant que celle-ci n'est pas sortie,
ces nouveautés peuvent être retirées.

# SQL

## TABLESAMPLE (9.5)

L'ordre `TABLESAMPLE` (apparu avec la verison 9.5) permet de faire une requête
sur un échantillon. Cela permet d'avoir un aperçu du résultat. Comme en
statistique, plus l'échantillon sera important, plus le résultat sera proche du
résultat réel.

## GROUPING SETS (9.5)

Toujours avec la version 9.5, PostgreSQL dispose de nouvelles clauses permettant
de faire des agrégations multiples appelé [GROUPING SETS](https://www.postgresql.org/docs/current/static/queries-table-expressions.html#QUERIES-GROUPING-SETS).
Les nouveaux agrégats sont : `GROUPING SETS`, `ROLLUP`, `CUBE`.

Voir cet article de Depesz : [Waiting for 9.5 – Support GROUPING SETS, CUBE and ROLLUP](https://www.depesz.com/2015/05/24/waiting-for-9-5-support-grouping-sets-cube-and-rollup/)

A noter que la version 10 apporte des gains très significatifs grâce à l'amélioration
des fonctions de hashage.


## Heritage sur les foreign tables (9.5)

Depuis la version 9.5 il est possible de declarer des foreign tables comme tables enfant.
Il est ainsi possible de distribuer les données sur différents serveurs et d'y
accéder depuis une seule instance. Cette technique s'apparente à du sharding.

Voir cet article de Michael Paquier : [Postgres 9.5 feature highlight - Scale-out with Foreign Tables now part of Inheritance Trees](http://paquier.xyz/postgresql-2/postgres-9-5-feature-highlight-foreign-table-inheritance/)

# Parallélisation

PostgreSQL étant multi-processus, le traitement d'une requête ne se faisait que
sur un coeur. On retrouve de nombreux posts où les utilisateurs se plaignent de ce fonctionnement :

  * [Query parallelization for single connection in Postgres](https://stackoverflow.com/questions/32629988/query-parallelization-for-single-connection-in-postgres?rq=1&utm_medium=organic&utm_source=google_rich_qa&utm_campaign=google_rich_qa)
  * [Any way to use >1 Core in PostgreSQL for a single Connection/Query?](https://stackoverflow.com/questions/18268130/any-way-to-use-1-core-in-postgresql-for-a-single-connection-query)

Depuis la version 9.6, le moteur est capable de mobiliser plusieurs processus (appelé *workers*)
pour le traitement d'une requête. Cette avancée majeure a été l'aboutissement de
plusieurs années de travail. Il a fallu mettre en place toute une infrastructure
pour permettre au moteur d'utiliser plusieurs processus. Voir cet article de
Robert Haas : [Parallelism Progress](http://rhaas.blogspot.fr/2013/10/parallelism-progress.html)

## Parcours séquentiel (9.6)

Depuis la version 9.6, PostgreSQL est capable d'utiliser plusieurs processus pour
paralléliser l'opération de lecture d'une table. Cette opération correspond au
noeud `Parallel Seq Scan`.


## Parcours d'index (10)

La version 10 a permis d'étendre la parallélisation aux parcours d'index.

Ainsi le moteur est maintenant capable d'utiliser plusieurs processus pour ces types de parcours :

  * Index Scan
  * Bitmap heap scan
  * Index Only Scan

## Jointures (9.6, 10, 11)

En plus de la parallisation des parcours séquentiel, la version 9.6 apportait la
possibilité de paralléliser les opérations de jointure aux noeuds suivants:

  * Nested-loop
  * Hash join

La version 10 a permis d'étendre la parallélisation au noeud de type *merge join*.

Enfin, la version 11 apporte un gros changement avec le *parallel hash join*.
Avec les versions précédentes, chaque worker devait construire sa propre table
de hashage. Il y avait une grosse perte d'efficacité :

  * Plusieurs worker faisaient en réalité la même opération
  * Une même table de hashage existait plusieurs fois en mémoire

Le *parallel hash join* permet aux worker de paralléliser la création de cette
table de la hashage et de partager une seule table de hashage.

L'auteur principal de cette fonctionnalité a écrit un excellent article : [Parallel Hash for PostgreSQL ](https://write-skew.blogspot.fr/2018/01/parallel-hash-for-postgresql.html)

## Agrégation (9.6)

Toujours avec la version 9.6, le moteur est capable d'utiliser plusieurs workers
pour réaliser des opérations d'agrégation (`COUNT`, `SUM`...).

En réalité, chaque worker fait une agrégation partielle *Partial Aggregate*, puis,
un noeud parent se charge de faire l'agrégation finale *Finalize Aggregate*.


## Union d'ensembles (11)

La version 11 apporte la possibilité de paralléliser les opérations d'union d'ensemble (noeud *append*).
Par exemple lors de l'utilisation d'un `UNION` ou lorsque des tables sont héritées.

# Méthodes d'accès

On confond souvent index et méthodes d'accès en employant régulièrement le terme "index".

En réalité, au sens large, dans le domaine des bases de données, un index est une méthode d'accès.

## Index BRIN (9.5)

Depuis la version 9.5, PostgreSQL propose un type d'index particulier : BRIN pour Bloc Range INdexes.

Ce type d'index contient le résumé d'un ensemble de blocs. Ils sont donc très compacts et tiennent facilement en mémoire.

Ils sont particulièrement adaptés aux fortes volumétries avec des requêtes manipulant
un gros volume de données. Attention, il est très important qu'il y ait une forte corrélation entre les données et leur emplacement.

J'ai présenté le fonctionnement de ce type d'index lors du PGDay France 2016 à Lille : [Index BRIN - Fonctionnement et usages possibles](http://blog.anayrat.info/talk/2016/05/31/index-brin---fonctionnement-et-usages-possibles/)

## BLOOM filters (9.6)

Depuis la version 9.6 il est possible d'utiliser des [filtres de Bloom](https://fr.wikipedia.org/wiki/Filtre_de_Bloom). Sans
rentrer dans les détails, ce type de structure de donnée permet d'affirmer avec
certitude que l'information recherchée ne se trouve pas un ensemble. Inversement,
l'information *peut* (avec une certaine probabilité) se trouver dans un autre ensemble.

L'intérêt des filtres bloom, c'est qu'ils sont très compacts et permettent de répondre
à des recherches multi-colonnes à partir d'un seul index.

Voir cet article de Depesz : [Waiting for 9.6 – Bloom index contrib module](https://www.depesz.com/2016/04/11/waiting-for-9-6-bloom-index-contrib-module/)

# Partionnement

La partitionnement existait sous forme d'héritage de table. Cependant cette approche
avait l'inconvénient de nécessiter la mise en place de trigger pour router les
écritures dans les bonnes tables. L'impact sur les performances était important.

La version 10 intègre une gestion native du partitionnement. Ainsi, il n'est plus
nécessaire de mettre en place des triggers. Les opérations de maintenance sont facilitées
et les performances sont améliorées.

## Types de partitionnement (10, 11)

PostgreSQL supporte le partitionnement par :

  * Liste - `LIST` (10)
  * Intervalle - `RANGE` (10)
  * Hashage - `HASH` (11)

Voir ces articles de Depesz :

  * [Waiting for PostgreSQL 10 – Implement table partitioning](https://www.depesz.com/2017/02/06/waiting-for-postgresql-10-implement-table-partitioning/)
  * [Waiting for PostgreSQL 11 – Add hash partitioning](https://www.depesz.com/2017/11/10/waiting-for-postgresql-11-add-hash-partitioning/)

## Index (11)

La version 11 apporte également une gestion plus facilitée des index. La création
d'un index sur une table partitionnée entraîne sa création sur toutes les partitions.
De même, il est possible d'avoir un index unique (à condition qu'il porte sur la
clé de partitionnement). Il est également possible d'avoir une clé étrangère sur
une table partitionnée.

## Exclusion de partition (11)

Le *partition pruning* consiste à exclure les partitions inutiles. Postgres s'appuie
sur les contraintes d'exclusion pour écarter des partitions à la planification.

L'algorithme n'a pas été prévu pour gérer un nombre important de partitions. Sa
complexité est linéaire en fonction du nombre de partitions
([How many table partitions is too many in Postgres?](https://stackoverflow.com/a/6131446))

Pour y remédier, la version 11 intègre un nouveau algorithme de recherche bien
plus performant : *Faster Partition Pruning*.

Enfin, le moteur ne pouvait exclure des partitions que lors de la planification.
Avec la version 11, le moteur est capable d'exclure une partition durant l'exécution.
Cette fonctionnalité s'appelle *Runtime Partition Pruning*.

## Jointures et agrégations (11)

La version 11 apporte de nouveaux algorithmes de jointure et d'agrégation.
L'idée est de réaliser les opérations de jointures et d'agrégation partition par
partition lors d'une jointure entre deux tables partitionnées (Voir [Partition and conquer large data with PostgreSQL 10](https://www.pgcon.org/2017/schedule/events/1047.en.html)).

# Améliorations intrinsèques

## Fonctions de hashage (10)

Les fonctions de hashage ont été améliorées dans la version 10. Ainsi, les opérations
d'agrégat (`GROUP BY`, `GROUPING SETS`, `CUBE`...) ainsi que les noeuds type *bitmap scans*
bénéficient de ces améliorations. Le temps d'exécution de certaines requêtes a été divisé par deux!

## Amélioration de l'exécuteur (10)

L'exécuteur a été amélioré dans la version 10, il est maintenant plus performant pour
le traitement des expressions. Ce thread mentionne des gains très significatifs : [Faster Expression Processing](https://www.postgresql.org/message-id/20170314065259.ffef4tfhgbsaieoe%40alap3.anarazel.de)

Voir cet extrait du livre [PostgreSQL - Architecture et notions avancées](https://books.google.fr/books?id=LztCDwAAQBAJ&lpg=PA301&ots=Xko2mw6c6S&dq=postgres%20simd&hl=fr&pg=PA300#v=onepage&q&f=false) de Guillaume Lelarge et Julien Rouhaud.

## Amélioration des tris

### Abbreviated keys (9.5)

L'algorithme de tri a été revu avec la version 9.5, celui-ci exploite mieux le
cache des processeurs. Les gains annoncés étaient entre 40 et 70%
 (voir [Use abbreviated keys for faster sorting of text datums](http://pgeoghegan.blogspot.fr/2015/01/abbreviated-keys-exploiting-locality-to.html)).

### Amélioration des tris externes (10)

La version 10 apporte également des gains très significatifs lorsqu'ils sont
réalisés sur disque. Certaines requêtes ont vu leur temps d'exécution divisé par deux.

## Just-In-time (11)

La version 11 intègre une infrastructure pour le Just-In-Time (JIT) ou littéralement "[compilation à la volée](https://fr.wikipedia.org/wiki/Compilation_%C3%A0_la_vol%C3%A9e)".
Le JIT consiste à compiler la requête pour générer un [bytecode](https://fr.wikipedia.org/wiki/Bytecode) qui sera exécuté.

A nouveau, les gains annoncés sont impressionnants comme en témoigne ces slides de conférence : [JITing PostgreSQL using LLVM](http://anarazel.de/talks/fosdem-2018-02-03/jit.pdf)

# Taches de maintenance

## VACUUM FREEZE (9.6)

Avant la version 9.6, un `VACUUM FREEZE` entraînait la lecture de l'intégralité
de la table. Même si des lignes avaient déjà été "freezée". La version 9.6 ajoute
une information supplémentaire dans la *visibility map* afin de savoir si un bloc
a déjà été freezé. Cette information permet au moteur de sauter les blocs déjà freezé.

## Réduction des parcours d'index lors des opérations de VACUUM (11)

Lors d'une opération de vacuum simple (où le moteur va nettoyer les lignes périmées).
Le moteur était capable de sauter les blocs où il savait qu'il n'y avait aucune ligne
à traiter. Néanmoins il devait quand même parcourir l'index, ce qui peut s'avérer coûteux
avec une table volumineuse. La version 11 permet d'éviter de phénomène.

Un nouveau paramètre fait son apparition : *vacuum_cleanup_index_scale_factor*.

Le moteur peut éviter le parcours de l'index si deux conditions sont réunies :

  * Aucun bloc ne doit être supprimé
  * Les statistiques sont bien à jour

Le moteur considère que les statistiques ne sont pas à jour si plus de :
[nombre de lignes insérées] > vacuum_cleanup_index_scale_factor * [nombre de lignes dans la table]

## Création d'index parallélisée (11)

Lors de la création d'un index, le moteur peut utiliser plusieurs processus pour
réaliser l'opération de tri. En fonction du nombre de processus, le temps de création
de l'index peut être divisé de 2 à 4 environ.

# Pushdown dans les Foreign Data Wrapper (9.6, 10)

Petit rappel, les Foreign Data Wrapper permettent d'accéder à des données externe.
C'est l'implémentation dans Postgres de la norme SQL/MED pour "Management of External Data".

Un FDW permet d'accéder à tout type de donnée externe pour peu qu'un FDW existe (voir [Foreign data wrappers](https://wiki.postgresql.org/wiki/Foreign_data_wrappers)).

La communauté maintient un FDW permettant de se connecter à une instance Postgres : *postgres_fdw*.
Mais ce n'est pas tout, certaines opérations peuvent être "poussées" au serveur distant. C'est ce qu'on appelle le *pushdown*

Avant la version 9.6, une opération de tri ou une jointure se traduisait par le
rapatriement de toutes les données et le serveur effectuait le tri et/ou la jointure
en local.

La version 9.6 inclue le sort et join pushdown. Ainsi, l'opération de tri ou de
jointure peut être réalisée par le serveur distant. D'une part cela décharge le
serveur exécutant la requête, d'autre part, le serveur distant va pouvoir utiliser
l'algorithme de jointure adéquat ou un index pour le tri des données.

Enfin, la version 10 inclue aussi l'exécution des agrégations et jointures de type
`FULL JOIN` sur le serveur distant.

# Futur

Le feature freeze de la version 11 s'est terminé le 7 Avril : [PostgreSQL 11 Release Management Team & Feature Freeze](https://www.postgresql.org/message-id/AA141CD1-19CB-414F-98CB-87A32F397295%40postgresql.org)

Cela signifie que certaines fonctionnalités n'ont pas été implémentées, certaines
étant jugées pas assez matures pour être intégrées. D'autres ne sont encore qu'à
l'état de discussion ou de démonstration.

A noter que certaines fonctionnalités peuvent être retirées après le feature freeze
si les développeurs considèrent qu'elles ne sont pas stables ou que
l'implémentation doit être revue.

Par chance, les développeurs des différentes sociétées qui contribuent au développement
de Postgres, communiquent sur leur travaux en cours. Cela donne un aperçu des
tendances des futures fonctionnalités.

## Extension du système de stockage

La communauté travaille pour rendre le système de stockage modulaire (*[pluggable storage](https://www.postgresql.org/message-id/flat/20160812231527.GA690404%40alvherre.pgsql#20160812231527.GA690404@alvherre.pgsql)*). Ainsi,
le moteur pourrait avoir différents moteurs de stockage. Dans les travaux en
cours, on peut compter :

  * Le stockage colonne (associé à la vectorisation)
  * La compression des tables
  * Le stockage en mémoire (*In-memory*)
  * [zheap](https://github.com/EnterpriseDB/zheap) : ce moteur permettrait de
  mettre à jour les enregistrements directement dans les tables. Sans dupliquer
  les données dans les tables selon le modèle MVCC. Et ainsi, s'affranchir de la fragmentation et des vacuum.

## Extension du JIT

L'auteur du JIT prévoit de l'étendre au reste du moteur (agrégation, hashage, tris...).

Dans les pistes évoquées, il y aurait aussi la mise en cache et le partage du bytecode
entre toutes les sessions. Cela permettrait d'utiliser le JIT même dans le cas
d'un traffic OLTP.

## Vectorisation

L'idée de la vectorisation serait de traiter les données par batch afin d'exploiter
des [instructions SIMD](https://fr.wikipedia.org/wiki/Single_instruction_multiple_data) des processeurs.

Couplé à un stockage colonne, les gains peuvent être impressionnants :

  * [Vectorized Postgres (VOPS extension)](https://pgconf.ru/media/2018/02/20/%D0%9A%D0%BD%D0%B8%D0%B6%D0%BD%D0%B8%D0%BA_Vectorized%20Postgres%20(VOPS%20extension).pdf)

## FDW et exécution asynchrone

Le sujet du sharding revient régulièrement. D'une certaine façon, l'usage des
FDW répond en partie à ce besoin. Néanmois, PostgreSQL a encore des progrès à
faire dans ce domaine. Par exemple, s'il effectue une requête d'agrégation portant
sur plusieurs tables distantes, le moteur doit requêter chaque serveur distant de manière
séquentielle. Une piste d'amélioration serait de requêter tous les serveurs distants
de manière asynchrone. Ainsi, l'opération serait parallélisée sur tous les serveurs distants.
Voir cette présentation [FDW-based Sharding Update and Future](https://fr.slideshare.net/masahikosawada98/fdwbased-sharding-update-and-future).
