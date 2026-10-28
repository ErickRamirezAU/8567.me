---
date: '2026-10-29'
draft: false
title: "What should I watch tonight? Build a RAG movie assistant with Python and Cassandra"
summary: A search returns a list of titles and leaves you to work out why they fit.
  Build a FastAPI endpoint that finds the right movies with a filtered ANN OF
  query, then has Gemini explain the picks using only those movies.
tags: ['Cassandra', 'SAI', 'Vector Search', 'RAG', 'Python', 'FastAPI', 'Tutorial']
cover:
  image: posts/images/c5-cmovie-wk06-rag-movie-assistant.webp
  alt: A field of outlined movie record cards, with a chain of three orange
    cards linked by orange lines running down the middle, next to the text
    "cMovie - Week 6 - What should I watch tonight?"
  relative: false
---

How do you turn a list of search results into an answer to "what should I watch
tonight?" You pass the best matches to a large language model (LLM) and ask it
to recommend from those movies only. This pattern is called
[retrieval-augmented generation](https://en.wikipedia.org/wiki/Retrieval-augmented_generation),
or RAG for short, and this post builds it into one
[FastAPI](https://fastapi.tiangolo.com/) endpoint on
[Apache Cassandra®](https://cassandra.apache.org/) 5.0 and Google's Gemini.

This is the sixth instalment in the [Cassandra 5.0
series](/posts/c5-cmovie-overview/) where I'll show you how to build an AI movie
recommender app with Python and TypeScript.

## A list of titles isn't a recommendation

Last week we ran a filtered vector search, and in [week
4](/posts/c5-cmovie-wk04-semantic-search/) we put a search like it behind an
endpoint. Here's what that endpoint says to a user who types "A gripping mystery
where nothing is what it seems" into **cMovie**:

<!-- Real output from Astra DB, 2026-10-21. -->

```bash
curl -s -G http://127.0.0.1:8000/movies/semantic \
  --data-urlencode "q=A gripping mystery where nothing is what it seems" \
  --data-urlencode "limit=3" | python -m json.tool

{
    "results": [
        {
            "title": "Wake Up Dead Man",
            "release_year": 2025
        },
        {
            "title": "A Haunting in Venice",
            "release_year": 2023
        },
        {
            "title": "The Others",
            "release_year": 2001
        }
    ]
}
```

The titles are good, but the user has to work out why they fit, whether they've
seen them and which to pick. You could ask an LLM the same question directly,
but then it answers from whatever it learnt in training. It doesn't know which
movies are in your catalogue, so it can recommend a movie your app doesn't have
and describe a plot it half remembers.

RAG sits between the two. The database finds the movies, and the LLM explains
them. This post builds that, and we'll see what happens when the LLM is allowed
to go beyond the movies it's given.

## What you'll learn and what you need

You'll learn:

- how a RAG request flows through your app in 3 steps,
- how to build a filtered `ANN OF` query from optional parameters,
- how to write a prompt that keeps the LLM to the movies you give it, and
- what the endpoint does when Google says no.

We'll continue to use the same [Astra
DB](https://www.ibm.com/products/datastax?utm_medium=social&utm_campaign=ai-db-scale&utm_content=erickramirez)
and the `movies` table from [week 3](/posts/c5-cmovie-wk03-vector-search/), with
every plot embedded and the vector index built. There's no new index this week.
The filters use the indexes from [week
1](/posts/c5-cmovie-wk01-sai-overview/), as they did in [week
5](/posts/c5-cmovie-wk05-filtered-vector-search/).

For full disclosure, I'm an Apache Cassandra committer and a Developer Advocate
at DataStax, now an IBM company. We're using Astra DB for convenience so you
can focus on building apps as a developer without worrying about how to install
or configure a cluster, but the examples in this post are verified to run on
Cassandra 5.0.

You also need Python 3.10 or later, the virtual environment from [the series
setup
page](https://github.com/ErickRamirezAU/cassandra-5-movie-search/blob/main/docs/setup.md)
and your `GEMINI_API_KEY` in the `.env` file, the Google AI Studio key from
[section 3 of the setup
page](https://github.com/ErickRamirezAU/cassandra-5-movie-search/blob/main/docs/setup.md#3-google-ai-studio).
If your database has been idle for a while, check that it isn't hibernated
first. [Section 2.4 of the setup
page](https://github.com/ErickRamirezAU/cassandra-5-movie-search/blob/main/docs/setup.md#24-resuming-a-hibernated-database)
shows how.

Each request in this post sends your text to Gemini twice, once to embed it and
once to write the answer, so the same note about [Google's
terms](https://ai.google.dev/gemini-api/terms) applies: only type requests you'd
be happy for Google to see.

## The 3 steps of a RAG request

Every request to the new endpoint does 3 jobs in order:

1. **Retrieve.** Embed the user's request with `gemini-embedding-2`, the model
   that embedded the plots, and ask Cassandra for the 3 movies whose plots sit
   closest to it. Any filters the user sent narrow the pool first, as in week 5.
2. **Augment.** Write those 3 movies into a block of text: title, year, genres,
   rating, the start of the plot and the Wikipedia link. *Augment* means adding
   to the question, here with facts the LLM wouldn't otherwise have.
3. **Generate.** Send the request and that block to Gemini's Flash-Lite model
   with instructions to recommend only from the block.

Step 1 uses ideas we've used since week 3, so we'll start there. Step 2 is
formatting, and step 3 is new. We'll build both before the endpoint that ties the
3 steps together.

## Step 1: retrieve with a filtered query

The endpoint accepts a `prompt` and 4 optional filters: `genre`, `year_from`,
`year_to` and `rating_min`. These are the same filters as week 2's search, with
the same names. The CQL changes with whichever filters the caller sends, so the
endpoint builds the `WHERE` clause from a list of pieces. With every filter set,
the statement reads:

```sql
SELECT title, release_year, genres, cmovie_rating, plot, wikipedia_url
FROM movies
WHERE genres CONTAINS ? AND release_year >= ? AND release_year <= ?
AND cmovie_rating >= ?
ORDER BY plot_embedding ANN OF ? LIMIT 3
```

The query returns 2 more columns than week 4's: `plot`, because the LLM needs
the story to explain why a movie fits, and `wikipedia_url`, so the answer can
link to the article that the plot text came from. Remember that `cmovie_rating`
is a made-up number generated by the loader for this fictional app, not a real
audience score.

The `?` markers are placeholders for values, and the driver fills them in so
that nothing a user types is ever pasted into the statement text. In the code
the placeholders are written `%s`, which is how the driver spells them for a
statement built at request time. That's also why this statement isn't prepared
once at startup, as week 4's was: its shape changes with the filters. A
consistency level has to be set on it too. As week 4 explained, a vector query
can only be read at `ONE` or `LOCAL_ONE`, so the statement asks for `ONE`
itself.

## Step 2: augment with the retrieved movies

Gemini only knows what's in the message you send it, so step 2 turns the 3 rows
into plain text and puts that text after the user's request. The
`movie_context()` function in the endpoint below does it, writing one block per
movie. This is the block for the first movie of a search for 1990s family
adventures, with the plot cut to its first 300 characters to keep the prompt
short. The plot text is from the English Wikipedia article on [*Home
Alone*](https://en.wikipedia.org/wiki/Home_Alone):

<!-- Real output from the series repo's movie_context(), 2026-10-21. -->

```text
Home Alone (1990)
Genres: Christmas, adventure, comedy, family
cMovie rating: 5.3 out of 10
Plot: The McCallister family is preparing to spend Christmas in Paris, gathering at Peter and Kate's home in a Chicago suburb the night before their departure. Their youngest son, Kevin, is the subject of ridicule by his older siblings because of his immaturity. During a scuffle with his oldest brother Bu
Wikipedia: https://en.wikipedia.org/wiki/Home_Alone
```

This is the only place the LLM gets movie facts from, which is why the query in
step 1 selects `plot` and `wikipedia_url`.

## Step 3: generate with Gemini

Gemini's `generateContent` endpoint takes your instructions and your text and
returns the model's reply. We call it the way
[`tools/embeddings.py`](https://github.com/ErickRamirezAU/cassandra-5-movie-search/blob/main/tools/embeddings.py)
calls the embedding endpoint, with Python's standard library only, so there's
nothing new to install. The new helper lives in
[`tools/llm.py`](https://github.com/ErickRamirezAU/cassandra-5-movie-search/blob/main/tools/llm.py):

<!-- markdownlint-disable MD013 -->

```python
import json
import urllib.error
import urllib.request

from tools.embeddings import (
    API_BASE,
    DEFAULT_RETRY_SECONDS,
    DailyLimitReached,
    RateLimited,
    _is_daily_limit,
    _retry_seconds,
)

MODEL = "gemini-3.5-flash-lite"


class LLMError(RuntimeError):
    """Any failure from the Gemini API other than a 429."""


def generate_text(api_key: str, system_prompt: str, user_prompt: str) -> str:
    body = json.dumps(
        {
            "system_instruction": {"parts": [{"text": system_prompt}]},
            "contents": [{"role": "user", "parts": [{"text": user_prompt}]}],
            "generationConfig": {"maxOutputTokens": 512},
        }
    ).encode("utf-8")
    request = urllib.request.Request(
        f"{API_BASE}/models/{MODEL}:generateContent",
        data=body,
        headers={"Content-Type": "application/json", "x-goog-api-key": api_key},
        method="POST",
    )
    try:
        with urllib.request.urlopen(request, timeout=60) as response:
            data = json.loads(response.read().decode("utf-8"))
    except urllib.error.HTTPError as e:
        raw = e.read().decode("utf-8", errors="replace")
        try:
            error_body = json.loads(raw)
        except json.JSONDecodeError:
            error_body = {}
        if e.code == 429:
            if _is_daily_limit(error_body):
                raise DailyLimitReached(error_body["error"]["message"].splitlines()[0]) from e
            raise RateLimited(_retry_seconds(error_body) or DEFAULT_RETRY_SECONDS) from e
        message = error_body.get("error", {}).get("message", raw[:300])
        raise LLMError(f"Gemini returned HTTP {e.code}: {message}") from e
    except (urllib.error.URLError, OSError) as e:
        raise LLMError(f"Couldn't reach Gemini: {e}") from e

    try:
        return data["candidates"][0]["content"]["parts"][0]["text"]
    except (KeyError, IndexError) as e:
        raise LLMError("Gemini's answer didn't contain any text") from e
```

<!-- markdownlint-enable MD013 -->

There are 4 things worth knowing:

- **The key goes in a header.** `x-goog-api-key` keeps it out of the URL, so it
  can't end up in a log or an error message.
- **There's no retry loop.** Week 4's `embed_texts()` needed a `retry=False`
  flag because the loader wants to wait. This helper is only for web requests,
  so it never waits: a `429` raises `RateLimited` straight away, carrying the
  delay Google asked for. It reuses the exceptions and the parsing from
  `tools/embeddings.py` because `generateContent` reports its limits in the same
  shape.
- **The system prompt is separate from the user's text.** It sets the rules
  that apply to every request, while the user's text only says what they want.
- **The model name is a constant.** I tested `gemini-3.5-flash-lite` on
  2026-10-21, after Google retired the older `gemini-2.0-flash-lite`. Google
  retires model names from time to time, so if the call fails with HTTP 404,
  check the current Flash-Lite name on the [Gemini models
  page](https://ai.google.dev/gemini-api/docs/models). I picked a Flash-Lite
  model because in my testing it left more room on the free tier than the larger
  Flash model, which matters when you're rewriting a prompt over and over. Check
  your own limits on the [Rate Limit page in Google AI
  Studio](https://ai.dev/rate-limit).

## The prompt

The system prompt is the most important text in this post, because it's what
keeps the LLM honest:

```python
SYSTEM_PROMPT = (
    "You are cMovie Assistant, a friendly movie concierge. "
    "Recommend movies ONLY from the list in the user's message. "
    "For each pick, say why it fits the request, using its plot, and mention its "
    "release year and cMovie rating. End each pick with its Wikipedia link. "
    "If none of the movies fit the request, say so politely and don't suggest "
    "anything else. Never invent titles, plot details or cast members."
)
```

Giving the model a way out matters as much as the rules. "If none of the movies
fit the request, say so" lets it decline instead of stretching a weak match.
Grounding the answer in the retrieved rows like this is what makes it RAG.

The user's message is the request followed by the block of movies from step 2.

## The endpoint

Here's the new endpoint in
[`app/main.py`](https://github.com/ErickRamirezAU/cassandra-5-movie-search/blob/main/app/main.py),
next to week 2's and week 4's. It uses the `session` and `GEMINI_API_KEY` that
the file already sets up:

<!-- markdownlint-disable MD013 -->

```python
from cassandra.query import SimpleStatement

from tools.llm import LLMError, generate_text


def movie_context(rows) -> str:
    return "\n\n".join(
        f"{row.title} ({row.release_year})\n"
        f"Genres: {', '.join(sorted(row.genres))}\n"
        f"cMovie rating: {row.cmovie_rating:.1f} out of 10\n"
        f"Plot: {row.plot[:300]}\n"
        f"Wikipedia: {row.wikipedia_url}"
        for row in rows
    )


def gemini_http_error(e: Exception) -> HTTPException:
    if isinstance(e, DailyLimitReached):
        return HTTPException(status_code=429, detail=f"Gemini's free tier limit for today is used up: {e}")
    if isinstance(e, RateLimited):
        return HTTPException(
            status_code=429,
            detail="Gemini is rate limiting this key, try again shortly",
            headers={"Retry-After": str(math.ceil(e.retry_after))},
        )
    return HTTPException(status_code=502, detail=str(e))


@app.get("/movies/assist")
def assist(
    prompt: str = Query(min_length=1, max_length=500),
    genre: Optional[str] = None,
    year_from: Optional[int] = None,
    year_to: Optional[int] = None,
    rating_min: Optional[float] = None,
):
    # Step 1: embed the request and find the 3 closest movies that pass the filters.
    try:
        (vector,) = embed_texts(GEMINI_API_KEY, [prompt], retry=False)
    except (DailyLimitReached, RateLimited, EmbeddingError) as e:
        raise gemini_http_error(e)

    clauses, params = [], []
    if genre:
        clauses.append("genres CONTAINS %s")
        params.append(genre)
    if year_from is not None:
        clauses.append("release_year >= %s")
        params.append(year_from)
    if year_to is not None:
        clauses.append("release_year <= %s")
        params.append(year_to)
    if rating_min is not None:
        clauses.append("cmovie_rating >= %s")
        params.append(rating_min)
    where = f"WHERE {' AND '.join(clauses)} " if clauses else ""

    query = SimpleStatement(
        "SELECT title, release_year, genres, cmovie_rating, plot, wikipedia_url "
        f"FROM movies {where}ORDER BY plot_embedding ANN OF %s LIMIT 3",
        consistency_level=ConsistencyLevel.ONE,
    )
    rows = list(session.execute(query, params + [vector]))

    # Nothing passed the filters, so there's nothing to ground an answer in.
    if not rows:
        return {
            "recommendation": "I couldn't find any movies that match those filters.",
            "candidates": [],
        }

    # Steps 2 and 3: put the movies in the prompt and ask Gemini to answer.
    try:
        recommendation = generate_text(
            GEMINI_API_KEY, SYSTEM_PROMPT, f"{prompt}\n\nMovies:\n\n{movie_context(rows)}"
        )
    except (DailyLimitReached, RateLimited, LLMError) as e:
        raise gemini_http_error(e)

    return {
        "recommendation": recommendation,
        "candidates": [
            {
                "title": row.title,
                "release_year": row.release_year,
                "genres": sorted(row.genres),
                "cmovie_rating": round(row.cmovie_rating, 1),
                "wikipedia_url": row.wikipedia_url,
            }
            for row in rows
        ],
    }
```

<!-- markdownlint-enable MD013 -->

There are 4 things worth knowing:

- **The vector goes last.** The `WHERE` placeholders come first in the statement,
  so their values come first in the list, and the vector for `ANN OF` is added
  at the end.
- **A genre label is used exactly as typed.** Week 5 showed that labels are
  case sensitive, and `Christmas` is a real label, so the endpoint doesn't
  change the case. A wrong case returns no rows, not an error.
- **Empty results skip Gemini.** If the filters match nothing, there's nothing to
  recommend, so the endpoint answers itself. That saves a call to Google and
  removes any chance of the LLM making something up.
- **The response has both halves.** `recommendation` is the text for a person,
  and `candidates` is the same 3 movies as data, so a front end can draw cards
  beside the text.

## Try it

Time to take it for a spin. Start the app from the root of the repo with your
virtual environment active, as in week 4, and leave it running:

```bash
uvicorn app.main:app --reload
```

You can also open `http://127.0.0.1:8000/docs` in a browser to try the endpoint
without a curl command. Let's start with the request from the top of this post,
in a second terminal window:

<!-- Real output from Astra DB, 2026-10-21. -->

<!-- markdownlint-disable MD013 -->

```bash
curl -s -G http://127.0.0.1:8000/movies/assist \
  --data-urlencode "prompt=A gripping mystery where nothing is what it seems" | python -m json.tool

{
    "recommendation": "Here are a few gripping mysteries from the list where nothing is quite what it seems:\n\n*   **A Haunting in Venice (2023)**\n    *   **Why it fits:** This gripping mystery fits your request as a retired detective is pulled into a seemingly supernatural situation on Halloween, where he must uncover the truth behind a medium at a palazzo, proving that deception and unexpected twists lurk around every corner. \n    *   **Rating & Release:** Released in 2023, it holds a cMovie rating of 7.3 out of 10.\n    *   **Wikipedia:** https://en.wikipedia.org/wiki/A_Haunting_in_Venice\n\n*   **The Others (2001)**\n    *   **Why it fits:** This atmospheric mystery and horror film is the ultimate tale of things not being what they seem. Set in a darkened country house where a mother protects her light-sensitive children, the arrival of new servants unravels chilling secrets in a narrative where reality is constantly questioned.\n    *   **Rating & Release:** Released in 2001, it holds a cMovie rating of 7.5 out of 10.\n    *   **Wikipedia:** https://en.wikipedia.org/wiki/The_Others_(2001_film)",
    "candidates": [
        {
            "title": "Wake Up Dead Man",
            "release_year": 2025,
            "genres": [
                "comedy",
                "detective fiction",
                "mystery"
            ],
            "cmovie_rating": 6.9,
            "wikipedia_url": "https://en.wikipedia.org/wiki/Wake_Up_Dead_Man"
        },
        {
            "title": "A Haunting in Venice",
            "release_year": 2023,
            "genres": [
                "crime",
                "detective",
                "drama",
                "mystery",
                "neo-noir",
                "police",
                "thriller"
            ],
            "cmovie_rating": 7.3,
            "wikipedia_url": "https://en.wikipedia.org/wiki/A_Haunting_in_Venice"
        },
        {
            "title": "The Others",
            "release_year": 2001,
            "genres": [
                "drama",
                "horror",
                "mystery"
            ],
            "cmovie_rating": 7.5,
            "wikipedia_url": "https://en.wikipedia.org/wiki/The_Others_(2001_film)"
        }
    ]
}
```

<!-- markdownlint-enable MD013 -->

The `\n` marks inside `recommendation` are line breaks, which a front end
renders. The same 3 movies came back as in week 4, but now there's a reason next
to each one, the year and rating, and a link. Notice that the model chose 2 of
the 3 candidates. It left out *Wake Up Dead Man*, which the search ranked first.
The search finds candidates, and the model decides which of them answer the
request, using the plots.

Now let's add filters. The `genre`, `year_from` and `year_to` parameters narrow
the pool before the ranking, exactly as in week 5. To make the output easier to
read, this command and the next pipe the response through a one-line Python
command that prints only the recommendation:

```bash
curl -s -G http://127.0.0.1:8000/movies/assist \
  --data-urlencode "prompt=A nostalgic adventure" \
  --data-urlencode "genre=family" \
  --data-urlencode "year_from=1990" \
  --data-urlencode "year_to=1999" \
  | python -c "import sys, json; print(json.load(sys.stdin)['recommendation'])"

Hello! I would love to help you find a nostalgic adventure. Here are some great picks from the list that fit your request:

* **Home Alone (1990)**
  * **Why it fits:** This classic adventure follows young Kevin McCallister as he is accidentally left behind while his family prepares to spend Christmas in Paris, leading to a nostalgic and hilarious holiday journey.
  * **Release Year:** 1990
  * **cMovie Rating:** 5.3 out of 10
  * **Wikipedia:** https://en.wikipedia.org/wiki/Home_Alone

* **Home Alone 2: Lost in New York (1992)**
  * **Why it fits:** Delivering a wonderful dose of nostalgia, this film takes the adventure to a new level when Kevin gets separated from his family's holiday trip and ends up navigating the big city of New York on his own.
  * **Release Year:** 1992
  * **cMovie Rating:** 8.6 out of 10
  * **Wikipedia:** https://en.wikipedia.org/wiki/Home_Alone_2:_Lost_in_New_York
```

Every movie is a 1990s family movie, because the database applied the filters
before the model saw anything. The model can't recommend a 2005 movie here, even
if it wanted to.

A request takes between 2.5 and 6 seconds on my laptop, and your time will
differ with your network. Most of it is the 2 calls to Google. The Cassandra
query takes under a second.

## The gotcha: an LLM that's allowed to improvise

Here's where RAG goes wrong. Ask for a story about competitive rowing, but limit
it to horror, which is a combination that can't give a sensible answer. The
`horror` filter leaves plenty of movies, so Cassandra still returns the 3
horror plots closest to "rowing", and none of them is about rowing. What does the
assistant say?

```bash
curl -s -G http://127.0.0.1:8000/movies/assist \
  --data-urlencode "prompt=A story about competitive rowing" \
  --data-urlencode "genre=horror" \
  | python -c "import sys, json; print(json.load(sys.stdin)['recommendation'])"

I'm sorry, but none of the movies in my current list are about competitive rowing.
```

That's the system prompt doing its job. It was allowed to decline, so it did.
Now let's see what happens with a looser prompt. I ran the same request and the
same 3 movies with the system prompt replaced by `"You are a helpful movie
assistant."`:

```text
It looks like none of the movies in your provided list are about competitive rowing!

- *A Quiet Place Part II* is a post-apocalyptic horror film about surviving blind alien monsters.
- *The Ring* is a psychological horror/mystery about a cursed videotape.
- *Frankenstein* (2025) is a Gothic sci-fi adventure based on the classic monster tale.

If you are looking for a movie about competitive rowing, you might want to check out films like **The Boys in the Boat** (2023), which tells the inspiring true story of the University of Washington rowing team during the Great Depression, or **True Blue** (1996), which focuses on the famous Oxford-Cambridge boat race.

Would you like help finding more movies about rowing?
```

The first half is fine, but then the model offers 2 movies from its own memory.
Neither is in the `movies` table, and I checked: a title search for either
returns nothing. A user of your app would go looking for 2 movies you can't
show them. In a real app, that's the failure RAG exists to prevent, and it
costs nothing to prevent: 2 sentences of instructions, "ONLY from the list" and
"don't suggest anything else".

There's a second layer too. When the filters match no rows at all, the endpoint
never calls Gemini, so there's nothing for the model to improvise from:

<!-- markdownlint-disable MD013 -->

```bash
curl -s -G http://127.0.0.1:8000/movies/assist \
  --data-urlencode "prompt=A lone gunslinger rides into town" \
  --data-urlencode "genre=western" \
  --data-urlencode "year_from=2020" | python -m json.tool

{
    "recommendation": "I couldn't find any movies that match those filters.",
    "candidates": []
}
```

<!-- markdownlint-enable MD013 -->

The table has no westerns from 2020 onwards, so the query returns 0 rows. This
isn't an error, as in week 5, and the endpoint answers in the time of one query
and one embedding.

## Another gotcha: 2 calls to Google per request

Week 4's endpoint made one call to Google per search. This one makes 2: the
embedding, then the answer. Either can be refused, because free tier limits
apply to both, and a refusal at the second step happens after the first has
already used some of your allowance. So both calls are wrapped, and both turn
the failure into the same HTTP status a client expects:

- A daily limit or a per-minute limit becomes `429`, with a `Retry-After` header
  for the per-minute case that says how many seconds to wait.
- Any other failure from Google becomes `502`, the code for "a service behind
  this app failed".

The `gemini_http_error()` function in the endpoint does the translation for both
steps, so week 4's rules apply unchanged. I tested these paths with a stand-in
for Google's replies, not by running my quota down again, so there's no output
to show. When a real limit does hit you, the message comes from Google and the
`429` is the endpoint's, as it was in week 4. Check your own limits on the
[Rate Limit page in Google AI
Studio](https://ai.dev/rate-limit), and remember that each request spends one
embedding text and one generation call from them.

## Insert your LLM here

You don't have to use Gemini for this step. Only `tools/llm.py` talks to
Google's generation endpoint, so swapping models means changing `MODEL`, or
rewriting `generate_text()` for another provider's API. The database and the
retrieval don't change. The embedding model is a different matter: that one
must keep matching the model that embedded the plots, as week 4 explained.

## Recap

- A RAG request retrieves matching rows from Cassandra, adds them to a prompt
  and asks an LLM to answer from them.
- The filters run in the database before the LLM sees anything, so the answer
  can only come from movies that qualify.
- A system prompt that says "only from this list" and allows "none fit" keeps the
  LLM from recommending movies your app can't show.
- Each request makes 2 calls to Google, so both are wrapped and turned into
  `429` or `502`.

> [!INFO]
> **Vector indexes are experimental in Cassandra 5.0.** [Storage Attached Index
> (SAI)](https://cassandra.apache.org/doc/5.0/cassandra/developing/cql/indexing/sai/sai-overview.html)
> vector indexes are not recommended for production use yet. The retrieval step
> in this post depends on one, so keep that in mind before you build on it. Astra
> DB doesn't print the 5.0 warning, but the restrictions apply there too. [Week
> 9](/posts/c5-cmovie-overview/) compares 5.0 and Astra DB in full.

Next week, we look under the hood of vector search and find out what "closest"
means, with [week 7](/posts/c5-cmovie-overview/) comparing cosine, Euclidean and
dot product similarity.

Try your own requests and tweak the system prompt to change the assistant's
personality, such as a noir detective or a movie trivia buff. The
statements and the code from this post are in
[`tutorials/week-06/`](https://github.com/ErickRamirezAU/cassandra-5-movie-search/tree/main/tutorials/week-06)
in the series repo. If you want next week's post when it lands, follow me using
the social buttons on the [8567.me](https://8567.me) homepage.

---

Movie facts from [Wikidata](https://www.wikidata.org),
[CC0](https://creativecommons.org/publicdomain/zero/1.0/). Plot text is
fetched by the loader from each movie's English Wikipedia article,
[CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/), and the
`wikipedia_url` column links to each article. The assistant's answers are
generated by Gemini from the start of those plots.

*Apache Cassandra, Cassandra, Apache, the Apache logo, and the Apache
Cassandra project logo are either registered trademarks or trademarks of The
Apache Software Foundation in the United States and other countries.*
