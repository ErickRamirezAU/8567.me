---
date: '2026-09-29'
draft: false
title: "Not every search needs Elasticsearch: build a Python FastAPI filter and keyword search on Cassandra"
summary: SAI matches whole values, not fragments. Split titles and cast names
  into words at load time, index the words, and serve fragment search for
  1,000 real movies from a small Python FastAPI app.
tags: ['Cassandra', 'SAI', 'Python', 'FastAPI', 'Tutorial']
cover:
  image: posts/images/c5-cmovie-wk02-keyword-search.webp
  alt: A field of outlined movie record cards, four larger cards linked by
    orange lines with a matching orange word segment, next to the text
    "cMovie - Week 2 - One FastAPI endpoint, five filters"
  relative: false
---

How do you let a user search [Apache
Cassandra®](https://cassandra.apache.org/) 5.0 for a text fragment, like "wick"
inside "John Wick", not just a whole value? Split the text into words when you
write it, index the words with a [Storage Attached Index
(SAI)](https://cassandra.apache.org/doc/5.0/cassandra/developing/cql/indexing/sai/sai-overview.html),
and match them with `CONTAINS`. This post builds that pattern into a small
Python API, using the same [1,000
movies](/posts/c5-cmovie-wk01-sai-overview/#load-the-films) from week 1.

This is the second instalment in the [Cassandra 5.0
series](/posts/c5-cmovie-overview/) where I'll show you how to build an AI movie
recommender app with Python and TypeScript.

## What you'll learn and what you need

You'll learn:

- the token search pattern,
- splitting text into words at load time and indexing the words,
- how to combine several SAI predicates across columns in one query, and
- how to wire that into a small Python API.

We will continue to use the same [Astra
DB](https://www.ibm.com/products/datastax?utm_medium=social&utm_campaign=ai-db-scale&utm_content=erickramirez)
and the loaded `movies` table from week 1 for this tutorial.

For full disclosure, I'm an Apache Cassandra committer and a Developer Advocate
at DataStax, now an IBM company. We are using Astra DB for convenience so you
can focus on building apps as a developer without worrying about how to install
or configure a cluster, but the examples in this post are verified to run on
Cassandra 5.0.

There's nothing new to set up for the database itself. You only need Python
3.10 or later and the virtual environment from [the series setup
page](https://github.com/ErickRamirezAU/cassandra-5-movie-search/blob/main/docs/setup.md).

Let's run the CQL in this post in the Astra CQL console, after
`USE default_keyspace;`, just like week 1.

## cMovie needs a search box

[Week 1](/posts/c5-cmovie-wk01-sai-overview/) got **cMovie** filtering by genre,
year and rating with SAI, but only as whole values or ranges. A real search box
takes fragments, though. What we really want is to search for "wick" and get
every John Wick movie back.

As a reminder, this is what we store for each movie (see
[`schema.cql`](https://github.com/ErickRamirezAU/cassandra-5-movie-search/blob/main/tutorials/week-01/schema.cql)):

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

SAI's text index options, `case_sensitive`, `normalize` and `ascii`, change how
a whole value matches, not whether a fragment does. Let's index the `title`
field with the most forgiving settings, so matching ignores case:

```sql
CREATE CUSTOM INDEX ON movies (title) USING 'StorageAttachedIndex'
    WITH OPTIONS = {'case_sensitive': false, 'normalize': true};
```

The index takes a few minutes to build on Astra DB. Until it's ready, queries
on `title` fail with a `ReadFailure` error (code 1300) rather than a friendly
"not ready yet" message, so give it a moment before running the next query.

```sql
SELECT title FROM movies WHERE title = 'JOHN WICK';

 title
-----------
 John Wick

(1 rows)
```

But searching for a fragment of the title with a filter `title = 'wick'` still
returns nothing:

```sql
SELECT title FROM movies WHERE title = 'wick';

 title
-------

(0 rows)
```

That's not a bug. [Week 1 already flagged
it](/posts/c5-cmovie-wk01-sai-overview/#what-sai-doesnt-do-in-50): `text` data
type columns support equality only, so there's no
[`LIKE`](https://github.com/apache/cassandra/blob/81b458017d8aef6e7dcd91d3f31d651bdad1aaa8/src/java/org/apache/cassandra/index/sai/utils/IndexTermType.java#L583-L589)
and no prefix search. Fragment search needs a different column, one that holds
the individual words rather than the whole title.

We created the `title` index for illustration only, so let's drop it before we
move on:

```sql
DROP INDEX movies_title_idx;
```

## Connect from Python

The virtual environment from the setup page already has the Python driver,
[`cassandra-driver`](https://pypi.org/project/cassandra-driver/), installed
from the repo's `requirements.txt`, along with FastAPI and Uvicorn for later in
this post. Let's open a session:

```python
from cassandra.cluster import Cluster
from cassandra.auth import PlainTextAuthProvider

cluster = Cluster(
    cloud={"secure_connect_bundle": "secure-connect-cmovies.zip"},
    auth_provider=PlainTextAuthProvider("token", ASTRA_DB_TOKEN),
)
session = cluster.connect("default_keyspace")
```

`ASTRA_DB_TOKEN` is the application token from your `.env` file. The week 2
[`app.py`](https://github.com/ErickRamirezAU/cassandra-5-movie-search/blob/main/tutorials/week-02/app.py)
reads it from `.env` for you, and finds the secure connect bundle in the root
of the repo.

## Index the search columns

Week 1's schema already has three columns this post hasn't used yet, and the
[data loader](https://github.com/ErickRamirezAU/cassandra-5-movie-search/blob/main/docs/setup.md#5-load-the-movies-dataset)
has been filling all three since the first load:

| Column | Type | Purpose in this post |
| --- | --- | --- |
| `actors` | `list<text>` | Display, poster billing order. Not indexed |
| `actor_words` | `set<text>` | Search. SAI indexed, one lowercase token per name part |
| `title_words` | `set<text>` | Search. SAI indexed, one lowercase token per title word |

Let's index the two search columns. Both statements are also in week 2's
[`schema.cql`](https://github.com/ErickRamirezAU/cassandra-5-movie-search/blob/main/tutorials/week-02/schema.cql):

```sql
CREATE CUSTOM INDEX ON movies (title_words) USING 'StorageAttachedIndex';
CREATE CUSTOM INDEX ON movies (actor_words) USING 'StorageAttachedIndex';
```

They use the same `CREATE CUSTOM INDEX ... USING 'StorageAttachedIndex'` form
as the [indexes in week
1](/posts/c5-cmovie-wk01-sai-overview/#add-storage-attached-indexes). Both
columns are `set<text>`, so we query them with `CONTAINS`, just like `genres`.
Give Astra DB a moment to finish building both in the background before you
query them.

> [!INFO]
>
> On creation of an index, Cassandra builds it in the background and will take a
> few seconds to a few minutes to complete before the index can be queried.
>
> But once the index is live, index updates are **synchronous** with writes to
> the base table, happening as part of the write itself, so new data can be
> queried through the index **immediately**.

## How the loader builds the tokens

This is not this post's focus, but I'll cover the loader's code here so you
understand how the table was populated.

One shared tokeniser in
[`tools/loader.py`](https://github.com/ErickRamirezAU/cassandra-5-movie-search/blob/41a018a94b47ccbb99c8aab025ab2eef5503d958/tools/loader.py#L81-L100)
fills both columns:

```python
TOKEN_STRIP_RE = re.compile(r"[:,.\-–—]")

def tokenise(text: str) -> list[str]:
    stripped = TOKEN_STRIP_RE.sub("", text)
    return [t.lower() for t in stripped.split() if t]
```

`title_words` is `tokenise(title)`. `actor_words` is every name in `actors`
tokenised and unioned into one set, so a two-word name contributes both
words. "John Wick: Chapter 3 – Parabellum" tokenises to
`{3, chapter, john, parabellum, wick}`, confirmed against the live table.

Dunkirk's cast shows both edge cases worth knowing before you rely on this
column:

```text
actors:      ['Fionn Whitehead', 'Tom Glynn-Carney', ..., "James D'Arcy", ...]
actor_words: {..., 'glynncarney', ..., "d'arcy", ...}
```

Apostrophes are left alone: "James D'Arcy" tokenises to `d'arcy`, not `d`
and `arcy`. Hyphens are deleted outright, not replaced with a space, so "Tom
Glynn-Carney" tokenises to `glynncarney`, one word. A search for "carney"
won't find Dunkirk. It finds Last Action Hero instead, because another actor
in that cast has Carney as a separate word. You'd need "glynncarney" to find
Dunkirk. That's the trade-off of matching whole words only: "wick" matches,
"wic" doesn't, and a hyphenated name becomes the one token the loader made
from it.

## Query cMovie by fragment

As they say, "on with the show". Let's build up one question at a time, each one
a `CONTAINS` on a token column. Every query is also in
[`queries.cql`](https://github.com/ErickRamirezAU/cassandra-5-movie-search/blob/main/tutorials/week-02/queries.cql)
in the series repo.

Query 1, search for movies with "wick" in the title:

```sql
SELECT title FROM movies WHERE title_words CONTAINS 'wick' LIMIT 10;

 title
-----------------------------------
 John Wick: Chapter 4
 John Wick: Chapter 3 – Parabellum
 John Wick: Chapter 2
 John Wick

(4 rows)
```

Query 2, search for movies with "murphy" in the cast:

<!-- markdownlint-disable MD013 -->
```sql
SELECT title, actors FROM movies WHERE actor_words CONTAINS 'murphy' LIMIT 10;

 title                 | actors
-----------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Dunkirk               | ['Fionn Whitehead', 'Tom Glynn-Carney', 'Jack Lowden', 'Harry Styles', 'Aneurin Barnard', 'James D''Arcy', 'Barry Keoghan', 'Kenneth Branagh', 'Cillian Murphy', 'Mark Rylance', 'Tom Hardy']
 Batman Returns        | ['Michael Keaton', 'Danny DeVito', 'Michelle Pfeiffer', 'Christopher Walken', 'Michael Gough', 'Pat Hingle', 'Michael Murphy']
 Batman Begins         | ['Christian Bale', 'Michael Caine', 'Liam Neeson', 'Katie Holmes', 'Gary Oldman', 'Cillian Murphy', 'Tom Wilkinson', 'Rutger Hauer', 'Ken Watanabe', 'Morgan Freeman']
 Girl, Interrupted     | ['Winona Ryder', 'Angelina Jolie', 'Clea DuVall', 'Brittany Murphy', 'Elisabeth Moss', 'Jared Leto', 'Jeffrey Tambor', 'Vanessa Redgrave', 'Whoopi Goldberg']
 A Quiet Place Part II | ['Emily Blunt', 'Cillian Murphy', 'Millicent Simmonds', 'Noah Jupe', 'Djimon Hounsou', 'John Krasinski']
 28 Days Later         | ['Cillian Murphy', 'Naomie Harris', 'Christopher Eccleston', 'Megan Burns', 'Brendan Gleeson']
 8 Mile                | ['Eminem', 'Kim Basinger', 'Brittany Murphy', 'Mekhi Phifer']
 Inception             | ['Leonardo DiCaprio', 'Ken Watanabe', 'Joseph Gordon-Levitt', 'Marion Cotillard', 'Elliot Page', 'Tom Hardy', 'Cillian Murphy', 'Tom Berenger', 'Michael Caine']
 Spider-Man 2          | ['Tobey Maguire', 'Kirsten Dunst', 'James Franco', 'Alfred Molina', 'Rosemary Harris', 'Donna Murphy']
 Sin City              | ['Jessica Alba', 'Benicio del Toro', 'Brittany Murphy', 'Clive Owen', 'Mickey Rourke', 'Bruce Willis', 'Elijah Wood']

(10 rows)
```
<!-- markdownlint-enable MD013 -->

Twelve movies in the dataset match, so `LIMIT 10` returns the first ten.
Cillian, Michael, Brittany and Donna Murphy all turn up, not just one actor.

Query 3, search for movies with both "wick" and "chapter" in the title:

```sql
SELECT title FROM movies
WHERE title_words CONTAINS 'wick' AND title_words CONTAINS 'chapter' LIMIT 10;

 title
-----------------------------------
 John Wick: Chapter 4
 John Wick: Chapter 3 – Parabellum
 John Wick: Chapter 2

(3 rows)
```

Compared to query 1, the results don't include the plain "John Wick", keeping
only the three "Chapter" movies. Multi-word queries are one `CONTAINS` per word,
`AND`ed.

## Combine fragment search with week 1's filters

These combined queries are what the API at the end of this post will serve,
with every filter so far usable together.

Query 4, search for movies released after 2015 with a Murphy in the cast:

<!-- markdownlint-disable MD013 -->
```sql
SELECT title, release_year, actors FROM movies
WHERE actor_words CONTAINS 'murphy' AND release_year > 2015 LIMIT 10;

 title                 | release_year | actors
-----------------------+--------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Dunkirk               |         2017 | ['Fionn Whitehead', 'Tom Glynn-Carney', 'Jack Lowden', 'Harry Styles', 'Aneurin Barnard', 'James D''Arcy', 'Barry Keoghan', 'Kenneth Branagh', 'Cillian Murphy', 'Mark Rylance', 'Tom Hardy']
 A Quiet Place Part II |         2020 | ['Emily Blunt', 'Cillian Murphy', 'Millicent Simmonds', 'Noah Jupe', 'Djimon Hounsou', 'John Krasinski']

(2 rows)
```
<!-- markdownlint-enable MD013 -->

Both movies star Cillian Murphy.

Queries 5 and 6 also filter on `cmovie_rating`. As a reminder, it's a made-up
number for the fictional cMovie app, not a real audience score.

Query 5, search for highly rated dramas with "wick" in the title:

```sql
SELECT title FROM movies
WHERE genres CONTAINS 'drama' AND title_words CONTAINS 'wick'
  AND cmovie_rating > 7 LIMIT 10;

 title
-------

(0 rows)
```

Query 5 returns nothing, and that's fine. An empty result set is handled
cleanly, not an error.

Query 6, search for highly rated science fiction movies with Murphy in the
cast:

<!-- markdownlint-disable MD013 -->
```sql
SELECT title, actors, genres, cmovie_rating FROM movies
WHERE actor_words CONTAINS 'murphy' AND genres CONTAINS 'science fiction'
  AND cmovie_rating > 7 LIMIT 10;

 title                   | actors                                                                                                                                                                          | genres                                     | cmovie_rating
-------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------------------------------+---------------
 Star Trek: Insurrection | ['Patrick Stewart', 'Jonathan Frakes', 'Brent Spiner', 'LeVar Burton', 'Michael Dorn', 'Gates McFadden', 'Marina Sirtis', 'F. Murray Abraham', 'Donna Murphy', 'Anthony Zerbe'] | {'action', 'adventure', 'science fiction'} |           7.1

(1 rows)
```
<!-- markdownlint-enable MD013 -->

Donna Murphy is the match here.

Every column in the `WHERE` clause must be SAI indexed, or the query
[needs `ALLOW FILTERING`](https://github.com/apache/cassandra/blob/81b458017d8aef6e7dcd91d3f31d651bdad1aaa8/src/java/org/apache/cassandra/cql3/statements/SelectStatement.java#L1601-L1612).

## Wire it into a small FastAPI app

Let's build one [FastAPI](https://fastapi.tiangolo.com/) endpoint,
`GET /movies/search`, that maps query parameters onto the predicates above. The
full file, which connects with the same code as the loader, is
[`app.py`](https://github.com/ErickRamirezAU/cassandra-5-movie-search/blob/main/tutorials/week-02/app.py)
in the series repo:

```python
import re
from typing import Optional

from fastapi import FastAPI, HTTPException

# The same rule tools/loader.py uses to fill title_words and actor_words, so
# a search term splits into exactly the tokens the loader stored.
TOKEN_STRIP_RE = re.compile(r"[:,.\-–—]")


def tokenise(text: str) -> list[str]:
    stripped = TOKEN_STRIP_RE.sub("", text)
    return [t.lower() for t in stripped.split() if t]


app = FastAPI()


@app.get("/movies/search")
def search_movies(
    title: Optional[str] = None,
    actor: Optional[str] = None,
    genre: Optional[str] = None,
    year_from: Optional[int] = None,
    year_to: Optional[int] = None,
    rating_min: Optional[float] = None,
):
    clauses, params = [], []
    for word in tokenise(title or ""):
        clauses.append("title_words CONTAINS %s")
        params.append(word)
    for word in tokenise(actor or ""):
        clauses.append("actor_words CONTAINS %s")
        params.append(word)
    if genre:
        clauses.append("genres CONTAINS %s")
        params.append(genre.lower())
    if year_from is not None:
        clauses.append("release_year >= %s")
        params.append(year_from)
    if year_to is not None:
        clauses.append("release_year <= %s")
        params.append(year_to)
    if rating_min is not None:
        clauses.append("cmovie_rating >= %s")
        params.append(rating_min)

    if not clauses:
        raise HTTPException(status_code=400, detail="at least one filter is required")

    query = (
        "SELECT title, release_year, cmovie_rating FROM movies WHERE "
        + " AND ".join(clauses)
        + " LIMIT 10"
    )
    rows = session.execute(query, params)
    # cmovie_rating is a 32-bit float, so round it back to the loader's one
    # decimal place rather than returning 4.900000095367432 for 4.9.
    return {
        "results": [
            {**row._asdict(), "cmovie_rating": round(row.cmovie_rating, 1)}
            for row in rows
        ]
    }
```

The endpoint runs each search term through the same `tokenise()` rule the
loader used, so a term splits into exactly the tokens stored in the table.
We'll see that in action once the app is running.

The type hints, such as `year_from: Optional[int]`, do real work: FastAPI
rejects bad input before the handler runs, so this section stays about query
design, not parsing.

Note we are using plain `def`, not `async def`, because the driver's
`session.execute()` blocks. A sync handler is the right choice, because
FastAPI runs it in a thread pool. The driver has `execute_async()`, but it
returns a callback-based result, not something you can `await`, so it isn't a
clean drop-in for `async def` either.

Let's run the app with [Uvicorn](https://uvicorn.dev/) from the root of the
repo:

```bash
uvicorn app:app --reload --app-dir tutorials/week-02
```

Then open `http://127.0.0.1:8000/docs` in a browser to try the endpoint
without assembling a curl command. If you prefer the command line, let's
search for movies released in 2015 or later with a Murphy in the cast. It's
close to query 4, but `year_from` is inclusive:

```bash
curl -s -G http://127.0.0.1:8000/movies/search \
  --data-urlencode "actor=murphy" --data-urlencode "year_from=2015" \
  | python -m json.tool

{
    "results": [
        {
            "title": "Dunkirk",
            "release_year": 2017,
            "cmovie_rating": 4.9
        },
        {
            "title": "A Quiet Place Part II",
            "release_year": 2020,
            "cmovie_rating": 7.4
        }
    ]
}
```

Let's search the titles for "john wick". The endpoint turns it into two
`title_words CONTAINS` predicates, `AND`ed together, just like query 3. All
four John Wick movies have both words in their titles, so all four come back:

```bash
curl -s -G http://127.0.0.1:8000/movies/search \
  --data-urlencode "title=john wick" | python -m json.tool

{
    "results": [
        {
            "title": "John Wick: Chapter 4",
            "release_year": 2023,
            "cmovie_rating": 6.4
        },
        {
            "title": "John Wick: Chapter 3 – Parabellum",
            "release_year": 2019,
            "cmovie_rating": 3.9
        },
        {
            "title": "John Wick: Chapter 2",
            "release_year": 2017,
            "cmovie_rating": 6.3
        },
        {
            "title": "John Wick",
            "release_year": 2014,
            "cmovie_rating": 8.7
        }
    ]
}
```

The `–` is the en dash in "Chapter 3 – Parabellum", which `json.tool`
escapes by default.

Let's search the cast for "Tom Glynn-Carney". The endpoint strips the hyphen
and splits the name into `tom` and `glynncarney`, the same tokens the loader
stored, so it finds Dunkirk:

```bash
curl -s -G http://127.0.0.1:8000/movies/search \
  --data-urlencode "actor=Tom Glynn-Carney" | python -m json.tool

{
    "results": [
        {
            "title": "Dunkirk",
            "release_year": 2017,
            "cmovie_rating": 4.9
        }
    ]
}
```

Let's put every kind of filter together, just like query 6: a word from the
cast, a genre and a minimum rating. `rating_min` is inclusive, using `>=`
rather than query 6's `>`, but the result is the same here. The genre has a
space in it, which `--data-urlencode` encodes for us:

```bash
curl -s -G http://127.0.0.1:8000/movies/search \
  --data-urlencode "actor=murphy" --data-urlencode "genre=science fiction" \
  --data-urlencode "rating_min=7" | python -m json.tool

{
    "results": [
        {
            "title": "Star Trek: Insurrection",
            "release_year": 1998,
            "cmovie_rating": 7.1
        }
    ]
}
```

Query 5's search, highly rated dramas with "wick" in the title, comes back
empty through the API too. The endpoint returns an empty `results` list, not
an error:

```bash
curl -s -G http://127.0.0.1:8000/movies/search \
  --data-urlencode "title=wick" --data-urlencode "genre=drama" \
  --data-urlencode "rating_min=7" | python -m json.tool

{
    "results": []
}
```

Finally, let's send a request with no filters at all. It gets a `400 Bad
Request` rather than a query that would scan the whole table. The `-w` option
prints the HTTP status code after the response body:

```bash
curl -s -w "\nHTTP %{http_code}\n" http://127.0.0.1:8000/movies/search

{"detail":"at least one filter is required"}
HTTP 400
```

## The gotcha: an unindexed filter

Add a restriction on `cmovie_votes`, which has no index, and [the `ALLOW
FILTERING` error](/posts/c5-cmovie-wk01-sai-overview/#cmovie-hits-a-wall) comes
straight back:

```sql
SELECT title FROM movies
WHERE actor_words CONTAINS 'murphy' AND cmovie_votes > 5000;

InvalidRequest: Error from server: code=2200 [Invalid query]
message="Cannot execute this query as it might involve data filtering and thus
may have unpredictable performance. If you want to execute this query despite
the performance unpredictability, use ALLOW FILTERING"
```

Wire that same restriction into the API's query builder and the client gets
a bare `500 Internal Server Error`, while the `ALLOW FILTERING` error only
shows up in the Uvicorn log. Adding a filter to the API means adding a new SAI
index first.

## Recap

- The token search pattern: split text into words at load time, index the
  words, match them with `CONTAINS` at query time.
- Several SAI predicates combine across columns with `AND` (actor, genre,
  year and rating together) without `ALLOW FILTERING`.
- A thin FastAPI layer turns query parameters into those predicates, with
  plain `def` handlers because the driver call underneath blocks.

Next week, movie plots become vectors, the start of semantic search.

If you want next week's post when it lands, follow me using the social
buttons on the [8567.me](https://8567.me) homepage.

---

Movie facts from [Wikidata](https://www.wikidata.org),
[CC0](https://creativecommons.org/publicdomain/zero/1.0/). Plot text is
fetched by the loader from each movie's English Wikipedia article,
[CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/).

*Apache Cassandra, Cassandra, Apache, the Apache logo, and the Apache
Cassandra project logo are either registered trademarks or trademarks of The
Apache Software Foundation in the United States and other countries.*
