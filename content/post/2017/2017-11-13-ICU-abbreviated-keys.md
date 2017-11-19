+++
title = "PostgreSQL 10 : ICU & Abbreviated Keys"
date = 2017-11-19T15:30:00+01:00
draft = false

# Tags and categories
# For example, use `tags = []` for no tags, or the form `tags = ["A Tag", "Another Tag"]` for one or more tags.
tags = ["postgres","tris","collation","ICU"]
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


A peu près tout le monde a entendu parler du partitionnement et de la réplication logique dans PostgreSQL 10. Avez-vous entendu parler du support des règles de collation ICU (_International Components for Unicode_)?

Cet article va présenter en quoi consiste cette nouvelle fonctionnalité mais également les gains possibles en exploitant les _abbreviated keys_.

<!--more-->

{{% toc %}}

# Court rappel sur le collationnement

En base de données, le collationnement correspond à des règles de classement de caractères.

Voici quelques exemples :

```SQL
create table t1 (c1 text);
insert into t1 values ('cote'),('coté'),('côte'),('côté');
```

Deux exemples de tris en fonctions de deux locales : française et allemande

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

Observez bien l'ordre de tri, en langue française le tri se fait sur le dernier accent ce qui n'est pas le cas de l'allemand.

Par défaut, Postgres utilise le collationnement de l'environnement. Il est possible de spécifier le collationnement à la création de l'instance, d'une base, d'une table... Même au niveau d'une colonne.

Il existe aussi un collationnement particulier "C" où postgres effectue le tri selon l'ordre de codage des caractères.

La table `pg_collation` du catalogue système énumère les différentes collations. La colonne `collprovider` indique la source :

  * `c`: libc
  * `i`: ICU

Par exemple :

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

# Collations ICU

Postgres s'appuie sur les librairies du système d'exploitation pour effectuer les opérations de tri. Sous linux, il s'appuie sur la très connue glibc.

L'intérêt majeur de s'appuyer sur cette librairie est de ne pas avoir à réécrire et maintenir tout un ensemble de règles de tri.

Cependant, cette approche peut présenter un inconvénient:

On doit faire "confiance" à cette librairie. Au sein d'une même distribution (et même release), la libc retournera toujours les mêmes résultats. Les choses peuvent se corser si on doit comparer les résultats obtenus depuis des libcs différentes. Par exemple, en utilisant une distribution différente ou une release différente. C'est pourquoi il est déconseillé de mettre des serveurs en streaming replication avec des distributions différentes : les index pourraient ne pas être cohérents. Cf : [The dangers of streaming across versions of glibc: A cautionary tale](https://www.postgresql.org/message-id/BA6132ED-1F6B-4A0B-AC22-81278F5AB81E@tripadvisor.com)

Les collations ICU sont plus riches. Il est possible de changer l'ordre des tris. Par exemple les mascules avant les minuscules: [Setting Options](http://unicode.org/reports/tr35/tr35-collation.html#Setting_Options). Peter Geoghegan a donné quelques exemples sur la mailing list hackers : [What users can do with custom ICU collations in Postgres 10](https://www.postgresql.org/message-id/flat/CAH2-Wz%3Dbcgmk97YaZ3rs4OoCdn1nOco1HCfRGBCOOty0ztnCnA%40mail.gmail.com#CAH2-Wz=bcgmk97YaZ3rs4OoCdn1nOco1HCfRGBCOOty0ztnCnA@mail.gmail.com)

# Histoire des abbreviated keys dans PostgreSQL

Petit tour dans le passé :) La version 9.5 améliorait l'algorithme de tri grâce aux _abbreviated keys_. Les gains annoncés étaient vraiment important, ici entre 40 et 70% : [Use abbreviated keys for faster sorting of text datums](https://www.depesz.com/2015/01/27/waiting-for-9-5-use-abbreviated-keys-for-faster-sorting-of-text-datums/)

En gros, les _abbreviated keys_ permettent de mieux exploiter le cache des processeurs. Peter Geoghegan, qui est l'auteur du patch, apporte une explication sur son blog : [Abbreviated keys: exploiting locality to improve PostgreSQL's text sort performance ](http://pgeoghegan.blogspot.fr/2015/01/abbreviated-keys-exploiting-locality-to.html)

Malheureusement cette fonctionnalité a dû être désactivée dans la version 9.5.2: [Disable abbreviated keys for string-sorting in non-C locales](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commitdiff;h=8aa6e97805a79eb30ac9c36acb1126280c2ffbdf)

La raison? Un bug dans certaines version de la libc! Ce bug entrainait une corruption de l'index: [Abbreviated keys glibc issue](https://wiki.postgresql.org/wiki/Abbreviated_keys_glibc_issue)

Quel est le lien avec les collations ICU? C'est tout simple, avec une collation ICU le moteur ne s'appuie plus sur la libc. Il est donc possible d'utiliser les _Abbreviated keys_.

On peut aller le vérifier dans le code :

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

# Quelques tests

Quand j'ai préparé ma présentation sur le [fonctionnement du Full Text Search dans Postgres](https://blog.anayrat.info/talk/2017-pgdayfr/), j'ai souhaité travailler sur un jeu de donnée "réel": la base de donnée de stackoverflow.

Voici les résultats de la création de 3 index sur la colonne "title" de la table posts (38Go):

  * Un ignorant la collation: `collate  "C"`
  * Un utilisant la collation en_US de la libc: `collate  "en_US"`
  * Un utilisant la collation en_US de la librairie ICU `collate  "en-US-x-icu"`

J'active quelques options pour avoir la durée d'exécution de la requête et des informations sur le tri:

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

Ici, postgres exploite les _abbreviated keys_ car il n'y a pas de règles de collation. Il a juste à trier selon le codage des caractères. Le tri ne tient donc pas compte des règles de collationnement.


```psql
create index idx2 on posts (title collate  "en_US");
LOG:  begin index sort: unique = f, workMem = 6291456, randomAccess = f
LOG:  performsort starting: CPU: user: 20.10 s, system: 17.32 s, elapsed: 104.80 s
LOG:  performsort done: CPU: user: 137.52 s, system: 17.32 s, elapsed: 222.25 s
LOG:  internal sort ended, 3519559 KB used: CPU: user: 142.41 s, system: 18.10 s, elapsed: 237.97 s
CREATE INDEX
Time: 238159.675 ms (03:58.160)
```

Postgres utilise la libc, il ne peut donc pas exploiter les _abbreviated keys_.

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
Postgres utilise la librairie ICU, il peut exploiter les _abbreviated keys_. Le gain est de l'ordre de **34%**. Contrairement au premier exemple, le tri tient compte des règles de collationnement en_US.


Note : Si comme moi il vous manquait des collations système (lors de l'initdb), il est possible d'importer les nouvelles collations grâce à la fonction `pg_import_system_collations`. Par exemple :

```SQL
select pg_import_system_collations('pg_catalog');
 pg_import_system_collations
-----------------------------
                           6
```
