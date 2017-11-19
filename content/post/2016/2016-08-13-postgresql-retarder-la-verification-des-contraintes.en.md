---
title: 'PostgreSQL : Deferrable constraints'
author: Adrien Nayrat
type: post
date: 2016-08-13T20:42:53+00:00
draft: false
categories:
  - Postgres
tags:
  - postgres

---
# Differ constraints verification

> Note: This article was written during my activity at [Dalibo](https://www.dalibo.com)

Postgres respects [ACID](https://en.wikipedia.org/wiki/ACID) properties, so it guarantees the consistency of the database: a transaction will bring the database from one valid state to another.

The data in the different tables is not independent but obeys semantic rules put in place when designing the conceptual model. The main purpose of integrity constraints is to ensure the consistency of the data between them, and therefore to ensure that they respect these semantic rules. If an insert, an update or a delete violates these rules, the transaction is purely and simply canceled.

Postgres performs constraint verification on each change (when constraints have been defined). It is also possible to delay the checking of the constraints at the end of the transaction, at the time of the commit. Thus, the verifications will only be produced on the effective changes between the delete, update and insert operations of the whole transaction.


<!--more-->

Example:

```SQL
CREATE TABLE t1 (c1 INT PRIMARY KEY, c2 text);
 CREATE TABLE t2 (c1 INT REFERENCES t1(c1), c2 text);

INSERT INTO t1  VALUES(1,'a');
 INSERT INTO t1  VALUES(2,'b');

INSERT INTO t2  VALUES(3,'a');
 ERROR:  INSERT OR UPDATE ON TABLE "t2" violates FOREIGN KEY CONSTRAINT "t2_c1_fkey"
 DETAIL:  KEY (c1)=(3) IS NOT present IN TABLE "t1".
 ```

The line inserted in t2 must respect the foreign key constraint, t1 does not contain any line where c1 = 3. Let's insert a correct line:

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

What happens if you want to change the primary key of t1?

```SQL
BEGIN;
 UPDATE t1 SET c1=3 WHERE c1=1;
 ERROR:  UPDATE OR DELETE ON TABLE "t1" violates FOREIGN KEY CONSTRAINT "t2_c1_fkey" ON TABLE "t2"
 DETAIL:  KEY (c1)=(1) IS still referenced FROM TABLE "t2".
 ```

Constraint check is done during the UPDATE and triggers an error. It is possible to ask postgres to perform the constraint check at the end of the transaction with the order `SET CONSTRAINTS ALL DEFERRED;`. Also note that the keyword ALL can be replaced by the name of a constraint (if it is named and is `DEFERRABLE`).

```SQL
BEGIN;
 SET CONSTRAINTS ALL DEFERRED;
 UPDATE t1 SET c1=3 WHERE c1=1;
 ERROR:  UPDATE OR DELETE ON TABLE "t1" violates FOREIGN KEY CONSTRAINT "t2_c1_fkey" ON TABLE "t2"
 DETAIL:  KEY (c1)=(1) IS still referenced FROM TABLE "t2".
 ```
It still does not work, in fact it must be specified that the application of the constraint can be delayed with the keyword `DEFERRABLE`

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

 In this case postgres accepts to do the verification at the end of the transaction.

 Other interest, if a line is erased and reinserted in the same transaction, the checks on this line are not executed (because useless).

 Example, we truncate tables and insert 1 million rows.


```SQL
TRUNCATE t1 cascade;
 NOTICE:  TRUNCATE cascades TO TABLE "t2"
 TRUNCATE TABLE

EXPLAIN analyse INSERT INTO T1 (c1,c2) (SELECT  *,md5(i::text) FROM (SELECT * FROM generate_series(1,1000000)) i);
```

We insert 100 000 lines, then we delete them to reinsert them again (without telling the engine to differ the constraint check).


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

Postgres will perform the checks at each insertion (about 18 seconds at each insertion).

Perform the same operation by delaying the constraint check:


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

Inserts are faster, but the commit is longer because postgres performs the constraint check. In the end, postgres performs a single check at the end of the transaction (commit). The operation is faster.

It is possible to create the constraint with another attribute `DEFERRABLE INITIALLY DEFERRED` which allows to get rid of` SET CONSTRAINTS ALL DEFERRED`. Also note that the keyword ALL can be replaced by the name of a constraint (if it is named and is `DEFERRABLE`)

If the modified rows do not respect the constraints, the transaction is canceled at commit time:

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

On the other hand, the solution of disabling the constraints can indeed pose problems if one wishes to reactivate them and that the data does not allow it (constraint actually broken), this solution remains possible, within a transaction, **however** this causes an exclusive lock on the modified table during the whole transaction which can pose serious performance problems.

It is also possible to declare the constraint in `NOT VALID`. The creation of the constraint will be almost immediate, the data currently present will not be validated. However, any data inserted or updated later will be validated against these constraints.

Then we can ask postgres to check the constraints for all records with the order `VALIDATE CONSTRAINT`. This order results in an exclusive lock on the table. As of version 9.4, the lock is lighter: `SHARE UPDATE EXCLUSIVE` on the modified table. If the constraint is a foreign key, the lock is of type `ROW SHARE` on the referencing table.

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

t2 contains a row that does not respect the t2\_c1\_fkey constraint. No error has been reported because the check is only for new changes:

```SQL
INSERT INTO t2 VALUES (3,'trois');
 ERROR:  INSERT OR UPDATE ON TABLE "t2" violates FOREIGN KEY CONSTRAINT "t2_c1_fkey"
 DETAIL:  KEY (c1)=(3) IS NOT present IN TABLE "t1".
 ```

Similarly, an error has been raised during the check:

```SQL
ALTER TABLE t2 VALIDATE CONSTRAINT t2_c1_fkey ;
 ERROR:  INSERT OR UPDATE ON TABLE "t2" violates FOREIGN KEY CONSTRAINT "t2_c1_fkey"
 DETAIL:  KEY (c1)=(2) IS NOT present IN TABLE "t1".
 ```

By deleting the problematic record, the constraints can be validated:

```SQL
DELETE FROM t2 WHERE t2.c1 NOT IN (SELECT c1 FROM t1);
 DELETE 1
 ALTER TABLE t2 VALIDATE CONSTRAINT t2_c1_fkey ;
 ALTER TABLE
 ```
