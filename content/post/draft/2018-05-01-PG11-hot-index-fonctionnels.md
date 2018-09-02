+++
title = "2018 05 01 PG11 Hot Index Fonctionnels"
date = 2018-04-25T14:09:22+02:00
draft = true

# Tags and categories
# For example, use `tags = []` for no tags, or the form `tags = ["A Tag", "Another Tag"]` for one or more tags.
tags = ["postgres","index"]
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

Cet article va porter sur une nouveauté de la version 11.

Durant le développement de cette version, une fonctionnalité a attiré mon attention.
On peut la retrouver dans les releases notes. Même si cette version n'est pas
encore officiellement sortie, on peut avoir accès à la documentation : <https://www.postgresql.org/docs/11/static/release-11.html>


> Allow heap-only-tuple (HOT) updates for expression indexes when the values of the expressions are unchanged (Konstantin Knizhnik)

J'avoue que ce n'est pas très explicite et cette fonctionnalité nécessite quelques
connaissances sur le fonctionnement du moteur que je vais essayer d'expliquer.

# MVCC

Dans l'implémentation MVCC de Postgres, le moteur ne met pas à jour une ligne
directement. Il duplique l'enregistrement et renseigne des informations de visibilité.

Pourquoi ce fonctionnement?

Il y a une composante capitale à prendre en compte quand on travaille avec un
SGBG : la concurrence d'accès.

La ligne que vous êtes en train de modifier est peut être utile par une transaction
antérieure. Une sauvegarde en cours par exemple :)

Pour cela les SGBD ont adopté différentes techniques :

  * Modifier l'enregistrement et stocker les versions antérieures sur un autre
  emplacement. C'est ce que fait oracle par exemple avec les redo logs.
  * Dupliquer l'enregistrement et stocker des informations de visibilité pour
  savoir quelle ligne est visible par telle ou telle transaction. Cela nécessite
  d'avoir un mécanisme de nettoyage des lignes qui ne sont plus visibles par
  personne. C'est l'implémentation dans Postgres et le vacuum a pour rôle d'effectuer ce nettoyage.

Prenons une table toute simple et regardons sont contenu évoluer à l'aide de l'extension pageinspect :

```
postgres=# create table t2(c1 int);
CREATE TABLE
postgres=# insert into t2 values (1);
INSERT 0 1
postgres=# select lp,t_data from  heap_page_items(get_raw_page('t2',0));
 lp |   t_data   
----+------------
  1 | \x01000000
(1 row)

postgres=# update t2 SET c1 = 2 WHERE c1 = 1;
UPDATE 1
postgres=# select lp,t_data from  heap_page_items(get_raw_page('t2',0));
 lp |   t_data   
----+------------
  1 | \x01000000
  2 | \x02000000
(2 rows)

postgres=# vacuum t2;
VACUUM
postgres=# select lp,t_data from  heap_page_items(get_raw_page('t2',0));
 lp |   t_data   
----+------------
  1 |
  2 | \x02000000
(2 rows)
```

On peut voir que le moteur a dupliqué la ligne et le vacuum a nettoyé
l'emplacement pour un usage futur.

# heap-only-tuple

Prenons un autre cas, un peu plus compliqué, une table avec deux colonnes et un
index sur une des deux colonnes :

```
create table t3(c1 int,c2 int);
create index ON t3(c1);
insert into t3(c1,c2) values (1,1);
insert into t3(c1,c2) values (2,2);
select ctid,* from t3;
 ctid  | c1 | c2
-------+----+----
 (0,1) |  1 |  1
 (0,2) |  2 |  2
(2 rows)

```

De la même façon qu'on peut lire les blocs d'une table, on peut lire les blocs
d'un index avec pageinspect :

```
select * from  bt_page_items(get_raw_page('t3_c1_idx',1));
 itemoffset | ctid  | itemlen | nulls | vars |          data           
------------+-------+---------+-------+------+-------------------------
          1 | (0,1) |      16 | f     | f    | 01 00 00 00 00 00 00 00
          2 | (0,2) |      16 | f     | f    | 02 00 00 00 00 00 00 00
(2 rows)
```

Jusque là c'est assez simple, ma table contient deux enregistrements et l'index
contient également deux enregistrements pointant vers les blocs correspondants de
la table (colonne ctid).

Si je mets à jour la colonne c1, avec la nouvelle valeur 3 par exemple, l'index
devra être mis à jour.

Maintenant si je mets à jour la colonne c2. Est-ce que l'index portant sur c1
sera mis à jour?

Au premier abord on pourrait se dire non car c1 n'est pas modifiée.

Mais à cause du modèle MVCC présenté plus haut, en théorie, la réponse sera oui :
nous venons de voir plus haut que le moteur va dupliquer la ligne, son emplacement
physique sera donc différent (le ctid suivant sera (0,3)).

Vérifions le :

```
update t3 set c2 = 3 WHERE c1=1;
UPDATE 1
postgres=# select lp,t_data,t_ctid from  heap_page_items(get_raw_page('t3',0));
 lp |       t_data       | t_ctid
----+--------------------+--------
  1 | \x0100000001000000 | (0,3)
  2 | \x0200000002000000 | (0,2)
  3 | \x0100000003000000 | (0,3)
(3 rows)


postgres=# select * from  bt_page_items(get_raw_page('t3_c1_idx',1));
 itemoffset | ctid  | itemlen | nulls | vars |          data           
------------+-------+---------+-------+------+-------------------------
          1 | (0,1) |      16 | f     | f    | 01 00 00 00 00 00 00 00
          2 | (0,2) |      16 | f     | f    | 02 00 00 00 00 00 00 00
(2 rows)
```

La lecture du bloc de la table confirme bien que la ligne a été dupliquée.
En observant attentivement le champ `t_data`, on arrive à distinguer le 1 de la
colonne c1 et le 3 de la colonne c2.

En lisant le bloc de l'index, on constate que son contenu n'a pas bougé! Si je
cherche la ligne WHERE c1 = 1, l'index m'oriente vers l'enregistrement (0,1) qui
correspond à l'ancienne ligne!

Que s'est-il passé?

En réalité nous venons de mettre en évidence un mécanisme un peu particulier
appelé *heap-only-tuple* alias HOT. Lorsqu'une colonne est mise à jour, qu'aucun
index ne pointe vers cette colonne et qu'on peut insérer l'enregistrement dans
le même bloc, le moteur va se contenter de faire un pointeur entre l'ancien
enregistrement le nouveau.

Cela permet au moteur d'éviter d'avoir à mettre à jour l'index. Avec tout ce que
cela entraîne :

  * Des opérations en lecture/écriture évitées
  * Réduction de la fragmentation de l'index et donc de sa taille (il est
    difficile de réutiliser les anciens emplacements d'un index)

Si vous observez la lecture du bloc de la table, la colonne `t_ctid` de la
première ligne pointe vers (0,3). Si la ligne était à nouveau mise à jour, la
première ligne de la table pointerait vers la ligne (0,3) et la ligne (0,3)
pointerait vers (0,4), formant ce qu'on appelle une chaîne. un vacuum nettoierait
les espaces libres mais conserverait toujours la première ligne qui pointera vers
le dernier enregistrement.


On modifie une ligne et l'index ne change toujours pas:
```
update t3 set c2 = 4 WHERE c1=1;
postgres=# select * from  bt_page_items(get_raw_page('t3_c1_idx',1));
 itemoffset | ctid  | itemlen | nulls | vars |          data           
------------+-------+---------+-------+------+-------------------------
          1 | (0,1) |      16 | f     | f    | 01 00 00 00 00 00 00 00
          2 | (0,2) |      16 | f     | f    | 02 00 00 00 00 00 00 00
(2 rows)

postgres=# select lp,t_data,t_ctid from  heap_page_items(get_raw_page('t3',0));
 lp |       t_data       | t_ctid
----+--------------------+--------
  1 | \x0100000001000000 | (0,3)
  2 | \x0200000002000000 | (0,2)
  3 | \x0100000003000000 | (0,4)
  4 | \x0100000004000000 | (0,4)
(4 rows)
```

Vacuum nettoie les emplacements disponibles:
```
postgres=# vacuum t3;
VACUUM
postgres=# select lp,t_data,t_ctid from  heap_page_items(get_raw_page('t3',0));
 lp |       t_data       | t_ctid
----+--------------------+--------
  1 |                    |
  2 | \x0200000002000000 | (0,2)
  3 |                    |
  4 | \x0100000004000000 | (0,4)
(4 rows)
```

Un update va réutiliser le second emplacement et l'index reste inchangé.
Observez la valeur de la colonne `t_ctid` pour reconstituer la chaîne.
```
postgres=# update t3 set c2 = 5 WHERE c1=1;
UPDATE 1
postgres=# select lp,t_data,t_ctid from  heap_page_items(get_raw_page('t3',0));
 lp |       t_data       | t_ctid
----+--------------------+--------
  1 |                    |
  2 | \x0200000002000000 | (0,2)
  3 | \x0100000005000000 | (0,3)
  4 | \x0100000004000000 | (0,3)
(4 rows)


postgres=# select * from  bt_page_items(get_raw_page('t3_c1_idx',1));
 itemoffset | ctid  | itemlen | nulls | vars |          data           
------------+-------+---------+-------+------+-------------------------
          1 | (0,1) |      16 | f     | f    | 01 00 00 00 00 00 00 00
          2 | (0,2) |      16 | f     | f    | 02 00 00 00 00 00 00 00
(2 rows)
```

Euh, la première ligne est vide et le moteur a réutilisé le troisième emplacement?

En fait, ça n’apparaît pas dans pageinspect. Allons lire directement le bloc avec
[pg_filedump](https://wiki.postgresql.org/wiki/Pg_filedump) :

Note : Il faut demander un `CHECKPOINT` au préalable sinon le bloc pourrait ne
pas encore être écrit sur le disque.

```
pg_filedump  11/main/base/16606/8890510

*******************************************************************
* PostgreSQL File/Block Formatted Dump Utility - Version 10.1
*
* File: 11/main/base/16606/8890510
* Options used: None
*
* Dump created on: Sun Sep  2 13:09:53 2018
*******************************************************************

Block    0 ********************************************************
<Header> -----
 Block Offset: 0x00000000         Offsets: Lower      40 (0x0028)
 Block: Size 8192  Version    4            Upper    8096 (0x1fa0)
 LSN:  logid     52 recoff 0xc39ea148      Special  8192 (0x2000)
 Items:    4                      Free Space: 8056
 Checksum: 0x0000  Prune XID: 0x0000168b  Flags: 0x0001 (HAS_FREE_LINES)
 Length (including item array): 40

<Data> ------
 Item   1 -- Length:    0  Offset:    4 (0x0004)  Flags: REDIRECT
 Item   2 -- Length:   32  Offset: 8160 (0x1fe0)  Flags: NORMAL
 Item   3 -- Length:   32  Offset: 8096 (0x1fa0)  Flags: NORMAL
 Item   4 -- Length:   32  Offset: 8128 (0x1fc0)  Flags: NORMAL
```

La première ligne contient `Flags: REDIRECT`, ceci indique que cette ligne
correspond à une redirection HOT. C'est documenté dans `src/include/storage/itemid.h` :

```
34  * lp_flags has these possible states.  An UNUSED line pointer is available     
35  * for immediate re-use, the other states are not.                              
36  */                                                                             
37 #define LP_UNUSED       0       /* unused (should always have lp_len=0) */      
38 #define LP_NORMAL       1       /* used (should always have lp_len>0) */        
39 #define LP_REDIRECT     2       /* HOT redirect (should have lp_len=0) */       
40 #define LP_DEAD         3       /* dead, may or may not have storage */   
```

En réalité il est possible de le voir avec pageinspect en affichant la colonne `lp_flags`:

```
postgres=# select lp,lp_flags,t_data,t_ctid from  heap_page_items(get_raw_page('t3',0));
 lp | lp_flags |       t_data       | t_ctid
----+----------+--------------------+--------
  1 |        2 |                    |
  2 |        1 | \x0200000002000000 | (0,2)
  3 |        1 | \x0100000005000000 | (0,3)
  4 |        1 | \x0100000004000000 | (0,3)
(4 rows)
```

Si on refait un update, suivi d'un vacuum :

```
postgres=# select lp,lp_flags,t_data,t_ctid from  heap_page_items(get_raw_page('t3',0));
 lp | lp_flags |       t_data       | t_ctid
----+----------+--------------------+--------
  1 |        2 |                    |
  2 |        1 | \x0200000002000000 | (0,2)
  3 |        0 |                    |
  4 |        0 |                    |
  5 |        1 | \x0100000006000000 | (0,5)
(5 rows)

postgres=# CHECKPOINT;

pg_filedump  11/main/base/16606/8890510

*******************************************************************
* PostgreSQL File/Block Formatted Dump Utility - Version 10.1
*
* File: 11/main/base/16606/8890510
* Options used: None
*
* Dump created on: Sun Sep  2 13:16:12 2018
*******************************************************************

Block    0 ********************************************************
<Header> -----
 Block Offset: 0x00000000         Offsets: Lower      44 (0x002c)
 Block: Size 8192  Version    4            Upper    8128 (0x1fc0)
 LSN:  logid     52 recoff 0xc39ea308      Special  8192 (0x2000)
 Items:    5                      Free Space: 8084
 Checksum: 0x0000  Prune XID: 0x00000000  Flags: 0x0005 (HAS_FREE_LINES|ALL_VISIBLE)
 Length (including item array): 44

<Data> ------
 Item   1 -- Length:    0  Offset:    5 (0x0005)  Flags: REDIRECT
 Item   2 -- Length:   32  Offset: 8160 (0x1fe0)  Flags: NORMAL
 Item   3 -- Length:    0  Offset:    0 (0x0000)  Flags: UNUSED
 Item   4 -- Length:    0  Offset:    0 (0x0000)  Flags: UNUSED
 Item   5 -- Length:   32  Offset: 8128 (0x1fc0)  Flags: NORMAL


*** End of File Encountered. Last Block Read: 0 ***
```

Il y a cependant quelques cas où le moteur ne peut pas utiliser ce mécanisme :

  * Lorsqu'il n'y a plus de place dans le bloc et qu'il doit écrire un autre bloc.
  La fragmentation de la table est ici bénéfique.
  * Un index porte sur la colonne mise à jour.

# Cas avec un index sur une colonne mise à jour

Reprenons l'exemple précédent.et rajoutons une colonne indexée :
```
postgres=# alter table t3 add column c3 int;
ALTER TABLE
postgres=# create index ON t3(c3);
CREATE INDEX
```

Les updates précédents portaient sur une colonne non-indexée. Que se passe-t-il
si l'update porte sur c3?


Etat de la table et des index avant update:
```
postgres=# select lp,lp_flags,t_data,t_ctid from  heap_page_items(get_raw_page('t3',0));
 lp | lp_flags |       t_data       | t_ctid
----+----------+--------------------+--------
  1 |        2 |                    |
  2 |        1 | \x0200000002000000 | (0,2)
  3 |        0 |                    |
  4 |        0 |                    |
  5 |        1 | \x0100000006000000 | (0,5)
(5 rows)

postgres=# select * from  bt_page_items(get_raw_page('t3_c1_idx',1));              
 itemoffset | ctid  | itemlen | nulls | vars |          data
------------+-------+---------+-------+------+-------------------------
          1 | (0,1) |      16 | f     | f    | 01 00 00 00 00 00 00 00
          2 | (0,2) |      16 | f     | f    | 02 00 00 00 00 00 00 00
(2 rows)

postgres=# select * from  bt_page_items(get_raw_page('t3_c3_idx',1));
 itemoffset | ctid  | itemlen | nulls | vars | data
------------+-------+---------+-------+------+------
          1 | (0,1) |      16 | t     | f    |
          2 | (0,2) |      16 | t     | f    |
(2 rows)
```
Pas de changement sur la table car la colonne c3 ne contient que des null, on
peut le constater en observant l'index `t3_c3_idx` où sur chaque ligne `nulls`
est à *true*.

```
postgres=# update t3 set c3 = 7 WHERE c1=1;
UPDATE 1
postgres=# select * from  bt_page_items(get_raw_page('t3_c3_idx',1));
 itemoffset | ctid  | itemlen | nulls | vars |          data
------------+-------+---------+-------+------+-------------------------
          1 | (0,3) |      16 | f     | f    | 07 00 00 00 00 00 00 00
          2 | (0,1) |      16 | t     | f    |
          3 | (0,2) |      16 | t     | f    |
(3 rows)

postgres=# select lp,lp_flags,t_data,t_ctid from  heap_page_items(get_raw_page('t3',0));
 lp | lp_flags |           t_data           | t_ctid
----+----------+----------------------------+--------
  1 |        2 |                            |
  2 |        1 | \x0200000002000000         | (0,2)
  3 |        1 | \x010000000600000007000000 | (0,3)
  4 |        0 |                            |
  5 |        1 | \x0100000006000000         | (0,3)
(5 rows)

postgres=# select * from  bt_page_items(get_raw_page('t3_c1_idx',1));              
 itemoffset | ctid  | itemlen | nulls | vars |          data
------------+-------+---------+-------+------+-------------------------
          1 | (0,3) |      16 | f     | f    | 01 00 00 00 00 00 00 00
          2 | (0,1) |      16 | f     | f    | 01 00 00 00 00 00 00 00
          3 | (0,2) |      16 | f     | f    | 02 00 00 00 00 00 00 00
(3 rows)
```
On remarque bien la nouvelle entée dans l'index portant sur c3, la table contient
bien un nouvel enregistrement. En revanche, l'index `t3_c1_idx` a également été
mis à jour entraînant l'ajout d'une troisième entrée même si la valeur de la
colonne c1 n'a pas changée.

Après un vacuum:

```
postgres=# vacuum t3;
VACUUM
postgres=# select * from  bt_page_items(get_raw_page('t3_c1_idx',1));
 itemoffset | ctid  | itemlen | nulls | vars |          data
------------+-------+---------+-------+------+-------------------------
          1 | (0,3) |      16 | f     | f    | 01 00 00 00 00 00 00 00
          2 | (0,2) |      16 | f     | f    | 02 00 00 00 00 00 00 00
(2 rows)

postgres=# select lp,lp_flags,t_data,t_ctid from  heap_page_items(get_raw_page('t3',0));
 lp | lp_flags |           t_data           | t_ctid
----+----------+----------------------------+--------
  1 |        0 |                            |
  2 |        1 | \x0200000002000000         | (0,2)
  3 |        1 | \x010000000600000007000000 | (0,3)
  4 |        0 |                            |
  5 |        0 |                            |
(5 rows)

postgres=# select * from  bt_page_items(get_raw_page('t3_c3_idx',1));
 itemoffset | ctid  | itemlen | nulls | vars |          data
------------+-------+---------+-------+------+-------------------------
          1 | (0,3) |      16 | f     | f    | 07 00 00 00 00 00 00 00
          2 | (0,2) |      16 | t     | f    |
(2 rows)
```

Le moteur a nettoyé les index et la table. La première ligne de la table n'a plus
le flag `REDIRECT`.

FIXME : pourquoi ça ne marche pas si un index porte sur la colonne?

# Nouveauté version 11 heap-only-tuple (HOT) avec index Fonctionnels



Lorsqu'un index fonctionnel porte sur la colonne modifiée, il peut arriver que
le résultat de l'expression reste inchangé malgré la mise à jour de la colonne.

Prenons un exemple : un index fonctionnel sur une *clé spécifique* d'un objet JSON.

```SQL
postgres=# create table t4 (c1 jsonb, c2 int,c3 int);
CREATE TABLE
postgres=# -- CREATE index ON t4 ((c1->>'prenom'))  WITH (recheck_on_update='false');
postgres=# CREATE index ON t4 ((c1->>'prenom')) ;
CREATE INDEX
postgres=# CREATE index ON t4 (c2);
CREATE INDEX
postgres=# insert into t4 values ('{ "prenom":"adrien" , "ville" : "valence"}'::jsonb,1,1);
INSERT 0 1
postgres=# insert into t4 values ('{ "prenom":"guillaume" , "ville" : "lille"}'::jsonb,2,2);
INSERT 0 1
postgres=#
postgres=#
postgres=# -- changement qui ne porte pas sur prenom
postgres=#  update t4 set c1 = '{"ville": "valence (#soleil)", "prenom": "guillaume"}' WHERE c2=2;
UPDATE 1
postgres=# select pg_stat_get_xact_tuples_hot_updated('t4'::regclass);
 pg_stat_get_xact_tuples_hot_updated
-------------------------------------
                                   0
(1 row)

postgres=# update t4 set c1 = '{"ville": "nantes", "prenom": "guillaume"}' WHERE c2=2;
UPDATE 1
postgres=# select pg_stat_get_xact_tuples_hot_updated('t4'::regclass);
 pg_stat_get_xact_tuples_hot_updated
-------------------------------------
                                   0
(1 row)
```

Les deux updates n'ont fait que modifier la clé "ville" et pas la clé "prenom".
Ce qui n’entraîne pas de modification de l'index car il n'indexe que la clé prénom.

Avec la version 11, le moteur est capable de constater que la modification ne porte
pas sur l'expression :

```SQL
postgres=# create table t4 (c1 jsonb, c2 int,c3 int);
CREATE TABLE
postgres=# -- CREATE index ON t4 ((c1->>'prenom'))  WITH (recheck_on_update='false');
postgres=# CREATE index ON t4 ((c1->>'prenom')) ;
CREATE INDEX
postgres=# CREATE index ON t4 (c2);
CREATE INDEX
postgres=# insert into t4 values ('{ "prenom":"adrien" , "ville" : "valence"}'::jsonb,1,1);
INSERT 0 1
postgres=# insert into t4 values ('{ "prenom":"guillaume" , "ville" : "lille"}'::jsonb,2,2);
INSERT 0 1
postgres=#
postgres=#
postgres=# -- changement qui ne porte pas sur prenom
postgres=#  update t4 set c1 = '{"ville": "valence (#soleil)", "prenom": "guillaume"}' WHERE c2=2;
UPDATE 1
postgres=# select pg_stat_get_xact_tuples_hot_updated('t4'::regclass);
 pg_stat_get_xact_tuples_hot_updated
-------------------------------------
                                   1
(1 row)

postgres=# update t4 set c1 = '{"ville": "nantes", "prenom": "guillaume"}' WHERE c2=2;
UPDATE 1
postgres=# select pg_stat_get_xact_tuples_hot_updated('t4'::regclass);
 pg_stat_get_xact_tuples_hot_updated
-------------------------------------
                                   2
(1 row)
```


FIXME : corriger la suite et éventuellement faire un bench pour montrer l'impact
sur les performances en lecture et insertion. Les lectures devraient être plus
performantes car l'index bloate moins.

Et dans le cas d'un index fonctionnel? Il est trop coûteux de verifier à chaque mise à jour si l'index doit être mis à jour.
Par exemple, un index fonctionnel qui n'indexe que certaines clés d'un objet JSON :

```SQL

begin;
drop table IF EXISTS t4;
create table t4 (c1 jsonb, c2 int,c3 int);
-- CREATE index ON t4 ((c1->>'prenom'))  WITH (recheck_on_update='false');
CREATE index ON t4 ((c1->>'prenom')) ;
CREATE index ON t4 (c2);
insert into t4 values ('{ "prenom":"adrien" , "ville" : "valence"}'::jsonb,1,1);
insert into t4 values ('{ "prenom":"guillaume" , "ville" : "lille"}'::jsonb,2,2);


-- changement qui ne porte pas sur prenom
 update t4 set c1 = '{"ville": "valence (#soleil)", "prenom": "guillaume"}' WHERE c2=2;
select pg_stat_get_xact_tuples_hot_updated('t4'::regclass);
update t4 set c1 = '{"ville": "nantes", "prenom": "guillaume"}' WHERE c2=2;
select pg_stat_get_xact_tuples_hot_updated('t4'::regclass);

rollback;

\timing
CREATE OR REPLACE FUNCTION upper_ville_field(obj jsonb)
RETURNS jsonb
AS $$
select pg_sleep(5);
select jsonb_set($1, '{"prenom"}',a::jsonb, true) from (select to_jsonb(upper($1->>'prenom'))) a(a);
$$
LANGUAGE SQL IMMUTABLE COST 1100;

begin;
create table t4 (c1 jsonb, c2 int,c3 int);
CREATE index ON t4 ((upper_ville_field(c1)->>'prenom')) ;
-- CREATE index ON t4 ((upper_ville_field(c1)->>'prenom')) WITH (recheck_on_update=œ'true');;

CREATE index ON t4 (c2);
insert into t4 values ('{ "prenom":"adrien" , "ville" : "valence"}'::jsonb,1,1);
insert into t4 values ('{ "prenom":"guillaume" , "ville" : "lille"}'::jsonb,2,2);
select * from t4;

-- changement qui ne porte pas sur prenom
update t4 set c1 = '{"ville": "valence (#soleil)", "prenom": "guillaume"}' WHERE c2=2;
select * from t4;
select pg_stat_get_xact_tuples_hot_updated('t4'::regclass);
update t4 set c1 = '{"ville": "nantes", "prenom": "guillaume"}' WHERE c2=2;
select pg_stat_get_xact_tuples_hot_updated('t4'::regclass);


-- changement qui ne porte pas sur prenom
 update t4 set c1 = '{"ville": "lillgegre", "prenom": "guillaume"}' WHERE c2=2;

```
