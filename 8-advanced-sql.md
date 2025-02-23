## 73.-Introduction-to-advanced-SQL

## 74.-Cross-joins
- Cartesian product: Every row on the left, gets matched up with **every** row on the right. 
- The number of rows in the cross join is: `m * n. m: num of rows in left table, n: num of rows in right table`.
- Since cross join is the default for an unqualified join(no join cond on how it should link them up), we can run it with
commas in the table name.
- useful for when trying to create permutations and combinations

```postgresql
-- this is a cross join
select *
from letters,
     numbers;

-- maybe we wanna create some coupon codes(sth that requires brute-force)
select upper(letter) || number
from letters
         cross join numbers;
-- A1
-- B1
-- ...
```

The furstrating part was we had to **create** the tables(letters and numbers tables) just to hold create the series. But with
set generating funcs, we don't need to have those tables:

EX) Gen on the fly a set of codes for coupon 
```postgresql
-- named the table of this series as `letters` and it's col `l`.
-- NOTE: 65, 75 is the range of A till K.
select (chr(l) || n) as code from generate_series(65, 90) as letters(l)
cross join 
generate_series(1, 100) as number(n)
-- A1
-- A2
-- ...
-- A100
-- B1
-- ...
-- Z100
```

## 75.-Grouping
Amount each employee has sold and the products he has sold:
```postgresql
select employee_id, array_agg(products), sum(amount)
from sales
group by employee_id;
```

any_value() picks a val randomly. Is an aggregate func.

Find employees that have only sold items over 100$:
```postgresql
select employee_id, array_agg(products), sum(amount), bool_and(amount > 100) as all_over_100
from sales
group by employee_id
having bool_and(amount > 100;
```

If we wanted to get employees that sold some items over 1000, we'd use `bool_or(amount > 1000)`.

How many sales an employee were made, how many were returned, how many were not returned:
```postgresql
select employee_id,
       count(*)                                                             as sales_num,
       count(*) filter ( where sales.is_returned is false )                 as non_returned_sales,
       count(*) filter ( where sales.is_returned is true )                  as returned_sales,
       string_agg(product, ', ') filter ( where sales.is_returned is true ) as returned_products -- string_agg is an aggregate func
From sales
group by employee_id;
```
Here, we're doing 3 different count()s without having to run 3 different queries.

**Note: count(*) counts the number of rows in a result set. Where as count(<col name>) counts the number of non-null vals in that col.**
count(*) is an optimized version. So you don't need to use sth like count(id) for counting number of rows. But if you need to count the num of
non-null vals, use count(<col name>).

EX) This gives an err:
```postgresql
select employee_id,
       e.first_name || ' ' || e.last_name                                   as employee_name,
       count(*)                                                             as sales_num,
From sales
         join employees e on sales.employee_id = e.id
group by employee_id;
```
ERROR: column "e.first_name" must appear in GROUP BY clause or be used in an aggregate function.

But we don't need to put it in GROUP BY:
```postgresql
select e.id,
       e.first_name || ' ' || e.last_name as employee_name,
       count(*)                           as sales_num,
From sales
         join employees e on sales.employee_id = e.id
group by e.id;
```

## 76.-Grouping-sets_-rollups_-cubes

## 77.-Window-functions
For operating on a chunk(partition of a result set. We get to define how that chunk(partition is determined. But thee funcs, 
won't collapse the rows(unlike aggregate funcs and group by), they just add additional info.

The window definition is passed to `over ()`.

EX) Maybe in the chunks, we wanna take an avg of all the rows there, but we don't want to collapse it into one row.

-- EX) Get avg of amount per region without losing discrete sales info:
```postgresql
select *, avg(amount) over (partition by region) from sales;
```

NOTE:
- If you wanna have a partition that's ordered asc by a col and another partition ordered desc by another col, you can.
- we can also declare a **frame** which tells us **within a partition**, how far ahead and how far behind can this particular row, see.

```postgresql
select *,
       avg(amount) over (partition by region) as avg_region,
       avg(amount) over ()                    as avg -- we're doing an aggregation while retaining the row vals
from sales;
```

EX) User journey through their bookmarks. Give me their first bookmark, then second, then ... , per user.

First iteration:
```postgresql
select *,
       first_value(id) over (partition by user_id order by id),
       last_value(id) over (partition by user_id order by id)
from bookmarks;
```
In the above query, first_value() gives the first bookmark id of each user. So everytime first_val() changes, user_id also 
changes because we're partitioning by user_id.

But last_value() is not giving back what we want. Because when pg is looking at the current row in this partition, it says:
"yeah, until now, the last value I've seen is the current row." So in the result, last_value() is the same as current row id!!!
This is not what we want.

To get the last id per partition: `first_value(id) over (partition by user_id order by id desc)`. Yeah we used first_value() but with
order by id desc. Now, the first row in each partition, has the last id, so by running first_value(id), we get the last row that has the
last id.

So the query is(although confusing):
```postgresql
select *,
       first_value(id) over (partition by user_id order by id),
       first_value(id) over (partition by user_id order by id desc)
from bookmarks;
```
This is how to have the same partition(by user_id) in different orders.

Clearly, this isn't readable and is confusing.

A better way to do this, is to define the frame.
Remember that we defined the partition by user_id. We said: Chunk the rows by user_id. And then we declared a sort order inside each
partition.

Now **within** each partition, we can declare a frame. A frame defines how far ahead or how far behind pg should look within a single
partition.

There are different ways to declare a frame:
- rows
- range
- groups

EX) Here we're defining a frame: `rows between unbounded preceding and `

- unbounded preceding is start of the frame
- unbounded following is the end of the frame: means there are no bounds, consider all the way to the end of the partition as our frame.

```postgresql
select *,
       first_value(id) over (partition by user_id order by id),
       
       -- declared a frame within the partition
       last_value(id) over (partition by user_id order by id desc rows between unbounded preceding and unbounded following)
from bookmarks;
```

We can say: `last_value(id) over (partition by user_id order by id desc rows between unbounded preceding and 3 following)`
Which means the end of the frame is 3 rows after the current one(based on the `order by`). So here we're declaring how far ahead it's allowed
to look, is 3 rows.

So a better definition for last_value() is: gets the furthest thing within the frame.

Since:
1. the start of the frame doesn't matter for last_value(), because it only looks for furthest row ahead
2. the end of the frame doesn't matter for first_value(), because it only looks at the furthest row behind
So first_value needs 3 preceding, but it's end of the frame doesn't matter, so we say: current row and ...:
```postgresql
select *, 
       first_value(id) over (partition by user_id order by id rows between 3 preceding and current row),
       last_value(id) over (partition by user_id order by id rows between current row and 3 following)
from bookmarks;
```

### Factor out window definition
Factor out user_bookmarks window definition. So we can change it in one place and it gets changed in all places.
This is also good for perf, because it will calculate the window once
```postgresql
select *,
       row_number() over user_bookmarks,
       first_value(id) over user_bookmarks,
       last_value(id) over user_bookmarks
from bookmarks
window user_bookmarks as (
        partition by user_id order by id rows between unbounded preceding and unbounded following
        );
```

### using range and groups as frame specification
Get the sales data with the biggest sale on 5 days either side of the current sale.

So within 5 days preceding nad 5 days following, was my sale the best?
```postgresql
select *, max(amount) over (order by sale_date asc range between '5 days' preceding and '5 days' following)
from sales; 
```
Here, the frame is `range between '5 days' preceding and '5 days' following`.

For example, in the img, the max is 1200 for the highlighted rows. Because first of all, the window is the whole table because there's no
`partition by`. Now the frame is 5 days before and 5 days after, so for sale_id=1, the only row in it's 5 days ahead frame, is sale_id=2.
The max(amount) between those is 1200. So the biggest sale in that 10 day span of 2024-01-15 is 1200.

So here, we're considering the sales within 10day span of each other, in the same frame.

![](img/77-1.png)

Q: Can we use '-5 days' instead of `preceding`? No, in sql we can't use '-5 days'. Instead, you must explicitly write the
number of days without the negative sign.

Note: With lead() and lag(), we can **peek** rows from behind or ahead in the current row's frame.

## 78.-CTEs
A way for refactoring query into distinct parts. They can imprv perf in situations where you have multiple same subquery, you can extract
that subquery into a CTE, then since it's referenced multiple times, PG will materialize it (into one temp table) which means running it once here
and reference that temp table multiple times instead of running it multiple times(the case without CTE).

Note: We need to consider some perf things here.
```postgresql
-- NOTE: with is_deleted col, we know which table a row came from
with all_users as (
    select false as is_deleted, * from users
    union all
    select true as is_deleted, * from users_archive
)
select * from all_users where email = 'sth@sth.com';
```

We could name the cte as `users` which is the name we have as a table as well! Why would we do that?
Maybe we have an orm, but we might need to search across regular users and users_archive at the same time. We might loose some of
the abilities of the orm. We don't loose those abilities if we redefine the users table as a `union all` of users and users_archive.
And then we let the ORM generate the query.

If we run `explain (analyze, costs off)` on prev cte + query, we get:
```
Append (actual time=0.072..2.266 rows=1 loops=1)
    -> Index Scan using email_btreeton users (actual ti...
        Index Cond: (email = 'aaron. francis@example. co... -- HERE
    -> Seq Scan on users_archive (actual time=2.058..2....
        Filter: (email = 'aaron.francisdexample.com'::... -- HERE
        Rows Removed by Filter: 10091
Planning Time: 0.379 ms
Execution Time: 2.337 ms
```
So the where cond gets pushed down from the outer query into each individual query of CTE.
What's happening here is PG can decide whether to materialize or not materialize a CTE.
Here, pg decided not to materialize it(the CTE I mean) because it's only being referenced once in the outer query.
In other words, it gets no benefit from running the query, writing it to a temp table and then running query against it.

Materialization: creating a temp table and operate against it.

Now if this CTE was being referenced multiple times, it will materialize the CTE so that it doesn't have to run the CTE multiple times.
It just run it once and use it multiple times. And it could be a good thing or bad, it depends on your query. But you can tell
PG to materialize a CTE or not. Because maybe you understand sth about your data set.

Note: So if the CTE is referenced twice or more in the outer query, it's gonna materialize it and sometimes that's exactly what you're looking for,
sometimes it's not. But if in a situation, you disagree with pg, you can force it make the CTE materialize or not.

**If you want everytime a CTE gets referenced, you can say: not materialized.**
```postgresql
with all_users as not materialized (
    -- ...
)
select *
from all_users;
```


If we make the prev query(that we ran explain on) materialized, it makes it worse! Because it has to scan both tables and then
put them together to form the CTE table, then apply the filter:
```
CTE Scan on all_users (actual time=208.195..550.485 rows=1 loops=1)
    Filter: (email = 'aaron.francis@example.com':: text)
    Rows Removed by Filter: 999998
    CTE all_users
        -> Append (actual time=0.040..128.183 rows=999999 loops=1)
            -> Seq Scan on users (actual time=0.039..88.980 rows=989908 loops=1)
            -> Seq Scan on users_archive (actual time=0.014..0.830 rows=10091 loo...
Planning Time: 0.174 ms
Execution Time: 554.132 ms
```

CTEs can reference prev CTEs.
```postgresql
with all_users as not materialized (
    -- ...
),
     aarons as (select *
                from all_users
                where email = 'aaron.francis@example.com')
select *
from aarons;
```

## 79.-CTEs-with-window-functions
## 80.-Recursive-CTE
## 81.-Hierarchical-recursive-CTE
## 82.-Handling-nulls
## 83.-Row-value-syntax
## 84.-Views
## 85.-Materialized-views
## 86.-Removing-duplicate-rows
## 87.-Upsert
## 88.-Returning-keyword
## 89.-COALESCE-generated-column