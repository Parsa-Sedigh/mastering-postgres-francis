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
## 43.-Primary-key-types
## 44.-Where-to-add-indexes
## 45.-Index-selectivity
## 46.-Composite-indexes
## 47.-Composite-range
## 48.-Combining-multiple-indexes
## 49.-Covering-indexes
## 50.-Partial-indexes
## 51.-Index-ordering
## 52.-Ordering-nulls-in-indexes