---
title: Partition xlogs pleine
authors: ['adrien']
type: post
date: 2015-05-18T20:00:46+00:00
aliases: /2015/05/18/postgres-partition-xlogs-pleine/
categories:
  - Linux
  - Postgres
tags:
  - postgres
show_related: true

---
Cette semaine j'ai été confronté à un incident sur une instance postgres. La partition xlogs était pleine :

```
PANIC: could not write to file "pg_xlog/xlogtemp.12352": No space left on device
LOG: startup process (PID 12352) was terminated by signal 6: Aborted
LOG: aborting startup due to startup process failure
```

Cette instance est répliquée sur un autre serveur et archive ses journaux de transaction sur un troisième serveur. Ce dernier était plein, Postgres est intelligent et a conservé ses journaux de transaction en attendant de pouvoir les archiver... Jusqu'à ce que la partition des journaux se retrouve pleine entraînant l'arrêt de Postgres.

Le nettoyage sur le serveur d'archivage n'était pas suffisant, il fallait libérer de la place sur la partition. La solution qui vient en tête est d'agrandir la partition. Je ne souhaitais pas agrandir la partition. J'ai donc fait un truc tout simple : Déplacer un fichier journal et faire un lien symbolique vers celui-ci. Postgres est reparti et j'ai lancé un « CHECKPOINT » immédiatement ce qui a libéré de la place. Ensuite il faut penser à supprimer le fichier car Postgres a juste supprimé le lien.

C'est plus rapide à mettre en place que d'agrandir la partition.

En fouillant sur google je suis tombé sur ce site :

[Solving pg_xlog out of disk space problem on Postgres](http://blog.endpoint.com/2014/09/pgxlog-disk-space-problem-on-postgres.html)

L'auteur nous donne une astuce toute simple : Créer un fichier plein de vide. Si un jour la partition est pleine il suffit de le supprimer pour faire repartir l'instance :

`dd if=/dev/zero of=DO_NOT_MOVE_THIS_FILE bs=1MB count=100`

Bilan : Il faut bien penser à superviser l'archivage de Postgres!
