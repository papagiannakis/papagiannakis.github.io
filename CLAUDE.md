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

## Workflow: ingest talk

When the user says **"ingest talk"** (or equivalent: "add this talk", "publish this talk"), run this flow. Talks are looser than publications — there is no DOI/CrossRef equivalent, so metadata extraction is best-effort and **must be confirmed by the user before files are written**, not after.

### Inputs — accept any of these three shapes

Try them in order; whichever the user provides is what you use.

1. **URL + filename** (canonical, most analogous to the publication flow):
   - **URL** to the talk's announcement / event page (conference program, IEEE/ACM session listing, keynote speakers page, etc.).
   - **Filename inside `files/`** for the slides PDF (e.g., `GP-IEEEVRkeynote2023.pdf`). If the talk is video-only, the user can give a video URL instead — set `talk_url` to that URL directly and skip the `files/` check.
2. **Pasted text blob + filename** — if there's no clean URL, the user pastes the CFP/program text/announcement and provides the slides filename.
3. **Explicit fields + filename** — if the user just lists `title:`, `venue:`, `date:`, `location:`, `type:`, optional `description:`, take them verbatim.

### Resolve metadata

- For Option 1: `WebFetch` the URL and extract `title`, `venue`, `date`, `location`, `description` (abstract), `type`.
- For Option 2: parse the blob for the same fields.
- For Option 3: trust the fields as given.
- For all: if `talk_url` is a slides PDF, verify with `ls files/<name>` and warn (don't block) if missing.

### Map to the talks schema

Columns in [markdown_generator/talks.csv](markdown_generator/talks.csv): `title, type, url_slug, venue, date, location, talk_url, description`.

- **`title`** — from the source.
- **`type`** — one of `Talk`, `Tutorial`, `Keynote Talk`, `Invited Talk`, `Panel`. Default to `Talk` if unclear. The `type` controls the slug prefix.
- **`url_slug`** — `talk-<N>` for nearly everything. The actual house convention (visible in `talks.csv`) is a **single global counter**, not type-prefixed: Invited Talks, Tutorials, workshops, and some Keynote Talks all use `talk-<N>`. The prefix `keynote-<N>` is an occasional exception applied to some keynotes (e.g., `keynote-17`, `keynote-25`) but is **not consistent** — other keynotes use `talk-<N>` (e.g., `talk-24` is a Keynote Talk). Default to `talk-<N>` unless there's a clear reason to mirror the `keynote-` style. `<N>` is a single global counter across both prefixes: extract trailing integers from every `url_slug` in `markdown_generator/talks.csv` and every filename in `_talks/*.md`, take the max, add 1. The CSV is the more reliable source because `_talks/` is sparse.
- **`venue`** — conference/event name (e.g., `IEEE Virtual Reality 2023`).
- **`date`** — `YYYY-MM-DD`. If only year+month known, default day to `01`.
- **`location`** — `"City, Country"` or `"online"` (matches existing rows).
- **`talk_url`** — `https://papagiannakis.github.io/files/<filename>` for slides, or the raw video/event URL.
- **`description`** — abstract/summary. May be empty.

### Steps to execute

1. **Resolve and show derived values FIRST** — before writing anything, print the extracted/derived values and ask the user to confirm or correct. This is the key difference from publications: CrossRef gives clean data, but talk extraction is fuzzy. Echo back something like:
   ```
   title:       …
   type:        Keynote Talk
   url_slug:    keynote-18
   venue:       …
   date:        2026-05-03
   location:    …
   talk_url:    …
   description: <first 200 chars>…
   ```
   Wait for an OK or corrections.
2. **Append one row** to `markdown_generator/talks.csv` with proper RFC-4180 CSV quoting (commas/quotes/newlines wrapped in `"…"`, embedded `"` doubled to `""`). Do **not** touch `talks.tsv` (stale demo data).
3. **Write the markdown file** to `_talks/<date>-<url_slug>.md`. Match the frontmatter shape that [markdown_generator/talks.py](markdown_generator/talks.py) produces — fields in this order: `title` (double-quoted), `collection: talks`, `type` (double-quoted), `permalink: /talks/<date>-<url_slug>`, `venue` (double-quoted) if present, `date`, `location` (double-quoted) if present. Body: optional `[More information here](<talk_url>)` line, then the description. Note `talks.py` does **not** HTML-escape the way `publications.py` does, so keep raw `"` / `'` in the description — but be careful that the title and venue, which the script wraps in double quotes, do not themselves contain unescaped `"`. If they do, replace with `&quot;` or use a different quoting style.
4. **Do NOT run `talks.py`** — same reasoning as publications: it reads `.tsv` not `.csv`, and would clobber any hand-edited talk pages.
5. **Show the diff** (`git diff` for the CSV, `git status` for the new `.md` and any slides PDF) and pause for the user's OK to commit.
6. On confirmation, **commit** with a message matching the publication style — short, lowercase, e.g. `added <venue-shortname> <type> on <short-topic>` (e.g., `added IEEE VR 2023 keynote on geometric algebra`).
7. **Regenerate the talk map** (see next subsection). This produces a separate commit when the map output changes.
8. Pause and **ask before `git push origin main`** — push both commits together. Never push without explicit OK. GitHub Pages rebuilds in ~30s.

### Regenerate the talk map

The home page and `/talks/` link to a Leaflet cluster map at [talkmap/map.html](talkmap/map.html), generated from the `location:` field of every `_talks/*.md`. Each new talk's location needs to be added to [talkmap/org-locations.js](talkmap/org-locations.js) so its pin appears.

**Canonical source is [_talks/talkmap.ipynb](_talks/talkmap.ipynb).** Do NOT use the root-level `talkmap.ipynb` or `talkmap.py` — both are stale upstream copies (the `.py` calls `Nominatim()` with no `user_agent` which breaks on `geopy >= 2.0`; the root `.ipynb` predates the rewrite).

The current notebook is **cache-then-incremental**: it parses the existing `org-locations.js`, scans `_talks/*.md` for `location:` fields, and geocodes **only** locations not already cached. It uses Komoot's **Photon** geocoder (free, no key, lenient rate limits) — Nominatim was abandoned because it bans us within seconds when we re-geocode every location on every run (which the original notebook did).

To run it, either open it in Jupyter or execute non-interactively from the repo root:

```bash
/Users/papagian/opt/anaconda3/bin/jupyter nbconvert --to notebook --execute --inplace _talks/talkmap.ipynb
```

The notebook is idempotent — running it with no new talks reports `Missing: []` and rewrites `org-locations.js` with the same content (no diff). After running, check `git diff talkmap/`:

- If the diff is **just a few `+` lines** for the new talk's location → expected, commit as `updated talkmap`.
- If the diff shows **many `-` lines** → something went wrong, revert with `git checkout -- talkmap/` and investigate.
- If `git status talkmap/` shows nothing → the new talk's location was already in the cache (e.g., another talk had the same city). Skip the talkmap commit.

Caveats:

- **Dependencies**: `getorg`, `geopy` (Photon is in geopy), and `jupyter`. Anaconda has all of these by default. If you get `ModuleNotFoundError` from a different Python, run `python3 -m pip install --user getorg geopy jupyter`.
- **Wrong-interpreter trap on macOS**: `pip` and `python3` often point at different installs (anaconda vs system). Use the full path `/Users/papagian/opt/anaconda3/bin/jupyter` and `/Users/papagian/opt/anaconda3/bin/python` to be safe, or use `python3 -m pip ...` for installs.
- **Photon quirks**: the geocoded `address` field may name a nearby business rather than the city center (e.g., "Coventry, UK" resolved to "FEV UK Ltd, Cheetah Road, Coventry"). The lat/lon coordinates are still correct for the city, which is all the cluster map needs. Don't worry about the verbose address string.
- **If Photon also fails** (rare — happens during outages): manually geocode via the Photon web UI at https://photon.komoot.io, copy the lat/lon, and append a new entry to `org-locations.js` by hand.
- **`map.html` is static** and never regenerated by the new notebook. Only `org-locations.js` changes per talk.

After the script runs, check the diff:

```bash
git status talkmap/ && git diff --stat talkmap/
```

If `map.html` or `org-locations.js` changed, **commit as a separate commit** matching the existing house style — `git log --oneline talkmap*` shows the pattern: `updated talkmap`, `updated talk map`, `fix talkmap`. Use `updated talkmap`.

If nothing changed (e.g., the new talk's location was already represented), skip the talkmap commit silently.

### Don't do

- Don't write files before the user confirms the derived metadata. Talk extraction is fuzzy — confirm first.
- Don't run `talks.py` or any bulk regeneration script.
- Don't run `talkmap.py` or the root `talkmap.ipynb` — both are stale. Use `_talks/talkmap.ipynb`.
- Don't switch back to Nominatim "to match the original data". The cache preserves Nominatim's original results; only new locations use Photon. Mixing is fine — both report WGS84 lat/lon to enough precision for a cluster pin.
- Don't modify `talks.tsv`. It is not the source of truth.
- Don't push without confirmation, even if the user OK'd the commit.
- Don't invent a `talk_url` — if the user gave neither a slides filename nor a video/event URL, leave the column blank and skip the "More information here" line in the markdown body.
- Don't skip talkmap regeneration silently when the script fails — surface the error so the user knows the map is stale.
