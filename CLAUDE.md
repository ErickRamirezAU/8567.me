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

## Hugo/PaperMod gotchas

- `resources.GetRemote`: `.Err` field access was removed in Hugo v0.141+. Use `{{ $result := try (resources.GetRemote $url) }}` then `{{ with $result.Err }}...{{ else with $result.Value }}...{{ end }}`.
- `transform.Unmarshal` on XML strips namespace prefixes (e.g. `<yt:videoId>` → key `videoId`) and prefixes XML attributes with `-` (e.g. `-url`) — access those with `index $m "-url"`.
- A singular XML element (e.g. one `<entry>` in an RSS feed) unmarshals to a map, not a one-item slice — normalize with `reflect.IsSlice` before ranging if the count can vary.
- Full-bleed sections: use `width: 100vw; margin-left: calc(50% - 50vw); margin-right: calc(50% - 50vw);` — don't fight the theme's `.main` padding with negative margins.
