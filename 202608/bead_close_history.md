---
tier: epic
title: Preserve close provenance when a +1 reopens a bead
goal: 'A bead that was closed and later reopened keeps the reason it was closed, and
  every surface that shows the bead says plainly that it was previously closed, why,
  and what reopened it — including which +1 did it.

  '
phases:
- id: core-model
  title: Durable close history in the bead event reducer
  depends_on: []
  size: medium
  description: 'core-model: add BeadCloseRecordWire and IssueWire.close_history to
    sase-core, archive close metadata instead of discarding it on every reopen path,
    unify the mutation and reducer paths on one helper, and release.

    '
- id: core-adopt
  title: Adopt the release and carry close history through Python storage
  depends_on:
  - core-model
  size: medium
  description: 'core-adopt: raise the sase-core-rs window to the release from core-model
    and thread close_history through the Python model, wire conversion, issues.jsonl,
    the SQLite mirror with its migration, and the projection repair guard.

    '
- id: presentation
  title: Shared reopen presentation vocabulary
  depends_on:
  - core-adopt
  size: small
  description: 'presentation: add sase/bead/reopen_presentation.py with the accent,
    glyph, section label, badge, record labels, search text, and the single shared
    join that decides which +1 entry reopened a bead.

    '
- id: cli
  title: sase bead show, JSON, list badges, and search
  depends_on:
  - presentation
  size: medium
  description: 'cli: render the PREVIOUSLY CLOSED section and the reopening-+1 marker
    in bead detail, emit close_history in detail JSON, add the reopen badge to list/ready/search
    rows, and make archived close reasons searchable.

    '
- id: triage
  title: Prior-close warning in the TaskTriage gate
  depends_on:
  - presentation
  size: small
  description: 'triage: put a prior-close callout above the description in the task
    triage preview, mark the reopening +1, add the reopen badge to the notification
    note, and include close history in the chop''s presentation fingerprint.

    '
- id: ace
  title: ACE beads pane close history
  depends_on:
  - presentation
  size: small
  description: 'ace: show the reopen badge on beads list rows, a Previously closed
    property and body section in the detail pane, and a has:reopened filter, with
    PNG snapshot coverage.

    '
- id: pages
  title: Generated bead pages close history
  depends_on:
  - presentation
  size: small
  description: 'pages: render a Previously Closed section and primary-fact badge on
    generated bead pages and add a reopen column to the lineage roster.

    '
- id: docs
  title: Document close history and reopen provenance
  depends_on:
  - cli
  - triage
  - ace
  - pages
  size: small
  description: 'docs: document the close-history record, the reopen causes, the retroactive
    recovery from the event log, and every surface that renders it in docs/beads.md.'
proposed_by: bbugyi200.athena.tr
create_time: 2026-08-05 21:17:46
status: wip
bead_id: sase-fr
---

- **PROMPT:** [prompts/202608/bead_close_history.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/bead_close_history.md)
- **BEAD:** [sase-fr](https://github.com/sase-org/sase--beads/blob/main/pages/sase-fr/README.md)

# Plan: Preserve close provenance when a +1 reopens a bead

## Context

When a SASE agent finds that a newly discovered issue duplicates an already-closed task, `/sase_new_task` tells it to
`sase bead +1` the closed task instead of filing a new one. That `+1` promotes the closed task back to `ready`, which is
the intended behavior. The problem is what the promotion destroys and what it fails to say.

### Reproduced defect 1 — the close reason is destroyed

`crates/sase_core/src/bead/mutation.rs` (`add_task_plus_one`) and the matching reducer branch in
`crates/sase_core/src/bead/events.rs` both call `clear_close_metadata`, which nulls `closed_at`, `close_reason`, and
`resolution` outright.

Reproduced against the current workspace build:

```text
created sase-1
close(reason="Not reproducible on main; retry shim covers it.", resolution=canceled)
plus_one(reporter="claude.probe")
after +1: Status.READY closed_at=None close_reason=None resolution=None
```

The reason a human or agent deliberately rejected this work is gone from the projected issue and therefore gone from
`sase bead show`, `sase bead show --format json`, `sase bead list`, the ACE beads pane, generated bead pages, and — most
consequentially — the `TaskTriage` gate preview that asks the project owner to choose **Launch** or **Close** for the
freshly reopened task. Launch is the gate's primary branch, so the owner is being nudged to relaunch work someone
already decided against, with the deciding rationale deleted.

The bytes are not actually lost. The canonical event log keeps them, verified on the same store:

```text
2026-08-06T01:08:52Z  axe.scout     issue_created            [created_at, created_by, id, issue_type, owner, size, title, updated_at]
2026-08-06T01:08:52Z  axe.scout     issue_closed             [close_reason, closed_at, resolution, status]
2026-08-06T01:08:52Z  claude.probe  task_plus_one_recorded   [close_reason, closed_at, plus_one_evidence, resolution, status]
```

They are only lost from the projection. That matters twice over: the fix is a reducer change rather than a new write
path, and because `MutableStore::load` rebuilds issues by replaying events, **every already-damaged bead in every
existing store recovers its close reason automatically on the next reduction**. No backfill script is needed.

### Reproduced defect 2 — nothing says a +1 reopened the bead

`sase bead show` renders its `RESOLUTION` block only when `status == CLOSED`, and the `+1 EVIDENCE` list is flat. A
reopened bead therefore shows a `ready` task with N corroborating reports and no indication that it was ever closed, or
that one of those reports is what brought it back.

### Reproduced defect 3 — the mutation path and the reducer disagree

The reopen paths write different bytes depending on whether you just mutated or reprojected. `open_issue` clears only
`resolution` in memory and then writes `issues.jsonl` from that in-memory state, while the reducer's `IssueOpened`
branch calls `clear_close_metadata`. Reproduced:

```text
after `open` jsonl: {'status': 'open', 'closed_at': '2026-08-06T01:09:00Z', 'close_reason': "Won't fix — by design.", 'resolution': None}
after reproject   : {'status': 'open', 'closed_at': None, 'close_reason': None, 'resolution': None}
DRIFT: True
```

`apply_update_fields` has the same asymmetry for `sase bead update -s open`. This is already known implicitly: the
`sase bead admin --fix-projection` guard in `src/sase/bead/cli_admin.py` whitelists exactly `closed_at`, `close_reason`,
and `updated_at` as fields a repair may silently change. Fixing this is a prerequisite rather than scope creep — a new
`close_history` field written by only one of the two paths would drift the same way, and the drift would be invisible
until a reprojection quietly rewrote someone's beads repo.

## Design

### One idea

A bead remembers how it was closed, even after it reopens. A close that gets undone is **archived**, not deleted.

### The record

Add `close_history` to the issue: an append-only list of close episodes that have since been undone, oldest first. The
_current_ close, when a bead is closed right now, stays exactly where it is today in the flat `closed_at` /
`close_reason` / `resolution` fields. Nothing about existing closed-bead behavior changes. `close_history` is strictly
the past.

```rust
pub struct BeadCloseRecordWire {
    pub closed_at: String,                              // non-empty
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub close_reason: Option<String>,
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub resolution: Option<BeadResolutionWire>,
    pub reopened_at: String,                            // non-empty
    pub reopened_via: BeadReopenCauseWire,              // plus_one | open | update | epic_preclaim
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub reopened_by: Option<String>,                    // see "Attribution" below
}
```

`close_history` applies to every bead type, not just tasks. Plans and phases are reopened by `sase bead open` and by
epic work preclaims, and "why was this epic closed before?" is the same question with the same answer.

Every record has a `reopened_at`, because a record only exists once the close has been undone. That keeps the type
total: no record can be half-populated.

### Attribution — deliberately narrow

`reopened_by` is populated only for causes whose event actor is genuinely the acting agent. Today that is `plus_one`
alone: `add_task_plus_one` passes the reporter as the event actor, but `close_issues`, `open_issue`, and
`set_ready_to_work` all pass `&issue.created_by` — the bead's _creator_, not whoever ran the command. Recording a
`closed_by` or a general `reopened_by` from those events would confidently print the wrong name.

So this epic records only what is true, and leaves `reopened_by` optional so the remaining causes can fill in later
without a wire change. The underlying actor defect is called out in **Out of scope** below.

### One chokepoint

Replace `clear_close_metadata` with `archive_close_metadata(issue, reopened_at, via, reopened_by)`:

- if `issue.closed_at` is `None`, do nothing (reopening an already-open bead archives nothing);
- otherwise move `closed_at` / `close_reason` / `resolution` into a new `close_history` record and null the flat trio.

All four reducer branches that currently clear metadata call it: `IssueOpened` (`via: open`), `IssueClosed`'s
counterpart is untouched, `IssueUpdated` with a non-closed status (`via: update`), `EpicWorkPreclaimed`
(`via: epic_preclaim`), and `TaskPlusOneRecorded` (`via: plus_one`, `reopened_by: Some(evidence.reporter)`). The
mutation-side functions call the same helper so the two paths cannot diverge again.

### Linking the +1 that reopened the bead

For a plus-one reopen, `add_task_plus_one` stamps one timestamp into both the evidence entry and the event, so
`record.reopened_at == evidence.timestamp` and `record.reopened_by == Some(evidence.reporter)` hold by construction, and
reporters are unique per task. That makes `(reporter, timestamp)` an exact join key.

The link is therefore _derived_ at presentation time rather than stored on the evidence entry. That keeps the
`TaskPlusOneEvidence` wire shape — which is persisted into gate payloads, the SQLite mirror, and `issues.jsonl` —
completely unchanged, and it puts one shared join function behind every surface. A sase-core test pins the invariant so
the join cannot silently stop matching.

### Rendering

The glyph `↺` is the visual thread: it marks the badge, the archived record, and the +1 entry that caused the reopen.
The accent is `#FF8700`, distinct from the `+1` pink `#FF87D7` and every status, type, and size color already in use.

`sase bead show` for the motivating case:

```text
◆ sase-2f · Flaky retry test in CI   [READY] [+1] [↺1]

Type: task · Owner: bryan
Size: small

CREATED
  2026-07-14 (3 weeks ago)

CREATED BY
  @axe.scout

PREVIOUSLY CLOSED
  ↺ Closed 2026-07-30T09:12:04Z · canceled
    Reason:
      Not reproducible on main; the retry shim already covers this.
    Reopened 2026-08-05T17:04:11Z by a +1 from @claude.probe

DESCRIPTION
  ...

+1 EVIDENCE
  +1 claude.probe · 2026-08-05T17:04:11Z  ↺ reopened this task
    Saw the same flake in CI run 4821 with a clean worktree.
```

Decisions behind that block:

- **Placement.** `PREVIOUSLY CLOSED` sits exactly where the `RESOLUTION` block sits — above `PARENT` and well above
  `DESCRIPTION`. The two are usually mutually exclusive; a bead closed, reopened, and closed again shows `RESOLUTION`
  first, then `PREVIOUSLY CLOSED`.
- **Order.** Records render newest first, because the freshest decision is the one that matters at triage. Storage stays
  oldest-first append order.
- **Narrative.** Closed → why → reopened, in that order, so the entry reads chronologically.
- **No relative time.** This block never renders "6 days ago". It is persisted verbatim into gate previews that gate
  validation re-derives and byte-compares, and an absolute date is the more useful fact for a go/no-go decision anyway.
- **Placeholders match the existing block.** Missing resolution renders `(unrecorded)`; missing reason renders
  `Reason: (none)` on one line rather than being omitted, because "no reason was ever recorded" is itself information.
- **The count lives in the badge.** `[↺2]` next to `[+2]`, always numeric, so `sase bead list` and the ACE beads list
  surface repeat-rejection at a glance without a new column.

## Phases

### Durable close history in the bead event reducer

This phase crosses the Rust core backend boundary. Open the sase-core checkout with
`sase repo open sase-core -r "<reason>"` and use only the path it prints. Everything in this phase lands there; no file
in the sase repo changes.

In `crates/sase_core/src/bead/wire.rs`:

- Add `BeadReopenCauseWire` (`PlusOne`, `Open`, `Update`, `EpicPreclaim`; serde `snake_case`) and `BeadCloseRecordWire`
  with the shape given in the Design section, following `TaskPlusOneEvidenceWire` as the template for derives, serde
  attributes, and a `validate()` method. `validate()` rejects a blank `closed_at` or `reopened_at`, and rejects a
  `reopened_by` that is present but blank.
- Add `pub close_history: Vec<BeadCloseRecordWire>` to `IssueWire` with
  `#[serde(default, skip_serializing_if = "Vec::is_empty")]`. The skip is load-bearing: without it the first reduction
  after upgrade would rewrite every bead row in every project's beads repo. Extend `IssueWire::validate()` to validate
  each record.

In `crates/sase_core/src/bead/events.rs`:

- Replace `clear_close_metadata` with `archive_close_metadata(issue, reopened_at, via, reopened_by)` per the Design
  section, including the no-op guard when `closed_at` is `None`.
- Update all four callers: `IssueOpened` → `open`; `IssueUpdated` (the trailing non-closed-status branch) → `update`;
  `EpicWorkPreclaimed` → `epic_preclaim`; `TaskPlusOneRecorded` → `plus_one` with
  `reopened_by: Some(evidence.reporter.clone())`. Use the event timestamp as `reopened_at` in every case.

In `crates/sase_core/src/bead/mutation.rs`, make the mutation paths produce byte-identical state to a reprojection:

- `add_task_plus_one` currently inlines the three nulls at the `matches!(issue.status, Open | Closed)` branch — switch
  it to the shared helper.
- `open_issue` currently clears only `resolution`, leaving stale `closed_at` / `close_reason` in the written
  `issues.jsonl`. Switch it to the shared helper.
- `apply_update_fields` has the same gap for a status change away from closed. Switch it to the shared helper.
- Check `preclaim_epic_work` for the same pattern and give it the same treatment.

Tests, in sase-core:

1. **Parity.** For each of the four reopen causes, assert that the issue produced by the mutation function equals the
   issue produced by reducing that store's event streams. This is the test that permanently closes the drift class in
   reproduced defect 3, and it should fail against the pre-change code for `open_issue` and the update path.
2. **Retroactive recovery.** Build a store whose `issues.jsonl` predates this change (no `close_history` key) but whose
   event log contains `issue_closed` followed by `task_plus_one_recorded`, reduce it, and assert the archived reason and
   resolution reappear.
3. **Join invariant.** For a plus-one reopen, assert `record.reopened_at == evidence.timestamp` and
   `record.reopened_by == Some(evidence.reporter)`.
4. **Multiple episodes.** close → +1 → close → `sase bead open` yields two records, oldest first, with the right causes.
5. **No-op guard.** Reopening a bead that was never closed adds no record.
6. **Round trip.** A `close_history` value survives serialize → deserialize, and an issue with an empty history
   serializes to bytes identical to today's.

Land through sase-core's normal review flow and get a release published; release-please cuts sase-core releases on merge
to master. This is an additive `feat`, so expect a minor bump. Record the exact released version in the phase notes —
`core-adopt` needs it.

### Adopt the release and carry close history through Python storage

Raise the `sase-core-rs` window in `pyproject.toml` to the version `core-model` released, moving the ceiling if the
release crossed a minor line (a `0.19.0` release needs `>=0.19.0,<0.20.0`), and refresh `uv.lock`. See **Sequencing**
below for the interaction with the in-flight CI recovery epic.

Then thread the field through Python storage. Nothing in this phase renders anything.

- `src/sase/bead/model.py`: add a frozen `CloseRecord` dataclass mirroring the wire shape (`closed_at`,
  `close_reason: str | None`, `resolution: Resolution | None`, `reopened_at`, `reopened_via: ReopenCause`,
  `reopened_by: str | None`) with a `validate()` matching `TaskPlusOneEvidence`, a `ReopenCause` enum, and
  `close_history: list[CloseRecord] = field(default_factory=list)` on `Issue`. Validate each record from
  `Issue.validate()`. Do **not** restrict `close_history` by issue type.
- `src/sase/core/bead_wire.py`: decode `close_history` from the Rust outcome dicts, tolerating absence.
- `src/sase/bead/jsonl.py`: encode and decode, omitting the key entirely when the list is empty so unaffected rows stay
  byte-identical.
- `src/sase/bead/db.py`: add a `close_history TEXT NOT NULL DEFAULT '[]'` column to `_SCHEMA`, a
  `_migrate_add_close_history` following the plain `PRAGMA table_info` + `ALTER TABLE` pattern used by
  `_migrate_add_refs` (this column needs no table rebuild, so it needs no new Rust binding — do not follow
  `_migrate_add_plus_one_evidence`, which delegates to Rust only because of a CHECK constraint), JSON encode/decode
  helpers next to `plus_one_evidence_json`, and the column in the insert/select field lists.
- `src/sase/bead/work.py`: include `close_history` in the issue payload it builds.
- `src/sase/bead/cli_admin.py`: add `close_history` to `_projection_repair_refusal`'s `allowed_fields`. The first
  `--fix-projection` run after upgrade legitimately materializes archived records for beads whose reasons were destroyed
  before this change, and the guard must not refuse that. Leave the existing `closed_at` / `close_reason` entries in
  place; they are still reachable through legacy stores.

Tests: JSONL round trip including a legacy row with no `close_history` key; DB migration from a pre-column database;
`Issue.validate()` rejection cases; and an end-to-end check through `BeadProject` that close → `+1` leaves the reason in
`close_history` and the flat trio null.

### Shared reopen presentation vocabulary

Add `src/sase/bead/reopen_presentation.py`, mirroring `src/sase/bead/plus_one_presentation.py` in structure and
docstring style. This module is the single source of truth for every downstream surface; the four consumer phases depend
on it and on nothing else from each other.

Exports:

- `REOPEN_ACCENT = "#FF8700"`, `REOPEN_RICH_STYLE`, `REOPEN_CLI_STYLE` (via `ansi_sgr`), `REOPEN_GLYPH = "↺"`,
  `REOPEN_SECTION_LABEL = "PREVIOUSLY CLOSED"`, `REOPEN_EVIDENCE_MARKER = "↺ reopened this task"`.
- `reopen_badge(count) -> str` — `"↺{count}"`, empty string at zero, matching `plus_one_badge`'s contract.
- `close_record_label(record) -> str` — `"↺ Closed {closed_at} · {resolution or '(unrecorded)'}"`.
- `close_record_reopened_label(record) -> str` — cause-specific, absolute time only: `plus_one` →
  `"Reopened {reopened_at} by a +1 from @{reopened_by}"`; `open` → ``"Reopened {reopened_at} by `sase bead open`"``;
  `update` → `"Reopened {reopened_at} by a status update"`; `epic_preclaim` →
  `"Reopened {reopened_at} by an epic work preclaim"`. When `reopened_by` is absent for `plus_one`, fall back to the
  causeless `"Reopened {reopened_at} by a +1"`.
- `close_history_display_order(history) -> tuple[CloseRecord, ...]` — newest first.
- `close_history_search_text(history) -> str` — flattens reasons, resolutions, and timestamps for in-memory search
  indexes, matching `plus_one_evidence_search_text`.
- `evidence_reopened_bead(evidence, close_history) -> bool` — the `(reporter, timestamp)` join. Every surface that marks
  a +1 entry calls this and nothing else.

Unit tests cover the badge boundary at zero, each cause's label, the newest-first ordering, and the join: matching
entry, non-matching reporter, non-matching timestamp, empty history, and a history whose only record has a non-plus-one
cause.

### sase bead show, JSON, list badges, and search

In `src/sase/bead/cli_detail.py`:

- Append `[↺N]` after the existing `[+N]` badge on the header line, styled with `REOPEN_CLI_STYLE`.
- Add `_render_close_history_lines`, modeled on `_render_plus_one_evidence_lines`, emitting the block specified in the
  Design section. The section header uses `palette.accent(REOPEN_SECTION_LABEL, REOPEN_CLI_STYLE)` — the same treatment
  `+1 EVIDENCE` gets, not `palette.section`. The reason body goes through `_prose_lines` with indent `      ` so it
  wraps and highlights like every other prose block; a missing reason renders one `Reason: (none)` line using
  `palette.placeholder`. Insert the call immediately after the `RESOLUTION` block so ordering holds in both the reopened
  and the closed-again case.
- In `_render_plus_one_evidence_lines`, append `REOPEN_EVIDENCE_MARKER` to the label line of any entry for which
  `evidence_reopened_bead` is true.

The module's existing contract still holds and must be tested: for any detail, style, and wrap, stripping SGR escapes
from the output reproduces the `DetailStyle.PLAIN` bytes exactly.

In `src/sase/bead/cli_detail_json.py`: emit `close_history` unconditionally (like `plus_one_evidence`, not like the
conditional `refs`) as a list of objects with the wire field names, and add a derived `"reopened_bead": <bool>` to each
`plus_one_evidence` entry using the same shared join. Agents read this JSON to decide whether a duplicate is worth
reviving; making them re-derive the join is exactly how the two renderings drift apart.

In `src/sase/bead/cli_query.py`: add the `[↺N]` badge everywhere `plus_one_badge` is used for list, ready, blocked, and
search rows, and add `close_history_search_text` to the search index payload next to `plus_one_evidence`. In
`src/sase/bead/filter_query.py`, make the same text searchable. An archived close reason that cannot be found by
`sase bead search` is only half-recovered.

Update the affected goldens under `tests/test_bead/golden/cli/`. The unconditional `close_history` key churns every
`*_json.stdout` golden by one line; that is mechanical and expected. Add fixture coverage for a reopened bead so the new
rendering has a golden of its own rather than only appearing as an empty list.

### Prior-close warning in the TaskTriage gate

This is the highest-value surface in the epic: the gate asks the project owner to Launch or Close, Launch is the primary
branch, and a previously-closed task is precisely the case where the default is most likely wrong.

In `src/sase/bead/task_gate.py`:

- Thread `close_history` into `create_task_triage_gate`, `_build_task_triage_gate_spec` (payload), and
  `render_task_triage_preview`.
- Render a callout **above** `## Description`, so it is impossible to miss when skimming:

  ```markdown
  > [!WARNING] **↺ Previously closed 2026-07-30 as canceled** Not reproducible on main; the retry shim already covers
  > this.
  >
  > Reopened 2026-08-05 by a +1 from `@claude.probe`.
  ```

  With more than one record, render one callout per record, newest first.

- Mark the reopening entry in `_task_triage_evidence_preview` with `REOPEN_EVIDENCE_MARKER` using the shared join.
- Add the `↺N` badge to `task_triage_presentation_note` alongside the `+N` badge. This note is recomputed and
  byte-compared by gate validation, so everything added here must be deterministic and absolute-time only — which the
  close-history fields are.

In `src/sase/scripts/sase_chop_bead_task_triage.py`: add `close_history` to `_presentation_fingerprint`'s payload and
pass it through the `create_task_triage_gate` call. Without the fingerprint change, a bead whose close history changed
while a gate was pending would keep a stale preview; the docstring on that function already states the rule ("every
persisted field that changes the pending triage presentation").

Test that a pending gate is refreshed when close history appears, and that the rendered preview is stable across two
identical renders.

### ACE beads pane close history

In `src/sase/ace/tui/widgets/artifacts/beads_rendering.py`: append the `↺N` badge with `REOPEN_RICH_STYLE` next to the
existing `+N` badge on bead list rows.

In `src/sase/ace/tui/widgets/artifacts/beads_detail.py`: add a `("Previously closed", ...)` property near the existing
`Resolution` / `Close reason` properties, summarizing count and the most recent archived close; and add a
`## Previously Closed` section to `bead_body_markdown` mirroring the CLI block, placed above the description so the
detail pane and `sase bead show` agree on emphasis.

In `src/sase/ace/tui/widgets/artifacts/beads_filtering.py`: add a `reopened` label to `has_labels` when `close_history`
is non-empty (alongside the existing `+1` label), and add `close_history_search_text` to the haystack.

Cover the rendering with the existing ACE test patterns and add a PNG snapshot for the detail pane showing a reopened
bead. Follow the visual snapshot workflow in `CLAUDE.md`: `just test-visual`, artifacts in `.pytest_cache/sase-visual/`,
and `--sase-update-visual-snapshots` only to accept an intentional change.

### Generated bead pages close history

In `src/sase/bead_pages/rendering_identity.py`: add `render_close_history(issue)` next to `render_plus_one_evidence`,
producing a `## Previously Closed` section of blockquote callouts (newest first, absolute times, bounded prose through
the existing `_bounded_prose` helper), and add a `**↺ Reopened:**` fact to `_primary_facts` next to the existing
`**+1 reports:**` fact. Wire the new section into `src/sase/bead_pages/rendering.py` next to the
`render_plus_one_evidence` call, placed above the description for the same reason as every other surface.

In `src/sase/bead_pages/roster.py`: add a right-aligned `↺` column next to the existing `+1` column in the lineage
roster table.

### Document close history and reopen provenance

Update `docs/beads.md`:

- Extend **Task Corroboration (+1)**, which currently says a `+1` "clears stale close metadata" — after this epic it
  archives that metadata instead, and that sentence becomes wrong.
- Add a **Close History** subsection near the close/open documentation covering: what a record contains; the four reopen
  causes; that `close_history` holds only _undone_ closes while the current close stays in the flat fields; that it
  applies to every bead type; that `reopened_by` is currently populated only for `+1` reopens and why; and that existing
  stores recover their lost reasons from the event log on the next reduction with no backfill.
- Document the `↺N` badge next to the `+N` badge, the `PREVIOUSLY CLOSED` section in `sase bead show`, the
  `↺ reopened this task` marker, the `close_history` and derived `reopened_bead` keys in `--format json`, the ACE
  `has:reopened` filter, and that archived close reasons are searchable.
- Update the `sase bead +1`, `sase bead open`, and `sase bead close` command entries where they describe close-metadata
  handling.

## Sequencing

An approved epic, `Restore master CI to green after the sase-core 0.18 skew and the parallelism restoration`
(`~/.sase/plans/202608/ci_master_red_recovery.md`), is already in flight and moves the `sase-core-rs` window from
`>=0.17.15,<0.18.0` to `>=0.18.1,<0.19.0`, with a later phase raising the floor again for a commit-budget fix.

`core-adopt` sets the same window. Whichever lands second must take the higher floor rather than reverting the other's
bump — only one window ships. If `core-model`'s release crosses to `0.19.x`, the ceiling moves to `<0.20.0` and that
supersedes both of the CI epic's bumps; confirm the commit-budget fix is included in the release being adopted before
raising the ceiling, so adopting this epic's release does not silently drop it.

## Risks and non-risks

- **Not a risk: old readers.** `IssueWire` does not use `deny_unknown_fields`, so a sase build on an older core ignores
  a `close_history` key it does not know. Python's `_issue_from_dict` reads by key and ignores extras.
- **Managed: store churn.** `skip_serializing_if = "Vec::is_empty"` keeps every unaffected bead row byte-identical. The
  rows that do change are beads that were genuinely closed and reopened, and the change restores real data. The first
  mutation in a project after upgrade rewrites those rows and auto-commits them.
- **Managed: `--fix-projection` refusing the upgrade.** Handled by the `allowed_fields` change in `core-adopt`.
- **Watch: gate preview byte stability.** Everything the triage phase adds to the preview and the presentation note is
  absolute and derived from persisted fields, so gate re-derivation stays byte-stable. The fingerprint change is what
  keeps a pending gate from going stale.

## Out of scope

- **Bead event actors are wrong for close, open, and ready-marking.** `close_issues`, `open_issue`, and
  `set_ready_to_work` all pass `&issue.created_by` as the event actor, so `sase bead history` attributes those events to
  the bead's creator rather than to whoever ran the command. `append_note` and `add_task_plus_one` pass a real actor.
  This is a genuine defect in an audit trail and it is why this epic records no `closed_by` and populates `reopened_by`
  only for `+1` reopens. Fixing it means threading acting-agent identity through the close/open mutation API on both
  sides of the Rust boundary, and deciding what to do about legacy events already written with the wrong actor. It
  deserves its own task bead, not a rider on this one. The wire shape here is forward-compatible with that fix:
  `reopened_by` simply starts being populated for the other causes, and a `closed_by` can be added to
  `BeadCloseRecordWire` without disturbing anything already written.
- **Folding the current close into `close_history`.** The most uniform data model would make every close a record and
  derive the flat `closed_at` / `close_reason` / `resolution` from the open one. That is a much larger blast radius
  across the DB schema, the wire, every consumer, and every golden, for no gain at the triage moment this epic is about.
- **Changing +1 promotion semantics.** A `+1` on a closed task still promotes it to `ready`. The user confirmed that
  behavior is correct; only the provenance it destroyed is in scope.
- **A `sase bead reopened` query or a reopen-count filter in `sase bead list`.** The badge and the searchable text cover
  the discovery need; a dedicated query is speculative until the badge proves insufficient.
