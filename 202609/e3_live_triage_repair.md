---
tier: epic
title: 'Finish E3: make live failure triage actually run, fix owner matching, and
  prove it on athena'
goal: 'An agent''s `sase tool run check` on athena records a triage for every settled
  named run and continues past all-KNOWN/FLAKY stages to `test (scoped)`. Possible
  owners are suggested only when a bead really names the failing file. The E3 landing
  criteria are proven live, so the E3 land agent can close `sase-18j`.

  '
parent_bead: sase-18j
phases:
- id: triage-inputs
  title: Send wire-valid evidence, store triage diagnostics, and clear the E3 stragglers
  depends_on: []
  size: medium
  description: 'triage-inputs: make the selection-record gatherer emit wire-valid
    evidence (failures extracted into items) for both settle and the mid-run stage
    verb. Persist dropped-input and failure diagnostics. Privatize stage_decision,
    fix the stopd marker, and fix the linked-repo monitor lookup. Add real-binding
    and smoke regressions that feed a real-shaped full-run selection record.'
- id: core-owner-match
  title: Match possible owners on file identity, not on shared path tokens
  depends_on: []
  size: medium
  description: 'core-owner-match: in sase-core only, replace the any-3-char-token
    substring owner match with a path-level match against a candidate''s location
    plus a guarded file-name title match. Record what matched, and add the sase-191.3
    probe as a regression fixture.'
- id: owner-pin
  title: Pin the owner-matching core and verify owners on live candidates
  depends_on:
  - core-owner-match
  - triage-inputs
  size: small
  description: 'owner-pin: move sase-core-revision.txt to a pushed sase-core commit
    that contains the new owner matcher, add a real-binding owner round trip, and
    re-run the owner probe against the live bead candidates.'
- id: live-acceptance
  title: Prove E3's landing criteria live on athena
  depends_on:
  - owner-pin
  size: medium
  description: 'live-acceptance: run the live athena acceptance, re-run and hand-audit
    the precision backtest, cross-check sase tool failures against the ledger, confirm
    the check digest, measure the post-landing baseline, and record the DoD checklist
    on sase-18j.'
proposed_by: bbugyi200.athena.sase-18j.land
create_time: 2026-09-25 19:07:39
status: wip
bead_id: sase-18j.10
---

- **PROMPT:** [prompts/202609/e3_live_triage_repair.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/e3_live_triage_repair.md)
- **PARENT:** [202609/tool_e3_failure_triage.md](https://github.com/sase-org/sase--plans/blob/main/202609/tool_e3_failure_triage.md)
- **BEAD:** [sase-18j.10](https://github.com/sase-org/sase--beads/blob/main/pages/sase-18j/sase-18j.10.md)

# Plan: finish E3 so failure triage runs in production

## Why this epic exists

E3 (`sase-18j`, `plan:202609/tool_e3_failure_triage.md`) has all nine phases closed. Its
commits are on master (`290cd1aa7c` … `49c32e19ec`), and the `tool_failure_triage` flag
is removed (flag bead `sase-19a` closed). The E3 land agent ran the live acceptance
before closing and found that E3's two agent-visible behaviors do not work on athena.
The E3 plan remains the authority for every rule, wire, and landing criterion. This epic
only finishes what E3 left broken. It does not redo landed work and never closes
`sase-18j`: that bead's land agent resumes the landing after this epic lands.

Facts checked at master `7512f4e9a1` with core pin `d64520bbf6`. Recheck each one before
relying on it (see `sase bead read sase-18j`, note #7):

1. **Selection evidence is not wire-valid.** `gather_selection_records` in
   `src/sase/tool/triage_inputs.py` emits a `failures` key on every witness record.
   `ToolRunTriageEvidenceRunWire` is `deny_unknown_fields`, so the core refuses the
   whole request. It fails with
   `unknown field 'failures', expected one of run_id, project, ..., items, selection_source`,
   in both `tool_run_triage_stage` and `tool_run_triage_settle`. Only
   `tools/tool_triage_backtest` converts `failures` into `items` (`_selection_items`,
   through `tool_run_triage_extract`), and production code must not import `tools/`.
2. **Consequences, live.** Run `f1c9ad8815926dac689f98882dc4c192` was
   `sase tool run check` from an agent on clean master. Its symvision stage recorded
   `stopped mode=known reason=helper_error` after 674 ms. Settle printed
   `triage unavailable: ... unknown field failures`, and `show -j` reports
   `triaged=false`, `verdict_reason=not_triaged`. `sase tool _triage-stage` run by hand
   against that run reproduces the refusal. 25 of the 27 named `check` runs recorded
   since `71b25fbf4e` were never triaged. The two that were got through only because
   their selection gatherer returned nothing. The athena store has 345 full-run records
   within the 7-day lookback, and about half of all records carry `failures`.
3. **Why tests missed it.** Every triage test and the smoke group run in an isolated
   `SASE_HOME` with no `test-selection` records, so the gatherer returns `[]`. No test
   feeds a real-shaped full-run record through the real binding.
4. **Diagnostics are not stored.** `_settle_failure_triage` in
   `src/sase/tool/executor.py` gives each gatherer a 1.0 s slice
   (`_TRIAGE_GATHERER_SECONDS`) inside a 5 s budget (`_TRIAGE_BUDGET_SECONDS`). A slow
   gatherer is dropped, which is the correct fail-open behavior. But its diagnostic,
   like a settle exception, only reaches the footer's `triage unavailable:` line and is
   never stored. So `show -j` cannot explain why a run is untriaged or thinly evidenced.
   E3's plan requires "store a diagnostic". Measured in fresh processes at load1 ≈ 17,
   owner candidates took 0.60 s (256 live candidates) and selection records 0.05 s.
   Owners have little headroom under heavier load.
5. **Master is red at `lint (symvision)`**: `stage_decision` in
   `src/sase/tool/triage_stage.py` is unused-public. Only its own file and tests call
   it.
6. **Owner matching over-matches** (`sase-18j` note #5, from `sase-191.3`). In
   `crates/sase_core/src/tool_run/triage/classify.rs` in `sase-core`, `locator_tokens`
   keeps every token of at least 3 characters (`src`, `sase`, `tool`, `tests`), and
   `match_owners` substring-tests them against the candidate's bead id, location, and
   title. Every `sase-*` bead id contains `sase`. So any item under `src/sase/`
   "matches" the first two open `ci`/`flake`/`bug` beads by id order. Example:
   `sase-106` (gate_shell handoff) and `sase-10a` (gateway fleet CI) for
   `src/sase/tool/executor.py`.
7. **Smaller E3 leftovers.**
   - `triage_display._stage_line` renders the stop marker as `stopd` (`f" {value}d"`);
     from `sase-18j.7` note #3.
   - `query._replay_monitor_log` and `control.monitor_output_path` look up the owning
     monitor with `list_monitors(project=run.project)`. A run recorded under a linked
     repo's identity (E3 phase `ledger-hygiene`), owned by a monitor filed under the
     host project, therefore misses. From `sase-18j.1` note #2.

## Constraints carried from E3

- Triage never changes an exit code or blocks a run. A missing input can only produce
  fewer KNOWN or FLAKY labels, never more
  (`decisions:triage-annotates-does-not-change-exit-codes`).
- Wires stay `schema_version: 1`, requests stay `deny_unknown_fields`, and changes are
  additive. No `ToolDefinitionWire` or catalog change, so the sase `check` digest must
  not move.
- Classification logic is pure Rust in `sase-core` (`rust_core_backend_boundary`).
  Python only gathers bounded inputs. Production code never imports from `tools/` or
  `tests/`.
- Pin discipline (E3 decision 10): a core phase changes only `sase-core`. The sase phase
  that needs the new core behavior moves `sase-core-revision.txt` to a pushed
  `sase-core` commit that contains it. Commits and pushes are host-owned.
- Knobs stay `MIN_WITNESSES=1` and `TOUCHED_REQUIRES_CLEAN_WITNESS=False` unless the
  re-run backtest fails its gate.
- Phases that touch `sase-core` open it with `/sase_repo`, read its `AGENTS.md`, and
  verify with `sase tool run check` from inside that checkout. Never run bare `cargo`.
  Use generous explicit timeouts, or `/sase_monitor` for anything that may outrun the
  turn.

## 1. triage-inputs

**Wire-valid selection evidence.**

- Move the `failures` → `items` conversion into `src/sase/tool/triage_inputs.py`, so
  that `gather_selection_records` returns records that validate as
  `ToolRunTriageEvidenceRunWire`. It extracts each record's failures with the pure
  `tool_run_triage_extract` binding, exactly as `tools/tool_triage_backtest`'s
  `_selection_items` does today: pytest `FAILED <node>` lines, stage key
  `test (scoped)`, and the catalog project root for normalization.
- The gatherer gains whatever root argument extraction needs. Records without failures
  get `items: []`. An extraction error drops only that record's items and adds a
  diagnostic.
- Keep the gatherer bounded (the newest 500 records within the lookback). Skip files
  whose mtime predates the lookback before parsing them.
- Make `tools/tool_triage_backtest` reuse the production conversion instead of its own
  copy. Its pytest twin must stay green.
- Both callers get the fix: the settle path (`_settle_failure_triage`) and the mid-run
  verb (`src/sase/tool/triage_stage.py::_gather_stage_inputs`).
- Audit the other gatherers' outputs against their wires in the same way: ancestry,
  flake baseline entries, owner candidates, and the settle request's own fields.

**Stored diagnostics.**

- When a settle-time gatherer is dropped (timeout, failure, malformed evidence, or
  budget exhausted), or the settle call raises or is refused, persist those diagnostics
  on the run's triage run facts through `tool_run_triage_record`. The E3 core already
  stores run-facts `diagnostics`; use it if present and do not add wire fields for this.
  `sase tool show RUN` and `show -j` then explain an untriaged or thin-evidence run.
- Recording stays fail-open and inside the existing 5 s budget.
- Re-measure the owner-candidate gatherer cold under athena load. If it routinely misses
  its 1.0 s slice, rebalance the per-gatherer slices within the unchanged 5 s total, or
  make the candidate read cheaper. Do not raise the total budget. Record the measurement
  in the phase note.

**E3 stragglers.**

- Rename `stage_decision` to a private helper and update its tests. That clears the
  master-red symvision item. Do not whitelist it.
- `_stage_line` renders `continued` / `stopped`.
- `_replay_monitor_log` and `monitor_output_path` retry `list_monitors()` unscoped when
  the project-scoped lookup finds no record, and resolve the id from that list.

**Regression tests (use the real binding, not mocks):**

- A real-shaped schema-2 `kind: full-run` record with `failures`, written under an
  isolated `SASE_HOME`'s `test-selection/<project_key>/` (or
  `SASE_TEST_SELECTION_HEALTH_DIR`), flows through `gather_selection_records` into
  `tool_run_triage_settle` and `tool_run_triage_stage` with no refusal. Its pytest
  failures appear as witness items. A matching pytest item with that witness is labeled
  KNOWN, within the rule's other conditions.
- `sase tool _triage-stage` answers `continue` for an all-KNOWN stage while such a
  record directory is present, and `stop` for NEW or UNKNOWN items. This is the
  known-gated path that was dead in production.
- A gatherer that times out, and a settle call that raises, each leave a stored
  diagnostic visible in `show -j`. The exit code is unchanged.
- The `stopped` marker, and the linked-repo monitor-lookup fallback.
- Add a case to the E3 triage smoke group (`tools/_smoke_tool_runs_cases_triage.py` and
  its pytest twin) that seeds a full-run selection record in the isolated `SASE_HOME`,
  then asserts the run is triaged and the all-KNOWN stage continues. Map it to DoD-7 and
  DoD-8.

**Verify:**

- `sase tool run check` on this phase's tree. Record the run id.
- `sase tool show RUN -j` must show `triaged: true`, and every stage decision's reason
  must be something other than `helper_error`.
- If master still has a witnessed lint failure, continuation should reach
  `test (scoped)`. Say which applied.

## 2. core-owner-match

Work only in `sase-core`, in `crates/sase_core/src/tool_run/triage/classify.rs` and its
tests. Class and verdict logic do not change.

- A candidate is a **possible owner** of an item only when one of these holds:
  1. **Location match.** One of the candidate's `location` paths equals one of the
     item's repo-relative locator paths, or is a directory prefix of it. Normalize the
     candidate's `location` first: split `A and B` / comma lists, strip a pytest
     `::node` suffix, line and column refs, and a `(lines …)` note.
  2. **Title match.** The candidate's title contains one of the item's full locator
     paths. Or it contains the locator's file name (with extension, e.g. `executor.py`)
     as a whole word, and that file name is not generic: `__init__.py`, `conftest.py`,
     `mod.rs`, `lib.rs`, `main.rs`, `main.py`, `Justfile`, and similar, as a small named
     constant.
- Never match on the bead id, on directory tokens alone, or on sub-word fragments.
- Keep "at most two", open ones before recently closed ones, and deterministic ordering.
- Add `matched_on` (`location` or `title`) to each possible-owner JSON object. That
  object is opaque on the wire. Readers ignore unknown keys, and stored labels keep
  their old owners.
- Tests:
  - the `sase-191.3` probe as a fixture: locator `src/sase/tool/executor.py` against
    candidates shaped like `sase-106` / `sase-10a` with no path in location or title
    yields no owners;
  - positive location and title cases, including a pytest node-id location;
  - the generic-name stoplist;
  - closed-within-lookback "possibly fixed";
  - the existing shuffle property test stays byte-identical.
  - Update any golden `classify` / `settle` fixtures whose owners legitimately change.
- Verify with `sase tool run check` inside the `sase-core` checkout. Record the
  pushed-commit expectation for the pin in this phase's bead note.

## 3. owner-pin

- Move `sase-core-revision.txt` to a pushed `sase-core` commit that contains phase 2.
  Confirm that it is reachable from the linked checkout's `origin/master`, and run
  `just install` so the workspace binding matches.
- Update `tools/validate_sase_core_rs`, `tools/check_sase_core_rs_bindings`, and
  `tools/smoke_sase_core_rs_tool_runs` only if a binding surface moved. Do not touch the
  published `sase-core-rs` window in `pyproject.toml`.
- Add a real-binding round trip in `tests/core/test_tool_run_store.py`. A settle with
  one location-matching candidate and one bead-id-only candidate stores exactly the
  former, with `matched_on`.
- Live probe (read-only). Classify fixture items at `src/sase/tool/executor.py`,
  `src/sase/tool/triage_stage.py`, and a real failing test node against
  `gather_owner_candidates()` on athena. Record the owner count distribution, and
  confirm that no owner is suggested without a path or file-name hit.
- Verify with `sase tool run check`.

## 4. live-acceptance

This is verification and evidence. Fix only defects that it proves were caused by E3 or
by this epic.

- **Live athena acceptance.** Run `sase tool run check` from an agent on the day's
  master. Record:
  - the run id, the footer's triage block, and the verdict line;
  - each stage decision with its reason and elapsed time;
  - whether continuation reached `test (scoped)` behind master-red lint stages;
  - `continuation_extra_ms`.

  If master is green, or its red items have no witness yet, say so and rely on the
  fixtures and the smoke group. Never fabricate a red stage.

- **Ledger health since phase 1 landed.** Across settled named runs on athena, report:
  - the share with `triaged: true`;
  - the stored diagnostics histogram;
  - the stage-decision reason histogram (`helper_error` should be rare and explained).
- **DoD-9.** Record `sase tool failures -j` beside a direct query of the same groups,
  run through the pinned binding. Linked-repo groups must stay out of sase's `check`.
- **DoD-12.** Record the sase `check` definition digest from `sase tool list -j`.
  `sase-18j.9` reported `12b1748a5cb76c1f…`. Explain any difference by the catalog's own
  history; E3 must not have moved it.
- **DoD-5 re-run.**
  - Re-run `tools/tool_triage_backtest` on athena with the final knobs, through
    `/sase_monitor` (inline attempts outran the turn in `sase-18j.9`).
  - Hand-audit a fresh seeded sample of at least 50 KNOWN labels, one justification line
    each. Gate: at least 95% pre-existing, zero KNOWN on added files, and every
    KNOWN-but-touched item dispositioned.
  - Register the report and the filled audit with `sase artifact create --bead` on this
    phase's bead.
  - If the gate fails, stop and record it. Never relax the audit.
- **Post-landing measurement baseline**, over the window since phase 1 landed (state the
  window and the run count):
  - the share of failed `check` runs that reached `test (scoped)` (11.5% before E3);
  - the share of agents whose final run executed tests (24% before E3);
  - extra continuation time per day.
- **Record on `sase-18j`** a single note with:
  - the DoD-0 to DoD-14 checklist, each with its evidence;
  - the measurements;
  - `PROPOSED FOLLOW-UP:` lines for the E3 reopen triggers (the `rerun` trigger at about
    2 h/week of FLAKY re-runs; the CI-evidence trigger at more than about 40% UNKNOWN)
    and for the E4 re-scope decision (§7 Q1 of the landing-criteria report).

  Do not close `sase-18j`.

## Out of scope

- Specific extractors for `lint (pyscripts)`, `lint (feature flags)`,
  `lint (test waits)`, and `SASE validation`: task `sase-19s`.
- ToolRun store busy flakes: task `sase-18t`.
- The stale `phase-pending` sentence in `tools/AGENTS.md`: task `sase-148`.
- E4 receipts and completion policy, and E5's TUI Failures view.
