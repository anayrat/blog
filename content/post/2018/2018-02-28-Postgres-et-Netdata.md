+++
title = "Postgres et Netdata : application à autovacuum et pg_stat_bgwriter"
date = 2018-02-20T18:35:22+01:00
draft = false
summary = "Présentation des graphiques autovacuum et bgwriter dédiés à PostgreSQL dans Netdata"

# Tags and categories
# For example, use `tags = []` for no tags, or the form `tags = ["A Tag", "Another Tag"]` for one or more tags.
tags = ["postgres","netdata","supervision","métrologie"]
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

Il y a quelque temps, j'ai découvert [Netdata](https://my-netdata.io/).
Cet outil permet de collecter de nombreuses métriques (cpu, réseau etc).

La force de Netdata est d'être assez léger et simple d'utilisation. Il effectue
une mesure toutes les secondes, tout est stocké en mémoire. Il n'y a pas vraiment
d'historique. Le but est d'avoir accès à un ensemble de métrique en temps réel. [^1]

J'ai rajouté plusieurs graphiques pour PostgreSQL, en m'inspirant largement de
ce qui existe dans [check_pgactivity](https://github.com/OPMDG/check_pgactivity).

Voici la présentation et les explications de quelques graphes. Certains,
permettent de mettre en évidence le comportement de PostgreSQL.

Pour information, les graphes présentés ici ne correspondent pas à la version finale.
Plusieurs graphes ont été séparés afin de distinguer les différentes opérations
(lecture, écrites) et les processus (backend, checkpointer, bgwriter). [^2]

Je remercie au passage [Guillaume Lelarge](https://twitter.com/g_lelarge) qui a
joué le rôle de relecteur pour cet article.

# Autovacuum

{{< figure src="/img/2018/netdata-postgres02.png" title="Autovacuum workers" >}}

Ce graphe présente l'activité des processus lancés par "l'autovacuum launcher".
Leur rôle est d'effectuer des tâches de maintenance afin de :

  * Nettoyer les lignes mortes
  * Mettre à jour les statistiques des données (voir [Rappels statistiques, cardinalité, sélectivité](https://blog.anayrat.info/2017/11/26/postgresql---jsonb-et-statistiques/#rappels-statistiques-cardinalit%C3%A9-s%C3%A9lectivit%C3%A9))
  * "Geler" les lignes anciennes afin d'éviter un wraparound (Voir [Transaction Id Wraparound in Postgres](http://malisper.me/transaction-id-wraparound-in-postgres/))

Normalement, il n'y a pas à s'inquiéter, la configuration par défaut suffit dans
la plupart des situations. Cependant, il peut arriver qu'il soit nécessaire
d'affiner la configuration de l'autovacuum.

Par exemple, si l'activité est très soutenue et que le nombre de processus lancés
atteint régulièrement l'`autovacuum_max_workers`. Dans ce cas, il peut être nécessaire :

  * soit d'augmenter le nombre de processus (s'il y a beaucoup de tables à traiter)
  * soit de rendre les processus plus aggressifs (lorsque les tables sont volumineuses).
  En effet, leur activité est bridée via les paramètres `autovacuum_vacuum_cost_delay`
  et `autovacuum_vacuum_cost_limit` afin que cette tâche se fasse en arrière plan
  pour impacter le moins possible les autres processus en charge de traiter les requêtes.

# Bgwriter

{{< figure src="/img/2018/netdata-postgres01.png" title="Bgwriter" >}}

Ce graphe s'appelle Bgwriter en référence à la vue qui permet d'obtenir les statistiques [pg_stat_bgwriter](https://www.postgresql.org/docs/current/static/monitoring-stats.html#PG-STAT-BGWRITER-VIEW)

Le rôle du "bgwriter" est de synchroniser des blocs "sales". Ce sont des blocs qui
ont été modifiés dans la mémoire partagée mais pas encore écrits dans les fichiers
de données (rassurez-vous ils sont bien écrits sur disque dans les journaux de transaction).

Le "checkpointer" est également en charge de synchroniser ces blocs. Les écritures
sont lissées en arrière plan (cela dépend du paramètre `checkpoint_completion_target`).
Le bgwriter intervient lorsque des backends ont besoin de libérer des blocs en
mémoire partagée,  contrairement au checkpointer qui le fait lors d'un checkpoint.

Pour en revenir au graphe, la courbe bleue correspond au nombre de blocs qui ont
dû être alloués en mémoire partagée. Le graphe a cette forme car il correspond à
un test de charge avec pgbench alors que le serveur venait d'être démarré.

On voit assez nettement l'effet du cache qui se charge progressivement. C'est
assez visible, car la collecte se fait à chaque seconde.

On peut voir cet effet de chargement du cache sur plusieurs graphes :

{{< figure src="/img/2018/netdata-postgres08.png" title="Accès disque" >}}
Au debut du graphe on peut constater une activité très soutenue en lecture.
Les pics en écriture correspondent aux checkpoints.


{{< figure src="/img/2018/netdata-postgres09.png" title="IOWait" >}}

Ici, c'est la zone rose qui est intéressante. On peut voir qu'au démarrage du test,
les processeurs étaient en attente d'accès disque.

{{< figure src="/img/2018/netdata-postgres03.png" title="Lectures disques - Transaction par seconde" >}}

Ici on peut voir que le moteur a dû lire les blocs depuis le disque. Le graphe
décroit, car les lectures suivantes se font depuis la mémoire partagée.

On peut également voir que le nombre de transactions par seconde augmente, puis
se stabilise une fois que les données sont chargées en mémoire partagée.

On observe le même phénomène sur ce graphe :

{{< figure src="/img/2018/netdata-postgres04.png" title="Statistiques base bench" >}}

On peut ainsi constater qu'il a fallu une 1min30 pour charger le cache.



[^1]: Voir : [Netdata Performance](https://github.com/firehol/netdata/wiki/Performance)
[^2]: La Pull Request a été mergée récemment : [Add charts for postgres](https://github.com/firehol/netdata/pull/3400)
