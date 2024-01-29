---
# Documentation: https://wowchemy.com/docs/managing-content/

title: "Optimisation du GROUP BY"
subtitle: "Nouveauté dans PostgreSQL 17. Réorganisation du GROUP BY"
summary: "Nouveauté dans PostgreSQL 17. Réorganisation du GROUP BY"
authors: []
tags: ['PostgreSQL 17','Nouveautés']
categories: ['Postgres']
date: 2024-01-26T12:00:00+01:00
lastmod: 2024-01-26T12:00:00+01:00
featured: false

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

Un commit a attiré mon attention lors de ma veille technique :

```diff
commit 0452b461bc405e6d35d8a14c02813c15e28ae516
Author:     Alexander Korotkov <akorotkov@postgresql.org>
AuthorDate: Sun Jan 21 22:21:36 2024 +0200
Commit:     Alexander Korotkov <akorotkov@postgresql.org>
CommitDate: Sun Jan 21 22:21:36 2024 +0200

    Explore alternative orderings of group-by pathkeys during optimization.

    When evaluating a query with a multi-column GROUP BY clause, we can minimize
    sort operations or avoid them if we synchronize the order of GROUP BY clauses
    with the ORDER BY sort clause or sort order, which comes from the underlying
    query tree. Grouping does not imply any ordering, so we can compare
    the keys in arbitrary order, and a Hash Agg leverages this. But for Group Agg,
    we simply compared keys in the order specified in the query. This commit
    explores alternative ordering of the keys, trying to find a cheaper one.

    The ordering of group keys may interact with other parts of the query, some of
    which may not be known while planning the grouping. For example, there may be
    an explicit ORDER BY clause or some other ordering-dependent operation higher up
    in the query, and using the same ordering may allow using either incremental
    sort or even eliminating the sort entirely.

    The patch always keeps the ordering specified in the query, assuming the user
    might have additional insights.

    This introduces a new GUC enable_group_by_reordering so that the optimization
    may be disabled if needed.

    Discussion: https://postgr.es/m/7c79e6a5-8597-74e8-0671-1c39d124c9d6%40sigaev.ru
    Author: Andrei Lepikhov, Teodor Sigaev
    Reviewed-by: Tomas Vondra, Claudio Freire, Gavin Flower, Dmitry Dolgov
    Reviewed-by: Robert Haas, Pavel Borisov, David Rowley, Zhihong Yu
    Reviewed-by: Tom Lane, Alexander Korotkov, Richard Guo, Alena Rybakina
```

On remarque le message du commit assez explicite en mentionnant les personnes impliquées (12 relecteurs!) ainsi que le lien vers la discussion :
[POC: GROUP BY optimization](https://www.postgresql.org/message-id/flat/7c79e6a5-8597-74e8-0671-1c39d124c9d6%40sigaev.ru)

Les travaux ont commencé en 2018! Il aura fallu 5 ans et demi pour aboutir à ce commit. C'est le fruit de nombreux échanges afin de parvenir à un consensus en prenant en compte les multiples idées.

Pour tester ce patch, j'ai compilé Postgres depuis les sources. Prenons un exemple tout simple :


```SQL
create table t1 (c1 int, c2 int, c3 int);
create index on t1 (c1,c2);
insert into t1 SELECT i%100, i%1000, i from generate_series(1,10_000_000) i;
vacuum analyze t1;
```

On obtient une table de 422Mo et un index de 66Mo.
Vous noterez au passage que j'ai utilisé une nouveauté de Postgres 16 en utilisant des "underscore" dans le *generate_series*[^1].

[^1]: Petite anecdote, pour ajouter cette fonctionnalité, l'auteur a fait évoluer le standard SQL : [Grouping digits in SQL](http://peter.eisentraut.org/blog/2023/09/20/grouping-digits-in-sql) .

Si on fait un `GROUP BY` sur *c1,c2*, on obtient un parcours d'index :

```
explain (settings, analyze,buffers) select count(*),c1,c2 from t1 group by c1,c2 order by c1,c2;
                                                                    QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------------------
 GroupAggregate  (cost=0.43..235342.27 rows=100000 width=16) (actual time=3.605..3501.887 rows=1000 loops=1)
   Group Key: c1, c2
   Buffers: shared hit=10447
   ->  Index Only Scan using t1_c1_c2_idx on t1  (cost=0.43..159340.96 rows=10000175 width=8) (actual time=0.070..1900.730 rows=10000000 loops=1)
         Heap Fetches: 0
         Buffers: shared hit=10447
 Settings: enable_group_by_reordering = 'off', random_page_cost = '1.1', max_parallel_workers_per_gather = '0'
 Planning:
   Buffers: shared hit=1
 Planning Time: 0.191 ms
 Execution Time: 3502.036 ms
```

*Vous remarquerez que j'ai volontairement désactivé la fonctionnalité dans un premier temps (enable_group_by_reordering=off). J'ai également désactivé la parallélisation pour plus de clarté.*

On a le plan attendu, le moteur va lire 10447 blocs avec un parcours *Index Only Scan*. Le moteur va chercher le plan qui manipule le moins de blocs possible.

En revanche, si on change l'ordre du group by :

```
explain (settings, analyze,buffers) select count(*),c1,c2 from t1 group by c2,c1 order by c1,c2;
                                                          QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------
 Sort  (cost=602422.32..602672.32 rows=100000 width=16) (actual time=5393.446..5393.503 rows=1000 loops=1)
   Sort Key: c1, c2
   Sort Method: quicksort  Memory: 79kB
   Buffers: shared hit=96 read=53959
   ->  HashAggregate  (cost=514992.50..594117.50 rows=100000 width=16) (actual time=5392.351..5392.894 rows=1000 loops=1)
         Group Key: c2, c1
         Batches: 1  Memory Usage: 3217kB
         Buffers: shared hit=96 read=53959
         ->  Seq Scan on t1  (cost=0.00..154055.00 rows=10000000 width=8) (actual time=0.033..1186.171 rows=10000000 loops=1)
               Buffers: shared hit=96 read=53959
 Settings: enable_group_by_reordering = 'off', random_page_cost = '1.1', max_parallel_workers_per_gather = '0'
 Planning:
   Buffers: shared hit=1
 Planning Time: 0.189 ms
 Execution Time: 5394.566 ms
```

Dans ce cas, le moteur va lire toute la table (*seqscan*) et faire un tri, alors que le résultat est identique.

Le moteur lit 422Mo de données contre 80Mo, soit 5 fois plus. Le résultat peut être désastreux suivant les performances du stockage.
Là, on a de la "chance", mon instance est dans un ramdisk donc la requête n'est pas beaucoup plus lente.
Avec du stockage mécanique ou des disques cloud, le temps de réponse peut augmenter drastiquement.


L'ordre des colonnes dans un group by est important, c'est une optimisation assez simple pour peu qu'on connaisse le schéma de la base et qu'on maitrise les requêtes exécutées sur le serveur.

Malheureusement, avec les ORM, on peut perdre cette maitrise ou rater des corrections.
C'est là où cette fonctionnalité est intéressante. Voyons son effet :

```
postgres=# set enable_group_by_reordering to on;
SET
postgres=# explain (settings, analyze,buffers) select count(*),c1,c2 from t1 group by c2,c1 order by c1,c2;
                                                                    QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------------------
 GroupAggregate  (cost=0.43..235338.33 rows=100000 width=16) (actual time=4.502..3658.168 rows=1000 loops=1)
   Group Key: c1, c2
   Buffers: shared hit=10447
   ->  Index Only Scan using t1_c1_c2_idx on t1  (cost=0.43..159338.33 rows=10000000 width=8) (actual time=0.081..1923.553 rows=10000000 loops=1)
         Heap Fetches: 0
         Buffers: shared hit=10447
 Settings: random_page_cost = '1.1', max_parallel_workers_per_gather = '0'
 Planning:
   Buffers: shared hit=1
 Planning Time: 0.230 ms
 Execution Time: 3658.337 ms
```

On retrouve le premier plan bien plus performant.

C'est une fonctionnalité assez simple qui devrait améliorer les temps d'exécutions de certaines requêtes dont la clause group by n'a pas été optimisée.

C'est une demande assez récurrente d'améliorer le planificateur afin de gérer des cas en apparence simple. Il faut avoir en tête que le risque est de d'augmenter
les opérations de calculs dans le planificateur. Or, on veut que cette étape soit la plus rapide possible La réponse est souvent résumée à : "on ne veut pas alourdir le planificateur alors qu'on peut corriger la requête".

Cependant, ce type d'optimisation peut être acceptée si on sait que ça ne sera pas couteux pour le planificateur.

Il n'y a plus qu'à espérer que cette fonctionnalité ne soit pas retirée d'ici la sortie de Postgres 17 :)


