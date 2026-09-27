---
tier: epic
title: Close the original sase-1aq live gates in place
goal:
  The original remote-dispatch, owner-to-viewer parity, and dispatch-memory beads pass
  their own acceptance criteria and close normally. Work is not handed to owners that
  are not running.
parent_bead: sase-1aq.10.7
phases:
  - id: fencing_proof
    title: Prove healthy-beside-hung host and real-locator fencing
    depends_on: []
    size: medium
    description:
      "fencing_proof: finish sase-xe.16.11.3 in sase-core (TLS-honoring RemoteHost,
      successful healthy host beside a hung host, captured-locator rejection) and close
      it normally."
  - id: exact_ops_receipts
    title: Make exact remote stop and retry settle certainly
    depends_on: []
    size: large
    description:
      "exact_ops_receipts: resolve the uncertain-receipt, catalog-lag, and killed-row
      reaping issues recorded on sase-xe.16.11 so exact remote operations meet
      acceptance."
  - id: live_matrix
    title: Run the Athena-driven live matrix and close the dispatch phases
    depends_on:
      - fencing_proof
      - exact_ops_receipts
    size: medium
    description:
      "live_matrix: install matched builds on both hosts, drive Athena over SSH through
      the full viewer and exact-operation matrix, and close the original dispatch phase
      beads that pass."
  - id: parity_capture
    title: Capture same-build owner and viewer parity and close sase-133.5.4
    depends_on:
      - live_matrix
    size: medium
    description:
      "parity_capture: capture matched Apollo owner and Athena machine:apollo Agents
      panes at identical geometry, compare them, and close sase-133.5.4 normally."
  - id: ancestor_landing
    title: Audit and land the original remote-dispatch and parity ancestors
    depends_on:
      - live_matrix
      - parity_capture
    size: medium
    description:
      "ancestor_landing: land the sase-xe and sase-133 ancestor chains bottom-up with
      full land audits, then close sase-1aq.5, .6 and .7 normally."
  - id: dispatch_memory
    title: Publish dispatch memory and close the memory backlog
    depends_on:
      - ancestor_landing
    size: medium
    description:
      "dispatch_memory: publish the sase-ya dispatch reference note, then close sase-ya,
      sase-1ae.4, sase-1aq.8, sase-1ae.5, sase-1ae and sase-1aq.9 normally."
proposed_by: bbugyi200.apollo.sase-1aq.10.7.land
create_time: 2026-09-26 21:57:02
status: wip
---

- **PROMPT:**
  [prompts/202609/1aq_close_original_gates.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/1aq_close_original_gates.md)
- **PARENT:**
  [202609/1aq_remaining_acceptance.md](https://github.com/sase-org/sase--plans/blob/main/202609/1aq_remaining_acceptance.md)

# Close the original sase-1aq live gates in place

## Why this plan exists

This is the remainder of epic sase-1aq.10.7, which is itself the remainder of
sase-1aq.10. Both epics closed every phase, but none of the original beads they were
meant to close actually closed. Their phases handed each live gate to "original owners"
through PROPOSED FOLLOW-UP notes, and none of those owners was running. That loop ends
here. Unless a phase says otherwise, **each phase closes the original beads in its scope
itself**, with a normal `sase bead close` once that bead's own acceptance criteria pass
on real evidence. There are two exceptions:

- If a live or WAITING agent, gate, or monitor already owns a bead, leave that bead to
  its owner. Record the owner's identity on the bead.
- Never close on partial, fixture-only, or proxy evidence. Never use `--force`.

A phase may end without closing its targets only for a concrete blocker, meaning a
failure it reproduced and could not fix inside its scope. Before ending, it must record
that failure on the affected original bead. "Needs an owner" and "needs a matched build"
are not blockers: installing builds and driving Athena are this plan's work.

## Accepted evidence (do not redo)

- The uncertain operation from sase-1aq.10.1 is reconciled, the stale fleet store was
  reset, and snapshot bead sase-xe.16.11.7.14.6.7.5 is closed.
- sase-1aq.10.2 fixed the doubled gateway bridge command (29f8240df5) and proved one
  live Athena-to-Apollo dispatch: receipt, owner RUNNING/DONE, reply and artifacts,
  source preflight, same-key duplicate refusal, and gateway restart.
- sase-1aq.10.7.1 fixed the lookup in `src/sase/ops/commands/machine.py` (fb0b91edce).
  `catalog_sync` now sends `include_terminal` and exact-matches `agent_id`,
  `agent_session_id` or `agent_label`. Live: dispatch-39f835d3… exact stop killed the
  Apollo pid, and same-key retry ×3 produced a single `.r0`. Evidence is on
  sase-xe.16.11.7.14.6.7.6 and sase-1aq.5.
- sase-1aq.10.7.2 fixed stale `%dispatch` Target/Source chrome after stack rebuilds
  (afca222271). The pilot and dispatch lanes passed at the sase-1aq.10.7 landing.
- sase-1aq.10.7.3: the production facts and roster oracles are green, the version
  diagnostics show no false skew, and the ×N rule is resolved: `docs/remote_dispatch.md`
  documents shell-only ×N (step rows are not served). An owner-only capture exists as
  `file:explicit:63581737124ee47427942851`.
- sase-1aq.10.7.4 pre-verified the dispatch source facts in `docs/remote_dispatch.md`
  and confirmed that sase-ya is the only non-closed memory task and that sase-134 is
  closed.

Read the bead notes on sase-1aq.10.7 and its four phases before starting. Read the
original beads' own notes and linked plans through `sase bead read` and
`sase artifact read`.

## Operating rules

- **Hosts.** Apollo owns the agents and runs this epic's workers. Athena is the dispatch
  controller and viewer. Apollo agents can reach Athena through `ssh athena` (Tailscale
  SSH), and Athena already has `apollo` enrolled (`sase machine list`). Run every
  viewer-side step on Athena through that SSH path. "No remotes configured" on Apollo is
  expected and is not a blocker. Never present an Apollo-local view as an Athena
  capture.
- **Builds.** Before any live proof, bring both hosts to the same published sase and
  sase-core build through the supported update path. Record the exact installed versions
  and the gateway, AXE and ACE identities on each host. Preserve enrollment and
  unrelated work. Never replay an uncertain operation under a new key.
- **Code placement.** Shared fleet and dispatch behavior belongs in sase-core; open it
  with `sase repo open sase-core`. A binding change needs a pin move in
  `sase-core-revision.txt` (see `docs/rust_backend.md`). Verify changed trees with
  `just check` through `sase tool run`. Do not run `just check-full`.
- **Known clean-base reds.** The clean-base rename and proc failures belong to active
  epic sase-1ab and to sase-th. The `test_dispatch_federation` `ipc_client` failures
  seen at landing were `AF_UNIX path too long` from a long workspace tmp path. Do not
  count either as dispatch work. Do not file duplicates.
- **Evidence.** Keep redacted panes, payloads and PNGs as audited artifacts with UTC
  timestamps. Put requirement-to-evidence rows on each bead you close.
- **Prompts.** Use sleep-free prompts for exact-output assertions.

## Phase fencing_proof

Finish sase-xe.16.11.3 against its land-audit note #4, in the linked sase-core checkout:

1. Make `RemoteHost` honor the validated `plan.tls` (the TLS pinning gap).
2. Add an authenticated HTTPS gateway fixture that succeeds.
3. Replace `worker_bounds_deadline_and_preserves_fast_host_beside_hung_host`'s
   closed-port "fast host is not ok" assertion with a real proof: a genuinely hung host,
   and a healthy host beside it that returns usable rows within the deadline.
4. Rewrite the replacement-instance test so it captures a real old locator from the
   running fixture, replaces the fixture instance, and then rejects the captured
   locator. Do not use a fabricated `other-run` locator.

Keep the existing bootstrap, stale-revision and instance-enforcement, and fault-overlap
coverage. Update the binding and the sase pin if they change, then run focused cargo
tests and `just check`. Close sase-xe.16.11.3 normally if every criterion in its
description and notes is met.

## Phase exact_ops_receipts

Resolve the three DISCOVERED ISSUE notes that the sase-1aq.10.7 landing recorded on
sase-xe.16.11:

- **(a) Uncertain receipts.** Remote stop and retry receipts settle uncertain on the 5s
  request timeout even though Apollo applied the effect.
- **(b) Catalog lag.** Fresh dispatch rows stay invisible to `catalog_sync` for about 7
  minutes, so an exact stop inside that window finds nothing.
- **(c) Reaped killed rows.** Killed fleet-dispatched rows are reaped within about 30
  minutes, which makes retry-after-kill unaddressable.

First trace each cause in sase-core and in the owner store. Then choose and implement
the supported contract, for example:

- a mutation acceptance window or asynchronous receipt polling that settles the actual
  outcome before any replay;
- provisional launch rows served from the settled receipt, or a documented and tested
  visibility bound;
- an explicit reaping and retention rule for killed rows that keeps retry context as
  long as retry is advertised.

Whatever you choose, keep uncertain operations non-replayable, keep cross-project
isolation, and keep stale-locator refusal. Add focused core, binding and Python
regressions. Update `docs/remote_dispatch.md` where behavior or documented windows
change. Move the pin if the binding changes, then run `just check`.

## Phase live_matrix

On matched builds, drive Athena over SSH against Apollo and finish the original
sase-xe.16.11.7.14.6.7.6 matrix:

- unfollowed-remote attention while another tab or filter is active;
- stale-gate refusal;
- composable project and machine queries;
- target-picker focus and clean-source preflight;
- a useful healthy host beside a genuinely hung host;
- exact stop and same-key retry, now with settled receipts;
- retry-after-kill according to the exact_ops_receipts contract;
- gateway restart.

For sase-xe.16.11.5, sase-xe.16.10 and reopened sase-xe.16.11.7.13, record one
requirement-to-evidence matrix that covers:

- receipt
- catalog
- follow
- output
- stop
- restart
- explicit `%id`
- picker
- freshness
- bridge persistence

Reuse accepted evidence rather than reopening beads for attribution. Fix failures, rerun
the focused tests and `just check`, inspect any visual goldens you touch, and attach
audited live evidence.

Close these beads normally, in order: .7.6, .7.13, .16.11.5, .16.10. Close each one only
when its own criteria pass. If a bead fails, fix it inside this phase or record the
reproduced blocker on that bead.

## Phase parity_capture

Use the same builds as live_matrix. Through the ordered `sase screenshot` flow, capture
the settled Apollo owner Agents pane and Athena's `machine:apollo` Agents pane close in
time at identical geometry. Drive the Athena capture over SSH.

Inspect both PNGs. Compare:

- visible identities;
- family membership;
- active, waiting and completed status;
- chips and runtime;
- shell ×N (shell-only, per contract);
- current and compact-index paths.

Register both captures as audited artifacts. Recheck sase-133.5.4's own proposals #3–#5
(fleet golden drift, step-row ×N, fixture data leaking into a real home). Route each one
that is still live through `/sase_new_task` or a note, rather than leaving it
unrecorded. Fix any mismatch and repeat. Close sase-133.5.4 normally when parity holds.

## Phase ancestor_landing

Land the original ancestors bottom-up:

- sase-xe.16.11.7.14.6.7, then .14.6, then .14;
- sase-xe.16.11.7;
- sase-xe.16.11, which also covers .16.11.3's siblings;
- sase-xe.16, then sase-xe;
- sase-133.5, then sase-133.

At each ancestor, first check whether a live or WAITING land agent owns it. If one does,
leave the ancestor to that agent and record its identity. Otherwise, perform the full
land audit on the ancestor:

- read every descendant and its notes;
- check post-start drift;
- dispose of every PROPOSED FOLLOW-UP through `/sase_new_task` or a recorded decline;
- run `sase bead epic-symbols` and retire any entries;
- check current verification;
- close the ancestor normally with an audit note;
- run `just symvision`;
- set its linked plan file to `status: done`.

Stop the chain at the first ancestor with a genuine unfinished descendant, and record
the blocker on it. Once their targets are closed, close sase-1aq.5, sase-1aq.6 and
sase-1aq.7 normally, each with a requirement-to-evidence note.

## Phase dispatch_memory

Confirm that sase-xe.16 and sase-133 are closed. Then use `/sase_memory_write` and the
existing sase-ya authorization to publish two things:

- a short `type: reference` note `sase/memory/dispatch.md` that points to
  `docs/remote_dispatch.md`;
- one `%dispatch` row in `sase/memory/xprompts.md`.

Before publishing, recheck these facts against the source:

- selector and V1 non-composition;
- the clean published-source preflight;
- same-key recovery, including the exact_ops_receipts contract;
- enrollment, repair and quarantine;
- picker and Machines behavior without a network probe;
- session terminology;
- shell-only ×N.

Include Focus/Fleet count claims only where landed parity supports them. Keep the hold
guidance and the queue multiplier wording intact. Run `sase memory init --no-commit`,
`sase memory init --check`, the audited memory reads, and `just check`, and inspect the
generated changes.

Then close sase-ya with its actual scope, followed by sase-1ae.4 and sase-1aq.8.

For sase-1ae.5, compare the original memory-task outcomes against an unbounded census of
non-closed memory tasks, and confirm that sase-134 is still closed. Then close
sase-1ae.5, and land sase-1ae with the same full land audit described in
ancestor_landing, unless a live land agent owns it. Close sase-1aq.9 only after its own
audit passes.

Do not close sase-1aq.10, sase-1aq.10.7, or sase-1aq. Their land agents resume through
`parent_bead` after this epic lands.
