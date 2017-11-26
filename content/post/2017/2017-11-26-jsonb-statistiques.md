+++
title = "PostgreSQL - JSONB et Statistiques"
date = 2017-11-26T21:11:38+01:00
draft = false

# Tags and categories
# For example, use `tags = []` for no tags, or the form `tags = ["A Tag", "Another Tag"]` for one or more tags.
tags = ["postgres","JSONB","Statistiques"]
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

# Rappels statistiques, cardinalité, sélectivité

Le SQL est un language dit "déclaratif". C'est un language où l'utilisateur demande ce qu'il souhaite. Sans préciser comment l'ordinateur doit procéder pour obtenir les résultats.

C'est le SGBD qui doit trouver "comment" réaliser l'opération en s'assurant de :

  * Retourner le résultat juste
  * Idéalement, le plus vite possible

"Le plus vite possible" se traduit par :

  * Réduire au maximum les accès disques
  * Privilégier les lectures séquentielles (praticulièrement important pour les disques mécaniques)
  * Réduire le nombre d'opérations CPU
  * Réduire l'empreinte mémoire

Pour se faire, un SGBD possède ce que l'on appèle un optimiseur dont le rôle est de trouver le meilleur *plan d'exécution*.

PostgreSQL possède un optimiseur basé sur un mécanisme de coût.
Sans rentrer dans les détails, chaque opération a un cout unitaire (lecture d'un bloc séquentiel, traitement CPU d'un enregistrement...).
Le moteur calcule le coût de plusieurs plans d'exécution (si la requête est simple) et choisi le moins couteux.

Comment le moteur peut estimer le coût d'un plan? En estimant le coût de chaque noeud du plan en se basant sur des statistiques.
PostgreSQL analyse les tables pour d'obtenir un échantillon statistique (opération normalement réalisée par l'*auto_vacuum*).


Quelques mots de vocabulaire :

*Cardinalité* : Dans la théorie des ensemble, c'est le nombre d'éléments dans un ensemble. Dans les bases de données, ça sera le nombre de lignes d'une table ou après application d'un prédicat.

*Sélectivité* : Fraction d'enregistrements retournés après application d'un prédicat. Par exemple, une table contenant des personnes et dont environ un tier correspond à des enfants. La sélectivité du prédicat `personne = 'enfant'` sera de 0.33.

Si cette table contient 300 personnes (c'est la cardinalité de l'ensemble "personnes"), on peut estimer le nombre d'enfants car on sait que le prédicat `personne = 'enfant'` est de 0.33 :

300 * 0.33 = 99

On peut obtenir ces estimations avec `EXPLAIN` qui affiche le plan d'exécution.

Exemple (simplifié) :

```SQL
explain (analyze, timing off) select * from t1 WHERE c1=1;
                                  QUERY PLAN
------------------------------------------------------------------------------
 Seq Scan on t1  (cost=0.00..5.75 rows=100 ...) (actual rows=100 ...)
   Filter: (c1 = 1)
   Rows Removed by Filter: 200
```

*(cost=0.00..5.75 rows=100 ...)* : Indique le coût estimé et le nombre d'enregistrements estimé (*rows*).

*(actual rows=100 ...)* : Indique le nombre d'enregistrement réellement obtenu.

La documentation de PostgreSQL fourni des exemples de calculs d'estimation : [Row Estimation Examples](https://www.postgresql.org/docs/current/static/row-estimation-examples.html)

Il est assez facile de comprendre comment obtenir des estimations à partir de types de données scalaires.

Comment ça se passe pour des types particuliers? Par exemple le JSON?

# Recherche sur du JSONB

## Jeu de données

Comme dans les précédents articles, j'ai utilisé le jeu de données de stackoverflow. J'ai créé une nouvelle table en aggrégeant les données de plusieurs tables dans un objet JSON :

```SQL
CREATE TABLE json_stack AS
SELECT t.post_id,
       row_to_json(t,
                   TRUE)::jsonb json
FROM
  (SELECT posts.id post_id,
          posts.owneruserid,
          users.id,
          title,
          tags,
          BODY,
          displayname,
          websiteurl,
          LOCATION,
          aboutme,
          age
   FROM posts
   JOIN users ON posts.owneruserid = users.id) t;
```

Le traitement est assez long car les deux tables concernées totalisent près de 40Go.

J'obtiens donc une table de 40Go qui ressemble à ça :

```SQL
 \dt+ json_stack
                      List of relations
 Schema |    Name    | Type  |  Owner   | Size  | Description
--------+------------+-------+----------+-------+-------------
 public | json_stack | table | postgres | 40 GB |
(1 row)


\d json_stack
             Table "public.json_stack"
 Column  |  Type   | Collation | Nullable | Default
---------+---------+-----------+----------+---------
 post_id | integer |           |          |
 json    | jsonb   |           |          |


select post_id,jsonb_pretty(json) from json_stack
    where json_displayname(json) = 'anayrat' limit 1;
 post_id  |
----------+-----------------------------------------------------------------------------------------
 26653490 | {
          |     "id": 4197886,
          |     "age": null,
          |     "body": "<p>I have an issue with date filter. I follow [...]
          |     "tags": "<java><logstash>",
          |     "title": "Logstash date filter failed parsing",
          |     "aboutme": "<p>Sysadmin, Postgres DBA</p>\n",
          |     "post_id": 26653490,
          |     "location": "Valence",
          |     "websiteurl": "https://blog.anayrat.info",
          |     "displayname": "anayrat",
          |     "owneruserid": 4197886
          | }
```

## Opérateurs et indexation sur du JSONB

PostgreSQL propose plusieurs opérateurs pour interroger du JSONB [^1]. Nous allons utiliser l'opérateur `@>`.

[^1]: <https://www.postgresql.org/docs/current/static/functions-json.html>

Il est également possible d'indexer du JSONB grâce aux index GIN :

```SQL
create index ON json_stack using gin (json );
```

Enfin, voici un exemple de requête :

```SQL
explain (analyze,buffers)  select * from json_stack
    where json @>  '{"displayname":"anayrat"}'::jsonb;
                                  QUERY PLAN
---------------------------------------------------------------------------------------
 Bitmap Heap Scan on json_stack
                (cost=286.95..33866.98 rows=33283 width=1011)
                (actual time=0.099..0.102 rows=2 loops=1)
   Recheck Cond: (json @> '{"displayname": "anayrat"}'::jsonb)
   Heap Blocks: exact=2
   Buffers: shared hit=17
   ->  Bitmap Index Scan on json_stack_json_idx
                          (cost=0.00..278.62 rows=33283 width=0)
                          (actual time=0.092..0.092 rows=2 loops=1)
         Index Cond: (json @> '{"displayname": "anayrat"}'::jsonb)
         Buffers: shared hit=15
 Planning time: 0.088 ms
 Execution time: 0.121 ms
(9 rows)
```

En lisant ce plan on constate que le moteur se trompe complètement.
Il estime obtenir 33 283 lignes, hors la requête retourne seulement deux enregistrements. Le facteur d'erreur est d'environ 15 000!

## Sélectivité sur du JSONB

Quelle est la cardinalité de la table? L'information est contenue dans le catalogue système :

```SQL
select reltuples from pg_class where relname = 'json_stack';
  reltuples
-------------
 3.32833e+07
```

Quelle est la sélectivité estimée?

```SQL
select 33283 / 3.32833e+07;
        ?column?
------------------------
 0.00099999098647069251
```

En gros 0.001.

## Plongée dans le code

Je me suis amusé à sortir le débugueur GDB pour trouver d'où pouvait venir ce chiffre. J'ai fini par arriver dans cette fonction :

```C
[...]
79 /*
80  *  contsel -- How likely is a box to contain (be contained by) a given box?
81  *
82  * This is a tighter constraint than "overlap", so produce a smaller
83  * estimate than areasel does.
84  */
85
86 Datum
87 contsel(PG_FUNCTION_ARGS)
88 {
89     PG_RETURN_FLOAT8(0.001);
90 }
[...]
```

Le calcul de la sélectivité dépend du type de l'opérateur. Regardons dans le catalogue système :

```sql
select oprname,typname,oprrest from pg_operator op
    join pg_type typ ON op.oprleft= typ.oid where oprname = '@>';
 oprname | typname  |   oprrest
---------+----------+--------------
 @>      | polygon  | contsel
 @>      | box      | contsel
 @>      | box      | contsel
 @>      | path     | -
 @>      | polygon  | contsel
 @>      | circle   | contsel
 @>      | _aclitem | -
 @>      | circle   | contsel
 @>      | anyarray | arraycontsel
 @>      | tsquery  | contsel
 @>      | anyrange | rangesel
 @>      | anyrange | rangesel
 @>      | jsonb    | contsel
```

On retrouve plusieurs types, en effet l'opérateur `@>`signifie (en gros) : "Est-ce que l'objet de gauche contient l'élément de droite?".
Il est utilisé pour différents types : géométrie, tableaux...

Dans notre cas, est-ce que l'objet JSONB de gauche contient l'élément `'{"displayname":"anayrat"}'` ?

Un objet JSON est un type particulier. Déterminer la sélectivité d'un élément serait assez complexe. Le commentaire est assez explicite :

```C
 25 /*
 26  *  Selectivity functions for geometric operators.  These are bogus -- unless
 27  *  we know the actual key distribution in the index, we can't make a good
 28  *  prediction of the selectivity of these operators.
 29  *
 30  *  Note: the values used here may look unreasonably small.  Perhaps they
 31  *  are.  For now, we want to make sure that the optimizer will make use
 32  *  of a geometric index if one is available, so the selectivity had better
 33  *  be fairly small.
[...]
```

Il n'est donc pas possible (actuellement) de déterminer la sélectivité sur des objets JSONB.

Mais tout n'est pas perdu :wink:

# Index fonctionnels

PostgreSQL permet de créer des index dits *fonctionnels*. On crée un index à partir d'une foction.

Vous allez me dire : "Oui mais on n'en a pas besoin. Dans ton exemple, postgres utilise déjà un index".

C'est vrai, la différence, c'est que le moteur collecte des statistiques à propos de cet index.
Comme si le résultat de la fonction était une nouvelle colonne.

## Création de la fonction et de l'index

C'est très simple :

```plpgsql
CREATE or replace FUNCTION json_displayname (jsonb )
RETURNS text
AS $$
select $1->>'displayname'
$$
LANGUAGE SQL IMMUTABLE PARALLEL SAFE
;

create index ON json_stack (json_displayname(json));
```

## Recherche en utilisant une fonction

Pour utiliser l'index que nous venons de créer, il faut l'utiliser dans la requête :

```SQL
explain (analyze,verbose,buffers) select * from json_stack
        where json_displayname(json) = 'anayrat';
                        QUERY PLAN
----------------------------------------------------------------------------
 Index Scan using json_stack_json_displayname_idx on public.json_stack
            (cost=0.56..371.70 rows=363 width=1011)
            (actual time=0.021..0.023 rows=2 loops=1)
   Output: post_id, json
   Index Cond: ((json_stack.json ->> 'displayname'::text) = 'anayrat'::text)
   Buffers: shared hit=7
 Planning time: 0.107 ms
 Execution time: 0.037 ms
(6 rows)
```

Cette fois le moteur estime obtenir 363 lignes, ce qui est bien plus proche du résultat final (2).


## Autre exemple et calcul de sélectivité

Cette fois nous allons effectuer une recherche sur le champ "age" de l'objet JSON :


```SQL
explain (analyze,buffers)  select * from json_stack
      where json @>  '{"age":27}'::jsonb;
                      QUERY PLAN
---------------------------------------------------------------------
 Bitmap Heap Scan on json_stack
        (cost=286.95..33866.98 rows=33283 width=1011)
        (actual time=667.411..12723.906 rows=804630 loops=1)
   Recheck Cond: (json @> '{"age": 27}'::jsonb)
   Rows Removed by Index Recheck: 2211190
   Heap Blocks: exact=391448 lossy=344083
   Buffers: shared hit=576350 read=881510
   I/O Timings: read=2947.458
   ->  Bitmap Index Scan on json_stack_json_idx
        (cost=0.00..278.62 rows=33283 width=0)
        (actual time=562.648..562.648 rows=804644 loops=1)
         Index Cond: (json @> '{"age": 27}'::jsonb)
         Buffers: shared hit=9612 read=5140
         I/O Timings: read=11.195
 Planning time: 0.073 ms
 Execution time: 12809.392 ms
(12 lignes)

set work_mem = '100MB';

explain (analyze,buffers)  select * from json_stack
      where json @>  '{"age":27}'::jsonb;
                      QUERY PLAN
---------------------------------------------------------------------
 Bitmap Heap Scan on json_stack
        (cost=286.95..33866.98 rows=33283 width=1011)
        (actual time=748.968..5720.628 rows=804630 loops=1)
   Recheck Cond: (json @> '{"age": 27}'::jsonb)
   Rows Removed by Index Recheck: 14
   Heap Blocks: exact=735531
   Buffers: shared hit=123417 read=780542
   I/O Timings: read=1550.124
   ->  Bitmap Index Scan on json_stack_json_idx
        (cost=0.00..278.62 rows=33283 width=0)
        (actual time=545.553..545.553 rows=804644 loops=1)
         Index Cond: (json @> '{"age": 27}'::jsonb)
         Buffers: shared hit=9612 read=5140
         I/O Timings: read=11.265
 Planning time: 0.079 ms
 Execution time: 5796.219 ms
(12 lignes)
```

Dans cet exemple on constate que le moteur estime toujours obtenir 33 283 enregistrements.
Hors il en obtient 804 644. Cette fois il est beaucoup trop optimiste.

PS : Dans mon exemple vous verrez que j'ai joué la même requête en modifiant la `work_mem`. C'est pour éviter que le bitmap ne soit `lossy` [^2]

[^2]: Un noeud bitmap devient lossy lorsque le moteur ne peut pas faire un bitmap de tous les enregistrements. Il passe donc en mode dit "lossy" où le bitmap ne se fait plus sur l'enregistrement mais pour le bloc entier. Ce qui nécessite de lire plus de blocs et de faire un "recheck" qui consiste à filter les enregistrements obtenus.

Comme vu ci-dessus nous pouvons créer une fonction :

```SQL
CREATE or replace FUNCTION json_age (jsonb )
RETURNS text
AS $$
select $1->>'age'
$$
LANGUAGE SQL IMMUTABLE PARALLEL SAFE
;

create index ON json_stack (json_age(json));
```

A nouveau l'estimation est bien meilleure:

```SQL
explain (analyze,buffers)   select * from json_stack
      where json_age(json) = '27';
                         QUERY PLAN
------------------------------------------------------------------------
 Index Scan using json_stack_json_age_idx on json_stack
  (cost=0.56..733177.05 rows=799908 width=1011)
  (actual time=0.042..2355.179 rows=804630 loops=1)
   Index Cond: ((json ->> 'age'::text) = '27'::text)
   Buffers: shared read=737720
   I/O Timings: read=1431.275
 Planning time: 0.087 ms
 Execution time: 2410.269 ms
```

Le moteur estime obtenir 799 908 enregistrements. nous allons le vérifier.

Comme je l'ai précisé, le moteur possède des informations de statistiques basées sur un échantillon de données.
Ces informations sont stockées dans un catalogue système lisible grâce à la vue `pg_stats`
Avec un index fonctionnel, le moteur le voit comme une nouvelle colonne.

```psql
schemaname             | public
tablename              | json_stack_json_age_idx
attname                | json_age
[...]
most_common_vals       | {28,27,29,31,26,30,32,25,33,34,36,24,[...]}
most_common_freqs      | {0.0248,0.0240333,0.0237333,0.0236333,0.0234,0.0229333,[...]}
[...]
```
La colonne *most_common_vals* contient les valeurs les plus fréquentes et la colonne *most_common_freqs* la sélectivité correspondante.

Donc pour `age=27` on a une sélectivité de 0.0240333.

Ensuite il suffit de multiplier la sélectivité par la cardinalité de la table :

```SQL
select n_live_tup from pg_stat_all_tables where relname ='json_stack';
 n_live_tup
------------
   33283258


select 0.0240333 * 33283258;
    ?column?
----------------
 799906.5244914
```

D'accord, l'estimation est bien meilleure. Mais est-ce grave si le moteur se trompe? Dans les deux requêtes ci-dessus on constate que le moteur exploite bien un index et que le résultat est obtenu rapidement.


# Conséquences d'une mauvaise estimation

En quoi une mauvaise estimation peut poser problème?

Quand ça entraine le choix d'un mauvais plan.

Par exemple cette requête d'aggregation qui compte le nombre de posts par age:

```SQL
explain (analyze,buffers)  select json->'age',count(json->'age')
                          from json_stack group by json->'age' ;
                             QUERY PLAN
--------------------------------------------------------------------------------------
 Finalize GroupAggregate
  (cost=10067631.49..14135810.84 rows=33283256 width=40)
  (actual time=364151.518..411524.862 rows=86 loops=1)
   Group Key: ((json -> 'age'::text))
   Buffers: shared hit=1949354 read=1723941, temp read=1403174 written=1403189
   I/O Timings: read=155401.828
   ->  Gather Merge
        (cost=10067631.49..13581089.91 rows=27736046 width=40)
        (actual time=364151.056..411524.589 rows=256 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         Buffers: shared hit=1949354 read=1723941, temp read=1403174 written=1403189
         I/O Timings: read=155401.828
         ->  Partial GroupAggregate
            (cost=10066631.46..10378661.98 rows=13868023 width=40)
            (actual time=363797.836..409187.566 rows=85 loops=3)
               Group Key: ((json -> 'age'::text))
               Buffers: shared hit=5843962 read=5177836,
                        temp read=4212551 written=4212596
               I/O Timings: read=478460.123
               ->  Sort
                  (cost=10066631.46..10101301.52 rows=13868023 width=1042)
                  (actual time=299775.029..404358.743 rows=11094533 loops=3)
                     Sort Key: ((json -> 'age'::text))
                     Sort Method: external merge  Disk: 11225392kB
                     Buffers: shared hit=5843962 read=5177836,
                              temp read=4212551 written=4212596
                     I/O Timings: read=478460.123
                     ->  Parallel Seq Scan on json_stack
                        (cost=0.00..4791997.29 rows=13868023 width=1042)
                        (actual time=0.684..202361.133 rows=11094533 loops=3)
                           Buffers: shared hit=5843864 read=5177836
                           I/O Timings: read=478460.123
 Planning time: 0.080 ms
 Execution time: 411688.165 ms
```

Le moteur s'attend à obtenir 33 283 256 d'enregistrement au lieu de 86. Il a également effectué un tri très couteux puisqu'il a généré plus de 33Go (11GO * 3 loops) de fichiers temporaires.

La même requête en utilisant la fonction *json_age* :

```SQL
explain (analyze,buffers)   select json_age(json),count(json_age(json))
                              from json_stack group by json_age(json);
                                             QUERY PLAN
--------------------------------------------------------------------------------------
 Finalize GroupAggregate
  (cost=4897031.22..4897033.50 rows=83 width=40)
  (actual time=153985.585..153985.667 rows=86 loops=1)
   Group Key: ((json ->> 'age'::text))
   Buffers: shared hit=1938334 read=1736761
   I/O Timings: read=106883.908
   ->  Sort
      (cost=4897031.22..4897031.64 rows=166 width=40)
      (actual time=153985.581..153985.598 rows=256 loops=1)
         Sort Key: ((json ->> 'age'::text))
         Sort Method: quicksort  Memory: 37kB
         Buffers: shared hit=1938334 read=1736761
         I/O Timings: read=106883.908
         ->  Gather
            (cost=4897007.46..4897025.10 rows=166 width=40)
            (actual time=153985.264..153985.360 rows=256 loops=1)
               Workers Planned: 2
               Workers Launched: 2
               Buffers: shared hit=1938334 read=1736761
               I/O Timings: read=106883.908
               ->  Partial HashAggregate
                  (cost=4896007.46..4896008.50 rows=83 width=40)
                  (actual time=153976.620..153976.635 rows=85 loops=3)
                     Group Key: (json ->> 'age'::text)
                     Buffers: shared hit=5811206 read=5210494
                     I/O Timings: read=320684.515
                     ->  Parallel Seq Scan on json_stack
                     (cost=0.00..4791997.29 rows=13868023 width=1042)
                     (actual time=0.090..148691.566 rows=11094533 loops=3)
                           Buffers: shared hit=5811206 read=5210494
                           I/O Timings: read=320684.515
 Planning time: 0.118 ms
 Execution time: 154086.685 ms

```

Ici le moteur effectue le tri plus tard sur beaucoup moins de lignes. Le temps d'exécution est nettement réduit et on s'épargne surtout 33Go de fichiers temporaires.


# Mot de la fin

Les statistiques sont indispensables pour choisir les meilleurs plan d'exécution. Actuellement Postgres dispose de fonctionnalités avancées pour le JSON [^4]
Malheureusement il n'y a pas encore de possibilité d'ajouter de statistiques sur le type JSONB. A noter que PostgreSQL 10 apporte l'infrastructure pour [étendre les statistiques](https://www.postgresql.org/docs/current/static/sql-createstatistics.html). Espérons que dans le futur il sera possible de les étendre pour des types spéciaux.

En attendant il est possible de contourner cette limitation en utilisant les index fonctionnels.

[^4]: La [documentation](https://www.postgresql.org/docs/current/static/functions-json.html) est très complète et fournie de nombreux exemples : usage des opérateurs, indexation.
