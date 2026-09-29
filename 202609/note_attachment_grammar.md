---
tier: tale
size: medium
title: Core note-attachment grammar in sase-core
goal:
  sase-core scans bead-note text for @path, @@, and @attachment tokens, composes stored
  text and its editing inverse, sanitizes and uniquifies attachment names, classifies
  media from magic and extension, and accepts attachment:<bead-id>/<name> artifact refs,
  with PyO3 bindings, a property test, and a corpus golden.
proposed_by: bbugyi200.athena.sase-1ck.1
bead: sase-1ck.1
create_time: 2026-09-29 08:32:19
status: wip
---

- **PARENT:**
  [202609/bead_note_attachments.md](https://github.com/sase-org/sase--plans/blob/main/202609/bead_note_attachments.md)
- **BEAD:** sase-1ck.1

# Plan: Core note-attachment grammar (sase-1ck.1)

## Context

Phase `core_grammar` of epic `sase-1ck` (plan `plan:202609/bead_note_attachments.md`,
Phase 1 and the shared "Grammar" and "Names, media, identity" sections). Those sections
are the authority for behavior. This tale settles the algorithms and wire shapes they
leave open, so one agent can implement them in the linked `sase-core` checkout.

Work only in that checkout (`sase repo open sase-core` is already done for this
workspace; read its `AGENTS.md` before editing). The sase repo changes only if the pin
bump below is actually possible. Leave parent epic `sase-1ck` open. Close only
`sase-1ck.1`. Record follow-up work as `PROPOSED FOLLOW-UP:` notes on `sase-1ck.1`.
Create no beads.

Out of scope, owned by later phases: event manifests, the reducer, mutation APIs,
placement and sensitive-path policy, the CAS, the CLI, the beta flag, rendering, and any
`require_rust_binding` call under `src/sase/`.

## Module layout

Add a flat top-level domain `note_attachment`. `crates/sase_core/src/lib.rs` gains
`pub mod note_attachment;` between `model_route` and `notifications`. The module doc
comment is one sentence, so `just modules` can summarize it. `mod.rs` is only `mod` and
`pub use` lines. Keep every file at or under 1,500 lines. Import by module path. Add
nothing to the root `pub use` list in `lib.rs` and no `core_*` alias in
`crates/sase_core_py/src/prelude.rs`. Use no `macro_rules!`. Edit no `version` field and
no `CHANGELOG.md`.

| File                                        | Owns                                                      |
| ------------------------------------------- | --------------------------------------------------------- |
| `note_attachment/mod.rs`                    | Facade and `NOTE_ATTACHMENT_SCAN_WIRE_SCHEMA_VERSION = 1` |
| `note_attachment/extensions.rs`             | Extension table and `classify_attachment`                 |
| `note_attachment/names.rs`                  | `sanitize_attachment_name`, `unique_attachment_name`      |
| `note_attachment/scan.rs`                   | Scan, compose, inverse, `stored_attachment_tokens`        |
| `note_attachment/tests/`                    | Unit and corpus tests                                     |
| `sase_core_py/src/note_attachment/mod.rs`   | Bindings, registered from `lib.rs`                        |
| `sase_core_py/src/note_attachment/tests.rs` | Binding round trip                                        |

Errors are a `thiserror` enum `NoteAttachmentError`. Spans are UTF-8 **byte** offsets,
the same convention as `scan_artifact_refs`. A test with `é @./a.png` pins the `@` at
byte 3 (`é` is two bytes).

## Shared left context and quotes

In `artifact_ref/scanner.rs`, `has_allowed_left_context` stays the single definition.
Add `<` to its matcher. Backtick stays, because it is already part of that set. Make the
function `pub(crate)` and `pub use` it from `artifact_ref/mod.rs`. The note scanner
calls that function. `@` after `<` becomes significant for artifact-ref scans too
(`![alt](@./t.log)` and `<@bead:…>`). Update any scanner test that assumed `<` was not a
boundary.

Make `scan_quoted_argument` and `unescape_quoted_argument` `pub(crate)` and re-export
them the same way. Quoted attachment paths use those two functions: `\"` and `\\` are
escapes, any other backslash is literal, and a quote never crosses a newline. An
unterminated quote ends at the newline or at EOF.

## Extension table and classification

`classify_attachment(name, head) -> AttachmentClassificationWire { mime_type, class }`.

`head` is the first 4096 bytes (the caller slices; the function also caps itself at
4096). `class` is one of `image`, `video`, `audio`, `pdf`, `text`, `archive`, `binary`.
MIME is metadata. SVG is classified and never decoded here.

Magic wins over the extension, then the extension table, then a UTF-8 heuristic (the
head is valid UTF-8 and contains no NUL) yielding `text/plain` / `text`, then
`application/octet-stream` / `binary`. An empty head is valid UTF-8, so an unknown
extension falls through to `text/plain`.

Check magic in this order:

| Signature                                   | MIME                          | Class   |
| ------------------------------------------- | ----------------------------- | ------- |
| `89 50 4E 47 0D 0A 1A 0A`                   | `image/png`                   | image   |
| `FF D8 FF`                                  | `image/jpeg`                  | image   |
| `GIF87a` or `GIF89a`                        | `image/gif`                   | image   |
| `RIFF` at 0 and `WEBP` at 8                 | `image/webp`                  | image   |
| `RIFF` at 0 and `WAVE` at 8                 | `audio/wav`                   | audio   |
| `BM`                                        | `image/bmp`                   | image   |
| `II*\0` or `MM\0*`                          | `image/tiff`                  | image   |
| `%PDF-`                                     | `application/pdf`             | pdf     |
| `PK\x03\x04`, `PK\x05\x06`, or `PK\x07\x08` | `application/zip`             | archive |
| `1F 8B`                                     | `application/gzip`            | archive |
| `BZh`                                       | `application/x-bzip2`         | archive |
| `FD 37 7A 58 5A 00`                         | `application/x-xz`            | archive |
| `28 B5 2F FD`                               | `application/zstd`            | archive |
| `37 7A BC AF 27 1C`                         | `application/x-7z-compressed` | archive |
| `\x7fELF`                                   | `application/x-elf`           | binary  |
| `ftyp` at bytes 4..8; brand `qt  ` at 8..12 | `video/quicktime`             | video   |
| `ftyp` at bytes 4..8, any other brand       | `video/mp4`                   | video   |
| `1A 45 DF A3`, and the head contains `webm` | `video/webm`                  | video   |
| `1A 45 DF A3` otherwise                     | `video/x-matroska`            | video   |
| `OggS`                                      | `audio/ogg`                   | audio   |
| `fLaC`                                      | `audio/flac`                  | audio   |
| `SQLite format 3\0`                         | `application/vnd.sqlite3`     | binary  |

Office files that are zips classify as `application/zip` / `archive` because magic wins.
That is the spec.

One table drives both classification and bare `stem.ext` detection. Lookup is
case-insensitive. Include at least:

- image: `png jpg jpeg gif webp bmp tif tiff svg ico heic heif avif`
- video: `mp4 mov m4v webm mkv avi mpg mpeg`
- audio: `mp3 m4a wav flac ogg opus aac`
- pdf: `pdf`
- text:
  `md txt log json jsonl yaml yml toml csv diff patch html htm svg xml rst ini cfg conf py rs ts tsx js jsx css sh bash zsh go rb java c h cpp hpp cs php sql vue svelte scss graphql proto`
- archive: `zip tar gz tgz bz2 xz 7z rar zst`
- binary: `sqlite db wasm`

`svg` is image (`image/svg+xml`) when chosen by extension; the text list above must not
override that. `ts` is text (`text/typescript`), as in the epic's source list. `sh` is
text even though it looks like a TLD. Representative MIME values: `text/markdown`,
`text/plain` (`txt`, `log`), `application/json`, `application/x-ndjson` (`jsonl`),
`application/yaml`, `application/toml`, `text/csv`, `text/html`, `text/x-python`,
`text/x-rust`, `text/x-shellscript`. Other source extensions use `text/plain`.

Exclude TLD-like extensions, including
`com org net io dev ai app me co uk us ly to tv cc xyz`. A bare `@pytest.mark.asyncio`
stays a bare word because `asyncio` is not in the table.

## Names

`ATTACHMENT_NAME_MAX_LEN` is 96.

`sanitize_attachment_name(candidate)`:

1. Map every character outside `[A-Za-z0-9._-]` to `_`.
2. Collapse `_` runs to one `_`.
3. Replace `_.` with `.` until stable, so `My Screenshot (1).PNG` becomes
   `My_Screenshot_1.PNG`.
4. Strip leading `.` and `-` until stable.
5. Split the last `.suffix` whose suffix matches `[A-Za-z0-9]+` and is shorter than the
   whole string. That suffix is the extension to preserve.
6. If the stem is empty or only `_`, the stem becomes `attachment`. If the whole result
   is then empty, return `attachment`.
7. If the result is longer than 96, shorten the stem so `stem + "." + ext` fits. If the
   extension alone cannot fit, truncate the whole string to 96. Strip a trailing `.` or
   `_` left on the stem by truncation.

`unique_attachment_name(candidate, sha256, existing)` where `existing` is
`{name, sha256}` pairs (roster plus names already assigned in this text):

1. `base = sanitize_attachment_name(candidate)`.
2. Compare digests by ASCII case-fold. The function does not check that the digest is 64
   hex characters.
3. If an existing pair has the same name and the same digest, return `base`.
4. If no pair has that name, return `base`.
5. Otherwise try `stem-2.ext`, `stem-3.ext`, … (no extension: `attachment-2`). Skip a
   candidate that is longer than 96 after the stem is shortened to fit. Return the first
   name whose digest matches, or the first name nobody has.
6. A different existing name with the same digest does not steal this candidate. Reuse
   is same name and same digest.

The caller detects a rename by comparing the result with `base`. The echo text is a
later phase.

## Scanner

`scan_note_attachment_refs(text, roster_names) -> NoteAttachmentScanWire`.

Literal zones are `fenced_block_ranges(text)` plus `inline_code_ranges(text, &fenced)`.
The scanner never classifies inside them, so `@@` and `@./x` in code stay verbatim. Walk
the rest left to right. `@` is significant only when `has_allowed_left_context` is true.
Collect every diagnostic. A syntax problem does not stop the walk.

Citation kinds are the `artifact_ref_kind_catalog` kinds and aliases, plus the baseline
list inside `known_document_kind_labels` (`agent`, `bead`, `bug`, `chat`, `commit`,
`file`, `patch`, `plan`, `plans`, `stitch`, and whatever else that array contains), plus
`research`. Remove `attachment` from that set. Extract the baseline array into one
`pub(crate)` function that `known_document_kind_labels` calls, so the pager list and
this set cannot drift. Adding `research` here must not change pager behavior: it is only
in the note-scanner citation set.

At a boundary `@`, match in this order:

1. **Escape.** The next character is `@`. Record an escape span covering both
   characters. The character after the pair is not a boundary, which falls out of the
   left-context set because `@` is not in it.
2. **Reuse.** The text continues with `attachment:` and then a name: non-empty, at most
   96 characters, `[A-Za-z0-9._-]`, not starting with `.` or `-`. `@attachment:` with a
   missing or illegal name is diagnostic `unknown_attachment` and is not a path. A
   canonical cross-bead form `@attachment:<bead-id>/<name>` does not match (the `/`
   stops the name). It stays ordinary text. Cross-bead prompt expansion is a later
   follow-up.
3. **Citation.** `@` + a citation kind + `:`. Emit nothing.
   `@plan:202609/bead_note_attachments.md` and `@research:x` stay literal even though
   they contain `/` or look like extensions.
4. **Quoted path.** The next character is `"`. Use the exported quote scanner. Empty
   content (`@""`) is diagnostic `empty_path`. An unterminated quote is diagnostic
   `unterminated_quote`. Otherwise record a path ref whose `path` is the unescaped
   content and whose `quoted` flag is true. The span covers `@` through the closing
   quote.
5. **Unquoted token.** Take the run until whitespace, `"`, `'`, or `` ` ``. Trim
   trailing `. , ; : ! ? ) ] } >` from the span (the punctuation stays outside the span
   and is copied by compose). Then:
   - prefix `/`, `~/`, `./`, or `../` → path, even when later characters would fail the
     segment class
   - the token contains `/` and every segment matches `[A-Za-z0-9._+~-]+` → path (`..`
     matches that class; refusing parent segments is Python's job in a later phase)
   - `stem.ext` whose final extension is in the table → path
   - otherwise a leading run of `[A-Za-z0-9._+-]` is a bare-word mention (`@large`,
     `@dataclass`, `@pytest.mark.asyncio`)
   - a lone `@` emits nothing

Path `path` is the trimmed, unescaped text without the leading `@`. Quoted paths keep
internal spaces.

Reuse binding, using only names, not digests:

- The name is in `roster_names` → `binding: "roster"`.
- Else an earlier path ref in this scan has `sanitize_attachment_name(basename)` equal
  to the name → `binding: "earlier_path"` and `earlier_path_index` of the **earliest**
  such path.
- Else → `binding: "unknown"` and diagnostic `unknown_attachment`.

`basename` is the last `/` segment (`~/logs/crash.log` → `crash.log`).

Stable diagnostic codes, with a non-empty `message` and `hint`:

| Code                 | When                                                | Hint                                                                               |
| -------------------- | --------------------------------------------------- | ---------------------------------------------------------------------------------- |
| `unterminated_quote` | `@"` does not close on this line                    | Close the quote, or write `@@` for a literal `@`.                                  |
| `empty_path`         | `@""`                                               | Write a path inside the quotes, or remove the reference.                           |
| `unknown_attachment` | `@attachment:` name is missing, illegal, or unbound | Attach the file in this note first, or write `@@attachment:NAME` for literal text. |

### Scan wire

```rust
pub struct NoteAttachmentScanWire {
    pub schema_version: u64, // NOTE_ATTACHMENT_SCAN_WIRE_SCHEMA_VERSION
    pub path_refs: Vec<NoteAttachmentPathRefWire>,
    pub reuse_refs: Vec<NoteAttachmentReuseRefWire>,
    pub escapes: Vec<NoteAttachmentSpanWire>,
    pub bare_words: Vec<NoteAttachmentBareWordWire>,
    pub diagnostics: Vec<NoteAttachmentDiagnosticWire>,
}
```

Each path ref has `raw` (the spanned source, including `@` and quotes), `path`,
`quoted`, and `span`. Each reuse ref has `name`, `span`, `binding` (`roster` |
`earlier_path` | `unknown`), and `earlier_path_index: Option<u64>`. Each bare word has
`word` and `span`. Each diagnostic has `code`, `message`, `span`, and `hint`. Spans are
`{start, end}` byte offsets. Serde uses snake_case. Skip empty `Option`s the way
neighboring wires do.

### Oracle

Roster for the reuse rows is `{login.png}` unless a row says otherwise. Assigned name
for a single path, when compose is applied, is `login.png` except `![t](@./t.log)`,
whose assigned name is `t.log`.

| Source                                                           | Result                                                                                    |
| ---------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| `me@host`                                                        | No record. `@` is not at a boundary.                                                      |
| `@large`                                                         | Bare word `large`.                                                                        |
| `@dataclass`                                                     | Bare word `dataclass`.                                                                    |
| `@pytest.mark.asyncio`                                           | Bare word `pytest.mark.asyncio`.                                                          |
| `@research:x`                                                    | No record.                                                                                |
| `@plan:202609/bead_note_attachments.md`                          | No record.                                                                                |
| `@@./x`                                                          | One escape. Compose yields `@./x`.                                                        |
| `@@@x`                                                           | One escape over the first two `@`. The third `@` is not a boundary. Compose yields `@@x`. |
| `` `@./x.png` ``                                                 | No record. Inline code.                                                                   |
| A fenced block containing `@@` and `@./x`                        | No record.                                                                                |
| `@"a b.png"`                                                     | Quoted path `a b.png`.                                                                    |
| `@./x.png,`                                                      | Path `./x.png`. The comma is outside the span.                                            |
| `(@./x.png)`                                                     | Path `./x.png`. The `)` is outside the span.                                              |
| `![t](@./t.log)`                                                 | Path `./t.log`.                                                                           |
| `@AGENTS.md`                                                     | Path `AGENTS.md` (`md` is in the table).                                                  |
| `@/tmp/a.png`                                                    | Path `/tmp/a.png`.                                                                        |
| `@~/logs/a.log`                                                  | Path `~/logs/a.log`.                                                                      |
| `@../x`                                                          | Path `../x`.                                                                              |
| `@docs/plan.md`                                                  | Path `docs/plan.md`.                                                                      |
| `@attachment:login.png`                                          | Reuse, binding `roster`.                                                                  |
| `@attachment:login.png` with an empty roster and no earlier path | Reuse, binding `unknown`, diagnostic `unknown_attachment`.                                |
| `@./shots/login.png and @attachment:login.png`                   | Path, then reuse bound to that path's index.                                              |
| `@""`                                                            | Diagnostic `empty_path`.                                                                  |
| `@"` at end of line                                              | Diagnostic `unterminated_quote`.                                                          |
| `@attachment:sase-1ck/login.png`                                 | No path and no reuse.                                                                     |

Compose of a path replaces the span with `@attachment:<assigned name>` and copies the
surrounding text, including trimmed punctuation.

## Compose and the editing inverse

`compose_note_attachment_text(text, scan, assigned_names) -> Result<String, NoteAttachmentError>`.

`assigned_names` aligns with `scan.path_refs`. A different length is
`AssignedNameCount`. A name that `sanitize_attachment_name` would change is
`UnsanitizedAssignedName`. Compose does not re-sanitize and does not consult the
filesystem.

Rebuild the string from `text` and the scan spans (they do not overlap; debug-assert
that in tests):

- Gap: copy.
- Escape span: write one `@`.
- Path span: write `@attachment:<assigned_names[i]>`.
- Reuse with `earlier_path`: write `@attachment:<assigned_names[index]>`, so a later
  uniquify to `login-2.png` retargets the same-text reuse.
- Roster or unknown reuse: copy the original span. Unknown reuses stay in the string.
  Callers must not persist a scan whose `diagnostics` are non-empty. Compose itself
  stays total.

`note_attachment_source_text(stored, manifest_names) -> String`.

This is the editing inverse. It works for legacy notes (`manifest_names` empty). Scan
`stored` with `roster_names = manifest_names`. Copy `stored`, and insert one extra `@`
immediately before each span that is an escape, a path, a quoted path, an unknown reuse,
or a diagnostic. Leave roster reuses (the manifest's own `@attachment:<name>` tokens)
and every literal zone untouched.

Property, for arbitrary `stored` and any manifest set `M`: let
`y = note_attachment_source_text(stored, M)` and
`scan = scan_note_attachment_refs(y, M)`. Then `scan.path_refs` is empty,
`scan.diagnostics` is empty, and `compose_note_attachment_text(y, scan, []) == stored`.

`stored_attachment_tokens(text)` returns `{name, span}` for every boundary
`@attachment:<name>` outside literal zones whose name matches the sanitized-name
predicate. It does not consult a roster. Trailing punctuation is outside the name. Wire
validation in the next phase uses this function. Include it now.

## `attachment` artifact-ref kind

Register the kind in `artifact_ref/kinds.rs` `registrations()`:

- kind `attachment`, display name `Attachment`, status `Live`, no aliases,
  `reserved: true`
- `argument_summary`: `attachment:<bead-id>/<name>`
- `offered_in_completion: false` (completion and cross-bead expansion are later work)
- `fragment_probe` uses `ArtifactRefKindWire::Document { role: "attachment" }`
- no diagnostic on the registration

Add `attachment` to the hardcoded baseline array in `known_document_kind_labels` as well
as the catalog entry. Catalog iteration already inserts registered kinds; the baseline
entry is what the phase asked for and keeps the label visible beside `bead` and `file`.

Keep the serde enums unchanged. `attachment:…` is
`ArtifactRefKindWire::Document { role: "attachment" }` with
`ArtifactRefPayloadWire::Document { path }`. In `parse_payload`, when the role is
`attachment`, require exactly one `/`:

- the left side passes the existing `validate_bead_id`
- the right side passes the sanitized-name predicate (non-empty, length ≤ 96,
  `[A-Za-z0-9._-]`, no leading `.` or `-`)
- `attachment:login.png` (no slash) and `attachment:a/b/c` fail validation
- `attachment:sase-1ck.1/login.png` parses and renders as itself

`kind_rejects_fragments` returns true for role `attachment`, same as the `tool` special
case.

In `resolve_artifact_ref`, match role `attachment` before generic document resolution.
Return `unknown_kind` with diagnostic
`attachment references resolve through the bead attachment store, not this crate`. A
document root whose kind is `attachment` must not turn the payload into a filesystem
path.

Extend the catalog tests: `attachment` is reserved, live, absent from completion, and
`accepts_fragment` is false. Add a parse test and a resolve test for the cases above.

## Python bindings

New domain `crates/sase_core_py/src/note_attachment/`. Follow `project_tag`:
`#[pyfunction]`, `#[pyo3(name = "<python name>")]`, `fn py_<python name>`, input dicts
through `py_to_json_value` into the `*Wire` types, core errors mapped to `PyValueError`,
output through `serialize_to_py`. Register each function in `register_note_attachment`,
and call that from `sase_core_rs` in `lib.rs` next to `register_beads`. The compiler
does not check registration.

Python names, matching the epic:

- `classify_attachment(name, head)` with `head` as `bytes`
- `scan_note_attachment_refs(text, roster_names)`
- `compose_note_attachment_text(text, scan, assigned_names)` where `scan` is the scan
  dict
- `note_attachment_source_text(stored, manifest_names)`
- `stored_attachment_tokens(text)`
- `sanitize_attachment_name(candidate)`
- `unique_attachment_name(candidate, sha256, existing)` where `existing` is a list of
  `{name, sha256}` dicts

The binding test builds a module, registers the domain, and round-trips one scan →
compose → source cycle, one classification of PNG bytes named `note.txt` (magic wins:
`image/png`), and one uniquify of `login.png` against a different digest
(`login-2.png`).

There is no `proptest` crate in this workspace. Satisfy the epic's property test with a
seeded xorshift loop of 256 cases inside `note_attachment/tests`, so hakari and
`Cargo.lock` stay untouched. Each case builds a string from literal chunks, escapes,
path tokens, `@attachment:` tokens, `@plan:` / `@research:` citations, and fenced or
inline code, then asserts the inverse property above. Also keep the hand-written oracle
table.

## Corpus golden

From the sase checkout, export notes with `sase bead list -s all -n 0 -f json`. The JSON
is a list of issues, or an object whose issue array contains them. Read `notes[].text`
only (`src/sase/bead/note_codec.py`). Drop empty texts. Keep those that contain `@`.
Deduplicate exact strings and sort them.

Commit two fixtures under `crates/sase_core/tests/fixtures/note_attachment/`:

- `at_bearing_notes.jsonl` — one JSON string per line, the deduplicated texts, and
  nothing else (no ids, authors, or timestamps)
- `path_or_diagnostic.json` — for an empty roster, every fixture text whose scan has a
  path ref or a diagnostic, as `{sha256, path_ref_count, diagnostic_codes}` sorted by
  `sha256`

The test loads both via `CARGO_MANIFEST_DIR`, re-scans, and compares. It also asserts
the fixture line count so a silent truncation fails.

The epic's research estimate is that about 0.3% of notes yield a path reference. After
generating, record the real counts on the phase bead with `sase bead note`: total notes
exported, `@`-bearing count, path-or-diagnostic count, and that count as a percentage of
total notes.

If the golden shows an avoidable false positive that is a real SASE document kind (for
example `@prompt:…` or `@memory:…` with a slash), add that label to the note-scanner
citation set in this same change and mention it in the close note. That is the epic's
allowed tuning step. Leave emails, `@large`, and mid-word `@` as non-matches; those
should already be clean.

## Pin bump

`sase-core-revision.txt` is the 40-hex SHA of sase-core's **remote** HEAD.
`just ratchet-core-revision` (`tools/ratchet_core_revision`) writes that SHA from
`git ls-remote https://github.com/sase-org/sase-core.git HEAD`. It cannot name a commit
that is not yet that remote HEAD, and a hand-written SHA CI cannot fetch breaks the sase
build.

This phase adds no `src/sase` caller, so `tools/check_sase_core_rs_bindings` stays green
against the current pin.

When the grammar commit is already remote HEAD, run `just ratchet-core-revision` and
then `tools/check_sase_core_rs_bindings` from the sase checkout. Include the pin file in
the sase commit.

When the grammar commit exists only in the local linked checkout, leave
`sase-core-revision.txt` unchanged. Record this on the phase bead:

`PROPOSED FOLLOW-UP: Bump sase-core-revision.txt with just ratchet-core-revision once the note-attachment grammar commit is sase-core remote HEAD — this phase adds no src/sase callers, so the pin cannot move until that commit is published.`

## Verification and close

In the linked sase-core checkout, `just fmt`, then targeted
`just test -p sase_core note_attachment` and `just test -p sase_core_py note_attachment`
while iterating. The gate is `sase tool run check` there (the wrapped check, not a raw
`just check`). Give it at least 10 minutes. A test that fails under the full check and
passes alone is a load flake until you confirm it against `sase bead list -T flake`;
leave the assertion intact.

Before closing, run `sase bead epic-symbols sase-1ck.1`. If any `--epic-symbol` entries
remain, retarget those Justfile lines at the still-open parent epic `sase-1ck` or at a
later open phase. `sase bead close` refuses while leftovers remain.

Close only `sase-1ck.1`:

`sase bead close sase-1ck.1 --note "<what you verified: check result, corpus counts, whether the pin moved>"`

A check failure that reproduces on the clean base tree gets a `PROPOSED FOLLOW-UP:` note
citing any bead that already tracks it, and the phase still closes.
