---
tier: epic
title: Fuzzy artifact-reference completion with matched-run highlighting
goal: 'Typing an artifact reference finds the file by any memorable fragment of its
  path or title in both the ACE prompt input and external editors, every candidate
  a reference can name is actually reachable, and every row shows exactly which characters
  the query matched.

  '
phases:
- id: fuzzy
  title: Canonical fuzzy matcher in sase-core
  depends_on: []
  size: medium
  description: 'fuzzy: add the shared tier/score/runs fuzzy matcher and its ordering
    comparator to crates/sase_core/src/editor/fuzzy.rs, with the three-pass alignment
    that makes highlights land on the segment the user meant.

    '
- id: docwalk
  title: Bundled document discovery depth
  depends_on: []
  size: small
  description: 'docwalk: teach plan/read.rs to discover markdown inside bundle directories
    one level below a month shard, skipping dot-directories and the prompts/specs
    trees the prompt corpus owns.

    '
- id: menu
  title: Fuzzy at-reference menu and match runs on the wire
  depends_on:
  - fuzzy
  size: medium
  description: 'menu: rewire build_at_reference_menu onto the fuzzy matcher for kinds,
    payloads, and the path partial; make payload rows carry the payload as their label
    plus title, match runs, and tier; keep empty-query order and prefix-only Tab extension.

    '
- id: binding
  title: Zero-marshalling payload index binding
  depends_on:
  - menu
  size: medium
  description: 'binding: add the opaque AtReferenceInventory pyclass built once off-thread,
    let at_reference_menu accept it instead of re-marshalling every payload row per
    keystroke, and expose the fuzzy matcher to Python.

    '
- id: lsp
  title: Server-side fuzzy completion for editors
  depends_on:
  - menu
  size: small
  description: 'lsp: return an incomplete list with filter text equal to the typed
    reference so clients keep server-ranked fuzzy rows, and put the matched runs and
    title into item documentation and label details.

    '
- id: catalog
  title: Reachable, bounded, per-kind payload catalogs
  depends_on:
  - docwalk
  - binding
  size: medium
  description: 'catalog: load document payloads per document role with per-role caps
    instead of one shared 500-row cap, drop the prompt corpus from the warm, and memoize
    the payload index plus row metadata per project and kind so keystrokes do no O(rows)
    Python work.

    '
- id: tui
  title: Prompt-input rendering of paths and matched runs
  depends_on:
  - catalog
  size: medium
  description: 'tui: render payload rows as the reference path with dim directories,
    a bright basename, and gold matched runs, add the shared highlight helper, and
    report match counts, fuzzy mode, and unscanned rows in the panel subtitle.

    '
- id: finder
  title: Ctrl+R finder on the shared matcher
  depends_on:
  - tui
  size: small
  description: 'finder: replace the duplicate Python fuzzy scorer in recursive_file_finder
    with the shared core matcher and the shared highlight helper so both surfaces
    rank and highlight identically.

    '
- id: land
  title: Docs, core floor bump, and end-to-end verification
  depends_on:
  - lsp
  - finder
  size: small
  description: 'land: update the editor and ace docs plus the sase-nvim completion
    table, raise the published sase-core-rs floor, and run the acceptance and performance
    verification listed for the epic.

    '
create_time: 2026-07-30 04:18:13
status: wip
---

# Plan: Fuzzy Artifact-Reference Completion With Matched-Run Highlighting

## Problem

Artifact-reference completion is a memory test. Both stages filter with a case-insensitive `starts_with`, so the user
must recall how a payload _begins_ — which, for documents, means recalling a `YYYYMM` shard directory:

`crates/sase_core/src/editor/at_reference.rs`

```rust
// build_kind_menu
.filter(|row| row.kind.to_lowercase().starts_with(&query))
// build_payload_menu
.filter(|row| {
    row.payload.to_lowercase().starts_with(&query)
        || row.label.to_lowercase().starts_with(&query)
})
```

The user's example is the whole indictment. In this repo, `@research:site` should find
`202607/sase_sites_hub_and_pages/sase_sites_hub_and_pages.md`. Verified today (probed against the real research sidecar
through `load_artifact_ref_completion_catalog`):

1. **`@research:site` returns zero rows.** No payload starts with `site`.
2. **That file is not in the catalog at all**, so no amount of matching could surface it. `read_document_corpus` scans
   only `<root>/*.md` and `<root>/<shard>/*.md`; the file lives in a bundle directory at
   `<root>/202607/sase_sites_hub_and_pages/`. 42 of the research sidecar's 305 markdown files are invisible for the same
   reason.
3. **Even the flat files are half missing.** `_load_document_candidates` asks for `limit=500` across _every_ document
   corpus at once and, because it passes `kinds=None`, also pulls in the 2809-file prompt corpora. The repo has 4598
   discoverable documents; 500 survive the cap, 369 survive dedup, and research gets 193 of its 263 flat files.
   Silently. `202607/github_sdd_repo_consolidated.md` is simply unreachable by completion.
4. **The rows never show the path.** Payload rows set `label` to the document _title_, so the panel renders
   `[D] Prior Art: Splitting Pluggy Plugins into Separate Repos/Packages  research · now` while inserting
   `@research:202602/pluggy_repo_separation.md`. There is no path on screen to highlight a match in, and what you see is
   not what you get.

So "make it fuzzy" is necessary but not sufficient. Fuzzy matching over a catalog that is missing the file, capped at an
arbitrary 500 rows, and displaying something other than what it inserts would be a worse kind of broken: it would _look_
like it searched everything.

There is one more constraint that shapes the whole design. Measured on this machine, the `at_reference_menu` binding
costs **~3.5 µs per payload row per call** because the pyo3 layer round-trips every row through `serde_json::Value`:

```
n=  500 rows   1.8 ms/call        n= 4500 rows  17.2 ms/call
n= 2000 rows   7.3 ms/call        n=10000 rows  35.9 ms/call
```

Every keystroke in a reference payload rebuilds that inventory. The 500-row cap is not a product decision, it is a
performance workaround — and `@plans:` legitimately has 4336 rows in this repo. Raising the cap without fixing the wire
would put a 17 ms hitch on every keystroke, blowing the p95 < 16 ms key-to-paint budget from `memory/tui_perf.md`.

## Goals

1. `@research:site` lists `202607/sase_sites_hub_and_pages/sase_sites_hub_and_pages.md`, in the ACE prompt input and in
   an editor over LSP.
2. Every row shows the matched characters, in the path and in the title, unmistakably.
3. Every document, chat, artifact file, bead, and agent a reference can _resolve_ is a candidate the completion can
   _find_; where a bound still applies, the UI says so.
4. Prefix behavior is preserved exactly. `@c` still lists `commit`, `chat`; `@pl` + Tab still extends to `@plans:`;
   `@src/` still navigates the filesystem.
5. Keystroke cost stays inside the TUI budget with thousands of candidates.

## Non-Goals

- Making every other completion provider fuzzy (`#+`, `#gh:`, directives, xprompt args stay prefix matched). This epic
  is the `@` menu and the Ctrl+R finder, which already is fuzzy.
- Remote or network-backed enumeration. `commit` and `bug` payloads keep their existing shape-validation-only treatment.
- Changing artifact-reference _resolution_. Nested payloads already resolve exactly today; only discovery is broken.
- Reworking the `age` column (see "Observed adjacent issues").

## UX Specification

### Behavior

Typing narrows with a **tiered** match. The tier is the product decision that keeps this intuitive: a fuzzy hit never
outranks a literal one.

| Tier | Meaning                                   | Example against `202607/sase_sites_hub_and_pages/…` |
| ---- | ----------------------------------------- | --------------------------------------------------- |
| 0    | query is a prefix of the primary text     | `202607/`                                           |
| 1    | query is a prefix of the basename segment | `sase_sites`                                        |
| 2    | query is a contiguous substring           | `hub_and`                                           |
| 3    | query is an ordered subsequence           | `site`, `shubp`                                     |

Rows sort by tier, then by score, then by shorter text, then case-insensitively — deterministic at every level. With an
**empty** query nothing is ranked at all: the provider's order (recency for chats and commits, builtin order for kinds,
directories-before-files for paths) is preserved byte-for-byte, so opening the menu looks exactly as it does today.

Consequences that are worth stating because they are the intuitive part:

- `@c` → `commit`, `chat` (both tier 0, tie broken by the existing builtin rank). Unchanged.
- `@rsch` → `research` (tier 3). New.
- Tab still extends the token to the shared prefix — but **only when every row in the leading group is tier 0**.
  Extending a fuzzy match set is meaningless, so once the query is fuzzy-only the shared extension is empty and Tab
  falls through to the existing "unique match accepts" path. That is a genuine win: `@research:site` narrowing to one
  row means Tab accepts it.
- Directory navigation in the `@` path group stays exact. Only the trailing partial is fuzzy, so `@src/` still lists
  `src/` and `@src/fcb` can find `_file_completion_base.py`.

### Rendering (ACE prompt input)

Payload rows change from title-first to **path-first**, because the path is what gets inserted and what the query
matches:

```
▸ [D] 202607/sase_sites_hub_and_pages/sase_sites_hub_and_pages.md   SASE Sites Hub and Pages · research · 3d
      └── dim ────────┘└──── bold basename ─────┘                   └──────────── dim tail ─────────────┘
          with gold matched runs anywhere in the path
```

Styles are taken verbatim from the Ctrl+R finder modal so the two surfaces read as one system:

| Element           | Style          | Source                 |
| ----------------- | -------------- | ---------------------- |
| matched runs      | `bold #FFD700` | `_MATCH_STYLE`         |
| directory portion | `dim`          | `_DIR_PORTION_STYLE`   |
| file basename     | `bold`         | `_FILE_BASENAME_STYLE` |
| directory row     | `bold #87D7FF` | `_DIR_STYLE`           |

The dim tail (`title · detail · age`) is truncated with the existing `truncate_cell` helper against the remaining panel
width so the path never wraps or elides. When the query matched the **title** rather than the path, the title's own runs
are gold too — the user always sees why a row is there.

Kind rows and local-path rows keep their current layout and gain the same gold runs.

The panel subtitle becomes honest about what it searched:

- `~ fuzzy · 3 of 305` when any visible row is tier ≥ 2.
- `⚠ 1203 not scanned` appended when a provider cap truncated the candidate set.

`memory/tui_perf.md` rule 8 and the "no silent caps" principle both point the same way: a cap the user cannot see reads
as "I searched everything and it isn't there".

### External editors (LSP)

Editors cannot render per-character highlights inside a completion label, so the affordance is different but equivalent
in information:

- `labelDetails.description` keeps the group word; `labelDetails.detail` gains the title.
- `documentation` (markdown) shows the payload with matched runs wrapped in `**`, then the title. That is the "what
  matched" view in the client's preview window.

The hard part is _survival_, not display. Verified against the installed Neovim 0.12.4 runtime
(`lua/vim/lsp/completion.lua`):

```lua
matches = function(item)
  if item.filterText then
    return match_item_by_value(item.filterText, prefix)   -- startswith(filterText, prefix)
  end
  ...
end
```

A client that prefix-filters `@research:202607/…` against the typed `@research:site` throws every fuzzy row away. So the
server sets **`filterText` to the typed reference text itself** (`@` + kind + `:` + query, or `@` + query in the kind
stage), which every prefix or fuzzy client trivially keeps, and relies on `sortText` for rank. The response becomes
`CompletionList { isIncomplete: true }`, which matters twice in that same file: line 961 makes Neovim re-request on each
keystroke instead of filtering a stale list, and line 473 disables the client's own fuzzy re-sort so the server's
ranking survives. Insertion is unaffected — non-snippet items take `word` from `textEdit.newText`.

## Architecture

The matcher and the menu are shared backend behavior: the TUI and the LSP must rank identically, and a future web or CLI
frontend must too. Per `memory/rust_core_backend_boundary`, both live in `sase-core`, and Python keeps only
presentation.

```
sase-core
  crates/sase_core/src/editor/fuzzy.rs        NEW  tier + score + runs, ordering comparator
  crates/sase_core/src/editor/at_reference.rs      menu uses the matcher; rows carry runs
  crates/sase_core/src/plan/read.rs                bundle-depth document discovery
  crates/sase_core_py/src/lib.rs                   AtReferenceInventory pyclass; fuzzy binding
  crates/sase_xprompt_lsp/src/{server,lsp_convert}.rs   incomplete list, filterText, docs

sase
  src/sase/ace/tui/widgets/artifact_ref_completion.py           per-kind catalogs + indexes
  src/sase/ace/tui/widgets/_completion_match_highlight.py  NEW  shared run renderer
  src/sase/ace/tui/widgets/_prompt_input_bar_completion_rows_artifacts.py  path-first rows
  src/sase/ace/tui/widgets/_prompt_input_bar_completion_panel.py           subtitle
  src/sase/ace/tui/widgets/recursive_file_finder.py             delegates to the core matcher
```

There is deliberately **one** matcher. The duplicate scorer in `recursive_file_finder.py` is ported to Rust with its
existing constants intact and then deleted, so the Ctrl+R finder and the `@` menu cannot drift.

## Detailed Design

### 1. The matcher (`editor/fuzzy.rs`)

```rust
pub struct FuzzyMatch {
    pub tier: u8,                    // 0 prefix, 1 basename prefix, 2 substring, 3 subsequence
    pub score: i32,
    pub runs: Vec<(u32, u32)>,       // half-open CHAR ranges into the matched text
}
pub fn fuzzy_match(query: &str, text: &str) -> Option<FuzzyMatch>;
pub fn compare_fuzzy(left: (&FuzzyMatch, &str), right: (&FuzzyMatch, &str)) -> Ordering;
```

Runs, not individual indices: they are compact on the wire and let the renderer emit one Rich span per run instead of
one per character.

Char ranges, not byte ranges: Python strings and Rich `Text` are character-indexed, and every consumer of `runs` is a
renderer.

**Alignment — three bounded passes.** A single greedy left-to-right scan produces ugly highlights. For `site` against
`202607/sase_sites_hub_and_pages/…` it lights up the `s` of `sase` plus `ite` of `sites`. So:

- **Pass A (forward).** Find the earliest end index `e` such that the query is a subsequence of `text[..=e]`.
- **Pass B (backward tightening).** Match the query in reverse from `e` to get the tightest alignment ending at `e`. For
  the example this collapses to the contiguous `site` inside `sites` — the run the user meant.
- **Pass C (basename retry).** If the alignment starts before the basename and the query is also a subsequence of the
  basename, redo A+B restricted to the basename and keep the higher-scoring alignment. This is what makes "I typed the
  file name" always highlight the file name.

Each pass is O(len(text)); the whole thing is allocation-light and deterministic.

**Scoring** reuses the constants already proven in `recursive_file_finder.py`, so migrating the finder in the `finder`
phase is behavior-preserving:

| Constant              | Value | Meaning                                                                      |
| --------------------- | ----- | ---------------------------------------------------------------------------- |
| `SCORE_MATCH`         | 16    | per matched character                                                        |
| `BONUS_BOUNDARY`      | 14    | character begins a path segment or word (`/_-. `, or a camelCase transition) |
| `BONUS_CONSECUTIVE`   | 10    | immediately follows the previous match                                       |
| `BONUS_BASENAME`      | 10    | falls inside the basename                                                    |
| `MAX_LEADING_PENALTY` | 10    | bounded penalty for a late first match                                       |

An empty query returns `Some(FuzzyMatch { tier: 0, score: 0, runs: vec![] })` so callers can treat "browse" uniformly
without special cases.

Tests: tier classification for every tier; the `site` alignment collapsing to a contiguous run; the basename retry
beating a cross-directory alignment; camelCase and separator boundary bonuses; non-ASCII text producing char ranges that
slice correctly; empty query; no-match; determinism under shuffled input.

### 2. The menu (`editor/at_reference.rs`)

`AtReferenceRowWire` gains four fields (additive; the Python reader already uses tolerant `.get()` access and the LSP is
in-process):

```rust
pub struct AtReferenceRowWire {
    pub group: AtReferenceGroup,
    pub label: String,          // kind name | file name | PAYLOAD  (was: document title)
    pub title: String,          // NEW  document title, commit subject, bug/bead title; else ""
    pub insertion: String,
    pub is_dir: bool,
    pub detail: String,
    pub builtin: bool,
    pub label_match: Vec<(u32, u32)>,   // NEW
    pub title_match: Vec<(u32, u32)>,   // NEW
    pub match_tier: u8,                 // NEW
}
```

Moving the payload into `label` is the wire half of the path-first decision. The LSP already labels items from
`insertion`, so its labels are unaffected.

- `build_kind_menu`: replace the `starts_with` filter with `fuzzy_match` over the kind name. Empty query keeps
  `compare_kind_rows` exactly. Non-empty query sorts by `compare_fuzzy` with the builtin rank as the final tiebreak, so
  `@c` is byte-identical to today.
- Path group: keep `path_query.directory` exact and the `show_hidden` rule; fuzzy-match only `path_query.partial`.
  Directories still sort before files, then tier, then score, then name.
- `build_payload_menu`: match against the payload _and_ the title, include the row if either matches, rank by the better
  of the two with the payload winning ties (it is what gets inserted), and report the runs for whichever fields matched.
- `shared_row_extension`: return `""` unless every row in the leading group is `match_tier == 0`.
- `AtReferenceMenuWire` gains `payload_count` beside the existing `artifact_count` / `file_count`, plus
  `truncated_payloads: usize` for rows the caller declared it never scanned.

Tests: `@research:site` fixture surfacing the bundled payload with the expected runs; tier ordering; empty-query order
preserved (the existing `payload_menu_preserves_order_and_matches_payload_or_label` test must still pass unchanged in
its empty-query form); title-only matches; shared extension present for a pure-prefix set and absent as soon as one
fuzzy row appears; group caps and pre-cap counts.

### 3. Document discovery depth (`plan/read.rs`)

`read_document_corpus` recurses one level further, into bundle directories below a shard:

```
<root>/*.md                       unchanged
<root>/<shard>/*.md               unchanged
<root>/<shard>/<bundle>/*.md      NEW
```

Skip dot-directories and the `prompts` and `specs` directories, which the prompt corpus owns. That skip is not cosmetic:
`sase/repos/plans/*/prompts/` holds **2809** markdown files that would otherwise be duplicated into every `@plans:`
menu. Verified shape of the real sidecars: research bundles are exactly one level deep, and the only other deep
directory in plans is `perf_artifacts` (one stray markdown file, harmless either way).

Ordering stays deterministic and shallow-first: flat files, then shard files, then bundle files, each `read_dir_sorted`.

Tests: a bundle fixture appearing with its full relative path; `prompts`/`specs`/dot-directories excluded; a fourth
level not descended into; ordering pinned. Also confirm `sase plan search` and the Plans/Research artifact tabs simply
gain the bundle rows — this is the same fix for them.

### 4. The payload index binding (`sase_core_py`)

The keystroke path must stop marshalling rows. Add an opaque, immutable, frozen pyclass:

```python
index = sase_core_rs.AtReferenceInventory(payloads=[{...}, ...])   # built ONCE, off-thread
menu = at_reference_menu(context, inventory, payload_index=index)  # per keystroke
```

`at_reference_menu` keeps its `(context, inventory)` dict signature for tests and existing callers; when `payload_index`
is supplied it replaces `inventory["payloads"]`. Only payloads get a handle: kinds are a handful of rows and the path
group is bounded by one directory listing, and the two are never large at the same time (paths only exist in the kind
stage, payloads only in the payload stage).

The handle precomputes, per row, the lowercased character vector and the basename start offset, so per-keystroke work is
native scoring with no allocation of match keys. Also expose `fuzzy_match(query, text) -> dict | None` for the `finder`
phase.

Verification gate for this phase: a bench asserting `at_reference_menu` with a 5000-row payload index stays under **8
ms** per call (half the 16 ms key-to-paint budget), versus the 17 ms measured today at 4500 dict rows.

### 5. LSP (`sase_xprompt_lsp`)

`at_reference_completion_response` returns `CompletionResponse::List(CompletionList { is_incomplete: true, items })`.
Per item: `filterText` = the typed reference text reconstructed from the context (`format!("@{kind}:{query}")` in the
payload stage, `format!("@{query}")` in the kind stage); `sortText` unchanged (`{group}:{index:04}`);
`labelDetails.detail` = `row.title`; `documentation` = markdown with the payload's matched runs bolded and the title on
a second line.

Tests in `crates/sase_xprompt_lsp`: a fuzzy payload query returns the bundled row; `isIncomplete` is true; every item's
`filterText` equals the typed text; documentation bolding matches `label_match`; `textEdit.newText` still carries the
full reference.

### 6. Python catalog (`artifact_ref_completion.py`)

Three separable fixes:

**Per-role loading with per-role caps.** Replace the single all-corpora `search(limit=500)` with one call per document
role. Pass explicit `kinds` so `_kind_selected(kinds, "prompt")` is false — that alone drops the 2809-file prompt scan
and takes the document warm from ~400 ms to ~230 ms (measured). Per-role cap `_MAX_DOCUMENT_ROWS_PER_KIND` at 5000, and
record the pre-cap count so the panel can report truncation. Research (263 rows, 21 ms) can no longer be starved by
plans (4336 rows).

Note the role/kind subtlety: the plans corpus reports frontmatter kinds (`tale`, `epic`, `plans`) which
`_document_role_for_path` maps back to the `plans` role, so per-role loading must scope by _corpus_, not by a
`kinds=[role]` filter.

**Memoized indexes.** `ArtifactRefCompletionCatalog` gains, keyed by casefolded kind, the `AtReferenceInventory` handle
and the `payload -> ArtifactRefPayloadCompletionMetadata` map. Both are built inside
`load_artifact_ref_completion_catalog`, which already runs off the UI thread in the `prompt-artifact-ref-kinds` worker.
`_payload_inventory` becomes a dictionary lookup — today it rebuilds 4336 dataclasses and 4336 dict entries per
keystroke. Commit and bug snapshots keep their current per-call construction but memoize by snapshot object identity.

**Honest caps.** Raise `_MAX_CHAT_ROWS` and `_MAX_ARTIFACT_FILE_ROWS` alongside the document cap and thread every
pre-cap count into `ArtifactRefCompletionResult` so the panel can render `⚠ N not scanned`.

Tests: a bundled document reachable through `@research:site` end to end from a fixture corpus; per-role caps not
starving a small role; prompt corpora absent from the warm; the index/metadata maps built once (assert the memo is
reused across two `build_artifact_ref_completion_result` calls); truncation counts propagated.

### 7. TUI rendering

New `_completion_match_highlight.py`:

```python
def append_highlighted(
    content: Text, text: str, runs: Sequence[tuple[int, int]],
    *, base_style: str, match_style: str = "bold #FFD700",
    segment_split: int = 0, dim_style: str = "dim",
) -> None
```

One `content.append` per run boundary, not per character. `segment_split` is the basename start, so the same helper
renders the dim-directory / bright-basename split used by both the `@` menu and the finder.

`append_artifact_ref_completion_row` renders payload rows path-first per the UX section, kind rows and file rows with
runs, and truncates the dim tail to the remaining width. Panel subtitle work goes next to `_model_completion_subtitle`,
which is the existing precedent for a computed subtitle.

Tab and accept semantics are untouched: `_file_completion_refresh.py` already auto-accepts when
`at_reference_leading_match_count == 1` and extends only when `shared_extension` is non-empty, and the core now
guarantees the latter is prefix-only.

Tests: widget tests for path-first rendering, run styling on path and title, tail truncation, kind and file run styling,
and the two subtitle forms; PNG snapshots for a fuzzy payload menu and a truncated menu (`just test-visual`; goldens
under `tests/ace/tui/visual/snapshots/png/`).

### 8. Finder migration

`recursive_file_finder.py` drops `_fuzzy_match`, `_score_positions`, `_is_boundary`, and the scoring constants and
delegates to the core binding; `FuzzyMatch` becomes a thin adapter carrying runs. `recursive_finder_modal.py` renders
through `append_highlighted`. Because the ported constants are identical,
`tests/ace/tui/widgets/test_recursive_file_finder.py` and the finder PNG snapshot must pass unchanged — that is the
migration's correctness gate. Any snapshot churn here means the port changed behavior and must be fixed, not accepted.

## Phases

Each phase is one agent instance. Phases in a linked repo must open it first with `sase repo open <repo> -r "<reason>"`
and use the printed path as the only path for reads and writes.

Phases 1–5 are in `sase-core` and **must be committed and pushed** before the `sase`-repo phases start: `sase repo open`
resets the linked checkout to `origin/master`, and `just install` builds `sase-core-rs` from that checkout. A dependent
phase cannot see an unpushed core change.

### Phase `fuzzy` — canonical matcher (repo: `sase-core`)

Section 1. New `crates/sase_core/src/editor/fuzzy.rs` exported through `editor/mod.rs` and `lib.rs`; no existing caller
changes. Verify: `cargo fmt --all -- --check`, `cargo clippy --workspace --all-targets -- -D warnings`,
`cargo test --workspace`.

### Phase `docwalk` — bundle discovery (repo: `sase-core`)

Section 3. Touches only `plan/read.rs`, so it may run in parallel with `fuzzy` and `menu`. Same verification commands.

### Phase `menu` — fuzzy menu and match runs (repo: `sase-core`; after `fuzzy`)

Section 2. Rewire both stage builders, extend the two wire structs, retier the ordering, restrict the shared extension.
Same verification commands.

### Phase `binding` — payload index (repo: `sase-core`; after `menu`)

Section 4, including the module docstring at the top of `crates/sase_core_py/src/lib.rs` that lists the `at_reference_*`
surface, and the 5000-row / 8 ms bench gate. Same verification commands.

### Phase `lsp` — editor completion (repo: `sase-core`; after `menu`)

Section 5. May run in parallel with `binding`. Same verification commands.

### Phase `catalog` — reachable payload catalogs (repo: `sase`; after `docwalk`, `binding`)

Section 6. **Read `memory/tui_perf.md` first** — rules 1, 8, and 11 all bear on this phase. Verify: `just install`
(builds the pushed core from the linked checkout), then `just check`. Acceptance probe: with the real research sidecar
opened via `sase repo open research`, assert `build_artifact_ref_completion_result` for `@research:site` contains
`202607/sase_sites_hub_and_pages/sase_sites_hub_and_pages.md`.

### Phase `tui` — rendering (repo: `sase`; after `catalog`)

Section 7. **Read `memory/tui_perf.md` first.** Verify: `just install`, `just check`, `just test-visual`. Manual:
`sase ace`, open the prompt input, type `@research:site` and confirm the row, the gold runs on `site`, the dim directory
prefix, and the subtitle.

### Phase `finder` — shared matcher for Ctrl+R (repo: `sase`; after `tui`)

Section 8. Verify: `just install`, `just check`, `just test-visual` with the finder snapshot unchanged.

### Phase `land` — docs, floor, verification (repos: `sase`, `sase-nvim`; after `lsp`, `finder`)

- `docs/editor.md`: rewrite the "Artifact references" completion-table row and the "Artifact assistance is local-only"
  paragraph, which currently promises prefix filtering ("editors filtering the typed word keep both groups visible");
  document the fuzzy tiers, `filterText`/`isIncomplete` contract, and the documentation-based match preview.
- `docs/ace.md`: document the fuzzy `@` menu, the path-first rows, and the subtitle indicators.
- `CHANGELOG.md` entry.
- `sase-nvim`: add the fuzzy artifact-reference row to the README completion table; extend `tests/lsp_*_smoke.lua`
  coverage only if the Rust-side LSP tests leave a client-observable gap.
- Raise `sase-core-rs>=` in `pyproject.toml` to the release-plz-published version containing phases 1–5. release-plz
  owns sase-core versions; do not hand-edit them there.
- Final verification: `just install`, `just check`, `just test-visual`; the `@research:site` acceptance check in
  `sase ace`; the same query in Neovim against a freshly built LSP binary via `SASE_XPROMPT_LSP_CMD`; and the keystroke
  bench confirming p95 < 16 ms with `@plans:` open.

## Risks & Mitigations

- **Fuzzy noise drowning literal matches.** The tier system, not score tuning, is the mitigation: a subsequence match
  can never outrank a prefix match regardless of scores. Empty-query order is untouched, so the common "open and browse"
  path cannot regress at all.
- **Keystroke latency.** The measured 3.5 µs/row marshalling cost is removed rather than worked around, and the
  `binding` phase carries an explicit 8 ms-at-5000-rows gate. If the gate fails, the fallback is a per-role cap
  reduction with the truncation notice already in the UI — degraded reach, never a slow TUI.
- **Tab regressing into nonsense.** `shared_row_extension` is prefix-only by construction and gets a test asserting it
  empties as soon as one fuzzy row joins the leading group.
- **Editors dropping server-ranked rows.** Mitigated by `filterText` = typed text plus `isIncomplete`, both verified
  against the Neovim 0.12.4 runtime source rather than assumed, and pinned by LSP tests.
- **Bundle recursion exploding the plans menu.** The `prompts`/`specs` skip is the whole point; without it 2809 prompt
  files enter `@plans:`. It gets a dedicated test.
- **Path-first rows losing the title affordance.** The title stays on the row (dim, after the path) and is still
  matchable and highlightable, so nothing is lost — the path is simply promoted to primary because it is what gets
  inserted.
- **Ranking drift between the finder and the `@` menu.** There is one matcher; the finder's existing tests and PNG
  snapshot are the port's correctness gate.

## Observed Adjacent Issues (out of scope)

- **The `age` column is meaningless for documents in ephemeral workspaces.** 184 of 253 flat research files have no
  frontmatter, so `created_at` falls back to file mtime — which in a freshly cloned sidecar is the clone time, rendering
  every row as `now`. Verified. A month-shard-derived fallback (`202607/` → `2026-07`) would fix it, but `created_at`
  also drives `sase plan search` sorting and the Plans/Research tabs, so it deserves its own change. Ranking here is
  score-based for non-empty queries and provider-ordered for empty ones, so this does not affect the feature.
- Chat, bead, and agent payload rows would benefit from the same path-first treatment audit; this epic renders them
  through the shared helper but does not redesign their labels.
