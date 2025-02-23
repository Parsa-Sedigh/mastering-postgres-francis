## 62.-Introduction-to-queries

## 63.-Inner-joins
Default joins:
- for unqualified(unconditional) join -> cross join 
- for qualified(conditional) join -> inner join 

Inner join takes two tables(left & right) and it matches them up on a qualification(this is because it's qualified). The qualification
is usually two cols matching each other. And then any rows that don't have a match on both tables, are eliminated.

So any rows on the left table that don't have a match on right side, eliminated and vice versa.

```postgresql
-- since we have two tables and both have the col `id`, we need to use qualified name here(add the table qualifier to specify which one you want).
select 
    users.id, users.first_name, bookmarks.user_id
from users
         join bookmarks on users.id = bookmarks.user_id
limit 10;
```

When you use `using()` in join conditions with `select *`, it's not gonna bring both cols that have the same name and were used in `using()`.
Because it would run into ambiguous error.

## 64.-Outer-joins
Contains 3 types of joins:
- left outer
- right outer
- full outer: everything from both left, right and matching ones are returned.

```postgresql
select *
from users
         left join bookmarks on users.id = bookmarks.user_id -- we could omit `bookmarks.` part because user_id is not ambiguous.
limit 10;
```
Every single user(everything from left table) will show up in this query, even if they don't have a match in bookmarks.

## 65.-Subqueries
We can use them in different ways:
- produce a set of results against which we'll join
- eliminate records based on criteria on another table, before joining the corresponding table in.
_EX) Show me all the people who have bookmarks that are secure._ But in the JOIN case: _Show me all the people **and** their secure bookmark_.
So we're gonna JOIN them.

```postgresql
select * from users
left join (
    
) on users.id = bookmarks.user_id
limit 10;

-- Here, we're creating a composite(compound) index. One of it's parts is an expression.
-- NOTE: Functional index, is an index on an expression. The expression should be wrapped by another ().
-- If we don't use another (), we're referring to a col.
create index bookmarks_secure_url on bookmarks (user_id, (starts_with(url, 'https')));
```

Just a side note: PG won't use bookmarks_secure_url index for this query, because of left to right no skip. We're skipping `user_id`:
```postgresql
explain
select *
from bookmarks
where starts_with(url, 'https') is true
limit 10;
```
```
Limit ...
    ->  Seq scan on bookmarks (...)
            Filter: ...
```

But that index will be picked up on the following query.

eliminate records(done in the subquery) before joining it with another table(or result set)
```postgresql
select * from users left join (
    select *
    from bookmarks
    where starts_with(url, 'https') is true
    limit 10
) as bookmarks_secure on users.id = bookmarks_secure.user_id
limit 10;
```
You could join everything together and then eliminate the rows. But that would be a lot of rows. So we filter them out earlier.

```
Limit ...
Merge join
...
    -> index scan using users_pkey
    -> index scan using bookmarks_secure_url
```

## 66.-Lateral-joins
Not a different type of join, just a different qualifier for existing join. You can have an inner join lateral, outer join lateral,
cross join lateral.

What lateral keyword tells pg is: for every row in preceding table, run this subquery and use this result set.
A lateral join can be terribly expensive. Depending on how big the preceding table is and how expensive the subquery is.
Note that you're gonna run that subqeury once per row of preceding table.

Get everyone's latest bookmark. There are a couple of ways to do this:
- lateral join
- CTE

```postgresql
select *
from users left join lateral (
    -- subquery to get top bookmark per user
    select * from bookmarks where user_id = users.id
                            order by id desc
                            limit 1
) as most_recent_bookmark on true
limit 10;
```
1. Why joining `on true`? Because we're actually constraining the result set inside of the subquery, so anything that comes back from the
subquery should be joined(we already got the last bookmark, we only need to join it with users).

But running it give this error: 
```
ERROR: invalid reference to FROM-clause entry for table "users".
Hint: To reference that table, you must mark this subquery with LATERAL.
```

So pg is telling that we have referenced a table incorrectly. We can't reference a preceding table inside of a subquery unless we mark
it as lateral. After using lateral, for every row in preceding table, it's gonna run the subquery.

2. why we used `left join`? Because some of the users might not have a bookmark at all, but we need all the users(left table). We can see this by
using `where most_recent_bookmark.id is null`.

## 67.-ROWS-FROM
With `rows from` we can put cols next to each other without `JOIN`ing.

Usually in a set generating func, you're not using the ordinality, we don't have the primary key and foreign key that we can `JOIN` on,
but you still want to have the cols next to each other.

```postgresql
-- generate_series is a set generating func
select generate_series(1, 10);

select generate_series(101, 110);

-- two queries side by side:
select *
from rows from (generate_series(1, 10),generate_series(101, 112)) as t(lower, upper); -- t is table_alias, in (), we specify the col aliases
```
We want the above queries side by side, but we can't do cross join, because that's gonna produce too many rows.
We just want them side by side. So we use `ROWS FROM`.

Note: In `rows from`, The shorter col gets null padded.

`ordinality: auto incrementing int.`

EX: Gen rows for the days of the year:
```postgresql
select date::date, num
from rows from (
         generate_series('2024-01-01'::date, '2024-12-31'::date, '1 day'),
         generate_series(1, 380) -- we could go up to 366 to cover everything just in case.
         ) as t (date, num)
where date is not null; -- to eliminate null-padded rows
```
In this ex, we could got away with just ordinality, since we started our generate_series() num at 1 and that's what ordinality does, it puts
an auto-incrementing int in there.

```postgresql
-- put arrays side by side.
select *
from rows from (
         unnest(array [101, 102, 103]),
         unnest(array ['Laptop', 'Smartphone', 'Tablet']),
         unnest(array [999.99, 499.99, 299.99])
         ) as combined(product_id, product_name, price);
```

## 68.-Filling-gaps-in-sequences
**We're gonna generate a series then `LEFT JOIN` it to cover gaps in the right table seq.** Why `left join`? Because we want all the rows
from the left table to show up, even if there are no matches in the right hand table.

EX) In the result of this report, some sale_date days are missing. Because there weren't any sales in some days.
```postgresql
select sale_date, sum(amount)
from sales
group by sale_date
order by sale_date;
```

To fill the gaps(missing dates):
```postgresql
select all_dates.sale_date::date, coalesce(total_amount, 0)
from generate_series('2024-01-01'::date, '2024-12-31'::date, '1 day') as all_dates(sale_date)
         left join (
    -- note: We don't need ordering here, we can do it outside of the subquery later
    select sale_date, sum(amount) as total_amount
    from sales
    group by sale_date) as sales on all_dates.sale_date = sales.sale_date;
```

We cast all_dates.sale_date::date, because we don't want to show it as timestamptz.

In many other dbs, you're stick with recursive CTE to do this. Because they don't have a generate_series() func in them.

## 69.-Subquery-elimination
Everything we've done with subqueries so far has been **producing a result set** that we then treat as a table LIKE use to `JOIN` or ... .
Now we wanna use subquery as **a way to filter a table based on data from a related table**.
Note: We don't want to show the data of the related table, so we don't want to use a `JOIN`. So:
1. producing a result set
2. a way to filter a table based on data from a related table

EX) Return users that(filter) have most bookmarks(data from another table).

A) We don't want to return the data from the bookmarks table, so we're not gonna use a `JOIN`, we just want the users based on
the bookmarks table, so we use a subquery.

- projects that have 0 tasks in it
- users that have > 16 bookmarks

```postgresql
select user_id, count(*)
from bookmarks
group by user_id
having count(*) > 16;
```

Now we could do an inner join on the prev query and then just select the users cols. That's fine, but we can also do it with subquery.
```postgresql
-- this query is called a semi-join. If you do, `id not in`, it's called anti-join.
select *
from users
where id in (select user_id
             from bookmarks
             group by user_id
             having count(*) > 16);
```

Query decomposition: running a query, get back it's result and then put the result into subsequent queries.
That's not performant. Because of multiple round trips to db server.

Note: If you wanted to bring count(*) from users ... back, you have to switch to JOIN, because in the subquery approach, we want exactly
one col back from the subquery.

users that have > 16 bookmarks:
```postgresql
-- eliminate rows from the users table based on bookmarks table.
-- For this, we start from actually the bookmarks table, then JOIN it with users.
select users.id, first_name, last_name, ct
from (select user_id, count(*) as ct
      from bookmarks
      group by user_id
      having count(*) > 16) as whales
         inner join users on whales.user_id = users.id
order by ct desc;
```
Note: We had to use a subquery for `from`, because we can't write JOIN after having part, we have to wrap the `whales` query.

### Another approach for subqueries for elimination
With `exists (...)`, we're referencing the outer table

```postgresql
select *
from users
where exists (select 1 -- all we're looking for is mere presence of a row
              from bookmarks
              where users.id = user_id -- we're referencing outer table
              group by user_id
              having count(*) > 16);
```

The fundamental difference is the query we pass to parentheses in exists (), is gonna run for every row in the outer query. Which can be good.
Because where exists () will short circuit the first time it finds a true val.

But this query is a lot worse than the first one in our example. We can see this using `explain analyze`.

**Note: `exists ()` operates on existence of a row. But in () looks at the vals.**

Let's see a use case where the `exists (...)` it's actually good:

All users that have bookmarked a secure url:
```postgresql
select *
from users
where exists (
    select 1 from bookmarks where users.id = user_id and starts_with(url, 'https')
);
```
Here, for every user row, if it finds a secure url, it won't keep looking at the bookmarks table anymore. Because we just want the
existence. That's a case where `exists ()` is faster than `in()`.

So choosing `in ()` vs `exists ()` depends on the case.

---


```postgresql
explain (analyze, costs off)
select *
from users
where id in (select user_id
             from bookmarks
             group by user_id
             having count(*) > 16);

explain (analyze, costs off)
select *
from users
where exists (select 1
             from bookmarks
             where users.id = user_id
             group by user_id
             having count(*) > 16);
```

```
FOR FIRST QUERY:

Merge Join (actual time=54.369..612.415 rows=19 loops=1)

    Merge Cond: (users. id = bookmarks.user_id)
    -> Index Scan using users_pkey on users (actual time=0.066..91.946 rows=940763 loops=1)
    -> GroupAggregate (actual time=47.241..495.349 rows=19 loops=1)
        Group Key: bookmarks.user_id
        Filter: (count (*) > 16)
        Rows Removed by Filter: 983392
        -> Index Only Scan using bookmarks_user_id_idx on bookmarks (actual time=0.053.281.866 rows=4967570 loops=1) -- loops=1 , so only did it once
                Heap Fetches: 0
Planning Time: 0.213 ms
Execution Time: 612.520

===============

For SECOND QUERY:

Seq Scan on users (actual time=98.419..1237.988 rows=19 loops=1)
    Filter: EXISTS(SubPlan 1)
    Rows Removed by Filter: 989889
    SubPlan 1
        →> GroupAggregate (actual time=0.001..0.001 rows=0 loops=989908)
                Filter: (count (*) > 16)
                Rows Removed by Filter: 1
                -> Index Only Scan using bookmarks_secure_url on bookmarks (actual time=0.001.. 0.001 rows=5 loops-989908)
                        Index Cond: (user_id = users.id)
                        Heap Fetches: 0
Planning Time: 0.146 ms
Execution Time: 1238.029 ms
```

In line `Index Only Scan using bookmarks_user_id_idx`, it only did it once (loops=1). So the subquery only ran once. Which is very good.
It also used it as covering index.

But the second query, has a lot of loops. So the subquery is running a lot but it's not getting any of the benefits from `exists ()`'s short circuit,
because we're saying: group on all bookmarks(`group by user_id`).

So in these kind of queries where we have some aggregate(like count(*)), we only want to do the subquery once.
But sometimes you actually want to run the subquery per row.

The subquery we pass to `where exists ()` is allowed to reference the outer table and it's gonna run(evaluate) for every row of the outer query.

### Careful with `where not in ()` and null
```postgresql
select *
from users
where id not in (values (1), (2), (null));
```
This query always returns nothing.

So the second we get a `null` in our table and we do not in (), the query returns nothing.

That's because the comparison to null is null. 

So use not exists ().

## 70.-Combining-queries
Instead of putting results side by side, here, we wanna put them on top of each other.
- union
- intercept
- except

```postgresql
-- should have the same number of cols and datatypes
select 1 
union 
select 2;

-- Let's say due to GDPR, we move the deleted users to users_archive table.
-- find out if a user has been deleted or not. If a user is deleted, it appears in `users_archive` and not in `users` anymore.
-- With this query, we can search across all the users even if they are in different tables
select true as active, * from users where email = 'aaron.francis@example.com'
union
select false, * from users_archive where email = 'aaron.francis@example.com';
-- active: 1, ...
```

### union all
This query only gives one row, instead of two. To have both, we have to use `union all`.
```postgresql
select 1 
union 
select 1;
```
Note: By default, union is gonna remove the duplicate rows which can be very expensive. So if you don't want that, use `union all`.
Why it's expensive?

Because it has to compare the entire row with every row in the result set and it has to do this for every row.

So if you know(either by pure logic or just some business process in your brain) there can't be duplicates, or you actually want to see
the duplicates, use `union all`, because it's gonna be way faster.

The same for `intersect all`. It's faster than `intersect`, because it doesn't perform removing duplicates op.

### Order of ops
Be aware about order of ops:

```postgresql
select generate_series(1, 5)
intersect all
( -- first this runs
    select generate_series(3, 7)
    union all 
    select generate_series(10, 15)
);
```

### except
Give me the result of first Q except where it overlaps with the second Q.

```postgresql
-- Give me entire result of first Q, except the vals in second Q. So 3, 4 and 5 will not be included, since they show up in the second Q.
select generate_series(1, 5)
except all
select generate_series(3, 7);
```

## 71.-Set-generating-functions
Funcs that return not just a single val, but a set of rows and potentially a set of cols.

```postgresql
select generate_series('2024-01-01'::date, '2024-01-10'::date, '1 day')::date as date;

-- even nums from 0 to 100
select generate_series(0, 100, 2)::int;

-- turn the arr into a set of rows by unnesting it
select unnest(array [1, 2, 3, 4, 5]) as elements;

-- There's an option to have ordinality. Ordinality is great when you need a unique inc id for these generated rows.
select ordinality, element
from unnest(array ['first', 'second', 'third']) with ordinality as t(element, ordinality); -- t is table alias and in (), we specify the col aliases

-- json_to_recordset turns a json arr of objs into a table(so we can query it and ...)
-- NOTE: We didn't cast the str we passed to the func, to json type. This is in contrast to jsonb. If you wanted jsonb, you had to
-- cast the str to jsonb too. This is because the `json` type is pure text.
select * from json_to_recordset('[
  {
    "id": 1,
    "name": "Alice",
    "email": "alice@example.com"
  },
  {
    "id": 2,
    "name": "Bob",
    "email": "bob@example. com"
  }
]') as x(id int, name text, email text);

-- jsonb
select * from jsonb_to_recordset('[
  {
    "id": 1,
    "name": "Alice",
    "email": "alice@example.com"
  },
  {
    "id": 2,
    "name": "Bob",
    "email": "bob@example. com"
  }
]'::jsonb) as x(id int, name text, email text);

select regexp_matches('The quick brown fox jumps over the lazy dog', '\m\w{4}\M', 'g') as match;
-- result:
-- {over}
-- {lazy}

-- CAPTURING
-- Let's say we have a formatted str from some other system that you want to extract data out of. Here, we're capturing the vals for Name and Age.
select regexp_matches('Name: Alice, Age: 30; Name: Bob, Age: 25', 'Name: (\w+), Age: (\d+)', 'g') AS match;
-- {Alice,30}
-- {Bob,25}

-- We could have a joined str from an outside system or a csv.
select string_to_table('apple,banana,cherry', ',') as fruit;
-- apple
-- banana
-- cherry
```

## 72.-Indexing-joins
### Difference between foreign key and foreign key constraint
A foreign key constraint enforces referential integrity and a foreign key is just a concept where you have a pointer to
another table and a row in that table.

We wanna add indexes to make JOINs faster, so we'll index our foreign keys.

NOTE: When you create a primary key or a unique constraint in the parent table, that's enforced by an index. So that index
is automatically created in the parent table. However, when you create a foreign key or even a foreign key constraint,
an index is not automatically created(in the child table which is referencing the parent using foreign key).

```postgresql
create table states (
    id bigint generated always as identity primary key ,
    name text
);

create table users (
    id bigint generated always as identity primary key ,
    state_id bigint references states(id),
    name text
);

select * from pg_indexes where tablename = 'states';
-- public states states_pkey null create unique index states_pkey on public.states using ...

select * from pg_indexes where tablename = 'cities';
-- public cities cities_pkey null create unique index cities_pkey on public.cities using ...
-- NOTE: So no index was automatically created for the state_id foreign key. But we can do it ourselves.
```

Here, the parent table is users(since each user can have many bookmarks)
```postgresql
explain select *
from users
         inner join bookmarks on bookmarks.user_id = users.id
where users.id < 100;
```
```
Gather (cost=1012.67..94051.93 rows=537 width=153)
    Workers Planned: 2
    → Hash Join (cost=12.67..92998.23 rows=224 width=153)
        Hash Cond: (bookmarks.user_id = users.id)
        
         -- but then we're scanning the entire bookmarks to look for bookmarks with matching the join cond
        -> Parallel Seq Scan on bookmarks (cost=0.00..87552.25 rows=2069825 width=77)
        -> Hash (cost=11.33..11.33 rows=107 width=76)
            →> Index Scan using users_pkey on users (cost=0.42..11.33 rows=107 width=76) -- uses index on where users.id < 100
                 Index Cond: (id < 100)
```

So create index on the child table foreign key:
```postgresql
-- name it idx_bookmarks_user_id or fkey_bookmarks_user_id
create index idx_bookmarks_user_id on bookmarks(user_id);
```

Now if we run the JOIN query again:
```
Nested Loop (cost=0.86..3059.26 rows=537 width=153)
    → Index Scan using users_pkey on users (cost=0.42..11.33 rows=107 width=76)
        Index Cond: (id < 100)
    → Index Scan using idx_bookmarks_user_id on bookmarks (cost=0.43..28.43 rows=6 width=77)
        Index Cond: (user_id = users.id)
```

Note: We can create a composite index with the left most col of it being the col we JOIN on. With this, we get the benefit
of JOIN using the index and the filtering.

So if you wanna JOIN tables, it usually makes sense to have an index on the foreign key(child table) either by just the foreign key col
or as a part of a composite index which it's left most col is the foreign key col.