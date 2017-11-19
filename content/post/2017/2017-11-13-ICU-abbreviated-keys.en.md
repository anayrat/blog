+++
title = "PostgreSQL 10 : ICU & Abbreviated Keys"
date = 2017-11-19T15:30:00+01:00
draft = false

# Tags and categories
# For example, use `tags = []` for no tags, or the form `tags = ["A Tag", "Another Tag"]` for one or more tags.
tags = ["postgres","sorts","collation","ICU"]
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

Almost everyone has heard of partitioning and logical replication in PostgreSQL 10. Have you heard about the support of ICU collations (_International Components for Unicode_)?

This article will present what this new feature is but also the possible gains by exploiting _abbreviated keys_.

<!--more-->

{{% toc %}}

# Short reminder about collation

In database, collation corresponds to character classification rules.

Here are some examples:

```SQL
create table t1 (c1 text);
insert into t1 values ('cote'),('coté'),('côte'),('côté');
```

Two examples of sorts with two differents locales: French and German

```SQL
select * from t1 order by c1 collate "de_DE";
  c1
------
 cote
 coté
 côte
 côté

select * from t1 order by c1 collate "fr_FR";
  c1
------
 cote
 côte
 coté
 côté
```

Look a sort order, in French language sorting is done on the last accent which is not the case of German.

By default, Postgres uses environment collation. It is possible to specify the collation at the creation of the instance, creation of a base, a table ... Even at column  level.

There is also a particular collation "C" where postgres performs sorting according to the encoding order of the characters.

The `pg_collation` table in the system catalog lists collations. The `collprovider` column indicates the source:

  * `c`: libc
  * `i`: ICU

For example:

```psql
-[ RECORD 1 ]-+-----------------------
collname      | en-US-x-icu
collnamespace | 11
collowner     | 10
collprovider  | i
collencoding  | -1
collcollate   | en-US
collctype     | en-US
collversion   | 153.64
-[ RECORD 4 ]-+-----------------------
collname      | en_US
collnamespace | 11
collowner     | 10
collprovider  | c
collencoding  | 6
collcollate   | en_US.utf8
collctype     | en_US.utf8
collversion   |
```

# ICU collations

Postgres relies on operating system libraries to perform sort operations. Under linux, it is based on the famous libc.

The major benefit of relying on this library is not having to rewrite and maintain a whole set of sorting rules.

However, this approach may have a disadvantage:

We must "trust" this library. Within the same distribution (and even release), the libc will always return the same results. Things can get complicated if you have to compare results from different libs. For example, using a different distribution or a different release. This is why it is not recommended to put servers in streaming replication with different distributions: indexes may not be consistent. Cf: [The dangers of streaming across versions of glibc: A cautionary tale](https://www.postgresql.org/message-id/BA6132ED-1F6B-4A0B-AC22-81278F5AB81E@tripadvisor.com)

ICU collations are richer. It is possible to change sort ordering. For example, uppercase before the lowercase: [Setting Options](http://unicode.org/reports/tr35/tr35-collation.html#Setting_Options). Peter Geoghegan gave some examples on the mailing list postgresql-hackers: [What users can do with custom ICU collations in Postgres 10](https://www.postgresql.org/message-id/flat/CAH2-Wz%3Dbcgmk97YaZ3rs4OoCdn1nOco1HCfRGBCOOty0ztnCnA%40mail.gmail.com#CAH2-Wz=bcgmk97YaZ3rs4OoCdn1nOco1HCfRGBCOOty0ztnCnA@mail.gmail.com)

# History of abbreviated keys in PostgreSQL

Back to the past :) Version 9.5 improved sorting algorithm with _abbreviated keys_. The announced gains were really important, here between 40 and 70%: [Use abbreviated keys for faster sorting of text datums](https://www.depesz.com/2015/01/27/waiting-for-9-5-use-abbreviated-keys-for-faster-sorting-of-text-datums/)

In summary, the _abbreviated keys_ allow to better exploit processor cache. Peter Geoghegan, who is the author of the patch, provides an explanation on his blog: [Abbreviated keys: exploiting locality to improve PostgreSQL's text sort performance ](http://pgeoghegan.blogspot.fr/2015/01/abbreviated-keys-exploiting-locality-to.html)

Unfortunately this feature has been disabled in version 9.5.2: [Disable abbreviated keys for string-sorting in non-C locales](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commitdiff;h=8aa6e97805a79eb30ac9c36acb1126280c2ffbdf)

The reason? A bug in some version of the libc! This bug caused a corruption of the index: [Abbreviated keys glibc issue](https://wiki.postgresql.org/wiki/Abbreviated_keys_glibc_issue)

What is the link with ICU collations? It's quite simple, with a ICU collation postgres no longer relies on the libc. It is therefore possible to use _Abbreviated keys_.

We check it in the code:

```C
src/backend/utils/adt/varlena.c
1876     /*
1877      * Unfortunately, it seems that abbreviation for non-C collations is
1878      * broken on many common platforms; testing of multiple versions of glibc
1879      * reveals that, for many locales, strcoll() and strxfrm() do not return
1880      * consistent results, which is fatal to this optimization.  While no
1881      * other libc other than Cygwin has so far been shown to have a problem,
1882      * we take the conservative course of action for right now and disable
1883      * this categorically.  (Users who are certain this isn't a problem on
1884      * their system can define TRUST_STRXFRM.)
1885      *
1886      * Even apart from the risk of broken locales, it's possible that there
1887      * are platforms where the use of abbreviated keys should be disabled at
1888      * compile time.  Having only 4 byte datums could make worst-case
1889      * performance drastically more likely, for example.  Moreover, macOS's
1890      * strxfrm() implementation is known to not effectively concentrate a
1891      * significant amount of entropy from the original string in earlier
1892      * transformed blobs.  It's possible that other supported platforms are
1893      * similarly encumbered.  So, if we ever get past disabling this
1894      * categorically, we may still want or need to disable it for particular
1895      * platforms.
1896      */
1897 #ifndef TRUST_STRXFRM
1898     if (!collate_c && !(locale && locale->provider == COLLPROVIDER_ICU))
1899         abbreviate = false;
1900 #endif
```

# Some tests

When I prepared my presentation on the [Full Text Search in Postgres](https://blog.anayrat.info/talk/2017-pgdayfr/), I wanted to work on a "real" data set: the stackoverflow database

Here are the results of creating 3 indexes on the "title" column of the posts table (38GB):

   * Ignore collation: `collate  "C"`
   * One using the en_US collation of the libc: `collate  "en_US"`
   * Another one using the en_US collation of ICU library `collate  "en-US-x-icu"`

I activate some options to have the execution time of the query and information about sorting:

```psql
\timing
set client_min_messages TO log;
set trace_sort to on;
```

```psql
create index idx1 on posts (title collate  "C");
LOG:  begin index sort: unique = f, workMem = 6291456, randomAccess = f
LOG:  varstr_abbrev: abbrev_distinct after 160: 56.532166 (key_distinct: 59.707363, norm_abbrev_card: 0.353326, prop_card: 0.200000)
LOG:  varstr_abbrev: abbrev_distinct after 321: 110.782140 (key_distinct: 121.985752, norm_abbrev_card: 0.345116, prop_card: 0.200000)
[...]
LOG:  varstr_abbrev: abbrev_distinct after 10485760: 523091.461475 (key_distinct: 4215367.096361, norm_abbrev_card: 0.049886, prop_card: 0.002693)
LOG:  varstr_abbrev: abbrev_distinct after 20971522: 852125.989455 (key_distinct: 8800364.815018, norm_abbrev_card: 0.040633, prop_card: 0.001750)
LOG:  performsort starting: CPU: user: 21.88 s, system: 17.27 s, elapsed: 104.98 s
LOG:  performsort done: CPU: user: 43.55 s, system: 17.27 s, elapsed: 126.65 s
LOG:  internal sort ended, 3519559 KB used: CPU: user: 48.14 s, system: 18.40 s, elapsed: 142.19 s
CREATE INDEX
Time: 142380.670 ms (02:22.381)
```
Here, postgres exploits _abbreviated keys_ because there are no collation rules. It just has to sort according to the character encoding. Sorting does not take into account collation rules.


```psql
create index idx2 on posts (title collate  "en_US");
LOG:  begin index sort: unique = f, workMem = 6291456, randomAccess = f
LOG:  performsort starting: CPU: user: 20.10 s, system: 17.32 s, elapsed: 104.80 s
LOG:  performsort done: CPU: user: 137.52 s, system: 17.32 s, elapsed: 222.25 s
LOG:  internal sort ended, 3519559 KB used: CPU: user: 142.41 s, system: 18.10 s, elapsed: 237.97 s
CREATE INDEX
Time: 238159.675 ms (03:58.160)
```
Postgres uses libc, so it can not exploit _abbreviated keys_.

```psql
create index idx3 on posts (title collate  "en-US-x-icu");
LOG:  begin index sort: unique = f, workMem = 6291456, randomAccess = f
LOG:  varstr_abbrev: abbrev_distinct after 160: 55.475952 (key_distinct: 59.707363, norm_abbrev_card: 0.346725, prop_card: 0.200000)
LOG:  varstr_abbrev: abbrev_distinct after 321: 110.782140 (key_distinct: 121.985752, norm_abbrev_card: 0.345116, prop_card: 0.200000)
[...]
LOG:  varstr_abbrev: abbrev_distinct after 10485760: 337228.120654 (key_distinct: 4215367.096361, norm_abbrev_card: 0.032161, prop_card: 0.002693)
LOG:  varstr_abbrev: abbrev_distinct after 20971522: 521498.943210 (key_distinct: 8800364.815018, norm_abbrev_card: 0.024867, prop_card: 0.001750)
LOG:  performsort starting: CPU: user: 30.22 s, system: 16.78 s, elapsed: 105.65 s
LOG:  performsort done: CPU: user: 66.86 s, system: 16.78 s, elapsed: 142.31 s
LOG:  internal sort ended, 3519559 KB used: CPU: user: 71.23 s, system: 18.04 s, elapsed: 157.79 s
CREATE INDEX
Time: 157979.957 ms (02:37.980)
```

Postgres uses the ICU library, it can operate _abbreviated keys_. The gain is of the order of **34%**. Unlike the first example, sorting takes into account the en_US collation rules.


Note: If you did not have system collations (during the initdb), you can import the new collations with the `pg_import_system_collations` function. For example:

```SQL
select pg_import_system_collations('pg_catalog');
 pg_import_system_collations
-----------------------------
                           6
```
