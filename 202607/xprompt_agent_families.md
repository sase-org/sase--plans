---
tier: epic
title: xprompt agent families via a %family directive
goal: 'Any launch can group parallel agents into one agent family purely through xprompt
  syntax: a new %family directive gives members execution-neutral family membership,
  members fold under a real root row whose status and metadata aggregate every member,
  and slot accounting, kill cascade, and wait/fork semantics are made safe for parallel
  members. Epic bead-work launches and the chezmoi research swarm both ship on the
  new directive.

  '
phases:
- id: directive
  title: '%family directive grammar, parsing, and validation'
  depends_on: []
- id: wire_admission
  title: Parallel-member wire field and runner-slot accounting
  depends_on: []
- id: launch_runner
  title: Launch-time family resolution and runner membership metadata
  depends_on:
  - directive
  - wire_admission
- id: kill_cascade
  title: Kill/dismiss cascade for parallel family members
  depends_on:
  - launch_runner
- id: tui_aggregate
  title: Aggregate root status and member counts on the Agents tab
  depends_on:
  - launch_runner
- id: tui_panel
  title: Consolidated root metadata panel and launch preview
  depends_on:
  - tui_aggregate
- id: epic_usecase
  title: Epic bead-work launches adopt %family
  depends_on:
  - launch_runner
  - tui_aggregate
- id: swarm_usecase
  title: Research swarm adoption and end-to-end verification
  depends_on:
  - tui_panel
  - epic_usecase
create_time: 2026-07-16 18:55:59
status: wip
bead_id: sase-6g
---

# Plan: xprompt agent families via a `%family` directive

## Context

Today three uncoordinated grouping mechanisms exist: plan-chain families (`foo--code` +
`agent_family`/`parent_timestamp` metadata, serial handoff semantics), hoods (`foo.bar`, navigation only), and tags
(`%group`, a side panel per tag). Parallel agents launched from one multi-prompt (epic phase workers + the land agent;
the chezmoi research swarm) have no way to group under one root row, so they sprawl across the Agents tab and their
metadata is scattered.

Prior research (`sase--research` sidecar,
`202607/parallel_family_children_consolidated/parallel_family_children_consolidated.md`) established that parallel
execution itself already works — both use cases spawn all agents up front and self-gate via `%wait`. The real work is
(a) execution-neutral membership, (b) true root aggregation, and (c) defusing three pre-existing hazards that membership
activates: family children are exempt from `max_running_agents` accounting, kill does not cascade to family members, and
root status is a literal mirror of one child. The research proposed writing epic membership "typed" from the launcher;
this plan supersedes that with a fully generic mechanism: **membership is declared only in xprompt syntax**, and the
epic launcher simply emits the same directive text any user can write.

Two findings made during plan preparation correct/extend the research and shape the design:

1. **Multi-prompt segments are planned and spawned one at a time** — timestamps are allocated per segment
   (`src/sase/agent/multi_prompt_launcher.py`, `_assign_missing_slot_timestamps`), not batch-preallocated across
   segments. A member launching before its root therefore cannot see the root's timestamp unless the launcher
   pre-allocates the root segment's name and timestamp in a pre-pass. This plan adds that pre-pass.
2. **Intra-family base-name references self-deadlock or mis-resolve.** Once the swarm's `final` agent carries
   `agent_family` metadata, `%wait:research.@.final` from the image agent would resolve through
   `is_agent_family_complete` — whole-family completion including the image agent itself (deadlock), and
   `#fork:research.@.final` would resolve to the newest successful _member_ instead of the lead's chat. Rule adopted:
   **inside a family, the base name refers to the root agent; outside, it refers to the whole family.**

## Design

### Directive syntax

New single-value directive `%family` (alias `%f`), stripped from the prompt like all `%` directives, defined in
`src/sase/xprompt/_directive_types.py` and parsed alongside the others:

```text
%family:<root-name>                    # join the family rooted at <root-name>
%family(<root-name>, role=<token>)     # optional role recorded as agent_family_role
```

- `<root-name>` may be a static agent name (`sase-6e`) or a name template (`research.@.final`). It is matched against
  the `%name` values of sibling segments in the same multi-prompt launch (template text matches template text, so `@`
  correlation needs no early resolution to _identify_ the root segment).
- The **root segment needs no directive**: the segment whose `%name` (force-reuse `!` prefix stripped) equals the
  members' target automatically becomes the family root.
- `role` defaults to `member`; the root records role `root` unless it carries an explicit one. Roles are free-form
  tokens stored in `agent_family_role` (already a free-form string on the scan wire); deliberately not planner roles
  (`plan`/`feedback`), so plan-chain supersession heuristics cannot fire.
- If no sibling segment matches the target, the target resolves against existing agents on disk (newest family
  generation root / exact named agent). This one fallback covers three real needs with a single mechanism:
  single-segment prompts that join an existing family, retries of a member re-resolving membership after relaunch, and
  late additions to a family launched earlier. If nothing resolves, the launch fails with a clear error.

### Semantics: membership is execution-neutral

`%family` writes metadata and nothing else. It must not change waits, workspace assignment, agent name, model/VCS
resolution, `workflow_name`, or launch ordering. This is the core contract distinguishing it from `%n(parent, suffix)`
(which bundles membership with lineage, an implied wait, and workspace inheritance — correct for serial handoff, wrong
for peers). `%family` and `%n(parent, suffix)` in the same segment is a validation error.

### Data model (no new grouping concept — reuse the family star)

Members get, in `agent_meta.json`:

- `agent_family: <resolved root name>` — family base (existing field).
- `parent_timestamp: <root artifact timestamp>` — generation identity + TUI row folding (existing field;
  `AgentChildLinkage.FAMILY_MEMBER` in `src/sase/ace/tui/models/agent.py` keys off it, and `find_agent_family` in
  `src/sase/agent/names/_lookup.py` groups a generation as root.timestamp + members pointing at it).
- `agent_family_role: <role>` (existing field).
- `agent_family_parallel: true` — **new** explicit marker distinguishing parallel members from serial plan-chain
  members. Explicit beats inference: admission, kill cascade, and the TUI aggregate all key off this flag rather than
  guessing from name shapes or role sets.

Members do **not** get `workflow_name = base` (avoids the `release_workspace` `(workspace_num, workflow, cl_name)`
collision hazard and keeps `is_workflow_complete` semantics untouched), do **not** get `role_suffix`, and do **not** get
`plan_chain_parent_timestamp`. The root gets `agent_family: <own name>`, `agent_family_parallel: true`, its role, and
**no** `parent_timestamp` (it stays a root record).

Generation identity comes free: re-running `sase bead work` (force-reuse names, new timestamps) creates a new root
timestamp, so `find_agent_family` naturally selects the fresh generation and old failures cannot poison it.

### Launch pipeline (the part the research missed)

In `_spawn_segments_into` (`src/sase/agent/multi_prompt_launcher.py`), a family pre-pass runs before the segment loop:

1. Text-scan each segment for a top-level `%family` directive and its `%name` value/template (reusing the
   `extract_static_name_directive`-style protected-region scanning so fenced blocks and disabled regions are respected).
2. Validate: exactly one root segment per family target; the root segment must resolve to exactly one launch slot
   (reject `%{...}` fan-out on a root segment; fan-out on _member_ segments is legal — every variant joins); a segment
   cannot target itself; duplicate `%family` in one segment is an error.
3. For each family with an in-launch root: pre-allocate the root segment's timestamp from the shared
   `LaunchTimestampBatchAllocator`, pre-plan its name (static names as-is; templates through the existing
   `PlannedNameAllocator` with the segment's template group so the `@` binding is pinned once and reused when the root
   segment actually launches), and pre-resolve the root segment's VCS context to compute its future artifacts dir.
4. During the loop, pin the root segment's slot timestamp and planned name to the pre-pass values, and inject a per-slot
   env payload (new `SASE_AGENT_FAMILY_MEMBERSHIP`, JSON — modeled on `FAMILY_ATTACH_ENV`) carrying: family base, root
   timestamp (artifacts format), root artifact dir, role, and an `is_root` flag.
5. Runner side (`extract_directives_and_write_meta` in `src/sase/axe/run_agent_directives.py`): consume the payload and
   write the membership fields above. When the `%family` directive is present but no payload exists (single-segment
   launch, retry), resolve the target on disk and write the same fields — the runner-side fallback.

Intra-family reference handling, wired at the same points:

- A member whose `%wait` list names its own family base gets that wait satisfied against the root artifact specifically,
  via the existing identity-pinned `wait_for_artifacts` mechanism (the same one `%n(parent, suffix)` uses when the
  parent is running) — the root's artifact dir and timestamp are in the payload. Defensively, `resolve_wait_dependency`
  / `resolve_resume_agent_name` (`src/sase/agent/names/_lookup.py`) gain a self-family check so a member's base-name
  wait/`#fork` resolves to the root agent exactly, never to whole-family completion or a sibling's chat. External
  waiters keep (and gain) the documented semantics: `%wait:<root>` from outside the family means whole-family
  completion.

### Aggregate root status (replacing the single-child mirror)

At the existing rollup chokepoint (`src/sase/ace/tui/models/_agent_status_apply.py`, the root-derivation block at the
end of `apply_status_overrides`): when a root's children include parallel members (`agent_family_parallel`), compute an
aggregate instead of mirroring one child. Priority order (first match wins), computed over the root plus all members:

1. needs-user-input (any member QUESTION / awaiting plan review)
2. failed (any member failed/killed)
3. running (any member running)
4. waiting (any member waiting)
5. done (all members terminal-success)

The row additionally renders member counts (`2 running · 1 done`), reusing the group-banner summary pattern
(`banner_summary_text` / `compute_banner_summary` in `src/sase/ace/tui/models/agent_groups/_tree.py`) in the
fold-annotation slot of `format_agent_option`, with the render cache key (`_agent_list_render_cache.py`) extended so
count changes invalidate rows. The serial plan-chain mirror path is preserved bit-for-bit (gated on the absence of
parallel members) — the existing status-override test suites are the golden-equivalence harness. Aggregation is
implemented as a small pure function at this presentation seam (it folds TUI display statuses, including TUI-only
overrides); a sase-core-owned aggregate over raw scan markers is the documented graduation path if another frontend
needs it, per the Rust-core boundary rule.

### Hazards being fixed (pre-existing, activated by parallel membership)

1. **Slot accounting** — `is_root_user_agent_record` (`src/sase/core/runner_slots/_admission.py`) excludes any record
   with `parent_timestamp`, from both the running count and the FIFO slot queue, so parallel members would bypass
   `max_running_agents` entirely. Fix: records with `agent_family_parallel` are countable and queueable; serial
   plan-chain members keep the existing exemption (no behavior change for plan chains). Requires `agent_family_parallel`
   on the Rust scan wire (`AgentMetaWire` in sase-core `agent_scan/wire.rs` + the hand-mirrored Python dataclass +
   parity fixtures).
2. **Kill cascade** — `_immediate_kill_identities` (`src/sase/ace/tui/actions/agents/_kill_identity.py`) and the Rust
   cleanup planner (`agent_cleanup/planner.rs`) cascade only to workflow-step children. Killing an epic root would
   orphan N live phase processes. Fix: cascade to parallel members (linkage: `parent_timestamp == root.raw_suffix` + the
   parallel flag) in both the legacy Python path and the Rust planner (`AgentCleanupTargetWire` gains the flag;
   schema-version bump + parity tests per the documented wire process). Killing a single member does not cascade;
   plan-chain kill behavior is unchanged.
3. **Role misclassification** — epic member names (`sase-6e.4`) have dotted-numeric suffixes that
   `_parse_plan_chain_suffix` (`src/sase/plan_chain.py`) classifies as legacy `feedback` _before_ consulting the stored
   `agent_family_role`. Fix (Python-only): explicit stored roles take precedence over the legacy numeric-feedback
   inference; behavior with no stored role is unchanged. This keeps planner-family supersession heuristics
   (`PLANNER_FAMILY_ROLES`) off parallel members even when their names look like feedback rounds.

## Phases

### Phase `directive` — grammar, parsing, validation

- Add `family` to `_KNOWN_DIRECTIVES`, alias `f`, in `src/sase/xprompt/_directive_types.py`; extend `PromptDirectives`
  with `family_target: str | None` and `family_role: str | None`.
- Parse colon and paren forms in `_directive_collect.py` / `_directive_values.py` / `_directive_extract.py`
  (single-value; duplicate occurrences and empty targets are targeted errors; `role` validated as a bare token).
  Directive text is stripped from the cleaned prompt exactly like the others.
- Validation errors (raised where other directive errors are raised, so every launch surface shows them): `%family`
  combined with `%n(parent, suffix)`; bare `%family` with no target.
- Editor/completion surfaces that enumerate known directives (directive completion, `directive_edit.py`, catalog/explain
  output) pick the new directive up from the shared tables; verify and extend where a hardcoded list exists.
- Tests: parsing round-trips (static names, templates with `@`, role kwarg, alias, fenced blocks/disabled regions
  ignored), duplicate/conflict errors, prompt-cleanup snapshots.
- Note: with only this phase landed the directive parses and strips but has no effect; that intermediate state is safe.

### Phase `wire_admission` — wire field + slot accounting (inert until writers exist)

- sase-core (open via `/sase_repo`; sibling linked repo): add `agent_family_parallel` (serde-default false) to
  `AgentMetaWire` in `crates/sase_core/src/agent_scan/wire.rs`, populate from `agent_meta.json`, mirror in
  `src/sase/core/agent_scan_wire_markers.py`, update parity fixtures/tests (`python_wire_parity.rs`,
  `agent_scan_parity.rs`) and bump the scan-wire schema version only if the JSON shape requires it; rebuild the local
  wheel (`just rust-install`) so `sase_core_rs` exposes the field.
- `src/sase/core/runner_slots/_admission.py`: parallel members (flag set) are included in `running_root_agent_count` and
  `live_runner_slot_waiters`; the `parent_timestamp` exemption remains for records without the flag.
  `running_listing.py` keeps its display semantics (decouple listing from admission: members still render as children,
  not as extra root listing rows — audit the one shared predicate and split it if needed).
- Tests: admission counts/queue ordering with mixed root + parallel-member records; serial plan-chain records unchanged;
  a parallel member parks in the FIFO queue and is admitted in order.

### Phase `launch_runner` — launch pre-pass + runner metadata + reference semantics

- The pre-pass, env payload, slot pinning, and runner writes described in Design → Launch pipeline. Key files:
  `src/sase/agent/multi_prompt_launcher.py`, `src/sase/agent/multi_prompt_reference_directives.py` (segment text
  scanning helpers), `src/sase/agent/launch_executor.py` (respect pre-pinned slot timestamps),
  `src/sase/axe/run_agent_directives.py` (metadata writes + wait identity deps), `src/sase/agent/names/_lookup.py`
  (self-family wait/fork exception), `src/sase/plan_chain.py` (stored-role precedence fix).
- Runner-side fallback resolution for `%family` without a payload (on-disk root lookup).
- Orphan tolerance: family lookups and the TUI must behave sanely in the window where members exist but the root
  artifact has not spawned yet (members render as normal rows until the root row appears, then fold under it).
- Execution-neutrality regression tests (the core reliability gate): launching the same multi-prompt with and without
  `%family` produces identical waits, workspace numbers, VCS refs, models, and admission counts — only the membership
  metadata differs.
- Reference-semantics tests: member `%wait` on the base is identity-pinned to the root and does not deadlock; member
  `#fork:<base>` resolves to the root's chat; external `%wait:<base>` resolves as whole-family completion; `%wait` on a
  member name stays an exact lookup.
- Generation tests: relaunching the family (force-reuse names) starts a fresh generation; a member retry rejoins the
  current generation via the fallback.

### Phase `kill_cascade` — root kill/dismiss covers parallel members

- Rust cleanup planner: `AgentCleanupTargetWire` gains the parallel-member flag; planner cascades a root kill to live
  parallel members (matching `parent_timestamp` linkage); schema-version bump + parity fixtures; Python mirror in
  `src/sase/core/agent_cleanup_wire.py` and target collection.
- Python legacy path `_immediate_kill_identities` mirrors the cascade so both paths agree; the dismissal gate's existing
  family check is verified/extended so dismissing a root dismisses the folded member rows consistently.
- Tests: killing an epic-shaped root kills all live members (no orphan processes); killing a single member leaves
  siblings and root running; plan-chain kill behavior unchanged.

### Phase `tui_aggregate` — aggregate status + member counts

- The aggregate described in Design → Aggregate root status, at `_agent_status_apply.py`'s root-derivation block, plus
  row counts and render-cache key updates. Reuse `runtime_children` (already attached in `_agent_ordering.py`) as the
  member list; `_aggregate_runtime` in `agent_time.py` is the cycle-guard pattern to copy.
- Guardrails: the mirror path for serial chains is untouched (existing suites such as
  `tests/test_agent_loader_status_override_followup_roots.py` pass unmodified except where they encode "newest single
  child wins" for _parallel_ members, which no test does today); supersession heuristics skip parallel members.
- Tests: unit tests over the aggregate priority order for simultaneous RUNNING/WAITING/QUESTION/DONE/FAILED members
  (root WAITING while members run must show running, not waiting); PNG visual snapshots for a family root row with mixed
  member states and counts (`tests/ace/tui/visual/test_ace_png_snapshots_agents.py` already has family fixtures to
  extend). Read `sase/memory/tui_perf.md` before touching loaders/render paths — no new synchronous I/O in loaders;
  counts derive from already-loaded agents.

### Phase `tui_panel` — consolidated root panel + launch preview

- New navigable "Members" section in `build_header_text`
  (`src/sase/ace/tui/widgets/prompt_panel/_agent_display_header.py`) via
  `append_section_heading(..., section_id="members")`: one line per member — role, name, status, model, elapsed —
  ordered by launch time. Selecting the root shows the family at a glance; child panels remain available by unfolding
  (`l`) and selecting a member.
- Audit the existing upstream child-field merging in `apply_status_overrides` (plan/feedback times, `meta_*`, diff path)
  so parallel members participate where the merge is already generic; per-member attribution stays visible through the
  Members section rather than through new one-off copy rules. Tokens/cost are explicitly out of scope (no such fields
  exist on Agent; a metrics subsystem is its own future effort).
- Launch preview (`src/sase/agent/launch_preview.py`): annotate segments that declare or root a family so LaunchApproval
  shows the grouping before spawn.
- Tests: header snapshot with a Members section; section navigation reaches it; preview text.

### Phase `epic_usecase` — `sase bead work` emits `%family`

- `render_multi_prompt` (`src/sase/bead/work.py`): phase segments append `%family(<land_agent_name>, role=phase)`; drop
  the per-epic `%group:<launch_tag_id>` from phase and land segments (the family row replaces the per-epic tag panel —
  two grouping mechanisms must not fight over the same agents). The land segment needs no new directive: its
  `%name:!<epic_id>` makes it the root. No naming changes, no env changes (`epic_work_segment_env` untouched).
- Tests: rendered multi-prompt snapshots; an integration-level test that an epic-shaped launch produces one root row
  with N folded members, correct aggregate transitions (phases running → land waiting shows running at root; phase
  failure shows failed at root), and correct slot accounting.
- Documentation: update `docs/agent_families.md` with the directive, semantics, and the epic example. (Any
  `sase/memory/*.md` or glossary updates are intentionally excluded — memory edits require the user's explicit approval
  and are listed as a follow-up for the user, not performed by any phase agent.)

### Phase `swarm_usecase` — research swarm adoption + end-to-end verification

- Update `home/sase/xprompts/research_swarm.md` in the chezmoi linked repo (open via `/sase_repo`; after committing, run
  `chezmoi update -a --force` per that repo's instructions): add `%family(research.@.final, role=researcher)` to the two
  researcher segments and `%family(research.@.final, role=image)` to the image segment; the lead (`research.@.final`)
  becomes the root automatically. Keep `%g:research` — it is a cross-launch tag, orthogonal to the per-launch family.
  Note: the request named `research_swarm_kiss`, which does not exist; `research_swarm.md` (the current KISS rewrite) is
  the confirmed target per the prior research.
- End-to-end verification of both use cases (a real swarm launch and an epic-shaped launch): one root row each, folded
  members, aggregate counts, root kill cascades, image agent's wait/fork on the lead resolves to the lead exactly,
  members count against `max_running_agents`.
- Sweep for regressions in the full test suite and visual snapshots.

## Edge cases addressed

- Root launches after members: root name + timestamp pre-allocated in the pre-pass; members carry the pointer from
  birth; TUI tolerates the pre-root window.
- Member `%wait`/`#fork` on the family base: root-agent-exact resolution (no self-deadlock, no wrong fork source).
  External base-name waits: whole-family completion, documented.
- Retry of a member: runner-side fallback re-resolves membership into the current generation.
- Family relaunch (`sase bead work` re-run): fresh generation via new root timestamp; stale failures cannot poison it.
- Fan-out (`%{a | b}`) on member segments: every variant joins; on a root segment: rejected.
- Names never constrain membership: dotted bead IDs (`sase-6e.4`), swarm names (`research.3.cdx`), templates, and
  auto-names all work — explicit metadata is authoritative; the accidental name-based family classification of dotted
  IDs stays inert (stored-role precedence fix guards the one path where it could bite).
- Workspace release safety: members keep per-slot `workflow_name`, so the `(workspace_num, workflow, cl_name)` release
  match cannot cross members.
- A failed member never silently disappears: the aggregate surfaces failure at the root. (Terminal cancellation of
  parked waiters on generation failure remains an explicit non-goal; the wait system's park-until-retry behavior is
  preserved because relaunching a failed member under the same name must keep unblocking its waiters.)
- `%family` + `%hide`, `%family` with `%group`: allowed and orthogonal. `%family` + `%n(parent, suffix)`: error.

## Non-goals (v1)

- Token/cost aggregation (no such per-agent fields exist; separate subsystem).
- Synthetic/rootless group records, cross-project families, per-family concurrency caps, dynamic post-launch membership,
  group-scoped actions beyond kill/dismiss.
- Terminal cancellation of dependents when a generation terminally fails (follow-up; design note: must key off
  generation-terminal state, not first member failure).
- Changing `%n(parent, suffix)`, the axe loop, or the launch scheduler.

## Risks and mitigations

- **Breaking serial plan chains** — every behavioral change is gated on `agent_family_parallel`; existing
  status/kill/admission suites act as golden tests and must pass unmodified.
- **Launcher pre-pass complexity** (template pinning, VCS pre-resolution) — confined to `_spawn_segments_into`; the
  pre-pass only activates when a `%family` directive is present, so every existing launch path is bit-for-bit unchanged
  without it.
- **Rust/Python wire drift** — follow the documented mirror + parity-test + schema-version process in both wire changes;
  rebuild the wheel before running Python tests.
- **TUI performance** — read `sase/memory/tui_perf.md` first; aggregation folds already-loaded rows, adds no I/O, and
  extends render-cache keys explicitly.

## Acceptance criteria

1. A multi-prompt using only xprompt syntax (`%name`, `%wait`, `%family`) yields N concurrently-running members folded
   under one root row, from launch time.
2. Adding/removing `%family` changes no waits, workspace assignments, VCS refs, models, or names (execution neutrality,
   regression-tested).
3. Every parallel member counts toward `max_running_agents` and participates in the slot queue; killing the root kills
   all live members; plan-chain behavior is unchanged.
4. Root status/counts are correct for simultaneous RUNNING/WAITING/QUESTION/DONE/FAILED members; a member failure is
   visible at the root row.
5. Root panel shows a per-member section (role, name, status, model, elapsed); member panels remain reachable;
   tokens/cost intentionally absent.
6. Intra-family base-name `%wait`/`#fork` resolve to the root agent; external base-name waits mean whole-family
   completion; both tested.
7. `sase bead work` epics and the chezmoi research swarm both render as single family roots with correct aggregates, and
   a family relaunch starts a clean generation.
