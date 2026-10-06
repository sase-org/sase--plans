---
tier: tale
title: "sase-listen: fetch arXiv paper URLs as their PDF"
goal:
  "`sase-listen render`/`script` recognize arXiv paper URLs (/abs/, /html/, /pdf/
  variants) and fetch the paper's canonical PDF URL, so an /abs/ link narrates the full
  paper and every URL form for one paper shares one cached source and episode."
size: small
proposed_by: bbugyi200.athena.0xa
create_time: 2026-10-06 10:06:56
status: wip
---

# Plan: sase-listen fetches arXiv paper URLs as their PDF

## Problem

`sase-listen render https://arxiv.org/abs/2602.16844 -e full` fetches the arXiv
**abstract landing page** as HTML. Trafilatura extracts only the abstract plus page
chrome (364 words), so the "full" edition is a 186-word script (W012: under 50% of the
2,400-word budget) and a 1m57s episode. The same paper given as
`https://arxiv.org/pdf/2602.16844` yields 11,452 words, a 1,229-word, 6-chapter script,
and an 8m48s episode. The abstract page clears the 150-word minimum, so nothing flags
the thin source. The two URL forms also land in different cached sources and different
library episodes because their canonical URLs differ.

Goal: when `render` / `script` are given an arXiv paper URL, recognize it and fetch the
paper's canonical PDF URL instead, so every arXiv URL form for one paper shares one
source, writer script, and library episode.

## Where the work happens

All changes are in the linked `sase-listen` repo. Open it with
`sase repo open sase-listen -r "<reason>"`, work only in the printed path, and read its
`AGENTS.md` first. Nothing changes in `sase` or `sase-core`: sase-listen must not import
`sase` (its no-sase-import rule), and rendering audio is not shared backend behavior.

Relevant code (sase-listen paths):

- `src/sase_listen/web/store.py::acquire(url, *, html_file, refresh, on_step)` is the
  URL-to-cached-source boundary. It computes `requested_key = normalize_url(url)`,
  checks `index.json` (`_cached`) unless `html_file`/`refresh`, emits
  `on_step(f"fetching {host}")`, calls `fetch_page(url)` (or
  `load_html_file(url, html_file)` for `--html FILE`), extracts, and writes
  `index[requested_key]` and `index[canonical]`.
- `src/sase_listen/pipeline.py::load_source()` calls `acquire()` for URL sources and
  derives `source_label`, `source_key_base = f"url:{canonical}"`, and `source_url` from
  `metadata["canonical_url"]`. For PDFs, `web/pdf.py::extract_pdf` sets
  `canonical_url = normalize_url(page.final_url)`.
- `web/pdf.py` already tunes extraction for arXiv: it reads the arXiv stamp for the
  date, sets site `arXiv`, and drops references. Using the PDF therefore gives the best
  result for arXiv.
- `web/fetch.py::fetch_page` is a generic HTTP fetcher. Keep it generic.

## Design

### 1. New pure module `src/sase_listen/web/arxiv.py`

The module does only string and URL logic: no I/O and no new dependencies. Public API:

```python
def arxiv_paper_id(url: str) -> str | None:
    """Return the arXiv identifier for an arXiv paper page URL, else None."""

def arxiv_pdf_url(url: str) -> str | None:
    """Return https://arxiv.org/pdf/<id> for an arXiv paper page URL, else None."""
```

Recognition rules for `arxiv_paper_id`:

- Parse with `urlsplit(url.strip())`. The scheme must be `http` or `https`
  (case-insensitive).
- The hostname (already lowercased by `urlsplit`) must exactly equal one of `arxiv.org`,
  `www.arxiv.org`, or `export.arxiv.org`. Exact matching rejects look-alikes such as
  `notarxiv.org` and `arxiv.org.evil.test`.
- Accepted paths (one optional trailing `/`):
  - `/abs/<id>`: the abstract landing page, which is the case being fixed.
  - `/html/<id>`: the arXiv HTML full-text view. It also resolves to the PDF so that one
    paper has one identity, and because the PDF extractor is the arXiv-tuned path.
  - `/pdf/<id>` and `/pdf/<id>.pdf`: already PDFs. They are only canonicalized, so the
    `.pdf`, `http://`, `www.`, and `export.` variants share one cache key.
- `<id>` is either:
  - a new-style ID `\d{4}\.\d{4,5}`, or
  - an old-style ID `[a-z]+(?:-[a-z]+)*(?:\.[A-Za-z]{2})?/\d{7}`, for example
    `hep-th/9901001` or `math.GT/0309136`.

  Either form may carry an optional version suffix `v\d+`, which is preserved exactly.
  An old-style ID contains a `/`, so match the whole path with an anchored regex rather
  than splitting on `/`.

- Query strings and fragments are ignored, for example `?context=cs.AI` or `#S3` on an
  `/html/` URL.
- Everything else returns `None` and is fetched as an ordinary URL, exactly as today:
  - the home, `/list/`, `/search/`, and `/a/` pages;
  - `/src/`, `/e-print/`, `/format/`, and `/ps/`;
  - `/abs/` with no ID or a malformed ID;
  - other hosts, including paper mirrors such as `alphaxiv.org` and
    `huggingface.co/papers` (not arXiv URLs; out of scope);
  - non-http schemes.

`arxiv_pdf_url` returns `f"https://arxiv.org/pdf/{paper_id}"`: always https, always the
bare `arxiv.org` host, never a `.pdf` suffix. This matches the documented form already
in use (`https://arxiv.org/pdf/2608.25174`), so existing cached PDF sources keep their
keys.

### 2. Resolve in `acquire()` before any cache or fetch work

In `web/store.py::acquire`, when `html_file is None`, resolve the URL first:

```python
paper_id = arxiv_paper_id(url) if html_file is None else None
if paper_id is not None:
    url = arxiv_pdf_url(url)  # or build it from paper_id; either is fine
```

Every later use then sees the PDF URL: `requested_key`, the `_cached` lookup, the fetch,
`page.requested_url`/`final_url`/`canonical_url`, and the index writes. Specifics:

- **Step text:** for a resolved arXiv URL, emit
  `on_step(f"fetching the arXiv PDF for {paper_id}")` instead of `fetching arxiv.org`,
  so the live checklist shows that the rewrite happened.
- **Cache and index:** write only the resolved PDF key plus the canonical key, as for
  any URL. Do not add an index entry for the original `/abs/` URL. Lookups always
  resolve first, so such an entry would never be read. Existing index entries for
  previously fetched abstract pages are bypassed, and no migration is needed. As a
  bonus, the user's earlier `render https://arxiv.org/pdf/2602.16844` source is reused
  when they now pass the `/abs/` URL.
- **`--html FILE` keeps the URL verbatim.** The user supplied the bytes, and the
  documented contract is that `--html` "keeps the URL identity". This also serves as the
  escape hatch for anyone who really wants the abstract page narrated, so no new flag or
  config key is added.
- **No fallback to the abstract page.** If the PDF fetch fails (withdrawn paper, HTTP
  error, bot challenge), raise the existing `fetch_page` error, which names the PDF URL.
  Silently falling back would reintroduce the thin-script bug this change fixes.
- **Placement:** do not put the rewrite in `fetch_page` (generic; also used directly by
  live tests) or in `pipeline.load_source` (the cache key is computed inside `acquire`).
  Pipeline, CLI, `script`, manifest, and feed code need no changes. The render header
  still shows `arxiv.org` because it derives from the raw source string. Manifest
  `source.url` and feed show-note links become the PDF URL, the same as typing the
  `/pdf/` URL today.
- **No metadata schema change** in `source.json`.

Re-exporting from `web/__init__.py` is optional. `store.py` can import from
`sase_listen.web.arxiv` directly.

### 3. Tests

New file `tests/test_arxiv.py` with parametrized recognizer tables for `arxiv_paper_id`
and `arxiv_pdf_url`.

Positive cases, each mapping to `https://arxiv.org/pdf/<id>`:

- `https://arxiv.org/abs/2602.16844`
- `https://arxiv.org/abs/2602.16844v2` (version kept)
- `http://arxiv.org/abs/0704.0001` (4-digit new-style ID)
- `https://www.arxiv.org/abs/2602.16844/` (trailing slash)
- `https://export.arxiv.org/abs/2602.16844`
- `HTTPS://ArXiv.org/abs/2602.16844`
- `https://arxiv.org/abs/2602.16844?context=cs.AI#x`
- `https://arxiv.org/html/2602.16844v1/#S3`
- `https://arxiv.org/pdf/2602.16844.pdf`
- `https://arxiv.org/pdf/2602.16844v3`
- `https://arxiv.org/abs/hep-th/9901001`
- `https://arxiv.org/abs/math.GT/0309136v2`

Negative cases, each returning `None`:

- `https://arxiv.org/`
- `https://arxiv.org/list/cs.AI/recent`
- `https://arxiv.org/abs/`
- `https://arxiv.org/abs/not-an-id`
- `https://arxiv.org/src/2602.16844`
- `https://example.com/abs/2602.16844`
- `https://notarxiv.org/abs/2602.16844`
- `https://arxiv.org.evil.test/abs/2602.16844`
- `ftp://arxiv.org/abs/2602.16844`

Additions to `tests/test_pdf.py`, reusing `_paper_pdf()` and the `FetchedPage` stub
pattern of `_stub_pdf_fetch`:

- `test_store_arxiv_abs_url_fetches_pdf`: use a recording stub fetch and an `on_step`
  list.
  - Call
    `acquire("https://arxiv.org/abs/2608.25174?context=cs.SE", on_step=steps.append)`.
    Assert that the only fetched URL is `https://arxiv.org/pdf/2608.25174`, that
    `source_format == "pdf"`, and that `metadata["url"]` is the PDF URL.
  - Assert that `"fetching the arXiv PDF for 2608.25174"` is in `steps`.
  - Then `acquire("https://arxiv.org/pdf/2608.25174")` and
    `acquire("https://www.arxiv.org/abs/2608.25174")` must both be `reused` with the
    same `directory` and no extra fetch.
  - Finally, `index.json` must contain no `/abs/` key.
- `test_store_arxiv_abs_with_html_file_keeps_url`: monkeypatch `fetch_page` to raise
  `AssertionError`. Write `_paper_pdf()` to a temp file and call
  `acquire("https://arxiv.org/abs/2608.25174", html_file=saved)`. Assert there was no
  fetch and that `metadata["url"] == "https://arxiv.org/abs/2608.25174"`.
- `test_pipeline_arxiv_abs_url_matches_pdf_identity`: call
  `load_source("https://arxiv.org/abs/2608.25174", edition="verbatim")` with the stub
  fetch. Assert that `source_key == "url:https://arxiv.org/pdf/2608.25174#verbatim"` and
  that `source_url == "https://arxiv.org/pdf/2608.25174"`.
- Optional live test (`@pytest.mark.live`, skipped without `SASE_LISTEN_LIVE=1`, same
  pattern as `test_live_arxiv_pdf_extraction`): `fetch_page` on
  `arxiv_pdf_url("https://arxiv.org/abs/2608.25174")` returns `application/pdf`.

Existing tests that use `https://arxiv.org/pdf/2608.25174` must keep passing unchanged,
since that form maps to itself.

### 4. Docs and help text

- `docs/web-articles.md`: add an `### arXiv papers` subsection under "PDF documents". It
  should say:
  - which URLs are recognized: `/abs/`, `/html/`, and `/pdf/` (with or without `.pdf`)
    on `arxiv.org`, `www.arxiv.org`, or `export.arxiv.org`; old- or new-style IDs; any
    query or fragment ignored; the version suffix kept;
  - that each is fetched as `https://arxiv.org/pdf/<id>`, and that all forms share one
    cached source, writer script, and episode;
  - that `--html FILE` keeps the URL as given;
  - that a failed PDF fetch is reported and does not fall back to the abstract page.

  Also switch the section's example to
  `sase-listen render https://arxiv.org/abs/2608.25174 -e full`.

- `docs/cli.md`, in the `render` `SOURCE` paragraph: add one sentence saying arXiv paper
  URLs (for example `/abs/…`) are fetched as the paper's PDF, linking to web-articles.
- `docs/architecture.md`, in the "Web articles" paragraph: one clause naming `arxiv.py`,
  which resolves arXiv paper URLs to their canonical PDF URL before `store.py` keys and
  fetches them.
- `src/sase_listen/cli/render.py` epilog: change the arXiv example to
  `https://arxiv.org/abs/2608.25174 -e full`. No test asserts the epilog text.
- Do **not** edit `CHANGELOG.md` (managed by release-please), `pyproject.toml`,
  `uv.lock`, or the repo's `AGENTS.md`/`CLAUDE.md`.

## Verification

1. In the sase-listen checkout, run `sase tool run check` (the guarded `just check`:
   ruff, format check, mypy `--strict`, codespell, pytest). Do not run bare
   `just check`.
2. Optional smoke test (network only, no Gemini usage):
   `sase-listen script https://arxiv.org/abs/2602.16844 -e verbatim --json`. It should
   report roughly 11k words, a PDF `source_dir`, and the
   `fetching the arXiv PDF for 2602.16844` step. If that paper's PDF was rendered
   before, it is a cache hit on the same source directory.

## Non-goals

- Pointing feed show-note or manifest links at the `/abs/` page instead of the PDF URL.
- Resolving arXiv mirrors and aggregators (alphaXiv, Hugging Face Papers, Semantic
  Scholar) or DOI links.
- Rewriting other sites' landing pages to PDFs. The new module is arXiv-specific by
  design.
- Cleaning up stale cached abstract-page sources or library episodes.

Commit in the sase-listen repo with a conventional message such as
`feat(web): fetch arXiv paper URLs as their PDF`.
