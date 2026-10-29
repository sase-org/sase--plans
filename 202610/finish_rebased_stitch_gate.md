---
tier: tale
title: Repair rebase gate guarantees and finish sase-1pm.5.1
goal:
  Prevent false clean reports and recovery bypasses, bound gate work, and close the
  rebase gate epic and delegated parent phase on verified evidence.
size: medium
proposed_by: bbugyi200.athena.sase-1pm.5.1.land
proposed_with: codex/gpt-6-astra
bead: sase-1pm.5.1
create_time: 2026-10-29 06:56:58
status: wip
---

- **PARENT:**
  [202610/rebased_stitch_gate.md](https://github.com/sase-org/sase--plans/blob/main/202610/rebased_stitch_gate.md)
- **BEAD:**
  [sase-1pm.5.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1pm/sase-1pm.5.1.md)

# Repair rebase gate guarantees and finish the landing

Implement only the remaining work in epic `sase-1pm.5.1`. This is a bounded repair of
existing seams, sized medium for one coding agent. It includes the landing: no land
agent will resume after this tale. Do not wait for this turn's own commit SHA, push, or
CI result before closeout; the host commits after the coding turn ends.

The final inherited decision is `rebase_gate = imports`. Keep the import-only hook, the
empty global default, project-local configuration, validated-findings-only blocking, and
infrastructure-error fail-open behavior. No mypy/full-check hook, global install, new
CLI, or memory edits are authorized. Cancellation must remain cancellation.

Read `sase-1pm.5.1` and its four child beads, including the landing audit and follow-up
triage notes, and use `sase artifact read plan:202610/rebased_stitch_gate.md` for the
original acceptance contract. The parent is phase `sase-1pm.5`, within epic `sase-1pm`.
The detailed audit and reproduction record is `file:explicit:116bb80ebf009ce33ea6bebd`;
read it with `sase artifact read` when exact probe details are needed. The original plan
is `202610/rebased_stitch_gate.md` in the plans repository; obtain its writable path
with `/sase_repo` and `sase repo open plans` when updating it.

## Evidence already established

At audit, primary HEAD and fetched origin/master both equal `aef552480b`. Epic commits
are `c76fee2af1` (facade), `bb5653cb7d` (runner), `475b2048ea` (checker), and
`aef552480b` (publication). The Rust policy/bindings commit
`60fe51d627a6b5df57511d9d9f0388ec39ddc4b7` is exactly the current
`sase-core-revision.txt` pin. The GitHub plugin inherits both commit and finalize
through `GitCommon`, with no separate implementation to patch.

The new facade/runner/checker/publication tests and artifact-directory audit produced 54
passes and one failure: the runner's `_cleanup_workdir` is unreviewed in
`tests/test_agent_artifact_directory_operation_audit.py`. Existing commit-first, hooks,
resume, checkpoint, and conflict-repair coverage produced another 64 passes.
`sase bead epic-symbols sase-1pm.5.1` listed no entries. No `check-full` was run.

Independent probes on the audited tree established:

- Compiling a module exporting `X`, changing its committed source to export `Y` with the
  same size and mtime, then checking an importer of `X` returns `clean`. The checker
  reads the old `.pyc`; disabling bytecode writes does not stop bytecode reads.
- An ignored, untracked `src/sase/ghost.py` exporting `X` satisfies a committed
  `from sase.ghost import X` even though the module is absent from the committed tree.
  The checker reports `clean` because `ls-files --exclude-standard` omits the source.
- Timeout `31.0` runs a marker-writing hook, then reports that invalid configuration
  skipped the gate. Numeric bounds are validated only after execution.
- A 0.3-second hook timeout took 2.31 seconds and left a SIGTERM-ignoring child alive
  holding stdout/stderr after the group leader exited. The probe killed that child.
- Raising `asyncio.CancelledError` from the runner in synchronous dispatch makes
  `_evaluate_context` return `(True, None)` instead of propagating cancellation.
- Saving the workflow's pre-dispatch checkpoint, creating the divergent rename/import
  scenario, and interrupting the runner after rebase leaves pending context in memory
  but absent from `checkpoint_load()`. Finalizing with that freshly loaded payload
  successfully pushes the broken commit without checking it.

## 1. Make execution obey the configured budget

Work in `rebase_gate_runner.py`, `rebase_gate_coordinator.py`, and their narrow IO
helpers. The coordinator currently gives each git query 60 seconds and starts the
runner's timer afterwards. Thread one remaining gate-work budget through configuration,
context discovery, changed-path queries, execution, draining, report reads, and
teardown. Do not charge ordinary fetch/rebase/push execution as gate overhead, or
refresh a full gate budget at every helper. Bound cleanup and result consumption as well
as waiting for the hook; reserve a small explicit bounded teardown allowance if
necessary.

Validate settings before launching any hook or computing gate context. Numeric
finite/positive/maximum policy belongs in `sase_core`, not duplicated Python checks.
Reuse or narrowly expose a Rust configuration/preflight result through the existing
facade. If core changes are needed, open `sase-core` through `/sase_repo`, follow its
AGENTS.md, add binding coverage, and include that repository in host finalization so the
source pin advances. Invalid settings warn and launch nothing.

Kill surviving process-group members even if the group leader already exited; reap the
owned process, bound reader joins, and propagate `KeyboardInterrupt`, `SystemExit`, and
`asyncio.CancelledError`. Use `finally` for transient-file cleanup. Keep ordinary
IO/checker exceptions fail-open with visible evidence. Coordinator git failures must not
silently become empty path sets and a false clean report. Result reads need an actual
byte cap, rather than stat followed by unbounded `read_text`, and must respect the
remaining budget. Preserve before/after hook behavior.

Add focused cases for invalid numeric timeouts (zero, negative, over 30, infinity, NaN),
exhausted context-discovery budget, failed git path discovery, inherited pipes, an
ignoring descendant after leader exit, and cancellation. Use short deterministic
fixtures and clock/control seams, without full-duration sleeps.

## 2. Inspect the committed source faithfully

Read `tools.md` before editing `tools/_check_rebase_imports.py` and its executable. Keep
Python AST/importlib mechanics here and shared gate decisions in Rust.

Make all first-party loads, including transitive imports and reexports, source-only
within the checker process. Preserve normal package/submodule behavior. Never consume an
old bytecode cache or change/delete caches in the checkout. Verify inspected/imported
source against the named committed tree, including ignored/untracked source, symlinks,
and source loaded transitively. Use targeted blob/source checks or an equivalent bounded
mechanism; do not export or scan the whole repository as the default path. Mismatch or
uncertain identity is an error/warning, never a clean certification or guessed finding.
Resolve deleted/renamed entries against the actual result tree when contexts accumulate.

Retain nested/function/TYPE_CHECKING/relative import handling, exact exceptions only,
submodule imports, star-module resolution, and dependency errors as errors. Add actual
checker subprocess fixtures for same-size/same-mtime stale pyc, ignored source, and
transitive source identity, alongside the existing committed-versus-dirty fixtures.

Exercise the production `sase/sase.yml` hook command with the workspace interpreter, not
only tests' `sys.executable` overrides. The current command is bare `python3` and the
runner inherits PATH; ensure it selects this checkout's environment as the approved plan
requires. Change the repo-local command or use the established interpreter resolver as
appropriate; do not install dependencies in the gate.

## 3. Persist and accumulate checks until publication succeeds

Work in the existing coordinator, `_git_commit_dispatch.py`, workflow checkpoint, and
resume seams. Bind a narrow persistence callback/context at the workflow boundary so
dispatch mutations save the current operation's checkpoint. Keep the checkpoint payload
JSON-serializable; do not introduce another store or persist callable state.

Persist eligible immutable input IDs and paths before starting a rebase that can stop
for conflict or cancellation, refresh and persist the resulting identity before gate
execution, and retain context through passing checks until the corresponding push has
succeeded. A failed push, retry rebase, or restarted process must re-evaluate and retain
the union of earlier inputs; no durable pass receipt authorizes skipping. Include paths
introduced by semantic/manual conflict resolution and later repair/amend, not just the
two original side diffs. Keep policy and context merging in Rust, IO in the coordinator.

An unchanged broken tree blocks on resume. A committed repair publishes once without
duplicate commits. Findings still preserve the checkpoint and `pushed=false` marker and
return workflow `RunResult.CONFLICT` / CLI exit 2 with repair-and-resume guidance.
Generic push failures retain their existing outcome. Preserve ownership/provenance
checks.

Add end-to-end workflow tests that reload the checkpoint from disk in a fresh
invocation, including interruption before/after rebase and before gate completion, a
clean gate followed by failed push and later repair, repeated retry rebases with
accumulated paths, and manual/semantic resolution adding an importer. The current
manual-conflict test throws away its original payload and checks an empty resume
payload; replace/strengthen it to prove the saved pending gate actually runs. Assert
origin, local HEAD, marker, checkpoint, and workflow result rather than only helper
calls. Keep existing clean, disabled, unmoved, fast-forward, no-origin, detached, and
fail-open behavior.

## 4. Integrate and verify

Add the reviewed directory-operation entry for the runner's temporary-workdir cleanup if
that operation remains. Explain its actual private temporary-directory ownership,
following the existing audit convention. Do not suppress the audit or file this as an
unrelated task: this epic introduced the operation.

The landing audit reviewed concurrent commits since epic creation: snapshot audit
`4d22ceefa5`, import budget `24561602cc`, SHA receipts `45ded07c14`, staged proposals
`9f88875b98`, invocation split `e056c6b8c0`, staged acceptance `cc32e69dbb`, staged
landing `894317dccd`, terminal isolation `bcddb9edb3`. The audit-site addition is
required integration. SHA receipts already reject `pushed=false`/dispatch errors; keep
that true for blocked gate outcomes and ensure staged completion cannot adopt them as
publication. Check any newer base-branch drift before closeout. Keep proposal/PR-only
publication outside the gate scope; do not fork the GitHub implementation.

Read required verification memory, run `just fix`, the affected focused tests (including
real Rust binding tests if changed), then `sase tool run check` in each changed code
repository. This is the guarded form of `just check`; do not run `just check-full`. Use
`/sase_monitor` if verification needs a handoff, with explicit instructions to finish
this tale's closeout after the result; do not assume another lander will resume it.

Repeat representative end-to-end enabled/disabled measurements using actual repo source
and record path/import counts, interpreter/cache conditions, total gate overhead, and
verdict. Prior evidence was 2.04 seconds dispatch, 1.29 seconds gate, 0.27 seconds
disabled; the hardened implementation must still finish useful checks within its
ten-second budget. Update existing hook docs only if implementation details changed,
after reading `docs_writing.md`; regenerate generated config docs instead of editing
them manually.

## 5. Finish this epic and its delegated parent phase in this turn

Before closing, re-read the epic and all descendants/notes and their linked-plan
readiness. Carry the following completed follow-up triage into the close note, including
originating phase IDs: `.1` note #1's stale `sase-1pj.1.1.4` exemptions were already
removed by `9f88875b98` (proposal declined as resolved); `.3` note #2 and `.4` note #2's
monitor env/ack failures were relayed as supplementary evidence to existing task
`sase-1pg` (no duplicate task); those notes' manifest mismatch was corroborated on
causally related active epic `sase-1pi`, whose phase `.3` introduced the missing
contract-manifest entry (no new task); `.4`'s artifact-audit proposal was retained and
fixed as this epic's work. The proposing lander recorded these outcomes on
`sase-1pm.5.1` before this proposal.

Run `sase bead epic-symbols sase-1pm.5.1`. Resolve each entry or re-key it only to a
still-open later bead that actually needs it, per Symvision policy. Then close normally:
`sase bead close sase-1pm.5.1 --note "<source/commit audit, repairs, integration, verification, timing, and every follow-up disposition>"`.
Run `just symvision` after closing. Open the plans repo with `/sase_repo` and set
`status: done` in `202610/rebased_stitch_gate.md` frontmatter. Audit reads of sidecar
plans must still use `sase artifact read`.

Re-read `sase-1pm.5.1` for its parent link and `sase-1pm.5` to confirm the delegated
phase remains the parent. Verify this child now completes its import-gate scope, append
`TASK RESOLVED: sase-1cb — <existing implementation commits, completed repairs, tests, timing>`
to `sase-1pm.5`, inspect and resolve/re-key `sase bead epic-symbols sase-1pm.5`, and
close only that phase with
`sase bead close sase-1pm.5 --note "<verified child gate completion>"`. Confirm
`just symvision` again if parent-owned exemptions were touched. Leave containing epic
`sase-1pm`, its plan status, and task `sase-1cb` to the waiting outer lander.

Never use `--force` to advance this successful nested landing. If close rejects
unfinished descendants, named phases, or epic symbols, finish the actual work and retry
normally. If the parent relationship is unexpectedly ambiguous, record the concrete
blocker on the parent and report it rather than guessing. Finish through `/sase_final`,
declaring all changed repositories (including plans and core if edited), without waiting
for the host's later commit.
