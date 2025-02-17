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
from rows from (generate_series(1, 10),generate_series(101, 112)) as t(lower, upper);
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
## 69.-Subquery-elimination
## 70.-Combining-queries
## 71.-Set-generating-functions
## 72.-Indexing-joins
## 73.-Introduction-to-advanced-SQL
## 74.-Cross-joins