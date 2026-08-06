---
tier: epic
title: sase-fp landing — a budget guard that measures the set, and a false-negative
  metric that measures selection
goal: The contract-set budget guard sase-fp introduced stops failing on host load
  and starts measuring what it claims to measure, the false-negative metric that is
  sase-fp's own exit criterion stops charging one workspace's flakes to another workspace's
  selection, and sase-fp lands with an honest reading of both.
phases:
- id: budget
  title: Load- and machine-normalized contract-set budget guard
  depends_on: []
  size: medium
  description: 'budget: replace the wall-clock ceiling in test_contract_set_serial_runtime_stays_within_budget
    with a calibration-probe-normalized CPU measurement, so the guard bounds the contract
    set''s size instead of the host''s load, and settle sase-fp.2''s deferred contract-set
    membership question with the re-measured headroom.'
- id: correlate
  title: Change-scoped false-negative correlation
  depends_on: []
  size: medium
  description: 'correlate: record the workspace and change-set identity that the false-negative
    definition requires on full-run health records, restrict find_false_negatives
    to pairs that describe the same change, and re-read the metric.'
- id: land
  title: Land epic sase-fp
  depends_on:
  - budget
  - correlate
  size: small
  description: 'land: file the collected follow-ups with /sase_new_task, close sase-fp
    with a note covering the verification and integration findings, run just symvision
    after the close, and mark plans:202608/test_suite_tier1.md done.'
proposed_by: bbugyi200.athena.sase-fp.land
parent_bead: sase-fp
create_time: 2026-08-06 01:41:36
status: wip
bead_id: sase-fp.8
---

- **PROMPT:** [prompts/202608/test_selection_landing.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/test_selection_landing.md)
- **PARENT:** [202608/test_suite_tier1.md](https://github.com/sase-org/sase--plans/blob/main/202608/test_suite_tier1.md)
- **BEAD:** [sase-fp.8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-fp/sase-fp.8.md)

# Plan: sase-fp landing — fix the two defects the epic left behind, then close it

## Why this plan exists

This is the landing plan for epic `sase-fp` (`plans:202608/test_suite_tier1.md`). All seven phases are closed and, on
review, delivered: the selection engine, contract set, scoped runner, `just check` / `just check-full` split, health
store, coverage contexts, and the memory policy edit all exist and match their bead notes.

Two defects **caused by the epic** remain open, and both undermine claims the epic itself makes. They are epic work, not
follow-ups, so they are fixed here before `sase-fp` closes.

Everything below was measured at `d66101e8f` on the 64-core development host with this workspace's own `.venv`.

## What verification and integration already established (do not redo)

- **Deliverables exist.** `tests/_test_selection.py`, `_test_selection_graph.py`, `_test_selection_contexts.py`,
  `_test_selection_health.py`, `_test_selection_health_plugin.py`, `_test_selection_fixtures.py`, `tools/select_tests`,
  `tools/selection_health`, `tools/refresh_contract_manifest`, `tools/fetch_coverage_contexts`, and the `test-scoped` /
  `test-contexts` / `refresh-contract-manifest` / `refresh-contexts-baseline` / `selection-health` recipes are all
  present.
- **`check` and `check-full` share an identical eleven-gate list** and differ only in the final stage
  (`just test-scoped` vs `just test`), which is what `tests/test_justfile_lint.py` pins.
- **The contract manifest is in sync**: `test_contract_manifest_matches_marker_selection` passes at HEAD.
- **Integration with post-epic commits is already complete.** `3e4b5955c` (the `test_run_pytest_tool.py` split into five
  modules) regenerated `tests/contract_manifest.txt` itself; `245d7c44f` changed the shared `.github/actions/setup-sase`
  composite action, which the epic's new `coverage-contexts` job consumes; the bead close-history commits (`1da5a3e27` …
  `d7ac0dab5`) added only bead-behaviour tests reachable through the import graph, so no contract-set membership
  changes; `a4a2c1a60` fixed the `progress_fingerprint` symvision finding that the epic plan flagged as known-red, and
  `just _lint-symvision` is green at HEAD; the new job's `upload-artifact@v4` matches the repo's dominant pin. **No
  integration work is outstanding.**

---

## Phase `budget` — Load- and machine-normalized contract-set budget guard

### The defect

`tests/test_contract_manifest.py::test_contract_set_serial_runtime_stays_within_budget` asserts a 30 s **wall-clock**
ceiling on a nested serial `pytest` run of the 34-file contract set. The enclosing suite runs that assertion under 12–28
xdist workers, so the number it measures is host load at least as much as the contract set's size.

This is not a dev-host-only annoyance. It failed **real CI**: master run `31066038583` at commit `1da5a3e27` failed this
node on one of the three Python legs while the other two legs passed the same node in the same run. Locally it has
recurred at least ten times across `sase-fp.3`, `sase-fp.4`, `sase-fp.5`, `sase-fp.6`, `sase-fp.7`, `sase-fq.7`,
`sase-fr.3`, `sase-fr.4`, `sase-fr.5`, `sase-fr.6`, and `toobig-1l.split_file` — every one of which confirmed it passes
standalone. Two other land agents routed it here rather than filing a task, precisely because `sase-fp` introduced it.

`sase-fp.3` already proposed the shape of the fix ("measure CPU time instead of wall clock"). **CPU time alone is not
enough**, which is why this phase specifies more than that.

### Measurements taken while writing this plan

Contract set as committed: 34 files, 289 tests, run exactly as the guard runs it (nested `pytest -q`,
`_nested_pytest_env()`).

| Condition                                     |                      Wall |                 child CPU |
| --------------------------------------------- | ------------------------: | ------------------------: |
| Quiet host (loadavg 10–23)                    | 23.90 s, 24.21 s, 25.96 s | 23.71 s, 24.02 s, 24.36 s |
| 96 spinner processes on 64 cores (loadavg 75) |          48.05 s, 49.36 s |          35.80 s, 36.06 s |

Wall inflates **2.0×** under contention; CPU inflates **1.5×**. Against a 24 s baseline and a 30 s ceiling, neither
survives.

Now bracket the nested run with a fixed calibration probe (a deterministic pure-Python loop, some `sha256` hashing, and
20 trivial subprocess spawns — roughly 0.75 s of CPU on this host) measured the same way, immediately before and after:

| Condition   |  Probe child CPU | Contract CPU | Contract ÷ probe |
| ----------- | ---------------: | -----------: | ---------------: |
| Quiet       | 0.745 s, 0.750 s |      24.02 s |        **32.13** |
| 96 spinners | 1.172 s, 1.064 s |      36.06 s |        **32.25** |

**The ratio is stable to 0.4% across a 1.5× CPU inflation.** Normalizing — `contract_cpu × (PROBE_BASELINE ÷ probe_cpu)`
— reads 24.02 s on a quiet host and 24.11 s under load that made the raw wall clock read 48 s.

Because the probe self-calibrates against whatever machine it runs on, the budget also becomes **machine-portable**,
which matters directly: the observed CI failure was on a GitHub runner, not on this host, and an absolute second count
was never going to mean the same thing in both places.

### Deliverables

- Replace the wall-clock assertion with the normalized measurement. Take child CPU as a
  `resource.getrusage(RUSAGE_CHILDREN)` delta around `subprocess.run`; grandchildren are included transitively as long
  as each intervening parent waits on its own children, which `subprocess.run` does.
- Bracket the nested run with one probe before and one after, and use their mean. Bracketing is what makes the ratio
  hold: a single probe taken before a 25 s run can miss a load change that arrives mid-run.
- **Keep `_BUDGET_SECONDS = 30.0`.** The plan's reasoning for the 30 s tax ceiling — the contract set is added to every
  scoped selection, so it is paid by every agent on every `just check` — is unchanged. Record the probe baseline
  constant and the measured quiet-host normalized figure in a comment next to it, with the date and host, in the style
  of the comment already there.
- The failure message must print raw wall, raw CPU, both probe CPUs, the normalization factor, and the normalized
  figure. A future failure of this guard should be immediately diagnosable as either "the set really grew" or "the
  normalization broke", never a mystery.
- `resource` is Unix-only. Check whether anything in the repo already depends on it; if not, skip the guard with an
  explicit reason on a platform that lacks it rather than raising `ImportError` at collection time.
- Unit coverage for the normalization arithmetic itself — pure-function tests over injected CPU numbers, which must not
  run the contract set. Include the two measured conditions above as regression cases: the same set under quiet and
  loaded probes must normalize to within a stated tolerance of each other.
- Re-measure and report the current headroom. `3e4b5955c` split `test_run_pytest_tool.py` into five contract-marked
  modules, which the splitting agent measured at roughly +0.35 s (~1.4% of the serial baseline) purely from extra module
  setup. That cost is real and should stay visible; the point of this phase is that the margin is no longer eaten by
  measurement noise, not that module count stops counting.
- **Settle `sase-fp.2`'s deferred question.** That phase curated `tests/test_suite_gate_integration.py` (~3.6 s) and
  `tests/test_markdown_template_packaging.py` (~1.3–1.7 s, invokes `uv build`) _out_ of the set purely for margin, and
  named the arrival of `sase-fp.3`'s scoped-run integration test in `test_suite_gate_integration.py` as the natural
  point to re-decide. That test has since landed. Re-measure with each candidate added and decide. Either outcome is
  acceptable; record the measured cost and the reasoning either way.

### Also worth knowing (not in scope)

The sibling guard `test_contract_manifest_matches_marker_selection` costs ~21 s on its own, because it runs a whole-repo
`--collect-only`. The module therefore costs roughly 45 s of every full-suite run. Leave it alone here; note it if it
looks cheap to improve.

### Verification

Beyond `just check-full`: run the guard repeatedly under a deliberate synthetic load (the spinner harness above is
enough) and confirm it passes where the current assertion fails, and run it at least twice inside a real
`just check-full` on a contended host.

---

## Phase `correlate` — Change-scoped false-negative correlation

### The defect

`find_false_negatives` in `tests/_test_selection_health.py` pairs a full-run failure with a scoped manifest on exactly
two conditions: the manifest did not escalate, and its `baseline.head` is an ancestor of the full run's head.

The epic plan defines the metric as "a test that failed in a full run and was excluded by a scoped run over an ancestor
of **the same change**". The implementation drops "the same change" — and it has to, because `full_run_record()` stores
only `head`, `mode`, `failures`, and `exit_status`. There is no change set and no workspace identity on a full-run
record, so the restriction cannot be expressed.

The store is host-local and **shared across every numbered workspace**, which is deliberate and is stated as a feature
in the epic plan. Those workspaces all sit on the same master HEAD, so `is_ancestor(head, head)` is trivially true and
**every** non-escalated scoped manifest matches **every** full-run failure written by **every** other workspace.

### The measured consequence

`just selection-health` at HEAD reports **9 false negatives across 32 matches**. Inspecting all nine: none are genuine.

The clearest pair — scoped manifest `20260806T040106Z-96183d71b3ef-2533672.json` has
`changed_files = [AGENTS.md, CLAUDE.md, GEMINI.md, OPENCODE.md, QWEN.md, sase/memory/README.md, sase/memory/build_and_run.md]`.
That is `sase-fp.7`'s docs-only change, which correctly fires `contract-set-only`. It is charged with the failure of
`tests/test_select_tests_tool.py::test_json_format_prints_the_manifest`, recorded in
`20260806T043506Z-96183d71b3ef-3055134-full-run.json` — a different workspace's full run, of a different change. The
remaining eight are the known load-sensitive flakes (`test_stall_watchdog.py`, `test_app_title.py`,
`test_notification_custom_gate.py`, `test_prompt_codeblock_highlight.py`, `test_cli_work_contention_regressions.py`)
plus the budget guard the `budget` phase fixes.

This is worse than a wrong count. The epic plan makes this metric the epic's **exit criterion**, and instructs the
reader that "if the rate is not zero, the response is to raise `SASE_TEST_SELECTION_DEPTH` to 3 or add the missed tests
to the contract set". Acting on today's reading means raising the depth — which `sase-fp.1` measured at a 9.9% median
selection against the plan's assumed 5.5%, i.e. **2.4× more expensive than the plan believes** — on the strength of pure
noise.

### Deliverables

- Put the linkage on the record. `full_run_record()` must carry the same identity a scoped manifest already carries: the
  change set, computed the same way the selector computes it, and a stable workspace identity (the repo root, or a
  digest of it — pick one and say why).
- Bump the health record schema. The store holds up to 30 days of existing records with neither field; they must be
  ignored for correlation, not crash the report. `just selection-health` should say how many records were skipped as
  pre-schema so a zero reading is not mistaken for a clean one.
- Restrict `find_false_negatives` to pairs that plausibly describe the same change: same workspace, HEAD ancestry as
  today, and the full run's change set a superset of or equal to the scoped run's. Agents commit as they go and the
  change set is computed against the `origin/master` merge-base, so a later full run in the same workspace normally sees
  a superset of an earlier scoped run's files.
- State the rule in the module docstring **and** in `just selection-health`'s output. A metric whose matching rule is
  invisible is a metric nobody can argue with.
- Flakes will still leak through: a flaky test can fail in a full run over the very change a scoped run excluded it
  from. **Do not attempt flake detection here** — that is a separate problem with its own follow-up. Do make the report
  distinguish a nodeid matched once from a nodeid matched by many _unrelated_ selections, because repetition across
  unrelated change sets is the signature of a flake rather than a selection miss.
- Unit coverage over synthetic records: same-workspace/same-change match; cross-workspace non-match; disjoint-change
  non-match; superset-change match; legacy records without the new fields; escalated manifests still skipped; visual
  paths still skipped.
- Re-run `just selection-health` and record the corrected reading in the phase's close note.

### Be honest about the sample

The store currently holds **5 scoped runs**. The epic's stated exit criterion is "zero false negatives across at least
30 varied changes". That sample does not exist yet and this phase cannot manufacture it. Report the corrected reading
_and_ the sample size; do not claim the criterion is met. Whether depth 2 is right is a question for real usage over the
coming weeks, and the corrected metric is what makes that question answerable at all.

---

## Phase `land` — Land epic sase-fp

Runs after `budget` and `correlate` are both committed and `just check-full` is green on the combined tree.

### Follow-ups to file

Use `/sase_new_task` for each, one at a time, naming the proposing bead. Record every outcome — including any the skill
corroborates as a duplicate or attaches to another active epic, and any you decline — in the close note.

1. **Share the import-graph cache host-locally** (`sase-fp.1`). Every fresh ephemeral workspace pays a cold ~8.4 s AST
   build because `.pytest_cache/sase-selection/graph.json` is per-workspace; a `${SASE_HOME}/test-selection/graph` cache
   keyed by `(path, size, mtime_ns)` would make the first scoped check in a new workspace as cheap as the second.
2. **Re-measure the depth table before any recommendation to raise `SASE_TEST_SELECTION_DEPTH`** (`sase-fp.1`). The
   implementation reproduces the plan at depths 0–2 but its depth-3 median is 9.9%, not the plan's 5.5% — depth 3 is
   ~2.4× more expensive than the plan assumes, and the epic plan's own remediation advice points at depth 3.
3. **Correlate GitHub Actions failures with local selection manifests** (`sase-fp.5`, and the epic plan's own
   out-of-scope list). CI, not just local full-lane runs, should feed the false-negative metric; needs the manifest to
   travel with the change.
4. **Attribute the escalation-rate delta to coverage contexts, and revisit the 0.25 max-ratio** (`sase-fp.6`). Contexts
   union real per-test hits into the selection, so changes in widely-executed modules will cross the ratio more often;
   `just selection-health` reports the rule histogram but cannot attribute the delta.
5. **Coverage-contexts baseline availability** (`sase-fp.6`). The `sase-coverage-contexts` artifact has 14-day retention
   and is published on master pushes only, so a workspace idle longer than that silently falls back to the static
   closure. Consider a scheduled refresh, longer retention, or a `just selection-health` warning when no baseline is
   within retention.
6. **The load-sensitive flake family — one task, not seven** (`sase-fp.3`, `.4`, `.5`, `.6`, `.7`, plus the health store
   at HEAD). All pass standalone and fail under full-suite contention:
   `tests/test_bead/test_cli_work_contention_regressions.py::test_concurrent_bead_mutations_wait_past_the_old_lock_timeout`
   (which also failed on a clean HEAD worktree with no local changes),
   `tests/ace/tui/util/test_stall_watchdog.py::{test_watchdog_keeps_hitch_and_stall_state_machines_independent, test_watchdog_writes_loop_recovery_record, test_watchdog_records_compact_pump_hitch_and_recovery}`,
   `tests/ace/tui/test_prompt_bar_xprompt_selector_requests.py::test_vcs_tag_{offers_project_local_xprompts_by_canonical_name, directory_key_spelling_also_resolves}`,
   `tests/ace/tui/test_app_title.py::test_on_mount_refines_title_to_resolved_version`,
   `tests/ace/tui/test_notification_custom_gate.py::test_tracked_executor_reports_terminal_and_extra_commands_live`, and
   `tests/ace/tui/widgets/test_prompt_codeblock_highlight.py::test_codeblock_band_replaces_cursor_line_fill_but_not_cursor`.
   These are **not** caused by `sase-fp`; the epic made them visible by measuring the full lane. They need real
   timing/isolation hardening rather than wall-clock budgets.
7. **Break up the 497-module `src/sase` import cycle** (epic plan). It is the root cause of static selection being
   unsound here.
8. **Import Pillow lazily in `tests/ace/tui/visual/conftest.py`** (epic plan, carried from Tier 0), so `just test` and
   `just test-cov` can drop `_setup-visual` the way `just test-scoped` already does.
9. **Rewrite the three dismissed-bundle scale tests** to build fixtures directly and return them to the fast lane (epic
   plan, carried from Tier 0).

`sase-fp.2`'s contract-set membership proposal is deliberately **not** on this list — the `budget` phase settles it.

### Close and finish

- `sase bead close sase-fp --note "<verification, integration, both fixes, the corrected metric reading and its sample size, and every follow-up outcome>"`.
- **After** the close, run `just symvision` — `sase-fp`'s epic-symbol whitelist entries expire at close — and remove any
  stale entries and unused code it reports. There are no `sase-fp` whitelist entries in the tree today and symvision is
  green at `d66101e8f`, so expect a no-op; verify rather than assume.
- Set `status: done` in the frontmatter of `plans:202608/test_suite_tier1.md`
  (`/home/bryan/.sase/plans/202608/test_suite_tier1.md`).
- If the close is rejected, the named phases were never completed: finish or reopen them, or record the outcome
  deliberately with `--force --reason ... --resolution canceled|superseded`. Never force merely to make the command
  succeed.

---

## Verification

Every phase runs `just install` first — workspaces are ephemeral and may be stale — and then `just check-full`.

`just _lint-symvision` is **green** at `d66101e8f`; the epic plan's note that it is known-red for `progress_fingerprint`
is stale, fixed by `a4a2c1a60`. Any symvision failure in these phases is real.

Expect the load-sensitive flakes in follow-up 6 to appear in contended `just check-full` runs. They pass standalone and
are untouched by this plan; confirm standalone rather than chasing them.
