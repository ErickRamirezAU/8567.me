# 8567.me

[![hugo](https://img.shields.io/badge/topic-hugo-E8622C)](https://github.com/topics/hugo)
[![papermod](https://img.shields.io/badge/topic-papermod-E8622C)](https://github.com/topics/papermod)
[![apache-cassandra](https://img.shields.io/badge/topic-apache--cassandra-E8622C)](https://github.com/topics/apache-cassandra)
[![vector-search](https://img.shields.io/badge/topic-vector--search-E8622C)](https://github.com/topics/vector-search)
[![ai](https://img.shields.io/badge/topic-ai-E8622C)](https://github.com/topics/ai)

This is the source for [8567.me](https://8567.me), where I write about AI,
databases and building apps on Apache Cassandra®. You'll find hands-on
tutorials, blog posts and the occasional deep dive, written for developers at
any level.

For full disclosure, I'm an Apache Cassandra committer and a Developer Advocate
at DataStax, now an IBM company.

## What's on the site

The posts cover three areas:

- **Cassandra for app developers.** Querying any column with Storage-Attached
  Indexing (SAI), keyword search and vector search, all in the same table as
  your data.
- **AI and vector search.** How embeddings work, and why a separate vector
  database isn't always the answer.
- **AI tools that talk to your database.** Using the Model Context Protocol
  (MCP) to create and update data in plain English.

If you'd like a place to start, I recommend the
[cMovie series](https://8567.me/posts/c5-cmovie-overview/). Over 10 weekly
posts, we build an AI movie recommender app in Python and TypeScript on
Cassandra 5.0. The code for every week is in the
[cassandra-5-movie-search](https://github.com/ErickRamirezAU/cassandra-5-movie-search)
repo.

To browse everything else, see the [tags](https://8567.me/tags/) page or
[search](https://8567.me/search/) the site.

## How it's built

The site is built with [Hugo](https://gohugo.io/) and the
[PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme. PaperMod is
included as a git submodule in `themes/PaperMod`, and my changes to it live in
`layouts/` and `assets/css/extended/custom.css`, so the theme itself stays
untouched. The posts are Markdown files in `content/posts/`.

Every push to `main` runs a GitHub Actions workflow that builds the site and
publishes it to [8567.me](https://8567.me).

## Run it locally

You will need [Hugo extended](https://gohugo.io/installation/) and git. Clone
the repo with its submodule, because the site doesn't build without the theme:

```bash
git clone --recurse-submodules https://github.com/ErickRamirezAU/8567.me.git
cd 8567.me
hugo server
```

Hugo prints the local address to open in your browser. Add `-D` to include
draft posts.

## Contributing

Thanks for stopping by. It's a personal site, so I'm not looking for content
contributions, but I'd love to hear from you:

- If you spot a typo, a broken link or an example that doesn't work, please
  [open an issue](https://github.com/ErickRamirezAU/8567.me/issues).
- If there's something you'd like to build with AI on Cassandra, let me know in
  an issue or on [LinkedIn](https://www.linkedin.com/in/ErickRamirez/).

---

_Apache Cassandra, Cassandra, Apache, the Apache logo, and the Apache Cassandra
project logo are either registered trademarks or trademarks of The Apache
Software Foundation in the United States and other countries._
