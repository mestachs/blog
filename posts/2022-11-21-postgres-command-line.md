---
title: "Postgres from the command line"
date: 2022-11-21T20:31:44+01:00
draft: true
tags: ["postgres", "cli"]
---

In this article I wanted to share tools, tricks I use everyday and "command line" oriented

# The basic : psql

Ok that's something you probably know... but why do I need to know `psql` ?
You already ended up on a machine where you only have vi or nano to edit files ?
That's exactly the same here, you have to know a few psql tricks to survive there too.

## DATABASE_URL

First if you have environnement variable like the `DATABASE_URL` (à la heroku) you don't have to split the url into its components to run psql against the db

```
psql postgresql://user:password@host.zone.rds.amazonaws.com:5432/demo
```

Note that pg_dump also support that but a bit differently

```
pg_dump -Fc -v --no-acl --no-owner --dbname=$DATABASE_URL -f myapp.dump
```

## Finding your way

To discover the databases, tables,... you have plenty of commands like these

```
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

## Pager : more is less

You can disable the pager (less)

Inside psql

```
\pset pager 0
```

Before launching psql

```
export PAGER=
psql
```

but another cool thing is replacing the default PAGER `less` with [pspg](https://github.com/okbob/pspg)

![pspg](https://raw.githubusercontent.com/okbob/pspg/master/screenshots/pspg-4.3.0-mc-export-111x34.png)

This open a lot of possibilities (navigation, search, export)

## Getting further with psql

There are plenty of articles on the web to learn a lot of things about psql

- [cheatsheets](https://gist.github.com/philippetedajo/91341cb4d14c7b07e381d70839acf0f5)
- [psql tips](https://psql-tips.org/psql_tips_all.html) (100+ tips)
- [psql tips](https://pgdash.io/blog/postgres-psql-tips-tricks.html) (timings, null,...)

# The coolkid : pgcli

Friends don't let their friends write join conditions... tell them to use pgcli autocompletion ;)

Think of [pgcli](https://www.pgcli.com/) as a psql on steroid, exploring the schema/data is 200% easier (Some stuff are still better in psql, ex dumping a lot of data). It's a python package easy to install (once you know a bit of python).

The autocomplete is just great, pgcli will proposed you join conditions based on foreign key constraints.

![pgcli](https://user-images.githubusercontent.com/371692/203144054-89a09739-51d5-452d-a9be-ebdb2ee727e2.png)

Want a `heroku psql -a ...` but for launching pgcli ? Add this bash function in your .bashrc

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

You'll need to config changes to make `pspg` work with pgcli : https://github.com/okbob/pspg#pgcli

# Gaining visibility

If you connect your self to the production database, it's probably that something isn't working as expected.
Let's see our options here.

## pg_stat_activity

My app is super slow, my ruby/python/java process just blow up...
You probably want to answer the question 'what is running right now in postgres' ?

Want a cool trick to take csv snapshots of the queries ?

```
psql -h crt.sh -p 5432 -U guest certwatch -c "COPY (select * from pg_stat_activity) TO STDOUT WITH CSV HEADER"
```

In my current work, I've combined a few tools to get a feeling of "what's running" when we have problems :

- thread stack (jstack, signal for ruby process,...), logs, http logs, top
- the queries on postgres

and generally this give a few supsects to investigate (no need of pricey apm ;)).

## pg_stat_statements

Ok you missed it ? May be you should keep some planning and execution statitics and analyze them afterwards.

[pg_stat_statements](https://www.postgresql.org/docs/current/pgstatstatements.html) is the answer.

Enabling it a bit of work but clearly worth it

- install the postgres-contrib package (if necessary, it's pre installed on rds)
- change `shared_preload_libraries` and `track_activity_query_size` to load the libraries
  - either in postgres.conf
  - or via parameter group in AWS RDS (that the less fun part, you need a custom parameter group for each version of postgres you want to support)
- create the extension
- profit
  - find queries
  - reset/replay the use case to find the sqls emitted

If you want to go further check the [crunchydata article](https://www.crunchydata.com/blog/tentative-smarter-query-optimization-in-postgres-starts-with-pg_stat_statements)

## Ok that's nice but do you have a tool like top ?

Tired of running, refreshing your psql prompt to hunt the bad sql ?
No problem there's a top like tool called [pg_activity](https://github.com/dalibo/pg_activity) (note that it plays well with rds)
He will refresh regularly, allow to cancel or kill selected backend connections.

At the top of the screen you will get a sense of the load on the db then bellow you get a list of the running queries.

![pgtop]https://camo.githubusercontent.com/bff0aaefca67fc68d4b3655403a5a70d730990080062e736acae84107feb7aeb/68747470733a2f2f7261772e6769746875622e636f6d2f64616c69626f2f70675f61637469766974792f6d61737465722f646f63732f696d67732f73637265656e73686f742e706e67

A mysql user ? Check [mytop](https://github.com/jzawodn/mytop)
