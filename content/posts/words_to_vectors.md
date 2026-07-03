---
date: '2026-06-03'
draft: false
title: "From Words to Vectors: A Beginner's Guide to Vector Search and AI Databases"
summary: New to embeddings, LLMs, and vector databases? Start from first principles and end up understanding how modern AI systems search for meaning, not just keywords — with a working Python example.
tags: ['AI', 'Vector Search', 'Tutorial']
cover:
    image: posts/images/words_to_vectors.png
    alt: "Cover image"
    relative: false
---

If you are a developer who is just starting to learn about AI, you have probably heard many new words: embeddings, LLMs, vector databases, semantic search. These words can feel confusing, especially when most articles assume you already know what they mean.

Don't worry. I'll attempt to explain it in as simple terms as possible. We start from the very beginning — what AI is — and by the end you will understand how modern AI systems search for meaning, not just keywords. We will also look at a short code example so you can see how this works in practice.

Let's start wide then zoom in.

## Computer intelligence

Artificial Intelligence is a broad field of computer science. The goal of AI is to make computers do things that normally require human thinking — recognising objects in a photo, translating a sentence, making a decision.

Inside this broad field, there are many sub-areas. One of the most important today is **generative AI**, a type of AI that can *create new content* — text, images, audio, video, and code. When you use a chatbot that writes an email for you, or an image tool that draws a picture from your description, you are using generative AI.

## Understanding human language

Before computers could generate text, they first needed to *understand* text. This is the job of a field called **Natural Language Processing** (NLP).

"Natural language" means the language humans use every day — English, French, Japanese. Computers do not naturally understand these languages. They only understand numbers.

NLP is a set of techniques that help computers work with human language. Early NLP systems could do simple things like finding words in a document or checking grammar. Over time, NLP became much more powerful — and that led to Large Language Models.

## AI language models

A **Large Language Model** (LLM) is a type of AI that has been trained on enormous amount of texts — books, websites, articles, code, and more. During training, it learns patterns in language: which words often appear together, how sentences are structured, what topics are related to each other.

The word "large" refers to two things: the amount of data it learned from, and the size of the model itself (measured in billions of parameters — internal numbers the model adjusts during training).

Examples of LLMs you may have heard of: GPT-4, Claude, Llama, Gemini. These models power most of the AI chatbots and writing tools available today.

Imagine a student who has read every book in every library in the world. That student has learned a huge amount about language and knowledge. An LLM is like that student — except it learned from text, not experience. It can answer questions, write text, and explain ideas based on patterns it learned.

LLMs are the engine behind Generative AI for text. But to build search systems that understand meaning, we need one more concept: vectors.

## Back to school

Before we talk about vectors as an AI concept, we need to go back to high school to do a quick maths revision.

**Scalar numbers** are single values that represent **magnitude**: how big something is or how much there is of something, otherwise a value with an associated unit. Temperature, mass, speed, and price are examples of scalar values.

27°C is a scalar. $150 is a scalar. 256 page impressions is a scalar.

**Vectors** are values that have both **magnitude AND direction**. Velocity, force, and displacement are vectors.

"60 km/h north" is a vector because it has both magnitude (60) and direction (north). 60 km/h alone is just a scalar because it only has magnitude (speed).

In programming and math, vectors are also used more loosely to mean an ordered list of numbers. A 3D vector might be [x, y, z]. This is the same concept: each element contributes to a position or direction in multi-dimensional space.

In machine learning, a **vector embedding** is a list of floating-point numbers that encodes the semantic meaning of a word, a sentence, or an image. The "direction" in high-dimensional space represents meaning, so similar things end up geometrically close to each other. That's what makes vector search work.

## Turning meaning into numbers

An **embedding** is the vector that an AI model produces when it processes some data — text, audio, image, or video. When we talk about a text embedding, we mean: "this sentence has been converted into a list of numbers that represents its meaning."

When an LLM reads a sentence, it does not just see individual words. It understands relationships. The model (LLM) produces a vector — a long list of numbers — that **encodes the meaning** of that sentence. Similar sentences produce vectors that are numerically close to each other.

Depending on the data type, models encode the semantic meaning of the data into vectors. For example, a vision model analyses pixel patterns, shapes, and objects in an image then produces a vector describing the visual content. An audio model analyses frequency, rhythm, and tone over time and encodes these into a vector.

Embeddings are numerical encoding of "features" (dimensions). It captures the semantic essence of objects or things.

To illustrate, consider a notional model that encodes features of objects in two dimensions: whether it is edible (can be eaten) or huggable. Bananas and mangoes are highly edible (encoded as positive vector magnitude) but not so huggable (negative vector). In contrast, cats and dogs are very huggable. The graph on the right is a representation of the objects in two-dimensional space.

![Example embeddings](../images/words_to_vectors-01.png)

The key insight is this: once any type of data is in vector form, a computer can perform **mathematical operations** on it. The most important operation for search is measuring the distance between two vectors. In the example above, the cat vector is "closer" to the dog vector than it is to the fruit vectors. If two vectors are numerically close, the data they represent is similar in meaning. This is the foundation of vector search.

## Searching for meaning, not text matches

Traditional search works by matching keywords. If you search for "fast database", the system looks for documents that contain the words "fast" and "database". If a document says "high-performance data store" but never uses the word "fast", traditional search might miss it.

In the following diagram, the word "capital" is used in two sentences.

![Semantics](../images/words_to_vectors-02.png)

"Capital" is a homonym; words with the same spelling but different meanings. In the first sentence, "capital" refers to the administrative seat of government. In the second, "capital" refers to the money or assets used for generating wealth.

If we do a traditional text search for "capital", it would return hits for both sentences because it is simply matching for the keyword with no context.

**Semantic search** is different. It searches for meaning, not keywords. If we provide some context, we can get results which are closer in meaning. A semantic search for "geographic capital" is more likely to return the first sentence, and a search for "initial capital" is closer to the meaning of the second sentence.

## Enter vector databases

A regular database which stores data in tables with rows and columns is good at answering "exact" questions:

- Find all the cities where country = 'Japan'
- Find all orders where the total amount > $1000

A **vector database** is designed for answering completely different questions like:

- Find the items most similar in meaning to [query]

Vector databases are specialised and purposed-built for storing and retrieving vector embeddings. But more importantly, they have the capability to perform vector searches.

![Vector DBs are built different](../images/words_to_vectors-03.png)

To answer the question efficiently, a vector database uses special indexing techniques (such as Approximate Nearest Neighbour algorithms) to quickly find vector embeddings that are "close" to the query vector even when there are millions or billions of vectors stored.

## Not all databases are created equal

Adding AI functionality isn't as simple as bolting on another database just to handle vector searches. Apart from adding another layer of complexity in your environment, it also adds latency to your application for having to query another DB.

So why not both? [**Apache Cassandra**](https://cassandra.apache.org/?utm_medium=social&utm_campaign=ai-db-scale&utm_content=erickramirez)® stores vector embeddings alongside your data in the same tables, no need for a separate vector store. And as you'd expect from any database, Cassandra does regular queries, vector search, or a **hybrid** of both in a single operation.

Additionally, Cassandra uses [**JVector**](https://github.com/datastax/jvector/) — an embedded vector search engine that [implements an incremental version of DiskANN](https://thenewstack.io/5-hard-problems-in-vector-search-and-how-cassandra-solves-them/), a more advanced algorithm than HNSW and ANN commonly found in other vector DBs. Unlike other vector DBs, data is **indexed at write-time** and is available to **query immediately**.

## Vectors in action

Let's make this concrete. The example below uses Python to:

- Generate an embedding for a text query using a popular open-source library
- Store a small collection of text embeddings in memory
- Find the most similar text to the query using cosine similarity

First, install the required library:

```bash
pip install sentence-transformers
```

Then run this Python script:

```python
from sentence_transformers import SentenceTransformer
from sklearn.metrics.pairwise import cosine_similarity
import numpy as np

# Load a pre-trained embedding model
model = SentenceTransformer('all-MiniLM-L6-v2')

# Our 'database' — a list of sentences
documents = [
    'Apache Cassandra handles billions of writes per day.',
    'Vector search finds results based on meaning, not keywords.',
    'The weather today is warm and sunny.',
    'NoSQL databases are designed for high scalability.',
    'Embeddings convert text into lists of numbers.',
]

# Convert all documents into vectors
doc_vectors = model.encode(documents)

# A query from the user
query = 'How do databases manage large amounts of data?'
query_vector = model.encode([query])

# Find the most similar document
scores = cosine_similarity(query_vector, doc_vectors)[0]
best_match_index = np.argmax(scores)

print('Query:', query)
print('Best match:', documents[best_match_index])
print('Similarity score:', round(scores[best_match_index], 4))
```

When you run this, the output will look like:

```text
Query: How do databases manage large amounts of data?
Best match: Apache Cassandra handles billions of writes per day.
Similarity score: 0.4821
```

Notice that the query never used the words "Cassandra", "writes", or "billions". But the meaning was close enough that semantic search found the right match.

In a production system, you would replace the in-memory list with a real vector database — such as [Astra DB](https://astra.datastax.com/?utm_medium=social&utm_campaign=ai-db-scale&utm_content=erickramirez) with vector search enabled (a Cassandra-as-a-service from DataStax, an IBM company) — so you can scale to millions of documents and query them in milliseconds.

## Putting It All Together

Here is a summary of the journey we took in this article:

- **AI** is a broad field. Generative AI is the part that creates content.
- **NLP** gives computers the ability to work with human language.
- **LLMs** learn patterns from enormous amounts of text and power most modern AI tools.
- **Vectors** are lists of numbers. They let computers represent meaning mathematically.
- **Embeddings** are vectors produced by AI models — they encode the meaning of text, images, audio, and more.
- **Semantic search** uses the distance between vectors to find results by meaning, not keyword.
- **Vector databases** are designed to store and search these vectors efficiently at scale.

These ideas are the foundation of almost every modern AI application — from chatbots to recommendation engines to RAG (Retrieval-Augmented Generation) pipelines. Once you understand how data becomes vectors and how vectors can be searched, a lot of AI systems start to make sense.

## It's your turn

Dive deeper into vector databases, then build an AI assistant or a Wikipedia chatbot.

[**Book a live demo**](https://www.ibm.com/forms/mkt-demo-dataaiwatsonxdata?utm_medium=social&utm_campaign=ai-db-scale&utm_content=erickramirez) and see for yourself.
