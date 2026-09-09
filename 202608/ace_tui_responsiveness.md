---
tier: epic
title: Restore ACE TUI responsiveness
goal: "Pressing a key in `sase ace` never blocks on provider discovery, git
  subprocesses, or filesystem walks; the steady-state agent refresh costs a fraction of
  today's ~800 ms warm / 2-3 s loaded full rebuild; and both are held there by benches
  that fail when the budgets regress.

  "
phases:
  - id: baseline
    title: Fresh perf baseline and durable budget benches
    depends_on: []
    size: small
    description:
      "baseline: capture a current-master keystroke and loader baseline, and add the
      benches that assert the epic's budgets so later phases have a pass/fail gate."
  - id: keypath
    title: Take provider discovery off the keystroke path
    depends_on: []
    size: medium
    description:
      "keypath: make accent/icon resolution for fixed panes table-driven, memoize fixed
      descriptors, and convert resolve_artifacts_subtabs to
      serve-stale-revalidate-in-background so no keystroke can fork git."
  - id: railcalls
    title: Collapse the redundant link-subject resolutions per keystroke
    depends_on:
      - keypath
    size: small
    description:
      "railcalls: cache the resolved LinkSubject per selection and coalesce the three
      call sites so one j/k costs at most one subject resolution instead of three."
  - id: delta
    title: Make the artifact delta the default refresh, not the 2% exception
    depends_on:
      - baseline
    size: medium
    description:
      "delta: replace the all-or-nothing watcher-path classification with partial
      application so an unmapped path no longer discards the whole queued batch and
      escalates to a full reload."
  - id: stepmeta
    title: Remove the per-workflow-step filesystem enrichment from every load
    depends_on:
      - baseline
    size: medium
    description:
      "stepmeta: eliminate the 1,532 per-load enrich_agent_from_meta filesystem calls
      the workflow-step builder makes, the single largest Python cost in the loader."
  - id: marshal
    title: Drop the double tree build in the artifact-index PyO3 binding
    depends_on:
      - baseline
    size: medium
    description:
      "marshal: serialize the query snapshot straight into Python objects instead of
      building an intermediate serde_json::Value tree, halving the GIL-held marshalling."
  - id: projection
    title: Project the heavy record_json leaves off the list-render path
    depends_on:
      - marshal
    size: large
    description:
      "projection: add a list-shaped record projection in sase-core that omits the four
      leaf fields making up 78% of the payload, and hydrate them lazily for the selected
      row."
  - id: viewport
    title: Honour the AgentsViewport contract instead of discarding it
    depends_on:
      - projection
      - delta
      - stepmeta
    size: large
    description:
      "viewport: wire the bounded read window through DirectAgentsDataProvider so a
      refresh stops building 430 agents to paint 12 rows; measurement-gated on whether
      the earlier phases already met budget."
  - id: hygiene
    title: Index retention tooling and self-inflicted stall fixes
    depends_on: []
    size: medium
    description:
      "hygiene: add retention/vacuum tooling for the unbounded index and registry state,
      open the index read-only on query paths, and stop the stall watchdog from
      extending the freeze it measures."
proposed_by: bbugyi200.athena.0ex
bead_id: sase-uv
create_time: 2026-09-09 19:49:39
status: wip
---

- **PROMPT:**
  [prompts/202608/ace_tui_responsiveness.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/ace_tui_responsiveness.md)
- **BEAD:**
  [sase-uv](https://github.com/sase-org/sase--beads/blob/main/pages/sase-uv/README.md)

<!-- sase:links:start -->

## Links

| Relation     | Artifact                                                                                                    | Why                                                                              |
| ------------ | ----------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| derives-from | [research:202608/ace_refresh_loop_and_link_rail_regression/ace_refresh_loop_and_link_rail_regression.md][1] | supplies the telemetry baseline and the two-problem framing this epic implements |

[1]:
  https://github.com/sase-org/sase--research/blob/main/202608/ace_refresh_loop_and_link_rail_regression/ace_refresh_loop_and_link_rail_regression.md

<!-- sase:links:end -->

# Plan: Restore ACE TUI responsiveness

## Context

The companion research report is the evidence base for this epic; read it before
starting any phase. Its central framing is correct and this plan adopts it: there are
**two independent problems**, and conflating them is why neither got fixed.

1. **Acute** — something that landed on 2026-08-27 put expensive, uncached work on the
   _keystroke_ path. `agents_ready` p50 went 4.90 s (08-26) → 11.15 s (08-27).
2. **Chronic** — the steady-state loader rebuilds ~430 agents from ~796 index records
   every 5-10 s to paint ~12 visible rows.

Everything below was re-verified against the working tree at `a646bdaf6` and the live
`~/.sase` state before this plan was written. Where re-verification **contradicts** the
research report, this plan follows the code and says so explicitly — those corrections
are load-bearing, not cosmetic.

### Correction 1: the acute regression is on the keystroke path, not the watcher path

The report attributes the regression to `LinkRail.refresh_from_app()` being invoked "on
the filesystem-watcher path" from `src/sase/ace/tui/_app_watchers.py`. That is wrong.
`_app_watchers.py` holds Textual **reactive** watchers — `watch_current_idx`,
`watch_current_tab`, `watch_current_artifacts_subtab` — which fire on cursor movement
and tab switches. Nothing there is driven by inotify.

The real mechanism is worse, and the stall log names it directly.
`~/.sase/logs/tui_stalls.jsonl` holds 61 records spanning 2026-08-27 14:50–15:32. Seven
have the link path on the main thread; **five of those seven have `last_action: "j"`** —
the plain cursor-move key — and each blocked for 1.52–1.59 s. One captured main-thread
stack, verbatim in shape:

```
on_key                       (_event_keyboard.py:47)
 _handle_link_prefix_key     (link_follow.py:81)
  _link_follow_available     (link_follow.py:99)
   link_edges_for_selection  (actions/link_subject.py:41)
    selected_link_subject    (relations/link_subject.py:162)
     _subject_from_agents    (relations/link_subject.py:120)
      accent_and_icon_for_ref(relations/link_subject.py:55)
       descriptor_for_artifacts_pane_id (artifact_tabs.py:128)
        resolve_artifacts_subtabs       (artifact_tabs.py:83)
         load_project_provider_records  (_artifact_tab_discovery.py:81)
          resolve_sdd_store → get_primary_workspace_dir → get_workspace_name
           bare_git_workspace._run_git  → subprocess.run → fork
```

So: **every key press** runs `resolve_artifacts_subtabs()`, and on a cache miss that
forks a git subprocess and walks marker files. Three independent call sites reach it per
keystroke:

- `on_key` → `_handle_link_prefix_key` → `_link_follow_available`
  (`actions/_event_keyboard.py:47`, dispatched before the fold/checkout/copy handlers,
  so it runs for _every_ key, not just `$`)
- `watch_current_idx` → `refresh_link_rail` → `LinkRail.refresh_from_app`
  (`_app_watchers.py:35-37`)
- `check_action("follow_artifact_link")` → `link_edges_for_selection`
  (`_app_action_availability.py:284`), plus `commands/context.py:275`

This violates `tui_perf.md` rule 8 (render paths never stat/glob) and rule 11 (keystroke
paths must not spawn subprocesses that can prompt interactively — a git credential
prompt seizes the tty). `f49030db6` cached `provider_source_token` and recovered about a
third of the regression, but the cache is keyed on a token with a 0.75 s TTL **that also
changes whenever any project file's mtime changes** — which concurrent agent runners do
constantly. So the cache misses on exactly the workload where it matters.

Measured on this host while idle: `_compute_provider_source_token` 1.7-3.2 ms,
`resolve_artifacts_subtabs` cold 29-99 ms, warm `descriptor_for_artifacts_pane_id` 0.9
µs. The idle floor is small; the 1.5 s stalls are what the same code costs when a fork
contends with ~29 concurrent agent runners. **That variance is the argument**: the work
does not belong on a keystroke path at any measured cost.

### Correction 2: the biggest Python cost is not what the report recommends fixing

A fresh `cProfile` of `load_tiered_agents` (warm, this host, 796 records → 430 agents):

| Cost                                          |           Time | Note                                   |
| --------------------------------------------- | -------------: | -------------------------------------- |
| **`load_workflow_agent_steps_from_snapshot`** |     **333 ms** | 365 records                            |
| — of which `enrich_agent_from_meta`           |         255 ms | **1,532 calls**, filesystem            |
| `query_agent_artifact_index` (Rust binding)   |         321 ms | 12.6 MB, 344,412 py objects            |
| `load_done_agents_from_snapshot`              |         186 ms | 796 records                            |
| `agent_scan_wire_from_dict`                   |         125 ms | dict → dataclass                       |
| `_done_extra_files`                           |         123 ms | 232 calls → 1,647 `realpath`           |
| **total**                                     | **771-851 ms** | 24,567 `lstat` + 7,024 `stat` per load |

Both source reports recommend `record_json` projection as "the single highest-leverage
change". Projection targets the 321 ms Rust binding. It does **not** touch the 333 ms
workflow-step stage, which is larger. `_build_workflow_agent_steps_for_record`
(`models/_loaders/_workflow_snapshot_loaders.py:271`) issues 1,532 of the 1,560
`enrich_agent_from_meta` calls, and the code comments why: a workflow step's
`agent_meta.json` lives in a _different_ artifacts dir than the parent record, so the
snapshot does not carry it. That is a real constraint, not an oversight — which is why
`stepmeta` is its own phase and does not depend on `projection`.

### Correction 3: SQL is not the bottleneck, and projection is subtler than "drop the blob"

Measured directly against the live 194.7 MB index (9,283 rows):

- current 12-column select incl. `record_json`, 1,000 rows: **14 ms**
- same select with `record_json` projected out: **5 ms**

A 9 ms saving. The cost of `record_json` is not reading it — it is
`serde_json::from_str` per row, then `serde_json::to_value`, then `json_value_to_py`,
then Python dataclass construction. The report's Part 3 already says this; the
recommendation section then partly forgets it.

More usefully, the payload is not diffusely large — it is **four leaf fields**. Measured
over the 800 most-recent rows (12.73 MB total, 15.9 KB/row average). Note the two
different bases: not every row has a `workflow_state` or a `done` marker, so "avg B when
present" is over a smaller denominator than the share column.

| Field                       | rows with it | avg B when present | share of total payload |
| --------------------------- | -----------: | -----------------: | ---------------------: |
| `done.step_output`          |      565/800 |              4,403 |                  22.8% |
| `prompt_steps` (whole list) |      800/800 |              3,433 |                  21.6% |
| `agent_meta.linked_repos`   |      799/800 |              1,823 |                  19.7% |
| `workflow_state.steps`      |      324/800 |              5,154 |                  13.6% |
| **those four combined**     |              |                    |              **77.7%** |

Projecting those four leaves is a far smaller blast radius than dropping the nested
objects wholesale: `agent_meta`, `done`, and `workflow_state` keep their names,
statuses, and timestamps, so most of the Python loader is untouched. That is the shape
`projection` should take.

### What the evidence does _not_ support

Do not act on these; they were checked and are not load-bearing:

- **Index write-lock contention.** `open_index` really does open READ_WRITE, replay 25
  DDL statements, and rewrite `schema_version` on every open. But that write costs ~14
  ms and the busy timeout is essentially never hit. Fix it in `hygiene` for correctness
  and multi-writer hygiene, not for speed.
- **Widget/DOM bloat.** Both source reports built a "hundreds of hidden Markdown
  widgets" thesis on the _same single_ stall record, and their class histograms disagree
  with each other. No leak-vs-steady-state conclusion is supportable from n=1. Not in
  scope. If a phase wants it, it needs a dedicated soak first.
- **A long-lived daemon / read-model service.** Report `__a`'s headline recommendation.
  Deferred deliberately: its main benefit is mostly delivered by `projection` +
  `viewport` at a fraction of the risk. Revisit only if the budgets are still missed
  after `viewport`.

### Budgets this epic is held to

From `tui_perf.md` plus the report's proposal. `baseline` turns these into benches.

| Budget                                                | Target                                    |
| ----------------------------------------------------- | ----------------------------------------- |
| keystroke-to-paint p95 (`SASE_TUI_PERF=1`, every tab) | < 16 ms                                   |
| event-loop stall, any                                 | < 250 ms                                  |
| `agents_ready` p50 (`tui_startup.jsonl`)              | < 4.9 s, ideally < 4.0 s                  |
| warm `load_tiered_agents`                             | < 300 ms                                  |
| `load_kind` distribution                              | majority `artifact_delta`, not 96% `full` |

---

## Phase `baseline`: Fresh perf baseline and durable budget benches

The existing `~/.sase/perf/tui_jk.jsonl` capture is from 2026-08-14 and is stale. Every
later phase needs a pass/fail gate, and a "before" number nobody can reproduce is not
one.

1. Capture a current-master baseline and record it in the phase bead:
   - `SASE_TUI_PERF=1 sase ace` keystroke capture per tab → `~/.sase/perf/tui_jk.jsonl`
     p50/p95/max. Follow `docs/perf_runbook.md`.
   - Warm `load_tiered_agents` timing and a `cProfile` cumulative table.
   - Current `tui_startup.jsonl` / `tui_agent_loads.jsonl` percentiles, split by
     concurrent ACE instances as the report's Part 1 does — a quiet hour must not be
     mistaken for a fix.
2. Extend the existing benches (`tests/ace/tui/bench_tui_jk.py`,
   `tests/perf/bench_tui_trace.py`, both `-m slow`) so the budgets above are asserted
   rather than merely printed. Where a budget cannot be asserted deterministically in
   CI, assert a generous ceiling locally and leave the tight number in the printed
   table.
3. Add a bench that fails if provider discovery, a subprocess, or a `stat`/`glob` walk
   is reachable from a synthetic key press. This is the regression gate for `keypath`
   and it must exist _before_ `keypath` lands. Prefer monkeypatching the discovery entry
   points to raise, then driving keys through the Textual test harness — a behavioural
   assertion, not a timing one, so it cannot flake.

Record the baseline numbers on the phase bead. Later phases compare against them.

## Phase `keypath`: Take provider discovery off the keystroke path

The acute fix. Highest value per unit of risk in this epic — do not let it queue behind
the chronic work.

Four changes, in `src/sase/ace/tui/`:

1. **Fixed panes must not need discovery at all.** `accent_and_icon_for_ref`
   (`relations/link_subject.py:38-58`) calls `descriptor_for_artifacts_pane_id` for
   _any_ non-`chop` target. But the overwhelmingly common case — the Agents tab, via
   `_subject_from_agents` at line 120 — always passes `pane_id="agents"`, one of the
   five **fixed** panes whose accent and icon are static entries in `ARTIFACTS_ACCENTS`
   / `ARTIFACTS_ICONS` (`_artifact_tab_model.py`). Resolve fixed pane ids straight from
   those tables and skip `resolve_artifacts_subtabs()` entirely. Only a genuine
   document-provider pane id should reach discovery.
2. **Memoize `fixed_descriptor`.** `_artifact_tab_descriptors.py:64` recompiles a
   builtin contract on every call, and `resolve_artifacts_subtabs` calls it five times
   per rebuild (visible in the captured stack). It is a pure function of its `subtab`
   argument — cache it.
3. **`resolve_artifacts_subtabs()` must never do synchronous discovery on a UI path.**
   Convert it to serve the last-known descriptors immediately and revalidate in the
   background on token change, the way `current_config_token()` already does
   (`config/core.py`) — `tui_perf.md` rule 10. Raise
   `_PROVIDER_SOURCE_TOKEN_REFRESH_INTERVAL_SECONDS` (`_artifact_tab_discovery.py:218`)
   from 0.75 s to ~30 s and refresh it off-thread on expiry rather than synchronously.
   Keep the existing "a `None` token is uncacheable" rule; it is there so a transient
   discovery failure cannot pin a degraded four-tab answer. First paint still needs a
   real answer, so keep exactly one synchronous resolve at startup, before the startup
   stopwatch ends (rule 9), and serve stale from then on.
4. **`_link_follow_available()` must be O(1).** `link_follow.py:99` resolves a full
   subject just to answer "is `$` armed-able?". Back it with a cached boolean recomputed
   on selection change or link-index generation change, not per key event.

Re-sweep the three `_app_watchers.py` call sites and `commands/context.py:275` against
`tui_perf.md` rules 1, 8, and 11 when done.

**Done when** the `baseline` behavioural bench passes: driving `j`/`k`/arbitrary keys
through the test harness reaches no discovery entry point, no subprocess, and no
project-tree walk.

## Phase `railcalls`: Collapse the redundant link-subject resolutions per keystroke

`keypath` makes each resolution cheap. This makes them rare. One `j` currently resolves
the subject at least three times (key handler, `watch_current_idx`, action
availability), and `LinkRail.refresh_from_app` (`widgets/link_rail.py:69-83`) resolves
it again on every invocation.

1. Cache the resolved `LinkSubject` on the app, keyed by
   `(current_tab, selected entity identity, _link_index_generation)`. Invalidate on
   selection change, tab change, and index refresh — not on a timer.
2. Have `link_edges_for_selection`, `refresh_link_rail`, `_link_follow_available`, and
   `check_action` all read that one cached value.
3. Coalesce `refresh_link_rail()` so repeated calls within a tick collapse to one
   repaint. `LinkRail._refresh` already has a `_last_signature` short-circuit; the waste
   is upstream of it.
4. Preserve the existing coalescing guards in `_schedule_link_index_refresh`
   (`actions/link_subject.py:70-109`) — `_link_index_loading` / `_link_index_pending` /
   `_link_index_generation` — and release them if spawning fails (`tui_perf.md` rule 2).

Re-capture UI state after every await (rule 4): the selection can move while an index
refresh is in flight.

## Phase `delta`: Make the artifact delta the default refresh

This is the phase that attacks **frequency**. `load_kind` across 10,482 logged slow
loads is 96% `full` and 2% `artifact_delta`. The bounded path is already built and is
being poisoned by its own fallback policy.

In `actions/event_refresh/_artifact_delta.py`:

1. **Fix the classifier's default.** `_agent_artifact_delta_dir_for_path` (lines 32-58)
   returns `(None, True, False)` — "affects agents, but unmapped" — for _every_ path
   that is not under an `artifacts` directory, including the whole projects root. Any
   one such path sets `unmapped_agents_path`, and `_agent_artifact_delta_dirs_for_paths`
   (lines 83-84) then throws away the entire batch. With concurrent runners writing
   constantly, this fires more or less always. Classify these paths honestly instead of
   defaulting them to "unknown".
2. **Make the batch partial, not all-or-nothing.** Apply the delta for the dirs that
   _did_ map and schedule a narrow reconcile for the remainder, instead of discarding
   both.
3. **Raise or shard `AGENT_ARTIFACT_DELTA_QUEUE_LIMIT = 64`** (`_constants.py:20`).
   Overflow escalates to a full reload, which is strictly more expensive than the delta
   it replaced — the limit is inverted from its intent.
4. Keep `FULL_SANITY_REFRESH_SECONDS = 60.0` as the backstop. Do not "fix" this phase by
   lengthening `refresh_interval` or the sanity cadence: that trades freshness for speed
   on a tool whose job is watching live agents, and the report rightly calls it
   palliative.

**Done when** `load_kind` in `tui_agent_loads.jsonl` inverts toward majority
`artifact_delta`, and slow loads per _active hour_ drop against the `baseline` figure.
Per active hour, not per day — 08-27 was a 13-hour day and a raw daily count understates
it by ~40%.

## Phase `stepmeta`: Remove the per-workflow-step filesystem enrichment

The largest single Python cost in the loader: 333 ms, of which 255 ms is 1,532
`enrich_agent_from_meta` calls from `_build_workflow_agent_steps_for_record`
(`models/_loaders/_workflow_snapshot_loaders.py:271`, filesystem call at :397).

The comment at that call site states the constraint plainly: a step's `artifacts_dir`
points to a _different_ directory than the parent record's, so the index snapshot does
not carry the step's `agent_meta.json`. Three viable approaches — choose with
measurements, do not assume:

- **Defer.** The list needs only a few fields per step row. Enrich a step fully when its
  row is selected or its workflow is expanded, not on every load. Cheapest and most in
  keeping with `tui_perf.md` rule 7.
- **Index it.** Carry step meta in the parent record in `sase-core` so
  `enrich_agent_from_meta_wire` can serve it. Correct but costs a wire change and
  couples this phase to `sase-core`.
- **Batch it.** Keep the reads but collapse the 24,567 `lstat` + 7,024 `stat` syscalls
  per load into far fewer, e.g. one directory scan per artifacts dir keyed by mtime
  (`tui_perf.md` rule 8). `load_json_cached` already caches contents; the cost here is
  the stat traffic and Python overhead around it, which is exactly what degrades under
  contention.

While here, `_done_extra_files` (`_loaders/_done_loaders.py:178-200`) costs 123 ms in
232 calls, almost all of it `Path.resolve(strict=False)` → 1,647 `realpath` syscalls,
purely to deduplicate a short attachment list. Deduplicate on the unresolved string and
resolve lazily only when a path is actually opened.

## Phase `marshal`: Drop the double tree build in the PyO3 binding

`py_query_agent_artifact_index` (`sase-core`,
`crates/sase_core_py/src/lib.rs:2606-2640`) correctly releases the GIL for the SQL and
Rust decode, then **re-acquires it to build the object graph twice**:

```rust
let snapshot = py.allow_threads(|| { core_query_agent_artifact_index(...) })?;
let value = serde_json::to_value(&snapshot)?;   // GIL HELD — tree build #1
json_value_to_py(py, &value)                    // GIL HELD — tree build #2
```

Replace the intermediate `serde_json::Value` with a serde `Serializer` that emits Python
objects directly, so the snapshot is walked once instead of twice. This is a pure
internal change: the Python-visible payload shape must be byte-for-byte identical, which
makes it straightforwardly testable against the existing wire tests.

Measured today: the binding is 321 ms of an 771-851 ms load, returning 12.6 MB and
344,412 Python objects. Roughly half of that is the two tree builds.

Do this **before** `projection` deliberately: it is contained, needs no wire change, and
lands a real win while the riskier schema work is still being designed. Both phases
touch this function, so they are sequenced rather than parallel.

Note for calibration: `to_thread` does not make this concurrent. The `allow_threads`
region yields only ~1.5× effective parallelism at 8 threads, so ~70% of the work stays
GIL-held, and each additional concurrent load makes the _worst_ UI stall worse (100 ms →
426 ms measured at 1 → 8 threads). Reducing the work is the only lever; adding threads
is not.

## Phase `projection`: Project the heavy `record_json` leaves off the list path

`select_records` (`sase-core`, `crates/sase_core/src/agent_scan/index.rs:3134-3148`)
selects `record_json` for every row on every refresh and `serde_json::from_str`s each
one, even though the table already denormalizes the 45 scalar columns the list renders.

Per Correction 3 above, do **not** frame this as "drop the blob". Frame it as omitting
the four leaf fields that are 77.6% of the payload — `workflow_state.steps`,
`done.step_output`, `prompt_steps`, `agent_meta.linked_repos` — while keeping the
surrounding structure intact, so the Python loader keeps working on the fields it
actually renders.

1. Add a list-shaped projection to `select_records`, selected by the query wire, that
   returns the scalar columns plus a record with those four leaves omitted. Keep the
   `*_sig` columns: `refresh_stale_rows` needs them.
2. Hydrate the omitted leaves lazily, per artifact dir, when a row is expanded, zoomed,
   or otherwise needs the full record.
   `SELECT record_json FROM agent_artifacts WHERE artifact_dir = ?1` already exists
   elsewhere in the file — reuse it.
3. Apply the same projection to `refresh_stale_rows`. This was flagged on 2026-08-12 and
   is still open.
4. Replace the three `record_json LIKE` predicates in
   `repair_abandoned_agent_artifact_index_rows` (`index.rs:684`) with an indexed scalar
   column. They force a full scan of a 138 MB text column (~0.15 s measured) to match
   ~36 rows.

Then update the Python consumers in this repo so an omitted leaf is a well-defined "not
loaded yet" state rather than an empty value that silently renders as "no output". That
distinction is the main correctness risk in this phase — a projected row that looks
identical to a genuinely empty one will produce wrong output in the detail panel, so
make it explicit in the type, not by convention.

This crosses the Rust core boundary, so the wire, bindings, and tests change in
`sase-core` and the callers change here. Open the linked repo with `/sase_repo`.

**Do not expect ~95% off the stage.** The payload drops ~78%, the Rust binding is only
~40% of the stage, and the per-agent Python and filesystem work is untouched by this
phase. Budget ~30-40% off the loader, and verify rather than assume.

## Phase `viewport`: Honour the `AgentsViewport` contract

The 36:1 waste ratio. `AgentsViewport` (`data_providers/_types.py:97-107`) defines
`visible_rows=40`, `prefetch_rows=80` → `requested_limit=120`. But
`make_agents_data_provider()` (`_factory.py:59-61`) always returns
`DirectAgentsDataProvider`, whose `load_agents` opens with `del search_query, viewport`
and closes with `full_reload=True` (`_direct.py:16-47`). The bounded read window is
designed, typed, and discarded on every call.

1. Wire the viewport and search query through `DirectAgentsDataProvider` instead of
   deleting them.
2. Push search/filter/order into the Rust query so the window is chosen before records
   are materialized, not after.
3. Return the requested window plus prefetch rather than the full recent set
   (`active_limit=1000`, `recent_completed_limit=200` today →
   `_agent_loader_artifacts.py:27-28`).
4. Fetch detail, logs, and relations only for the selected row.

This is also the only phase that reduces the per-agent filesystem enrichments in
proportion to what is on screen.

**This phase is measurement-gated.** Re-measure against the `baseline` numbers first. If
`projection`, `delta`, `stepmeta`, `marshal`, `keypath`, and `railcalls` together
already meet the budgets, say so on the phase bead and close the phase as unnecessary
rather than doing large risky work for its own sake. It is in the plan because the
earlier phases make each load cheaper and rarer without stopping ACE from building 430
agents to show 12 — but if the user can no longer feel that, it is not worth the risk.

## Phase `hygiene`: Retention tooling and self-inflicted stalls

Independent of every other phase; cheap; do any time.

1. **Retention.** The index carries 38,243 `dismissed_agents` rows and 4,838 freelist
   pages (~19.8 MB, ~10% of the file) that are never `VACUUM`ed;
   `agent_name_registry.json` is ~17 MB / 13,118 entries and `dismissed_agents.json` is
   ~2.4 MB. Add retention policy and a vacuum/compact path. **Ship the tooling; do not
   run destructive lifecycle mutations against the user's live `~/.sase` state.** Adding
   a command the user can run is in scope; pruning their data is not.
2. **Read-only index opens.** `open_index` (`sase-core`, `index.rs:1961-2265`) opens
   `READ_WRITE|CREATE`, replays 25 `CREATE TABLE/INDEX IF NOT EXISTS` statements, and
   unconditionally `INSERT OR REPLACE`s the `schema_version` row on **every** open,
   including logically read-only queries. Give query paths a read-only open and drop the
   unconditional write. Correctness and multi-writer hygiene — the measured cost is ~14
   ms, so do not claim this as a speed fix.
3. **Stop the watchdog extending the freeze it measures.**
   `util/_stall_watchdog_records.py:150-179` dispatches `_write_pump_stall_record`
   through `call_soon_threadsafe`, so stack capture and JSONL serialization run **on the
   event loop**. One such record was 2.29 MB on a single line. Write it from a worker
   thread and cap `asyncio_task_stacks`.

## Out of scope

- **A daemon-backed read-model service.** Deliberately deferred; see "What the evidence
  does not support".
- **Widget/DOM lazy mounting.** Rests on n=1 evidence. Needs a dedicated soak before
  anyone plans against it.
- **`tui_perf.md` rule 1 amendment.** The report proposes adding a GIL corollary to rule
  1 — that for CPU-bound Python and PyO3 marshalling, `to_thread` moves work off the
  loop but only partially off the GIL, and additional concurrent loads make the worst UI
  stall worse. The measurements support it and it would be genuinely useful guidance.
  **It requires the user's explicit approval before any memory file is edited** —
  flagging it here, not doing it. The `land` agent should raise it.
- **Two unrelated bugs the reports surfaced**, which belong in task beads rather than
  this epic: 14 ×
  `WorkerFailed: RuntimeError('asyncio.run() cannot be called from a running event loop')`
  in `tui.log`, and 90+ `git rebase failed` warnings on workspace plans clones.

## Verification

Every phase runs `just check` before handing off. The combined tree runs
`just check-full` via `/sase_monitor` before landing — it routinely outruns a single
agent turn.

Beyond the test suite, confirm against the instrumentation that already exists:

- `tui_startup.jsonl` — `agents_ready` p50 back under 4.9 s, **split by concurrent ACE
  instances**, so a quiet hour is not mistaken for a fix.
- `tui_agent_loads.jsonl` — slow loads per active hour, and `load_kind` inverting from
  96% `full` toward majority `artifact_delta`.
- `tui_stalls.jsonl` — no main-thread stack reaching provider discovery or a subprocess;
  hitch inter-arrival decoupled from the refresh tick.
- `SASE_TUI_PERF=1` → `~/.sase/perf/tui_jk.jsonl`, p95 < 16 ms on every tab.
