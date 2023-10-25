---
title: PostgreSQL 10 and Logical replication - Setup
authors: ['adrien']
type: post
date: 2017-08-05T11:13:43+00:00
draft: false
categories:
  - Linux
  - Postgres
tags:
  - postgres
  - logical replication
show_related: true
---
This article is the result of a series of articles on logical replication in PostgreSQL 10

This one will focus on the implementation of logical replication.

<!--more-->

1. [PostgreSQL 10 : Logical replication - Overview][1]
2. [PostgreSQL 10 : Logical replication - Setup][2]
3. [PostgreSQL 10 : Logical replication - Limitations][3]

{{% toc %}}

# Installation

When writing this article Postgres 10 is not released yet. However, the community provides packages of beta versions. Of course, **do not use in production**.


## Repository installation (PostgreSQL Developpement Group)

From <http://www.postgresql.org> then "download" -> "debian". On this page, the site tells you how to install the pgdg repository. However it will only offer stable versions. But, the site sends you to a wiki page:

> For more information about the apt repository, including answers to frequent questions, please see the apt page on [the wiki][4].

You will find :

> For packages of development/alpha/beta versions of PostgreSQL, see the [FAQ entry about beta versions][5].

Who tells you :

> To use packages of postgresql-10, you need to add the 10 component to your `/etc/apt/sources.list.d/pgdg.list` entry, so the 10 version of libpq5 will be available for installation

So for debian, commands are :

```bash
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ $(lsb_release -cs)-pgdg main 10" > /etc/apt/sources.list.d/pgdg.list'
sudo apt-get update
```

## Packages installation

```bash
sudo apt install postgresql-10
```


## Create instances

A first instance is created during installation. We will install a second one. Thus one will be the publisher and the other will be subscriber.
So for debian, commands are:

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

# Logical replication setup

## Prerequisites

There are very few changes to the default configuration. Most parameters are already set to set up logical replication. However, there is one parameter to modify on the instance that "publishes": `wal_level`. Set to "replica", it must be "logical".

In `/etc/postgresql/10/main/postgresql.conf`

```
wal_level = logical
```

To apply changes we must restart the cluster:

```
pg_ctlcluster 10 main restart
```

## Publication


Let's create a database b1 that contains a table t1.

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

Then, we use this order to create publication:

```
b1=# CREATE PUBLICATION pub1 FOR TABLE t1 ;
```

And that's all! Note that it is possible to use the keyword `FOR ALL TABLES` to add all present and future tables to the publication.

We can verify that the publication was created with the following psql meta-command:

```
b1=# \dRp+
 Publication pub1
 All tables | Inserts | Updates | Deletes
------------+---------+---------+---------
 f | t | t | t
Tables:
 "public.t1"
```

## Subscription

Logical replication does not replicate DDL orders, so we must create the table on the "sub" instance. It is not necessary to have the same database name, so we will create a database b2.

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

As for the publication, the subscription is created with an SQL order:

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

Most importing part is: `CONNECTION 'host=/var/run/postgresql port=5432 dbname=b1'`

Indeed, we indicate to the subscriber how to connect to the publication. In my example the two instances are on the same server and listen on a local socket. If you have instances on different server you will have to specify IP address. Then the port and do not forget the database where the publication was created.

As you can see, Postgres automatically synchronize data that was already present in table t1.

Now if I add rows in table t1, they will automatically be replicated to the instance "sub":

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

As for the publication, there is a meta-command to display subscriptions:

```psql
\dRs+
 List of subscriptions
 Name | Owner | Enabled | Publication | Synchronous commit | Conninfo
------+----------+---------+-------------+--------------------+----------------------------------------------
 sub1 | postgres | t | {pub1} | off | host=/var/run/postgresql port=5432 dbname=b1
(1 row)
```

That's all for this article, in a future article we will see limitations.

[1]: https://blog.anayrat.info/en/2017/07/29/postgresql-10-and-logical-replication-overview/
[2]: https://blog.anayrat.info/en/2017/08/05/postgresql-10-and-logical-replication-setup/
[3]: https://blog.anayrat.info/2017/08/27/postgresql-10-et-la-replication-logique-restrictions/
[4]: https://wiki.postgresql.org/wiki/Apt
[5]: https://wiki.postgresql.org/wiki/Apt/FAQ#I_want_to_try_the_beta_version_of_the_next_PostgreSQL_release "Apt/FAQ"
