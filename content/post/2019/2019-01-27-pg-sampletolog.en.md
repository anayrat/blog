+++
title = "pg_sampletolog: An extension to log a sample of statements"
date = 2019-01-28T08:00:00+02:00
draft = false

summary = "This extension can be useful for diagnosing requests with very short execution times."


# Tags and categories
# For example, use `tags = []` for no tags, or the form `tags = ["A Tag", "Another Tag"]` for one or more tags.
tags = ["postgres","extension","pg_sampletolog","sampling","logs"]
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

This article will introduce you to an extension that I developed in order to log
a sample of statements.

When a DBA is faced with a performance problem, he will inspect
logs, but also the `pg_stat_stat_statements` view.
An expensive query will appear in `pg_stat_stat_statements` and in the logs if
the query exceeds `log_min_duration_statement`. Thus, the DBA can replay the query,
and obtains its execution plan to investigate.

To go even further, it is possible to enable the `auto_explain` extension.
This way we will have the plan of the query directly. For information, the option
`auto_explain.log_analyze` does not imply a double execution of the query.
This parameter can be activated without fear. However, this can be costly because
postgres must activate per node instrumentation to obtain *timings*. If we have a
high throughput, it is also possible to sample with `auto_explain.sample_rate`.
This option can produce a large quantity of logs which can be problematic on
an instance with high throughput.


I was faced with a very simple problem: how to investigate a request with a
very short execution time? It's very simple "Look at `pg_stat_stat_statements`!".

Here is what you could get on a pgbench test:
```
query               | SELECT abalance FROM pgbench_accounts WHERE aid = $1
calls               | 12000
total_time          | 214.564185000001
min_time            | 0.013751
max_time            | 0.044711
mean_time           | 0.0178803487499999
```

The query is standardized. Without parameter it is impossible to get your plan.
Choosing a random parameter is not the right solution: it is not necessarily
representative of the real production traffic.

A few months ago, I proposed a patch to log a sample of queries.
This one has been integrated in version 12 still under development:

```
commit 88bdbd3f746049834ae3cc972e6e650586ec3c9d
Author:     Alvaro Herrera <alvherre@alvh.no-ip.org>
AuthorDate: Thu Nov 29 18:42:53 2018 -0300
Commit:     Alvaro Herrera <alvherre@alvh.no-ip.org>
CommitDate: Thu Nov 29 18:42:53 2018 -0300

    Add log_statement_sample_rate parameter

    This allows to set a lower log_min_duration_statement value without
    incurring excessive log traffic (which reduces performance).  This can
    be useful to analyze workloads with lots of short queries.

    Author: Adrien Nayrat
    Reviewed-by: David Rowley, Vik Fearing
    Discussion: https://postgr.es/m/c30ee535-ee1e-db9f-fa97-146b9f62caed@anayrat.info
```

In the thread, Nikolay Samokhvalov put forward the idea of having the same type
functionality but at the transaction level: <https://www.postgresql.org/message-id/CANNMO%2BLg65EFqHb%2BZYbMLKyE2y498HJzsdFrMnW1dQ6AFJ3Mpw%40mail.gmail.com>

I also proposed a patch to this purpose: [Log a sample of transactions](https://commitfest.postgresql.org/21/1923/)

All this stuff is interesting, but you'll have to wait until version 12, and
there's no guarantee that the second patch will be committed (or that the first
one will not be reverted).

All this gave me the idea and the desire to create an extension, that's how
[pg_sampletolog](https://github.com/anayrat/pg_sampletolog) was born.

pg_sampletolog allows to:

  * Log a sample of statements
  * Log a sample of transactions
  * Log before or after execution (in order to be compatible with pgreplay)
  * Log all `DDL` or `MOD` statements, same as [log_statement](https://www.postgresql.org/docs/current/runtime-config-logging.html#GUC-LOG-STATEMENT)

It should works on all supported version from 9.4 to 11.

pg_sampletolog must me loaded either:

  * In local session with `LOAD 'pg_sampletolog';` order.
  * In `session_preload_libraries`. To be loaded at connection start.

New settings should be visible in `pg_settings` view:

```
select name from pg_settings where name like 'pg_sampletolog%';
                 name                 
----------------------------------------
 pg_sampletolog.disable_log_duration
 pg_sampletolog.log_before_execution
 pg_sampletolog.log_level
 pg_sampletolog.log_statement
 pg_sampletolog.statement_sample_rate
 pg_sampletolog.transaction_sample_rate
(6 rows)
```

Here are a few examples:

  * Log only 10% of statements: `pg_sampletolog.statement_sample_rate = 0.1`

pg_sampelog will log 10% of requests. For each statements, postgres will make a
random selection using `random()` function. The cost of this function is very low,
so there should be no impact on performance.

After a few requests you should get this kind of message in the logs :

```
2019-01-27 12:50:39.361 CET [27047] LOG:  Sampled query duration: 0.014 ms - SELECT 1;
```
pg_sampelog will log the statement and its execution time.

  * Log only 10 of transactions: `pg_sampletolog.transaction_sample_rate = 0.1`

The operation is the same as before, with the difference that postgres will choose
whether or not to log all statements for the same transaction. This can be very
useful in understanding what an application does. For example, when application
code is not accessibled or when statements are generated by an ORM.

Example with a simple transaction: `BEGIN; SELECT 1; SELECT 1; COMMIT;`
```
2019-01-27 12:51:40.562 CET [27069] LOG:  Sampled transaction duration: 0.008 ms - SELECT 1;
2019-01-27 12:51:40.562 CET [27069] LOG:  Sampled transaction duration: 0.005 ms - SELECT 1;
```

Both SELECTs have been successfully logged. By changing  `log_line_prefix`, we
can see that it is the same transaction (look at the *lxid*):

```
2019-01-27 16:32:16 CET [18556]: lxid=3/177,db=postgres,user=anayrat LOG:  Sampled transaction - SELECT 1;
2019-01-27 16:32:16 CET [18556]: lxid=3/177,db=postgres,user=anayrat LOG:  Sampled transaction - SELECT 1;
```

  * Log all DDL statements: `pg_sampletolog.log_statement = 'ddl'`:

pg_sampletolog will log all DDL orders (`CREATE TABLE`,`CREATE INDEX`,...).
This can be useful if you just want to log a read sample but all DDL orders.

```
2019-01-27 12:53:47.564 CET [27103] LOG:  Sampled ddl CREATE TABLE t1(c1 int);
```

  * Log all data-modifying statements: `pg_sampletolog.log_statement = 'mod'`:

Exactly like the previous example, but this time we also log all the `UPDATES, DELETE`.
This includes DDL orders.

```
2019-01-27 12:59:54.043 CET [27160] LOG:  Sampled query duration: 0.246 ms - INSERT INTO t1 VALUES(1);
2019-01-27 13:00:16.468 CET [27160] LOG:  Sampled ddl CREATE INDEX ON t1(c1);
```

  * Log before execution: `pg_sampletolog.log_before_execution = on`

This option could be useful to replay logs with [pgreplay](https://github.com/laurenz/pgreplay).


# Bonus

The extension also works on standby servers.

# Extra bonus

If `pg_stat_stat_statements` is enabled, the queryid is also logged. This can be
very useful if you identify a query in `pg_stat_stat_statements` and want to find
it in the logs by using its queryid.

# Conclusion

I enjoyed this personal project. I learned a lot about ~~segfaults~~ postgres
code and it also shows how postgres is extensible.

In the future I would like to add the possibility to log a sample statement
corresponding to such queryid. I also have to look to support prepared statements.

Finally, I would like to test this extension with pgreplay: By logging all *MOD*
orders (to ensure consistency during replay) as well as a fraction of the read
queries. Then, restore a PITR backup and on the one hand replay writes.
On the other hand, replay a portion of reads with a *speed_factor*. For example
x10 by replaying 10% of the traffic. Even if it will never be perfect (it will
lack reads consistency), I will be curious to see the results that we can get.
Especially if logging all statements would be too expensive.

I'm interested in any feedback to make on [the github of the project](https://github.com/anayrat/pg_sampletolog).
