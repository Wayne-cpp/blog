# AGENTS.md — Hugo PaperMod Blog

Hugo static site (PaperMod theme) hosted on GitHub Pages. Content is primarily Chinese.

## Commands

```bash
# Local dev server (requires Hugo extended installed)
hugo server -D

# Production build (matches CI)
hugo --buildDrafts --gc --minify

# Output goes to ./public (git-ignored)
```

Hugo extended is required (not just standard Hugo). It may not be installed locally — CI handles all builds.

## Deployment

Push to `main` triggers GitHub Actions (`hugo-deploy` workflow) which builds and deploys to `gh-pages`. The workflow ignores changes to `images/`, `LICENSE`, and `README.md`.

No manual deploy step needed.

## Adding Posts

Create `.md` files in `content/posts/`. Required frontmatter:

```yaml
---
title: "文章标题"
date: 2026-05-11
tags: ["标签"]
categories: ["分类"]
summary: "摘要"
ShowToc: true
---
```

Optional frontmatter: `math: true` (KaTeX), `draft: true` (skip publish), `author`, `description`.

Goldmark `unsafe: true` is enabled — raw HTML in markdown is allowed.

## Architecture

- `config.yml` — main Hugo config (theme, menus, params, taxonomies, markup settings)
- `configTaxo.yml` — auxiliary config (unused in normal builds; taxonomies already in `config.yml`)
- `content/posts/` — blog posts
- `content/archives.md`, `content/search.md` — special layout pages
- `static/` — favicons and cover image
- `themes/hugo-PaperMod/` — git submodule from `adityatelange/hugo-PaperMod`
  - Clone with `git clone --recursive` or run `git submodule update --init --recursive`

## Gotchas

- Theme is a git submodule. If `themes/hugo-PaperMod/` is empty, run `git submodule update --init --recursive`.
- `baseURL` is set to `https://Wayne-cpp.github.io/blog` (subdirectory hosting). All internal links are relative to this path.
- The CI fetches full history (`fetch-depth: 0`) — likely needed for Hugo's lastmod or archive features.
- Commit messages use Chinese prefixes: `Add:`, `Fix:`, `Update:` (follow existing convention).
