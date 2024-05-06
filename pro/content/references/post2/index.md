---
title: Construction d'infrastructure
subtitle: Postgres on-premise

# Summary for listings and search engines
summary: There is no cloud. It’s just someone else’s computer.

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

tags:
  - Infrastructure
  - Linux
  - Tuning
  - Replication
  - Sauvegardes PITR
  - pgBackRest
  - S3


#
#
#categories:
#  - Realisations

---

{{% toc %}}

# There is no cloud. It’s just someone else’s computer

![Someone else computer](someone.png)

Même si j'ai suivi le virage du cloud, j'ai toujours souhaité garder un pied dans l'infrastructure et le système [^1].

[^1]: La page que vous lisez est hébergée chez moi. J'administre mon petit serveur (HTTPS, reverse proxy, sauvegardes...).

Je pense qu'il est indispensable de maintenir ce type de connaissances pour comprendre les problématiques de performance.

Cela me permet aujourd'hui de construire des infrastructures complètes pour des clients pour différentes raisons :

* Gros acteur public qui possède déjà une infrastructure physique et ne souhaite pas dépendre d'un cloud provider.
* Recherche de performances : des serveurs dédiés avec stockage local sont bien plus performance que les instances dans le cloud.
* Economiques : quand l'infrastructure devient conséquente, le cloud peut s'avérer très cher.


Je constate depuis le courant de l'année 2023 un virement à propos du cloud et visiblement, je ne suis pas le seul à partager ce constat :

* [Les DSI reviennent sur leurs stratégies 100% cloud](https://www.cio-online.com/actualites/lire-les-dsi-reviennent-sur-leurs-strategies-100-cloud-15499.html) - *01 Mars 2024*
  *"L'abandon du cloud a été un thème important en 2023 et il y a de fortes chances qu'il devienne une véritable tendance en 2024. Les économies sont tout simplement trop importantes pour être ignorées par un grand nombre d'entreprises"*
* [FinOps : les mentalités évoluent, les dépassements de budget restent](https://www.cio-online.com/actualites/lire-les-dsi-reviennent-sur-leurs-strategies-100-cloud-15499.html) - *17 Avril 2024*
   *"7 entreprises sur 10 ne parviennent toujours pas à tenir leur budget cloud, faute de visibilité suffisante et de capacité à intégrer le contrôle des coûts dès les phases amont des projets"*

David Heinemeier Hansson, créateur du framework Ruby On Rails a écris de nombreux articles à ce sujet :


* [Why we're leaving the cloud](https://world.hey.com/dhh/why-we-re-leaving-the-cloud-654b47e0)
* [Our cloud exit has already yielded $1m/year in savings](https://world.hey.com/dhh/our-cloud-exit-has-already-yielded-1m-year-in-savings-db358dea)
* [The Big Cloud Exit FAQ](https://world.hey.com/dhh/the-big-cloud-exit-faq-20274010)
* [We have left the cloud ](https://world.hey.com/dhh/we-have-left-the-cloud-251760fb)
* [Cloud exit pays off in performance too](https://world.hey.com/dhh/cloud-exit-pays-off-in-performance-too-4c53b697)
* [Five values guiding our cloud exit](https://world.hey.com/dhh/five-values-guiding-our-cloud-exit-638add47)
* [Hardware is fun again](https://world.hey.com/dhh/hardware-is-fun-again-b819d0b4)
* [We stand to save $7m over five years from our cloud exit](https://world.hey.com/dhh/we-stand-to-save-7m-over-five-years-from-our-cloud-exit-53996caa)
* [The hardware we need for our cloud exit has arrived](https://world.hey.com/dhh/the-hardware-we-need-for-our-cloud-exit-has-arrived-99d66966)

TL;DR :

* Ils ont déployé plusieurs serveurs physique sur deux sites séparés (4000 vCPUs, 7680GB de RAM, et 384TB de stockage NVMe)
* Ils vont économiser ~~7 millions~~ 10 **millions** de dollars sur 5 ans.
* L'achat du matériel a été rentabilisé en 6 mois.
* Les performances sont bien meilleures.

![The Big Cloud Exit FAQ](cloud-spend-2022.png)


# Exemple d'architecture


[![Exemple architecture](https://mermaid.ink/img/pako:eNpt0UFrgzAUB_CvEnJSqFBjexHay7zsMCizp9Ud3pKnDdNE0qRj1H73RRthQm95P_4h773cKNcCaU7rVv_wMxhLjkWlSN-s01N0MLIDaTD-fBA7RSVyrcSIJJ05WzCbebPgbObtgjcTf3336zSqIa8h8U2IRBh5RVKCu2LjayTFSxxyLOR4q534HymzOLSZJPsBLr-KD2MbYZwRZ2NPLAvDLC5vQ3C32w-H1-P7MHX6DBld0Q6N35fw67z5CKmoPWOHFc39UWANrrUVrdTdR8FZXfpHaG6NwxV1vQCLhYTGQDdjD-pDa1_W0F58jUJabd4ePzZ93P0P5KeVog?type=png)](https://mermaid.live/edit#pako:eNpt0UFrgzAUB_CvEnJSqFBjexHay7zsMCizp9Ud3pKnDdNE0qRj1H73RRthQm95P_4h773cKNcCaU7rVv_wMxhLjkWlSN-s01N0MLIDaTD-fBA7RSVyrcSIJJ05WzCbebPgbObtgjcTf3336zSqIa8h8U2IRBh5RVKCu2LjayTFSxxyLOR4q534HymzOLSZJPsBLr-KD2MbYZwRZ2NPLAvDLC5vQ3C32w-H1-P7MHX6DBld0Q6N35fw67z5CKmoPWOHFc39UWANrrUVrdTdR8FZXfpHaG6NwxV1vQCLhYTGQDdjD-pDa1_W0F58jUJabd4ePzZ93P0P5KeVog)

Voici un exemple d'architecture que j'ai réalisé pour un client :

* 1 primaire et 5 réplicas : 2 synchrones, 2 asynchrones en cascade
* Serveur Postgres :
  * AMD EPYC 9354 32-Core / 64 threads
  * 512Go RAM DDR5
  * 5.5To de stockage NVMe utile (plusieurs **millions** d'IOPS)
  * Réseau en 25 Gb/s
* Sauvegarde "locale" :
  * Point In Time Recovery
  * AMD EPYC 9124 16-Core
  * 128Go DDR5
  * 15To de stockage utile en NVMe
  * Réseau en 25 Gb/s
* Sauvegarde externalisée sur équivalent S3
* Cout : environ 4000€HT/mois vs plus de 300 000$/mois sur AWS RDS. Les performances sont même meilleures grace au stockage.
* Temps de sauvegarde pour une base de 1To moins de 5 minutes pour une sauvegarde full, quelques secondes pour sauvegarde différentielle.
* Temps de restauration 5 minutes.
* Tuning Linux : RAID, Kernel...
* Tuning Postgres
* Rédaction des procédures d'installation, sauvegardes / restauration, bascules...


Bien sûr, il m'est arrivé de réaliser des architectures plus modestes :

* Sur des machines virtuelles
* Avec un pooler de connexion (PgBouncer)
* Accompagnement sur supervision : Datadog, Nagios like (Icinga, Thruk) avec [check_pgactivity](https://github.com/OPMDG/check_pgactivity).
