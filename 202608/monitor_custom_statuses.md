---
tier: epic
status: done
title: Required custom monitor statuses with deterministic pair colors
goal: "Every sase monitor declares its own running and finished status labels, those
  labels are capped at 20 characters, and they appear -- in one deterministic,
  pair-derived color -- on every surface that shows a monitor, including agent family
  container rows and the Admin Center Procs tab.

  "
phases:
  - id: contract
    title: Monitor status contract module
    depends_on: []
    size: medium
    description: "contract: add the shared sase.monitor_status module that owns the
      default labels, the 20-character clamp, pair normalization, the 12-color accent
      palette, and the state-aware style rule; retire the scattered MONITORING/MONITORED
      literals onto it.

      "
  - id: cli
    title: Required start and stop status flags
    depends_on:
      - contract
    size: medium
    description: "cli: make -s/--start-status and -S/--stop-status required on sase
      monitor start, clamp over-length labels with a warning, make the two fields
      required on StartMonitorRequest, and render the effective label in sase monitor
      list, show, markdown, and JSON.

      "
  - id: model
    title: Status pair plumbing and terminality
    depends_on:
      - contract
    size: medium
    description: "model: carry monitor_stop_status alongside monitor_start_status
      through the scan-wire and filesystem loaders, the TUI Agent row, RunningAgentInfo,
      the integrations entry, and the mobile summary; fix settled monitors with custom
      stop labels being treated as non-terminal.

      "
  - id: agents_tab
    title: Agents tab and agent list coloring
    depends_on:
      - model
    size: medium
    description: "agents_tab: color the monitor status token by its pair accent in the
      ACE agent list, add the outcome glyphs that replace the retired success green, add
      a Status field to the prompt panel MONITOR section, and color the sase agent list
      STATUS badge.

      "
  - id: family
    title: Agent family container status
    depends_on:
      - model
    size: small
    description: "family: mirror a monitor's custom status onto its agent family
      container row for plain families as well as plan roots, and copy the status pair
      so the mirrored row renders in the same accent as the monitor itself.

      "
  - id: procs
    title: Procs tab monitor status chip
    depends_on:
      - model
    size: small
    description: "procs: resolve each monitor proc row's status pair from the loaded
      agent rows and render the effective label as an accent-colored chip in the Admin
      Center Procs tab row labels and output header.

      "
  - id: guidance
    title: Guidance, skill, and docs
    depends_on:
      - cli
      - agents_tab
    size: small
    description:
      "guidance: teach TESTING/TESTED in the build-and-run memory note, make the
      sase_monitor skill and monitors/ACE/CLI docs state that both flags are required,
      and regenerate the derived instruction files."
proposed_by: bbugyi200.athena.07k
bead_id: sase-qv
create_time: 2026-09-09 19:50:55
---

- **PROMPT:**
  [prompts/202608/monitor_custom_statuses.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/monitor_custom_statuses.md)
- **BEAD:**
  [sase-qv](https://github.com/sase-org/sase--beads/blob/main/pages/sase-qv/README.md)

# Plan: Required custom monitor statuses with deterministic pair colors

## Problem

`sase monitor start` already accepts `-s/--start-status` and `-S/--stop-status`, but
both are optional and default to `MONITORING` / `MONITORED`. In practice every monitor
looks identical: a wall of `MONITORING` rows that say nothing about _what_ is being
waited on. The feature exists but nothing makes anyone use it.

Three consequences follow:

1. **Nobody opts in.** The one place we tell agents to run monitors —
   `sase/memory/build_and_run.md`, for `just check-full` — says nothing about statuses.
2. **The labels are half-plumbed.** `monitor_stop_status` never reaches the ACE `Agent`
   row at all, the Admin Center Procs tab shows no status, `sase monitor list` shows the
   machine state (`running`) rather than the human label, and `sase agent list` renders
   custom labels with no color.
3. **Arbitrary labels quietly break status semantics.** `_TERMINAL_STATUSES` in
   `src/sase/agent/status_buckets.py:132` contains the literal `"MONITORED"`, so a
   monitor that finishes as `TESTED` is not recognized as terminal by
   `date_anchor_time()` (`src/sase/ace/tui/models/agent_groups/_buckets.py:84`) and
   anchors on `start_time` instead of `stop_time`. That is a latent bug today; making
   custom labels mandatory makes it universal.

Meanwhile the 48-character cap in `src/sase/main/monitor_handler.py:59` _rejects_
over-length labels, which turns a cosmetic problem into a failed monitor start.

## Design

### The pair is the unit

A monitor's identity, for display purposes, is the **ordered pair** of its two labels:
`(start_status, stop_status)` — for example `(TESTING, TESTED)`. Everything below keys
off that pair rather than off either label alone:

- One deterministic accent color per pair, so every `TESTING`/`TESTED` monitor is the
  same color in the Agents tab, the Procs tab, `sase monitor list`, and
  `sase agent list`.
- Present participle → past participle is the recommended convention (`TESTING` →
  `TESTED`, `SLEEPING` → `SLEPT`, `DEPLOYING` → `DEPLOYED`), so the two halves read as
  one thing in two tenses.

### Three orthogonal visual signals

Making labels arbitrary means the _word_ can no longer carry the outcome, so the
existing "green means it worked, red means it did not" convention has to be re-expressed
without stealing the accent hue. The rule is:

| signal     | carries                         | mechanism                                                            |
| ---------- | ------------------------------- | -------------------------------------------------------------------- |
| **hue**    | _which_ kind of monitor this is | pair accent                                                          |
| **weight** | live or settled                 | `bold` while running, normal weight once settled                     |
| **glyph**  | how it went                     | `✓` completed, `⊘` stopped, `✗ <code>` failed, `⧖` timeout, `⚠` lost |

with exactly one override: **failure keeps red.** `failed`, `timeout`, and `lost` render
`bold #FF5F5F` regardless of the pair accent, because a failed monitor must be
unmistakable at a glance. The accent palette therefore reserves the red hue wedge so a
_running_ monitor can never be mistaken for a failed one.

This is the same weight-plus-hue reasoning already documented for the settled gear badge
in `src/sase/ace/tui/widgets/_agent_list_styling.py:105`, extended to the status token.

### Determinism without an enumerable key set

`project_accent()` (`src/sase/ace/tui/project_styles.py`) hashes an identifier into a
frozen palette and, because the enabled-project set _is_ enumerable, forward-probes past
collisions to guarantee uniqueness. Monitor status pairs are an open set, so this plan
takes the same hash-into-a-frozen-palette primitive but **hash-only, with no
`among=`-style probing**:

> A pair's color must be identical in the Agents tab, the Procs tab, and both CLIs. If
> it depended on which other monitors happened to be visible, the same monitor would
> change color when you switched tabs. Global stability beats guaranteed uniqueness.

Two different pairs can therefore collide on one color. That is acceptable and must be
documented: the _words_ still differ, and the collision rate for a 12-color palette at a
realistic handful of live pairs is low.

The `sha256(key)[:8] % len(palette)` primitive is currently duplicated in
`project_styles._hash_index` and `_artifact_tab_descriptors._provider_accent_for_kind`
(`src/sase/ace/tui/_artifact_tab_descriptors.py:289`). This plan extracts it once and
has all three call sites share it.

### Cap and truncation

The user asked for a 20-character cap with the overflow replaced by an ellipsis. This
plan uses `…` (U+2026) rather than three periods: it is one terminal cell instead of
three, and it is already this repo's truncation glyph (`_ELLIPSIS` in
`src/sase/task_type_gate_presentation.py:34`, Rich `overflow="ellipsis"` throughout the
TUI). **If three periods are wanted instead, only `MONITOR_STATUS_ELLIPSIS` changes.**

The clamp is applied at the write boundary so the durable `agent_meta.json` already
holds a ≤20-character label and no reader has to re-truncate, and defensively on read so
monitors written before this change (up to 48 characters) still render inside the cap.

### What this plan does not change

- **No `sase-core` change is required.** `crates/sase_core/src/agent_scan/wire.rs`
  already carries `monitor_start_status` and `monitor_stop_status` as opaque
  `Option<String>` pass-throughs, and `scanner.rs` already reads both. The clamp is a
  Python write-side validation rule on a Python-written marker, and the accent palette
  is presentation — the same call this repo already makes for `project_accent()`, which
  the CLI (`src/sase/main/project_handler_current.py`) imports from the TUI package.
- **No durable proc-store schema change.** The Procs tab already joins monitor rows to
  loaded agent rows by `monitor_id` (`_resolve_monitor_agent_names` in
  `src/sase/ace/tui/modals/procs_pane_selection.py:36`); the status pair rides that same
  join. Adding fields to `Proc` would mean bumping `PROC_WIRE_SCHEMA_VERSION` in both
  Python and `crates/sase_core/src/procs/wire.rs` plus the parity test, for no benefit.

## Monitor status contract module

Add `src/sase/monitor_status.py` — a new top-level module beside the existing
`src/sase/monitor_state.py`, deliberately free of `rich` and of any `sase.ace` import so
the monitor core, both CLIs, the integrations layer, and the TUI can all import it.

### Constants

```python
DEFAULT_MONITOR_START_STATUS = "MONITORING"
DEFAULT_MONITOR_STOP_STATUS = "MONITORED"
MONITOR_STATUS_MAX_CHARS = 20
MONITOR_STATUS_ELLIPSIS = "…"
MONITOR_STATUS_FAILURE_STYLE = "bold #FF5F5F"
```

`DEFAULT_MONITOR_STOP_STATUS` already exists in `src/sase/monitor_state.py:7`; move it
here and re-export from `monitor_state` so existing importers keep working. These remain
**read-side fallbacks only** — they are what a pre-change or externally written record
projects to. They are no longer what a new monitor gets by omission.

### Clamping

```python
def clamp_monitor_status(value: str) -> str
```

Strips surrounding whitespace and returns at most `MONITOR_STATUS_MAX_CHARS` characters.
When the stripped value is longer, return `value[:MAX - 1] + MONITOR_STATUS_ELLIPSIS` so
the result is exactly `MAX` characters including the ellipsis. Raise `ValueError` for an
empty value or one containing `\n`/`\r` — truncation handles length, but an empty or
multi-line status is a caller mistake, not an overflow.

Add a companion `clamp_monitor_status_or_default(value, *, default)` that returns
`default` for `None`/empty and never raises, for read paths projecting historical
records.

### Pair normalization and accents

```python
@dataclass(frozen=True, slots=True)
class MonitorStatusPair:
    start: str
    stop: str

    @property
    def key(self) -> str: ...   # f"{start.upper()}{stop.upper()}"

def monitor_status_pair(start: str | None, stop: str | None) -> MonitorStatusPair
```

`monitor_status_pair` clamps both halves, filling each missing half from its default,
and uppercases only for the hash key (the displayed text keeps the author's casing).
Keying case-insensitively means `Testing`/`TESTING` are one identity.

```python
MONITOR_STATUS_ACCENTS: tuple[str, ...] = (
    "#FEA775",  # oklch h=50.0
    "#F8AD08",  # oklch h=77.3
    "#CCBF08",  # oklch h=104.5
    "#81D005",  # oklch h=131.8
    "#0BD68B",  # oklch h=159.1
    "#00D2C4",  # oklch h=186.4
    "#0BCDEC",  # oklch h=213.6
    "#6FC4FF",  # oklch h=240.9
    "#A1BAFF",  # oklch h=268.2
    "#C4B0FE",  # oklch h=295.5
    "#F39CFE",  # oklch h=322.7
    "#FF9ECD",  # oklch h=350.0
)

def monitor_status_accent(pair: MonitorStatusPair) -> str
```

Palette construction, to be recorded in the module docstring so it can be regenerated:
12 hues spaced evenly across the OKLCH arc `50°..350°`, each chroma-solved to a shared
WCAG relative luminance of **0.50**. The reserved wedge `350°..50°` is the red band that
`MONITOR_STATUS_FAILURE_STYLE` owns. 0.50 is the center of the band the agent list's
existing status colors already occupy (`#5FD75F` 0.519, `#FFAF5F` 0.528, `#00D7AF`
0.517, `#FFAF00` 0.519), so a monitor status token reads at the same visual weight as
`RUNNING` or `DONE` beside it. Every entry clears a contrast ratio of at least **8.7**
against the app's dark shell surfaces (`#121212` / `#1E1E1E`).

> Tradeoff to record in the docstring: like every other agent-list status color in this
> repo, this band is tuned for the dark shell and only reaches ~1.4:1 against the light
> surfaces `#E0E0E0`/`#D8D8D8`. Matching `PROJECT_ACCENTS`' dual-surface 0.196 band
> instead would make monitor statuses visibly darker than every status beside them.
> Dark-surface parity was chosen; revisit only if the whole agent-status palette moves.

`monitor_status_accent` is
`MONITOR_STATUS_ACCENTS[hash_palette_index(pair.key, len(MONITOR_STATUS_ACCENTS))]`.

### The style rule

```python
def monitor_status_style(pair: MonitorStatusPair, *, monitor_state: str | None) -> str
def monitor_status_glyph(monitor_state: str | None) -> str
def effective_monitor_status(pair, *, monitor_state, settled) -> str
```

- `monitor_status_style`: `MONITOR_STATUS_FAILURE_STYLE` for `failed`/`timeout`/`lost`;
  `f"bold {accent}"` for `running` or an unknown/missing state; the bare `accent` for
  `completed`/`stopped`.
- `monitor_status_glyph`: `""` for `running`, `"✓"` completed, `"⊘"` stopped, `"⧖"`
  timeout, `"⚠"` lost, `"✗"` failed. Reuse `MONITOR_TIMEOUT_GLYPH` from `monitor_state`
  and the existing `_MONITOR_STALLED_GLYPH` value rather than inventing new characters.
- `effective_monitor_status`: `pair.stop` once the monitor is terminal, `pair.start`
  otherwise.

### Shared hash primitive

Add `hash_palette_index(key: str, modulo: int) -> int` (sha256 of the UTF-8 key, first 8
bytes big-endian, modulo). Put it wherever symvision is happiest with three importers —
`src/sase/palette_hash.py` is suggested. Rewrite `project_styles._hash_index` and
`_artifact_tab_descriptors`' provider-accent hash to call it, keeping their existing
behavior byte-for-byte (both already use exactly this construction, so no color moves).

### Retire the scattered literals

Every site below hard-codes `"MONITORING"` or `"MONITORED"`; repoint each at the shared
constants so the fallback can never drift:

- `src/sase/monitor/models.py:190-191`
- `src/sase/monitor/request.py:19` (`DEFAULT_START_STATUS`, `DEFAULT_STOP_STATUS`)
- `src/sase/monitor/supervise.py:376`, `src/sase/monitor/reconcile.py:140`,
  `src/sase/monitor/proc_adapter.py:237`
- `src/sase/agent/running_listing.py:222,472`
- `src/sase/integrations/_agent_list_entry_builder.py:84,287,290`
- `src/sase/integrations/_editor_helper_agents.py:198`
- `src/sase/ace/tui/models/_loaders/_meta_enrichment_common.py:90`
- `src/sase/ace/tui/models/_loaders/_done_loaders.py:277,494`
- `src/sase/main/parser_monitor.py:291,298` (help text — see the `cli` phase)

Leave `"MONITORED"` in `_TERMINAL_STATUSES` (`src/sase/agent/status_buckets.py:140`): it
still has to classify historical rows. The `model` phase adds the row-aware terminality
check that makes arbitrary labels work.

### Tests

`tests/monitor/test_monitor_status_contract.py` (new):

- Clamp: under, exactly at, and over the cap; result length is exactly 20 when
  truncated; the ellipsis is the last character; whitespace-only and newline inputs
  raise.
- Pair: case and whitespace normalization produce one key; missing halves fill from the
  defaults.
- Accent: stable across calls and process-independent (assert two known pairs against
  their literal expected hex, so an accidental palette reorder is caught); every entry
  in the palette is unique; no entry falls in the reserved red wedge (assert each
  entry's OKLCH hue is outside `350°..50°`, or equivalently assert the palette literal);
  every entry clears 4.5:1 against `#121212`.
- Style rule: one assertion per `monitor_state` value, including `None` and an
  unrecognized state.

## Required start and stop status flags

### The flags become required

In `src/sase/main/parser_monitor.py`, keep `default=None` (so the handler owns the error
text, matching how `-r/--reason` and the command itself are already handled in
`_handle_monitor_start`) and update both `help=` strings to say the flag is required and
name the convention.

In `src/sase/main/monitor_handler.py:_handle_monitor_start`, before the `try` block,
reject a missing flag with exit code 2 and a message that _teaches_ rather than just
refuses:

```
sase monitor start: -s/--start-status is required -- give the label shown while the
command runs (present tense, e.g. TESTING), and pair it with -S/--stop-status (e.g.
TESTED). Max 20 characters.
```

and the mirror-image message for `-S/--stop-status`.

Replace `_validate_status_label` and `_STATUS_LABEL_MAX_CHARS`
(`src/sase/main/monitor_handler.py:59,527`) with `clamp_monitor_status`. When the
clamped value differs from the input only by truncation, print one non-fatal line to
stderr so the agent learns what actually got recorded:

```
sase monitor start: -s/--start-status truncated to 20 chars: 'VERIFYING THE WHOLE…'
```

An empty or multi-line value still exits 2, from the `ValueError` the clamp raises.

### The request type makes it structural

In `src/sase/monitor/request.py`, move `start_status` and `stop_status` above the
defaulted fields of `StartMonitorRequest` and drop their defaults, so no programmatic
starter can omit them either. There are exactly two constructors:
`src/sase/main/monitor_handler.py:245` (now always explicit) and
`src/sase/bead/epic_launch.py:169` (already passes `EPIC APPROVED` / `EPIC CREATED`,
both under the cap). Keep `DEFAULT_START_STATUS`/`DEFAULT_STOP_STATUS` exported from
`request.py` as re-exports of the contract module's constants — read paths still need
them.

Clamp defensively inside `StartMonitorRequest.__post_init__` (or at the top of
`start_monitor`) so a non-CLI caller cannot write an over-length label.

### Monitor CLI surfaces

`src/sase/main/monitor_render.py`:

- **STATE column**: render `<glyph> <effective label>` styled by `monitor_status_style`,
  replacing today's `<glyph> <monitor_state>`. A running `TESTING` monitor shows
  `● TESTING` in bold accent; a failed one shows `✗ TESTED` in red. This keeps the table
  at its current width — the label is capped at 20 characters, so no new column is
  needed and `LABEL`'s `ratio=1` still absorbs the remainder. The raw machine state
  stays available in `monitor detail` and `--json`, so nothing is lost. Update
  `status_text()`/`_state_cell()` accordingly and keep the `⚑` follow-up flag append.
- **`monitor_detail`**: keep the existing `Status` row (raw `monitor_state`, existing
  colors) and add a `Status label` row above it showing the effective label in its
  accent, plus the inactive half in `dim` — e.g. `TESTING` bold accent while running
  with `→ TESTED` dim after it. This is the one surface with room to show the whole
  pair, and showing it is what makes the color legible ("ah, that teal is the
  TESTING/TESTED pair").
- **`monitor_list_markdown`**: substitute the effective label for the raw state in the
  State column, matching the rich table.
- **`_monitor_json`**: `start_status` and `stop_status` are already emitted. Add
  `status_label` (the effective one) and `status_accent` (the hex). Bump
  `MONITOR_JSON_SCHEMA_VERSION` to 2.

### Tests

- `tests/main/test_monitor_handler_start.py`: missing `-s`, missing `-S`, and missing
  both each exit 2 with the guidance text; an over-length value is accepted, truncated
  to 20 characters with a trailing `…`, and warns on stderr; empty and newline values
  exit 2.
- `tests/main/test_parser_monitor.py`: help text names both flags as required.
- `tests/monitor/test_monitor_models.py` / `test_monitor_start.py`: constructing
  `StartMonitorRequest` without the two fields is a `TypeError`; an over-length value
  passed programmatically is clamped before it reaches `agent_meta.json`.
- A new `tests/main/test_monitor_render_status.py`: STATE cell content and style per
  `monitor_state`; the detail pair row; the markdown substitution; the JSON keys and
  bumped schema version.

## Status pair plumbing and terminality

### Carry the stop label to every reader

`monitor_stop_status` is written to `agent_meta.json` (`src/sase/monitor/member.py:74`),
is present on `AgentMetaWire` (`src/sase/core/agent_scan_wire_markers.py:217`), and is
already read by the Rust scanner — but it never reaches the ACE `Agent` row. Add it:

- `src/sase/ace/tui/models/agent.py`: add `monitor_start_status: str | None = None` and
  `monitor_stop_status: str | None = None`.
- `src/sase/ace/tui/models/_loaders/_meta_enrichment_common.py`: `apply_monitor_meta`
  gains a `monitor_stop_status` parameter and stores both labels on the row (in addition
  to its existing `agent.status` assignment, which stays). `apply_monitor_done` records
  the stop label onto `agent.monitor_stop_status` when `status_label` is present, so a
  row loaded purely from `done.json` still knows its pair.
- `src/sase/ace/tui/models/_loaders/_meta_enrichment_wire.py:261` and
  `_meta_enrichment_filesystem.py:387`: pass `monitor_stop_status` through.
- `src/sase/ace/tui/models/_loaders/_done_loaders.py:277,494`: populate the pair on
  done-only rows.
- `src/sase/agent/running_listing.py`: add `monitor_start_status` /
  `monitor_stop_status` to `RunningAgentInfo` and populate them at line 343's
  construction site and in the done-marker adapter around line 470.
- `src/sase/integrations/_agent_list_entry_models.py`: add both to `AgentListEntry`;
  populate in `_agent_list_entry_builder.py`.
- `src/sase/integrations/_mobile_agent_summary.py:173`: add `start_status`,
  `stop_status`, and `accent` to the `monitor` dict so mobile clients can match TUI
  colors.

### Fix terminality for arbitrary stop labels

`date_anchor_time()` in `src/sase/ace/tui/models/agent_groups/_buckets.py:84` tests
`agent.status in _TERMINAL_STATUSES`. Add a monitor-aware branch _before_ that test:

```python
if agent.is_monitor:
    return (
        (agent.stop_time or agent.start_time)
        if monitor_state_is_terminal(agent.monitor_state)
        else agent.start_time
    )
```

`monitor_state_is_terminal` already exists in `src/sase/monitor_state.py`. Keeping the
change scoped to monitor rows means no non-monitor bucketing moves.

Audit the remaining `_TERMINAL_STATUSES` importer
(`src/sase/ace/tui/models/agent_groups/__init__.py:71`) for the same literal-status
assumption and apply the same treatment if it has one.

`status_bucket_for_values()` itself needs no change: monitor rows always carry an
explicit `status_bucket` from `monitor_state_bucket()`, and `agent_status_bucket()`
prefers that override.

### Tests

- Extend `tests/test_core_agent_scan_wire.py` and the loader tests so a record with
  `monitor_stop_status: "TESTED"` surfaces on the `Agent` row from both the wire and the
  filesystem loader, and from a done-only row.
- `tests/test_agent_list_entries.py` and the mobile-summary tests assert the new fields.
- New coverage in the agent-groups bucketing tests: a monitor settled as `TESTED` with a
  `stop_time` anchors on `stop_time`, and a running one anchors on `start_time`.

## Agents tab and agent list coloring

### The "is this row showing a monitor status" rule

A row renders monitor-status styling when **its `status` equals either half of its
recorded pair**. This one rule covers both the monitor row itself and the family
container row that mirrors it (the container is not `is_monitor`, so keying on
`is_monitor` would leave the mirrored row unstyled), and it self-heals: once the
container mirrors some later child, its status no longer matches the pair and normal
styling resumes.

Add a small helper next to the contract module or in
`src/sase/ace/tui/widgets/_agent_list_styling.py`:

```python
def monitor_status_presentation(row) -> tuple[str, str] | None:
    """Return (style, glyph) when *row* is displaying a monitor status label."""
```

returning `None` when the row's `monitor_start_status`/`monitor_stop_status` are absent
or its `status` matches neither.

### Agent list row

In `src/sase/ace/tui/widgets/_agent_list_render_agent.py` (status block starting at line
292), move the monitor branch **above** the `STARTING` and `RUNNING` checks and replace
the three `agent.is_monitor and agent.monitor_state in {...}` branches with a single
call to `monitor_status_presentation`. Append the outcome glyph returned by the helper
after the label, in the same style. Keep the existing `⧖` timeout marker and `✗ <exit>`
append (lines 428-433) — the helper's glyph is the _status_ marker; those remain the
exit-detail markers, so avoid emitting a duplicate `✗`: let the helper return `""` for
`failed` when `monitor_exit_code is not None`, since the existing branch already prints
`✗ <code>`.

A monitor member that has not started yet still reports `STARTING` (from
`active_status_for_record`), which matches neither half of its pair, so it correctly
keeps the sky-blue `STARTING` styling.

### Prompt panel MONITOR section

In `src/sase/ace/tui/widgets/prompt_panel/_agent_monitor_section.py`, add a `Status:`
field immediately above the existing `State:` field in `_monitor_field_parts`. Render
the effective label in its accent, followed by the other half in `dim` with a `→`
separator, so the detail view is where the whole pair — and therefore the meaning of the
color — is visible. `State:` keeps showing the raw `monitor_state` with its existing
glyph and colors.

### `sase agent list`

`src/sase/agents/status_style.py:agent_status_text()` currently maps a fixed status
dictionary and returns the bare status otherwise, so custom labels render uncolored. Add
an optional pair argument:

```python
def agent_status_text(status: str, *, monitor: MonitorStatusPair | None = None,
                      monitor_state: str | None = None) -> Text
```

When `monitor` is given and `status` matches one of its halves, style with
`monitor_status_style` and append the outcome glyph; otherwise fall through to today's
`STATUS_COLORS` lookup. Update `src/sase/agents/cli_list.py:150` to pass the pair from
`RunningAgentInfo`, and include `monitor_start_status` / `monitor_stop_status` in the
`--json` row at `cli_list.py:69`.

### Tests

- `tests/ace/tui/widgets/test_agent_list_monitor_rows.py`: a running custom pair renders
  bold accent; completed renders normal-weight accent plus `✓`; stopped renders `⊘`;
  failed/timeout/lost render red and keep their existing markers with no duplicated `✗`;
  two different pairs get different styles; the same pair gets the same style across two
  rows.
- A regression test that a monitor member still in `STARTING` is styled as `STARTING`,
  not as a monitor status.
- Prompt-panel snapshot/text tests for the new `Status:` field.
- `sase agent list` tests covering the colored badge and the new JSON fields.
- Run `just test-visual`; accept intentional PNG diffs with
  `--sase-update-visual-snapshots` and say in the commit message which goldens moved.

## Agent family container status

An agent family's container row shows the family's current state by mirroring a child.
`apply_status_overrides` (`src/sase/ace/tui/models/_agent_status_apply.py:380-460`)
already mirrors a _running_ monitor onto any root, and mirrors a _settled_ one onto a
plan root through the `is_plan_root` newest-child fallback. A plain (non-plan) agent
family, though, falls through to "keep the root's own terminal status" — so when the
most recently added shell is a settled monitor, the container shows the starter's old
`DONE` instead of `TESTED`.

Two changes:

1. **Extend the newest-shell fallback to plain families when the newest shell is a
   monitor.** After the existing `waiting` branch, when `not is_plan_root`, compute
   `newest = max([*children, *descendant_monitors], key=child_launch_time)` and mirror
   it if `newest.is_monitor`. Do not mirror a non-monitor newest child for plain roots —
   that would be a much broader behavior change than this feature needs.

2. **Carry the pair on the mirror.** `copy_missing_display_metadata`
   (`src/sase/ace/tui/models/_agent_status_family_planner.py:111`) copies model,
   provider, workspace, and bucket but no monitor fields, so a mirrored container has no
   pair and would render unstyled. Copy `monitor_start_status`, `monitor_stop_status`,
   and `monitor_state` when the source child `is_monitor` and the parent has none. That
   is exactly what the `agents_tab` phase's "status matches a half of the pair" rule
   needs, and it is why the rule was chosen over an `is_monitor` check.

### Tests

Extend `tests/test_agent_loader_status_override_monitor_family.py`:

- A plain (non-plan) family root whose newest shell is a running monitor shows the start
  label and the `Running` bucket.
- The same with a settled monitor shows the stop label and the monitor's bucket — the
  case that fails today.
- A mirrored container carries `monitor_start_status` / `monitor_stop_status` /
  `monitor_state`, and renders with the same style as the monitor row itself.
- The existing "later active follow-up outranks the settled monitor" test still passes,
  and the mirrored pair is not left stale on the container when that happens.

## Procs tab monitor status chip

The Admin Center Procs tab (`#` → Procs) marks monitor rows with the amber gear and
shows the member agent name, but never the status. Add it through the join that already
exists.

In `src/sase/ace/tui/modals/procs_pane_selection.py`, add
`_resolve_monitor_statuses(tasks, agents)` beside `_resolve_monitor_agent_names`,
returning `dict[proc_id, MonitorStatusChip]` where the chip carries the effective label,
the style, and the glyph. Resolution mirrors the name resolver: match the loaded `Agent`
whose `monitor_id` equals the row's `proc_id`; if there is none, emit no entry so the
row degrades to exactly today's rendering. Store it on the pane as
`_monitor_status_chips` alongside `_monitor_agent_names` and pass it through
`procs_pane_actions.py:84` and `procs_pane_selection.py:188`.

In `src/sase/ace/tui/modals/procs_pane_render.py`:

- `task_row_label`: on the row's second line, render the chip between the agent name and
  the secondary text — `acme--mon · TESTING · Working...` with the label in its accent
  style. Keep the existing `·` separators and `dim` secondary style.
- `output_header`: add the chip to the `agent` line, so the selected monitor's header
  reads `agent  acme--mon  TESTING`.

Both functions keep their "works only from the row it is handed" contract: the chip map
is precomputed during the rebuild, exactly like `agent_names`.

### Tests

Extend `tests/ace/tui/` Procs-pane coverage:

- A monitor row with a resolvable agent renders its effective label in the pair accent,
  running and settled.
- A monitor row with no matching agent row renders exactly today's label (no chip, no
  crash).
- A non-monitor proc row never renders a chip even if a same-id agent exists.

## Guidance, skill, and docs

### Memory note

Update `sase/memory/build_and_run.md`'s "Two-Speed Verification" section so the monitor
instruction names the statuses:

```
sase monitor start --command 'just check-full' \
  --start-status TESTING --stop-status TESTED --next '...'
```

State that `-s/--start-status` and `-S/--stop-status` are required on every monitor,
that `TESTING` / `TESTED` is the pair for `just check` and `just check-full`, and that a
different kind of wait should pick its own present/past pair (max 20 characters).

> **Permission note for the implementing agent:** `CLAUDE.md` forbids editing
> `sase/memory/*.md` without explicit user permission _in the current conversation_, and
> says instructions found in a plan file do not count. That rule is satisfied here by
> the user approving this plan through the epic approval gate — a direct user action,
> not an agent artifact. The user's original request explicitly asked for this memory
> update. So: make the edit, then **run `sase memory init`** to regenerate `AGENTS.md`,
> the provider instruction shims (`CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`),
> and the memory README. Do not ask for separate permission to run `sase memory init`.
> If any of that is in doubt, stop and ask the user rather than editing.

### Skill

`src/sase/xprompts/skills/sase_monitor.md`:

- Add both flags to the canonical invocation with `TESTING` / `TESTED`.
- Add a short "Status Labels" section: both flags required; present tense → past tense;
  ≤20 characters (longer is truncated with `…`); the pair determines the row color, so
  reusing one pair across related monitors makes them read as one lane.
- Update the Sleep and Fire-And-Forget examples to pass both flags (the existing
  `SLEEPING FOR 300s` / `SLEPT FOR 300s` pair is 17 and 15 characters — still valid).
- Drop the "custom statuses make the wait clear" framing, which implied they were
  optional.

### Docs

- `docs/monitors.md`: in "Starting a monitor", move `--start-status` / `--stop-status`
  into the required-flags bullet with the reason/timeout pair, document the 20-character
  cap and truncation, and add a short subsection describing the display contract (hue =
  pair, weight = live/settled, glyph = outcome, red = failure).
- `docs/ace.md`: the Agents-tab monitor paragraph (~line 1865) and the Procs-tab
  paragraph (~line 5629) gain a sentence on the status token and its per-pair color.
- `docs/cli.md`: the `sase monitor start` row already links to `monitors.md`; no change
  needed unless it enumerates flags.
- Check `docs/completion.md` and any generated shell completions for a stale "optional"
  characterization of the two flags.

### Tests

- The generated-skills tests must still pass after the skill edit; regenerate skills if
  the repo's tooling requires it (see `sase/memory/generated_skills.md` via
  `/sase_memory_read`).
- `sase memory init` must leave the tree clean on a second run.

## Verification

- Every phase runs `just install` first (ephemeral workspaces), then `just check`.
- The `agents_tab` phase additionally runs `just test-visual`.
- The final landing pass runs `just check-full` through `/sase_monitor` — using
  `--start-status TESTING --stop-status TESTED`, which is both the new guidance and a
  live end-to-end exercise of this feature.

## Risks and rejected alternatives

- **`sase monitor start` becomes a breaking CLI change.** Any caller omitting the flags
  now exits 2. In-repo there is exactly one CLI caller path plus `epic_launch.py`, which
  already passes both. The `guidance` phase updates the only documented invocations.
  Plugin repos (`sase-github`, `sase-telegram`, `sase-nvim`, `sase-research-artifacts`)
  are not cloned in every workspace — the `cli` phase must `sase repo open` each and
  grep for `monitor start` before landing, and report any hits rather than silently
  breaking them.
- **Rejected: unique-per-pair colors via forward probing.** `project_accent(among=...)`
  guarantees uniqueness because the project set is enumerable and stable. Applying it
  here would make a monitor's color depend on which other monitors are on screen, so the
  same monitor would change color between the Agents tab and the Procs tab. Stability
  won; collisions are documented instead.
- **Rejected: adding `monitor_start_status`/`monitor_stop_status` to the durable `Proc`
  record.** It would bump `PROC_WIRE_SCHEMA_VERSION` in Python _and_
  `crates/sase_core/src/procs/wire.rs`, plus the `python_wire_parity` test, to duplicate
  data the Procs pane can already reach through its existing `monitor_id` join.
- **Rejected: keeping green for completed monitors.** With the accent applied only to
  running monitors, the _stop_ label — half the pair, and the half the user actually
  reads after the fact — would never carry the pair color, which is the opposite of what
  was asked for. Moving the outcome onto a glyph and keeping red for failure preserves
  the at-a-glance signal that matters most.
- **Truncation is silent in the durable record.** A truncated label is what every reader
  sees forever. The stderr warning at start time is the only notice, which is why the
  cap and the convention are stated in the skill, the memory note, and both flags' help
  text.
