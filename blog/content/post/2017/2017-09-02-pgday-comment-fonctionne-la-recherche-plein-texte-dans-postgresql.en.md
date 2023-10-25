---
title: 'PGDay : How does Full Text Search works?'
authors: ['adrien']
type: post
date: 2017-09-02T11:04:58+00:00
categories:
  - Postgres
tags:
  - full text search
  - postgres
show_related: true

---

During last [PGDay][1]  I gave a presentation how Full Text Search works in PostgreSQL.

This feature is unfortunately not well known. I see several reasons for this:

* Complexity: The FTS uses unknown notions from DBA: stemming, vector representation of a document ...
* The tendency to use a dedicated tool for full-text search: ElasticSearch, SOLR ...
* PostgreSQL's advanced features are not known.

However, there are several advantages to use the PostgreSQL FTS:

   * We keep one language: the SQL
   * No duplication of data between the database and the indexing engine
   * It retains consistency:
     * No need to synchronize data in your database with indexing engine
     * A document deleted from the database will not be forgotten in indexing engine
   * We benefit from all advanced features of PostgreSQL, including powerful indexes.
   * The engine is expandable, we can customize the FTS configuration

When I started diving in the FTS I was soon put off by terms that were unknown to me, special syntax with new operators. Instead of making a presentation on the wide possibilities of FTS, I focused on presenting how it work from the base. That is, all the mechanics behind.

Here is the video of the talk:

{{< youtube 9S5dBqMbw8A >}}

Here the [slides](https://blog.anayrat.info/presentations/2017/nayrat_Le_Full_Text_Search_dans_PostgreSQL.pdf).

[1]: http://pgday.fr/programme.html
