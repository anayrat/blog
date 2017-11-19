---
title: PostgreSQL 10 and Logical replication - Overview
author: Adrien Nayrat
type: post
draft: false
date: 2017-07-29T20:53:37+00:00
categories:
  - Postgres
tags:
  - postgres
  - logical replication


---

Next PostgreSQL version is approaching. This version comes with an impressive feature list :

  * Native partionning
  * Sorts and aggregation improvements
  * Better parallelism support : parallel index scan, parallel hash join, parallelism for subquery
  * Extended statistics
  * ICU collation: enable use of "abbreviated keys", disabled in 9.5.2 due to libc bug. Abbreviated keys brings sort improvements (arround 20-30%). It is usefull when a query need a sort or for index creation.
  * ... look at wiki page [New in Postgres 10](https://wiki.postgresql.org/wiki/New_in_postgres_10) or [releases notes](https://www.postgresql.org/docs/10/static/release-10.html).


Another attended feature is _logical replication_. I will present it in a serie of articles.

<!--more-->

  1. [PostgreSQL 10 : Logical replication - Overview][1]
  2. [PostgreSQL 10 : Logical replication - Setup][2]
  3. [PostgreSQL 10 : Logical replication - Limitations][3]


# Reminder

Replication already exists in PostgreSQL [^5]. It was based on "physical" replication: Postgres does not replicate queries but the *result*. More precisely modifications of the blocks.

[^5]: Standby replays WAL to apply blocks modifications. This technique is quite simple and is particularly effective and reliable.

However it has some limitations :

  * there is no granularity, we have to replicate the whole instance
  * it is not possible to replicate between different architectures (x86, ARM ...).
  * standby does not accept any request for writing. It is not possible to create custom views or indexes.


# Operation

Unlike physical replication, logical replication does not replicate data blocks. It decodes the result of queries that are sent to secondary. Then, the secondary applies SQL changes from the logical replication stream.

In fact, secondary server is a primary server, in the sense that it is a server that accepts write requests like any other primary instance.

It will be possible to choose tables to replicate, add views and indexes on the secondary server etc ...

{{< figure src="/img/2017/schema-repli-logique.png" title="Logical replication" >}}


  1. A "publication" is created on the primary server ("publisher"), which includes tables belonging to the publication.
  2. The secondary subscribes to this publication, it is a *subscriber*
  3. Then a special process is launched: the _bgworker logical replication_. It will connect to a replication slot on the primary.
  4. The primary will logically decode WAL to retrieve the results of the SQL statements.
  5. The logical stream is passed to the secondary that applies them.

Implementation in next article...

[1]: http://blog.anayrat.info/2017/07/29/postgresql-10-et-la-replication-logique-fonctionnement/
[2]: http://blog.anayrat.info/2017/08/05/postgresql-10-et-la-replication-logique-mise-en-oeuvre/
[3]: https://blog.anayrat.info/2017/08/27/postgresql-10-et-la-replication-logique-restrictions/
[4]: http://blog.anayrat.info/wp-content/uploads/2017/07/schema-repli-logique.png
