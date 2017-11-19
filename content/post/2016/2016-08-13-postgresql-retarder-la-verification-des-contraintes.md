---
title: 'PostgreSQL : Retarder la vérification des contraintes'
author: Adrien Nayrat
type: post
date: 2016-08-13T20:42:53+00:00
aliases: /2016/08/13/postgresql-retarder-la-verification-des-contraintes/
categories:
  - Postgres
tags:
  - postgres

---
# Retarder la vérification des contraintes

> Remarque : Cet article a été rédigé durant le cadre de mon activité chez [Dalibo](https://www.dalibo.com)]

Postgres respecte le modèle [ACID](https://fr.wikipedia.org/wiki/Propri%C3%A9t%C3%A9s_ACID), ainsi il garantie la cohérence de la base : une transaction amène la base d'un état stable à un autre.

Les données dans les différentes tables ne sont pas indépendantes mais obéissent à des règles sémantiques mises en place au moment de la conception du modèle conceptuel des données. Les contraintes d'intégrité ont pour principal objectif de garantir la cohérence des données entre elles, et donc de veiller à ce qu'elles respectent ces règles sémantiques. Si une insertion, une mise à jour ou une suppression viole ces règles, l'opération est purement et simplement annulée.

Le moteur effectue la vérification des contraintes à chaque modification (lorsque des contraintes ont été définies). Il est également possible de retarder la vérification des contraintes à la fin de la transaction, au moment du commit. Ainsi, les vérifications ne seront produites que sur les changements effectifs entre les opérations de delete, update et insert de la transaction.

<!--more-->

Exemple :

```SQL
CREATE TABLE t1 (c1 INT PRIMARY KEY, c2 text);
 CREATE TABLE t2 (c1 INT REFERENCES t1(c1), c2 text);

INSERT INTO t1  VALUES(1,'a');
 INSERT INTO t1  VALUES(2,'b');

INSERT INTO t2  VALUES(3,'a');
 ERROR:  INSERT OR UPDATE ON TABLE "t2" violates FOREIGN KEY CONSTRAINT "t2_c1_fkey"
 DETAIL:  KEY (c1)=(3) IS NOT present IN TABLE "t1".
 ```

La ligne insérée dans t2 doit respecter la contrainte d'intégrité référentielle, la table t1 ne contient aucune ligne où c1=3. Insérons une ligne correcte :

```SQL
INSERT INTO t2  VALUES(1,'a');

SELECT * FROM t1;
 c1 | c2
 ----+----
 1 | a
 2 | b
 (2 ROWS)

SELECT * FROM t2;
 c1 | c2
 ----+----
 1 | a
 (1 ROW)
 ```

Que se passe t-il si on souhaite modifier la clé primaire de t1?

```SQL
BEGIN;
 UPDATE t1 SET c1=3 WHERE c1=1;
 ERROR:  UPDATE OR DELETE ON TABLE "t1" violates FOREIGN KEY CONSTRAINT "t2_c1_fkey" ON TABLE "t2"
 DETAIL:  KEY (c1)=(1) IS still referenced FROM TABLE "t2".
 ```

La vérification de la contrainte se fait lors de l'UPDATE et déclenche une erreur. Il est possible de demander au moteur d'effectuer la vérification des contraintes à la fin de la transaction avec l'ordre `SET CONSTRAINTS ALL DEFERRED;`. A noter également que le mot clef ALL peut-être remplacé par le nom d'une contrainte (si elle est nommée et est `DEFERRABLE`)

```SQL
BEGIN;
 SET CONSTRAINTS ALL DEFERRED;
 UPDATE t1 SET c1=3 WHERE c1=1;
 ERROR:  UPDATE OR DELETE ON TABLE "t1" violates FOREIGN KEY CONSTRAINT "t2_c1_fkey" ON TABLE "t2"
 DETAIL:  KEY (c1)=(1) IS still referenced FROM TABLE "t2".
 ```

Ça ne fonctionne toujours pas, en effet il faut préciser que que l'application de la contrainte peut être retardée avec le mot clé `DEFERRABLE`

```SQL
ALTER TABLE t2 ALTER CONSTRAINT t2_c1_fkey DEFERRABLE;

BEGIN;
 SET CONSTRAINTS ALL DEFERRED;
 UPDATE t1 SET c1=3 WHERE c1=1;
 UPDATE t2 SET c1=3 WHERE c1=1;
 commit;

SELECT * FROM t1;
 c1 | c2
 ----+----
 2 | b
 3 | a
 (2 ROWS)

SELECT * FROM t2;
 c1 | c2
 ----+----
 3 | a
 (1 ROW)
 ```

Dans ce cas le moteur accepte de faire la vérification en fin de transaction.

Autre intérêt, si une ligne est effacée et réinsérée dans la même transaction, les vérifications sur cette ligne ne sont pas exécutées (car inutiles).

Exemple, on vide les tables puis on insère 1 million de lignes.

```SQL
TRUNCATE t1 cascade;
 NOTICE:  TRUNCATE cascades TO TABLE "t2"
 TRUNCATE TABLE

EXPLAIN analyse INSERT INTO T1 (c1,c2) (SELECT  *,md5(i::text) FROM (SELECT * FROM generate_series(1,1000000)) i);
```

Ensuite on insère 100 000 lignes, puis on les supprime pour les réinsérer à nouveau (sans indiquer au moteur de différer la vérification de contraintes).

```SQL
BEGIN;

EXPLAIN analyse  INSERT INTO T2 (c1,c2) (SELECT  *,md5(i::text) FROM (SELECT * FROM generate_series(1,1000000)) i);

QUERY PLAN
 ------------------------------------------------------------------------------------------------------------------------------------
 INSERT ON t2  (cost=0.00..17.50 ROWS=1000 width=36) (actual TIME=3451.308..3451.308 ROWS=0 loops=1)
 ->  FUNCTION Scan ON generate_series  (cost=0.00..17.50 ROWS=1000 width=36) (actual TIME=170.218..1882.406 ROWS=1000000 loops=1)
 Planning TIME: 0.054 ms
 TRIGGER FOR CONSTRAINT t2_c1_fkey: TIME=16097.543 calls=1000000
 Execution TIME: 19654.595 ms
 (5 ROWS)

TIME: 19654.971 ms

DELETE FROM t2 WHERE c1 <= 1000000;
 DELETE 1000000
 TIME: 2088.318 ms

EXPLAIN analyse  INSERT INTO T2 (c1,c2) (SELECT  *,md5(i::text) FROM (SELECT * FROM generate_series(1,1000000)) i);
 QUERY PLAN
 ------------------------------------------------------------------------------------------------------------------------------------
 INSERT ON t2  (cost=0.00..17.50 ROWS=1000 width=36) (actual TIME=3859.265..3859.265 ROWS=0 loops=1)
 ->  FUNCTION Scan ON generate_series  (cost=0.00..17.50 ROWS=1000 width=36) (actual TIME=169.826..1845.421 ROWS=1000000 loops=1)
 Planning TIME: 0.054 ms
 TRIGGER FOR CONSTRAINT t2_c1_fkey: TIME=14600.258 calls=1000000
 Execution TIME: 18558.108 ms
 (5 ROWS)

TIME: 18558.386 ms
 commit;
 COMMIT
 TIME: 8.563 ms
 ```

Le moteur va effectuer les vérifications à chaque insertion (environ 18 secondes à chaque insertion).

Effectuons la même opération en retardant la vérification des contraintes :

```SQL
BEGIN;
 BEGIN
 TIME: 0.130 ms

SET CONSTRAINTS ALL deferred ;
 SET CONSTRAINTS

EXPLAIN analyse  INSERT INTO T2 (c1,c2) (SELECT  *,md5(i::text) FROM (SELECT * FROM generate_series(1,1000000)) i);

QUERY PLAN
 ------------------------------------------------------------------------------------------------------------------------------------
 INSERT ON t2  (cost=0.00..17.50 ROWS=1000 width=36) (actual TIME=3241.172..3241.172 ROWS=0 loops=1)
 ->  FUNCTION Scan ON generate_series  (cost=0.00..17.50 ROWS=1000 width=36) (actual TIME=169.770..1831.893 ROWS=1000000 loops=1)
 Planning TIME: 0.096 ms
 Execution TIME: 3269.624 ms
 (4 ROWS)

TIME: 3270.018 ms

DELETE FROM t2 WHERE c1 <= 1000000;
 DELETE 2000000
 TIME: 2932.070 ms

EXPLAIN analyse  INSERT INTO T2 (c1,c2) (SELECT  *,md5(i::text) FROM (SELECT * FROM generate_series(1,1000000)) i);
 QUERY PLAN
 ------------------------------------------------------------------------------------------------------------------------------------
 INSERT ON t2  (cost=0.00..17.50 ROWS=1000 width=36) (actual TIME=3181.294..3181.294 ROWS=0 loops=1)
 ->  FUNCTION Scan ON generate_series  (cost=0.00..17.50 ROWS=1000 width=36) (actual TIME=170.137..1811.889 ROWS=1000000 loops=1)
 Planning TIME: 0.055 ms
 Execution TIME: 3209.712 ms
 (4 ROWS)

TIME: 3210.067 ms

commit;
 COMMIT
 TIME: 16630.155 ms
 ```

Les insertions sont plus rapides, en revanche le commit est plus long car le moteur effectue la vérification des contraintes. Au final, le moteur effectue une seule vérification à la fin de la transaction (commit). L'opération est donc plus rapide.

Il est possible de créer la contrainte avec un autre attribut `DEFERRABLE INITIALLY DEFERRED` qui permet de s'affranchir du `SET CONSTRAINTS ALL DEFERRED`. A noter également que le mot clef ALL peut-être remplacé par le nom d'une contrainte (si elle est nommée et est `DEFERRABLE`)

Si les enregistrements modifiés ne respectent pas les contraintes, la transaction est annulée au moment du commit :

```SQL
SELECT * FROM t1;
 c1 | c2
 ----+----
 1 | un
 (1 ROW)

BEGIN;
 SET constraints ALL deferred ;
 SET CONSTRAINTS
 anayrat=# INSERT INTO t2 VALUES ('2','un');
 INSERT 0 1
 anayrat=# commit;
 ERROR:  INSERT OR UPDATE ON TABLE "t2" violates FOREIGN KEY CONSTRAINT "t2_c1_fkey"
 DETAIL:  KEY (c1)=(2) IS NOT present IN TABLE "t1".
 ```

D'autre part, la solution de désactivation des contraintes peut effectivement poser des problèmes si on souhaite les réactiver et que les données ne le permettent pas (contrainte effectivement rompue), cette solution reste possible, au sein d'une transaction, **toutefois** cela provoque un verrouillage exclusif sur la table modifiée pendant toute la transaction ce qui peut poser de sérieux problèmes de performance.

Il est également possible de déclarer la contrainte en `NOT VALID`. La création de la contrainte sera quasi immédiate, les données actuellement présentes ne seront pas validées. Cependant, toutes les données insérées ou mises à jour par la suite seront validées vis à vis de ces contraintes.

Ensuite on peut demander au moteur de faire la vérification des contraintes pour l'intégralité des enregistrements avec l'ordre `VALIDATE CONSTRAINT`. Cet ordre entraîne un verrou exclusif sur la table. A partir de la version 9.4 le verrou est plus léger : `SHARE UPDATE EXCLUSIVE` sur la table modifiée. Si la contrainte est une clé étrangère, le verrou est de type `ROW SHARE` sur la table référente.

```SQL
ALTER TABLE t2 DROP CONSTRAINT t2_c1_fkey ;
 ALTER TABLE
 SELECT * FROM t1;
 c1 | c2
 ----+----
 1 | un
 (1 ROW)

SELECT * FROM t2;
 c1 | c2
 ----+----
 (0 ROWS)

INSERT INTO t2 VALUES (2,'deux');
 INSERT 0 1
 SELECT * FROM t2;
 c1 |  c2
 ----+------
 2 | deux
 (1 ROW)

ALTER TABLE t2 ADD CONSTRAINT t2_c1_fkey FOREIGN KEY (c1) REFERENCES t1(c1) NOT VALID;
 ALTER TABLE
 ```

La table t2 contient un enregistrement ne respectant pas la contrainte t2\_c1\_fkey. Aucune erreur n'est remontée car la vérification se fait seulement pour les nouvelles modifications :

```SQL
INSERT INTO t2 VALUES (3,'trois');
 ERROR:  INSERT OR UPDATE ON TABLE "t2" violates FOREIGN KEY CONSTRAINT "t2_c1_fkey"
 DETAIL:  KEY (c1)=(3) IS NOT present IN TABLE "t1".
 ```

De même, une erreur est bien remontée lors de la vérification de contrainte :

```SQL
ALTER TABLE t2 VALIDATE CONSTRAINT t2_c1_fkey ;
 ERROR:  INSERT OR UPDATE ON TABLE "t2" violates FOREIGN KEY CONSTRAINT "t2_c1_fkey"
 DETAIL:  KEY (c1)=(2) IS NOT present IN TABLE "t1".
 ```

En supprimant l'enregistrement posant problème, les contraintes peuvent être validées :

```SQL
DELETE FROM t2 WHERE t2.c1 NOT IN (SELECT c1 FROM t1);
 DELETE 1
 ALTER TABLE t2 VALIDATE CONSTRAINT t2_c1_fkey ;
 ALTER TABLE
 ```
