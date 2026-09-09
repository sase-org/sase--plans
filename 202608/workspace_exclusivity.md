---
tier: epic
title: Guarantee one agent per workspace
goal: "Two SASE agents can never run in the same workspace checkout: workspace
  allocation is atomic on every path, a destructive workspace preparation refuses to run
  when another live agent occupies the checkout, and every RUNNING-field mutation is
  recorded so a future occupancy incident is diagnosable from the ledger alone.

  "
phases:
  - id: ledger
    title: Durable RUNNING-field mutation ledger
    depends_on: []
    size: small
    description: "ledger: record every workspace claim, transfer, hold, and release to a
      durable JSONL ledger with the full before/after occupancy of the affected
      workspace, so a vanished or duplicated claim row is attributable after the fact.

      "
  - id: atomic
    title: Atomic workspace allocation on every path
    depends_on: []
    size: medium
    description: "atomic: replace the deferred-workspace check-then-claim with the
      atomic allocate-and-claim helper, materialize the checkout only after the claim is
      held, and remove the remaining unchecked pinned-target claim.

      "
  - id: guard
    title: Refuse destructive preparation of an occupied checkout
    depends_on:
      - atomic
    size: medium
    description: "guard: write a per-checkout occupant record when an agent takes a
      workspace and make every destructive preparation path verify exclusive occupancy
      before it cleans, resets, or checks out, failing the run instead of deleting
      another agent's work.

      "
  - id: detect
    title: Detect and surface occupancy conflicts
    depends_on:
      - ledger
      - guard
    size: small
    description:
      "detect: add a doctor check and a concurrency regression test that prove
      simultaneous allocation bursts never hand the same workspace to two agents and
      that a conflicting occupant is reported rather than silently overwritten."
proposed_by: bbugyi200.athena.06g
status: done
bead_id: sase-q0
create_time: 2026-09-09 19:52:06
---

- **PROMPT:**
  [prompts/202608/workspace_exclusivity.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/workspace_exclusivity.md)
- **BEAD:**
  [sase-q0](https://github.com/sase-org/sase--beads/blob/main/pages/sase-q0/README.md)

# Plan: Guarantee one agent per workspace

## Background: the incident

On 2026-08-18 two agents ran concurrently in workspace `#17`
(`.../workspaces/sase-org/sase/sase_17`) and destroyed each other's work. The
reconstruction below comes from `~/.sase/logs/tui_launch_timing.jsonl{,.1}`,
`~/.sase/logs/runs.jsonl`, the per-run output logs under `~/.sase/workflows/202608/`,
and the surviving `agent_meta.json` / `retry_state.json` artifacts.

| time (local) | event                                                                                                                                                                               |
| ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 12:21:06     | `#17` freed by `ace(run)-260818_121851`; unclaimed afterwards                                                                                                                       |
| 12:59:40     | `sase-pv.2` completes, releasing every clan sibling waiting on it                                                                                                                   |
| ~12:59:47    | `sase-pv.4` (`ace(run)-260818_112717`) resumes from its `%w` wait, re-execs the runner for a code refresh, then runs the deferred claim and prints `Claimed workspace #17`          |
| 12:59:57.754 | `06e--plan` (`ace(run)-260818_125956`, pid 2448818) is spawned by ACE with `workspace_num=17`; its `workspace_claim` stage succeeds in 1.08 ms                                      |
| 13:00:14     | `sase-pv.4` fails preparing linked repo `sase-core`; the runner reports `Workspace #17 held (visible failed run)`                                                                   |
| 13:02:30.397 | replacement `sase-pv.4` (`ace(run)-260818_130227`, pid 2472885) is spawned with the deferred placeholder `#0`                                                                       |
| ~13:02:35    | the replacement's deferred claim prints `Claimed workspace #17` and immediately cleans `sase_17` and resets it to `origin/master` — while `06e--plan` is live in that same checkout |
| 13:03:41     | `06e--plan`'s first provider attempt fails (`API Error: 529`) — the _first_ `06e` failure, and it happens **after** the collision                                                   |
| 13:0x–13:1x  | `sase-pv.4` narrates `Some edits look like they didn't stick` and `Source files were reset while another agent edited ACE details`                                                  |
| 13:19:19     | `06e--code` starts in `sase_17` on the same pid and runs its `gh__setup` / `gh__prepare` / `gh__checkout` steps against the shared checkout                                         |
| 13:21:24     | `sase-pv.4` is SIGTERMed with its work gone                                                                                                                                         |

### The reported suspicion is not what happened

The suspicion was that releasing `#17` after `06e` failed let a second agent in. The
evidence rules that out:

- `06e`'s retries were **in-process**, in one pid, recorded in `retry_state.json`
  (`retry_count: 2`) with per-attempt snapshots under `attempts/01` and `attempts/02`.
  No workspace was released between them, and `claude`'s retry config sets
  `preserve_workspace=True`, so the retry path never re-prepared the checkout.
- `06e`'s first failure was at 13:03:41. The collision was already established at
  ~13:02:35. A later event cannot cause an earlier one.
- Release is not a blunt instrument: `release_workspace_from_content` in
  `sase_core::agent_cleanup` matches on workspace number **and** workflow **and**
  cl_name when they are supplied, and every agent-lifecycle caller supplies both. One
  agent's release cannot delete another agent's row.

The instinct behind the suspicion was still sound — a workspace did change hands around
a failure. It was `sase-pv.4`'s own failure at 13:00:14, and the defect is in how the
next agent _acquires_ a workspace, not in how a finished one gives it back.

### Root cause

`claim_deferred_workspace` in `src/sase/axe/run_agent_phases.py` is the last
check-then-claim allocation path in the codebase. Every attempt of its retry loop does:

1. `get_first_available_axe_workspace(project_file)` — read the RUNNING field under one
   lock, drop the lock, return a number;
2. resolve the workspace directory, which for a VCS workflow goes through
   `sase.workspace_provider.get_workspace_directory` and can materialize, clone, and
   **clean** the checkout — unlocked git work measured in seconds;
3. `claim_workspace(...)` — take a second lock and write the row.

`claim_next_axe_workspace` exists precisely to remove this window; its docstring says it
"Combines `get_first_available_axe_workspace` and `claim_workspace` into a single
operation to eliminate the TOCTOU race window". Every launcher path uses it. The
deferred path does not. This is the path both `sase-pv.4` runs took, and the incident
happened in the exact burst that opens the window: `sase-pv.2` finishing released
`sase-pv.3` **and** `sase-pv.4` from their `%w` waits in the same second, while ACE
spawned `06e` seventeen seconds later.

Two amplifiers made a transient race catastrophic:

- **A deferred agent can run the claim twice in one run.** After a dependency wait,
  `run_agent_runner_refresh` `os.execv`s the runner when the installed sase code
  changed. `sase-pv.4`'s log shows exactly that
  (`Refreshing sase runner code after dependency wait: 6ed7e05 -> 50d837a`), so the racy
  allocation ran once per exec, in a window where the runner's own code was mid-swap.
- **Nothing verifies exclusive occupancy before destroying a checkout.**
  `prepare_workspace_if_needed` (`src/sase/axe/run_agent_runner_setup.py`) goes straight
  to `prepare_workspace`, which cleans the tree and hard-resets it to the update target.
  It never asks whether another live agent is already working there. The same is true of
  `prepare_linked_repo_workspaces_if_needed`, of the retry re-prep in
  `handle_workflow_error`, and of the per-phase VCS setup steps. A single bad allocation
  therefore converts directly into unrecoverable data loss.

### One anomaly the current logs cannot settle

At 13:00:14 `hold_workspace_claim` succeeded for `#17`. That helper re-adds the row
through `plan_claim_workspace_from_content`, which rejects the claim when _any_ row for
that workspace number already exists, and `hold_workspace_claim` checks that outcome. So
at that instant the file appears to have held exactly one `#17` row — yet `06e` had
claimed `#17` seventeen seconds earlier and kept using it for another 23 minutes. Either
a row was lost from the ProjectSpec file by a stale-content write (the file has 49
`write_patch_atomic` call sites across 26 modules), or the interleaving differs from the
reconstruction above in a way the logs cannot show.

This is deliberately left open. The `ledger` phase exists to make it answerable, and the
`atomic` and `guard` phases are designed so the guarantee holds regardless of which
explanation is correct — the fix must not rest on a single check.

## Scope and non-goals

In scope: workspace allocation atomicity, occupancy enforcement before destructive
preparation, and observability of RUNNING-field mutations.

Not in scope: changing `%w` / clan scheduling semantics, changing the post-wait
`os.execv` code refresh, redesigning the ProjectSpec file format, or auditing all 49
`write_patch_atomic` call sites for stale-content writes. If the `ledger` phase produces
evidence of a lost update, file a task bead for that audit rather than widening this
epic.

Per `sase/memory/` guidance on the Rust core boundary, occupancy decisions ("is this
checkout occupied by someone else?", "does this ledger record conflict?") are backend
domain logic and belong in `../sase-core/crates/sase_core`, with Python and the TUI
calling through `sase_core_rs`. Path resolution, printing, and process signalling stay
in Python.

## `ledger`: durable RUNNING-field mutation ledger

Every mutation of the RUNNING field must leave a durable record. Today a claim row can
appear or vanish with no trace beyond the resulting file.

Add a ledger append to the four mutating helpers in
`src/sase/running_field/_operations.py` — `claim_workspace`, `claim_next_axe_workspace`,
`transfer_workspace_claim`, `hold_workspace_claim`, and `release_workspace`. Write each
record while the ProjectSpec lock is still held, after the successful
`write_patch_atomic`, so ledger order matches file order.

Each record carries: wall-clock timestamp, operation, project file, workspace number,
workflow label, cl_name, artifacts timestamp, the acting pid and its parent pid, the
outcome (success plus the rejection reason on failure), and — this is the part that
makes the record diagnostic rather than decorative — **the full set of claim rows for
the affected workspace number immediately before and immediately after the write**.
Include a short caller tag so a record names the code path that produced it
(`launcher-preclaim`, `deferred-claim`, `retry-transfer`, `agent-finalize`,
`stale-cleanup`, `dismiss`, and so on).

Write to a rotating JSONL under `~/.sase/logs/` alongside the existing telemetry logs,
reusing the append helper in `src/sase/logs/`. Failure to write the ledger must never
fail a claim: wrap the append so any exception is swallowed and logged at debug level.

Add `sase doctor`-adjacent read access — a small reader function the `detect` phase can
consume — plus tests that a claim/release round trip produces the two expected records
with correct before/after occupancy.

## `atomic`: atomic workspace allocation on every path

Rewrite `claim_deferred_workspace` in `src/sase/axe/run_agent_phases.py` so allocation
is a single atomic operation and no slow work happens between deciding a number and
owning it.

- Replace the `get_first_available_axe_workspace` + `claim_workspace` pair with
  `claim_next_axe_workspace`, which allocates and claims under one ProjectSpec lock.
- Resolve and materialize the workspace directory **after** the claim is held. If
  materialization then fails, release the claim explicitly before retrying, so a failed
  attempt cannot leak the slot.
- The `SASE_AGENT_DEFERRED_TARGET_WORKSPACE_NUM` pinned-target path (set by
  `src/sase/agent/_family_attach_launch.py`) currently bypasses the availability check
  entirely and retries the same number on every attempt. Keep the pin — reusing a
  predecessor's checkout is the point of family attach — but make the claim of a pinned
  target a hard, single-shot operation: if the pinned workspace is already claimed by a
  live agent, fail the run with a clear message naming the current occupant rather than
  looping or proceeding. A family member silently landing on an occupied checkout is the
  same bug in a different costume.
- Preserve the existing behaviour of releasing the `#0` placeholder first, and keep the
  bounded retry with the existing attempt limit for genuine contention.

Then sweep for any other check-then-claim pair: grep for
`get_first_available_axe_workspace` and `get_first_available_workspace` and confirm
every caller either claims atomically or is a read-only reporting path. Convert or
document each one.

Tests: a unit test that a deferred claim never selects a number that is claimed in the
RUNNING field at claim time; a test that a materialization failure after a successful
claim releases the claim; a test that a pinned target already held by a live pid fails
with the occupant named rather than double-claiming.

## `guard`: refuse destructive preparation of an occupied checkout

Allocation correctness is necessary but must not be the only thing standing between two
agents and a shared checkout. Add an independent guard at the point where damage occurs.

Introduce a per-checkout **occupant record** stored inside the checkout — next to the
existing marker written by `src/sase/workspace_provider/marker.py`, for example
`<checkout>/.sase/occupant.json`. It names the current occupant: pid, agent artifacts
timestamp, agent name, workflow label, project, workspace number, and claim time. It is
written when an agent takes the workspace (both the launcher-preclaim/transfer path and
the deferred path), and cleared when the claim is released. The occupant file must be
covered by the per-clone `.git/info/exclude` entries already installed by
`enter_agent_workspace` so `git clean` cannot remove it.

Put the conflict decision in `sase_core`: given an occupant record, the caller's own
identity, and whether the recorded pid is alive, return whether preparation may proceed,
plus a rendered reason. Python supplies the liveness answer and the paths.

Then require that decision before every destructive operation:

- `prepare_workspace_if_needed` in `src/sase/axe/run_agent_runner_setup.py`, before
  `prepare_workspace` cleans and hard-resets the tree.
- `prepare_linked_repo_workspaces_if_needed`, before it cleans linked-repo checkouts
  under the same workspace.
- The retry re-prep in `handle_workflow_error` (`src/sase/axe/run_agent_exec_retry.py`),
  which calls `prepare_workspace` when the provider config does not set
  `preserve_workspace`.
- The per-phase VCS setup steps a family agent runs when it moves from one phase to the
  next; `06e--code` ran `gh__setup` / `gh__prepare` / `gh__checkout` against the shared
  checkout at 13:19:19.

On conflict the agent must **fail loudly and stop** — print the occupying agent's name,
pid, and artifacts timestamp, write the failure into the run's done marker, and exit
non-zero. Do not attempt automatic relocation to another workspace: this guard exists to
prove the invariant is holding, and silently papering over a violation would hide the
next bug. Also cross-check the occupant record against the RUNNING field and treat a
disagreement between the two as a conflict, since disagreement means one of the two
sources of truth has already been corrupted.

Guard against the obvious failure modes: a stale occupant record from a dead pid must
not block a legitimate new agent (a dead recorded pid is a takeover, not a conflict); a
retry-spawn child inheriting its parent's workspace must be recognised as the same
occupant lineage, not an intruder; and a missing occupant file on an existing checkout
must be treated as unoccupied so this change cannot brick workspaces created before it.

Tests: preparation proceeds when the occupant record is absent, is this agent, is this
agent's retry-chain parent, or names a dead pid; preparation refuses when it names a
different live agent; refusal happens before any git mutation runs.

## `detect`: detect and surface occupancy conflicts

Close the loop with detection and a regression test that would have caught this.

- Add a `sase doctor` check that reads the RUNNING field for every project and reports
  any workspace number claimed by more than one row, any claim whose pid is alive but
  whose checkout's occupant record names a different live pid, and any occupant record
  with no corresponding claim. Report, do not auto-repair — an occupancy conflict means
  live work is at risk and needs a human decision.
- Add a concurrency regression test that starts several allocation attempts against one
  temporary ProjectSpec at once, mixing the launcher path and the deferred path, and
  asserts that the resulting claim rows contain no duplicate workspace number and that
  the count of distinct workspaces equals the number of successful claims. Drive real
  concurrency (threads or processes over a real file lock), not mocks; the bug being
  fixed is a race, and a mocked test cannot fail on it.
- Add a test that reproduces the incident's shape end to end at the unit level: agent A
  holds workspace N, agent B attempts a deferred claim, and the assertion is both that B
  does not receive N and that if it somehow did, the `guard` phase's check prevents any
  destructive preparation.
- Use the `ledger` reader so the doctor check reports when a claim row was last mutated
  and by which caller tag.

## Verification

Each phase runs `just check` before handing off. The final phase, and the epic's land
agent, must run `just check-full` through `/sase_monitor` — this epic touches the agent
launch path, the runner startup path, and the Rust core, all of which are in the
broadening set.

Manual verification for the land agent: launch a clan whose members wait on a common
predecessor (the `%w` shape that triggered this incident) together with an unrelated
ACE-launched agent, and confirm from the ledger that every agent received a distinct
workspace number and that no occupant conflict was reported.
