## 39.-Introduction-to-indexes
The best way to unlock a performant db. It's a separate DS that maintains copy of part of the table and it's optimized for some ops.
It contains pointers(CTIDs) back to the table and the row.

1. Indexes are separate DSs from your table. That DS is optimized for particular ops. The most common DS for this is btree. 
Most of your indexes is gonna be using btree.
2. Index maintains a copy of part of your table. So if we put an index on last_name, it takes all the `last_name`s, copies them from
the table and puts them into the index DS, in a way that it makes it easy to traverse the index to quickly find a row by last_name.
![](img/39-1.png)
3. Don't put index on every col. It's gonna slow the writes. Because it's a separate DS that maintains copy of part of the table,
then everytime we update or insert, that index needs to be updated and also maybe the order of the index has to be re-arranged to make
the new val fit in the appropriate place. We could be potentially updating multiple indexes.
4. each index contains a pointer back to the table and row, so we can get the full data of that row. ![](img/39-2.png).
Note: In most DBs(not in postgres), every index contains a pointer to the **primary key** which is how the table is arranged. But that's not
how it works in pg. In pg, every index contains a pointer to the **table and the physical location where that row is**.

With indexes, we don't have to scan the table sequentially to get the row. 

## 40.-Heaps-and-CTIDs
### Storage arrangement
Q: How pg stores the rows in the disk?

A: There are equal-size blocks of data, called pages and in each page, there are positions where the rows are.
`EX: (4, 10) means page 4, position (row) 10`. And that's a unique identifier and these identifiers are used in the index and
with them we go to the table, page 4 and row 10.

The structure that these rows exist in, is a heap. It's a good name, because it's a pile. Put the rows wherever there is space.
This makes INSERTs very fast, because all it's doing is looking for some blank space.

CTIDs:
```postgresql
select *, ctid from reservations; -- citd: (0, 2) . In page 0, position 2, exists this entire row(physically on disk).

-- don't do this!
select *, ctid from reservations where ctid = '(0, 2)';
```
Don't use ctid for finding rows, because ctids can & will change. If you update a row and maybe some of the row's vals for
variable-size cols get a lot bigger and there might not be enough space, so pg will take that row from page 0 to 84, because
p 84 has ton of space. So the ctid changed.

Also when pg performs vacuum, it's gonna rearrange rows, so ctids will change.

Every index contains a ctid such that it can get back to the row in table.

## 41.-B-Tree-overview
![](img/41-1.png)

**The data & related ctid are at leaf nodes and at that level, data & ctids are sorted.**
```postgresql
select *
from users
where name = 'jennifer';
```
According to the index img, Jennifer falls in the middle path. In other words: `isaac < jennifer < steve`.
Then in the middle node: `jennifer < simon`. So we go to left path.

Then in leaf node, we search for the val we want.

EX: We're looking for taylor. At root: `isaac < steve < taylor`. So we go to right subtree.
Then we hit a non-leaf node that contains the value we want. Since it's equal, we go to right subtree.

If we didn't have index(secondary DS), we had to scan the entire table and it's called **table scan**.

When we have a table that have small num of data, it might be faster to just scan the whole table than using index.

## 42.-Primary-keys-vs.-secondary-indexes
In mysql, when you declare a primary key, you're simultaneously declaring the clustered index. And in mysql, the clustered index
is the way that the data is arranged on the disk. So in mysql, everything is in index, including the table. The table itself is a btree index,
where the entire rows are held down at leaf nodes.
![](img/42-1.png)
![](img/42-2.png)

In mysql, a secondary index is any other index that's not the clustered index(but in pg, every index is secondary).
So data on disk is sorted based on the primary key(the clustered index of the table).

In pg, every index is a secondary index.

In pg, the data on disk is stored in a heap - a big pile. Meaning there's no clustered index. Therefore, every index is a secondary index.
Every index lookup requires traversing the index and then hopping over to the heap and finding the rows(that's mostly true because of
covering indexes).

So what's a primary key in pg?

A primary key is a special type of secondary index which has more abilities:
1. it enforces uniqueness
2. not null
3. automatically creates the underlying index for you. So you don't have to create the index for primary key yourself.

Every table can have only one primary key.

## 43.-Primary-key-types
Bigintegers vs uuids for primary key

**In 98% of use cases, you should use bigint type for primary key.**

### UUID
At this point, there are 7 variants of a UUID. We also have ULID(lexographically sortable uuid).

Which variant of uuids to use?

The builtin uuid in pg that we get from gen_random_uuid(), is not time-ordered but with auto incrementing bigint, it's always
increasing which fits nicely with the structure of a btree. Since it's always increasing and when inserting new vals, 
everything will be added to one side and then new internal nodes will be added.
But when you have a random primary key, they will be inserted at random points down at leaf nodes and you might have to break
and reblance that btree over and over again. Now this is not as much of a problem in pg as it is in mysql where the table is actually
arranged on physical disk by primary key. So the penalty in pg is not as great as dbs with primary key as clustered index
but there's still a penalty for inserting a random key.

There are 2 drawbacks when using uuids as primary key:
1. it's size is larger. But in pg, we have it at 16bytes, so that's a lot more compact than would otherwise be
2. real problem: random insertion. Causes the btree index to have to fracture and re-balance. You can get around that by using uuid v7 which is
a time-ordered uuid. It's first several bytes are dedicated to time such that when you generate new ones, it's always gonna come after
the uuids that we generated before. So functionally, we're back to auto-incrementing style but instead, it's a uuid. ULIDs are the
same way, meaning they have the time portion at the beginning.

**The real benefit to use a uuid b7 or ulid: You can generate them without coordination and without talking to the DB.**

EX: Let's say you have multiple clients and you need optimistic UI(create sth on UI immediately without first getting the ok res from
the backend). You create an entity on client, now you need that id **immediately** and then you send it to the backend to store it in db.

### Is there security risk when using bigint auto-incrementing bigint as primary key
WOW, using auto-incr bigint as primary key is a security risk. It can give away information like how manu invoices have been created pr how
many users we have. To mitigate this, we can have a public id along side your bigint primary key.
Now in cases you don't want to expose your auto-incr bigint primary key, create a secondary key using lib like nanoid(compact, very random, 
impossible to guess) and use that in your api or url or wherever it's public facing and then you can look up by that
nanoid in table(when client sends that nanoid to get the resource), but you still get all the benefits of auto-incr bigint primary key.

### Summary
Prefer bigint unless you have a good reason not to. If you're gonna use uuids, use a time-ordered variant which is v7 or a ULID.
If you're afraid of exposing auto-inc primary key, favor a secondary key like nanoid.

## 44.-Where-to-add-indexes
You can look at the data for creating the schema. But we can't look at the data or schema and derive good indexes out of them.
You must look at the access patterns(how are we gonna query the table) AND THEN, look at the data itself.

Do not add an index on every col! Because an index is a separate DS that maintains a copy of part of your data(one or few cols).
Now by adding index on every col:
- we duplicated our data
- writes(INSERT, UPDATE, DELETE) are gonna be slower, because all of those separate DSs need to be maintained
- your reads perf are not gonna be as fast as you want because probably it's better to have one composite index over multiple cols
instead of having multiple single indexes over single cols

This is better than "adding index to every col", but not entirely correct: "anything that shows up after the `WHERE`, you should have
an index on that".

Not entirely correct, because: we should consider the entire query. Yes, the `WHERE` clause is important, also ORDER, GROUP, JOIN,
SELECT, ... all of those things are important for designing effective indexing strategy.

Note: Users table has 1M rows.

We're prove 5 cases where btree index is used:
- strict equality
- unbounded range(there will be a index selectivity issue in one case)
- bounded range
- grouping
- ordering

Note: We won't look at indexes in `JOIN` yet.

```postgresql
explain select * from users where birthday = '1989-02-14';
```

```
Gather (cost=1000.00..19845.77 rows=90 width=76)
    Workers Planeed: 2
    ->  Parallel Seq scan on users (cost=0.00..18836.77)
            Filter: (birthday = '1989-02-14'::date)
```

Here, pg is doing a parallel scan and it dedicated 2 workers to this and each gets it's own pages, they go off and scan the table
and then it will gather the results at the end(`Gather ...` line).

`Seq scan` is not what you want to see.

```postgresql
create index bday on users using btree(birthday);
```

Then if we run the explain again:
```
Bitmap heap scan on users (cost=5.12..344.35 rows=90 width=76) -- 2) then goes to table to get the full rows 
    Rechecked cond: (birthday = '1989-02-14'::date)
    -> Bitmap index scan on bday (cost=0.00..5.10 rows=90 width=0) -- 1) is using our index
            Index cond: (birthday = '1989-02-14'::date)
```

Note the `Bitmap index scan on bday`.

So an index helps us with strict equality.

Now change cond to use less than operator: `birthday < '1989-02-14'`, it will still use the bday index
just the cost of Bitmap heap scan is higher(4683.62..23589.14).

But if we change the cond to greater operator, it won't use the index!!!
```
Seq Scan on users (cost=0.00..26054.85 rows=571857 width=)
    Filter: (birthday = '1989-02-14'::date)
```

Why? Let's see selectivity for these queries:
```postgresql
select count(*) from users where birthday < '1989-02-14'; -- 417K
 
select count(*) from users where birthday > '1989-02-14'; -- 572K. The index on birthday is giving back more than half the table. So the index will be skipped.
```
So in first case, pg will eliminate more rows to get the rows we want. In other words, the final result set is smaller than the second query, so
it will use the index more likely.

So when an index doesn't help eliminating enough rows, pg will go straight to the table and won't use the index.

Note: The cardinality and selectivity with range queries are a bit different.

So an index helps on strict equality and first unbounded range(not being between two vals, just less than or greater than),
but not the second unbounded range. That's because of index selectivity.

Bounded range:
```postgresql
explain
select *
from users
where birthday between '1989-01-11' and '1989-12-31';
```
It uses the index:
```
Bitmap heap scan on users (cost=459.28..14642.01 rows=33449 width=76)
    Rechecked cond: ((birthday >= '1989-01-11'::date) AND (birthday <= '1989-12-31'::date))
    -> Bitmap index scan on bday (cost=0.00..450.92 rows=33449 width=0)
            Index cond: ((birthday >= '1989-01-11'::date) AND (birthday <= '1989-12-31'::date))
```

### Order by
Wherever you have sorting that might be used often, you definitely want to get that query index assisted.

```postgresql
explain
select *
from users
order by birthday;
```
Is using the index:
```
Index scan using bday on users (cost=0.42..72991.82 width=76)
```
> When you order by using an index, the vals are already in order. So pg just reads the index from front to back or in some cases back to front,
> and then it goes picking up the rows in that order out of heap.
> But when you don't have an index, it's gonna read the whole heap and **then** sort it.

Using explain on a sort query would look quite different than a query with `WHERE`:
```
Gather merge (cost=75605.27..28..171853.12 rows=824924 wudth=76)
    Workers Planned: 2
    ->  Sort (cost=74605.24..75636.40 rows=412462 wudth=76)
        Sort key: created_at
        ->  Parallel seq scan on users (cost=0.00..17805.62 rows=412462 width=76)
```
So it's doing seq scan on table with sort key created_at, then it sorts, using 2 parallel workers, then it merges the result of those
workers.

### Grouping
```postgresql
explain
select count(*), birthday
from users
group by birthday;
```
Is using the index(we don't see any sequence scan):
```
Finalize HashAggregate (cost=17963.69..18073.09 rows= width=)
    Group key: birthday
    ->  workers planned: 2
    ->  Partial GroupAggregate (cost=0.42..14666.)
        group key: birthday
        ->  parallel index only scan using bday on users (cost=0.42)
```

## 45.-Index-selectivity
1. look at queries to derive the candidates for indexing
2. look at data to determine if it's a good or bad candidate. For example: Every row in users table had a first name of Aaron.
Our query has: `where first_name = 'Aaron'`. That index won't help us narrow down anything very fast.

**To find out if it's a good index candidate:**
- Cardinality: number of distinct(unique) vals in the col. Let's say our col is bool. So it can only have 2 vals. So the cardinality is 2.
There are 2 unique vals in that col. Is that good or bad? Well kinda depends(most cases bad though, because we're gonna have lots of rows).
If the table only has 2 rows(not considering seq scan is better here), it's good. But if the cardinality is very low and we have million rows,
that index isn't gonna help us narrow down to the val we're looking for. 
- selectivity: ratio. how many distinct vals are there **as a percentage** of the total num of rows in the table. The most selective, perfect
col is the primary key. It's selectivity is 1.00 . So 1 is perfect selectivity(meaning all vals of this col are unique). So the closer
to 1, the better selectivity of a candidate index. Meaning the more rows we'll filter out and we'll faster get to what we're looking for,
using this index. Selectivity is a good indicator **IF** the data is normally distributed. But if the data is highly skewed, we can benefit
from an index on that col in queries that are looking for that skewed data.

But to use an index:
- query
- data

These factors could hide a useful index on a col. For example on a bool col, the selectivity on the whole rows of a col might be very close to 0,
but on some query with `where x = true`, we could have a selectivity close to 1.

Calculate the cardinality and selectivity of birthday col:
```postgresql
select count(distinct birthday),                                             -- 10950. cardinality of birthday col
       (count(distinct birthday)::decimal / count(*)::decimal)::decimal(7, 4)-- 0.0111
from users;

-- so the selectivity of this col is very bad(because there are only 2 distinct vals in that col):
select (count(distinct is_pro)::decimal / count(*)::decimal)::decimal(17, 14)-- 0.0000002
from users;
```
Is that good selectivity val? Do we want to be closer to 0 or closer to 1.

So is indexing a bool col a good or bad idea? We still don't know! Because we still don't know what the query pattern is. There are queries
in which indexing a bool col is a good idea and queries where indexing it is terrible!

```postgresql
select count(*) filter ( where is_pro is true )
from users; -- 44382 (across 1M rows)

select ((count(*) filter ( where is_pro is true ))::decimal / count(*)::decimal)::decimal(17, 14)-- 0.0448
from users;
```
So the selectivity of is_pro bool col in cases where is_pro is true, is even better than our birthday col.

So while a col across it's whole rows might not be selective, but if it's data is quite skewed to a point AND that skewed portion is
what you're looking for, an index on it is helpful. In our case, we have about 1M rows and 44k that is_pro = true. Now if a common
query that we're running is: `where ... and is_pro = true`, putting an index on is_pro is helpful.

So both of these matters when deciding when to actually use an index:
1. query(access pattern)
2. data(skewness of the data)

You want your index to narrow down to just a few rows. That's why a primary key(not uuid v4) is perfectly selective,
because it gets you down to 1 row very fast.

PG doesn't run these queries in real time to determine if the index is a good candidate for usage or not.
It keeps statistics under the hood and they can be updated  by running `analyze` on the table or by `auto vacuum`.
These stats do get updated but if you do a massive UPDATE, INSERT or DELETE, you might want to update the stats **manually**. 

## 46.-Composite-indexes
Composite index: Creating one index on multiple cols at the same time, instead of multiple indexes on those cols.
Note that pg still has the ability to scan multiple **separate** indexes and then combine the results in an intelligent way.
So if you don't have a composite index that fits perfectly for a query, pg is gonna do it's best to help you out
by using the existing separate indexes. This is an awesome capability and not every db has it.

But we get better perf especially for sorting, if we have that composite index.

2 rules:
1. left to right no skipping(technical name of this rule: left most prefix).
Most btree impls, follow this rule. But There's a bit of a caveat with pg for this rule. Because pg has the ability to kinda skip.
2. stops at the first range

```postgresql
create index multi on users using btree (first_name, last_name, birthday);
```

```postgresql
explain
select *
from users
where last_name = 'Francis';
```

Doesn't use the composite index:
```
Gather (cost=1000.00..20045.17 rows=2086 width=76)
    Workers Planeed: 2
    ->  Parallel Seq scan on users (cost=0.00..18836.77)
            Filter: last_name = 'Francis'::text)
```

When declaring the composite index, the order matters a lot. But in the query, the order doesn't matter. But still, you have to obey the
left most prefix rule. So you have to include the cols from left to right of declared index, but their order in query doesn't matter,
**they just need to appear from left to right no skipping**. So if we do this query, it still **uses the index**, although 
last_name appears before first_name in the query:
```postgresql
select *
from users
where last_name = 'Francis'
  and first_name = 'Aaron';
```

If we skip a col, pg won't use the composite index:
```postgresql
select *
from users
where last_name = 'Francis'; -- left most col didn't appear, we skipped it. PG can't use the index
```
```
...
Parallel seq scan on users ...
```

But if we include the left-most col of the declared index, it uses the composite index, because we formed a left-most prefix
```postgresql
select *
from users
where first_name = 'Aaron';
```
```
Bitmap heap scan on users (...)
    Rechecked cond: ...
    -> Bitmap index scan on multi (...)
        Index cond: (first_name = 'Aaron'::text)  
```

But if we do:
```postgresql
select *
from users
where last_name = 'Francis'
  and first_name = 'Aaron';
```
```
Index scan using multi on users (...)
    Index cond: ...
```

But if we skip last_name, we didn't obey the left most prefix rule, so it shouldn't use the index, but it does use it with a caveat:
```postgresql
select *
from users
where last_name = 'Francis'
  and last_name = '1989-02-14';
```
```
Index scan using multi on users (...)
    Index cond: ...
```
What is happening here is a bit of pg optimization. Normally if we had an index on just first_name and birthday(no last_name in between),
it navigates in the index to rows that matches the cond. But here, we have a col in index in the middle, but not in the query.
Now pg says: I can still use the first col which is also in the query, so I will navigate through the btree to the chunk of leaf nodes
that have the first_name = 'Aaron'. I can get there by just using the index. However, since last_name is in the middle of the declared index
but not in the query, then I'm gonna scan **all the way through the leaf nodes** to look for rows with and `birthday = '1989-02-14'`.
So instead of directly traversing to the exact rows, it's doing a traversal down to the rows matching the first col cond(which narrows
down the results quite a bit) and then it scans through the leaf nodes looking for rows matching `birthday` cond.

Why it does that? Because the tree structure is not set up to jump over the levels using last_name, since we didn't include last_name in the query.

But if we don't skip the last_name, it's just using the btree traversal, it doesn't have to scan all of those leaf nodes at the bottom.

Note: We don't want a big index scan at the leaf nodes. If we form the left most prefix, it limits the amount of the index that must be
scanned.

```postgresql
select *
from users
where first_name = 'Aaron'
  and last_name = 'Francis'
  and birthday = '1989-02-14'; -- doesn't do index scan at the leaf node
```

Order of efficiency:
- direct btree traversal
- index scan
- table scan

---

```postgresql
create index multi2 on users using btree(first_name, birthday);

select *
from users
where first_name = 'Aaron'
  and birthday = '1989-02-14'; -- uses the multi2 index not multi, because that's more efficient
```
```
Index scan using multi2 on users ...
...
```

So your most common conditions(conditions that are common in a lot of queries) need to go on the left side of the declared index.
So if you have many queries that have first_name in the cond, but only some of them use birthday, put `first_name` towards the left
and `birthday` more at the right of the declared index.

## 47.-Composite-range

## 48.-Combining-multiple-indexes
## 49.-Covering-indexes
## 50.-Partial-indexes
## 51.-Index-ordering
## 52.-Ordering-nulls-in-indexes