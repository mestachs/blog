---
title: "Postgres from the command line"
date: 2022-11-21T20:31:44+01:00
draft: true
tags: ["postgres", "cli"]
---

In this article I wanted to share tools, tricks I use everyday around postgres and especially "command line" oriented.

# The basic : psql

Ok that's something you probably know... but why do I need to know `psql` ?
You already ended up on a machine where you only have vi or nano to edit files ?
That's exactly the same here, you have to know a few psql tricks to survive there too.
In production you often have limited/restricted access, these command line knowledge are a gift to get you out of troubles.
The last years , I've pushed this further, I don't have Rich/UI db client on my laptop either.

## Get a connection : DATABASE_URL

While psql allows to pass host, port, databasename, user, password... most of the applications I host have a url to connect the database.
Switching from one form to the other is always painful : host is `-H` or `-h` ? and `-p` is for `password` or `port` ?
If you have environnement variable like the `DATABASE_URL` (à la heroku) you don't have to split the url into its components to run psql against the db

```
psql postgresql://user:password@host.zone.rds.amazonaws.com:5432/demo
```

Note that pg_dump also support that but a bit differently

```
pg_dump -Fc -v --no-acl --no-owner --dbname=$DATABASE_URL -f myapp.dump
```

## Find your way

To discover the databases, tables,... you have plenty of commands like these

```
\?              help with most 'backslash' commands
\l+             listing databases
\d+             listing tables and views
\d+ <<table>>   listing columns, indexes, foreign keys constraints, partitions, triggers
```

Example output of details about a table

```
                                             Partitioned table "public.certificate"
    Column    |  Type   | Collation | Nullable |                 Default                 | Storage  | Stats target | Description
--------------+---------+-----------+----------+-----------------------------------------+----------+--------------+-------------
 id           | bigint  |           | not null | nextval('certificate_id_seq'::regclass) | plain    |              |
 issuer_ca_id | integer |           | not null |                                         | plain    |              |
 certificate  | bytea   |           | not null |                                         | extended |              |
Partition key: RANGE (COALESCE(x509_notafter(certificate), 'infinity'::timestamp without time zone))
Indexes:
    "c_ica_canissue" btree (issuer_ca_id) WHERE x509_canissuecerts(certificate)
    "c_ica_notafter" btree (issuer_ca_id, COALESCE(x509_notafter(certificate), 'infinity'::timestamp without time zone))
    "c_ica_notbefore" btree (issuer_ca_id, x509_notbefore(certificate))
    "c_id" btree (id)
    "c_identities" gin (identities(certificate))
    "c_pubkey_md5" btree (x509_publickeymd5(certificate))
    "c_serial" btree (x509_serialnumber(certificate))
    "c_sha1" btree (digest(certificate, 'sha1'::text))
    "c_sha256" btree (digest(certificate, 'sha256'::text))
    "c_ski" btree (x509_subjectkeyidentifier(certificate))
    "c_spki_sha1" btree (digest(x509_publickey(certificate), 'sha1'::text))
    "c_spki_sha256" btree (digest(x509_publickey(certificate), 'sha256'::text))
    "c_subject_sha1" btree (digest(x509_name(certificate), 'sha1'::text))
Foreign-key constraints:
    "c_ica_fk" FOREIGN KEY (issuer_ca_id) REFERENCES ca(id)
Triggers:
    cert_counter AFTER DELETE OR UPDATE ON certificate FOR EACH ROW EXECUTE FUNCTION cert_counter()
Partitions: certificate_2013andbefore FOR VALUES FROM (MINVALUE) TO ('2014-01-01 00:00:00'),
            certificate_2014 FOR VALUES FROM ('2014-01-01 00:00:00') TO ('2015-01-01 00:00:00'),
            certificate_2015 FOR VALUES FROM ('2015-01-01 00:00:00') TO ('2016-01-01 00:00:00'),
            certificate_2016 FOR VALUES FROM ('2016-01-01 00:00:00') TO ('2017-01-01 00:00:00'),
            certificate_2017 FOR VALUES FROM ('2017-01-01 00:00:00') TO ('2018-01-01 00:00:00'),
            certificate_2018 FOR VALUES FROM ('2018-01-01 00:00:00') TO ('2019-01-01 00:00:00'),
            certificate_2019 FOR VALUES FROM ('2019-01-01 00:00:00') TO ('2020-01-01 00:00:00'),
            certificate_2020 FOR VALUES FROM ('2020-01-01 00:00:00') TO ('2021-01-01 00:00:00'),
            certificate_2021 FOR VALUES FROM ('2021-01-01 00:00:00') TO ('2022-01-01 00:00:00'),
            certificate_2022 FOR VALUES FROM ('2022-01-01 00:00:00') TO ('2023-01-01 00:00:00'),
            certificate_2023 FOR VALUES FROM ('2023-01-01 00:00:00') TO ('2024-01-01 00:00:00'),
            certificate_2024andbeyond FOR VALUES FROM ('2024-01-01 00:00:00') TO (MAXVALUE)

```

## Don't remember the sql syntax ?

do a `\h` followed by the sql statement (insert, delete, update, alter table) (or `\h if really don't remember anything`)
sadly it's not showing concrete sample queries but it's already good.

```
db=> \h insert
Command:     INSERT
Description: create new rows in a table
Syntax:
[ WITH [ RECURSIVE ] with_query [, ...] ]
INSERT INTO table_name [ AS alias ] [ ( column_name [, ...] ) ]
    [ OVERRIDING { SYSTEM | USER } VALUE ]
    { DEFAULT VALUES | VALUES ( { expression | DEFAULT } [, ...] ) [, ...] | query }
    [ ON CONFLICT [ conflict_target ] conflict_action ]
    [ RETURNING * | output_expression [ [ AS ] output_name ] [, ...] ]

where conflict_target can be one of:

    ( { index_column_name | ( index_expression ) } [ COLLATE collation ] [ opclass ] [, ...] ) [ WHERE index_predicate ]
    ON CONSTRAINT constraint_name

and conflict_action is one of:

    DO NOTHING
    DO UPDATE SET { column_name = { expression | DEFAULT } |
                    ( column_name [, ...] ) = [ ROW ] ( { expression | DEFAULT } [, ...] ) |
                    ( column_name [, ...] ) = ( sub-SELECT )
                  } [, ...]
              [ WHERE condition ]

URL: https://www.postgresql.org/docs/14/sql-insert.html
```

## Find your trails

Want to re-run a previous sql statement ?

A first option is just using the arrow keys to go a few queries back.

The second option is like in a bash terminal : CTRL-r then start filtering by fragment you remember and cycle between matching queries with CTRL-r

I like to keep track of queries and outputs (number of records deleted/updated,...) that I've run on a production server but sadly there's no easy way to wiretype them (the history is not a perfect solution there). For the moment I copy paste them in gists/jira/text files for later reference. Generally some of these end up being reused and more formalised as [runbooks](https://en.wikipedia.org/wiki/Runbook).

If you know such a tool let me know ! (I know you can setup this server side but client would be great for me)

## Pager : more is less

You can disable the pager (generally it defaults to [less](https://man7.org/linux/man-pages/man1/less.1.html))

Inside psql you can type

```
\pset pager 0
```

Or for a whole session before launching psql

```
export PAGER=
psql
```

but another cool thing is replacing the default PAGER `less` with [pspg](https://github.com/okbob/pspg) (Postgres Pager)

![pspg](https://raw.githubusercontent.com/okbob/pspg/master/screenshots/pspg-4.3.0-mc-export-111x34.png)

This open a lot of possibilities (navigation, search, export)

## Getting further with psql

There are plenty of articles on the web to learn a lot of things about psql, a few selected articles

- [cheatsheets](https://gist.github.com/philippetedajo/91341cb4d14c7b07e381d70839acf0f5)
- [psql tips](https://psql-tips.org/psql_tips_all.html) (100+ tips)
- [psql tips](https://pgdash.io/blog/postgres-psql-tips-tricks.html) (timings, null, watch, ...)

# The coolkid : pgcli

Think of [pgcli](https://www.pgcli.com/) as a psql on steroid, exploring the schema/data is 200% easier (Some stuff are still better in psql, ex dumping a lot of data). It's a python package easy to install (once you know a bit of python). Most of the interactive tricks,I showed you, works in pgcli too.

## Autocompletion for the win

Friends don't let their friends write join conditions... tell them to use pgcli autocompletion ;)

The autocomplete is just great, pgcli will propose you join conditions based on foreign key constraints, also auto complete tables and fields names.

![pgcli](https://user-images.githubusercontent.com/371692/203144054-89a09739-51d5-452d-a9be-ebdb2ee727e2.png)

## pgcli and heroku

You want a `heroku psql -a ...` but for launching pgcli ? Add this bash function in your .bashrc

```bash
pgh() {
    pgcli $(heroku config:get DATABASE_URL --app "$1")
}
```

Then just pass the app as parameter

```
pgh myherokuapp
```

Note that if you have other database to support (mysql, mssql) there are other cli tool [here](https://www.dbcli.com/)

## Pager

You'll need some config changes to make `pspg` work with pgcli : https://github.com/okbob/pspg#pgcli

# Gaining visibility

If you connect your self to the production database, it's probably that something isn't working as expected or much slower than usual.
Let's see our options here.

## pg_stat_activity

My app is super slow, my ruby/python/java process just blow up...
You probably want to answer the question 'what is running right now in postgres' ?

Want a cool trick to take csv snapshots of the running queries ?

```
psql -h crt.sh -p 5432 -U guest certwatch -c "COPY (select * from pg_stat_activity) TO STDOUT WITH CSV HEADER" > queries.csv
```

There are 2 small trick here

- using `-c` to pass an sql statement and don't go in interactive/prompt mode.
- using `COPY ( .... ) TO STDOUT WITH CSV HEADER` to generate a csv
  - the STDOUT works well on RDS when you don't have access to the local file system
  - this also generates "robust" csv (other csv tricks often break on escaping delimiters or cariage return or handling null)

The generated file looks like this :

![image](https://user-images.githubusercontent.com/371692/203362136-c014bf02-4506-41e9-b6f3-de7c65d63319.png)

You get the query (sometimes truncated) and the status :

- active (running right now),
- idle (so probaby show the last executed sql with this connection),
- idle in transaction (waiting for other tx to complete, or cpu time)
- other [statuses](https://www.postgresql.org/docs/current/monitoring-stats.html#MONITORING-PG-STAT-ACTIVITY-VIEW)

In my current work, I've combined a few tools to get a feeling of "what's running" when we have problems :

- thread stack (jstack, signal for ruby process,...), logs, http logs, top
- the queries on postgres

and generally this give a few supsects to investigate (no need of pricey apm ;)).

## pg_stat_statements

Ok you missed the query with pg_stat_activity or it's getting too fast to collect or it's in fact 1000 times the same sql ? May be you should keep some planning and execution statitics and analyze them afterwards.

[pg_stat_statements](https://www.postgresql.org/docs/current/pgstatstatements.html) is the answer.

Enabling it a bit of work but clearly worth it

- install the postgres-contrib package (if necessary, it's pre installed on rds)
- change `shared_preload_libraries` and `track_activity_query_size` to load the libraries
  - either in postgres.conf
  - or via parameter group in AWS RDS (that's the less fun part, you need a custom parameter group for each version of postgres you want to support)
- create the extension
- profit
  - find queries
    - most time consuming (# of excution x cpu time)
    - slow queries more (running more than 5 seconds)
    - most executed against nearly static data (cache opportunity at app level)
  - reset/replay the use case to find the sqls emitted

If you want to go further check the [crunchydata article](https://www.crunchydata.com/blog/tentative-smarter-query-optimization-in-postgres-starts-with-pg_stat_statements)

## Explain plan visualizer

You have spotted a slow query ? You can now run an explain plan !

```
psql $DATABASE_URL -qAtc "EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) select * from iaso_instance join iaso_form ON iaso_form.id = iaso_instance.form_id where iaso_form.name ilike '%admin%' limit 10" | pbcopy
```

This will generate the explain plan of the query in quotes and copy it in the clipboard (assuming you have [xclip](https://garywoodfine.com/use-pbcopy-on-ubuntu/) and pbcopy alias)

Next step is to paste it on this site : https://tatiyants.com/pev/#/plans/new (no server side parsing/rendering, everything is parsed/stored/rendered in your browser)

Tada ! Thanks to [AlexTatiyants](https://twitter.com/AlexTatiyants) you have nice explain plan.

![image](https://user-images.githubusercontent.com/371692/204056951-6bf8ff3e-f34b-4b1e-9acb-46be5b0cb93d.png)

What to look for ?

- `seq scan` on large tables (an index might help ?)
- cartesian join
- really bad row estimates (`ANALYZE` needed ?)

Note that you can use other sites like :

- https://explain.depesz.com/ or
- https://explain.dalibo.com/ but they stored your info serverside, (note dalibo seem to provide a local install too https://github.com/dalibo/pev2)
- https://flame-explain.com/visualize/input this looks great also offering a flamegraph rendering

## Ok that's nice but do you have a tool like top ?

Tired of running, refreshing your psql prompt to hunt the bad sql ? Want to gain visibility when restoring a backup ?

No problem there's a top like tool called [pg_activity](https://github.com/dalibo/pg_activity) (note that it plays well with rds)
He will refresh regularly, allows to cancel or kill selected backend connections.

At the top of the screen you will get a sense of the load on the db then bellow you get a list of the running queries.

![pgtop](https://camo.githubusercontent.com/bff0aaefca67fc68d4b3655403a5a70d730990080062e736acae84107feb7aeb/68747470733a2f2f7261772e6769746875622e636f6d2f64616c69626f2f70675f61637469766974792f6d61737465722f646f63732f696d67732f73637265656e73686f742e706e67)

A mysql user ? Check [mytop](https://github.com/jzawodn/mytop)

# Nice graphs ?

This is more for fun than profit but why not ;)
It's more about csv then postgres but this can help visualize things.
Note that some people do generate graphs from [sql](https://blog.jooq.org/how-to-plot-an-ascii-bar-chart-with-sql)

## Termgraph

Can you generate a graph ? Well sort of, piping the data into [termgraph](https://github.com/mkaz/termgraph) (without the csv headers)

```
psql $DATABASE_URL -c "copy (select EXTRACT(YEAR FROM created), count(*) from datavalue group by EXTRACT(YEAR FROM created)) TO STDOUT WITH CSV;" | termgraph
```

will output the year number of datavalues created that year

```
2019: ▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇ 924.29K
2020: ▏ 925.00
2021: ▏ 502.00
2022: ▏ 10.00
```

## Sparklines

I'm not impressed what about sparklines ? No problem a bit of cut and pipe it to [spark](https://github.com/holman/spark)

```
psql $DATABASE_URL -c "copy (select EXTRACT(YEAR FROM created), count(*) from datavalue group by EXTRACT(YEAR FROM created) order by 1 asc)  TO STDOUT WITH CSV;" | cut -d, -f2 | spark
```

you'll get something nice : `█▁▁▁`

## A bit unrelated, ascii art in slack

Just for your info and completely unrelated to postgres, I'm using such technique to render the downtime of our servers in slack ;)

:warning: `▅▇▅▆▇▇▇▇▇▆▇▇▇▇▇▇▇▇▇▇▇▇▂▁`

You might want to have a look at other charts solution from the command line

- [gnuplot](https://gist.github.com/darencard/ffc850949a53ff578260c8b7d3881f28)
- [asciichart](https://github.com/kroitor/asciichart)

# Conclusion

I hope, I gave you a sense of the power the cli. And how to diagnose performance problem without heavy tooling.

Note that once you have started using the cli everyday, this opens also a lot of options to automate things around your database or development environment.

I'll probably write more about command line and data ingestion in a futur article.
