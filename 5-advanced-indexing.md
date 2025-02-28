
## 53.-Functional-indexes
Putting index on the result of a function or an expression. 

Similar to generated cols, but instead of storing that val in a col, we're gonna store the value in a btree(i.e. an index).
So that we don't have to do the computation of that value, since the result of func is stored in the btree.

Everytime the row is INSERTed, UPDATed or DELETEd, that func is evaluated and the res of that val is sent off to the btree.

Extracting a domain off of an email.
```postgresql
-- the second pair of () is important. If you don't put it, it indicates that you're writing a col name. But with another (),
-- we can put an expression there. Here, we wanna index the result of that expression.
create index domain on users((split_part(email, '@', 2)));

explain select email
from users
where split_part(email, '@', 2) = 'beer.com'
limit 10;
```
```
Limit (cost=0.00..57.64 rows=10 width=76)
    ->  Index scan using domain on users (cost=0.00..28529.62 rows=4950 width=76)
            Index cond: (split_part(email, '@'::text, 1) = 'beer.com':: text)
```

One use case is indexing json keys. We can extract keys that we commonly query for, out of a big blob and pull those out into
a functional index on their own.

### Case insensitive indexing
```postgresql
create index email_lower on users((lower(email)));

select *
from users
where lower(email) = lower('AARON.FRANCIS@example.com'); -- uses the email_lower index
```

## 54.-Duplicate-indexes
You might accidentally have a duplicate index.
```postgresql
create index email on users(email);

-- after some time, we add this composite index for email and is_pro:
create index email_is_pro on users(email, is_pro);
```
But we created a duplicate index. These indexes have the exact same left-most prefix(email col). So when we solely query for email,
we encounter two indexes that can be used. PG will prefer the smaller or more compact index(single col index).
But if email idx didn't exist, it can use email_is_pro idx. So these indexes are duplicate.

So indexes that share a left-most prefix, are considered duplicate. Get rid of the smaller ones(since they all share a left most prefix,
we can use the composite or larger ones). Because the composite ones, can satisfy the single-col or smaller indexes but not vice-versa.

## 55.-Hash-indexes
Only useful for strict equality lookups.

So can't be used for range searches, ordering(sorting), wild card searches(no partial matches) or ... . Because the val that is being
indexed, is gonna run through a hash func, so the semantic meaning of the val is lost on the other side. That's why we can't do range
lookups and ..., because what exists in the index DS, is not the value, it's the hashed version of that val.

This is similar to creating a btree index on a md5 hash of a col, except it's not a btree, it's fundamentally different DS that's optimized
for storing these hashes.

Another benefit of a hash index is the result is constant size. So we can run a massive piece of text or data through that hash func and
it comes out as the same size, so the index structure stays quite compact.
```postgresql
create index email_btree on users using btree(email); -- could use the shorter syntax, since in shorter syntax btree is used(default)

create index email_hash on users using hash(email);
```

Now if we run a query with `where email = `, it's gonna use the hash index instead of btree, because hash one is faster for strict
equality lookups. But if we do range query or use `like`, it won't use the hash and will probably use the btree one.
```postgresql
explain
select *
from users
where email = 'aaron.francis@example.com';
```
```
Index Scan using email_hash on users (cost=0.00..8.02...
    Index Cond: (email = 'aaron.francis@example.com': : te...
```

Note: Prior to postgres 10, hash indexes shouldn't be used.

So when you need strict equality on potentially cols with large data, use hash index.

## 56.-Naming-indexes
Index names are not global to a db, but they are global to schema.

For example: If we have two tables both having index on their email col, we can't name both indexes `email`.

`tablename_column(s)_type`.

Type is:
- idx: like `users_email_idx`
- unique
- check