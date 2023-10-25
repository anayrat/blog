+++
title = "Postgres: partitionwise join et partitionwise aggregate"
date = 2018-04-25T14:09:38+02:00
draft = true
authors = ['adrien']

summary = "Présentation des jointures et aggrégats par partition"


# Tags and categories
# For example, use `tags = []` for no tags, or the form `tags = ["A Tag", "Another Tag"]` for one or more tags.
tags = ["postgres","TABLESAMPLE"]
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


Souvenez-vous, dans l'article sur [les évolutions de PostgreSQL pour le traitement des fortes volumétries](https://blog.anayrat.info/2018/04/16/les-%C3%A9volutions-de-postgresql-pour-le-traitement-des-fortes-volum%C3%A9tries/), j'avais évoqué une fonctionnalité
de la version 11 de PostgreSQL : les jointures et agrégations par partition.

J'ai voulu tester cette fonctionnalité sur un jeu de données réel et j'ai ressorti
ma base de stackoverflow.

# Préparation du jeu de données

Commençons par partitionner quelques tables. J'ai choisi deux grosses tables :
celles des posts et celle des commentaires. elles font respectivement 37Go et 13Go.

La table de posts a cette structure :

```
\d posts
                                 Table "public.posts"
        Column         |            Type             | Collation | Nullable | Default
-----------------------+-----------------------------+-----------+----------+---------
 id                    | integer                     |           | not null |
 posttypeid            | integer                     |           | not null |
 acceptedanswerid      | integer                     |           |          |
 parentid              | integer                     |           |          |
 creationdate          | timestamp without time zone |           | not null |
 score                 | integer                     |           |          |
 viewcount             | integer                     |           |          |
 body                  | text                        |           |          |
 owneruserid           | integer                     |           |          |
 lasteditoruserid      | integer                     |           |          |
 lasteditordisplayname | text                        |           |          |
 lasteditdate          | timestamp without time zone |           |          |
 lastactivitydate      | timestamp without time zone |           |          |
 title                 | text                        |           |          |
 tags                  | text                        |           |          |
 answercount           | integer                     |           |          |
 commentcount          | integer                     |           |          |
 favoritecount         | integer                     |           |          |
 closeddate            | timestamp without time zone |           |          |
 communityowneddate    | timestamp without time zone |           |          |
```

J'ai choisi de faire un partitionnement assez simple avec une partition par année à partir de a colonne creationdate.

Pour ce faire, j'ai créé une nouvelle table `posts_part` avec la même structure que la table `posts`, en rajoutant la clause `PARTITION BY RANGE (creationdate` :

```
CREATE TABLE public.postspart (
    id integer NOT NULL,
    posttypeid integer NOT NULL,
    acceptedanswerid integer,
    parentid integer,
    creationdate timestamp without time zone NOT NULL,
    score integer,
    viewcount integer,
    body text,
    owneruserid integer,
    lasteditoruserid integer,
    lasteditordisplayname text,
    lasteditdate timestamp without time zone,
    lastactivitydate timestamp without time zone,
    title text,
    tags text,
    answercount integer,
    commentcount integer,
    favoritecount integer,
    closeddate timestamp without time zone,
    communityowneddate timestamp without time zone
)
PARTITION BY RANGE (creationdate);
```

Petite astuce, vous pouvez récupérer la structure de la table avec pg_dump, ensuite il n'y a plus qu'à rajouter la clause de partionnement :
```
pg_dump --section=pre-data -t posts stackoverflow
CREATE TABLE public.posts (
    id integer NOT NULL,
    posttypeid integer NOT NULL,
    acceptedanswerid integer,
    parentid integer,
    creationdate timestamp without time zone NOT NULL,
    score integer,
    viewcount integer,
    body text,
    owneruserid integer,
    lasteditoruserid integer,
    lasteditordisplayname text,
    lasteditdate timestamp without time zone,
    lastactivitydate timestamp without time zone,
    title text,
    tags text,
    answercount integer,
    commentcount integer,
    favoritecount integer,
    closeddate timestamp without time zone,
    communityowneddate timestamp without time zone
);
```

Jusque là c'est assez simple. Maintenant il faut créer toutes les partitions. La syntaxe est :
```
CREATE TABLE public.postspart_2008 PARTITION OF public.postspart
FOR VALUES FROM ('2008-01-01 00:00:00') TO ('2009-01-01 00:00:00');
```

Le problème, c'est que j'ai plein de partitions à créer, c'est assez pénible de
le faire à la main et source d'erreur. Nous allons générer les ordres :

```
SELECT 'CREATE TABLE postspart_'  || date_part('year', i) ||
' PARTITION OF postpart FOR VALUES FROM (''' || i || ''') TO (''' ||
i + interval '1 year' || ''')'
FROM ( SELECT generate_series('2008-01-01 00:00:00'::timestamp,'2020-01-01 00:00:00'::timestamp,'1 year')) AS sub(i);
```
Vous noterez que j'en ai profité pour générer les ordres des futures partitions.
Je pourrais également créer une partition par défaut si jamais aucune partition
ne correspond à la nouvelle lignes, après le 1er janvier 2021 par exemple.

Ce qui me donne :

```
CREATE TABLE postspart_2008 PARTITION OF postpart FOR VALUES FROM ('2008-01-01 00:00:00') TO ('2009-01-01 00:00:00')
CREATE TABLE postspart_2009 PARTITION OF postpart FOR VALUES FROM ('2009-01-01 00:00:00') TO ('2010-01-01 00:00:00')
CREATE TABLE postspart_2010 PARTITION OF postpart FOR VALUES FROM ('2010-01-01 00:00:00') TO ('2011-01-01 00:00:00')
CREATE TABLE postspart_2011 PARTITION OF postpart FOR VALUES FROM ('2011-01-01 00:00:00') TO ('2012-01-01 00:00:00')
CREATE TABLE postspart_2012 PARTITION OF postpart FOR VALUES FROM ('2012-01-01 00:00:00') TO ('2013-01-01 00:00:00')
CREATE TABLE postspart_2013 PARTITION OF postpart FOR VALUES FROM ('2013-01-01 00:00:00') TO ('2014-01-01 00:00:00')
CREATE TABLE postspart_2014 PARTITION OF postpart FOR VALUES FROM ('2014-01-01 00:00:00') TO ('2015-01-01 00:00:00')
CREATE TABLE postspart_2015 PARTITION OF postpart FOR VALUES FROM ('2015-01-01 00:00:00') TO ('2016-01-01 00:00:00')
CREATE TABLE postspart_2016 PARTITION OF postpart FOR VALUES FROM ('2016-01-01 00:00:00') TO ('2017-01-01 00:00:00')
CREATE TABLE postspart_2017 PARTITION OF postpart FOR VALUES FROM ('2017-01-01 00:00:00') TO ('2018-01-01 00:00:00')
CREATE TABLE postspart_2018 PARTITION OF postpart FOR VALUES FROM ('2018-01-01 00:00:00') TO ('2019-01-01 00:00:00')
CREATE TABLE postspart_2019 PARTITION OF postpart FOR VALUES FROM ('2019-01-01 00:00:00') TO ('2020-01-01 00:00:00')
CREATE TABLE postspart_2020 PARTITION OF postpart FOR VALUES FROM ('2020-01-01 00:00:00') TO ('2021-01-01 00:00:00')
```

Je n'ai plus qu'à faire `\gexec` dans psql qui va exécuter les ordres qui viennent
d'être générés (`\gexec` est apparu avec la version 9.6).


Maintenant il va falloir remplir les partitions, c'est également assez simple :
```
INSERT INTO postspart select *  from posts;
```

Postgres va automatiquement rediriger les enregistrements vers les bonnes partitions :
```
\dt+ postspart*
                           List of relations
 Schema |      Name      | Type  |  Owner   |    Size    | Description
--------+----------------+-------+----------+------------+-------------
 public | postspart      | table | postgres | 0 bytes    |
 public | postspart_2008 | table | postgres | 194 MB     |
 public | postspart_2009 | table | postgres | 959 MB     |
 public | postspart_2010 | table | postgres | 1661 MB    |
 public | postspart_2011 | table | postgres | 2794 MB    |
 public | postspart_2012 | table | postgres | 3872 MB    |
 public | postspart_2013 | table | postgres | 4914 MB    |
 public | postspart_2014 | table | postgres | 5119 MB    |
 public | postspart_2015 | table | postgres | 5310 MB    |
 public | postspart_2016 | table | postgres | 5359 MB    |
 public | postspart_2017 | table | postgres | 5320 MB    |
 public | postspart_2018 | table | postgres | 3518 MB    |
 public | postspart_2019 | table | postgres | 8192 bytes |
 public | postspart_2020 | table | postgres | 8192 bytes |
```

J'ai réalisé exactement la même opération avec la table de commentaires :
```
\dt+ comments_part*
                             List of relations
 Schema |        Name        | Type  |  Owner   |    Size    | Description
--------+--------------------+-------+----------+------------+-------------
 public | comments_part      | table | postgres | 0 bytes    |
 public | comments_part_2008 | table | postgres | 26 MB      |
 public | comments_part_2009 | table | postgres | 251 MB     |
 public | comments_part_2010 | table | postgres | 519 MB     |
 public | comments_part_2011 | table | postgres | 954 MB     |
 public | comments_part_2012 | table | postgres | 1372 MB    |
 public | comments_part_2013 | table | postgres | 1771 MB    |
 public | comments_part_2014 | table | postgres | 1837 MB    |
 public | comments_part_2015 | table | postgres | 1912 MB    |
 public | comments_part_2016 | table | postgres | 1907 MB    |
 public | comments_part_2017 | table | postgres | 1854 MB    |
 public | comments_part_2018 | table | postgres | 1177 MB    |
 public | comments_part_2019 | table | postgres | 8192 bytes |
 public | comments_part_2020 | table | postgres | 8192 bytes |
```


Maintenant, imaginons que je souhaite compter le nombre de posts avec commentaire
