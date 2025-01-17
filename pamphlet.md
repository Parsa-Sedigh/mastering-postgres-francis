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
- postgres: postgres server, cluster and the postgres process. Inside it, you can have multiple dbs, schemas and then tables

So tables live in schemas. By default, there's a public schema.

#### Schema as definition of the table

Table schema guidelines:
- keep them small(compact). If our range is 0-100, do not chose a bigger data type. Note: we can order the cols of the table to
make the table even more compact.
- simple
- representative of data: we need to chose one data type that encompasses full range of our data. So do not mangle your data to save
a few bytes. For example numbers shouldn't be stored in char cols.

The boundaries and shape of the data can match with one of the postgres data types.


## 06.-Integers
- int2 - smallint
- int4 - integer
- int8 - bigint

Integers that fit in a **bounded range**.

Postgres doesn't have the concept of unsigned integers as type. But we can still enforce integers of col being positive using check constraints.

int2: integer that takes 2 bytes => smallint.

It's better not to rely on datatype for data validation. For example, choosing smallint or int2 for age. It's better ot use check constraints
for business logic data validation.

So rely on data types for compactness instead of business data validation.

The only place to give our data way too much room to grow is primary key ids, because we don't want to run out of space. 
For those, default to bigint(int8). It can also be used for sth like `file_size_bytes` col.

## 07.-Numeric
- integers
    - only support whole numbers
    - accurate
    - very fast
- numerics(or decimals)
    - support fractions of numbers
    - accurate
    - extremely slow
- floating points
    - support fractions of numbers
    - not accurate(approximation)
    - very fast

- numeric == decimal
- When dealing with financial data like interest_rate, use numeric. 
- Numeric supports a lot more bigger nums than int8.
- without passing args to numeric or decimal, it's unbounded precise num. 
- numeric without args, accepts everything, kinda like `text` col. But with parameters, enforces some sort of range in terms of number of digits, kinda like
character varying col.
- If you don't pass in the second arg, it would be 0.
- if you pass negative as scale, like: select 1234567.345::numeric(5, -2). It will round it from decimal place to 2 places in.
- the precision, represents number of significant un-rounded digits.

numeric(<precision>, <scale>): the total number of digits is called **precision**. allowed number of digits after fraction point is **scale**.
```postgresql
-- we can represent 12.345 as numeric(5, 3) without losing precision
select 12.345::numeric(5, 3); -- 12.345
```

If you wanna store or cast a number with:
- total num of digits larger than the precision, you will get an err. Example: `select 123.345::numeric(5, 3)` -> `ERROR: numeric field overflow...`
- num of fractional digits larger than scale, it will round them and store it without err. Example: `select 12.345::numeric(5, 2)` -> stores: `12.35`

Interesting example:
```postgresql
select 1234567.345::numeric(5, -2)
```
result: 1234600 . So this works although the number we passed has 7 digits before decimal point, but scale is 5. This is because two digits
gonna be rounded down, so we would have 5 digits(the 0s are not counted). But look at the result. It has 7 digits! Since 5 represents significant un-rounded digits,
we CAN chop them off with passing negative val to scale, but CAN'T store vals with more than 5 significant digits.

So passing negative as scale, will round down digits before decimal point starting from right side. So in this example, 6 & 7 gonna chopped off and we will
have 0s there. The 0s are not actually stored, so it's not a padded digit.

## 08.-Floating-point

## 09.-Storing-money
## 10.-NaNs-and-infinity
## 11.-Casting-types
## 12.-Characters-types
## 13.-Check-constraints
## 14.-Domain-types
## 15.-Charsets-and-collations
## 16.-Binary-data