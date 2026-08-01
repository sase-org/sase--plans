---
tier: epic
title: AXE Chop Reports
goal: 'Selecting a chop on the ACE AXE tab shows a beautiful, colored, width-responsive
  report of that run — a universal result card for every chop plus an optional chop-authored
  report document — and all four bugyi-chops scripts publish rich reports of their
  own.

  '
phases:
- id: contract
  title: Chop report document in the Rust core
  depends_on: []
  size: medium
  description: 'contract: add the optional `report` block document to the chop result
    wire in sase-core, with a closed block/tone/glyph vocabulary, fail-closed bounds
    validation, and Rust tests.

    '
- id: sdk
  title: ChopReport builder in the sase.chops SDK
  depends_on:
  - contract
  size: medium
  description: 'sdk: add a typed `ChopReport` builder plus `ChopResultBuilder.report`
    to the public `sase.chops` SDK so chop packages compose reports without hand-writing
    JSON or markup, and document the contract in docs/axe.md.

    '
- id: render
  title: AXE chop-run result card and report rendering
  depends_on:
  - contract
  size: medium
  description: 'render: build the shared block renderer and repaint the AXE chop-run
    detail pane as RESULT card, REPORT, then OUTPUT, width-responsive and cached,
    and reuse the same renderer for `sase axe chop run`.

    '
- id: visual
  title: PNG snapshot coverage for chop reports
  depends_on:
  - render
  size: small
  description: 'visual: add AXE-tab PNG snapshot fixtures and goldens covering a report-rich
    run, a report-less run, a failing run, and a narrow terminal.

    '
- id: chops
  title: Reports for every bugyi-chops chop
  depends_on:
  - sdk
  size: medium
  description: 'chops: author reports for ci_watch, toobig_split, recent_bug_audit,
    and recent_improvement_audit in the bugyi-chops repo behind one shared house style,
    and unify toobig_split''s clan summary with its report.

    '
- id: verify
  title: End-to-end verification on the real AXE tab
  depends_on:
  - visual
  - chops
  size: small
  description: 'verify: run each bugyi chop against the real configuration, confirm
    the AXE tab renders each report correctly at several widths, and confirm no chezmoi
    config change is required.

    '
create_time: 2026-07-29 09:49:51
status: done
bead_id: sase-ar
---

- **PROMPT:** [prompts/202607/axe_chop_reports.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/axe_chop_reports.md)
- **BEAD:** [sase-ar](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ar/README.md)

# Plan: AXE Chop Reports

## Problem

Selecting a chop on the ACE AXE tab shows only two things: a one-line header
(`src/sase/ace/tui/widgets/_axe_dashboard_status.py::_render_chop_display`) and the raw stdout tail of the selected run,
rendered as plain ANSI (`src/sase/ace/tui/widgets/axe_dashboard.py::update_chop_run_display`).

Everything a chop actually produced is already persisted on disk and already cached in memory, and none of it is shown.
`ChopRunEntry` (`src/sase/axe/_state_chops.py`) carries `result` (the full validated result document), `counters`,
`proposals`, `launches`, `evidence`, `reason`, `error`, `dry_run`, and `source`; the TUI's `ChopSnapshot` already holds
all of it (`src/sase/ace/tui/actions/axe_display/_data.py`). The AXE pane throws it away and prints log text.

Meanwhile agent clans do have a beautiful per-clan document: a chop attaches `clan_summary` to a proposal, the runner
threads it to the launched agent, and the Agents tab renders it with `Text.from_markup` inside the clan detail panel
(`src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan.py`). `bugyi_chop_toobig_split` already hand-builds such a
summary. Chops deserve the same treatment on their own tab.

## Design

Two layers, one of which needs no cooperation from chops at all.

### Layer 1 — the universal RESULT card (every chop, no chop changes)

The AXE chop-run pane opens with a card rendered by the TUI from `ChopRunEntry` alone: status, counters, reason, dry-run
and source markers, the proposals/launches roster, and evidence files. Builtin `sase_chop_*` chops, third-party chops,
and chops that never adopt reports all get this. This is what makes the feature feel finished instead of "four chops
look nice".

### Layer 2 — the chop-authored REPORT (opt-in, structured)

A chop may add an optional `report` to its result document. Taking the clan summary as inspiration but not as a
template, a report is **not** a Rich-markup blob. It is a small structured document of typed blocks carrying **semantic
tones**, never colors:

```json
{
  "schema_version": 1,
  "status": "ok",
  "summary": "ci_watch: repos=5 green=4 red=1",
  "counters": { "repos": 5, "green": 4, "red": 1 },
  "report": {
    "title": "CI WATCH",
    "blocks": [
      { "kind": "headline", "text": "4 green · 1 red · 1 fix proposed", "tone": "warn" },
      { "kind": "heading", "text": "REPOSITORIES" },
      {
        "kind": "rows",
        "columns": ["REPOSITORY", "STATE", "EVIDENCE"],
        "rows": [
          { "cells": ["sase-org/sase", "red", "ci / test · streak 2/2"], "tone": "error", "glyph": "▲" },
          { "cells": ["sase-org/sase-core", "green", "a1b2c3d"], "tone": "ok" }
        ]
      },
      { "kind": "divider" },
      { "kind": "kv", "items": [{ "key": "mode", "value": "dry run", "tone": "muted" }] }
    ]
  }
}
```

Choosing structure over markup is the core design decision of this epic, and it is what makes the result both reliable
and beautiful:

- **Beautiful by construction.** Tones map to the AXE palette in exactly one place, so every chop — present and future,
  first- and third-party — is automatically coherent with the sidebar taxonomy. A chop cannot emit a clashing hex color.
- **Width-responsive.** A markup blob has fixed line breaks and cannot reflow. Structured rows let the renderer compute
  column widths from content, elide long paths, and stack into a compact two-line-per-row layout on a narrow panel,
  matching the degradation the AXE pane already does at `_NARROW_OVERVIEW_WIDTH`
  (`src/sase/ace/tui/widgets/_axe_dashboard_output.py`).
- **Reliable.** No markup is ever parsed, so there is no `MarkupError` path, no accidental style injection, and no way
  for a chop's stray `[` to corrupt the pane. All strings render as literal text.
- **Frontend-agnostic.** Per the `rust_core_backend_boundary` memory, the document and its bounds live in Rust so any
  future frontend renders the same report; only the Rich rendering is presentation and stays in Python.

Because the runner already persists the whole parsed document verbatim (`src/sase/axe/chop_runner_script_result.py`
passes `parse_chop_result(...)` straight into `finish_chop_run(result=...)`) and the PyO3 layer serializes the wire
struct generically (`chop_result_to_py` in `crates/sase_core_py/src/lib.rs`), adding `report` to the Rust wire makes it
appear in `ChopRunEntry.result` with no new Python plumbing.

### Layout of the pane

`#axe-output-section` inside `#axe-output-scroll` becomes one document:

```
  ━━ RESULT ────────────────────────────────────────────────
   ✓ ok      1 proposal launched      dry run
   summary   ci_watch: repos=5 green=4 red=1
   counters  repos 5   green 4   red 1   fix_proposed 1
   launches  ci_fix.sase   ↗ launched
   evidence  chop.decisions.json

  ━━ CI WATCH ──────────────────────────────────────────────
   ... report blocks ...

  ━━ OUTPUT · 42 lines ─────────────────────────────────────
   ... existing ANSI log tail, unchanged ...
```

**No new keybinding.** The card and report sit above the log in the existing scroll region, so nothing is hidden behind
state the operator has to discover, there is no footer or help-modal churn, and the pane matches how the clan summary
sits inline in the clan detail document. The existing description banner above the scroll region is untouched.

### Non-goals

- No changes to chop scheduling, guards, triggers, dedupe, proposals, or launch behavior.
- No fold-level integration: `panel_fold_level` is Agents-tab machinery and stays there.
- No new AXE keybindings, footer entries, or help-modal sections.
- No new `sase axe` subcommands or CLI options.

## Cross-cutting rules for every phase

- Repos other than the current workspace checkout — `sase-core`, `chezmoi`, and `gh:bbugyi200/bugyi-chops` — MUST be
  opened with the `/sase_repo` skill, and only the path it prints may be read or written.
- In the sase repo, run `just install` before `just check`, and run `just check` before reporting completion.
- Never edit `sase/memory/*.md`, `AGENTS.md`, or the generated provider shims (`CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`,
  `QWEN.md`).
- Before fixing any Symvision lint failure, read `sase/memory/symvision.md` with the `/sase_memory_read` skill.
- TUI work must obey `sase/memory/tui_perf.md` (read it with `/sase_memory_read` first). In particular: **the render
  path performs no disk I/O**. Everything the card and report need is already in the cached `ChopSnapshot`; do not add a
  read, stat, or glob to any paint path.
- Keep new modules small; the repo enforces file-size limits at 700/850/1000 lines.

## Phase: contract

**Repo: `sase-core` (open with `/sase_repo`).** Files: `crates/sase_core/src/axe_chop/{wire,validation,tests}.rs`, and
`crates/sase_core_py/src/lib.rs` only if its module docstring inventory needs a line.

Add one optional field to `ChopResultDocumentWire` in `wire.rs`, following the existing convention of the neighboring
optional fields (`#[serde(default)]`, no `skip_serializing_if`, so an absent report serializes as `null` exactly like
`summary` does today):

```rust
#[serde(default)]
pub report: Option<ChopReportWire>,
```

`ChopResultDocumentWire` carries `#[serde(deny_unknown_fields)]`, which is why this field cannot be added from Python
alone and why this phase blocks the rest.

Define the report types in `wire.rs` with `deny_unknown_fields` on every struct, matching the surrounding style:

- `ChopReportWire { title: Option<String>, blocks: Vec<ChopReportBlockWire> }`
- `ChopReportBlockWire` — an internally tagged enum on `kind`, snake_case, with these variants and no others:
  - `headline { text: String, tone: Option<ChopReportToneWire> }`
  - `heading { text: String }`
  - `text { text: String, tone: Option<ChopReportToneWire> }`
  - `kv { items: Vec<ChopReportKvItemWire> }` where an item is `{ key: String, value: String, tone: Option<...> }`
  - `rows { columns: Option<Vec<String>>, rows: Vec<ChopReportRowWire> }` where a row is
    `{ cells: Vec<String>, tone: Option<...>, glyph: Option<String> }`
  - `bullets { items: Vec<ChopReportBulletWire> }` where a bullet is
    `{ text: String, tone: Option<...>, glyph: Option<String> }`
  - `gauge { label: String, value: i64, max: i64, tone: Option<...> }`
  - `divider {}`
- `ChopReportToneWire` — a snake_case enum with exactly: `neutral | muted | info | ok | warn | error | accent`.

Validation lives in `validation.rs`, called from `validate_chop_result` right after the existing
`validate_optional_text` calls, so both `parse_chop_result` and `validate_chop_result` enforce it. Fail closed, in the
established `ChopEngineError::new(code, path, message)` style with JSON-pointer style paths such as
`$.report.blocks[3].rows[7].cells[1]`:

- The whole report, re-serialized, must not exceed `CHOP_REPORT_MAX_BYTES = 32 * 1024` — reuse the same 32 KiB ceiling
  already used by `CLAN_SUMMARY_MAX_BYTES` in this module.
- `blocks` must be non-empty and hold at most 48 blocks.
- Every string field (title, text, heading, key, value, cell, label, column name) must be non-blank where required, must
  not contain a NUL byte, must not contain any control character **including `\n`, `\r`, and `\t`** (the block model
  owns all line breaking, which is what keeps layout deterministic), and must be at most 512 characters. `title` is
  capped at 64 characters.
- `kv.items`, `rows.rows`, and `bullets.items` must be non-empty and hold at most 64 entries.
- `rows.columns`, when present, must hold 1–6 non-blank names, and every row's `cells` length must equal the column
  count. When `columns` is absent, every row must have the same non-zero cell count as the first row, capped at 6.
- `glyph` must be exactly one character drawn from a fixed allowlist — `▲ ◆ • · ● ○ ✓ ✗ ↗ ↷ ⏱ ! ▸ ─` — so a chop cannot
  smuggle control characters or ambiguous-width emoji that would break column alignment. Reject anything else rather
  than silently substituting.
- `gauge` requires `max > 0` and `0 <= value`; `value` may exceed `max` (an over-budget gauge is meaningful and the
  renderer clamps the bar).

Keep `CHOP_RESULT_SCHEMA_VERSION` at 1: this is a purely additive optional field, and the existing version check already
rejects documents from a different major schema. Note explicitly in the doc comment that an unknown block kind is
rejected fail-closed, consistent with this module's posture, so a newer plugin emitting a newer block kind fails loudly
on an older SASE rather than rendering a partial report.

Add Rust tests to `axe_chop/tests.rs` covering: a valid full report round-tripping through `parse_chop_result`; absent
`report` still parsing; unknown block kind rejected; unknown tone rejected; embedded newline rejected; over-long string
rejected; oversize report rejected; too many blocks and too many rows rejected; ragged `cells` versus `columns`
rejected; disallowed glyph rejected; `gauge` with `max = 0` rejected.

Verify the crate with the repo's own commands (`cargo test -p sase_core`, plus `cargo fmt` and `cargo clippy` as that
repo's tooling requires).

Because dev installs of sase build the extension from the local `sase/repos/linked/sase-core` checkout (see `Justfile`,
`just rust-install`), later phases pick this up automatically after `just install`. Bumping the published
`sase-core-rs>=0.12.5,<0.13.0` window in sase's `pyproject.toml` is a release concern and is **out of scope**; note it
in the phase's completion report so the release is not forgotten.

## Phase: sdk

**Repo: sase.** Files: new `src/sase/chops/report.py`, plus `src/sase/chops/sdk.py`, `src/sase/chops/__init__.py`,
`docs/axe.md`, and `tests/test_chop_sdk.py`.

Give chop authors a typed builder so nobody hand-writes report JSON. This is what keeps every chop's report visually
consistent — the house style lives in the SDK, not in each chop.

```python
from sase.chops import ChopReport, ChopResultBuilder

report = ChopReport(title="CI WATCH")
report.headline("4 green · 1 red · 1 fix proposed", tone="warn")
report.heading("REPOSITORIES")
table = report.rows(columns=("REPOSITORY", "STATE", "EVIDENCE"))
table.row(("sase-org/sase", "red", "ci / test · streak 2/2"), tone="error")
table.row(("sase-org/sase-core", "green", "a1b2c3d"), tone="ok")
report.divider()
report.kv({"mode": "dry run"}, tone="muted")

result = ChopResultBuilder(status="ok", summary=line, report=report)
result.write(context=invocation.context)
```

Requirements:

- `ChopReport` is a mutable builder with one method per block kind: `headline`, `heading`, `text`, `kv`, `rows`
  (returning a small row-collector handle), `bullets`, `gauge`, `divider`. Every method returns something chainable and
  accepts an optional `tone`.
- `Tone` is a `Literal["neutral", "muted", "info", "ok", "warn", "error", "accent"]`. There is no API for supplying a
  color. When a caller omits `glyph`, omit it in the document too and let the renderer choose from the tone.
- The builder normalizes defensively so a chop cannot accidentally emit an invalid document: collapse internal
  whitespace and strip control characters from every string, truncate to the documented limits with a trailing `…`, drop
  empty blocks, and refuse to add a row whose cell count disagrees with the declared columns (raise `ValueError` at
  build time, where the chop author sees it, rather than producing a document the runner rejects at parse time).
- `ChopReport.to_dict()` returns the JSON-compatible document; `ChopResultBuilder` gains a
  `report: ChopReport | Mapping[str, Any] | None = None` field and emits `report` in `to_dict()` only when present.
  `write()` continues to route through `validate_chop_result`, so the Rust bounds from the `contract` phase are the
  single source of truth and the builder's own limits are only a friendlier first line of defense.
- Export `ChopReport` and the tone type from `sase.chops.__init__` and from `sdk.__all__`.

Document the contract in `docs/axe.md`, in the "Structured Results and Launch Proposals" section directly after the
`clan_summary` paragraphs: the `report` field, the block kinds and their fields, the tone vocabulary and what each tone
means semantically, the glyph allowlist, every bound, that tones — not colors — are what a chop supplies, and that
reports render on the ACE AXE tab. Extend the existing "Chop output is part of the operator contract" paragraph to say
that a chop with a meaningful structured story should publish a report while keeping its compact stdout summary line
unchanged, since the summary line still drives logs and notifications.

Add tests to `tests/test_chop_sdk.py`: each block kind round-trips; tone validation; truncation and control-character
stripping; mismatched cell count raises; a builder-produced document passes `validate_chop_result`; a result without a
report is byte-identical to today's output.

Do not change `emit_summary`, `ChopLogger`, `parse_summary`, or any existing SDK behavior — chops that never adopt
reports must be completely unaffected.

## Phase: render

**Repo: sase.** New module `src/sase/axe/chop_report_render.py` (shared block renderer), new module
`src/sase/ace/tui/widgets/_axe_chop_result_card.py` (the RESULT card), and edits to
`src/sase/ace/tui/widgets/axe_dashboard.py`, `src/sase/ace/tui/widgets/_axe_dashboard_output.py`,
`src/sase/axe/chop_render.py`, `docs/axe.md`, and `docs/ace.md`.

### Shared block renderer

`src/sase/axe/chop_report_render.py` turns a report document into a `rich.text.Text`:

```python
def render_chop_report(report: Mapping[str, Any], *, width: int | None = None) -> Text: ...
```

- One module-level `dict[str, str]` maps tone to style, drawn from the palette already used across the AXE pane
  (`_axe_dashboard_render.py`, `axe_log_renderer.py`, `bgcmd_list.py`): lumberjack gold `#FFD700`, chop copper
  `#D7AF87`, label blue `#87D7FF`, value teal `#00D7AF`, plus red `#FF5F5F` for `error`, amber `#FFAF5F` for `warn`,
  green `#5FD75F` for `ok`, and `dim #A8A8A8` for `muted`. Keep every color literal in this one map so the palette can
  be retuned in one edit.
- A second map gives each tone its default glyph (`ok → ✓`, `warn → ◆`, `error → ▲`, `info → •`, `muted → ·`,
  `neutral`/`accent` → none), used whenever a block omits `glyph`.
- `title`, when present, renders as a section rule (`━━ TITLE ────…`) in the chop copper accent, matching the `RESULT`
  and `OUTPUT` rules described below.
- `rows` computes each column's width from its content, right-aligns a column whose every cell is numeric, left-aligns
  otherwise, and elides over-long cells with a leading `…/` for path-like cells and a trailing `…` otherwise. Reuse the
  elision approach already proven in `bugyi_chops/toobig_split.py::_elide_path`.
- `gauge` renders a proportional bar built from `█` and `░` at a fixed cell width, clamped when `value > max`, followed
  by `value/max`.
- Narrow mode: when `width` is a positive value below the existing `_NARROW_OVERVIEW_WIDTH` (60), `rows` stacks into a
  header line plus an indented detail line per row and `kv` stacks one pair per line, exactly like
  `render_compact_chop_list` does today. Never truncate mid-value.
- The renderer treats every string as literal text; it must not call `Text.from_markup` or `Text.from_ansi` anywhere.
- Unknown block kinds and unknown tones are skipped silently here (Rust already rejects them on the write path; the
  renderer is the last line of defense against a hand-edited state file and must never raise).

### The RESULT card

`_axe_chop_result_card.py` builds the card from a `ChopRunEntry`, with no report required:

- A `━━ RESULT ──…` rule, then a status line combining the status label and style from the existing
  `chop_status_label()` with launch/proposal counts, a `dry run` marker when `entry.dry_run`, and a `manual`/`oneshot`
  marker when `entry.source` is not `scheduled`.
- `summary` and `reason` lines from `entry.result` when present.
- `counters` from `entry.result["counters"]` as aligned `name value` chips, wrapping across lines to the available
  width, values in the teal value style.
- `launches` and `proposals` from `entry.launches` / `entry.proposals`: agent name, clan when set, and outcome, one per
  line, capped at 12 with an `…and N more` tail.
- `evidence` file names from `entry.result["evidence"]`.
- `error` and `traceback` when present, in the error style, so a `check_error` run explains itself at the top of the
  pane instead of only inside the log.
- Omit any line whose data is absent; a chop that only exits zero shows a two-line card, not a wall of `-` placeholders.

### Wiring the pane

In `AxeDashboard.update_chop_run_display`, replace the current single call to `output_section.update_display(...)` with
a composed document: the RESULT card, then the rendered report when `entry.result` carries a non-null `report`, then a
`━━ OUTPUT · N lines ──…` rule and the existing ANSI-rendered log tail produced exactly as today by
`render_axe_output(source_id, run.output_tail, "ansi")`. Preserve today's behavior for a run with no output at all (the
`Waiting for output…` / error / reason / `Run captured no output.` fallbacks) — those messages move below the OUTPUT
rule. The empty state for a chop with no recorded runs is unchanged.

Pass the section's rendered width down using the existing `section_width()` helper so the report degrades on a narrow
panel just like the lumberjack overview does.

Performance, per `sase/memory/tui_perf.md`:

- The card and report render entirely from the already-cached `ChopSnapshot`. Add **no** disk read, stat, or glob to any
  paint path.
- Cache the composed card+report `Text` in a module-level dict keyed by
  `(lumberjack_name, chop_name, run_id, status, finished_at, width_bucket)`, mirroring the tail-biased cache in
  `axe_log_renderer.py`. Including `status`/`finished_at` keeps a live `running` run repainting while a terminal run is
  served from cache. Bound the cache the same way the existing render cache is bounded.
- Scroll behavior: `_refresh_axe_display` currently force-scrolls to the bottom whenever `_axe_pinned_to_bottom` is set
  (`src/sase/ace/tui/actions/axe_display/_render.py`). For a chop run that has reached a terminal status, do not force
  scroll-to-bottom — otherwise the card and report are scrolled off screen the instant the chop is selected. Keep the
  follow-the-tail behavior for runs whose status is `running` or `launched`, where the operator is watching live output.

### CLI reuse

`sase axe chop run` already prints a structured result panel via `render_chop_run_result()` in
`src/sase/axe/chop_render.py`. Append the rendered report there when the result carries one, using the same
`render_chop_report()`. One renderer, two frontends, no drift between what an operator sees in the terminal and on the
AXE tab.

### Tests and docs

Extend `tests/ace/tui/widgets/test_axe_dashboard_chop_detail.py` and add focused unit tests for the renderer and the
card: each block kind; the tone-to-style map; numeric column right-alignment; cell elision; narrow-mode stacking;
unknown kind/tone skipped without raising; a run with no report renders card plus output only; a `check_error` run
surfaces its error in the card; counters and launches render; the cache key changes when the run's status changes.
Update `docs/ace.md`'s AXE-tab section and the `docs/axe.md` chop-run-history text to describe the three-section pane.

## Phase: visual

**Repo: sase.** Files: `tests/ace/tui/visual/_ace_axe_png_snapshot_fixtures.py`, the AXE PNG snapshot test module that
consumes those fixtures, and new goldens under `tests/ace/tui/visual/snapshots/png/`.

The existing AXE fixtures (`axe_lumberjack_tree_data`, `axe_running_chop_data`, `axe_disabled_chop_data`, …) already
build `AxeCollectedData` with `ChopRunEntry` values, so adding report-bearing entries is a matter of populating
`result`. Add fixtures and goldens for:

- `axe_chop_report_rich_120x40` — a chop run whose report exercises headline, heading, a multi-column `rows` block with
  mixed tones and glyphs, `kv`, `gauge`, and `divider`, with a short log tail below the OUTPUT rule.
- `axe_chop_report_absent_120x40` — a successful run with counters, launches, and evidence but no report, proving the
  RESULT card carries the pane on its own.
- `axe_chop_report_error_120x40` — a `check_error` run whose error and reason surface in the card.
- `axe_chop_report_narrow_70x36` — the same rich report at a narrow width, showing stacked rows and stacked `kv` pairs
  with nothing truncated mid-value.

Follow the existing snapshot workflow: `just test-visual`, inspect `.pytest_cache/sase-visual/` artifacts on a mismatch,
and accept intentional new goldens with `--sase-update-visual-snapshots`. **Look at each generated PNG** before
accepting it — these goldens are the acceptance criterion for "beautiful", not just a regression net. Fix alignment,
spacing, and color-balance problems in the `render` phase's renderer rather than papering over them in fixtures.

## Phase: chops

**Repo: `gh:bbugyi200/bugyi-chops` (open with `/sase_repo`).** Files: new `src/bugyi_chops/_report.py`, plus
`ci_watch.py`, `toobig_split.py`, `recent_audits.py`, `_common.py`, `pyproject.toml`, `README.md`, and the corresponding
modules under `tests/`.

First, the dependency floor. `pyproject.toml` currently pins `sase>=0.12.0,<0.13.0` while sase is already at 0.13.2, so
the pin is stale independent of this work. Raise it to the sase version that first ships `ChopReport`
(`sase>=0.13.<minor>,<0.14.0`, using the actual released version) and update the matching sentence in `README.md`. Do
not add a runtime `hasattr` fallback — a hard version floor is the honest contract.

Add `src/bugyi_chops/_report.py` holding this package's house style on top of `ChopReport`: a `start_report(title)`
helper, a shared severity-to-tone mapping, a shared path-elision helper, and a shared "facts footer" helper. Every chop
below builds through it so all four reports look like they belong to one family. Wire the report into
`_common.py::result_with_summary` (accept an optional `report=` and attach it to the returned `ChopResultBuilder`) so no
chop touches the result plumbing directly.

### `ci_watch`

The richest report, and the one that pays for the whole feature. `build_ci_watch_result` already computes everything
needed: `states`, `heads`, `failures`, `streaks`, `counters`, `ledger_repos`, `release_plans`, and `mode`. Build the
report from that ledger, immediately before the existing `result.add_evidence(...)` call:

- Title `CI WATCH`. Headline tallying green/red/pending/error counts with tone `error` when any repo is red, `warn` when
  any is pending or errored, `ok` otherwise.
- Heading `REPOSITORIES` and a `rows` block with columns `REPOSITORY`, `STATE`, `EVIDENCE`. Tone and glyph per state:
  green → `ok`/`✓`, red → `error`/`▲`, pending → `warn`/`◆`, error → `error`/`!`, no_ci → `muted`/`·`. The evidence cell
  carries the short head SHA for green repos; for red repos it carries the failing job names (bounded, joined with `·`)
  and the debounce streak as `streak N/M`; for suppressed repos it carries the suppression reason (`red_debounce`,
  `fix_in_flight`, `fix_cap_reached`, `fix_disabled`, `agents_check_failed`).
- Heading `RELEASE` and a `rows` block with columns `REPOSITORY`, `PR`, `DECISION`, one row per release-eligible repo,
  tone `ok` when merged, `muted` when skipped for a benign reason, `warn` when skipped for a blocking reason. Omit the
  whole section when no repo has a release generator configured.
- A `divider`, then a `kv` footer: `mode` (`live` / `dry run` / `unavailable`, tone `warn` when not live),
  `agents running`, `fix cap`, `merge cap`, and `debounce ticks`.
- The `check_error` path in `run_chop` must still produce a valid result; give that path a minimal one-headline report
  explaining the failure rather than no report.

### `toobig_split`

This chop already hand-builds a Rich-markup clan summary in `_render_clan_summary`. Refactor so the severity
classification, sort order, elision, and row content are computed **once** into a shared list of rows, and both the clan
summary markup and the new report are projected from it. The clan panel and the AXE pane then stay in visual lockstep by
construction — which is exactly the relationship the two views should have.

- Title `TOOBIG SPLIT`. Headline `N files over limits` with tone `error` when any file exceeds the hard limit, `warn`
  otherwise.
- Heading `TARGETS`, then a `rows` block with columns `LINES`, `FILE` — `LINES` is numeric so the renderer right-aligns
  it — one row per file, tone and glyph from the existing violation/warning/fyi/neutral classification, capped at the
  existing `CLAN_SUMMARY_MAX_ROWS` with an `…and N more` text block when overflowing.
- A `gauge` for the worst offender: `label` the elided path, `value` its line count, `max` the hard limit — an immediate
  visual read on how far over budget the largest file is.
- A `kv` footer: scan roots, limits, and queue mode (`sequential`).
- The `no_files_over_limits` no-op path gets its own small report: one `ok`-toned headline saying every scanned file is
  within limits, plus the limits `kv` footer. A no-op run should still look intentional and informative on the AXE tab.

### `recent_bug_audit` and `recent_improvement_audit`

Both flow through `build_audit_result`, so add the report once there and drive the differences from `AuditKind`. Extend
`AuditKind` with `report_title`, `scope_bullets`, and `exclusion_bullets` tuples, populated from the prose already
sitting in each kind's `instructions`.

- Title from `report_title` (`RECENT BUG AUDIT` / `RECENT IMPROVEMENT AUDIT`).
- Headline naming the project and the audit subject.
- Heading `SCOPE` and a `kv` block: `project`, `workspace`, `head` (short SHA, or `unknown` toned `muted` when
  `git_head` returned nothing), `pr`, and `agent`.
- Heading `LOOKING FOR` and a `bullets` block from `scope_bullets`, tone `info`.
- Heading `OUT OF SCOPE` and a `bullets` block from `exclusion_bullets`, tone `muted`.

### Tests

Extend `tests/test_ci_watch.py`, `tests/test_toobig_split.py`, and `tests/test_recent_audits.py`: every chop emits a
report whose document validates through `sase.chops`; ci_watch's row tones follow repo state; toobig_split's report rows
and clan-summary rows are derived from the same source and stay consistent; the no-op and `check_error` paths still
produce valid results with a report. Run this repo's own `just` targets for lint and tests.

## Phase: verify

**Repos: sase (primary), `gh:bbugyi200/bugyi-chops`, `chezmoi` — the latter two opened with `/sase_repo`.**

1. In the sase workspace, `just install` (so the local sase-core build and the chops SDK are both current) and
   `just check`.
2. Install the updated bugyi-chops into the same managed environment (`sase plugin install bugyi-chops -g`, per that
   repo's README) and run each chop in preview mode:
   `sase axe chop run 'toobig_split[sase]' -L <lumberjack> --dry-run --chop-verbose`, and likewise for `ci_watch`,
   `recent_bug_audit`, and `recent_improvement_audit`, using the lumberjack names configured in chezmoi
   (`home/dot_config/sase/sase_athena.yml` for the bugyi chops; `home/dot_config/sase/sase.yml` for the Telegram lane).
   Confirm the CLI prints the report panel and that the result documents validate.
3. Open `sase ace`, go to the AXE tab, select each of the four chops, and confirm the pane renders RESULT, the report,
   and OUTPUT correctly. Check at a wide terminal and again below 60 columns. Confirm a builtin chop with no report (for
   example a `sase_chop_*` chop under the Telegram or docs lumberjack) still renders a clean RESULT card followed by its
   log.
4. Confirm navigation stays responsive with `SASE_TUI_PERF=1` while moving up and down the chop rows — p95 key-to-paint
   should stay under 16 ms, per `docs/perf_runbook.md`.
5. Confirm **no chezmoi change is required**: reports are additive to the result document and need no config. Read
   `home/dot_config/sase/sase_athena.yml` and `home/dot_config/sase/sase.yml` to confirm nothing pins a chop result
   shape or a `sase` version that this work invalidates. Report the finding either way; if a change turns out to be
   needed, make the smallest one that works and say so explicitly.
6. Report the outstanding release follow-ups noted by earlier phases: publishing `sase-core-rs` and bumping its window
   in sase's `pyproject.toml`, and publishing bugyi-chops with its raised `sase` floor.
