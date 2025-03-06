## 57.-Introduction-to-explain
When you issue a query to db, you're telling it what to do, you're not telling it how to do it.

## 58.-Explain-structure
Explain shows the estimates. The actual reporting of a query that ran, use analyze + explain, but it's actually gonna **run** the query,
even if it's an UPDATE or DELETE.

An `EXPLAIN` plan is a tree structure that's made up of nodes. So it's a hierarchical structure.

A `Scan` node produces rows. There are couple different kinds of `Scan` nodes, there are also some other kinds of nodes that
produce rows as well.

Read this tree structure not from top or bottom up, but from **inside out moving up**, because child nodes emit the rows to their
parent nodes and ... . Don't read it top to bottom. The very top row is in fact the very last thing.

```postgresql
explain select * from users
limit 10;
```
```
Limit (cost=0.00..0.24 rows=10 width=76)
    -> Seq Scan on users (cost=0.00.23580.08 rows=989908 width=76)
```

Here, it thinks(based on statistics) that it's gonna get 989908 rows. Then it's gonna pass it to `Limit` node.

You can do:
```postgresql
explain (format json) select * from users
limit 10;
```
This is not human readable, but machine parsable.

There's another type of indented row but not indented with an arrow:
```postgresql
explain
select *
from users
where first_name = 'Aaron'
Limit 10;
```

```
Limit (cost=0.00..796.78 rows=10 width=76)
    -> Seq Scan on users (cost=0.00..26054.85 rows=327 width=76)
         Filter: (first_name = 'Aaron':: text) -- I'm taking about this indented row. But not indented by an arrow.
```

To see if we can get more info without weird indentation, we can run `explain (format json)` on the same query.
There, we see `Node type: Seq scan`, doesn't have a sub node of `Filter`. It just has an attr for `Filter`.

So an arrow(->) indicates this is a new node(sth discrete), but an indentation with spaces tells: hey this is sth you should care
about with regard to the current node.

## 59.-Scan-nodes
3 types of scan nodes:
1. index scan
2. bitmap index scan
3. sequential table scan: When query planner decides to use this one, it means it's gonna read a large portion of the table(potentially entire
table). So it makes sense to read this table in **physical disk order**. Note: Under the hood, in a table, there a bunch of pages
and the rows are written in scattered manner wherever there was space. So DB has to decide:
    1. Should I read all of those pages in physical disk order? Which actually would be fast in terms of not having 
        to do a lot of random IO(it's not jumping around to different pages which
        are in different locations of disk). Seq scan means using this approach.
    2. OR should I go to an index, find some row locations(CTID) and then go grab them out of pages?(index scan)

### Bitmap index and heap scans

Assume there's an index on email field. With this query, query planner uses a bitmap index scan + bitmap heap scan instead of index scan:
```postgresql
explain select * from users where email < 'b';
```

```
Bitmap Heap Scan on users (cost=3356.31.. 18398.43 row...
    Recheck Cond: (email < 'b':: text)
    ->  bitmap index scan on email_btree (cost=...)
           Index Cond: (email < 'b'::text) -- shows when it's going over the index, this is the condition
```

Q: What is `Bitmap Heap Scan`?

A: Pg decides: I don't need to read the whole table(seq scan), it's better to go over the index and get some info out. But the information
that I need to get out of index is still quite large. I still do need to go check a whole bunch of pages in the heap.

This is still better than seq scan on the table. Here's what's actually happening:

It goes to the index and it finds all of the emails that are: < 'b'. Then it constructs a map(bitmap) that says: according to this index,
here are all the pages and row addresses that match that `index cond`. Then the `bitmap index scan` emits this map up to the `bitmap heap scan`.
So these two `bitmap index scan` and `bitmap heap scan` always work together.

Now after it's done in the index, it means it has constructed this map that says: Here's all the row addresses that matched. Go get them.
Then it does a bitmap heap scan: I got a lot of work to do. I'm gonna put all these addresses in physical order. 
Note that when having physical disk order, we won't be jumping around the disk.

So the bitmap heap scan takes the map and puts it in physical order and then gonna read through the table in 
physical order visiting all of the addresses that are in that map.

Bitmap heap scan has `Recheck Cond: (...)` attr. Note that in prev result of explain, we check the cond twice.
This is because when `Bitmap index scan` emits the bitmap, it might not be perfect. The reason for not being perfect could be:
1. the bitmap index scan emitted too many row addresses and eventually it just says: there are some rows in the page x. Go check them.
   So then during the heap scan, it goes over to that page and it rechecks the cond to make sure that it's only pulling the rows that
   satisfy the cond. So the recheck condition happens on the heap.
2. The index that we got the bitmap from, don't support some operations, so it didn't consider some of the valid row addresses. So during the
bitmap heap scan, we recheck the cond of the rows to make sure we get all.

So when you see `Rechecked cond: (...)`, it's because potentially the node below it produced too many results in the map, so the map
became lossy or potentially the node below it never could produce a perfect map and it didn't consider some of the correct rows(based on the
type of index that we got the bitmap from).

The reason it produces a map is because there are a lot of rows to look for. Since there are a lot of rows, the random IO will be so
much expensive. So it produces a map, sort it and fetch it in order.

### Index scan
```postgresql
explain select * from users where email = 'aaron.francis@example.com';
```

There's only 1 node here:

```
-- the `rows` here is just what query planner THINKS about the number of rows. Not the actual rows num. But in this case, it's right.
Index scan using email_hash on users (cost=... rows=1 width=76)
    Index Cond: (email = 'aaron.francis@example.com'::text) -- this is an attr of `index scan` node above. Shows the cond during the above node
```
Index scan means: There are so few estimated rows(in this case 1), that it can go read them in random order(performs random IO).
It doesn't care about IO penalty, because they are a few of them. So there's no map creation, sorting the key-vals in map and reading
the rows in physical order.


### Covering index scan
Note: We dropped the hash index on email and created a btree on that col to get a covering index scan here.

```postgresql
explain select email from users where email = 'aaron.francis@example.com';
```

```
Index only scan using email_btree on users (cost=... rows=1 width=76)
    Index Cond: (email = 'aaron.francis@example.com'::text)
```

### Summary
From worst to best:
- seq scan: read the entire table in physical order. If query planner thinks it's gonna have to read a large portion of the table,
it doesn't even make sense to waste time over in the index. So it goes straight for a seq scan on the heap. This may be the fastest
way to do the query, especially on relatively small tables. It doesn't make sense to visit index and then get a lot of row locations and then
do random IO. It would be slower than seq scan on heap.
- bitmap index scan: scans(traverses) the index, produces a bitmap, then reads pages in physical order to prevent random IO(jumping around the disk).
index scan + map creation + sorting of map + visit a large portion of rows in heap but in physical order
- index scan: scan the index, get the rows(random IO). So index scan + heap scan(in random order)
- covering index. Doesn't visit the heap at all

The main thing that the query planner uses in node selection to choose which type of scan to use, is how many rows it 
estimates it's going to have to emit to it's parent node.

## 60.-Costs-and-rows
There are some info in () for each node.

```postgresql
explain select * from users;
```
```
-- for the cost, the startup cost is 0.00 and the total cost is 23.. .
Sea Scan on users (cost=0.00..23580.08 rows=989908 width=76)
```

Let's try `explain (format json)`:
```
[
    {
        "Plan": {
        "Node Type": "Seq Scan",
        "Parallel Aware": false,
        "Async Capable": false,
        "Relation Name": "users",
        "Alias": "users",
        "Startup Cost": 0.00,
        "Total Cost": 23580.08,
        "Plan Rows": 989908,
        "Plan Width": 7E
    }
]
```

Width: how wide in bytes each row is estimated.

Total cost: A total cost of a node includes the cost of it's children. The total cost of the very top node is the total cost of
the query that the planner is trying to bring down. Whichever plan the query planner comes up with that has the lowest cost,
that's the one that it goes with.

The cost unit is arbitrary but they're consistent. So we can't say it's milliseconds or ..., it's just units. It's made up of
a couple of different factors, like: the cost to read a sequential page, the cost to read a random page, some cpu costs and ... .
These are all tunable.

Startup cost: how long this node can get started. The first node has startup cost of 0.00 .

```postgresql
explain select * from users where email < 'b';
```

```
Bitmap Heap Scan on users (cost=3356.31..18398.43 rows=108889 width=76)
    Recheck Cond: (email < 'b':: text)
        -> Bitmap Index Scan on email_btree (cost=0.00..332g.09 rows=108889 width=0) -- startup cost is 0. This is why we read from inner most to outer.
              Index Cond: (email < 'b'::text)
```

As you see, the startup cost of outer node is a bit higher than total cost of the inner node. Because the outer node is waiting for
the inner one to finish.

The `rows` is not the number of rows that need to be visited or inspected. That's the num of rows it thinks it will emit to the parent node.
It might actually inspect order of magnitude more rows.

If you look at the result carefully, the `rows` num didn't change, which means the Recheck Cond didn't eliminate any rows which means
the bitmap created in bitmap index scan step was actually perfect!

Note: If you expect to end up with just a few rows at the end, but the result of explain is constantly showing millions of
rows emitting up to the parents, maybe there's a better way to limit the amount of rows that's being worked on the inner nodes
before it gets out to the outer node. Because if you're carrying around the rows that you don't need. you're doing extra work.

The parent nodes have the sum of their child nodes. But look at this example where the Limit node has less total cost than it's child:
```
Limit (cost=0.00..2.39 rows=10 width=76) -- cost has decreased!
    ->  Seq Scan on users (cost=0.00..26054.85 rows=108889 width=76)
          Filter: (email < 'b'::text)
```
This is because the `Seq scan` can end early(stops after 10 rows) because of the `limit` and here, we don't have an `order by`.
If we had an `order by`, `Seq scan` **had to** scan all the rows since there's a certain sorting for them and it doesn't know which
rows will end up at the top to limit them. In that case, the cost of the Limit node would also be higher than Seq scan node.

So if there was an `order by` in this query, it has to scan the entire table, then sort it and then apply the limit.
So a limit doesn't **always** decrease the cost. It decreases the cost **if** the query can end early.
The seq scan can end early if it's able to read the rows in order or if it doesn't need ordering at all(which is the case here).

**So when there's limit with no ordering, it can change the rule of top level node being inclusive on it's children nodes(having a greater
cost than it's children).**

## 61.-Explain-analyze
```postgresql
-- we can omit the parentheses as well.
explain (analyze) select * from users where email < 'b';
```

```
Bitmap Heap Scan on users (cost=3356.31..18398.43 rows=108889 width=76) (actual time=23.064..36.601 rows=98778 loops=1)
    Recheck Cond: (email < 'b'::text)
    Heap Blocks: exact=13673
    -> Bitmap Index Scan on email_btree (cost=0.00..3329.09 rows=108889 width=0) (actual time=20.589..20.
          Index Cond: (email < 'b'::text)
Planning Time: 3.410 ms
Execution Time: 38.712
```

The `cost` and `time` in `actual` have fundamentally different units. The cost is postgres arbitrary cost units
but the time is in milliseconds.

```postgresql
-- only shows the actual time, not the costs
explain (analyse, costs off)
```

### Loops
Each noce could be run multiple times. It could be run once for every row in the result set. Because of this,
the `costs` shows the cost per iteration based on an avg.

If you can't find the issue with explain, you might reach for explain analyze.