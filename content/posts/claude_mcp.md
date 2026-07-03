---
date: '2026-06-11'
draft: false
featured: true
title: 'Talk to your DB with no code, just Claude'
summary: Connect Claude to your DB via MCP and watch it create collections, insert records, and run bulk updates through plain English, zero code required.
tags: ['MCP', 'AI', 'Tutorial', 'Databases']
cover:
    image: posts/images/claude_mcp.png
    alt: "Cover image"
    relative: false
---

You've probably used Claude to write code, summarise documents, or debug a gnarly stack trace. But what if Claude could talk directly to your database to read and write data, run queries, manage collections, and perform bulk operations, all without you writing a single line of code?

That's exactly what the [**Model Context Protocol**](https://www.ibm.com/think/topics/model-context-protocol?utm_medium=social&utm_campaign=ai-db-scale&utm_content=erickramirez) (MCP) makes possible. In this tutorial, you'll connect **Claude Desktop** to **Astra DB** to create a document collection, insert and retrieve documents, and perform bulk updates entirely through natural language!

## Wait, what is MCP?

**MCP** (Model Context Protocol) is an open standard that lets AI assistants connect to external tools and data sources. Think of it as a USB-C port for AI: a single, consistent interface that lets Claude plug into databases, APIs, file systems, and more.

Without MCP, Claude only knows what you paste into the chat window. With MCP, Claude becomes an active agent — it can read from and write to real systems as part of the conversation.

[**Astra DB**](https://astra.datastax.com/?utm_medium=social&utm_campaign=ai-db-scale&utm_content=erickramirez) is an enterprise distribution of [**Apache Cassandra**](https://cassandra.apache.org/?utm_medium=social&utm_campaign=ai-db-scale&utm_content=erickramirez) deployed as a fully-managed service running on a public cloud of your choice. [Astra DB ships with an MCP server](https://github.com/datastax/astra-db-mcp) out-of-the-box which exposes the database as a set of tools Claude can call directly: creating collections, inserting records, running vector searches, and more. The entire conversation becomes your interface.

## Prerequisites

- [Claude Desktop](https://claude.ai/download) installed
- [Node.js](https://nodejs.org/) 18 or later installed (`node --version` to check)
- A free [**Astra DB**](https://astra.datastax.com/?utm_medium=social&utm_campaign=ai-db-scale&utm_content=erickramirez) account ([sign up here](https://astra.datastax.com/signup?utm_medium=social&utm_campaign=ai-db-scale&utm_content=erickramirez), no credit card required)

### Step 1: Create your Astra DB database

Sign in to [astra.datastax.com](https://astra.datastax.com/?utm_medium=social&utm_campaign=ai-db-scale&utm_content=erickramirez) and create a new **Serverless (Vector)** database.

Give the database any name you like (e.g. `movies-db`) and wait for it to become active. This usually takes about two minutes.

![Create Astra DB](../images/claude_mcp-a01.png)

### Step 2: Get your DB credentials

You need two things to connect to your database: an **application token** and the **API endpoint**. From the Astra DB portal, select your database.

Copy the `API_ENDPOINT` field. It looks something like `https://*.apps.astra.datastax.com`.

![Copy endpoint](../images/claude_mcp-a02.png)

Then click the **Generate Token** button to generate an application token with a DB admin role. Save the new token in a safe place. It looks something like `AstraCS:...`.

![Generate token](../images/claude_mcp-a03.png)

### Step 3: Connect Claude Desktop to your [Astra DB MCP server](https://github.com/datastax/astra-db-mcp)

Open your Claude Desktop configuration file. The location depends on your operating system:

- **macOS:** `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Windows:** `%APPDATA%\Claude\claude_desktop_config.json`

If the file doesn't exist yet, create it. Add the following, replacing the placeholder values with your actual token and endpoint:

```json
{
  "mcpServers": {
    "astra-db-mcp": {
      "command": "npx",
      "args": ["-y", "@datastax/astra-db-mcp"],
      "env": {
        "ASTRA_DB_APPLICATION_TOKEN": "AstraCS:your_token_here",
        "ASTRA_DB_API_ENDPOINT": "https://your-endpoint_here.apps.astra.datastax.com"
      }
    }
  }
}
```

**Windows users:** `npx` is a batch command on Windows. Use this modified version instead:

```json
"command": "cmd",
"args": ["/k", "npx", "-y", "@datastax/astra-db-mcp"]
```

Save the file, then **fully quit and relaunch Claude Desktop**.

And that's it for setting up your environment. If configured correctly, you should see the Astra DB MCP connector enabled in Claude Desktop.

![MCP server enabled](../images/claude_mcp-b01.png)

## Talk to your database

Now that you've got Claude Desktop configured, the fun part starts.

### New collection

First, let's create a new collection of movies. Open a new conversation in Claude Desktop and type:

```text
Create a new collection called `movies` then keep checking if it has been created.
Let me know when it has been created.
```

Claude will search for the tools required to execute the request and use the `astra-db-mcp` connector. Then it will call the `CreateCollection` tool to create the new collection and use the `GetCollections` tool for verification.

Claude then responds with a confirmation that the collection is ready.

### Load movies

Now ask Claude to populate the collection with sample movies. You can have fun with which movies you want to load but here's an example:

```text
Insert 10 mainstream movies from the last 10 years to the `movies` collection.
```

![BulkCreateRecords](../images/claude_mcp-b02.png)

Claude will call `BulkCreateRecords` to insert all ten movies at once. In this example, Claude generated movie data which includes the title, release year, director, genre and IMDB rating.

![Movies loaded](../images/claude_mcp-b03.png)

### What just happened?

Claude translated your natural language requests and mapped them to the right tools it needed to execute queries against the database, it interpreted the results, and presented those results in a very useful way.

Claude was the interfacing client to your Astra DB's MCP server, interpreting inputs and outputs without you writing code.

Let's try to do more with the database.

### Extend the schema

Ask Claude to show us the fields in our collection:

```text
Describe the fields of the `movies` collection.
```

![Describe fields](../images/claude_mcp-b04.png)

Let's try to add a new empty overview field:

```text
Add a new field to the collection called `overview` 
and set the value to empty for all records.
```

![Add overview field](../images/claude_mcp-b05.png)

Now ask Claude to generate an overview for each movie and update the records:

```text
For each movie in the collection, generate an overview 
then update the `overview` field one record at a time.
```

And just like that, the movies now have overviews included. Here's what the entry for Oppenheimer looks like:

![Oppenheimer overview](../images/claude_mcp-b06.png)

### Add a more complex data type

Let's see if we can stretch the MCP tools a bit further by adding something more complex like cast members:

```text
For each movie, generate a maximum list of 3-5 leading actors 
and their corresponding character then update each record one at a time.
Add a new `cast` field with an ordered array of objects, 
each with the `actor` and `character` as keys.
```

Let's see the results:

```text
Show me 5 movies, but only include the movie title, release year and cast members.
```

![cast members](../images/claude_mcp-b07.png)

Claude is quite clever here, filtering the results on the client-side and only showing the requested fields.

In another example when I asked to see the listing for the movie Barbie, Claude formatted the results and presented in an easy to read output:

![Barbie](../images/claude_mcp-b08.png)

## Ready to build with real data?

This tutorial used a small sample dataset to keep the setup fast but that shouldn't stop you from asking Claude to load more movies. **Astra DB**'s free tier gives you enough headroom to store millions of documents and run production-grade workloads.

If you want to build applications on Cassandra but don't know where to start, check out the reference applications at [**KillrVideo**](https://killrvideo.github.io/) with all the full working code provided.

Otherwise, [book a live demo](https://www.ibm.com/forms/mkt-demo-dataaiwatsonxdata?utm_medium=social&utm_campaign=ai-db-scale&utm_content=erickramirez) and we'll show you how!
