---
tier: tale
title: Recover approved tale gates whose coder handoff never completed
goal:
  Make approved coder handoffs durable, diagnosable, and safely resumable without
  repeating the approval or plan commit.
size: medium
proposed_by: bbugyi200.athena.0jg
create_time: 2026-09-11 11:00:46
status: wip
---

# Recover approved tale coder handoffs

## Observed incident

The user approved `plan:202609/usage_group_refinement.md` for agent `0j8.f0.f2`,
including both the coder launch and plan commit. Investigation on September 11, 2026
found no running, queued, or recently completed coder for that family;
`sase agent show 0j8.f0.f2--code` explicitly reported no matching agent.

Stable evidence:

- Gate `c117f874-83de-4840-8405-58a8dc1efd66` belongs to `0j8.f0.f2--gate`. Its response
  selected `approve` and `commit` at 10:05:46 Eastern. The translated result requests
  `run_coder: true` and records the host-owned archive protocol `host_v2` and canonical
  plan reference above.
- Notification `6ea5311d-6592-4f8d-881a-902dfa8b1092`, sender `plan`, was created at
  09:51:04 Eastern and is dismissed. This is an answered approval, not a pending request
  for the user to approve again.
- The shell artifacts are under
  `~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202609/11/20260911095103/`.
  `gate_decision.md` confirms the selected options. `done.json` records `gated`,
  `answered`, and `TALE APPROVED`. `agent_meta.json` records the same state, the
  `--code` suffix, and `plan_committed: true`, but has no follow-up agent, outcome,
  error, or saved coder prompt.
- `sase gate show` reports `followup_needs_attention: false` and a `Running` bucket
  despite the missing coder. The source planner artifacts are at the same date's
  `20260911093317` directory. A separate descendant, `0j8.f0.f2.f0`, has its own pending
  freshness-marker plan; it is not this missing coder.
- The approved plan still passes `sase plan validate` with zero warnings. No launch
  failure tied to this gate was found in the inspected logs.

The initiating exception or interruption is **not established**. Do not attribute it to
model routing, Git contention, plan validity, or a code reload without further evidence.
The failure mechanism that makes such an interruption permanent is visible in current
code and was reproduced with an isolated injected exception:

1. `src/sase/gate_shell/settlement.py:settle_gate_shell` publishes the terminal
   decision, chat, done marker, and `gate_state: answered` before preparing and
   launching the successor. Publishing these before launch is necessary for family
   ordering and fork resolution.
2. `launch_gate_followup_agent` rebuilds the plan prompt in `_base_prompt_kwargs`,
   through `plan_shell/followup.py` and `prepare_accepted_plan_successor`.
3. Exceptions from that preparation escape
   `shells/settlement.py:settle_shell_claim_and_followup`. The isolated reproduction
   left both follow-up fields unset and called neither persistence nor claim cleanup.
4. The next settlement call returns immediately for terminal metadata, and the reclaimer
   skips terminal records. The missing follow-up therefore stays missing.

## Intended behavior and scope

Implement this as one medium tale spanning the Rust core and its Python adapters. An
approval decision and its coder-launch disposition are separate durable facts. Keep the
successful approval and its archive receipt even if subsequent preparation or launch
fails. Every requested handoff must expose a confirmed launch, an active attempt, or an
actionable failure/interruption. Approval alone must not imply that the coder is
running.

Keep existing branch semantics: approve-only launches without a second plan commit;
approve plus commit launches from the committed reference; commit-only and rejection
launch no coder; feedback follows its existing replanner path; `%auto` retains its
in-process successor and must not acquire an additional detached coder. Preserve epic
host-owned launch behavior and generic gate/monitor behavior.

Do not change the contents of either usage-indicator plan. Do not migrate all gate
execution or plan archival into a new subsystem. Limit shared changes to the handoff
disposition, recovery decision, and necessary persistence/launch integration.

## Implementation

### 1. Define shared handoff disposition and recovery decisions in Rust

Open `gh:sase-org/sase-core` using `sase repo open` before working there. New domain
rules belong in `crates/sase_core`, with a typed `sase_core_rs` binding and a thin
Python adapter. The existing `agent_scan/wire.rs` and `agent_scan/scanner.rs` carry gate
follow-up metadata; extend that wire only where needed and update binding serialization
and compatibility tests together.

Add a small gate-follow-up decision API, following the existing typed planning patterns
in `agent_ownership` and `agent_cleanup`. It must distinguish a settled decision from a
completed follow-up and classify intentional no-follow-up, in-process suppression,
preparation/launch in progress, confirmed launch, explicit failure, and
legacy/interrupted missing disposition. Preserve existing outcome values where their
meanings fit, including `launched`, `launched-degraded`, `not-launchable`, and
`suppressed`.

Persist an attempt identity tied to the gate ID and request fingerprint before starting
fallible successor preparation. Record ownership/liveness and a durable successor
identity or launch receipt sufficient to reconcile an interruption. Serialize competing
settlement/resume attempts using the existing durable locking and family-name
reservation mechanisms. A held live attempt is never stolen.

Recovery must inspect the actual family attachment/launch evidence as well as the gate
fields. If a coder already started but the gate receipt was not written, adopt that
successor and repair the receipt. If evidence is ambiguous, report that state and do not
start a second provider. A completed coder still counts as an existing successor. A
stale attempt can be resumed only after ruling out a launched successor.

### 2. Make settlement exception-safe and preserve prepared metadata

Update `gate_shell/settlement.py`, `gate_shell/followup.py`, and the affected shared
shell glue to consume the core disposition decisions. Retain decision/chat/index
publication before successor launch; simply moving `done.json` after spawning would
break the established ordering contract.

Wrap the entire fallible preparation and launch boundary, including the kind hook, model
selection, prompt composition, workspace acquisition, and launch receipt recording. On a
caught failure, durably record its stage, exception type/message, and available
diagnostics in the gate's metadata and done projection. Save a complete prepared prompt
when one exists; when preparation failed, retain the canonical plan reference and
reconstruction context rather than inventing a generic coder prompt. Keep plan-kind
failures strict: never launch only the placeholder `Implement the approved plan.` after
reconstruction fails.

Release a failed attempt's workspace claim using ownership-checked cleanup. Do not
release a claim already transferred to a successor. Preserve both the primary error and
any cleanup error. Reload or atomically merge metadata after preparation so the outer
settlement writer cannot erase fields written by successor preparation. Publish one
durable failure notification through the existing gate/error reporting path, keyed to
the attempt, with gate/agent identity and a supported resume command.

Ensure the reconstructed coder still uses the approved canonical plan, correct family
and `--code` suffix, size-derived model or explicit override, reviewer additions, wait
directives, and VCS context. Audit source-versus-gate artifact selection in
`plan_shell/followup.py`: original planner prompt/workflow inputs come from the recorded
source artifacts; settlement receipts belong to the gate. Existing host archive receipts
must prevent duplicate plan publication during retries. Keep archival side effects
mocked at their external boundary in tests, not the entire successor-preparation
function.

### 3. Diagnose old incomplete gates and resume the approved handoff

Extend the existing `sase gate answer --resume` behavior for an already answered
shell-backed tale whose selected branch requests a coder. Use the recorded response and
the same shared recovery API. Require supplied option/input values to agree with the
persisted answer. Bypass option command execution and plan archival on this path; resume
only the unfinished handoff. Preserve the current meaning of `--resume` for partially
executed option commands and preserve `--restart` semantics.

For an existing launched/suppressed/intentional-no-follow-up disposition, report an
accurate idempotent result and launch nothing. For a failed or interrupted eligible
handoff, resume under the per-gate attempt guard. Route the detached CLI executor and
ordinary settlement through the same adapter so concurrent callers cannot race.

Teach bounded reconciliation to recognize terminal gates with a requested successor and
missing disposition, including this incident. Do not equate every old terminal gate with
a failed launch: derive eligibility from the verified recorded branch and account for
auto suppression and existing successors. Mark an abandoned/ambiguous handoff as needing
attention. Legacy diagnosis does not automatically relaunch every old gate; the explicit
resume path performs the recovery.

Keep reconciliation off the TUI event loop and message pump. Select candidates from the
existing artifact index with a bounded batch/cursor, not a full archive walk on each
refresh. Persist progress so older incomplete gates can eventually be diagnosed. Use
bounded lock waits and existing refresh/change-token mechanisms. The explicit
single-gate resume must work even if the background pass has not reached that gate.

Update `gate_shell/projection.py`, `gate_shell/state.py`, and their consumers to use the
shared disposition. `sase gate list/show` and ACE must show the same failure or
interruption and resume guidance. Retain the approval label as the decision history, but
do not infer a running coder from `TALE APPROVED` alone. Check actual ownership before
claiming that an incomplete terminal gate has released its workspace.

Update `docs/notifications.md`, `docs/cli.md`, and the existing gate option summary in
`docs/configuration.md` to document answered-gate resume, preserved approvals, failure
reporting, and the difference between command retry and handoff retry. Update existing
`--resume` help text; introduce no extra public command or flag.

## Verification

Use isolated fixtures with a real approved tale and host archive receipt modeled on the
incident. Mock actual Git/network publication and provider spawning, while keeping
response translation, input reconstruction, successor preparation, settlement, core
decisions, and metadata projection real. Extend the existing suites under
`tests/plan_shell`, `tests/gate_shell`, `tests/gate_conformance`, and the CLI/plan
approval tests instead of only checking mock launcher call counts.

Required cases:

- Approve plus commit reaches exactly one `--code` launch with the canonical plan
  reference and correct model/waits; approve-only also works. Commit-only, reject,
  feedback, epic, and `%auto` retain their intended behavior.
- A preparation exception, model-resolution failure, and spawn failure persist an
  actionable error in metadata, done marker, CLI projection, and notification, with
  ownership-correct cleanup and no placeholder launch.
- Interruption after decision publication, during preparation, and after spawn but
  before receipt persistence can be diagnosed/reconciled. A same-fingerprint retry
  neither reruns option commands nor republishes the committed plan.
- Two simultaneous resume/settlement callers produce at most one provider launch; an
  existing running or completed successor is adopted. A live attempt or ambiguous
  attachment is not duplicated. A failed receipt write after a real spawn is treated as
  an uncertain launch until actual launch evidence resolves it.
- The incident's legacy terminal metadata with no follow-up fields is classified as
  incomplete, while terminal no-follow-up and suppressed fixtures are not. An old valid
  gate remains readable after the new metadata fields are introduced.
- Preparation metadata survives final settlement writes, and the terminal decision,
  chat, and index are visible before the successor starts.
- CLI `--resume` still resumes partial option execution, while an answered-gate resume
  uses the stored answer and rejects conflicting input. Repeated success is a no-op with
  the recorded successor reported.

Run Rust core and PyO3/binding tests for the new API and scanner projection, then the
core repository's required `just check` or `scripts/check.sh`; do not verify only
`cargo test -p sase_core`. Build/install the matching binding for the SASE checkout. Run
the focused Python suites and `just check` in SASE, following the current
`lint_and_test.md` guidance. Use `sase_monitor` for long verification. Finish with
`git diff --check` in each changed repository.

After the tested fix is available to the running host, re-inspect gate
`c117f874-83de-4840-8405-58a8dc1efd66`. Its existing user approval authorizes the
original coder handoff. If it still lacks a successor, recover it using the supported
`sase gate answer --kind plan --id c117f874-83de-4840-8405-58a8dc1efd66 --option approve --option commit --resume`
path. Do not hand-edit gate files or invoke bundle commands. Confirm both the launch
receipt and the actual `0j8.f0.f2--code` agent state, including any legitimate runner
queue. Leave the separate `0j8.f0.f2.f0` pending plan alone. If a host rollout is still
needed, report that concrete dependency and the ready recovery command rather than
claiming that the original coder has started.
