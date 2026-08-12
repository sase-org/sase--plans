---
tier: epic
title: ACE startup — take badge classification, hidden-row repair, and a double ProjectSpec
  parse off first paint
goal: Warm `sase ace` time-to-interactive drops from roughly 3.5–4 s to under 2 s
  on athena, the part of startup that grows with every dismissed agent stops growing,
  and a durable startup telemetry record makes both claims checkable in a real terminal
  instead of modelled from component measurements.
phases:
- id: telemetry
  title: Durable startup telemetry
  depends_on: []
  size: small
  description: 'telemetry: record one JSONL row per ACE session carrying both a visible-surface-ready
    and an all-surfaces-ready time plus the component budget, make the loader-stage
    log threshold env-overridable so sub-2 s stages are capturable, and document the
    before/after capture recipe.'
- id: imports
  title: Two module-level import defects
  depends_on: []
  size: xsmall
  description: 'imports: move the `sase.axe.state` import in toast_log out of module
    scope, drop the module-level `unittest.mock` import from the Patch loader in favor
    of a sys.modules-guarded check, and add a subprocess import-graph guard test.'
- id: badges
  title: Deferred persisted diff-badge classification
  depends_on:
  - telemetry
  size: medium
  description: 'badges: stop classifying persisted diff badges inside the loader''s
    status-override pass and compute them for visible rows only in a coalesced background
    pass modeled on the existing bead-warmup and live-hint mixins, with carry-over
    across reloads so badges do not flicker on refresh.'
- id: repair
  title: Read-only freshness policy for ACE's Tier-1 index query
  depends_on:
  - telemetry
  size: medium
  description: 'repair: add a freshness knob to the artifact-index query wire in sase-core
    so ACE''s startup and auto-refresh queries skip hidden-row repair and per-row
    marker revalidation, stop selecting record_json in refresh_stale_rows, and run
    one coalesced revalidating reconcile after first paint on a long cadence.'
- id: axe
  title: AXE collect stops parsing every ProjectSpec twice
  depends_on:
  - telemetry
  size: small
  description: 'axe: route the two global runner counters through one shared cached
    Patch snapshot instead of two uncached full-archive parses, and end the startup
    stopwatch on the initially visible tab so a future hidden-tab feature cannot silently
    regress every startup mode.'
- id: land
  title: Land the epic
  depends_on:
  - telemetry
  - imports
  - badges
  - repair
  - axe
  size: small
  description: 'land: re-measure the full budget in a real terminal against the phase
    `telemetry` baseline, file the named follow-ups with /sase_new_task, and close
    the epic with an honest reading of what each phase bought.'
proposed_by: bbugyi200.athena.yo
create_time: 2026-08-12 11:36:00
status: done
bead_id: sase-k3
---

- **PROMPT:** [prompts/202608/ace_startup_critical_path.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/ace_startup_critical_path.md)
- **BEAD:** [sase-k3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-k3/README.md)

# Plan: ACE startup — stop paying for hidden rows, persisted diffs, and a double archive parse before first paint

## Why this plan exists

`sase ace` takes upwards of 5 s from launch to the startup stopwatch stopping. Three
independent research passes converged on a budget model
(`research:202608/ace_startup_critical_path/ace_startup_critical_path.md`); this plan
re-measured every load-bearing number against today's tree before committing to an
ordering, and found one significant cost that none of the three passes attributed
correctly.

Startup is `import + max(agents first load, axe first load)`, gated by
`_maybe_end_startup_stopwatch` (`src/sase/ace/tui/actions/_startup_mount.py:162`), which
waits on **both** `_agents_first_load_done` and `_axe_first_load_done` regardless of
which tab is visible. Rendering, widget mounting, and Textual are not the problem and
must not absorb effort here.

## What was measured while writing this plan

All figures below were taken on athena at `sase` master `2f1512c7c` and `sase-core`
`95c5028`, in this workspace's own `.venv` (Python 3.14), against the live `~/.sase`
state, at loadavg 25–41. **The host was busy, so absolute numbers are inflated by
roughly 15–40% against a quiet host. The A/B deltas are the portable result; do not
treat the absolute column as a target.**

### Import

| Metric                                        |                Measured |
| --------------------------------------------- | ----------------------: |
| Modules imported by `import sase.ace.tui.app` |                   2,402 |
| Sum of `-X importtime` self-times             |                 1.426 s |
| `sase.*` share                                | 1.087 s / 1,677 modules |
| `sase.ace.*` share                            |   0.451 s / 787 modules |
| Wall clock, warm, best of 5                   |                  1.87 s |

Reproduces the research exactly on module counts (2,402 vs 2,401–2,402). Not one slow
import: 1,677 `sase` modules averaging ~0.65 ms.

### Agents first load

Fresh process per run, real dismissed-agent set,
`load_agents_from_disk_with_state(..., source="startup")`, with the classifier stubbed
at the _effective_ call site (`_agent_status_overrides._classify_diff_badges`, **not**
`_agent_status_diff.classify_diff_badges`, which is not the live reference):

| Mode               |               Load time | Classify calls | Unique paths | Unique bytes | Rows |
| ------------------ | ----------------------: | -------------: | -----------: | -----------: | ---: |
| production         | 1.876 / 1.834 / 1.818 s |          1,178 |          492 |      19.8 MB |  213 |
| classifier stubbed | 1.515 / 1.452 / 1.375 s |              0 |            0 |            0 |  213 |

**Badge delta ≈ 0.40 s, ~22% of the loader.**

> **Correction to the research report.** The adjudicating pass put this at **0.8 s** and
> called it the largest single warm item, explicitly overruling report `__a`'s 0.42–0.46
> s as an under-measurement. Re-measured here at the same patch point with the same call
> and byte counts, the delta is **0.40 s** — `__a`'s figure, not the adjudicated one.
> Treat 0.4 s as the number to beat, and re-measure rather than assume the 0.8 s
> projection.

### The Tier-1 index query

Fresh process per run, `active_limit=1000`, `recent_completed_limit=200`:

| Query                                  |            Time | Records |
| -------------------------------------- | --------------: | ------: |
| production (`include_hidden=False`)    | 0.867 / 0.638 s |     531 |
| repair skipped (`include_hidden=True`) | 0.478 / 0.380 s |     862 |
| zero-record floor (repair still runs)  | 0.326 / 0.294 s |       0 |

**Repair delta ≈ 0.26–0.39 s**, to return _fewer_ rows. The zero-record floor proves the
floor is the repair pass, not SQLite and not selection.

### The repair pass repairs nothing

Recomputing every hidden row's `MarkerSignatures` in Python — `len:secs:nanos` per
marker, `prompt_step_*.json` entries sorted and joined with `|`, exactly as
`index.rs:2175` builds them — and diffing against the stored `*_sig` columns:

```text
index rows: 6740  (visible 2034, hidden 4706)
hidden rows checked: 4706
hidden rows with a STALE signature: 0 (0.00%)
elapsed: 0.22 s
```

**Zero.** Reproduced independently of the research pass that first reported it.
Freshness is already guaranteed by `update_agent_artifact_index_for_marker_mutation`
(`src/sase/core/agent_artifact_index_lifecycle_mutations.py:149`, ~92 write-path call
sites) and by ACE's inotify `ArtifactWatcher`. Query-time repair is a redundant third
detector, and ~0.3 s and thousands of syscalls per query buy exactly nothing today.

### AXE — the cost is somewhere nobody looked

`collect_axe_status_data` (`src/sase/ace/tui/actions/axe_display/_data.py:263`) measures
0.512 / 0.520 / 0.543 s, matching the research. But the research attributed that cost to
"log tails, chop history, and historical run detail" and prescribed a summary/detail
split. **That attribution is wrong.** Instrumenting the collector's own helpers:

| Component                                         |              Cost | Calls |
| ------------------------------------------------- | ----------------: | ----: |
| `get_axe_status()`                                | **0.427–0.469 s** |     1 |
| ├ `count_hook_runners_global()`                   | **0.203–0.223 s** |     1 |
| ├ `count_agent_runners_global()`                  | **0.194–0.211 s** |     1 |
| └ everything else in it                           |          < 0.02 s |     — |
| all 31 `collect_chop_snapshot` calls              |           0.034 s |    31 |
| all 9 `read_lumberjack_log_tail(name, 500)` calls |           0.004 s |     9 |
| `load_axe_config()`                               |           0.010 s |     1 |
| `read_output_log_tail(500)`                       |           0.000 s |     1 |

The chop history and log tails the research wanted to split out cost **0.04 s
combined**. The real cost is that `count_hook_runners_global` and
`count_agent_runners_global` (`src/sase/ace/patch/validation.py:155` and `:172`) each
call the **uncached** `find_all_patches()`, so every AXE collect parses the entire
ProjectSpec archive twice:

| Call                        |    Cold |         Warm (repeat in same process) |
| --------------------------- | ------: | ------------------------------------: |
| `find_all_patches()`        | 0.248 s | 0.222 s, 0.222 s — **no memoization** |
| `find_all_patches_cached()` | 0.239 s |                  **0.002 s**, 0.002 s |

`find_all_patches_cached` already exists (`src/sase/ace/patch/cache.py:102`), keys on
`(mtime_ns, size)` per project file, has an identical signature, and is already what the
TUI's own Patch load uses (`src/sase/ace/tui/actions/patch/_loading.py:102`). AXE simply
never routes through it. This is the cheapest win in the plan and it also lands on every
10 s refresh tick, not just startup.

### Production timing log

`~/.sase/logs/tui_agent_loads.jsonl` (+`.1`), 7,862 records:

| Stage                          |     p50 |     p90 |     max |     n |
| ------------------------------ | ------: | ------: | ------: | ----: |
| `disk`, `source: startup`      | 2.663 s | 4.554 s | 225.4 s |   460 |
| `disk`, `source: auto_refresh` | 2.637 s | 6.811 s | 374.7 s | 6,310 |

**This log is censored.** `_SLOW_LOADER_STAGE_THRESHOLD_SECONDS = 2.0`
(`src/sase/ace/tui/actions/agents/_loading_disk_support.py:24`) means only stages
already over 2 s are written; the observed minimum in the file is 2.000304 s. These are
percentiles _of loads already known to be slow_. They are the right numbers for "why is
it sometimes >5 s" and the wrong numbers for a mean — and they are the reason phase
`telemetry` comes first.

Note also 6,310 `auto_refresh` loads against 460 `startup` loads: every cost in this
plan is paid roughly 14× more often off startup than on it.

## The resulting budget and ordering

| Cost                                        |         Warm | Grows with usage?            | Phase                                         |
| ------------------------------------------- | -----------: | ---------------------------- | --------------------------------------------- |
| Python import                               |   ~1.4–1.9 s | no                           | `imports` (partial); the rest is out of scope |
| Diff-badge classification                   |      ~0.40 s | with referenced diffs        | `badges`                                      |
| Hidden-row repair inside the Tier-1 query   | ~0.26–0.39 s | **yes, ~46 hidden rows/day** | `repair`                                      |
| Double ProjectSpec parse inside AXE collect |      ~0.41 s | with archive size            | `axe`                                         |

Import is the floor every other fix exposes, and it is the one thing this epic does not
seriously attack — the `sase.ace` 787-module graph is its own epic, filed as a
follow-up.

## Explicitly not the problem — do not spend effort here

Ruled out by measurement, in the research and again here:

- **Textual, widget mounting, rendering.** `textual` is ~0.07 s of import; `apply` p50
  is 0.058 s.
- **Agent count.** `disk` p50 is flat across a 5× range in agent count (100–599).
- **AXE log tails, chop run history, and run snapshots.** 0.04 s combined, measured
  above. Do not build the summary/detail split.
- **Reducing the recent-completed limit.** The 0.29 s zero-record floor is
  limit-independent.
- **Deleting artifacts or rebuilding the index.** Destructive, temporary, and restores
  the same slope.
- **Moving loaders to threads.** They are already workers; the stopwatch still waits.

---

## Phase `telemetry` — Durable startup telemetry

Every claim this epic makes is currently unfalsifiable in a real terminal. The startup
stopwatch (`src/sase/ace/tui/widgets/_keybinding_status.py:79`) renders elapsed time and
then throws it away; nothing persists it. The one durable signal that exists is censored
at 2.0 s. The research's own "known gaps" section names this: _"No end-to-end
measurement in a real terminal — the 3.9 s / 5.8 s totals are modeled."_

This phase runs first so that `badges`, `repair`, and `axe` each have a real
before/after instead of a component A/B.

### Deliverables

- **One JSONL record per ACE session**, written after startup settles, to a new durable
  log alongside the existing ones in `sase/logs/tui_telemetry.py`. It must carry, at
  minimum: process start → `on_mount` entry, `on_mount` → first paint, agents first load
  ready, axe first load ready, the two readiness metrics below, agent row count, index
  row count if cheaply available, and `source`/`tier`/`artifact_source` from
  `AgentLoadState`.
- **Two named metrics, recorded from day one:**
  - `all_surfaces_ready_seconds` — today's semantics: both `_agents_first_load_done` and
    `_axe_first_load_done`.
  - `visible_ready_seconds` — the initially visible tab's own surface is interactive.

  Phase `axe` changes which one drives the stopwatch. Recording both from the start is
  what keeps that change from silently invalidating this phase's baseline. Say so in the
  module docstring.

- **Make the loader-stage threshold overridable.**
  `_SLOW_LOADER_STAGE_THRESHOLD_SECONDS` becomes env-overridable
  (`SASE_TUI_LOADER_LOG_THRESHOLD_SECONDS` or similar) with the 2.0 s default unchanged,
  so a verification run can capture sub-2 s stages without a code edit. Follow the
  naming of the existing `SASE_TUI_*` knobs.
- **Do not add startup cost.** Reuse the established shape from
  `_record_slow_loader_stages` (`_loading_disk_support.py:280`): build the record on the
  UI thread, then `spawn_pump_free_task(self, asyncio.to_thread(...))`. Never `open()`
  on the event loop or inside a pump callback. Re-read `sase/memory/tui_perf.md` rules
  1, 2, and 9 before writing the scheduling code.
- **A capture recipe in `docs/perf_runbook.md`**: how to launch, where the record lands,
  and how to compare two runs. The land phase and every later phase will use it.

### Verification

Beyond `just check`: launch `sase ace` in a real terminal at least three times, confirm
one record per session with plausible values, confirm the record is written whether the
initially visible tab is `agents` or `artifacts`, and confirm no new work appears on the
event loop (`SASE_TUI_TRACE=1`, and the stall watchdog stays quiet). **Record the
measured baseline in the phase close note** — that is this phase's real output.

---

## Phase `imports` — Two module-level import defects

Both were found by the research and both reproduce verbatim at HEAD. Neither is
speculative and neither needs measurement to justify.

### Deliverables

- **`src/sase/logs/toast_log.py:21`** — `from sase.axe.state import read_tail_seek` at
  module scope drags `sase.axe` → `sase.agent` → `sase.xprompt` into a _toast logging_
  module. It is used at exactly one site (`:147`). Move it into that function.
- **`src/sase/ace/tui/actions/patch/_loading.py:8`** — `from unittest.mock import Mock`
  at module scope, used only for `isinstance(x, Mock)` test-double detection at `:57`,
  `:59`, `:88–89`, `:327`, `:330`. **Do not simply move the import into those
  functions** — they are on the production Patch-load path, so a function-scoped import
  moves the cost from import time to first-load time rather than removing it. Instead
  add a small private helper that short-circuits on `sys.modules`:

  ```python
  def _is_mock(value: object) -> bool:
      mock_module = sys.modules.get("unittest.mock")
      if mock_module is None:
          return False
      return isinstance(value, mock_module.Mock)
  ```

  If `unittest.mock` was never imported, nothing in the process can be a `Mock`, so the
  fast path is correct, not merely a heuristic. State that reasoning in the docstring.

- **A subprocess import-graph guard test.** Assert in a fresh interpreter (not the
  pytest process, which imports `unittest.mock` itself) that
  `import sase.logs.toast_log` does not pull in `sase.axe`, and that
  `import sase.ace.tui.actions.patch._loading` does not pull in `unittest.mock`. This is
  the regression guard; without it the imports come back.

### Verification

`just check`, plus `-X importtime` module counts before and after — report the delta in
the close note. Expect a small win; report the real number rather than a projection.

---

## Phase `badges` — Deferred persisted diff-badge classification

### The defect

`normalize_loaded_agents` (`src/sase/ace/tui/models/_agent_loader_normalization.py:82`)
calls `apply_status_overrides` with the default `classify_diff_badges=True`, which wires
in `_classify_diff_badges` (`_agent_status_overrides.py:63`) and reads every referenced
persisted diff before first paint: 1,178 calls over 492 unique paths, 19.8 MB, ~0.40 s.

Two facts make this safe to defer:

1. **Nothing in the override pass depends on it.** `classify_diff_badges` runs at the
   very end of `apply_status_overrides` (`_agent_status_apply.py:440–442`), after all
   `diff_path` propagation. It is pure display enrichment.
2. **The repo already established this pattern for exactly this reason.** The docstring
   of `classify_live_file_change_hint` (`_agent_status_diff.py:23`) records that the
   live VCS pencil hint was pulled off the loader because "computing it inline ran
   hundreds of live VCS diffs on the first agents load and dominated startup." The
   persisted-diff classifier is the same shape of work, one step cheaper, still on the
   critical path.

### Deliverables

- **Stop classifying in the loader.** `normalize_loaded_agents` passes
  `classify_diff_badges=False`. **Do not miss the third call site**:
  `_loading_compute_merge.py:182` also calls `apply_status_overrides` with the default
  `True` on the artifact-delta merge path, and it must change too.
  `normalize_live_plan_agents` already passes `False` and needs no change.
- **A new coalesced background mixin**, e.g.
  `src/sase/ace/tui/actions/agents/_loading_diff_badges.py`. Model it on
  `AgentBeadWarmupMixin` (`_loading_bead_warmup.py`) — that is the closest analogue and
  the newer of the two. Copy its shape exactly, including:
  - the four coalescing flags (`scheduled` / `running` / `pending` / `source`),
    initialized in `_state_init.py` next to the existing sets;
  - `spawn_pump_free_task(..., registry_attr=...)` so Textual's pump never awaits it;
  - the `_nav_gate.is_navigating()` deferral with the synchronous timer respawn;
  - `asyncio.to_thread` for the actual disk work;
  - results keyed by `agent.identity`, re-matched against the _current_ agent objects
    after the await, applied with `_try_patch_agent_row`.
- **Candidates are visible rows only** (`self._agents`), and the referenced diff paths
  must be **deduplicated before scheduling** — 1,178 references collapse to 492 paths.
  `diff_has_real_edits` caches on `(path, mtime_ns, size)` (`_diff_badge.py:62`) but a
  cache hit still pays an `os.stat`, so dedupe at the candidate level, not by relying on
  the cache.
- **Both fields**: `diff_has_real_edits` and `linked_file_change_hint`. The
  linked-commit branch (`_classify_linked_commit_diffs` → `agent_commit_diffs`) reads
  persisted per-commit diffs and is where the research suspects the byte volume
  concentrates; confirm or refute that while you are in there and record it.
- **Carry-over across reloads.** Add the badge analogue of `carry_over_live_hints`
  (`_loading_live_hints.py:51`) and call it from the same site in
  `_loading_apply.py:394`. Without it every 10 s auto-refresh drops badges back to the
  fallback and the pencil column flickers. This is not optional polish; the loader runs
  ~14× more often off startup than on it.
- **Schedule it** from `_apply_loaded_agents_prepared_inner`, next to the existing
  `_schedule_live_hint_refresh(source="apply")` and
  `_schedule_bead_confirmation_warmup(source="apply")` calls (`_loading_apply.py:438`,
  `:444`).

### First-paint behavior — state it, do not hide it

With classification deferred, `agent_file_change_hint`
(`src/sase/ace/tui/widgets/_agent_list_render_cache.py:65`) falls through to
`bool(agent.diff_path)` for rows whose badge is still `None`. So on the _first_ paint of
a session:

- a row whose persisted diff touches only plan/prompt bookkeeping briefly shows a pencil
  and then loses it;
- a row whose only change is a linked-repo commit diff briefly shows no pencil and then
  gains one.

Both settle within the deferred pass, and carry-over means it happens once per session
rather than every refresh. This is the same stale-while-revalidate contract the live
hint already ships. Document it in the new module's docstring the way
`_loading_live_hints.py` documents its own.

### Tests

`tests/test_agent_loader_status_override_tale.py` (`:502`, `:546`) and
`tests/test_agent_loader_live_file_change_hint.py` assert loader-time badge
classification and will need to drive the deferred pass instead. Do not weaken them into
"the field is None" assertions — port each to assert the same final state after the
deferred pass runs. Add coverage for: dedup (N references over M paths ⇒ M reads),
coalescing under a burst of refreshes, identity re-matching when the list is rebuilt
mid-flight, and carry-over across a reload.

### Verification

`just check`, plus the phase `telemetry` capture in a real terminal before and after.
Report the agents-load delta and the badge-settle time separately — the second number is
new user-visible latency and should be reported honestly, not folded into the first.

---

## Phase `repair` — Read-only freshness policy for ACE's Tier-1 index query

This phase is worth less warm than `badges` (~0.3 s vs ~0.4 s) but it is the change that
stops startup degrading daily.

### The defect

`query_agent_artifact_index` calls `repair_stale_rows_for_query`
(`sase-core/crates/sase_core/src/agent_scan/index.rs:1251`) _before_ any selection. When
`include_hidden` is false — which the TUI always passes
(`src/sase/ace/tui/models/_agent_loader_artifacts.py:118`) — it pushes the clause
`hidden = 1` and hands every matching row to `refresh_stale_rows` (`:1493`), which
stat-walks 4,706 artifact directories the query has explicitly asked to _exclude_. Per
row that is 8 marker `stat`s, one `read_dir`, and a `stat` per `prompt_step_*.json`
found. Then `select_records` (`:1571`) repeats `MarkerSignatures::from_artifact_dir` for
**every selected row**, so the validation is doubled, not merely broad. The read path
also performs durable writes: `select_records` upserts refreshed rows (`:1597`).

`hidden` is set on dismissal and hidden rows are never pruned. At ~46 new hidden
rows/day and a 10 s default refresh interval (`refresh_interval: int = 10`,
`src/sase/ace/tui/app.py:227`), this is the mechanism by which startup gets worse every
week.

### Deliverables — sase-core

Work in `sase-core` through `/sase_repo` (`sase repo open sase-core`). Read that repo's
`AGENTS.md` first: release-plz owns versions, do not hand-edit `Cargo.toml` versions,
and verify with its own `just check` — never `cargo test -p sase_core` alone.

- **Add a freshness knob to `AgentArtifactIndexQueryWire`** (`index.rs:58`). Use a
  serde-defaulted enum, not a bool, so a third policy can be added later without another
  wire decision:

  ```rust
  #[serde(default)]
  pub freshness: AgentArtifactIndexFreshness,  // Revalidate (Default) | Cached
  ```

  Serialize as `"revalidate"` / `"cached"`.

- **`Cached` behavior:** skip `repair_stale_rows_for_query` entirely; in
  `select_records`, skip the per-row `MarkerSignatures::from_artifact_dir`, decode
  `record_json` directly, and perform **no** `upsert_record`. Selection itself is
  unaffected — `active_where` / `completed_where` filter purely on stored columns, and
  `record_matches_selection` (`:1803`) only reads the DB, never the artifact tree.
- **`Revalidate` remains the default and is byte-for-byte today's behavior.** Every
  other caller keeps it.
- **Independently, stop selecting `record_json` in `refresh_stale_rows`** (`:1499`). The
  loop never reads `row.record_json` — it only uses `artifact_dir`, `stored`, and
  `row_projects_root` — so ~29.3 MB of blob reads per revalidating query is pure waste.
  Add a sig-only SELECT and a `PendingRow`-without-`record_json` constructor rather than
  changing `pending_row_from_sql`, which `select_terminalization_candidates` (`:1278`)
  legitimately needs the blob for. This benefits every remaining revalidating caller,
  including this phase's own background reconcile.
- **Rust tests:** a `Cached` query returns the same rows as a `Revalidate` query over a
  fresh index; a `Cached` query over an index whose on-disk markers changed out of band
  returns the _stored_ record and does **not** write; an unknown/absent `freshness` key
  deserializes to `Revalidate` (extend the existing
  `index_query_wire_round_trips_active_limit` legacy-payload test at `:2836`).

### Deliverables — sase

- Add `freshness` to `AgentArtifactIndexQueryWire`
  (`src/sase/core/agent_scan_wire_records.py:109`) defaulting to `"revalidate"`, and
  emit it from `agent_artifact_index_query_to_dict`
  (`src/sase/core/agent_scan_wire_conversion.py:84`).
- Pass `freshness="cached"` from the TUI's Tier-1 loader query
  (`_agent_loader_artifacts.py:112`). Leave the other five `AgentArtifactIndexQueryWire`
  construction sites (bead pages, family attach candidates, `agents/cli_index.py`,
  `agents_sync`, `sdd`) on the default.
- **One coalesced revalidating reconcile after first paint**, through the existing
  refresh coordinator. Constraints, not a prescription: it must be coalesced like the
  other deferred passes, must not run on the pump, must respect the navigation gate, and
  must run on a **much longer cadence than the 10 s auto-refresh** —
  `sase/memory/tui_perf.md` rule 10 is exactly this case ("periodic ticks revalidate;
  recomputes get a longer cadence"). Once after first paint plus a multi-minute interval
  is the recommended shape. Do not put it back on the 10 s tick; that reinstates the
  cost this phase removes.

### Compatibility — this is safe in both directions

`AgentArtifactIndexQueryWire` has no `#[serde(deny_unknown_fields)]`, so:

- a **new sase** sending `freshness` to an **old published core** has the key ignored
  and gets today's revalidating behavior — correct, just slower;
- an **old caller** omitting the key against a **new core** gets `Revalidate` by
  default.

Feature and master CI build `sase-core` from source (`.github/workflows/ci.yml:41`), so
this phase is testable end to end before any release. The published-floor job
(`release-core-floor-smoke`) runs only on the release-please branch, and the Justfile
already states that dev installs build from the local checkout and that the
release-branch reconciler ratchets the published window at release time. **Do not
hand-edit the `sase-core-rs` pin in `pyproject.toml`.**

### The correctness argument

This is not "accept staleness for speed". The pass repairs **0 of 4,706** rows today
(measured above), because ~92 lifecycle upsert sites and the inotify `ArtifactWatcher`
already cover freshness. This removes a third detector for events two cheaper mechanisms
already catch, and keeps a post-paint reconcile as the backstop.

### Do not land the mtime gate here

Report `__b` proposed gating each row on one directory `mtime`. That gate is **unsound
today**: an in-place `open(path, "w")` rewrite of an existing marker does not move the
parent directory's mtime, and SASE has such writers: `write_done_marker`
(`src/sase/axe/runner_artifacts.py:216`) writes `done.json` through a plain
`open(done_path, "w")` at `:277`. Re-counted here against the live tree, **1,535 of
6,740 artifact directories (22.8%)** already hold a marker newer than their own
directory — 1,501 `agent_meta.json`, 51 `done.json`, 31 `prompt_step_*.json`, 30
`workflow_state.json`, 1 `waiting.json`. The gate becomes a good follow-up _after_
marker writes go through an atomic-replace helper — filed in `land`, not built here.

### Verification

`just check` in `sase` and `just check` in `sase-core`. Then, with a scale fixture
approximating today's host (≥4,700 hidden rows, ~530 selected rows), assert:

1. the startup query's syscall count is **independent of hidden-row count**;
2. the startup query performs **zero** artifact-tree `stat`/`read_dir` calls and
   **zero** writes (today: tens of thousands and thousands respectively);
3. agents data is painted **before** reconciliation begins, and reconciliation converges
   to the same row set a revalidating query returns;
4. warm cached selection plus decode stays under 250 ms.

`strace -c` on the two query modes is the practical way to check (1) and (2); the
research's reproduction section has the recipe.

---

## Phase `axe` — AXE collect stops parsing every ProjectSpec twice

### The defect

`get_axe_status()` (`src/sase/axe/_process_status.py:10`) calls
`count_hook_runners_global()` and `count_agent_runners_global()`
(`src/sase/ace/patch/validation.py:155`, `:172`). Each independently calls the
**uncached** `find_all_patches()`, so one AXE collect parses the whole ProjectSpec
archive twice — 0.203 s + 0.194 s of a 0.543 s collect. There is no memoization: three
consecutive calls each cost the full 0.22 s.

`find_all_patches_cached()` (`src/sase/ace/patch/cache.py:102`) has an identical
signature, revalidates per file on `(mtime_ns, size)` so it is **not** a stale read, and
costs 0.002 s warm. The TUI's own Patch load already uses it, so in an ACE process the
cache is typically already warm by the time AXE collects.

### Deliverables

- **Compute both counts from one Patch list**, obtained from
  `find_all_patches_cached()`. Either give the two counters a shared internal helper
  that takes an already-loaded list, or add a single `count_all_runners_global`-style
  entry point that both the AXE status path and any other caller use. Keep the two
  public function names working.
- **Guard the shared-object hazard.** `PatchSnapshotCache.get_file_specs` returns
  `list(cached[3])` — a shallow copy, so the `Patch` objects themselves are shared
  across callers. The counters are read-only, which is why this is safe _here_. Say so
  in a comment, and do not make `find_all_patches` itself cached wholesale as a shortcut
  — its other ~10 callers include mutating paths (`archive.py`, `revert.py`,
  `restore.py`, workspace-provider plugins).
- **Do not build the AXE summary/detail split.** The research prescribed it on the
  belief that log tails, chop history, and run snapshots dominate. They measure 0.04 s
  combined. Record that correction in the close note so a future reader does not
  re-derive the wrong conclusion from the research report.
- **Make startup readiness visible-surface based.** End the stopwatch when the
  _initially visible_ tab is interactive rather than when every hidden tab's deep data
  is ready (`_maybe_end_startup_stopwatch`, `_startup_mount.py:162`). Today the default
  `initial_tab` is `agents` (`src/sase/ace/tui/app.py:230`) and Agents is the slower
  surface, so this changes nothing in the common case — its value is structural: it
  prevents the next hidden-tab feature from silently regressing every startup mode,
  which is how this regression accumulated. Keep phase `telemetry`'s
  `all_surfaces_ready_seconds` recording unchanged so the baseline stays comparable.

### Verification

`just check`, plus a real-terminal capture confirming `collect_axe_status_data` drops by
roughly 0.4 s once the Patch snapshot cache is warm — which it normally already is in an
ACE process, since the TUI's own Patch load populates it — and that the runner counts
are unchanged. A cold first collect still pays one archive parse; report both. Confirm
the counts still update when a ProjectSpec changes on disk mid-session — that is the
assertion that would catch a wrongly-scoped cache.

---

## Phase `land` — Land the epic

Runs after all four implementation phases are committed and `just check-full` is green
on the combined tree.

### Re-measure, then report honestly

Using phase `telemetry`'s recipe, capture the full budget in a real terminal and report
a single table: import, agents first load, axe first load, `visible_ready_seconds`,
`all_surfaces_ready_seconds`, before and after. Also re-check `disk` p50 in
`tui_agent_loads.jsonl` and `-X importtime` module counts.

The target is warm Agents-tab time-to-interactive **under 2 s, with p95 under 2.5 s** on
athena, with background badge-settle time reported separately. If the measured result
misses that, say so and say by how much — a projection that does not survive measurement
is a finding, not a failure to hide.

### Follow-ups to file

Use `/sase_new_task` for each, one at a time, naming the proposing bead. Record every
outcome — including duplicates the skill corroborates, attachments to another active
epic, and any you decline — in the close note.

1. **The `sase.ace` import graph** — 787 modules and ~0.45 s of self-time for one import
   of the app. After this epic, import is the majority of what remains. This is an epic,
   not a task; size it accordingly and require `-X importtime` before/after rather than
   intuition.
2. **Route all marker writes through an atomic-replace helper, then add the directory
   mtime gate.** `write_done_marker` (`src/sase/axe/runner_artifacts.py:275`) uses a
   plain `open(path, "w")`, and the `run_agent_helpers_artifacts.py` read-modify-write
   helpers update `agent_meta.json` in place; 22.8% of live artifact directories already
   show a marker newer than their directory. Several atomic helpers exist already (e.g.
   `sase/axe/agent_meta.py:write_agent_meta_atomic`). With writes atomic, the gate turns
   the background reconcile from 4,700 × ~13 syscalls into 4,700 × 1. **The task must
   carry the test that would have caught the unsoundness: rewriting an existing marker
   in place still triggers repair.**
3. **Prune or archive hidden index rows.** 4,706 of 6,740 rows are hidden and nothing
   ever removes them short of a full rebuild. Even with the read path fixed, they still
   cost the background reconcile and the index's 103 MB footprint.
4. **Persist the diff-badge signature and result on the artifact row.** Immutable
   persisted diffs are parsed once per process today; they could be parsed once per
   machine. This belongs in the Rust core rather than a third frontend-side cache — take
   phase `badges`' finding on where the byte volume actually concentrates as the input
   to the schema design.
5. **Cold-cache behavior is still unmeasured.** Every measurement in this plan and in
   all three research passes is warm; dropping the page cache needs root. Both `badges`
   and `repair` target cold-sensitive work, so their relative ranking could shift cold.
6. **The plans sidecar clone in this workspace was left with an unresolved `AA` merge
   conflict** on `202608/dynamic_artifact_panes.md`, and `sase repo open plans` failed
   twice with
   `sase_hg_clean failed: git stash push failed: error: could not write index`. File
   only if it reproduces on a clean workspace — it may be local state from a prior agent
   rather than a defect in `sase repo open`.

Then close the epic with a note covering the re-measured budget, the two corrections
this plan makes to the research report (badge cost is ~0.4 s not ~0.8 s; AXE's cost is a
double ProjectSpec parse, not log tails), and every follow-up outcome. Run
`just symvision` after the close.
