---
tier: epic
title:
  "Unblock E3: repair the triage backtest, pass the precision gate, close sase-18j.5"
goal: "Phase sase-18j.5 closes on a DoD-5 precision backtest that was measured correctly
  and hand-audited for real. Closing it releases the E3 agents that are already queued
  (sase-18j.6 through sase-18j.9 and sase-18j.land). They finish the epic, and
  sase-18j.land closes sase-18j.

  "
phases:
  - id: backtest-repair
    title: Fix the backtest's metrics, witness evidence, and workspace attribution
    depends_on: []
    size: medium
    description:
      "backtest-repair: fix four defects in tools/tool_triage_backtest (added-file
      metric, witness evidence, workspace attribution, selection lookback), make the
      audit worksheet auditable, and add the fixture twin and real-binding round trips
      that sase-18j.5 skipped."
  - id: continued-stage-output
    title: Show continued stage failures as failures
    depends_on: []
    size: small
    description:
      "continued-stage-output: make tools/run_silent print the failure marker and the
      captured output for a stage it continues past, instead of a check mark and no
      output, and pin that with a keep-going test."
  - id: precision-gate
    title: Run and hand-audit the DoD-5 backtest, then close sase-18j.5
    depends_on:
      - backtest-repair
      - continued-stage-output
    size: medium
    description:
      "precision-gate: run the repaired backtest on athena, hand-audit at least 50 KNOWN
      labels, tighten the knobs only if the audit fails, record the evidence and
      follow-ups, and close sase-18j.5 so the queued E3 phases resume."
proposed_by: bbugyi200.athena.0rz
create_time: 2026-09-25 07:43:27
status: wip
---

- **PROMPT:**
  [prompts/202609/e3_precision_gate.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/e3_precision_gate.md)

# Plan: Unblock E3 by finishing phase `bindings-and-backtest` (`sase-18j.5`)

## Why E3 is stuck

Epic `sase-18j` (E3 failure triage, `plan:202609/tool_e3_failure_triage.md`) has phases
1 to 4 closed. Phase 5 (`sase-18j.5`, approved phase plan
`plan:202609/e3_bindings_and_backtest.md`) landed its code in `cdcbcdd9d`. Lint fixes
followed in `7840592c5` and in the `sase-18f` landing. Phase 5 then stopped at the DoD-5
precision gate and left itself `in_progress`, with a `PROPOSED FOLLOW-UP` that says the
gate failed with 139 KNOWN items on added or untracked files.

The agents for `sase-18j.6`, `.7`, `.8`, `.9`, and `sase-18j.land` are alive and
`WAITING` on bead dependencies. `sase-18j.6` waits on `sase-18j.5`, and the rest wait
behind it. So the whole epic is blocked on one phase bead, and closing `sase-18j.5`
restarts it. This epic does **not** reimplement phases 6 to 9. It never closes
`sase-18j`, because that bead's land agent does. It never runs `sase bead work sase-18j`
while those runners are alive, and it never stops or edits them.

Planning found that the gate never really ran. The following facts were checked on
athena against the live ledger and the three reports that phase 5 left in `/tmp`.
Recheck them before relying on them.

1. **The added-file metric is wrong.** `known_on_added_or_untracked` counts a KNOWN item
   when _any_ dirty path of its run has status `added`/`untracked`. It should count an
   item only when the item's own locator is an added file. Agents almost always have
   some untracked scratch file, so the count is mostly noise. In the 60-item
   clean-witness sample, 0 locators were on added or untracked files, although the
   report claimed 139 overall.
2. **The audit worksheet has no witnesses.** `_item_audit` reads
   `evidence["witnesses"]`. The core emits `witness_run_ids` and `selection_record_ids`
   (`crates/sase_core/src/tool_run/triage/classify.rs` in `sase-core`). So every sample
   has `witnesses: []` and `locator_unchanged_since_witness: false`, and the
   justification column was never filled in. No hand audit ever happened.
3. **Workspace attribution fails for 689 of 692 runs.** `runs.workspace` is NULL on
   historical rows. The `agent_meta.json` fallback reads `created_ts`/`started_ts` keys
   that do not exist, so it falls back to the file's mtime. That mtime is later than the
   run, so `_workspace_for` returns `None`. The classifier then treats every dirty run
   as sharing the subject's workspace. That has two effects:
   - Only clean-tree runs could witness.
   - `min_witnesses = 2` could never be met by ledger runs. That is why the tightened
     runs showed 0 KNOWN symvision/mypy labels and 620 `insufficient_witnesses`.

   Reading `run_started_at` (ISO 8601; present in about 97% of meta files) resolves 608
   of 692 runs across 20 workspaces. `executor_recording.py` has recorded the workspace
   live since `cdcbcdd9d`.

4. **Selection-record lookback uses the wall clock.** The backtest passes the real "now"
   to `gather_selection_records`, so selection witnesses for older subjects are dropped.
   The subject's `settled_ts` is the replay's "now". This only ever removed evidence.
5. **Phase 5 skipped several tests.** The fixture-ledger assertions for the pytest twin
   are missing (only an empty-ledger test exists). The real-binding triage round trips
   in `tests/core/test_tool_run_store.py` are missing. The owner-matching confirmation
   was never recorded.
6. **Continued failures print `✓`.** `tools/run_silent` sets `exit_code=$decision_code`
   after a `continued` decision, so it prints `✓ <stage>` and discards the output of a
   stage that failed. This is epic note #1 on `sase-18j`. It becomes every agent's
   default experience once phase 7 makes `known` the agent default.
7. **The pin is fine.** `sase-core-revision.txt` is `321e7b4762ff…`, which is on
   `sase-core` `origin/master` and exposes all 8 triage bindings. The Justfile has no
   `--epic-symbol` entries keyed on `sase-18j.5`. The seven `sase-18j(...)` entries
   belong to phases 6 and 8 and must stay.
8. **Owner matching is broken in the core.** `locator_tokens` keeps every token of 3 or
   more characters (`src`, `sase`, `tests`, `tool`), and `match_owners` does a substring
   test against `node_id` + location + title. Every `sase-…` bead id contains `sase`, so
   nearly every item matches the first two candidates. `sase-18j.6` renders owners, so
   this must reach it before then (see phase `precision-gate`).

The "Relationship to open work" rules of the E3 plan still apply. So do decisions 3
(pure Rust classification), 6 (no label before the gate), and 14 (legacy rows stay as
history). Every phase runs `sase tool run check` per `sase/memory/lint_and_test.md`,
with an explicit tool timeout of at least 20 minutes. Master-red failures in files the
phase did not touch are not blockers; name them in the phase bead note. Phase agents
record discoveries as `PROPOSED FOLLOW-UP:` notes on their own phase bead and create no
beads.

## 1. backtest-repair

Work in `tools/tool_triage_backtest`, its twin
`tests/test_tool_triage_backtest_tool.py`, and `tests/core/test_tool_run_store.py`.
Classification stays in Rust, and this phase changes no `sase-core` code. The script
stays read-only with respect to the ledger. The empty-ledger test must keep passing: a
missing store is never created.

**Fixes:**

- **Added-file metric.** An item counts toward `known_on_added_or_untracked` only when
  at least one of its own locator paths meets one of these:
  - it is absent from `base(R)`'s tree (`git cat-file -e <base>:<path>` fails). This is
    the authoritative test, because porcelain reports a staged-then-edited file as
    `modified`;
  - it equals a subject dirty entry whose status is `added`, `untracked`, `R`, or `C`.

  Keep the counts key name. Add a report list `known_on_added` with the full audit
  record of each counted item. Also add `known_items_in_runs_with_untracked_paths`, the
  old any-path count, labeled as informational, so the earlier 139 can be explained.

- **Witness evidence.**
  - Resolve a label's `witness_run_ids` and `selection_record_ids` against the evidence
    rows the classifier actually received. Build one `{run_id: row}` lookup from
    `previous` plus that subject's selection rows.
  - For each witness, emit run id, agent, workspace, `base_head`, whether that base
    equals `base(R)`, `clean_tree`, `dirty_paths`, and whether it is a selection record.
  - The newest witness is the one whose base has the smallest index in the subject's
    `ancestry` (equal to `base(R)` counts as index 0), not the last list element.
  - `locator_unchanged_since_witness` is
    `git diff --quiet <newest witness base> <base(R)> -- <locators>`.
  - Also carry the label's `distinct_workspaces`, `distinct_agents`, `first_seen_ts`,
    `touched`, and `extractor`.
- **Workspace attribution.**
  - Prefer the ledger row's own `workspace` when it is non-empty.
  - Otherwise use the agent_meta map, timestamped by `run_started_at` (ISO → epoch).
    Fall back to the artifacts directory name (`YYYYMMDDHHMMSS`, local time), and only
    then to the file mtime.
  - Keep the existing rule: the newest meta at or before the run's `created_ts`.
  - Add `counts.workspace_attribution`: `{ledger, agent_meta, none}` over the selected
    runs.
- **Selection lookback.** Pass `now_ts=run["settled_ts"]` to `gather_selection_records`.
- **Worksheet.**
  - `audit.md` gets one row per sample with these columns: `#`, item display, stage,
    extractor, subject run (agent, workspace), `base(R)` (12 chars), witnesses
    (run/agent/workspace/base12/clean), in subject diff, unchanged since witness,
    `Pre-existing (Y/N)`, and `Justification`. The last two columns start empty.
  - Follow the rows with a `## KNOWN on added files` list and a `## KNOWN-but-touched`
    list. Each list entry has an empty `Disposition` field.
  - Keep the sample deterministic for a given `--seed`.

**Tests.** Build fixtures in `tmp_path`: a tiny git repo with at least two commits and a
Justfile whose `check` recipe has `tools/run_silent "<desc>"` lines, retained log files
with `✓/✗ <desc>` markers, `agent_meta.json` files that carry `run_started_at`, and a
selection dir. For the ledger, use the real core store (`tool_run_begin` / events /
finish) if it can set historical timestamps. Otherwise split `replay()` into a loader
and a `replay_runs(rows, …)` function, and feed it rows shaped exactly like
`tool_run_list` output. Use the real pure bindings (extract/classify/verdict), and mark
the tests the way the other real-binding tool tests are marked. Assert:

- A symvision item seen by a run from another workspace (attributed through
  `run_started_at`) at an ancestor base is KNOWN. The first occurrence is UNKNOWN.
- A KNOWN item in a run that also has an unrelated untracked file (for example a
  `sase_plan_x.md` scratch file) is **not** counted in `known_on_added_or_untracked`. An
  item whose locator is an untracked file in the subject is never KNOWN and, if forced
  through the metric helper, is counted.
- Sample records list the resolved witnesses. `locator_unchanged_since_witness` is true
  when the file did not change between the witness base and `base(R)`, and false when it
  did.
- A ledger `workspace` value beats the agent_meta value.
- A contaminated stage row is dropped. A missing-log run is excluded and never used as
  evidence. An unresolvable head is excluded.
- `report.json` and `audit.md` exist with the documented keys and headings, and the
  sample is identical for the same seed.

Add the missing real-binding round trips to `tests/core/test_tool_run_store.py`, as
phase 5's plan (step 4) required:

- extract a pytest item;
- record, then show;
- classify one KNOWN fixture (a witness in another workspace at the subject's base) and
  one UNKNOWN fixture;
- verdict maps a legacy `failed` exit 1 to `verification`;
- settle a failed run;
- `tool_run_failures` groups that run's signature.

If `tests/test_validate_sase_core_rs_tool.py` asserts on the binding list or the probe
output, extend it for the triage probe; otherwise leave it alone.

Update the `## Triage backtest` section of `tools/AGENTS.md`: the corrected metric, the
witness-resolved worksheet, and workspace attribution. Then regenerate its provider
shims with `sase memory init`; `sase memory init --check` must pass. Run `just fix`,
then the new tests, then `sase tool run check`. Do not run the backtest against the live
ledger in this phase, and do not change the knob constants.

## 2. continued-stage-output

In `tools/run_silent`, remember whether the helper's `decide` step returned 0
(`continued`). For a continued stage:

- print `✗ <description>` and then the captured output, exactly as for a fail-fast
  failure;
- exit 0, so the recipe proceeds.

Never print `✓` for a stage whose command failed. Keep the marker line byte-identical to
the fail-fast form (`✗ <description>` with no suffix). The backtest and any log reader
split stages on exact descriptions. The recipe-level
`✗ N stage(s) failed; continued past them (first exit C)` trailer from `--finish`
already tells the reader that the run continued. Update the header comment. Output with
no handshake variable must stay byte-identical, and the existing golden tests prove it.

Test in `tests/tool/test_keep_going.py`: a `-k` run whose failing stages echo a
distinctive line. Assert that stdout contains `✗ one` and that stage's line, does not
contain `✓ one`, and still contains `✓ two`. Keep every existing continuation assertion
(first-exit parity, records, and the no-exit-0 property).

Append a note to `sase-18j`:
`FIXED BY <this phase bead>: note #1's '✓ <stage>' for continued failures; persisting failed-stage output in the ledger remains sase-18j.6's failed-stage output capture.`
Run `just fix`, the keep-going and nested-stage tests, and `sase tool run check`.

## 3. precision-gate

Starts after both phases above have landed on master. Confirm with `git log` that the
backtest fixes are present, then run `just install` so the venv's `sase_core_rs` exposes
the triage bindings. If it still lacks them, also run `just rust-install`.

**Run.** Run
`.venv/bin/python tools/tool_triage_backtest --out-dir <scratch>/e3-gate-r1 --sample 60 --seed <N>`
with the default knobs (`min_witnesses 1`, clean witness off). It took about 2 minutes
before. Give it an explicit 15-minute timeout, or use `/sase_monitor` if the turn might
end first. Then sanity-check the report before auditing:

- `counts.workspace_attribution.none` is a small minority;
- symvision and mypy items can reach KNOWN;
- `known_on_added_or_untracked` is computed per locator.

If attribution is still mostly `none`, stop. Record a `PROPOSED FOLLOW-UP:` against the
`backtest-repair` fix and leave `sase-18j.5` open.

**Hand-audit every sampled KNOWN item (at least 50).** An item is pre-existing when its
cause exists at `base(R)` and is unchanged by R's diff. Check this against the worksheet
evidence and `git` (`git show <base>:<path>`,
`git log <witness base>..<base> -- <path>`, `git grep <base>`). Fill in `Pre-existing`
and one justification line per row. Apply these checks per extractor:

- **symvision** (unused/misused symbol): at `base(R)`, the symbol is defined at that
  path and has no non-test consumer (`git grep -w <symbol> <base> -- src ':!tests'`). If
  its only consumer at `base(R)` is in R's dirty paths, R caused it, so mark N.
- **mypy:** the offending construct exists at `base(R)` at that path, and neither that
  file nor the module defining the named type is in R's dirty paths. Otherwise
  disposition it explicitly.
- **pytest / cargo test:** the test exists at `base(R)`, and the same signature failed
  in a witness whose base is in R's ancestry. That witness must be clean, a
  selection-health full run, or dirty only in paths unrelated to the test's imports.
  None of R's dirty paths may plausibly be the cause.
- **Anything you cannot establish** counts as **N**. Never relax the audit.

Disposition every item under `KNOWN-but-touched` and `KNOWN on added files` the same
way.

**Gate (DoD-5):** all of the following must hold:

- at least 95% of the audited sample is pre-existing;
- `known_on_added_or_untracked == 0`;
- every KNOWN-but-touched item is dispositioned, and each one judged not pre-existing
  counts against the 95%.

If the gate fails:

1. Re-run with `--min-witnesses 2` (now meaningful), then add
   `--touched-requires-clean-witness`. Each round uses a new `--seed` and a fresh audit.
2. Keep every round's report and audit.
3. If no knob setting passes, leave `sase-18j.5` open. Add a `PROPOSED FOLLOW-UP:` for a
   `sase-core` rule change under a new rule version, including the evidence. For
   example, the subject wire cannot mark added paths, so the core cannot enforce "never
   KNOWN on a file the run added". Stop there.

**On a pass:**

- Set `MIN_WITNESSES` and `TOUCHED_REQUIRES_CLEAN_WITNESS` in
  `src/sase/tool/triage_inputs.py` to the passing values.
- Copy every round's `report.json` and filled `audit.md` to
  `$SASE_ARTIFACTS_DIR/triage_backtest/`.
- Register each copy with
  `sase artifact create -p <file> -l "<round, knobs, label>" --bead sase-18j.5`.

**Owner matching.** Confirm fact 8 with one real `tool_run_triage_classify` call. Use a
KNOWN fixture whose locator is `src/sase/tool/executor.py`, with candidates from
`gather_owner_candidates(<repo root>)`, and show that unrelated beads match. This call
is read-only: never mutate a bead. Then:

- append a `DISCOVERED ISSUE:` note to `sase-18j`: the core needs a token stoplist or a
  path-level match, plus a pin move, before owners render;
- append a matching note to `sase-18j.6`, because it renders owners;
- add the `PROPOSED FOLLOW-UP:` that phase 5's plan asked for.

**Notes on `sase-18j.5`:**

- the label distribution and UNKNOWN reason histogram of the passing round, plus a
  one-line summary of each failed round;
- the audit summary: N sampled, N pre-existing, %, the added-file count, and the touched
  dispositions;
- the final knob values and the artifact refs;
- the pin SHA and the workspace-recording fix;
- that note #1's 139 was the any-path metric (defect 1). Link the corrected report.

**Verify and close.**

1. Run `just fix` and `sase tool run check`.
2. Run `sase bead epic-symbols sase-18j.5`; it must print nothing.
3. As the last action before `/sase_final`, run
   `sase bead close sase-18j.5 --note "<pin, adapters, gatherers, backtest gate result and knobs>"`.
4. Check `sase agent list -a -j`: `sase-18j.6` should no longer be blocked on
   `sase-18j.5`. It may take one scheduler poll.
5. Run `sase bead work sase-18j` only if the `sase-18j.6`..`.land` runners have
   disappeared rather than still waiting, and say so in the note.
6. Never close `sase-18j`.

## Landing

This epic's land agent confirms three things before closing this epic's plan bead:

- `sase-18j.5` is closed, with a passing gate note, or remains open with an evidenced
  failed-gate follow-up;
- the `sase-18j` notes from phases 2 and 3 exist;
- the queued `sase-18j.6` agent was released.

It turns the phases' `PROPOSED FOLLOW-UP:` notes into task beads only when they are not
already carried by `sase-18j` notes.
