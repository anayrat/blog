---
title: 'PostgreSQL 10 : Performances improvements'
author: Adrien Nayrat
type: post
date: 2017-10-04T09:51:44+00:00
categories:
  - Linux
  - Postgres
tags:
  - postgres

---

PostgreSQL 10 is coming soon, it is scheduled for tomorrow : [See this commit](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commitdiff;h=086fda9073d37b519519926136c9fe5418451c0e)


This release includes expected features :

  * Logical replication
  * Native partitioning
  * Better parallelism support
  * Multi-column statistics
  * ...

  For an exhaustive list see:

  * [Releases notes](https://www.postgresql.org/docs/devel/static/release-10.html)
  * [Wiki page about v10][1]

In this article I will expose you performance improvements that are not listed in the releases notes! Surprisingly, the community does not list these kinds of improvements: they do not represent a significant change from user experience.

<!--more-->

Bruce Momjian kindly reminded me when I asked if it was possible to indicate these improvements: <https://www.postgresql.org/message-id/flat/f6949bce-068f-5de5-fd62-5c1e45a611fa%40dalibo.com#f6949bce-068f-5de5-fd62-5c1e45a611fa@dalibo.com>

## Rewrite part of the executor

Andres Freund has rewritten part of the executor:

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
As you can read, this commit provides a gain on performance but will facilitate the implementation of the "JIT" for [Just-In-Time compilation][2]) in future versions.

The thread mentions significant gains: <https://www.postgresql.org/message-id/20170314065259.ffef4tfhgbsaieoe%40alap3.anarazel.de>

## Improvements on external sorting

Following commits improve significantly external sorts:

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

For simplicity, the algorithms were revised and allow better use of caches. Here again, the gains are very significant, we found twice as fast sorting on some platforms (the result may depend on many parameters, including the size of processor's cache etc).

So sorting that can not be kept in memory and must be done on disk should be much faster. For example, when creating an index on a large table 🙂

## Improvements on hash functions

Andreus Freund (again!) has reviewed hash functions and again, gains are impressive.

Peter Geoghegan mentions it in this conference:

{{< youtube aic_9KNwKn0 >}}


<https://speakerdeck.com/peterg/sort-hash-pgconfus-2017>


(See slide 52)

Andres Freund has set up an infrastructure:

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

Yes you read "improvements of over 100%" :)

Then he used this infrastructure to improve the bitmap scan and hash join:

```
commit 75ae538bc3168bf44475240d4e0487ee2f3bb376
Author: Andres Freund <andres@anarazel.de>
Date:   Fri Oct 14 16:05:30 2016 -0700

    Use more efficient hashtable for tidbitmap.c to speed up bitmap scans.

    Use the new simplehash.h to speed up tidbitmap.c uses. For bitmap scan
    heavy queries speedups of over 100% have been measured. Both lossy and
    exact scans benefit, but the wins are bigger for mostly exact scans.
```

Another 100% gain :)

Finally, this same infrastructure was used to improve the performance on the GROUPING SETS (that is mentioned in the releases notes):

```
commit 5dfc198146b49ce7ecc8a1fc9d5e171fb75f6ba5
Author: Andres Freund <andres@anarazel.de>
Date:   Fri Oct 14 17:22:51 2016 -0700

    Use more efficient hashtable for execGrouping.c to speed up hash aggregation.

    The more efficient hashtable speeds up hash-aggregations with more than
    a few hundred groups significantly. Improvements of over 120% have been
    measured.
```

Here the commit mentions a gain of 120% :)

Overall, it must be said that each new version brings performance gains even if they are not mentioned in the Releases Notes. So, I strongly encourage you to do your own tests to compare performance.

PostgreSQL 10 is a real turning point, PostgreSQL is ready cut to tackle big volumetry (and it's not over :)).

 [1]: https://wiki.postgresql.org/wiki/New_in_postgres_10
 [2]: https://en.wikipedia.org/wiki/Just-in-time_compilation
