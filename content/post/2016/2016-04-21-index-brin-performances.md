---
title: Index BRIN – Performances
authors: ['adrien']
type: post
date: 2016-04-21T20:58:55+00:00
aliases: /2016/04/21/index-brin-performances/
categories:
  - Linux
  - Postgres
tags:
  - brin
  - index
  - postgres
show_related: true

---
La version 9.5 de PostgreSQL sortie en Janvier 2016 propose un nouveau type d'index : les Index BRIN pour Bloc Range INdex. Ces derniers sont recommandés pour les tables volumineuses et corrélées avec leur emplacement. J'ai décidé de consacrer une série d'article sur ces index :

* [Index BRIN - Principe][1]
* [Index BRIN - Fonctionnement][2]
* [Index BRIN - Corrélation][3]
* [Index BRIN - Performances][4]

Pour information, je serai présent au [PGDay France][4] à Lille le mardi 31 mai pour présenter cet index. Il y aura également plein d'autres [conférences intéressantes][5]!

Cet article est la dernier de la série, il sera consacré aux performances (maintenance, lecture, insertion...)

<!--more-->

# Performances

Les articles précédents ont abordé le fonctionnement et les spécificités des index BRIN. Cet article sera plutôt consacré aux performances. Les exemples précédents portaient sur de petites volumétries. Maintenant nous allons voir ce que peuvent apporter ces index sur une volumétrie plus importante.

Les tests ont été effectués sur un PC portable, les journaux de transaction, la table et les index sont stockés sur un disque mécanique. Les résultats seront différents suivant la matériel utilisé. Ces chiffres sont purement indicatifs et servent surtout à se donner un ordre d'idée.

## Exemple

Dans un premier temps il est nécessaire de créer une table avec une volumétrie importante.

Par exemple : un système de mesure avec 100 sondes et une mesure toutes les secondes. Nous allons donc obtenir 100\*365\*24*3600 mesures => un peu plus de 3 milliards de lignes.

```SQL
-- Utilisation d'une fonction pour générer du texte aléatoire.
-- Trouvée ici : http://stackoverflow.com/questions/3970795/how-do-you-create-a-random-string-in-postgresql

CREATE OR REPLACE FUNCTION random_string(LENGTH INTEGER) RETURNS text AS
$$
DECLARE
  chars text[] := '{0,1,2,3,4,5,6,7,8,9,A,B,C,D,E,F,G,H,I,J,K,L,M,N,O,P,Q,R,S,T,U,V,W,X,Y,Z,a,b,c,d,e,f,g,h,i,j,k,l,m,n,o,p,q,r,s,t,u,v,w,x,y,z}';
  RESULT text := '';
  i INTEGER := 0;
BEGIN
  IF LENGTH < 0 THEN
    raise exception 'Given length cannot be less than 0';
  END IF;
  FOR i IN 1..LENGTH loop
    RESULT := RESULT || chars[1+random()*(array_length(chars, 1)-1)];
  END loop;
  RETURN RESULT;
END;
$$ LANGUAGE plpgsql;
-- Creation de la table contenant les sondes
CREATE TABLE probe (id serial PRIMARY KEY, name text);
INSERT INTO probe (name ) SELECT random_string(5) FROM generate_series(1,100);

CREATE TABLE data AS
WITH generation AS (
SELECT '2015-01-01'::TIMESTAMP + i * INTERVAL '1 second' AS date_metric,sonde::text,random() AS metric
FROM generate_series(0, 3600*24*365) i,
LATERAL (SELECT name FROM probe) sonde)
SELECT * FROM generation;
```

La table obtenue fait un peu plus de 150Go, elle ne rentre donc pas dans  la RAM de la machine hébergeant l'instance et encore moins dans la mémoire partagée de PostgreSQL.

## Maintenance

On crée plein d'index pour comparer leur taille :

```SQL
CREATE INDEX metro_btree_idx ON data USING btree (date_metric);
CREATE INDEX metro_brin_idx_8 ON data USING brin (date_metric) WITH (pages_per_range = 8);
CREATE INDEX metro_brin_idx_16 ON data USING brin (date_metric) WITH (pages_per_range = 16);
CREATE INDEX metro_brin_idx_32 ON data USING brin (date_metric) WITH (pages_per_range = 32);
CREATE INDEX metro_brin_idx_64 ON data USING brin (date_metric) WITH (pages_per_range = 64);
CREATE INDEX metro_brin_idx_128 ON data USING brin (date_metric);
CREATE INDEX metro_brin_idx_256 ON data USING brin (date_metric) WITH (pages_per_range = 256);
```

Voici les résultats obtenus pour la durée de création des index et leurs tailles :

![size-large](/img/2016/size-large.png)

![duration-large](/img/2016/duration-large.png)


La création de l'index a été 4 fois plus rapide pour les index BRIN. Il est possible que leur création aurait été plus rapide avec un stockage plus performant.

La taille des index est également frappante,  l'index b-tree fait 66 Go alors que l'index BRIN avec le pages\_per\_range par défaut fait seulement 5 Mo.

On peut tout suite constater le gain sur l'espace utilisé et la rapidité de création des index. Les opérations de maintenances (REINDEX) seront grandement facilitées.

## Performances en lecture

Nous allons effectuer plusieurs tests, l'idée est d'essayer de mettre en valeur les différences de comportements entre les index BRIN et b-tree.

La requête utilisée sera tout simple :

```SQL
EXPLAIN (ANALYSE,BUFFERS,VERBOSE) SELECT date_metric,sonde,metric FROM DATA WHERE date_metric = 'xxx';
```

Pour obtenir un résultat avec peu de lignes :

```SQL
WHERE date_metric = '2015-05-01 00:00:00'::TIMESTAMP
```

Pour obtenir plus de résultat on prendra un intervalle avec :

```SQL
WHERE date_metric BETWEEN 'xxx'::TIMESTAMP AND 'xxx'::TIMESTAMP;
```

Voici les résultats obtenus :

<table class="inline">
  <tr class="row0">
    <th class="col0" rowspan="2">
      lignes
    </th>

    <th class="col1" colspan="2">
      BRIN
    </th>

    <th class="col3" colspan="2">
      Btree
    </th>

    <th class="col5" colspan="2">
      Gain
    </th>
  </tr>

  <tr class="row1">
    <th class="col0">
      Durée
    </th>

    <th class="col1">
      Blocs lus
    </th>

    <th class="col2">
      Durée
    </th>

    <th class="col3">
      Blocs lus
    </th>

    <th class="col4">
      Durée
    </th>

    <th class="col5">
      Volume données lues
    </th>
  </tr>

  <tr class="row2">
    <td class="col0">
      100
    </td>

    <td class="col1">
      24ms
    </td>

    <td class="col2">
      697
    </td>

    <td class="col3">
      0.06ms
    </td>

    <td class="col4">
      7
    </td>

    <td class="col5">
      Btree (x400)
    </td>

    <td class="col6">
      Btree (x100)
    </td>
  </tr>

  <tr class="row3">
    <td class="col0">
      267 millions
    </td>

    <td class="col1">
      170s
    </td>

    <td class="col2">
      13Go
    </td>

    <td class="col3">
      228s
    </td>

    <td class="col4">
      18Go
    </td>

    <td class="col5">
      BRIN (x1.3)
    </td>

    <td class="col6">
      BRIN (x1.4)
    </td>
  </tr>

  <tr class="row4">
    <td class="col0">
      777 millions
    </td>

    <td class="col1">
      8min
    </td>

    <td class="col2">
      38Go
    </td>

    <td class="col3">
      11min
    </td>

    <td class="col4">
      54Go
    </td>

    <td class="col5">
      BRIN (x1.37)
    </td>

    <td class="col6">
      BRIN (x1.4)
    </td>
  </tr>

  <tr class="row5">
    <td class="col0">
      1.3 milliard
    </td>

    <td class="col1">
      13min
    </td>

    <td class="col2">
      63Go
    </td>

    <td class="col3">
      32min (seqscan)<br /> 18min
    </td>

    <td class="col4">
      153 Go (seqscan)<br /> 90 Go
    </td>

    <td class="col5 leftalign">
      BRIN (x2) vs seqscan<br /> BRIN (1.4x) vs Btree
    </td>

    <td class="col6">
      BRIN (x2.4) vs seqscan<br /> BRIN (1.4x) vs Btree
    </td>
  </tr>
</table>

Pour comparer le volume de données lues et la durée d'exécution nous pouvons désactiver les index dans une transaction :

```SQL
BEGIN;
DROP index ...;
explain (analyse,verbose,buffers) SELECT ...
rollback;
```

Pour le 1er test, le moteur choisit l'index-btree. En supprimant l'index b-tree il choisit l'index BRIN.

Pour les tests 2 et 3, le moteur choisit l'index BRIN, en supprimant l'index BRIN il choisit l'index b-tree.

Pour le dernier test j'ai rajouté d'autres mesures. En effet, en supprimant l'index BRIN le moteur va effectuer un seqscan (parcours de toute la table). Pour obtenir les mêmes comparaisons que les résultats précédents j'ai donc supprimé l'index BRIN et désactivé les parcours séquentiels (`set enable_seqscan to 'off';`)

Globalement on peut constater un gain de 30-40% dans les cas où beaucoup de résultats sont demandés. Le moteur lit moins de blocs lorsqu'il utilise les index BRIN, l'index b-tree étant volumineux, ses lectures sont coûteuses.

En revanche l'index b-tree s'avère particulièrement performant lorsque la requête est très sélective et que peu de résultats sont retournés. En effet, en utilisant un index BRIN, le moteur commence par lire l'intégralité de l'index. Puis il va lire un ensemble de blocs qui contiennent la valeur recherchée, certains ne contenants aucun résultat. Ces lectures supplémentaires se ressentent sur la durée d’exécution de la requête.

## Performances en insertion

Vu que les index BRIN sont plus petits et leur durée de création plus courte, on peut se demander ce qu'il advient du surcoût de cet index lors d'insertion de données. Pour cela on va créer une table et mesurer l'insertion de 10 millions de lignes en fonction des index déjà présents sur la table. Afin de cibler le surcoût dû à la mise à jour des index, la table est non-journalisée, ceci permet d'éviter les écritures dans les journaux de transaction. L'autovacuum est également désactivé.

```SQL
CREATE UNLOGGED TABLE brin_demo_2 (c1 INT);
INSERT INTO brin_demo_2 SELECT * FROM generate_series(1,10000000);
TRUNCATE brin_demo_2;

CREATE INDEX brin_demo_2_brin_idx ON brin_demo_2 USING brin (c1);
INSERT INTO brin_demo_2 SELECT * FROM generate_series(1,10000000);
DROP INDEX brin_demo_2_brin_idx;
TRUNCATE brin_demo_2;
 
CREATE INDEX brin_demo_2_brin_idx ON brin_demo_2 USING brin (c1) WITH (pages_per_range = 256);
INSERT INTO brin_demo_2 SELECT * FROM generate_series(1,10000000);
DROP INDEX brin_demo_2_brin_idx;
TRUNCATE brin_demo_2;
...
```

Voici les résultats obtenus :

![insertion](/img/2016/insertion.png)


Cependant on peut constater qu'il est moins coûteux d'insérer des données dans une table avec un index BRIN qu'avec un index b-tree. On constate également qu'il n'y a pas d'écart significatif entre les différents types d'index BRIN.

# Conclusion

Cette série d'articles a permis de présenter les principes des index BRIN. puis leur fonctionnement à travers des exemples simples.

Ensuite nous avons vu l'importance de la corrélation pour exploiter pleinement ces index. Enfin, nous avons essayé de mesurer le gain que pouvait apporter cet index sur de multiples aspect (maintenance, performance en lecture et insertion).

Décrire le fonctionnement d'un index en simplifiant sa représentation est un exercice compliqué. On peut vite sacrifier le fond à la forme. Présenter des chiffres est également délicat tellement ils peuvent dépendre du contexte. J'ai fait l'effort de détailler comment je les ai obtenu afin que chacun puisse reproduire ses propres tests. L'idée est de donner un aperçu des cas d'utilisation de ce type d'index.

Globalement il faut retenir que les index BRIN sont utiles pour les tables volumineuses et où la corrélation avec l'emplacement des données est importante. Ils seront plus lents que les index b-tree lorsque la recherche nécessite de parcourir peu de blocs. Ils seront un peu plus rapide que les index b-tree dans les situations où le moteur doit lire beaucoup de blocs (moins de blocs à lire dans l'index).

L'étude de cet index ouvre d'autres pistes de réflexion. Comme la prise en compte de la corrélation dans le calcul du coût. J'avais également pensé à la possibilité d'utiliser un index pour créer un autre index.

Dans l'exemple avec la table volumineuse (150Go). Si on souhaite créer un index partiel sur le mois précédent, le moteur va parcourir l'intégralité de la table pour créer d'index. On pourrait envisager créer l'index b-tree en utilisant l'index BRIN pour ne parcourir que les lignes correspondant au moins précédent.

[1]: http://blog.anayrat.info/2016/04/19/index-brin-principe/
[2]: http://blog.anayrat.info/2016/04/20/index-brin-fonctionnement/
[3]: http://blog.anayrat.info/2016/04/20/index-brin-correlation/
[4]: http://blog.anayrat.info/2016/04/21/index-brin-performances/
[5]: http://www.pgday.fr/index.html
[6]: http://www.pgday.fr/programme.html
