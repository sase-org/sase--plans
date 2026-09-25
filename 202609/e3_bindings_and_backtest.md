---
tier: tale
title:
  "E3 bindings-and-backtest: pin the core, gather triage inputs, pass the precision
  backtest"
goal:
  sase calls every E3 triage binding through thin adapters at a pinned core, bounded
  fail-open gatherers feed ancestry, flake baseline, selection-health witnesses, and
  bead candidates, runs record their workspace, and tools/tool_triage_backtest proves at
  least 95% hand-audited KNOWN precision on athena before any label is stored.
size: medium
proposed_by: bbugyi200.athena.sase-18j.5
bead: sase-18j.5
create_time: 2026-09-24 21:47:17
status: wip
---

- **PARENT:**
  [202609/tool_e3_failure_triage.md](https://github.com/sase-org/sase--plans/blob/main/202609/tool_e3_failure_triage.md)
- **BEAD:**
  [sase-18j.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-18j/sase-18j.5.md)

# Plan: E3 phase `bindings-and-backtest` (bead `sase-18j.5`)

This plan completes phase 5 of epic `sase-18j` (`plan:202609/tool_e3_failure_triage.md`,
section "5. bindings-and-backtest"). It moves the core pin, adds thin adapters and
validators for every triage binding, builds the bounded input gatherers, writes
`tools/tool_triage_backtest`, and passes the DoD-5 precision gate on athena. No label is
stored anywhere. Recording, the footer, the flag, and continuation belong to later
phases. Phase `record-and-render` (`sase-18j.6`) consumes this work.

## Facts established while planning (recheck when implementing)

- The linked `sase-core` `origin/master` is `321e7b4` ("pure classification, verdict,
  stage/settle, and failures aggregation"). It sits on top of `8315364` (failure items,
  extract/record/show) and descends from the current pin `83153645fe14`. Together those
  two commits expose all 8 bindings in the telemetry domain: `tool_run_triage_extract`,
  `_record`, `_show`, `_classify`, `_verdict`, `_stage`, `_settle`, and
  `tool_run_failures`. Pure bindings take `(request)`. Store bindings take
  `(store_path, request, busy_timeout_ms=250)`.
- The installed `sase_core_rs` in a stale workspace venv lacks these bindings. Run
  `just rust-install` (a wheel-cache hit is likely) before any real-binding test.
- **Workspace gap.** `runs.workspace` is NULL on all 637 recorded `check` runs.
  `executor_recording.py` reads only `SASE_WORKSPACE_NUM`, but agents export
  `SASE_AGENT_WORKSPACE_NUM`. The classifier's witness rule requires "a different
  workspace or a clean tree", and it counts distinct workspaces. So with NULL
  workspaces, live KNOWN labels could only come from clean-tree witnesses, which are
  rare for agents. This phase fixes the recording (step 3) because the gatherers and the
  backtest share the identity helper.
- Historical workspace numbers can be recovered. Each agent's
  `~/.sase/projects/<project>/artifacts/ace-run/<yyyymm>/<dd>/<ts>/agent_meta.json`
  carries `name`, `workspace_num`, and `workspace_dir`. The backtest maps `runs.agent` →
  `workspace_num` using the newest meta whose timestamp is at or before the run's
  `created_ts`.
- Ledger shape on athena:
  - 533 failed `check` runs; 485 have a retained `stdout.log` and 7 have only an owner
    log.
  - Every row has `fingerprint_before_json`.
  - 60 rows have a head that does not resolve in the sase repo. These are the pre-fix
    linked-repo rows of decision 14, and the backtest excludes them.
  - Stage rows include fixture leaks (`stage one`, 36 rows). Stage lists changed over
    time: `lint (toobig)` used to be a stage.
- Selection-health `full-run` records (schema 2) live under
  `~/.sase/test-selection/<project_key>/*-full-run.json`. Each carries `recorded_at`,
  `head`, `workspace` (an absolute path), `changed_files` (possibly null), `tree_dirty`,
  `exit_status`, and `failures` (node ids).
- Owner candidate wires use `node_id` = the bead id. `location` holds the task's
  `task_type_fields` `node_id`/`location` text.

## Steps

### 1. Move the pin (decision 10)

- Set `sase-core-revision.txt` to the full 40-character SHA of the pushed `sase-core`
  commit containing phases 2–3 (`321e7b4…`, or a later `origin/master` commit if one
  landed).
- Verify that it is an ancestor of the linked checkout's `origin/master` and descends
  from the old pin.
- Do not touch the `sase-core-rs` window in `pyproject.toml`.
- Run `just rust-install`, then confirm that the venv's `sase_core_rs` exposes all 8
  names.

### 2. Adapters in `src/sase/core/tool_run.py`

Add thin wire-only adapters in the existing style (`require_rust_binding`, default
`tool_run_store_path()`, `busy_timeout_ms=250`) and extend `__all__`:

- pure: `tool_run_triage_extract(request)`, `tool_run_triage_classify(request)`,
  `tool_run_triage_verdict(request)`;
- store: `tool_run_triage_record`, `tool_run_triage_show`, `tool_run_triage_stage`,
  `tool_run_triage_settle`, `tool_run_failures` (request mapping plus the `store_path`
  and `busy_timeout_ms` keywords).

Every payload defaults `schema_version: 1`.

**Symvision:**

- `extract`, `classify`, and `verdict` have a real consumer in
  `tools/tool_triage_backtest`. Keep them alive with
  `# symvision: tools/tool_triage_backtest` pragmas, or with no pragma if Symvision
  already sees the tools reference. Let the lint decide, and never keep an unnecessary
  pragma.
- The adapters that later phases consume (`record`, `show`, `stage`, `settle`,
  `failures`) get `--epic-symbol 'sase-18j(<name>)'` entries in the Justfile
  `_lint-symvision` recipe, under a comment naming the consuming phases
  (`record-and-render` for settle/show/record, `known-gated-continuation` for stage,
  `failures-and-followups` for failures). They are keyed on the epic, not on this phase,
  so closing `sase-18j.5` leaves no stale entry. Do the same for any gatherer symbol
  from step 3 that only a later phase consumes.

### 3. Input gatherers in a new `src/sase/tool/triage_inputs.py`

Each gatherer is bounded and fail-open, and returns `(value, diagnostics)`. A missing
input can only produce fewer KNOWN or FLAKY labels. No gatherer imports from `tests/` or
`tools/`. The module docstring states this contract.

- **Constants:** the tightening knobs `MIN_WITNESSES = 1` and
  `TOUCHED_REQUIRES_CLEAN_WITNESS = False` (the final values come from step 5),
  `ANCESTRY_MAX = 2000`, `LOOKBACK_SECONDS = 7 * 86400`, and per-gatherer subprocess
  timeouts. Also `triage_knobs()` →
  `{"min_witnesses": …, "touched_requires_clean_witness": …}`.
- **`workspace_identity(path)`:** the `<name>_<N>` workspace number (as a string) found
  in the path's components, else `None`. It is used by the executor fix below and by
  selection records.
- **`gather_ancestry(repo_root, base)`:**
  `git rev-list --first-parent --max-count=2000 <base>` in the catalog repo root, newest
  first. On an unresolvable base or a timeout it returns `[]` plus a diagnostic.
- **`gather_flake_baseline(repo_root, base)`:**
  - read `git show <base>:tests/reproducible_flake_baseline.txt`;
  - parse it with `tools/selection_health`'s rules, reimplemented: blank lines skipped;
    `#` comments skipped except `# fixed-at: <ts> <node>`, which retires that node;
    every other line is an active node id;
  - turn active node ids into pytest signatures through one `tool_run_triage_extract`
    call over synthetic `FAILED <node>` lines (stage key `test (scoped)`), so the core
    owns normalization;
  - return `ToolRunTriageFlakeEntryWire` dicts with
    `source = "tests/reproducible_flake_baseline.txt@<base12>"`.
- **`gather_selection_records(project_key, *, before_ts=None, now_ts)`:**
  - store dir: `SASE_TEST_SELECTION_HEALTH_DIR`, else
    `sase_home()/test-selection/<SASE_TEST_SELECTION_HEALTH_PROJECT_KEY or project_key>`;
  - read schema-2 `kind: full-run` records within the lookback, bounded to the newest
    500;
  - return evidence-run dicts with `selection_source: true` and the subject's
    project/tool/extra-args digest (the caller supplies these);
    `settled_ts = recorded_at`; `base_head = head`;
    `workspace = workspace_identity(workspace)`; `dirty_paths = changed_files`;
    `dirty_unknown = changed_files is None and tree_dirty`;
    `clean_tree = not tree_dirty`; `complete_fingerprint: true`; no fingerprint digest;
    `failed = exit_status != 0`; `stage_completions: []` (witnesses only, per decision
    4); items = pytest signatures of `failures`, via one batched extract call.
- **`gather_owner_candidates(project_root, *, now_ts)`:**
  - resolve the bead store with
    `resolve_beads_location(cwd=project_root, require_existing=True)`, then call
    `sase.core.bead_read_facade.list_issues(..., issue_types=[TASK])`;
  - keep `task_type` ∈ {ci, flake, bug} that are open, or closed within the lookback;
  - map each to
    `{node_id: id, location: <task_type_fields node_id/location>, title, status: open|closed, closed_ts}`;
  - read-only: never mutate a bead.

**Executor workspace fix (`src/sase/tool/executor_recording.py`).** The begin request's
`workspace` becomes `SASE_WORKSPACE_NUM`, else `SASE_AGENT_WORKSPACE_NUM`, else
`workspace_identity(resolved.cwd or os.getcwd())`. Add a test in the existing executor
tests that an agent-like env records the workspace number.

### 4. Validators and smokes (the E2 `b6b9f4f59` pattern)

- `tools/validate_sase_core_rs`:
  - add the 8 names to `REQUIRED_BINDINGS`;
  - add `_validate_tool_run_triage_contract(module)`. It extracts a mypy line and checks
    for one parsed item with a 64-hex signature. It verdicts `{exit_code: 0}` and checks
    for `pass`. It records and shows one item in a temp store and checks that it
    round-trips. Finally it checks that `tool_run_failures` returns `groups`. Wire it
    into `main`.
- `tools/smoke_sase_core_rs_tool_runs`: add the names to `TOOL_RUN_BINDINGS` and a
  triage round trip in the temp store (extract → record → show → settle → failures),
  then surface a few result fields in the returned dict.
- Run `tools/check_sase_core_rs_bindings`. Its static scan picks the new call sites up
  and must pass against the rebuilt venv.
- Update `tests/test_validate_sase_core_rs_tool.py` and
  `tests/test_sase_core_rs_tool_runs_smoke_tool.py` for the new names and outputs.
- Add real-binding round trips to `tests/core/test_tool_run_store.py`:
  - extract a pytest item;
  - record, then show;
  - classify a KNOWN fixture (one witness in another workspace at the subject's base)
    and an UNKNOWN fixture;
  - check that verdict maps the legacy `failed` exit 1 to `verification`;
  - settle on a failed run;
  - check that failures groups the run's signature.

### 5. `tools/tool_triage_backtest` plus pytest twin `tests/test_tool_triage_backtest.py`

This is a read-only Python script with argparse. It follows the header conventions of
other extensionless tools and is type-checked by `typecheck_extensionless_tools`. It
imports the sase adapters and gatherers.

- **Options:** `--store` (default `tool_run_store_path()`), `--repo-root` (default: repo
  root), `--project` (default `gh_sase-org__sase`), `--tool check`, `--artifacts-root`
  (agent_meta scan root; default `sase_home()/projects/<project>/artifacts`),
  `--selection-dir`, `--out-dir` (default
  `sase_home()/tools/triage_backtest/<UTC stamp>/`), `--sample N` (default 60), `--seed`
  (default 1), `--min-witnesses`, and `--touched-requires-clean-witness` (the defaults
  are the gatherer constants).
- **Selection:** settled runs with `tool_name = --tool` and `project = --project`, a
  complete `fingerprint_before`, and a head that resolves as a commit in `--repo-root`
  (decision 14). Ad-hoc rows are never selected.
- **Stage reconstruction:**
  - read the stage list of the `check` recipe at `base(R)`: parse
    `git show <base>:Justfile` for `tools/run_silent "<desc>"` lines in the recipe
    (cached per base);
  - drop stage rows whose description is outside that list or repeated (fixture
    contamination);
  - split `stdout.log` (or `owner_log_path` when that is the only log) at top-level
    `✓ <desc>` / `✗ <desc>` marker lines, where `<desc>` is in the list and not yet
    seen;
  - each `✗` section runs to the next top-level marker or EOF;
  - a failed run with no failed stage uses the whole log tail (256 KiB) under stage key
    `*`.
- **Exclusions (reported):** missing log, missing failed-stage section, unresolvable
  head, and incomplete fingerprint. An excluded failed run is never used as evidence,
  because its unextracted failure would look like a clearing run. Its completed stages
  are also withheld.
- **Replay:**
  1. Process runs in `(settled_ts, run_id)` order.
  2. Extract each failed stage with `tool_run_triage_extract` (`project_root` = the repo
     root; the workspace roots come from the core's own normalization).
  3. Classify each verification run's items with `tool_run_triage_classify`. The subject
     comes from `fingerprint_before`: base, dirty paths, complete flag, and the digest
     from `tool_run_canonicalize_fingerprint`. Workspace comes from the agent_meta map.
     Evidence is the prior runs only (`settled_ts <` the subject's, within `L`, newest
     500), and it includes succeeded, signaled, and lost runs, with their completed
     top-level stages and extracted items.
  4. Other inputs: selection records recorded before the subject's settle, ancestry, and
     the flake baseline at base. There are no owner candidates.
  5. Compute each run's verdict with `tool_run_triage_verdict`, for the distribution.
- **Outputs** in `--out-dir`:
  - `report.json`, which carries `schema_version: 1`, the knobs, counts, the label
    distribution overall and per stage, the UNKNOWN reason histogram, the verdict
    distribution, the excluded runs by reason, and the KNOWN count on added or untracked
    files (dirty status `added`/`untracked` for the item's locator);
  - `KNOWN-but-touched` items;
  - the seeded random sample of at least 50 KNOWN labels. Each carries the item display,
    locators, stage, subject run and agent, base, the subject's dirty paths, witnesses
    (run id, agent, workspace, base, and whether that base equals the subject's), and
    two audit aids: `locator_in_subject_diff`, and `locator_unchanged_since_witness`
    from `git diff --quiet <newest witness base> <base(R)> -- <locators>`;
  - `audit.md`, a worksheet with one row per sample and an empty justification column,
    followed by the `KNOWN-but-touched` list.
- **Pytest twin:** build a fixture ledger in a temp `SASE_HOME` with the real core
  `tool_run_begin`/`append_event`/`finish` (or direct SQL matching the schema if begin
  cannot set historical timestamps). Add fixture logs, a tiny git repo with two commits
  and a Justfile `check` recipe, and agent_meta files. Assert that:
  - a pre-existing symvision item seen by another workspace at an ancestor base is
    KNOWN;
  - the first occurrence is UNKNOWN;
  - an item on the subject's added file is not KNOWN;
  - a contaminated stage row is dropped;
  - a missing-log run is excluded and is not used as evidence;
  - the linked-repo head is excluded;
  - `report.json` and `audit.md` exist with the documented keys;
  - the sample is deterministic for a seed.

  Mark the twin with whatever marker the other real-binding tool tests use.

- Add a short `## Triage backtest` note to `tools/AGENTS.md` (purpose, read-only,
  outputs). The pytest twin also satisfies `pyscripts` reference rule 1.

### 6. Run the backtest on athena and hand-audit (DoD-5)

1. Run `.venv/bin/python tools/tool_triage_backtest` against the live ledger (it is
   read-only; use `/sase_monitor` if it risks outliving the turn).
2. Copy `report.json` and `audit.md` into this agent's artifacts directory
   (`$SASE_ARTIFACTS_DIR/triage_backtest/`).
3. Hand-audit every sampled KNOWN label and fill its justification line. An item counts
   as pre-existing when its cause exists at `base(R)` unchanged by R's diff, checked
   against the evidence and `git` (for example with `git show <base>:<path>` or
   `git log <witness base>..<base> -- <path>`). Symvision items also need the symbol's
   consumer state checked at base.
4. **Gate:** at least 95% pre-existing, zero KNOWN on added files, and every
   KNOWN-but-touched item dispositioned.
5. On failure, tighten the knobs (`--min-witnesses 2`, then
   `--touched-requires-clean-witness`), re-run, and re-audit a fresh sample. Never relax
   the audit.
6. Set the final knob values as the `triage_inputs` constants.
7. If no knob setting passes, record the failure. The rule change belongs in `sase-core`
   under a new rule version, and `record-and-render` must move the pin first. Record
   this as a `PROPOSED FOLLOW-UP:` and leave the phase open with the evidence in a note,
   rather than closing it with a failed gate.

### 7. Verify and close

- Run `just fmt`, then `sase tool run check` with a generous explicit timeout.
  Pre-existing master-red items (for example the `sase_core_wheel_cache` mypy failure
  noted by `sase-18j.4`) are not blockers. Record any that reproduce on the clean base
  as a follow-up.
- Run `sase bead epic-symbols sase-18j.5` and resolve every entry keyed on this phase.
  Entries keyed on `sase-18j` stay.
- Bead notes:
  - the label distribution and the UNKNOWN reason histogram;
  - the audit summary (N sampled, N pre-existing, %, the KNOWN-on-added count, and the
    touched dispositions);
  - the final knob values and the report location;
  - the pin SHA;
  - the workspace-recording fix.
- `PROPOSED FOLLOW-UP:` notes:
  - Owner matching in `sase-core` tokenizes locator paths into generic tokens such as
    `src` and `sase`, and those tokens appear in every candidate's bead id (`sase-…`).
    Nearly every item would therefore match arbitrary beads. Confirm this with a real
    call during step 4; if it is confirmed, the core needs a token stoplist or a
    path-level match before `record-and-render` renders owners.
  - Anything else discovered.
- Close with
  `sase bead close sase-18j.5 --note "<pin, adapters, gatherers, backtest gate result>"`.
  Never close the epic.

## Out of scope

- Storing labels or items in the live ledger, the settle-time triage call, and the
  `tool_failure_triage` flag (`record-and-render`).
- The `_triage-stage` verb (`known-gated-continuation`).
- `sase tool failures` (`failures-and-followups`).
- Docs and memory edits (`acceptance-and-governance`).
