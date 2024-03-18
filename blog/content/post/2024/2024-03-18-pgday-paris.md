---
# Documentation: https://docs.hugoblox.com/managing-content/

title: "De retour du pgDay Paris 2024."
subtitle: "Une belle édition avec un contenu très riche."
summary: "Une belle édition avec un contenu très riche."
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


Me voilà rentré du pgDay Paris. J’ai beaucoup aimé cette édition. J’étais déjà venu à celle de [2019](https://www.postgresql.eu/events/pgdayparis2019/schedule/) et je dois dire que je n’ai pas été déçu. Pour rappel, le pgDay Paris est une conférence internationale. Les présentations sont en anglais et permettent d’attirer des orateurs et un public plus anglophone.

Je vois souvent cette conférence comme un petit [PGConf Europe](https://www.postgresql.eu/events/series/pgconfeu-1/) : on retrouve un contenu assez riche avec la dimension internationale.

On croise des têtes déjà connues : public, orateurs, bénévoles, contributeurs : Tout ce petit monde qui fait vivre la communauté Postgres.

Puis, c’est l’occasion de mettre des visages sur des noms que l’on croise en lisant des articles ou les mailing-list Postgres.

Voici un petit retour sur les conférences auxquelles j’ai assisté :

# Elephant in a nutshell - Navigating the Postgres community 101


Avec [Carole et Stéphanie](https://2024.pgday.paris/organization/), on a pensé que ce serait bien d’avoir cette conférence en introduction.

Je l’ai trouvé très complète. Elle parle de Postgres, mais aussi et surtout de sa communauté. C’est ce qui fait sa force.

[Les slides sont disponibles.](https://www.postgresql.eu/events/pgdayparis2024/sessions/session/5293/slides/481/pgDay%20Paris_Valeria's%20talk%20-%20Elephant%20in%20a%20nutshell.pdf)

# Sustainable Database Performance profiling in PostgreSQL

Postgres dispose de nombreuses vues statistiques apportant des informations sur son fonctionnement.

Elles peuvent être natives (*pg_stat_statements* …) ou via des extensions *pg_stat_kcache*, *pg_wait_sampling*.

Cependant, et c’est ce que souligne cette conférence, elles apportent une vue à un instant T.

Pour les exploiter, il faut les historiser. L’orateur présente un outil basé sur ce qui se fait sur Oracle : [pg_profile](https://github.com/zubkov-andrei/pg_profile).

En revanche, il le présente comme l’unique outil permettant d’exploiter ces informations. Une personne dans le public lui a fait remarquer qu’il existait un autre projet : [PoWA](https://powa.readthedocs.io/en/latest/).

Les deux outils ne fournissent pas les mêmes fonctions, *pg_profile* génère un rapport HTML. Il est plutôt facile à installer, il ne nécessite pas de librairies externe car il est écrit en pl/pgsql.

PoWA apporte des graphes, de la suggestion d’index… mais est plus lourd à installer.

[Les slides sont disponibles](https://www.postgresql.eu/events/pgdayparis2024/sessions/session/5067/slides/482/20240314_dkrautschick_PGdayParis_SustainableDatabasePerformanceProfilingInPostgreSQL.pdf).

# PostgreSQL without permanent local data storage

Là, on rentre dans une conférence très technique. Matthias travaille sur le projet [Neon](https://neon.tech/) et il participe régulièrement sur les [mailing-list Postgres](https://www.postgresql.org/list/).

Neon est un fork opensource de Postgres visant à séparer le *compute* et le *storage*. C’est ce qu’a fait AWS avec le projet Aurora.

Heikki Linnakangas, un des co-fondateurs de Neon et committer de Postgres a fait plusieurs présentations :

* [Neon: Serverless PostgreSQL! (Heikki Linnakangas)](https://www.youtube.com/watch?v=rES0yzeERns)
* [Architecture decisions in Neon](https://neon.tech/blog/architecture-decisions-in-neon)

Matthias a présenté les différents problèmes que cela posait de vouloir séparer le stockage du reste du moteur. C’est clairement un projet très ambitieux. Pour le moment, il est difficile de savoir si ce fork perdurera ou non.

Cependant, on peut souligner que les personnes travaillant dessus ont une très grande connaissance du moteur. On peut espérer que ça ait des implications positives dans le projet Postgres.

Affaire à suivre…

# Lightning Talks!

Les Lightning Talks sont une bonne occasion de balayer certains sujets un peu plus légers. Ca passe plutôt bien après le repas. C’est assez divertissant. J’ai bien aimé la présentation de Léo Unbekandt sur l’architecture des instances Postgres à Scalingo. Pas de K8S, une architecture simple et éprouvée.

J’ai aussi aimé la présentation de Chris Ellis où il fabrique un objet décoratif lumineux avec de l’ESP32 dedans.


# Multi-tenant database: the good, the bad, the ugly.

Après, les Lightning Talks, on retourne dans le vif du sujet avec cette présentation de Pierre Ducroquet.

Je connais bien Pierre, la première fois que je l’ai croisé, c’était au pgDay France 2016. Il avait fait une très bonne présentation sur [Comprendre pourquoi une requête est lente, et comment régler le soucis](https://2016.pgday.fr/programme.html#comprendre-requete-lente). Ensuite, ça a été mon collègue de travail pendant quelques années où on a rencontré les problèmes que pouvaient poser les bases de données *multi-tenant*.

Il présente différentes façons de faire du *multi-tenant* avec les contraintes que cela implique. Il donne des chiffres assez impressionnants comme plus de 200 000 tables!

[Quelque chose me dit qu’on ne va pas tarder à le revoir](https://www.pinaraf.info/2024/03/look-ma-i-wrote-a-new-jit-compiler-for-postgresql/).

# Beyond B-trees looking at Columnar Storage and LSM trees


Une autre conférence assez bas niveau sur le stockage colonne et les LSM Tree.

Ce type de stockage peut effectivement faire partie du futur de Postgres. Le moteur commence à être utilisé pour de plus en plus de projets analytiques où on manipule de gros volumes de données.

Il faut donc des méthodes de stockage capable d’encaisse des écritures rapides.


# Postgres 16 highlight: Logical decoding on standby


Une nouveauté assez attendue sur Postgres 16 : La possibilité de faire du décodage logique sur un secondaire.

Qui mieux qu’un des auteurs de cette fonctionnalité pour en parler ?

Bertrand présente les difficultés pour faire un décodage logique sur un secondaire, et comment elles ont été résolues.

Il a aussi fait une démonstration. C’est vraiment impressionnant, car c’est un sujet assez compliqué.

Point bonus, il explique aussi une nouveauté de Postres 17 : la synchronisation des slots de réplication.

Actuellement, on peut difficilement “raccrocher” une réplication logique en cas de bascule. Il y a un risque de perte de transaction.

Postgres 17 devrait permettre de synchroniser un slot de réplication. En cas de bascule, on peut reprendre la réplication.
On est chanceux, il vient tout juste d’écrire un article sur le sujet : [Postgres 17 highlight: Logical replication slots synchronization](https://bdrouvot.github.io/2024/03/16/postgres-17-highlight-logical-replication-slots-synchronization/).

# Closing

C’était la dernière conférence de la journée. On a pu se retrouver pour le social event.

C’est un moment que j’affectionne particulièrement. Car au-delà des conférences, c’est surtout une occasion d’échanger entre passionnés. De retrouver des connaissances et amis de longue date.

J’ai réalisé que j’avais un peu délaissé les conférences, entre le covid, mon passage en freelance etc… A l’avenir, j’essaierai d’être plus présent sur ce genre d’évènement (si je peux prendre le train).

# Merci

J’en profite aussi pour remercier les [organisateurs et bénévoles](https://2024.pgday.paris/organization/). On a du mal à réaliser l’investissement que ça demande. Un grand merci à eux pour cet évènement !

On se verra sûrement au pgDay 2025 !




