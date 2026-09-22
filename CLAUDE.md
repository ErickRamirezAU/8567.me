# 8567.me

Personal site for Erick Ramirez ("AI Decoded"), built with Hugo (extended) and the PaperMod theme. Articles on AI, databases, and software development.

## Stack

- Hugo static site, theme **PaperMod** vendored as a git submodule at `themes/PaperMod` — never edit the theme directly.
- Theme overrides live in `layouts/` (`layouts/index.html`, `layouts/about.html`, `layouts/_partials/`).
- Custom styling and design tokens live in `assets/css/extended/custom.css` — accent colors (`--accent`, `--accent-strong`, etc.), surface tokens, and font vars, with light/dark variants under `:root[data-theme="dark"]`.
- Content lives in `content/posts/*.md`; post images go in `content/posts/images/` using `<slug>.ext` for the cover image and `<slug>-a01.ext`, `-a02.ext`, ... for inline images in order.
- Site config: `hugo.yaml`.

## Local dev

Run the dev server with `hugo server -D` (a `.claude/launch.json` config named
`hugo-server` already does this). Stop it as soon as you're done verifying a
change — don't leave it running.

## Deployment

Push to `main` triggers `.github/workflows/deploy.yml`: builds with
`hugo --minify`, then uploads `public/` to `public_html/` via **SFTP**
(rsync-based actions won't work on this host). Don't reach for rsync-based
GitHub Actions here.

## Conventions

- Don't edit `themes/PaperMod/` directly — it's a submodule; overrides belong in `layouts/` or `assets/css/extended/custom.css`.
- `public/` and `resources/` are generated/gitignored — never hand-edit them.
- New posts: use the `scrape-to-hugo-post` skill (`.claude/skills/scrape-to-hugo-post/`) to convert a source URL into a Hugo/PaperMod markdown post.

## Cover images

Workflow established for the cMovie series, reused for each new post:

1. Generate a ChatGPT image prompt for the concept — dark background, a
   motif matching the brand (e.g. node-graph/tile pattern), Ember accent
   colour, with a clear region (typically the left third) reserved empty
   for a text overlay.
2. Draft 2-3 hook/headline text options (eyebrow + headline) for that
   overlay alongside the image prompt.
3. Import the generated image into the
   `CREATIVES-2026-site_thumbnails` Google Slides deck
   (`My Drive/8567.me/CREATIVES-2026-site_thumbnails.gslides`) and
   composite the text overlay there — image-gen models don't render
   legible text reliably, so text is added in Slides instead.
4. Export the finished slide, convert to WebP with `cwebp -q 90`
   (macOS `sips` can read PNG but can't write WebP), and place it at
   `content/posts/images/<slug>.webp`, wired into the post's `cover:`
   frontmatter.
5. Once the post is live, cross-link it with the series overview post:
   make that week's heading in the overview a link to the new post, and
   add a link back to the overview near the top of the new post. See
   [content/posts/c5-cmovie-overview.md](content/posts/c5-cmovie-overview.md)
   and
   [content/posts/c5-cmovie-wk01-sai-overview.md](content/posts/c5-cmovie-wk01-sai-overview.md)
   for the pattern.

Gotchas hit building this workflow:

- `sips -s format webp` fails (`Error: Can't write format:
  org.webmproject.webp`) — use `cwebp` instead (Homebrew,
  `/opt/homebrew/bin/cwebp`). `-q 90` gives near-visually-lossless output
  (~54dB PSNR) at roughly a fifth the size of the source PNG.
- A post's `date:` with no explicit offset is parsed as UTC. Combined
  with `hugo server -D` (drafts only, not `-F` future), a post dated
  "today" in local AEST time can land in the future in UTC terms and get
  silently excluded from the build — no error, it just won't appear.
  Match the date to the rest of the batch, or add `timeZone` to
  `hugo.yaml` if this keeps happening.
- `.featured-post .entry-cover img` (in `custom.css`): don't combine an
  explicit `height: 100%` with `aspect-ratio` when the container also has
  `align-items: stretch` — the explicit height wins over `aspect-ratio`
  and stretches the image to match the sibling column's content height,
  which `object-fit: cover` then crops hard to compensate. Use
  `align-items: center` and size the image from `aspect-ratio` alone.
- `object-fit: cover`'s default `object-position: 50% 50%` crops evenly
  from both edges. If the source image has a text overlay near one edge,
  set `object-position` to protect that edge (e.g. `left center`) rather
  than padding the source image itself.

## Hugo/PaperMod gotchas

- `resources.GetRemote`: `.Err` field access was removed in Hugo v0.141+. Use `{{ $result := try (resources.GetRemote $url) }}` then `{{ with $result.Err }}...{{ else with $result.Value }}...{{ end }}`.
- `transform.Unmarshal` on XML strips namespace prefixes (e.g. `<yt:videoId>` → key `videoId`) and prefixes XML attributes with `-` (e.g. `-url`) — access those with `index $m "-url"`.
- A singular XML element (e.g. one `<entry>` in an RSS feed) unmarshals to a map, not a one-item slice — normalize with `reflect.IsSlice` before ranging if the count can vary.
- Full-bleed sections: use `width: 100vw; margin-left: calc(50% - 50vw); margin-right: calc(50% - 50vw);` — don't fight the theme's `.main` padding with negative margins.
