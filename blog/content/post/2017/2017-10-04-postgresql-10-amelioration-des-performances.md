---
title: 'PostgreSQL 10 : Amélioration des performances'
authors: ['adrien']
type: post
date: 2017-10-04T09:51:44+00:00
aliases: /2017/10/04/postgresql-10-amelioration-des-performances/
categories:
  - Linux
  - Postgres
tags:
  - postgres
show_related: true

---

La sortie de la version 10 approche à grands pas, elle est prévue pour demain : [Voir ce commit](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commitdiff;h=086fda9073d37b519519926136c9fe5418451c0e)

Cette nouvelle version inclue des fonctionnalités très attendues comme :

  * La réplication logique
  * Le partitionnement natif
  * Amélioration du parallélisme
  * Statistiques multi-colonnes
  * ...

Pour une liste exhaustive voir :

  * [Les releases notes](https://www.postgresql.org/docs/devel/static/release-10.html)
  * [La page wiki sur la version 10][1]

Dans cet article je vais vous présenter des améliorations sur les performances qui ne sont pas listées dans les releases notes! Aussi surprenant que cela puisse paraître, la communauté ne liste pas ce genre d'améliorations : elles ne représentent pas un changement significatif du point de vue de l'utilisateur.

<!--more-->

Bruce Momjian me l'a gentiment rappelé quand j'ai demandé s'il était possible d'indiquer ces améliorations : <https://www.postgresql.org/message-id/flat/f6949bce-068f-5de5-fd62-5c1e45a611fa%40dalibo.com#f6949bce-068f-5de5-fd62-5c1e45a611fa@dalibo.com>

## Réécriture d'une partie de l'exécuteur

Andres Freund a réécris une partie de l'exécuteur :

```
commit b8d7f053c5c2bf2a7e8734fe3327f6a8bc711755
Author: Andres Freund <andres@anarazel.de>
AuthorDate: Tue Mar 14 15:45:36 2017 -0700
Commit: Andres Freund <andres@anarazel.de>
CommitDate: Sat Mar 25 14:52:06 2017 -0700

Faster expression evaluation and targetlist projection.

This replaces the old, recursive tree-walk based evaluation, with
 non-recursive, opcode dispatch based, expression evaluation.
 Projection is now implemented as part of expression evaluation.

This both leads to significant performance improvements, and makes
 future just-in-time compilation of expressions easier.

The speed gains primarily come from:
 - non-recursive implementation reduces stack usage / overhead
 - simple sub-expressions are implemented with a single jump, without
 function calls
 - sharing some state between different sub-expressions
 - reduced amount of indirect/hard to predict memory accesses by laying
 out operation metadata sequentially; including the avoidance of
 nearly all of the previously used linked lists
 - more code has been moved to expression initialization, avoiding
 constant re-checks at evaluation time
...
```

Comme vous pouvez le lire, ce commit apporte un gain sur les performances mais facilitera la mise en œuvre du « JIT » pour « Just In Time » : [Compilation à la volée][2]) dans les futures versions.

Le thread mentionne des gains significatifs : <https://www.postgresql.org/message-id/20170314065259.ffef4tfhgbsaieoe%40alap3.anarazel.de>

## Améliorations sur les tri externes

Les commits suivants améliorent très significativement les tris externes :

```
commit e94568ecc10f2638e542ae34f2990b821bbf90ac
Author: Heikki Linnakangas <heikki.linnakangas@iki.fi>
Date: Mon Oct 3 13:37:49 2016 +0300

Change the way pre-reading in external sort's merge phase works.

 Don't pre-read tuples into SortTuple slots during merge. Instead, use the
 memory for larger read buffers in logtape.c. We're doing the same number
 of READTUP() calls either way, but managing the pre-read SortTuple slots
 is much more complicated. Also, the on-tape representation is more compact
 than SortTuples, so we can fit more pre-read tuples into the same amount
 of memory this way. And we have better cache-locality, when we use just a
 small number of SortTuple slots.
...
```

```
commit 24598337c8d214ba8dcf354130b72c49636bba69
Author: Heikki Linnakangas <heikki.linnakangas@iki.fi>
Date: Sun Sep 11 16:27:27 2016 +0300

Implement binary heap replace-top operation in a smarter way.

 In external sort's merge phase, we maintain a binary heap holding the next
 tuple from each input tape. On each step, the topmost tuple is returned,
 and replaced with the next tuple from the same tape. We were doing the
 replacement by deleting the top node in one operation, and inserting the
 next tuple after that. However, you can do a "replace-top" operation more
 efficiently, in one "sift-up". A deletion will always walk the heap from
 top to bottom, but in a replacement, we can stop as soon as we find the
 right place for the new tuple. This is particularly helpful, if the tapes
 are not in completely random order, so that the next tuple from a tape is
 likely to land near the top of the heap.
```

Pour faire simple, les algorithmes ont été revus et permettent de mieux exploiter les caches. Là aussi les gains sont très significatifs, on a constaté des tris deux fois plus rapides sur certaines plateformes (le résultat peut dépendre de beaucoup de paramètres, notamment la taille du cache du processeur etc).

Ainsi les tris qui ne peuvent pas tenir en mémoire et qui doivent donc être  effectués sur disque devraient être beaucoup plus rapides. Par exemple, lors de la création d'un index sur une table volumineuse 🙂

## Améliorations sur les fonctions de hashages

Andreus Freund (encore!) a revu les fonctions de hashage et à nouveau les gains sont impressionnants.

Peter Geoghegan en fait mention dans cette conférence :

{{< youtube aic_9KNwKn0 >}}


<https://speakerdeck.com/peterg/sort-hash-pgconfus-2017>

(Voir le slide 52)

Andres Freund a mis en place une infrastructure :

```
commit b30d3ea824c5ccba43d3e942704f20686e7dbab8
Author: Andres Freund ><andres@anarazel.de>
Date:   Fri Oct 14 16:05:30 2016 -0700

    Add a macro templatized hashtable.

    dynahash.c hash tables aren't quite fast enough for some
    use-cases. There are several reasons for lacking performance:
    - the use of chaining for collision handling makes them cache
      inefficient, that's especially an issue when the tables get bigger.
    - as the element sizes for dynahash are only determined at runtime,
      offset computations are somewhat expensive
    - hash and element comparisons are indirect function calls, causing
      unnecessary pipeline stalls
    - it's two level structure has some benefits (somewhat natural
      partitioning), but increases the number of indirections
    to fix several of these the hash tables have to be adjusted to the
    individual use-case at compile-time. C unfortunately doesn't provide a
    good way to do compile code generation (like e.g. c++'s templates for
    all their weaknesses do).  Thus the somewhat ugly approach taken here is
    to allow for code generation using a macro-templatized header file,
    which generates functions and types based on a prefix and other
    parameters.

    Later patches use this infrastructure to use such hash tables for
    tidbitmap.c (bitmap scans) and execGrouping.c (hash aggregation,
    ...). In queries where these use up a large fraction of the time, this
    has been measured to lead to performance improvements of over 100%.
...
```

Oui vous lisez bien « improvements of over 100% » :)

Ensuite il a utilisé cette infrastructure pour améliorer les bitmap scan et les jointures par hash join :

```
commit 75ae538bc3168bf44475240d4e0487ee2f3bb376
Author: Andres Freund <andres@anarazel.de>
Date:   Fri Oct 14 16:05:30 2016 -0700

    Use more efficient hashtable for tidbitmap.c to speed up bitmap scans.

    Use the new simplehash.h to speed up tidbitmap.c uses. For bitmap scan
    heavy queries speedups of over 100% have been measured. Both lossy and
    exact scans benefit, but the wins are bigger for mostly exact scans.
```

Encore un gain de 100% :)

Enfin, cette même infrastructure a été utilisée pour améliorer les performances sur les GROUPING SETS (ça, c'est bien mentionné dans les releases notes):

```
commit 5dfc198146b49ce7ecc8a1fc9d5e171fb75f6ba5
Author: Andres Freund <andres@anarazel.de>
Date:   Fri Oct 14 17:22:51 2016 -0700

    Use more efficient hashtable for execGrouping.c to speed up hash aggregation.

    The more efficient hashtable speeds up hash-aggregations with more than
    a few hundred groups significantly. Improvements of over 120% have been
    measured.
```

Ici le commit mentionne un gain de 120% :)

Globalement, il faut se dire que chaque nouvelle version apporte des gains en performance même s'ils ne sont pas mentionnés dans les Releases Notes. Ainsi, je vous encourage vivement à faire vos propres tests pour comparer les performances.

La version 10 marque un vrai tournant, PostgreSQL devient vraiment taillé pour s'attaquer aux grosses volumétries (et ce n'est pas fini :) ).

 [1]: https://wiki.postgresql.org/wiki/New_in_postgres_10
 [2]: https://fr.wikipedia.org/wiki/Compilation_%C3%A0_la_vol%C3%A9e
