---
tier: epic
title: "Agents-tab `,X` kill-and-edit for the last launched agent"
goal:
  Pressing `,X` on the Agents tab kills and re-edits the most recently launched agent of
  this ACE session — including one whose launch proc has not finished — so a premature
  <enter> can be undone instantly, edited, and resubmitted.
phases:
  - id: launch-record-stack
    title: Session launch-record stack
    depends_on: []
    description:
      "launch-record-stack: record every accepted launch in a bounded session stack
      (LaunchRecord state machine, ObservedProc return change, submit-site pushes,
      completion/failure stamping) with ordering-discipline tests; no user-visible change."
    size: medium
  - id: kill-last-keymap-resolved
    title: "`,X` action registration and resolved-branch behavior"
    depends_on:
      - launch-record-stack
    description:
      "kill-last-keymap-resolved: register kill_and_edit_last on key X across all eight
      keymap surfaces plus docs, refactor _kill_and_edit_agent to take an explicit target,
      and make `,X` reveal and kill-and-edit resolved launch records (bulk fan-out
      included), with an interim toast for in-flight records."
    size: medium
  - id: deferred-kill-inflight
    title: In-flight deferred kill
    depends_on:
      - kill-last-keymap-resolved
    description:
      "deferred-kill-inflight: make `,X` on an in-flight launch restore the prompt
      immediately and kill the launched agents from the completion callback, holding
      replacement launches until the kill settles, with typed-admission scoping, failure
      handling, and race tests."
    size: medium
proposed_by: bbugyi200.kellys_mbp.l
bead_id: sase-w8
status: done
---

- **BEAD:** [sase-w8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-w8/README.md)

# Plan: Agents-tab `,X` kill-and-edit for the last launched agent

## Goal

Pressing `,X` on the Agents tab kills and re-edits **this ACE session's most recently
accepted launch** — including one whose launch proc has not finished — so a premature
`<enter>` can be undone instantly, edited, and resubmitted.

`,X` is not "kill the focused row" and not "kill the newest disk-visible agent." It
resolves the last launch this session accepted, reveals it when a row exists, and runs
the existing `,x` kill-and-edit flow on that target. When the launch is still in
flight, it restores the prompt immediately and defers the kill until the launch proc
returns concrete `AgentLaunchResult`s.

## Session-scoped target

"Most recently launched" means the last launch **this ACE session accepted**, recorded
synchronously at submit time. Disk-scan alternatives (newest visible row,
`get_most_recent_agent_name`) are blind to the in-flight window — the case this feature
exists for — and can retarget onto clan members, monitors, or agents spawned *by* the
agent the user launched.

Record the launch at the moment it is accepted. `_submit_durable_proc` already returns
the placeholder `ObservedProc` (or `None` on a duplicate/scope rejection).
`_submit_launch_proc` must return that `ObservedProc` so submit sites can push a
`LaunchRecord` onto a bounded session stack (~8 entries):

```
LaunchRecord(
    proc_ids: list[str],          # placeholder IDs; >1 only for bulk-Patch fan-out
    prompt: str,                  # the resolved submitted prompt
    display_name / project_file / cl_name / launch context snapshot,
    results: list[AgentLaunchResult] | None,   # stamped at proc completion
    state: IN_FLIGHT | RESOLVED | FAILED | KILL_PENDING | CONSUMED,
)
```

Push a record in both `_submit_resolved_launch` and `_submit_one_bulk_patch`. Stamp
results or `FAILED` in `_on_launch_proc_complete`. Treat a bulk-Patch submission (N
procs from one gesture) as one logical record so `,X` undoes it as a unit. A
multi-prompt `---` launch (N agents from one proc) is likewise one record with N
results.

## Accepted-submit ordering

- **Only accepted submissions become the target.** A rejected Enter (`None` return)
  must not clobber the previous valid target.
- **Completion updates only its own record.** An older concurrent launch finishing late
  must not steal or overwrite the newest target.
- **`,X` consumes a record idempotently.** A second press re-focuses or re-mounts the
  same attempt; it must never fall through and kill the previous launch. Stale entries
  (already killed or dismissed) are popped and skipped.
- The bounded stack makes `,X` composable: launch two agents, press `,X` twice, get the
  obviously-right behavior.

## Never kill the launch proc

Agent runners are spawned as detached processes in their own session. Killing the outer
`sase run` proc after spawn orphans the child and destroys the only channel
(`AgentLaunchResult`) that identifies it. There is no stop API for operation procs
today, and SIGTERM bypasses partial-launch rollback. v1 never cancels the launch proc.

The prompt is fully known at submit time, so the *edit* half of kill-and-edit never
needs to wait. ACE cannot identify the launched agent until the launch proc completes
and returns `AgentLaunchResult`s; the deferred-kill branch waits for that payload and
then uses the ordinary kill path.

## The `,X` action

1. Pop the newest live `LaunchRecord` (skipping stale entries). None → toast
   `No recent launch to kill and edit`.
2. `RESOLVED` → map results to rows, reveal the first target with the existing
   navigation machinery so the action is legible, then run the same code `,x` runs:
   `_kill_and_edit_agent` takes an explicit target (defaulting to the focused agent),
   and multi-result records route through `_edit_and_relaunch_agents_bulk`. Same
   confirmation rule, same forced-name-reuse rewrite, same cleanup barrier.
3. `IN_FLIGHT` → deferred-kill branch: restore the prompt immediately, mark
   `KILL_PENDING`, hold the replacement launch, and kill returned results from the
   completion callback. No confirmation on this branch.
4. `KILL_PENDING` (repeat press) → re-focus the restored prompt; register nothing new.

Explicitly: **`,X` ignores marks** (the two keys are adjacent; say so in help);
**never hand a clan container to the kill path** (records hold concrete results);
a launch that *failed* discards the intent, releases the gate, and must **not**
double-stash the prompt — the failure path already stashes it.

Naming: action ID `kill_and_edit_last` on key `X`. Keep the retired
`kill_marked_and_edit` filter untouched so a stale user override cannot revive the old
action or collide with the new binding. Label: `Kill last launched agent and edit`.
Availability: runnable iff a live record exists, independent of the focused row;
conditional footer hint per the footer convention.

## Keymap, help, and default-config surfaces

A new leader action touches eight registration points plus docs:

1. `src/sase/default_config.yml` (`kill_and_edit_last: "X"`)
2. `LeaderModeKeymaps`
3. the leader-mode dispatcher
4. command label + `AGENTS_ONLY` scope
5. availability
6. the conditional footer
7. the help modal
8. `docs/ace.md`

Phase `kill-last-keymap-resolved` registers the action and implements the resolved
branch. While the in-flight branch is still unbuilt, an in-flight record may toast that
the launch is still starting.

## Resolved-branch behavior

Reuse `,x`'s exact confirmation rule: `ConfirmKillModal` iff the target has a live PID;
dismiss silently otherwise. `,X` can land on an agent launched twenty minutes ago, and
silently killing that is a real footgun.

Reveal the first matched target best-effort. Forced name reuse and relaunch barriers
remain shared with `,x`. Bulk launch order maps to bulk prompt-pane order: N results
yield N kills and N panes.

## In-flight deferred kill

`,X` while the record is `IN_FLIGHT` does two independent things:

- **Now:** mount the prompt bar seeded from the record's prompt, mark the record
  `KILL_PENDING`, hold the replacement launch, and toast that the launched agent will
  be killed when its launch finishes.
- **At proc completion:** after the record is stamped, see the pending intent, kill
  every returned `AgentLaunchResult` through the ordinary kill path, and settle from
  `on_settled`. Join results to rows via the existing launch-delta artifact-dir helper.

No confirmation on this branch: there is no row yet, and a modal firing minutes later
would be absurd. The `,X` press *is* the confirmation.

Repeat `,X` during one flight is idempotent and never advances to the previous launch.
A failed launch while intent is pending discards the intent, releases the gate, warns,
and leaves the prompt stashed exactly once.

## Relaunch-hold invariant

`apply_force_reuse_launch` wipes old name state; a late cleanup write could resurrect
the name the replacement just claimed. That is the single most important invariant to
preserve.

Hold the replacement launch for the whole submit → kill-settled span, not just the
cleanup leg. Treat "a pending launch kill exists" as an additional condition in the
relaunch-cleanup pending check so the existing hold parks the submit through the launch
window and hands off to the ordinary cleanup barrier when results arrive. Give the
pending-kill leg its own warn-and-release budget (implemented as 180 seconds); the
existing 30-second timeout continues to govern only the cleanup leg it was tuned for.
If the launch outlives that budget, warn-and-release: the prompt is already restored;
the auto-kill is abandoned and the user can `,x` the row when it appears. Never silently
relaunch into a live name.

Intent is session-local. ACE restart mid-flight drops the record; the agent appears and
ordinary `,x` applies. Cross-restart durability is out of scope for this epic.

## Typed-admission v1 limitation

A prompt containing `%if`/`%proc` dispatches through typed admission. When the
completion payload has `admission_complete: false`, a detached coordinator may keep
launching remaining units after the launch proc has exited. Killing the returned
`AgentLaunchResult`s therefore under-kills a gated launch.

v1 scoping: when the deferred kill fires on a payload with `admission_complete: false`,
kill the returned results and toast that gated units continue in the background.
`WAITING`/`QUEUED` targets take the dismiss path. A true "abort launch bundle"
operation belongs in `sase-core` and is follow-up, not a blocker for `,X`.

## Failure modes

| Scenario | Required behavior |
| --- | --- |
| No launch this session | Toast, do nothing. |
| Submission rejected (dup/scope) | Prior target unchanged. |
| Launch fails while intent pending | Discard intent, release gate, warn; prompt stashed exactly once. |
| `,X` twice during one flight | Idempotent; never advances to the previous launch. |
| Older launch completes after newer target set | Updates its own record only. |
| Target already killed/dismissed by hand | Pop, try next record, or warn — never kill an unrelated row. |
| Multi-prompt / bulk-Patch record | Kill all children of the record; one pane each, in order. |
| Typed launch, `admission_complete: false` | Kill returned results; warn that gated units continue (v1). |
| `WAITING`/`QUEUED` target | Dismiss path. |
| Launch outlives barrier budget | Warn-and-release; prompt already restored; no silent relaunch race. |
| ACE restart mid-flight | v1: no record, toast; agent appears and `,x` applies. |
| Prompt bar already holds a draft | Same as `,x`: unmount, draft goes to history/stash. |

The `,X` handler does no synchronous disk or proc-store work.

## Verification

Focused coverage, beyond keymap/dispatch/availability assertions:

1. Accepted-target ordering — a rejected submit and a late-finishing older launch never
   steal the slot.
2. Pending-placeholder race — press `,X` before the durable ID exists, release the
   submit worker, assert the kill intent is found via the existing callback bridge
   exactly once.
3. In-flight branch — prompt bar mounts before proc completion, kill fires from the
   completion callback, relaunch stays parked until the kill settles, forced name reuse
   preserved.
4. Failed launch — intent discarded, prompt stashed exactly once.
5. Fan-out — N results → N kills → N panes in order.
6. `WAITING`/`QUEUED` teardown.
7. `,X` handler does no synchronous disk/proc-store work.

Homes: launch-record state-machine tests, resolved and in-flight `,X` tests,
keymap-dispatch, command-availability, keymap-registry (including the retired
`kill_marked_and_edit` override drop), and the PNG snapshot suite for footer/help
changes. `just check` is the agent lane.

## Provenance

This plan was reconstructed during landing of child epic `sase-w8.4` (phase
`sase-w8.4.2`) because the canonical file `plan:202609/kill_and_edit_last_launch.md`
was missing from both managed plan roots and the plans repository had no historical
copy. The reconstruction uses only the already-approved contract from:

- `research:202609/agents_tab_kill_last_launch/agents_tab_kill_last_launch.md`
  (consolidated `,X` research this plan originally derived from)
- Canonical bead records `sase-w8`, `sase-w8.1`, `sase-w8.2`, and `sase-w8.3`

It does not invent new scope. `status` stays `in_progress`; the resumed `sase-w8` land
agent owns the transition to `done` after normal parent closure.
