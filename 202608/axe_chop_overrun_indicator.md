---
tier: epic
title: Mark chops that outrun their lumberjack's interval in the AXE tab
goal: 'The AXE tab visibly marks every chop whose run time reaches or exceeds its
  lumberjack''s interval — on the sidebar row, in the lumberjack overview table, and
  in the chop detail header — so the operator can tell at a glance which chop is stretching
  its lumberjack''s cycle and needs a longer interval or a lumberjack of its own.

  '
phases:
- id: core_classifier
  title: Overrun classifier in the Rust core
  depends_on: []
  size: medium
  description: 'core_classifier: add the host-I/O-free `axe_overrun` module, its versioned
    wire records, and the `classify_chop_overrun` PyO3 binding that turns one chop''s
    cached run history plus its lumberjack interval into a level/ratio verdict.

    '
- id: blocking_duration
  title: Record how long a chop actually blocked its tick
  depends_on: []
  size: small
  description: 'blocking_duration: persist `script_duration_ms` on chop run entries
    so an agent-launching run keeps its script wall-clock after lifecycle finalization
    overwrites `duration_ms`, and make the run-entry reader tolerate unknown keys.

    '
- id: snapshot_wiring
  title: Classify each chop while collecting AXE snapshots
  depends_on:
  - core_classifier
  - blocking_duration
  size: medium
  description: 'snapshot_wiring: add the typed Python facade, carry the effective
    interval and the verdict on `ChopSnapshot`/`LumberjackSnapshot`, and compute both
    on the full collector and the targeted single-chop refresh path.

    '
- id: tab_indicator
  title: Render the overrun mark across the AXE tab
  depends_on:
  - snapshot_wiring
  size: medium
  description: 'tab_indicator: paint the sidebar chop chip and lumberjack roll-up,
    the overview table''s PACE column and advisory line, the chop detail header segment,
    the help guide legend, docs, and the PNG snapshots that pin all of it.'
proposed_by: bbugyi200.athena.ye
create_time: 2026-08-12 09:05:51
status: wip
bead_id: sase-jx
---

- **BEAD:** [sase-jx](https://github.com/sase-org/sase--beads/blob/main/pages/sase-jx/README.md)

# Plan: Mark chops that outrun their lumberjack's interval in the AXE tab

## Summary

A lumberjack ticks on a fixed `interval`. Every chop it owns runs inside that tick, so a
chop whose run time reaches the interval stretches the whole cycle: the next tick starts
late, and every other chop under that lumberjack is delayed with it. The fix is always
one of two things — raise the lumberjack's `interval`, or move the chop into a
lumberjack of its own — but the AXE tab gives no signal that the situation exists. The
interval lives in the lumberjack overview header; the durations live in a different
column, a different view, or the sidebar's parent row is collapsed entirely. The
operator has to hold one number in their head and compare it against N others.

This epic computes the comparison once, at collection time, in the Rust core, and paints
it in the three places a chop is visible in the AXE tab. Nothing about scheduling
behavior changes; this is a diagnostic the tab already had all the raw inputs for.

## Evidence

**A single slow chop delays the whole lumberjack.** `sase: src/sase/axe/lumberjack.py`
runs a tick's eligible chops concurrently in a `ThreadPoolExecutor` and blocks on
`as_completed` until the slowest one returns (`lumberjack.py:217-225`). Tick duration is
therefore ≈ the slowest chop, and the loop already knows when that is too long — it logs
`Tick overrun: took {x}s but interval is {y}s` (`lumberjack.py:245-250`). That log line
is the existing, correct vocabulary for this condition; the tab just never surfaces it,
and the log does not name which chop caused it. `schedule.run_pending()` runs a late
tick late; it does not overlap or replay, so the practical effect is a cycle that
silently drifts to the slow chop's period.

**`run_every` does not exempt a chop.** A chop with `run_every` set is skipped on ticks
where its cadence has not elapsed (`lumberjack.py:204-215`), but on the ticks where it
_does_ run it blocks like any other. The interval remains the right budget to compare
against for every enabled chop.

**`duration_ms` is not always blocking time.** This is the single most important
correctness detail in this epic. For a script chop, `finalize_script_chop_run` stamps
`duration_ms` with the script's wall clock
(`sase: src/sase/axe/chop_runner_script_lifecycle.py:63-64`). A chop that launched
agents stays in the active `launched` state with `finished_at=None`
(`chop_runner_script_lifecycle.py:71`) and, when the launched agents finally complete,
`finalize_launched_chop_runs` **overwrites** `duration_ms` with the span from the run's
start to the agents' completion (`sase: src/sase/axe/chop_lifecycle.py:437-449`, using
`_duration_ms` at `chop_lifecycle.py:157-166`) and moves the status to
`action_succeeded` / `action_failed`. That number can be hours. The lumberjack tick
never waited for it. Classifying it as an overrun would put a permanent red mark on
every proposal chop in the tree and destroy trust in the indicator on day one.

**The tab already caches everything else it needs.** `collect_axe_status_data` loads the
axe config and, per lumberjack, its `LumberjackStatus`, metrics, log tail, and a
`ChopSnapshot` per configured chop with up to `MAX_CHOP_RUN_HISTORY` (10) newest-first
runs (`sase: src/sase/ace/tui/actions/axe_display/_data.py:268-325`). The interval is
available from two places in that same function: `LumberjackConfig.interval`
(`sase: src/sase/axe/_config_types.py:86-98`) and the running process's
`LumberjackStatus.interval` (`sase: src/sase/axe/_state_lumberjack.py:26-38`). The
collector runs off the event loop, which is where any classification must happen: TUI
render paths do no work per keypress (`sase/memory/tui_perf.md` rules 1 and 8).

**Auto-refresh cadence.** `refresh_interval` defaults to 10 seconds
(`sase: src/sase/ace/tui/app.py:227`), which bounds how stale a collection-time verdict
can be.

**Rendering surfaces that show a chop today.**

| Surface                     | File                                                              | Today                                                     |
| --------------------------- | ----------------------------------------------------------------- | --------------------------------------------------------- |
| Sidebar chop row            | `sase: src/sase/ace/tui/widgets/bgcmd_list.py:275-319`            | `└─ [✓] name` + optional `disabled` / `instance` chip     |
| Sidebar lumberjack row      | `sase: src/sase/ace/tui/widgets/bgcmd_list.py:227-273`            | `▌ [*] name` + `Ne` / `Nc` chip (`bgcmd_list.py:423-433`) |
| Overview chop table (wide)  | `sase: src/sase/ace/tui/widgets/_axe_dashboard_render.py:191-242` | NAME / LAST RUN / WHEN / DURATION in a 68-cell rule       |
| Overview chop list (narrow) | `sase: src/sase/ace/tui/widgets/_axe_dashboard_render.py:245-280` | stacked `status · when · duration`                        |
| Chop detail header          | `sase: src/sase/ace/tui/widgets/_axe_dashboard_status.py:294-367` | status, When, Took/Elapsed, Exit, PID, Run N/M            |

The sidebar already widens itself to the widest formatted row via `WidthChanged`
(`bgcmd_list.py:213-216`), and the `disabled` / `instance` chips already establish
"append a short chip after the chop name" as the idiom, so the sidebar mark introduces
no new layout concept.

## Design

### Vocabulary

**Overrun**: a chop run whose blocking time reached or exceeded its lumberjack's
interval. The word is already the lumberjack's own (`Tick overrun`,
`lumberjack.py:247`); reuse it in code, UI, and docs rather than inventing a second
term.

### The rule

For one chop, with `interval_ms = interval_seconds * 1000`:

1. **Blocking time per run.** For each cached run entry, derive `blocking_ms`:
   - `script_duration_ms` when present (phase `blocking_duration` persists it) — the
     authoritative "how long the tick was held";
   - else, when `status == "running"`, `now - started_at` — the tick is being held right
     now;
   - else, when `status in {"action_succeeded", "action_failed"}`, **unknown** — a
     legacy entry whose `duration_ms` was overwritten with the agent lifetime;
   - else `duration_ms`.
2. **Sampling.** Sample only runs whose script actually did the work: `success`,
   `failure`, `timeout`, `no_op`, `running`, `launched`, and `action_*` runs that carry
   a `script_duration_ms`. Exclude `skipped`, `missing_script`, and `check_error` —
   those are "did not do the work" outcomes whose near-zero durations would otherwise
   look like evidence of health. Runs with an unparsable `started_at` are skipped, never
   fatal.
3. **Over.** A sampled run is over when `blocking_ms >= interval_ms`. Equality counts: a
   run that exactly fills the interval leaves zero room for the rest of the cycle.
4. **Level.**
   - `over` — the **newest sampled run** is over. The problem is current.
   - `intermittent` — the newest sampled run is not over, but some older sampled run in
     the cached window is. The problem is real and recent but did not happen last time.
   - `none` — no sampled run in the window is over, or nothing is sampleable.

Level 2 exists for one reason: **flap resistance**. A chop that overruns every other run
would blink its mark on and off across refreshes under a naive latest-run-only rule,
which is the worst possible behavior for a diagnostic. Aging out of the 10-run window is
the only way a mark clears, so the mark is stable and its disappearance means something.

### The effective interval

Prefer the running process's `LumberjackStatus.interval` when it is a positive int, and
fall back to `LumberjackConfig.interval` otherwise; record which was used
(`interval_source: "runtime" | "config"`). A config edit that has not been picked up by
a restart must not change the marks, because the old interval is the one actually being
violated. When neither is a positive int, the chop is simply not classified.

### Visual language

Amber (`#FFAF5F`), already the palette's warning hue (dry-run and skipped markers), not
red: an overrunning chop is frequently a `✓ success`, and red is spoken for by failure.
Severity is carried by weight — `bold #FFAF5F` for `over`, `dim #FFAF5F` for
`intermittent` — with one glyph, `⚠`, in both. `⚠` and `×` measure one cell in Rich's
width table (verified), matching the ambiguous-width glyphs the tab already ships (`⏱`,
`↷`, `↗`).

Ratio formatting, shared by every surface: `< 10` → one decimal (`2.4×`); `< 100` →
integer (`23×`); otherwise `99×+`, so a chop that ran 400× its interval cannot widen a
column.

**Sidebar** — the scanner. The chop chip shows the **worst** sampled ratio in the
window, so a collapsed-then-expanded tree tells the same story every time. The
lumberjack row carries a roll-up chip counting only its `over` chops, placed before the
existing cycles/errors chip, because a collapsed fold hides the chop rows entirely and
the parent must still show that something under it needs attention.

```
▌ [*] hooks           ⚠1   12c
  └─ [✓] fix_hooks
  └─ [●] mentor_sweep    ⚠ 4.0×
  └─ [✓] refresh_docs    instance
▌ [*] checks               31c
  └─ [✓] bead_triage     ⚠ 1.2×      <- dim: intermittent
```

**Lumberjack overview** — the table. Every other column in that table describes the
chop's latest run, so the new `PACE` column does too: the latest sampled run's ratio,
amber and `⚠`-prefixed when that run was over, dim otherwise, `—` when nothing about the
latest run is sampleable. The window-level history is stated in prose directly below the
table, which is also where the remediation belongs, since this is the view where the
`interval` is read and edited.

Column widths change from `NAME 20 / LAST RUN 14 / WHEN 14 / DURATION 10` to
`NAME 20 / LAST RUN 14 / WHEN 12 / DURATION 10 / PACE 10`. That totals 66 plus the
2-cell indent = exactly the 68-cell rule the table already draws, so the wide layout
neither overflows nor changes its narrow-mode threshold (`_NARROW_OVERVIEW_WIDTH = 60`).
`WHEN` at 12 still fits every `format_relative_time` output.

```
  CHOPS
  ────────────────────────────────────────────────────────────────────
  NAME                LAST RUN      WHEN      DURATION      PACE
  ────────────────────────────────────────────────────────────────────
  fix_hooks           ✓ success     22s ago       3.1s      0.1×
  mentor_sweep        ● running     4m ago       4m 2s   ⚠ 4.0×
  refresh_docs        ✓ success     1h ago       12.4s      0.2×
  bead_triage         ↷ skipped     55s ago        0ms         —

  ⚠ mentor_sweep reached 4.0× this lumberjack's 60s interval on its last run.
    Raise `interval` or move the chop into its own lumberjack.
```

With more than one affected chop the advisory collapses to a count and names the worst;
`intermittent` chops are named on their own line
(`bead_triage exceeded the interval on 2 of its last 8 runs`). The advisory renders only
when at least one chop is marked, and appends a dim
`(configured interval — lumberjack not running)` when `interval_source == "config"`.

**Chop detail header** — the explanation. One segment after `Took:` / `Elapsed:`, keyed
on the run currently being viewed rather than the window: `⚠ 4.0× of 60s interval`.
Selecting the chop the sidebar flagged answers "how far over, and against what" without
a second hop.

**Help guide** — one legend line in the "Lumberjacks own chops" card of `AxeOnboarding`,
which the `?` panel's Guide view renders, plus the matching paragraph in `docs/axe.md`'s
"AXE Tab Views".

### Where the logic lives

The classification is domain logic by the `rust_core_backend_boundary` test — a CLI or
web frontend showing the same fleet would have to agree with the TUI about which chops
overrun — so it goes in `sase-core` as a pure function, mirroring `axe_status`'s shape:
`classify_chop_overrun(request) -> verdict`, no clock, no filesystem, no process I/O.
Python passes `now`, the interval, and the already-cached run history; core returns the
level and the ratios. Formatting (`2.4×`, colors, glyphs, column widths) stays in the
TUI where it belongs.

The call is made per chop inside `collect_chop_snapshot`, which both the full collector
and the targeted `y` refresh already funnel through, so one code path produces complete
snapshots for both. Roughly one FFI hop per configured chop per refresh, off the event
loop, on a 10-second cadence.

**The indicator is presentation-only and must never break the tab.** The TUI collector
wraps the facade call in `try/except Exception` and degrades to "no verdict", so a
missing, stale, or erroring `sase_core_rs` binding leaves the AXE tab rendering exactly
as it does today. The facade itself stays strict (`require_rust_binding`, schema-version
check, raise on error) — defensiveness belongs at the presentation call site, not in the
backend contract.

### Staleness, stated deliberately

Verdicts are computed at collection time, so a run that crosses its interval between
refreshes is marked on the next collection — at most `refresh_interval` (10 s default)
later, and immediately on any targeted `y` refresh. This is accepted rather than worked
around: threading a "deadline" instant into the render path to make live marks
instantaneous would split the rule across the Rust/Python boundary for a bounded and
small delay on a diagnostic that is fundamentally about a pattern across runs.

## Rejected alternatives

- **Flag chops whose configured `timeout` exceeds the interval.** A static, config-only
  risk signal. It is a different claim ("could overrun") from the one asked for ("did
  overrun"), and mixing the two in one glyph makes both untrustworthy. Worth considering
  later as a `sase axe chop doctor` diagnostic, not as a tab mark.
- **A configurable warning threshold (e.g. mark at 80% of interval).** The interval is
  an objective budget; ≥ 100% needs no tuning knob and no config surface. Adding a
  "near" tier would also dilute the mark the user actually asked for.
- **Latest-run-only marking.** Simplest rule, but it flaps on any chop that alternates,
  which is exactly the population this feature exists to expose.
- **Compare against `interval × run_every` for cadenced chops.** Wrong: the tick still
  blocks for the full run on the ticks where a cadenced chop fires.
- **Reusing `classify_axe_status`.** That classifier answers a different question
  (daemon/lumberjack process health,
  `sase-core: crates/sase_core/src/axe_status/classify.rs`) from a different request
  shape, and the AXE tab does not call it. A separate module keeps both schema versions
  independent.
- **Marking in `sase axe status` / `sase axe chops` too.** Out of scope here; putting
  the rule in core is what makes that a small follow-up rather than a second
  implementation.

## Overrun classifier in the Rust core

Work in the `sase-org/sase-core` checkout opened through the `/sase_repo` skill
(`sase repo open gh:sase-org/sase-core -r "..."`). Follow `axe_status`'s module shape
exactly (`crates/sase_core/src/axe_status/mod.rs`).

1. **New module** `crates/sase_core/src/axe_overrun/` with `mod.rs` (module doc + `mod`
   declarations + `pub use`), `wire.rs`, `classify.rs`, `tests.rs`; register
   `pub mod axe_overrun;` in `crates/sase_core/src/lib.rs` in alphabetical position
   (between `axe_chop` and `axe_status`).

2. **`wire.rs`** — `pub const CHOP_OVERRUN_SCHEMA_VERSION: u32 = 1;`, a
   `ChopOverrunError { code, path, message }` mirroring `AxeStatusError`
   (`axe_status/wire.rs:9-31`), and:

   ```jsonc
   // ChopOverrunRequestWire  (serde deny_unknown_fields)
   {
     "schema_version": 1,
     "now": "2026-08-12T09:15:00-04:00",   // RFC3339, host clock
     "interval_seconds": 60,               // > 0
     "runs": [                             // newest-first, as cached; may be empty
       {
         "status": "success",
         "started_at": "2026-08-12T09:12:35-04:00",
         "duration_ms": 145000,
         "script_duration_ms": null        // Option<i64>, serde default
       }
     ]
   }
   // ChopOverrunWire
   {
     "schema_version": 1,
     "level": "none" | "intermittent" | "over",
     "sampled_runs": 7,
     "over_runs": 3,
     "worst_ratio": 2.42,        // Option<f64>, null when sampled_runs == 0
     "worst_blocking_ms": 145000,// Option<i64>
     "latest_ratio": 2.42        // Option<f64>, newest sampled run
   }
   ```

3. **`classify.rs`** —
   `pub fn classify_chop_overrun(request: &ChopOverrunRequestWire) -> Result<ChopOverrunWire, ChopOverrunError>`
   implementing the four steps under "The rule" above. Request-level validation errors:
   schema mismatch, blank/unparsable `now`, `interval_seconds <= 0`. Per-run defects are
   **not** errors — a run with an unparsable `started_at`, a negative duration, or an
   unknown status string is dropped from the sample, so one corrupt metadata file cannot
   blank the panel.

4. **`tests.rs`** — cover, at minimum: newest sampled run over ⇒ `over`; only an older
   run over ⇒ `intermittent`; a fast `skipped` newest run in front of an over `success`
   still ⇒ `over` (the sampling rule is what makes this work); `action_succeeded` with a
   huge `duration_ms` and no `script_duration_ms` ⇒ ignored, `sampled_runs` excludes it;
   the same entry _with_ `script_duration_ms` ⇒ sampled on that value; a live `running`
   run past the interval ⇒ `over` from `now - started_at`; exact equality ⇒ over;
   `interval_seconds = 0` ⇒ error; empty history ⇒ `none` with null ratios; unparsable
   `started_at` ⇒ dropped, not fatal.

5. **Binding** in `crates/sase_core_py/src/lib.rs`, following `py_classify_axe_status`
   (`lib.rs:6449-6477`): `chop_overrun_wire_schema_version() -> int` and
   `classify_chop_overrun(request: dict) -> dict`, both registered in the module init
   next to the axe_status pair (`lib.rs:8287-8288`), both listed in the `//! -` binding
   inventory near `lib.rs:154-155`, with round-trip tests in the crate's `tests` module
   asserting the JSON shape and the version constant.

Verify with `just check` from the `sase-core` root (never `cargo test -p sase_core`
alone — it skips the binding tests, per that repo's `AGENTS.md`). Do not hand-edit crate
versions; release-plz owns them.

## Record how long a chop actually blocked its tick

Work in the `sase-org/sase` checkout opened through the `/sase_repo` skill. This phase
is independent of the Rust work and lands on its own.

1. **`src/sase/axe/_state_chops.py`** — add `script_duration_ms: int | None = None` to
   `ChopRunEntry` (`_state_chops.py:44-75`), documented as "wall-clock the lumberjack
   tick was blocked by this chop's script, which `duration_ms` stops representing once a
   launched run is finalized against its agents' lifetime".

2. Add a `script_duration_ms: int | None = None` keyword to `finish_chop_run`
   (`_state_chops.py:289-349`) that writes the key only when not `None`. Because
   `finish_chop_run` reads the existing metadata dict and overwrites only the keys it is
   given, a value written at script-finalization time **survives untouched** through the
   later launched-run finalization in `chop_lifecycle.py:437-449` — that call site needs
   no change at all.

3. **`src/sase/axe/chop_runner_script_lifecycle.py`** — `finalize_script_chop_run`
   already computes the script's wall clock at `:63-64`; pass it through as
   `script_duration_ms=duration_ms` on every status, so the field always means the same
   thing and the classifier needs no per-status special case for entries written from
   this point on.

4. **Harden `read_chop_run`** (`_state_chops.py:367-377`). It currently does
   `ChopRunEntry(**data)` inside a `try/except TypeError` that returns `None`, so a
   metadata file written by a newer sase (one extra key) makes an _older_ reader drop
   the run from history entirely — a real mixed-version window whenever the daemon
   restarts into new code under a long-lived `sase ace`. Filter to known dataclass
   fields before construction, exactly as `AxeStatus` rehydration already does in
   `src/sase/ace/tui/actions/axe_display/_data.py:238-240`. Keep the `except TypeError`
   for genuinely malformed payloads.

5. **Tests** — extend `tests/test_axe_chop_runner_script.py` (script run records
   `script_duration_ms` equal to its `duration_ms`) and
   `tests/test_axe_chop_lifecycle.py` (a `launched` run finalized to `action_succeeded`
   has `duration_ms` overwritten with the agent-lifetime span while `script_duration_ms`
   keeps the script's value). Add a `read_chop_run` test in
   `tests/test_axe_lumberjack_state.py` proving an unknown key in the JSON no longer
   drops the entry.

The new field has no consumer until the next phase; that is intentional. If `symvision`
objects to it, read `sase/memory/symvision.md` with the `/sase_memory_read` skill rather
than deleting or renaming the field.

## Classify each chop while collecting AXE snapshots

Work in the `sase-org/sase` checkout. Build the extension from the `sase-core` checkout
that carries phase `core_classifier` before running tests — in a SASE workspace,
`sase repo open sase-core` then `just install`; otherwise point `SASE_CORE_DIR` at the
checkout and run `just rust-install` (see `docs/rust_backend.md`, "Source / development
workflow"). No `pyproject.toml` version-window edit: the published `sase-core-rs` floor
is owned by the release lane, not by feature work.

1. **Facade** `src/sase/axe/chop_overrun.py`, modeled on
   `src/sase/axe/status_models.py:37-53`:
   - `CHOP_OVERRUN_WIRE_SCHEMA_VERSION = 1`;
   - `@dataclass(frozen=True) class ChopOverrun` with
     `level: Literal["none", "intermittent", "over"]`, `sampled_runs`, `over_runs`,
     `worst_ratio`, `worst_blocking_ms`, `latest_ratio`;
   - `classify_chop_overrun(*, now: datetime, interval_seconds: int, runs:   Sequence[ChopRunEntry]) -> ChopOverrun | None`
     — returns `None` (no verdict, no error) when `interval_seconds` is not a positive
     int or `runs` is empty; otherwise checks `chop_overrun_wire_schema_version()`
     against the module constant, calls `classify_chop_overrun` through
     `require_rust_binding`, and rehydrates the response. Export both names from
     `src/sase/axe/__init__.py` alongside `classify_axe_status`.

2. **`ChopSnapshot`** (`src/sase/ace/tui/actions/axe_display/_data.py:70-96`) gains
   `interval_seconds: int | None = None`,
   `interval_source: Literal["runtime", "config"] | None = None`, and
   `overrun: ChopOverrun | None = None`. **`LumberjackSnapshot`** gains
   `overrun_chop_count: int = 0` (chops at level `over`) and
   `intermittent_chop_count: int = 0`.

3. **`collect_chop_snapshot`** (`_data.py:171-215`) takes `interval_seconds` and
   `interval_source` keywords, and after building `runs` calls the facade inside
   `try/except Exception` → `overrun = None`, emitting
   `trace_event( "axe.chop_overrun.unavailable", ...)` on the failure path. Disabled
   chops and chops whose `config_status` is not `configured` are never classified — they
   do not run.

4. **`collect_axe_status_data`** (`_data.py:268-325`) resolves the effective interval
   once per lumberjack (`status.interval` when a positive int, else
   `config.lumberjacks[name].interval`; record the source), passes it into every
   `collect_chop_snapshot` call, and fills the two roll-up counts on
   `LumberjackSnapshot`.

5. **Targeted refresh**
   (`src/sase/ace/tui/actions/axe_display/_loader_refresh.py:182-231`) rebuilds one
   `ChopSnapshot` from `existing`; it must carry `existing.interval_seconds` and
   `existing.interval_source` into the `collect_chop_snapshot` call, or a `y` press
   would silently erase the mark on the chop the user is looking at. After splicing the
   new snapshot into the parent (`_loader_refresh.py:224-228`), recompute that
   `LumberjackSnapshot`'s two counts from its `chops` list.

6. **Tests** — `tests/ace/tui/test_axe_collector.py`: a lumberjack whose status interval
   and config interval disagree classifies against the status interval and reports
   `interval_source == "runtime"`; a stopped lumberjack with no status falls back to
   config; a chop with a slow newest run gets `level == "over"`; a disabled chop gets no
   verdict; a raising/absent binding leaves `overrun is None` and every other field of
   the snapshot intact. Add a facade unit test file for the schema-version guard and the
   `interval_seconds <= 0` → `None` path.

## Render the overrun mark across the AXE tab

Work in the `sase-org/sase` checkout, on top of `snapshot_wiring`. Presentation only —
no new reads, no policy decisions, no work in a render path beyond formatting values
that are already on the snapshot (`sase/memory/tui_perf.md` rules 1, 8, 12).

1. **Shared formatting** in `src/sase/ace/tui/widgets/_axe_dashboard_render.py`: an
   `OVERRUN_STYLES` mapping (`over` → `bold #FFAF5F`, `intermittent` → `dim #FFAF5F`),
   `format_overrun_ratio(ratio) -> str` implementing the `2.4×` / `23×` / `99×+` ladder,
   and `overrun_chip(overrun) -> tuple[str, str] | None` returning `("⚠ 4.0×", style)`.
   Every surface below uses these; no surface re-derives a threshold.

2. **Sidebar** (`src/sase/ace/tui/widgets/bgcmd_list.py`):
   - `_format_chop_option` (`:275-319`) appends the chip after the name and after any
     `instance` chip, using the chop's **worst** ratio. Disabled chops never get one.
   - `_format_lumberjack_option` (`:227-273`) prepends a roll-up chip `⚠N` in
     `bold #FFAF5F` before the existing cycles/errors chip when `N > 0`; `N` counts only
     `over` chops. Feed it through a new optional
     `lumberjack_overruns: dict[str, int] | None = None` parameter on `update_list`,
     populated in `src/sase/ace/tui/actions/axe_display/_render.py:264-283` from the
     cached `LumberjackSnapshot` counts, matching the existing optional-cache parameter
     style.
   - Both chips participate in `_last_line_cell_len` width sizing, so the panel grows
     only while a problem exists.

3. **Overview table** (`_axe_dashboard_render.py`): re-space `render_wide_chop_table`
   (`:191-242`) to `NAME 20 / LAST RUN 14 / WHEN 12 / DURATION 10 / PACE 10` with a
   `PACE` header in the existing `bold #87D7FF`, and render the latest sampled run's
   ratio right-aligned — `⚠ 4.0×` in the level style when that run was over, a dim plain
   ratio otherwise, dim `—` when the latest run is not sampleable. Disabled and
   never-run rows keep their existing em-dash treatment. `render_compact_chop_list`
   (`:245-280`) appends ` · ⚠ 4.0×` to its metadata line.

4. **Advisory line** in `AxeOutputSection.update_lumberjack_overview`
   (`src/sase/ace/tui/widgets/_axe_dashboard_output.py:119-202`), rendered directly
   below the chops table and above `RECENT LOG`, only when at least one chop is marked:
   - one over chop →
     `⚠ {name} reached {ratio} this lumberjack's {interval}s interval on its last run.`
   - several →
     `⚠ {n} chops reached this lumberjack's {interval}s interval (worst {ratio}: {name}).`
   - a second dim line for intermittent chops →
     `{name} exceeded the interval on {k} of its last {m} runs.`
   - always followed by `Raise \`interval\` or move the chop into its own lumberjack.`
   - append a dim `(configured interval — lumberjack not running)` when
     `interval_source == "config"`. Keep it inside the 68-cell rule width and make it
     wrap sanely in the narrow layout.

5. **Chop detail header** (`src/sase/ace/tui/widgets/_axe_dashboard_status.py:294-367`):
   after the `Took:` / `Elapsed:` segment, add `│ ⚠ {ratio} of {interval}s interval`
   when the **displayed run** is itself over. Thread the needed values through
   `update_chop_display` from `AxeDashboard.update_chop_run_display`
   (`src/sase/ace/tui/widgets/axe_dashboard.py:285-351`), which already has the
   `ChopSnapshot`. Do not touch `_axe_chop_result_card.py`: its render cache is keyed on
   `(lumberjack, chop, run_id, status, finished_at, width)`
   (`_axe_chop_result_card.py:72-80`) and would serve a stale card after an interval
   change.

6. **Help and docs**: one legend line in `AxeOnboarding._build_chops_card`
   (`src/sase/ace/tui/widgets/axe_onboarding.py:149-185`) — `⚠ 2.4×` marks a chop that
   ran as long as its lumberjack's interval; raise the interval or give it its own
   lumberjack — which the `?` panel's Guide view picks up automatically. Extend the "AXE
   Tab Views" section of `docs/axe.md` (around `docs/axe.md:983-1000`) with a paragraph
   stating the rule, the two levels, the deliberate sidebar-worst / table-latest split,
   and that agent-launch lifetime is excluded from the measurement.

7. **Tests**:
   - `tests/ace/tui/widgets/test_bgcmd_list_formatters.py` — chop chip present/absent,
     bold vs dim by level, no chip on a disabled chop, lumberjack roll-up count and its
     order relative to the cycles/errors chip.
   - `tests/ace/tui/widgets/test_axe_dashboard_lumberjack_overview.py` — PACE column
     values and alignment in the wide layout, the compact layout's chip, the advisory
     line's three shapes, and the `(configured interval …)` suffix. Assert the wide
     table's rendered line width still matches its rule.
   - `tests/ace/tui/widgets/test_axe_dashboard_status_section.py` — the detail-header
     segment appears for an over run and is absent for a fast one.
   - A formatting unit test for the `2.4×` / `23×` / `99×+` ladder.
   - PNG snapshots via `tests/ace/tui/visual/_ace_axe_png_snapshot_fixtures.py` +
     `test_ace_png_snapshots_axe.py`: `axe_chop_overrun_120x40` (tree with one `over`
     chop, one `intermittent` chop, a healthy chop, and the advisory line) and
     `axe_chop_overrun_narrow_70x36` (compact layout). Accept with
     `--sase-update-visual-snapshots` and eyeball the produced PNGs before committing —
     this phase is where "beautiful" is actually decided.

## Verification

Per repo, before landing:

```bash
# sase-core checkout
just check                      # fmt + clippy + full workspace tests incl. binding tests

# sase checkout
just install                    # rebuilds sase_core_rs from the sase-core checkout
just check                      # whole-repo lint gates + diff-scoped tests
just test-visual                # AXE PNG snapshot suite
```

Run `just check-full` in the `sase` checkout before the epic's combined tree lands.

Then confirm by eye in a live `sase ace`, on the AXE tab:

- a lumberjack with a chop that outruns it shows the chip on the chop row, the roll-up
  on the parent row, and keeps showing the roll-up when the parent's fold is collapsed;
- the overview table's PACE column agrees with its DURATION column, and the advisory
  line names the same chop;
- selecting that chop shows `⚠ N× of Ms interval` in the detail header;
- paging older runs with `Ctrl+N` / `Ctrl+P` moves the detail-header segment with the
  run being viewed while the sidebar chip stays put;
- a chop that launches agents and whose script is fast carries **no** mark, however long
  its agents run — the single most important thing to confirm manually;
- `?` → Guide explains the glyph.

## Out of scope

- `sase axe status`, `sase axe chops`, and any CLI or notification surface. The rule
  lands in core precisely so those become small follow-ups; none of them change here.
- Changing scheduling behavior: no auto-splitting of lumberjacks, no interval
  auto-tuning, no new inhibit. This epic is diagnosis only.
- A "near the interval" warning tier and any configurable threshold.
- Flagging chops whose configured `timeout` exceeds the interval (static risk rather
  than observed overrun).
- Sorting, filtering, or a jump-to-next-overrunning-chop keybinding.
- The lumberjack's existing `Tick overrun` log line and its wording.
