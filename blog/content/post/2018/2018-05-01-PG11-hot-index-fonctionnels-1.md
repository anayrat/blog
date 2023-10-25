+++
title = "PostgreSQL et updates heap-only-tuples - partie 1"
date = 2018-11-12T08:00:00+01:00
draft = false
summary = "Fonctionnement du MVCC et update *heap-only-tuples*"
authors = ['adrien']

# Tags and categories
# For example, use `tags = []` for no tags, or the form `tags = ["A Tag", "Another Tag"]` for one or more tags.
tags = ["postgres","index","heap-only-tuple"]
categories = ["Postgres"]
show_related = true

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


# Fonctionnement MVCC

Dans l'implémentation [MVCC](https://fr.wikipedia.org/wiki/Multiversion_Concurrency_Control)
de Postgres, le moteur ne met pas à jour une ligne directement : il duplique
l'enregistrement et renseigne des informations de visibilité.

Pourquoi ce fonctionnement?

Il y a une composante capitale à prendre en compte quand on travaille avec un
SGBG : la concurrence d'accès.

La ligne que vous êtes en train de modifier est peut être utilisée par une transaction
antérieure. Une sauvegarde en cours par exemple :)

Pour cela, les SGBD ont adopté différentes techniques :

  * Modifier l'enregistrement et stocker les versions antérieures sur un autre
  emplacement. C'est ce que fait oracle par exemple avec les undo logs.
  * Dupliquer l'enregistrement et stocker des informations de visibilité pour
  savoir quelle ligne est visible par telle ou telle transaction. Cela nécessite
  d'avoir un mécanisme de nettoyage des lignes qui ne sont plus visibles par
  personne. C'est l'implémentation dans Postgres et le vacuum a pour rôle d'effectuer ce nettoyage.

Prenons une table toute simple et regardons son contenu évoluer à l'aide de l'extension pageinspect :

```sql
CREATE TABLE t2(c1 int);
INSERT INTO t2 VALUES (1);
SELECT lp,t_data FROM  heap_page_items(get_raw_page('t2',0));
 lp |   t_data
----+------------
  1 | \x01000000
(1 row)

UPDATE t2 SET c1 = 2 WHERE c1 = 1;
SELECT lp,t_data FROM  heap_page_items(get_raw_page('t2',0));
 lp |   t_data
----+------------
  1 | \x01000000
  2 | \x02000000
(2 rows)

VACUUM t2;
SELECT lp,t_data FROM  heap_page_items(get_raw_page('t2',0));
 lp |   t_data
----+------------
  1 |
  2 | \x02000000
(2 rows)
```

On peut voir que le moteur a dupliqué la ligne et le vacuum a nettoyé
l'emplacement pour un usage futur.

# Mécanisme d'update heap-only-tuple

Prenons un autre cas, un peu plus compliqué, une table avec deux colonnes et un
index sur une des deux colonnes :

```sql
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

De la même façon qu'il est possible de lire les blocs d'une table, il est possible
de lire les blocs d'un index avec pageinspect :

```sql
SELECT * FROM  bt_page_items(get_raw_page('t3_c1_idx',1));
 itemoffset | ctid  | itemlen | nulls | vars |          data
------------+-------+---------+-------+------+-------------------------
          1 | (0,1) |      16 | f     | f    | 01 00 00 00 00 00 00 00
          2 | (0,2) |      16 | f     | f    | 02 00 00 00 00 00 00 00
(2 rows)
```

Jusque-là, c'est assez simple, ma table contient deux enregistrements et l'index
contient également deux enregistrements pointant vers les blocs correspondants de
la table (colonne ctid).

Si je mets à jour la colonne c1, avec la nouvelle valeur 3 par exemple, l'index
devra être mis à jour.

Maintenant, si je mets à jour la colonne c2, est-ce que l'index portant sur c1
sera mis à jour ?

Au premier abord, on pourrait se dire non car c1 n'est pas modifiée.

Mais à cause du modèle MVCC présenté plus haut, en théorie, la réponse sera oui :
nous venons de voir que le moteur va dupliquer la ligne, son emplacement physique
sera donc différent (le ctid suivant sera (0,3)).

Vérifions le :

```sql
UPDATE t3 SET c2 = 3 WHERE c1=1;
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

En lisant le bloc de l'index, on constate que son contenu n'a pas bougé ! Si je
cherche la ligne `WHERE c1 = 1`, l'index m'oriente vers l'enregistrement (0,1) qui
correspond à l'ancienne ligne !

Que s'est-il passé ?

En réalité, nous venons de mettre en évidence un mécanisme un peu particulier
appelé *heap-only-tuple* alias HOT. Lorsqu'une colonne est mise à jour, qu'aucun
index ne pointe vers cette colonne et qu'on peut insérer l'enregistrement dans
le même bloc, le moteur va se contenter de faire un pointeur entre l'ancien
enregistrement le nouveau.

Cela permet au moteur d'éviter d'avoir à mettre à jour l'index. Avec tout ce que
cela entraîne :

  * Des opérations en lecture/écriture évitées
  * Réduction de la fragmentation de l'index et donc de sa taille (il est
    difficile de réutiliser les anciens emplacements d'un index)

Si vous observez le bloc de la table, la colonne `t_ctid` de la
première ligne pointe vers (0,3). Si la ligne était à nouveau mise à jour, la
première ligne de la table pointerait vers la ligne (0,3) et la ligne (0,3)
pointerait vers (0,4), formant ce qu'on appelle une chaîne. Un vacuum nettoierait
les espaces libres mais conserverait toujours la première ligne qui pointerait vers
le dernier enregistrement.


On modifie une ligne et l'index ne change toujours pas :

```sql
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

Vacuum nettoie les emplacements disponibles :

```sql
VACUUM t3;
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
Observez la valeur de la colonne `t_ctid` pour reconstituer la chaîne :

```sql
UPDATE t3 SET c2 = 5 WHERE c1=1;
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

Euh, la première ligne est vide et le moteur a réutilisé le troisième emplacement ?

En fait, une information n’apparaît pas dans pageinspect. Allons lire directement le bloc avec
[pg_filedump](https://wiki.postgresql.org/wiki/Pg_filedump) :

Note : Il faut demander un `CHECKPOINT` au préalable. Dans le cas contraire, le bloc pourrait ne
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

La première ligne contient `Flags: REDIRECT`. Ceci indique que cette ligne
correspond à une redirection HOT. C'est documenté dans `src/include/storage/itemid.h` :

```c
/*
 * lp_flags has these possible states.  An UNUSED line pointer is available
 * for immediate re-use, the other states are not.
 */
#define LP_UNUSED       0       /* unused (should always have lp_len=0) */
#define LP_NORMAL       1       /* used (should always have lp_len>0) */
#define LP_REDIRECT     2       /* HOT redirect (should have lp_len=0) */
#define LP_DEAD         3       /* dead, may or may not have storage */
```

Il est AUSSI possible de le voir avec pageinspect en affichant la colonne `lp_flags`:

```sql
SELECT lp,lp_flags,t_data,t_ctid FROM  heap_page_items(get_raw_page('t3',0));
 lp | lp_flags |       t_data       | t_ctid
----+----------+--------------------+--------
  1 |        2 |                    |
  2 |        1 | \x0200000002000000 | (0,2)
  3 |        1 | \x0100000005000000 | (0,3)
  4 |        1 | \x0100000004000000 | (0,3)
(4 rows)
```

Si on refait un UPDATE, puis VACUUM, suivi d'un CHECKPOINT pour écrire le bloc sur disque :

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

Le moteur a conservé la première ligne (flag `REDIRECT`) et écrit une nouvelle
ligne à l'emplacement 5.

Il y a cependant quelques cas où le moteur ne peut pas utiliser ce mécanisme :

  * Lorsqu'il n'y a plus de place dans le bloc et qu'il doit écrire un autre bloc.
  On peut en déduire que la fragmentation de la table est ici bénéfique pour bénéficier
  du HOT.
  * Un index porte sur la colonne mise à jour. Dans ce cas le moteur doit mettre
  à jour l'index. Le moteur peut détecter s'il y a eu un changement
  en effectuant une comparaison binaire entre la nouvelle valeur et la précédente [^5].

Dans le prochain article nous verrons justement un exemple où le moteur
ne peut pas employer le mécanisme HOT. Puis, la nouveauté de la version 11
où le moteur peut utiliser se mécanisme.

[1]: https://blog.anayrat.info/2018/11/12/postgresql-et-updates-heap-only-tuples-partie-1/
[2]: https://blog.anayrat.info/2018/11/19/postgresql-et-updates-heap-only-tuples-partie-2/
[3]: https://blog.anayrat.info/2018/11/26/postgresql-et-updates-heap-only-tuples-partie-3/

[^4]: [Disable recheck_on_update optimization to avoid crashes](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=05f84605dbeb9cf8279a157234b24bbb706c5256)
[^5]: [README.HOT](https://git.postgresql.org/gitweb/?p=postgresql.git;a=blob;f=src/backend/access/heap/README.HOT;h=4cf3c3a0d4c2db96a57e73e46fdd7463db439f79;hb=HEAD#l128)
