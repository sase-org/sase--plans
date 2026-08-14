---
tier: epic
title: One live agent per numbered workspace — close the monitor claim hole
goal: "A numbered `<project>_<N>` workspace checkout is never occupied by two live
  agents at once. Every process that works inside a numbered workspace holds the
  RUNNING-field claim for that exact number for as long as it is in there, the claim's
  PID is always a live process, and any code path that cannot satisfy that invariant
  fails loudly instead of silently running unclaimed.

  "
phases:
  - id: meta
    title: Record the agent's real workspace number in agent_meta.json
    depends_on: []
    size: small
    description:
      "meta: write and maintain `workspace_num` in every agent's `agent_meta.json` so
      downstream consumers stop reading `None`."
  - id: lookup
    title: Authoritative workspace-directory to workspace-number lookup
    depends_on: []
    size: small
    description:
      "lookup: add a registry-backed helper that resolves a checkout directory to its
      owning workspace number."
  - id: monitor-claim
    title: A monitor holds the claim on the workspace it runs in
    depends_on:
      - meta
      - lookup
    size: medium
    description:
      "monitor-claim: replace the silent `workspace_num = 0` fallback in monitor start
      so the monitor always claims the numbered workspace its command runs in, or
      refuses to start."
  - id: orphan
    title: A monitor handoff never orphans the starter's claim
    depends_on:
      - monitor-claim
    size: medium
    description:
      "orphan: make the runner's `monitored` shutdown skip conditional on the claim
      actually having moved to a live supervisor, so a dead-PID claim is never left
      behind for the stale-claim reaper."
  - id: followup
    title:
      Follow-up and family-attach launches never pair workspace 0 with a numbered
      directory
    depends_on:
      - meta
      - lookup
    size: medium
    description:
      "followup: normalize workspace number and directory together in the monitor
      follow-up and family-attach launch paths so a degraded launch moves out of the
      numbered workspace instead of squatting in it unclaimed."
  - id: finalizer
    title: The commit finalizer stops attributing pre-existing dirt to the agent
    depends_on: []
    size: medium
    description:
      "finalizer: capture a dirty-path baseline at runner start and exclude those paths
      from the finalizer's must-commit set, reporting them as pre-existing instead."
  - id: guard
    title: Occupancy diagnostics and an end-to-end regression exercise
    depends_on:
      - monitor-claim
      - orphan
      - followup
    size: small
    description:
      "guard: add doctor/inventory diagnostics for unclaimed-but-occupied and
      double-occupied workspaces, plus a regression test that replays the original
      incident sequence."
proposed_by: bbugyi200.athena.015
parent_bead: sase-lb
create_time: 2026-08-14 11:09:06
status: wip
---

- **PROMPT:**
  [prompts/202608/workspace_claim_invariant.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/workspace_claim_invariant.md)

# Plan: One live agent per numbered workspace — close the monitor claim hole

## Problem

Bead `sase-lb` recorded a confirmed live incident: two agents of the same project ran
concurrently in the same `<project>_14` clone. One was a legitimately allocated phase
worker; the other was a land-family agent whose in-flight, uncommitted edits then showed
up in the other agent's very first `git status`, tripping its post-completion commit
finalizer twice and nearly causing it to commit another bead's unverified work.

The workspace allocator itself is not broken.
`allocate_and_claim_workspace_from_content` in the Rust core picks the first number in
`10..999` that has no RUNNING-field row, and `plan_claim_workspace_from_content` rejects
a duplicate claim on any non-zero number, all under the ProjectSpec lock. The invariant
fails because a live agent can be working inside `<project>_N` while **no RUNNING row
for `#N` exists**, which makes `#N` a legitimate allocation for the next launcher.

## Root cause

Reconstructed from the incident's own artifacts and confirmed against currently running
agents. Four defects compose into the failure; the first is the trigger and each of the
rest independently keeps the invariant broken.

### 1. `agent_meta.json` never records `workspace_num` for an ordinary agent

`src/sase/axe/run_agent_directive_metadata.py` assembles the metadata dict and writes
`"workspace_dir": inputs.workspace_dir` (~line 161). `AgentMetadataInputs` has no
`workspace_num` field at all. The **only** writer of `workspace_num` is the
family-attach override further down the same module (~lines 284-287), which copies the
_parent's_ number.

Verified against three agents that were live while this plan was written, each holding a
real non-zero claim in the RUNNING field: every one of their `agent_meta.json` files has
`workspace_dir` pointing at the correct numbered clone and **no `workspace_num` key at
all**. The incident's own agents show the same shape.

So `meta.get("workspace_num")` is `None` for essentially every ordinary agent, while
`meta["workspace_dir"]` is correct. Consumers that read the pair silently disagree with
reality.

### 2. Monitor start silently degrades to a `#0` claim while running in a numbered workspace

`src/sase/monitor/start.py::_resolve_lane_start` (~lines 284-320) inherits the lane's
workspace claim only when

```
request.inherit_lane_workspace_claim
and cwd_matches_lane
and lane_workspace_num is not None      # <-- always None, per defect 1
and runner_pid is not None
```

and otherwise takes `resolved_workspace_num = 0`. Because of defect 1 the third conjunct
is never satisfied for a monitor started by an ordinary agent, so **every
`sase monitor start` from an agent workspace takes the `#0` branch**. `#0` is the
primary/placeholder identity: the Rust claim planner deliberately permits unlimited
duplicate `#0` rows, so that claim reserves nothing. Meanwhile the supervisor runs the
command with `cwd` set to the numbered workspace.

This is not a rare race. It fires on every monitored command, and this repo's own agent
instructions tell agents to run `just check-full` through `/sase_monitor`.

### 3. The starter's claim is then orphaned with a dead PID, and the reaper frees it

`sase monitor start` SIGTERMs the starter runner, whose execution loop returns the
`"monitored"` outcome (`src/sase/axe/run_agent_exec_monitor.py`). In
`src/sase/axe/run_agent_runner_lifecycle.py::finalize_runner_shutdown` the whole
workspace disposition block is skipped for that outcome:

```python
if not context.is_home_mode and state.exec_outcome != "monitored":
```

That skip is correct **only** when the claim was handed to the supervisor. After defect
2 it was not, so `#N` stays in the RUNNING field owned by the starter's now-dead PID.
The ACE agent loader
(`src/sase/ace/tui/models/_loaders/_running_loaders.py::load_agents_from_running_field`,
~lines 61-131) treats any claim whose PID is not live as stale and releases it. `#N`
returns to the free pool **while the monitored command is still running inside it**, and
the next launcher is handed that directory.

### 4. The follow-up agent re-enters the numbered workspace holding only `#0`

When the monitor settles, `src/sase/monitor/followup.py::launch_followup_agent` reads

```python
original_workspace_num = int(meta.get("workspace_num") or 0)   # 0, from defect 2
original_workspace_dir = str(meta.get("workspace_dir") or "")  # the numbered clone
```

and spawns the follow-up with that mismatched pair. The claim transfer of `#0` from the
supervisor fails or lands on the duplicate-permitted `#0` row, so the follow-up agent
starts with `cwd` in the numbered clone and a claim that reserves nothing.

The incident's RUNNING field still carries the resulting row: a pinned `#0` claim whose
`agent_meta.json` records `workspace_num: 0` next to a `workspace_dir` pointing at
`<project>_14` — the exact directory the other agent was legitimately allocated.

### Why the fix belongs here and not in the Rust core

The allocator, the claim planner, and the release planner already enforce mutual
exclusion correctly for every non-zero number. Nothing in
`../sase-core/crates/sase_core/src/agent_launch/mod.rs` needs to change: the callers are
handing it `#0` for work that occupies `#N`. Every phase below is Python-side caller
correctness. If any phase concludes a wire or planner change is genuinely required, that
change goes to `../sase-core` first (opened with the `/sase_repo` skill), with bindings
and tests, before the Python callers here are updated.

## Invariant this epic establishes

> If a process's working directory is the checkout for workspace `#N` of project `P`,
> then `P`'s ProjectSpec RUNNING field contains exactly one claim row for `#N`, that
> row's PID is live, and that PID is the process (or the supervisor/parent that owns
> it).

Corollary enforced throughout: **`workspace_num == 0` may only ever be paired with the
primary checkout directory.** Pairing `0` with a numbered checkout is a bug, never a
degradation.

---

## Phase `meta`: Record the agent's real workspace number in agent_meta.json

Make `agent_meta.json` state the workspace number the agent actually holds, so every
consumer stops silently reading `None`.

- Add `workspace_num: int` to `AgentMetadataInputs` in
  `src/sase/axe/run_agent_directive_metadata.py` and emit it in the assembled metadata
  dict next to `workspace_dir`. Thread the value from the runner state at the
  `extract_directives_and_write_meta` call site in
  `src/sase/axe/run_agent_runner_bootstrap.py`.
- A deferred (`%wait`) agent boots with the placeholder number and only later takes a
  real one in `src/sase/axe/run_agent_phases.py::claim_deferred_workspace`. After that
  claim succeeds, rewrite both `workspace_num` and `workspace_dir` in the agent's
  metadata (the existing `update_meta_field` helper is the right tool) so the recorded
  pair matches the claim. Today both stay at their placeholder values for the rest of
  the run.
- Preserve `workspace_num` across a runner re-exec in `preserved_agent_metadata` so a
  refreshed pass does not drop it.
- The family-attach override in the same module currently sets `workspace_num` from the
  parent's plan. It must not overwrite a number this run actually claimed: the run's own
  claim always wins, and the parent's value is only a fallback for a run that has not
  claimed anything yet.
- Home-mode agents keep `workspace_num: 0`, which is correct — their directory is not a
  numbered checkout.

Tests: an ordinary launched agent records the number it claimed; a `%wait` agent records
the placeholder before its wait and the claimed number after; a family-attach child does
not have its own claimed number clobbered by its parent's; the value survives a runner
re-exec.

## Phase `lookup`: Authoritative workspace-directory to workspace-number lookup

Several call sites know a checkout directory but not its number. There is already a
durable, authoritative mapping — the per-project workspace registry (`registry.json`,
see `src/sase/workspace_provider/registry.py`), which maps each workspace number to its
`checkout_dir`, including `0` for the primary.

- Add a public helper to `src/sase/workspace_provider/` that resolves a directory to its
  owning workspace number for a given project: consult the registry first, fall back to
  `WorkspaceStore.resolve` for the primary checkout, and return `None` when the
  directory is not a managed checkout of that project.
- Normalize before comparing: expand `~`, resolve symlinks, and treat trailing slashes
  as equivalent. The store emits managed checkout paths _with_ a trailing slash while
  most callers pass paths without one — a naive string compare silently misses.
- Do not parse the number out of the directory basename. The registry is authoritative
  and the basename convention is a rendering detail of `WorkspaceStore.resolve`.
- Export it from the workspace-provider package surface so the monitor and launch paths
  can use it without reaching into private modules.

Tests: primary checkout resolves to `0`; each managed checkout resolves to its registry
number; trailing-slash, symlinked, and `~`-relative spellings of the same directory all
resolve identically; a directory outside the project returns `None`; a registry missing
an entry for an existing directory returns `None` rather than guessing.

## Phase `monitor-claim`: A monitor holds the claim on the workspace it runs in

Rewrite the inheritance decision in `src/sase/monitor/start.py::_resolve_lane_start` so
the number the monitor claims is derived from the directory the command will actually
run in, never from an absent metadata field.

- Resolve the workspace number as: the lane member's recorded `workspace_num` (now
  reliable after `meta`), else the `lookup` helper applied to `request.cwd`. Keep
  `cwd_matches_lane` as the condition for reusing the lane member's _claim row_ and PID,
  but stop letting a missing number decide whether a claim is taken at all.
- Take the claim in all cases where the resolved number is non-zero:
  - lane member's PID is live and holds the row → transfer it to the supervisor, as
    today via `claim_monitor_workspace(..., transfer_from_pid=...)`;
  - otherwise → take a fresh `claim_workspace` on that number for the supervisor PID.
- If the resolved number is non-zero and the claim cannot be taken (another agent
  already holds it), `sase monitor start` must **fail** with an actionable error naming
  the workspace and the conflicting claim. Silently proceeding is what caused the
  incident. Roll back cleanly through the existing `_teardown_failed_member` path so a
  failed start leaves no half-created member.
- `resolved_workspace_num = 0` stays legal only when the resolved directory is not a
  numbered managed checkout — the approved-epic launch path
  (`src/sase/bead/epic_launch.py`, `inherit_lane_workspace_claim=False`) relies on it.
  Redefine that flag to mean "do not take over the starter's claim row", not "run
  unclaimed": when its cwd _is_ a numbered checkout, that path must still take a fresh
  claim.
- The existing rollback helper `undo_monitor_claim` in `src/sase/monitor/start_claim.py`
  already returns a transferred claim to a still-live starter; extend it so a _fresh_
  claim taken by this phase is released on the same failure paths.

Tests: a monitor started by an agent whose metadata lacks `workspace_num` still claims
the correct numbered workspace; the RUNNING row after start names the supervisor PID and
the monitor workflow label; a monitor started in a workspace claimed by a different live
agent refuses to start and leaves the RUNNING field untouched; the epic-launch path from
a non-numbered cwd still starts with `#0`; a supervisor that never acknowledges gives
the claim back and leaves no orphan.

## Phase `orphan`: A monitor handoff never orphans the starter's claim

Make the runner's shutdown skip conditional on the handoff having actually happened.

- In `src/sase/axe/run_agent_runner_lifecycle.py::finalize_runner_shutdown`, replace the
  unconditional `state.exec_outcome != "monitored"` guard with a check that the claim
  for `state.workspace_num` has genuinely moved: the RUNNING row for that number must
  exist, be owned by a **live** PID that is not this runner, and carry the monitor claim
  workflow label. When that holds, skip release exactly as today.
- When it does not hold, fall through to the normal disposition (release, or hold/pin
  for a visibly failed run). A monitored handoff whose claim transfer failed must not
  leave a dead-PID row behind — that row is what the stale-claim reaper converts into a
  double allocation.
- Keep this check cheap and non-fatal: read the claims once, and on any read error
  prefer releasing over leaking, since a stale row is the more dangerous failure.
- Audit the SIGTERM release handler in `src/sase/axe/run_agent_runner_signals.py` for
  the same hazard: the monitor path SIGTERMs the starter, so the handler must not race
  the supervisor's transfer and release a claim the supervisor now owns. Gate it on the
  same ownership check.

Tests: a successful monitor handoff leaves exactly one live-PID row for the workspace
and the starter releases nothing; a handoff whose transfer failed causes the starter to
release its own claim on shutdown; a killed monitored run does not double-release; the
SIGTERM handler does not release a claim already owned by a live supervisor.

## Phase `followup`: Follow-up and family-attach launches never pair workspace 0 with a numbered directory

Enforce the corollary — `0` only ever means the primary checkout — in the two launch
paths that carry a directory and a number as separate values.

- In `src/sase/monitor/followup.py::launch_followup_agent`, stop defaulting the number
  to `0`. Resolve it from the monitor member's metadata, else from
  `original_workspace_dir` via the `lookup` helper. With `monitor-claim` in place the
  monitor holds a real number, so the normal path becomes a genuine claim transfer to
  the follow-up agent.
- The existing degraded fallbacks stay, but must move the agent _out_ of the numbered
  workspace rather than squatting in it: whenever the follow-up ends up with
  `workspace_num = 0`, its `workspace_dir` must be the primary checkout
  (`_workspace_dir_for_num(project_name, 0)`), never the monitor's numbered directory.
  The existing degraded-reason text already tells the follow-up agent its workspace
  changed; extend it to state which directory it is actually starting in.
- Apply the same normalization in
  `src/sase/agent/_family_attach_launch.py::prepare_family_attach_launch`. Both branches
  copy `parent_workspace_dir`/`parent_workspace_num` from a plan whose values come from
  the parent's `agent_meta.json`; a numbered directory arriving with number `0` or
  `None` must be repaired through `lookup`, and an unresolvable pair must fail the
  launch with a clear error instead of producing an unclaimed occupant.
- Add a single shared assertion helper for "this (dir, num) pair is self-consistent" and
  call it from both paths, so a future third caller has an obvious thing to reuse.

Tests: a follow-up after a normal monitor inherits the monitor's numbered claim; a
follow-up forced onto the `0` fallback starts in the primary checkout and its prompt
says so; a family-attach plan carrying a numbered directory with a missing number is
repaired from the registry; an unresolvable pair fails the launch loudly.

## Phase `finalizer`: The commit finalizer stops attributing pre-existing dirt to the agent

The incident's blast radius came from the finalizer: the land agent was told, twice,
that its run fails unless it commits changes it did not make. It declined only because
the foreign change happened to name its own bead in a docstring. Make that attribution
structural rather than lucky.

- At runner start, before the agent's first turn, capture the workspace's already-dirty
  paths (the existing `git status --porcelain --untracked-files=all` reader in
  `src/sase/llm_provider/commit_finalizer_git.py` is the right shape) and persist the
  snapshot into the run's artifacts directory.
- In the finalizer's dirty-state assembly (`src/sase/llm_provider/commit_finalizer.py`
  and its `_git`/`_state` helpers), subtract the baseline paths from the set the agent
  is told it must commit. Report them separately as pre-existing changes the agent did
  not make and must not commit.
- Only paths whose status is unchanged since the baseline are excluded. A path that was
  dirty at start and was then modified again by this agent is this agent's problem and
  stays in the must-commit set — compare content/status, not just the path list.
- Cover linked repos and sidecars, not just the main checkout: `DirtyState` spans
  several repos and the incident's foreign edit could as easily have landed in one of
  them.
- Baseline capture must never fail a run: on any error, record no baseline and behave
  exactly as today.

Tests: a workspace dirty at start with a file the agent never touches yields a finalizer
prompt that lists it as pre-existing and excludes it from the must-commit set; a
baseline-dirty file the agent then edits stays in the must-commit set; a clean start is
byte-identical to today's behavior; a missing or corrupt baseline degrades to today's
behavior.

## Phase `guard`: Occupancy diagnostics and an end-to-end regression exercise

Make a recurrence visible immediately instead of via a corrupted commit, and pin the
incident sequence.

- Extend the workspace inventory in `src/sase/workspace_provider/inventory.py`, which
  already reports "multiple RUNNING claims reference workspace #N" and "RUNNING claim #N
  has no workspace registry entry", with the inverse condition: a numbered checkout that
  is some live process's working directory but has no RUNNING claim, and a numbered
  checkout that is the working directory of more than one live agent process. Reading
  `/proc/<pid>/cwd` for known agent PIDs is exactly how the original report was
  confirmed; keep it best-effort and Linux-guarded.
- Surface both through the existing workspace doctor checks in
  `src/sase/doctor/checks_workspace.py` so `sase doctor` reports them.
- Add the regression exercise that fails on today's code: an agent holding `#N` starts a
  monitor in its own workspace, the starter runner shuts down with the `monitored`
  outcome, a stale-claim sweep runs, and the allocator is then asked for a workspace.
  The assertion is that the allocator does not hand out `#N` and that `#N`'s claim is
  owned by a live PID throughout. Then settle the monitor and assert the follow-up agent
  lands in `#N` with the claim transferred to it.

Tests: the two new inventory conditions each produce their diagnostic and a clean
project produces none; the regression exercise passes only with all prior phases in
place.

## Verification

- Every phase runs `just check` before handing off; the combined tree runs
  `just check-full` through `/sase_monitor` before landing, since this epic touches the
  runner lifecycle, the launch path, and the monitor — all in the broadening set.
- Manual confirmation of the fix on a live host: start a monitored command from an agent
  workspace, then assert with `sase agent list -j` and the project's RUNNING field that
  the numbered claim survives the starter's shutdown with a live supervisor PID, and
  that a concurrently launched agent is allocated a different number.
