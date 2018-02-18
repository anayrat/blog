+++
title = "Replication Logique Fonctionnement Interne"
date = 2018-02-04T12:19:41+01:00
draft = true

# Tags and categories
# For example, use `tags = []` for no tags, or the form `tags = ["A Tag", "Another Tag"]` for one or more tags.
tags = ["postgres","replication logique"]
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


Intro

<!--more-->

Difficulté de la réplication logique est d'appliquer les changements dans le *bon* ordre.



J'ai remarqué que lors d'une grosse transaction, des fichiers apparaissaient dans le répertoire pg_repslot.

PostgreSQL doit réordonner les modifications pour qu'elles soient appliquées dans le bon ordre.

Les processus wal sender procèdent au décodage logique et réordonnent les changements en mémoire.

Si la transaction comprend beaucoup de changements, cela consommerait beaucoupde mémoire. Ils écrivent donc ces changements sur disque.

Après une exploration dans le code, on arrive dans le fichier `src/backend/replication/logical/reorderbuffer.c` :

```c
 139 /*
 140  * Maximum number of changes kept in memory, per transaction. After that,
 141  * changes are spooled to disk.
 142  *
 143  * The current value should be sufficient to decode the entire transaction
 144  * without hitting disk in OLTP workloads, while starting to spool to disk in
 145  * other workloads reasonably fast.
 146  *
 147  * At some point in the future it probably makes sense to have a more elaborate
 148  * resource management here, but it's not entirely clear what that would look
 149  * like.
 150  */
 151 static const Size max_changes_in_memory = 4096;
```

A partir de 4096 modifications le moteur écrit les changements sur disque :

```C
2048 /*
2049  * Check whether the transaction tx should spill its data to disk.
2050  */
2051 static void
2052 ReorderBufferCheckSerializeTXN(ReorderBuffer *rb, ReorderBufferTXN *txn)
2053 {
2054     /*
2055      * TODO: improve accounting so we cheaply can take subtransactions into
2056      * account here.
2057      */
2058     if (txn->nentries_mem >= max_changes_in_memory)
[...]
2065 /*
2066  * Spill data of a large transaction (and its subtransactions) to disk.
2067  */
2068 static void
2069 ReorderBufferSerializeTXN(ReorderBuffer *rb, ReorderBufferTXN *txn)
2070 {
2071     dlist_iter  subtxn_i;
2072     dlist_mutable_iter change_i;
2073     int         fd = -1;
2074     XLogSegNo   curOpenSegNo = 0;
2075     Size        spilled = 0;
2076     char        path[MAXPGPATH];
2077
2078     elog(DEBUG2, "spill %u changes in XID %u to disk",
2079          (uint32) txn->nentries_mem, txn->xid);
```

En plaçant le paramètre `log_min_messages` sur `debug2` on peut arriver à mettre en évidence ce comportement.

Voici le contenu de mon répertoire `pg_repslot` :
```
ls pg_replslot/* -alh
pg_replslot/mysub:
total 4,0K
drwx------ 2 postgres postgres  19 févr. 18 15:46 .
drwx------ 4 postgres postgres  33 févr. 18 11:51 ..
-rw------- 1 postgres postgres 176 févr. 18 15:46 state

pg_replslot/mysub3:
total 4,0K
drwx------ 2 postgres postgres  19 févr. 18 15:46 .
drwx------ 4 postgres postgres  33 févr. 18 11:51 ..
-rw------- 1 postgres postgres 176 févr. 18 15:46 state
```

J'ai deux répertoires, correspondants à deux slots de réplication. En effet,
j'ai créé une publication et deux instances qui ont souscrit à la même publication.

Premier exemple simple : on ajoute des lignes une table appartenant à une publication.

```SQL
begin;
BEGIN
postgres=# select count(*) from t1;
 count
-------
     0
(1 ligne)

postgres=#  insert into t1 (select  i, md5(i::text) from generate_series(1,1000) i);
INSERT 0 1000
postgres=#  insert into t1 (select  i, md5(i::text) from generate_series(1,1000) i);
INSERT 0 1000
postgres=#  insert into t1 (select  i, md5(i::text) from generate_series(1,1000) i);
INSERT 0 1000
postgres=#  insert into t1 (select  i, md5(i::text) from generate_series(1,1000) i);
INSERT 0 1000
postgres=#  insert into t1 (select  i, md5(i::text) from generate_series(1,95) i);
INSERT 0 95
select count(*) from t1;
 count
-------
  4095
(1 ligne)
```
Jusqu'ici nous n'avons pas atteint le seuil de 4096. Le repertoire ne contient encore que le fichier state :

```
ls pg_replslot/* -alh
pg_replslot/mysub:
total 4,0K
drwx------ 2 postgres postgres  19 févr. 18 15:52 .
drwx------ 4 postgres postgres  33 févr. 18 11:51 ..
-rw------- 1 postgres postgres 176 févr. 18 15:52 state

pg_replslot/mysub3:
total 4,0K
drwx------ 2 postgres postgres  19 févr. 18 15:52 .
drwx------ 4 postgres postgres  33 févr. 18 11:51 ..
-rw------- 1 postgres postgres 176 févr. 18 15:52 state
```

Rajoutons une ligne :

```SQL
insert into t1 (select  i, md5(i::text) from generate_series(1,1) i);
INSERT 0 1
postgres=# select count(*) from t1;
 count
-------
  4096
(1 ligne)
```

Cette fois le seil est atteint (`if (txn->nentries_mem >= max_changes_in_memory)`.
Les processus wal sender vont écrire sur disque les changements à appliquer.

On retrouve dans les logs ces messages :
```
[1977] postgres@postgres DEBUG:  spill 4096 changes in XID 51689068 to disk
[2061] postgres@postgres DEBUG:  spill 4096 changes in XID 51689068 to disk
```

Correspondant aux deux processus wal sender :

```
ps faux |grep -E '(1977|2061)'
postgres  1977  3.3  0.3 390496 37984 ?        Ss   11:38   8:54  \_ postgres: 10/main: wal sender process postgres 192.168.1.26(44188) idle
postgres  2061  3.4  0.3 390496 37932 ?        Ss   11:38   9:10  \_ postgres: 10/main: wal sender process postgres 192.168.1.26(44192) idle
```

Dans le même temps, si on regarde le contenu du répertoire pg_repslots, on remarque que de nouveaux fichiers sont apparus :

```
ls pg_replslot/* -alh
pg_replslot/mysub:
total 660K
drwx------ 2 postgres postgres   60 févr. 18 15:53 .
drwx------ 4 postgres postgres   33 févr. 18 11:51 ..
-rw------- 1 postgres postgres  176 févr. 18 15:53 state
-rw------- 1 postgres postgres 656K févr. 18 15:53 xid-51689068-lsn-92-98000000.snap

pg_replslot/mysub3:
total 660K
drwx------ 2 postgres postgres   60 févr. 18 15:53 .
drwx------ 4 postgres postgres   33 févr. 18 11:51 ..
-rw------- 1 postgres postgres  176 févr. 18 15:53 state
-rw------- 1 postgres postgres 656K févr. 18 15:53 xid-51689068-lsn-92-98000000.snap
```

Chaque wal sender a du sérialiser les changements sur disque.

Deuxième exemple, avec deux transactions modifiant la table t1.

Le xid de la première transaction :
```SQL
select txid_current();
 txid_current
--------------
     51689070
(1 ligne)
```

Celui de la seconde transaction :

```SQL
select txid_current();
 txid_current
--------------
     51689071
(1 ligne)
```

Si on insère 4096 lignes dans la première transaction :

```
ls pg_replslot/* -alh
pg_replslot/mysub:
total 660K
drwx------ 2 postgres postgres   60 févr. 18 16:08 .
drwx------ 4 postgres postgres   33 févr. 18 11:51 ..
-rw------- 1 postgres postgres  176 févr. 18 16:08 state
-rw------- 1 postgres postgres 656K févr. 18 16:07 xid-51689070-lsn-92-98000000.snap

pg_replslot/mysub3:
total 660K
drwx------ 2 postgres postgres   60 févr. 18 16:08 .
drwx------ 4 postgres postgres   33 févr. 18 11:51 ..
-rw------- 1 postgres postgres  176 févr. 18 16:08 state
-rw------- 1 postgres postgres 656K févr. 18 16:07 xid-51689070-lsn-92-98000000.snap
```

Ensuite, si on insère 4096 lignes dans la seconde transaction, nous obtenons de nouveux fichiers *.snap :

```
ls pg_replslot/* -alh
pg_replslot/mysub:
total 1,3M
drwx------ 2 postgres postgres  101 févr. 18 16:08 .
drwx------ 4 postgres postgres   33 févr. 18 11:51 ..
-rw------- 1 postgres postgres  176 févr. 18 16:08 state
-rw------- 1 postgres postgres 656K févr. 18 16:07 xid-51689070-lsn-92-98000000.snap
-rw------- 1 postgres postgres 656K févr. 18 16:08 xid-51689071-lsn-92-98000000.snap

pg_replslot/mysub3:
total 1,3M
drwx------ 2 postgres postgres  101 févr. 18 16:08 .
drwx------ 4 postgres postgres   33 févr. 18 11:51 ..
-rw------- 1 postgres postgres  176 févr. 18 16:08 state
-rw------- 1 postgres postgres 656K févr. 18 16:07 xid-51689070-lsn-92-98000000.snap
-rw------- 1 postgres postgres 656K févr. 18 16:08 xid-51689071-lsn-92-98000000.snap
```

J'ai essayé en insérant 3000 lignes dans chaque transaction, aucun fichier snap
n'est écrit. Ce n'est que lors que chaque transaction dépasse 4096 changements
que ceux-ci sont écrit sur disque.

Commitons la première transaction :

```ls pg_replslot/* -alh
pg_replslot/mysub:
total 660K
drwx------ 2 postgres postgres   60 févr. 18 16:15 .
drwx------ 4 postgres postgres   33 févr. 18 11:51 ..
-rw------- 1 postgres postgres  176 févr. 18 16:15 state
-rw------- 1 postgres postgres 656K févr. 18 16:08 xid-51689071-lsn-92-98000000.snap

pg_replslot/mysub3:
total 660K
drwx------ 2 postgres postgres   60 févr. 18 16:15 .
drwx------ 4 postgres postgres   33 févr. 18 11:51 ..
-rw------- 1 postgres postgres  176 févr. 18 16:15 state
-rw------- 1 postgres postgres 656K févr. 18 16:08 xid-51689071-lsn-92-98000000.snap
```

Le fichier snap correspondant est supprimé. Si on commite la deuxième transaction, le second fichier est également supprimé.

A noter que ces fichiers apparaissent pour toute modification (`DELETE, UPDATE`...).


Que se passe t'il pour une grosse transaction?

Dans cet exemple nous allons insérer une grande quantité d'enregistrement dans
une transaction qui sera commitée plus tard. Ceci afin de mettre en évidence le
fonctionnement de la réplication logique.

Voici les requêtes qui ont a été exécutée

```SQL
BEGIN.
INSERT INTO t1 (select  i, md5(i::text) FROM generate_series(1,10000000) i);
-- attente de plusieurs minutes
COMMIT;
```

Pour information, la requête d'insertion a été lancée peu après 16h27min13s et s'est terminée à 16h28min16s.

Le commit n'a été fait que bien plus tard, à 16h33min46s.

Nous constatons que les deux processus wal sender vont sérialiser les changements sur disque :
```
[1977] postgres@postgres DEBUG:  spill 4096 changes in XID 51689075 to disk
[2061] postgres@postgres DEBUG:  spill 4096 changes in XID 51689075 to disk
[1977] postgres@postgres DEBUG:  spill 4096 changes in XID 51689075 to disk
[2061] postgres@postgres DEBUG:  spill 4096 changes in XID 51689075 to disk
[1977] postgres@postgres DEBUG:  spill 4096 changes in XID 51689075 to disk
[2061] postgres@postgres DEBUG:  spill 4096 changes in XID 51689075 to disk
```

Ce qui produit une grosse quantité de fichiers :
```
[...]
pg_replslot/mysub3:
total 1,6G
drwx------ 2 postgres postgres 8,0K févr. 18 16:28 .
drwx------ 4 postgres postgres   33 févr. 18 11:51 ..
-rw------- 1 postgres postgres  176 févr. 18 16:28 state
-rw------- 1 postgres postgres 1,1M févr. 18 16:27 xid-51689075-lsn-92-98000000.snap
-rw------- 1 postgres postgres  15M févr. 18 16:27 xid-51689075-lsn-92-99000000.snap
-rw------- 1 postgres postgres  16M févr. 18 16:27 xid-51689075-lsn-92-9A000000.snap
-rw------- 1 postgres postgres  16M févr. 18 16:27 xid-51689075-lsn-92-9B000000.snap
-rw------- 1 postgres postgres  16M févr. 18 16:27 xid-51689075-lsn-92-9C000000.snap
[...]
```

Voici quelques graphes issus de Netdata :

CPU :
On constate une forte charge CPU, jusqu'à 16h28min16s où elle baisse un peu. Puis à 16h28min48s la charge redescent à 0.
On retrouve une hausse de la charge à 16h33min46s.

Comment expliquer ces résultats?

Lors du premier pic de charge, il y avait plusieurs processus qui consommait de la ressource :
  * le backend qui insérait les données
  * les deux processus wal sender

La machine dispose de 8 coeurs logique, chaque processus n'exploite qu'un seul coeur. Soit environ 13% par processus.

A 16h28min16s, le backend venait de terminer l'insertion. Cependant les processus
wal sender contininaient d'être actifs. Le décodage des journaux et la sérialisation des transactions est assez couteux.

Enfin, à 16h33min46s, la transaction est commitée. Ca n'a eu aucune incidence sur
la charge du backend. en revanche, les processus wal sender se sont mis à lire
les fichiers snap produits sur disque pour transmettre le des modifications aux
souscripteurs.

A noter que les changements n'ont été visible côté souscripteur que vers 16h35min40s.
Une fois que les changements ont été rejoués. Le rejeu est plus long que l'insertion.
Cela dépend beaucoup des performances des processeurs.


db_stat :
On constate un pic d'insertion à 16h33min46s. Cela correspond bien au moment où
la transaction a été commitée.

Network :

On constate un important trafic réseau entre 16h33min46s et 16h35min40s. On peut
en déduire que les changements ne sont transmis qu'après le commit.

Les changements ne sont pas transmis "au fil de l'eau". Cela présente l'avantage
d'éviter d'envoyer des changements qui ne seraient pas commité. En revanche,
celà rajoute du délai dans l'application des changements.

FIXME: parler du patch de vondra

Replication :

Ces graphes sont plus compliqués. On peut noter que les graphes "Streaming
replication delta" mesurant le delta de réplication à partir de la vue `pg_stat_replication`
sont difficile à expliquer.
Normalement il présente le retard de réplication de serveurs secondaires. Or
nous venons de voir que l'application des changements ne s'est fait que lors du
commit, soit à 16h33min46s. Or les graphes semblent indiquer que les serveurs
secondaire avaient du retard jsuqu'à 16h28min48s.

Les courbes "write delta", "flush delta" et "replay delta" se confondent.
La courbe "sent delta" est assez différente.

Ma compréhension de ces graphes est que "write delta", "flush delta" et "replay delta"
correspondent au retard de décodage logique par rapport aux données envoyées "sent delta".

A 16h28min16s, la requête s'est terminée, le "sent delta" arrête de croitre. et
décroit aux fur et à mesure de l'avancée du décodage logique.

On peut confirmer cela grâce aux graphes "replication slot files", la courbe
"pg_repslot files" correspond au nombre de fichiers présent dans le répertoire
"pg_repslot" de chaque slot de réplication. Celle ci est croissance et s'arrête
de croitre au même moment ou les courbes "write delta", "flush delta" et
"replay delta" redescendent à 0.

On constate également que le moteur est contraint de conserver les journaux de
transaction tant que la transaction n'a pas été commitée.

Enfin, à 16h35min40s les changements ont bien été transmis côté souscripteur, le
moteur peut nettoyer les journaux de transaction et les fichiers snaps.


Et avec un trafic OLTP? Pour ce test, j'ai utilisé pgbench lancé pendant 10 minutes
sur une base dont toutes les tables sont répliquées.

On remarque que le trafic réseau chute au même moment que nous observons un
retard de réplication qui augmente. Celà correspondait à des période où le
secondaire saturait côté CPU. Le rejeu ne se fait que grâce à un seul processus
bgworker qui avait du mal à supporter la charge. D'ailleurs on peut voir que le
secondaire rattrape le retard quelques dizaines secondes après la fin du bench.

Autre point important, on constate qu'il n'y a aucun fichier snap écrit sur disque.
Cela confirme le commentaire dans le code, indiquant qu'en cas de trafic OLTP,
les changements sont conservé en mémoire et non sur disque.



Et avec un trafic OLTP sur une base non répliquée (logiquement)? Pour ce test, j'ai utilisé pgbench lancé pendant 10 minutes.

Network :

Globalement, on constate que le flux réseau est assez constant tout au long du test.

Replication :

On remarque que les courbes ne sont pas nulles. Pourtant aucune publication ne
correspond à la base de bench. Il s'agit du retard de décodage. On constate qu'il reste faible.

Pour les courbes sur les slots de réplication. On constate que le moteur conserve
les journaux en fonction du décodage logique. Vu que les publications ne sont
pas sur la base bench, il n'y a donc aucun fichier snap produit.
