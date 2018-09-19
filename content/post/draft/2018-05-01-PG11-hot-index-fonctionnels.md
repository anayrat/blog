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
# SET `preview` to `false` to disable the thumbnail in listings.
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

Prenons une table toute simple et regardons son contenu évoluer à l'aide de l'extension pageinspect :

```
CREATE TABLE t2(c1 int);
CREATE TABLE
INSERT INTO t2 VALUES (1);
INSERT 0 1
SELECT lp,t_data FROM  heap_page_items(get_raw_page('t2',0));
 lp |   t_data   
----+------------
  1 | \x01000000
(1 row)

UPDATE t2 SET c1 = 2 WHERE c1 = 1;
UPDATE 1
SELECT lp,t_data FROM  heap_page_items(get_raw_page('t2',0));
 lp |   t_data   
----+------------
  1 | \x01000000
  2 | \x02000000
(2 rows)

vacuum t2;
VACUUM
SELECT lp,t_data FROM  heap_page_items(get_raw_page('t2',0));
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
CREATE TABLE t3(c1 int,c2 int);
CREATE INDEX ON t3(c1);
INSERT INTO t3(c1,c2) VALUES (1,1);
INSERT INTO t3(c1,c2) VALUES (2,2);
SELECT ctid,* FROM t3;
 ctid  | c1 | c2
-------+----+----
 (0,1) |  1 |  1
 (0,2) |  2 |  2
(2 rows)

```

De la même façon qu'on peut lire les blocs d'une table, on peut lire les blocs
d'un index avec pageinspect :

```
SELECT * FROM  bt_page_items(get_raw_page('t3_c1_idx',1));
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
UPDATE t3 SET c2 = 3 WHERE c1=1;
UPDATE 1
SELECT lp,t_data,t_ctid FROM  heap_page_items(get_raw_page('t3',0));
 lp |       t_data       | t_ctid
----+--------------------+--------
  1 | \x0100000001000000 | (0,3)
  2 | \x0200000002000000 | (0,2)
  3 | \x0100000003000000 | (0,3)
(3 rows)


SELECT * FROM  bt_page_items(get_raw_page('t3_c1_idx',1));
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
pointerait vers (0,4), formant ce qu'on appelle une chaîne. Un vacuum nettoierait
les espaces libres mais conserverait toujours la première ligne qui pointera vers
le dernier enregistrement.


On modifie une ligne et l'index ne change toujours pas:
```
UPDATE t3 SET c2 = 4 WHERE c1=1;
SELECT * FROM  bt_page_items(get_raw_page('t3_c1_idx',1));
 itemoffset | ctid  | itemlen | nulls | vars |          data           
------------+-------+---------+-------+------+-------------------------
          1 | (0,1) |      16 | f     | f    | 01 00 00 00 00 00 00 00
          2 | (0,2) |      16 | f     | f    | 02 00 00 00 00 00 00 00
(2 rows)

SELECT lp,t_data,t_ctid FROM  heap_page_items(get_raw_page('t3',0));
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
vacuum t3;
VACUUM
SELECT lp,t_data,t_ctid FROM  heap_page_items(get_raw_page('t3',0));
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
UPDATE t3 SET c2 = 5 WHERE c1=1;
UPDATE 1
SELECT lp,t_data,t_ctid FROM  heap_page_items(get_raw_page('t3',0));
 lp |       t_data       | t_ctid
----+--------------------+--------
  1 |                    |
  2 | \x0200000002000000 | (0,2)
  3 | \x0100000005000000 | (0,3)
  4 | \x0100000004000000 | (0,3)
(4 rows)


SELECT * FROM  bt_page_items(get_raw_page('t3_c1_idx',1));
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
pas être encore écrit sur le disque.

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

La première ligne contient `Flags: REDIRECT`. Ceci indique que cette ligne
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

En réalité, il est possible de le voir avec pageinspect en affichant la colonne `lp_flags`:

```
SELECT lp,lp_flags,t_data,t_ctid FROM  heap_page_items(get_raw_page('t3',0));
 lp | lp_flags |       t_data       | t_ctid
----+----------+--------------------+--------
  1 |        2 |                    |
  2 |        1 | \x0200000002000000 | (0,2)
  3 |        1 | \x0100000005000000 | (0,3)
  4 |        1 | \x0100000004000000 | (0,3)
(4 rows)
```

Si on refait un update, suivi d'un CHECKPOINT pour écrire le bloc sur disque :

```
SELECT lp,lp_flags,t_data,t_ctid FROM  heap_page_items(get_raw_page('t3',0));
 lp | lp_flags |       t_data       | t_ctid
----+----------+--------------------+--------
  1 |        2 |                    |
  2 |        1 | \x0200000002000000 | (0,2)
  3 |        0 |                    |
  4 |        0 |                    |
  5 |        1 | \x0100000006000000 | (0,5)
(5 rows)

CHECKPOINT;

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
  * Un index porte sur la colonne mise à jour. Dans ce cas le moteur doit mettre
  à jour l'index. Le moteur peut détecter s'il y a eu un changement
  en effectuant une comparaison binaire entre la nouvelle valeur et la précédente [^1].

# Cas avec un index sur une colonne mise à jour

Reprenons l'exemple précédent et rajoutons une colonne indexée :
```
alter table t3 add column c3 int;
ALTER TABLE
CREATE INDEX ON t3(c3);
CREATE INDEX
```

Les updates précédents portaient sur une colonne non-indexée. Que se passe-t-il
si l'update porte sur c3?


Etat de la table et des index avant update:
```
SELECT lp,lp_flags,t_data,t_ctid FROM  heap_page_items(get_raw_page('t3',0));
 lp | lp_flags |       t_data       | t_ctid
----+----------+--------------------+--------
  1 |        2 |                    |
  2 |        1 | \x0200000002000000 | (0,2)
  3 |        0 |                    |
  4 |        0 |                    |
  5 |        1 | \x0100000006000000 | (0,5)
(5 rows)

SELECT * FROM  bt_page_items(get_raw_page('t3_c1_idx',1));              
 itemoffset | ctid  | itemlen | nulls | vars |          data
------------+-------+---------+-------+------+-------------------------
          1 | (0,1) |      16 | f     | f    | 01 00 00 00 00 00 00 00
          2 | (0,2) |      16 | f     | f    | 02 00 00 00 00 00 00 00
(2 rows)

SELECT * FROM  bt_page_items(get_raw_page('t3_c3_idx',1));
 itemoffset | ctid  | itemlen | nulls | vars | data
------------+-------+---------+-------+------+------
          1 | (0,1) |      16 | t     | f    |
          2 | (0,2) |      16 | t     | f    |
(2 rows)
```
Pas de changement sur la table car la colonne c3 ne contient que des null, on
peut le constater en observant l'index `t3_c3_idx` où `nulls` est à *true* sur
chaque ligne.

```
UPDATE t3 SET c3 = 7 WHERE c1=1;
UPDATE 1
SELECT * FROM  bt_page_items(get_raw_page('t3_c3_idx',1));
 itemoffset | ctid  | itemlen | nulls | vars |          data
------------+-------+---------+-------+------+-------------------------
          1 | (0,3) |      16 | f     | f    | 07 00 00 00 00 00 00 00
          2 | (0,1) |      16 | t     | f    |
          3 | (0,2) |      16 | t     | f    |
(3 rows)

SELECT lp,lp_flags,t_data,t_ctid FROM  heap_page_items(get_raw_page('t3',0));
 lp | lp_flags |           t_data           | t_ctid
----+----------+----------------------------+--------
  1 |        2 |                            |
  2 |        1 | \x0200000002000000         | (0,2)
  3 |        1 | \x010000000600000007000000 | (0,3)
  4 |        0 |                            |
  5 |        1 | \x0100000006000000         | (0,3)
(5 rows)

SELECT * FROM  bt_page_items(get_raw_page('t3_c1_idx',1));              
 itemoffset | ctid  | itemlen | nulls | vars |          data
------------+-------+---------+-------+------+-------------------------
          1 | (0,3) |      16 | f     | f    | 01 00 00 00 00 00 00 00
          2 | (0,1) |      16 | f     | f    | 01 00 00 00 00 00 00 00
          3 | (0,2) |      16 | f     | f    | 02 00 00 00 00 00 00 00
(3 rows)
```
On remarque bien la nouvelle entrée dans l'index portant sur c3. La table contient
bien un nouvel enregistrement. En revanche, l'index `t3_c1_idx` a également été
mis à jour entraînant l'ajout d'une troisième entrée même si la valeur de la
colonne c1 n'a pas changée.

Après un vacuum:

```
vacuum t3;
VACUUM
SELECT * FROM  bt_page_items(get_raw_page('t3_c1_idx',1));
 itemoffset | ctid  | itemlen | nulls | vars |          data
------------+-------+---------+-------+------+-------------------------
          1 | (0,3) |      16 | f     | f    | 01 00 00 00 00 00 00 00
          2 | (0,2) |      16 | f     | f    | 02 00 00 00 00 00 00 00
(2 rows)

SELECT lp,lp_flags,t_data,t_ctid FROM  heap_page_items(get_raw_page('t3',0));
 lp | lp_flags |           t_data           | t_ctid
----+----------+----------------------------+--------
  1 |        0 |                            |
  2 |        1 | \x0200000002000000         | (0,2)
  3 |        1 | \x010000000600000007000000 | (0,3)
  4 |        0 |                            |
  5 |        0 |                            |
(5 rows)

SELECT * FROM  bt_page_items(get_raw_page('t3_c3_idx',1));
 itemoffset | ctid  | itemlen | nulls | vars |          data
------------+-------+---------+-------+------+-------------------------
          1 | (0,3) |      16 | f     | f    | 07 00 00 00 00 00 00 00
          2 | (0,2) |      16 | t     | f    |
(2 rows)
```

Le moteur a nettoyé les index et la table. La première ligne de la table n'a plus
le flag `REDIRECT`.



# Nouveauté version 11 heap-only-tuple (HOT) avec index fonctionnels



Lorsqu'un index fonctionnel porte sur la colonne modifiée, il peut arriver que
le résultat de l'expression reste inchangé malgré la mise à jour de la colonne.
La clé dans l'index serait donc inchangé.

Prenons un exemple : un index fonctionnel sur une *clé spécifique* d'un objet JSON.

```SQL
CREATE TABLE t4 (c1 jsonb, c2 int,c3 int);
CREATE TABLE
-- CREATE INDEX ON t4 ((c1->>'prenom'))  WITH (recheck_on_update='false');
CREATE INDEX ON t4 ((c1->>'prenom')) ;
CREATE INDEX
CREATE INDEX ON t4 (c2);
CREATE INDEX
INSERT INTO t4 VALUES ('{ "prenom":"adrien" , "ville" : "valence"}'::jsonb,1,1);
INSERT 0 1
INSERT INTO t4 VALUES ('{ "prenom":"guillaume" , "ville" : "lille"}'::jsonb,2,2);
INSERT 0 1


-- changement qui ne porte pas sur prenom
UPDATE t4 SET c1 = '{"ville": "valence (#soleil)", "prenom": "guillaume"}' WHERE c2=2;
UPDATE 1
SELECT pg_stat_get_xact_tuples_hot_updated('t4'::regclass);
 pg_stat_get_xact_tuples_hot_updated
-------------------------------------
                                   0
(1 row)

UPDATE t4 SET c1 = '{"ville": "nantes", "prenom": "guillaume"}' WHERE c2=2;
UPDATE 1
SELECT pg_stat_get_xact_tuples_hot_updated('t4'::regclass);
 pg_stat_get_xact_tuples_hot_updated
-------------------------------------
                                   0
(1 row)
```

La fonction `pg_stat_get_xact_tuples_hot_updated` indique le nombre de ligne mises
à jour pas le mécanisme HOT.

Les deux updates n'ont fait que modifier la clé "ville" et pas la clé "prenom".
Ce qui n’entraîne pas de modification de l'index car il n'indexe que la clé prénom.

Le moteur n'a pas pu faire d'HOT. En effet, pour lui l'update a porté sur la colonne et
l'index doit être mis à jour.

Avec la version 11, le moteur est capable de constater que le résultat de
l'expression ne change pas. Effectuons le même test sur la version 11 :

```SQL
CREATE TABLE t4 (c1 jsonb, c2 int,c3 int);
CREATE TABLE
-- CREATE INDEX ON t4 ((c1->>'prenom'))  WITH (recheck_on_update='false');
CREATE INDEX ON t4 ((c1->>'prenom')) ;
CREATE INDEX
CREATE INDEX ON t4 (c2);
CREATE INDEX
INSERT INTO t4 VALUES ('{ "prenom":"adrien" , "ville" : "valence"}'::jsonb,1,1);
INSERT 0 1
INSERT INTO t4 VALUES ('{ "prenom":"guillaume" , "ville" : "lille"}'::jsonb,2,2);
INSERT 0 1


-- changement qui ne porte pas sur prenom
UPDATE t4 SET c1 = '{"ville": "valence (#soleil)", "prenom": "guillaume"}' WHERE c2=2;
UPDATE 1
SELECT pg_stat_get_xact_tuples_hot_updated('t4'::regclass);
 pg_stat_get_xact_tuples_hot_updated
-------------------------------------
                                   1
(1 row)

UPDATE t4 SET c1 = '{"ville": "nantes", "prenom": "guillaume"}' WHERE c2=2;
UPDATE 1
SELECT pg_stat_get_xact_tuples_hot_updated('t4'::regclass);
 pg_stat_get_xact_tuples_hot_updated
-------------------------------------
                                   2
(1 row)
```

Cette fois le moteur a bien utilisé le mécanisme HOT. On peut le vérifier en
regardant le contenu physique de l'index avec pageinspect :

Version 10 :
```
SELECT * FROM  bt_page_items(get_raw_page('t4_expr_idx',1));
itemoffset | ctid  | itemlen | nulls | vars |                      data                       
------------+-------+---------+-------+------+-------------------------------------------------
         1 | (0,1) |      16 | f     | t    | 0f 61 64 72 69 65 6e 00
         2 | (0,4) |      24 | f     | t    | 15 67 75 69 6c 6c 61 75 6d 65 00 00 00 00 00 00
         3 | (0,3) |      24 | f     | t    | 15 67 75 69 6c 6c 61 75 6d 65 00 00 00 00 00 00
         4 | (0,2) |      24 | f     | t    | 15 67 75 69 6c 6c 61 75 6d 65 00 00 00 00 00 00
(4 rows)
```

Version 11 :

```
SELECT * FROM  bt_page_items(get_raw_page('t4_expr_idx',1));
itemoffset | ctid  | itemlen | nulls | vars |                      data
------------+-------+---------+-------+------+-------------------------------------------------
         1 | (0,1) |      16 | f     | t    | 0f 61 64 72 69 65 6e 00
         2 | (0,2) |      24 | f     | t    | 15 67 75 69 6c 6c 61 75 6d 65 00 00 00 00 00 00
(2 rows)
```

Ce comportement peut se contrôler grace à une nouvelle option lors de la création
de l'index : `recheck_on_update`.

Par défaut à `on`, le moteur effectue la vérification du résultat de l'expression
pour faire un update HOT. On peut le paramétrer à `off` s'il y a de fortes chances
pour que le résultat de l'expression change lors d'un update. Cela permet d'éviter
d'exécuter l'expression inutilement.

A noté également que le moteur évite l'évaluation de l'expression si sont coût est
supérieur à 1000. FIXME : a tester, voir cas où expression est coûteuse.

# Impact sur les performances

Voici un test assez simple pour mettre en évidence l'intérêt de cette fonctionnalité.
On pourrait s'attendre à des gains en terme de performances pure car le moteur évite
de mettre à jour les index. Ainsi qu'en terme de taille d'index, comme vu précédemment
on évite la fragmentation.

FIXME : c'est tout faux!

```sql
CREATE TABLE t5 (c1 jsonb, c2 int,c3 int);
CREATE INDEX ON t5 ((c1->>'prenom')) ;
CREATE INDEX ON t5 (c2);
INSERT INTO t5 VALUES ('{ "prenom":"adrien" , "valeur" : "1"}'::jsonb,1,1);
INSERT INTO t5 VALUES ('{ "prenom":"guillaume" , "valeur" : "2"}'::jsonb,2,2);
\dt+ t5
                   List of relations
 Schema | Name | Type  |  Owner   | Size  | Description
--------+------+-------+----------+-------+-------------
 public | t5   | table | postgres | 16 kB |
(1 row)

\di+ t5*
                           List of relations
 Schema |    Name     | Type  |  Owner   | Table | Size  | Description
--------+-------------+-------+----------+-------+-------+-------------
 public | t5_c2_idx   | index | postgres | t5    | 16 kB |
 public | t5_expr_idx | index | postgres | t5    | 16 kB |
(2 rows)
```

Puis ce test pgbench :

```
\set id  random(1, 100000)
\set id2  random(1, 100000)

UPDATE t5 SET c1 = '{"valeur": ":id", "prenom": "guillaume"}' WHERE c2=2;
UPDATE t5 SET c1 = '{"valeur": ":id2", "prenom": "adrien"}' WHERE c2=1;
```

Qu'on exécute pendant 60 secondes :

Avec `recheck_on_update=on` (par défaut):

```
pgbench -f test.sql -n -c6 -T 120
transaction type: test.sql
scaling factor: 1
query mode: simple
number of clients: 6
number of threads: 1
duration: 120 s
number of transactions actually processed: 2743163
latency average = 0.262 ms
tps = 22859.646914 (including connections establishing)
tps = 22859.938191 (excluding connections establishing)

 \dt+ t5*
                    List of relations
 Schema | Name | Type  |  Owner   |  Size  | Description
--------+------+-------+----------+--------+-------------
 public | t5   | table | postgres | 376 kB |
(1 row)
\di+ t5*
                           List of relations
 Schema |    Name     | Type  |  Owner   | Table | Size  | Description
--------+-------------+-------+----------+-------+-------+-------------
 public | t5_c2_idx   | index | postgres | t5    | 16 kB |
 public | t5_expr_idx | index | postgres | t5    | 32 kB |
(2 rows)

select * from pg_stat_user_tables where relname = 't5';
-[ RECORD 1 ]-------+------------------------------
relid               | 8890622
schemaname          | public
relname             | t5
seq_scan            | 4
seq_tup_read        | 0
idx_scan            | 7999055
idx_tup_fetch       | 7999055
n_tup_ins           | 4
n_tup_upd           | 7999055
n_tup_del           | 0
n_tup_hot_upd       | 7998236
n_live_tup          | 2
n_dead_tup          | 0
n_mod_since_analyze | 0
last_vacuum         |
last_autovacuum     | 2018-09-19 06:29:37.690575+00
last_analyze        |
last_autoanalyze    | 2018-09-19 06:29:37.719911+00
vacuum_count        | 0
autovacuum_count    | 5
analyze_count       | 0
autoanalyze_count   | 5

```

Avec `recheck_on_update=off` :

Même jeu de donnée que précédemment mais cette fois l'index est créé avec cet ordre :
`CREATE INDEX ON t5 ((c1->>'prenom')) WITH  (recheck_on_update=off);`


```
pgbench -f test.sql -n -c6 -T 120
transaction type: test.sql
scaling factor: 1
query mode: simple
number of clients: 6
number of threads: 1
duration: 120 s
number of transactions actually processed: 1065688
latency average = 0.676 ms
tps = 8880.679565 (including connections establishing)
tps = 8880.796478 (excluding connections establishing)

\dt+ t5
                    List of relations
 Schema | Name | Type  |  Owner   |  Size   | Description
--------+------+-------+----------+---------+-------------
 public | t5   | table | postgres | 9496 kB |
(1 row)

\di+ t5*
                           List of relations
 Schema |    Name     | Type  |  Owner   | Table |  Size  | Description
--------+-------------+-------+----------+-------+--------+-------------
 public | t5_c2_idx   | index | postgres | t5    | 768 kB |
 public | t5_expr_idx | index | postgres | t5    | 58 MB  |
(2 rows)

select * from pg_stat_user_tables where relname = 't5';
-[ RECORD 1 ]-------+------------------------------
relid               | 8890635
schemaname          | public
relname             | t5
seq_scan            | 2
seq_tup_read        | 0
idx_scan            | 2131376
idx_tup_fetch       | 2131376
n_tup_ins           | 2
n_tup_upd           | 2131376
n_tup_del           | 0
n_tup_hot_upd       | 19
n_live_tup          | 2
n_dead_tup          | 0
n_mod_since_analyze | 0
last_vacuum         |
last_autovacuum     | 2018-09-19 06:34:42.045905+00
last_analyze        |
last_autoanalyze    | 2018-09-19 06:34:42.251183+00
vacuum_count        | 0
autovacuum_count    | 3
analyze_count       | 0
autoanalyze_count   | 3
```

L'écart de performance est assez impressionnant de même que la taille des tables
et index.


J'ai refait le premier test en désactivant l'autovacuum et voici le résultat :

```
pgbench -f test.sql -n -c6 -T 120
transaction type: test.sql
scaling factor: 1
query mode: simple
number of clients: 6
number of threads: 1
duration: 120 s
number of transactions actually processed: 2752479
latency average = 0.262 ms
tps = 22937.271749 (including connections establishing)
tps = 22937.545872 (excluding connections establishing)


select * from pg_stat_user_tables where relname = 't5';
-[ RECORD 1 ]-------+--------
relid               | 8890643
schemaname          | public
relname             | t5
seq_scan            | 2
seq_tup_read        | 0
idx_scan            | 5504958
idx_tup_fetch       | 5504958
n_tup_ins           | 2
n_tup_upd           | 5504958
n_tup_del           | 0
n_tup_hot_upd       | 5504258
n_live_tup          | 2
n_dead_tup          | 2416
n_mod_since_analyze | 5504960
last_vacuum         |
last_autovacuum     |
last_analyze        |
last_autoanalyze    |
vacuum_count        | 0
autovacuum_count    | 0
analyze_count       | 0
autoanalyze_count   | 0

postgres=# \di+ t5*
List of relations
-[ RECORD 1 ]------------
Schema      | public
Name        | t5_c2_idx
Type        | index
Owner       | postgres
Table       | t5
Size        | 16 kB
Description |
-[ RECORD 2 ]------------
Schema      | public
Name        | t5_expr_idx
Type        | index
Owner       | postgres
Table       | t5
Size        | 40 kB
Description |

postgres=# \dt+ t5
List of relations
-[ RECORD 1 ]---------
Schema      | public
Name        | t5
Type        | table
Owner       | postgres
Size        | 1080 kB
Description |
```

Puis le second test :


```
pgbench -f test.sql -n -c6 -T 120
transaction type: test.sql
scaling factor: 1
query mode: simple
number of clients: 6
number of threads: 1
duration: 120 s
number of transactions actually processed: 881434
latency average = 0.817 ms
tps = 7345.208875 (including connections establishing)
tps = 7345.304797 (excluding connections establishing)

select * from pg_stat_user_tables where relname = 't5';
-[ RECORD 1 ]-------+--------
relid               | 8890651
schemaname          | public
relname             | t5
seq_scan            | 2
seq_tup_read        | 0
idx_scan            | 1762868
idx_tup_fetch       | 1762868
n_tup_ins           | 2
n_tup_upd           | 1762868
n_tup_del           | 0
n_tup_hot_upd       | 23
n_live_tup          | 2
n_dead_tup          | 1762845
n_mod_since_analyze | 1762870
last_vacuum         |
last_autovacuum     |
last_analyze        |
last_autoanalyze    |
vacuum_count        | 0
autovacuum_count    | 0
analyze_count       | 0
autoanalyze_count   | 0

postgres=# \di+ t5*
List of relations
-[ RECORD 1 ]------------
Schema      | public
Name        | t5_c2_idx
Type        | index
Owner       | postgres
Table       | t5
Size        | 600 kB
Description |
-[ RECORD 2 ]------------
Schema      | public
Name        | t5_expr_idx
Type        | index
Owner       | postgres
Table       | t5
Size        | 56 MB
Description |

postgres=# \dt+ t5*
List of relations
-[ RECORD 1 ]---------
Schema      | public
Name        | t5
Type        | table
Owner       | postgres
Size        | 55 MB
Description |
```

A nouveau l'écart de performance est important, il en est de même pour la taille
des tables et index. On voit ici aussi l'importance de laisser l'autovacuum activé.

FIXME : expliquer l'écart de la taille de la table.

# Cas où l'expression est coûteuse

FIXME : corriger la suite et éventuellement faire un bench pour montrer l'impact
sur les performances en lecture et insertion. Les lectures devraient être plus
performantes car l'index bloate moins.

Et dans le cas d'un index fonctionnel? Il est trop coûteux de verifier à chaque mise à jour si l'index doit être mis à jour.
Par exemple, un index fonctionnel qui n'indexe que certaines clés d'un objet JSON :

```SQL

BEGIN;
DROP TABLE IF EXISTS t4;

CREATE TABLE t4 (c1 jsonb, c2 int,c3 int);
-- CREATE INDEX ON t4 ((c1->>'prenom'))  WITH (recheck_on_update='false');
CREATE INDEX ON t4 ((c1->>'prenom')) ;
CREATE INDEX ON t4 (c2);
INSERT INTO t4 VALUES ('{ "prenom":"adrien" , "ville" : "valence"}'::jsonb,1,1);
INSERT INTO t4 VALUES ('{ "prenom":"guillaume" , "ville" : "lille"}'::jsonb,2,2);


-- changement qui ne porte pas sur prenom
UPDATE t4 SET c1 = '{"ville": "valence (#soleil)", "prenom": "guillaume"}' WHERE c2=2;
SELECT pg_stat_get_xact_tuples_hot_updated('t4'::regclass);
UPDATE t4 SET c1 = '{"ville": "nantes", "prenom": "guillaume"}' WHERE c2=2;
SELECT pg_stat_get_xact_tuples_hot_updated('t4'::regclass);

ROLLBACK;

\timing
CREATE OR REPLACE FUNCTION upper_ville_field(obj jsonb)
RETURNS jsonb
AS $$
SELECT pg_sleep(5);
SELECT jsonb_set($1, '{"prenom"}',a::jsonb, true) FROM (SELECT to_jsonb(upper($1->>'prenom'))) a(a);
$$
LANGUAGE SQL IMMUTABLE COST 1100;

BEGIN;
CREATE TABLE t4 (c1 jsonb, c2 int,c3 int);
CREATE INDEX ON t4 ((upper_ville_field(c1)->>'prenom')) ;
-- CREATE INDEX ON t4 ((upper_ville_field(c1)->>'prenom')) WITH (recheck_on_update=œ'true');;

CREATE INDEX ON t4 (c2);
INSERT INTO t4 VALUES ('{ "prenom":"adrien" , "ville" : "valence"}'::jsonb,1,1);
INSERT INTO t4 VALUES ('{ "prenom":"guillaume" , "ville" : "lille"}'::jsonb,2,2);
SELECT * FROM t4;

-- changement qui ne porte pas sur prenom
UPDATE t4 SET c1 = '{"ville": "valence (#soleil)", "prenom": "guillaume"}' WHERE c2=2;
SELECT * FROM t4;
SELECT pg_stat_get_xact_tuples_hot_updated('t4'::regclass);
UPDATE t4 SET c1 = '{"ville": "nantes", "prenom": "guillaume"}' WHERE c2=2;
SELECT pg_stat_get_xact_tuples_hot_updated('t4'::regclass);


-- changement qui ne porte pas sur prenom
UPDATE t4 SET c1 = '{"ville": "lillgegre", "prenom": "guillaume"}' WHERE c2=2;
```

[^1]: [https://git.postgresql.org/gitweb/?p=postgresql.git;a=blob;f=src/backend/access/heap/README.HOT;h=4cf3c3a0d4c2db96a57e73e46fdd7463db439f79;hb=HEAD#l128]
