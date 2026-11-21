---
tier: tale
size: medium
title: Finish gate failure recovery and land sase-1ut
goal:
  Failed gate turns have one race-safe shared failure projection, honor the
  once-after-dismissal notification policy, expose recovery on every promised surface,
  and epic sase-1ut closes with verified evidence.
proposed_by: bbugyi200.athena.sase-1ut.land
proposed_with: codex/gpt-6-astra
bead: sase-1ut
status: done
---

- **PARENT:**
  [202611/gate_turn_carry_and_failed_execution.md](https://github.com/sase-org/sase--plans/blob/main/202611/gate_turn_carry_and_failed_execution.md)
- **BEAD:**
  [sase-1ut](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ut/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-1ut.land](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.sase-1ut.land.md)
- **COMMITS:**
  - [da2165b](https://github.com/sase-org/sase/commit/da2165b80fc181ee5f6c8e136436f12fe01ebe6a)
    — feat(gate-turn): share failure projection, harden dismissal timing and
    once-per-acceptance recovery

# Finish gate failure recovery and land sase-1ut

This is the remaining work from the landing audit of epic `sase-1ut`, not a replacement
for its completed implementation. One coding agent owns this bounded repair and the
epic's final closeout. There is no subsequent land agent for this tale.

Read `sase bead read sase-1ut -r "Need the current landing evidence"`, both children
(`sase-1ut.1` and `sase-1ut.2`) and all their notes, and the approved plan through
`sase artifact read plan:202611/gate_turn_carry_and_failed_execution.md "Need the approved contracts"`.
The original decisions remain final: `failed_bucket = waiting` (the pending/Stopped
bucket, with the workspace claim retained) and `resurface = once`. Do not introduce new
reviewer choices or change those decisions.

## Already verified

- Phase `.1`, commit `e6251b55d2`, correctly adds dirty-inherit carry hints to final
  context and the staged descriptor, preserves the explicit-commit template, and updates
  source skills and documentation. Keep those semantics.
- Phase `.2`, commit `07411341e3`, adds persisted member failure facts, failure
  labels/styles, awaiting-human bucketing, CLI show details, ACE details and reuse of
  its recovery dialog. Keep that useful implementation while correcting the gaps below.
- Forty-eight focused tests passed: `tests/gate_turn/test_failed_gate_attention.py`,
  `test_staged_contract.py`, `test_entry.py`, and
  `tests/test_finalizer_declaration_channel_staged_handoff.py`.
- The original epic has no parent bead. Both original phase beads are closed, and
  `sase bead epic-symbols sase-1ut` returned no entries at audit time.
- Fetched `origin/master` at `07411341e3`: no non-epic commits landed between the first
  epic commit and that tip, nor after the epic's creation. Its immediate predecessor,
  `39b594a30c`, predates the epic. Repeat this small drift check before closeout.

## Confirmed gaps

1. `_maybe_resurface_failure_notification` in `src/sase/gate_turn/_reclaim_pending.py`
   returns on the first undismissed `GateExecutionFailed` row before matching the gate
   or acceptance. A probe with only the target dismissed row restores it; prepending an
   unrelated visible failure changes the result to no restoration.
2. The hour is measured from `failure.at_unix`, not dismissal. A two-hour-old failure
   dismissed now can reappear immediately. Notification rows currently have no dismissal
   timestamp. Also, `build_gate_execution_failure` always sets `resurfaced: false`;
   writing a new failed attempt for the same acceptance resets a previously true marker.
   Identical retries replay the same acceptance in `notification_gates/executor.py`.
   Both behaviors violate the chosen policy.
3. Failure writes, backfill and resurfacing use unlocked read/modify/write decisions.
   Reclaim classifies outside the acceptance lock and does not revalidate before these
   effects. A stale reclaim must not overwrite a resumed owner, restore a resolved
   notification, or replace a newer acceptance. The current `mark_undismissed` also
   returns true for an already-visible row; that is not proof this invocation restored
   it.
4. The new shared failure policy and persisted contract live in Python.
   `AgentSessionTurnGateWire` in `sase-core` has no failure field. Python
   `gate_turn/store.py::_with_execution_failure` and the compact/wire ACE enrichment
   paths reopen `agent_meta.json` per gate to compensate. This bypasses the required
   Rust backend boundary and leaves shared snapshot consumers without the facts.
5. Human CLI list output (`gate_turn_table` and `gate_turn_list_markdown`) has a glyph
   but no failure summary or recovery commands, despite the original list/show
   acceptance requirement. CLI show already renders those fields.
6. The nine new failure tests mostly exercise isolated helpers. They do not execute a
   real failed option followed by resume success or cancel, nor verify ACE Enter
   recovery without a notification or snapshot-only projection.
7. `sase skill init --diff` still shows the old carry instructions in generated
   `sase_final` and `sase_gate` skills. The installed Codex final skill still says to
   stop when submission is optional. Source edits alone have not delivered this fix.

## Implementation

### 1. Deliver the already-landed skill correction

Before making source edits, read `generated_skills.md` through `/sase_memory_read` and
use `/sase_repo` to open `chezmoi`. Follow the supported generated-skill deployment
workflow from the clean canonical source containing `e6251b55d2`; inspect the preview,
regenerate rather than hand-edit provider files, and apply through chezmoi. Verify the
generated `sase_gate` and `sase_final` wording includes the carry-versus-explicit-commit
rule. These template changes already landed, so this step must not depend on this tale's
future code commit. Do not use dirty-source overrides or redeploy stale source.

### 2. Make the failure contract shared and its lifecycle reliable

Open `sase-core` with `/sase_repo` and read its `AGENTS.md`. Add a typed, optional
failure record to the existing gate member wire and scanner/index/compact projection,
including current acceptance/attempt/outcome identity and recovery data. Preserve
old-record defaults and the existing on-disk failure key when reading historical
members. Put shared eligibility and state-transition decisions in `sase_core`, with
bindings and round-trip tests; Python should gather facts and perform thin adapter
calls. Pure Rich/Textual formatting stays in Python.

Update `src/sase/core/agent_scan_wire_agent_session_turn.py`, the record adapters,
`gate_turn/models.py`, `gate_turn/store.py`, and the ACE enrichment paths to consume the
loaded wire instead of per-row filesystem fallbacks. Confirm a failed member can be
rendered from a supplied snapshot with its artifact directory unavailable. Keep
`followup_needs_attention` semantics separate from execution failure and ignore stale
failure data on terminal members.

Use existing acceptance/member lock conventions, including a consistent lock order, to
recheck current acceptance, response/terminal state, and execution owner immediately
before write/backfill/clear/resurface effects. Reuse atomic meta/index update plumbing;
preserve unrelated fields. Do not clear a new owner's pid or a newer failure from a
stale observation. A failure marker may remain for a retry in progress only if its
projection accurately distinguishes that running attempt; after successful recovery,
supersede, cancel, or terminal settlement it must no longer solicit recovery.

For resurfacing:

- Match request kind, gate ID and acceptance before considering another notification's
  dismissed/visible state. Unrelated rows cannot suppress the target.
- Persist dismissal time through the existing Rust notification mutation contract,
  including single and bulk dismiss paths, and carry it through Python wire/model
  adapters. For legacy dismissed rows with no timestamp, record first observation and
  wait a full hour from that observation; never infer dismissal from failure time.
- Preserve once-per-acceptance history across new attempts of the same acceptance. A
  genuinely new acceptance starts a new allowance. Keep this history separate from
  transient fields if recovery temporarily clears the failure object.
- Serialize competing reclaimers and recheck current failure immediately before
  restoring. Use the existing notification store update path, preserving the current
  payload-sidecar support and activity timestamp semantics. No global notification
  rewrite or new inbox mechanism is needed.

Both repos belong in the final declaration. The configured linked-core `revision_pin`
flow commits/pushes core first and writes its SHA to `sase-core-revision.txt` before
committing sase (see `docs/contribute/rust-backend.md`, CI source revision pin). Use
that host flow; do not hand-commit or postpone epic close until this turn's SHA or CI
exists. If consuming a core change already landed by another agent, ratchet the pin to
that containing revision through the existing command instead.

### 3. Finish the promised surfaces and prove recovery

Render an actionable failure summary and available resume/restart/cancel commands in
both human CLI list formats, using the same projected facts as show. Avoid duplicate
failure glyphs. Keep awaiting-human membership, claim retention and existing terminal
cleanup behavior unchanged. Reuse the current ACE recovery modal; test Enter from a
failed member when the original and failure notifications are absent or dismissed.

Add meaningful integration coverage using the existing gate executor fixtures:

- Real option failure persists the current facts, clears only its settler ownership,
  retains the workspace claim and produces matching CLI/ACE/shared-wire attention.
- Resume success, supersede, cancel and terminal settlement remove current failure
  attention and commands. Owner-lost and legacy backfill reach the same representation.
- An unrelated visible notification cannot block recovery; recent dismissal of an old
  failure waits an hour; a repeated failure on the same acceptance never earns a second
  resurface; a new acceptance does; competing reclaim/retry/terminal transitions do not
  resurrect stale failures or overwrite live ownership. Cover legacy timestamp absence.
- Assert CLI list/show fields, snapshot-only loading, and ACE Enter behavior, rather
  than merely calling the storage helpers directly.

Read `ace.md`, `tui.md`/`tui_perf.md`, `symvision.md`, `lint_and_test.md`, and
`docs_writing.md` through `/sase_memory_read` before relevant edits. Update the gate
reference with the precise dismissal and legacy behavior. Run focused Rust and Python
tests, `sase tool run check` in both changed repos, and targeted TUI screenshot
verification for changed rendered output after reading `tui_screenshot.md`. Use
`/sase_monitor` when a command cannot finish inline. Do not run `just check-full`.

The retention failure is already triaged: phase `sase-1ut.2` note #1 exactly duplicates
ready task `sase-1uk` (introduced by independent commit `92dc6274e5`). The lander
independently reproduced it, added a +1, and recorded that outcome on `sase-1ut`. Do not
create another task or make that pre-existing bug part of this repair. No other child
follow-up proposals remain. Classify any new failures on evidence; a check-full failure
after a passing check is test infrastructure, not remaining epic work.

### 4. Finish this epic's landing in the same coding turn

Re-read `sase-1ut`, both original children, all notes and the linked plan; check that
all descendants and their linked plans are ready. Review non-epic changes since
`e6251b55d2` and integrate any newly relevant callers. Complete this tale's own
readiness/status bookkeeping too. Do not wait for this work's commit SHA, push or CI
result.

Run `sase bead epic-symbols sase-1ut`. Resolve every listed entry by wiring,
privatizing, adding a valid non-test pragma, or deleting it under the Symvision policy.
Re-key only when a still-open later bead really needs the exemption. Then close
normally:

```bash
sase bead close sase-1ut --note "Verified both original phases and all notes; repaired shared failure projection, dismissal timing and once-per-acceptance recovery; verified CLI and ACE recovery, generated carry skills, current drift integration and recorded checks. Follow-up from sase-1ut.2 note #1 corroborated existing sase-1uk; no proposals remain untriaged."
just symvision
```

Expand that note with actual checks/results and relevant integration commits. If close
rejects leftover epic symbols, finish cleanup and retry. If it rejects unfinished phases
or plans, finish or reopen them; never force a successful landing. `--force` is only for
a deliberately canceled/superseded outcome with an explicit reason.

Use `/sase_repo` to open `plans` and set `status: done` in
`202611/gate_turn_carry_and_failed_execution.md`, the PLAN path for `sase-1ut`, only
after the successful close and Symvision check. Read artifact content through
`sase artifact read`, never direct file reads. Recheck the parent's link with
`sase bead read sase-1ut -r "Need the parent link"`; this epic had no parent at audit
time, so finish normally if that remains true. If a parent has since appeared, apply the
original lander's phase-versus-plan ancestor verification rules, stopping with an
explicit note on the first incomplete or ambiguous ancestor. Finish through
`/sase_final`, declaring every changed repository, including linked core and plans.
