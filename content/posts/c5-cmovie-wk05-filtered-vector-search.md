---
date: '2026-10-20'
draft: false
title: "Why pure vector search returns the wrong movies: filtered ANN with SAI in Cassandra"
summary: A vector search ranks by meaning and can't know what else you meant.
  Add ordinary SAI filters to the same ANN OF query, see which columns qualify,
  and learn what a narrow filter does to your LIMIT.
tags: ['Cassandra', 'SAI', 'Vector Search', 'CQL', 'Tutorial']
cover:
  image: posts/images/c5-cmovie-wk05-filtered-vector-search.webp
  alt: A field of outlined movie record cards, with a chain of four orange
    cards linked by orange lines inside two crossing orange frames, next to
    the text "cMovie - Week 5 - Vibe search, plus a WHERE clause"
  relative: false
---

Why does a vector search return a love story when you asked for a robot? A
vector search ranks movies by how close their meaning is to your whole phrase,
and a phrase can mean two things at once. This post adds ordinary `WHERE`
clauses to an `ORDER BY ... ANN OF` query in [Apache
Cassandra®](https://cassandra.apache.org/) 5.0, so you can say "and it must be
science fiction" and get only the movies that qualify.

This is the fifth instalment in the [Cassandra 5.0
series](/posts/c5-cmovie-overview/) where I'll show you how to build an AI movie
recommender app with Python and TypeScript.

## The robot is missing from the robot love story

Let's ask the **cMovie** database for "a lonely robot learns what love is". The
phrase has 2 halves: a lonely robot, and learning what love is. Here are the 10
closest movies. In the statement, `[...]` stands for the 3,072 numbers that
represent the phrase. We'll see where they come from shortly.

<!-- Real output from Astra DB, 2026-10-19. Vector cut short. -->

```sql
SELECT title, release_year FROM movies
ORDER BY plot_embedding ANN OF [...] LIMIT 10;

 title                | release_year
----------------------+--------------
            Companion |         2025
       The Iron Giant |         1999
          The Creator |         2023
                  Her |         2013
  Lost in Translation |         2003
    Chungking Express |         1994
                M3GAN |         2022
         Transformers |         2007
   500 Days of Summer |         2009
 Terminator Salvation |         2009

(10 rows)

Warnings :
Top-K queries can only be run with consistency level ONE / LOCAL_ONE / NODE_LOCAL. Consistency level LOCAL_QUORUM was requested. Downgrading the consistency level to LOCAL_ONE.
```

The warning under the results isn't a problem with the query. The console
sends statements at a high consistency level, `LOCAL_QUORUM`, which means
several copies of the data must answer. A vector query can only run at a low
level, where one copy answers, so Cassandra runs it at `LOCAL_ONE` and says so.
Every vector query in this post that runs prints the same warning. The blocks
below leave it out to keep them short.

Most of these movies are good matches. Three aren't: *Lost in Translation*,
*Chungking Express* and *500 Days of Summer* are romance movies with no robot in
them. They matched the love half of the phrase, and the search weighs both
halves together. Nothing in the query says "and it must be science fiction", so
the search can't know you needed that. That's the sense in which pure vector
search returns the wrong movies: it can't tell which half of your phrase you
cared about most.

## What you'll learn and what you need

You'll learn:

- how to put SAI filters and `ANN OF` in one query,
- which columns you can filter on, and why, and
- what a narrow filter does to the number of rows you get back.

A quick recap, in case you skipped earlier weeks. A Storage Attached Index
(SAI) lets you filter on columns other than the primary key. An embedding is a
list of numbers that captures what a text means, and the `plot_embedding` column
holds one for every movie's plot. `ANN OF` asks the vector index for the stored
embeddings closest to the one you give it. ANN stands for approximate nearest
neighbour: the index finds very close matches quickly instead of comparing every
row, so a result near the edge of a list can occasionally swap places.

We'll continue to use the same [Astra
DB](https://www.ibm.com/products/datastax?utm_medium=social&utm_campaign=ai-db-scale&utm_content=erickramirez)
and the `movies` table from [week 3](/posts/c5-cmovie-wk03-vector-search/), with
every plot embedded and the vector index built. There's no new index this week.
The filters use the indexes you built in [week
1](/posts/c5-cmovie-wk01-sai-overview/) on `genres`, `release_year` and
`cmovie_rating`, alongside week 3's vector index.

For full disclosure, I'm an Apache Cassandra committer and a Developer Advocate
at DataStax, now an IBM company. We're using Astra DB for convenience so you
can focus on building apps as a developer without worrying about how to install
or configure a cluster, but the examples in this post are verified to run on
Cassandra 5.0.

You also need the repo and virtual environment from [the series setup
page](https://github.com/ErickRamirezAU/cassandra-5-movie-search/blob/main/docs/setup.md)
and your `GEMINI_API_KEY` in the `.env` file, the Google AI Studio key from
[section 3 of the setup
page](https://github.com/ErickRamirezAU/cassandra-5-movie-search/blob/main/docs/setup.md#3-google-ai-studio).
This post sends one phrase to Gemini, so the same note about [Google's
terms](https://ai.google.dev/gemini-api/terms) applies: only type phrases you'd
be happy for Google to see.

We run every statement in the [Astra CQL
console](https://docs.datastax.com/en/astra-db-serverless/cql/cqlsh.html), the
page in Astra DB where you type Cassandra Query Language (CQL) statements.
Start each console session with `USE default_keyspace;`. A keyspace is the
container that holds your tables, and this statement points the console at the
one where the loader put the `movies` table. If you skip it, the console
rejects the next statement with the error `No keyspace has been specified`. If
your database has been idle for a while, check that it isn't hibernated first.
[Section 2.4 of the setup
page](https://github.com/ErickRamirezAU/cassandra-5-movie-search/blob/main/docs/setup.md#24-resuming-a-hibernated-database)
shows how.

## Get the query vector once

The CQL console can't call Gemini, so we use the helper from week 3 to turn the
phrase into a vector. Every query in this post uses the same phrase, so we
embed it once and reuse the vector. That costs one text of your free tier
allowance. From the repo's root directory, run:

```bash
python tools/embed_query.py "a lonely robot learns what love is" --limit 10
```

The helper prints a complete `SELECT` statement with the 3,072 numbers filled
in, about 37 KB of text. Copy it into the console and run it. For each query
that follows, you don't need to call Gemini again. Edit the `WHERE` clause of
the statement you already have, and keep the same vector. A 37 KB statement can
be awkward to edit in the console, so paste it into a text editor first, make
your change, then paste the result back. The statements below leave the vector
out for space.

The helper can also write the `WHERE` clause for you. The `--where` option adds
a filter, and `--columns` picks the columns to show. For example, the next
section's query comes from this command, though it embeds the phrase again and
uses another text of your allowance, so editing the statement is the cheaper
route:

```bash
python tools/embed_query.py "a lonely robot learns what love is" --limit 10 \
  --where "genres CONTAINS 'science fiction'"
```

## Add a genre

Each movie has a `genres` column, a set of labels such as `science fiction` or
`romantic comedy`, taken from Wikidata. A set is a collection that holds each
value once, and `CONTAINS` asks whether a set holds a given value. Here are one
movie's labels, so you can see what we match against:

```sql
SELECT title, release_year, genres FROM movies
WHERE genres CONTAINS 'science fiction' LIMIT 1;

 title                    | release_year | genres
--------------------------+--------------+--------------------------------------------
 X-Men Origins: Wolverine |         2009 | {'action', 'science fiction', 'superhero'}

(1 rows)
```

Now let's add that filter to the vector query. The `WHERE` clause goes before
`ORDER BY`:

```sql
SELECT title, release_year FROM movies
WHERE genres CONTAINS 'science fiction'
ORDER BY plot_embedding ANN OF [...] LIMIT 10;

 title                               | release_year
-------------------------------------+--------------
                           Companion |         2025
                      The Iron Giant |         1999
                         The Creator |         2023
                                 Her |         2013
                               M3GAN |         2022
                        Transformers |         2007
                Terminator Salvation |         2009
 Transformers: Revenge of the Fallen |         2009
                             Contact |         1997
    Transformers: Rise of the Beasts |         2023

(10 rows)
```

Compared with the first query, the 3 romance movies are gone, and *Contact* and
2 more *Transformers* movies took their places. The first query never mentioned
a genre, so this is the whole difference one `WHERE` clause makes.

The label has to match exactly. The index compares the text as it's written,
including upper and lower case, so this finds nothing, because the label is
`science fiction` and not `Science Fiction`:

```sql
SELECT title, release_year FROM movies
WHERE genres CONTAINS 'Science Fiction' LIMIT 3;

 title | release_year
-------+--------------

(0 rows)
```

An empty result isn't an error, so a wrongly cased label looks like "no movies
match". Most labels are lowercase, with a few capitalised ones such as
`Christmas`, so copy the label from the `genres` column.

Here's what Cassandra does with the two parts of the query. It first finds the
rows that match the `WHERE` clause, then ranks only those by closeness of
meaning. It doesn't rank every movie and throw the wrong ones away afterwards.
I checked this in the [5.0 source, pinned to commit
`81b458017d`](https://github.com/apache/cassandra/blob/81b458017d8aef6e7dcd91d3f31d651bdad1aaa8/src/java/org/apache/cassandra/index/sai/plan/QueryController.java#L389),
where a comment on the filter-plus-vector path says "We apply all other
predicates before ordering and limiting". That holds for every filter in this
post. The source has a different path for filters that match more than 16,000
rows by default, which I haven't tested and cMovie's 1,000 movies can't reach.

## Add a decade, then a minimum rating

Filters combine with `AND`, the same as the `WHERE` clauses in
[week 1](/posts/c5-cmovie-wk01-sai-overview/). Let's keep science fiction and
add the 2010s. `release_year` has an SAI index from week 1, which is why a range
works:

```sql
SELECT title, release_year FROM movies
WHERE genres CONTAINS 'science fiction'
AND release_year >= 2010 AND release_year < 2020
ORDER BY plot_embedding ANN OF [...] LIMIT 10;

 title                           | release_year
---------------------------------+--------------
                             Her |         2013
        Star Wars: The Last Jedi |         2017
                     The Martian |         2015
                  Captain Marvel |         2019
                            Lucy |         2014
                      Prometheus |         2012
                    Tron: Legacy |         2010
    Star Wars: The Force Awakens |         2015
 Transformers: Age of Extinction |         2014
                      Ex Machina |         2014

(10 rows)
```

Now a minimum rating. `cmovie_rating` is a fictional in-app number the loader
generates, so it isn't a real audience score. It's indexed too. We'll use 7.0,
which sits above the median (the middle value) of 6.4 and just under the 75th
percentile of 7.3, so it keeps about a third of the movies. cqlsh prints a
rating of 7.0 as `7`:

```sql
SELECT title, release_year, cmovie_rating FROM movies
WHERE genres CONTAINS 'science fiction'
AND release_year >= 2010 AND release_year < 2020
AND cmovie_rating >= 7.0
ORDER BY plot_embedding ANN OF [...] LIMIT 10;

 title                                 | release_year | cmovie_rating
---------------------------------------+--------------+---------------
              Star Wars: The Last Jedi |         2017 |             7
                           The Martian |         2015 |           8.6
                        Captain Marvel |         2019 |           7.3
                          Tron: Legacy |         2010 |             7
                        Justice League |         2017 |           8.8
 The Hunger Games: Mockingjay – Part 1 |         2014 |           7.3
                         Suicide Squad |         2016 |             7
                  Thor: The Dark World |         2013 |           7.8
                        Men in Black 3 |         2012 |           7.5
        Guardians of the Galaxy Vol. 2 |         2017 |           7.7

(10 rows)
```

None of these statements needs `ALLOW FILTERING`, the keyword that lets
Cassandra read rows and discard the ones that don't match, which gets slower as
the table grows. Every column we restricted has an SAI index, and all the SAI
indexes on a table work together as one group, so Cassandra can use them
alongside the vector index.

A filter has a limit, though: it narrows the pool, but it doesn't make the
matches closer. With genre, decade and rating together, the 10 results are the
10 nearest of what's left, and they drift from the robot theme: *Justice League*
and *Men in Black 3* aren't about lonely robots. `ANN OF` always returns the
nearest `LIMIT` rows, however far away they are.

## What a narrow filter does to `LIMIT`

Every query so far matched more than 10 movies, so each returned exactly 10.
What happens when a filter matches fewer than the `LIMIT`? Only 4 of the 1,000
movies carry the `boxing` label. Let's ask for 50:

```sql
SELECT title, release_year, genres FROM movies
WHERE genres CONTAINS 'boxing'
ORDER BY plot_embedding ANN OF [...] LIMIT 50;

 title               | release_year | genres
---------------------+--------------+-------------------------------------
              Snatch |         2000 |          {'boxing', 'crime comedy'}
 Million Dollar Baby |         2004 |        {'boxing', 'buddy', 'drama'}
         The Fighter |         2010 | {'biographical', 'boxing', 'drama'}
             Rocky V |         1990 |       {'action', 'boxing', 'drama'}

(4 rows)
```

We get all 4, ranked by closeness. Fewer rows than the `LIMIT` means fewer
movies matched the filter, not that Cassandra missed some. I got the same
behaviour on Astra DB and Cassandra 5.0.9. It's what you want, because your app
can show the user 4 results without guessing. `LIMIT` is still required, and by
default Cassandra 5.0 caps it at 1,000, as we saw in [week
4](/posts/c5-cmovie-wk04-semantic-search/).

## The gotcha: a filter on a column with no index

So far we've filtered on indexed columns. What about one without an index?
`cmovie_votes` is a plain column. Let's ask for science fiction movies with more
than 1,000 votes:

```sql
SELECT title, release_year FROM movies
WHERE genres CONTAINS 'science fiction' AND cmovie_votes > 1000
ORDER BY plot_embedding ANN OF [...] LIMIT 10;

InvalidRequest: Error from server: code=2200 [Invalid query] message="Ordering on non-clustering column requires each restricted column to be indexed except for fully-specified partition keys"
```

That wording is Astra DB's. A 5.0 cluster says `ANN ordering by vector
requires all restricted column(s) to be indexed`, as with week 4's `LIMIT`
errors. Both mean the same thing: a vector query can only filter on columns that
have an SAI index. In an ordinary query, you'd add `ALLOW FILTERING` and accept
the slow scan. Unfortunately it doesn't help here:

```sql
SELECT title, release_year FROM movies
WHERE genres CONTAINS 'science fiction' AND cmovie_votes > 1000
ORDER BY plot_embedding ANN OF [...] LIMIT 10 ALLOW FILTERING;

InvalidRequest: Error from server: code=2200 [Invalid query] message="Ordering on non-clustering column requires each restricted column to be indexed except for fully-specified partition keys"
```

The same error comes back. The fix is an index on `cmovie_votes`, created with
the same [`CREATE CUSTOM INDEX`
statement](/posts/c5-cmovie-wk01-sai-overview/#add-storage-attached-indexes) as
in week 1. Plan the indexes for the filters your app offers before you build the
screen that uses them. We don't create one in this post.

## Recap

- One query can combine meaning and ordinary filters: `WHERE` goes before
  `ORDER BY ... ANN OF`.
- Cassandra applies the filters first, then ranks the matches by closeness. A
  filter narrows the pool, and the results are only as close as what's left.
- You can only filter on columns with an SAI index, and `ALLOW FILTERING`
  doesn't change that.
- `LIMIT` is still required. A filter that matches fewer rows than the `LIMIT`
  returns every match.

> [!INFO]
> **Vector indexes are experimental in Cassandra 5.0.** [Storage Attached Index
> (SAI)](https://cassandra.apache.org/doc/5.0/cassandra/developing/cql/indexing/sai/sai-overview.html)
> vector indexes are not recommended for production use yet. The 5.0 warning
> lists several restrictions we met in this post, such as the required `LIMIT`
> and the indexed-columns-only filters. Astra DB doesn't print the 5.0 warning,
> but the restrictions apply there too. [Week 9](/posts/c5-cmovie-overview/)
> compares 5.0 and Astra DB in full.

Next week, we hand these filtered results to an LLM and build a movie assistant
in Python, so its suggestions come from movies that match what you asked for.

Try your own phrase and filters. The statements from this post are in
[`tutorials/week-05/`](https://github.com/ErickRamirezAU/cassandra-5-movie-search/tree/main/tutorials/week-05)
in the series repo. Swap in a different genre label, change the years or pick a
new minimum rating, and see which movies drop out. If you want next week's post
when it lands, follow me using the social buttons on the
[8567.me](https://8567.me) homepage.

---

Movie facts from [Wikidata](https://www.wikidata.org),
[CC0](https://creativecommons.org/publicdomain/zero/1.0/). Plot text is
fetched by the loader from each movie's English Wikipedia article,
[CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/), and the
`wikipedia_url` column links to each article.

*Apache Cassandra, Cassandra, Apache, the Apache logo, and the Apache
Cassandra project logo are either registered trademarks or trademarks of The
Apache Software Foundation in the United States and other countries.*
