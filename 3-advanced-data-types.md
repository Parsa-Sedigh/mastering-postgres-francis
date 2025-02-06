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
Preferred for auto incrementing ids(used a lot for primary key). Underlying Bigints are best type for primary keys.

We will talk about bigints(bigserial) vs uuids as primary keys later, because we need to learn about indexes.

```postgresql
create table id_example
(
    id   bigint generated always as identity primary key,
    name text
);

insert into id_example(name)
values ('Aaron');

-- ERROR: cannot insert a non-DEFAULT value into column "id". Column "id" is an identity column defined as GENERATED ALWAYS.
-- NOTE: Since we used GENERATED ALWAYS, this will give error.
insert into id_example(id, name)
values (8, 'Aaron');

-- INSTEAD, we can do 2 things:
-- 1. if for whatever reason, you can't omit the id col from the query, you can use `DEFAULT`.
insert into id_example (id, name)
values (DEFAULT, 'Aaron');

-- 2. not recommended. Since our explicit id that we use here, won't update the sequence. Therefore, the underlying seq is not incremented, so
-- now the seq and table are out of sync. So if you use this approach, you have to setval the sequence to the id after this id you used here.
insert into id_example (id, name) overriding system value
values (16, 'Aaron');

-- The takeaway is: if the primary key is auto-incremented, we would just let it do it's thing and won't specify an id explicitly. Otherwise,
-- first get the seq name and then call setval():
-- NOTE: We have to get the underlying name of the seq, since pg created it itself not us.
select pg_get_serial_sequence('<table name>', '<col name>'); -- gives us the name of the internal seq that's being used for the specified underlying col.

select pg_get_serial_sequence('id_example', 'id'); -- public.id_example_id_seq. so identity cols are still using sequences under the hood

-- then set the seq to max(id) of the table to sync the seq and table. So that next automatic call to nextval() by pg, will return the correct
-- result and we won't get duplicate id down the road.
-- NOTE: we have to wrap the select statement inside parentheses to make it a subquery
select setval('public.id_example_id_seq', (select max(id) from id_example));
```

So overriding system val like seq is a bad idea.

### generated by default
Not recommended.

Instead of `generated always`, we can say: `generated by default`. But this one is more loose than the former syntax. Because now
we can actually pass in an explicit val and won't get an error. But the problem with duplicate primary key down the road is still there.

Why we don't get err? Because we're specifying a DEFAULT here, meaning if we omit the val, it will use the DEFAULT, otherwise, it uses our val.

So generated always gives us some safety, so that if we provide a val ourselves, it will give us an error unless we specify `DEFAULT` or
`overriding system value`.

## 27.-Network-and-mac-addresses
TODO

## 28.-JSON
Putting unstructured data like json in a relational DB!

When to use JSON col?

- If you need to constantly query inside of json docs, that's a good hint that they should be broken out to cols.
We can also keep the json blob intact but still pull data out to other cols. But this is **not** a hard rule that if you need to query by some
key in the json doc, then it **MUST** be a separate col. That's not what I meant. I mean if you have a json blob and maybe user email is in there and
that's the main thing by which you look users up, let's break that out to a separate col.
Postgres works best with normalized data. So look at your access patterns(how do you query the data?).
- If it's a rigid, well-defined, well-understood schema, you might consider breaking it into cols. Especially if the keys are updated independently
    - This is not a hard rule, since pg supports **json patching** for updating individual keys of json. This also helps us to update individual keys
    instead of sending the whole json again for updating just one key.
- a json col supports 255MB of json. But remember, `can it support it` and `should I do it` are different questions. If your json is massive,
the perf will suffer, so it's good to break it into multiple json doc cols responsible for different scopes of the json.

So these points could potentially point you towards storing those fields into separate cols in addition to the json col:
- how are you gonna access it
- constantly querying for specific keys
- is it a well-defined schema? and the keys are updated independently or all in one go?
- size of the json and could you break it down into multiple json cols

A good reason to keep it as a blob of json is if the entire blob is updated all at once.

### JSON vs JSONB
json type is text under the hood with some json validation. Use jsonb unless you're relying on quirks of json and you don't want to
store a parsed version of it. And you don't want to use good adequate features of jsonb like:
- removing duplicate keys
- removing whitespaces
- not preserving the order of keys(which is a practice good and is according to json spec)

jsonb size is a bit larger, but a lot faster than json when doing ops. Because the given string val is parsed and stored as binary format instead
of text.

```postgresql
select '1'::json, pg_typeof('1'::json), pg_column_size('1'::json); -- json is likely not what you want. This has some use cases, but they're rare.

select '1'::jsonb, pg_typeof('1'::jsonb); -- ✅

select pg_column_size('1'::json), pg_column_size('1'::jsonb); -- 5, 20
```

You might expect to hear: Use `json` instead of `jsonb` because it's more compact. But no don't use it.

jsonb is bigger because it has been deconstructed or parsed the json string and stored in a binary representation and it keeps around a few
extra bytes of info to help pg doing querying and writing of json faster.

So jsonb type is not stored as text under the hood whereas json is stored as text.

Note: As the json val gets bigger, the size difference between json and jsonb gets a bit closer, because the overhead starts to get amortized.
```postgresql
select pg_column_size('{"a": "hello world"}'::json) as json, 
       pg_column_size('{"a": "hello world"}'::json) as jsonb; -- 24, 28
```

#### Difference
1. Anything you throw at the `json` col, is gonna be retained. So if we put `{"a":    "hello world"}`, the whitespaces gonna be retained, but not in `jsonb`.
So jsonb compresses the given value and removes the whitespaces.
This is because jsonb is not storing literal text representation, it's storing the binary form of the json which adheres to certain rules.
2. One of the rules of json is no duplicate keys. If one key appears more than once, in jsonb, the last one is stored, but in json
all of the duplicates remain. EX: '{"a": 2, "a": 3}', in jsonb: '{"a": 3}', in json: '{"a": 2, "a": 3}'.

Note:
- Both json and jsonb only accept valid json. So we can't throw anything at json.
- Don't rely on key ordering in json types. Spec doesn't guarantee that. jsonb could potentially reorder the keys. But since json
stores the val as text, it maintains the ordering. It keeps whatever you give it.

### getting keys
One way to extract keys out of jsonb val:
```postgresql
select '{
  "string": "Hello, world",
  "number": 42,
  "boolean": true,
  "null": null,
  "array": [
    1,
    2,
    3
  ],
  "object": {
    "key": "value"
  }
}'::jsonb->'string'; -- "Hello, world"
  
-- get deeply nested fields with: ->'object'->'key'

-- get arr el: ->'array'->2
```
You can use json unquoting operator (->>) to do that.

## 29.-Arrays
**Postgres array is 1-based index. But an arr in json val is 0-based index.**

When should you use arrays vs breaking the data into it's own **table**.

- storing an arr of censor readings is ok. That necessarily need to be broken out into it's own table. Because perhaps those readings are
always associated with censors at a **certain timestamp**, maybe doing it every minute. So you store the whole chunk of readings together.
- removing the need for many-to-many tables: storing the tag ids together which removes the need for intermediate(linking) table. 
But this causes issues as well like we don't have that referential integrity that we get by using foreign keys. 
So if a tag gets deleted, that's not gonna necessarily delete that tag from the arr.

```postgresql
create table array_example
(
  id           bigint generated always as identity primary key,
  int_array    integer[], -- or integer array
  text_array   text[],
  bool_array   boolean[],
  nested_array integer[][]
);

insert into array_example(int_array, text_array, bool_array, nested_array)
values (array [1, 2, 3, 4],
        array ['marigold', 'daisy', 'poppy', 'sunflower'],
        [true, false, true, false],
        '{{1, 2, 3}}');
```

Array literal is specified with two formats:
- `array [...]` . verbose mode
- `{...}`. This is the standard output format when we do `select`

```postgresql
select id,
       array_text[1],
       array_text[1:3], -- slicing
       array_text[1:10] -- even if there are less than 10 els, it won't throw err
from array_example;
```

### Array includes operator
Get rows that their text_array col include 'poppy'.
```postgresql
select id, text_array
from array_example
where text_array @> array ['poppy']; -- or @> '{poppy}'
```

### unnest
Takes an arr and turns it into a result set and the items that are not arr, will be copied down into each row.
So it takes us from the arr land back to SQL land.

Result set means rows.

```postgresql
with flowers as (
  select id, 
         unnest(text_array) as flower
  from array_example    
)
select * from flowers
where flower = 'poppy';
```

## 30.-Generated-columns


## 31.-Text-search-types
## 32.-Bit-string
## 33.-Ranges
## 34.-Composite-types
## 35.-Nulls
## 36.-Unique-constraints
## 37.-Exclusion-constraints
## 38.-Foreign-key-constraints