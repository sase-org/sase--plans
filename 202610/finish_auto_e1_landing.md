---
tier: tale
title: Finish autonomy E1 inheritance and evaluation contracts, then land sase-1ip
goal:
  Repair the remaining verified autonomy E1 gaps, prove the integrated behavior, and
  close epic sase-1ip with its plan marked done in this coding turn.
size: medium
proposed_by: bbugyi200.athena.sase-1ip.land
bead: sase-1ip
create_time: 2026-10-09 15:57:58
status: wip
---

- **BEAD:**
  [sase-1ip](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ip/README.md)

# Remaining work for sase-1ip

This is the final implementation and landing of epic `sase-1ip`, not a new feature. One
coder can repair the bounded seams below using the existing core APIs. The coder must
complete the closeout in this turn; there is no later land agent for this tale. Do not
make any step depend on this turn's own commit SHA, push, or CI result.

Read `sase bead read sase-1ip -r "Need landing audit and final decisions"` and
`sase artifact read plan:202610/auto_e1_autonomy_record.md "Need E1 acceptance"`. The
original epic's `decision_record=no` remains final; its skipped memory work is already
recorded as task `sase-1j8`.

## Audit already completed

The lander read the epic's own notes and every note of all seven closed phases, the full
original plan, Python source and the six Python implementation commits `563f046a85`,
`9c5000f2db`, `73f593a3a5`, `9fd8a081f4`, `6f6754f97d`, and `70c51adbdd`, plus core
commits `01b0ad73` and `51b66fdb`. The core pin
`844b6c1d5cf82cf7a909c3164f2b4a4faca2fcaf` contains both core phases. The Rust
record/evaluator, profile catalog, mutation/inheritance APIs, summary/sentences,
decision log and bindings exist. The Python adapter, flag, gates, awareness hook, CLI
and tests exist. No epic commit edited memory. Flag bead `sase-1j0` exists; `sase-11g`
remains ready and has the required auto-half completion note.
`sase bead epic-symbols sase-1ip` was empty. There was no parent bead on `sase-1ip`.

The lander ran
`.venv/bin/python -m pytest tests/autonomy_contract tests/fakey/test_autonomy_lifecycle_e2e.py tests/monitor/test_monitor_followup.py tests/test_run_agent_runner_refresh.py -q`:
**141 passed, zero xfails**, in 208.72 s. Those passing tests lack the boundary cases
reproduced below; retain their existing expectations while adding the regressions.

All unrelated follow-up triage is complete and recorded on `sase-1ip`:

- Each phase's note #1 proposed the same skipped decision strand. All seven are
  consolidated into `sase-1j8`, a ready, small memory task.
- `sase-1ip.4` note #2 corroborated existing flake `sase-13a` (zsh sbd completion).
- `sase-1ip.6` note #2 corroborated existing flake `sase-120` (git-identity nested test
  timeout), citing ToolRun `77e35a42601cb848644144e4585ee9e9`.
- `sase-1ip.7` note #2 needs no new task: `191bc6d2e3` repaired the obsolete monitor
  follow-up prefix assertion to expect structural inheritance.
- `sase-1ip.5` note #2 is explicitly required by the original inherit phase and stays
  epic work: gateless `sase plan approve` coders must inherit structurally.

The integration review covered all non-epic commits since `563f046a85` through
`fb1186a229` (then both HEAD and freshly fetched origin/master):

- `7e87589fb5` preserves live auto state in runner marker writes; retain that rule for
  the whole record.
- `63a8f7a62e` split directive persistence behind its facade; edit the new focused
  modules, especially `_directive_persistence_autonomy.py`, not an obsolete copy.
- `6ac3dc734e` moved live auto prompt reconciliation into refreshed bootstrap so the
  pre-exec path performs no new sase imports. Preserve its import firewall.
- `191bc6d2e3` already integrated the monitor expectation.
- The other commits concern plugin UI/docs, fleet request coalescing, dictionary and
  pager display, bead read/performance plumbing, test splits and goldens; inspection
  found no additional autonomy consumer to migrate there.

At implementation start inspect any newer commits for overlap. Stay on the host-managed
checkout. Open `sase-core` with `/sase_repo` before reading/changing it.

## 1. Complete structural inheritance at the real entry points

`main/plan_direct_approval_run.py::_launch_coder` calls `launch_agents_from_cwd` with
`origin="generated"`, but `main/plan_direct_approval_prompt.py` only emits a normal
`%id(..., session=...)`. Only `agent/detached_child.py` currently sets
`AgentSessionAttachLaunchPlan.host_composed=True`. Thread a trusted host-composed attach
through direct approval and coder recovery when a planner/session exists. Use existing
session placement and detached-launch plumbing; preserve receipts, capacity behavior and
the standalone case. A human-authored session attach still resolves from its own prompt.
Do not restore inheritance by re-emitting `%auto`.

`axe/run_agent_directive_metadata.py::build_agent_meta` derives explicit selection only
when `directives.auto_enabled` is true. Thus an explicit `%auto:manual` or `%auto:off`
is indistinguishable from absence. The lander patched only the prompt input of the
existing `test_host_composed_attach_inherits` to those spellings: it still passed its
`profile == tale` assertion for both. Preserve explicit presence through the existing
core classifier/directive representation and pass the manual selection to
`autonomy_inherit`.

The in-process `continue_as_successor` path passes no authored prompt selection to
`create_followup_artifacts`; `_inherit_autonomy_record` always passes
`explicit_selection=None`. Apply explicit narrowing for pipe/handoff and other
agent-authored successors at their actual entry point too. Resolve shared policy
semantics in core; Python only extracts and transports them. Widening retains the live
inherited policy and emits the promised diagnostic. An unreadable predecessor must not
cause a host-composed child to adopt a wider authored policy.

Add production-entry regression coverage (not only direct core-binding calls) for direct
approval/recovery and in-process/out-of-process successor paths: inheritance, live
A-off, explicit manual/off and tale narrowing, refused widening, and the human attach
control. Test both flag states. Upgrade the existing function-driven lifecycle test to
call actual context entry points where it currently substitutes repeated calls to
`adapt_followup_artifacts` with different suffixes.

## 2. Keep the record authoritative across refresh and projections

The refreshed runner rewrites the prompt correctly but then rebuilds a new record:
`preserved_agent_metadata` omits autonomy. A lander probe through the real
preserve/build functions turned `{profile: manual, revision: 2, last: tale}` into
`{profile: manual, revision: 1, last: null}`. Preserve the live full record for the same
agent's re-exec, including last/revision/digest/source, and prove A-off then refresh
then A-on restores tale. Do not blindly preserve a record into a genuinely new human
launch. Retain the new refresh bootstrap/import firewall.

`autonomy/record.py::with_legacy_projection` lets stored legacy keys override the
record. A manual record combined with stale `approve=True` and action `tale` still
projects those automatic fields into loader/wire consumers. Make an existing valid
record authoritative, including false/absent projection values; translate legacy keys
only when no record exists. Preserve the separate meaning of the dual-use plan-flow
marker. Make `apply_record_meta_patch` remove stale legacy auto keys in flag-off writes
centrally instead of relying on one toggle caller's extra cleanup. Test mixed-state
listing, wire and TUI loader projections and both flag states.

Core `autonomy/mutate.rs::apply_target` currently retains `base.source` even on a human
TUI/CLI mutation. The probe of `mutate_record(..., 'manual', surface='tui')` returned
source `prompt`. Make provenance truthful for human mutation while inheritance stays
`inherited`; cover this in core and Python round-trip tests.

## 3. Use one effective gate evaluation and its declared capabilities

`notification_gates/validation.py` calls evaluate, then service creation calls it again.
`autonomy/gates.py::evaluate_gate` supplies a separate hardcoded capability map rather
than `adapter.auto_capabilities`. A lander probe using a question adapter with an empty
capability set produced two evaluations and an automatic outcome. Pass the actual
adapter capability set and evaluate once for a new creation, then use that decision
consistently for input validation, parking/execution, policy blocks and the decision
log. Static spec validation may validate the record/grammar without independently
deciding the gate again.

Cover cached/idempotent creation and journal recovery: reuse the persisted decision
snapshot when one exists, avoid duplicate log rows, and keep request/result/response
policy identity consistent. A generated request ID must appear in the log rather than an
empty ID. Keep privileged/unknown kinds manual and missing-option selections
all-or-nothing. Preserve Plan Decisions normalization, receipts, and tier parking. Add
tests for one evaluator call, empty/reduced adapter capabilities, idempotent and
recovered creation, explicit/generated IDs, and manual/auto/ask policy blocks.

## 4. Make log filtering real

`autonomy/cli_log.py` passes `--since` verbatim to core, whose log filter compares
timestamp strings. A temporary-store probe with entries in 2000 and 2026 returned both
for `since='1h'`. Reuse `sase.vcs_log.dates.parse_time_bound` and its operation
clock/timezone semantics to normalize CLI bounds before calling the log binding; do not
introduce a second date grammar. Normalize to the log's UTC timestamp format, reject
invalid DATE input, and resolve agent shorthand as the other autonomy agent views do.
Cover before/after the bound, today/day/relative forms, invalid input, and aliases.
Correct misleading docstrings claiming core accepts relative strings.

## 5. Verify the completed scope and isolation

Epic note #1 records a historical fixture escape that created real epic `sase-1ir`. The
current contract harness stubs `_archive_plan_for_approval` and
`plan_approval_actions.prepare_epic_launch`, which are the actual production aliases.
Prove the complete suite cannot publish plans/prompts or create beads/agents: use
temporary SASE/SDD roots and fail-fast spies at the durable publication and launch
boundaries, without mocking policy decisions. Keep the historical note in the final
evidence; do not claim a current leak without one.

Run the contract suite with zero xfails, the CLI parity/surface tests, the strengthened
fakey lifecycle, direct-approval/recovery, real successor, toggle, live-meta and refresh
tests. Exercise unchanged Plan Decisions receipt/default tests. Read the TUI memory
before changing TUI code; keep existing glyph/layout output unchanged and verify
affected visual snapshots if rendered output changes.

Run `just fix`, then `sase tool run check` (the governed `just check`) in each changed
repo, including core if changed. Do not run `just check-full`. Use `/sase_monitor` when
a long verification needs handoff, with a follow-up explicitly carrying this tale's
remaining closeout. A prepared completion that only commits would skip the required
closeout. If both repos change, the final host declaration pins core; no step needs the
unpublished core SHA first.

Spot-check the original landing demo with the checkout CLI (`.venv/bin/python -m sase`
if the global install is older): static and live-agent explain, filtered log, agent AUTO
column, and audited saved root prompts with one live awareness block for automatic
agents and none for manual agents. The lander's static/live explain demos worked from
the checkout; the global CLI had not yet learned `autonomy`, so do not mistake
deployment lag for a missing source implementation or globally reinstall.

## 6. Finish the epic landing in this coding turn

Re-read `sase-1ip` and all seven children with `sase bead read`, review new notes and
post-audit drift, and confirm all descendants and the original linked plan's exit
criteria are ready. Include every follow-up outcome from the audit above in the close
note. Confirm `sase-1j0` exists and leave `sase-11g` open for its queue half.

Run `sase bead epic-symbols sase-1ip`. Resolve every listed entry (wire, privatize,
valid non-test pragma, or delete under Symvision policy), or re-key only to a still-open
later bead that actually needs it. Then run:

```sh
sase bead close sase-1ip --note "Reviewed every phase and note, honored decision_record=no, verified E1 core and Python contracts and post-start integration, repaired landing inheritance/record/evaluation/log gaps, and passed the recorded verification. Follow-ups: memory proposals consolidated in sase-1j8; zsh and git-identity flakes corroborated on sase-13a and sase-120; obsolete monitor assertion fixed by 191bc6d2e3; direct-approval inheritance completed here. Isolation and live demo evidence recorded in the landing note."
just symvision
```

Expand the note with actual test results and integration evidence before closing. If
symbols block close, clean them and retry. If named phases block close, finish or reopen
them. Never force a successful close; force with canceled/superseded is only for a
deliberately abandoned scope and reason, not this successful landing.

Open the plans repository through `/sase_repo`, then set `status: done` in the
frontmatter of `202610/auto_e1_autonomy_record.md` (canonical reference
`plan:202610/auto_e1_autonomy_record.md`). Include that changed repo in the final
declaration. Re-read `sase-1ip -r "Need the parent link"` after closing. The audit found
no parent, so no ancestor close is expected. If a parent was linked since, follow the
original land prompt's normal phase/plan ancestor readiness rules and record any
incomplete/ambiguous parent blocker instead of forcing it.

Declare all changed repositories through `/sase_final` after closeout, with the assigned
epic complete. Host commits happen after the turn and do not delay close.
