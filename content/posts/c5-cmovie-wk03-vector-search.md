---
date: '2026-10-06'
draft: false
title: "Skip the separate vector database: vector search in the same Cassandra table as your data"
summary: Store what each movie plot means as a vector column next to the
  movie itself, index it with SAI, and search by meaning with ORDER BY ... ANN
  OF in plain CQL.
tags: ['Cassandra', 'SAI', 'Vector Search', 'CQL', 'Tutorial']
cover:
  image: posts/images/c5-cmovie-wk03-vector-search.webp
  alt: A field of outlined movie record cards, with an orange dot and ring at
    the upper left linked by orange lines to a chain of four larger cards with
    orange text lines, next to the text "cMovie - Week 3 - Search by what a plot
    means"
  relative: false
---

Store each movie's plot as a vector in a column next to the movie, index that
column with a [Storage Attached Index
(SAI)](https://cassandra.apache.org/doc/5.0/cassandra/developing/cql/indexing/sai/sai-overview.html),
which is the built-in index for searching by a column's value, not just the
primary key, and order by closeness to your question. That lets you search
[Apache Cassandra®](https://cassandra.apache.org/) 5.0 by what a plot means,
not by the words in it, and you don't need a second database for it. This post
adds that column to the `movies` table in plain Cassandra Query Language (CQL).

This is the third instalment in the [Cassandra 5.0
series](/posts/c5-cmovie-overview/) where I'll show you how to build an AI movie
recommender app with Python and TypeScript.

## What you'll learn and what you need

You'll learn:

- what the `vector<float, 3072>` type holds,
- what the loader, the Python script that fills the `movies` table, sends to
  [Gemini](https://ai.google.dev/gemini-api/docs/embeddings) to fill it,
- how to index it with SAI, and
- how to search it with `ORDER BY ... ANN OF ... LIMIT`, where ANN stands for
  approximate nearest neighbour.

> [!INFO]
> **What is ANN?** A "neighbour" is a stored vector that sits close to your
> query vector, and the closer it is, the more similar its meaning. An exact
> search would compare your query with every vector in the table, which gets
> slow as the table grows. ANN, short for approximate nearest neighbour,
> skips most of that work, so it's fast but can occasionally miss a true
> neighbour. That trade-off is why the results are "very good, not exact", as
> we'll see below.

We'll continue to use the same [Astra
DB](https://www.ibm.com/products/datastax?utm_medium=social&utm_campaign=ai-db-scale&utm_content=erickramirez)
and the loaded `movies` table from [week 1](/posts/c5-cmovie-wk01-sai-overview/)
or [week 2](/posts/c5-cmovie-wk02-keyword-search/) for this tutorial.

For full disclosure, I'm an Apache Cassandra committer and a Developer Advocate
at DataStax, now an IBM company. We're using Astra DB for convenience so you
can focus on building apps as a developer without worrying about how to install
or configure a cluster, but the examples in this post are verified to run on
Cassandra 5.0.

From this week, **cMovie** needs a second key alongside your Astra DB token.
The loader uses Google's Gemini API to turn each movie's plot into an
embedding, which is a list of numbers that captures what the text means. We'll
use the same key to turn our own search phrases into embeddings too. If you
skipped it in week 1, follow [section 3 of the setup
page](https://github.com/ErickRamirezAU/cassandra-5-movie-search/blob/main/docs/setup.md#3-google-ai-studio)
to create a free API key in Google AI Studio, then paste it after
`GEMINI_API_KEY=` in your `.env` file, the file where the loader reads your
keys and settings.

Before you send anything to Gemini, know this: [Google's
terms](https://ai.google.dev/gemini-api/terms) for the free tier let it use your
requests to improve its products, and human reviewers can read them. So the
series only ever sends public [Wikipedia](https://en.wikipedia.org/) plot text
and our own search phrases.

Let's run the CQL in this post in the [Astra CQL
console](https://docs.datastax.com/en/astra-db-serverless/cql/cqlsh.html).
Start with `USE default_keyspace;`, which tells the console to run our
statements against the keyspace where the loader put the `movies` table.

> [!WARNING]
> **Run `USE default_keyspace;` before anything else.** A keyspace is the
> container that holds your tables, a bit like a namespace. The loader put the
> `movies` table in `default_keyspace`, and `USE` tells the console to look
> there. Run it at the start of every console session. If you skip it, the
> console doesn't know which keyspace you mean, so it rejects the statement
> with the error `No keyspace has been specified`. The failed statement changes
> nothing, so run `USE default_keyspace;` and try again. You can also name the
> keyspace in the statement itself, for example
> `SELECT title FROM default_keyspace.movies LIMIT 1;`.

## Where keyword search stops

[Week 2](/posts/c5-cmovie-wk02-keyword-search/) let a reader type "wick" and get
every John Wick movie back. That works because the word is in the data: we
split titles and cast names into words and indexed the words. Nothing in the
plot was searchable. Even if it were, matching words only helps when the reader
guesses the words the movie plot uses.

Now think about someone typing "a retired hitman is pulled back in". The right
movies might say "former assassin" or "returns to the underworld" and never use
those words. What we want is to search by meaning. To do that, we store what
each movie plot "means" as a list of numbers, an **embedding**, in the same row
as the movie. Next, we ask for the rows whose numbers are closest to the
numbers for the question.

## What vectors and embeddings are

An embedding model reads a piece of text and returns a list of numbers
(vectors), so that texts with similar meaning get lists that point the same
way.

Why turn meaning into numbers? A database can't compare the meaning of 2 pieces
of text directly, but it can do maths on numbers. Once each text is a list of
numbers, the database can measure how close 2 lists are, and the closer they
are, the more similar the meaning. That measurement is what makes searching by
meaning possible, and we'll discuss the mathematical functions behind it in
[week 7](/posts/c5-cmovie-overview/).

We use Google's `gemini-embedding-2`, which returns 3,072 numbers for each
plot. Each number is called a dimension. Cassandra stores that list in a
`vector<float, 3072>` column like any other value, which means 3,072 decimal
numbers (`float` values). An SAI vector index finds the nearest neighbours of a
query vector, the list of numbers for your question, quickly. That's the whole
idea.

Astra DB can also generate embeddings for you with a feature called
[Vectorize](https://docs.datastax.com/en/astra-db-serverless/databases/embedding-generation.html),
so you'd never call Gemini yourself. We don't use it here, because it isn't
part of Cassandra 5.0, and because it would hide the one idea this post is
about: a vector is just another column you write and query.

> [!INFO]
> **Why 3,072 dimensions?** It's `gemini-embedding-2`'s native size, and we
> keep all of it. The model is trained so that you can ask for fewer numbers,
> such as 1,536 or 768, and still get useful results, which saves storage. I
> checked, and when you ask Gemini's API for fewer numbers, it hands back
> vectors that are ready to compare, so there's nothing extra to do. The catch
> is slicing numbers off a longer vector yourself: the shortened vector isn't
> rescaled to match. Cosine similarity doesn't mind, but the dot product we
> meet in week 7 does. Keeping the full size means one less thing to get wrong.
> At 4 bytes a number, each movie's vector is about 12 KB, around 12 MB for all
> 1,000 movies before compression.

## Add the columns and backfill

If you loaded cMovie in week 1 or 2, you're nearly there: your `movies` table
has everything except 2 new columns. Let's add them by running the following
statement in the Astra CQL console:

```sql
ALTER TABLE movies ADD (plot_embedding vector<float, 3072>, wikipedia_url text);
```

`ALTER TABLE` changes a table that already exists, and `ADD` puts new columns
on it. Your existing rows keep their data, and the 2 new columns stay empty
until we fill them below. `plot_embedding` holds the vector, and
`wikipedia_url` links each movie to the
English Wikipedia article its plot came from. The table now looks like this
(see
[`schema.cql`](https://github.com/ErickRamirezAU/cassandra-5-movie-search/blob/main/tutorials/week-03/schema.cql)):

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
    title_words set<text>,
    plot_embedding vector<float, 3072>,
    wikipedia_url text
);
```

Then run the loader in backfill mode, which fills the new columns for movies
that already exist. Run it in your terminal, from the root directory of your
clone of the series repo, with your Python virtual environment active. It reads
the movies already in your table, so it doesn't pick a new set from Wikidata,
and only fills the 2 new columns:

```bash
python tools/loader.py --backfill
```

Backfill sends every movie's plot to Gemini, which takes a while because the
free tier limits how fast you can send. Grab a cuppa while it runs. If it stops
part-way, for example because you've reached a limit, run the same command
again. It skips the movies that already have an embedding and carries on with
the rest.

I ran it in 2 goes, a day apart, so that I stayed inside my free tier
allowance. This is the first run. The `[429]` lines are Gemini replying with the
standard "too many requests" code, which asks the loader to slow down, and it
waits as asked:

```text
Connecting to Astra keyspace 'default_keyspace' via secure-connect-cmovies.zip...

=== Stage 2: wikipedia_url and plot_embedding ===
  looking up 1000 Wikipedia article URL(s)...
  1000 of 1000 URL(s) saved
  embedding 700 plot(s), 100 per request...
  [429] Gemini asked us to wait, retrying in 51s (attempt 1)
  ...embedded 100/700
  [429] Gemini asked us to wait, retrying in 51s (attempt 1)
  ...embedded 200/700
  ...
  done: 700 plot(s) embedded
```

And this is the same command the next day. It finds the 300 movies still
without an embedding and skips the rest:

```text
Connecting to Astra keyspace 'default_keyspace' via secure-connect-cmovies.zip...

=== Stage 2: wikipedia_url and plot_embedding ===
  every movie already has a wikipedia_url
  embedding 300 plot(s), 100 per request...
  ...embedded 100/300
  ...embedded 200/300
  ...embedded 300/300
  done: 300 plot(s) embedded
```

No worries if you're starting fresh this week: you don't need either step. A full
loader run creates the table with both columns and fills them.

## What the loader sends to Gemini

We won't build the embedding step in this post, but it's worth seeing what the
loader does so you know what's in the `plot_embedding` column. For each movie,
it:

1. Takes the plot text it already fetched from the movie's English Wikipedia
   article.
2. Sends it to Gemini's `gemini-embedding-2` model, 100 plots per request,
   using the batch endpoint, which embeds many plots in one call.
3. Gets back 3,072 numbers per plot and writes them to `plot_embedding`.

Google's free tier limits how much you can send, and it counts every plot in a
batch, not just the request. So batching doesn't stretch your allowance. It
saves round trips. What protects you is that the loader paces itself, waits
when Google asks it to, and skips plots that already have an embedding. So you
can re-run it after a stop without paying for the same plots twice. On my API key,
one full load of 1,000 movies would use up the whole day's allowance, which is
why I ran mine over 2 days. Avoid re-running it from scratch. Your limits
may differ from mine, so check them on the [Rate Limit page in Google AI
Studio](https://ai.dev/rate-limit).

You don't have to use Gemini. Any embedding model works with Cassandra, as long
as you keep 2 things in step:

- **The dimension is part of the schema.** `vector<float, 3072>` only holds
  vectors with exactly 3,072 numbers. A model with a different size, say 1,536,
  means a different column type, and so a new column and a new index.
- **The query model must match the load model.** A search phrase has to be
  embedded by the same model that embedded the plots. Vectors from two
  different models aren't comparable, even when they happen to be the same
  size, and the search returns wrong results rather than an error.

To swap models, change the embedding call in the loader and in the query
helper (we meet it below), change the dimension in the column type, then
re-create the index and re-embed every movie. On the free tier, keep the daily
limit in mind when you re-embed.

## Index the vector column

Now we create the vector index by running this statement in the Astra CQL
console:

```sql
CREATE CUSTOM INDEX movies_plot_embedding_idx ON movies (plot_embedding)
USING 'StorageAttachedIndex'
WITH OPTIONS = {'similarity_function': 'COSINE'};
```

`CREATE CUSTOM INDEX` builds an SAI index on the `plot_embedding` column, and
`similarity_function` picks how the index measures closeness between 2 vectors.
`COSINE` is the default, and we state it so the choice is visible. It measures
how closely two vectors point the same way, which suits text embeddings. [Week
7](/posts/c5-cmovie-overview/) compares the other functions.

Astra DB builds the index in the background, and until it finishes a vector
query fails with a `ReadFailure` error (code 1300), which means the database couldn't
read from an index that isn't ready yet. Give it a moment before the next step.

On Cassandra 5.0, creating this index returns a client warning, which is a
message the database sends back along with the result:

<!-- Real output from the cassandra:5.0 container (5.0.9), 2026-10-02. -->

```text
Warnings :
SAI ANN indexes on vector columns are experimental and are not recommended for production use.
They don't yet support SELECT queries with:
 * Consistency level higher than ONE/LOCAL_ONE.
 * Paging.
 * No LIMIT clauses.
 * PER PARTITION LIMIT clauses.
 * GROUP BY clauses.
 * Aggregation functions.
 * Filters on columns without a SAI index.
```

> [!INFO]
> **Vector indexes are experimental in Cassandra 5.0.** The warning means what
> it says: SAI vector indexes are not recommended for production use yet, and
> the list shows what they don't support. You don't need to understand every
> item yet. We meet the ones that matter for this series, such as `LIMIT` and
> consistency level, as they come up. Astra DB doesn't print that warning,
> but the restrictions behind it apply there too, such as the cap on `LIMIT`.
> [Week 9](/posts/c5-cmovie-overview/) compares 5.0 and Astra DB in full.

## Turn a phrase into a query

The Astra CQL console can't call Gemini. So we need a way to turn a phrase such
as "a retired hitman is pulled back in" into a vector we can paste into a query.
The repo has a small helper for that. From the repo's root directory, run:

```bash
python tools/embed_query.py "a retired hitman is pulled back in"
```

It prints a complete `SELECT` statement with the vector already filled in,
about 37 KB of text, so here it is cut short:

```sql
SELECT title, release_year FROM movies ORDER BY plot_embedding ANN OF [0.00828382, -0.00360816, 0.0107079, ...] LIMIT 5;
```

Copy all of it into the CQL console and run it. The helper sends your phrase to
Gemini, so it counts against your free tier allowance too.

## Search by meaning

Time for the fun part. Let's start with the phrase from the top of this post.
Here's the query the helper wrote, with the vector cut short, and the results:

<!-- Real output from Astra DB, identical in the Astra CLI and the Astra CQL
console, 2026-10-02. -->

```sql
SELECT title, release_year FROM movies
ORDER BY plot_embedding ANN OF [0.00828382, -0.00360816, 0.0107079, ...] LIMIT 5;

 title                  | release_year
------------------------+--------------
              Ballerina |         2025
   John Wick: Chapter 4 |         2023
 Léon: The Professional |         1994
        La Femme Nikita |         1990
              Assassins |         1995

(5 rows)

Warnings :
Top-K queries can only be run with consistency level ONE / LOCAL_ONE / NODE_LOCAL. Consistency level LOCAL_QUORUM was requested. Downgrading the consistency level to LOCAL_ONE.
```

In this query, `ORDER BY plot_embedding ANN OF [...]` sorts the movies by how
close their `plot_embedding` is to the vector for your phrase, and `LIMIT 5`
keeps the 5 closest. The list of numbers in square brackets is the embedding of
your phrase, which the helper filled in for you.

The nearest neighbours are all movies about professional killers, and we never
typed "assassin". The results are ranked by closeness,
nearest first, with no scores yet. Scores come in week 7. The search is also
approximate, so a result near the edge can swap places between runs or between
databases.

The Astra CQL console prints the same warning under the rows. A consistency
level is how many copies of the data, called replicas, must answer a read.
Vector queries can only be read at `ONE` or `LOCAL_ONE`, which is a single
replica. When the client asks for something higher, here `LOCAL_QUORUM`, the
query still runs, but at
`LOCAL_ONE`, and says so ([see the 5.0
source](https://github.com/apache/cassandra/blob/cassandra-5.0/src/java/org/apache/cassandra/cql3/statements/SelectStatement.java)).
I'll leave the warning out of the results that
follow.

> [!WARNING]
> **ANN search is approximate.** To stay fast, the index doesn't compare your
> query with every vector, so it can miss a movie that is truly among the
> nearest, and results near the edge of a `LIMIT` can swap places. In my run,
> asking for `LIMIT 1000` on this table of 1,000 movies returned 999 rows, even
> though every movie has an embedding. Treat the results as very good, not exact.

A different kind of phrase works too. Let's ask for "a lonely robot learns to
love". Run the helper with that phrase, then copy the `SELECT` statement it
prints into the CQL console and run it:

```bash
python tools/embed_query.py "a lonely robot learns to love"
```

You should see 5 movies about robots and artificial intelligence, nearest
first:

```text
 title          | release_year
----------------+--------------
      Companion |         2025
 The Iron Giant |         1999
    The Creator |         2023
            Her |         2013
          M3GAN |         2022

(5 rows)
```

Rewording a query shifts the results without throwing them away. Compared to
the first query, let's ask for "an ex-assassin is dragged back into his old
life". Run the helper again with that phrase, then run the `SELECT` statement it
prints in the CQL console:

```bash
python tools/embed_query.py "an ex-assassin is dragged back into his old life"
```

You should see mostly the same movies as the first query, in a different order:

```text
 title                  | release_year
------------------------+--------------
        La Femme Nikita |         1990
              Assassins |         1995
 Léon: The Professional |         1994
                    Red |         2010
              Ballerina |         2025

(5 rows)
```

4 of the 5 movies are the same, in a different order. *John Wick: Chapter
4* dropped out and *Red* came in, which is what we'd expect when the question
means nearly the same thing.

`LIMIT` is required with `ANN OF`. It can't be above 1,000 on Astra DB, and by
default it can't on Cassandra 5.0 either ([see the 5.0
source](https://github.com/apache/cassandra/blob/cassandra-5.0/src/java/org/apache/cassandra/index/sai/StorageAttachedIndex.java)).
Week 5 shows what that means when you combine vector
search with filters.

## The gotcha: a vector of the wrong size

A query vector must have exactly 3,072 numbers, the size of the column. Anything
else is rejected:

```sql
SELECT title, release_year FROM movies
ORDER BY plot_embedding ANN OF [0.1, 0.2, 0.3] LIMIT 5;

InvalidRequest: Error from server: code=2200 [Invalid query]
message="Invalid vector literal for plot_embedding of type vector<float, 3072>;
expected 3072 elements, but given 3"
```

The error is the same on Astra DB and on Cassandra 5.0. The mistake that
doesn't produce an error is a query vector from a different model that happens
to have the same size. It runs, and the results are wrong, though they can look
plausible. That's why the helper uses the same model as the loader.

## Recap

- A vector is just another column. `vector<float, 3072>` sits next to the
  movie's other data, so there's no second database to keep in step.
- An SAI vector index and `ORDER BY ... ANN OF ... LIMIT` search that column by
  closeness, and the query vector has to come from the same model, at the same
  size, as the stored ones.
- SAI vector indexes are experimental in Cassandra 5.0, and not recommended for
  production use yet. [Week 9](/posts/c5-cmovie-overview/) compares 5.0 and
  Astra DB in full.

Next week, we turn this into a semantic search API with Python, embedding the
reader's question at request time.

If you want next week's post when it lands, follow me using the social buttons
on the [8567.me](https://8567.me) homepage.

---

Movie facts from [Wikidata](https://www.wikidata.org),
[CC0](https://creativecommons.org/publicdomain/zero/1.0/). Plot text is
fetched by the loader from each movie's English Wikipedia article,
[CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/), and the
`wikipedia_url` column links to each article.

*Apache Cassandra, Cassandra, Apache, the Apache logo, and the Apache
Cassandra project logo are either registered trademarks or trademarks of The
Apache Software Foundation in the United States and other countries.*
