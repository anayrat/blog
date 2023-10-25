---
title: PostgreSQL 10 et la réplication logique – Fonctionnement
authors: ['adrien']
type: post
date: 2017-07-29T20:53:37+00:00
aliases: /2017/07/29/postgresql-10-et-la-replication-logique-fonctionnement/
categories:
  - Postgres
tags:
  - postgres
  - replication logique
show_related: true

---
Certains d'entre vous sont déjà au courant, la nouvelle version majeure de PostgreSQL approche à grands pas. Elle devrait sortir dans le courant du mois de septembre.

Comme chaque nouvelle version la liste de nouveautés est assez impressionnante :

  * partitionnement
  * amélioration des performances sur les tris et fonctions d’agrégation
  * extension du parallélisme : parcours d'index parallélisé,  jointure parallélisée, parallélisation des sous-requêtes...
  * statistiques étendues
  * support des collations  ICU : va permettre d'exploiter les « abbreviated keys  » qui avaient dû être désactivées en 9.5 à cause d'un bug dans la libc. Les « abbreviated keys » permettaient un gain de l'ordre de 20-30% sur les tris et créations d'index.
  * et je m'arrête là, vous pouvez avoir un aperçu des nouveautés sur la page [wiki](https://wiki.postgresql.org/wiki/New_in_postgres_10) ou dans les [releases notes](https://www.postgresql.org/docs/10/static/release-10.html).


Une grande nouveauté de la version 10 que je vais présenter dans une série d'articles est la _réplication logique_.

<!--more-->

  1. [PostgreSQL 10 et la réplication logique - Fonctionnement][1]
  2. [PostgreSQL 10 et la réplication logique - Mise en oeuvre][2]
  3. [PostgreSQL 10 et la réplication logique - Restrictions][3]


# Rappel

La réplication existe déjà dans PostgreSQL [^5]. Elle reposait sur la réplication dite « physique » : le moteur ne réplique pas des requêtes mais le résultat de la requête. Plus précisément les modifications des blocs de données.

[^5]: Par transfert de journaux de transaction depuis la version 8.2 sortie en 2006 et en flux depuis la version 9.0 sortie en 2010.

Le serveur secondaire se contente de rejouer les journaux de transaction. Cette technique est assez simple et est particulièrement efficace et fiable.

Néanmoins elle présente quelques limitations :

  * il n'existe aucune granularité, on est obligé de répliquer l'intégralité de l'instance
  * il n'est pas possible de faire une réplication entre différentes architecture (x86, ARM...).
  * le secondaire n'accepte aucune requête en écriture. Il n'est dont pas possible de créer des vues personnalisées ou des index.

# Fonctionnement

Contrairement à la réplication physique, la réplication logique ne réplique pas les blocs de données. Elle décode le résultat des requêtes qui est transmis au secondaire. Celui-ci applique les modifications SQL issues du flux de réplication logique.

Le serveur secondaire est en réalité un serveur primaire, dans le sens où il s'agît d'un serveur qui accepte des requêtes en écritures comme n'importe quel instance primaire.

Il sera ainsi possible de choisir les tables à répliquer, de rajouter des vues et index sur le serveur secondaire etc...

{{< figure src="/img/2017/schema-repli-logique.png" title="Schéma réplication logique" >}}

  1. Une « publication » est créée sur le serveur primaire (« publieur »), celle-ci comprend les tables appartenant à la publication.
  2. Le secondaire souscrit à cette publication, c'est un « souscripteur ».
  3. Ensuite un processus spécial est lancé : le  « bgworker logical replication ». Il va se connecter à un slot de réplication sur le serveur primaire.
  4. Le serveur primaire va procéder à un décodage logique des journaux de transaction pour extraire les résultats des ordres SQL.
  5. Le flux logique est transmis au secondaire qui les applique sur les tables.

La mise en oeuvre dans un prochain article...

[1]: http://blog.anayrat.info/2017/07/29/postgresql-10-et-la-replication-logique-fonctionnement/
[2]: http://blog.anayrat.info/2017/08/05/postgresql-10-et-la-replication-logique-mise-en-oeuvre/
[3]: https://blog.anayrat.info/2017/08/27/postgresql-10-et-la-replication-logique-restrictions/
[4]: http://blog.anayrat.info/wp-content/uploads/2017/07/schema-repli-logique.png
