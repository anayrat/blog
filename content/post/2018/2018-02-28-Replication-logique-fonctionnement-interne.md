+++
title = "Replication Logique Fonctionnement Interne"
date = 2018-03-10T12:19:41+01:00
draft = false
summary = "Cet article détaille le fonctionne de la réplication logique, notamment les différences de comportement en fonction du type de trafic"

# Tags and categories
# For example, use `tags = []` for no tags, or the form `tags = ["A Tag", "Another Tag"]` for one or more tags.
tags = ["postgres","replication logique", "netdata"]
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

# Introduction


J'ai présenté la réplication à travers plusieurs articles :

1. [PostgreSQL 10 et la réplication logique - Fonctionnement][1]
2. [PostgreSQL 10 et la réplication logique - Mise en oeuvre][2]
3. [PostgreSQL 10 et la réplication logique - Restrictions][3]

Cet article va creuser un peu plus le sujet. Nous verrons plus en détail son
fonctionnement. Pour mieux l'appréhender, j'ai rajouté des graphes dans Netdata
que j'ai présenté dans un précédent article. Je vous invite à cliquer sur les
images pour les agrandir, elles seront plus lisibles :smile:

Je remercie au passage [Guillaume Lelarge](https://twitter.com/g_lelarge) qui a
joué le rôle de relecteur pour cet article.

# Sérialisation des changements sur disque

J'ai remarqué que lors d'une grosse transaction, des fichiers apparaissaient
dans le répertoire pg_replslot de chaque slot de réplication. C'est ce qui m'a poussé
à chercher à comprendre à quoi servaient ces fichiers.

PostgreSQL doit réordonner les modifications pour qu'elles soient appliquées
dans le *bon* ordre. Pour ce faire, les processus wal sender procèdent au décodage
logique et réordonnent les changements en mémoire.

Cependant, si la transaction comprend beaucoup de changements, cela consommerait
beaucoup de mémoire. Ils écrivent donc ces changements sur disque.

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

À partir de 4096 modifications, le moteur écrit les changements sur disque :

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

En plaçant le paramètre `log_min_messages` sur `debug2`, on peut arriver à mettre
en évidence ce comportement. Le but est d'afficher le message correspondant à la
ligne 2078.

Sans activité particulière, voici le contenu du répertoire `pg_replslot` :
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
j'ai créé une publication et deux instances ont souscrit à la même publication.
Elles ont chacune leur slot de réplication : mysub et mysub3.

## Cas avec une seule transaction

Ajoutons des lignes à une table appartenant à une publication.

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
Jusqu'ici, nous n'avons pas atteint le seuil de 4096. Le répertoire ne contient
encore que le fichier state :

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

Cette fois, le seuil est atteint (`if (txn->nentries_mem >= max_changes_in_memory`).
Les processus wal sender vont écrire sur disque les changements à appliquer.

On retrouve dans les logs ces messages (avec `log_min_messages = debug2`) :
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

Dans le même temps, si on regarde le contenu du répertoire pg_replslot, on
remarque que de nouveaux fichiers sont apparus :

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

Chaque wal sender a dû sérialiser les changements sur disque.

On remarque également que la notion de changement correspond à une ligne et non
un ordre. Par exemple, un seul insert de 4096 lignes entrainera l'écriture d'un
fichier \*.snap.

## Cas avec deux transactions

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

Ensuite, si on insère 4096 lignes dans la seconde transaction, nous obtenons deux
nouveaux fichiers \*.snap :

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
n'est écrit. Ce n'est que lorsque chaque transaction dépasse 4096 changements
que ceux-ci sont écrits sur disque.

Commitons la première transaction :

```
ls pg_replslot/* -alh
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

Le fichier snap correspondant est supprimé par chaque wal sender. Si on commite
la deuxième transaction, le second fichier est également supprimé.

Il est à noter que ces fichiers apparaissent pour toute modification (`DELETE, UPDATE`...).


Que se passe t'il pour une grosse transaction?

Dans cet exemple, nous allons insérer une grande quantité d'enregistrements dans
une transaction qui sera commitée plus tard, afin de mettre en évidence le
fonctionnement de la réplication logique.

Voici les requêtes qui ont été exécutées :

```SQL
BEGIN.
INSERT INTO t1 (select  i, md5(i::text) FROM generate_series(1,10000000) i); --16h27min13s
-- attente de plusieurs minutes
COMMIT; --16h33min46s
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

## CPU

{{< figure src="/img/2018/logical-rep-big-tx/cpu.png" link="/img/2018/logical-rep-big-tx/cpu.png" title="CPU" >}}


On constate une forte charge CPU, jusqu'à 16h28min16s où elle baisse un peu.
Puis à 16h28min48s la charge redescend à 0. On retrouve une hausse de la charge
à 16h33min46s.

Comment expliquer ces résultats?

Lors du premier pic de charge, il y avait plusieurs processus qui consommait de la ressource :

  * le backend qui insérait les données
  * les deux processus wal sender

La machine dispose de 8 coeurs logiques, chaque processus n'exploite qu'un seul
coeur. Soit environ 13% par processus.

A 16h28min16s, le backend venait de terminer l'insertion. Cependant, les processus
wal sender continuaient d'être actifs. Le décodage des journaux et la sérialisation
des transactions sont assez coûteux.

Enfin, à 16h33min46s, la transaction est commitée. Ca n'a eu aucune incidence sur
la charge du backend. En revanche, les processus wal sender se sont mis à lire
les fichiers snap produits sur disque pour transmettre les modifications aux
souscripteurs.

À noter que les changements n'ont été visibles côté souscripteur que vers 16h35min40s,
une fois que les changements ont été rejoués. Le rejeu est plus long que l'insertion.
Cela dépend beaucoup des performances des processeurs.

## Statistiques sur les enregistrements


{{< figure src="/img/2018/logical-rep-big-tx/db_stat.png" link="/img/2018/logical-rep-big-tx/db_stat.png" target="\_blank" title="tuple inserted" >}}



On constate un pic d'insertion à 16h33min46s. Cela correspond bien au moment où
la transaction a été commitée.

## Réseau

{{< figure src="/img/2018/logical-rep-big-tx/network.png" link="/img/2018/logical-rep-big-tx/network.png" title="Réseau" >}}

On constate un important trafic réseau entre 16h33min46s et 16h35min40s. On peut
en déduire que les changements ne sont transmis qu'après le commit.

Les changements ne sont pas transmis "au fil de l'eau". Cela présente l'avantage
d'éviter d'envoyer des changements qui ne seraient pas commités. En revanche,
cela rajoute du délai dans l'application des changements.

Thomas Vondra a soumis un patch afin d'améliorer ce cas d'usage :
[logical streaming for large in-progress transactions](https://commitfest.postgresql.org/16/1429/)

Pour le moment le patch n'a pas encore été accepté. Il nécessite de rajouter des
informations dans les journaux de transaction. La communauté est très prudente en
ce qui concerne tout ce qui peut avoir un impact sur les performances.

## Réplication

{{< figure src="/img/2018/logical-rep-big-tx/replication.png" link="/img/2018/logical-rep-big-tx/replication.png" title="Réplication delta" >}}


On peut noter que les graphes "Streaming replication delta" mesurant le delta de
réplication à partir de la vue `pg_stat_replication` sont difficiles à expliquer.
Normalement, ils présentent le retard de réplication de serveurs secondaires. Or
nous venons de voir que l'application des changements ne se fait que lors du
commit, soit à 16h33min46s. Or les graphes semblent indiquer que les serveurs
secondaires avaient du retard jusqu'à 16h28min48s.

Les courbes "write delta", "flush delta" et "replay delta" se confondent.
La courbe "sent delta" est assez différente.

Ma compréhension de ces graphes est que "write delta", "flush delta" et "replay delta"
correspondent au retard de décodage logique par rapport aux données envoyées "sent delta".

A 16h28min16s, la requête s'est terminée, le "sent delta" arrête de croître. Puis,
décroit au fur et à mesure de l'avancée du décodage logique.

On peut confirmer cela grâce aux graphes "replication slot files", la courbe
"pg_replslot files" correspond au nombre de fichiers présents dans le répertoire
"pg_replslot" de chaque slot de réplication. Celle-ci est croissante et s'arrête
de croitre au même moment où les courbes "write delta", "flush delta" et
"replay delta" redescendent à 0.

On constate également que le moteur est contraint de conserver les journaux de
transaction tant que la transaction n'a pas été commitée. C'est assez logique,
si un crash survenait, il faudrait pouvoir aller lire les journaux de transaction
pour procéder à nouveau au décodage logique.

Enfin, à 16h35min40s les changements ont bien été transmis côté souscripteur, le
moteur peut enfin nettoyer les journaux de transaction et les fichiers snaps.

# Trafic OLTP

Et avec un trafic OLTP? Pour ce test, j'ai utilisé pgbench lancé pendant 10 minutes
sur une base dont toutes les tables sont répliquées.

{{< figure src="/img/2018/logical-rep-oltp/cpu.png" link="/img/2018/logical-rep-oltp/cpu.png" title="Charge CPU" >}}

{{< figure src="/img/2018/logical-rep-oltp/network.png" link="/img/2018/logical-rep-oltp/network.png" title="Activité réseau" >}}

{{< figure src="/img/2018/logical-rep-oltp/db_stat.png" link="/img/2018/logical-rep-oltp/db_stat.png" title="Activité transactions" >}}

{{< figure src="/img/2018/logical-rep-oltp/replication.png" link="/img/2018/logical-rep-oltp/replication.png" title="Réplication delta" >}}

On remarque que le trafic réseau chute au même moment que nous observons un
retard de réplication qui augmente. Cela correspondait à des périodes où le
secondaire saturait côté CPU (malheureusement je n'ai pas de graphe Netdata sur
le secondaire).

Cela peut s'expliquer par le fait que le rejeu ne se fait que grâce à un seul processus
bgworker qui avait du mal à supporter la charge (machine moins puissante).
D'ailleurs, on peut voir que le secondaire rattrape le retard quelques dizaines
de secondes après la fin du bench.


Autre point important, on constate qu'il n'y a aucun fichier snap écrit sur disque.
Cela confirme le commentaire dans le code, indiquant qu'en cas de trafic OLTP,
les changements sont conservés en mémoire et non sur disque.

# Bilan

On peut constater que la réplication logique s'applique assez bien dans le cas
d'un trafic OLTP pour peu que le secondaire arrive à suivre l'application des
changements. Cela implique que le trafic en écriture ne peut être trop important
au risque d'avoir un secondaire qui a du mal à encaisser la charge, l'application
des changements s'effectuant par un seul processus.

En revanche, dans les cas où le trafic est différent (OLAP), où des requêtes entraînent des
changements importants, le moteur devra sérialiser les changements sur disque. Les
opérations de décodage logique, puis de transfert réseau induisent un délai qui
peut être non négligeable.


[1]: http://blog.anayrat.info/2017/07/29/postgresql-10-et-la-replication-logique-fonctionnement/
[2]: http://blog.anayrat.info/2017/08/05/postgresql-10-et-la-replication-logique-mise-en-oeuvre/
[3]: https://blog.anayrat.info/2017/08/27/postgresql-10-et-la-replication-logique-restrictions/
