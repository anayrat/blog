---
title: PostgreSQL 10 et la réplication logique – Mise en oeuvre
author: Adrien Nayrat
type: post
date: 2017-08-05T11:13:43+00:00
aliases: /2017/08/05/postgresql-10-et-la-replication-logique-mise-en-oeuvre/
categories:
  - Linux
  - Postgres
tags:
  - postgres
  - replication logique

---
Cet article est la suite d'une série d'articles sur la réplication logique dans la version 10 de PostgreSQL

Celui-ci va porter sur la mise en œuvre de la réplication logique.

<!--more-->

1. [PostgreSQL 10 et la réplication logique - Fonctionnement][1]
2. [PostgreSQL 10 et la réplication logique - Mise en oeuvre][2]
3. [PostgreSQL 10 et la réplication logique - Restrictions][3]

{{% toc %}}

# Installation

Lors de la rédaction de cet article la version 10 n'est pas encore sortie. Cependant la communauté met à disposition les paquets des versions bêtas. Bien entendu, à **ne pas utiliser en production**.

## Installation du dépôt pgdg (PostgreSQL Developpement Group)

Il suffit d'aller sur le site http://www.postgresql.org puis « download » -> « debian ». Sur cette page, le site vous indique la marche à suivre pour installer le dépôt du pgdg. Cependant il ne va vous proposer que les versions stables. Mais, le site vous renvoie vers une page du wiki :

> For more information about the apt repository, including answers to frequent questions, please see the apt page on [the wiki][4].

Sur la page wiki vous trouverez :

> For packages of development/alpha/beta versions of PostgreSQL, see the [FAQ entry about beta versions][5].

Qui vous indique :

> To use packages of postgresql-10, you need to add the 10 component to your `/etc/apt/sources.list.d/pgdg.list` entry, so the 10 version of libpq5 will be available for installation

Donc pour debian, les commandes se résument à :

```bash
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ $(lsb_release -cs)-pgdg main 10" > /etc/apt/sources.list.d/pgdg.list'
sudo apt-get update
```

## Installation des paquets

```bash
sudo apt install postgresql-10
```


## Création des instances

Une première instance est créée lors de l'installation. Nous allons installer une seconde instance. Ainsi l'une sera le « publisher » et l'autre sera « subscriber ».

```bash
# pg_lsclusters
Ver Cluster Port Status Owner Data directory Log file
10 main 5432 online postgres /var/lib/postgresql/10/main /var/log/postgresql/postgresql-10-main.log

# pg_createcluster 10 sub
Creating new PostgreSQL cluster 10/sub ...
/usr/lib/postgresql/10/bin/initdb -D /var/lib/postgresql/10/sub --auth-local peer --auth-host md5
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "en_US.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are disabled.

fixing permissions on existing directory /var/lib/postgresql/10/sub ... ok
creating subdirectories ... ok
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting dynamic shared memory implementation ... posix
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok

Success. You can now start the database server using:

/usr/lib/postgresql/10/bin/pg_ctl -D /var/lib/postgresql/10/sub -l logfile start

Ver Cluster Port Status Owner Data directory Log file
10 sub 5433 down postgres /var/lib/postgresql/10/sub /var/log/postgresql/postgresql-10-sub.log

# pg_ctlcluster 10 sub start

# pg_lsclusters
Ver Cluster Port Status Owner Data directory Log file
10 main 5432 online postgres /var/lib/postgresql/10/main /var/log/postgresql/postgresql-10-main.log
10 sub 5433 online postgres /var/lib/postgresql/10/sub /var/log/postgresql/postgresql-10-sub.log
```

# Mise en place de la réplication logique

## Prérequis

Il y a très peu de changements à apporter à la configuration par défaut. La plupart des paramètres sont déjà positionnés pour mettre en place la réplication logique. Toutefois il y a un paramètre à modifier sur l'instance qui « publie » : le `wal_level`. Celui-ci est à "`replica`" , il doit être placé à "`logical`" .

Ca se passe dans `/etc/postgresql/10/main/postgresql.conf`

```
wal_level = logical
```

Pour appliquer le changement il faut redémarrer l'instance :

```
pg_ctlcluster 10 main restart
```

## Publication

Créons une base b1 qui contient une table t1.

```
postgres=# create database b1;
CREATE DATABASE
postgres=# \c b1
You are now connected to database "b1" as user "postgres".

b1=# create table t1 (c1 text);
CREATE TABLE
b1=# insert into t1 values ('un');
INSERT 0 1
b1=# select * from t1;
 c1
----
 un
(1 row)
```

Ensuite nous allons jouer l'ordre pour créer la publication :

```
b1=# CREATE PUBLICATION pub1 FOR TABLE t1 ;
```

Et c'est tout! Notez qu'il est possible d'utiliser le mot clé « `FOR ALL TABLES` » pour que le PostgreSQL ajoute toutes les tables présentes et futures à la publication.


On peut vérifier que la publication a bien été créée avec la méta-commande psql suivante :


```
b1=# \dRp+
 Publication pub1
 All tables | Inserts | Updates | Deletes
------------+---------+---------+---------
 f | t | t | t
Tables:
 "public.t1"
```

## Souscription

La réplication logique ne réplique pas les ordres DDL, nous devons donc créer la table sur l'instance « sub ». Il n'est pas nécessaire d'avoir le même nom de base, nous allons donc créer une base b2.

```
postgres@blog:~$ psql -p 5433
psql (10beta2)
Type "help" for help.

postgres=# CREATE DATABASE b2;
CREATE DATABASE
postgres=# \c b2
You are now connected to database "b2" as user "postgres".
b2=# create table t1 (c1 text);
CREATE TABLE
```

Comme pour la publication, la souscription se crée avec un ordre SQL :

```
b2=# CREATE SUBSCRIPTION sub1 CONNECTION 'host=/var/run/postgresql port=5432 dbname=b1' PUBLICATION pub1;
NOTICE: created replication slot "sub1" on publisher
CREATE SUBSCRIPTION
b2=# select * from t1;
 c1
----
 un
(1 row)
```

La partie la plus importante est : `CONNECTION 'host=/var/run/postgresql port=5432 dbname=b1'`

En effet, on indique au « subscriber » comment se connecter à la publication. Dans mon exemple les deux instances sont sur la même machine et écoutent sur une socket locale. Si vous avez des instances sur différentes machines il faudra spécifier l'adresse IP. Ensuite le port et ne surtout pas oublier la base où a été créée la publication.

Comme vous pouvez le voir, le moteur s'est automatiquement chargé de rapatrier les données qui étaient déjà présentes dans la table t1.

Maintenant si j'ajoute des lignes dans la table t1, elles seront automatiquement répliquées sur l'instance « sub » :

```psql
postgres@blog:~$ psql b1
psql (10beta2)
Type "help" for help.

b1=# insert into t1 values ('deux');
INSERT 0 1
postgres=# \q
postgres@blog:~$ psql -p 5433 b2
psql (10beta2)
Type "help" for help.
b2=# select * from t1;
 c1
------
 un
 deux
(2 rows)
```

Comme pour la publication, il existe une méta-commande pour afficher les souscriptions :

```psql
\dRs+
 List of subscriptions
 Name | Owner | Enabled | Publication | Synchronous commit | Conninfo
------+----------+---------+-------------+--------------------+----------------------------------------------
 sub1 | postgres | t | {pub1} | off | host=/var/run/postgresql port=5432 dbname=b1
(1 row)
```

C'est tout pour cet article, dans un prochain article nous verront les restrictions d'usage.

[1]: http://blog.anayrat.info/2017/07/29/postgresql-10-et-la-replication-logique-fonctionnement/
[2]: http://blog.anayrat.info/2017/08/05/postgresql-10-et-la-replication-logique-mise-en-oeuvre/
[3]: https://blog.anayrat.info/2017/08/27/postgresql-10-et-la-replication-logique-restrictions/
[4]: https://wiki.postgresql.org/wiki/Apt
[5]: https://wiki.postgresql.org/wiki/Apt/FAQ#I_want_to_try_the_beta_version_of_the_next_PostgreSQL_release "Apt/FAQ"
