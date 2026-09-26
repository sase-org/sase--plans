---
tier: epic
title: Finish the live dispatch, parity, and memory gates of sase-1aq
goal: Close sase-1aq and every promised descendant through verified released-build
  live acceptance, landed guidance, and normal bead landing.
parent_bead: sase-1aq
phases:
- id: recover_dispatch
  title: Reconcile the uncertain dispatch and close the accepted snapshot proof
  size: small
  depends_on: []
  description: 'recover_dispatch: reconcile Apollo''s uncertain launch by its original
    operation key, establish one active owner, and normally close the already evidenced
    snapshot phase.'
- id: unified_proof
  title: Complete the unified live dispatch and exact-operation matrix
  size: medium
  depends_on:
  - recover_dispatch
  description: 'unified_proof: finish the original unified live proof and the remaining
    fault cases on a matching released cohort.'
- id: dispatch_landing
  title: Land the original remote-dispatch epic chain
  size: medium
  depends_on:
  - unified_proof
  description: 'dispatch_landing: close the original dispatch acceptance beads and
    their ancestors through sase-xe with evidence and normal land owners.'
- id: parity_landing
  title: Prove and land deployed owner-to-viewer Agents parity
  size: medium
  depends_on:
  - dispatch_landing
  description: 'parity_landing: complete the production parity oracle and same-build
    live captures, then land sase-133.5 and sase-133.'
- id: publish_memory
  title: Publish the landed hold and dispatch guidance
  size: medium
  depends_on:
  - dispatch_landing
  - parity_landing
  description: 'publish_memory: publish the two authorized memory changes, close their
    task beads, and finish sase-1ae.4.'
- id: final_audit
  title: Audit the backlog and close sase-1ae and sase-1aq
  size: small
  depends_on:
  - publish_memory
  description: 'final_audit: finish the memory census and verify every requested descendant
    and plan before normal parent landing.'
proposed_by: bbugyi200.apollo.23
create_time: 2026-09-26 17:34:34
status: wip
bead_id: sase-1aq.10
---

- **PROMPT:** [prompts/202609/finish_1aq_live_closeout.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/finish_1aq_live_closeout.md)
- **BEAD:** [sase-1aq.10](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1aq/sase-1aq.10.md)

# Finish `sase-1aq` on the live Athena–Apollo cohort

## Outcome and current boundary

Close the existing `sase-1aq` epic and the hold, dispatch, parity, and memory
descendants it promises, with live acceptance and normal `done` outcomes. This is a
user-requested recovery plan for existing beads, not a new product design. Implement and
close the original acceptance gates in place; the phases here coordinate and verify
those gates. Do not create another implementation epic or replace a failed gate with a
follow-up task.

At the 2026-09-26 planning read, `sase-1aq.1`–`.4`, `sase-11l.11`, `sase-11l`, and
runtime adoption `.7.4` are closed. `sase-1aq.5`–`.9` remain in progress. Original
snapshot phase `sase-xe.16.11.7.14.6.7.5` has a detailed accepted Apollo proof but
remains in progress; `.7.6`, reopened `.7.13`, `.16.11.3`, `.16.11.5`, and `.16.10`
still need their respective proof or normal close. `sase-xe.16` and `sase-xe` are open.
`sase-133.5.4`, `sase-133.5`, `sase-133`, `sase-1ae.4`, `sase-1ae.5`, `sase-1ae`, and
memory tasks `sase-134`/`sase-ya` are non-closed. Re-read the live store before acting;
this snapshot is not a completion claim.

The immediate ambiguity is the earlier `sase-1aq.5--1` launch: its recorded handoff
reported `unsupported fleet launch store schema_version` and an uncertain Apollo
outcome. It was absent from the live agent lists when checked from apollo and Athena;
the later `sase-1aq` workers and lander were WAITING on Athena. Do not infer that the
launch failed, retry with a new operation key, or launch a duplicate worker based only
on those lists. Apollo is this agent's host and target; Athena is the dispatch
controller and screenshot viewer. Use authenticated `ssh athena` or the supported remote
command path for Athena-side operations, and inspect Apollo's actual local gateway,
journal, agent, and installed state on apollo. Never treat an Apollo-local view as an
Athena viewer capture.

Read `research:202609/stalled_epics_sase-1ah8_1aq_1ae.md`,
`plan:202609/finish_blocking_epics_and_memory.md`,
`plan:202609/launch_recovery_and_xe_closeout.md`,
`plan:202609/remote_parity_landing_repairs.md`, and
`plan:202609/close_memory_bead_backlog.md` through `sase artifact read`. Read the
original beads, notes, and current dependencies with `sase bead read`. Read any further
sidecar artifact through `sase artifact read`, not its file path. Open linked
repositories with `/sase_repo`; shared backend behavior and wire contracts belong in
`sase-core`, with the SASE Python/TUI side as adapter or presentation. Follow
`/sase_memory_write` for the authorized memory changes, and read current reference
guidance before editing. Use host finalizers for commits.

Reconcile agent, gate, monitor, and bead state on both hosts before recovering an
assignment. Keep one worker per original bead. Preserve raw prompts/checkpoints and
existing wait ownership; only use a supported retry or recovery after establishing the
prior worker is ended or abandoned. A handoff receipt, a healthy hello, a passing
fixture, or an automatic bead close alone does not satisfy acceptance. Record exact
versions, UTC timestamps, operation keys or redacted identifiers, check results, and
artifact references in the original beads.

## Phase `recover_dispatch`

Compare the original uncertain launch's operation key with Apollo's fleet launch
journal, target agent/process list, gateway logs, and any receipt or agent checkpoint on
Athena. Use same-key status/reconciliation, never a new-key replay; if a schema
migration or stale installed component caused
`unsupported fleet launch store schema_version`, correct the installed contract through
the supported update path, preserving enrollment and credentials, then recheck both
hosts' source SHA, `sase-core-rs` wheel, gateway/worker/AXE/ACE binary identities, and
authenticated machine status. The latest recorded good cohort was SASE
`0.17.1+1524.gfedf207c1`, core `0.34.73+40.g9f86897f8`, and Apollo gateway fleet schema
v6; verify present state instead of assuming it is unchanged. Use clean published
installs, not an editable core override. Record whether the uncertain operation created
one agent, no agent, or remains genuinely ambiguous and how that result was established.

Read `sase-1aq.4` and `.7.5` evidence. Its controlled agent, dismissal, death
reconciliation, 100+68 history pages, snapshot mismatch, and gateway restart were proved
on the released cohort; verify the supporting evidence and current relevance, then have
the proper owner normally close `.7.5` with an evidence-bearing reason. Preserve the two
`.4` proposed follow-ups: frozen snapshot/false freshness and live `status` versus
running bucket. If either still violates the unified acceptance matrix, repair it here
or in `unified_proof`; do not erase it as merely historical. Close this recovery phase
only when there is one owner, no uncertain unaccounted execution, and `.7.5` is normally
closed.

## Phase `unified_proof`

Complete `sase-1aq.5` and original `.7.6` on matching published versions. Use the one
bounded Athena-to-Apollo cohort required by the approved plans: a dispatched agent with
receipt and fresh owner/viewer visibility, nonempty output, exact-instance stop
confirmed on Apollo, same-key recovery after an uncertain reply and ACE restart without
duplicate execution, attention from an unfollowed remote agent while another tab/filter
is active, stale-gate refusal, composable project/machine queries, target-picker focus
and source preflight, useful healthy-host navigation beside a hung host, and recovery
after gateway restart. Use harmless controlled gates/questions and short-lived agents;
preserve unrelated live work and credentials. A test transport may inject reply loss,
but it must exercise production reconciliation; label fixture proof separately from live
panes.

Recheck the original `.16.11.3` requirements with a genuinely captured old locator that
is replaced before exact-instance rejection, and a successful authenticated healthy host
beside a real hung host. A fabricated locator and fast-failing peer do not meet them.
Confirm `.16.11.5` end-to-end launch/receipt, fresh catalog, follow, output, stop, and
restart behavior, and `.16.10` original live requirements. For `.7.13`, cover its
recorded freshness, bridge persistence, restart, explicit `%id`, and target-picker
incidents within the approved contract; attach a requirement-to-evidence matrix to the
original beads. Repair in-scope defects, reverify affected cases, and retain redacted
panes/payloads as audited artifacts. Run focused core and SASE tests plus
`sase tool run check`/`just check` on changed trees; run `just check-full` only if an
explicit current requirement or a check-full-only CI failure calls for it, through
`/sase_monitor`. Inspect visual diffs and any partial update manifest. Close `.7.6` and
`sase-1aq.5` only when the matrix is complete and traceable.

## Phase `dispatch_landing`

After unified proof, use the existing land owners in the order from the approved
closeout plan, reconciling the live store before each step: `.16.11.7.14.6.7`, `.14.6`,
`.14`; then reopened `.7.13`, `.16.11.3`, `.16.11.5`; then `.16.11.7`, `.16.11`,
reopened `.16.10`, `.16`, and root `sase-xe`. Include the already closed original
`.14.6.6` proof in the audit without reopening solely for attribution. At each ancestor,
review descendants, original notes/follow-up dispositions, post-child drift, checks,
linked plan status, and relevant epic-symbol/Symvision results. The original land agent
closes its parent and marks its linked plan done; a coordinator must not force-close or
use an automatic commit close to bypass proof. Preview any recovery and preserve a
healthy waiting lander; recover only a confirmed abandoned owner. Require a fresh
unbounded descendant census showing root and all descendants normally closed, and a
reviewed artifact tying released builds to the live Apollo proof. Then close
`sase-1aq.6`.

## Phase `parity_landing`

Complete `sase-133.5.4` against the real production owner loader, gateway serialization,
federation, and viewer renderer on the landed same-build cohort. Compare normalized
visible identities, family grouping, active/waiting and completed shells, statuses,
chips, runtime, nested shell counts, and current/compact-index paths. The September 20
pre-fix screenshots failed parity; the development binding replay reaching 84/84 named
identities is only repair evidence. Resolve or precisely scope the still-noted
workflow-step `×N`/shell-chip discrepancy against the actual promised node parity
contract, with a regression if it is part of that contract. Review changed goldens and
retain any unrelated visual issue under its existing owner.

From Apollo capture the settled owner Agents view; from Athena capture `machine:apollo`
close in time at identical viewport geometry with the ordered screenshot flow. Check
fresh identities and installed versions on both hosts, authenticated status without
false fleet-version skew, representative active/waiting and completed families, and
inspect both PNGs before saving them as audited artifacts. Exercise local and remote
screenshot driving without touching unrelated gates. If they differ, fix the responsible
owner, wire, or viewer behavior and repeat. Only then let the original `.5.4`, `.5`, and
`sase-133` owners normally close and mark plans done after their landing audits. Close
`sase-1aq.7` on that evidence.

## Phase `publish_memory`

Begin only once `sase-11l`, `sase-11l.11`, `sase-xe.16`, and `sase-133` are closed, as
the existing `sase-1ae.4` dependencies require. Read their landed contracts and current
`docs/xprompt.md`, `docs/remote_dispatch.md`, and memory `xprompts.md`; treat the
backlog audit as wording guidance rather than the final source. `sase-134` and `sase-ya`
are the explicit authorizations. Use `/sase_memory_write` and edit canonical notes only.
For `sase-134`, add concise `%hold` and standalone proc queue guidance, including other
pending work only, optional bounded TTL (currently default 2h/cap 12h), selectors/scope,
armer session/proc terminal release, running-work immunity, fail-open store, actual
CLI/rejection matrix, and correct `%queue` weight-0/capacity-at-dispatch behavior.
Preserve newer multiplier wording. Add the requested concise decision strand for pull,
TTL/fail-open and release scope with rejected alternatives, cost, and reopen condition.

For `sase-ya`, add a short `type: reference` `dispatch.md` pointer to the runbook and
one `%dispatch` table row. State actual selector, clean published source, same-key
recovery, enrollment/repair/quarantine, and picker/Machines no-network-probe behavior
using _session_ terminology. State Focus/Fleet counts only if the final parity proof
supports them; otherwise omit the claim and explain that scope correction in the close
reason. Run `sase memory init --no-commit`, `sase memory init --check`, inspect
generated diffs and authored links, read the new note/strand through audited
`sase memory read`, and run `just check`. Close both memory tasks individually with
evidence, then normally close `sase-1ae.4` and `sase-1aq.8`.

## Phase `final_audit`

Finish the assigned `sase-1ae.5` audit: read the original 15 memory-task outcomes,
verify both new close reasons, rendered memory, generated shims, and an unbounded census
of non-closed memory tasks. Resolve any newly discovered in-scope memory task or record
a specific blocker; never infer zero from a limited listing. Normally close `.5`; let
`sase-1ae`'s land owner review all five phases, linked plan, checks, and post-close
Symvision, then normally close the parent.

Finally re-read the full trees for `sase-11l`, `sase-xe`, `sase-133`, and `sase-1ae`,
plus `sase-134`, `sase-ya`, and all `sase-1aq` phases. Require every promised descendant
closed with legitimate resolution, relevant plans marked done, current checks and
released-build live artifacts, and no abandoned acceptance or landing owner. The
`sase-1aq` land agent performs the normal final close and plan update after checking
post-phase drift. If any condition fails, keep the appropriate original phase and this
closeout open with an exact repair owner and continuation. Do not use `--force`,
canceled resolution, or a status edit to declare success.
