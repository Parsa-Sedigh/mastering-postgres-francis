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
67.-ROWS-FROM
68.-Filling-gaps-in-sequences
69.-Subquery-elimination
70.-Combining-queries
71.-Set-generating-functions
72.-Indexing-joins
73.-Introduction-to-advanced-SQL
74.-Cross-joins