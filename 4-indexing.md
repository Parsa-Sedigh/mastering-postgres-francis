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

So if the data distribution is highly skewed in one direction, it might still be a good idea to put an index on it depending on the query.
Another option is using partial index(pg and sqlite have but not mysql).

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

If any of these are encountered, other dbs won't use the index after that, but pg continues to use the index but not traversing it,
just scanning(which is not as efficient):
1. left to right no skipping(technical name of this rule: left most prefix).
Most btree impls, follow this rule. But There's a bit of a caveat with pg for this rule. Because pg has the ability to kinda skip.
2. stops at the first range: still pg uses the index up until the first range cond is encountered. After that, it scans the index

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

So pg does have the ability to go from a btree traversal(when using an index) to a scan, if you skip over a col.
While this is not ideal, pg does cover you. The same thing applies for range conditions.

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

**So your most common conditions(conditions that are common in a lot of queries) need to go on the left side of the declared index.
So if you have many queries that have first_name in the cond, but only some of them use birthday, put `first_name` towards the left
and `birthday` more at the right of the declared index.**

## 47.-Composite-range
```postgresql
create index first_last_birth on users using btree(first_name, last_name, birthday);

create index first_birth_last on users using btree(first_name, birthday, last_name);
```

```postgresql
explain select *
from users
where first_name = 'Aaron'
  and last_name = 'Francis'
  and birthday < '1989-12-31';
```
Is using first_last_birth:

```
Index scan using first_last_birth on users (...)
Index cond: ((first_name = 'Aaron'::text) and (last_name = 'Francis'::text) and (birthday = '1989-12-31'))
```

Why it didn't use first_last_birth instead of first_birth_last index? Because that's the most efficient btree index for this query.

In this query, pg will traverse the nodes of the index using the first two strict equality checks. After that, we encounter a range
condition. We don't know how to traverse the index by that condition. So we go to the very first node that matched those strict equality checks
and we start **scanning** until we reach birthday: 1989-12-31 and that is our chunk of rows(since we're using less than), so we go
grab the full info from the heap.

> **So the first time pg encounters a range condition, immediately it starts scanning the index(not traversing it).**

Which is why you want your left most prefix to be commonly used strict equality conditions and then as you move to the right of your declared index,
you're gonna have less commonly used equality **or** your range conditions. Because if you skip over a col or you encounter a range condition,
then it starts scanning the index not traversing(which is not as efficient as traversal which is done on strict equality).

## 48.-Combining-multiple-indexes
PG has your back if you have multi separate indexes. It can scan those indexes and put the results together, even if it's less performent
than composite index, it's still better than a table scan and usually it's better than reading one index and then doing remaining of
the filtering after pulling rows out of the heap.

```postgresql
-- individual indexes:
create index "first" on users using btree(first_name);

create index "last" on users using btree(last_name);

-- composite index
-- create index "first_last" on users using btree(first_name, last_name);
```

```postgresql
explain
select *
from users
where first_name = 'Aaron'
  and last_name = 'Francis';
```

```
Bitmap heap scan on users (...) -- get the rows out of heap
    Rechecked cond: (...)
    ->  BitmapAnd (cost=... rows=1 width=0) -- if the cond was using `OR`, this would be: BitmapOr
        -> Bitmap index scan on first (...)
            Index cond: (first_name = 'Aaron'::text)
        -> Bitmap index scan on last (...)
            Index cond: (last_name = 'Francis'::text)
```
Once the Bitmap index scans are completed, it goes to BitmapAnd. What's happening is: 
1. pg is using the `first` index and finding all the rows that match
2. Does the same for `last_name` cond with `last` index.
3. combining them(bit mapping them) by only considering rows that both conds are true for them

Now if you also create the composite index, in addition to those individual indexes, if you run the query again, pg will use
first_last composite index not those individual indexes:
```
Index scan using first_last on users (...)
Index cond: ...
```

Caveat: If you run the query with `OR`(first_name = 'Aaron' OR last_name = 'Francis'), pg still decides to use the individual indexes
instead of composite index! The way that a btree is strucutred, this or cond is hard to satisfy with a single btree.
So test on your own schema, query patterns and data.

### Summary
So combining multiple separate indexes might be the strategy you're relying on and that's because pg is prefering it over composite
in some cases. But in most cases, when you're `AND`ing the conds, a single composite index is gonna perform better, but you might not
always be `AND`ing them together and you might `OR`ing them in which separate indexes combined with BitmapOr is more performant and is used.

## 49.-Covering-indexes
Instances where indexes are used: 
1. single index over a single col
2. single index over multiple cols(composite index)
3. single indexes over single cols that are combined together using bitmapAnd(or bitmapOr)

In all these cases, pg is using an index to find row addresses(CTIDs) and then go to the heap, potentially rechecking
the conds over there and grabbing the rows from heap.

With covering indexes, we don't have to make that journey back to the heap to grab the full data of rows, because everything
the query needs(cols in SELECT, WHERE, GROUP, ORDER and ...) is on the index itself. This is not sth you can rely on all the time.

```postgresql
create index "first" on users(first_name);
```

```postgresql
explain select *
from users
where first_name = 'Aaron';
```

```
Bitmap Heap scan on users (...) -- still going to heap for getting all data it needs
    Rechecked cond (...)
    ->  Bitmap index scan on first (...)
        Index cond: (...)
```

Here, it won't go to the heap at all:
```postgresql
explain select first_name
from users
where first_name = 'Aaron';
```
```
Index only scan using first on users (...)
    Index cond: (...)
```

Note: A convering index is not a special type of index, it's a regular index(single col index, composite index) in a special situation.
It depends on the query and if all the requirements are satisfied with only using the index and not heap.

---

You can add a piece of data alongside the index without using it for indexing.
You can add that piece at the end in pg, but in most other DBs, you have to add it at the end.

Does not include id at the btree DS(not included in the btree traversal), instead, it shoves id down at the leaf nodes and says:
Hey, once you get to the bottom using the btree structure, there's a bit of extra data down there(col `id`).
```postgresql
create index on users(first_name, last_name) include (id);

explain
select first_name, last_name, id
from users
where first_name = 'Aaron'
order by last_name; -- still `index only scan`
```

What's the drawback of this? Why not do this all the time? Why not just include a bunch of cols that we don't need in the index
but we just want for having a covering index situation?

1. You're gonna bloat up your btree, if you include a bunch of extra col there(especially large cols like text or json that could have
large data), just because you think you might get a covering index. Kinda you're recreating the table there!
2. we know if there are lots of writes to the table, pg has to go back to index and fetch it out of table anyway. In covering index situation,
there's a step in which pg will check: Hey I got all these rows out of the index, however, I don't know if those rows are visible
to the current tx right now. So it gets the CTIDs from the index and it checks a visibility map. And if the table is written to(changing) often,
that visibility map is gonna say: Hey, those rows aren't visible to you, so it has to go check the heap anyway(therefore we get no covering index
in that case). So for a table with high write throughput, you might not hit covering indexes ever!

Note that when using covering index, checking the visibility map does incur some cost but it's still much faster than checking the table.
This is because when not in covering index situation, pg will go check the heap anyway.

These are very good for certain hot paths.

### Chatgpt on covering index in high-write table:
Why Does PostgreSQL Check the Heap?

PostgreSQL does not store visibility information in the index. Instead, it stores visibility metadata in the heap (table data). 
This is different from databases like MySQL’s InnoDB, where indexes contain transaction visibility information.

When using an Index-Only Scan, PostgreSQL tries to fetch all necessary data from the index without accessing the heap.
However, before it can return the data to the query, it must ensure that each row is visible to the current transaction. This check is
performed using the visibility map.

How the Visibility Map Works
- The visibility map keeps track of whether a page in the heap contains only tuples (rows) that are visible to all transactions.
- If a page is marked as “all-visible”, PostgreSQL can skip checking the heap and return the index-only scan result directly.
- If the visibility map does not mark the page as all-visible, PostgreSQL must fetch the row from the heap to check if it is valid for the current transaction.

What Happens in a High-Write Table?
- Every time a row is INSERTED, UPDATED, or DELETED, the visibility of tuples changes.
- PostgreSQL must clear the all-visible flag for the affected heap pages.
- This means the next query cannot rely on the index alone and must check the heap for visibility.
- As a result, index-only scans become ineffective, forcing PostgreSQL to go back to heap pages frequently.

## 50.-Partial-indexes
Put an index on a **portion** of the table. Even putting a unique constraint over a portion of the table.

```postgresql
-- add a predicate to make it a partial index
create index email on users (email)
    where is_pro is true;
```
Here, the btree DS is not being created as a composite index, it's still an index on a single col.

Now if you try to get a row that matches the predicate of declared index, pg won't use the index!
```postgresql
explain
select *
from users
where email = 'aaron...'; -- this row has is_pro = true
```
```
Gather (...)
    Workers planned: 2
    ->  Parallel seq scan on users
        Filter: ...
```
So it doesn't use the index. This is because we haven't matched the query to our index predicate. If we do:

```postgresql
explain
select *
from users
where email = 'aaron...'
  and
    is_pro is true; -- this part matches the predicate of the declared index
```
Now is using the index:
```
Index scan using email on users (...)
    Index cond: ...
```

Note: Pg won't use the index, if we had `is_pro is false` in the query. Because the predicate doesn't match.

Note: When you have a massive table but just a few rows are interested to you for querying fast, that's a great use case for a
partial index. So we prevent the data we don't want from entering the btree in the first place. That's gonna make the btree small
and won't slow the writes because we're not maintaing a massive btree with values that we don't care and also since the btree is smaller,
the reads that use the index will still be fast.

### Partial unique index
We can't create a unique index on a col that already has duplicate val. But we can create a partial unique constraint.
This allows in cases where we have a soft-deleted row, but he wants to signup again.

Another example: could an order only have 1 state or can it have multiple states? We can answer this question based on the business needs
using unique index.

```postgresql
create unique index email on users (email) where deleted_at is null;
```

### Summary
Remember to include the predicate that was used in declared partial index, in the query as well. Otherwise the planner won't use that
partial index.

## 51.-Index-ordering
We should create an index(especially for composite index), in the order that we wanna read it. So if we're commonly sorting a col descending,
you can create an index over that col in descending order.
```postgresql
select * from pg_indexes where tablename = 'users';

create index created_at on users(created_at);
```

If we order by created_at in asc(default):
```postgresql
select *
from users
order by created_at
limit 10;
```
```
Limit (...)
    ->  index scan using created_at on users (...)
```

But if we order by desc:
```
Limit (...)
    ->  Index scan Backward using created_at on users (...) -- notice `Backward`
```
So pg has the ability to read the index from front to back(asc) or start at the end and read from back to front(when order by in desc).
So it has the ability to do backwards index scan. 

> However, when you have a composite index and you're sorting them in different directions, pg can't use the index anymore. So you have
> to create the index in the order you'll read.
> Note: If all of the cols have the same direction(all asc or all desc), pg uses the index.

```postgresql
-- composite index
create index birthday_created_at on users (birthday, created_at);

explain
select *
from users
order by birthday, created_at
limit 10; -- uses the index: Index scan using birthday_created_at ...

-- if we change both to desc, it's still using the index
explain
select *
from users
order by birthday desc, created_at desc
limit 10; -- index scan backward

explain
select *
from users
order by birthday desc, created_at
limit 10;
-- limit ...
    --  -> incremental sort ...
            -- Sort key: birthday desc, created_at
            -- presorted key: birthday
            --   index scan backward using birthday_created_at on users ...
```
So we got an `incremental sort`. That's because an index is one singular DS, pg can't read part of it desc and part of it ascending, so 
it can't read part of it forward and part of it backwards. So pg has to do gathering phase and `incremental sort` after.

To fix this, create the index in the order you want:
```postgresql
create index birthday_created_at on users (birthday, created_at desc);
```

Now the query will use the index.

Note: If you run this query, it will be using the index in backward, because the query uses the cols in complete opposite of 
the cols in declared index.
```postgresql
explain
select *
from users
order by birthday desc, created_at
limit 10; -- index scan backward ...
```
But you can't just flip one of them, it won't use the index: `birthday desc, created_at desc -- gets incremental sort`.

## 52.-Ordering-nulls-in-indexes
By default NULLs are treated as larger than any other value, but we can change that both in the query and in the index construction.

In thi squery, nulls would be at the beginning. If it was asc though, nulls would be last, since they're larger than all other vals.
```postgresql
select * From users
order by birthday desc
limit 10;
```

So when the order by direction is `desc`, `nulls first` is the default. So here, nulls first is redundant: `order by x desc nulls first`.
And also here: `order by x nulls last`.

```postgresql
create index birthday_null_first on users(birthday nulls first);

explain select * From users
order by birthday nulls first
limit 10;
```
```
Limit (cost=0.42..1.16 rows=10 width=76)
    -> Index Scan using birthday_null_first on users (cost=0.42..72992.27
```
Note: If we `order by birthday desc nulls last`, it will use the index in backward:
```
Limit (cost=0.42..1.16 rows=10 width=76)
    -> Index Scan Backward using birthday_null_first on users (cost=0.42..7299..
```

But if we change it to `nulls last`, or sort in `desc` dir it won't use the index:
```
Limit (cost=27718.80..27719.96 rows=10 width=76)
    -> Gather Merge (cost=27718.80.. 123966.65 rows=824924 width=76)
        Workers Planned: 2
        -> Sort (cost=26718.77..27749.93 rows=412462 width=76)
            Sort Key: birthday
            -> Parallel Seq Scan on users (cost=0.00..17805.62 rows=412462..
```

So if you find yourself reaching for nulls first last in a query often, you might consider creating an index that represents 
that same exact order.