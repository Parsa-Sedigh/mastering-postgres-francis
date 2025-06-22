## 90.-Introduction-to-full-text-search
The places where pg search starts to go behind is:
1. scale
2. features

## 91.-Searching-with-LIKE
Not a good way to build full-text-search engine.

Like is case-sensitive.

If you're just searching in an email col or a col like `title` or `username`, you might get away with `like` and `ilike`.

But searching with these in cols with longer text vals, will taker long.

## 92.-Vectors_-queries_-and-ranks
To get a ts_vector:
```postgresql
select title, to_tsvector(title)
from movies
limit 50;
```
It gives us back sth like: `'Kansas Saloon Smashers', 'kansa':1 'saloon':2 'smasher':3`.

- **The tsvector gives us the `lexemes` in asc alphabetical order and their location in the text.**
- Lexemes are root words.
- it removes the words like `the`, `a`

Full text search on 'star':
```postgresql
select title
from movies
where to_tsvector(title) @@ to_tsquery('star')
limit 50;
```

This gives us an err: `to_tsquery('star wars')`: `ERROR: syntax error in tsquery: "star wars"`.

This is because `ts_query` format requires kinda a domain specific language(DSL). There are some helper funcs that help us convert a text
into that DSL.

The reason `to_tsquery('star wars')` returns err is we didn't specify how those two words should **relate to each other**. Should
they both be there? One of them is enough(or operator)? For this, some operators exist: `to_tsquery('star & wars')` meaning both words
should exist.

- `to_tsquery('star | wars')`
- `to_tsquery('star <-> wars')` means wars should immediately come after star
- `to_tsquery('star & (wars | trek)')` means immediately after star, either wards or trek should appear.
  This expression is equivalent to: `to_tsquery('star <1> wars')` which means the next(<1>) word must be wars or trek.

So <-> is syntactic sugar for <1>.

`star <2> wars` means skip the next word and then look for word `wars`.

### Rank funcs
```postgresql
select title, ts_rank(to_tsvector(title) @@ to_tsquery('star')) as rank
from movies
where to_tsvector(title) @@ to_tsquery('star')
order by rank desc
limit 50;
```

The highest rank is the most relevant based on the vector and query we have given it.

Fine-tuning opts:

- We can make a col having a higher weight in ranking than other cols.
- Also we can make exact matches having higher ranks.

## 93.-Websearch
```postgresql
select plainto_tsquery('star wars'); -- 'star' & 'wars'

-- enforces strict following rules
select phraseto_tsquery('star wars'); -- 'star' <-> 'wars'

-- gives us google-search style operators
-- for example double quotes means exact match
select websearch_to_tsquery('"star wars"'); -- 'star' <-> 'wars'

select websearch_to_tsquery('"star wars" or clone'); -- 'star' <-> 'wars' | 'clone'

-- dash means negating
select websearch_to_tsquery('"star wars" -clone'); -- 'star' <-> 'wars' & !'clone'

select websearch_to_tsquery('"star wars" +clone'); -- 'star' <-> 'wars' & 'clone'

select title
from movies
where to_tsvector(title) @@ plainto_tsquery('star wars');
```

So websearch_to_tsquery is more usable than other ones among these three.

## 94.-Ranking
Currently the right results are coming back but they're not being ordered that we as human might expect.
So we need to tune the rankings of the searches.

```postgresql
show default_text_search_config; -- pg_catalog.english

-- if you wanna override the lang, you can pass in 2 args and the first arg will be the lang. One option for lang is 'simple'.
select to_tsvector('english', title), to_tsvector('simple', title) from movies;
```

The difference between simple opt and other langs is it preserves words like `by`, `the`, `and` and ... .

### ranking
- We want `fight` to be above than `fighting`.
- we want the **shorter** texts that contains fight to be above longer ones. This is because htey're closer to an exact match
  which are the ones we want to see first.

If we pass `1` as third param to ts_rank() favors shorter vals:
```postgresql
select title, ts_rank(to_tsvector(title), to_tsquery('fight'), 1) as rank
from movies
order by rank desc;
```

We can pass in 1, 2, 4, 8, 16, 32. Meaning if you want to give even more weight to shorter docs, you can pass in 2.

Let's allow users to do full-text search in both title & plot cols. For this, concat the vectors of `title` & `plot`.
Also we want to consider higher weight for title than plot vector:
```postgresql
select 
    setweight(title, 'A'),
    ts_rank(
            (setweight(to_tsvector(title), 'A') || ' ' || to_tsvector(plot)),
            to_tsquery('fight')
    ) as rank
from movies
where 
    (
        setweight(to_tsvector(title), 'A') || ' ' || to_tsvector(plot)
    ) @@ to_tsquery('fight')
order by rank desc;
```

Notes:
- Although we used the same expression in `where` and `ts_rank()`, you can use different expr for them.
- We didn't use the third param for ts_rank() here because we don't want the length of the plot col affecting the ranking(here
  we have both title and plot in ts_rank()). When we only had title in ts_rank(), it made sense that the shorter titles should come first,
  but here we have both title and plot in ts_rank().
- `setweight()` accepts letters `'A'` - `'D'` as weights.

The prev query could output sth like this as the result of vector: `'assault':4A,80` which means the word assault is both in the title(because
it has A in it's value which is from `title` col) and plot. So it's in 4th pos of `title` and 80th position in `plot`.

We can affect the rank outside of `ts_rank()`. Because `ts_rank()` just returns a float, we can add a number to it: `ts_rank(...) + 1.3`.
A use case is giving higher ranks to **popular** movies:
```postgresql
select title,
       ts_rank(
               (setweight(to_tsvector(title), 'A') || ' ' || to_tsvector(plot)),
               to_tsquery('fight')
       ) + (
               case when genre like '%action%' then 0.1 else 0 end -- give action genre more weight 
           ) as rank
from movies
```
Here the ranking is multi-faceted meaning it includes rank on multiple cols: high rank on `title`, a bit on the `plot` and a bit on `genre`.

## 95.-Indexing-full-text-search
Instead of calculating tsvectors on every single query, we're gonna create a generated col and store the vectors in there and then
we can run ts_query() against a stored value. Also, we can put an index on that generated col. Now anytime we update the `title` or `plot`,
the generated col is updated automatically as well.

With this generated col, instead of having a big query to actually calculate the vectors, we have a named col that has the vectors
based on the formula.

```postgresql
alter table movies
    add column search_vectors tsvector generated always as (
        setweight('english', to_tsvector(coalesce(title, '')), 'B')
            || ' '
            || to_tsvector('english', coalesce(plot, ''))
        ) stored;
```
If we don't specify the lang for `to_tsvector()` it will generate an err: ERROR: generation expression is not immutable.

This is because the config could change which causes the default lang config of to_tsvector to change as well. So specify the lang as well.

```postgresql
select *
from movies
where search_vectors @@ websearch_to_tsquery('"star wars"')
order by ts_rank(search_vectors, websearch_to_tsquery('"star wars"')) desc;
```

### Adding gin index on vector generated col
```postgresql
create index idx_movies_search_gin on movies using gin (search_vectors);
```

Now if you run explain on prev query:
```
Sort (cost=34.84..34.84 rows=1 width=794)
    Sort Key: (ts_rank(search_vectors, websearch_to_tsquery ('"stars wars"':: text))) DESC
    -> Bitmap Heap Scan on movies (cost=30.31..34.83 rows=1 width=794)
           Recheck Cond: (search_vectors @@ websearch_to_tsquery '"stars wars"':: text))
           -> Bitmap Index Scan on idx_movies_search_gin (cost=0.00..30.31 rows=1 width=..
                 Index Cond: (search_vectors Ca websearch_to_tsquery ('"stars wars"':: text))
```

## 96.-Highlighting
`ts_headline()` is the highlighting func. Pass it the original cols not their vector versions.

By default, it returns the matched words inside `<b></b>`. We can change this with 4th param.

```postgresql
select 
    ts_headline('english', title, websearch_to_tsquery('"star wars"'), 'StartSel=<mark>,StopSel=</mark>') as title_highlighted,
    *
from movies
where search_vectors @@ websearch_to_tsquery('"star wars"')
order by ts_rank(search_vectors, websearch_to_tsquery('"star wars"')) desc;
```