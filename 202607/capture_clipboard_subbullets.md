---
tier: tale
goal: "bob capture supports a trailing %[header] marker (and matching --clip/--no-clip
  options) that captures the system clipboard as a formatted sub-bullet of the captured
  task or bullet, attaching file paths and long text as vault files under img/ and file/
  with reliable, atomic, beautifully rendered output.

  "
create_time: 2026-09-09 19:53:04
status: wip
---

# Plan: Clipboard sub-bullet capture for `bob capture`

## Product context

`bob capture` (src/native/capture.rs) captures one task or bullet line into the Bob
Obsidian vault. It already supports a small terminal-token grammar at the end of the
input: `@route` / `@route#Section` / `@route:block-id` markers and a `s:<N>` schedule
token, plus `--route`/`--section` forced modes, `--dry-run`, and `--format human|json`.
Insertion machinery places the rendered line into a Tasks section, a matching non-Tasks
section, or the Pomodoro-linked two-file flow.

This plan adds clipboard capture: a trailing `%` marker tells capture to read the system
clipboard and record its contents as an indented sub-bullet of the newly captured line.
Small text pastes inline; file paths become vault attachments; bulky or structured text
is preserved verbatim in a new vault file and linked. The feature must feel native to
the existing grammar, fail loudly rather than write partial captures, and produce output
that looks great in Obsidian.

## Marker syntax and grammar

- A clip marker is a whitespace-delimited token `%` or `%<header>`, where `<header>`
  matches `[A-Za-z0-9_-]+`. Tokens starting with `%` whose remainder violates that
  charset stay literal text (same forgiveness rule as invalid `s:` tokens).
- The marker is recognized only in the terminal token region, symmetric with the
  existing `s:<N>` contract: it may be the final token, or sit among the trailing
  special tokens just before a final route-like token (`@route`, `@route#...`,
  `@route:block-id`, `@!route:block-id`). Examples that all work: `body %`,
  `body % @notes`, `body s:1 % @groceries`, `body % s:1 @groceries`,
  `body %log @dev:blockid`.
- Implement by generalizing `extract_trailing_schedule` into one terminal- marker
  extraction pass: note an optional final route-like token, then strip schedule and clip
  tokens from the tail (each at most once, any order), stopping at the first non-marker
  token. Existing behavior must be preserved exactly (e.g. `Buy Milk s:1 s:2` still
  keeps `s:1` literal; mid-text `%` and `@` tokens stay literal; `50%` or `100%` endings
  are never markers because they do not start with `%`).
- A bare `%` defaults the header to `clip`. Header rendering: uppercase the header,
  replace `_` with a space, and wrap as `**<HEADER>:** `. So `%` → `**CLIP:** ` and
  `%foo_bar_baz` → `**FOO BAR BAZ:** `; hyphens are kept (`%foo-bar` → `**FOO-BAR:** `).
- The marker composes with every capture kind: plain task, forced `--route`/`--section`,
  bullet (`@notes#Ideas`), and Pomodoro (`@dev:block-id`) captures all gain the
  sub-bullet under the captured line in the routed note. Like `s:<N>`, the marker is
  still extracted when `--route`/`--section` are used.
- Known accepted quirk (document it): a genuine trailing token like `%20` parses as a
  marker with header `20`. `--no-clip` is the escape hatch.

## New CLI options

Follow the repo CLI rules: options listed alphabetically, every public long option has a
short alias, help output is excellent, and human output is colored via the existing
`Styler`.

- `-c, --clip[=HEADER]` — force clipboard capture without a `%` marker. Uses
  `require_equals` semantics with a `clip` default-missing value so it never swallows
  the following TEXT argument. HEADER shares the marker header charset; invalid values
  are usage errors. Mirroring how `--route` keeps `@tokens` literal, `--clip` keeps `%`
  tokens in the text literal.
- `-n, --no-clip` — disable clip-marker parsing so a trailing `%...` token stays
  literal. Conflicts with `--clip` (usage error when both are given).

Argument registration in `build_cli` stays alphabetical: bob-dir, clip, dry-run, format,
help, no-clip, route, section, text.

## Reading the clipboard

New module `src/native/capture_clip.rs` (wired via `mod capture_clip;` in src/native.rs)
owns clipboard access, classification, rendering, and file saving, keeping capture.rs
focused on grammar and insertion.

Resolution order for the clipboard source (first available wins):

1. `BOB_CLIPBOARD_CMD` env var — whitespace-split argv, run, stdout captured. This is
   both the power-user override and the deterministic test hook.
2. macOS builds: `pbpaste`.
3. Linux: if `WAYLAND_DISPLAY` is set, `wl-paste --no-newline --type text`; else if
   `DISPLAY` is set, `xclip -selection clipboard -o`, falling back to
   `xsel --clipboard --output` when xclip is missing.
4. If `TMUX` is set (e.g. SSH sessions with no display), `tmux show-buffer`.
5. Otherwise fail with a clear error that names the tools tried and suggests setting
   `BOB_CLIPBOARD_CMD`.

Post-processing: reject non-UTF-8 output or embedded NUL bytes with an error telling the
user to copy a file path when they want to attach binary content. Normalize CRLF/CR to
LF. Trim trailing newlines/whitespace-only tail lines. Empty or whitespace-only
clipboard is an error ("clipboard is empty"), never a silent no-op — the user asked for
a clip, so a missing clip must fail loudly before any file is written.

## Classifying clipboard contents

Path candidate detection (per line): strip one pair of matching surrounding quotes;
convert `file://` URIs (strip scheme/host, percent-decode — this is what file managers
put on the clipboard); expand a leading `~/`. A line is a path candidate only when the
result is absolute (starts with `/`) — never treat arbitrary words as relative paths. A
candidate that exists as a regular file is an attachment source; an existing directory
is not attachable and falls through to the text rules.

Decision tree over the normalized text:

1. **Single line**
   - attachment path → _attachment mode_ (one file).
   - longer than 1000 characters → _snippet mode_ (a pasted wall of minified text should
     not wreck the note line).
   - otherwise → _inline mode_: `  - **CLIP:** <text>`.
2. **Multiple lines**
   - every non-empty line is an attachment path and there are at most 10 → _attachment
     mode_ (multi-file copy from a file manager); more than 10 is an error naming the
     count.
   - any interior blank line, or more than 10 lines → _snippet mode_.
   - any line is "structural" — after optional leading spaces it starts with a list
     marker (`- `, `* `, `+ `, `1. `-style), `#`, `> `, or a ``` / ~~~ fence — or any
     line starts with indentation → _snippet mode_, because re-bulleting such content
     would mangle it; verbatim file preservation is the smart default.
   - otherwise (2–10 flat plain lines) → _lines mode_: a `  - **CLIP:**` header
     sub-bullet with each clipboard line as a nested child (`    - <line>`).

## Attachment and snippet files

- Image detection is by lowercased extension: avif, bmp, gif, heic, ico, jpeg, jpg, png,
  svg, tif, tiff, webp.
- Images copy into `<bob-dir>/img/`, everything else into `<bob-dir>/file/`
  (`create_dir_all` on demand; `--bob-dir`/`BOB_DIR` respected — the literal `~/bob/img`
  and `~/bob/file` paths are just the defaults).
- File names are sanitized for Obsidian link safety: control characters and
  `[ ] # ^ | : \` become `-`, runs collapse, leading/trailing dots and spaces are
  trimmed; Unicode letters are kept.
- Collisions: if the destination name exists with identical content (sha256 — already a
  dependency), reuse it without copying and mark it `reused`. If the content differs,
  fall back to `<stem>-<first 8 hex of sha256>.<ext>`; a further mismatch is an error.
  Copies go through a temp file + rename in the destination directory.
- References: images render as `![[img/<name>|400]]` (400px embed width as a single
  named constant — "appropriately sized"); other files as `[[file/<name>]]`.
- Snippet files are written verbatim (single trailing newline ensured) to
  `<bob-dir>/file/clip-<YYYYMMDD>-<HHMMSS>[-<slug>].md`, where the timestamp comes from
  `bob_env::current_datetime()` (so `BOB_NOW` keeps tests deterministic) and the slug is
  up to 40 chars of lowercased alphanumeric runs joined by `-` from the first non-blank
  line; omit the slug segment when it comes up empty. Same-second collisions append
  `-2`, `-3`, …. The reference is `[[file/clip-...]]` (no `.md`).

## Rendering and insertion

- The capture becomes a multi-line block: the existing parent line (task/bullet/pomodoro
  rendering is unchanged, including the `^block-id` final token) followed by child lines
  indented two spaces, with nested children at four spaces:
  - inline / single attachment / snippet: `  - **CLIP:** <reference-or-text>`
  - lines / multi-attachment: `  - **CLIP:**` followed by `    - <line>` or
    `    - <reference>` children.
- Generalize the insertion helpers (`insert_task_line`, `insert_bullet_line`, and the
  Pomodoro flow's target-side insertion) to insert a block of lines instead of a single
  line. The existing `task_block_end` logic already treats indented lines as part of a
  task block, so subsequent captures continue to insert after the whole block.

## Ordering, atomicity, and dry-run

Pipeline order: read clipboard → classify → validate sources and compute all destination
names → fully plan and validate the note edit(s), including the existing Pomodoro
two-file validation → only then write attachment/snippet files → write the note(s). If a
note write fails after new files were created, best-effort remove only the files this
run created (never a reused attachment), and say so in the error. `--dry-run` performs
everything up to and including planning but creates no directories and writes no files,
while reporting exactly what it would save and where.

## Output

- JSON: add an optional `clip` object (omitted when no clip capture ran) with stable
  fields: `header` (rendered, e.g. `"CLIP"`), `mode`
  (`"inline" | "lines" | "attachments" | "snippet"`), `lines` (the rendered child lines
  exactly as written), `attachments` (array of `{source, saved, kind, reused}` with
  vault-relative `saved` paths and `kind` `"image" | "file"`), and `snippet`
  (vault-relative path, when mode is snippet). `task_line` remains the parent line only.
  Failures keep the existing `{ "ok": false, "error": ... }` shape.
- Human: after the existing dimmed task line, print each dimmed child line; for every
  file written, print an aligned confirmation row in the existing style
  (`✓ saved     img/<name>` in cyan, `would save` under `--dry-run`, `reused` noted when
  applicable).

## Documentation

- Update `bob capture` `long_about`, examples, and the Environment section (document
  `BOB_CLIPBOARD_CMD`, the resolution order, and the `%` grammar) — help output must
  stay clear and scannable.
- Extend the `bob capture` section of README.md with the marker grammar, the
  classification rules, the attachment/snippet directories and naming, the new options,
  the new JSON `clip` field, and the `%20`-style literal quirk with its `--no-clip`
  escape hatch.

## Testing

Unit tests (capture.rs / capture_clip.rs):

- Marker extraction: every terminal-position combination with `s:<N>`, plain routes,
  bullet routes, and Pomodoro markers; leading-route interaction; mid-text `%` stays
  literal; invalid header charset stays literal; bare `%` default; regression coverage
  that existing schedule/route tests still pass unchanged.
- Header formatting: default, underscores, hyphens, digits, case handling.
- Classification: inline vs lines vs snippet vs attachment decisions, including quoted
  paths, `file://` percent-decoding, `~` expansion, directories falling through to text,
  blank-line and >10-line snippets, structural/indented lines forcing snippet, the
  1000-char single-line rule, and the multi-attachment cap.
- Naming: sanitization, identical-content reuse, differing-content hash suffix, snippet
  slug and same-second collision counters.
- Rendering: exact child-line output for each mode, including under Pomodoro captures
  (block ID stays the parent line's final token).

Integration tests (tests/cli.rs), using `BOB_CLIPBOARD_CMD` pointed at small fixture
scripts and `BOB_NOW` for determinism:

- End-to-end `%` capture into a temp vault: parent + sub-bullet placement in Tasks
  sections, bullet sections, and the Pomodoro two-file flow.
- Image and non-image attachment copy (correct directory, embed vs link, reuse on
  identical re-capture), snippet file creation and contents.
- `--clip`, `--clip=HEADER`, `--no-clip` (literal `%`), conflict error.
- JSON shape with the `clip` object; human output rows; `--dry-run` writes nothing (no
  img/, file/, or note changes) while reporting planned paths.
- Failure modes: empty clipboard, binary clipboard, missing attachment source, too many
  attachments — each leaves the vault untouched.

`just all` (fmt --check, clippy, test) must pass.

## Risks and edge cases already accounted for

- Clipboard tooling varies by host (macOS pbpaste, Wayland, X11, tmux-only SSH); the
  resolution order plus `BOB_CLIPBOARD_CMD` covers all of them and the error message
  teaches the fix.
- Binary clipboards (copied image data rather than a path) are rejected with guidance
  instead of corrupting a note.
- Multi-file copies from file managers arrive as multiple `file://` lines and are
  handled as multi-attachments rather than garbage snippets.
- Markdown-structured clipboard text is never re-bulleted; it goes to a verbatim snippet
  file.
- No partial writes: every failure path happens before the first byte lands in the
  vault, or rolls back the files this run created.

## Non-goals

- Reading raw image data from the clipboard (copy the file path instead).
- OSC52 or Windows clipboard support.
- Configurable image embed width (single constant for now).
