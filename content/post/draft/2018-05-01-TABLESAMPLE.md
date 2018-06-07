+++
title = "Postgres: GROUPING SETS et TABLESAMPLE"
date = 2018-04-25T14:09:38+02:00
draft = true

summary = "Depuis la version 9.5, Postgres dispose des ordres `TABLESAMPLE` et `GROUPING SETS`."


# Tags and categories
# For example, use `tags = []` for no tags, or the form `tags = ["A Tag", "Another Tag"]` for one or more tags.
tags = ["postgres","TABLESAMPLE"]
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

# TABLESAMPLE

Depuis la version 9.5, PostgreSQL dispose de la clause `TABLESAMPLE`. Cette clause
permet d'avoir une estimation du résultat de la requête en l'exécutant sur un
échantillon plutôt que sur l'intégralité de la table.

Comme en statistiques, plus l'échantillon est important, plus le résultat sera
proche du résultat réel.

Voici la syntaxe:

```sql
```


Il existe différentes options controlant la méthode d'échantillonnage.

La première, consiste à choisir 

La seconde,

# GROUPING SETS

Les `GROUPING SETS` correspondent à plusieurs clauses permettant de réaliser
des aggrégations multiples.

# Association des deux ordres


