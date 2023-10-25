---
title: Backup, restauration – Part 6
authors: ['adrien']
type: post
date: 2015-02-26T17:30:51+00:00
aliases: /2015/02/26/backup-restauration-pitr-part-6/
categories:
  - Linux
  - Postgres
tags:
  - linux
  - postgres
show_related: true

---
J'ai consacré les précédents articles aux différents mécanismes de réplications. Le principe est à chaque fois le même, on utilise une copie d'une instance Postgres puis la réplication se fait par rejeu des journaux de transaction (soit par transfert de fichier ou par flux).

Cet article va aborder les différentes technique de sauvegarde ... ainsi que la restauration. On ne pense pas assez à la restauration : « la priorité est la sauvegarde, la restauration on verra quand on aura un peu de temps ».

  1. Sauvegarde fichier d'une instance (manuelle + pg\_base\_backup)
  2. Sauvegarde par dump des bases avec pg_dump
  3. Restauration (manuelle + pg_restore)
  4. Restauration PITR

<!--more-->

# Sauvegarde fichier d'une instance

La sauvegarde la plus simple et la plus basique est la sauvegarde à froid des fichier d'une instance. Il suffit d'arrêter postgres et de faire une copie du dossier data de postgres. Cette solution a l'inconvénient majeur de devoir arrêter Postgres. Nous allons voir d'autres solutions plus élaborées qui évitent d'éteindre le moteur.

# Sauvegarde par dump des bases avec pg\_dump et pg\_dumpall

Pour éviter d'éteindre le serveur on peut tout simplement faire un dump de la base de donnée avec pg\_dump. Je ne vais pas recopier la page de documentation de pg\_dump ici mais me contenter de vous présenter les options utiles de l'outil.

L'inconvénient ou l'avantage de cet outil est qu'il ne sauvegarde pas toutes les bases d'une instance, il faut donc scripter le tout pour qu'il sauvegarde base par base. J'ai écrit « inconvénient ou avantage », en effet vous pouvez décider de ne restaurer qu'une seule base ou avoir des politique de sauvegarde différentes en fonction des bases. Si vous souhaitez sauvegarder toutes les bases d'un coup vous pouvez utiliser pg\_dumpall. Attention, avec pg\_dump vous ne sauvegardez pas les rôles donc en cas de restauration vous devriez recréer les rôles. Pour y remédier vous pouvez utiliser -roles-only.

Par défaut, pg_dump va sauvegarder les tables, les données de schéma. Il est possible de les sélectionner avec les options : `--data-only` ou `--schema-only`

Petite astuce, à partir de la version 9.3 pg_dump peut paralléliser la sauvegarde des tables avec l'option `--jobs=njobs`

Pour approfondir :

<http://docs.postgresqlfr.org/9.3/app-pgdump.html>

<http://www.dalibo.org/_media/pgdayfr-postgresql_9.3-start.pdf>

# Restauration depuis un dump

## pgsql

La technique la plus simple consiste à envoyer à psql les commandes issues du dump. Voici mon dump :

```SQL
pg_dump -h /var/run/postgresql/ test

--
-- PostgreSQL database dump
--

SET statement_timeout = 0;
SET lock_timeout = 0;
SET client_encoding = 'UTF8';
SET standard_conforming_strings = on;
SET check_function_bodies = false;
SET client_min_messages = warning;

--
-- Name: plpgsql; Type: EXTENSION; Schema: -; Owner:
--

CREATE EXTENSION IF NOT EXISTS plpgsql WITH SCHEMA pg_catalog;


--
-- Name: EXTENSION plpgsql; Type: COMMENT; Schema: -; Owner:
--

COMMENT ON EXTENSION plpgsql IS 'PL/pgSQL procedural language';


SET search_path = public, pg_catalog;

SET default_tablespace = '';

SET default_with_oids = false;

--
-- Name: films; Type: TABLE; Schema: public; Owner: postgres; Tablespace:
--

CREATE TABLE films (
    code character(5),
    title character varying(40),
    did integer,
    date_prod date,
    kind character varying(10),
    len interval hour to minute
);


ALTER TABLE public.films OWNER TO postgres;

--
-- Data for Name: films; Type: TABLE DATA; Schema: public; Owner: postgres
--

COPY films (code, title, did, date_prod, kind, len) FROM stdin;
bla     rambo   12      2014-12-01      action  01:00:00
\.


--
-- Name: production; Type: CONSTRAINT; Schema: public; Owner: postgres; Tablespace:
--

ALTER TABLE ONLY films
    ADD CONSTRAINT production UNIQUE (date_prod);


--
-- Name: public; Type: ACL; Schema: -; Owner: postgres
--

REVOKE ALL ON SCHEMA public FROM PUBLIC;
REVOKE ALL ON SCHEMA public FROM postgres;
GRANT ALL ON SCHEMA public TO postgres;
GRANT ALL ON SCHEMA public TO PUBLIC;


--
-- PostgreSQL database dump complete
--
```

Comme vous pouvez le constater, pg_dump utilise des « copy » plutôt que des « insert ». La commande copy est bien plus rapide pour l'import en masse de données.

Pour restaurer le dump :

```
psql -h /var/run/postgresql/ test -f dump.sql
SET
SET
SET
SET
SET
SET
CREATE EXTENSION
COMMENT
SET
SET
SET
CREATE TABLE
ALTER TABLE
ALTER TABLE
REVOKE
REVOKE
GRANT
GRANT
```

Pour approfondir :

[Documentation de pg_dump](http://www.postgresql.org/docs/9.3/static/app-pgdump.html)

[Documentation de pg_dumpall](http://www.postgresql.org/docs/9.3/static/app-pg-dumpall.html)

## pg_restore

Cette restauration est assez simple, postgres fourni l'utilitaire pg_restore qui offre des fonctionnalités intéressantes telles que :

  * Parallélisation
  * Sélection des sections à restaurer. Très utile si vous ne souhaitez restaurer qu'une seule table.

Pour pouvoir utiliser pg\_restore il faut spécifier le format « custom » à pg\_dump: « -format=custom » ou « -F c ». En effet si vous fournissez un fichier au format « à plat » vous aurez le message suivant :

```
pg_restore: [archiveur] Le fichier en entrée semble être une sauvegarde au format texte. Merci d'utiliser psql.
```

Je ne vais pas recopier la documentation de pg_restore mais juste vous fournir les options les plus intéressantes :

Parallélisation :

```
-j number-of-jobs
--jobs=number-of-jobs
```

Lister les sections du dump :

```
-l
 --list
 ```

Exemple :

```
; Archive created at Sun Jan 11 12:40:21 2015
;     dbname: test
;     TOC Entries: 11
;     Compression: -1
;     Dump Version: 1.12-0
;     Format: CUSTOM
;     Integer: 4 bytes
;     Offset: 8 bytes
;     Dumped from database version: 9.3.5
;     Dumped by pg_dump version: 9.3.5
;
;
; Selected TOC Entries:
;
1939; 1262 16486 DATABASE - test postgres
6; 2615 2200 SCHEMA - public postgres
1940; 0 0 COMMENT - SCHEMA public postgres
1941; 0 0 ACL - public postgres
171; 3079 11756 EXTENSION - plpgsql
1942; 0 0 COMMENT - EXTENSION plpgsql
170; 1259 16502 TABLE public films postgres
1934; 0 16502 TABLE DATA public films postgres
1826; 2606 16506 CONSTRAINT public production postgres
```

Si je souhaite supprimer quelques sections du dump il me suffit de commenter la ligne avec un point-virgule, de sauvegarder le fichier (db.list) puis de lancer pg_restore avec l'option « -L » :

```
pg_restore -L db.list db.dump
```

Pour approfondir : [Documentation de pg_restore](http://www.postgresql.org/docs/9.3/static/app-pgrestore.html)

Nous venons de voir les différentes technique pour restaurer des données. Cette façon de faire présente quelques inconvénients. Il faut faire des backups régulier. On peut avoir une perte de données conséquente depuis le dernier le backup.

En mettant en place une réplication on peut penser que le problème est réglé. Cependant la réplication nous protège d'une perte du maître mais pas d'une erreur humaine. Un DROP TABLE sur une mauvaise base par exemple! Ainsi le [RPO](http://www.journaldunet.com/solutions/systemes-reseaux/analyse/rto-rpo-les-indicateurs-de-securite-en-cas-de-sinistre.shtml) correspond au temps qui s'est écoulé depuis le dernier backup.



# Archivage des WAL + restauration PITR

A venir dans un prochain article.

Edit : Le voici : [Restauration PITR - Part 7](http://blog.anayrat.info/2015/03/22/restauration-pitr-part-7/)
