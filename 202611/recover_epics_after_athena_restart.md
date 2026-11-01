---
tier: tale
title: Recover the epics stalled by athena's 2026-11-01 restart
goal:
  Every epic clan the restart left FAILED again has a live owner for each unfinished
  phase and a live land agent.
size: medium
decisions:
  core_fmt:
    ask:
      Should the recovery agent land a rustfmt-only fix to sase-core so its master CI
      turns green again?
    choices:
      fix:
        Land the formatting-only fix and record it on sase-1pj.6.1; the daily core cut
        can publish again
      note:
        Only note it on sase-1pj.6.1; release-chain waits stay blocked until that epic
        fixes it
    default: fix
    why:
      core CI is red on formatting alone since 73ec04a02e, blocking the core cut 3 epics
      wait on
    answer: fix
  gc_fix:
    ask: Should gc-fix (sase-1nf.14) be relaunched instead of closed as superseded?
    choices:
      relaunch:
        A fresh gc-fix worker replaces its run history and re-plans the remaining GC
        work
      settle:
        Close sase-1nf.14 as superseded by ready tasks sase-1f3 and sase-1nc; sase-1nf
        can land
    default: settle
    why: sase-1f3 and sase-1nc already own the remaining capture and producer work
    answer: settle
proposed_by: bbugyi200.athena.16p
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-01 09:46:26
status: wip
---

# Recover the epics stalled by athena's 2026-11-01 restart

## Goal

Every epic clan that athena's restart left FAILED in the Agents tab once again has a
live owner for each unfinished phase plus a live land agent. Phases that were already
stalled before the restart (finished work never closed, or work blocked on conditions
that have since cleared) are closed or continued too, so each epic can reach its land.
No phase loses its run history, and every relaunch uses SASE's sanctioned recovery
paths.

This plan changes no tracked files in the sase repo. The only repo edit is the optional
sase-core formatting fix (decision `core_fmt`).

## What happened (evidence gathered while planning)

athena shut down at about 07:26 local (12:26 UTC) on 2026-11-01 and came back up at
about 07:33. The shutdown killed everything in the 16 epic clans in the user's
screenshot: sase-1qh, sase-1pj.6.1, sase-1pj.7.1, sase-1q2.9, sase-1pt.9, sase-1pu.5.1,
sase-1pu, sase-1pj, sase-1pi, sase-1om, sase-1nv.5.1, sase-1oe, sase-1nv, sase-1nf.10.1,
sase-1nf, and sase-1n6.

**27 FAILED agents, all of them dependency waiters that had not started any LLM work.**
The TUI shows each one as "Runner exited without recording an error". They have no
`done.json`, and their PID is dead.

- 16 land agents: `<epic>.land` for each epic above.
- 11 phase waiters: sase-1pj.6.1.7, sase-1pj.6.1.8, sase-1pj.7.1.4, sase-1q2.9.3,
  sase-1q2.9.4, sase-1pj.8, sase-1pj.9, sase-1nf.11, sase-1nv.5.1.5, sase-1nv.5.1.6,
  sase-1nv.5.1.7.

**4 monitors were SIGTERM'd. None of their follow-ups launched.** Each is now `stopped`
with exit -15. Settlement crashed inside the long-lived supervisor because it was
running stale code: the install is editable, the primary checkout and install were
updated at 07:08, and every supervisor had started before that.
`sase monitor list -a -j` shows each one's verbatim requested next action under
`continuation.next_action`.

| Phase        | Monitor member (id)              | Starter turn to fork | Settlement error                                                               |
| ------------ | -------------------------------- | -------------------- | ------------------------------------------------------------------------------ |
| sase-1oe.4   | sase-1oe.4--mon-3 (g3we4m00jd30) | sase-1oe.4--4        | `release_workspace() got an unexpected keyword argument 'artifacts_timestamp'` |
| sase-1q2.9.2 | sase-1q2.9.2--mon (6arwy6wqewnh) | sase-1q2.9.2--plan   | `sase_core_rs ... does not expose binding 'decide_workspace_retention'`        |
| sase-1pt.9.2 | sase-1pt.9.2--mon (5zwebkfet1r3) | sase-1pt.9.2--plan   | same missing-binding error                                                     |
| sase-1n6.10  | sase-1n6.10--mon (hdywq4jz9fjt)  | sase-1n6.10--plan    | same missing-binding error                                                     |

The stranded-work reducer still reports these four phases as `owned` by their dead
monitors, so the TUI's "resolve stranded" never offers them.

**Several land agents were waiting on phases that had already stalled before the
restart.** None of these phases had a live owner:

| Phase                    | State                  | Evidence                                                                                                                                                              | Action            |
| ------------------------ | ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------- |
| sase-1pj.6.1.6           | kept since Oct 31      | Last notes say only the close remained, blocked on host landing. Its commit 406c57f126 is on origin/master, and `sase bead epic-symbols sase-1pj.6.1.6` is now empty. | verified close    |
| sase-1pj.7.1.2           | kept since Oct 31      | Note: "re-run sase bead close after the stitch lands". Commit 745d7b57b1 has landed, and epic-symbols is empty.                                                       | verified close    |
| sase-1pu.5.1.4           | open since Oct 30      | Note: "Close blocked on host-owned landing ... No code changes needed". Commit b1618348d4 has landed, and epic-symbols is empty.                                      | verified close    |
| sase-1nv.1               | kept since Oct 24      | Its agent finished the work: it closed sase-1nf.13 and wrote the gc handoff note on sase-1nf.14. It then declared `keep` on its own phase by mistake.                 | verified close    |
| sase-1qh.7               | unclosed since Oct 31  | It stopped at its precondition: the primary checkout had to include 4900d9a0f3. The primary now has it, and `sase agent sync -P` exists.                              | fork continuation |
| sase-1pi.1               | unclosed since Oct 29  | sase-core 0.43.2 and 0.43.3 have published since its last turn. However, core master CI is red again (see below).                                                     | fork continuation |
| sase-1om.8               | open since Oct 27      | Notes record 7 failing visual scenes and a precise diagnosis for the next agent.                                                                                      | fork continuation |
| sase-1nv.5.1.4           | unclosed since Oct 26  | The Athena canary wrapper exited 127 because `/usr/bin/time` is missing. Production `sase agent sync` never ran.                                                      | fork continuation |
| sase-1nf.14              | open (reopened Oct 30) | The land of child epic sase-1nf.14.1 left it for a user decision: schedule more capture and producer work, or settle.                                                 | decision `gc_fix` |
| sase-1nf.10.1.9.1 (epic) | land lost since Oct 21 | sase-1nv.5 (via sase-1nv.5.1) owns closing it and sase-1nf.10.1.9 under DECISION `check1_canary=fresh_canary`.                                                        | do NOT relaunch   |

**sase-core master CI has been red since 73ec04a02e** (`feat(staged-recovery)`, epic
sase-1pj.6.1). Both OSes fail `cargo fmt --check` on
`crates/sase_core/src/staged_handoff/mod.rs` and `recovery.rs` (run 41225348868). A red
CI blocks the daily core cut, and the release chain is waiting on that cut:
sase-1q2.9.2, sase-1oe.4, and sase-1pi.1, and through them sase-1n6.10. No bead tracks
the failure. sase-1oe.4's last checkpoint names sase-1pj.6.1 as its owner.

**Delegated phases.** These phases are blocked on their own child epics, and
`sase bead work` already skips them: sase-1pj.6→sase-1pj.6.1, sase-1pj.7→sase-1pj.7.1,
sase-1pu.5→sase-1pu.5.1, sase-1nf.10→sase-1nf.10.1, sase-1nf.10.1.9→sase-1nf.10.1.9.1,
and sase-1nv.5→sase-1nv.5.1. No gates are pending on any of these epics.

## Recovery tools, and why these ones

- **`sase bead work <epic> --yes-to-all`** is the documented recovery for dead runners.
  It reassigns every non-closed bead and relaunches every non-closed, non-delegated
  phase plus the land agent under their original names, and it never touches closed
  phases. Before relaunching a phase, it **deletes every prior artifact of that phase**
  (the REMOVE rows). It does not do this when a live agent descended from the phase name
  owns the bead. Such an agent is PRESERVEd as a "live retry"; `<phase>.f<N>` and
  `<phase>--<N>` names both qualify. So run it only after each in-progress phase is
  either closed or owned by a live continuation. Approving this plan authorizes exactly
  the `sase bead work` reruns listed below.
- **Continuations** use the TUI's stranded-Continue shape:
  `%id(bead=<phase>) #fork:<last turn>`. The forked child is named `<phase>.f<N>`. A
  monitor follow-up would have done the same thing by forking the monitor's starter
  turn. Agent-origin launches must go through `/sase_run`, which creates a
  LaunchApproval.
- **Do not use these:** `sase agent restart` (it cannot see crash-killed agents),
  `sase monitor resume` (it refuses `stopped` monitors), TUI retry (it allocates new
  names), and `%id(!...)` forced-reuse prompts.

## Steps

### 0. Re-verify state (turn 1)

The state may have moved since planning, so re-check everything before mutating
anything.

1. Run `sase agent list -a -j` and `sase monitor list -a -j -p gh_sase-org__sase`.
   Confirm that the 27 waiters are still dead and the 4 monitors are still `stopped`
   with no follow-up launched.
2. Get the stranded-work report through sase's interpreter:
   `from sase.bead.stranded import load_stranded_report; load_stranded_report('gh_sase-org__sase')`.
   Then use `sase bead read -r "<why>" <ids>` (notes included) for every phase named
   above.
3. Record any drift: for example, a phase someone has since closed, a newly live owner,
   or a pending gate.

Treat the drift as authoritative: skip anything that already recovered, and never
duplicate a live owner.

### 1. Verified closes (turn 1)

These phases are sase-1pj.6.1.6, sase-1pj.7.1.2, sase-1pu.5.1.4, and sase-1nv.1. Before
closing each one, confirm all three of the following:

- its commit (or, for sase-1nv.1, the sase-1nf.13 closure and the sase-1nf.14 handoff
  note) is present on origin/master or in the bead store;
- `sase bead epic-symbols <id>` is empty;
- its latest notes leave nothing but the close.

Then run
`sase bead close <id> --note "Recovery after athena restart: <evidence and SHAs>"`.

If any check fails, or the close is refused, do not force it. Move that phase into the
continuation batch (step 4), and move its epic to batch B (step 5).

Under `gc_fix=settle`, also run
`sase bead close sase-1nf.14 -R superseded --reason "Remaining capture and producer work is owned by sase-1f3 and sase-1nc" --note "<cite the 1nf.14.1 land blocker note>"`.

> [!decision] gc_fix = relaunch Leave sase-1nf.14 open. Batch A's
> `sase bead work sase-1nf` relaunches it fresh. Its dry run is expected to REMOVE
> sase-1nf.14--gate, sase-1nf.14--mon, and sase-1nf.14--plan as well.

### 2. Batch A: relaunch epics that need no continuation (turn 1)

The epics, in this order: sase-1pj.6.1, sase-1pj.7.1, sase-1pu.5.1, sase-1pu, sase-1pj,
sase-1nv, sase-1nf.10.1, sase-1nf.

For each epic:

1. Run `sase bead work <epic> --dry-run`.
2. Proceed only if every REMOVE row is an `(interrupted)` dead waiter or land agent from
   the list above. The one allowed exception is the sase-1nf.14 history under
   `gc_fix=relaunch`.
3. Then promptly run `sase bead work <epic> --yes-to-all`.

Expected relaunches once step 1 is done:

- sase-1pj.6.1: .7, .8, and land.
- sase-1pj.7.1: .4 and land.
- sase-1pu.5.1: land.
- sase-1pu: land.
- sase-1pj: .8, .9, and land.
- sase-1nv: land.
- sase-1nf.10.1: land.
- sase-1nf: .11 and land, plus .14 under `relaunch`.

Any other REMOVE or BLOCKED row means stop for that epic, diagnose it, and report it.
Never relaunch sase-1nf.10.1.9.1.

### 3. File the discovered SASE defects (turn 1)

Use `/sase_new_task` for each defect below. It routes each one to a new task bead, a +1
on an existing bead, or a `DISCOVERED ISSUE:` note on a causally related active epic
(candidates: sase-1pj.7.1 and sase-1pj.6.1).

1. Monitor settlement after a live `sase update` crashes on stale in-process code and
   drops the follow-up. Cite the four errors above.
2. A host-shutdown SIGTERM is recorded as `stopped`, `sase monitor resume` refuses
   `stopped` monitors, and the stranded reducer keeps reporting their phases as `owned`.
   Together these mean a reboot silently strands monitor-owned phases.
3. `sase agent restart <name>` cannot find agents that died without a done marker, even
   though the TUI lists them as FAILED. The cause: `find_named_agent` skips non-done
   records whose PID is dead.
4. Nothing relaunches `%w` waiters or land agents after a host restart. Recovery takes a
   manual `sase bead work` per epic. Compare `sase agent auto-restart`, which covers
   live updates only.
5. A phase whose close is refused only by upstream `--epic-symbol` rows declares `keep`,
   and nothing closes it after the host lands the Justfile edit. Examples:
   sase-1pj.6.1.6, sase-1pj.7.1.2, and sase-1pu.5.1.4.

Separately, record the sase-core rustfmt failure on sase-1pj.6.1 as a
`DISCOVERED ISSUE:` note naming 73ec04a02e and run 41225348868. Under `core_fmt=fix`,
also say that this recovery lands the fix.

### 4. Request the continuations (end of turn 1)

Build one `/sase_run` LaunchApproval request with one segment per phase below, joined by
`---` separator lines, with `max_slots` equal to the segment count. Set
`requester_continuation` to `resume_requester`, with a checkpoint pointing at step 5.

Every segment follows this shape:

```text
%auto
#gh:gh_sase-org__sase
%id(bead=<phase>)
#fork:<turn to fork>
%model:<model>@<effort>
<body>
```

`<model>@<effort>` is the forked turn's model and effort from its `agent_meta.json`, or
the monitor's `next_model` if set.

Rules for the segment bodies:

- Put any pasted next-action text or output between `%macros_enabled:false` and
  `%macros_enabled:true` lines.
- Make sure no pasted text contains a bare `---` line.
- Preflight each segment with `sase macro expand` **without** the `#gh:` line. That
  expansion runs the workspace setup step.

Stranded phases. Each body starts from the TUI stub: "Continue stranded phase `<phase>`:
`<last turn>` <cause>. Re-read `sase bead read <phase>` (notes included), finish the
phase, and close it with `sase bead close`. If it must stay open, arm a monitor or gate
before ending."

- **sase-1qh.7**, forking `sase-1qh.7`. Add: the step-1 precondition now holds (the
  primary checkout and the install include 4900d9a0f3). Run steps 2–6 per DECISION
  `cleanup_apply=direct`.
- **sase-1pi.1**, forking `sase-1pi.1--3`. Add: cores 0.43.2 and 0.43.3 have published
  since your last turn. Core master CI has been red again since 73ec04a02e, on rustfmt
  only; state whether a fix is landing per `core_fmt`. Wait for a green core master CI
  through `/sase_monitor` before judging acceptance.
- **sase-1om.8**, forking `sase-1om.8--2`. Add: refresh onto current master first, then
  follow the REMAINING WORK and DIAGNOSIS notes.
- **sase-1nv.5.1.4**, forking `sase-1nv.5.1.4--1`. Add: GNU time is absent on athena, so
  time with Python UTC stamps or bash `TIMEFORMAT`. Production `sase agent sync` never
  ran. Honor DECISION `check1_canary` and the phase's budget rules.

Monitor-killed phases (sase-1oe.4, sase-1q2.9.2, sase-1pt.9.2, sase-1n6.10). Each forks
the starter turn from the table above. The body says:

- monitor `<id>` (`<label>`) was SIGTERM'd by athena's restart at about 07:26 local on
  2026-11-01, after `<elapsed>`;
- it never completed, and its follow-up never launched (`<error>`);
- inspect the captured output with `sase monitor show <id>`;
- the requested next action, verbatim: `<next_action>`;
- re-check the awaited condition now, and if it still does not hold, re-arm the same
  wait through `/sase_monitor` instead of ending idle.

For sase-1oe.4, sase-1q2.9.2, and sase-1n6.10, also mention the core CI state and
`core_fmt`.

Submit the request through `sase launch request -f <file> -o json` in the foreground.
The turn ends when the descriptor prints.

### 5. Batch B: relaunch the rest (resumed turn)

If the LaunchApproval was rejected, timed out, or failed:

- do **not** run `sase bead work` for batch B (it would wipe these phases' history);
- report which epics remain unrecovered and why, and skip to step 7.

Otherwise:

1. Confirm that each continuation is live (RUNNING, QUEUED, or WAITING), is named
   `<phase>.f<N>`, and is the bead's assignee.
2. For each of sase-1qh, sase-1q2.9, sase-1pt.9, sase-1pi, sase-1om, sase-1nv.5.1,
   sase-1oe, and sase-1n6, run the step-2 dry-run guard. Each continued phase must show
   **PRESERVE**, and REMOVE rows may be only the `(interrupted)` waiters and land
   agents.
3. Then run `sase bead work <epic> --yes-to-all`.

### 6. sase-core formatting fix (resumed turn)

Under `core_fmt=fix`:

1. Open sase-core with `/sase_repo` and refresh onto its origin/master.
2. Run its pinned-toolchain `cargo fmt --all`. Keep the diff limited to the files CI
   flagged; no logic changes.
3. Run the fmt, clippy, and test gate that sase-core's AGENTS.md prescribes.
4. Let host completion land the fix through `/sase_final`.

No sase pin change is needed, because no binding changes.

> [!decision] core_fmt = note Skip this step. The step-3 note on sase-1pj.6.1 is the
> only action.

### 7. Verify and report (resumed turn)

Re-run `sase agent list -a -j`, the stranded-work report, and
`sase monitor list -p gh_sase-org__sase`. Then report per epic:

- which agents died in the restart;
- what was already stalled before it, and since when;
- what you closed (with evidence), continued, relaunched, or left alone, and why;
- the current live owner for every non-closed phase and land;
- any epic still lacking an owner, with the reason.

Also list the task beads and notes filed in step 3. Finish with `/sase_final`.

## Guardrails

- Never close an epic or plan bead, never set statuses by hand, and never answer or
  approve gates on the user's behalf.
- Never run `sase bead work` on an epic whose dry run would wipe the history of a phase
  that is neither closed nor live-owned. The only exception is sase-1nf.14 under
  `gc_fix=relaunch`.
- Leave sase-1nf.10.1.9.1 to the sase-1nv.5 chain.
- Change no tracked sase files. The only repo edit is the step-6 sase-core fix.
