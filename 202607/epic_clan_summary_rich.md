---
tier: epic
title: Rich, reliable epic clan summaries
goal: 'Epic clan panels always show a complete, launch-fresh summary — never the bare
  "EPIC <id>" fallback — and that summary is an information-dense, beautifully colored
  overview (markdown-aware highlighting, phase status glyphs, size chips, progress,
  and the plan reference) that finally meets the sase-7r "rendered beautifully" bar.

  '
phases:
- id: fresh
  title: Launch-fresh epic bead reads with loud fallback diagnostics
  depends_on: []
  size: medium
  description: '''Phase: Launch-fresh epic bead reads with loud fallback diagnostics''
    section: retry the epic lookup against a synchronously integrated bead store when
    the epic id is missing from a warm sidecar clone, report script failures to stderr
    so they land in the agent log, and widen the summary-script timeout budget to
    cover the blocking retry.'
- id: rich
  title: Information-dense Rich rendering for epic summaries
  depends_on:
  - fresh
  size: medium
  description: '''Phase: Information-dense Rich rendering for epic summaries'' section:
    rewrite the epic summary layout — full markdown-highlighted goal, per-phase status
    glyphs, size chips and one-line descriptions, launch-time progress, child epics,
    and the plan reference — and pin the new look with PNG goldens.'
- id: smoke
  title: End-to-end epic summary exercises
  depends_on:
  - rich
  size: small
  description: '''Phase: End-to-end epic summary exercises'' section: fakey-provider
    bead-work launches asserting the persisted clan summary is the full rich overview
    (never the bare fallback), including recovery from a stale sidecar clone.'
  model: haiku
create_time: 2026-07-20 10:48:51
status: done
bead_id: sase-85
---

# Plan: Rich, reliable epic clan summaries

## Context

sase-7r shipped clan summaries: a Rich-markup block resolved once at launch (literal or `summary_script=`), persisted in
`agent_meta.json` as `clan_summary`, threaded through the sase-core scan wire, and rendered in the clan detail panel by
`build_clan_detail_text` (`src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan.py`) via `Text.from_markup(...)`.
Epic clans declared by `sase bead work` get the built-in `sase_clan_summary_epic` script
(`src/sase/scripts/sase_clan_summary_epic.py`).

The user's screenshot of a live epic clan shows the panel summary as literally just **`EPIC sase-83`**. Two distinct
problems hide behind that:

### Problem 1 (bug): the persisted summary is the script's _fallback_, not its real output

Evidence chain, verified against the live run:

- The declaring member's `agent_meta.json` (under the project's `artifacts/ace-run/...` tree) contains
  `"clan_summary": "[bold]EPIC sase-83[/]"` — exactly `_fallback_summary()`.
- The plans-sidecar commit `chore: mark bead work launched for sase-83` (which is also `sdd_base_sha` in that meta file)
  landed the **same second** as `run_started_at`. The epic and phase beads were created moments before launch by the
  plan-approval flow.
- The summary script runs with `cwd=<the epic agent's claimed workspace>` (`resolve_clan_summary_script` in
  `src/sase/axe/clan_summary_script.py`). Its `get_read_view()` → `get_project()` (`src/sase/bead/cli_common.py`) treats
  a _warm, valid_ sidecar clone as authoritative: `resolve_beads_location(require_existing=True)` +
  `resolved_beads_location_is_usable(...)` only re-materializes when the clone is missing/mismatched or
  `bead_refresh_mode() == "blocking"` (default mode is `background`, TTL-gated and async —
  `_maybe_schedule_bead_refresh` in `src/sase/bead/sync.py`).
- The claimed workspace's plans sidecar clone predated the epic-creation push, so `project.show("sase-83")` raised, and
  the script's blanket `except Exception` printed the fallback — **silently**: no traceback reaches stderr, even though
  `resolve_clan_summary_script` already pipes script stderr into the agent log.
- Re-running the same load from that workspace _after_ its sidecar refreshed produces the full summary — confirming a
  pure freshness race, not a data problem.

This race is structural for epics: `sase bead work` is _always_ launched right after the beads it needs were created, so
any warm-but-stale sidecar clone in the claimed workspace reproduces it.

### Problem 2 (enhancement): even the non-fallback output is far below the "rendered beautifully" bar

`_render_epic_summary` deliberately emits only: one header line, the goal `textwrap.shorten`-ed to a single 76-col line,
and bare numbered phase titles. The user wants the epic summary to show substantially more information and to look like
a properly syntax-highlighted markdown document — great colors throughout.

## Design decisions

**Keep the `summary_script=` architecture; fix freshness inside the script.** Resolving the summary in the launcher
(which has a fresh store) and passing a literal was considered and rejected: clan forks re-run the declaring member's
directives, and a script re-resolves against _current_ bead state at each generation, which is the behavior we want. No
changes to `%clan` syntax, the runner, `agent_meta.json` shape, or the sase-core wire.

**Freshness algorithm (fast path stays fast).** The script first reads through the existing warm view. Only when the
epic id cannot be resolved does it force a synchronous integration of the sidecar store and retry once. Candidate hooks,
in preference order: `push_bead_work_launch(beads_dir)` (`src/sase/bead/sync.py`, the synchronous integrate-and-push
used by the async worker) or `run_managed_sync_worker` (`src/sase/bead/sync_worker.py`); a
`resolve_beads_location(materialize=True)` re-open is acceptable only if it demonstrably pulls an existing clone. When
`bead_refresh_mode()` is `"off"` the user has opted out of remote syncs — skip the blocking retry and fall back as
today.

**Loud fallback.** Any exception that leads to the fallback (both the initial miss and a failed retry) prints the
traceback plus a one-line explanation to **stderr** before printing the fallback to stdout and exiting 0. Stderr already
routes to the agent log, so silent degradation like the observed one becomes diagnosable after the fact. Launches must
still never fail or block on a broken summary.

**Timeout budget.** `CLAN_SUMMARY_TIMEOUT_SECONDS` (10s in `src/sase/axe/clan_summary_script.py`) now has to cover a
possible blocking git integration. Raise the constant to 20s. The summary resolves outside the name-allocation lock
(documented in `run_agent_directives.py`), so the worst case only delays the declaring member's own launch, and only
when a script actually hangs.

**Script-side richness; no wire/format changes.** The panel keeps rendering a plain Rich-markup string. The richer look
is produced entirely inside the epic script, which pre-renders at a fixed 76-column width (matching the current
`_SUMMARY_WIDTH` convention the panel already accommodates). A `clan_summary_format` field with panel-side
`rich.markdown.Markdown` re-flow (directive → runner → Rust wire schema bump → TUI) was considered and is explicitly
**out of scope** — it is the right follow-up if other clans later need markdown summaries, but the epic summary does not
require it and this keeps sase-core untouched.

**Markdown-aware highlighting technique (verified against the pinned Rich).** Bead goals and descriptions may contain
inline markdown (`` `code` ``, `**bold**`, lists). Render such text through `rich.markdown.Markdown` into a
`Console(width=76, force_terminal=True, color_system="truecolor")` capture, convert each captured line with
`Text.from_ansi(line)`, `rstrip()` it (the capture pads lines to full width, including full-width code-block
backgrounds), and emit `.markup`. `Text.from_ansi(...).markup` round-trips styles and escapes `[` correctly. Pin
`code_theme` explicitly for deterministic output. Values interpolated directly into hand-written markup (ids, titles,
paths) keep using `rich.markup.escape`.

**Launch-stable statuses are deliberate.** Phase status glyphs reflect status _at launch_ (matching sase-7r's
launch-time-stable contract); live liveness remains the job of the panel's Status field and CLAN MEMBERS roster. When an
epic resumes with earlier phases already closed, the glyphs make that immediately visible — the main informational win.

**Out of scope.** No `sase/memory/*.md`, `AGENTS.md`, or provider-shim edits. No new keybindings, footer entries, or
help-modal changes. No sase-core changes. Member-row panels unchanged.

## Target layout

Rendered by the script at ≤76 columns, well under the 32 KiB cap (the panel shows it between the `Members:` line and the
fold header, at every fold level):

```text
◆ EPIC sase-83 · Provider-aware comprehensive update experience

ACE's cached startup and periodic update checks include supported agent
CLI providers without adding UI-thread or per-tick network cost, and the
global ,U action safely updates exactly the provider CLIs identified by
the latest completed background check…                (full goal, wrapped,
                                            markdown-highlighted, ~6-line cap)

PHASES · 1/3 done at launch
 ✓ 1. Provider-aware background update snapshot                   [medium]
      One-line dimmed phase description, shortened to fit.
 ◐ 2. Snapshot-gated comprehensive update flow                    [medium]
      One-line dimmed phase description, shortened to fit.
 ○ 3. Distinct update surfaces and release validation              [small]
      One-line dimmed phase description, shortened to fit.

CHILD EPICS · 1                                        (only when present)
 ◐ sase-83.4 · Some child epic title

Plan: sase/repos/plans/202607/agent_cli_update_awareness.md
```

Styling intent: bold-magenta header with a `◆` marker (consistent with the panel's clan identity color idiom in
`_agent_list_styling.py`); status glyphs `✓`/`◐`/`○` in green/yellow/dim reusing the color idiom `sase bead show`
already established (`src/sase/agent/bead_display.py` / bead CLI rendering); size chips right-aligned and colored by
size; goal paragraph and phase descriptions rendered through the markdown pipeline so inline code spans, bold, and lists
get real highlighting; the plan-path line dim. Exact glyph and hex choices are the implementer's, but they must match
the TUI's existing palette and be pinned by the PNG goldens.

## Phase: Launch-fresh epic bead reads with loud fallback diagnostics

All in the sase repo:

1. `src/sase/scripts/sase_clan_summary_epic.py`: restructure `main()`/`_load_epic` per the freshness algorithm — warm
   read first; on lookup failure with `bead_refresh_mode() != "off"`, run the synchronous sidecar integration (see
   design decisions for candidate hooks; pick the one that cleanly integrates an existing clone without pushing
   unrelated state, and factor a small helper in the bead layer if the existing entry points don't fit a read-only
   caller) and retry the load once. On final failure print the traceback and a one-line cause to stderr, then the
   existing fallback to stdout, exit 0.
2. `src/sase/axe/clan_summary_script.py`: raise `CLAN_SUMMARY_TIMEOUT_SECONDS` to 20.0 and update its tests/docs.
3. Tests (`tests/test_bead/test_clan_summary_epic_script.py` plus the clan-summary runner suites as needed):
   - Regression: epic id absent from the warm store, appears after the integration hook runs (monkeypatch the hook to
     materialize the bead) → full summary printed, no fallback.
   - `bead_refresh_mode() == "off"` → no integration attempted, fallback printed.
   - Failure path → traceback text observed on stderr, fallback on stdout, exit code 0.
   - Existing happy-path and fallback tests keep passing.

## Phase: Information-dense Rich rendering for epic summaries

All in the sase repo:

1. Markdown-to-markup helper (new small module near the script, e.g. `src/sase/scripts/_markdown_markup.py`, or a
   better-fitting presentation-layer home): implements the verified Markdown → ANSI capture → `Text.from_ansi` →
   per-line `rstrip()` → `.markup` pipeline at a caller-supplied width, with a pinned `code_theme` and a plain-text
   passthrough when the input has no markdown constructs worth rendering. Unit tests: inline code/bold/list inputs,
   trailing-padding trimmed, `[` characters survive as literals, output parses under `Text.from_markup`.
2. `src/sase/scripts/sase_clan_summary_epic.py::_render_epic_summary`: rebuild to the target layout — header, full
   wrapped goal (~6-line cap) through the markdown helper, `PHASES · <done>/<total> done at launch` with per-phase
   status glyph + number + title + right-aligned size chip + one-line dimmed description, `CHILD EPICS` block for
   `get_epic_children` entries that are epic-tier plans (current code filters to `IssueType.PHASE` only), and the dim
   `Plan:` reference from the epic bead's plan field when set. Keep every line ≤76 columns and the whole output
   comfortably under the 32 KiB cap for large epics (shorten per-entry, never truncate mid-markup).
3. Visual pinning: update the epic-style fixture summary in
   `tests/ace/tui/visual/_ace_agents_png_snapshot_clan_fixtures.py` (and the cases in
   `tests/ace/tui/visual/test_ace_png_snapshots_agents_clan_panel.py`) to the new output style, regenerate affected
   goldens with `--sase-update-visual-snapshots`, and inspect `.pytest_cache/sase-visual/` artifacts for spacing,
   wrapping, and color quality before accepting.
4. Unit tests: renderer content assertions (goal wrapped not shortened-to-one-line, glyph per status, size chips,
   child-epic block presence/absence, plan line, markup parses cleanly), plus a many-phase epic staying within
   width/size bounds.

## Phase: End-to-end epic summary exercises

Extend the existing end-to-end suites (`tests/test_bead/test_cli_work_epic_launch.py`,
`tests/test_axe_smoke_clan_summary.py`) with fakey-provider `sase bead work` launches asserting:

- The declaring member's `agent_meta.json` `clan_summary` contains the epic title and **every** phase title — i.e. the
  real rendering, never `[bold]EPIC <id>[/]`.
- A launch whose workspace-visible bead store is initially missing the epic (stale-clone simulation at whatever layer
  the fixtures allow; if true end-to-end staleness is impractical, assert the retry hook is exercised) still persists
  the full summary.
- The clan panel renders the persisted block (reuse the existing panel-rendering assertion pattern from the sase-7r
  smoke tests).

## Testing & verification

- Every phase runs `just install` then `just check` before finishing; the rendering phase also runs `just test-visual`
  and inspects `.pytest_cache/sase-visual/` artifacts before accepting golden updates.
- Manual spot-check after the rendering phase: launch a real epic via `sase bead work` and confirm the clan panel shows
  the full new summary.

## Risks

- **ANSI→markup drift across Rich/pygments versions**: the conversion output depends on the pinned Rich and the chosen
  `code_theme`. Pin the theme explicitly and cover the pipeline with unit tests so an upgrade surfaces as a test diff,
  not a silent visual regression. PNG goldens already pin fonts/colors per the visual-test fixtures.
- **Blocking integration latency**: a slow or wedged git remote now costs up to the 20s script timeout on the epic
  declaring member's launch. Acceptable: it only triggers on a store miss, and the timeout still guarantees launch.
- **Concurrent integration contention**: the synchronous integration may race the TTL-gated background worker; the
  chosen hook must use the same locking the worker uses (`run_managed_sync_worker` already does).
- **Golden churn**: the clan-panel goldens change deliberately; keep the update to the fixtures introduced here and
  inspect diffs so unrelated snapshots stay untouched.
