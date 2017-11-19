---
title: Restauration PITR – Part 7
author: Adrien Nayrat
type: post
date: 2015-03-22T12:25:41+00:00
aliases: /2015/03/22/restauration-pitr-part-7/
categories:
  - Postgres
tags:
  - postgres

---
Je profite d'un week end de mauvais temps (m’empêchant de retourner en falaise 🙁 ) pour poursuivre mes articles sur Postgres.

Dans le précédent article : [Backup, restauration - Part 6][1] J'ai évoqué la restauration PITR pour Point In Time Recovery. Je vous encourage vivement à mettre en place ce qui va suivre afin de réduire la perte de données en cas de modification accidentelle sur des enregistrements : Un drop table par exemple.

<!--more-->

Pour la mettre en place il vous faut :

  * Une instance Postgres maître.
  * Archivage des WAL, externalisé de préférence.

Je ne vais pas revenir sur la mise en place de l'archivage, je l'ai déjà détaillé dans les précédents articles. Pour vous expliquer cette restauration je vais utiliser un scénario « catastrophe ».

Il est 12H30 et on vous informe qu'une table a été supprimée par erreur sur un serveur de production.

La réplication que vous avez mis en place ne vous est d'aucune aide car l'opération a été reportée sur le réplicat.

_Remarque : Avec la version 9.4 il est possible d'avoir un réplicat avec du retard. Avec un peu de chance l'opération de suppression de la table n'est peut être pas encore réalisée sur le réplicat._

Par chance, vous effectuez un backup full toutes les nuits à 1H00 et vous avez mis en place l'archivage des journaux. Leur nettoyage se fait après le backup full grâce à >pg_archivecleanup.

Pour être tranquille vous effectuez un checkpoint afin que les derniers journaux soient archivés. Ensuite vous stoppez votre instance Postgres et déplacez le répertoire (au cas où).

Vous restaurez les fichiers du backup full. Il ne vous reste plus qu'a configurer correctement la restauration via le fichier recovery.conf.

Celui-ci devra contenir le paramètre restore_command afin que Postgres récupére les journaux de transactions archivés.

Il devra également contenir un paramètre pour la cible de récupération, Si on ne le spécifie pas le moteur va rejouer les journaux de transaction dans leur totalité, jusqu'au « drop table » fatal. Il est possible de demander la restauration jusqu'à un point de restauration, une date ou un identifiant de transaction.

Pour approfondir : <http://docs.postgresqlfr.org/current/recovery-target-settings.html>

Revenons à notre scénario, imaginons que nous souhaitons revenir à la dernière transaction jouée avant le drop fatal. En grattant un peu, on peut utiliser l'outil  [pg_xlogdump](http://www.postgresql.org/docs/current/static/pgxlogdump.html) pour lire les journaux de transaction.

```
/usr/lib/postgresql/9.3/bin/pg_xlogdump 000000010000000300000066
rmgr: Heap        len (rec/tot):     21/   197, tx:    1213547, lsn: 3/66000028, prev 3/650000F0, bkp: 1000, desc: insert: rel 1663/16486/16519; tid 0/2
rmgr: Transaction len (rec/tot):     12/    44, tx:    1213547, lsn: 3/660000F0, prev 3/66000028, bkp: 0000, desc: commit: 2015-03-22 12:23:35.834785 CET
rmgr: Standby     len (rec/tot):     16/    48, tx:    1213548, lsn: 3/66000120, prev 3/660000F0, bkp: 0000, desc: AccessExclusive locks: xid 1213548 db 16486 rel 16519
rmgr: Standby     len (rec/tot):     16/    48, tx:    1213548, lsn: 3/66000150, prev 3/66000120, bkp: 0000, desc: AccessExclusive locks: xid 1213548 db 16486 rel 16522
rmgr: Standby     len (rec/tot):     16/    48, tx:    1213548, lsn: 3/66000180, prev 3/66000150, bkp: 0000, desc: AccessExclusive locks: xid 1213548 db 16486 rel 16524
rmgr: Heap        len (rec/tot):     26/  6642, tx:    1213548, lsn: 3/660001B0, prev 3/66000180, bkp: 1000, desc: delete: rel 1663/16486/11774; tid 7/35 KEYS_UPDATED
rmgr: Heap        len (rec/tot):     26/  3270, tx:    1213548, lsn: 3/66001BA8, prev 3/660001B0, bkp: 1000, desc: delete: rel 1663/16486/11903; tid 46/44 KEYS_UPDATED
rmgr: Heap        len (rec/tot):     26/  4774, tx:    1213548, lsn: 3/66002888, prev 3/66001BA8, bkp: 1000, desc: delete: rel 1663/16486/11821; tid 2/26 KEYS_UPDATED
rmgr: Heap        len (rec/tot):     26/  5730, tx:    1213548, lsn: 3/66003B30, prev 3/66002888, bkp: 1000, desc: delete: rel 1663/16486/11786; tid 42/37 KEYS_UPDATED
rmgr: Heap        len (rec/tot):     26/    58, tx:    1213548, lsn: 3/660051B0, prev 3/66003B30, bkp: 0000, desc: delete: rel 1663/16486/11786; tid 42/38 KEYS_UPDATED
rmgr: Heap        len (rec/tot):     26/  7902, tx:    1213548, lsn: 3/660051F0, prev 3/660051B0, bkp: 1000, desc: delete: rel 1663/16486/11797; tid 0/80 KEYS_UPDATED
```

C'est un extrait du résultat de la commande, le première ligne indique un insert avec la transaction N° 1213547, la ligne ne dessous indique un commit. En revanche à la troisième ligne on est passé à la transaction 1213548 avec un lock suivi de plusieurs lignes indiquant un delete. Il s'agit du drop fatal.

On va donc revenir à la transaction N°1213547. Voici le contenu de mon fichier recovery.conf :

```
restore_command = 'rsync "192.168.1.128:/var/lib/postgresql/9.3/wal_archive/%f" "%p"'

recovery_target_xid = '1213547'
recovery_target_inclusive = 'true'
```

La directive recovery\_target\_inclusive indique au moteur s'il inclue la transaction correspondant au N°1213547 ou s'il s'arrête juste avant. Ainsi j’aurais pu spécifier un xid de 1213548  et un target_inclusive à « false » pour obtenir le même résultat.

Ensuite il suffit de redémarrer postgres, voici ce qu'on peut observer dans les logs :

```
2015-03-22 12:41:17 CET LOG:  database system was interrupted; last known up at 2015-03-22 12:22:59 CET
2015-03-22 12:41:17 CET LOG:  creating missing WAL directory "pg_xlog/archive_status"
2015-03-22 12:41:17 CET LOG:  starting point-in-time recovery to XID 1213547
2015-03-22 12:41:17 CET LOG:  connection received: host=[local]
2015-03-22 12:41:17 CET LOG:  incomplete startup packet
2015-03-22 12:41:18 CET LOG:  connection received: host=[local]
2015-03-22 12:41:18 CET FATAL:  the database system is starting up
2015-03-22 12:41:18 CET LOG:  restored log file "000000010000000300000065" from archive
2015-03-22 12:41:18 CET LOG:  redo starts at 3/65000028
2015-03-22 12:41:18 CET LOG:  consistent recovery state reached at 3/650000F0
2015-03-22 12:41:18 CET LOG:  connection received: host=[local]
2015-03-22 12:41:18 CET FATAL:  the database system is starting up
2015-03-22 12:41:19 CET LOG:  connection received: host=[local]
2015-03-22 12:41:19 CET FATAL:  the database system is starting up
2015-03-22 12:41:19 CET LOG:  restored log file "000000010000000300000066" from archive
2015-03-22 12:41:19 CET LOG:  recovery stopping after commit of transaction 1213547, time 2015-03-22 12:23:35.834785+01
2015-03-22 12:41:19 CET LOG:  redo done at 3/660000F0
2015-03-22 12:41:19 CET LOG:  last completed transaction was at log time 2015-03-22 12:23:35.834785+01
rsync: link_stat "/var/lib/postgresql/9.3/wal_archive/00000002.history" failed: No such file or directory (2)
rsync error: some files/attrs were not transferred (see previous errors) (code 23) at main.c(1536) [Receiver=3.0.9]
2015-03-22 12:41:19 CET LOG:  selected new timeline ID: 2
rsync: link_stat "/var/lib/postgresql/9.3/wal_archive/00000001.history" failed: No such file or directory (2)
2015-03-22 12:41:19 CET LOG:  connection received: host=[local]
2015-03-22 12:41:19 CET FATAL:  the database system is starting up
rsync error: some files/attrs were not transferred (see previous errors) (code 23) at main.c(1536) [Receiver=3.0.9]
```

On remarque que le moteur créée le répertoire des journaux de transaction, ensuite la ligne suivante est assez explicite : « starting point-in-time recovery to XID 1213547 ». Quelques lignes plus bas on remarque qu'il rejoue les journaux de transaction (« restored log file « 000000010000000300000065 » from archive »).

Il arrête la restauration jusqu’à la transaction spécifiée :

```
recovery stopping after commit of transaction 1213547, time 2015-03-22 12:23:35.834785+01
```

On remarque quelques lignes d'erreur correspondant à la sortie de rsync. Visiblement il cherche à copier plusieurs fichiers en « x.history ».  En regardant dans le répertoire d'archivage on constate que le moteur a créé un fichier 00000002.history contenant :

`1       3/66000120      after transaction 1213547`

Point important également, on constate que la numérotation des journaux de transaction a changée :

```
-rw------- 1 postgres postgres 16777216 mars  22 12:23 000000010000000300000065
-rw------- 1 postgres postgres      305 mars  22 12:23 000000010000000300000065.00000028.backup
-rw------- 1 postgres postgres 16777216 mars  22 12:41 000000010000000300000066
-rw------- 1 postgres postgres 16777216 mars  22 12:46 000000020000000300000066
-rw------- 1 postgres postgres       39 mars  22 12:41 00000002.history
```

Le fichier « 000000010000000300000065 » correspond au wal lors du pg_basebackup, « 000000010000000300000066 » correspond au checkpoint que j'ai lancé après l'incident. Après la restauration on peut voir le fichier « 000000020000000300000066 ». Le huitième caractère est passé de 1 à 2.

En retournant dans les logs du moteur on peut lire la ligne suivante un peu plus bas : "selected new timeline ID: 2" En fait, lors de la restauration, Postgres créée une nouvelle [timeline](http://docs.postgresqlfr.org/current/continuous-archiving.html#backup-timelines). C'est un concept que j'aborderai plus tard, en attendant je vous invite à voir cette présentation sur le fonctionnement des timeline après une restauration PITR :
<https://wiki.postgresql.org/images/e/e5/FOSDEM2013-Timelines.pdf>

**Bien entendu comme tout système de sauvegarde vous devez jouer la procédure afin de valider qu'il y a pas eu d'erreur ou oubli.**



 [1]: http://blog.anayrat.info/2015/02/26/backup-restauration-pitr-part-6/ "Backup, restauration – Part 6"
