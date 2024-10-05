---
title: "Data last longer than code"
date: 2024-10-05T07:58:44+01:00
draft: false
tags: ["development", "secrets", "trufflehog"]
---


# Data last longer then your code

If you never ever have mistakes or bugs in the code don't put constraint in your db.

As corollary, put most of the constraint in your db.
Another thing I noticed generally the data stays, not the code. 
From small to big rewrite, db constraint stays, where code checking data consistency might be omitted/forgotten/buggy.

If it's big rewrite, you will be happy once the customer will ask to import this "legacy" data that is not too broken.

Your data will probably drive business decisions, if it's not clean enough, first the datascientist will take most of its time cleaning it, worst the decision will be made on Best Available Data better know by its acronym `BAD`.

# From simple things 

  - use the correct type (a date, jsonb, bigint, geometry)
  - not null 
  - foreign key reference
  - default values
  - enums
  - positive numerics
  - prefer bigint instead of integer for keys 
    - migrating when you are close the MAX integer value or worst when you hit the limit is generally [problematic](https://gitlab.com/gitlab-org/gitlab-foss/-/issues/54445)
    - especially for high volume or high turn over tables (audit logs, background tasks tracking, uploads,...)
    - uuid are even better ;)
  
# To less trivial things like unicity

  - "identifier" (id, uuid, external_id) especially if provided by a client/mobile/external system...
  - generally if it has a 'name' column, you probably want some kind of unicity on it (once displayed in a dropdown which one should I pick ?).  
  - don't trust your orm... `get_or_create(...)` is subject to [race condition](https://medium.com/newton-school/django-a-story-of-race-conditions-with-get-or-create-and-unique-constraints-39b04aa12a54) so put the damned unicity constraint in the db

Fixing __duplicates__ is probably the more costly and touchy part of these bugs.
Identifying/Fixing/Preventing new duplicates can take days or even worst never achieved due to old data. You might end up in situation where tradeoff should be made like "newer records" will be unique and old one might be duplicated.


# Stuff you might not know about postgresql

ORMs are an abstraction over your db, normally they let you acces the specific feature of postgresl

You might be interested to know : 

## You can add checks

```sql
  username VARCHAR(255) NOT NULL CHECK (username <> '')
``` 
see django : https://docs.djangoproject.com/en/5.1/ref/models/constraints/#checkconstraint

## You can have [partial index](https://www.postgresql.org/docs/current/indexes-partial.html) marked as unique 

```sql
CREATE UNIQUE INDEX unique_non_blank_email ON users (email) WHERE email IS NOT NULL AND email <> ''
```
see django : https://docs.djangoproject.com/en/5.1/ref/models/constraints/#id1

## Enum or checks are possible, don't let a typo in your code like `inative` reach your db

```sql 
CHECK (status IN ('active', 'inactive', 'suspended'))`
```
or 
```sql
CREATE TYPE USER_STATUS_ENUM AS ENUM ('active', 'inactive', 'suspended');
CREATE TABLE users (
  ...
  status USER_STATUS_ENUM NOT NULL
)
```
