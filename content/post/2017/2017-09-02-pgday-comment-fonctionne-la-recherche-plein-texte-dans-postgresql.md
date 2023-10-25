---
title: 'PGDay : Comment fonctionne la recherche plein texte dans PostgreSQL?'
authors: ['adrien']
type: post
date: 2017-09-02T11:04:58+00:00
aliases: /2017/09/02/pgday-comment-fonctionne-la-recherche-plein-texte-dans-postgresql/
categories:
  - Postgres
tags:
  - full text search
  - postgres
show_related: true

---

Lors du dernier [PGDay][1] j'ai fait une présentation sur le fonctionnement de la recherche plein texte (Full Text Search - FTS) dans PostgreSQL.


Cette fonctionnalité est malheureusement trop peu connue. Je vois plusieurs raisons à cela :

* Complexité : Le FTS fait appels à des notions inconnues d'un DBA : lemmatisation, représentation vectorielle d'un document...
* La tendance à utiliser un outil dédié à la recherche plein texte : Elasticsearch, SOLR ...
* Ignorance des fonctionnalités avancées de PostgreSQL

Pourtant, il y a plusieurs avantages à utiliser le FTS de PostgreSQL :

  * On conserve un seul et même langage : le SQL
  * Pas de duplication des données entre la base de donnée et le moteur d'indexation
  * On conserve une certaine cohérence :
    * Pas besoin de faire des synchronisations entre les données contenues dans votre base de donnée et le moteur d'indexation
    * Un document supprimé de la base de donnée ne sera pas oublié dans le moteur d'indexation
  * On bénéficie de toutes les fonctionnalités avancées de PostgreSQL, notamment des index performants.
  * Le moteur est extensible, on peut personnaliser la configuration FTS

Quand j'ai commencé à me plonger dans le FTS j'ai vite été rebuté par des termes qui m'était inconnus, des syntaxes particulières avec de nouveaux opérateurs. Au lieu de faire une présentation sur l'étendue des possibilités du FTS, je me suis donc attaché à présenter son fonctionnement par « la base ». C'est à dire toute la « mécanique » qui se cache derrière.

Voici la vidéo de la conférence :

{{< youtube 9S5dBqMbw8A >}}

Ainsi que les [slides](https://blog.anayrat.info/presentations/2017/nayrat_Le_Full_Text_Search_dans_PostgreSQL.pdf).

[1]: http://pgday.fr/programme.html
