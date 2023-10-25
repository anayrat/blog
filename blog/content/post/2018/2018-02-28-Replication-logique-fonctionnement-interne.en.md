+++
title = "Logical replication internals"
date = 2018-03-10T12:19:41+01:00
draft = false
summary = "This post explains how logical replication works. Especially, difference behaviors depending of workloads"
authors = ['adrien']

# Tags and categories
# For example, use `tags = []` for no tags, or the form `tags = ["A Tag", "Another Tag"]` for one or more tags.
tags = ["postgres","logical replication", "netdata"]
categories = ["Postgres"]
show_related = true

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


I introduced replication through several posts:

1. [PostgreSQL 10 : Logical replication - Overview][1]
2. [PostgreSQL 10 : Logical replication - Setup][2]
3. [PostgreSQL 10 : Logical replication - Limitations][3]

This new post will dig a little deeper. We will see postgres internals about
logical replication. To ease comprehension, I added charts in Netdata that I
presented in a previous article. I invite you to click on the images to enlarge them,
they will be more readable :smile:

I would like to thank [Guillaume Lelarge](https://twitter.com/g_lelarge) who
reviewed this post :)

# Spills changes on disk

I noticed that during a big transaction, files appeared
in the pg_replslot directory of each replication slot. That's pushed me to try
to understand what these files were for.

PostgreSQL must reorder changes to be applied in good order. To do this, wal sender
processes proceed to logical decoding and reorder changes in memory.

However, if the transaction contains a lot of changes, it would eat
a lot of memory. So, Postgres writes these changes on disk.

After an exploration in the code, we arrive in the file `src/backend/replication/logical/reorderbuffer.c` :

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

From 4096 changes Postgres spills changes to disk:

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

Setting `log_min_messages` parameter to `debug2`, we can highlight this behavior.
The goal is to display the message corresponding to the line 2078.

Without any particular activity, here is the content of the `pg_replslot` directory:

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

I have two directories, corresponding to two replication slots. Indeed,
I created a publication and two instances subscribed to the same publication.
They each have their replication slot: mysub and mysub3.

## Example with a single transaction

Let's add rows to a table belonging to a publication.


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

So far we have not reached the threshold of 4096. The directory does not contain
the state file yet:


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

Let's add a line:

```SQL
insert into t1 (select  i, md5(i::text) from generate_series(1,1) i);
INSERT 0 1
postgres=# select count(*) from t1;
 count
-------
  4096
(1 ligne)
```

This time, threshold is reached (`if (txn-> nentries_mem> = max_changes_in_memory`).
Wal sender process will spills changes to apply on disk.

We find in the logs these messages (with `log_min_messages = debug2`):

```
[1977] postgres@postgres DEBUG:  spill 4096 changes in XID 51689068 to disk
[2061] postgres@postgres DEBUG:  spill 4096 changes in XID 51689068 to disk
```

Corresponding to both wal sender processes:

```
ps faux |grep -E '(1977|2061)'
postgres  1977  3.3  0.3 390496 37984 ?        Ss   11:38   8:54  \_ postgres: 10/main: wal sender process postgres 192.168.1.26(44188) idle
postgres  2061  3.4  0.3 390496 37932 ?        Ss   11:38   9:10  \_ postgres: 10/main: wal sender process postgres 192.168.1.26(44192) idle
```

At the same time, if we look at the contents of the directory pg_replslot, we
note that new files have appeared:

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

Each wal sender had to serialize changes on disk.

Please note that one change here means one tuple, not one SQL statement.
For example, a single insert of 4096 lines will result in the writing of a \*.snap file.

Edit: Postgres hackers changes the `snap` extension to `spill` for Postgres 12:

```
commit ba9d35b8eb8466cf445c732a2e15ca5790cbc6c6
Author:     Jeff Davis <jdavis@postgresql.org>
AuthorDate: Sat Aug 25 22:45:59 2018 -0700
Commit:     Jeff Davis <jdavis@postgresql.org>
CommitDate: Sat Aug 25 22:52:46 2018 -0700

    Reconsider new file extension in commit 91f26d5f.

    Andres and Tom objected to the choice of the ".tmp"
    extension. Changing to Andres's suggestion of ".spill".

    Discussion: https://postgr.es/m/88092095-3348-49D8-8746-EB574B1D30EA%40anarazel.de
```

## Example with two transactions

Second example, with two transactions modifying the table t1.

The xid of the first transaction:

```SQL
select txid_current();
 txid_current
--------------
     51689070
(1 ligne)
```

That of the second transaction:

```SQL
select txid_current();
 txid_current
--------------
     51689071
(1 ligne)
```

If we insert 4096 rows in the first transaction:

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

Then, if we insert 4096 rows in the second transaction, we get new \*.snap files:

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

I tried to insert 3000 rows in each transaction, no snap file were written.
Only when each transaction exceeds 4096 changes changes are spilled on disk.

Let's commit the first transaction:

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

Corresponding snap file is deleted by each wal sender. If we commit
the second transaction, the second file is also deleted.

Note that these files appear for any modification (`DELETE, UPDATE` ...).

What's happen for a big transaction?

In this example we will insert a large amount of records into
a transaction that will be committed later. This is to highlight how logical
replication works.

Here are the queries that have been executed:

```SQL
BEGIN.
INSERT INTO t1 (select  i, md5(i::text) FROM generate_series(1,10000000) i); --16:27:13
-- wait few minutes
COMMIT; --16:33:46
```

For information, inserts were done after 16:27:13 and ended at 16:28:16.

The commit was only done much later, at 16:33:46.

We find that both wal sender processes spill changes on disks:

```
[1977] postgres@postgres DEBUG:  spill 4096 changes in XID 51689075 to disk
[2061] postgres@postgres DEBUG:  spill 4096 changes in XID 51689075 to disk
[1977] postgres@postgres DEBUG:  spill 4096 changes in XID 51689075 to disk
[2061] postgres@postgres DEBUG:  spill 4096 changes in XID 51689075 to disk
[1977] postgres@postgres DEBUG:  spill 4096 changes in XID 51689075 to disk
[2061] postgres@postgres DEBUG:  spill 4096 changes in XID 51689075 to disk
```

Which produce a large amount of files:
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

Here are some charts from Netdata:

## CPU

{{< figure src="/img/2018/logical-rep-big-tx/cpu.png" link="/img/2018/logical-rep-big-tx/cpu.png" title="CPU" >}}

There is a high CPU load, up to 16:28:16 where it drops a little.
Then at 16:28:48 load drops to 0. There is a rise in the load at 16:33:46.

How to explain these results?

At the first peak of load, there were several processes that consumed the resource:

  * the backend that inserted the data
  * both wal sender processes

The machine has 8 logical cores, each process uses only one core. About 13% per
process.

At 16:28:16, the backend had just completed the insertion. However wal sender processes
continued to be active. WAL Decoding and transactions serialization  are quite expensive.

Finally, at 16:33:46, the transaction is committed. It did not affect
the backend load. On the other hand, wal sender processes began to read
snap files produced on disk to transmit changes to subscribers.

Note that the changes were visible on the subscriber side only around 16:35:40.
Once the changes have been replayed. Replay is longer than the insertion.
It depends a lot of the performances of the processors (CPU bound).

## Database-wide statistics

{{< figure src="/img/2018/logical-rep-big-tx/db_stat.png" link="/img/2018/logical-rep-big-tx/db_stat.png" target="\_blank" title="tuple inserted" >}}

There is a peak insertion at 16:33:46. This corresponds to the moment when
the transaction has been committed.

## Network

{{< figure src="/img/2018/logical-rep-big-tx/network.png" link="/img/2018/logical-rep-big-tx/network.png" title="Network" >}}

There is a significant network traffic between 16:33:46 and 16:35:40. We can
deduce that the changes are only transmitted after the commit.

Changes are not transmitted by using streaming method. This has the advantage
to avoid sending changes that would not be committed. On the other hand,
this adds time to changes application.

Thomas Vondra has submitted a patch to improve this use case:
[logical streaming for large in-progress transactions](https://commitfest.postgresql.org/16/1429/)

For the moment the patch has not been accepted yet. It requires adding
information in the WAL. The community is very cautious in
which concerns everything that can have an impact on performance.

## Replication

{{< figure src="/img/2018/logical-rep-big-tx/replication.png" link="/img/2018/logical-rep-big-tx/replication.png" title="Replication delta" >}}


We can notice that "Streaming replication delta" charts, measuring the delta of
replication from the `pg_stat_replication` view are hard to explain.
Normally they show secondary server replication delay. Except
we have just seen that the application of the changes is only made at commit time,
at 16:33:46. But charts seem indicate that the secondaries servers were
delayed until 16:28:48.

The curves "write delta", "flush delta" and "replay delta" are similar.
The curve "sent delta" is quite different.

My understanding of these graphs is that "write delta", "flush delta" and "replay delta"
correspond to the logical decoding delay compared to sent data.

At 16:28:16, the query finished, "sent delta" stops growing and decreases as
logical decoding advance.

We can confirm this thanks to "replication slot files" chart, the curve
"pg_replslot files" is the number of files in the directory
"pg_replslot" of each replication slot. This one is growing and stops
to grow at the same time as the curves "write delta", "flush delta" and
"replay delta" go back to 0.

We can notice postgres have to keep WALs until the transaction has been committed.
It is quite logical, if a crash occurs, you should be able to read WALs
to proceed to logical decoding again.

Finally, at 16:35:40 changes have been transmitted to the subscriber side, postgres
can finally clean WALs and snaps files.

# OLTP workload

And with OLTP workload? For this test, I used pgbench launched for 10 minutes
on a base from which all tables are replicated.

{{< figure src="/img/2018/logical-rep-oltp/cpu.png" link="/img/2018/logical-rep-oltp/cpu.png" title="CPU" >}}

{{< figure src="/img/2018/logical-rep-oltp/network.png" link="/img/2018/logical-rep-oltp/network.png" title="Network activity" >}}

{{< figure src="/img/2018/logical-rep-oltp/db_stat.png" link="/img/2018/logical-rep-oltp/db_stat.png" title="Xact" >}}

{{< figure src="/img/2018/logical-rep-oltp/replication.png" link="/img/2018/logical-rep-oltp/replication.png" title="Replication delta" >}}

Note that network traffic drops at the same time as we observe replication delay growth.
This corresponds to periods when secondary were CPU bound (unfortunately I do
not have Netdata chart on the secondary).

This can be explained by the fact that the replay is done only by a single
bgworker process who had trouble supporting the load (less powerful machine).
Moreover we can see that the secondary catch up the delay a few seconds after
the end of the bench.

Another important point is that there is no snap file written on disk.
This confirms the comment in the code, indicating that in case of OLTP workload,
changes are kept in memory and not spilled on disk.

# Conclusion

We can see that the logical replication applies quite well in the case
OLTP workload as long as the secondary is able to follow application of the
changes. This implies that write traffic is not too important
at the risk of having a secondary that has difficulty in follow the charge:
changes replay are made by a single process.

On the other hand, in cases where workload is different (OLAP), where requests lead to
important changes, postgres will have to spill changes on disk. Logical decoding
and network transfer result in a delay which can be significant.



[1]: https://blog.anayrat.info/en/2017/07/29/postgresql-10-and-logical-replication-overview/
[2]: https://blog.anayrat.info/en/2017/08/05/postgresql-10-and-logical-replication-setup/
[3]: https://blog.anayrat.info/2017/08/27/postgresql-10-et-la-replication-logique-restrictions/
