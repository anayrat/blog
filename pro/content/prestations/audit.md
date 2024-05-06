---
title: Audit
subtitle: Est-ce que votre base de données est en bonne santé ?

# Summary for listings and search engines
summary: Est-ce que votre base de données est en bonne santé ? Un audit vous assure que votre base de donnée est correctement configurée pour des questions de fiabilité, performances, mais également coûts.

# Date published
date: '2023-10-26T00:00:00Z'

# Date updated
lastmod: ''

# Is this an unpublished draft?
draft: false

# Show this page in the Featured widget?
featured: false
reading_time: false
show_date: false

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
image:
  caption: ''
  focal_point: ''
  placement: 2
  preview_only: false

authors:
  - adrien
profile: false
highlight_name: false

# tags:
#   - Audit
#
# categories:
#   - Audit
#
#categories:
#  - Prestations
---

C'est souvent le point d'entrée de la plupart de mes prestations. Avant de travailler sur des problèmes spécifiques, il faut s'assurer que les fondations sont bien construites.

Je commence toujours par les couches basses, puis, je les remonte jusqu'à arriver à l'applicatif :

## Configuration et tuning de Linux

J'inspecte le stockage (cartes contrôleur, raid, LVM...), La configuration du kernel Linux (mémoire, scheduler, disque).

Chez les cloud provider, on n'a évidemment pas accès à ces informations, cependant, il faut faire une étape assez similaire qui consiste
à étudier les documentations et spécifications techniques des instances (CPU, Burst, IOPS...).


## Revue de l'activité système


Je regarde comment se comporte le serveur ou l'instance. Est-ce qu'il y a une saturation du stockage, CPU, RAM ?

## Revue de l'activité Postgres

J'identifie le comportement de Postgres : taches de maintenance (*vacuum*,*checkpoint*...), les problèmes de contention (locks), erreurs applicatives...

Cela va m'aider à comprendre la charge de travail de votre instance et savoir quoi et où regarder. Je pourrais aussi proposer une configuration adaptée.

## Tuning Postgres

*Savez-vous qu'il existe plus de 350 paramètres de configuration ? Leur nombre augmente (raisonnablement) au fil des versions. Certains disparaissent, d'autres apparaissent.*

Revue de toute la configuration de Postgres. La configuration par défaut est très conservatrice, il faut absolument la revoir.
Les outils de configuration automatiques tels que [PGTune](https://pgtune.leopard.in.ua/) sont souvent insuffisants et inadapté.

*Dites-vous que s'il suffisait d'appliquer de simples règles, elles seraient déjà implémentées dans le moteur !*

## Revue des requêtes lentes / consommatrices

Maintenant que j'ai une meilleure idée de l'état de votre instance, je peux commencer à inspecter les requêtes lentes et / ou consommatrices.

C'est en général à cette étape que l'on peut espérer les plus gros gains en performance.

Évidemment, je ne me contente pas de vous indiquer les requêtes qui posent un problème. Je vais vous proposer :

* Des corrections sur la requête
* La création de nouveaux index
* Revoir le comportement applicatif

## Revue du schéma

* Est-ce qu'il manque des index ? Utilisez-vous le bon type d'index ?
* Est-ce que les types sont correctement utilisés ?

## Bilan

Le bilan, c'est un rapport complet en PDF. Je ne me contente pas d'appliquer des corrections. J'explique les corrections afin que vous gagniez en autonomie et évitiez de faire les mêmes erreurs.
