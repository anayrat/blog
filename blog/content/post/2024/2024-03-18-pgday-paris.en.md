---
# Documentation: https://docs.hugoblox.com/managing-content/

title: "Back from pgDay Paris 2024."
subtitle: "A beautiful edition with high-quality content."
summary: "A beautiful edition with high-quality content."
authors: []
tags: ['PostgreSQL','pgDay Paris']
categories: ['Postgres']
date: 2024-03-18T10:39:28+01:00
lastmod: 2024-03-18T10:39:28+01:00
featured: false
draft: false

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image:
  caption: ""
  focal_point: ""
  preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---

I'm back from [pgDay Paris](https://2024.pgday.paris/). I really enjoyed this edition.
I'd already come to the [2019](https://www.postgresql.eu/events/pgdayparis2019/schedule/) one, and I must say I wasn't deceived.
As a reminder, pgDay Paris is an international conference. Presentations are in English, attracting more English speakers and audiences.

I often see this conference as a little [PGConf Europe](https://www.postgresql.eu/events/series/pgconfeu-1/): the content is dense with an international dimension.

You meet familiar faces: attendees, speakers, volunteers, contributors: all the people who keep the Postgres community going.

It's also an opportunity to put faces on names you've come across while reading articles or Postgres mailing lists.

Here's a quick recap of the conferences I attended:

# Elephant in a nutshell - Navigating the Postgres community 101

With [Carole and Stéphanie](https://2024.pgday.paris/organization/), we thought it would be good to have this conference as an introduction.

I found it very complete. It talks about Postgres, but also and above all about its community.
That's what makes Postgres so powerful.

[Slides are availables](https://www.postgresql.eu/events/pgdayparis2024/sessions/session/5293/slides/481/pgDay%20Paris_Valeria's%20talk%20-%20Elephant%20in%20a%20nutshell.pdf).

# Sustainable Database Performance profiling in PostgreSQL

Postgres provides a variety of statistical views to give you information on its activity.

These can be native (*pg_stat_statements* ...) or via extensions *pg_stat_kcache*, *pg_wait_sampling*.

However, and this is what this conference emphasizes, they provide a view at a given moment in time.

To use them, they need to be historized. The speaker presents a tool based on what is done on Oracle: [pg_profile](https://github.com/zubkov-andrei/pg_profile).

On the other hand, he presents it as the only tool that can be used to process such information. A participant in the audience
pointed out that there was another project: [PoWA](https://powa.readthedocs.io/en/latest/).

The two tools don't provide the same functions, *pg_profile* generates an HTML report. It's fairly easy to install, and requires no external libraries, as it's written in pl/pgsql.

PoWA provides graphs, index suggestion... but is heavier to install.

[Slides are available](https://www.postgresql.eu/events/pgdayparis2024/sessions/session/5067/slides/482/20240314_dkrautschick_PGdayParis_SustainableDatabasePerformanceProfilingInPostgreSQL.pdf).

# PostgreSQL without permanent local data storage

Now we're getting into a very technical conference. Matthias is working on the [Neon](https://neon.tech/) project and is a regular contributor to the [Postgres mailing-list](https://www.postgresql.org/list/).

Neon is an opensource fork of Postgres designed to separate *compute* and *storage*. This is what AWS has done with the Aurora project.

Heikki Linnakangas, one of Neon's co-founders and Postgres' committer, gave several presentations:

* [Neon: Serverless PostgreSQL! (Heikki Linnakangas)](https://www.youtube.com/watch?v=rES0yzeERns)
* [Architecture decisions in Neon](https://neon.tech/blog/architecture-decisions-in-neon)

Matthias presented the various problems involved in separating storage from the rest of the core.
It's clearly a very ambitious project. For the moment, it's hard to say whether this fork will survive or not.

However, it's worth pointing out that the people working on it have a very extensive knowledge of Postgres internals.
Hopefully, this will have positive implications for the Postgres' core.

Stay tuned...

# Lightning Talks!

Lightning Talks are a good opportunity to discuss some lighter subjects. It fits in quite well after lunch.
It's quite entertaining. I liked Léo Unbekandt's presentation on the architecture of Postgres instances in Scalingo.
No K8S, simple and proven architecture.

I also liked Chris Ellis' presentation where he makes a decorative light object with ESP32 inside.

# Multi-tenant database: the good, the bad, the ugly.

After the Lightning Talks, we return to the core of the subject with this talk by Pierre Ducroquet.

I know Pierre well, the first time I came across him was at pgDay France 2016. He gave a very good presentation on [Understanding why a request is slow, and how to fix the problem (French)](https://2016.pgday.fr/programme.html#comprendre-requete-lente).
Then, he was my colleague for a few years, where we encountered the issues of *multi-tenant* databases.

He presents different ways of doing *multi-tenant* with the constraints this implies. He gives some pretty impressive numbers, such as more than 200,000 tables!

[Something tells me it won't be long before we see him again](https://www.pinaraf.info/2024/03/look-ma-i-wrote-a-new-jit-compiler-for-postgresql/)

# Beyond B-trees looking at Columnar Storage and LSM trees

Another low-level conference on columnar storage and LSM Tree.

This type of storage may well be part of Postgres' future. Postgres is starting to be used for more and more analytical projects involving large volumes of data.

This requires storage methods capable of handling fast writes.

# Postgres 16 highlight: Logical decoding on standby

A long-awaited feature in Postgres 16: the ability to perform logical decoding on a secondary.

Who better than one of the authors of this feature to talk about it?

Bertrand presents the difficulties involved in performing logical decoding on a secondary, and how they were resolved.

He also showed a demonstration. It's really impressive, as it's quite a complicated subject.

As a bonus, he also explained a new feature of Postres 17: the synchronization of replication slots.

Currently, it's difficult to "hang up" a logical replication in the case of a switchover. There is a risk of transaction loss.

Postgres 17 should make it possible to synchronize a replication slot. In case of a failover, replication can be resumed.
We're lucky, he's just written an article on this: [Postgres 17 highlight: Logical replication slots synchronization](https://bdrouvot.github.io/2024/03/16/postgres-17-highlight-logical-replication-slots-synchronization/).

# Closing

It was the last conference of the day. We all met up again for the social event.

It's a moment I'm particularly attached to.
Because beyond the conferences, it's above all an opportunity to exchange ideas with other passionate people. To catch up with old friends and acquaintances.

I've realized that I've been neglecting conferences a bit, between covid, my move to freelance work and so on...
In the future, I'll try to be more present at this kind of event (if I can take the train).

# Thank you

I'd also like to take this opportunity to thank the [organizers and volunteers](https://2024.pgday.paris/organization/). It's hard to realize how much it takes.
A big thank you to them for this event!

See you at pgDay 2025!









