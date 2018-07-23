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

Durant le développement de cette version, une fonctionnalité a attiré mon attention. On peut la retrouver dans les releases notes. Même si cette version n'est pas encore officiellement sortie, on peut avoir accès à la documentation : <https://www.postgresql.org/docs/11/static/release-11.html>


> Allow heap-only-tuple (HOT) updates for expression indexes when the values of the expressions are unchanged (Konstantin Knizhnik)

J'avoue que ce n'est pas très explicite et cette fonctionnalité nécessite quelques connaissances sur le fonctionnement du moteur que je vais essayer d'expliquer.

# MVCC

Dans l'implémentation MVCC de Postgres, le moteur ne mets pas à jour une ligne directement. Il duplique l'enregistrement et renseigne des informations de visibilité.

Pourquoi ce fonctionnement?

Il y a une composante capitale à prendre en compte quand on travaille avec un SGBG : la concurrence d'accès.

La ligne que vous êtes en train de modifier est peut être utile par une transaction antérieure. Une sauvegarde en cours par exemple :)

Pour cela les SGBD ont adopté différentes techniques :

  * Modifier l'enregistrement et stocker les versions antérieures sur un autre emplacement. C'est ce que fait oracle par exemple.
  * Dupliquer l'enregistrement et stocker des informations de visibilité pour savoir quelle ligne est visible par telle ou telle transaction. Cela nécessite d'avoir un mécanisme de nettoyage des lignes qui ne plus visibles par personne. C'est l'implémentation dans Postgres et le vacuum a pour rôle d'effectuer ce nettoyage.

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

Prenons un autre cas, un peu plus compliqué, une table avec deux colonnes et un index sur une des deux colonnes :

```
create table t3(c1 int,c2 int)
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

De la même façon qu'on peut lire les blocs d'une table, on peut lire les blocs d'un index avec pageinspect:

```
select * from  bt_page_items(get_raw_page('t3_c1_idx',1));
 itemoffset | ctid  | itemlen | nulls | vars |          data           
------------+-------+---------+-------+------+-------------------------
          1 | (0,1) |      16 | f     | f    | 01 00 00 00 00 00 00 00
          2 | (0,2) |      16 | f     | f    | 02 00 00 00 00 00 00 00
(2 rows)
```

Jusque là c'est assez simple, ma table contient deux enregistrements et l'index
contient également deux enregistrements pointant vers les blocs correspondants de la table (colonne ctid).

Si je mets à jour la colonne c1, avec la nouvelle valeur 3 par exemple, l'index devra être mis à jour.

Maintenant si je mets à jour la colonne c2. Est-ce que l'index portant sur c1 sera mis à jour?

Au premier abord on pourrait se dire non car c1 n'est pas mise en jour.

Mais à cause du modèle MVCC présenté plus haut, en théorie, la réponse sera oui : nous venons de voir plus haut que le moteur va dupliquer la ligne, son emplacement physique sera donc différent (le ctid suivant sera (0,3)).

Vérifions le :

```
update t3 set c2 = 3 WHERE c1=1;
UPDATE 1
postgres=# select * from  heap_page_items(get_raw_page('t3',0));
 lp | lp_off | lp_flags | lp_len | t_xmin | t_xmax | t_field3 | t_ctid | t_infomask2 | t_infomask | t_hoff | t_bits | t_oid |       t_data       
----+--------+----------+--------+--------+--------+----------+--------+-------------+------------+--------+--------+-------+--------------------
  1 |   8160 |        1 |     32 |   1221 |   1223 |        0 | (0,3)  |       16386 |        256 |     24 |        |       | \x0100000001000000
  2 |   8128 |        1 |     32 |   1222 |      0 |        0 | (0,2)  |           2 |       2048 |     24 |        |       | \x0200000002000000
  3 |   8096 |        1 |     32 |   1223 |      0 |        0 | (0,3)  |       32770 |      10240 |     24 |        |       | \x0100000003000000
(3 rows)

postgres=# select * from  bt_page_items(get_raw_page('t3_c1_idx',1));
 itemoffset | ctid  | itemlen | nulls | vars |          data           
------------+-------+---------+-------+------+-------------------------
          1 | (0,1) |      16 | f     | f    | 01 00 00 00 00 00 00 00
          2 | (0,2) |      16 | f     | f    | 02 00 00 00 00 00 00 00
(2 rows)
```

La lecture du bloc de la table confirme bien que la ligne a été dupliquée. En observant attentivement le champ `t_data`, on arrive à distinguer le 1 de la colonne c1 et le 3 de la colonne c2.

En lisant le bloc de l'index, on constate que son contenu n'a pas bougé! Si je vais chercher la ligne WHERE c1 = 1, l'index m'oriente vers l'enregistrement (0,1) qui correspond à l'ancienne ligne!

Que s'est-il passé?

En réalité nous venons de mettre en évidence un mécanisme un peu particulier appelé heap-only-tuple alias HOT.
Lorsqu'une colonne est mise à jour, qu'aucun index ne pointe vers cette colonne et qu'on peut insérer l'enregistrement dans le même bloc.
Le moteur va se contenter de faire un pointeur entre l'ancien enregistrement le nouveau.

Cela permet au moteur d'éviter d'avoir à mettre à jour l'index. Avec tout ce que cela entraîne :

  * Des opérations en lecture/écriture évitées
  * Réduction de la fragmentation de l'index et donc de sa taille (il est difficile de réutiliser les anciens emplacements d'un index)

Si vous observez la lecture du bloc de la table, la colonne t_ctid de la première ligne pointe vers (0,3). Si la ligne était à nouveau mise à jour, la première ligne de la table pointerait vers la ligne (0,3) et la ligne (0,3) pointerait vers (0,4), formant ce qu'on appelle une chaîne. un vacuum nettoierait les espaces libres mais conserverait toujours la première ligne qui pointera vers le dernier enregistrement.


On rajoute une ligne et l'index ne change toujours pas:
```
update t3 set c2 = 4 WHERE c1=1;
postgres=# select * from  bt_page_items(get_raw_page('t3_c1_idx',1));
 itemoffset | ctid  | itemlen | nulls | vars |          data           
------------+-------+---------+-------+------+-------------------------
          1 | (0,1) |      16 | f     | f    | 01 00 00 00 00 00 00 00
          2 | (0,2) |      16 | f     | f    | 02 00 00 00 00 00 00 00
(2 rows)

postgres=# select * from  heap_page_items(get_raw_page('t3',0));
 lp | lp_off | lp_flags | lp_len | t_xmin | t_xmax | t_field3 | t_ctid | t_infomask2 | t_infomask | t_hoff | t_bits | t_oid |       t_data       
----+--------+----------+--------+--------+--------+----------+--------+-------------+------------+--------+--------+-------+--------------------
  1 |   8160 |        1 |     32 |   1221 |   1223 |        0 | (0,3)  |       16386 |       1280 |     24 |        |       | \x0100000001000000
  2 |   8128 |        1 |     32 |   1222 |      0 |        0 | (0,2)  |           2 |       2304 |     24 |        |       | \x0200000002000000
  3 |   8096 |        1 |     32 |   1223 |   1224 |        0 | (0,4)  |       49154 |       8448 |     24 |        |       | \x0100000003000000
  4 |   8064 |        1 |     32 |   1224 |      0 |        0 | (0,4)  |       32770 |      10240 |     24 |        |       | \x0100000004000000
(4 rows)
```

Vacuum nettoie les emplacements disponibles:
```
postgres=# vacuum t3;
VACUUM
postgres=# select * from  heap_page_items(get_raw_page('t3',0));
 lp | lp_off | lp_flags | lp_len | t_xmin | t_xmax | t_field3 | t_ctid | t_infomask2 | t_infomask | t_hoff | t_bits | t_oid |       t_data       
----+--------+----------+--------+--------+--------+----------+--------+-------------+------------+--------+--------+-------+--------------------
  1 |      4 |        2 |      0 |        |        |          |        |             |            |        |        |       |
  2 |   8160 |        1 |     32 |   1222 |      0 |        0 | (0,2)  |           2 |       2304 |     24 |        |       | \x0200000002000000
  3 |      0 |        0 |      0 |        |        |          |        |             |            |        |        |       |
  4 |   8128 |        1 |     32 |   1224 |      0 |        0 | (0,4)  |       32770 |      10496 |     24 |        |       | \x0100000004000000
(4 rows)
```

Un update va réutiliser le second emplacement et l'index reste inchangé. Observez la valeur de la colonne `t_ctid` pour reconstituer la chaîne.
```
postgres=# update t3 set c2 = 5 WHERE c1=1;
UPDATE 1
postgres=# select * from  heap_page_items(get_raw_page('t3',0));
 lp | lp_off | lp_flags | lp_len | t_xmin | t_xmax | t_field3 | t_ctid | t_infomask2 | t_infomask | t_hoff | t_bits | t_oid |       t_data       
----+--------+----------+--------+--------+--------+----------+--------+-------------+------------+--------+--------+-------+--------------------
  1 |      4 |        2 |      0 |        |        |          |        |             |            |        |        |       |
  2 |   8160 |        1 |     32 |   1222 |      0 |        0 | (0,2)  |           2 |       2304 |     24 |        |       | \x0200000002000000
  3 |   8096 |        1 |     32 |   1225 |      0 |        0 | (0,3)  |       32770 |      10240 |     24 |        |       | \x0100000005000000
  4 |   8128 |        1 |     32 |   1224 |   1225 |        0 | (0,3)  |       49154 |       8448 |     24 |        |       | \x0100000004000000
(4 rows)

postgres=# select * from  bt_page_items(get_raw_page('t3_c1_idx',1));
 itemoffset | ctid  | itemlen | nulls | vars |          data           
------------+-------+---------+-------+------+-------------------------
          1 | (0,1) |      16 | f     | f    | 01 00 00 00 00 00 00 00
          2 | (0,2) |      16 | f     | f    | 02 00 00 00 00 00 00 00
(2 rows)
```

Il y a cependant quelques cas où le moteur ne peut pas utilise ce mécanisme :

  * Lorsqu'il n'y a plus de place dans le bloc et qu'il doit écrire un autre bloc. La fragmentation de la table est ici bénéfique.
  * Un index porte sur la colonne mise à jour.

Et dans le cas d'un index fonctionnel ou partiel? Il est trop coûteux de verifier à chaque mise à jour si l'index doit être mis à jour.
Par exemple, un index fonctionnel qui n'indexe que certaines clés d'un objet JSON :

```
create table t4 (c1 jsonb, c2 int,c3 int);
CREATE index ON t4 (c1) WHERE c1->>'prenom' = 'adrien';
CREATE index ON t4 (c3);
