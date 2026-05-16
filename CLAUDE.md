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

## Workflow: ingest publication

When the user says **"ingest publication"** (or equivalent: "add this paper", "publish this"), run this exact flow. Ask for only two things, nothing else:

1. **The DOI** (e.g., `10.1007/s00264-023-06038-8`, or a full URL like `https://doi.org/10.1007/s00264-023-06038-8` — strip the URL prefix). For arXiv-only papers, the arXiv DOI form is `10.48550/arXiv.<id>`.
2. **The PDF filename inside `files/`** (e.g., `kenanidis-et-al-00264-023-06038-8.pdf`). The user is responsible for putting the PDF there; verify it exists with `ls files/<name>` and warn if missing, but proceed anyway — they may push the PDF in the same commit.

### Fetch the BibTeX from the DOI

Run CrossRef content negotiation:

```bash
curl -sLH "Accept: application/x-bibtex" "https://doi.org/<DOI>"
```

This returns a `@article{...}` / `@inproceedings{...}` / `@incollection{...}` etc. entry with `title`, `author`, `journal`/`booktitle`, `volume`, `number`, `pages`, `year`, `month`, `publisher`, `DOI`, sometimes `abstract`. Use that BibTeX as the source for every derivation below.

**Fallbacks**:
- If `curl` returns empty or HTML (not BibTeX), the DOI is wrong or the registrar doesn't support content negotiation. Stop and ask the user to double-check the DOI.
- For arXiv DOIs (`10.48550/arXiv.<id>`), CrossRef may not have the entry. Fall back to the arXiv API: `curl -s "http://export.arxiv.org/api/query?id_list=<id>"` and parse the Atom XML for `<title>`, `<author><name>`, `<summary>` (abstract), `<published>` (date). Set `venue` to `"arXiv preprint"`.
- If the fetched BibTeX has no `abstract` field, that's fine — leave `excerpt` blank. Don't ask the user for an abstract.

### Derive everything else from the BibTeX

Map BibTeX fields to the CSV schema (`pub_date,title,venue,excerpt,citation,url_slug,paper_url`):

- **`pub_date`** (`YYYY-MM-DD`):
  - `year` + `month` present → `YYYY-MM-01` (use month number; convert `May` → `05`).
  - Only `year` → `YYYY-01-01`.
  - `@misc` with `eprint = {2408.02275}` (arXiv) → use the arXiv submission date if derivable from the ID's `YYMM` prefix → `20YY-MM-01`.
  - Call out the chosen date explicitly in the diff so the user can correct it before committing.
- **`title`** → BibTeX `title` field, with braces stripped.
- **`venue`** → `journal` (for `@article`), `booktitle` (for `@inproceedings`/`@incollection`), `publisher` (for `@book`), or `"arXiv preprint"` for arXiv `@misc`.
- **`excerpt`** → BibTeX `abstract` field if present; otherwise leave blank (the `.py` handles empty via `len > 5` check, so a blank cell is fine).
- **`citation`** → assemble in the existing house style, matching rows already in `publications.csv`:
  ```
  Authors (Last, F., Last, F., …), "Title", in Venue (optional: eds. X & Y), Publisher, doi:<doi>, Year
  ```
  Use the exact author order from BibTeX. The `doi:<doi>` segment is mandatory (DOI is a required input — see step 1). For arXiv, this is `doi:10.48550/arXiv.<id>`.
- **`url_slug`** → follow the established `paper-<type>-<N>` pattern:
  - `@article` → `paper-journal-<N>`
  - `@inproceedings`/`@conference` → `paper-conference-<N>`
  - `@incollection`/`@inbook` → `paper-chapter-<N>`
  - `@misc` (arXiv) → `paper-arxiv-<N>`
  - `<N>` = (max existing N for that type, found by `ls _publications/` + grep) + 1. If unclear, pick the next integer above the max across all types.
- **`paper_url`** → `https://papagiannakis.github.io/files/<filename>` using the filename the user provided.

### Steps to execute

1. **Append one row** to [markdown_generator/publications.csv](markdown_generator/publications.csv). Use proper RFC-4180 CSV quoting — fields with commas, quotes, or newlines wrapped in `"…"`, embedded `"` doubled to `""`. Do **not** touch `publications.tsv` (it's stale demo data).
2. **Write the markdown file** directly to `_publications/<pub_date>-<url_slug>.md`. Match the exact frontmatter shape that `publications.py` would produce (see [_publications/2024-08-05-paper-arxiv-100.md](_publications/2024-08-05-paper-arxiv-100.md) as the canonical template — fields in this order: `title`, `collection: publications`, `permalink: /publication/<pub_date>-<url_slug>`, `excerpt` (if non-empty), `date`, `venue`, `paperurl`, `citation`). HTML-escape `&` → `&amp;`, `"` → `&quot;`, `'` → `&apos;` in `title`, `excerpt`, `venue`, `citation` to match the script's output. Body: `[Download paper here](<paper_url>)` then the unescaped excerpt then `Recommended citation: <citation>`.
3. **Do NOT run `publications.py`** — it reads `.tsv` not `.csv` and would overwrite hand-edited `.md` files. Writing the single new file directly produces the same end state without collateral damage.
4. **Show the diff** (`git diff` for the CSV, `git status` + `cat` for the new `.md`) and pause for the user's OK.
5. On confirmation, **commit** with a message matching the existing style — short, lowercase, e.g. `added <venue-shortname> paper <short-title>` (see `git log --oneline _publications/` for examples like `added arxiv paper SIG-ASIA`, `updated MAGES publication`).
6. Pause again and **ask before `git push origin main`** — pushing is the public deploy step. Never push without an explicit OK in this turn. GitHub Pages rebuilds in ~30s after the push.

### Don't do

- Don't ask the user for the abstract, slug, date, citation format, or anything else — derive or default everything from the BibTeX. If a field is genuinely underivable (e.g., truly missing year), leave it blank or pick a sensible default and flag it in the diff for the user to fix.
- Don't run any bulk regeneration script. Single-row append, single-file write.
- Don't push without confirmation, even if the user OK'd the commit.
- Don't modify `publications.tsv`. It is not the source of truth and is not used by the live site.
