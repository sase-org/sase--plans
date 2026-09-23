---
tier: epic
title: Tribe clan summaries and durable clan records
goal: "Selecting an agent tribe panel shows the summary of every clan in the tribe
  through a fast, fold-aware CLAN SUMMARIES section with a useful one-line-per-clan
  default. A clan's summary and chosen tribe are recorded durably per clan generation,
  so they survive member kills, dismissals, relaunches, and full reloads, and they seed
  the defaults when a clan with the same name is created again.

  "
phases:
  - id: tribe_clan_summaries
    title: CLAN SUMMARIES section in the tribe metadata panel
    depends_on: []
    size: medium
    description:
      "tribe_clan_summaries: add a fold-aware CLAN SUMMARIES section after TRIBE
      MEMBERS, backed by worker-side cached summary digests (kicker, headline, lede,
      styled body lines), with a Glance index, Triage ledes, Inspect previews, Forensics
      full bodies, per-entry za/zA folds, docs, unit tests, and PNG goldens."
  - id: clan_record_core
    title: Durable clan record store in sase-core
    depends_on: []
    size: medium
    description:
      "clan_record_core: in the linked sase-core repo, add the per-clan JSON record
      store (schema, merge rules, bounded locks, atomic writes, mtime cache), apply
      records over clan context in all three scan/index paths through a new
      clan_records_dir scan option, add capture-from-artifacts and launch-default
      resolution, and expose four Python bindings with tests."
  - id: clan_record_wiring
    title: Record, capture, and read clan attributes from sase
    depends_on:
      - clan_record_core
    size: medium
    description:
      "clan_record_wiring: bump the sase-core pin, add the Python facade, pass the
      records dir through every scan, record summaries and tribes at launch and on
      summary refresh, capture records before any artifact deletion, make the wait index
      honor recorded tribes, update docs, and add regression tests for the lost summary
      scenarios."
  - id: clan_tribe_edits
    title: Clan-level tribe edits from the Agents tab
    depends_on:
      - clan_record_wiring
    size: medium
    description:
      "clan_tribe_edits: make the tribe modal on a clan member or the synthetic clan row
      write an Edited clan record (including sticky unsets) through the durable
      directive path, with optimistic display, modal copy, docs, and tests."
  - id: clan_launch_defaults
    title: Inherit remembered tribe and summary for new clan generations
    depends_on:
      - clan_record_wiring
    size: medium
    description:
      "clan_launch_defaults: when a launch creates a new generation of a previously
      recorded clan without explicit tribe or summary, inherit the remembered tribe,
      re-run the remembered summary script (falling back to the remembered text), record
      them as Inherited, log it, document it, and test it."
proposed_by: bbugyi200.athena.0pw.w0
create_time: 2026-09-23 11:39:51
status: wip
---

# Plan: Tribe clan summaries and durable clan records

## Background

### What exists today

- Selecting a tribe panel (whole-panel focus) renders a fold-aware `TRIBE` document
  built by `build_tribe_detail_text` in
  `src/sase/ace/tui/widgets/prompt_panel/_agent_display_tribe.py`. Its sections are
  NEEDS ATTENTION, TRIBE MEMBERS, PROMPTS, ERRORS, OUTPUT/WORKFLOW VARIABLES, REPLIES,
  SLOW TOOL CALLS, and RUNTIME STATISTICS. The four tribe fold levels are Glance
  (COLLAPSED), Triage (EXPANDED), Inspect (FULLY_EXPANDED), and Forensics (EXHAUSTIVE),
  from `TRIBE_FOLD_SCALE`.
- The new PROMPTS section (`_agent_tribe_prompts.py` for worker-side digests,
  `_agent_display_tribe_prompts.py` for rendering) is the model to follow:
  - digests are computed off-thread in the tribe enrichment worker
    (`build_tribe_enrichment` in `_agent_tribe_aggregation.py`) and cached by raw-text
    hash;
  - the renderer only appends precomputed strings and replays precomputed style spans;
  - entries carry the roster jump-number chips and fold independently through
    `append_fold_anchor` section ids.
- Selecting a clan row renders the clan's summary as a freestanding block after CLAN
  MEMBERS (`build_clan_detail_text` in `_agent_display_clan.py`). The Rich markup is
  parsed by `clan_summary_text(agent)` in `_agent_clan_summary_text.py`, which escapes
  unrenderable tags and falls back to plain text.
- Clan units are top-level roots in a tribe panel. Each clan unit root is the synthetic
  clan container `Agent` built by `_container_for_clan` in
  `src/sase/ace/tui/models/_agent_tree.py`, and it already carries the resolved
  `clan_summary`. The summaries a tribe needs are therefore in memory. Only parsing and
  digesting them is CPU work, which must stay off the UI thread.

### Real summary shapes (surveyed from local `agent_meta.json` files)

- **Epic** (`sase_clan_summary_epic`): 16–180 lines, up to ~17 KB of markup, pre-wrapped
  at 76 columns. It starts with the banner line `◆ EPIC <clan>`, then `Title:` (which
  may wrap onto continuation lines indented to the value column), a multi-line `Goal:`,
  `Counts:`, `Path:`, `Prompt:`, `Bead:`, and `Page:`, and then one block of about five
  lines per phase.
- **Chop splits**: the banner `◆ TOOBIG SPLIT · 2 FILES`, a label-only line `MISSION`, a
  paragraph, and then a file list (9–19 lines).
- **Research**: `RESEARCH PROMPT: <long first paragraph>…` (1–37 lines).
- **Literal**: short free text, for example `Audit authentication and authorization`.

### Why clan summaries (and clan tribes) get lost

A clan's summary is stored on exactly one member's `agent_meta.json`: the `%clan`
declarer, or the member the epic workflow nominates to run the summary script. A clan's
tribe is stored on the declarer, plus on every epic member through
`SASE_EPIC_CLAN_TRIBE`. No durable clan-level record exists.

Every consumer rebuilds the clan attributes from member artifacts that are still on
disk. This includes the Rust scan and index `clan_context`
(`crates/sase_core/src/agent_scan/context.rs` and `select_clan_context_for_keys` in
`agent_scan/index/output_variables.rs` in sase-core), the TUI container,
`sase agent list`, the editor catalog, and the `%wait:@tribe` index. The ranked root
causes:

1. **The only copy is deleted.** Every single-row `x` in the Agents tab plans
   kill-and-dismiss. Dismiss items are bundled, and then their artifact directories are
   deleted (`_dismiss_persistence.py`, `_kill_persistence.py`, and
   `delete_agent_artifacts` in `_killing_utils.py`, which call
   `try_delete_agent_artifacts` in `src/sase/core/agent_cleanup_execution.py`).
   Forced-reuse wipes also `rmtree` artifact directories
   (`src/sase/agent/names/_wipe_execute.py`). The typical epic case: phase 1 declares
   the clan and finishes first, and the user dismisses it while the other phases keep
   running. After that, the Rust context has `clan_summary=None`. The loss looks
   intermittent because `_merge_clan_context` in
   `src/sase/ace/tui/actions/agents/_loading_compute_merge.py` keeps cached context on
   partial loads. It does not do so on complete-history loads or after a TUI restart.
2. **Relaunch and retry drop the declaration.** Retry and forced reuse rewrite the
   declarer into the join form (`src/sase/agent/relaunch_prompt.py`), and join-form
   members cannot carry `summary=`, `summary_script=`, or `tribe=`. TUI relaunches also
   do not recreate the `SASE_EPIC_*` environment.
3. **Non-epic clan tribes live only on the declarer**, so deleting it drops the tribe.
4. **Tribe edits lose to newer members.** Resolution picks the latest explicit
   declaration by launch timestamp. Every epic member carries `clan_tribe=epic`, so a
   tribe-modal edit on one member, including an unset, is overridden by newer members.
5. **Partial snapshots.** Bounded scans and artifact-delta scans see only part of the
   clan and rely on the TUI cache merge.

The fix is a durable per-clan record that is written whenever a clan attribute is
declared, generated, or edited, and captured just before any artifact directory is
deleted. The Rust clan context applies this record over member-derived values, so every
consumer stays consistent. The record is keyed by clan name, so it can also seed the
defaults for a new generation of the same clan.

## Design overview

- **One record per clan**:
  - Location: `<sase_home>/agent_clans/<path-safe clan>.json`.
  - Contents: a small map from generation to attribute records (`tribe`, `summary`,
    `summary_script`). Each attribute record has a value (or an explicit unset
    tombstone), a source (`declared`, `script`, `edited`, `inherited`, `propagated`, or
    `captured`), `recorded_at`, and `source_identity`.
  - Rust `sase-core` owns the schema, merge rules, locking, and I/O. This follows the
    Rust-core boundary rule: every frontend must see the same clan attributes.
- **Precedence**: for a `(clan, generation)` key, an attribute present in the record
  (including an unset tombstone) wins over member-derived values. Member-derived values
  still cover legacy, imported, and remote artifacts that have no local record.
- **Merge rules**: `declared`, `script`, `edited`, and `inherited` overwrite.
  `propagated` (the epic environment tribe on joiners) and `captured` (copied from a
  meta just before deletion) only fill missing attributes. This makes a clan-level tribe
  edit stick even when newer epic members carry `clan_tribe=epic`. An empty or failed
  summary never erases a recorded one.
- **Read path**: a new scan option, `clan_records_dir`, makes the Rust scanner and index
  query apply records over `clan_context`. The Python scan facade always passes it, so
  the TUI, `sase agent list`, and the editor helper all see recorded values. This holds
  on full, bounded, and delta loads alike.
- **Tribe CLAN SUMMARIES section**: a worker-computed digest per clan summary drives a
  four-level ladder: headline index, then lede, then preview, then full body. Every
  entry carries the roster number chip and folds on its own.

## Phase `tribe_clan_summaries`: CLAN SUMMARIES section in the tribe metadata panel

This phase is independent of the record-store phases. It renders whatever `clan_summary`
each clan unit root already carries. Before starting, read the `tui`, `tui_perf`, and
`tui_screenshot` memory notes.

### Placement and interaction

- New section `CLAN SUMMARIES` with section id `tribe:clan-summaries`, added to
  `_TribeSectionIds` in `_agent_display_tribe_common.py`. It sits directly after TRIBE
  MEMBERS and before PROMPTS, because curated clan intent reads before raw prompts.
  Order: NEEDS ATTENTION → TRIBE MEMBERS → CLAN SUMMARIES → PROMPTS → ERRORS → …
- One entry per clan unit whose `clan_summary` is non-blank, in roster order. Clans
  without a summary are skipped. If no clan has a summary, the section is absent (the
  known-empty rule).
- Heading: `CLAN SUMMARIES · <n>`, where `n` is the number of entries. Use
  `append_fold_heading`, so the heading is a Ctrl+J/Ctrl+K stop and shows the shared
  fold glyph.
- Each entry is a fold anchor, `tribe:clan-summary:<entry key>`. The key is a stable
  12-hex blake2b of `f"{clan}\0{generation or ''}"`. `za`/`zA` on an entry cycles or
  toggles only that entry; entries inherit the section's effective level unless
  overridden (same mechanics as PROMPTS entries).
- Like PROMPTS, this section is a level-1 exception: it shows content at Glance instead
  of only a heading.

### Entry line (every level)

`<number chip>` `<clan label>` `<kicker>` `<headline>` `· <N lines>`

- **Number chip**: the roster jump digits for the clan unit (`unit_numbers`, built from
  the TRIBE MEMBERS jump map exactly as PROMPTS does), styled `bold black on` the tribe
  accent. Show a dim `•` when the unit has no number.
- **Clan label**: the unit's roster label, in bold clan orchid (`#D75FFF`, the value of
  `_CLAN_IDENTITY_COLOR`).
- **Kicker**: the summary's banner kind (for example `EPIC` or `TOOBIG SPLIT · 2 FILES`)
  in `bold #AF87FF`, separated by two spaces. Omit it when empty.
- **Headline**: plain text in the tribe `BODY_STYLE`, at most 120 characters. It
  deliberately does not reuse the author's spans, so the index reads as one uniform
  typographic list.
- **Size tag**: a dim `· N lines` tag when the body has more than one line.

### Level ladder

| Level     | Content                                                                                                                                                                                        |
| --------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Glance    | Entry lines only, for up to 8 entries (`TRIAGE_LIMIT`), then a dim `+N more`                                                                                                                   |
| Triage    | Up to 24 entries; each entry line is followed by its lede: up to 4 authored lines, styled as authored, behind the clan gutter                                                                  |
| Inspect   | Up to 24 entries; each is followed by the first 16 body lines behind the gutter, then `▎ … +N more lines` (dim italic) when truncated; one blank line between entries                          |
| Forensics | Every entry, with the full body behind the gutter under a 500-line per-entry safety cap; when that cap truncates, the tail says to press the clan's number to open its clan panel for the rest |

- **Gutter**: `"  ▎ "` in muted orchid `#875FAF`. It is deliberately different from the
  PROMPTS `│` gutter, so authored clan text visibly belongs to its clan.
- **Body**: the parsed summary minus the kicker banner line (the banner is already on
  the entry line as the kicker). Leading blank lines after the banner are dropped.
  Everything else is verbatim, with the author's styles.
- **Stale entries**: the renderer only renders entries whose unit identity is present in
  the current `AgentTribeSummarySnapshot.units`.

### Digest (worker-side, pure)

New module `src/sase/ace/tui/widgets/prompt_panel/_agent_tribe_clan_summaries.py`.

**Data types:**

- `ClanSummaryDigest` (frozen, slots):
  - `key`: 12-hex blake2b of `clan\0raw`
  - `kicker: str`
  - `headline: str`
  - `lines: tuple[str, ...]`: plain body lines
  - `line_spans: tuple[tuple[StyleSpan, ...], ...]`: spans relative to each line; a
    style may be a `str` or a Rich `Style`
  - `lede_start: int` and `lede_count: int`: indices into `lines`
- `TribeClanSummaryEntry`: `unit_identity`, `unit_label`, `entry_key`, `digest`.
- `TribeClanSummariesSnapshot`: `entries` and `signature`.

**Parsing.** Refactor `_agent_clan_summary_text.py` to expose
`clan_summary_markup_text(raw: str) -> Text` (the existing parse, escape, and fallback
logic) and make `clan_summary_text(agent)` delegate to it. The clan panel's behavior
does not change. The digest parses with `clan_summary_markup_text`, applies `rstrip`,
and splits with `Text.split("\n", allow_blank=True)`.

**Headline algorithm** (over plain lines). The clan name is known.

1. The first non-blank line is a **banner** if either:
   - it starts with a non-alphanumeric glyph (such as `◆`, `▸`, `●`, `■`); or
   - it has no lowercase letters, is at most 60 characters, and does not contain `: `.

   A banner line is consumed. Its kicker text is the line with the leading glyphs and
   spaces removed, a trailing token equal to the clan name (or its owner-qualified form)
   removed, and whitespace collapsed.

2. Skip blank lines and label-only lines: at most 32 characters, uppercase letters and
   spaces only (for example `MISSION`).
3. The headline block starts at the next line:
   - **Field line**: the line matches `^(\s*)([A-Za-z][A-Za-z ]{0,23}):\s+(\S.*)$`.
     - The headline is the value plus continuation lines. Continuation lines are the
       following non-blank lines that are not field lines and whose indentation is at
       least the value column.
     - If the label is ALL CAPS (for example `RESEARCH PROMPT`) and there is no kicker
       text yet, the label becomes the kicker.
   - **Otherwise**: the block is a paragraph ending at a blank line or a field line.
4. Collapse whitespace, strip a leading Markdown heading marker, and truncate to 120
   characters at a word boundary with `…` (the same helper shape as PROMPTS).
5. **Fallbacks**:
   - If the headline is empty, use the kicker text as the headline and clear the kicker.
   - If both are empty, use `first_meaningful_line` of the plain text.
6. **Lede**:
   - If the headline was truncated, the lede is the headline block's own lines, so the
     reader sees the rest of the sentence.
   - Otherwise, the lede is the next 4 non-blank lines after the block.
   - Record the lede as body-line indices.

Expected results for the fixture shapes:

| Shape    | Kicker                   | Headline                                              |
| -------- | ------------------------ | ----------------------------------------------------- |
| Epic     | `EPIC`                   | The Title value, joined across its continuation lines |
| Chop     | `TOOBIG SPLIT · 2 FILES` | The MISSION paragraph                                 |
| Research | `RESEARCH PROMPT`        | The first paragraph, truncated                        |
| Literal  | (none)                   | The text itself                                       |

For the epic shape, the lede starts at the Goal lines.

**Caching and failure handling:**

- A module-level `OrderedDict` LRU cache (256 entries, guarded by a `threading.Lock`)
  keyed by `(blake2b(raw), clan)`.
- Any exception while digesting yields a plain fallback digest (headline =
  `first_meaningful_line`, unstyled lines), so one bad summary never fails the worker.

### Enrichment wiring (`_agent_tribe_aggregation.py`)

- Add `"clan-summaries"` to `TribeEnrichmentSection`. It is not a disk section.
- `tribe_enrichment_sections_for_fold_state` always requires it, at every level.
- `TribeSectionSnapshot` gains `clan_summaries: TribeClanSummariesSnapshot | None`,
  where `None` means not loaded. `TribeEnrichmentResult` carries the worker result.
- `prepare_tribe_section_snapshot`:
  - Compute the current signature: a tuple of `(unit_identity, len(raw), hash(raw))` for
    each clan-container source with a non-blank summary. `hash` of a string is computed
    once per string object and costs a bounded amount of work per summary.
  - Store the signature on the cache entry.
  - Keep any cached `clan_summaries` even when the source signature changes, so
    membership churn never blanks the section.
  - When the signature is empty, publish the shared empty snapshot synchronously. No
    worker runs and no scanning tail appears.
- `tribe_sections_to_refresh` requests `"clan-summaries"` only when the cached snapshot
  is `None` or its signature differs from the current one. There is no timer: summaries
  change only when their text changes.
- `build_tribe_enrichment` builds the snapshot from `source.root.clan_summary` for
  clan-container sources. `cache_tribe_enrichment` merges the result.
- `_has_pending_enrichment` treats `clan_summaries is None` as pending, which shows the
  shared `⋯ scanning member data…` tail.

### Renderer

- New module `_agent_display_tribe_clan_summaries.py` with
  `append_clan_summaries(text, section_snapshot, *, level, overrides, unit_numbers, present_units)`.
- Call it from `_append_tribe_body` right after the roster.
- It replays precomputed per-line spans over bounded line ranges. It must never parse
  markup or touch the filesystem.

### Performance requirements

- The UI thread does no markup parsing: all parsing and digesting happens in the tribe
  worker.
- A re-selected panel reuses the cached snapshot and paints immediately.
- Glance and Triage render at most about 8 and about 120 body lines.
- Add a guard test: with a loaded snapshot, rendering at every level succeeds while
  `rich.text.Text.from_markup` is monkeypatched to raise.
- Check that j/k over tribe panels keeps the TUI performance targets. Use
  `SASE_TUI_PERF=1` if unsure.

### Docs and tests

- Docs: update the four-level tribe table in `docs/ace.md`, and add a paragraph next to
  the PROMPTS paragraph that describes CLAN SUMMARIES: placement, entry anatomy, number
  chips, per-entry folds, and the ladder.
- Unit tests:
  - Digest cases: epic, chop, research, literal, wrapped Title continuation, invalid
    markup such as an unclosed `[@file:x]`, banner-only, and empty.
  - Cache hit behavior.
  - Renderer ladder and limits: `+N more`, truncation tails, the Forensics cap, gutter,
    kicker, chips, and a unit missing from the roster.
  - Fold anchors and per-entry overrides.
  - Enrichment requirements and freshness: signature change triggers a refresh, an empty
    signature needs no worker, and a membership change keeps cached entries.
- Update the existing requirement assertions in
  `tests/ace/tui/widgets/test_agent_display_tribe_sections.py`, and the known-section
  helper in `tests/ace/tui/widgets/test_summary_fold_contracts.py`.
- Add `tests/ace/tui/visual/test_ace_png_snapshots_agents_tribe_clan_summaries.py` with
  Glance and Inspect goldens at 120x40. Model it on
  `test_ace_png_snapshots_agents_tribe_prompts.py`. Use a tribe containing an
  epic-shaped clan (a short synthetic epic markup summary) and a literal-summary clan.

## Phase `clan_record_core`: Durable clan record store in sase-core

Work in the linked checkout: open it with `sase repo open sase-core -r "<why>"` and use
the printed path. Read its `AGENTS.md` first. Model store I/O on `agent_hold.rs`
(sase-home-relative path helpers) and `store_lock.rs` (bounded file locks).

### Module and wire types

Put the domain module at `crates/sase_core/src/agent_clan_record` (or a single `.rs`
file if it stays small), with tests beside it. Use serde wires with `#[serde(default)]`
throughout for forward compatibility.

- `AGENT_CLAN_RECORD_SCHEMA_VERSION = 1`.
- `AgentClanRecordWire`:
  - `schema_version`
  - `clan`
  - `latest_generation: Option<String>`
  - `generations: BTreeMap<String, ClanGenerationRecordWire>`
- `ClanGenerationRecordWire`:
  - `first_recorded_at`
  - `tribe`, `summary`, `summary_script`: each `Option<ClanAttributeRecordWire>`
- `ClanAttributeRecordWire`:
  - `value: Option<String>`: `None` together with `source=edited` is an explicit unset
    tombstone
  - `source: ClanAttributeSourceWire`
  - `recorded_at`: UTC RFC 3339
  - `source_identity: Option<String>`: an artifacts dir, or `tui`/`cli` for edits
- `ClanAttributeSourceWire`, serialized snake_case: `declared`, `script`, `edited`,
  `inherited`, `propagated`, `captured`.
- `ClanRecordUpdateWire`:
  - `clan`
  - `generation`
  - `tribe`, `summary`, `summary_script`: each
    `Option<ClanAttributeUpdateWire { value, source, source_identity }>`
- `ClanRecordUpdateOutcomeWire`: `{ changed: bool, record: AgentClanRecordWire }`.
- `ClanLaunchDefaultsWire`:
  - `clan`
  - `tribe`, `summary`, `summary_script`, each with its own `*_generation` provenance
    field

### Store location and paths

- `clan_records_dir(sase_home) = <sase_home>/agent_clans`.
- `clan_record_path(records_dir, clan)`:
  - Percent-encode every byte outside `[A-Za-z0-9_.-]`.
  - Reject empty names, `.`, `..`, and anything that would escape the directory.
  - Append `.json`.
- Records require a non-empty generation. Legacy contexts with a `None` generation are
  never overlaid.

### Merge rules (`record_clan_attributes`)

- **Normalization**: trim values. An empty string is treated as absent, making the
  update a no-op. Truncate summaries at a character boundary to the 32 KiB launch cap.
- **Overwriting sources**: `declared`, `script`, `edited`, and `inherited` overwrite the
  existing attribute.
- **Fill-only sources**: `propagated` and `captured` write only when the attribute is
  absent, meaning it has no value and no tombstone.
- **Tombstones**: only `edited` may write a tombstone (a tribe with `value: None`).
- **Summary scripts**: `captured` never writes `summary_script`.
- **Generations**:
  - Set `latest_generation` to the written generation when it compares greater than or
    equal to the current one. Generation keys are artifact-directory timestamp names, so
    lexicographic order is chronological.
  - Keep only the 8 newest generations per clan.
- **No-op updates**: a no-op merge returns `changed: false` and does not write.

### I/O and locking

- **Writes**: take a bounded exclusive `store_lock` on `<file>.lock` with a timeout of
  about 2 s. Read, merge, write a temp file in the same directory, then rename. Create
  the directory on demand.
- **Reads** (`load_clan_record`):
  - Lock-free; the atomic rename guarantees a consistent file.
  - A missing file returns `None`. A corrupt file returns an error that callers treat as
    absent.
  - Use an in-process cache keyed by path and validated by `(mtime, len)`, bounded to
    about 512 entries.

### Capture and launch defaults

- **`capture_clan_record_from_artifacts(records_dir, artifacts_dir)`**:
  - Read `agent_meta.json`. Do nothing if it is missing, corrupt, or lacks `agent_clan`
    or `agent_clan_generation`.
  - Fill `clan_tribe` and `clan_summary` as `captured`, with
    `source_identity = artifacts_dir`.
  - Honor the legacy clan key rules in `clan_key_from_meta`, including the parallel
    family fallback.
- **`resolve_clan_launch_defaults(records_dir, clan, exclude_generation)`**:
  - Resolve each attribute from the newest generation, other than the excluded one, that
    has the attribute.
  - A tombstone yields `None` and stops the search for that attribute.
  - Report which generation supplied each value.

### Clan-context overlay

- Add `clan_records_dir: Option<String>` to `AgentArtifactScanOptionsWire`, as
  `#[serde(default)]`.
- When it is set, apply the records after the clan context is computed in all three
  places:
  - `scan_agent_artifacts`;
  - `scan_agent_artifact_dirs`;
  - `query_agent_artifact_index`, on both the projection-key and record-key branches.
- For each context with a non-empty generation whose record generation has the
  attribute, including a tombstone:
  - Set `clan_tribe` / `clan_summary` from the record.
  - Set the matching `*_source_identity` from the record, and clear
    `*_source_launch_timestamp`. First check how Python consumers such as
    `_loading_compute_merge.py` use these fields, and keep their contract.
- Overlay failures are soft: keep the member-derived values and continue.
- Index rebuild and upsert ignore the new option.

### Bindings

Add these next to `resolve_clan_summary` in `crates/sase_core_py/src/agent_scan/`,
releasing the GIL:

- `load_agent_clan_record(records_dir, clan)`
- `record_agent_clan_attributes(records_dir, update)`
- `capture_agent_clan_record_from_artifacts(records_dir, artifacts_dir)`
- `resolve_agent_clan_launch_defaults(records_dir, clan, exclude_generation=None)`

Update any golden and schema fixtures that pin the scan options wire.

### Tests

- Merge-rule table: overwrite vs fill-only for each source, tombstones, empty-summary
  no-op, truncation, and generation retention.
- Path encoding and traversal rejection.
- Concurrent writers (two threads writing different attributes both persist).
- Corrupt files; cache invalidation on an mtime change.
- Capture: fill-only, and not overriding an edit.
- Launch defaults: per-attribute resolution and tombstones.
- Overlay in all three scan paths:
  - the declarer's artifact is deleted and the context still has the recorded summary
    and tribe;
  - an edited tribe beats member-derived `epic`;
  - a record for another generation is not applied.
- Binding round-trip tests.

Run `sase tool run check` in the sase-core checkout before finishing. This is a new
optional wire field plus new bindings, so it is not a breaking change.

## Phase `clan_record_wiring`: Record, capture, and read clan attributes from sase

Read the `rust_core_backend_boundary` note (always loaded) and `docs/rust_backend.md`.

### Core pin and facade

- Open sase-core with `sase repo open sase-core`. Make sure the checkout contains the
  `clan_record_core` commit and that the commit is on sase-core's remote master. The
  local setup rebuilds `sase_core_rs` when the linked source changes. Then move
  `sase-core-revision.txt` with `just ratchet-core-revision`.
- New facade `src/sase/core/agent_clan_record.py`:
  - `clan_records_dir()` returns `sase_home() / "agent_clans"`.
  - Thin typed wrappers for the four bindings: `load_clan_record`,
    `record_clan_attributes`, `capture_clan_record_from_artifacts`,
    `resolve_clan_launch_defaults`.
  - Mutations made on launch and cleanup paths are best-effort: log and swallow errors,
    and never block a launch or a deletion. Add a flag or separate helper so that
    user-initiated edits can surface errors.

### Scan facade

- Add `clan_records_dir: str | None = None` to the Python `AgentArtifactScanOptionsWire`
  dataclass.
- In `_options_to_dict` (`src/sase/core/agent_scan_facade.py`) and the index-query
  wrapper, fill it with `str(clan_records_dir())` when it is unset. Every consumer (the
  TUI loaders, `sase agent list`, `_editor_helper_agents.py`) then gets the overlay.
- Make sure the options echoed back in scan results still parse.

### Launch writes (`src/sase/axe/run_agent_directives.py`)

After `write_agent_meta`, for a member with a clan membership plan:

- **Declarer** (`directives.clan_declared`):
  - record `declared` tribe when one is declared;
  - record either a `declared` literal summary, or a `declared` `summary_script` (the
    raw directive value) plus a `script` summary when the script produced non-empty
    text.
- **Epic-nominated joiner** (`epic_clan_summary_script`):
  - record a `script` summary when it is non-empty;
  - record a `propagated` `summary_script`.
- **Any member** carrying the epic environment tribe records a `propagated` tribe
  (fill-only).

Then in `run_agent_runner_launch.py::_refresh_clan_summary`, record a `script` summary
after every successful non-empty post-preparation refresh.

### Capture before deletion

- Call the facade's capture in `try_delete_agent_artifacts`
  (`src/sase/core/agent_cleanup_execution.py`) before the delete binding runs. This
  covers dismiss, kill-delete, bulk cleanup, and `sase agent` cleanup commands.
- Also capture in `src/sase/agent/names/_wipe_execute.py` before `rmtree` of an
  artifacts directory.
- Grep for any other deletion of `agent_meta.json`-bearing directories (`rmtree`,
  `delete_agent_artifacts`) and cover it.
- Confirm that none of these run on the Textual event loop. They run in durable procs
  and workers today.

### Wait index

In `_refresh_effective_clan_tribe`
(`src/sase/core/wait_dependency_resolution/_index.py`), apply the record's tribe for
`(clan, generation)` over the member resolution. A tombstone means no tribe. Load each
clan's record at most once per index build.

### Leave in place

`_container_for_clan` already lets context win. Keep `_merge_clan_context` as a safety
net.

### Docs

- `docs/agent_families.md`: replace "persists the result as `clan_summary` on the
  declaring agent's metadata" with the clan-record contract: what is recorded and when,
  precedence, fill-only capture before deletion, and what survives kills, dismissals,
  relaunches, and restarts.
- Touch `docs/axe.md` and `docs/plugins.md` only where they state declarer-only storage.

### Tests

- The reported regression: build an epic-shaped clan where the declarer carries the
  summary and tribe. Dismiss the declarer through the real deletion path. A complete
  history reload through the loader must still yield a clan container with that summary
  and tribe. Repeat for kill-delete and for a relaunched join-form member.
- Bounded and delta scans keep the recorded values.
- The wait index honors a recorded tribe and a tombstone.
- Launch records for a declared literal summary, a script summary, an epic nominee, and
  a propagated tribe (fill-only).
- Capture is a no-op for non-clan directories.
- A binding error inside capture does not break deletion.

## Phase `clan_tribe_edits`: Clan-level tribe edits from the Agents tab

The tribe modal (`N`) runs `_apply_agent_tribe_change` in
`src/sase/ace/tui/actions/agents/_tribe_assignment.py`.

### Directive payload

- For clan-bound targets, add one `clan_record` update per `(clan, generation)`:
  `{"clan": ..., "generation": ..., "tribe": <value or null>}`. Deduplicate when several
  members of the same clan are affected.
- Replace the refusal "Cannot edit a synthetic clan row directly" with a record-only
  edit: no `artifacts_dir`, meta, or prompt updates for the synthetic row.
- Member edits keep their existing meta and `%clan` prompt rewrites, for older readers,
  and also emit the record edit.

### Applying the edit

- `src/sase/ops/commands/_agent_directive.py` applies `clan_record` updates with source
  `edited` and `source_identity` `tui`. Apply them after the same
  `canonicalize_public_tribe_name` resolution used for member edits.
- This is user-initiated, so a failure is reported as a directive error, not swallowed.

### Display

- Update the clan container's and member rows' visible clan tribe optimistically,
  reusing the existing rollback machinery. Let the established refresh path reconcile
  through the Rust context overlay. Add no new refresh code paths.
- Modal copy for clan-bound targets names the clan, for example
  `Set tribe for clan <name>`.

### Docs and tests

- Docs (`docs/agent_families.md` and the tribe-assignment text in `docs/ace.md`): `N` on
  a clan row or a clan member sets the clan's recorded tribe, and an unset sticks even
  when members carry an epic tribe.
- Tests:
  - the synthetic-row edit writes the record;
  - a member edit writes the meta and the record;
  - an unset beats `propagated` and newer-member `epic` values after a reload;
  - deduplication;
  - error surfacing;
  - optimistic display with rollback on failure.

## Phase `clan_launch_defaults`: Inherit remembered tribe and summary for new clan generations

### Detecting a new generation

In `src/sase/axe/run_agent_directives.py`, after identity resolution, treat the launch
as having created the generation when
`clan_membership_plan.generation == Path(artifacts_dir).name`. Verify this invariant
against the generation derivation in `run_agent_directive_identity.py` and the
planned-clan reservation in `src/sase/agent/names/_registry_group_mutations.py`. If it
does not hold for multi-segment planned clans, carry an explicit `created` flag out of
the reservation instead.

### Applying defaults

When the launch created the generation, call
`resolve_clan_launch_defaults(clan, exclude_generation=generation)`, then:

- **Tribe**: when there is no declared `tribe=` and no epic environment tribe, set
  `agent_meta["clan_tribe"]` to the default, and record it as `inherited`.
- **Summary**: when there is no declared `summary=` or `summary_script=` and no epic
  nomination:
  - If a remembered `summary_script` exists, run it through the existing
    `ClanSummaryResolutionRequest` / `resolve_clan_summary_script` path, so the
    post-preparation refresh also re-runs it. Record the script as `inherited`.
  - If the script output is empty, use the remembered summary text.
  - Otherwise, use the remembered literal summary.
  - Record the summary as `inherited`.
- Append an agent-log line naming the generations the values came from.
- The rules apply both to bare `%clan(<name>)` declarations and to join-form
  `%id(<id>, clan=<name>)` launches that create a new generation.
- A runner re-exec must not re-apply defaults that are already present in the preserved
  metadata.

### Docs and tests

- Docs: add a "Re-creating a clan" paragraph in `docs/agent_families.md` and a note in
  the `%clan` section of `docs/xprompt.md`.
- Tests:
  - a new generation inherits the tribe and summary;
  - explicit values override;
  - a remembered script is re-run;
  - a script failure falls back to the remembered text;
  - members joining an existing generation are unaffected;
  - a tombstoned tribe is not inherited;
  - with no record, nothing changes;
  - a re-exec is idempotent.

## Non-goals

- Recovering summaries that were already lost before this change. Dismissed bundles
  still contain them, so a recovery import is a possible follow-up task bead.
- Summaries for remote fleet rows. Clans whose metadata has no generation are also
  excluded.
- A clan target for the `sase agent tribe` CLI.
- Folding or collapsing the summary inside the clan panel itself.
- Store-wide pruning beyond the per-clan generation cap.

## Verification

- Before finishing, each sase phase reads the `lint_and_test` memory note and runs
  `just check` through `sase tool run check`. The sase-core phase runs
  `sase tool run check` in its checkout.
- Manual smoke test after `clan_record_wiring`:
  1. Launch a small two-member clan with a literal summary and a tribe.
  2. Dismiss the declarer.
  3. Restart the TUI.
  4. The clan panel and the tribe CLAN SUMMARIES section still show the summary, and the
     clan remains in its tribe.
