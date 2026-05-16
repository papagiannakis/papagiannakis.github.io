# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Dr. George Papagiannakis' personal academic website, deployed to GitHub Pages at `https://papagiannakis.github.io`. It is a fork of [academicpages](https://github.com/academicpages/academicpages.github.io), which is itself a fork of the [Minimal Mistakes](https://github.com/mmistakes/minimal-mistakes) Jekyll theme. Most of the codebase is upstream theme machinery — day-to-day work is content edits, not code changes.

## Running locally

```bash
bundle install                  # delete Gemfile.lock first if you hit version errors
bundle exec jekyll liveserve    # serves at http://localhost:4000 with live reload
```

`_config.dev.yml` overrides the production `_config.yml` for local dev (sets `url: http://localhost:4000`, disables analytics). Jekyll picks it up automatically with `liveserve` via the `hawkins` plugin declared in `Gemfile`.

Note: `_config.yml` is **not** reloaded automatically — restart the server after changing it.

There is no test suite and no linter. The `package.json` scripts (`uglify`, `watch:js`, `build:js`) are upstream theme tooling for rebuilding `assets/js/main.min.js` from the un-minified sources in `assets/js/`; they are rarely needed.

## Content model

Content is organized as Jekyll [collections](https://jekyllrb.com/docs/collections/), declared in `_config.yml` under `collections:`. Each collection is a folder of markdown files with YAML frontmatter:

- [_publications/](_publications/) — papers, with fields like `title`, `collection`, `permalink`, `excerpt`, `date`, `venue`, `paperurl`, `citation`
- [_talks/](_talks/) — talks (uses the dedicated `talk` layout)
- [_teaching/](_teaching/) — courses
- [_portfolio/](_portfolio/) — projects (currently hidden from nav)
- [_posts/](_posts/) — blog posts
- [_pages/](_pages/) — top-level pages (about, CV, contact, publications index, etc.)

Each collection has `defaults` in `_config.yml` that set its layout and which features (author profile, comments, share) are enabled — when adding a new collection or page type, set defaults there rather than per-file.

Main nav links live in [_data/navigation.yml](_data/navigation.yml). Author/bio/social data is in `_config.yml` under `author:` and rendered by [_includes/author-profile.html](_includes/author-profile.html).

## Bulk content generation

[markdown_generator/](markdown_generator/) holds Jupyter notebooks and equivalent `.py` scripts that convert TSV/CSV/BibTeX into the per-item markdown files for `_publications/` and `_talks/`:

- `publications.ipynb` / `publications.py` — from `publications.tsv`
- `PubsFromBib.ipynb` / `pubsFromBib.py` — from BibTeX
- `talks.ipynb` / `talks.py` — from `talks.tsv`

These scripts write into the collection folders. Prefer editing the TSV/BibTeX and regenerating over hand-editing many `.md` files; for one-off changes, edit the markdown directly.

[talkmap.py](talkmap.py) / `talkmap.ipynb` scrape the `location:` field from `_talks/*.md`, geocode via Nominatim, and emit a Leaflet cluster map into `talkmap/`. Run from inside `_talks/`. The `/talkmap/` page is linked from `talks.html` when `talkmap_link: true` in `_config.yml`.

## Theme structure (when you need to touch presentation)

- [_layouts/](_layouts/) — page templates (`single`, `talk`, `archive`, `splash`, …)
- [_includes/](_includes/) — partials referenced from layouts (header, footer, author profile, archive item renderers per collection)
- [_sass/](_sass/) — SCSS partials; entry point is `assets/css/main.scss`, compiled by Jekyll
- [assets/](assets/) — compiled CSS, JS bundles, fonts, images used by the theme itself
- [images/](images/) and [files/](files/) — site content assets (publication figures, PDFs). Files in `files/` are served at `/files/...` because `_config.yml` `include:` lists it explicitly.

## Upstream patching

Per [CONTRIBUTING.md](CONTRIBUTING.md) and `README.md`, the academicpages upstream tracks theme changes via GitHub issues labeled `code change`. If pulling in upstream fixes, browse those issues rather than merging the whole upstream branch — this fork has diverged in content and config and a straight merge will conflict heavily.
