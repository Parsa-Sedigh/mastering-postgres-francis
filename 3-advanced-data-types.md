## 23.-Intervals
Instead of a discrete point in time, is a duration(period). Interval is like a duration.

Intervals are good for this because of the way it stores the vals in discrete units under the hood to handle edge cases.

- If you want to find out two or more ranges overlap, or rather you want to prevent bookings from overlapping, intervals can make that query
easy.
- find out if a point in time exists inside a given interval or duration.

### ways to create intervals
1. using pg standard format `'[unit] [quantity]'` which then need to be cast to interval
2. `interval '[unit]' [quantity]`

#### First way
```postgresql
select '1 year 2 months 3 days 4 hours 5 minutes 6 seconds'::interval; -- 1 year 2 mons 3 days 04:05:06
```
The query above shows we can write the hour part and below, into less verbose format:
```postgresql
select '1 year 2 months 3 days 04:05:06'::interval;
```

Note: Like `datestyle`, `intervalstyle` is configurable:
```postgresql
-- this just controls the output style
show intervalstyle; -- postgres

set intervalstyle = 'iso_8601';

-- with this new style, if we run the query, we would get a shorter result.
select '1 year 2 months 3 days 04:05:06'::interval; -- P1Y2M3DT4H5M6S
```
The result of query in more readable format: `P Y2M3D T 4H5M6S`

Note:
- `P` is for iso_8601 standard.
- `T` denotes the start of the time

There's an alternate iso_8601. It still starts with P but then it looks like a timestamp:
```postgresql
select 'P0001-02-03T04:0506'::interval -- 1 year 2 mons 3 days 12:26:00(the interval style is postgres)
```
Yeah 0001 is a very early year but here, it means 1year duration.

#### Second way
```postgresql
select interval '1' year;

select interval '1-6' year to month; -- 1 year 6 mons. The first digit is for year and second is for month

select interval '6000' second; -- 01:40:00. So 6000 seconds is 1 hour and 40 minutes
```

The way postgres stores intervals under the hood is it stores discretely sum of the date parts. Because let's say `'2 month 12 days'`.
You can't really convert that to days. You can't say: Well a month has 30 days ... . Nope! A month doesn't always have 30 days.
So under the hood pg is gonna store them in discrete units such that it's still gonna work across daylight saving times, months that
are short or long due to leap years and ... .

## 24.-Serial-type
It's an alias that creates a real datatype(integer), then it does two or three other things. Uses sequences under the hood.

Most often `serial` type is used for primary key. That's still fine, but since postgres v10 there's a better way to create an auto-incrementing
primary key, which is `identity` type which is simpler and has fewer caveats related to permissions.

**Don't use `serial` type for primary keys anymore, use `identity` type. If you're still using `serial` for primary key, at least prefer `bigserial`.**

```postgresql
create table serial_example (
    id bigserial primary key
);

-- what `serial` actually does?

-- 1. creates a `sequence`
create sequence serial_example_id_seq as bigint;
-- 2.
create table serial_example (
    id integer not null default nextval('serial_example_id_seq')
);
-- 3. the sequence is owned by the id col of serial_example table, such that when you drop the table or the col, the sequence is also dropped
alter sequence serial_example_id_seq owned by serial_example.id;
```

Note: 
- Corresponding actualy datatype for `serial` is `integer`.
- the auto incrementing part is: `default nextval('serial_example_id_seq')`
- the only place to be extremely generous with the room to grow in datatypes, is in defining schemas, is primary key col. In other places,
pick the smallest datatype. Because you don't want to run out of room for primary keys

### Gaps in sequences
We could end up with gaps in sequences even if we never deleted any rows! Gaps are totally fine though.
If you have a business use case for strictly not having gaps, this isn't gonna work, you have to roll your own solution.

Why we could have gaps?

Because `nextval()` doesn't work in a transaction-aware or transaction-safe way which is a good thing! Because let's say we have
two tx concurrently and since we assumed that the seq is tx-aware, it would return the same val in both of the transactions.
Now when txs want to commit, we've got the same val for both txs and therefore we're gonna run into duplicate key error for primary key.

But we know seq is not tx-aware. So each call to nextval() is gonna return the next val regardless of being in tx or not.

Let's say one tx rolls back, that would cause a gap in the seq.

Note: Make sure to use the `primary key` keyword, that would prevent duplicate vals(and null) for that col. Because while the seq itself
won't have duplicates(always returns next val), we can reset the seq ourselves and we can also insert a val by generating it **ourselves** instead of
leaving it blank in `INSERT` or calling `nextval(<seq name>)` and that would cause a duplicate val.

### Use sequence as an order number generator instead of primary key
```postgresql
create table orders
(
    id            bigint generated always as identity primary key, -- preferred datatype for auto incrementing primary keys
    order_number  serial,
    customer_name varchar(100),
    order_date    date,
    total_amount  numeric(10, 2)
);

-- we don't insert either id and order_number because both have default val that are auto incrementing.
-- NOTE: To avoid using a SELECT to get the values(especially good for auto-generated cols), we can use postgres RETURNING.
-- Sometimes we can't even get the vals back because they are not unique and we have no way of using WHERE in the next SELECT,
-- so we have to use RETURNING.
insert into orders(customer_name, order_date, total_amount)
values ('aasdasd', '2024-09-24', 150.00),
       ('aasdasd 2', '2024-09-25', 200.00),
       ('aasdasd 3', '2024-09-24', 75.00)
       returning *;
```

You could change the start number of sequences.

You can potentially change your ids if they aren't autoincrement integers, like uuids, but in that case, you might want something prettier
to show your customers. We can also use a generated col to have an ordered number.

## 25.-Sequences
Sequences don't need to necessarily be created by using a serial.

```postgresql
create sequence <name> as <data type>
increment 1 -- 1 is default

-- setting start is important when importing data from another system that already has some sort of sequence or pseudo-seq. You can look and
-- see what the max number is there and then add 1 to it and say that's my start val for our sequence.
start 1
minvalue 1
maxvalue 200
      
-- tells postgres how many of sequence vals it has to cache before it has to go pull new ones. We usually leave it as 1. 
cache 1;

-- pulls a value off seq and increments
select nextval('<seq name>');

select currval('<seq name>');
```

Note: currval() returns the current val of the **current session**! So if one session calls nextval() with whatever op and 
then another session calls currval(), it won't get the latest global counter, it returns the curr val of it's own session.

So if session A is at 3 and another session calls nextval() 5 times, now if session A calls currval(), it won't get 8, it still
gets 3!!!

What does that mean?

It means inside a single session or transaction, you can pull a val off of a seq using nextval() but then we can use it over and over again
without assigning it to some var, so using currval(). So currval() doesn't give you the current **global** val! 
In fact, if you haven't run nextval() at all, running currval() will give error, because there is no current value because you've never run
nextval().

```postgresql
-- NOTE: resets the seq to 1. Be careful with this. Because if seq is used in primary key using serial, it would cause duplicate entry error.
-- NOTE: You can also set it to higher number and the next calls to nextval() will start from there too(the next one of the val we set).
select setval('<seq name>', 1);
```

## 26.-Identity
## 27.-Network-and-mac-addresses
## 28.-JSON
## 29.-Arrays
## 30.-Generated-columns
## 31.-Text-search-types
## 32.-Bit-string
## 33.-Ranges
## 34.-Composite-types
## 35.-Nulls
## 36.-Unique-constraints
## 37.-Exclusion-constraints
## 38.-Foreign-key-constraints