---
tier: epic
title: Reduce measured TUI GC producers and verify both hosts
goal:
  Reduce graph and render churn and retained UI caches, preserve quiet-time memory
  backstops, and establish the GC and navigation targets on restarted Athena and Apollo
  TUIs.
parent_bead: sase-1nf.14
phases:
  - id: report-windows
    title: Make GC acceptance calculations trustworthy
    depends_on: []
    size: medium
    description:
      "report-windows: repair percentile, heartbeat coverage, boundary, swap, and
      late-hitch accounting; missing evidence must never pass."
  - id: interaction-evidence
    title: Attribute navigation and prompt activity to an instance
    depends_on:
      - report-windows
    size: medium
    description:
      "interaction-evidence: add attributable perf and prompt-state evidence, consume
      actual next/prev samples in acceptance, and document capture preflight."
  - id: graph-allocation
    title: Avoid copying Agent graphs that a load will discard
    depends_on: []
    size: medium
    description:
      "graph-allocation: classify cached-row use before detaching graphs, copy only
      required ownership closures, and preserve worker isolation and aliases."
  - id: render-allocation
    title: Skip unchanged row rendering and Option replacement
    depends_on: []
    size: medium
    description:
      "render-allocation: remove unconditional AgentList row invalidation when complete
      render inputs are unchanged, preserve visual updates, and measure Strip churn."
  - id: snapshot-retention
    title: Bound retained event and completion cache ownership
    depends_on: []
    size: medium
    description:
      "snapshot-retention: bound per-agent and context event caches and completion index
      ownership, preserving result semantics and invalidation."
  - id: quiet-policy
    title: Resolve the backstop quiet-window conflict
    depends_on:
      - interaction-evidence
      - graph-allocation
      - render-allocation
      - snapshot-retention
    size: small
    description:
      "quiet-policy: require three quiet seconds for backstops, retain their due
      conditions and activity gates, and consistently attribute census collections."
  - id: fixed-capture
    title: Verify busy-hour and four-hour acceptance on fixed code
    depends_on:
      - quiet-policy
    size: medium
    description:
      "fixed-capture: verify installed revisions and user restarts, preflight real
      navigation, capture representative interaction and four-hour coverage on both
      hosts, and publish results."
  - id: fixed-soak
    title: Complete the same-instance 24-hour soak
    depends_on:
      - fixed-capture
    size: small
    description:
      "fixed-soak: continue the retained fixed-code instances through 24 post-warmup
      hours, verify coverage and targets, and publish the final acceptance matrix."
proposed_by: bbugyi200.athena.sase-1nf.14
proposed_with: codex/gpt-6-astra
create_time: 2026-10-24 13:37:41
status: wip
---

- **PROMPT:**
  [prompts/202610/measured_tui_gc_fixes.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/measured_tui_gc_fixes.md)
- **PARENT:**
  [202610/athena_apollo_audit_fixes.md](https://github.com/sase-org/sase--plans/blob/main/202610/athena_apollo_audit_fixes.md)

# Plan: Reduce measured TUI GC producers and verify both hosts

## Scope and evidence

This is the implementation child epic for `sase-1nf.14`, whose governing design is
`plan:202610/athena_apollo_audit_fixes.md`, Track 4. Its final `gc_soak = no` decision
means this work owns the 24-hour soak. It does not waive the soak. No new reviewer
decision or memory edit is needed.

Read `research:202610/tui_gc_measurement_20261023.md` through `sase artifact read`
before implementing. The earlier phase completed measurement, not acceptance:

| R5 result                       | Athena / Python 3.14.7 | Apollo / Python 3.12.3 |
| ------------------------------- | ---------------------- | ---------------------- |
| Gap-free post-warmup span       | 14,452.309 s           | 14,405.583 s           |
| Idle full-GC p95                | 2.761 s                | 0.184 s                |
| GC share                        | 5.709%                 | 0.664%                 |
| Loop hitch share                | 3.583%                 | 0%                     |
| RSS endpoint growth             | 694,468,608 B          | 178,610,176 B          |
| Maximum sampled process swap    | 621,490,176 B          | 0 B                    |
| Actual Agents next/prev samples | 1                      | 0                      |

There is no completed representative busy hour, other-tab distribution, prompt-active
comparison, or 24-hour acceptance. Historical r4 navigation is a baseline only: Athena
Agents p95 215.26 ms with 52 samples. Apollo's busiest historical hour has 12 actual
next/prev actions. Do not pool these with r5 or with fixed-code runs.

The producer experiments found approximately 800 KB retained until cyclic GC per 100-row
copied Agent graph, versus immediate release in the acyclic control. Full-width Textual
Strip divide/crop operations create self-cycles retaining seven FIFOCaches per Strip.
Large event/archive/completion populations also survive, but their isolated converters
are acyclic. These establish mechanisms and priorities, not percentages of live
allocation traffic or proof that every population leaks. LinkIndex already uses native
shared handles; another Python-to-Rust migration of that index is not part of this plan.

Current source supplies concrete bounded changes:

- `prepare_loaded_agents_worker_boundary()` detaches the cached, capacity, and fleet
  rosters before the merge decides whether it will use the cached history.
- `patch_row()` invalidates the agent render cache and replaces the Option prompt
  unconditionally, even though the render key already describes its inputs.
- Event snapshot caches have a project limit, but the per-agent and context caches in
  the read/event loaders are unbounded dictionaries. Completion's
  `_ARTIFACT_INDEX_CACHE` is also an unbounded dictionary keyed by path.
- `BACKSTOP_QUIET_S = 1.0` contradicts the two-second input exclusion requirement.
- The report computes idle p95 from unsorted samples, counts the first heartbeat's
  preceding interval, checks only final swap, and allows missing evidence to pass.

This needs separate coding phases and operational handoffs, rather than a single coder
responsible for all changes and a day-long capture. The prior note on `sase-1nf.14` is a
measurement handoff, not evidence of an unfinished coding attempt.

## Common implementation rules

1. Read `ace.md`, `tui.md`, and `tui_perf.md` through the memory skill before TUI work;
   read `tools.md`, `cli_rules.md`, and `docs_writing.md` where applicable. Do not
   modify any memory note. Keep notes about discovered work on the assigned phase as
   `PROPOSED FOLLOW-UP:`; do not create task beads.
2. Keep these changes in TUI presentation, cache ownership, and diagnostic tooling. Do
   not change event attribution, persisted history semantics, query semantics, or graph
   domain rules in Python. If a necessary shared-domain change emerges, open `sase-core`
   with `/sase_repo`, implement it there with bindings and tests, and advance the sase
   pin and workspace binding per the Rust backend contract.
3. Preserve pump-free worker scheduling, generation rejection, bounded startup,
   last-request-wins refreshes, immediate highlights, and detail debouncing. No new
   data-scaled work or disk I/O belongs on a keypress or render path.
4. Fix producers before policy. Do not disable automatic GC, remove time/RSS backstops,
   raise thresholds further, repeat `gc.freeze()`, or move collection to a worker as a
   purported pause fix. Do not edit installed Textual files or add a process-wide
   monkeypatch. The initial render fix is in SASE's update cadence.
5. Every behavior change gets a regression that fails on the old behavior. Preserve the
   existing graph-isolation and rendering assertions. Use scoped allocation comparisons,
   object lifetime/call-count checks, and existing perf harnesses; avoid noisy
   wall-clock thresholds in ordinary unit tests.
6. Run `sase tool run check` in each changed repository after reading
   `lint_and_test.md`. Use `/sase_monitor` for known-long setup, benchmarks, and
   verification; do not run `check-full`. Use the existing benchmark entry points under
   `tests/ace/tui/bench_tui_jk.py` for relevant scenarios. If rendered output changes,
   follow the screenshot memory and targeted visual verification rules.
7. Record source/native dependency revisions, Python/Textual/Rich versions, fixture
   sizes, and before/after counts with each producer result. Reuse the prior producer
   methodology for changed paths; do not repeat the entire old census investigation.
   Publish evidence with `sase artifact create` and attach its ref to the phase. Record
   failures identical on a clean base as proposed follow-ups with the existing tracking
   bead, if any; they do not prevent phase closure.
8. Phase workers close only their assigned phase, after `sase bead epic-symbols` reports
   no leftovers (resolve or re-key them to an open owner). The child epic's land agent
   owns completion of `sase-1nf.14`; no worker may close `sase-1nf`, `sase-1nc`,
   `sase-1f3`, or any other ancestor/task. The older instruction to close `sase-1nc` is
   evidence preparation for the outer land agent, not authorization.

## Phase report-windows

Change `tools/tui_freeze_report` and `tests/test_tui_freeze_report_tool.py`. Split
helpers/tests by responsibility if needed instead of extending an oversized file.

- Sort full-generation idle samples before all percentile calculations and use one
  documented quantile convention. Preserve trigger/generation breakdowns. Census spans
  and collector time overlap: never add their durations together.
- Build coverage from adjacent heartbeat endpoints for one instance, after its own
  20-minute startup warmup. Sum the later endpoint's interval counters only; the first
  endpoint's counters belong outside the window. Validate `window_s` against adjacent
  wall times with the existing measurement's two-second tolerance. Missing intervals,
  malformed required fields, resets, and insufficient startup provenance produce
  explicit incomplete evidence, not implicit zeros.
- Preserve raw epoch boundaries end-to-end. A requested envelope is not observed
  coverage. For ISO limits rounded to microseconds, apply a documented tolerance bounded
  by that precision; never use a tolerance wide enough to bridge a missing heartbeat.
  Report both requested and actual endpoints/span.
- Report peak and endpoint RSS and swap, including the swap trajectory. Evaluate RSS on
  a contiguous four-hour span. Use the bead's decimal 0.5 GB threshold (500,000,000
  bytes), labeling units explicitly. Keep the current conservative zero-byte swap
  criterion and evaluate all samples, not just the final one.
- Check late hitches explicitly. Reconstruct/clamp episodes with boundary context and
  compare collection start/end intervals. An overlap with any idle pause does not excuse
  a mixed or partly unexplained late hitch. Only positively attributable idle-collection
  episodes qualify for the exception; insufficient attribution remains unknown. Report
  suppressed hitch totals separately without double-counting.
- Use per-target PASS / FAIL / INCOMPLETE. An empty selection, missing required
  evidence, missing j/k, missing input age for relevant collections, or short RSS
  coverage cannot yield overall PASS or an acceptance exit code of zero. Diagnostic
  non-acceptance mode remains usable with old logs. Keep failed targets visible even
  when another target is incomplete.
- Add `-j/--json` for reproducible structured summaries with schema version, target
  values, counts, provenance, coverage diagnostics, and overall status. Keep public
  options sorted and document exit behavior; retain existing flags.

Regression fixtures must cover shuffled idle durations; first-heartbeat exclusion; gaps;
paired boundaries and rounded ISO endpoints; swap that peaks then falls; late idle,
non-idle, and mixed episodes; empty/short/legacy windows; and no j/k. Replay the
retained r5 corrected summaries where practical using the report's audited evidence
refs. Small checked-in fixtures should reproduce the defects without copying the full
operational archive into the source tree.

## Phase interaction-evidence

Extend `util/perf.py`, existing TUI telemetry/activity hooks, the report, and their
focused tests. This phase owns the perf-input and workload parts of the reporter; it
follows report-windows to avoid concurrent edits to its contract.

- Add `app_instance_id`, PID, a wall-clock timestamp paired with monotonic action time,
  and the existing source/runtime provenance to perf evidence as appropriate. Do not
  rediscover revision/config/provider data on every key. Preserve the existing perf
  fields and bounded in-memory sample window. Count overwritten in-flight samples so a
  sparse distribution is not mistaken for all input.
- Emit an initial activity state and state transitions for prompt-active and modal
  text-entry activity, with instance/time attribution. Use existing UI lifecycle hooks
  and the telemetry writer; never log prompt contents. Reconstruct durations from state
  transitions, not input age or isolated typing actions. Missing starting state or
  transition coverage is explicitly unknown. Instrument only when perf capture is
  enabled, with bounded storage/rotation.
- Add `-p/--perf-path` to the report, with the existing environment/default path
  convention and rotation support. Only `next`/`prev` and verified corresponding
  artifact-pane action names contribute to j/k. Report leader, prompt, and panel actions
  separately. Group latency by instance and tab; report n, p50, p95, max, dropped
  samples, and ten-minute activity bins.
- New rows are joined by instance. Legacy rows need explicit one-process and
  wall/monotonic witnesses as in r5; the generic CLI must not silently attribute them.
  Supply an incomplete diagnosis when attribution cannot be proved.
- Keep the busy-hour requirement distinct from four-hour RSS coverage. Report candidate
  hourly windows and their activity. Add `-b/--busy-since` and `-e/--busy-until` to
  select the observed hour inside the covered interval, plus repeatable `-t/--tab` to
  declare the required navigation tabs (Agents always required). Validate its actual
  duration and containment. The operational report identifies why this normal-use
  workload is representative; the CLI checks its measurable evidence. A quiet window is
  not a busy hour. Require at least 100 measured Agents next/prev actions across at
  least four ten-minute bins as a sampling floor, not proof of representativeness by
  itself. Other normally used tabs need at least 30 attributable samples each and
  separate distributions. An omitted busy window or insufficient declared-tab samples
  remains INCOMPLETE.
- Report prompt-active versus idle durations and GC/hitch observations; mixed heartbeat
  intervals cannot be assigned wholesale to either state. Keep the j/k target INCOMPLETE
  until an attributable representative distribution exists.

Tests cover two simultaneous instances, restart/PID reuse, legacy unassignable rows,
action filtering, per-tab quantiles, overlapping input samples, and prompt
open/close/modal intervals. Preserve the GC callback's lock-free/I/O-free structural
test. Update `docs/contribute/perf-runbook.md` with the new evidence and preflight,
after reading the docs routing contract. This is the only phase editing that page.

## Phase graph-allocation

Focus on `actions/agents/_loading_graph.py`, `_loading_compute.py`,
`_loading_compute_merge.py`, and the existing copy/isolation tests.

1. Separate read-only classification of cached-row requirements from mutation. Determine
   whether a load is authoritative replacement, an artifact delta, same-query bounded
   prefix, or a merge over reconciled history before copying the cached graph. Preserve
   all current load-state/query predicates.
2. Detach only the cached/capacity roots actually needed by that path and their
   reachable mutation dependencies. An authoritative load with an available proc
   projection should not copy discarded cached history. Preserve fallback cached
   named-proc rows when `proc_projection is None`, required capacity history, and fleet
   rows that projection mutates. An empty proc projection is authoritative.
3. Use one memo across required roots so overlapping visible/capacity/fleet rows have
   one detached clone. Loader-owned new rows remain owned without another copy. Never
   pass live mutable rows into normalization, slot annotation, folding, or relationship
   rebuild. Do not replace deep ownership with shallow copies or clear old graph links
   while widgets/workers can still hold them.
4. Keep standalone `copy_agent_graph()` semantics, including child-only roots, cycles,
   mutable leaves, alias preservation, and deferred fields. This bounded phase avoids a
   weak-reference/domain graph redesign.

Extend `test_agent_graph_copy.py`, `test_agents_tab_graph_isolation.py`, and their
helpers: zero unused history clones on authoritative replacement; unchanged source rows
during worker processing; one clone per shared reachable row; correct
delta/bounded/query-change/capacity/named-proc/fleet behavior; stale worker rejection.
Compare old/new repeated real worker preparation at 100 and 1,000 rows, including cyclic
sessions and partial updates. Record clone counts, traced peak/retained bytes, and
objects released after dropping the result before/after explicit GC. Do not claim all
graph garbage was removed merely because unused copies disappear.

## Phase render-allocation

Focus on `_agent_list_build_patch_single.py`, the row render-key/cache modules, and the
incremental AgentList update paths.

- Establish the full render input signature at the patch boundary. Audit every formerly
  force-invalidated input, including dynamic side-cache hints, wait counts,
  marks/unread, status, selection, machine/tab chips, clan wave state, width,
  theme/style, and ticking runtime. Add a version/input to the signature for mutable
  external state that otherwise would escape the key.
- When that signature and layout are unchanged, preserve the current Option and Textual
  render caches without calling `replace_option_prompt_at_index`. Do not
  force-invalidate a valid content-keyed render entry. Update bookkeeping needed for
  future deltas even when no repaint is required.
- A changed signature still renders immediately through the existing patch path. Keep
  width-growth/full-rebuild fallbacks, runtime suffix optimization, stable option
  identity, programmatic-highlight suppression, and selective row updates. Cache new
  signatures per widget and prune them when rows disappear.
- Check callers for unchanged roster refreshes that unnecessarily rebuild whole panels.
  Fix only demonstrated bypasses of the established incremental path; preserve existing
  fleet projection signature/repaint skipping.

Regressions assert unchanged repeated patches cause zero option replacements and changed
visible inputs still repaint. Exercise selection, runtime, theme/resize, marks/unread,
waiting, and fleet changes. Compare repeated unchanged versus changed refreshes with a
real Textual harness, recording render/Strip construction counts and cyclic garbage
after release. Preserve output parity. Do not assert that the upstream Strip self-cycle
itself has been removed.

## Phase snapshot-retention

Bound presentation cache ownership in `memory_reads.py`, `artifact_reads.py`,
`skill_uses.py`, `glossary_reads.py`, `_bead_touches_loader.py`, and
`widgets/_artifact_ref_completion_catalog.py`.

- Convert per-agent and context memo tables to bounded LRU ownership, initially 128
  entries per table. Hits refresh recency; insertion evicts the oldest owner. Preserve
  per-entry result limits, context identity keys, labels, ordering, deduplication,
  mtime/size invalidation, throttling, and thread synchronization. Existing
  eight-project snapshot bounds remain; they are not evidence that the derived tables
  are bounded. Eviction changes caching, never visible history.
- Bound the path-keyed artifact index cache to eight sources. Revalidate current entries
  as before and release evicted entries, including obsolete source roots. Check catalog
  replacement/teardown does not retain superseded catalogs through worker result or
  widget fields. Fix demonstrated redundant ownership only.
- Use the known large archive/event/completion populations as verification inputs, not a
  license to truncate persisted records or redesign the search backend. Already
  bounded/native LinkIndex ownership remains outside this phase.

Add focused eviction/reload/invalidation tests for each affected family and context
aggregation. Cover a selected entry evicted then revisited, changing source generations,
project switches, and worker completion races. Demonstrate bounded cache cardinality and
release of evicted values after navigating many distinct agents, plus equivalent results
before/after eviction. Measure retained objects/bytes with warmed event and completion
fixtures; record populations that remain legitimately owned without labeling them leaks.

## Phase quiet-policy

After producer results land, change `util/gc_policy.py` and its focused tests:

- Set backstop quiet time to three seconds, matching the existing idle quiet interval.
  Both time and RSS backstops must remain ineligible at ages 1 s, 2 s, and just over 2
  s; they may become eligible at 3 s when all other gates pass.
- Preserve the five-minute and RSS-growth due paths, minimum interval, loading,
  navigation, prompt/modal gates, startup's once-only freeze, allocator trimming,
  disable switch, and threshold restoration. Test that sustained activity defers a due
  collection and the first eligible quiet interval still runs it.
- Keep all collections belonging to an opt-in census under explicit census/idle
  attribution, including its cleanup collection. Preserve restoration of debug flags and
  safe cleanup on exceptions. Expose census overhead separately so the reporter can
  diagnose it without hiding time from the acceptance totals.
- Do not infer collector behavior solely from the runtime's version or change the
  existing threshold capability policy as part of this quiet-window repair. Verify the
  supported Python 3.12 and 3.14 paths with available fixtures/runtime checks and carry
  exact runtime versions into live acceptance.

Re-run policy and telemetry structural tests and summarize producer improvements and the
policy change for the capture owner. Local benches are readiness evidence, not a
substitute for the live targets.

## Phase fixed-capture

This is operational verification of landed code, with research artifacts as the
deliverable. Read `/sase_repo`, `/sase_questions`, `/sase_gate`, and `/sase_monitor`
before their first use. Open the research sidecar with the repo skill; read prior
evidence through `sase artifact read`. Use `ssh apollo 'zsh -lic "..."'` for the remote
read-only probes after reading `tailnet.md`.

1. Record installed source revision, loaded binding/package versions, Python,
   Textual/Rich, boot/PID/start ticks, and capture paths on both hosts. If fixed code is
   not installed, prepare the exact host update through `/sase_gate`. Global
   installation is human-only. Only the user restarts their TUI: request it through
   `/sase_questions`, explaining the governing epic's explicit rule.
2. Request launch from a shell with `SASE_TUI_PERF=1` and `SASE_TUI_GC_CENSUS=1`.
   Confirm both flags and the fixed source revision in the actual replacement processes.
   Every restart defines a new instance even if exec reuses the PID. Keep one witnessed
   TUI per host for capture.
3. Before a long wait, ask the user to exercise actual j/k on each host, use the usual
   tabs, and open/close the prompt. Verify new attributable samples and state rows
   immediately. If they used the TUI but rows are missing, diagnose capture first. Do
   not spend another four-hour timeout on missing instrumentation.
4. Agree a normal representative workload/hour with the user. Preserve roster, archive,
   fleet, and host-load context. Do not inject commands into their TUI or manufacture a
   passing workload. Synthetic pilot measurements are separately labeled. Both hosts
   need actual interaction, including other normally used tabs and prompt-active/idle
   intervals.
5. After each instance's 20-minute warmup, retain raw active/rotated telemetry, startup
   and perf rows plus process/clock witnesses through `/sase_monitor`. Capture a busy
   hour and a gap-free four-hour interval on the same instance per host. Retain enough
   boundary rows for paired accounting and start/end events. Preserve files outside
   rotating logs so they can support the subsequent soak.
6. Run the corrected `tools/tui_freeze_report --acceptance` with explicit instance, raw
   boundaries, perf input, selected busy hour, and required tabs; retain text, JSON,
   exit status, and coverage diagnostics. Cross-check full-collection counts against
   heartbeat totals. Do not subtract census collectors to turn a failing headline metric
   into a pass; report census-free diagnostic figures separately.
7. Publish a research report with raw refs and a per-host before/after matrix for every
   target below, counts, actual coverage, workload, prompt comparison, and remaining
   misses. Clearly mark old/new workload differences; do not attribute the host
   difference to Python alone. Preserve the complete r5 baseline.

| Required evidence                                | Acceptance                                                   |
| ------------------------------------------------ | ------------------------------------------------------------ |
| Automatic/backstop/RSS-backstop full GC at input | Zero starts within 2 seconds                                 |
| Idle full-collection p95                         | At most 500 ms                                               |
| All GC share                                     | Below 3%                                                     |
| Loop hitch share                                 | Below 0.5%                                                   |
| Late hitches                                     | None except positively attributable idle-collection episodes |
| RSS endpoint growth                              | Below 500,000,000 bytes over four observed hours             |
| VmSwap                                           | Zero across sampled window; report maximum/trajectory        |
| Actual Agents j/k                                | Representative distribution, p95 below 16 ms                 |
| Other normally used tabs                         | Separate distributions, p95 below 16 ms                      |
| Prompt activity                                  | Attributed active/idle duration and pause comparison         |

If a target misses, record `PROPOSED FOLLOW-UP:` with the failed metric, source
revision, evidence ref, and suspected mechanism only where supported. A complete
measurement can close this operational phase with an honest FAIL matrix; that does not
authorize closing `sase-1nf.14` as fixed. Do not leave the operational phase silently
open. For a missing workload/restart, hand off through questions; for elapsed windows,
hand off through monitor. A valid handoff preserves ownership.

## Phase fixed-soak

Continue the same fixed-code instance per host from fixed-capture until each has at
least 24 hours of observed post-warmup coverage. Reuse its capture state and files; do
not restart simply to begin this phase. Use `/sase_monitor` for the remaining duration
and retain the full interval through log rotation.

If a process restarts, keep the interrupted segment as evidence, report the
interruption, and establish a new complete interval after user-confirmed preflight.
Never pool instances. If a correctness fix is needed, the changed revision needs a fresh
warmup/capture; do not splice pre-fix and post-fix observations.

Verify coverage, hourly latency/GC/hitch summaries, full-window pause quantiles, swap
history, and four-hour RSS windows including the final four hours. Report long-term
peaks/growth as well as endpoints and any additional busy-hour samples. Carry forward
the fixed-capture workload and prompt evidence only for the same instances. Record
interruptions, unknown evidence, and misses as such. Publish the final report and refs,
then close this measurement phase when its complete honest matrix exists. Missing
duration requires a monitor continuation, not a completion claim.

## Child epic landing and completion of sase-1nf.14

The land agent checks all producer changes, policy regressions, check results, raw
evidence, and the two-host acceptance matrix. It must not confuse completed measurement
phases with successful performance acceptance.

If bounded implementation defects remain, fix and verify them. If substantive target
misses remain beyond one coder, record concrete proposed follow-ups and submit a child
epic for the measured remainder through `/sase_plan`, retaining ownership of
`sase-1nf.14`; do not merely leave it open with a promise to resume. Do not weaken
thresholds, call missing data a pass, or mark this phase fixed on the strength of
producer microbenchmarks alone.

Once the fixes and required busy-hour, four-hour, per-tab/prompt, and same-instance
24-hour evidence meet the contract on both hosts, add concise closure evidence to
`sase-1nf.14`, including source revisions and report refs. Run
`sase bead epic-symbols sase-1nf.14`, resolve or re-key any remaining entries, and close
only `sase-1nf.14` with a note stating what passed. The child's own phases and plan
settle through their normal host lifecycle. Leave `sase-1nf` and all other ancestors
open. Leave `sase-1nc` and `sase-1f3` for the outer epic's land agent, with their
acceptance evidence linked from this phase.
