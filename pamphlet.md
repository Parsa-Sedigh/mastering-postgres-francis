## 01.-Introduction-to-the-course

## 02.-Overview-of-course-structure
GUIs:
- Postgres.app
- db engine

## 03.-Postgres-vs.-everyone

## 04.-The-psql-CLI
```shell
psql -d <db name>

\?

\d # describes the schemas, tables

\d <table name> 
```

The result of running queries in raw format is not shown good, we can change the output format with:
```shell
\x auto # auto formating based on the num of cols in output and terminal size

# then run the query
```

You can set up a `psqlrc` file. So if you wanna configure psql, you can set it up in a file so that it uses it upon load.

## 05.-Introduction-to-schema
### Schema
#### Schemas as namespaces
In sqlite and mysql you have a DB that has tables and ... . But in postgres, inside a db, first you have multiple schemas.
Think of them as schemas, and then inside of schemas you have tables. So in postgres there's one more level
of organization or hierarchy available to you.

You can't have nested schemas.

- sqllite: a single file that has a db in it and inside db are tables
- mysql: we have mysql server that inside of it can have multiple dbs and tables inside those dbs
- postgres: postgres server or cluster, then postgres process and inside it, you can have multiple dbs, schemas and then tables

So tables live in schemas. By default, there's a public schema.

#### Schema as definition of the table

Table schema guidelines:
- keep them small
- simple
- representative of data: for example numbers shouldn't be stored in char cols.

The boundaries and shape of the data can match with one of the postgres data types.


## 06.-Integers

07.-Numeric

08.-Floating-point

09.-Storing-money
10.-NaNs-and-infinity
11.-Casting-types
12.-Characters-types
13.-Check-constraints
14.-Domain-types
15.-Charsets-and-collations
16.-Binary-data