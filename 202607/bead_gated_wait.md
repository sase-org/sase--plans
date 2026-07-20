---
tier: epic
title: Bead-gated %wait for delegated epic phases
goal: 'A dependent epic phase agent (and the land agent) does not start just because
  its blocker phase agents finished: %wait gains a bead=<bead_id> kwarg that also
  requires the named beads to be closed, sase bead work emits both conditions for
  every dependency edge, and a delegated phase bead (one whose agent handed its work
  to an auto-approved child epic) is closed automatically when that child epic lands,
  so bead waits eventually resolve.

  '
phases:
- id: core-close-cascade
  title: Core upward close cascade and delegated-phase scheduling
  depends_on: []
  size: medium
  description: '''Core: upward close cascade and delegated-phase scheduling'' section:
    in sase-core, auto-close a parent phase bead when its last open child (a delegated
    child epic) closes, exclude phases with an open plan-type child from relaunch
    waves, and additively expose per-phase blocker bead IDs and the epic''s full phase
    bead ID list on the work-plan payload.'
- id: wait-directive
  title: The bead= kwarg on %wait
  depends_on: []
  size: medium
  description: '''The bead= kwarg on %wait'' section: extend the %wait directive grammar
    with a repeatable bead=<bead_id> keyword collected alongside time=/runners=, expose
    AgentDirectives.wait_beads, keep bare-wait rewriting and templates ignoring bead-only
    occurrences, and round-trip the kwarg through directive editing.'
- id: wait-resolution
  title: Bead conditions in wait resolution
  depends_on:
  - wait-directive
  size: large
  description: '''Bead conditions in wait resolution'' section: thread wait_beads
    through the agent runner into waiting.json as wait_for_beads, and resolve bead-closed
    conditions in dependency_resolution_status, the wait_checks chop, and the runner
    fallback via a shared project-to-bead-store locator, failing closed when a bead
    or store is unavailable.'
- id: bead-work-emission
  title: Emit bead waits from sase bead work
  depends_on:
  - core-close-cascade
  - wait-directive
  size: medium
  description: '''Emit bead waits from sase bead work'' section: render %w(bead=...)
    lines for every in-epic blocker on phase segments and for every phase bead on
    the land segment, keep dry-run/approval previews in parity, and bump the sase_core_rs
    floor to the release carrying the new payload fields.'
- id: surfaces-docs
  title: Waiting surfaces and documentation
  depends_on:
  - wait-resolution
  - bead-work-emission
  size: small
  description: '''Waiting surfaces and documentation'' section: show bead waits in
    ACE waiting descriptions and wait modals, and sweep xprompt/axe/bead docs for
    the new kwarg, the upward close cascade, and delegated-phase retry behavior; flag
    (do not edit) the xprompts memory table.'
- id: smoke
  title: End-to-end delegation smoke exercises
  depends_on:
  - core-close-cascade
  - wait-resolution
  - bead-work-emission
  - surfaces-docs
  size: small
  model: haiku
  description: '''End-to-end delegation smoke exercises'' section: in a scratch project,
    drive a nested phase-delegates-to-child-epic flow and verify dependents and the
    land agent stay parked until the child epic lands, the parent phase auto-closes,
    retries skip delegated phases, and waiting displays name the beads.'
create_time: 2026-07-20 11:01:43
status: wip
bead_id: sase-87
---

# Plan: Bead-gated %wait for delegated epic phases

## Context

All references below were verified against the current checkout and the sase-core linked repo (open it with the
`/sase_repo` skill; per the Rust Core Backend Boundary memory, shared domain behavior belongs there).

**The gap.** Every epic phase segment rendered by `render_multi_prompt` (`src/sase/bead/work.py:292-379`) carries
`%auto` (line 352) and waits only on blocker phase _agents_ (`%w:<names>`, line 354); the land segment likewise waits
only on phase agents (line 375). Medium/large phases also get `#plan` (lines 356-357). When such a phase agent proposes
an epic-tier plan, `%auto` approves it and `prepare_epic_launch` (`src/sase/_plan_approval_epic.py:16`) spawns the child
epic as a separate detached clan named after the child epic bead (e.g. phase bead `foo-5.2` begets clan `foo-5.2.1`).
The proposing phase agent's house then completes, and wait resolution (`src/sase/core/wait_dependency_resolution/`) —
which consults only agent artifacts (clan/family/workflow/named/planner candidates, `_index_queries.py:335-379`) —
releases every dependent phase and eventually the land agent while the child epic is still running. The `%wait` grammar
accepts only positional agent names plus `time=`/`runners=` kwargs (`src/sase/xprompt/_directive_collect.py:103-116`),
so there is no way to express "and the underlying work is done".

**Why bead closure is the right condition.** Phase agent names are exactly the phase bead IDs (`<epic_id>.<N>`,
`docs/beads.md:349`), and the `work_phase_bead` xprompt (`src/sase/default_config.yml:644-651`) makes closing the bead
the agent's definition of done. Waiting on the agent alone releases too early under delegation; waiting on the bead
alone would release before the blocker agent's commit/finalizer completes. Requiring both gives "agent finished AND work
landed".

**The deadlock this creates, and its fix.** Today nothing closes a delegated phase bead when its child epic lands: the
child's land agent closes only the child epic (`bd/land_epic` xprompt, `src/sase/default_config.yml:588-612`), and the
parent phase bead is eventually swept up by the parent land agent's recursive `sase bead close <epic>` (`close_issues`,
`crates/sase_core/src/bead/mutation.rs:362-424`, cascades downward only). If the land agent also waits on `bead=<phase>`
closure, it would wait for a close that only it performs. The fix is an upward cascade in core: closing a plan-type bead
whose parent is a _phase_-type bead auto-closes that parent phase once all of the phase's children are closed.

**Resolution plumbing today.** The runner extracts directives into `AgentInfo`
(`src/sase/axe/run_agent_directives.py: 89-107,232`) and calls `wait_for_dependencies`
(`src/sase/axe/run_agent_wait.py:351`), which writes a `waiting.json` marker (`waiting_for`, optional
`wait_for_artifacts`/`wait_duration`/`wait_until`) and polls for `ready.json` with a direct-resolution fallback every
60s. The `wait_checks` chop (`src/sase/scripts/sase_chop_wait_checks.py`) scans all markers and writes `ready.json` when
`dependency_resolution_status` (`src/sase/core/wait_dependency_resolution/ _resolution.py`) reports resolved; it skips
markers where `not waiting_for and not wait_for_artifacts` (`sase_chop_wait_checks.py:107-108`). Bead-store access
outside a workspace already has a precedent: `_canonical_beads_dir_for_project`
(`src/sase/integrations/_mobile_helper_beads.py:213`) resolves a project name to its `sdd/beads` dir, and `BeadProject`
(`src/sase/bead/project.py`) reads through the Rust facade.

**Work-plan payload.** `_PhaseAssignment.waits_on` holds only same-run scheduled blockers' agent names
(`src/sase/bead/work.py:40-49,175-177`); closed blockers and (after this plan) delegation-excluded blockers are absent,
so the renderer needs the full in-epic blocker bead IDs from the Rust payload, not just `waits_on`.

## Design

### Wait semantics

`%wait(bead=<bead_id>)` adds a bead-closure condition to the agent's launch barrier:

- Repeatable; accumulates across `%wait` occurrences like positional names, and combines freely with positional agent
  names, `time=`, and `runners=` (e.g. `%w(foo-5.1, bead=foo-5.1)`). Order-preserving dedupe.
- The condition holds when the named bead exists in the waiting agent's own project bead store with status closed. All
  bead conditions AND all agent/artifact conditions must hold; the `time=` floor still starts only after all
  dependencies resolve.
- Fail closed: a missing bead, missing store, or store read error keeps the agent waiting (matching an unknown agent
  name). The existing ACE wait cancel/resume actions remain the escape hatch. Resolution is edge-triggered at poll time;
  a later reopen does not re-park a released agent.
- Bead-only `%wait` occurrences behave like `time=`-only ones for bare-wait rewriting and agent-name templates
  (`src/sase/agent/multi_prompt_reference_directives.py:111-165`, `_directive_values.py:26-47,256`): they name no agent
  and are ignored by both.
- Non-goals: no cross-project bead waits, no TUI affordance for adding bead waits (display only), no bead conditions on
  `#fork` or family attachment.

### Upward close cascade

In sase-core `close_issues`: after closing a plan-type issue whose `parent_id` names a phase-type issue, close that
parent phase too when every one of the phase's children is now closed (same event stream and projections, with a
distinct close*reason such as "delegated work landed"). Apply the rule recursively upward only through phase parents —
never auto-close a plan/epic bead (landing an epic remains its land agent's job; a closed phase can complete a _phase*
grandparent chain only through another delegated epic in between). `remove_issue` deliberately gets no upward cascade:
removal is rollback, and the phase must stay open so a retry reschedules it.

### Delegated-phase scheduling

`build_epic_work_plan` (`crates/sase_core/src/bead/work.rs`) currently schedules every non-closed phase child. A
parent-epic retry while a delegated child epic is in flight would relaunch the phase agent, which could re-plan and
spawn a second child epic. Exclude from waves any phase that owns a non-closed plan-type child; its dependents still
gate correctly through their `bead=` waits, and if the child epic is removed (rollback) the phase is scheduled again.
The "no non-closed phase children" epic error must not fire when open-but-delegated phases exist yet nothing is
schedulable — that state launches only the land segment.

### Emission

`render_multi_prompt` appends, after the existing `%w:<agents>` line, one `%w(bead=<id>)` line per in-epic blocker bead
of the phase (including already-closed blockers — they resolve instantly and keep retries uniform), and on the land
segment one per phase bead of the epic (all children, not just this run's assignments). Source the IDs from new additive
payload fields (per-assignment blocker bead IDs; epic-wide phase bead ID list) exposed by both Rust work-plan bindings
and mirrored into `_PhaseAssignment`/`EpicWorkPlan` by `_plan_from_payload`.

### Resolution

`waiting.json` gains `wait_for_beads: [<bead_id>, ...]` parallel to `waiting_for`. `dependency_resolution_status`
accepts the bead IDs plus an injected bead-status lookup (a callable or prebuilt mapping produced by callers), so the
`wait_dependency_resolution` package stays free of store I/O. Both resolvers supply it: the `wait_checks` chop resolves
each project's beads dir once per cycle and reuses it across that project's markers, and the runner fallback resolves
its own project. Promote `_canonical_beads_dir_for_project` into a shared bead-store locator module used by the chop,
the runner, and the existing mobile helper. Reads go through the existing `BeadProject` Rust read facade.

## Core: upward close cascade and delegated-phase scheduling

In the sase-core linked repo. Implement the upward close cascade in `close_issues`
(`crates/sase_core/src/bead/mutation.rs:362-424`) per the Design section, emitting normal `IssueClosed` events so
JSONL/SQLite projections and search stay consistent; extend the existing cascade tests (`mutation.rs:1167-1288`) with
child-epic-closes-parent-phase, phase-with-second-open-child-stays-open, deep-nesting (`foo-5.2.1.3.1` chains), and
no-cascade-on-remove cases. In `build_epic_work_plan` (`crates/sase_core/src/bead/work.rs`), exclude phases owning a
non-closed plan-type child from waves (Design: delegated-phase scheduling), keeping the land segment launchable when all
remaining phases are delegated. Additively extend the work-plan payload with per-assignment in-epic blocker bead IDs and
the epic's full phase bead ID list, leaving existing fields untouched so the released Python side keeps working until
its floor bump. Run the crate checks (`just rust-check`) and `just bead-perf-smoke`.

## The bead= kwarg on %wait

In this repo's directive layer. Accept `bead` in the `%wait` named-argument allowlist
(`src/sase/xprompt/_directive_collect.py:103-116`), collecting repeatable values into a `wait_bead_args` list alongside
`wait_time_args`/`wait_runners_args`, and update the unsupported-key error text. Add a `resolve_wait_bead_args` step in
`src/sase/xprompt/_directive_values.py` (non-empty word validation, order-preserving dedupe, clear DirectiveError
messages) and a `wait_beads: list[str]` field on `AgentDirectives` (`src/sase/xprompt/_directive_types.py:155-158`)
threaded through the extract pipeline. Bead-only occurrences must not trigger bare-wait resolution or template
substitution (follow the `time=`-only precedent through `resolve_wait_agent_args` and
`has_bare_wait_directive`/`rewrite_bare_wait_directives`). Round-trip the kwarg through directive editing
(`src/sase/xprompt/directive_edit.py`) so ACE edits preserve it. Tests in `tests/test_xprompt/`: grammar
acceptance/rejection, mixing with positional names and other kwargs, dedupe, bare-wait interaction, and edit
round-trips.

## Bead conditions in wait resolution

Thread `directives.wait_beads` into `AgentInfo` (`src/sase/axe/run_agent_directives.py:89-107,232`) and into
`wait_for_dependencies` (`src/sase/axe/run_agent_wait.py:351-586`): bead waits count toward the has-dependencies gate,
are written to `waiting.json` as `wait_for_beads`, appear in the "Waiting for ..." runner message, and are re-checked by
the runner's direct-resolution fallback. Extend `dependency_resolution_status`
(`src/sase/core/wait_dependency_resolution/_resolution.py`) with bead IDs plus an injected closed-bead lookup per the
Design section (unknown → waiting). Update the `wait_checks` chop (`src/sase/scripts/sase_chop_wait_checks.py`): include
`wait_for_beads` when deciding a marker is actionable (`:107-108`), resolve each project's bead store once per cycle
through a new shared locator module promoted from `_canonical_beads_dir_for_project`
(`src/sase/integrations/_mobile_helper_beads.py:213`; refit the mobile helper onto it), and pass the lookup through.
Keep `resolved_deps` memoization agent-only — bead checks are cheap and monotonic. Tests: chop-level scenarios via
`tests/_axe_chop_wait_checks_helpers.py` fixtures (open → parked, closed → ready, missing bead/store → parked, mixed
agent+bead markers), runner-level `wait_for_dependencies` marker and fallback coverage, and resolution-unit cases in the
wait-dependency tests.

## Emit bead waits from sase bead work

Mirror the new payload fields into `_PhaseAssignment`/`EpicWorkPlan` (`src/sase/bead/work.py:40-63,145-198`) and emit
the `%w(bead=...)` lines in `render_multi_prompt` per the Design emission rules, covering both `vcs_context` and
`changespec_context` forms. Bump the `sase_core_rs` dependency floor to the release carrying the core phase (per the
established floor-bump process). Keep `sase bead work --dry-run` and the approval preview
(`src/sase/bead/cli_work_from_plan_render.py`) in parity with the emitted prompt, including bead waits. Tests in
`tests/test_bead/`: segment rendering (first-wave phases get bead waits for closed blockers only when edges exist, later
waves get agent+bead pairs, land lists every phase bead), retry rendering against a store where one phase is delegated
(excluded from waves, dependents still carry its `bead=` wait), and preview parity.

## Waiting surfaces and documentation

Show bead waits wherever agent waits are described: the agents-list waiting description
(`src/sase/integrations/_agent_list_entry_builder.py`), the ACE wait modal and helpers
(`src/sase/ace/tui/modals/wait_modal.py`, `src/sase/ace/tui/actions/agents/_wait_helpers.py`, `_wait_resume.py`) —
display and resume/cancel correctness only, no new add-bead-wait affordance. Docs sweep: the `%wait` kwarg examples and
directive reference in `docs/xprompt.md` (~1062-1071), the wait-resolution description in `docs/axe.md` (~141), and
`docs/beads.md` epic lifecycle — delegated phases, the upward close cascade, retry-skips-delegated-phases, and the
bead-gated barrier in the `sase bead work` section. `docs/agent_families.md` wait-target prose if it enumerates wait
forms. The `%wait` row in the long-term memory `sase/memory/xprompts.md` also becomes stale: memory files require
explicit user permission in the implementing conversation — flag it in the final report; do NOT edit memory files or run
`sase memory init`.

## End-to-end delegation smoke exercises

In a scratch bare-git project with an initialized bead store: author an epic where a medium phase blocks a small phase
and the land agent; have the medium phase agent propose an epic-tier plan (auto-approved) so it delegates to a child
epic; verify the dependent phase and land agent park with `waiting.json` naming both agent and bead conditions and stay
parked after the medium phase agent completes; verify a parent-epic `sase bead work` retry excludes the delegated phase
from waves (dry-run) while the child epic runs; land the child epic and verify the parent phase bead auto-closes with
the delegation close_reason, dependents release, and the land agent runs and closes the epic; exercise the fail-closed
path (a bead wait naming a removed bead keeps parking; cancel via the ACE wait action); check waiting displays in
`sase agent` listings and ACE. Report gaps as findings rather than fixing consequential code.

## Testing

Each phase carries the tests described in its section; Rust work runs `just rust-check` (plus `just bead-perf-smoke` for
the mutation/work changes), Python phases run `just check`. The smoke phase is observation-only. Existing wait suites
must stay green: bead-less `%wait` behavior, waiting-marker shapes consumed by ACE loaders, and chop skip-reasons are
all asserted today and must be extended, not rewritten.

## Risks

- **Missed close path → parked forever.** If any delegation path fails to close a phase bead, dependents and the land
  agent park indefinitely. This is the designed fail-closed behavior (visible parked agents instead of silent premature
  landings), with the ACE wait cancel and manual `sase bead close` as escape hatches; the smoke phase exercises the full
  nested flow.
- **Behavior change for the land agent.** Landing now genuinely blocks on all phase beads closing, including phases
  whose agents crashed before closing. Previously the land agent started anyway and could repair; now the user must
  relaunch or close manually. Documented in the beads docs update.
- **Rust/Python lockstep.** Payload additions must stay additive; the floor bump lands in the emission phase, and the
  upward cascade is inert for Python until then. A wire-version bump would break ordering and must be avoided.
- **Chop cost.** One bead-store read per project per chop cycle; the store locator reuses the read facade's cached
  SQLite and only runs for projects that actually have bead-wait markers.
- **Auto-close correctness.** The upward cascade fires only when all of a phase's children are closed, so a phase with a
  second in-flight child epic stays open; manual `sase bead close` of a child epic triggers the same rule by design
  (uniform domain behavior).
