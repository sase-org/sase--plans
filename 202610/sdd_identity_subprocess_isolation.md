---
tier: tale
title: Isolate the empty-home SDD identity subprocess
goal:
  Eliminate the full-lane SDD identity subprocess timeout while preserving real Git
  assertions and cleanup guards.
size: medium
proposed_by: bbugyi200.athena.sase-1pu.4
proposed_with: codex/gpt-6-astra
bead: sase-1pu.4
create_time: 2026-10-29 20:16:58
status: wip
---

- **PARENT:**
  [202610/task_backlog_epic_3.md](https://github.com/sase-org/sase--plans/blob/main/202610/task_backlog_epic_3.md)
- **BEAD:**
  [sase-1pu.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1pu/sase-1pu.4.md)

# Isolate the empty-home SDD identity subprocess

Complete phase `sase-1pu.4` of `plan:202610/task_backlog_epic_3.md`, addressing the
timeout tracked by `sase-120`. Keep the real nested pytest invocation and its Git
identity assertions, isolate its temporary files and cleanup, and verify the result in
the governed full test lane.

## Scope and prior work

Read `sase bead read sase-1pu.4 -r "Need current phase notes and decisions"` and
`sase bead read sase-120 -r "Need timeout evidence"` before implementing. The phase was
in progress with no previous work notes when this plan was authored. There is no
recorded remainder requiring a child epic. This is one bounded test-harness
investigation and fix, suitable for one medium coding agent.

The parent's final `optional_scope = keep_five_seeds` decision applies. This tale
implements only the SDD identity flake. It authorizes no memory changes, adjacent
features, or closure of the seed task or any ancestor. Record discoveries as
`PROPOSED FOLLOW-UP:` notes on `sase-1pu.4`; do not create task beads.

The relevant earlier work has already landed:

- `8989d0a724` removed `SASE_PYTEST_TMP_REDIRECTED` from the nested environment and set
  `SASE_TMP_LEAK_GUARD_DISABLED=1`, preventing the child's session leak guard from
  attributing sibling scratch to this test. Preserve this protection.
- `f1c20a9f6b` disabled automatic Git maintenance through the shared identity fixture
  and added a trace regression. Preserve it and its assertions.
- Retained ToolRun `e996bbfb6e8d632139c7686cc9268d44` still failed this exact node with
  `subprocess.TimeoutExpired` at 180.12 seconds after those changes. Its full scoped
  lane recorded 56,500 passes and one failure in 827.07 seconds. Earlier evidence on
  `sase-120` also records a successful inner test before the parent subprocess timed
  out. Historical evidence is not a fresh reproduction.

## Diagnosis to confirm

Start with `tests/sdd/test_git_identity_fixture.py`. Its child receives inherited
`TMPDIR` and possibly `PYTEST_DEBUG_TEMPROOT`, and its argv has no `--basetemp`.
`tests/conftest.py::_configure_test_git_identity` uses `tmp_path_factory`, so the child
initializes pytest's default numbered temporary directory machinery.

The installed pytest implementation in `_pytest/tmpdir.py` and `_pytest/pathlib.py`
registers cleanup of old `pytest-*` and `garbage-*` directories under that shared root.
An explicit `--basetemp` avoids that numbered-directory cleanup. This is the leading
explanation for load-sensitive shutdown work; it is not yet a stack-confirmed
explanation of every recorded timeout. Disabling SASE's leak guard does not disable
pytest's own retention cleanup.

Inspect the existing temp and checkout guards, session fixtures, and suite-gate hooks
only as needed to separate test execution, teardown, process exit, and guard failure.
Keep the change local to this test module unless a captured failure proves a shared
helper must change. This is test infrastructure, not shared product behavior requiring a
Rust backend change.

## Implementation

1. Establish evidence on the unchanged tree. Run the exact node and its entire file,
   retain durations and exit status, and attempt the reported full-lane failure with the
   normal governed `just test` recipe. Read `/sase_monitor` before starting long
   commands and use its handoff for the full run. Do not repeatedly launch full suites
   hoping for a flake. If it does not reproduce, report that honestly and use the
   historical timeout plus the deterministic isolation regression below. Capture the
   child stack during any reproduced stall before its existing timeout, using temporary
   diagnostic instrumentation within the checkout's disposable scratch, to distinguish
   cleanup from imports, fixtures, or process shutdown. Remove diagnostic
   instrumentation from the final diff. Avoid inspecting or deleting real host or
   sibling-workspace scratch.
2. Give the nested process an explicitly owned temporary base beneath the outer test's
   `tmp_path`, passed through `--basetemp`. Use a dedicated child directory, never the
   outer base, because pytest clears an explicitly supplied base at startup. Redirect
   the child's general temporary-file environment to a separate private directory under
   the same outer `tmp_path`; account for `PYTEST_DEBUG_TEMPROOT` so inherited overrides
   cannot restore shared cleanup. Leave cleanup ownership with the outer pytest fixture,
   including the existing failure-retention behavior. Keep all parent environment
   changes reversible.
3. Retain empty `HOME` and `XDG_CONFIG_HOME`, the blank initial system Git config,
   removal of inherited `GIT_*` identity/config variables, the real
   `test_test_created_repo_can_commit_without_local_identity` target, captured
   stdout/stderr on failure, and the existing 180-second timeout. Keep the current
   nested leak-guard exemption local to the child; never disable the outer guard or
   broaden its ignore list. Add a short explanation of the cleanup ownership. If the
   captured stack instead proves another local cause, fix that cause and retain the same
   isolation and assertion contracts; do not claim an unrelated temp-path change
   resolved an observed stall.
4. Add a deterministic behavioral regression using only synthetic scratch under
   `tmp_path`. Seed an inherited pytest temp root with an old numbered directory and a
   marker that default numbered-directory cleanup would remove. Exercise the actual
   nested-launch path and prove the unrelated marker survives and the child uses only
   its own base. Cover an inherited `PYTEST_DEBUG_TEMPROOT` override as well as the
   ordinary temp-root case without multiplying expensive child runs unnecessarily.
   Demonstrate that removing the isolation change makes this regression fail. Prefer
   extending the existing integration test or a small shared launcher within this module
   over a separate framework. Do not assert a fragile elapsed-time threshold, add
   sleeps, mock away Git/pytest, or merely assert that an argv list contains
   `--basetemp`.

## Verification and acceptance

Read `lint_and_test.md` through `/sase_memory_read` before verification. If changes
under `tools/` prove necessary, read `tools.md` first; those changes are not the
expected solution. Keep global pytest retention settings and timeouts unchanged.

Run these focused checks against the final tree:

```bash
.venv/bin/python -m pytest tests/sdd/test_git_identity_fixture.py::test_sdd_git_identity_survives_empty_home_subprocess -q
.venv/bin/python -m pytest tests/sdd/test_git_identity_fixture.py -q
```

Run the file through `just test tests/sdd/test_git_identity_fixture.py` to exercise the
governed runner and its temp guard. Use the runner's worker controls rather than
bypassing the host token pool. Run existing temp-guard tests if guard-related code
changes; retain a useful negative control that proves a real leak still fails.

Run `just fix`, then `sase tool run check`. Complete a resulting-tree full governed test
lane with `just test` if the check did not already escalate to the full lane. Use
`/sase_monitor` for commands that will outlast a synchronous turn, and retain its result
before closing the phase. This plan does not request `just check-full`: the full pytest
lane supplies the relevant concurrency and cleanup evidence without introducing
screenshot-update work.

Acceptance requires the exact node, whole file, and full-lane node to pass with the
identity assertions preserved, no shared-temp deletion by the child, no new leak guard
failure, and no timeout increase. Record total child-process completion, not just an
inner pytest success line. A full-suite failure elsewhere that reproduces identically on
the clean base is a follow-up, not a reason to leave this assigned phase open. Record
the clean-base witness and any existing tracking bead in a `PROPOSED FOLLOW-UP:` note. A
remaining failure in this node is not completion of the requested fix.

## Phase closeout

Record
`TASK RESOLVED: sase-120 — <cause, regression evidence, focused/file/full results and ToolRun IDs>`
on `sase-1pu.4` when the acceptance criteria are met. The parent land agent owns closing
`sase-120` and `sase-1pu`.

Immediately before closure, run `sase bead epic-symbols sase-1pu.4`. No entries were
present during planning; recheck after implementation. Resolve each entry or re-key it
to a still-open parent/later bead before closing. Then close only this phase with
`sase bead close sase-1pu.4 --note "<what was verified>"` and finish through
`/sase_final`. Do not hand-edit bead status or close any ancestor.
