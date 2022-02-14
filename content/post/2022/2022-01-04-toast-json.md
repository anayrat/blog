+++
title = "PostgreSQL compression du TOAST et toast_tuple_target"
date = 2022-02-14T09:00:00+02:00
draft = false

summary = "Effets du TOAST et application du toast_tuple_target sur le JSONB"

# Tags and categories
# For example, use `tags = []` for no tags, or the form `tags = ["A Tag", "Another Tag"]` for one or more tags.
tags = ["postgres","toast", "jsonb"]
categories = ["Postgres"]

+++

Quelques rappels sur le TOAST et présentation d'un changement apparu avec PostgreSQL 11.


{{% toc %}}

# Qu'est-ce que le TOAST ?

Vous êtes-vous déjà posé la question sur comment Postgres fait pour stocker des lignes dépassant la taille d'un bloc? Pour rappel, la taille par défaut d'un bloc est de 8Ko.

Postgres utilise un mécanisme appelé TOAST pour [The Oversized-Attribute Storage Technique](https://www.postgresql.org/docs/current/storage-toast.html).

Lorsqu'un enregistrement devient trop gros pour être stocké dans un bloc, le moteur va le stocker "à part", dans une table de toast.
L'enregistrement sera découpé en *chunks*, ainsi la table principale (appelée *heap*) contiendra un pointeur (*chunk_id*) pointant vers le bon *chunk* dans la table de toast.

Ce *chunk* sera stocké sur plusieurs lignes, pour un *chunk_id* on peut avoir plusieurs lignes dans cette table de toast.
Ainsi, cette table de toast est composée de 3 colonnes:

* *chunk_id* : Numéro du chunk référencé dans la heap
* *chunk_seq* : Numéro de chaque segment d'un chunk
* *chunk_data* : Partie données de chaque segment

La réalité est un peu plus complexe, en vrai le moteur va tenter d'éviter de stocker la donnée dans la table toast.
Si la ligne dépasse `TOAST_TUPLE_THRESHOLD` (2Ko), il va tenter de compresser les colonnes pour essayer de faire rentrer la ligne dans le bloc.
Plus précisément, il faut que la taille soit inférieure à `TOAST_TUPLE_TARGET` (2Ko par défaut, on va en reparler).

Si on a de la chance, la ligne compressée rentre dans la heap. Sinon, il va tenter de compresser les colonnes,
de la plus grande à la plus petite et les stocker dans la partie toast jusqu'à ce que les colonnes restantes rentrent dans une ligne de la heap. [^toastpass]

[^toastpass]: Voir [heap_toast_insert_or_update](https://git.postgresql.org/gitweb/?p=postgresql.git;a=blob;f=src/backend/access/heap/heaptoast.c;h=55bbe1d584760a849960871296dfbdd7447b2b67;hb=refs/heads/REL_14_STABLE#l160)

A noter également que si le gain en compression est trop faible, il considère qu'il est inutile de dépenser
de la ressource de calcul à tenter de compresser. Il stocke donc la donnée sans compression. [^toastcompress]

[^toastcompress]: Il existe deux algorithmes de compression supportés : *pglz* (historique et intégré dans Postgres) et *lz4* (depuis Postgres 14).

    * Pour *pglz*, voir [PGLZ_Strategy](https://git.postgresql.org/gitweb/?p=postgresql.git;a=blob;f=src/include/common/pg_lzcompress.h;h=3e53fbe97bd0a10e3fbf7ed4396924084f657868;hb=refs/heads/REL_14_STABLE#l25)
et [strategy_default_data](https://git.postgresql.org/gitweb/?p=postgresql.git;a=blob;f=src/common/pg_lzcompress.c;h=a30a2c2eb83a71725754d8dd680621a02e7557e9;hb=refs/heads/REL_14_STABLE#l223)
    * Pour *lz4*, il s'agit d'une librarie externe. Voir [LZ4_compress_default](https://github.com/lz4/lz4/blob/dev/lib/lz4.h#L145).

Avez-vous déjà prêté attention à la colonne "Storage" lorsque vous affichez les caractéristiques d'une table à l'aide de la méta commande `\d+ table` ?

```
stackoverflow=# \d+ posts
                   Table "public.posts"
    Column     |  Type   | Collation | Nullable | Default | Storage  |
---------------+---------+-----------+----------+---------+----------+
 id            | integer |           | not null |         | plain    |
 posttypeid    | integer |           | not null |         | plain    |
 score         | integer |           |          |         | plain    |
 viewcount     | integer |           |          |         | plain    |
 body          | text    |           |          |         | extended |
```


Dans cet exemple, la colonne prend comme valeur *plain* ou *extended*. En réalité, il existe 4 valeurs possibles selon le type de donnée :

* *plain* : la colonne est stockée dans la heap uniquement et sans compression.
* *extended* : la colonne peut être compressée et stockée dans le toast si nécessaire.
* *external* : la colonne peut être stockée dans le toast mais uniquement sans compression. Parfois on peut utiliser ce mode pour avoir un gain en performance (évite la compression/décompression)
au prix d'une consommation plus importante de l'espace disque.
* *main* : La colonne est stockée dans la heap uniquement mais contrairement au mode *plain*, on autorise la compression.

Au premier abord, on peut penser que l'intérêt est surtout sur la possibilité de stocker
des lignes dépassant la taille d'un bloc et de compresser la donnée pour gagner de l'espace disque.

Il y a un autre intérêt : lors d'une mise à jour d'une ligne, si les colonnes "toastées" ne sont pas modifiées, le moteur n'a pas besoin de modifier la table toast.
On va ainsi éviter de devoir décompresser et recompresser le toast et écrire tout ça dans des journaux de transaction.

Nous allons voir qu'un autre avantage est que le moteur peut éviter de lire le toast si ce n'est pas nécessaire.


# Exemple avec le JSONB

Pour étudier ça, on va utiliser le type JSONB. De manière générale, je déconseille l'usage de ce type :

* On perd les avantages d'avoir un schéma :
  * vérification des types
  * contraintes d'intégrité
  * pas de clés étrangères
* L'écriture des requêtes devient plus complexe
* Absence des statistiques sur les clés d'un champ json
* Perte d'efficacité du stockage vu qu'on stocke les clés pour chaque ligne
* Pas de mise à jour partielle du JSONB. Si on modifie une clé on est obligé de *detoaster* et *toaster* tout le JSONB
* Pas de *detoast* partiel : si on souhaite lire une seule clé du JSONB, on est contraint de *detoaster* tout le JSONB [^1]

[^1]: Voir les slides de la conférence d'Oleg Bartunov et Nikita Glukhov : [json or not json that is the question](http://www.sai.msu.su/~megera/postgres/talks/jsonb-nizhny-2021.pdf)


Cependant, il y a quelques exceptions où le JSON peut être utile :

* Lorsqu'on n'a pas besoin de chercher dans de multiples champs et qu'on récupère le json via une autre colonne. (Statistiques sur les clés du json inutiles).
* Et, lorsqu'il serait très difficile de faire rentrer le json dans un schéma relationnel. Certains cas impliqueraient d'avoir énormément de colonnes et la plupart à `NULL`.

Par exemple, pour stocker des caractéristiques de produit où une version normalisée entrainerait l'usage de beaucoup de colonnes dont la plupart seraient à `NULL`.
Imaginons que vous stockez des produits, une télévision aurait des caractéristiques spécifiques (type d'écran, taille etc). Une machine à laver aurait aussi d'autre caractéristiques spécifiques (vitesse essorage, poids accepté...).

On pourrait ainsi envisager d'avoir des colonnes "normales" comprenant le modèle, son prix, sa référence etc, et une colonne contenant toutes les caractéristiques.
On accèderait à la ligne via la référence et ainsi on récupèrerait toutes les caractéristiques du produit stockées dans le json.

Je vais réutiliser la table des posts de Stackoverflow en déplaçant quelques colonnes dans une colonne de type jsonb (colonne *jsonfield* dans cet exemple):

```
\d posts
                            Unlogged table "public.posts"
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
 jsonfield             | jsonb                       |           |          |
```

Voici une requête toute simple d'agrégation :

```sql
SELECT
  avg(viewcount),
  avg(answercount),
  avg(commentcount),
  avg(favoritecount)
FROM posts;
```


```
                                                          QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=10265135.77..10265135.78 rows=1 width=128) (actual time=170221.557..170221.558 rows=1 loops=1)
   Buffers: shared hit=1 read=9186137
   I/O Timings: read=138022.290
   ->  Seq Scan on posts  (cost=0.00..9725636.88 rows=53949888 width=16) (actual time=0.014..153665.913 rows=53949886 loops=1)
         Buffers: shared hit=1 read=9186137
         I/O Timings: read=138022.290
 Planning Time: 0.240 ms
 Execution Time: 170221.627 ms
(8 rows)
```

La requête lit 70 Go de données et met environ 2min 50s à s'exécuter.

Maintenant la même requête, mais cette fois en utilisant les clés présentes dans le json.


```sql
SELECT
  avg((jsonfield ->> 'ViewCount')::int),
  avg((jsonfield ->> 'AnswerCount')::int),
  avg((jsonfield ->> 'CommentCount')::int),
  avg((jsonfield ->> 'FavoriteCount')::int)
FROM posts;
```


```
                           QUERY PLAN
------------------------------------------------------------------------------
 Aggregate  (cost=11883632.41..11883632.42 rows=1 width=128)
            (actual time=520917.028..520917.030 rows=1 loops=1)
   Buffers: shared hit=241116554 read=13625756
   ->  Seq Scan on posts  (cost=0.00..9725636.88 rows=53949888 width=570)
                          (actual time=0.972..70569.365 rows=53949886 loops=1)
         Buffers: shared read=9186138
 Planning Time: 0.118 ms
 Execution Time: 520945.395 ms
(10 rows)
```

La requête met environ 8min 40s à s'exécuter. En revanche le nombre de blocs lus semble un peu délirant :

Le Seq Scan indique comme tout à l'heure 70Go. En revanche, le nœud parent indique plus de 1.9 To lus!

Voici la taille de la table avec le paramétrage par défaut. Il faut savoir que pour certains enregistrements,
le moteur va, soit compresser la ligne dans la heap, soit la compresser et la placer dans le toast.


```
SELECT
  pg_size_pretty(pg_relation_size(oid)) table_size,
  pg_size_pretty(pg_relation_size(reltoastrelid)) toast_size
FROM pg_class
WHERE relname = 'posts';

 table_size | toast_size
------------+-----------
 70 GB      | 33 GB
(1 row)
```

Comment expliquer les 1.9 To lus ?

Par curiosité, j'ai fait la même requête, mais avec une seule agrégation et j'obtiens environ 538 Go.

On peut se poser plusieurs questions :

1. Comment savoir si le moteur va lire le toast ?
2. Pourquoi un tel écart de temps d'exécution entre la version "colonne standard" et champ jsonb?
3. A quoi correspondent les compteurs dans le nœud `Aggregate` ?

Pour répondre à la première question, il suffit de lire la vue `pg_statio_user_tables`.

Avant exécution de la requête :

```
select relid,schemaname,relname,heap_blks_read,heap_blks_hit,toast_blks_read,toast_blks_hit from pg_statio_all_tables where relname in ('posts','pg_toast_26180851');
  relid   | schemaname |      relname      | heap_blks_read | heap_blks_hit | toast_blks_read | toast_blks_hit
----------+------------+-------------------+----------------+---------------+-----------------+----------------
 26180851 | public     | posts             |      422018238 |      87673549 |       129785076 |      628153337
 26180854 | pg_toast   | pg_toast_26180851 |      129785076 |     628153337 |                 |
(2 rows)
```

Après :

```
  relid   | schemaname |      relname      | heap_blks_read | heap_blks_hit | toast_blks_read | toast_blks_hit
----------+------------+-------------------+----------------+---------------+-----------------+----------------
 26180851 | public     | posts             |      431204376 |      87673549 |       134156898 |      686299551
 26180854 | pg_toast   | pg_toast_26180851 |      134156898 |     686299551 |                 |
(2 rows)
```

Ce qui nous fait :

```sql
SELECT
pg_size_pretty(
    ((431204376 + 87673549) - (422018238 + 87673549) ) * 8*1024::bigint
) heap_buffers,
pg_size_pretty(
    ((134156898 + 686299551) - (129785076 + 628153337) ) * 8*1024::bigint
) toast_buffers;

 heap_buffers | toast_buffers
--------------+---------------
 70 GB        | 477 GB
(1 row)
```

Le moteur va bien lire le toast. En revanche les compteurs laissent penser que le moteur va lire plusieurs fois le toast.

Si je fais le même calcul, mais cette fois en effectuant l'agrégation que sur un seul champ, j'obtiens 119 Go (~ 477 Go / 4)
J'imagine que le moteur lit le toast pour chaque fonction.

Ensuite, l'écart du temps d'exécution s'explique par plusieurs facteurs :

* Le moteur va devoir lire et *detoaster* (décompresser) le toast
* Faire des opérations supplémentaires sur le jsonb pour accéder à la valeur

Avec la première requête, le moteur n'avait pas à lire le toast. D'une part, il a moins de données à lire,
d'autre part, il n'a pas à manipuler le json pour identifier la clé et extraire la valeur à calculer.

Enfin, les compteurs du nœud aggregate doivent correspondre aux données décompressées pour chaque fonction qui va lire dans le json.
En effet, si on prend le total moins le *seqscan* de la table, donc que la partie *toast*, on a :

* 468 Go pour un seul champ
* 936 Go, le double pour deux champs
* 1873 Go pour les 4 champs (donc environ 4 x 468 Go)

C'est ce qui explique pourquoi on obtient une valeur aussi élevée.

# Paramétrage avancé

Maintenant, on va encourager Postgres à placer le maximum de données dans le toast grâce à l'option *toast_tuple_target* apparue avec la version 11 de Postgres.

Cette option permet de manipuler le seuil à partir duquel les données sont stockée dans le *toast*.

Par ailleurs, étant sous Postgres 14, j'en ai profité pour utiliser l'algorithme de compression lz4 (paramètre *default_toast_compression*).
Cet algorithme offre un ratio de compression similaire à pglz, cependant, il est beaucoup plus rapide (Voir [What is the new LZ4 TOAST compression in PostgreSQL 14, and how fast is it?](https://www.postgresql.fastware.com/blog/what-is-the-new-lz4-toast-compression-in-postgresql-14)).


```sql
CREATE TABLE posts_toast
  WITH (toast_tuple_target = 128) AS
    SELECT *
    FROM posts;
```

Voici la taille de la table obtenue.

```
SELECT
  pg_size_pretty(pg_relation_size(oid)) table_size,
  pg_size_pretty(pg_relation_size(reltoastrelid)) toast_size
FROM pg_class
WHERE relname = 'posts_toast';

 table_size | toast_size
------------+------------
 59 GB      | 52 GB
```

Au total, la table avec le toast fait grosso-modo la même taille. Dans l'exemple avec la première table,
il faut savoir que le moteur compresse aussi les données dans la heap.

Rejouons notre requête d'agrégation :

```sql
SELECT
  avg(viewcount),
  avg(answercount),
  avg(commentcount),
  avg(favoritecount)
FROM posts_toast;
```

Cette fois la requête lit 59 Go de données et met 2min 17 secondes.
On a gagné environ 20% de temps d'exécution sur cet exemple.

On pourrait gagner beaucoup plus si la partie stockée en toast était plus importante. Le volume de donnée à lire dans la heap serait beaucoup plus réduit.

Par curiosité, j'ai aussi exécuté la requête qui fait l'agrégation depuis les données du champ json. J'obtiens un temps d'exécution de 7min 17s.

# Conclusion

Résumé en quelques chiffres :

* Agrégation type standard, stockage standard : 2min 50s
* Agrégation type jsonb, stockage standard : 8min 40s
* Agrégation type standard, stockage avec *toast_tuple_target* = 128 : 2min 17s
* Agrégation type jsonb, stockage avec *toast_tuple_target* = 128 : 7min 17s

On constate que l'usage du JSON est bien plus couteux que d'utiliser les types standards. Le moteur doit faire plus d'opérations pour accéder à la valeur d'une clé json.

Par ailleurs, il est obligé de décompresser les données dans le toast pour y accéder. Néanmoins, on peut aussi jouer avec le paramètre `toast_tuple_target` pour pousser plus
d'informations dans le toast. Ainsi, dans certains cas, cela peut permettre de réduire la quantité de données lues en évitant de lire le toast.

# Bonus

Comment souvent dans Postgres, tout évolue au fil des versions. Le TOAST n'échappe pas à cette règle.
Ainsi, quelques nouveautés pourraient apparaitre dans les prochaines versions :

1. Un premier patch a été proposé pour avoir plus de statistiques sur le toast : [pg_stat_toast](https://commitfest.postgresql.org/37/3457/).
L'idée, est d'avoir des statistiques sur la compression : gain compression, stockage inline ou séparé dans le toast...
2. Un second patch appelé [Pluggable toaster](https://commitfest.postgresql.org/37/3490/). Celui-ci est beaucoup plus important. Il propose d'étendre le *"toaster"*.
L'idée serait de pouvoir proposer différents *"toaster"* selon le type de donnée (notamment le JSONB).
