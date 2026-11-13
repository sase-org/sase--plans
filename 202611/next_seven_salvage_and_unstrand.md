---
tier: epic
title: 'Next seven: salvage the display-name fix, then unstrand the fleet (sase-zo,
  sase-1s2, sase-1re, sase-1o1, sase-1qm, sase-1qi, sase-1db)'
goal: Complete and close the seven ready task beads ranked by the 202611 next-seven
  research report, running the phases strictly one at a time.
decisions:
  qr_ride_along:
    ask: Should the agent-identity phase also fix sase-1qr (Rust scans drop process_identity)?
    choices:
      include: The agent-identity phase fixes sase-1qi and sase-1qr together; the
        epic closes both beads
      defer: The phase fixes only sase-1qi; sase-1qr stays ready for a later ranking
    default: include
    why: Three researchers flagged it, and it is the other half of agent-identity
      correctness.
    answer: include
  sase29_salvage:
    ask: Should the salvage phase also rebase and land the stranded sase_29 signature-skew
      diff?
    choices:
      sase21_only: Salvage only the sase_21 fix sase-zo needs; record sase_29's state
        for the owner
      both_workspaces: Salvage the sase_29 diff too, before workspace retention reclaims
        it
    default: sase21_only
    why: sase-1ra.7 owns the sase_29 diff and resuming that epic is the report's own
      recommendation.
    answer: sase21_only
  reopen_guard:
    ask: Should the salvage phase extend the carried_repo guard to reopen beads whose
      successor never adopts?
    choices:
      include: The salvage phase lands the reopen guard (mirroring sase-1oh.4) alongside
        the -p fix
      defer: Leave the guard to epic sase-1ra phase .8, which owns the adjacent staged-monitor
        surface
    default: defer
    why: sase-zo closes on the -p fix alone; sase-1ra.8 owns the structural reopen
      repair.
    answer: defer
phases:
- id: salvage-display-name
  title: Salvage and land the verified -p display-name resolution fix (sase-zo)
  depends_on: []
  size: medium
  description: 'salvage-display-name: sase repo list/log/open/path -p still resolves
    the project token as a directory key only, so `sase repo list -p sase` exits 2
    on master (+10 corroboration). The complete, verified fix — registry-canonicalized
    -p accepting key, display name, or alias, with regression tests — sits uncommitted
    in the sase_21 workspace clone on athena (base 99932634cd). Salvage that diff,
    rebase it onto current master, re-run its tests, and land it before workspace
    retention reclaims the state.'
- id: sigterm-import
  title: Stop runner module imports from replacing the process SIGTERM handler (sase-1s2)
  depends_on:
  - salvage-display-name
  size: small
  description: 'sigterm-import: importing sase.axe.run_agent_runner (line 87) or sase.axe.mentor_runner
    (line 30) installs a soft SIGTERM handler at import time, replacing the caller''s
    handler in every importer (pytest workers, tool runs). Move the install into the
    runner-owned entry lifecycle, keeping production shutdown semantics, and add fresh-interpreter
    import plus real-entry signal tests.'
- id: monitor-ownership
  title: Stop done monitors with stale followup_pending from owning beads forever
    (sase-1re)
  depends_on:
  - sigterm-import
  size: medium
  description: 'monitor-ownership: a monitor member that is done but still reports
    followup_pending=true (stale next_action, no followup_outcome) reduces to owned/monitor
    in sase-core stranded_work/classify.rs owner_kind() forever, hiding the bead from
    STRANDED and the recovery reconciler. Root-cause why the follow-up never settled,
    then bound ownership using actual delivery/recovery facts, distinguishing a dead
    abandoned follow-up from a queued successor, a live monitor, and settlement grace.
    Needs a sase-core change and pin move.'
- id: continuation-capture
  title: Stop continuation capture from nesting fork ancestry past the 256 KiB cap
    (sase-1o1)
  depends_on:
  - monitor-ownership
  size: large
  description: 'continuation-capture: persist_agent_delta records the #fork-expanded
    step prompt as authored_local_request and replay renders it as the parent user
    block, so every session member re-nests its whole ancestry; a live athena land
    session is at 708,391 bytes against the 262,144-byte validator cap, and an over-cap
    turn fails capture and settles the next monitor follow-up not-launchable. Capture
    only the local pre-fork request, make replay walk the parent chain, keep fork-replay
    output byte-stable, and add a repair or explicit error for in-flight over-cap
    sessions. Needs sase-core and sase changes.'
- id: fingerprint-refusal
  title: Stop host completion from refusing checks that passed (sase-1qm)
  depends_on:
  - continuation-capture
  size: large
  description: 'fingerprint-refusal: prepared completions backed by a fully green
    just check are refused with stale_worktree_fingerprint even when every observed
    fingerprint matches on hand audit (5 refusals in October, 11 so far in November;
    fresh sase recurrences 11-07 and 11-10). Diff the sealed observation against the
    host-time observation for the fresh repro runs (suspect: cross-repo scope over
    main plus sase-core), fix the confirmed mechanism, and make every refusal record
    name which fingerprint component differed. Needs sase-core and sase changes.'
- id: agent-identity
  title: Make sase agent restart find crash-killed agents (sase-1qi, ride-along sase-1qr)
  depends_on:
  - fingerprint-refusal
  size: medium
  description: 'agent-identity: a kill or reboot that never writes done.json leaves
    the TUI showing FAILED while sase agent restart returns not-found, because find_named_agent
    skips non-done records with dead PIDs (correct for wait resolution, wrong for
    restart). Give restart a lookup that includes dead-without-done records without
    deleting the liveness skip waiters rely on. Ride-along per decision qr_ride_along:
    project process_identity through the sase-core agent scan so the PID-reuse guard
    stops degrading to PID-only for scanned records.'
- id: private-remote-verify
  title: Verify an existing attachments-private remote is actually private before
    pushing (sase-1db)
  depends_on:
  - agent-identity
  size: medium
  description: 'private-remote-verify: preflight_sidecars compares visibility only
    when creating a missing attachments-private remote; an existing remote is trusted
    on its configured role alone and describe_label hard-codes "(private)", so attachment
    bytes can be pushed to a public repository under a private label. Probe visibility
    for existing remotes with resolve_remote_visibility at repo-init preflight and
    at the shared write boundary before the first upload per process, define honest
    unknown-visibility handling, and stop hard-coding the label.'
proposed_by: bbugyi200.athena.1bp
proposed_with: claude/claude-fable-5
decided_by: reviewer
decided_via: tui
create_time: 2026-11-12 05:26:12
status: done
bead_id: sase-1sd
---

- **PROMPT:** [prompts/202611/next_seven_salvage_and_unstrand.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/next_seven_salvage_and_unstrand.md)
- **BEAD:** [sase-1sd](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1sd/README.md)

# Next Seven: Salvage the Display-Name Fix, Then Unstrand the Fleet

## Context

This epic implements the seven ranked task beads from the consolidated research report
`research:202611/next_seven_task_beads_salvage_and_unstrand/next_seven_task_beads_salvage_and_unstrand.md`
(2026-11-11, five researcher reports plus a lead verification pass against sase master
`671ff5bde0`, release 0.19.0). The lead re-verified that all seven seeds — sase-zo,
sase-1re, sase-1s2, sase-1o1, sase-1qm, sase-1qi, sase-1db — plus the sase-1qr
ride-along are `ready` and unassigned, and that no active epic owns any of them; the
proposer re-confirmed all eight are still `ready` today.

**The owner asked for strictly sequential phases.** Every phase depends on the previous
one; never run two phases of this epic at once. The order front-loads the time-sensitive
salvage and the cheapest disjoint fix, then runs the three core-touching repairs
adjacently so their sase-core releases and pin moves batch naturally, and finishes with
the two medium sase-side fixes.

| Bead                  | Why it made the cut                                                                                                                               |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| sase-zo (bug, S, +10) | `sase repo list -p sase` still exits 2 on master; the complete verified fix is uncommitted in athena's sase_21 workspace and exposed to retention |
| sase-1s2 (bug, S)     | Runner imports overwrite the process SIGTERM handler; reproduced in fresh interpreters by two researchers                                         |
| sase-1re (bug, M, +1) | Ranked by all five researchers; done monitors with stale followup_pending blind the stranded-work classifier the whole recovery stack reads       |
| sase-1o1 (bug, L)     | Fork-ancestry nesting is over the 256 KiB capture cap in production (708,391 bytes live); long land/monitor sessions dead-end                     |
| sase-1qm (bug, L)     | Fingerprint refusals of green checks are rising (5 in Oct, 11 so far in Nov); each discards a 30-60 minute verification                           |
| sase-1qi (bug, M)     | Restart cannot find crash-killed agents; its deferral behind Agent Hijack (sase-1qz) has expired, and boot recovery needs the primitive           |
| sase-1qr (bug, M)     | Ride-along with sase-1qi per decision; Rust scans drop process_identity, degrading the PID-reuse guard                                            |
| sase-1db (bug, M)     | The one irreversible-consequence item: bytes believed private can be published to a public remote with no check                                   |

**Out of scope — owner actions, not phases.** The report's owner actions (resume epic
sase-1ra after trimming sase-1ra.11.1's stale gates, reclaim sase-j0, keep sase-1ot
owner-gated, triage the stalled-epic census) are decisions reserved for the owner. No
phase of this epic resumes sase-1ra, touches sase-j0's claim, or performs any sase-1ot
remediation. The lander may restate these reminders in its closing note, nothing more.

**Coordination with a resumed sase-1ra.** The owner may resume epic sase-1ra while this
epic runs. Its phases .8-.12 touch surfaces adjacent to this epic: the staged-monitor
settle guard, the stranded-work classifier, and the boot reconciler. Before starting
work, every phase re-runs `sase agent list -j` and, if a live sase-1ra member is editing
the same files (notably `names/_lookup_named.py`, the carried_repo obligation guard, or
`crates/sase_core/src/stranded_work/`), coordinates through a phase-bead note and waits
or narrows scope rather than racing.

> [!decision] sase29_salvage = sase21_only
>
> The salvage phase lands only the sase_21 display-name fix. It records the sase_29
> dirty state (8 files including
> `tests/test_agent_auto_restart_signature_skew_replay.py`, base `0cc9fad474`) in a
> phase-bead note so the owner and a resumed sase-1ra.7 can recover it.

> [!decision] sase29_salvage = both_workspaces
>
> The salvage phase also rebases, re-tests, and lands the sase_29 signature-skew diff,
> recording on the phase bead that sase-1ra.7's planned work was landed here so the
> resumed epic does not redo it.

> [!decision] reopen_guard = defer
>
> The salvage phase does not touch the carried_repo reopen guard. It records the
> structural gap (a carried_repo obligation whose staged successor never adopts leaves
> the predecessor bead closed) as a `PROPOSED FOLLOW-UP:` note naming sase-1ra.8 as the
> owning surface.

> [!decision] reopen_guard = include
>
> The salvage phase extends the completion guard so a carried_repo obligation whose
> staged successor never adopts reopens the predecessor's bead, mirroring the sase-1oh.4
> guard, with a regression test.

## Rules for every phase

- Read every assigned task bead in full, including every note and +1 entry, with
  `sase bead read <id> -r "<why>"`. Reproduce the failure on a tree that can exhibit it
  before changing code.
- Keep assertions, safety refusals, and existing budgets. Do not add skips, xfails,
  flake-baseline entries, timeout or elapsed-bound increases, or suppressions merely to
  make a gate pass.
- Shared backend behavior belongs in sase-core. Open it with `/sase_repo` before reading
  or editing it. Keep test-fixture and presentation fixes in the sase repo. If a Rust
  binding changes, let host completion move the `sase-core-revision.txt` pin; never
  invent a SHA. Core-touching phases coordinate with the steward train (sase-1n6.11) if
  it is live rather than racing it.
- Before starting, re-run `sase agent list -j`; if a resumed sase-1ra member is editing
  the same surface, coordinate per the Context section.
- Before finishing, read the `lint_and_test` reference memory. Run focused regressions,
  then `sase tool run check` in each changed repository. Do not run `just check-full`.
- Record each failure's base, host, before/after results, and limits on the phase bead.
  Do not create task beads. Record out-of-scope findings as `PROPOSED FOLLOW-UP:` notes
  on your own phase bead.
- Record `TASK RESOLVED: <task-id> — <evidence>` on the phase bead for each assigned
  task bead, resolve any `--epic-symbol` entries the phase keyed, and close only the
  phase bead. The lander closes the task beads.

## Phase salvage-display-name: salvage and land the verified -p fix (sase-zo)

Read sase-zo in full, including all ten +1 entries. The configured
`sase repo open <name>` half was fixed earlier; the remaining root (per the 2026-10-16
+1) is that `sase repo list/log/open/path -p` resolves the project token through
`src/sase/main/workspace_handler_context.py` `resolve_project_context` →
`src/sase/workflows/utils.py` `get_project_file_path`, which accepts the directory key
only, while `sase project show` (`project_aliases.resolve_project_alias_ref`) and
`sase artifact stats -p` (display-name snapshot `project_key_for_ref`) accept key,
display name, or alias. Decision `project-labels-never-keys` makes the display name the
identity every surface shows, so `-p sase` failing closed breaks the documented contract
for every agent and human. Verified still broken at `671ff5bde0`:
`sase repo list -p sase` exits 2 with "Project 'sase' has no WORKSPACE_DIR".

The complete fix already exists, uncommitted, in the sase_21 workspace clone on athena
(base `99932634cd`): changes to `workspace_handler_context.py` and `patch_sync.py`, plus
a new `tests/main/test_project_ref_canonicalization.py` with seven regression tests, all
lint green when written. Phase sase-1ra.1 closed done on Nov 4 but its monitor crashed
at launch and the dirty work was carried to a successor that never started, which is why
it never landed. This phase is a salvage, and it is time-sensitive: the sase-zw
disk-retention machinery exists precisely to reclaim held workspace state.

- Inspect the sase_21 diff read-only (`git status` / `git diff` against its base
  `99932634cd`). Copy the diff out; do not develop inside sase_21.
- Rebase the changes onto current master in your own workspace. Re-run the salvaged
  regression tests plus the repo list/log/open/path focused tests. Re-verify the
  acceptance matrix below. If the salvaged diff no longer applies cleanly or master has
  drifted, reimplement the same registry-canonicalized resolution; the salvaged tests
  remain the spec.
- Only after the rebased fix is committed in this epic's workspace, restore the salvaged
  files in sase_21 to a clean state so retention can proceed; if anything in sase_21
  does not match what you landed, leave it untouched and record why.
- Record the sase_29 state per decision `sase29_salvage`.
- Record the carried_repo reopen-guard gap per decision `reopen_guard`.

Acceptance: `sase repo list -p sase`, `-p SASE` (alias/case per the registry), and
`-p gh_sase-org__sase` all resolve to the same project for list/log/open/path; an
ambiguous token lists the candidates and exits 2; a stale `~/.sase/projects/sase/`
directory no longer resolves the display name to the wrong ProjectSpec; the regression
tests are in-tree and green; `sase tool run check` passes.

## Phase sigterm-import: runner imports must not replace the SIGTERM handler (sase-1s2)

Read sase-1s2 in full. At current master, importing `sase.axe.run_agent_runner` executes
`install_sigterm_handler("agent", soft=True)` at module scope (line 87), and
`sase.axe.mentor_runner` executes `install_sigterm_handler("mentor")` at module scope
(line 30). Any importer — pytest xdist workers, tool runs, unrelated processes —
silently loses `SIG_DFL` and gains a soft handler that records a kill without exiting,
so processes can survive SIGTERM until a hard kill and signal behavior becomes
import-order-dependent. The contention epic observed this live in monitors
`kyhnsmgpcjx9` and `aa3fzgn042n3` and filed this bead as a separate follow-up; the
`sase-1rq` members are WAITING and none edits this surface, but re-check per the rules.

- Move both installs out of import time into the runner-owned entry lifecycle; the
  pattern already exists in `run_agent_runner_signals.py`. Preserve existing production
  shutdown semantics: the handlers must be installed at least as early in the real agent
  and mentor entry paths as they effectively are today.
- Add a fresh-interpreter test that importing each module leaves
  `signal.getsignal(SIGTERM)` at the caller's handler, and real-entry tests that the
  agent and mentor entry points still install the soft handler and settle correctly
  under real signals.

Acceptance: a fresh-interpreter import of either module leaves SIGTERM untouched; the
real entry points still install the handler and existing shutdown/kill-recording tests
pass; `sase tool run check` passes.

## Phase monitor-ownership: bound done-monitor ownership (sase-1re)

Read sase-1re in full, including the 11-05 +1 naming four live holders (sase-1ih.2,
sase-kp, sase-ru.6, sase-s5). In sase-core
`crates/sase_core/src/stranded_work/classify.rs`, `owner_kind()`'s monitor/gate branch
treats a monitor member that is `done` with `followup_pending=true` (stale
`next_action`, no `followup_outcome`) as `owned/monitor` forever, so the bead can never
become stranded. ACE's STRANDED view, the recovery reconciler, and the zombie-epic
cleanup all read this classifier, so every bead hiding behind a done-but-pending monitor
is invisible to the whole recovery stack.

- First, root-cause why these follow-ups never settled (why `next_action` went stale
  with no `followup_outcome`) on at least one of the four live examples. Fix or file the
  settlement cause per the follow-up rules; do not paper over it in the classifier
  alone.
- Then bound ownership using actual delivery/recovery facts: a dead, abandoned follow-up
  must reduce to `stranded`, while a real queued successor, a live monitor, and
  legitimate settlement grace stay owned. Reproduce the decisive classification first: a
  `StrandedWorkMemberFactsWire` with `turn_kind: "monitor"`, `lifecycle: "done"`,
  `followup_pending: true`, no `followup_outcome`, and cold timestamps currently reduces
  to `owned/monitor`.
- This is distinct from sase-1ra.8 (orphaned _staged_ monitors) — coordinate on the
  shared classifier surface, do not merge the two repairs.
- Update binding round-trip tests as needed; the sase pin moves through host completion.

Acceptance: the synthetic reduce flips to `stranded` when the follow-up is dead and
abandoned; the four live examples classify as stranded (or their beads are verifiably
settled); a queued successor, a live monitor, and in-grace settlement still classify as
owned; core and sase checks pass.

## Phase continuation-capture: stop nesting fork ancestry (sase-1o1)

Read sase-1o1 in full. `persist_agent_delta`
(`src/sase/continuation_capture/agent_delta.py`) records the `#fork`-expanded step
prompt as `authored_local_request` (`authored_local_request=step_prompt` in
`src/sase/macro/workflow_executor_steps_prompt_prepare.py`), and replay
(`src/sase/history/chat_fork/continuation/_render.py::_render_agent_delta`) renders that
text as the parent's user block, so each member re-nests its whole ancestry and the
field grows superlinearly with session length. The Rust validator
(`crates/sase_core/src/continuation/schema.rs`, `validate_agent_delta`,
`MAX_TEXT_BYTES = 256 * 1024`) rejects over-cap captures; the member then gets no
continuation node and its next monitor result settles `not-launchable`. This is live: a
land session measured 708,391 bytes on 2026-11-11 (trend 126 KB Oct 24 → 254 KB Nov 6 →
708 KB Nov 11).

This phase is sized large: plan before implementing. The report's fix shape, to be
validated during that planning:

- Capture only the local pre-fork request; make replay walk the parent chain to
  reconstruct ancestry (text-ref storage, like segments, as a fallback for bounding the
  field).
- Keep fork-replay output byte-stable for existing sessions; add a long-session
  regression (for example 20 members) proving the authored field stays bounded under the
  cap while replay output is unchanged.
- Add a one-shot repair for in-flight over-cap sessions, or at minimum an explicit,
  diagnosable capture error plus a documented `sase monitor resume` path instead of the
  silent dead-end.
- Check `<artifacts>/continuation/capture_errors.jsonl` for
  `authored_local_request is too large` before and after on the live athena session.

Acceptance: the long-session regression holds the authored field under the cap with
byte-stable replay; an in-flight over-cap session either repairs or fails with an
explicit actionable error; core and sase checks pass.

## Phase fingerprint-refusal: host completion must not refuse green checks (sase-1qm)

Read sase-1qm in full. A prepared completion backed by a fully green `just check` (exit
0, evidence complete, every dirty-file hash and HEAD matching the sealed fingerprints
byte-for-byte on hand audit) is refused with `stale_worktree_fingerprint`, costing a
recovery turn and discarding a 30-60 minute verification. The root cause is unconfirmed;
candidates are a fingerprint computed over a different repository set or scope than the
sealed observation (the cross-repo prepared intent covering main plus sase-core is the
lead suspect) or a time-of-check difference between seal and host evaluation.
Recurrences are rising: the 11-01 incident, fresh sase recurrences on 11-07 and 11-10
(both diverted to recovery), and seven bob-cli refusals this month (six in one day,
fingerprint-plus-missing-stages — confirm whether that is the same mechanism in this
investigation).

This phase is sized large: plan before implementing, and investigate before fixing.

- Start by diffing the sealed observation against the host-time observation for the
  fresh repro runs. Locations: sase-core
  `crates/sase_core/src/continuation/completion_eval.rs` (`stale_worktree_fingerprint`)
  and sase `src/sase/monitor/followup_prompt.py`.
- Fix the confirmed mechanism. Do not weaken the refusal: a genuinely stale worktree
  must still be refused; the bug is refusing matching fingerprints, and any fix must
  preserve fail-closed behavior for real mismatches.
- Whatever the root cause, make the refusal record name _which_ fingerprint component
  differed (repository, path set, hash, HEAD, index tree, mode, protection flag) so a
  future refusal is diagnosable instead of a guess. Land this diagnostic even if the
  root cause resists confirmation this phase; in that case record exactly what was ruled
  out on the phase bead so the lander can judge whether sase-1qm can close or must stay
  open.

Acceptance: either the confirmed mechanism is fixed with a regression reproducing the
refusal shape, or (at minimum) the refusal diagnostics name the differing component and
the investigation's ruled-out candidates are recorded; no green-refusal behavior is
replaced by blanket acceptance; core and sase checks pass.

## Phase agent-identity: restart must find crash-killed agents (sase-1qi, sase-1qr)

Read sase-1qi in full. A kill or reboot that never writes `done.json` leaves the TUI
showing FAILED while `sase agent restart <name>` returns not-found: `find_named_agent`
(`src/sase/agent/names/_lookup_named.py`, the non-done dead-PID `continue` around lines
148-150) deliberately skips those records — correct for wait resolution, wrong for
restart. The 11-01 reboot forced manual per-epic bead surgery for 27 stranded waiters.
The earlier deferral behind Agent Hijack has expired (sase-1qz is closed).

- Give restart a lookup mode that includes dead-without-done records (the
  FAILED/exit-without-error class) without deleting the liveness skip that wait
  resolution and parent-phase artifact filtering rely on. Keep the existing priority
  rules for live and done records.
- Regression: kill a waiter-style agent process without `done.json`, observe the record
  is skipped for wait resolution but found by restart.

> [!decision] qr_ride_along = include
>
> Also read sase-1qr in full and fix it in this phase. The Python scan-wire mirrors
> declare `process_identity` on agent_meta and the RUNNING marker
> (`src/sase/core/agent_scan_wire_markers.py`) and the liveness probes read it
> (`record_liveness_probe` in `src/sase/core/runner_slots/_scan.py`, the listing probe
> in `src/sase/agent/_running_listing_common.py`), but the sase-core agent scan
> (`crates/sase_core/src/agent_scan/{scanner.rs,wire.rs}`) never projects it, so every
> non-monitor Rust-scanned record degrades the PID-reuse guard to PID-only. Project
> `process_identity` from `agent_meta.json` and the RUNNING marker in the core scanner
> and wire, update binding round-trip tests, decide whether existing artifact-index
> payloads need a refresh or a schema bump, and add a sase-side test that a scanned
> record carries the identity through to the liveness probe. Only the monitor-specific
> `supervisor_identity` projection exists today; leave it intact. The pin moves through
> host completion.

> [!decision] qr_ride_along = defer
>
> Fix only sase-1qi. Leave sase-1qr `ready` and record on the phase bead that it was
> deliberately deferred by this epic's decision, so later rankings keep its evidence.

Acceptance: the sase-1qi repro (dead PID, no `done.json`, TUI FAILED) is findable by
`sase agent restart` and still skipped by wait resolution, with a regression for each
side; under `include`, a Rust-scanned record with `process_identity` (for example
`boot:99`) carries it through to the liveness probe with binding tests green; checks
pass in every changed repository.

## Phase private-remote-verify: verify private attachment remotes (sase-1db)

Read sase-1db in full, including the sase-1d5.land integration note. `sase repo init`
preflight (`preflight_sidecars` in `src/sase/sdd/_sidecar_init.py`) compares visibility
only when the provider reports `not_found` (creation); an existing attachments-private
remote is trusted on its configured role alone, and `describe_label()` in
`src/sase/bead/attachments/upload/discovery.py` unconditionally labels the store
"(private)". A misconfigured or legacy remote pointing at a public repository publishes
sensitive attachment bytes under a private label — the one failure in this epic that
cannot be repaired after the fact. The prerequisite machinery already landed with
sase-1d5: `resolve_remote_visibility(url)` in
`src/sase/bead/attachments/remote_visibility.py` (public|private|unknown, cached,
injectable prober) and the doctor check `_private_store_anonymously_readable` in
`src/sase/doctor/checks_attachment_store.py`.

- Enforce rejection of a known-public "private" destination at the shared write boundary
  (the `GitAttachmentStore` upload path, once per process before the first upload), not
  just in preflight and labels, so outbox retries and alternate frontends cannot bypass
  it. The public-store injection guard (probing the beads remote before creating the
  public role) is the in-tree template.
- Probe existing attachments-private remotes in `sase repo init` preflight and refuse or
  warn per the policy you define; define honest `unknown`-visibility handling (never
  silently treat unknown as private; pick warn-and-proceed or refuse and state the
  choice in the phase record).
- Stop hard-coding the "(private)" label; have `describe_label()` reflect the probed
  visibility, including an honest unknown.
- Cover: an existing-public remote, a remote whose visibility changed after creation,
  cached probe results, and queued or retried uploads.
- This is prevention only. sase-1ot (historical purge) is owner-rejected; do not perform
  or propose any history rewrite here.

Acceptance: pointing `repos.sidecar.builtin.attachments-private` at a public repository
causes repo-init preflight and the first upload of a process to refuse (or warn, per the
defined unknown policy — public is always a refusal); the label reflects probed
visibility; the doctor check stays consistent; `sase tool run check` passes.

## Landing

The lander re-reads the epic and every task, phase, and nested bead and note. It checks
each `TASK RESOLVED` claim against the source and commits, and integrates any drift that
landed while the epic ran (in particular, work landed by a resumed sase-1ra). It closes
each of the epic's task beads — sase-zo, sase-1s2, sase-1re, sase-1o1, sase-1qm,
sase-1qi, sase-1db, and sase-1qr when decision `qr_ride_along` is `include` — normally,
each with its own commit or test evidence. sase-1qm may close only if its confirmed
mechanism was fixed; if only the diagnostics landed, the lander records the evidence on
the bead and leaves it open. It triages every `PROPOSED FOLLOW-UP:` note through
`/sase_new_task`. Then it:

1. Resolves every `sase bead epic-symbols` entry.
2. Closes the epic normally, with the follow-up triage in its note.
3. Runs `just symvision`.
4. Marks this plan file done.

Do not use `--force` to advance a successful landing. The lander does not perform the
report's owner actions (resuming sase-1ra, reclaiming sase-j0, anything on sase-1ot, the
stalled-epic census); it may restate them as reminders in its closing note.
