+++
title = "Postgres and Netdata : apply to autovacuum and pg_stat_bgwriter"
date = 2018-02-20T18:35:22+01:00
draft = false
summary = "Introduction of autovacuum and bgwriter charts dedicated to PostgreSQL in Netdata"


# Tags and categories
# For example, use `tags = []` for no tags, or the form `tags = ["A Tag", "Another Tag"]` for one or more tags.
tags = ["postgres","netdata","monitoring","metrology"]
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

Some time ago, I discovered [Netdata](https://my-netdata.io/).
This tool allows to collect many metrics (CPU, network, etc).

The strength of Netdata is to be quite light and easy to use. It collects
data every second, everything is stored in memory. There is actually no
history. The goal is to have access to a set of metrics in real time. [^1]

I added several charts for PostgreSQL, taking a lot of ideas from
[check_pgactivity](https://github.com/OPMDG/check_pgactivity).

Here is the presentation and explanations of some charts. Some
highlight the behavior of PostgreSQL.

NB: The charts presented here may not correspond to the final version.
Several charts have been splited in order to distinguish different operations
(reads, writes), and processes (backend, checkpointer, bgwriter). [^2]

I would like to thank [Guillaume Lelarge](https://twitter.com/g_lelarge) who
reviewed this post :)

# Autovacuum

{{< figure src="/img/2018/netdata-postgres02.png" title="Autovacuum workers" >}}

This chart shows the activity of the processes called "autovacuum workers",
launched by the "autovacuum launcher".  Their role is to perform maintenance
tasks in order to:

  * Clean dead rows
  * Update statistics (see [Statistics, cardinality, selectivity](https://blog.anayrat.info/en/2017/11/26/postgresql---jsonb-and-statistics/#statistics-cardinality-selectivity))
  * Freeze old rows to avoid wraparound (See [Transaction Id Wraparound in Postgres](http://malisper.me/transaction-id-wraparound-in-postgres/))

Usually, there is no need to worry, the default configuration is enough in
most situations. However, it may happen that it is required to fine tune the
configuration of the autovacuum.

For example, if the activity is very sustained and the number of processes launched
reaches regularly `autovacuum_max_workers`. In such a case, it may be advisable:


  * to increase the number of autovacuum workers (if there are many tables to
  process at one time)
  * to make the processes more aggressive (when tables are large).
  Indeed, their activity is constrained by the parameters `autovacuum_vacuum_cost_delay`
  and `autovacuum_vacuum_cost_limit` so that this task is done in background
  to minimize the impact on other processes that process queries.

# Bgwriter

{{< figure src="/img/2018/netdata-postgres01.png" title="Bgwriter" >}}

This chart is called Bgwriter because of the [pg_stat_bgwriter](https://www.postgresql.org/docs/current/static/monitoring-stats.html#PG-STAT-BGWRITER-VIEW) view that gives these statistics.

The role of the "bgwriter" is to synchronize "dirty" blocks. These are blocks that
have been modified in shared buffers but not yet written to data files
(don't worry, they already are written on disk within a WAL).

The "checkpointer" is also in charge of synchronizing these blocks. Writes
are smoothed in background (this depends on the `checkpoint_completion_target` parameter).
The bgwriter occurs when backends need to release blocks in
shared buffers, unlike the checkpointer which does it at a checkpoint.


Going back to the chart, the blue curve corresponds to the number of blocks that
had to be allocated in shared buffers. The chart has this shape because the
screenshot was taken during a pgbench load test while the server had just been
started.

We can see quite clearly the effect of the cache which is gradually loading. It is
quite visible because the collection is done every second.

We can see this effect of loading the cache on several charts:

{{< figure src="/img/2018/netdata-postgres08.png" title="Disk IO" >}}
At the beginning of the chart, we can see a very sustained activity in reading.
Write peaks correspond to the checkpoints.


{{< figure src="/img/2018/netdata-postgres09.png" title="IOWait" >}}

On the graph, the interesting area is the pink one. You see that, at the beginning
of the test, CPUs were waiting for disk access.

{{< figure src="/img/2018/netdata-postgres03.png" title="Reads - Transaction per second" >}}

Here we can see that postgres had to read the blocks from the disk. The curve
decreases because subsequent reads are from shared buffers.

We can also see that the number of transactions per second increases, then
stabilizes once the data is loaded into shared buffers.

We observe the same phenomenon on this chart:

{{< figure src="/img/2018/netdata-postgres04.png" title="Bench database statistics" >}}

We can see that it took 1min30sec to load the cache.


[^1]: See : [Netdata Performance](https://github.com/firehol/netdata/wiki/Performance)
[^2]: The Pull Request has been merged : [Add charts for postgres](https://github.com/firehol/netdata/pull/3400)
