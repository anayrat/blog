+++
title = "Cas d'usage du partionnement"
date = 2019-06-25T15:12:26+02:00
draft = true

summary = "Nous allons voir quels peuvnet être les différents cas d'usages où on peut envisager le partionnement"

# Tags and categories
# For example, use `tags = []` for no tags, or the form `tags = ["A Tag", "Another Tag"]` for one or more tags.
tags = ["postgres","partitionnement"]
categories = ["Postgres"]

+++


# Histoire partitionnement dans PostgreSQL

PostgreSQL permet depuis très longtemps de partitonner des tables en exploitant l'héritage de table. Toutefois, cette méthode était assez lourde à mettre en oeuvre : elle impliquait de mettre en place soi-même des triggers pour rediriger les écritures (moins performant que le partitionnement natif), le temps de planification pouvait augmenter fortement au délà d'une centain de partitions...


La partitionnement natif est arrivé avec la version 10. C'est depuis cette version que le moteur est capable (entre autre) de diriger lui même les écritures vers les bonnes tables, lire seulement les tables concernnées, d'utiliser des algorithmes exploitant le partitionnement etc.
Il offre de meilleures performances, une facilité d'exploitation. On peut entre autre :

  * Partitionner :
    * Par liste
    * Par hashage
    * Par intervalle
  * Faire des partitionnement à plusieurs niveaux
  * Partitionner sur plusieurs colonnes
  * Utiliser des clés primaires et clés étrangères

Toutes ces fonctionnalités sont intéressantes mais on en vient à se poser une question toute bête : quand mettre en oeuvre le partitionnement?

Je vais vous présenter plusieurs cas d'usages, plus ou moins connus. Mais avant, voici quelques erreurs courantes sur la partitionnement.


# Erreurs courantes

## "Il faut partitionner dès que la volumétrie est importante"

Déjà, qu'est une volumétrie "importante"?

Certains diront que c'est au delà de plusieurs centaine de Go, d'autres au-delà du teraoctet, d'autres encore au-delà du petaoctet...

Il n'existe pas vraiment de réponse à cette question et globalement ça va dépendre du type d'activité : rapport INSERT/UPDATE/DELETE, type de SELECT (OLTP, OLAP...).
Ca dépendra également du matériel. Il y a 10 ans, quand les serveurs n'avaient que quelques Go de RAM avec des disques mécaniques, il est probable qu'une base de quelques centaines de Go était perçu comme une grosse base.
Maintenant il n'est pas rare de voir des serveurs avec plus d'un teraoctet de RAM, des disques NVMe. Ainsi, une base de quelques centaines de Go n'est plus tout considérée comme une grosse base. Mais plutôt comme une base de taille modeste.

Petite anecdote, une jour, pour se rassurer, un client m'a questionné si Postgres était déjà utilisé pour des volumétries importantes. On parlait d'une quarantaine de Go sur un serveur qui disposait de 64Go de RAM. Toutes les lectures se faisaient depuis le cache... :)

Il peut tout à fais être superflu de partitionner une base de quelques To comme il peut être nécessaire de partitionner une base de quelques centaines de Go. Par exemple, si l'activité consiste juste à ajouter des lignes à des tables et que les requêtes se résument à de simple `WHERE colonne = 4` qui retourne quelques lignes. Un simple Btree fera l'affaire. Et si la requête retourne un nombre assez important de ligne, rappelons qu'il est possible d(utilsier les index BRIN ou les bloom filter.

Les index BRIN présentes des bénéfices similaires au partitionnement ou sharding en évitant la complexité de mise en oeuvre[^1].


## "Il faut partitionner pour répartir les données sur plusieurs disques"

L'idée serait de créer des partitions et des tablespaces sur différentes disques afin de répartir les opérations I/O.

Pour PostgreSQL, un tablespace n'est ni plus, ni moins qu'un chemin vers un répertoire. Il est tout à fait possible
d'aggreger plusieurs disques (en RAID10) par exemple et de stocker la table sur le volume créé. Ainsi, on peut répartir les I/O.

Il n'est donc pas nécessaire de mettre en oeuvre le partitionnement. iToutefois, nous verront plus loin un cas où ça pourrait avoir du sens.



# Partionner pour gérer la rétention

Un 

Exemple, stockage de logs avec une certaine rétention, on peut facilement virer une partition.

# Partionner pour contrôler la fragmentation des index

=> à étendre au fait de pouvoir faire un vacuum full ou cluster qui permettrait d'avoir une correlation élevée avec le stockage

# Partioner pour faciliter l'exécution de requête lorsque la cardinalité est faible

Exemple : table de commande, au bout de quelques années 99% des commandes sont livrées et très peu en cours de paiement ou livraison. 

Récupérer 100 commandes en cours de livraison peut se faire facilement avec un seqscan, sans utiliser un parcours d'index.

Meilleure localité du cache

# partitionwise_join & partitionwise_aggregate

hash partioning

# Stockage avec tiering


    Stats moins précise avec forte volumétrie
    bloat
    rétention
    Stats avec des valeur à faible cardinalité

[^1]: "BRIN indexes provide similar benefits to horizontal partitioning or sharding but without needing to explicitly declare partitions." - <https://en.wikipedia.org/wiki/Block_Range_Index> 

