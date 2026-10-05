---
tier: tale
title: "sase-listen: render PDF sources (URLs and local files)"
goal:
  "`sase-listen render https://arxiv.org/pdf/2608.25174 -e full` extracts the paper
  locally, writes and narrates a full edition, and publishes a new episode to the
  private podcast feed the user follows in AntennaPod; local .pdf files work as sources
  too."
size: medium
proposed_by: bbugyi200.athena.0wx.w0
create_time: 2026-10-05 12:13:23
status: wip
---

# Plan: sase-listen PDF sources (URLs and local files)

## Goal

Make this command work end to end and publish a new episode to the private podcast feed
the user subscribes to in AntennaPod:

```
sase-listen render https://arxiv.org/pdf/2608.25174 -e full
```

Today it fails with `PDF sources are not supported yet.` (raised in
`src/sase_listen/web/fetch.py` when the response is `application/pdf`). After this
change, a PDF works as a source everywhere an article URL does (`render`, `script`, all
three editions `brief` / `full` / `verbatim`, caching, `--refresh`, auto-publish), and a
local `*.pdf` file is a valid source too.

All work happens in the linked **sase-listen** repo. Open it with
`sase repo open sase-listen -r "Implement PDF source support"`, work only in the printed
path, and read its `AGENTS.md` first. Do not import `sase` there (the no-sase-import
rule). This tale intentionally edits `pyproject.toml` and `uv.lock` to add one
dependency. The AGENTS.md "do not edit pyproject/uv.lock" rule applies to the original
parallel scaffold phases, not to this work.

## Findings that shape the design

- URL sources flow `fetch_page` (`web/fetch.py`) → `extract_article` (`web/extract.py`,
  trafilatura) → `acquire` (`web/store.py`, writes `page.html`, `source.md`,
  `source.json` under `$XDG_DATA_HOME/sase-listen/sources/<slug>-<hash>/` plus
  `index.json`) → `load_source` (`pipeline.py`). `brief`/`full` call the Gemini writer
  (`writer/author.py`, `writer/prompt.py`); `verbatim` normalizes `source.md`.
  Everything downstream consumes only `source.md` and `source.json`, so a PDF just has
  to produce the same shape (an `Article`) and store it the same way.
- The target is a 9-page, two-column ACM/arXiv LaTeX paper (5 MB,
  `content-type: application/pdf`, no redirect). It has good Info metadata (Title;
  Author `James C. Davis; Kelechi Kalu; Huiyun Peng; Parth V. Patil`; CreationDate) and
  a 26-entry bookmark outline (`Abstract`, `1 Introduction`, `2.1 …`, …, `References`).
  The arXiv sidebar stamp reads `arXiv:2608.25174v1 [cs.SE] 25 Aug 2026`. Running
  headers alternate between `Davis et al.` and the title.
- Prototyped with **pdfminer.six** (MIT, ships `py.typed`, pure Python). Its only
  transitive dependencies (`cryptography`, `charset-normalizer`, `cffi`, `pycparser`)
  are already in `uv.lock`. Layout analysis took about 0.5 s for the paper and returned
  correct two-column reading order as paragraph-level text boxes, with per-character
  font size and font name. Body text is 9 pt, section headings 10.9 pt bold, captions 8
  pt bold, and table cells, footnotes, and references 6–7 pt. A crude prototype
  (outline-matched headings, small-text drop, running-header drop, references cut) gave
  about 5,700 clean words with all 25 non-reference headings restored.
  - Rejected: PyMuPDF/pymupdf4llm (AGPL, incompatible with this MIT package);
    docling/marker (torch-sized); pypdfium2 (good order, but no paragraph boxes or easy
    font sizes); Gemini PDF→Markdown transcription (nondeterministic, costs tokens even
    for `verbatim`, output-token caps on long PDFs, and it breaks the "extracted
    locally" promise in `docs/web-articles.md`).
- `kind: article` drives the intro phrase (`by {author} at {site}, published {date}`),
  the `(Brief)/(Full)/(Reading)` display title, the coverage sentence, and auto-publish
  (`meta.kind in {"research", "article"}` in `pipeline.render`). PDF sources therefore
  keep `kind: article`. No new kind and no manifest schema change: the feed host
  (apollo) runs its own sase-listen install and must keep accepting what this machine
  publishes.
- The normalizer already turns `##` into chapters, speaks `###` as a lead-in sentence,
  and omits an h2 `References`/sources section. Lint does not require `source:` for
  articles.
- The globally installed `sase-listen` is an **editable `uv tool` install of a different
  checkout** (`~/projects/github/sase-org/sase-listen`). It will not see this change, or
  the new dependency, until that checkout pulls and the tool is reinstalled.
  Verification therefore runs the command through the opened checkout's own environment
  (see Verification).

## Implementation

### 1. Dependency

Add `"pdfminer.six>=20250506"` to `[project].dependencies` (keep the list's style), run
`uv lock` (expect only `pdfminer-six` to be added), then `just install`. pdfminer.six is
typed, so add no mypy override unless `mypy --strict` proves one necessary. If it does,
keep the override narrowly scoped.

### 2. Fetch: accept PDFs (`web/fetch.py`)

- Add `PDF_CONTENT_TYPE = "application/pdf"`, `MAX_PDF_BYTES = 64 * 1024 * 1024`, and a
  helper `is_pdf_bytes(data: bytes) -> bool` (`b"%PDF-"` within the first 1024 bytes,
  per the PDF spec).
- `fetch_page(..., max_pdf_bytes: int = MAX_PDF_BYTES)`: choose the size limit from the
  response content type before streaming. PDF and octet-stream types use
  `max_pdf_bytes`; everything else keeps `max_bytes`. After the challenge and status
  checks:
  - `application/pdf` or `application/x-pdf` with a PDF body → accept as
    `content_type="application/pdf"`. A declared PDF whose body is not a PDF →
    `SaseListenError` (UNEXPECTED) naming the mismatch.
  - `application/octet-stream`, `binary/octet-stream`, or a missing type with a PDF body
    → accept as PDF.
  - HTML types are unchanged. Any other type is still rejected; update the hint to "Only
    HTML pages and PDF documents are supported."
  - Delete the "PDF sources are not supported yet" error.
- `load_html_file(url, path)` (the `--html FILE` path): if the file's bytes are a PDF,
  return it with `content_type="application/pdf"` (size cap `MAX_PDF_BYTES`).
  `render URL --html saved.pdf` then keeps URL identity for a PDF the user had to
  download by hand. Update the docstring and `--html` help text to "saved browser page
  (HTML or PDF)".
- Update the `FetchedPage` docstring: an accepted HTML page or PDF document.

### 3. PDF → Markdown extraction (new `web/pdf.py`)

Public API:

```python
MAX_PDF_PAGES = 400

@dataclass(frozen=True)
class PdfExtraction:
    article: Article          # reuse web.extract.Article
    pages: int
    authors: list[str]
    outline_source: str       # "bookmarks" | "fonts" | "none"
    dropped_small_words: int  # footnotes / table cells / figure labels removed

def extract_pdf(data: bytes, *, source_url: str = "", filename: str = "") -> PdfExtraction
```

Import pdfminer lazily inside the function, the same way `extract_article` imports
trafilatura. Algorithm:

1. **Open**: `PDFParser(io.BytesIO(data))`, then `PDFDocument`. Map
   `PDFPasswordIncorrect`/encryption errors to USAGE "The PDF is password-protected."
   Map `PDFSyntaxError`/`PSException`/other pdfminer failures to USAGE "Could not read
   the PDF: …". More than `MAX_PDF_PAGES` pages → USAGE error naming the count and the
   limit.
2. **Metadata**: read the Info dict, resolving indirect objects (`resolve1`) and
   decoding PDFDocEncoding / UTF-16 values (`pdfminer.utils.decode_text`).
3. **Outline**: `doc.get_outlines()` (catch `PDFNoOutlines`). Normalize levels so the
   top level is 1.
4. **Lines**: run `extract_pages(..., laparams=LAParams())` and collect every
   `LTTextLine` with its page, text-box index, bbox, dominant char size (rounded to
   0.1), and a bold flag. A font is bold when its name contains `Bold`, `Black`,
   `Heavy`, `Semibold`, `Demi`, or `CMBX`, or ends in `TB`/`-B`. Build each line's text
   from its chars so you can drop superscript footnote markers: digits and `*†‡§` whose
   size is ≤ 0.8 × the line's dominant size. Body size is the char-weighted mode of line
   sizes.
5. **Filter**:
   - Running headers and footers: lines in the top or bottom 8% of the page whose
     digit-stripped, lowercased alphanumeric text recurs on ≥ 2 pages. Also drop
     standalone page numbers (`^\d{1,4}$`, `^Page \d+( of \d+)?$`) in those bands.
   - The arXiv stamp
     (`^arXiv:\d{4}\.\d{4,5}(v\d+)?\s*\[[^\]]+\]\s+\d{1,2}\s+\w{3}\s+\d{4}$`). Parse its
     date first.
   - Small text: lines with size < 0.85 × body (footnotes, table cells, figure labels,
     small reference lists). Count their words in `dropped_small_words`.
   - Front-matter boilerplate paragraphs starting with `CCS Concepts`, `Keywords`,
     `Index Terms`, `ACM Reference Format`, `Permission to make digital or hard copies`,
     `©`, or `Copyright`.
   - Page-1 front matter (author names, emails, affiliations): everything on page 1
     before the first heading, except the title lines. Skip this rule when page 1 has no
     heading.
6. **Headings**:
   - With an outline (`outline_source="bookmarks"`): walk lines in order and match the
     next unconsumed outline entries (look ahead about 3 entries so entries absent from
     the text do not stall matching). Normalize both sides to lowercase alphanumerics.
     Also accept a heading wrapped across 2 consecutive lines of one box, and a line
     that starts with the heading text (split the remainder off as paragraph text). Emit
     outline level 1 as `##` and deeper levels as `###`. Use the page text (it keeps
     symbols that bookmarks drop, e.g. `→`), with leading numbering stripped (`1`,
     `2.1`, `A`, `A.1`, `IV.`) when a letter follows. Matched entries become
     `restored_headings`; unmatched entries become `missing_headings`; `outline` is all
     number-stripped entries.
   - Without an outline (`"fonts"`): a heading is a non-title line with size ≥ 1.15 ×
     body, or a short bold body-size line (≤ 12 words, no terminal period) matching a
     numbered pattern (`^(\d+(\.\d+)*|[IVX]+\.|[A-Z]\.)\s+[A-Z]`) or named exactly
     `Abstract`, `Introduction`, `Conclusion(s)`, `Discussion`, `References`,
     `Bibliography`, `Acknowledg(e)ments`, or `Appendix…`. The largest heading size maps
     to `##` and smaller sizes or dotted numbering (`2.1`) to `###`. Use `"none"` when
     nothing qualifies.
   - Drop the `References` / `Bibliography` / `Works Cited` / `Literature Cited`
     section: from that heading to the next heading at the same or a higher level (so a
     later appendix survives), or to the end.
7. **Paragraphs**: lines in one text box form one paragraph. Join a line ending in `-`
   (or U+00AD / U+FFFE) to a next line starting lowercase with no hyphen; otherwise join
   with a space. This is a known trade-off: a real compound broken at a line end loses
   its hyphen. Merge a box into the previous paragraph when that paragraph lacks
   terminal punctuation (`.?!:;"”’)]`) and the box starts lowercase (column and page
   breaks). Split bullet starts (`•◦▪‣` or a dash plus space) into `- ` list items. Keep
   `Figure N:` / `Table N:` captions as their own paragraphs.
8. **Inline cleanup**: NFKC (ligatures), strip numeric citation brackets
   (`\s?\[\d+(?:\s*[-–,]\s*\d+)*\]`), remove `(cid:NN)` tokens (count them), and
   collapse whitespace.
9. **Title**: use the Info Title if plausible: non-empty, ≥ 4 chars, not `untitled`, no
   `Microsoft Word - ` prefix, not ending in `.pdf/.doc/.docx/.tex/.dvi`. Otherwise use
   the largest-size consecutive line(s) on page 1, else the humanized filename stem or
   last URL path segment. Drop a leading body line equal to the title, as
   `extract_article` does.
10. **Authors / credit**: split Info Author on `;` or `and` (or on commas when a single
    remaining entry has ≥ 2 commas). Store the list in `authors`. Set `Article.author`
    to a speakable credit: 1 → `A`; 2 → `A and B`; 3 → `A, B, and C`; ≥ 4 →
    `A and colleagues`. The target paper should read `James C. Davis and colleagues`.
11. **Date / site**: date is the arXiv-stamp date, else Info `CreationDate`
    (`D:YYYYMMDD…`), as ISO `YYYY-MM-DD`, else empty. Site is `arXiv` when the stamp, an
    Info `arXivID` key, or an `arxiv.org` URL host is present, else empty. Description
    is empty.
12. **Quality gates**: fewer than 150 words → `SaseListenError` (UNEXPECTED) "The
    extracted PDF text is too short (N words)." with hint "The PDF may be a scanned
    image without a text layer; OCR is not supported. Convert it to Markdown and render
    that file." `(cid:` tokens > 2% of words → the same shape, saying the fonts could
    not be decoded.
13. `Article.markdown` is the body with no H1 (the store adds `# {title}`), blank-line
    separated. `canonical_url` is `normalize_url(source_url)` when given, else `""`.
    `words` is the body word count.

### 4. Storage (`web/store.py`)

- `AcquiredSource.page_path` becomes "the retained original": `page.html` for HTML and
  `source.pdf` for PDF. Add a `source_format` property reading
  `metadata.get("format", "html")`, so entries written before this change still load.
  `_cached` derives the original's filename from that format.
- `acquire(url, html_file=None, refresh=False)`: after fetching or loading, when
  `page.content_type == "application/pdf"`, call
  `extract_pdf(page.body, source_url=page.final_url)` instead of `extract_article`.
  Canonical is `normalize_url(page.final_url)`; index both the requested and canonical
  keys (as today). Write `source.pdf`, `source.md`, and `source.json`.
- PDF `source.json` reuses the HTML keys (`url`, `canonical_url`, `final_url`, `title`,
  `author`, `site`, `date`, `description`, `fetched_at`, `http_status`, `source_sha256`,
  `words`, `extractor`, `outline`) and adds `"format": "pdf"`, `"pdf_sha256"` (in place
  of `html_sha256`), `"pages"`, `"authors"`, `outline.source`, `"dropped_small_words"`,
  and `"extractor": {"name": "pdfminer.six", "version": …}`. HTML entries gain
  `"format": "html"`.
- New `acquire_file(path, *, refresh=False) -> AcquiredSource` for local PDFs: read
  bytes (cap `MAX_PDF_BYTES`, require PDF magic, clear USAGE errors for missing or
  unreadable files), take the sha256, and use index key `pdf:sha256:<hex>`. Reuse the
  cache unless `refresh`. Extract with `filename=path.name`; directory name
  `_source_name(title, key)`. Metadata has `url`/`canonical_url`/`final_url` empty,
  `"file"` set to the resolved path, and `fetched_at` set to now.

### 5. Pipeline (`pipeline.py`)

- Add `looks_like_pdf_file(source) -> bool`: an existing regular file whose suffix is
  `.pdf` (case-insensitive) or whose first bytes are PDF magic.
- Refactor the URL branch of `load_source` into a shared helper, e.g.
  `_load_acquired(acquired, *, edition, refresh, config, source_label, source_key_base, source_url)`,
  holding the edition default (`brief`), the edition validation, the writer cache /
  `author_script` call, and the verbatim `_article_script` path. Both branches call it:
  - URL: unchanged behavior and keys (`url:{canonical}#{edition}`).
  - PDF file (checked right after the URL branch and before the brief/full guard, the
    ref check, and plain-path handling): `acquire_file`; then
    `source_label=str(resolved_path)`, `source_key=f"pdf:{pdf_sha256}#{edition}"`,
    `source_url=""`, and `source_meta=metadata`. Episode ids stay stable across renders
    of the same file.
- Update the brief/full guard message to "…available for article URLs and PDF files."
  and the "Source not found" hint to list PDF files.
- No manifest changes. Local PDF renders use the existing `path` payload branch; URL
  PDFs use the existing `url` branch.

### 6. Writer prompt (`writer/prompt.py`, `writer/author.py`)

- `system_prompt(edition, source_format="html")`: for `"pdf"`, append a
  `PDF source rules:` block. It says the text was extracted from a PDF, often an
  academic paper (say "the paper" and "the authors" when it is one). Ignore layout
  residue (stray figure labels, table fragments, repeated captions, broken hyphenation).
  Convey figures and tables only through captions and the surrounding prose. Never read
  citation numbers, emails, affiliations, or copyright notices.
- `user_prompt`: for PDFs only, add a `Format: PDF document (N pages)` line. Label the
  outline "Document outline (headings in source order)" and use `outline.restored`.
- The HTML system and user prompts must stay byte-identical, so do **not** bump
  `WRITER_PROMPT_VERSION`; existing article writer caches stay valid. PDF sources start
  with fresh caches. `author_script` passes `article.metadata.get("format", "html")`
  through.

### 7. CLI help (`cli/render.py`, `cli/script_cmd.py`)

- `render`: source help
  `Narration script, Markdown file, PDF file, kind:path ref, or http(s) URL`; `-e` help
  `Edition for URL and PDF sources (default: brief).`; `--html` help per step 2;
  `--refresh` help `Fetch or extract the source again and replace the cached copy.`; add
  a PDF example to the epilog.
- `script`: route `looks_like_url(source) or looks_like_pdf_file(source)` into the
  `load_source` branch (JSON output unchanged, including `source_dir`, `outline`,
  `writer`). Update the brief/full error text and the source help to mention PDF files.

### 8. Docs

- `docs/web-articles.md`: replace the "PDFs … are not extracted" sentence. Add a
  `## PDF documents` section covering PDF URLs (by content type or `%PDF` bytes), local
  `.pdf` files, `--html saved.pdf`, and local extraction with pdfminer.six. Say how
  headings come from bookmarks or font sizes. List what is dropped (running headers and
  footers, page numbers, footnotes and other small text, table cells, citation brackets,
  page-1 author block, references) and the metadata (title, author credit, arXiv date
  and site). State the limits (64 MiB, 400 pages, no OCR, so scanned PDFs are rejected),
  that editions and caching match articles, and that `script … --json` shows outline
  diagnostics. Keep the mkdocs nav unchanged.
- `docs/cli.md`: mention PDF sources for `render` and `script`.
- `docs/architecture.md` "Web articles" section and the `AGENTS.md` architecture map:
  mention `pdf.py`.
- `README.md`: mention PDF sources wherever URL sources are described.
- `docs/troubleshooting.md`: add an entry for "extracted PDF text is too short" (scanned
  PDFs, OCR, convert to Markdown).
- Do not hand-edit `CHANGELOG.md` (release-please owns it).

### 9. Tests

- **PDF builder helper** (e.g. `tests/pdf_builder.py`): emit minimal valid PDFs in pure
  Python. Use base-14 `Helvetica` / `Helvetica-Bold` text at chosen sizes and positions,
  support multiple pages and two-column placement, plus an optional Info dict (Title,
  Author, CreationDate, arXivID), an optional outline (with nesting), and a correct xref
  table. No binary fixtures are committed.
- **`tests/test_pdf.py`**:
  1. A two-column, multi-page document with an outline, a running header plus page
     numbers, a small-font footnote, a superscript marker, an author/email block, a
     `CCS Concepts` paragraph, a `References` section followed by an appendix, and an
     arXiv stamp line. Assert: number-stripped `##`/`###` headings in order; the left
     column precedes the right; header, page numbers, footnote, superscript, author
     block, CCS paragraph, stamp, and reference entries are absent; the appendix
     survives; restored/missing lists; `outline_source == "bookmarks"`; site `arXiv` and
     the date from the stamp.
  2. No outline → font-size and numbered-bold fallback headings
     (`outline_source == "fonts"`).
  3. Dehyphenation, cross-column paragraph continuation, citation-bracket stripping, and
     bullet splitting.
  4. Metadata: 1/2/3/≥4-author credit strings; `CreationDate` → ISO date; an implausible
     Info title (`Microsoft Word - draft.docx`) falls back to the largest first-page
     line.
  5. Errors: non-PDF bytes; a password error (monkeypatch pdfminer to raise); page cap
     (monkeypatch `MAX_PDF_PAGES`); too-short or text-less PDF (OCR hint).
- **Fetch** (`tests/test_web.py`): rework `test_fetch_detects_challenge_and_pdf`.
  `application/pdf` with a PDF body is now accepted; octet-stream plus `%PDF` is
  accepted as PDF; a declared PDF with a non-PDF body errors; other types are still
  rejected; the PDF size limit is separate from the HTML one; `load_html_file` with PDF
  bytes yields a PDF page.
- **Store**: acquiring a stubbed PDF URL writes `source.pdf` and
  `format/pdf_sha256/pages/extractor` metadata; a second acquire reuses the cache (no
  fetch); `refresh` refetches; a pre-existing HTML entry without `format` still loads;
  `acquire_file` caches by content hash.
- **Pipeline / CLI**: `load_source(pdf_url, edition="verbatim")` yields `kind: article`
  with chapters from the headings. `load_source(local.pdf, edition="verbatim")` has a
  stable `pdf:<sha>#verbatim` key across calls. `brief` with a stub writer (follow the
  existing writer-test pattern) puts the PDF rules in the system prompt, while
  `system_prompt("full")` for HTML does not mention PDF and equals its pre-change text.
  `main(["script", pdf, "-e", "verbatim", "--json"])` succeeds. A `render` of a stubbed
  PDF URL with the tone narrator and `--no-publish` produces an `(Reading)` episode.
- **Live test** (`@pytest.mark.live`, skipped without `SASE_LISTEN_LIVE=1`): extract
  `https://arxiv.org/pdf/2608.25174` and assert the title, `site == "arXiv"`, date
  `2026-08-25`, ≥ 20 restored headings, no `Davis et al.`, no `@purdue.edu`, no
  `[12, 22]`, no References section, and 4,000–8,000 words.

## Verification (required; the user asked for it explicitly)

1. In the sase-listen checkout, run `sase tool run check` (lint + mypy --strict +
   codespell + pytest). It must pass. Also run
   `SASE_LISTEN_LIVE=1 .venv/bin/pytest -m live -k pdf`.
2. Sanity-check the real extraction (free, no API calls):
   `uv run sase-listen script https://arxiv.org/pdf/2608.25174 -e verbatim --json`. Then
   read the stored `source.md` in the reported `source_dir`. Expect the title
   "Model-Based Agentic Software Engineering", author "James C. Davis and colleagues",
   site arXiv, date 2026-08-25, about 26 outline entries with about 25 restored, and no
   running headers, emails, citation brackets, table-cell fragments, or reference list.
   Fix heuristics and re-run with `--refresh` until it reads cleanly.
3. **Run the user's command** through the checkout's environment, in the foreground with
   a generous timeout (or hand it to `/sase_monitor`). The writer and TTS take several
   minutes: `uv run sase-listen render https://arxiv.org/pdf/2608.25174 -e full`. This
   spends Gemini writer tokens and TTS on a roughly 16-minute episode; the user
   requested this run. It must finish with
   `Done: Model-Based Agentic Software Engineering (Full)` and `Published to feed.` If
   the publish was queued (SSH to the feed host failed), run
   `uv run sase-listen publish --pending` and investigate until it publishes. Do **not**
   use the globally installed `sase-listen` for this: it is an editable install of a
   different checkout without the new code or dependency. Do **not** reinstall or
   repoint the user's `uv tool` install at this checkout.
4. **Confirm the feed AntennaPod polls has the new episode**:
   - `uv run sase-listen feed --json` → `episode_ids` (fetched via the feed host)
     contains the new episode id; take it from the `MP3:` path the render printed.
   - Read the unmasked feed URL from `uv run sase-listen feed --show-url --json` (`url`)
     into a shell variable without echoing it. `curl -fsS` the feed XML and confirm an
     `<item>` titled "Model-Based Agentic Software Engineering (Full)" whose
     `<enclosure>` URL answers `curl -sI` with HTTP 200 and `audio/mpeg`. Never print,
     log, or commit the token-bearing URL.
   - Report the episode id, duration, chapter count, item title, and enclosure status
     (URLs masked).
5. In the final response, tell the user the installed CLI needs a refresh once the
   change lands on master: `git -C ~/projects/github/sase-org/sase-listen pull`, then
   `uv tool upgrade --reinstall sase-listen` to install the new pdfminer.six dependency
   into the tool environment.

## Out of scope

- OCR for scanned PDFs, math-to-speech, and table reading.
- Passing the PDF itself to Gemini as a multimodal attachment.
- arXiv abstract-page metadata lookups.
- A new script `kind`, or changes to manifest, feed, or receive-protocol.
- Re-rendering a local-PDF writer script directly by path gets a path-based episode id,
  not the `pdf:<sha>` one, because it has no `source` URL. This is an acceptable known
  limitation.
