---
tier: epic
title: Bead note attachments
goal: "Any file — screenshot, log, trace, archive, or multi-GiB binary — can be attached
  to a bead note with an inline `@<path>` reference. The bead keeps an immutable
  snapshot of the bytes that is available on every machine and outlives the source file.
  Humans see beautiful chips, badges, and optional inline image previews. Agents get
  plain, extension-preserving file paths.

  "
phases:
  - id: core_grammar
    title: Core attachment grammar, names, and media classification (sase-core)
    depends_on: []
    size: large
    description:
      "core_grammar: add the note-text scanner for @path/@@/@attachment tokens,
      stored-text composition and its editing inverse, name sanitizing and uniquing,
      media classification, and the attachment artifact-ref kind in sase-core, with
      bindings, a corpus golden test, and the sase pin bump."
  - id: cas
    title: Local content-addressed attachment store and streaming ingest
    depends_on: []
    size: medium
    description:
      "cas: build the ~/.sase/attachments content-addressed store, with one-pass
      streaming ingest (sparse-aware, change-detecting, per-digest locked),
      extension-preserving views, image probing, and the BlobStore protocol."
  - id: core_wire
    title: Attachment wire, reducer, mutation APIs, and policy (sase-core)
    depends_on:
      - core_grammar
    size: medium
    description:
      "core_wire: add the optional attachments manifest on note events and BeadNoteWire,
      token/manifest validation, reducer and mutation API support (append, edit, close,
      +1), roster and reference queries, placement/fetch/sensitive-path policy, and
      tombstone wire, with bindings and the pin bump."
  - id: note_cli
    title: Author and read attachments from the CLI (beta flag)
    depends_on:
      - core_grammar
      - core_wire
      - cas
    size: large
    description:
      "note_cli: create the bead_note_attachments beta flag, then build the shared
      authoring service. Wire it into note, close -n, update -n, +1 -n, and the TUI
      add-note modal. Add sase bead attach and attachment list/path, text rendering for
      show/read/JSON, caret diagnostics, and the write echo."
  - id: shared_store
    title: Private attachments sidecar, upload outbox, and lazy fetch
    depends_on:
      - note_cli
    size: large
    description:
      "shared_store: add the reserved private attachments sidecar role (a hidden bare
      partial clone), the git BlobStore written with plumbing, and placement with
      explicit -L local-only. Add pre-publication uploads with an outbox fallback,
      capped lazy fetch, availability badges, attachment push, and a doctor check."
  - id: large_files
    title: Large-file store, background uploads, and progress
    depends_on:
      - shared_store
    size: medium
    description:
      "large_files: add the optional rclone large-object store tier, background uploads
      for big objects, progress UI, and end-to-end large and sparse file acceptance
      tests."
  - id: show_images
    title: Image previews and full-fidelity viewing
    depends_on:
      - note_cli
    size: large
    description:
      "show_images: add the show -i/--images auto|cells|kitty|never option and
      bead.show.images config. Add cell thumbnails, kitty inline pixels, and text
      previews. Make attachment chips labeled pager links into the existing viewer, add
      sase bead attachment open, and resolve attachment: artifact refs."
  - id: tui
    title: Beads pane attachments and add-note authoring UX
    depends_on:
      - show_images
    size: medium
    description:
      "tui: add an attachments block with thumbnails and badges to Beads pane note
      detail, an open-attachments key, and @ path completion, paste handling, and inline
      diagnostics in the add-note modal, all off the event loop."
  - id: lifecycle
    title: Purge, doctor, cache pruning, and bead pages
    depends_on:
      - large_files
    size: medium
    description:
      "lifecycle: add tombstone-based attachment purge, bead doctor attachment checks
      and repairs, and cache/orphan pruning, and render private attachments on bead
      pages."
  - id: ga
    title: Remove the beta flag and finish docs
    depends_on:
      - large_files
      - show_images
      - tui
      - lifecycle
    size: medium
    description:
      "ga: delete the flag's Off branches and close the flag bead. Finish user docs,
      help, and skill sources, run the end-to-end acceptance sweep, and record the
      follow-up proposals, including the memory update."
proposed_by: bbugyi200.athena.0tv
create_time: 2026-09-29 08:13:33
status: wip
---

- **PROMPT:**
  [prompts/202609/bead_note_attachments.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/bead_note_attachments.md)

# Plan: Bead note attachments

## Background

Research report: `research:202609/bead_note_attachments/bead_note_attachments.md` (read
it with `sase artifact read`). It consolidates five reports and independent checks. Its
central findings drive this plan:

- **A whole-note `@<path>` does not attach anything today.** `read_at_path_value()`
  (`src/sase/cli_file_values.py`) reads the file's UTF-8 text _into_ the note, with no
  size cap. The same rule applies to about ten flags. Agents rely on it for long prose:
  land agents run `sase bead close <id> --note @/tmp/…/close_note.md` in dozens of
  transcripts.
- **Evidence cited by path is dying.** 45 of 90 local evidence paths cited in beads no
  longer exist, and none of them can be read from apollo or mac.
- **`@` is everywhere in notes.** 7.7% of notes contain `@`: `@large`, `repo@sha`,
  `@research:…` citations. Only about 0.3% contain a boundary `@` that is path-shaped.
- **The public beads repo cannot hold bytes.** Every SASE sidecar is public, and the
  beads sidecar is hot and cloned into most workspaces.
- **Events accept new optional fields, but not new operations.** Bead event payloads are
  serde-tagged with no `deny_unknown_fields`, so an optional field is ignored by old
  readers. An unknown event _operation_ is a hard read error.
- **Viewing pieces already exist.** SASE has an image viewer (`kitten icat`), a subpixel
  cell renderer (`src/sase/ace/tui/graphics/cell.py`, 25 MiB / 40 MP caps), and pager
  `MEDIA` link targets. `artifact_file_view_mode()` picks the viewer by file suffix, so
  paths must keep their extension.
- **Every note-bearing verb already runs in Python.** `note`, `close`, `+1`, and
  `update -n` route there (`src/sase/main/bead_fast_path.py`, sase-core
  `bead/cli/dispatch.rs`). Notes reach Rust through `bead_append_note`,
  `bead_note_edit`, `bead_close(note=…)`, and `bead_plus_one`.

## The mental model (verbatim in help and docs)

> **`@` in bead notes.** Inside note text, `@<path>` attaches a snapshot of that file.
> The bead keeps the exact bytes on every machine, even after the file is gone. Accepted
> forms: `@./shot.png`, `@~/logs/crash.log`, `@/tmp/trace.json`, `@docs/plan.md`,
> `@"name with spaces.png"`, or a bare `@name.ext` for a known file type. Write `@@`
> where you need a literal `@` that would otherwise start a reference. `me@host`,
> `@large`, and `@research:…` citations never need escaping. A note argument that is
> _only_ `@<file>` still reads the note's text from that file, and any `@<path>` inside
> that text then attaches. To attach files without prose, use `sase bead attach`.

## Settled design decisions

These refine the literal request. Each choice is deliberate, and the reason is given.

1. **`@<path>` anywhere in note text attaches a snapshot; it never splices file text
   in.** Splicing is impossible for binaries and bloats events and search. The note
   stores `@attachment:<name>` tokens plus a manifest of content-addressed descriptors.
   It never stores paths.
2. **`@@` is required wherever a single `@` would start a reference.** That means at a
   word boundary before a path-shaped token, a quoted path, `@attachment:`, or another
   `@@`. `@@` at a boundary always collapses to one literal `@`, so it is always a safe
   escape. A mid-word `@` and non-path `@words` stay literal with no escape. A
   path-shaped reference that does not resolve is a **hard error with carets**, and
   nothing is written. So nothing is silently misread, and SASE's own `@` vocabulary is
   not taxed.
3. **A whole-argument `@file` keeps meaning "read this note's text from the file".**
   Transcripts show land agents depend on it. Then:
   - The loaded text is scanned for attachment references.
   - A whole argument that starts with `@@` passes through untouched to the text
     grammar, so escapes are processed exactly once.
   - Note prose read this way must be UTF-8 and at most 256 KiB. A binary or oversized
     file fails with the exact `sase bead attach …` command and the inline alternative
     (`"… @./shot.png"`).
   - On a TTY, a dim line confirms
     `note text read from ./crash.log (10 KiB) · to attach it instead: sase bead attach sase-ab ./crash.log`.
4. **`sase bead attach` is the prose-free verb.** `-a` on `note` is already `--author`,
   and a verb reads better and takes stdin.
5. **Snapshot on write, bytes outside the bead store.** A one-pass streaming ingest
   writes into a local CAS before the event is appended. Bytes are then shared through a
   **private `attachments` sidecar**: a private git repo, held on each machine as a
   hidden bare partial clone, for objects up to 50 MiB. An optional **rclone large
   store** takes objects above that. Keeping bytes on one machine only is an **explicit,
   badged choice** (`-L/--local-only`).
6. **Descriptors are location-free:**
   `{name, sha256, size_bytes, mime_type, image?, origin?}`. Stores are an ordered,
   digest-keyed configuration list, so a local-only object can be promoted later without
   editing any note. Field names follow existing core wires (`size_bytes`, `mime_type`).
7. **Images are optional in every sense.** Only `sase bead show` draws, controlled by
   `-i/--images auto|cells|kitty|never` and the permanent config field
   `bead.show.images`. `sase bead read`, JSON, piped output, and agents never draw; they
   get absolute, extension-preserving paths that agent Read tools open directly.
8. **Authoring is gated by a `beta` flag, `bead_note_attachments`; rendering is never
   gated.** Notes written with attachments on one machine render correctly on a machine
   where the flag is off. The `ga` phase removes the flag.
9. **Core owns every rule a second frontend would need to match.** That covers grammar,
   tokens, names, media classification, wire, reducer, placement, sensitive-path, and
   fetch policy, and tombstones. Python owns I/O: files, stores, the outbox, terminal
   rendering, the CLI, and the TUI (`rust_core_backend_boundary`).

## Shared specification (all phases conform)

### Grammar (core)

The scanner walks note text left to right. It never scans inside **literal zones**:
fenced code blocks and inline code spans (reuse sase-core `fenced_block_ranges` /
`inline_code_ranges`). Code stays verbatim, including any `@@`.

| Rule          | Behavior                                                                                                                                                                                                                                                                           |
| ------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Boundary      | `@` is significant only at the start of the text or after the artifact-ref scanner's left-context set (whitespace, `"`, `'`, `(`, `[`, `{`, `,`, `=`), plus `<`. Share one definition with `has_allowed_left_context`.                                                             |
| Escape        | Boundary `@@` becomes a literal `@`. The next character is _not_ a boundary.                                                                                                                                                                                                       |
| Reuse         | `@attachment:<name>` references an attachment already on this bead, or one attached earlier in the same text by its assigned name. An unknown name is an error.                                                                                                                    |
| Citation      | `@<kind>:<arg>` for a registered or known document kind stays literal text, as today.                                                                                                                                                                                              |
| Quoted path   | `@"…"` uses the existing quoted-candidate rules (`\"`, `\\`, never crosses a newline).                                                                                                                                                                                             |
| Path-shaped   | `@/…`, `@~/…`, `@./…`, `@../…`. Also a token containing `/` whose segments all match `[A-Za-z0-9._+~-]+`. Also a bare `stem.ext` where `ext` (case-insensitive) is in the curated core extension table, which has no TLD-like entries. Trailing `. , ; : ! ? ) ] } >` are trimmed. |
| Anything else | Literal (`@large`, `@dataclass`, `@pytest.mark.asyncio`, bare `@`). The scanner reports bare-word mentions so the CLI can hint "write `@./word` to attach it" when `./word` exists.                                                                                                |

Resolution happens in Python. Paths resolve against the **invocation cwd** (also for
text loaded from a whole-argument `@file`), with `~` expanded. The target must be a
readable regular file after following symlinks. Directories get a `tar czf` hint; FIFOs,
devices, and sockets are refused. Sensitive paths are refused unless
`-S/--allow-sensitive` (see Policy). Every problem is collected and reported together,
**before anything is written**:

```text
Error: 1 attachment problem in note text — nothing was written.
  Crash after login @./shots/login.png — see @AGENTS.md
                                             ^^^^^^^^^^
  @AGENTS.md: file not found (resolved to …/AGENTS.md).
    Write @@AGENTS.md for literal text, or fix the path.
```

**Stored text** is the display form: escapes are collapsed and each path reference is
replaced by `@attachment:<name>`. **Editing** uses the core inverse,
`note_attachment_source_text(stored, manifest_names)`. It re-escapes every boundary `@`
that would otherwise parse as a reference, except the manifest's own tokens. It works
the same for legacy notes with no manifest. Property: scanning and composing
`source(x, M)` yields exactly `x`, with no path references.

### Names, media, identity (core)

- **Names** are sanitized to `[A-Za-z0-9._-]`: other characters become `_`, runs of `_`
  collapse, leading `.`/`-` are stripped, and the length is capped (about 96) with the
  extension preserved. An empty result becomes `attachment`.
- **Names are unique per bead roster.** Same name with the same digest reuses the
  attachment. Same name with a different digest becomes `login-2.png`, and the echo says
  so.
- **Identity** is the SHA-256 of the bytes.
- **`mime_type` classification** checks magic signatures in the first 4 KiB first (PNG,
  JPEG, GIF, WEBP, BMP, TIFF, PDF, ZIP, gzip, bzip2, xz, zstd, 7z, ELF, MP4/MOV `ftyp`,
  Matroska/WebM, Ogg, FLAC, WAV, SQLite). Then the extension table. Then a UTF-8 text
  heuristic (valid UTF-8 and no NUL). The fallback is `application/octet-stream`.
- **MIME is metadata, never admission control.** SVG is classified but never decoded for
  previews.

### Wire (core, additive, no new operation)

```json
{
  "kind": "note_appended",
  "entry": "Crash right after login @attachment:login.png — full log: @attachment:crash.log",
  "attachments": [
    {
      "name": "login.png",
      "sha256": "9f2c…e1",
      "size_bytes": 188416,
      "mime_type": "image/png",
      "image": { "width": 1280, "height": 720 },
      "origin": "athena"
    },
    {
      "name": "crash.log",
      "sha256": "41aa…07",
      "size_bytes": 2202009,
      "mime_type": "text/plain",
      "origin": "athena"
    }
  ]
}
```

- `NoteAppended.attachments: Vec` (default, skipped when empty).
- `NoteEdited.attachments: Option<Vec>`. `None` means an older writer's edit and keeps
  the manifest; `Some` replaces it.
- `BeadNoteWire.attachments: Vec` (skipped when empty). Existing events and projections
  stay **byte-identical**, and no migration is needed.
- When a manifest is non-empty, tokens and manifest must match one-to-one.
- No paths, store locations, or machine-local ids are ever recorded. **Descriptors are
  as public as the bead itself; bytes are private.**
- `BEAD_EVENT_SCHEMA_VERSION` stays 1.

### Bytes and stores

- **Local CAS** at `<sase home>/attachments/` (honor `SASE_HOME`):
  - `objects/sha256/<xx>/<sha256>` — read-only, mode 0444
  - `views/<sha256[:16]>/<name>` — relative symlinks, so every path shown to a human,
    the viewer, or an agent ends in the real filename
  - `tmp/`, `locks/`, `tombstones/`

  The CAS is always written first, and it doubles as the read cache.

- **Placement** (core policy) picks stores by size from an ordered tier list:
  - The **git tier** (`bead.attachments.git_max_bytes`, default 50 MiB, validated at or
    below 95 MiB).
  - The **large tier** (`bead.attachments.large_store`, optional, default max 2 GiB).
  - If no configured store accepts the size, the command fails before writing unless
    `-L` is given; a TTY asks y/N.
  - If the project has no shared store at all, attachments are local by design and the
    echo says so every time.
- **Remote layout** reuses `artifact_object_relpath`
  (`files/objects/sha256/<xx>/<sha>`), with tombstones at
  `files/tombstones/sha256/<xx>/<sha>.json`.
- **Uploads never run under the bead lock.** The default order is: append and commit the
  event under the lock, then upload in a _pre-publication_ step, then publish the bead
  store. So in the common case other machines never see a note before its bytes.
  - A failed upload goes to a durable outbox and shows `⇡ pending upload`.
  - `bead.attachments.require_upload: true` uploads _before_ the event and aborts with
    nothing written if the upload fails.
- **Fetch order:** local hit, then stores in order. Fetches are digest-verified.
  Automatic fetching is capped at `bead.attachments.auto_fetch_max_bytes` (25 MiB).
  Explicit `path`/`open`/`-d/--download` always fetch.

### Availability states and badges

`cached` (no badge) · `remote` (fetchable) · `not_downloaded` (above the auto-fetch cap:
`⇣ not downloaded · 1.8 GiB — sase bead attachment path sase-ab dump.core`) ·
`pending_upload` (`⇡ pending upload (athena)`) · `local_only` (`⚠ only on athena`) ·
`unavailable` (`✕ unavailable offline`) · `purged` (`(purged)`) · `corrupt`
(`‼ digest mismatch`). **Prose always renders.** A preview or fetch failure never fails
`show`/`read`. Beads without attachments pay **zero** cost: no attachment imports, git,
or network.

### Presentation

`src/sase/bead/note_presentation.py` (or a sibling module) owns chips, glyphs, badges,
and humanized sizes/dims, so CLI, pager, pages, and TUI agree.

- **Glyphs:** `🖼` image, `🎞` video, `♫` audio, `📕` PDF, `≡` text, `▤` archive, `◇`
  other. Measure every glyph with `rich.cells.cell_len` and tune against screenshots.
- **Chips** use the path accent color. Filenames are stripped of control and bidi
  characters for display.

`sase bead show` (TTY, `-i auto`):

```text
  #3 · 2026-09-29 06:12 EDT · 3m ago · bryan · 📎 2
     Crash right after login 🖼 login.png — full log: ≡ crash.log

     🖼 login.png   image/png · 1280×720 · 184 KiB                          [a]
        (cell thumbnail, ≤ 10 rows)
     ≡ crash.log    text/plain · 2.1 MiB                                    [b]
        │ 2026-09-29T06:11:58 ERROR session: token refresh failed …  (≤ 5 dim lines)
```

`sase bead read`, and `show -i never` or piped output, are identical and free of escape
codes and file contents:

```text
  #3 · 2026-09-29 06:12:40 EDT · bryan · 2 attachments
     Crash right after login [login.png] — full log: [crash.log]
     ATTACHMENTS
       login.png   image/png · 1280×720 · 184 KiB · sha256:9f2c1e…
         /home/bryan/.sase/attachments/views/9f2c1e0b77aa4c10/login.png
       crash.log   text/plain · 2.1 MiB · sha256:41aa07…
         /home/bryan/.sase/attachments/views/41aa07c3e9b1d2f0/crash.log
```

Write echo (stderr; plain for agents):

```text
$ sase bead note ab "Crash right after login @./shots/login.png — log: @/tmp/crash.log"
  🖼 login.png  image/png · 1280×720 · 184 KiB   ⇡ sase-org/sase--attachments (private) 0.8s
  ≡ crash.log   text/plain · 2.1 MiB             ⇡ sase-org/sase--attachments (private) 0.4s
Noted: sase-ab — Login crashes on fresh profile  (#3 · 📎 2)
```

- **JSON:** `notes[].attachments[]` carries the descriptor, `availability`, and
  `local_path` (only when cached). Never bytes or ages.
- **Compact format:** `📎N`.
- **Attachment text is never written raw to a terminal:** build Rich `Text`, never
  `from_ansi`.

### CLI surface (alphabetical; every long option has a short alias)

```text
sase bead attach <id> <file|->... [-a/--author NAME] [-L/--local-only] [-n/--note TEXT] [-N/--name NAME] [-S/--allow-sensitive]
sase bead attachment list  <id> [-j/--json]          # bare `sase bead attachment` delegates to list
sase bead attachment open  <id> [<name>]             # show_images
sase bead attachment path  <id> <name>               # materialize (fetching if needed); print the absolute view path
sase bead attachment prune [-y/--yes]                # lifecycle; dry-run plan unless -y
sase bead attachment purge <id> <name> -r/--reason WHY [-y/--yes]   # lifecycle
sase bead attachment push  [<id>]                    # shared_store; drain outbox, promote local-only
sase bead note|close -n|update -n|+1 -n ... [-L/--local-only] [-S/--allow-sensitive]
sase bead show ... [-d/--download] [-i/--images auto|cells|kitty|never]
sase bead read ... [-d/--download]                   # never draws
```

- `-N/--name` is valid only with a single file and is required for `-`.
- `attach` writes one note: the optional `-n` prose, a blank line, then the tokens.
- Before adding any short letter, recheck it per parser (`-x`, `-a`, `-n`, and `-r`
  differ across verbs).
- Place `attach`/`attachment` alphabetically in `sase bead -h`.
- Bare `attachment` uses the central default-list delegation
  (`_default_list_subcommands()`).

### Configuration (`src/sase/default_config.yml`, `sase.schema.json`, `src/sase/bead/config.py`)

```yaml
bead:
  attachments:
    auto_fetch_max_bytes: 26214400 # 25 MiB
    background_upload_min_bytes: 67108864 # 64 MiB (large_files)
    git_max_bytes: 52428800 # 50 MiB; must be <= 95 MiB
    large_store: null # {remote: "<rclone remote:path>", max_bytes: 2147483648}
    local_cache_max_bytes: 10737418240 # 10 GiB prune budget (lifecycle)
    require_upload: false
    sensitive_patterns: [] # extra globs refused unless -S
  show:
    images: auto # auto | cells | kitty | never
```

Accessors fail open to the defaults, like the existing `bead` accessors. Each phase adds
only the keys it uses, and documents them in `docs/configuration.md`.

### Policy (core)

Refuse sensitive paths unless `-S`:

- `~/.ssh/**`, `~/.gnupg/**`
- `**/.env`, `**/.env.*`
- `*.pem`, `*.key`, `*id_rsa*`, `*id_ed25519*`, `*.kdbx`
- `**/credentials.json`, `**/.netrc`, `**/.aws/credentials`, `**/.git-credentials`
- `**/.config/gh/hosts.yml`
- plus any configured `sensitive_patterns`

Objects stay pinned while any current _or historical_ event references them.

## Phase 1: `core_grammar`

Work in the linked `sase-core` checkout (`sase repo open sase-core`) and follow its
`AGENTS.md`. Add a `bead` attachments module; exact file splits are the implementer's
choice.

- **Curated extension table:** extension → (`mime_type`, class ∈ image, video, audio,
  pdf, text, archive, binary). Cover common image, video, audio, document, data, log,
  archive, and source extensions (`md`, `txt`, `log`, `json`, `jsonl`, `yaml`, `toml`,
  `csv`, `diff`, `patch`, `html`, `svg`, `py`, `rs`, `ts`, `sh`, …). Exclude TLD-like
  extensions (`com`, `org`, `net`, `io`, `dev`, `ai`, `app`, …).
- **`classify_attachment(name, head) -> {mime_type, class}`** as specified above.
- **`scan_note_attachment_refs(text, roster_names)`** returns path references (raw
  token, path, quoted flag, byte span), reuse references, escapes, bare-word mentions,
  and diagnostics (`code`, `message`, `span`, `hint`; for example `unterminated_quote`,
  `empty_path`, `unknown_attachment`).
- **`compose_note_attachment_text(text, scan, assigned_names)`** and
  **`note_attachment_source_text(stored, manifest_names)`**.
- **`stored_attachment_tokens(text)`** finds boundary `@attachment:<name>` tokens
  outside literal zones. Wire validation and renderers use it.
- **`sanitize_attachment_name`** and
  **`unique_attachment_name(candidate, sha256, existing)`**.
- **Register the `attachment` artifact-ref kind** (canonical
  `attachment:<bead-id>/<name>`) in `artifact_ref/kinds.rs`, plus any hardcoded
  known-label list, so `parse_artifact_ref` accepts it.
- **PyO3 bindings** in a new `sase_core_py/src/` domain module, registered by hand in
  `lib.rs`, with a binding round-trip test.
- **Tests:**
  - A table-driven test for every grammar row, including `me@host`, `@large`,
    `@research:x`, `@@./x`, `@@@x`, code spans/fences, quoted paths, trailing
    punctuation, `(@./x.png)`, and `![t](@./t.log)`.
  - A proptest for the source/compose round trip.
  - A **corpus golden:** export notes with `sase bead list -s all -n 0 -f json` from the
    sase checkout. Commit only the deduplicated `@`-bearing note texts as a fixture,
    plus a golden list of the notes that would yield a path reference or diagnostic. Pin
    the count (research: about 0.3% of notes) and report it on the phase bead.
  - Tune the path-shape rule only if the golden shows avoidable false positives.
- **Pin bump:** move sase's `sase-core-revision.txt` past the sase-core commit
  (`docs/rust_backend.md`). `tools/check_sase_core_rs_bindings` must pass.

## Phase 2: `cas`

Create the package `src/sase/bead/attachments/`. Note that `sase.attachments` already
exists and is unrelated. This phase has no user-facing surface and needs no flag.

- **`LocalAttachmentStore`:** the layout above, `object_path`, `has`, `verify` (rehash),
  `remove`, and idempotent `materialize_view(sha, name)` (relative symlink).
- **`ingest_path(path, progress=None)` and `ingest_stream(fileobj, progress=None)`**
  return `IngestedBlob(sha256, size_bytes, head, object_path)`:
  1. Open once and `fstat`. Require a regular file, with a clear message for each
     refused kind. Check free space.
  2. Stream 1 MiB chunks into a 0600 temp file under `tmp/`, on the same filesystem,
     hashing as it goes and keeping the first 4 KiB. **All-zero chunks become holes**
     (`seek`, then a final `truncate`), so sparse sources stay sparse.
  3. Re-`fstat`, and fail with "changed while attaching" if the size, `mtime_ns`, or
     bytes read differ. Then `fsync`.
  4. Under a per-digest `flock`, install with `os.replace`, or discard the temp file if
     the object already exists. Set mode 0444 and `fsync` the directory.

  This is one read of the source, in bounded memory.

- **`probe_image(path)`:** a Pillow header-only open, bounded by the existing
  `MAX_CELL_IMAGE_PIXELS` / `MAX_CELL_IMAGE_FILE_BYTES` caps. Never SVG, never raises.
  Import Pillow lazily.
- **`BlobStore` protocol:** `name`, `describe()` (a destination/visibility label for the
  echo), `has`, `put(sha, src, size, progress)`, `get(sha, dest, progress)`, and
  `delete`. Include a `BlobStoreError(transient: bool)`.
- **Tests:**
  - Byte-exact round trips for random binary, NUL-heavy, and non-UTF-8 data.
  - A sparse fixture of at least 256 MiB apparent size, with bounded `tracemalloc` peak
    and sparse `st_blocks`.
  - Two concurrent ingests of the same bytes produce one object.
  - A mutated source is detected.
  - FIFOs and directories are refused.
  - stdin ingest works.
  - Views keep the extension, and objects are read-only.

## Phase 3: `core_wire`

Work in sase-core.

- **`BeadNoteAttachmentWire`** (plus the image dims struct) with `validate()`:
  - the name is already sanitized
  - the sha256 passes `validate_sha256`
  - `mime_type` is shaped `type/subtype`
  - image dims are greater than 0
  - names are unique within a manifest
- **Payload changes:**
  - `NoteAppended.attachments`
  - `NoteEdited.attachments: Option<Vec<…>>`
  - `BeadNoteWire.attachments`

  In `validate_for`, enforce a one-to-one match between tokens
  (`stored_attachment_tokens`) and the manifest whenever the manifest is non-empty.

- **Reducer** (`events/reduction.rs`) and write-side mirror
  (`mutation/notes_update.rs`): append sets the manifest, edit with `Some` replaces and
  with `None` keeps, remove drops the note. Thread the attachments through every
  `append_note_to_store` caller: close, +1 (both appends), and create, which passes
  empty.
- **Binding kwargs:**
  - `bead_append_note(..., attachments=None)`
  - `bead_note_edit(..., attachments=None)`
  - `bead_close(..., note_attachments=None)`
  - `bead_plus_one(..., note_attachments=None)`

  Existing positional calls keep working.

- **Queries:**
  - `bead_attachment_roster(issue)`: current notes only; the latest note wins per name;
    includes the note id and ordinal.
  - `bead_attachment_references(beads_dir)`: every current and historical
    `(issue, note, name, sha256, current)` from the event store. Used for pinning, purge
    preview, and doctor.
- **Pure policy functions:**
  - `attachment_placement(size_bytes, tiers, local_only)`
  - `attachment_should_auto_fetch(size_bytes, cap)`
  - `attachment_sensitive_path_reason(path, home, extra_patterns)`
- **`AttachmentTombstoneWire`:** `schema_version`, `sha256`, `purged_at`, `actor`,
  `reason`.
- **Fixtures and tests:**
  - Event round trip with attachments.
  - Existing fixtures stay byte-identical.
  - A forward-compatibility fixture proving an unknown extra field on a note payload
    still parses.
  - Validation failures: missing token, orphan descriptor, bad sha, duplicate names.
  - Reducer edit semantics for `None` versus `Some([])`.
- **Pin bump** in sase, as in phase 1.

## Phase 4: `note_cli`

After `just rust-install` (use `/sase_monitor` if it is long), build the whole authoring
path behind the flag.

1. **Flag.** Run `sase flag new bead_note_attachments -k beta` with:
   - `--when-enabled "@<path> references inside bead note text attach content-addressed file snapshots, and sase bead attach is available."`
   - `--when-disabled "Bead note text keeps today's behavior (only a whole-argument @<path> reads text from a file) and sase bead attach refuses with an enable hint."`
   - `--remove-when "The bead note attachments epic lands with sharing, image viewing, TUI support, and purge verified on athena."`

   Paste the printed registry entry and the schema mirror. Test both states everywhere
   below. Rendering, `attachment list`, and `attachment path` are **not** gated.

2. **Model.** Add `BeadNote.attachments` (a frozen dataclass descriptor tuple) to
   `src/sase/bead/model.py`, `note_codec.py`, and `notes_to_dicts`.
3. **Authoring service** (`src/sase/bead/attachments/authoring.py`). This is the single
   pipeline shared by the CLI and the TUI:
   1. Scan (core).
   2. Resolve and stat every path relative to the cwd, and apply the sensitive-path
      policy.
   3. Collect _all_ problems and raise one error, which renders caret diagnostics using
      display columns (`cell_len`).
   4. Ingest each unique path once.
   5. Classify, probe images, and add `origin` (`get_machine_name()`).
   6. For each target bead: uniquify names against its roster, then compose the stored
      text.

   It returns the stored text, the manifest, and echo rows. Nothing is written until
   every step succeeds.

4. **Source layer.** Add a flag-aware `read_note_text_value()` in `cli_file_values.py`,
   following decision 3: `@@` pass-through, UTF-8 and 256 KiB cap, the binary/oversize
   error with `attach` hint, and the TTY "read from" line. Use it for every note-bearing
   value. With the flag off, behavior is exactly today's.
5. **Wiring:**
   - `note`: append, and `--edit` using the roster for reuse; echo attachments detached
     by an edit.
   - `close -n`, `update -n` (multi-id: ingest once, compose per bead), and `+1 -n`.
   - Add `-S/--allow-sensitive` to all of them.
   - Leave `close -r`, `snooze -r`, and descriptions unchanged.
   - The TUI add-note modal (`action_beads_add_note`) calls the service when the flag is
     on, so TUI notes never store raw `@path` text. Surface errors via a notification;
     the full UX comes in `tui`.
6. **New commands:** `sase bead attach` (including stdin with `-N`), plus
   `sase bead attachment list` and `path`, following the CLI surface above.
7. **Rendering:**
   - Prose chips.
   - The per-note ATTACHMENTS block and the note-label `📎 N`.
   - The compact count and JSON fields.
   - `sase bead history` lists attachment descriptors on note events.

   Availability here is only `cached` or `unavailable`.

8. **Fast-path invariant test.** Every note-bearing argv containing `@` anywhere reaches
   the Python grammar when the flag is on, never the Rust arm, and never stores a raw
   `@path`.
9. **Help and docs:**
   - Replace the note help's "single-token `@<path>`" sentence with the mental model,
     and update `tests/main/test_parser_command_help.py`.
   - Add an **Attachments (beta)** section to `docs/beads.md`.
   - Add `docs/cli.md` rows.
10. **Tests (both flag states):**
    - Grammar through the CLI.
    - Every failure mutates nothing.
    - The source file can be deleted right after the command.
    - Multi-id update.
    - Edit/reuse/detach.
    - `+1` and close notes.
    - stdin attach.
    - read/show `-i never`/JSON output with no escape codes and no bytes.

## Phase 5: `shared_store`

- **Reserved sidecar role `attachments`:**
  - Add it to the schema enum and `RESERVED_SIDECAR_ROLES`, with default
    `visibility: private`.
  - It is never cloned into workspaces.
  - Its clone is a **bare partial clone** (`git clone --bare --filter=blob:none`) at
    `hidden_sidecar_clone_dir(project_key, "attachments")`, and it is shown by
    `sase repo list`.
  - `sase repo init` offers creation behind a default-no consent prompt that names the
    visibility (mirror `_confirm_agents_sidecar_creation`).
  - **Phases never create real GitHub repositories.** Tests use local bare remotes with
    `uploadpack.allowFilter=true`. Creating `sase-org/sase--attachments` is a user step
    through `sase repo init`.
- **`GitAttachmentStore` (BlobStore):**
  - **has:** looks up the cached remote-tracking tree. Refresh with at most one bounded
    `git fetch` per process, with a timeout.
  - **put:** `hash-object -w`, then a temp-index `read-tree` of the tip,
    `update-index --add --cacheinfo`, `write-tree`, `commit-tree`, and `push`. Retry on
    non-fast-forward; content-addressed paths never conflict. Commit messages carry no
    filenames.
  - **get:** `rev-parse <tip>:<relpath>`, then a streamed `cat-file blob` (a lazy
    promisor fetch) into a CAS temp file, digest-verified.
  - Serialize writers with a lock in the bare git dir. Reuse the prompt-archive
    publisher's validation and quarantine ideas; parameterize helpers instead of copying
    them.
- **Placement and `-L/--local-only`:** placement is core policy, as specified. Add `-L`
  to note, close, update, +1, and attach.
- **Upload timing:**
  - Add a pre-publication hook to `bead_store_mutation` (`src/sase/bead/cli_common.py`)
    so uploads run after the event commit, outside the lock, and before
    `_push_committed_bead_store`.
  - With `require_upload`, upload before the append instead.
  - The echo shows the destination, visibility, and timing.
- **Outbox:** `~/.sase/projects/<key>/attachment-upload-outbox.json` (flock plus atomic
  write, like `agents-publication-outbox.json`). Drain it:
  - at the start of `_run_locked_sync` in `bead/_sync_worker_run.py`, where failures
    never block bead publication
  - via `sase bead attachment push`
  - opportunistically, time-bounded, before attachment-writing commands

  `push` also promotes local-only objects once a store accepts them.

- **Fetch and availability:**
  - Fetch lazily for `read`, `show`, and `path`, capped by `auto_fetch_max_bytes`.
  - `-d/--download` on `show`/`read` lifts the cap.
  - Compute all availability states and render the badges.
  - `local_only` versus `pending_upload` is derived from `origin`, the outbox, and store
    presence.
- **`sase doctor` check:** store reachability and outbox backlog.
- **Tests:** a two-"machine" simulation with two SASE homes sharing one local bare
  remote.
  - Write on A, read on B: fetched, digest-verified, extension-preserving path.
  - A failed upload queues and later drains.
  - A size over every tier fails without `-L`.
  - `require_upload` aborts with nothing written.
  - Concurrent pushes from both homes.
  - A corrupt remote blob is detected.

## Phase 6: `large_files`

- **`RcloneAttachmentStore` (BlobStore)** for
  `bead.attachments.large_store: {remote, max_bytes}`:
  - Upload: `rclone copyto` to a `.partial-<uuid>` name, then `moveto`, then verify the
    size (and sha256 when the backend supports it).
  - `has`: `lsjson`. `get`: streamed `rclone cat` with digest verification. `delete`:
    `deletefile`.
  - Bounded timeouts, and stderr captured into errors. A missing `rclone` binary gives a
    clear error and a doctor finding.
- **Background uploads:** objects of at least `background_upload_min_bytes` are queued
  and drained by a detached worker (reuse the async bead-push launch pattern), so the
  command returns promptly. The echo says
  `⇡ uploading in background (1.8 GiB) — sase bead attachment push`, and `push` shows
  live progress.
- **Progress:** a Rich progress bar on stderr for ingest, upload, and download above 8
  MiB on a TTY; agents get only the final line.
- **Docs:** rclone setup for **SFTP to athena over the tailnet** (recommended: private,
  free, unlimited) and for Cloudflare R2, including per-machine `rclone.conf` notes.
- **Acceptance tests:**
  - A local-path rclone remote; skip when `rclone` is absent.
  - An end-to-end sparse multi-hundred-MiB attach → upload → fetch on a second home in
    bounded memory.
  - Tier boundaries at exactly `git_max_bytes`.
  - A background drain after a simulated crash.

## Phase 7: `show_images`

- **`-i/--images auto|cells|kitty|never`** on `show` only, plus `bead.show.images`
  (default `auto`). Modes:
  - `never`: no previews, identical to `read`.
  - `cells`: cell thumbnails via `CellImageRenderable`, rendered to an ANSI string with
    a Rich `Console`, sized from `shutil.get_terminal_size()`, at most 10 rows and 4 per
    note (the rest become chips), with the aspect ratio preserved.
  - `kitty`: real pixels through the kitty graphics protocol. It prints directly,
    bypassing the pager (which cannot host graphics), and falls back to `cells` with a
    one-time dim hint inside tmux or on terminals without kitty support. Reuse or expose
    the doctor's kitty detection instead of copying it.
  - `auto`: never when stdout is not a TTY, `NO_COLOR` is set, or `SASE_AGENT` is set.
    `kitty` when the output is printed directly on a kitty-capable terminal outside
    tmux. Otherwise `cells`.
- **Text previews:** up to 5 dim, sanitized lines for text attachments of 1 MiB or less
  whenever previews are on.
- **Pager links:**
  - Attachment chips become labeled `AttachedTarget`s resolving to
    `attachment:<bead-id>/<name>`.
  - Add a resolver branch in `src/sase/pager/resolve.py` that materializes the view and
    then reuses `link_target_for_existing_path`. Images, video, and PDF suspend into the
    existing viewer, with n/p across _all_ of the bead's viewable attachments. Text
    opens a sanitized pager section. Other binaries open the binary card.
  - `cli_pager` must accept attached handlers if the design needs them.
  - Add attachment icons to the label marker tables.
- **`sase bead attachment open <id> [<name>]`** and `attachment:` resolution for
  `sase artifact path|open|read`.
- **Tests:**
  - Mode resolution matrix (TTY, pipe, `NO_COLOR`, agent, tmux, kitty env).
  - Thumbnail rendering on a recorded `Console`.
  - Decode-failure card.
  - Kitty escape encoding (chunked base64 PNG, placement).
  - Pager resolution for each file class.
  - Pager visual snapshots updated via `just fix-tui-screenshots` (monitor), with the
    report inspected.

## Phase 8: `tui`

Read `tui.md`, `tui_perf.md`, and `tui_screenshot.md` via `/sase_memory_read` first.

- **Beads pane note detail:** an attachments block per note with chips, metadata,
  badges, and cell thumbnails. If the Markdown body cannot host renderables, add a
  compact attachments strip under the note. Fetching and decoding run off the event
  loop, with cached results.
- **New keymap `beads_open_attachments`:** pick a free key in the Beads scope, and
  update `default_config.yml` plus bindings, keymap metadata, the help modal, command
  metadata, and availability. It opens the viewer via `suspend_for_external_tool` +
  `view_artifact_files`.
- **Add-note modal (`N`):**
  - `@` path completion by reusing `FileCompletionMixin`.
  - Pasting a single existing file path (drag and drop) inserts `@"<path>"`.
  - Save runs the authoring service off-thread. Diagnostics show inline with carets, and
    the typed text is never lost.
  - A success toast mirrors the write echo.
- **Tests:** widget and unit tests, plus visual snapshots via `just fix-tui-screenshots`
  (monitor) with the report and golden diff inspected.

## Phase 9: `lifecycle`

- **`sase bead attachment purge <id> <name> -r WHY [-y]`:**
  1. Preview every note in the project that references the digest (core references
     query) and confirm on a TTY.
  2. Write a tombstone to every configured store. For git, add the tombstone and remove
     the object from the tip in one plumbing commit. For rclone, write the tombstone and
     `deletefile`.
  3. Delete the local object and its views, and record a local tombstone.

  Notes render `(purged)`, and fetches refuse tombstoned digests. No bead event changes.
  Print the exact documented `git filter-repo` procedure for erasing history from the
  private repo; do not automate it.

- **`sase bead doctor` attachments check** (choose a free short alias) with a confirmed
  `--fix-attachments` repair. Report:
  - token/manifest mismatches
  - dangling descriptors (no bytes anywhere)
  - pending outbox items
  - local-only objects
  - tombstoned references
  - local digest mismatches
  - orphan local objects (unreferenced and older than 7 days)

  The repair removes orphans, re-drains the outbox, and quarantines corrupt objects.

- **`sase bead attachment prune [-y]`:** a dry-run plan by default. It evicts cached
  objects confirmed present in a shared store, oldest views first, to fit under
  `bead.attachments.local_cache_max_bytes`. It never evicts pending or local-only
  objects.
- **Bead pages** (public): render
  `🔒 login.png · image/png · 184 KiB (private attachment)` with no paths or links.
  Refresh the page renderer tests.
- **Tests:** purge across all stores and homes, the doctor matrix, prune safety, and
  page rendering.

## Phase 10: `ga`

- **Remove the flag.** Delete every Off branch and make the On branch unconditional.
  Remove the registry entry and schema mirror, delete the flag-off tests, and close the
  flag bead with a note.
- **Docs.**
  - `docs/beads.md` Attachments section (drop "beta"):
    - the mental model
    - the grammar table
    - commands
    - storage tiers and privacy
    - viewing
    - JSON
    - troubleshooting badges
    - mixed-fleet upgrade guidance
  - `docs/configuration.md`, `docs/cli.md`, and the `sase bead onboard` quick start.
- **Skill source.** Update `src/sase/xprompts/skills/sase_new_task.md` to show attaching
  repro evidence (`sase bead +1 <id> -n "… @./repro.png"`). Edit the source template
  only; deployment happens after landing, per `generated_skills.md`.
- **Acceptance sweep.** Run it on athena in a scratch bead store (never the real project
  store), and record the results on the phase bead: every acceptance criterion below,
  plus a real `show` screenshot in cells and kitty modes.
- **Record `PROPOSED FOLLOW-UP:` notes** on the phase bead (phase workers never create
  beads):
  1. A **memory** update: `sase_beads.md` "Notes And History" and the init template
     `memory-sase-beads.template.md` should mention `@path` attachments, `@@`, and
     `sase bead attach`.
  2. Attachments in descriptions and `create`.
  3. `@attachment:<bead>/<name>` prompt expansion and cross-bead reuse.
  4. Folding `sase artifact create --bead` into attachments.
  5. Gate/Telegram rendering and photo delivery.
  6. sase-nvim highlighting of the grammar.
  7. iTerm2/sixel protocols if a user needs them.

## Cross-cutting requirements

- Every Python phase runs `sase tool run check`, not raw `just check`, and runs
  `just fix` first. Keep modules under the `toobig` limit by splitting early.
- Core phases run the wrapped `check` inside the sase-core checkout.
- Tests cover both flag states in every phase from `note_cli` until `ga`.
- **Performance:**
  - `show` and `read` of beads without attachments do no attachment work at all.
  - Pillow and git imports are lazy.
  - TUI work stays off the event loop.
- **Security:**
  - Sensitive-path refusal and private-by-default storage.
  - Decoding is bounded, with no automatic SVG or EPS decoding.
  - Sanitized display names and no raw attachment text in the terminal.
  - Descriptors never contain paths.
- **Mixed fleet:** enable the flag only after athena, apollo, and mac run a build with
  `core_wire`. Otherwise an older writer regenerates `issues.jsonl` without manifests.
  The event store remains authoritative, and `sase bead doctor --fix-projection` repairs
  the projection. Document this in the beta docs.

## Epic acceptance criteria

- Prose with any number of inline, quoted, reused, and escaped references round-trips.
  `@@` and the diagnostics behave identically in core, CLI, and TUI tests.
- Files are captured byte-exact regardless of content (non-UTF-8, NUL-filled, sparse
  multi-hundred-MiB), in bounded memory. The source can be deleted immediately
  afterward.
- A failed attachment command mutates nothing, and with `require_upload` it uploads
  nothing.
- A note written on one home is readable on another, with verified bytes, or shows an
  honest badge for every availability state.
- `read`, JSON, and piped `show` contain no image data, escape codes, or file contents.
  `show` previews degrade from kitty to cells to cards to chips without ever failing.
- Existing events and projections are byte-identical, and legacy notes render unchanged.
