+++
title = "pg_sampletolog : Une extension permettant de loguer un échantillon de requêtes"
date = 2019-01-28T08:30:00+02:00
draft = true

summary = "Cette extension peut s'avérer utile pour diagnostiquer des requêtes dont le temps d’exécution est très court."


# Tags and categories
# For example, use `tags = []` for no tags, or the form `tags = ["A Tag", "Another Tag"]` for one or more tags.
tags = ["postgres","extension","pg_sampletolog","échantillonnage","logs"]
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

Cet article va vous présenter une extension que j'ai développé dans le but de
loguer un échantillon de requêtes.

Lorsqu'un DBA est confronté à un problème de performance, il aura pour
réflexe d'inspecter les logs, mais également la vue `pg_stat_statements`.
Une requête coûteuse apparaîtra dans `pg_stat_statements` et dans les logs si la
requête dépasse `log_min_duration_statement`. On peut ainsi rejouer la requête,
obtenir son plan d'exécution et investiguer.

Pour aller encore plus loin, il est possible d'activer l'extension `auto_explain`.
Ainsi on aura directement le plan de la requête. Au passage, l'option `auto_explain.log_analyze`
n'implique pas une double exécution de la requête. Ce paramètre peut être activé
sans crainte. Néanmoins cela peut s'avérer coûteux car le moteur doit mettre en
place l'instrumentation pour obtenir le *timing* des différents nœuds. Si on a
beaucoup de trafic, il est également possible de faire un échantillonnage avec
`auto_explain.sample_rate`. Cette option peut produire une quantité importante
de logs ce qui peut être problématique sur une instance à fort trafic.

J'ai été confronté à un problème tout simple : comment investiguer sur une requête
dont le temps d'exécution est très court? C'est très simple "Regardez `pg_stat_statements`!".

Voici ce qu'on pourrait obtenir sur un test pgbench :
```
query               | SELECT abalance FROM pgbench_accounts WHERE aid = $1
calls               | 12000
total_time          | 214.564185000001
min_time            | 0.013751
max_time            | 0.044711
mean_time           | 0.0178803487499999
```

La requête est normalisée. Sans paramètre impossible d'obtenir son plan. choisir
un paramètre au hasard n'est pas la bonne solution : ce n'est pas forcément représentatif
du véritable trafic de production.

Il y a quelques mois, j'ai proposé un patch pour loguer un échantillon de requêtes.
Celui-ci a été intégré dans la version 12 en cours de développement :

```
commit 88bdbd3f746049834ae3cc972e6e650586ec3c9d
Author:     Alvaro Herrera <alvherre@alvh.no-ip.org>
AuthorDate: Thu Nov 29 18:42:53 2018 -0300
Commit:     Alvaro Herrera <alvherre@alvh.no-ip.org>
CommitDate: Thu Nov 29 18:42:53 2018 -0300

    Add log_statement_sample_rate parameter

    This allows to set a lower log_min_duration_statement value without
    incurring excessive log traffic (which reduces performance).  This can
    be useful to analyze workloads with lots of short queries.

    Author: Adrien Nayrat
    Reviewed-by: David Rowley, Vik Fearing
    Discussion: https://postgr.es/m/c30ee535-ee1e-db9f-fa97-146b9f62caed@anayrat.info
```

Dans le fil des échanges, Nikolay Samokhvalov a émis l'idée d'avoir ce même type
fonctionnalité mais au niveau d'une transaction : <https://www.postgresql.org/message-id/CANNMO%2BLg65EFqHb%2BZYbMLKyE2y498HJzsdFrMnW1dQ6AFJ3Mpw%40mail.gmail.com>

J'ai également proposé un patch en ce sens : [Log a sample of transactions](https://commitfest.postgresql.org/21/1923/)

Tout ça est intéressant, mais il faudra attendre la version 12. Et encore, rien
ne garanti que le second patch soit commité (ou que le premier ne soit pas reverté).

Tout ça m'a donné l'idée et l'envie de créer une extension, c'est ainsi qu'est née
[pg_sampletolog](https://github.com/anayrat/pg_sampletolog).

Cette extension permet de :

  * Loguer un échantillon de requêtes
  * Loguer un échantillon de transactions
  * Choisir de loguer avant ou après l'exécution (pour un usage futur avec pgreplay)
  * Choisir de loguer tous les ordres de type DDL ou qui impliquent une modification des donnée

Elle fonctionne sur toutes les versions supportées, de la 9.4 à la 11.

L'activation se fait soit dans une session en chargeant l'extension : `LOAD 'pg_sampletolog';`.
Ou dans le fichier postgresql.conf pour qu'elle soit chargée lors de toute nouvelle
connexion avec  `session_preload_libraries = 'pg_sampletolog'`

Si l'extension est bien chargée, ces nouveaux paramètres apparaissent :
```
select name from pg_settings where name like 'pg_sampletolog%';
                 name                 
----------------------------------------
 pg_sampletolog.disable_log_duration
 pg_sampletolog.log_before_execution
 pg_sampletolog.log_level
 pg_sampletolog.log_statement
 pg_sampletolog.statement_sample_rate
 pg_sampletolog.transaction_sample_rate
(6 rows)
```

Voici quelques exemples :

  * Loguer seulement 10% des requêtes : `pg_sampletolog.statement_sample_rate = 0.1`

pg_sampelog va loguer 10% des requêtes. Pour chaque requête, le moteur va faire
un tirage aléatoire à l'aide de la fonction `random()`. Le coût de cette fonction
est très faible, donc il ne devrait pas y avoir d'impact sur les performances.
Après quelques requêtes vous devriez obtenir ce genre de message dans les logs :

```
2019-01-27 12:50:39.361 CET [27047] LOG:  Sampled query duration: 0.014 ms - SELECT 1;
```
pg_sampelog va loguer la requête ainsi que son temps d'exécution.

  * Loguer seulement 10% des transactions : `pg_sampletolog.transaction_sample_rate = 0.1`

Le fonctionnement est le même que précédemment, à la différence que le moteur va
choisir de loguer ou non toutes les requêtes d'une même transaction. Cela peut s'avérer
très utile pour comprendre ce que fait un applicatif. Par exemple lorsqu'on ne peut
pas accéder au code de l'applicatif ou lorsque les requêtes sont générées par un ORM.
Exemple avec une transaction toute simple `BEGIN; SELECT 1; SELECT 1; COMMIT;`
```
2019-01-27 12:51:40.562 CET [27069] LOG:  Sampled transaction duration: 0.008 ms - SELECT 1;
2019-01-27 12:51:40.562 CET [27069] LOG:  Sampled transaction duration: 0.005 ms - SELECT 1;
```

Les deux SELECT ont bien été logués. En adaptant le `log_line_prefix`, on peut
voir qu'il s'agit de la même transaction (regardez le *lxid*):

```
2019-01-27 16:32:16 CET [18556]: lxid=3/177,db=postgres,user=anayrat LOG:  Sampled transaction - SELECT 1;
2019-01-27 16:32:16 CET [18556]: lxid=3/177,db=postgres,user=anayrat LOG:  Sampled transaction - SELECT 1;
```

  * Loguer tous les ordres DDL : `pg_sampletolog.log_statement = 'ddl'`:

pg_sampletolog va loguer tous les ordres de type DDL (`CREATE TABLE`,`CREATE INDEX`,...).
Ca peut être utile si on veut juste loguer un échantillon en lecture mais tous
les ordres DDL.

```
2019-01-27 12:53:47.564 CET [27103] LOG:  Sampled ddl CREATE TABLE t1(c1 int);
```

  * Loguer tous les ordres modifiants des données : `pg_sampletolog.log_statement = 'mod'`:

Exactement comme l'exemple précédent, mais cette fois on logue aussi tous les `UPDATES, DELETE`.
Cela comprend aussi les ordres DDL.

```
2019-01-27 12:59:54.043 CET [27160] LOG:  Sampled query duration: 0.246 ms - INSERT INTO t1 VALUES(1);
2019-01-27 13:00:16.468 CET [27160] LOG:  Sampled ddl CREATE INDEX ON t1(c1);
```

  * Loguer la requête avant son exécution : `pg_sampletolog.log_before_execution = on`

Cette option pourrait être utile pour rejouer logs avec [pgreplay](https://github.com/laurenz/pgreplay).


# Bonus

L'extension fonctionne aussi sur les serveurs secondaires.

# Deuxième bonus

Si `pg_stat_statements` est activée, le queryid est également logué. Ca peut être
très utile si vous identifiez une requête dans `pg_stat_statements` et que vous
souhaitez la retrouver dans les logs à l'aide de son queryid.

# Conclusion

Je me suis régalé avec ce projet personnel. J'ai beaucoup appris sur ~~les segfaults~~
le code de postgres et ça montre également les possibilités d'extension du moteur.

A l'avenir j'aimerai rajouter la possibilité de loguer un échantillon de requête
correspondant à tel queryid. Il faut également que je regarde pour supporter les
requêtes préparées.

Enfin, j'aimerai tester cette extension avec pgreplay : En loguant tous les ordres
*MOD* (afin d'assurer la cohérence lors du rejeu) ainsi qu'une fraction des
requêtes en lecture. Puis, restaurer un backup PITR et d'un côté rejouer le trafic en écriture.
De l'autre côté, rejouer une portion du trafic en lecture avec un *speed_factor*.
Par exemple x10 en rejouant 10% du trafic. Même si ça ne sera jamais parfait (il
manquera la cohérence des lectures), je serai curieux de voir les résultats qu'on
peut obtenir. Surtout dans le cas où loguer toutes les requêtes s’avérerait trop
coûteux.


Je suis preneur de tout retour à faire sur [la page github du projet](https://github.com/anayrat/pg_sampletolog).
