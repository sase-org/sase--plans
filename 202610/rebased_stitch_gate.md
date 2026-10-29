---
tier: epic
title: Check rebased stitches before publication
goal: Block publication of confirmed broken first-party imports introduced by combining
  concurrent commits, while keeping unchanged-base landings cheap and checker failures
  bounded and fail-open.
parent_bead: sase-1pm.5
phases:
- id: gate-policy
  title: Define the shared rebase gate contract in Rust
  depends_on: []
  size: medium
  description: 'gate-policy: implement shared gate policy in sase-core, expose and
    test its Python bindings, and add the thin sase adapter. Own eligibility, changed-path
    union, pending-context merging, and verdict validation. Keep the gate disabled.'
- id: gate-runner
  title: Execute configurable rebase hooks with a hard deadline
  depends_on:
  - gate-policy
  size: medium
  description: 'gate-runner: implement project-scoped configuration, bounded process
    execution and evidence, fail-open errors, default configuration, schema, and hook
    documentation. Do not wire publication or enable the repo hook yet.'
- id: import-checker
  title: Resolve changed-file first-party imports on the rebased tree
  depends_on:
  - gate-policy
  size: medium
  description: 'import-checker: implement repo tooling using Python AST and importlib,
    with exact TYPE_CHECKING alias exceptions and subprocess fixtures. Follow the
    gate wire contract; do not modify commit dispatch or enable config.'
- id: publication
  title: Gate initial rebases, push retries, and checkpoint resumes
  depends_on:
  - gate-runner
  - import-checker
  size: medium
  description: 'publication: wire the shared git dispatch and resume paths, enable
    this repo''s import gate, add temporary-repository integration coverage, and measure
    complete gate latency. Preserve markers and return the recoverable conflict result
    for confirmed findings.'
proposed_by: bbugyi200.athena.sase-1pm.5
proposed_with: codex/gpt-6-astra
create_time: 2026-10-29 02:12:55
status: wip
bead_id: sase-1pm.5.1
---

- **PROMPT:** [prompts/202610/rebased_stitch_gate.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/rebased_stitch_gate.md)
- **PARENT:** [202610/task_backlog_epic_1.md](https://github.com/sase-org/sase--plans/blob/main/202610/task_backlog_epic_1.md)
- **BEAD:** [sase-1pm.5.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1pm/sase-1pm.5.1.md)

# Check rebased stitches before publication

This child epic implements only phase `rebase-gate` of
`plan:202610/task_backlog_epic_1.md`, assigned as `sase-1pm.5`, for task `sase-1cb`. The
parent's final decision is **rebase_gate = imports**. There is no new decision: do not
add mypy, a full verification run, or memory edits. The parent's import-budget choice
belongs to a different phase.

## Verified starting point and scope

At planning time, sase HEAD was `0f73623591`. `sase-1pm.5` was in progress with no
previous implementation or remaining-work notes. Searching source, tests, tools, and
project config found no rebase gate implementation. `sase bead epic-symbols sase-1pm.5`
reported no entries. Recheck the live tree before implementing.

The incident is a clean textual rebase of incompatible changes: upstream privatizes `X`
to `_X`, while a concurrent local commit adds `from sase.module import X` in a different
file. The September 28 incident broke TUI startup after publication; three earlier
incidents are recorded in `sase-1cb`.

Relevant existing seams:

- `src/sase/vcs_provider/plugins/_git_commit_dispatch.py` owns `vcs_create_commit`,
  `_sync_with_origin_after_commit`, `_rebase_onto_origin_default`,
  `_push_current_branch_with_rebase_retry`, and `vcs_finalize_commit`. Successful
  semantic conflict resolution returns through the same rebase helper. Resume
  finalization currently attempts a push immediately.
- `src/sase/workflows/commit/command_hooks.py` runs `before` and `after` hooks and
  captures capped logs, but has no hard execution deadline and returns only a boolean.
  Preserve those existing hooks' behavior.
- `workflow.py`, `workflow_resume.py`, `checkpoint.py`, and `workflow_support.py` in
  that same directory own checkpoint recovery and unpushed markers. Today a push failure
  preserves a marker but returns `FAILED`; textual conflicts return `CONFLICT` (exit 2).
  A semantic gate finding needs both preservation and exit 2.
- `sase-github/src/sase_github/plugin.py` inherits `vcs_create_commit` through
  `GitCommon`; it does not implement another commit path. No plugin fork is needed.
- Config is declared in `src/sase/default_config.yml` and
  `src/sase/config/sase.schema.json`. This repo sets its before hook in `sase/sase.yml`.
  `load_merged_config_for_local_path` exists for explicit project configuration.
- Existing integration scaffolding lives in
  `tests/test_vcs_provider_commit_first_rebase.py`; recovery coverage lives in
  `tests/test_commit_workflow_resume*.py`, `tests/test_commit_workflow_checkpoint*.py`,
  and `tests/test_commit_dispatch_conflict_repair_resume.py`.

This merits an epic because shared policy/bindings, subprocess containment, Python
import resolution, and persisted publication recovery each need focused coverage. The
runner and checker may proceed independently after the policy contract lands. Each phase
is bounded direct implementation work.

## Shared contract

### When the gate runs

Capture immutable git IDs before each rebase: old local HEAD `H`, fetched upstream tip
`U`, and their merge base `B`. A successful rebase that combines divergent work
(`B != H` and `B != U`) creates pending gate context for its resulting HEAD/tree. An
already integrated upstream, a no-op rebase, or a pure fast-forward does not. Check the
actual successful rebase result; do not infer movement from HEAD changing alone. Capture
before semantic auto-resolution too. A textual conflict retains the context so manual
resolution and resume cannot bypass the check.

The input path set is the union of paths changed in `B..H` and `B..U`, using NUL-safe
git output and keeping rename destinations and deletion information. Resolve files
against the resulting tree. Include changes introduced by conflict resolution and later
repair/amend, without scanning the whole repository or limiting to `HEAD^`. Pending
contexts accumulate across additional rebases; each newly combined tree must be checked
before the next push attempt. With no configured hook, avoid extra git queries,
subprocesses, or imports for gate computation.

The wire context has schema version 1, repository root, the immutable input IDs, current
HEAD/tree, and normalized repository-relative changed paths. A persisted context stores
references and paths, not approval or a reusable pass receipt.

### Configuration and command protocol

Add `commit_hooks.after_rebase` (string, default empty) and
`commit_hooks.after_rebase_timeout_seconds` (positive finite seconds, default 10,
maximum 30). Resolve the project-local layer for the repository being committed, even if
the host's process cwd belongs to another project. Invalid gate settings warn and skip
the gate; existing before/after behavior stays intact.

Run the command with repository-root cwd and two environment variables:
`SASE_REBASE_CONTEXT` names a temporary JSON input file and `SASE_REBASE_RESULT` names
the fresh result file to write. Never interpolate changed paths into shell text. Result
JSON includes schema version, checked HEAD/tree, status `clean|findings|error`,
findings, and diagnostics. Findings identify path, line, module, imported name, and
explanation. Bound result size as well as log size.

- Exit 0 plus a valid matching `clean` report permits the push.
- Exit 1 plus a valid matching `findings` report with at least one concrete finding
  blocks the push.
- Timeout, signal, spawn failure, other exits, malformed/missing/stale reports, invalid
  config, or explicit `error` reports produce a visible warning and allow the push. A
  generic shell crash returning 1 must never become a semantic finding.

Use one monotonic wall-clock budget for context preparation and command execution; bound
gate-specific git queries, stream draining, teardown, and result reads too. Terminate
the subprocess group on timeout, then kill after a short bounded grace period, and reap
it. An inherited pipe in a descendant must not hang the finalizer. Keep evidence
containing IDs, changed-file count, duration, verdict/reason, and capped output.
Observability and cleanup failures must also fail open. Cancellation of the overall
stitch remains cancellation, not permission to keep pushing.

## Phase gate-policy: shared Rust policy and bindings

Open `sase-core` with `/sase_repo` and read its `AGENTS.md`. Add a focused
`sase_core::rebase_gate` domain, following its free-function and serde `*Wire`
conventions. It owns eligibility from git facts, deterministic changed-path union,
pending-context merge/update, configuration bounds, and classification of command facts
plus result JSON into `skip|pass|block|warn`. Make the policy reject stale tree IDs and
block only on validated findings. Python supplies IO facts; it must not duplicate these
rules or provide a fallback implementation.

Expose the domain through `crates/sase_core_py/src/vcs/` using module-qualified core
imports and registered PyO3 functions. Add a small typed facade under `src/sase/core/`
using `require_rust_binding`. Register binding/schema validation wherever the existing
binding checks require it. Include real-binding round-trip tests, not only mocks.

Cover divergent, unchanged, already-integrated and fast-forward histories, malformed
facts, context accumulation, renamed/deleted paths, stale verdict identity, timeout, and
findings versus infrastructure failures. Keep these pure policy tests small.

Run the required sase-core and sase checks. Host finalization commits the linked core
first and advances `sase-core-revision.txt` through its configured revision pin when
both repositories change. Do not invent a SHA or alter released dependency versions.
Later phases must verify their source pin contains these bindings.

Acceptance: policy and binding tests pass, the core pin contains the new API, and no
production path executes a gate yet.

## Phase gate-runner: configuration, bounded execution, and evidence

Build a small executor in the commit workflow package that takes the gate context, loads
configuration for its explicit repository, launches the command, and passes the
collected facts to the Rust policy. Reuse capped logging and step reporting from
`command_hooks.py`; refactor narrowly if needed to avoid changing existing hook
semantics. All policy decisions come from the new facade.

Add default YAML and schema keys with useful descriptions, including the deadline and
fail-open behavior. Regenerate `docs/reference/config/repositories.md` with `just fix`;
never hand-edit it. Update the existing hook reference in
`docs/reference/config/finalizers.md` and, only where necessary, the evidence contract
in `docs/contribute/commit-workflow-orchestrator.md`. Read `docs_writing.md` first.
Document the context/result contract and distinguish the new phase's error policy from
`before` and `after`. Keep the project hook disabled in this phase.

Use subprocess tests for clean/finding/error, exit-1 crash without a report, stale
report, timeout with a child that holds stdout open, excessive output, failed log
creation, and cleanup. Verify configuration isolation between two project roots. Ensure
the disabled case does not launch a process. Use tight fixture deadlines; do not add a
timing-sensitive ten-second sleep to normal tests.

Acceptance: valid findings block, all specified gate errors warn and return promptly,
existing hook tests pass, and no actual publication path is changed yet.

## Phase import-checker: changed-file AST/importlib tooling

Read `tools.md`. Implement a repo-local tool such as `tools/check_rebase_imports` with a
tested helper module. This is Python-language inspection tooling: Python's `ast` and
`importlib` own language mechanics; the shared gate policy stays in Rust. Use the
manifest/result protocol above. No new public `sase` CLI is required.

Inspect existing changed `.py` files from both rebase sides, including tests and tools.
Walk every nested `ast.ImportFrom`, including function bodies and `TYPE_CHECKING`
blocks. Resolve absolute `sase`/`sase.*` imports and relative imports that resolve into
that package. Handle aliases and package submodule imports correctly; do not treat a
legitimate `from sase.package import submodule` as a missing attribute. Define and test
star-import behavior explicitly (at least resolve its module; do not invent individual
names). Skip deleted/non-Python paths without importing them.

Run in a fresh interpreter using the workspace environment, explicitly placing the
checked repo's `src` root first. Avoid importing installed/stale sase code or cached
bytecode from a previous tree. Verify first-party module origins. Imports must never
pollute the long-lived finalizer. Disable bytecode writes. Do not install dependencies,
change the checkout, execute tests, or import each changed file just to inspect it.

A successfully loaded first-party module missing the requested name/submodule is a
finding. A demonstrably absent requested first-party module is also a finding.
Dependency failures, ambiguous/transitive import exceptions, interpreter errors,
syntax/parser failures, origin mismatches, and checker bugs are `error` with useful
diagnostics, not guessed findings. Do not classify all ImportErrors as findings. Resolve
repeated module/name pairs once per invocation and bound the whole worker through the
runner's deadline.

Add only exact, justified exceptions for existing TYPE_CHECKING-only aliases that cannot
resolve at runtime. Each entry identifies module/name and its reason; apply it only to a
proven type-checking context. Never blanket-skip those blocks or allow an exception to
conceal the same missing name in a runtime import. Derive entries from current source,
not the old incident's count of seven.

The report names the committed tree, so verify the Python source being read/imported
matches it. In the presence of autostashed or excluded Python changes that prevent that
guarantee, emit an error/warning and fail open; never certify a repaired working file
over a broken committed file. A whole-tree export or scan is not the default cost. Cover
the distinction in a fixture.

Use small fake `sase` packages in isolated subprocesses for top-level, function-local,
TYPE_CHECKING, relative, re-export, submodule, removed-module, rename, and dependency
error cases. Include stale-installed-package and exact-allowlist counterexamples.

Acceptance: the checker catches the rename/import incident at every supported AST depth,
inspects both sides' input files, and emits only protocol-valid results.

## Phase publication: integrate every push and recovery path

Wire a narrow gate coordinator into the shared git commit dispatch. Capture context
around the initial rebase and each retry rebase, including semantic auto-resolution,
then execute it before pushing the resulting tree. Keep generic execution and policy out
of the dispatch mixin itself. No checks on proposal saving or unrelated PR push paths
are required by this phase. Verify GitHub's inherited commit/finalize path is covered;
change the plugin only if a real override is discovered.

Persist pending context with the existing checkpoint payload, using a private key and
existing checkpoint saves rather than introducing a second recovery store. Save the
context before a rebase that might stop for manual resolution. On a finding, leave HEAD
local, record its current SHA/tree with `pushed=false` through the existing
unpushed-marker path, keep the checkpoint, print findings and repair instructions, and
return `RunResult.CONFLICT` / exit 2. Distinguish a gate finding explicitly in the saved
payload/result handling; do not broaden every ordinary push failure into a conflict or
rely on searching arbitrary hook output for the word `conflict`.

`vcs_finalize_commit` and `--resume` must run pending gate work even when no fresh
rebase occurs. An unchanged broken tree must block again. Recompute the context for the
actual repaired HEAD and include repair/amend paths. The message must explain that
repairs need to be included in the local commit through the existing SASE repair flow
before resume; unstaged edits cannot count as a fixed committed tree. Preserve existing
provenance checks and avoid duplicate commits. Clear pending state only when the
corresponding tree is allowed onward; never use a prior process's pass report as a skip
receipt. Persist enough state across a failed push/crash that recovery cannot silently
bypass a previously required check.

Enable only this repo's hook in `sase/sase.yml`, invoking the checker with the workspace
Python interpreter and ten-second budget. Leave the global default empty.

Add real temporary-git-repository integration tests using the existing fixtures:

1. Upstream privatizes a name; local adds its old import in another file. Both sides
   work separately; the combined rebase is textually clean. Assert origin does not move,
   local commit survives, findings name the import, marker/checkpoint survive, and the
   workflow returns exit 2 with `sase stitch create --resume` guidance.
2. Reverse which side adds the importer, proving incoming paths are included.
3. Advance origin immediately before the first push, forcing retry rebase. Assert the
   new tree is checked and a finding prevents the retry push.
4. Resume unchanged blocked HEAD and block again; repair through the normal commit
   repair path, resume, and publish once. Include a manual textual-conflict resume.
5. Clean divergent rebase publishes; unchanged-base, fast-forward, disabled-hook,
   no-origin, and detached cases avoid unnecessary hook execution.
6. Timeout, missing executable, crash, and malformed report warn and still permit an
   otherwise valid push. Genuine git/rebase/push failures retain their behavior.
7. An unrelated dirty Python file or an unstaged repair cannot produce a false
   certification of committed code. Exercise the documented fail-open result.

Run existing commit-first rebase, hook, checkpoint, resume, and finalizer-repair
coverage together with the new integration tests. Test external effects and recovery,
not merely call order. Git fixtures set local identities and isolate global hooks.

Measure end-to-end gate overhead, including context discovery and interpreter startup,
on a representative moved-base case using this repo's real source. Record checked paths,
import counts, cache/environment conditions, and elapsed time in the phase note. Compare
a disabled/unmoved-base case. A measurement that only times out is not evidence of a
useful check: tighten imports/startup work until representative valid cases complete in
seconds within the budget. Do not increase to a full check or scan all source to
manufacture coverage.

## Verification and landing

Each coding phase reads required memory before touching its domain, runs focused tests
and `just fix`, and runs `sase tool run check` for every changed repository. Follow
`/sase_monitor` for long checks and host-owned finalization for commits. Do not run
`check-full`, global installs, or manual git commits. Open other repositories only with
`/sase_repo`. No memory edits are authorized by this plan.

Coding phases close only their assigned child phase. Before closing, inspect
`sase bead epic-symbols <phase-id>` and resolve or correctly re-key entries. Record
out-of-scope findings as `PROPOSED FOLLOW-UP:` on the phase; create no task beads. A
failure reproduced identically on a clean base is recorded with that evidence and does
not keep the phase open. Carry necessary evidence back to `sase-1pm.5` for the outer
epic's land agent.

The child epic's land agent verifies all four implementations and pins are landed,
reruns the relevant integration acceptance on the combined tree, and records the
measured latency and any limitations. It closes its completed child epic through the
normal landing workflow, then completes the specifically delegated parent phase
`sase-1pm.5`: append `TASK RESOLVED: sase-1cb — <commits, tests, and timing>`, run
`sase bead epic-symbols sase-1pm.5`, resolve leftovers, and close that phase with a
verification note. This is the completion responsibility delegated to this child land
agent; individual coding phases must never close ancestors.

Do not close `sase-1pm`, any higher plan bead, or task `sase-1cb`. The outer epic land
agent owns task closure and its backlog-chain obligations. If acceptance remains
unfinished, leave concrete evidence and remaining work rather than claiming resolution.
