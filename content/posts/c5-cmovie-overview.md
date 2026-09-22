---
date: '2026-09-22'
draft: false
featured: true
title: 'Build an AI movie recommender Python and TypeScript app on Cassandra'
summary: Ten weekly posts that build one AI movie app, cMovie, from scratch on
  Apache Cassandra 5.0 - SAI filtering, vector search and a RAG assistant, one
  working part at a time.
tags: ['Cassandra', 'SAI', 'Vector Search', 'AI', 'Tutorial']
cover:
  image: posts/images/c5-cmovie-overview.webp
  alt: A dense field of flat tiles thinning toward the left, with three ranked
    results picked out in orange, next to the text "A 10-week series - Search
    by what happens, not the title"
  relative: false
---

Type a sentence into a movie app:

```text
a heist that goes wrong
```

No title, no actor, no genre. The app returns a list of films whose plots are about
exactly that, closest in meaning first. Ask it "what should I watch tonight?"
and an LLM picks from films it has just retrieved for you, and tells you why.

That is where this series ends up. It takes ten weeks, and each week adds one
working part.

## What the series is

It's ten weekly posts on Apache Cassandra® 5.0 for developers building apps.
They share one scenario, **cMovie**, a fictional movie discovery app, and one
dataset: 1,000 real films from [Wikidata](https://www.wikidata.org), with plot
text from [English Wikipedia](https://en.wikipedia.org).

Learn how to retrieve data from a database with a relaxed data model using
[Storage Attached Indexes (SAI)](https://cassandra.apache.org/doc/5.0/cassandra/developing/cql/indexing/sai/sai-overview.html)
and [vector search](https://cassandra.apache.org/doc/5.0/cassandra/vector-search/overview.html).
Everything in the series is based on
[new features in Cassandra 5.0](https://cassandra.apache.org/doc/5.0/cassandra/new/index.html),
but the tutorials are run on DataStax's
[Astra DB](https://www.ibm.com/products/datastax?utm_medium=social&utm_campaign=ai-db-scale&utm_content=erickramirez)
(fully-managed Cassandra-as-a-service), so you spend your time building
rather than operating a cluster. For full disclosure, I'm an Apache
Cassandra committer and a Developer Advocate at DataStax, now an IBM
company.

The weeks alternate. A concept week is plain CQL typed into the Astra CQL
console, with nothing to install. A build week turns the idea into a small app
in Python or TypeScript. Five pairs, ten posts.

## Who it's for

Developers building apps, including developers new to Cassandra and anyone
adding AI search capabilities to an app. It isn't for database administrators.
You won't create keyspaces, pick replication strategies or operate nodes,
because Astra DB handles all that.

## One assistant, built a part at a time

Picture the finished "what should I watch tonight?" assistant as three parts.
It needs eyes to find films by their facts. It needs a sense of plot to find
films by what happens in them. And it needs taste to say which film is closer
to the one you just loved. The series builds them in that order, in three acts.

Weeks 1 and 2 come first because the assistant can't understand a film before
it can find one. The AI arrives in week 3, by design, and lands on a database
that already knows how to filter.

## Act 1: Give it eyes (weeks 1 and 2)

<!-- markdownlint-disable-next-line MD013 -->
### [Week 1 - Query any column, not just the primary key: a SQL developer's guide to Cassandra SAI](/posts/c5-cmovie-wk01-sai-overview/)

Concept week, CQL.

The question:

```text
Which science fiction films from the 2010s are rated above 7?
```

By the end you'll have SAI indexes on the `movies` table and queries that
combine genre, year and rating in one statement, without `ALLOW FILTERING`.

<!-- markdownlint-disable-next-line MD013 -->
### Week 2 - Not every search needs Elasticsearch: build a Python FastAPI filter and keyword search on Cassandra

Build week, Python.

The question:

```text
Have you got anything with Cillian Murphy in it?
```

By the end you'll have a working Python API that filters films by genre, year
range, minimum rating and actor, and searches by the words in a title or an
actor's name.

## Act 2: Give it a sense of plot (weeks 3 to 6)

<!-- markdownlint-disable-next-line MD013 -->
### Week 3 - Skip the separate vector database: vector search in the same Cassandra table as your data

Concept week, CQL.

The question:

```text
Can the database tell what a film is about?
```

By the end you'll have film plots stored as vectors, and a CQL query that
returns the plots nearest in meaning to one you describe. You'll need a free
Google AI Studio key from this week onwards.

<!-- markdownlint-disable-next-line MD013 -->
### Week 4 - Vibe search, not keywords: build semantic search with Python and Cassandra

Build week, Python.

The question:

```text
What's that film where a heist goes wrong?
```

By the end you'll have a natural language plot search in Python, with plot
embeddings generated in batches and loaded into Astra. It's the sentence from
the top of this post, working.

<!-- markdownlint-disable-next-line MD013 -->
### Week 5 - Why pure vector search returns the wrong films: filtered ANN with SAI in Cassandra

Concept week, CQL.

The question:

```text
Something like that, but only from the 1990s?
```

By the end you'll be able to combine SAI filters with vector ordering, and
you'll know how filtering interacts with the number of results you get back.

<!-- markdownlint-disable-next-line MD013 -->
### Week 6 - What should I watch tonight? Build a RAG movie assistant with Python and Cassandra

Build week, Python.

The question:

```text
What should I watch tonight?
```

By the end you'll have the assistant from the top of this post: filtered vector
retrieval feeding an LLM prompt, which is the retrieval layer of a RAG app.

## Act 3: Give it taste (weeks 7 to 10)

<!-- markdownlint-disable-next-line MD013 -->
### Week 7 - Cosine, Euclidean or dot product? Vector similarity functions in Cassandra

Concept week, CQL.

The question:

```text
How alike are these two films, really?
```

By the end you'll be able to score similarity between films in CQL with cosine,
Euclidean and dot product, and choose the right similarity function for your
index.

<!-- markdownlint-disable-next-line MD013 -->
### Week 8 - CQL can't filter on similarity: build "more like this" recommendations with TypeScript and Cassandra

Build week, TypeScript.

The question:

```text
I loved that one, so what next?
```

By the end you'll have a TypeScript recommender that starts from a seed film
and returns similar films with their similarity scores.

<!-- markdownlint-disable-next-line MD013 -->
### Week 9 - Same CQL, different fine print: SAI and vector search in Cassandra 5.0 vs Astra DB

Concept week, CQL.

The question:

```text
Will this work the same on my own cluster?
```

By the end you'll know exactly which restrictions belong to the database and
which belong to the platform. Vector search is generally available on Astra DB
but experimental in Cassandra 5.0, and this series says so plainly rather than
once in passing. This is the honest week.

<!-- markdownlint-disable-next-line MD013 -->
### Week 10 - Relevance isn't just similarity: rank vector search results with TypeScript and Cassandra

Build week, TypeScript.

The question:

```text
What's a good match for this film, and also popular?
```

By the end you'll have a TypeScript ranking that blends plot similarity with
popularity and rating, because CQL can't sort by a score you compute yourself.

## Before you start

Everything you need to
[get set up](https://github.com/ErickRamirezAU/cassandra-5-movie-search/blob/main/docs/setup.md)
is on one page, which every post links to rather than repeating. It covers
the free Astra DB account, the token, the Google AI Studio key you'll need
from week 3, and running the loader that puts the 1,000 films in your
table.

The code for all ten weeks lives in one public repo: [cassandra-5-movie-search](https://github.com/ErickRamirezAU/cassandra-5-movie-search).

You need Python 3.10 or later for the loader. Week 1 walks you through the rest.

If you want each post when it lands, follow me using the social buttons on the
[8567.me](https://8567.me) homepage.

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
