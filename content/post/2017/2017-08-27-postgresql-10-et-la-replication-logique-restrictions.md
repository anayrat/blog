---
title: PostgreSQL 10 et la réplication logique – Restrictions
authors: ['adrien']
type: post
date: 2017-08-27T16:48:22+00:00
aliases: /2017/08/27/postgresql-10-et-la-replication-logique-restrictions/
categories:
  - Linux
  - Postgres
tags:
  - postgres
  - replication logique
show_related: true

---
Cet article est la suite d'une série d'articles sur la réplication logique dans la version 10 de PostgreSQL

Celui-ci va porter sur les restrictions de la réplication logique.


<!--more-->

1. [PostgreSQL 10 et la réplication logique - Fonctionnement][1]
2. [PostgreSQL 10 et la réplication logique - Mise en oeuvre][2]
3. [PostgreSQL 10 et la réplication logique - Restrictions][3]

# Restrictions

La documentation de postgreSQL est très bien faite. Il y a toute une page sur les [restrictions][4].

## Ordres DDL

Le schéma et les ordres DDL ne sont pas répliqués. Il faudra donc veiller à bien reporter toute modification de schéma. Dans le cas contraire un conflit de réplication pourrait se produire ce qui bloque toute réplication. Une fois la correction apportée la réplication peut reprendre. Pour éviter toute erreur il est préférable d'appliquer les changements de schéma sur le “secondaire” en premier.

## Séquences

Les séquences ne sont pas répliquées. Les données d'une colonne _serial_ seront bien répliquées mais le compteur de la séquence ne sera pas incrémenté. Si le “secondaire” est seulement utilisé en lecture seule ça ne posera pas de problème. En revanche, en cas de bascule il faudra mettre à jour la séquence ou la placer à une valeur supérieure à la valeur la plus haute présente dans la table.

## Ordre TRUNCATE

Les ordres `TRUNCATE` ne sont pas répliqués. Il peut être utile de révoquer le droit de `TRUNCATE` sur les tables répliquées avec : `REVOKE TRUNCATE ON TABLE matable FROM utilisateur`.

## Large Objects

Les _Large Objets_ ne sont pas répliqués

## Schéma et tables

La réplication logique ne réplique que des tables, il n'est pas possible de répliquer des vues, vues matérialisées, partition racine, _foreign tables_… Les schéma « source » et « destination » doivent être identiques. Il n'est pas possible de répliquer la table t1 du schéma s1 dans un schéma s2 sur un secondaire.

## Replica Identity

Pour les opérations UPDATE et DELETE, la table doit avoir une [_replica identity_][5]. Dans le cas contraire le moteur remontera une erreur. Par défaut c'est la clé primaire qui est utilisée.

 [1]: http://blog.anayrat.info/2017/07/29/postgresql-10-et-la-replication-logique-fonctionnement/
 [2]: http://blog.anayrat.info/2017/08/05/postgresql-10-et-la-replication-logique-mise-en-oeuvre/
 [3]: https://blog.anayrat.info/2017/08/27/postgresql-10-et-la-replication-logique-restrictions/
 [4]: https://www.postgresql.org/docs/10/static/logical-replication-restrictions.html
 [5]: https://www.postgresql.org/docs/10/static/sql-altertable.html#sql-createtable-replica-identity
