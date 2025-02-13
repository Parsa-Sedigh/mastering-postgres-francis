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


## 45.-Index-selectivity
## 46.-Composite-indexes
## 47.-Composite-range
## 48.-Combining-multiple-indexes
## 49.-Covering-indexes
## 50.-Partial-indexes
## 51.-Index-ordering
## 52.-Ordering-nulls-in-indexes