---
tier: epic
title: 'Task-bead backlog epic 1: fleet-wide check and landing breakage'
goal: 'The five costliest open task beads (sase-1nh, sase-1ll, sase-1p5, sase-1ic,
  sase-1cb) are fixed on master and closed. The land agent then proposes the next
  five-bead backlog epic under the same rules.

  '
decisions:
  import_budget:
    ask: How should the TUI import-budget guard count modules?
    choices:
      first_party: Budget only sase.* modules; dependency and stdlib drift stop failing
        the guard
      total_headroom: Keep the total count with ~40 modules of headroom; drift re-breaks
        it later
    default: first_party
    why: Every October re-break had no sase.* growth, which a first-party count ignores
    answer: first_party
  rebase_gate:
    ask: What should this repo's pre-push check run after a stitch rebase moves the
      base?
    choices:
      imports: AST import resolver only; seconds, catches 3 of the 4 past master breaks
      imports_and_mypy: Adds scoped mypy; catches all 4 past breaks, slower per landing
      skip: Build no gate; sase-1cb stays open
    default: imports
    why: Import breaks crash master for everyone, and the resolver costs only seconds
    answer: imports
phases:
- id: receipt
  title: Count stitch SHA receipts as successful commits (sase-1nh)
  depends_on: []
  size: small
  description: 'receipt: accept the SHA results write_result_marker actually writes,
    so passing prepared-completion monitors stop launching recovery successors; make
    the tests use production-shaped markers.'
- id: term
  title: Make rendering tests hermetic against the ambient terminal (sase-1ll)
  depends_on: []
  size: small
  description: 'term: add a session autouse conftest fixture that pins TERM and clears
    NO_COLOR/FORCE_COLOR, so the 16 TERM=dumb rendering nodes pass in agent shells.'
- id: audit
  title: Review the five workspace snapshot directory-operation sites (sase-1p5)
  depends_on: []
  size: small
  description: 'audit: review the five unreviewed workspace snapshot directory-operation
    sites, drop the stale sase-n1 baseline entry, and retire sase-p9''s baseline node
    with a fixed-at line.'
- id: import-budget
  title: Stop the TUI import-budget guard from re-breaking weekly (sase-1ic)
  depends_on: []
  size: medium
  description: 'import-budget: change what the module-count guard measures per the
    import_budget decision, align tools/tui_import_closure and the perf runbook, and
    keep the deferred probe intact.'
- id: rebase-gate
  title: Check the combined tree before pushing a rebased stitch (sase-1cb)
  depends_on: []
  size: large
  description: 'rebase-gate: add a project-configurable commit hook that runs only
    when a stitch rebase moved the base, blocks the push on real findings, and fails
    open on gate errors.'
proposed_by: bbugyi200.athena.15r
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-29 01:44:33
status: wip
bead_id: sase-1pm
---

- **PROMPT:** [prompts/202610/task_backlog_epic_1.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/task_backlog_epic_1.md)
- **BEAD:** [sase-1pm](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1pm/README.md)

# Task-Bead Backlog Epic 1: Fleet-Wide Check and Landing Breakage

## Context

A backlog triage on 2026-10-29 (master `dd59c28f44`) read all 398 non-closed task beads.
The work was split into eight category slices, and each claim was verified against the
code, git history, and focused test runs.

- **Closed 8** verified beads:
  - Obsolete: sase-1p3, sase-1pf, sase-1ol, sase-1ls, sase-1ik, sase-p9.
  - Duplicates: sase-1a8 → sase-13c, sase-1p4 → sase-1ll.
- **Owned by active epics** (leave alone, do not re-plan): sase-1pk (sase-1pj.8),
  sase-1a7 (sase-1ou.6), sase-1bw, sase-1nc, sase-1f3 (all sase-1nf).
- **Roughly 385 remain open and valid.**

This epic works the five that cost the most fleet-wide. Each one either breaks every
agent's verification, breaks Full CI, or wastes or breaks every landing:

| Bead     | Problem                                                                                                                                  | Phase           |
| -------- | ---------------------------------------------------------------------------------------------------------------------------------------- | --------------- |
| sase-1nh | Every passing prepared-completion monitor launches a pointless recovery successor (receipt matcher wants `result == "ok"`, gets a SHA)   | `receipt`       |
| sase-1ll | 16 rendering nodes fail in every agent/ToolRun shell because `TERM=dumb` leaks into tests (triaged KNOWN by 7+ agents)                   | `term`          |
| sase-1p5 | `test_artifact_directory_operation_sites_are_reviewed` is red on every Full CI Python job; a stale baseline entry hides it locally       | `audit`         |
| sase-1ic | TUI import-budget guard re-breaks about weekly (+33 corroborations); cap 3550 vs measured 3549, so there is zero headroom                | `import-budget` |
| sase-1cb | `sase stitch create` pushes auto-rebased trees nobody checked; this has broken master four times (sase-lg, sase-183, sase-1b7, sase-1cb) | `rebase-gate`   |

All five phases touch disjoint files and have no ordering constraints, so they may run
in parallel. Each phase leaves a `TASK RESOLVED: <task-id> — <evidence>` note on its own
phase bead. Phase workers do not close task beads and do not create beads. Follow-ups go
in `PROPOSED FOLLOW-UP:` notes on the phase bead.

## Phase `receipt`: count stitch SHA receipts as successful commits (sase-1nh)

`src/sase/monitor/host_completion_state.py::_marker_is_successful` requires
`marker["result"] == "ok"`. But `write_result_marker`
(`src/sase/workflows/commit/commit_tracking.py`) stores `result` as the short SHA for
commits, alongside `commit_sha`. So `_successful_receipt_repo_ids` stays empty, and host
completion recovers with `outstanding_finalizer_actions` after the stitch has already
landed.

- Make the matcher accept a marker whose `result` is the commit result
  `write_result_marker` actually writes: a SHA that is consistent with
  `commit_sha`/`sha`, or `ok`. Read `write_result_marker` and its docstring first.
  `result` is a PR URL for `create_pull_request` and a diff path for proposals, so
  decide from the marker's documented ledger fields (`commit_sha`/`commit_tree`) rather
  than from a loose "non-empty result". Failure markers must still fail.
- Fix `tests/monitor/test_monitor_host_completion.py`, which mints `"result": "ok"`
  markers production never writes. Mint production-shaped markers by calling
  `write_result_marker` (or a fixture derived from it). Add a regression that runs
  `execute_host_completion` against a production-shaped SHA marker and asserts there is
  no recovery successor.
- Check the other consumers of commit result markers
  (`grep -rn 'load_commit_results' src/`) for the same `== "ok"` assumption, and fix any
  you find in the same way.

Acceptance: the new regression passes, and the existing host-completion tests pass with
production-shaped markers.

## Phase `term`: make rendering tests hermetic against the ambient terminal (sase-1ll)

Agent and ToolRun shells export `TERM=dumb`, and some also set `NO_COLOR`. Rich then
disables color and interactivity. The 16 nodes listed in sase-1ll assume default
terminal capabilities, so they fail locally while CI stays green. One of those nodes is
`tests/main/test_repo_log.py::test_repo_log_summary_renders_external_kind`, which also
absorbs the closed sase-1p4.

- Add a session-wide autouse fixture in the shared conftest layer (`tests/conftest.py` /
  `tests/_conftest_environment.py`). It pins `TERM` to a real terminal value (for
  example `xterm-256color`) and removes `NO_COLOR` and `FORCE_COLOR` unless a test sets
  them explicitly. Tests that already `monkeypatch.setenv("TERM", ...)` must keep
  working, and so must the visual conftests under `tests/ace/tui/visual/` and
  `tests/pager/visual/`. Do not weaken any color or width assertion.
- Verify with sase-1ll's reproduction: run the listed nodes under
  `env -u NO_COLOR TERM=dumb` and under `NO_COLOR=1 TERM=dumb`, then under
  `TERM=xterm-256color`. All three must pass. Rendering tests that deliberately exercise
  `TERM=dumb` or `NO_COLOR` behavior must still set those values themselves and pass.

Acceptance: the full sase-1ll node set passes with `TERM=dumb NO_COLOR=1` in the
environment.

## Phase `audit`: review the five workspace snapshot directory-operation sites (sase-1p5)

`tests/test_agent_artifact_directory_operation_audit.py::test_artifact_directory_operation_sites_are_reviewed`
fails deterministically. The workspace checkout-snapshot adapter introduced five
unreviewed sites. Line numbers may have moved after the toobig split, so re-run the test
to get the current set diff. At triage time the five were:

- `_snapshot_restore.py:_clear_worktree`
- `_snapshot_restore.py:_restore_operation_markers`
- `_snapshot_capture_payload.py:_cleanup_capture_temps`
- `_snapshot_capture_payload_objects.py:verify_pack` (formerly `_verify_pack`)
- `_snapshot_capture_source.py:_discard_incomplete`

Fix it as follows:

- Review each site's directory operation for the hazard the audit guards against. Then
  either add the site to the reviewed set with an accurate coverage declaration (see the
  sibling `test_reviewed_dir_operation_sites_declare_coverage`), or route it through an
  already-reviewed helper. Fix any real hazard you find; do not rubber-stamp.
- In `tests/reproducible_flake_baseline.txt`, remove the stale
  `# sase-n1: post-fix dirty-record artifact-audit historical debt.` entry for this node
  once it is green, because that entry is masking this deterministic failure.
- In the same file, retire the closed sase-p9 entry
  (`tests/completion/test_install_zsh.py::test_real_zsh_zcompile_and_registration`,
  fixed by `abb80f44ac` on 2026-08-21). Replace it with a `# fixed-at:` line that names
  sase-p9 and that commit, using the file header's convention.

Acceptance: the audit test and its coverage-declaration sibling pass, and the baseline
file no longer masks either node.

## Phase `import-budget`: stop the TUI import-budget guard from re-breaking weekly (sase-1ic)

`tests/ace/tui/test_app_import_budget.py` caps `len(sys.modules)` after
`import sase.ace.tui.app`. Its history comment shows the cap was raised five times in
October with "no sase.\* module growth (stdlib/dependency)". The count depends on the
environment: release CI and local venvs measure differently. So the guard keeps
re-breaking for every agent on drift that no commit caused.

> [!decision] import_budget = first_party
>
> Keep the ratchet policy, but apply it to first-party modules. Measure the `sase.*`
> modules in the fresh-interpreter import (count `sys.modules` keys equal to `sase` or
> starting with `sase.`). Set the first-party cap at measured plus a small fixed
> headroom, under the same strict `<`. Keep the "cap only moves down / attributed raises
> only" comment policy for the new number. Third-party and stdlib growth no longer fail
> the test. If you also keep a total count, make it a much looser second bound (for
> example, total plus 10%) so that only a big dependency regression trips it.

> [!decision] import_budget = total_headroom
>
> Keep counting all modules. Re-measure on clean master, and set the cap at measured
> plus a fixed headroom that absorbs about a month of observed dependency drift (about
> 40 modules). Document in the comment that dependency drift is tolerated up to the
> headroom. Recurrence is expected when the headroom runs out.

Under either choice:

- Keep the deferred-module probe and the CPU budget unchanged. Never weaken the deferred
  module list. sase-1ai, the CPU-budget flake in the same test, stays separate.
- Make `tools/tui_import_closure` report the number the guard enforces. For
  `first_party`, the bare form and `--diff` must report first-party counts, so
  attribution matches the guard.
- Update the "TUI import-budget ratchet (sase-13p)" section of
  `docs/contribute/perf-runbook.md` to describe the new policy. Read the
  `docs_writing.md` reference memory before editing docs.

Acceptance: the guard passes on clean master. It fails when a synthetic eager import of
a new `sase.*` module is added to the closure; demonstrate this with a test or a
documented manual check. Under `first_party`, it does not fail when only third-party
module counts change.

## Phase `rebase-gate`: check the combined tree before pushing a rebased stitch (sase-1cb)

`vcs_create_commit` in `src/sase/vcs_provider/plugins/_git_commit_dispatch.py` commits,
then `_sync_with_origin_after_commit` rebases onto `origin/<default>` (including the
semantic-conflict auto-resolution), and `_push_current_branch_with_rebase_retry` may
rebase again before pushing. The configured `commit_hooks.before` hook (`just fix`) runs
before any of that. So when the rebase moves the base, nobody has checked the pushed
tree. Read sase-1cb in full for the 2026-09-28 incident and the three earlier instances.

Requirements:

- Run the gate only when a rebase in `vcs_create_commit` actually moved the base. That
  includes rebases inside the push-retry loop. A fast-forward-only or no-op sync costs
  nothing.
- Make the gate project-configurable, not hard-coded to this repo, because
  `sase stitch create` serves every project. The likely shape is a new `commit_hooks`
  phase that runs on the rebased tree before `git push`. Extend `default_config.yml`
  (default: empty, meaning no gate) and the commit-hook docs in
  `docs/reference/config/`. Configure this repo's own gate in project config
  `sase/sase.yml`. Check whether the GitHub VCS plugin (linked repo `sase-github`,
  opened with `/sase_repo`) has its own commit path that needs the same hook.
- On gate failure, do not push. Leave the commit local, record the unpushed-commit
  marker the same way the existing rebase-conflict path does, and exit with an
  actionable message and the existing `--resume` flow. The landing agent then repairs
  the semantic conflict.
- Bound the cost. The gate must take seconds, scale with the changed files on both sides
  of the rebase (local commit plus incoming upstream commits), and never become a full
  `just check`. If the check itself errors or exceeds its time budget, push anyway with
  a loud warning (fail open) so that landings never wedge on the gate. Only a real
  finding blocks the push.

> [!decision] rebase_gate = imports
>
> This repo's gate is an AST `from sase... import X` resolver. It walks every
> from-import, including function-local and `TYPE_CHECKING` ones, in the files changed
> on either side of the rebase, and resolves each name with importlib. Add an allowlist
> for the `TYPE_CHECKING`-only aliases it legitimately cannot resolve. It catches the
> ImportError class (sase-lg, sase-1b7, sase-1cb) in seconds.

> [!decision] rebase_gate = imports_and_mypy
>
> Same as `imports`, plus mypy limited to the files changed on either side, using the
> workspace's incremental cache and a hard time budget. It also catches
> renamed-attribute breaks like sase-183, but adds mypy time to every landing whose base
> moved.

> [!decision] rebase_gate = skip
>
> Build nothing. Leave a `TASK RESOLVED` note saying sase-1cb was deliberately deferred,
> with the reason, so the land agent leaves sase-1cb open.

Acceptance:

- Integration tests in temporary git repos reproduce the incident's shape: commit A
  privatizes a symbol, and commit B adds an import of the old name in a different file.
  Rebasing B onto A must be blocked from pushing, the commit must stay local, and the
  resume message must be shown. Pushes that are clean or have an unmoved base must be
  unaffected.
- The gate's added wall time on a moved-base landing in this repo is measured and
  reported in a phase note.

## Landing

The land agent verifies that each phase landed and then closes the resolved task beads.

- For each phase bead's `TASK RESOLVED` note: confirm the fix is on master and the named
  tests pass at the landing SHA. Then run
  `sase bead close <task-id> --note "<commit SHAs and verification>"`. Leave a task bead
  open, with a note explaining why, if its phase did not resolve it (for example
  `rebase_gate = skip`).
- Triage the phases' `PROPOSED FOLLOW-UP` notes through `/sase_new_task` as usual.

### Mandatory: continue the backlog chain

After closing this epic's beads, the land agent MUST author and propose the next backlog
epic. It must follow these exact rules, and must pass them on verbatim to the next
epic's land agent:

1. Re-list every non-closed task bead (`sase bead list -t task -n 0`). Close only beads
   you can verify are duplicates (resolution `superseded`, naming the canonical bead,
   with a note on the canonical) or obsolete (resolution `done`, citing the fixing
   commit or the grep that proves absence, or a passing focused test). Never close a
   bead an active epic owns. Never close a snoozed bead early. Beads that are merely
   dormant stay open.
2. Pick the 5 most important remaining task beads. Rank them by fleet-wide cost first
   (this means: breaks every agent's check or CI, breaks or wastes every landing,
   security or data-loss risk, heavy +1 corroboration), then by cost to fix.
3. Write an epic plan with at most 5 phases and at most 5 task beads. One phase may work
   several beads. Use plan decisions sparingly, but always give the owner a way to opt
   out of feature work and memory changes that might not be wanted.
4. That epic's Landing section must carry this same "continue the backlog chain" rule,
   until no non-closed task beads remain.
5. Propose it with `/sase_plan`. Proposing ends the land agent's turn, so do it last.

Seed ranking for the next epic. These are triage leads from 2026-10-29, not commitments,
and each must be re-verified before it is planned:

- **sase-163** (security): SASE Claude agents expose `ListAgents`, `SendMessage`,
  `Cron*`, and `Monitor`, and accept unaudited cross-session messages. Confirmed live.
- **sase-1p9**: a nested-preservation launch refusal poisons the workspace slot, so
  every later launch there fails within seconds.
- **sase-o7** (+24): epic close guard scans only the local Justfile, so sibling-phase
  `--epic-symbol` rows escape it.
- **sase-1cs + sase-vr**: `.[dev]` leaves Textual, ruff, and mypy unpinned. This caused
  CI-only reds and blocked the 0.18.1 release for about 7 hours.
- **Top flakes by gate impact**:
  - sase-120 (+17, 180s timeouts)
  - sase-13a (+14, zsh PTY)
  - sase-xo (+7, recurred after its "fix")
  - sase-1gz (+10, sase-core federation socket)
  - sase-1fi (49 ToolRun failures from 43 agents)
- **Safety/data**:
  - sase-1kp: `beads.db` is both the Rust lock file and Python's WAL mirror (needs a
    design decision).
  - sase-1nm: sudo re-approval reruns privileged commands.
  - sase-1p0: athena root-NVMe writes of 2–4 TB/day.
- **Stale or needs a decision**:
  - sase-j0: `in_progress` since August with no live holder; check-full test-cost
    budgets are still red.
  - sase-1ot: owner rejected the purge gate on 2026-10-28; candidate to close as
    won't-do with owner consent.
- **Likely closable on re-verification**:
  - sase-1nr: probably fixed by 79fa96dfc7.
  - sase-1ms: apollo-only stale skill.
  - sase-v4: shards gone, creator unidentified.
  - The hint/prompt-render cluster (sase-1ia, sase-1l3, sase-1hy, sase-1i9) passes on
    athena but reproduced on apollo 2026-10-27, so re-check it on apollo.
