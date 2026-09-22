---
date: '2026-09-22'
draft: false
title: "Query any column, not just the primary key: a SQL developer's guide to Cassandra SAI"
summary: Cassandra only answers queries on the primary key until you index the
  other columns. See how Storage Attached Indexes lift that restriction, using
  1,000 real films.
tags: ['Cassandra', 'SAI', 'Tutorial', 'Databases']
cover:
  image: posts/images/c5-cmovie-wk01-sai-overview.webp
  alt: A field of grey, crossed-out column tiles thinning into connected orange
    tiles, next to the text "cMovie - Week 1 - Stop typing ALLOW FILTERING"
  relative: false
---

How do you query a non primary key column in Apache Cassandra® 5.0? Create a
[Storage Attached Index (SAI)](https://cassandra.apache.org/doc/5.0/cassandra/developing/cql/indexing/sai/sai-overview.html)
on it. After that you can filter on that column with equality, ranges and
`CONTAINS`, and combine several indexed columns in one query, all without
`ALLOW FILTERING`. This post shows how, using 1,000 real films.

## cMovie hits a wall

**cMovie** is a fictional movie discovery app, and
[this series](/posts/c5-cmovie-overview/) builds it one Cassandra 5.0 feature
at a time. Its first feature is browsing: show me films by genre, release
year and rating.

In a relational database you'd add a `WHERE` clause and move on. In Cassandra,
the first attempt looks like this:

```sql
SELECT title, release_year FROM movies WHERE release_year = 1999;
```

```text
Cannot execute this query as it might involve data filtering and thus may have
unpredictable performance. If you want to execute this query despite the
performance unpredictability, use ALLOW FILTERING
```

By the end of this post that query works, and so do much more interesting ones.

## What you'll learn and what you need

You'll learn why Cassandra restricts queries on non key columns, how SAI lifts
that restriction, and which query shapes SAI supports.

You need a free
[Astra DB](https://www.ibm.com/products/datastax?utm_medium=social&utm_campaign=ai-db-scale&utm_content=erickramirez)
account and Python 3.10 or later, which the loader uses. You don't need a
local database, a JVM, a `cqlsh` install or Docker, and there's no embedding or
LLM key this week. Launching a cluster takes 5 clicks and 90 seconds.

The lesson itself is plain CQL typed into the Astra CQL console. Next week's
build post turns it into a Python API.

## Get a database

Create a free Astra DB database and open its CQL console in your browser. The
[setup page](https://github.com/ErickRamirezAU/cassandra-5-movie-search/blob/main/docs/setup.md)
covers the account, the token, the secure connect bundle and resuming a
hibernated database, so I won't repeat it here.

Why Astra DB? It's Cassandra without the job of operating it, so your time
goes into building the app. Everything this series teaches is an Apache
Cassandra 5.0 feature. For full disclosure, I'm an Apache Cassandra committer
and a Developer Advocate at DataStax, now an IBM company.

You don't create a keyspace either. Astra DB provides `default_keyspace`.
Select it at the start of every console session:

```sql
USE default_keyspace;
```

## Create the movies table

Here is the `movies` table that every post in this series builds on. The same
statement, plus the indexes you add later, is in
[`schema.cql`](https://github.com/ErickRamirezAU/cassandra-5-movie-search/blob/main/tutorials/week-01/schema.cql)
in the series repo:

```sql
CREATE TABLE IF NOT EXISTS movies (
    movie_id text PRIMARY KEY,
    title text,
    release_year int,
    genres set<text>,
    runtime int,
    cmovie_rating float,
    cmovie_votes int,
    cmovie_popularity float,
    plot text,
    actors list<text>,
    actor_words set<text>,
    title_words set<text>
);
```

The primary key, `movie_id`, is the film's
[Wikidata](https://www.wikidata.org) ID, such as `'Q25188'`. Cassandra spreads
rows across the cluster by partition key, so a lookup by `movie_id` goes
straight to the right place. Any other question is a problem, which is the
wall from the start of this post.

This post queries `release_year`, `genres`, `runtime` and `cmovie_rating`, and
uses `cmovie_votes` for one deliberate failure. `plot` waits until week 3, and
`actor_words` and `title_words` until week 2.

One thing to be clear about: the three `cmovie_` columns are made up numbers for
the fictional app, not real audience scores. Everything else is real data from
Wikidata and Wikipedia.

## Load the films

The series has one loader, a Python script that all ten posts reuse. It pulls
1,000 films, 250 per decade from the 1990s onwards, and fills every column in
the table, so later weeks don't need a reload. Week 3 is the exception, when
the embedding column arrives.

Where the data comes from:

- **Film facts** (title, release year, genres, runtime) come from the Wikidata
  Query Service.
- **Plot text** and the starring actors come from
  [English Wikipedia](https://en.wikipedia.org). Actors are kept in poster
  billing order.
- **Rating, votes and popularity** are generated by the loader. They're seeded
  from each film's Wikidata ID, so you get the same numbers I did and the
  results below match what you see. Ratings cluster around the middle of the
  scale, and popularity has a long tail, which matters in weeks 9 and 10.
- **Search tokens** for titles and actors are generated by the loader too, and
  come into play in week 2.

You don't need an API key for any of this, because Wikidata and Wikipedia are
open. Wikidata film facts are
[CC0](https://creativecommons.org/publicdomain/zero/1.0/), and I credit them
anyway. Plot text is
[CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/): the plots
aren't queried this week, but they're in your table, so the repo carries a
link to each article and the licence notice.

Follow the
[setup page](https://github.com/ErickRamirezAU/cassandra-5-movie-search/blob/main/docs/setup.md)
to create the virtual environment, then run the loader from the root of the
repo:

```bash
python tools/loader.py
```

A full run takes about an hour and a quarter, because the loader
queries Wikimedia one request at a time to stay within its limits. It finishes
with a line like `all 1000 rows inserted`. Confirm the load in the CQL console:

```sql
SELECT movie_id, title, release_year FROM movies LIMIT 5;
```

## The problem, properly explained

Cassandra finds rows by partition key. A query on any other column has no idea
which partitions hold matches, so the only way to answer it is to read every
partition on every node.

`ALLOW FILTERING` tells Cassandra to do exactly that. On a laptop with 1,000
films you'll never notice. In production, the cost grows with the table, and
so does the unpredictability the error message warns about.

The traditional fix is a table per query pattern: one keyed by year, another by
genre, each holding a copy of the data. It works, but every new question means
a new table and another write on every insert. I'll mention it and leave it
there.

The other fix is to index the columns.

## Add Storage Attached Indexes

Create one SAI index per column you want to query:

```sql
CREATE CUSTOM INDEX ON movies (release_year) USING 'StorageAttachedIndex';
CREATE CUSTOM INDEX ON movies (cmovie_rating) USING 'StorageAttachedIndex';
CREATE CUSTOM INDEX ON movies (runtime) USING 'StorageAttachedIndex';
CREATE CUSTOM INDEX ON movies (genres) USING 'StorageAttachedIndex';
```

A syntax note that will save you time. Cassandra 5.0 also accepts a shorthand,
`CREATE INDEX ... USING 'sai'`. Astra DB rejects that shorthand, so this
series uses the full `CREATE CUSTOM INDEX ... USING 'StorageAttachedIndex'`
form, which is the one that runs everywhere.

If you leave `USING` off entirely, Astra DB still gives you SAI, but 5.0 gives
you a legacy secondary index by default. Spelling it out avoids the surprise
if you later move to your own cluster.

Astra DB builds each index in the background after you create it. Until the
build finishes, a query that uses the index fails with
`INDEX_BUILD_IN_PROGRESS`. On my 1,000 films that took about three minutes, so
if you see that error, wait a moment and run the query again.

## Query cMovie

Now build the browse feature one question at a time. Every query is also in
[`queries.cql`](https://github.com/ErickRamirezAU/cassandra-5-movie-search/blob/main/tutorials/week-01/queries.cql)
in the series repo, if you'd rather paste them than type them.

```sql
-- 1. Films released in 1999
SELECT title, release_year FROM movies WHERE release_year = 1999 LIMIT 10;

-- 2. Films released between 2010 and 2019
SELECT title, release_year FROM movies
WHERE release_year >= 2010 AND release_year <= 2019 LIMIT 10;

-- 3. Films rated 7.5 or higher in cMovie
SELECT title, cmovie_rating FROM movies WHERE cmovie_rating >= 7.5 LIMIT 10;

-- 4. Science fiction films
SELECT title, genres FROM movies WHERE genres CONTAINS 'science fiction' LIMIT 10;

-- 5. Science fiction films from the 2010s rated above 7
SELECT title, release_year, cmovie_rating FROM movies
WHERE genres CONTAINS 'science fiction'
  AND release_year >= 2010 AND release_year <= 2019
  AND cmovie_rating > 7 LIMIT 10;

-- 6. Short dramas under 100 minutes
SELECT title, runtime FROM movies
WHERE genres CONTAINS 'drama' AND runtime < 100 LIMIT 10;
```

Rows come back in token order, not sorted, so `LIMIT 10` gives you the first
ten the index finds. Here is what queries 1, 5 and 6 return on my load.

Query 1, films released in 1999:

```text
               title                | release_year
------------------------------------+--------------
 South Park: Bigger, Longer & Uncut |         1999
 American Beauty                    |         1999
 Cruel Intentions                   |         1999
 The Ninth Gate                     |         1999
 All About My Mother                |         1999
 The Insider                        |         1999
 Anna and the King                  |         1999
 Dogma                              |         1999
 Girl, Interrupted                  |         1999
 Fight Club                         |         1999

(10 rows)
```

Query 5, science fiction films from the 2010s rated above 7:

```text
             title              | release_year | cmovie_rating
--------------------------------+--------------+---------------
 Justice League                 |         2017 |           8.8
 The Martian                    |         2015 |           8.6
 Solo: A Star Wars Story        |         2018 |           9.7
 The Avengers                   |         2012 |           7.2
 Thor: The Dark World           |         2013 |           7.8
 Jupiter Ascending              |         2015 |           8.3
 Dawn of the Planet of the Apes |         2014 |           7.4
 John Carter                    |         2012 |           8.4
 Captain Marvel                 |         2019 |           7.3
 Home                           |         2015 |           8.3

(10 rows)
```

Query 6, short dramas under 100 minutes:

```text
        title         | runtime
----------------------+---------
 La Haine             |      96
 Hancock              |      92
 Maleficent           |      97
 Cruel Intentions     |      97
 The Father           |      97
 Gordy                |      90
 The Truman Show      |      99
 Rambo                |      90
 Clerks               |      92
 In the Mood for Love |      98

(10 rows)
```

Without the `LIMIT`, the six queries return 33, 250, 219, 229, 22 and 35
films.

Each query shows one feature:

| Query | Feature |
| --- | --- |
| 1 | Equality on an indexed `int` |
| 2 | Range on an indexed `int` |
| 3 | Range on an indexed `float` |
| 4 | `CONTAINS` on an indexed `set<text>` |
| 5 | Several indexed predicates combined with `AND` |
| 6 | `CONTAINS` combined with a range |

None of them needs `ALLOW FILTERING`, as long as every restricted column has an
index. Query 5 is the payoff: three conditions on three different columns, one
query.

Wikidata labels genres as "science fiction film" and "drama film". The loader
trims the trailing "film", so the table holds `'science fiction'` and
`'drama'`, and the queries read naturally. `CONTAINS` matches a whole label,
so `'drama'` doesn't match `'comedy drama'` or `'crime drama'`. Those are
separate labels in the data.

Now the deliberate failure. `cmovie_votes` has no index, so restricting on it
brings the original error straight back:

```sql
SELECT title FROM movies WHERE release_year = 1999 AND cmovie_votes > 5000;
```

Index a column and you can query it. Skip the index and you're back to
`ALLOW FILTERING`.

## How SAI works, briefly

SAI index data is
[attached to the SSTables and memtables](https://cassandra.apache.org/doc/5.0/cassandra/developing/cql/indexing/sai/sai-concepts.html)
the rows live in, so an index follows its data through flushes and
compaction. Strings are indexed in a
[trie](https://github.com/apache/cassandra/blob/81b458017d8aef6e7dcd91d3f31d651bdad1aaa8/src/java/org/apache/cassandra/index/sai/disk/v1/trie/TrieTermsDictionaryWriter.java#L35-L42)
and numbers in a
[block balanced tree](https://github.com/apache/cassandra/blob/81b458017d8aef6e7dcd91d3f31d651bdad1aaa8/src/java/org/apache/cassandra/index/sai/disk/v1/bbtree/BlockBalancedTreeWriter.java#L49-L90).
Row offsets are stored
[once per SSTable](https://github.com/apache/cassandra/blob/81b458017d8aef6e7dcd91d3f31d651bdad1aaa8/src/java/org/apache/cassandra/index/sai/disk/PerSSTableIndexWriter.java#L25-L28),
which keeps disk use down when you index
[several columns](https://github.com/apache/cassandra/blob/81b458017d8aef6e7dcd91d3f31d651bdad1aaa8/src/java/org/apache/cassandra/index/sai/disk/PerColumnIndexWriter.java#L27-L30)
on the same table.

That shared storage is what lets SAI combine indexes. In 5.0, each
[legacy secondary index](https://cassandra.apache.org/doc/5.0/cassandra/developing/cql/indexing/indexing-concepts.html)
is its own separate index group, so a query like number 5 would still demand
`ALLOW FILTERING`. All SAI indexes on a table share
[one group](https://github.com/apache/cassandra/blob/81b458017d8aef6e7dcd91d3f31d651bdad1aaa8/src/java/org/apache/cassandra/index/sai/StorageAttachedIndexGroup.java#L71-L84).

| | Legacy secondary index | SAI |
| --- | --- | --- |
| Default for plain `CREATE INDEX` in 5.0 | Yes | No, opt in |
| On Astra DB | Not supported, defaults to SAI | Every index is SAI |
| Storage | Hidden local table per index | Attached to SSTables and memtables |
| Combine several indexed columns without `ALLOW FILTERING` | No | Yes |

Cassandra also ships SASI, an older index marked
[experimental](https://github.com/apache/cassandra/blob/81b458017d8aef6e7dcd91d3f31d651bdad1aaa8/NEWS.txt#L960-L964)
since 3.11.5 and
[disabled by default](https://github.com/apache/cassandra/blob/81b458017d8aef6e7dcd91d3f31d651bdad1aaa8/NEWS.txt#L840-L843)
since 4.0, so I'll skip it. For the design behind SAI, read
[CEP-7](https://cwiki.apache.org/confluence/x/7DZ4CQ).

## What SAI doesn't do in 5.0

Knowing the edges saves you a debugging session. These are the ones a new
cMovie developer will hit:

- **No `OR`.** The `WHERE` clause only joins restrictions with
  [`AND`](https://github.com/apache/cassandra/blob/81b458017d8aef6e7dcd91d3f31d651bdad1aaa8/src/antlr/Parser.g#L447-L450).
- **No pattern matching.** `text` columns support equality only, so there's no
  [`LIKE`](https://github.com/apache/cassandra/blob/81b458017d8aef6e7dcd91d3f31d651bdad1aaa8/src/java/org/apache/cassandra/index/sai/utils/IndexTermType.java#L583-L589)
  and no title prefix search. Week 2 shows a way around this.
- **[`IN`](https://github.com/apache/cassandra/blob/81b458017d8aef6e7dcd91d3f31d651bdad1aaa8/src/java/org/apache/cassandra/index/sai/plan/Expression.java#L82-L107)
  and
  [`!=`](https://github.com/apache/cassandra/blob/81b458017d8aef6e7dcd91d3f31d651bdad1aaa8/src/java/org/apache/cassandra/cql3/Relation.java#L139-L160)
  aren't served by the index.** `IN` falls back to `ALLOW FILTERING` and `!=`
  is rejected.
- **Every restricted column needs an index** to avoid
  [`ALLOW FILTERING`](https://github.com/apache/cassandra/blob/81b458017d8aef6e7dcd91d3f31d651bdad1aaa8/src/java/org/apache/cassandra/cql3/statements/SelectStatement.java#L1601-L1612).
- **No sorting by indexed columns.** Results come back in
  [token order](https://github.com/apache/cassandra/blob/81b458017d8aef6e7dcd91d3f31d651bdad1aaa8/src/java/org/apache/cassandra/cql3/statements/SelectStatement.java#L1399-L1405).
  The one exception is vector `ANN OF` ordering, which arrives in week 3.
- **[One column per index, and one SAI index per
  column](https://github.com/apache/cassandra/blob/81b458017d8aef6e7dcd91d3f31d651bdad1aaa8/src/java/org/apache/cassandra/index/sai/StorageAttachedIndex.java#L246-L268).**

For text columns you also get three index options, `case_sensitive`, `normalize`
and `ascii`, which change how whole values match. They don't tokenise, which is
why week 2 does the word splitting at load time.

## Recap

- Cassandra only serves queries on the primary key unless you index the other
  columns, and `ALLOW FILTERING` is a scan, not a fix.
- SAI indexes lift the restriction. Create one per column with `CREATE CUSTOM
  INDEX ... USING 'StorageAttachedIndex'`.
- You get equality, ranges and `CONTAINS`, combined with `AND` across as many
  indexed columns as you like.

Next week you'll build the cMovie browse API in Python on these indexes. In
three weeks, film plots become vectors.

If you want next week's post when it lands, follow me using the social buttons
on the [8567.me](https://8567.me) homepage.

---

Film facts from [Wikidata](https://www.wikidata.org),
[CC0](https://creativecommons.org/publicdomain/zero/1.0/). Plot text from
English Wikipedia,
[CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/), with links to
each article in the
[series repo](https://github.com/ErickRamirezAU/cassandra-5-movie-search).

*Apache Cassandra, Cassandra, Apache, the Apache logo, and the Apache Cassandra
project logo are either registered trademarks or trademarks of The Apache
Software Foundation in the United States and other countries.*
