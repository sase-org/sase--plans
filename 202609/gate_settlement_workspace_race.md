---
tier: tale
title: Fix gate-shell settlement race that sent a coder into workspace 0
goal:
  An approved-plan coder never silently runs in the primary checkout; the gate claim
  survives settlement, VCS-tagged follow-ups fall back to a fresh pool workspace, and
  any degraded workspace is disclosed in the successor's prompt.
size: medium
proposed_by: bbugyi200.athena.09m
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.09m](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.09m.md)
  - [bbugyi200.athena.sase-yj.1](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-yj.1.md)
  - [bbugyi200.athena.sase-yj.2](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-yj.2/README.md)
  - [bbugyi200.athena.sase-yj.3](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-yj.3/README.md)
  - [bbugyi200.athena.toobig-51.claude_support.0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.toobig-51.claude_support.0/README.md)
  - [bbugyi200.athena.toobig-51.commit_tracking.0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.toobig-51.commit_tracking.0/README.md)
  - [bbugyi200.athena.toobig-51.file_completion_workers.0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.toobig-51.file_completion_workers.0/README.md)
  - [bbugyi200.athena.toobig-51.fleet_agents.0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.toobig-51.fleet_agents.0/README.md)
  - [bbugyi200.athena.toobig-51.providers.0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.toobig-51.providers.0/README.md)
  - [bbugyi200.athena.toobig-51.tailnet_discovery.0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.toobig-51.tailnet_discovery.0/README.md)
- **COMMITS:**
  - [a37e1fb](https://github.com/sase-org/sase/commit/a37e1fbaa0814875571e1bae01aadb27d958cde5)
    — fix(gate-shell): preserve follow-up workspace claims
  - [c235300](https://github.com/sase-org/sase/commit/c235300c6228bdd28f806760bdbd15284aa242c9)
    — feat(xprompt): add thin Python adapter for shared %queue/%q contract
  - [0770357](https://github.com/sase-org/sase/commit/0770357cd84dfc16b7bd1ac59f0bfae3dc3408b7)
    — feat(xprompt): wire queue directive into python runtime
  - [3f23a53](https://github.com/sase-org/sase/commit/3f23a53745761c38d0c25a268f634d98b5720bdf)
    — refactor(xprompt): retire wait_queue flag, make %queue directive unconditional
  - [f994133](https://github.com/sase-org/sase/commit/f994133fc4012c537b6c8d3d90ed8dc429c9dff9)
    — refactor(ace-tui): split fleet_agents.py into focused private modules
  - [a41e3c3](https://github.com/sase-org/sase/commit/a41e3c3d4775e4c892f0cae5f3d56fb6c3400ed0)
    — refactor(ace): split file-completion workers into domain modules
  - [2771ba2](https://github.com/sase-org/sase/commit/2771ba29553808f2d218d08000aeee1055fa405e)
    — refactor(dispatch): split providers.py into hook, inventory, and discovery modules
  - [b41d5b6](https://github.com/sase-org/sase/commit/b41d5b6e8d134440b7ba8ed1cec880285a88a128)
    — refactor(dispatch): split tailnet discovery helpers
  - [7e84441](https://github.com/sase-org/sase/commit/7e84441582f70f23b4df76f80c22a55e52b9873a)
    — refactor(llm-provider): split _claude_support.py into focused private modules
  - [c6387f3](https://github.com/sase-org/sase/commit/c6387f3b676449f1a98e91989698efa0a2da382a)
    — refactor(commit): split commit_tracking into focused modules

# Fix Gate-Shell Settlement Race That Sent A Coder Into Workspace #0

## Incident (evidence)

On 2026-09-08 the `08z` ace-run family planned and implemented the "pager bead links"
fix. `08z--plan` and `08z--gate` both ran in workspace #10, but the approved-plan coder
`08z--code` ran in **workspace #0 — the user's primary checkout**
(`~/projects/github/sase-org/sase`), implemented the plan there, and left a dirty
`tests/pager/_rendered_link_corpus.py` behind after the host commit finalizer ran.

The workspace-claim ledger (`~/.sase/logs/workspace_claims.jsonl`) shows the exact
sequence for workspace #10:

1. `123121 transfer wf=ace-gate pid=605379 tag=gate-shell-create ok=True` — gate
   created; claim moved to the `ace-gate` workflow, deliberately keeping the dead
   creator pid (the plan agent's turn ended at gate creation).
2. Gate answered at 12:32:07 (approve + commit). `settle_gate_shell` wrote the terminal
   done-marker (`gate_state: answered`) _before_ launching the follow-up.
3. `123238 release wf=ace-gate pid=605379 tag=ace-agents-loader ok=True` — the ACE
   agents-loader dead-PID reaper released the claim mid-settlement:
   `gate_claim_is_releasable` returned True because the marker was already terminal.
4. `123303 claim_next_axe wf=ace(run)-260908_092843 ok=True` — queued agent
   `sase-yf.land--plan` immediately claimed the freed workspace #10.
5. `123322 transfer ... ok=False err=workspace #10 with pid 605379 was not found` — the
   follow-up's claim transfer failed.
6. `123458 claim ... ok=False err=workspace #10 is already claimed` — the fresh-claim
   retry failed.
7. `launch_shell_followup` then fell back to workspace #0 by design
   (`gate_followup_outcome: launched-degraded` on the gate meta), and the
   degraded-workspace warning **never reached the coder's prompt** because plan-approval
   follow-ups use `raw_prompt=True`.

## Root causes (three distinct defects)

1. **Settlement race.** `settle_gate_shell` (`src/sase/gate_shell/settlement.py`) writes
   the terminal done-marker before `settle_shell_claim_and_followup` disposes of the
   workspace claim. `gate_claim_is_releasable` (`src/sase/gate_shell/claims.py`) treats
   "marker terminal + pid dead" as releasable, so any dead-PID reaper (the ACE agents
   loader's `_release_stale_running_claim` in
   `src/sase/ace/tui/models/_loaders/_running_loaders.py`, and the scheduler's
   `cleanup_stale_running_entries`) can free the workspace during the settlement window
   (which includes a `wait_for_starter` poll of up to 60s). The claims.py docstring even
   warns about exactly this failure: "or the gate's own settlement finds its workspace
   gone."
2. **Workspace #0 is a fallback for code-writing follow-ups.** `launch_shell_followup`
   (`src/sase/shells/followup.py`) degrades transfer-failure → same-number fresh claim →
   workspace #0. For a plan-accept coder whose prompt re-runs a full `#gh:` VCS
   workflow, landing in the user's pinned primary checkout is dangerous
   (clobbers/absorbs the user's uncommitted work; the leftover dirty file from this
   incident was later scooped into an unrelated commit).
3. **Raw-prompt follow-ups drop the degraded-workspace warning.**
   `_compose_policy_prompt` (`src/sase/gate_shell/followup.py`) discards
   `workspace_degraded_reason` in its `policy.raw_prompt` branch (it is swallowed by
   `**kwargs`), so the coder ran in the primary checkout with no warning at all.

The monitor/proc shell is _not_ exposed to defect 1: its claim pid is the live
supervisor process, so pid-liveness already protects it. The gate claim is uniquely
vulnerable because it intentionally parks on the dead creator pid.

## Changes

### 1. Close the settlement race: move the gate claim onto the live settling process

In `src/sase/gate_shell/settlement.py` (`settle_gate_shell`), before the first
`write_done_marker_and_update_index` call:

- When `gate_workspace_policy != "release"`, `workspace_num` is a nonzero int, and a
  claim matching (`workspace_num`, `gate_creator_claim_pid`) exists, transfer the claim
  to the settling process's own pid via `transfer_workspace_claim` (keep workflow
  `GATE_WORKSPACE_CLAIM_WORKFLOW` and the gate's artifacts timestamp). Use a distinct
  `caller_tag` (e.g. `"gate-shell-settle-hold"`) so the ledger shows the hold.
- Record the new holder in meta (e.g. `gate_claim_holder_pid`) with `update_meta_field`
  so it survives to the follow-up launch and is visible in artifacts.
- Put this in a small helper in `src/sase/gate_shell/start_claim.py` next to the other
  claim moves; a transfer failure must not abort settlement (log to the gate shell log
  and continue — behavior is then no worse than today).

In `src/sase/gate_shell/followup.py` (`launch_gate_followup_agent`), compute
`transfer_from_pid` from `gate_claim_holder_pid` when present, falling back to
`gate_creator_claim_pid`.

Why this shape: pid liveness is the first check every reaper performs, so a live holder
pid protects the claim for the whole settlement window with no new marker semantics; if
the settling process crashes, its pid dies and the existing "terminal marker + dead pid"
rule correctly frees the workspace. No change to `gate_claim_is_releasable` is needed —
add a comment there pointing at the settle-time hold.

Note `_restore_live_creator_claim` in `start_claim.py` still consults
`gate_creator_claim_pid` for the live-creator (`%auto` short-circuit) case; that path
settles with `creator_live=True` and never reaches the hold, so it needs no change —
verify this in tests rather than restructuring it.

### 2. Prefer a fresh pool workspace over workspace #0 for VCS-tagged follow-ups

In `src/sase/shells/followup.py` (`launch_shell_followup`), in the branch where the
same-number fresh claim raises `WorkspaceClaimError` (the workspace is genuinely claimed
by someone else):

- If the resolved `vcs_ref` is non-None (the composed prompt carries a live
  `#gh:`/`#git:` tag, so the successor re-runs full VCS workspace setup and does not
  need the original workspace's files), claim a fresh workspace from the unified pool
  via `claim_next_axe_workspace_dir` and spawn there with a new degraded reason
  ("original workspace was taken; launched in freshly claimed workspace #N instead").
- Only if the pool claim also fails, record the follow-up as **not launchable** (the
  existing `record_not_launchable` path already persists the composed prompt as an
  artifact for manual relaunch). A code-writing agent in the user's primary checkout is
  strictly worse than a saved prompt.
- Follow-ups whose prompt has no VCS tag (report-style: inspect monitor output,
  summarize gate results) keep today's workspace #0 fallback — they only read artifacts.

Add the new degraded-reason callback to `ShellFollowupWorkspace` and provide messages
for both shell kinds (`src/sase/gate_shell/followup.py`,
`src/sase/monitor/followup.py`). Keep the existing `workspace_zero_reason` for the
non-VCS path.

Watch the claim-ownership bookkeeping: the pool claim is taken by the launching
(settling) process but the spawned child re-claims/transfers inside
`launch_executor_workspace`; follow the same pid/hand-off convention the `spawn` path
already uses for the fresh-claim case so the ledger stays consistent and no claim leaks
if the spawn then fails.

### 3. Deliver the degraded-workspace warning in raw prompts

In `src/sase/gate_shell/followup.py` (`_compose_policy_prompt`), the `policy.raw_prompt`
branch must append `workspace_degraded_reason` (when non-None) to the returned prompt as
a clearly separated trailing paragraph (e.g. prefixed `WARNING (degraded workspace):`),
instead of dropping it. The successor must always know when it is not in the workspace
its family expected — especially now that the remaining degraded outcomes are rare.

## Tests

Extend the existing gate-shell and shells test modules (find them with
`rg -l "launch_shell_followup|settle_gate_shell|gate_claim_is_releasable" tests/`):

1. Settlement hold: settling an answered gate transfers the ws claim to the settling pid
   before the done-marker write; the ledger/claims show a live holder; the follow-up
   launch transfers from the holder pid. Simulate the incident: release the claim
   mid-settlement is _prevented_ because `is_process_running(holder)` is true for
   reapers.
2. Crash backstop: a claim held by a dead settling pid with a terminal marker is still
   releasable (existing `gate_claim_is_releasable` behavior unchanged).
3. Pool fallback: transfer + fresh claim both fail with a VCS-tagged prompt → pool
   workspace claimed and spawn receives it; pool exhausted → follow-up recorded
   not-launchable with the prompt persisted; non-VCS prompt → old workspace #0 fallback
   preserved.
4. Raw prompt warning: `_compose_policy_prompt` with `raw_prompt=True` and a degraded
   reason includes the warning text; without a degraded reason the prompt is
   byte-identical to today's output.

## Verification

Run the standard agent lane (`just check`) plus the targeted test files for
`gate_shell`, `shells`, and `monitor` follow-up/settlement. No TUI rendering, keymap,
CLI-surface, or flag changes are involved. The implementing agent must read
`sase/memory/lint_and_test.md` before finishing.

## Out of scope

- The plan-approval `saved_plan_path` pointing at another workspace's plans sidecar
  checkout (cosmetic; plan archived correctly).
- Reworking `move_gate_shell_claim`'s deliberate dead-pid parking during gate _pendency_
  — `gate_claim_is_releasable` already protects that phase.
- The scheduler's `cleanup_stale_running_entries` gate handling (it consults the same
  rule and is fixed by the same hold).
