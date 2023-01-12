---
title: "Postgres : archive old audit records"
date: 2022-12-28T07:58:44+01:00
draft: false
tags: ["postgres", "historize", "archive"]
---

# The problem : my audit tables grows forever

Most applications have audit/change logs containing

- which data was changed
- by which user and sometimes
- the previous/new/diffed content of it and
- when.

These tables grows as time goes by, as more users use the system or background jobs ingest data.

After years, to fetch the "latest changes" in the application, you can't query these tables without either

- upgrading your server to bigger instances or
- creating partial index
- partitioning them in several tables
- sharding

and sometimes you need the 4 of them to keep things smooth.

Most of these old records are probably useless except for legal requirement or exceptionnaly checking who did that in 2018.

It would be great to be able to keep these audit table to a reasonable size and move these older records in other table or even better on a cheaper long term storage.

# The good news : sql can move records !

The good news is that these records have tons of foreign key to other tables but are rarely referenced by other tables. They are a perfect candidate for the technique, I'll describe here.

To show a concrete exemple, I'll take the `datavalueaudit` of [dhis2](https://dhis2.org/). Which is open source software generating tons of audit entry each time you update a value in the system (1 value in a form = 1 line in this table), after years of usage you end up with millions of records.

To have you a sense of how the table looks like let's run

```
 \d+ datavalueaudit
```

The table looks like this :

```
"Column","Type","Modifiers","Storage","Stats target","Description"
"datavalueauditid","bigint"," not null","plain","<null>","<null>"
"dataelementid","bigint"," not null","plain","<null>","<null>"
"periodid","bigint"," not null","plain","<null>","<null>"
"organisationunitid","bigint"," not null","plain","<null>","<null>"
"categoryoptioncomboid","bigint"," not null","plain","<null>","<null>"
"attributeoptioncomboid","bigint"," not null","plain","<null>","<null>"
"value","character varying(50000)","","extended","<null>","<null>"
"modifiedby","character varying(100)","","extended","<null>","<null>"
"audittype","character varying(255)"," not null","extended","<null>","<null>"
"created","timestamp without time zone","","plain","<null>","<null>"
Indexes:
    "datavalueaudit_pkey" PRIMARY KEY, btree (datavalueauditid)
    "id_datavalueaudit_created" btree (created)
    "in_datavalueaudit" btree (dataelementid, periodid, organisationunitid, categoryoptioncomboid, attributeoptioncomboid)
Foreign-key constraints:
    "fk_datavalueaudit_attributeoptioncomboid" FOREIGN KEY (attributeoptioncomboid) REFERENCES categoryoptioncombo(categoryoptioncomboid)
    "fk_datavalueaudit_categoryoptioncomboid" FOREIGN KEY (categoryoptioncomboid) REFERENCES categoryoptioncombo(categoryoptioncomboid)
    "fk_datavalueaudit_dataelementid" FOREIGN KEY (dataelementid) REFERENCES dataelement(dataelementid)
    "fk_datavalueaudit_organisationunitid" FOREIGN KEY (organisationunitid) REFERENCES organisationunit(organisationunitid)
    "fk_datavalueaudit_periodid" FOREIGN KEY (periodid) REFERENCES period(periodid)
Has OIDs: no
```

## First get an idea of number of records

Issuing a count statement is probably too heavy, you might want just an estimated count of the number of records. This can be done in various ways depending on your [situation](https://stackoverflow.com/a/7945274/613936)

If we wanted to be cautious

```
 SELECT reltuples::bigint AS estimate FROM   pg_class WHERE  oid = 'public.datavalueaudit'::regclass;
┌───────────┐
│ estimate  │
├───────────┤
│ 128150552 │
│ SELECT 1  │
└───────────┘
(2 rows)
Time: 0.035s
```

Let's be crazy and do a count

```
select count(*) from datavalueaudit;
┌───────────┐
│   count   │
├───────────┤
│ 129381221 │
│ SELECT 1  │
└───────────┘
(2 rows)
Time: 86.390s (1 minute 26 seconds), executed in: 86.390s (1 minute 26 seconds)
```

So you the number are not 100% the same, the estimated count is bit behind the reality.
So we have `129.381.221` records in there (And I've already historized some past data 1 year ago)

Ok you might be interested to know the size of it on disk

```
SELECT pg_size_pretty( pg_total_relation_size('datavalueaudit') );
┌────────────────┐
│ pg_size_pretty │
├────────────────┤
│ 26 GB          │
└────────────────┘
```

In my case I was interested in archiving by year the audit records, so let's have an idea of the distribution by year.

```
select extract(year from created) as yyyy, count(datavalueauditid) as number_of_records from datavalueaudit group by extract(year from created)
┌──────────┬───────────────────┐
│   yyyy   │ number_of_records │
├──────────┼───────────────────┤
│ 2018.0   │           4615646 │
│ 2019.0   │          72338936 │
│ 2020.0   │          24493392 │
│ 2021.0   │          10792465 │
│ 2022.0   │          17140804 │
└──────────┴───────────────────┘
(6 rows)
Time: 90.877s (1 minute 30 seconds), executed in: 90.877s (1 minute 30 seconds)
```

## Now let's move

Ok we are nearly ready to archive but first make a backup ! (rds snapshot, pgdump, whatever you have in your infra)

So let's say we want to move all the entries created before (including) 2019.

So a first step is to create an empty table that is defined exactly with the same columns as our audit table

```
CREATE TABLE datavalueaudit_history_2019 (LIKE datavalueaudit INCLUDING ALL);
```

Postgres is a super nice offering a syntax that allows to delete records and return the deleted records.

```
WITH moved_rows AS (
    DELETE FROM datavalueaudit a
    WHERE a.created <= '2019-12-31 23:59:59.000'
    RETURNING a.*
)
INSERT INTO datavalueaudit_history_2019
SELECT * FROM moved_rows;
```

Depending on your where clause and the data in the table this might take a few seconds or several minutes or forever.

## And now archive the data to another cheaper storage

One option might be to just the dump as sql the new `datavalueaudit_history_2019` table.

```
pg_dump --data-only -v --no-acl --no-owner --table datavalueaudit_history_2019 --dbname=$DATABASE_URL -f dhis2-datavalueaudit_history_2019.sql
```

Check the content, verify warning/error messages.

Another option might could have been to dump the table as a csv or other queryable format.
Pick a "stable and humanreadable" format that would survive several years (and versions of that format).

Once the file in a safe place (or multiple places like s3, glacier,... ) then you can drop `datavalueaudit_history_2019` table.

## That's it !

You can repeat that process for other years (2020, 2021)

Ok but did we reclaimed some storage in postgres ?

Well not yet. A good idea might be to run `VACCUUM FULL, ANALYZE, REINDEX` to reclaim part of it. Not that some of these commands are blocking access to the table, you might need to find a quiet period to run them.

After that querying this huge table (well a bit smaller now) should be less heavier for the system.

# You plan to do on your infra/table ?

- test this procedure localy
  - check there's no foreign key issue
  - check your app is still functioning
- then on a test/staging infrastructure
- each time plan for storage to hold the data, dump file

Hope you enjoyed this article ! See you here, on twitter, mastodon or irl.
