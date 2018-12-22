+++
title = "PostgreSQL et updates heap-only-tuples - partie 2"
date = 2018-11-19T08:00:00+01:00
draft = false
summary = "Quand le moteur ne fait pas d'update *heap-only-tuple* et présentation de la nouveauté de la version 11"

# Tags and categories
# For example, use `tags = []` for no tags, or the form `tags = ["A Tag", "Another Tag"]` for one or more tags.
tags = ["postgres","index","heap-only-tuple"]
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

Voici une série d'articles qui va porter sur une nouveauté de la version 11.

Durant le développement de cette version, une fonctionnalité a attiré mon attention.
On peut la retrouver dans les releases notes : <https://www.postgresql.org/docs/11/static/release-11.html>


> Allow heap-only-tuple (HOT) updates for expression indexes when the values of the expressions are unchanged (Konstantin Knizhnik)

J'avoue que ce n'est pas très explicite et cette fonctionnalité nécessite quelques
connaissances sur le fonctionnement du moteur que je vais essayer d'expliquer à travers
plusieurs articles :

1. [Fonctionnement du MVCC et update *heap-only-tuples*][1]
2. [Quand le moteur ne fait pas d'update *heap-only-tuple* et présentation de la nouveauté de la version 11][2]
3. [Impact sur les performances][3]

**Cette fonctionnalité a été désactivée en 11.1 car elle pouvait conduire à des
crash d'instance[^4]. J'ai tout de même choisi de publier ces articles car ils permettent
de comprendre le mécanisme des updates HOT et le gain que pourrait apporter cette
fonctionnalité.**

Je remercie au passage Guillaume Lelarge pour la relecture de cet article ;).


L'article précédent présentait le fonctionnement des UDATES heap-only-tuple. Dans
cet article nous allons voir le cas où Postgres ne fait pas d'UPDATE heap-ony-tuple.
Ce qui nous permettra d'aborder la fonctionnalité qui aurait du arriver dans la version 11.


# Cas avec un index sur une colonne mise à jour

Reprenons l'exemple précédent et rajoutons une colonne indexée :

```sql
ALTER TABLE t3 ADD COLUMN c3 int;
CREATE INDEX ON t3(c3);
```

Les UPDATE précédents portaient sur une colonne non-indexée. Que se passe-t-il
si l'UPDATE porte sur c3 ?


État de la table et des index avant UPDATE :

```sql
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

Pas de changement sur la table car la colonne c3 ne contient que des nulls, on
peut le constater en observant l'index `t3_c3_idx` où `nulls` est à *true* sur
chaque ligne.

```sql
UPDATE t3 SET c3 = 7 WHERE c1=1;
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
mis à jour. Entraînant l'ajout d'une troisième entrée, même si la valeur de la
colonne c1 n'a pas changée.

Après un VACUUM :

```sql
VACUUM t3;
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


# Nouveauté de la version : 11 heap-only-tuple (HOT) avec index fonctionnels


Lorsqu'un index fonctionnel porte sur la colonne modifiée, il peut arriver que
le résultat de l'expression reste inchangé malgré la mise à jour de la colonne.
La clé dans l'index serait donc inchangée.

Prenons un exemple : un index fonctionnel sur une *clé spécifique* d'un objet JSON.

```SQL
CREATE TABLE t4 (c1 jsonb, c2 int,c3 int);
CREATE INDEX ON t4 ((c1->>'prenom')) ;
CREATE INDEX ON t4 (c2);
INSERT INTO t4 VALUES ('{ "prenom":"adrien" , "ville" : "valence"}'::jsonb,1,1);
INSERT INTO t4 VALUES ('{ "prenom":"guillaume" , "ville" : "lille"}'::jsonb,2,2);

-- changement qui ne porte pas sur prenom, on change que la ville
UPDATE t4 SET c1 = '{"ville": "valence (#soleil)", "prenom": "guillaume"}' WHERE c2=2;
SELECT pg_stat_get_xact_tuples_hot_updated('t4'::regclass);
 pg_stat_get_xact_tuples_hot_updated
-------------------------------------
                                   0
(1 row)

UPDATE t4 SET c1 = '{"ville": "nantes", "prenom": "guillaume"}' WHERE c2=2;
SELECT pg_stat_get_xact_tuples_hot_updated('t4'::regclass);
 pg_stat_get_xact_tuples_hot_updated
-------------------------------------
                                   0
(1 row)
```

La fonction `pg_stat_get_xact_tuples_hot_updated` indique le nombre de lignes mises
à jour par le mécanisme HOT.

Les deux UPDATE n'ont fait que modifier la clé "ville" et pas la clé "prenom".
Ce qui n’entraîne pas de modification de l'index car il n'indexe que la clé "prenom".

Le moteur n'a pas pu faire d'HOT. En effet, pour lui, l'UPDATE a porté sur la colonne et
l'index doit être mis à jour.

Avec la version 11, le moteur est capable de constater que le résultat de
l'expression ne change pas. Effectuons le même test sur la version 11 :

```SQL
CREATE TABLE t4 (c1 jsonb, c2 int,c3 int);
-- CREATE INDEX ON t4 ((c1->>'prenom'))  WITH (recheck_on_update='false');
CREATE INDEX ON t4 ((c1->>'prenom')) ;
CREATE INDEX ON t4 (c2);
INSERT INTO t4 VALUES ('{ "prenom":"adrien" , "ville" : "valence"}'::jsonb,1,1);
INSERT INTO t4 VALUES ('{ "prenom":"guillaume" , "ville" : "lille"}'::jsonb,2,2);

-- changement qui ne porte pas sur prenom
UPDATE t4 SET c1 = '{"ville": "valence (#soleil)", "prenom": "guillaume"}' WHERE c2=2;
SELECT pg_stat_get_xact_tuples_hot_updated('t4'::regclass);
 pg_stat_get_xact_tuples_hot_updated
-------------------------------------
                                   1
(1 row)

UPDATE t4 SET c1 = '{"ville": "nantes", "prenom": "guillaume"}' WHERE c2=2;
SELECT pg_stat_get_xact_tuples_hot_updated('t4'::regclass);
 pg_stat_get_xact_tuples_hot_updated
-------------------------------------
                                   2
(1 row)
```

Cette fois, le moteur a bien utilisé le mécanisme HOT. On peut le vérifier en
regardant le contenu physique de l'index avec pageinspect :

Version 10 :
```sql
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

```sql
SELECT * FROM  bt_page_items(get_raw_page('t4_expr_idx',1));
itemoffset | ctid  | itemlen | nulls | vars |                      data
------------+-------+---------+-------+------+-------------------------------------------------
         1 | (0,1) |      16 | f     | t    | 0f 61 64 72 69 65 6e 00
         2 | (0,2) |      24 | f     | t    | 15 67 75 69 6c 6c 61 75 6d 65 00 00 00 00 00 00
(2 rows)
```

Ce comportement peut se contrôler grâce à une nouvelle option lors de la création
de l'index : `recheck_on_update`.

Par défaut à `on`, le moteur effectue la vérification du résultat de l'expression
pour faire un UPDATE HOT. On peut le paramétrer à `off` s'il y a de fortes chances
pour que le résultat de l'expression change lors d'un UPDATE. Cela permet d'éviter
d'exécuter l'expression inutilement.

A noter également que le moteur évite l'évaluation de l'expression si son coût est
supérieur à 1000.

Dans le troisième et dernier article, nous verrons un cas un peu plus concret
pour voir l'impact en terme de performance et de volumétrie.


[1]: https://blog.anayrat.info/2018/11/12/postgresql-et-updates-heap-only-tuples-partie-1/
[2]: https://blog.anayrat.info/2018/11/19/postgresql-et-updates-heap-only-tuples-partie-2/
[3]: https://blog.anayrat.info/2018/11/26/postgresql-et-updates-heap-only-tuples-partie-3/

[^4]: [Disable recheck_on_update optimization to avoid crashes](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=05f84605dbeb9cf8279a157234b24bbb706c5256)
