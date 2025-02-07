## 39.-Introduction-to-indexes
The best way to unlock a performant db. It's a separate DS that maintains copy of part of the table and it's optimized for some ops.
It contains pointers back to the table and the row.

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


## 41.-B-Tree-overview
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