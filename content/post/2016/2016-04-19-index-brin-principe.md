---
title: Index BRIN – Principe
author: Adrien Nayrat
type: post
date: 2016-04-19T06:00:21+00:00
aliases: /2016/04/19/index-brin-principe/
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

<!--more-->

# Introduction

PostgreSQL propose déjà différents type d'index : B-Tree, GIN, GiST, SP-GiST, Hash [^8]

Les discussions ont commencées en 2008 : [Segment Exclusion](http://www.postgresql.org/message-id/1199296574.7260.149.camel@ebony.site)

Suivi de ce RFC en 2008 qui propose les « [Minmax indexes](https://www.postgresql.org/message-id/20130614222805.GZ5491@eldon.alvh.no-ip.org) » qui sera renommé plus tard en BRIN.

Physiquement, une table est composée de blocs de 8Ko et chaque bloc contient des enregistrements [^9]. L'idée des index BRIN est de stocker dans un index la valeur minimale et maximale d'un attribut d'un ensemble de bloc (_range_) [^10]. Ainsi il est possible d'exclure un ensemble de bloc si vous recherchez une valeur qui n'est pas comprise dans l'intervalle.

Note : Les index BRIN sont similaires aux _storage indexes_ d'Oracle ([Exadata Storage Indexes][7]).

Voici les types supportés : [Built-in Operator Classes](http://www.postgresql.org/docs/current/static/brin-builtin-opclasses.html#BRIN-BUILTIN-OPCLASSES-TABLE)

```
abstime
bigint
bit
bit varying
box
bytea
character
"char"
date
double precision
inet
inet
integer
interval
macaddr
name
numeric
pg_lsn
oid
any range type
real
reltime
smallint
text
tid
timestamp without time zone
timestamp with time zone
time without time zone
time with time zone
uuid
```

Il est également possible d'indexer plusieurs colonnes. Notez que la documentation actuelle ne le mentionne pas, c'est corrigé dans la version en cours de développement : [Multicolumn Indexes](http://www.postgresql.org/docs/devel/static/indexes-multicolumn.html)


 [1]: http://blog.anayrat.info/2016/04/19/index-brin-principe/
 [2]: http://blog.anayrat.info/2016/04/20/index-brin-fonctionnement/
 [3]: http://blog.anayrat.info/2016/04/20/index-brin-correlation/
 [4]: http://blog.anayrat.info/2016/04/21/index-brin-performances/
 [5]: http://www.pgday.fr/index.html
 [6]: http://www.pgday.fr/programme.html
 [7]: https://en.wikipedia.org/wiki/Block_Range_Index#Exadata_Storage_Indexe

[^8]: `SELECT amname FROM pg_am;` ou encore <http://www.postgresql.org/docs/current/static/indexes-types.html>
[^9]: Si un enregistrement ne tient pas dans un bloc il est stocké séparément dans ce qu'on appelle le TOAST (The Oversized-Attribute Storage Technique). Plus exactement, si un enregistrement dépasse 2ko.
[^10]: L'index contient également deux booléens indiquant si l'ensemble contient des valeurs NULL (hasnulls ) ou s'il ne contient **que** des valeurs NULL (allnulls)
