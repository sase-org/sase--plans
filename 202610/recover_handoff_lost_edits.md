---
tier: epic
title: Recover the edits lost across epic-plan handoffs on 2026-10-27/28
goal: Every source change that a land or closeout agent made and then lost by ending
  its turn with an epic `sase plan propose` is either on master in its repository
  or verified as no longer needed. Every bead and plan that cited the lost work as
  landed tells the truth again.
decisions:
  core_stress:
    ask: Rerun the full 200-run tool_run::store stress loop before closing sase-1p6?
    choices:
      reuse: Rely on the governed check and focused tests; restored code matches the
        stress-tested copy
      rerun: Rerun the loaded 200-run stress loop; stronger evidence, but it saturates
        athena
    default: reuse
    why: The restored connection.rs is byte-identical to the stress-tested version
    answer: reuse
  tale_status:
    ask: Mark the four stranded closeout tale plans done once their work lands?
    choices:
      mark: Set them done in the bob-cli plans sidecar, as bob-cli-dw.land did for
        epic plans
      leave: Leave them wip; the bead notes alone record that the work landed
    default: mark
    why: Their beads are closed and their work will be on master, so wip would be
      false
    answer: mark
phases:
- id: core-contention
  title: Restore the lost sase-core ToolRun contention and quarantine fix (sase-1p6)
  depends_on: []
  size: small
  description: 'core-contention: apply the replayed sase-1ou.land patch to sase-core
    (contention.rs test budgets plus the connection.rs open/quarantine fix), verify
    it with the governed sase-core check, and close sase-1p6 with fresh evidence.'
- id: flake-baseline
  title: Restore the lost sase flake-baseline reconciliation
  depends_on: []
  size: medium
  description: 'flake-baseline: apply the replayed sase-1ou.land edit to tests/reproducible_flake_baseline.txt,
    rerun the selection-health flake gate, and reconcile any drift since 2026-10-28
    20:14Z.'
- id: nested-plans-validation
  title: Re-implement nested committed-plan validation read context in sase
  depends_on: []
  size: medium
  description: 'nested-plans-validation: make validate_committed_plans resolve a repo-open
    nested checkout''s own plans before the inherited SASE_SDD_PLANS_DIR, re-implementing
    the change bob-cli-dp.land--code made and lost.'
- id: heading-predicate
  title: Restore bob-cli's manual-heading-number predicate fix
  depends_on: []
  size: small
  description: 'heading-predicate: apply the recovered bob-cli-d7.land--code patch
    to bob-cli src/native/highlights_ref/create.rs (has_manual_number plus pandoc-backed
    regressions) and verify it.'
- id: rename-verifier
  title: Port bob-cli's lost theme-rename trailing-whitespace verifier fix
  depends_on: []
  size: medium
  description: 'rename-verifier: reproduce, then port onto current master, the lost
    bob-cli-dd.land--code fix that lets verify_renamed accept byte-preserved trailing
    whitespace after a Pomodoro name while still rejecting collateral edits.'
proposed_by: bbugyi200.athena.research.5f.final.f0
proposed_with: claude/opus
decided_by: reviewer
decided_via: telegram
create_time: 2026-10-28 20:02:57
status: done
bead_id: sase-1ph
---

- **PROMPT:** [prompts/202610/recover_handoff_lost_edits.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/recover_handoff_lost_edits.md)
- **BEAD:** [sase-1ph](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ph/README.md)

# Recover the edits lost across epic-plan handoffs

## Background

`sase plan propose` hands off by `SIGTERM` and is exempt from `/sase_final`, so the
host's commit finalizer never runs for the proposing turn. For an epic proposal, the
successors run in other workspaces and the proposer's workspace is later reset. Every
uncommitted edit that turn made is silently discarded. bob-cli task `bob-cli-e0` tracks
the root cause; the follow-up research report
`research:202610/staged_handoffs_through_host_completion/staged_handoffs_through_host_completion__final.md`
recommends the long-term fix. This epic does **not** change the handoff mechanism. It
only restores the work that was already lost.

A 2026-10-28 audit swept every run since 2026-10-14 that ended in an epic-plan handoff
and checked each edited path against `origin`. Results:

| Lost turn                                                   | Lost edit                                                         | Status on 2026-10-28                                                  | Phase                     |
| ----------------------------------------------------------- | ----------------------------------------------------------------- | --------------------------------------------------------------------- | ------------------------- |
| Five bob-cli backlog landers (`d7`, `da`, `dd`, `df`, `dp`) | Epic plan `status: done`                                          | **Recovered** by `bob-cli-dw.land` in plans commit `98fc47f`          | none                      |
| `bob-cli-df.land--code`                                     | sase `_RecoveryAssessment` → `RecoveryAssessment` rename          | **Recovered**: sase `196690f6cb` (`bob-cli-dw.1`) contains every hunk | none                      |
| Older chain landers (10-24 to 10-26) and tale proposers     | Various                                                           | **Committed**: their successors or land sessions committed them       | none                      |
| `sase-1ou.land`                                             | sase-core `connection.rs` + `tests/contention.rs`                 | **Lost**; `sase-1p6` reopened (READY)                                 | `core-contention`         |
| `sase-1ou.land`                                             | sase `tests/reproducible_flake_baseline.txt`                      | **Lost**                                                              | `flake-baseline`          |
| `bob-cli-dp.land--code`                                     | sase `_link_files.py`, `validate_committed_plans.py`, new test    | **Lost**; `bob-cli-dg` closed citing it                               | `nested-plans-validation` |
| `bob-cli-d7.land--code`                                     | bob-cli `src/native/highlights_ref/create.rs`                     | **Lost**; `bob-cli-d7` closed citing it                               | `heading-predicate`       |
| `bob-cli-dd.land--code`                                     | bob-cli `pomodoro_themes/rename.rs`, `tests/cli/themes_rename.rs` | **Lost**; `bob-cli-dc` closed citing it                               | `rename-verifier`         |

Recovery material was reconstructed from the provider session logs and registered as
explicit artifacts. Read each one with `sase artifact read <ref> "<reason>"`. To apply a
patch, get its file path with `sase artifact path <ref>` and pass that path to
`git apply`.

| Ref                                      | Contents                                                                                                                                                                                                    |
| ---------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `file:explicit:3e00bb097428b300e2b1c4dd` | sase-core patch, base `ed8328f9` (replayed `sase-1ou.land` edit scripts; `connection.rs` is byte-identical to the copy the agent saved before its passing guarded check `a05a43f41777782d48c2b0fdb5e1a1c2`) |
| `file:explicit:998989b3ac5a5db433b1ef6d` | sase flake-baseline patch (replayed; applies to current master)                                                                                                                                             |
| `file:explicit:5bcd6b6b3a8152c5942a2ea6` | Lost-change record for `bob-cli-dp.land--code` (its diff was not retained)                                                                                                                                  |
| `file:explicit:a2e93efe2ce93278fbd40f8a` | bob-cli `create.rs` patch, applies to bob-cli master `74874cb5` (exact hunks from the muse session)                                                                                                         |
| `file:explicit:c1d60567687a710ec8a65a9c` | bob-cli rename-verifier **reference** patch, base `04211926`; it no longer applies because `50f6c91b` and later commits refactored `rename.rs`                                                              |

## Rules for every agent in this epic

- **Never end a turn with `sase plan propose`, and never hand off while edits are
  uncommitted.** That is exactly how this work was lost. Finish with `/sase_final`. A
  `/sase_monitor` continuation is fine, because it carries the dirty tree.
- Open sase-core and bob-cli with `/sase_repo` (`sase repo open sase-core` and
  `sase repo open bob-cli`), use only the printed paths, and read each repo's
  `AGENTS.md`. Opened repos become commit obligations of your turn.
- Fetch first. If a recovered patch no longer applies because master moved, rebase it by
  hand (preserving its intent) and say so in your bead note. If master already contains
  an equivalent fix, record that with the commit SHA instead of re-landing.
- Run checks only through `sase tool run check` in the changed repo. Never run
  `just check-full`. Read sase's `lint_and_test.md` before finishing sase changes.
- Bead commands: sase beads (`sase-1p6`, `sase-1ou`) use the default project; bob-cli
  beads need `-P bob-cli`. Use audited `sase bead read`.

## Phase core-contention

In sase-core, apply `file:explicit:3e00bb097428b300e2b1c4dd`. It changes only:

- `crates/sase_core/src/tool_run/store/connection.rs`. Every call is owed one open, a
  quarantine is owed one recreate, and the WAL conversion attempts once before honoring
  its deadline. Before this fix, a call whose budget was spent before its first open, or
  whose corrupt-store quarantine outlasted its budget, returned `Busy` with no
  contender: a product regression from `6d11122e`.
- `crates/sase_core/src/tool_run/store/tests/contention.rs`. Release schedules use an
  explicit 10 s `RELEASE_BUDGET` and release only after the contender thread starts.
  Adds `zero_budget_waits_the_core_default_before_busy`,
  `spent_budget_still_owes_the_first_open`, and
  `slow_quarantine_still_recreates_the_store`. Drops a redundant wall-clock bound from
  `wal_reserved_writer_coexists_with_readonly_show`.

Confirm that the two new regression tests fail with `connection.rs` reverted and pass
with it restored. Then run the focused `tool_run::store` tests and the governed
sase-core `sase tool run check`.

> [!decision] core_stress = rerun Before closing, also rerun the loaded stress loop that
> `sase-1ou.land` recorded on `sase-1p6` note #1: 200 runs of the `tool_run::store`
> tests at `--test-threads=64`, 24 parallel processes, 96 CPU spinners plus 4 dsync
> writers. Hand it to `/sase_monitor`. Record failures by test name.

Do not move sase's `sase-core-revision.txt` pin: sase calls no new binding, and the
scheduled ratchet picks the commit up. Close `sase-1p6` with `sase bead close` and a
note naming the sase-core commit and check witness. Its earlier close evidence was
invalidated because the edits never landed.

## Phase flake-baseline

In sase, apply `file:explicit:998989b3ac5a5db433b1ef6d` to
`tests/reproducible_flake_baseline.txt` (the file has not changed on master since
2026-10-27). It records the `sase-1ou` landing reconciliation:

- Re-keys parked nodes to their real owners: `sase-1fi` (fast_success), `sase-1p2`
  (group-kill), `sase-1al` (panel semicolon), `sase-19m` (agy usage probe), `sase-xo`
  (bounded drain), `sase-1pc` (doc-phrase audits), `sase-1pd` (8 unowned TUI nodes), and
  `sase-1pe` (9 unowned nodes).
- Adds `fixed-at` directives for the capacity-scan nodes (`e7d9f7a479`, `f83711219c`).
  Corrects the `sase-12f` directive to the real `a42eedc1c4` landing instant.
- Drops the demand_runs, executor, and known_continuation nodes (closed owners postdate
  their failures) and the superseded `sase-165.5` directive.

Verify each named bead exists and is still the right owner. Then run
`.venv/bin/python tools/selection_health --fail-on-new-flake` and
`tests/test_selection_health_tool.py`. At the time of the lost turn, the gate went from
exit 2 with 41 exceedances to exit 1 with 20. Reconcile anything that changed since
then, following the file's header rules (every added node needs a filed owner bead). Do
not weaken the gate to make it pass. Report the final exit code and exceedance count.

## Phase nested-plans-validation

On sase master,
`src/sase/scripts/validate_committed_plans.py::_resolve_committed_plans_root` returns
`SASE_SDD_PLANS_DIR` before considering checkout identity. When an agent from another
project (for example bob-cli) runs sase's `just check` inside a repo-open external sase
checkout, it inherits the host's plans dir. The committed-plan stage then scans the
host's plans instead of the checkout's own provisioned plans sidecar. `7531093713` added
the nested sidecar provisioning; this integration was the remaining piece of
`bob-cli-dg` and was lost.

Read `file:explicit:5bcd6b6b3a8152c5942a2ea6` for the lost implementation's shape, then
re-implement it:

- In `src/sase/sdd/_link_files.py`, expose narrow public helpers over the existing
  nested repo-open checkout detection: the lost version had
  `nested_repo_open_checkout_root`, `nested_repo_open_sdd_root`,
  `nested_repo_open_plans_root`, and `nested_repo_open_plans_remedy`. Reuse the
  detection; do not copy marker logic or import private symbols. Keep every new public
  helper used so no epic-symbol whitelist entry is needed.
- `_resolve_committed_plans_root` checks for a nested checkout first, including from its
  subdirectories, and uses that checkout's own materialized plans record even when an
  inherited host `SASE_SDD_PLANS_DIR` is set. An unprovisioned nested checkout fails
  `invalid-plans-root` with an actionable `sase repo open` remedy. It never falls back
  to the host's plans and never skips the stage.
- Outside nested checkouts, preserve current behavior exactly: explicit env overrides,
  primary checkouts, numbered workspaces, and sidecar clones. Do not change
  `resolve_sdd_store`, `get_primary_workspace_dir`, or any bead/plan write path.

Add regressions in `tests/sdd_store/test_nested_repo_open_sdd_root.py` using temporary
repos, reusing the existing nested-sidecar fixtures. Cover:

- a broken external plan fails even when valid host plans are named in the env;
- broken host plans cannot fail a valid external scan;
- external and linked kinds;
- unset env and a subdirectory cwd;
- the unprovisioned fail-closed path through `main()`;
- unchanged non-nested resolution.

Assert the selected root and the real validation outcome. Run the B5 suites
`tests/main/test_repo_open_external_plans_sidecar.py` and
`tests/sdd_store/test_nested_repo_open_sdd_root.py`, then `sase tool run check`.

## Phase heading-predicate

In bob-cli, apply `file:explicit:a2e93efe2ce93278fbd40f8a`. It changes the embedded Lua
`has_manual_number` in `src/native/highlights_ref/create.rs`. Master still accepts a
bare `1.` and malformed `2.3.. Topic`, which unnumbers every heading in the document.
The approved grammar has two forms:

- a leading integer plus `.` or `)` followed by whitespace;
- a dotted sequence of complete integer components (no empty component) with at most one
  trailing `.` or `)`, followed by whitespace.

A bare year such as `2026 roadmap` is not a prefix. The patch also adds pandoc-backed
regressions beside `section_numbering_self_numbered_doc_prints_author_numbers_only`:

- negative cases: bare `1.` and `1)`, `2.3.. Topic`, `2..3 Topic`;
- positive cases: `1. Evidence`, `10) Risks`, `2.3 Details`, `2.3. Details`,
  `2.3) Details`;
- a numbered lone H1 and an explicitly unlisted numbered heading, neither of which
  triggers the mode alone.

Confirm the new negative tests fail without the predicate change and pass with it, using
a real `pandoc` on athena. Then run bob-cli's `sase tool run check`.

## Phase rename-verifier

In bob-cli, `verify_renamed` (`src/native/pomodoro_themes/rename.rs`, about line 1388)
still compares `capture_pomodoros::without_name_tail(before)` with `(after)`.
`without_name_tail` strips a parsed, `trim_end`ed body as a physical suffix. On a row
whose theme name is followed by spaces or tabs, it returns the whole line, so
verification rejects an intended byte-preserving rename: `verify_failed`, then rollback.
Rows with no trailing whitespace pass. This gap was the remaining acceptance for
`bob-cli-dd.3` and the reason `bob-cli-dc` was reopened.

1. Reproduce first on current master. Port the reference patch's tests
   (`file:explicit:c1d60567687a710ec8a65a9c`) onto master:
   - the CLI test
     `themes_rename_preserves_trailing_whitespace_and_mixed_endings_offline`: open and
     completed `JOB` rows with trailing spaces or tabs, mixed CRLF/LF, and a changed
     final row without a newline; one commit with the expected subject; one journal
     line;
   - the verifier unit tests built on a `verifier_fixture` helper, covering suffix
     whitespace accepted and collateral or terminator rewrites rejected.
2. If they fail, port the fix. `verify_renamed` re-derives the expected after-image with
   `rewrite_file_contents` and compares it with the reread bytes, and rejects mismatched
   selected tails. Since `50f6c91b`, rename and split share `verify_raw_tail_splice`.
   Integrate there rather than reintroducing a private copy.
3. `split.rs` (about line 1103) uses the same `without_name_tail` comparison. Check
   whether `bob themes split` has the same trailing-whitespace rejection. If it does,
   fix it in this phase with a matching regression, because it is the same defect in the
   shared verifier. Otherwise, record that it does not.
4. If master already passes the ported tests, land only the regressions and record which
   commit fixed the behavior.

Run `cargo test --test cli themes_rename` and `themes_split`, then bob-cli's
`sase tool run check`.

## Landing

The land agent verifies each phase's commits on its repository's `origin`, then makes
the records truthful:

- Add a note to `sase-1ou` naming the sase-core and baseline commits. That resolves its
  notes #3 and #4 blockers, so its parent lander can re-verify on master and proceed. Do
  not close `sase-1ou`; its own land agent owns that.
- Add notes to `bob-cli-dg` (nested validation commit), `bob-cli-d7` (predicate commit),
  and `bob-cli-dc` and `bob-cli-dd` (rename verifier commit, or the master commit that
  already fixed it). Each note says that the earlier close note's claim was lost at an
  epic handoff and is now true on master.
- Add one note to `bob-cli-e0` with the full loss inventory above, as corroborating
  evidence for the root cause. Do not close it: it tracks the prevention fix.

> [!decision] tale_status = mark In the bob-cli plans sidecar, set `status: done` on
> `202610/d7_heading_prefix_closeout.md`, `202610/backlog_b3_rename_closeout.md`,
> `202610/finish_backlog_b4.md`, and `202610/b5_validation_closeout.md`. Their coders
> each ended with an epic proposal, so the host never recorded them. Change nothing else
> in those files.

Do not restart the paused bob-cli backlog chain (B7) or propose any plan. End with
`/sase_final`.
