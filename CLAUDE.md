# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Hugo-based personal site (vpetersson.com) combining a tech blog, video podcast ("Nerding Out with Viktor"), and professional portfolio. Hosted on GitHub Pages.

## Build & Development Commands

```bash
# Local development (live reload on localhost:1313)
hugo serve

# Production build
hugo --minify

# Linting (markdown via dprint, SCSS via stylelint, TypeScript type checking)
bun run lint

# Individual linters
bun run lint:markdown
bun run lint:markdown:fix    # auto-fix markdown
bun run lint:scss
bun run lint:scss:fix        # auto-fix SCSS
bun run typecheck

# Install JS dependencies (Tailwind, Lunr.js, linting tools)
bun install
```

## Build Pipeline

Hugo handles the entire build pipeline — no separate build step needed:

- **Sass**: `assets/css/main.scss` → compiled via Hugo's built-in Sass transpiler
- **Tailwind CSS v4**: `assets/css/tailwind.css` → processed via `css.TailwindCSS` pipe
- **TypeScript**: `assets/js/*.ts` → compiled via `js.Build` (esbuild)
- **Concatenation**: Sass + Tailwind + native.css → single bundled CSS file
- **Production**: minified + fingerprinted assets

## Content Structure

**Blog posts** (`content/blog/`): `YYYY-MM-DD-slug.md` with frontmatter:

```yaml
title: "Title"
date: 'YYYY-MM-DDTHH:MM:SSZ'
tags:
- tag1
- tag2
```

**Podcast episodes** (`content/podcast/`): `SXXEXX.md` (e.g., `S03E01.md`) with frontmatter:

```yaml
title: "Episode Title"
date: "YYYY-MM-DDTHH:MM:SS+00:00"
duration: 72          # minutes
season: 3
episode: 1
guests:
  - Guest Name
description: |
  Multi-line description...
youtube: "https://..."
spotify: "https://..."
apple: "https://..."
amazon: "https://..."
```

**Podcast transcripts** (`data/transcripts/`): JSON files accessed via `site.Data.transcripts`.

**Events** (`data/events.yaml`): Speaking engagements, panels, and external podcast appearances organized by year, newest year first. Set `upcoming: true` for future events.

Each event takes an optional `date`, a quoted ISO-8601 string — `"YYYY-MM-DD"` when the exact day is known, `"YYYY-MM"` when only the month is. The about page sorts each year's events on it, newest first; undated events render after the dated ones, in file order. Keep the file itself newest-first too, since `/speaking` and the homepage highlight read it in file order.

**Data** (`data/main.yaml`): Site-wide config — navigation, social links, podcast links, header image.

## Architecture

- **Base layout** (`layouts/_default/baseof.html`): HTML shell with head, header, main block, footer, JS loading
- **Section layouts** (`layouts/blog/`, `layouts/podcast/`): `single.html` and `list.html` for each content type
- **Page layouts** (`layouts/about/`, `layouts/projects/`, `layouts/consulting/`, `layouts/search/`): Custom single-page layouts
- **Partials** (`layouts/partials/`): ~25 reusable components — head, header, footer, nav, content cards, schema markup, hero, events, pagination
- **Taxonomies**: Hugo auto-generates tag pages (`layouts/tags/term.html`, `layouts/tags/list.html`)
- **TypeScript** (`assets/js/`): Client-side modules — mobile menu, Lunr.js search, code block copy buttons, parallax effects, GitHub star counts
- **Styling**: Tailwind CSS v4 (dark theme, custom palette in `assets/css/tailwind.css`) + custom Sass (`assets/css/`)
- **Config** (`hugo.toml`): Permalinks, taxonomies, output formats (HTML, RSS, VideoSitemap, llms.txt, Markdown copies), markup settings

## Key Patterns

- Blog permalink format: `/:year/:month/:day/:slug/` (matches legacy Jekyll URLs)
- Podcast URLs: `/podcast/:filename/`
- Transcript loading: `{{ index site.Data.transcripts (lower .File.BaseFileName) }}`
- Reading time: 183 WPM (custom partial, not Hugo's built-in `.ReadingTime`)
- YouTube ID extraction: `replaceRE` in templates
- Combined content feeds: `union` of blog + podcast pages, sorted by date

## Markdown Copies for AI Agents

Every page is also built as clean Markdown at `index.md` next to its `index.html` (`/about/` → `/about/index.md`, home → `/index.md`). `/llms.txt` lists the pages with links to their Markdown, and `/llms-full.txt` is the content of all of them in one file.

- `hugo.toml` adds the `markdown` output format to every page kind. A page that sets its own `outputs` in front matter must list `markdown` too (`content/search/` opts out on purpose).
- `layouts/partials/markdown/page.html` builds the whole copy: title, description, per-section facts (blog author/dates/tags, podcast guests/duration/listen links, project repo/license), the content, the podcast transcript from `data/transcripts/`, the events on about and speaking, and on list pages links to everything listed. `layouts/_default/single.markdown.md`, `list.markdown.md` and `layouts/home.markdown.md` only call it.
- `layouts/partials/markdown/body.html` is the content alone, also used by `llms-full.txt`. Markdown files keep their source (raw HTML in them goes through `markdown/inline-html.html`, code is left alone); `.html` content goes through `markdown/from-html.html`. Pages whose content lives in a layout (about, consulting, speaking, projects index, remote work) keep that markup in `layouts/partials/pages/`, which both the layout and the Markdown copy render, so **edit the page copy there, not in the layout**. Root-relative links are made absolute.
- `<head>` links the copy with `<link rel="alternate" type="text/markdown">`, and each sitemap URL carries the same as an `xhtml:link`. The copies are not sitemap URLs of their own. `robots.txt` points agents at `llms.txt`.
- Lychee skips `index.md` like `llms.txt`: code examples in them carry placeholder URLs.

## CI/CD

Two GitHub Actions workflows on `master`:

- **build-deploy.yml**: Hugo + Bun setup → `bun install` → `hugo --minify` → deploy to GitHub Pages (also runs daily at 1 AM UTC)
- **lint.yml**: Runs `bun run lint`; also runs `hugo --minify` build check

## AI Prompt Templates

`.prompts/` contains templates for podcast content generation (episode descriptions, transcripts, YouTube shorts).
