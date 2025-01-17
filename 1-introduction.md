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