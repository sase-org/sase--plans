---
tier: epic
title: '%wait(..., for_epic=): a wait that follows its agent into the epic it launches'
goal: '`%wait:planner` waits for the planner and then for every epic bead the planner
  (or any member of its session, clan, workflow, or bound tribe) launches, so users
  can submit follow-up prompts before the epic''s ID exists. A per-occurrence `for_epic=true|false`
  keyword controls the behavior. It defaults to true for user-authored agent targets,
  and using it without an agent target is a hard error in the launcher and the editor.
  The hand-off is recorded reliably, never deadlocks the epic machinery, and is clearly
  visible in the TUI as a teal `↪` hand-off.

  '
phases:
- id: record
  title: Record the epics a run launched
  depends_on: []
  size: medium
  description: 'record: write an authoritative, lock-protected `created_epics` list
    on the creating run''s agent_meta.json when `sase bead work` materializes an epic.
    Add readers with fallbacks, stop overwriting workers'' inherited `epic_bead_id`,
    and mirror the field in the Rust and Python scan wires.

    '
- id: links
  title: Derive produced-by links from recorded epics
  depends_on:
  - record
  size: small
  description: 'links: publish portable `created_epic_ids` and project `bead:<epic>
    produced-by agent:<creator>` edges, both from published metadata and from bead-store
    attribution, so unpublished planners also get the edge. Widen the relation guidance
    in sase-core.

    '
- id: contract
  title: Grammar, diagnostics, and persisted policy
  depends_on:
  - record
  size: medium
  description: 'contract: accept and strictly validate `for_epic=` on `%wait` with
    identical launcher and editor errors. Compute the effective positive `wait_for_epics_of`
    list (default still false), persist it in agent_meta.json and waiting.json and
    the scan wires, round-trip it through PromptWaitDirective, and add editor completion.

    '
- id: reducer
  title: Epic-follow reducer and fact collector
  depends_on:
  - record
  size: medium
  description: 'reducer: add the pure sase-core reducer that maps per-member facts
    to AGENT/NONE/LAUNCHING/FOLLOWING/BLOCKED with the deadlock guard and the cycle
    hook, bind it to Python, and add the Python fact collector over the wait-dependency
    index. Nothing is wired into release paths yet.

    '
- id: release
  title: Follow through in every release path
  depends_on:
  - contract
  - reducer
  size: large
  description: 'release: route every release path (runner initial check, parked-runner
    fallback, AXE wait_checks chop, kill/dismiss) through one shared decision function.
    Promote FOLLOWING targets into pinned bead waits under the directive lock with
    compare-and-set, persist `wait_epic_follows`, fix the two-stage rewrite clobber,
    and document the opt-in keyword.

    '
- id: model
  title: Follow state in the agent model and shared view
  depends_on:
  - release
  size: medium
  description: 'model: load `wait_for_epics_of` and `wait_epic_follows` into the TUI
    Agent model and render cache key. Make wait-satisfied and status-count logic follow-aware
    without I/O, and add one shared follow view model plus plain-text phrasing for
    every surface.

    '
- id: safety
  title: Blocker notifications and the cycle guard
  depends_on:
  - release
  size: medium
  description: 'safety: send one deduplicated inbox notification for a LAUNCHING follow
    past its grace period, for a BLOCKED follow (with the resume command), and for
    a followed epic whose land agent failed. Fill the reducer''s cycle facts, and
    document the states in docs/axe.md.

    '
- id: tui
  title: The ↪ hand-off in rows, lanes, toasts, and timeline
  depends_on:
  - model
  size: medium
  description: 'tui: render the teal `↪` hand-off in agent rows and in the detail
    `[agents]` lane. Add one coalesced toast on live transitions, a `↪EPIC` timeline
    milestone, phase progress, help legend, docs, and visual snapshot goldens.

    '
- id: surfaces
  title: Wait modal toggle, CLI, Jinja, and Telegram parity
  depends_on:
  - model
  size: medium
  description: 'surfaces: add a tri-state "Follow epics" toggle to the `w` wait modal,
    filter derived beads out of edits and relaunch rewrites, export the follow state
    in `sase agent list -j` and `sase agent wait` rows, synthesize `agents["x"].created_epic(s)`
    for Jinja, and show the hand-off in Telegram status text when waits are shown
    there.

    '
- id: flip
  title: Flip the default on and finish the docs
  depends_on:
  - links
  - safety
  - tui
  - surfaces
  size: medium
  description: 'flip: make `for_epic` default to true for user-authored agent targets
    (never `--plan` rows), have `sase bead work` emit `for_epic=false` for intra-epic
    sequencing waits, and prove with regression tests that epic phases and land agents
    still release. Update editor docs and user docs, and propose the memory follow-up.'
proposed_by: bbugyi200.athena.research.3v.linker.w0
create_time: 2026-10-06 18:17:30
status: done
bead_id: sase-1h7
---

- **PROMPT:** [prompts/202610/wait_for_epic.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/wait_for_epic.md)
- **BEAD:** [sase-1h7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1h7/README.md)

# Plan: `%wait(..., for_epic=)`, a Wait That Follows Its Agent Into Its Epic

## Background

Today the motivating flow takes two steps. You launch a planner, watch until its epic
bead exists, copy the ID, and only then launch the follow-up with `%wait(bead=<epic>)`.
`docs/axe.md` documents the gap: "A wait on an epic-approved planner waits for that
planner, not for the host-owned epic it launched."

The meaning of `%wait:planner` also depends on a tier choice the planner makes later.
When the planner picks a tale, the wait covers the implementation, because the tale
coder `<base>--code` is a session member. When it picks an epic, the wait releases at
the epic launch. This epic makes `%wait:planner` mean "the planner's work is done"
either way.

Read the consolidated research report before starting any phase. The user agreed with
all of its recommendations (A1–A10, the 13 disagreement resolutions, and the Q1–Q4
recommendations):

```bash
sase artifact read explicit:ccd173b1ad8ee7cd437bf3e7 "Context for the %wait for_epic epic"
```

This plan is the authoritative contract. Where it deviates from the research, the reason
is given in [Deviations from the research](#deviations-from-the-research). Paths are
relative to the sase repo unless marked `sase-core:`. sase-core is the linked repo,
opened with `sase repo open sase-core`. Shared backend logic belongs in sase-core, per
the Rust core boundary. A sase-core change that sase calls needs
`sase-core-revision.txt` moved past it, using the procedure in `docs/rust_backend.md`.

## Contract (every phase honors this)

### Grammar

```text
%wait:planner                          # planner, then every epic it launches (default after `flip`)
%wait(planner, reviewer)               # both targets follow
%wait(planner, for_epic=false)         # release when planner finishes, even if it launched an epic
%wait(planner, time=30m)               # follow, then a 30m floor after everything resolves
%wait(planner) %wait(bead=sase-87.2)   # follow planner AND require sase-87.2 closed
%wait(a, b, for_epic=false) %wait:c    # a and b agent-only; c follows
```

- `for_epic` applies to every agent target in its own `%wait(...)` occurrence. Agent
  targets are positional names, `agent=`, `@tribe` references, and session, clan, and
  workflow names. They are whatever the existing name resolver accepts.
- The colon form never takes keywords, so it always gets the default. To opt out, use
  the parenthesized form.
- Values are `true` or `false`, case-insensitive, matching `%if(should_run=)`. A
  duplicate keyword within one occurrence is already rejected.
- An explicit value overrides another occurrence's default. Two explicit values conflict
  only when one is `true` and the other is `false` for the same target.
- `<base>--plan` targets never follow, because a plan row deliberately releases at
  submission. Use `%wait:<base>` to wait through approval and into the epic.

**Errors.** These are hard errors, worded identically at launch (`DirectiveError`) and
in the editor (an Error-severity diagnostic in LSP and the prompt bar):

| Code                          | Case                                              | Message                                                                                                                                                                |
| ----------------------------- | ------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `wait-for-epic-without-agent` | `for_epic=` in an occurrence with no agent target | `%wait(for_epic=...) needs an agent target in the same %wait, e.g. %wait(planner, for_epic=false). bead=, hood=, proc=, unit=, and time= waits never launch epics.`    |
| `wait-for-epic-invalid-value` | value not true/false                              | `Invalid %wait for_epic= value 'yes': use true or false.`                                                                                                              |
| `wait-for-epic-conflict`      | explicit true and explicit false for one target   | `Conflicting for_epic= values for %wait target 'planner'.`                                                                                                             |
| `wait-for-epic-plan-row`      | explicit `for_epic=true` on a `--plan` target     | `%wait target 'planner--plan' cannot use for_epic=true: --plan rows release when the plan is submitted. Use %wait:planner to wait through approval and into its epic.` |

`%wait(for_epic=false)` with no positional target is an error, matching the user's
requirement that the keyword needs an agent name. A bare-target `%wait` resolves to the
previous agent and gets the default.

### Persistence

| Where                                                   | Field                                                                                                                              | Written by                          | Meaning                                                                                            |
| ------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------- | -------------------------------------------------------------------------------------------------- |
| creator's `agent_meta.json`                             | `created_epics: [{bead_id, project, plan_ref, created_at, via}]`                                                                   | `sase bead work` (`record`)         | Authoritative run → epic record. `via` is `host_launch` or `agent_command`.                        |
| `PromptDirectives`                                      | `wait_for_epics_of: list[str]`                                                                                                     | parser (`contract`)                 | Effective, positive armed-target list.                                                             |
| `PromptWaitDirective`                                   | `epic_follow_agents: tuple[str, ...] \| None`                                                                                      | rewrite callers                     | `None` means every agent uses the default.                                                         |
| waiter `agent_meta.json` and `waiting.json`             | `wait_for_epics_of`                                                                                                                | runner at launch; TUI/ops edits     | Armed targets. Exact strings from `wait_for`/`waiting_for`. **Absent means never follow.**         |
| waiter `waiting.json` and mirrored in `agent_meta.json` | `wait_epic_follows: [{target, state, epic_ids, added_bead_ids, members, since, reason, detail, resume_command, skipped_epic_ids}]` | shared release decision (`release`) | Persisted stage. `state` is one of `launching`, `following`, `blocked`; `none` is never persisted. |
| waiter `waiting.json` and `agent_meta.json`             | `wait_for_beads` gains `added_bead_ids`; `resolved_deps` gains the target                                                          | promotion (`release`)               | Reuses all bead-wait machinery and pins the target.                                                |

Mirror every new field in sase-core's `AgentMetaWire`/`WaitingMarkerWire`
(`sase-core: crates/sase_core/src/agent_scan/wire.rs` plus the hand-written readers in
`scanner.rs`, `agent_meta_from_object` and `waiting_marker_from_object`). Also mirror
them in the Python dataclasses in `src/sase/core/agent_scan_wire_markers.py` and their
converters in `agent_scan_wire_conversion.py`. Use commit `988af8f3bb`
(`finalizer_status`) as the template for a new wire field. Fields are additive and
serde-default. Bump `AGENT_SCAN_WIRE_SCHEMA_VERSION` only if the indexed-row cache would
otherwise serve rows without the field. If you bump it, keep the Python
supported-version set in sync.

### Which epics count

An epic counts when `sase bead work` materializes an epic-tier plan bead on a run's
behalf. That covers a host-approved launch of a plan the run proposed, and the run's own
`sase bead work <plan>` call. Two kinds of epic never count:

- the epic a phase or land worker belongs to, which is its inherited `epic_bead_id`
  (counting it deadlocks every epic);
- a backlog epic filed with `sase bead create`.

Epics are followed from **every member run** of the entity the existing resolver
selected for the target. A target waits for **all** of them.

### Follow state machine

The reducer evaluates each armed target only after the existing predicate has resolved
that target's entity:

```text
            agent predicate unresolved
  AGENT ◄──────────────────────────────── (as today)
    │ resolved
    ├── epics E non-empty (after the deadlock guard) ───► FOLLOWING (or BLOCKED: cycle)
    ├── E empty, a launch reservation in flight/settling ─► LAUNCHING (overdue after grace)
    ├── E empty, a reservation that ended without an epic ─► BLOCKED (resume hint)
    └── E empty, no reservation ──────────────────────────► NONE → released exactly as today
```

BLOCKED fails closed but is never terminal. Each evaluation recomputes the state, so an
epic that appears later (for example, after the user runs the resume command) moves the
target to FOLLOWING.

### Promotion and pinning

When a target first reaches FOLLOWING, a writer runs under the per-artifact directive
lock with a compare-and-set check. It re-reads the marker and aborts if the target is no
longer armed or present. It then:

1. appends the epics that are not already present to `wait_for_beads` in both files and
   records them as `added_bead_ids`;
2. adds the target to `resolved_deps`;
3. writes the FOLLOWING entry.

Release then follows the ordinary path: all dependencies including the followed beads,
then the time floor, then capacity, then holds. The promotion write never writes
`ready.json` itself; the next evaluation does. Once a target is pinned, re-runs of the
same name cannot redirect it, and the target's artifacts may be dismissed. Derived beads
are never written back into authored `%wait` text.

### Presentation (summary; details in [tui](#phase-tui))

- **Glyph and colours.** `↪` in bold `#5FD7AF`, the same teal as `EPIC CREATED`. Bead
  tokens reuse the existing bead glyphs, and IDs stay `#FF87D7`.
- **Row.** It progresses `WAITING ▶1` → `WAITING ↪ epic…` → `WAITING ↪ ◐ sase-7k`.
  Several epics read `↪ ◐2`, and a blocked follow reads `↪ !` in red.
- **Lane.** The hand-off is narrated in place in `[agents]`, the way `@review → name ✓`
  already reads.
- **Signals.** Add one coalesced toast, only on a transition the live TUI observes, and
  a `↪EPIC` timeline milestone. Inbox entries are for blockers only.

## Deviations from the research

1. **When `created_epics` is recorded.** The research says to record "right after
   `plan_link_committed`", but the epic can still be rolled back after that point. A
   zero-spawn failure triggers `_rollback_epic_creation`, and `EpicGraphRelocatedError`
   retries mint new IDs (`src/sase/bead/epic_from_plan.py`,
   `src/sase/bead/cli_work_from_plan_launch.py`). So record when the epic's survival is
   decided:
   - after `create_and_launch_epic_from_plan` returns;
   - in the `EpicFromPlanError(state_preserved=True)` branch;
   - idempotently on resume.

   This still happens before the launch command exits, so the monitor-member session
   barrier still seals the main path.

2. **The fallback reader uses bead-store attribution, not plan frontmatter.** Real
   planner artifacts show that `plan_path.json` points at the pre-archive copy
   (`~/.sase/plans/...`), while `bead_id:` lands in the archived plans-sidecar copy.
   That makes frontmatter lookups depend on archive-path mapping across sidecar clones.
   The epic bead's `created_by` (the plan's `proposed_by`) plus its `design` plan ref
   live in the synced bead store that bead waits already trust.
3. **Settlement is "launch in flight".** It uses the active epic-launch monitor or proc
   for the run, plus a short settle window. It does not parse settlement records.
4. **Un-overloading is limited to `epic_bead_id`** on worker rows. Real data confirms
   the bug: a phase worker's `epic_bead_id` was overwritten with its child epic. Other
   back-filled fields keep today's behavior.
5. **One tri-state "Follow epics" toggle row** in the wait modal instead of per-target
   chips. The modal's Agents field is free text, and the tri-state `mixed` value
   preserves per-target policies.
6. **The kill/dismiss path is a release path.**
   `_resolve_waiters_before_artifact_delete` memoizes a deleted target as resolved, so
   it must promote first. The two-stage dependency-then-time rewrite in
   `src/sase/axe/run_agent_wait.py` overwrites `waiting.json` from memory, so it must
   re-read the marker under the lock.
7. **The timeline tag is `↪EPIC`**, because tags are padded to 5 characters.
   **CHANGELOG.md is generated by release-please**, so the `flip` commit message carries
   the default change and nobody hand-edits the changelog.
8. **No feature flag.** Until `flip`, the keyword defaults to false. The `contract`
   phase lands `for_epic=` as an accepted but undocumented keyword, inert for one phase.
   Default behavior changes only in `flip`. `for_epic=false` is a permanent user choice,
   so it is a keyword, not a flag.
9. **Memory edits are out of scope.** `flip` records a `PROPOSED FOLLOW-UP` to update
   the `%wait` row in `sase/memory/macros.md`. It does not edit memory.

## Phase: record

Goal: when `sase bead work` materializes an epic, record that epic durably, under a
lock, on the run that created it.

- **Locked meta updater.** Add a shared helper, e.g.
  `src/sase/core/agent_meta_update.py: update_agent_meta_locked(artifacts_dir, mutate)`.
  - Build it from the private `_locked_agent_meta` in
    `src/sase/core/agent_output_variables.py` (an flock on `.agent_meta.json.lock`) plus
    `write_agent_meta_atomic` (`src/sase/axe/agent_meta.py`).
  - Have it call `update_agent_artifact_index_for_marker_mutation`.
  - Refactor `set_agent_output_variables` onto it.
  - Register every new writer in `tests/test_agent_artifact_marker_mutation_audit.py`.
- **Creator dir.** `sase bead work <plan-file>` records on `--artifacts-dir` when it is
  given (`via: host_launch`). Otherwise it records on `$SASE_ARTIFACTS_DIR` when that
  names a directory containing `agent_meta.json` (`via: agent_command`).
  - The env fallback is used **only** for recording. Do not change `finish_epic_launch`
    semantics or its notifications, which key off `--artifacts-dir`.
  - Thread a `creator_artifacts_dir` parameter from `bead/cli_work_entry.py`
    (`_handle_bead_work_locked`) through `work_from_plan_file` →
    `_work_from_plan_file_locked` → `work_from_plan_file_locked`.
- **Write points** (see [deviation 1](#deviations-from-the-research)). Call
  `record_created_epic(creator_dir, entry)`, which dedupes by `bead_id`, at these
  points:
  - after `create_and_launch_epic_from_plan` returns;
  - in the `state_preserved=True` error branch;
  - in the resume path (`cli_work_from_plan_resume.py`), when the epic is known.

  Never record an ID that was rolled back or relocated. A failed record write is logged
  and never fails the launch.

- **Un-overload.** Move `_update_epic_launch_metadata` (`src/sase/bead/epic_launch.py`)
  onto the locked updater. On worker rows (`phase_bead_id` or `epic_plan_ref` present),
  stop overwriting `epic_bead_id`. Keep today's behavior for every other field and row.
  Audit the `epic_bead_id` readers that mean "the epic this run launched" rather than
  "the epic I belong to" (`ace/tui/models/agent_associated_plan.py` and
  `_agent_associated_plan_phase.py`). Switch those readers to the new reader, so a phase
  worker that delegated still shows its child epic.
- **Readers.** Add `src/sase/core/created_epics.py`:
  - `created_epic_ids_from_meta(meta)` returns `created_epics` IDs. Only when there are
    none, and only for non-worker rows, it falls back to the legacy `epic_bead_id`.
  - `attributed_epic_ids(project, *, creator_global_name, plan_ref | plan_basename)`
    returns epic-tier plan beads whose `created_by` equals the creator and whose
    `design` names the run's plan. Derive the plan from `epic_launch_argv.json`'s plan
    argument, else `plan_path.json`. Read through the same project store locator bead
    waits use (`src/sase/bead/store_locator.py`). It is meant for rare fallback use and
    must be cheap to memoize per pass.
- **Wires.** Add `created_epics` to `AgentMetaWire` in Rust and Python, with a
  `CreatedEpicWire` struct and dataclass.
- **Tests:**
  - the record is written at the survival points, and not on rollback or relocation;
  - the env fallback is used for agent-run `sase bead work`;
  - the lock is honored under concurrent writers;
  - a worker's `epic_bead_id` is preserved, and its child epic is recorded on the
    worker;
  - the reader priority and legacy rules hold;
  - attribution matches and rejects correctly;
  - the scan wire round-trips the field.

## Phase: links

Goal: link every recorded epic to its creator in the artifact-link graph, derived from
the record and never on the wait's critical path.

- **Publish.** Publish a portable `created_epic_ids: list[str]`. Add it to
  `V2_METADATA_FIELDS` (`src/sase/agents_sync/v2_validation.py`) and to the
  `portable_metadata` shaping in `inventory_io.py`, following the `wait_for_beads`
  precedent in commit `8d97ef7661`.
- **Projection rules.**
  - `artifact_links/projection/_agent_created_epic.py` (rule id `agent-created-epic`)
    emits `bead:<id> produced-by agent:<global>` from published metadata.
  - A second rule emits the same edge from bead-store attribution: epic-tier plan beads
    whose `created_by` is an agent global name. It needs a best-effort, optional bead
    store root on `ProjectionInputs`. This gives unpublished planners the edge too.
  - Register both rules in `_entry.py`. Each rule stays best-effort.
- **Relation guidance.** Add `bead` to the `produced-by` source kinds in
  `sase-core: crates/sase_core/src/artifact_link/relation.rs` (guidance only) and update
  its tests. Update the matching Python vocabulary in
  `src/sase/sdd/_artifact_link_store_support.py` if it lists kinds. Document the edge in
  `docs/artifact_links.md`.
- **Tests:**
  - published `created_epic_ids` projects the edge;
  - the bead-store rule covers an unpublished planner;
  - a worker gets `implements` but no `produced-by` for its parent epic;
  - an older reader is unaffected.

## Phase: contract

Goal: the keyword parses, validates, persists, and round-trips everywhere, with the
default still **false**.

- **Python parser.**
  - In `src/sase/macro/_directive_collect.py`, add `for_epic` to `%wait`'s
    `supported_keys` and update the unsupported-keyword message. Record each
    occurrence's raw value together with its agent targets (positional and `agent=`).
  - In `_directive_extract.py`, after `resolve_wait_agent_args`, validate per the
    [error table](#grammar) using the exact messages. Compute
    `PromptDirectives.wait_for_epics_of` from explicit values, else
    `WAIT_FOR_EPIC_DEFAULT`. Define that constant as `False` in one module that `flip`
    will change; `--plan` targets are never armed.
- **Launch persistence.**
  - Carry the list through `src/sase/axe/run_agent_directives_extract.py`
    (`AgentMetadataInputs`), `run_agent_directive_metadata.py` (`build_agent_meta`), and
    the `waiting_data` built in `run_agent_wait.py`. Write it only when non-empty.
  - Implicit targets added by `resolve_wait_state` (fork sources, batch predecessors,
    the session parent) are never armed.
  - Assert `wait_for_epics_of ⊆ waiting_for` after normalization.
- **Rewrites.**
  - `PromptWaitDirective.epic_follow_agents` (`src/sase/macro/_directive_edit_wait.py`)
    drives `_format_wait_directive`. Agents that match the default render in the main
    `%wait(...)`; the others render as a separate `%wait(<agents>, for_epic=<value>)`.
  - `src/sase/ace/tui/actions/agents/_directive_persistence.py`
    (`waiting_marker_patch_for_token`, `wait_meta_patch_for_token`,
    `_write_waiting_marker`) replaces `wait_for_epics_of` together with the other
    condition keys.
  - Until `surfaces` adds the toggle, `_wait_actions._apply_wait` preserves the current
    policy for kept agents and gives new agents the default, so edits never erase it.
- **Rust editor.**
  - Add a `for_epic` `DirectiveKeywordSpec` to `WAIT_KEYWORDS`
    (`sase-core: crates/sase_core/src/editor/directive/metadata.rs`) with
    `DirectiveValueRole::Bool`. Give it dedicated suggestions: `true` "Also wait for any
    epic these agents launch" and `false` "Release when these agents finish, even if
    they launched an epic". Mark `false` "(default)" for now.
  - Update the `%wait` argument hint.
  - Add a `wait_directive_diagnostics` pass to `editor/diagnostics.rs` and chain it into
    `analyze_document_with_snapshot`. It emits the four Error codes with the identical
    messages.
  - Update every test that enumerates wait keywords: `sase_core_py` `surfaces.rs`;
    `editor/directive/tests.rs`; `editor/completion/tests/assist_candidates.rs`;
    `sase_macro_lsp` server completion tests and `jsonrpc_stdio.rs`.
  - On the Python side, update `tests/test_macro_directive_contract.py` and
    `tests/ace/tui/widgets/test_wait_directive_completion_interactions.py`.
- **Typed launch (beta `typed_launch_units`).** `parse_wait_directive` in
  `sase-core: agent_launch/typed_units.rs` accepts and validates `for_epic` with the
  same codes, and carries the policy so typed launches persist the same
  `wait_for_epics_of`. Verify how typed launches reach agent metadata.
- **Wires.** Add `wait_for_epics_of` to `AgentMetaWire` and `WaitingMarkerWire` in Rust
  and Python.
- **Tests** (in `tests/test_directives_wait.py` or a sibling
  `test_directives_wait_for_epic.py`):
  - every error case, with exact text;
  - per-occurrence scope;
  - an explicit value overriding the default across occurrences;
  - the colon form getting the default;
  - `--plan` handling;
  - the round-trip through `_format_wait_directive` and directive persistence;
  - the markers carrying the field.

  Rust tests assert the same literal messages.

## Phase: reducer

Goal: one small, pure, well-tested decision function in sase-core, plus the Python
collector that feeds it. It is not wired into any release path yet.

- **Core.** Add a new top-level module
  `sase-core: crates/sase_core/src/wait_epic_follow.rs`, following the
  `agent_hold_deadlock.rs` precedent: serde wires with `deny_unknown_fields` plus
  defaults, and a pure function with tests beside it.
  - **Input** `WaitEpicFollowInputWire`:
    `{waiter_own_bead_ids, now, launching_grace_seconds (600), launch_settle_seconds (120), targets: [WaitEpicFollowTargetFactsWire]}`.
  - **Target facts** `WaitEpicFollowTargetFactsWire`:
    `{target, agent_resolved, previous_state, previous_since, cycle_epic_ids, members: [WaitEpicFollowMemberFactsWire]}`.
  - **Member facts** `WaitEpicFollowMemberFactsWire`:
    `{name, artifact_dir, recorded_epic_ids, attributed_epic_ids, legacy_epic_bead_id, is_epic_worker, launch_reserved, launch_argv_present, launch_in_flight, launch_reserved_age_seconds, member_dismissed, resume_command}`.
  - **Output**, one `WaitEpicFollowDecisionWire` per target:
    `{target, state, epic_ids, members, since, reason, detail, resume_command, skipped_epic_ids, launching_overdue}`.
- **Rules, in order:**
  1. If `!agent_resolved`, the state is `agent`.
  2. E is the deduplicated union, over all members, of recorded and attributed epics. If
     it is empty, use the union of `legacy_epic_bead_id` over non-worker members.
  3. **Deadlock guard.** Drop any epic `e` where some own bead equals `e` or starts with
     `e.`, and record it in `skipped_epic_ids`.
  4. If E is non-empty and some `e` is in `cycle_epic_ids`, the state is `blocked`
     (`cycle`). Otherwise it is `following`, with the contributing member names.
  5. If E is empty and some member has `launch_reserved` and either `launch_in_flight`
     or an age below the settle window, the state is `launching`. `launching_overdue` is
     set once `since` is older than the grace period.
  6. If E is empty and a reserved member was dismissed, the state is `blocked`
     (`target_dismissed_during_launch`).
  7. If E is empty and some member is reserved, the state is `blocked`, with reason
     `launch_skipped` if no argv is present and `launch_ended_without_epic` otherwise,
     and its `resume_command`.
  8. Otherwise the state is `none`. When the guard skipped every epic, include
     `skipped_epic_ids` as a diagnostic.

  `since` is kept when the state equals `previous_state`; otherwise it becomes `now`.

- **Binding.** Expose `wait_epic_follow_reduce` from a new binding domain in
  `sase-core: crates/sase_core_py/src/`, following the sase-core AGENTS.md recipe. Add a
  binding round-trip test, and do not add prelude aliases. Move the pin.
- **Python collector.** Add `src/sase/core/wait_dependency_resolution/_epic_follow.py`.
  - **Signature.**
    `collect_epic_follow_facts(index, *, armed_targets, resolved_deps, previous_follows, waiter_meta, now, dismissed_artifact_dir=None)`.
  - **Calling the reducer.** It builds the input and calls the binding through
    `require_rust_binding`, with typed Python dataclasses on both sides.
  - **Facts from the index.** Members come from the index's member enumeration for the
    resolved entity (`dependency_member_dirs` in `_index_queries.py`). Recorded epics,
    worker flags, legacy IDs, and the outcome come from `ArtifactCandidate`. Extend the
    candidate from the scan wire, so the hot path does no extra I/O.
  - **Facts from files.** Only members that would otherwise be LAUNCHING or BLOCKED cost
    file probes:
    - `epic_launch_argv.json` presence and age;
    - whether the launch is in flight, by reusing the active-launch lookups in
      `src/sase/bead/epic_launch.py` and `epic_launch_handoff_io.py`;
    - `attributed_epic_ids`, memoized per pass.
  - **Reservations.** A reservation is outcome `epic_approved`, an `EPIC APPROVED`
    status, or `epic_launch_argv.json`.
  - **Resume command.** Built with
    `build_epic_launch_argv(plan, artifacts_dir=<member dir>)`, so a manual resume
    records on the planner.
  - **Pinned targets.** Already-FOLLOWING (pinned) targets are skipped.
- **Tests:**
  - Rust: one test per rule and edge case, including several epics, a clan with several
    members, the guard and its prefix semantics, `since` preservation, and the settle
    and grace windows.
  - Python: collector tests over fixture artifact trees, covering the main monitor path,
    the proc fallback, skip mode, a lost record write healed by attribution, legacy
    planners, and phase workers.

## Phase: release

Goal: every release path agrees by construction, and an armed target is never released
past an epic it launched. Size `large`: plan against the live code before implementing.

- **Shared decision.** Add
  `resolve_wait_release(index, marker, *, waiter_dir, closed_bead_ids, now, dismissed_artifact_dir=None) -> WaitReleaseDecision`,
  next to the resolver in `src/sase/core/wait_dependency_resolution/`.
  - It returns the existing `WaitDependencyStatus`, per-target follow decisions, an
    optional promotion patch, an optional stage patch, and `releasable`.
  - `releasable` requires the existing status to be resolved, no armed target in
    `agent`, `launching`, or `blocked`, and no pending promotion.
  - Armed targets are `wait_for_epics_of ∩ waiting_for`. A marker without the field
    behaves exactly as today.
  - Before a promotion, re-confirm that target's membership with the same fresh-index
    confirmation the release path uses (`_confirmation.py`). This stops the follow from
    missing a late session successor.
- **Lock.** Move `_agent_directive_lock`, and the runner-slot marker lock its writer
  takes, out of the TUI module into a core module. The chop and runner must not import
  TUI actions.
  - Add `apply_wait_epic_follow_patch(waiter_dir, expected, patch)`. It writes
    `waiting.json` and mirrors the follow fields into `agent_meta.json` under that lock,
    with compare-and-set.
  - Register it in the marker-mutation audit test.
- **Callers, all through `resolve_wait_release`:**
  - **Runner initial check** (`run_agent_wait.py` and
    `run_agent_wait_deps.initial_dependencies_resolved`): a promotion is folded into the
    first `waiting.json` and the runner parks.
  - **Parked-runner 60 s fallback** (`waiting_marker_dependencies_resolved`): it applies
    patches and never releases on a promotion pass.
  - **AXE chop** (`scripts/_chop_wait_checks_run.py` `_process_one_waiter`): it applies
    patches, persists stage changes (LAUNCHING, BLOCKED, and their `since`), and writes
    `ready.json` only when `releasable`.
  - **Kill/dismiss** (`ace/tui/actions/agents/_killing_utils.py`
    `_resolve_waiters_before_artifact_delete`): for an armed target, it promotes using
    the deleted run's facts before any name memoization. If the target was LAUNCHING, it
    records BLOCKED `target_dismissed_during_launch` and never memoizes the target as
    resolved.
  - **Run-now** keeps releasing immediately, as today.
- **Two-stage rewrite.** The dependency-then-time rewrite in `run_agent_wait.py`
  re-reads `waiting.json` under the lock and changes only `wait_until`, which preserves
  follows, derived beads, and pins.
- **Wires.** Add `wait_epic_follows` to `WaitingMarkerWire` and `AgentMetaWire` in Rust
  and Python.
- **Docs.** In `docs/macros.md`'s wait section and completion matrix, document
  `for_epic`: its grammar, errors, and semantics, with the default still false here. Add
  a note next to the `@epic` tribe so the two are never confused. In `docs/axe.md`,
  rewrite the "epic-approved planner" sentence.
- **Tests:**
  - one per edge-case row from the research, over identical snapshots, showing that the
    initial check, the fallback, the chop, and kill/dismiss all agree;
  - a skip-mode or proc-fallback LAUNCHING target does not release;
  - a re-run does not redirect a pinned follow;
  - a stale promotion aborts on compare-and-set after a concurrent `w` edit;
  - the two-stage rewrite keeps follows;
  - markers that predate the feature never follow;
  - a canceled or superseded followed epic releases;
  - an epic in another project routes through full-ID bead routing;
  - `awaits` appears after promotion
    (`tests/artifact_links/test_agent_wait_bead_projection.py`).

  Extend `tests/test_run_agent_wait_deps_initial.py`,
  `tests/test_run_agent_wait_fallback.py`,
  `tests/test_wait_dependency_release_confirmation.py`,
  `tests/test_axe_chop_wait_checks*.py`, and
  `tests/test_kill_named_agent_dismiss_waiting.py`.

## Phase: model

Goal: the follow state is cheap to read everywhere and never triggers I/O in a render
path.

- **Agent model.** Add `wait_for_epics_of` and `wait_epic_follows` to the TUI Agent
  model (`ace/tui/models/_agent_state_queue.py`) and its loaders: the wire twin
  `_meta_enrichment_wire.py` and the filesystem twin `_meta_enrichment_filesystem.py`,
  with the `waiting.json` override taking precedence.
- **Render cache key.** Add both fields to `agent_render_key`
  (`widgets/_agent_list_render_cache.py`).
- **Follow-aware status** in `ace/tui/_agent_completion_wait.py`:
  - **Counts.** In `wait_dependency_status_counts`, a FOLLOWING target leaves the agent
    counts. Its epics feed a separate follow segment that uses cached bead statuses, and
    `added_bead_ids` leave the authored bead counts.
  - **Satisfied.** `wait_dependencies_satisfied` is false while any armed target is
    LAUNCHING or BLOCKED. A resolved armed target with no persisted stage displays as
    today, so a target that launched no epic causes no flicker.
- **Shared view.** Add `src/sase/core/wait_epic_follow_view.py` with an `EpicFollowView`
  per target: state, epics with cached statuses, members, since, reason, detail, and
  resume command. Add plain-text phrasing beside it:
  - "waits on planner's epic sase-7k"
  - "waits on planner's epic launch"
  - "blocked: planner's epic launch ended without an epic (resume: …)"

  The TUI, CLI, and Telegram all use these strings.

- **Authored beads.** Add an `authored_wait_beads(marker_or_agent)` helper:
  `wait_for_beads` minus `added_bead_ids`. A bead the user also authored stays authored.
- **Tests:**
  - the loaders read both sources;
  - the cache key changes on a stage change;
  - counts and satisfied logic cover every state;
  - the view and phrasing are correct;
  - a guard (patch or spy) proves no I/O happens in these functions.

## Phase: safety

Goal: something that is wrong is loud and actionable. Something that is merely in
progress stays quiet.

- **Notifications.** Extend `scripts/_chop_wait_checks_terminal.py`, following
  `upsert_terminal_blocked_wait_notification` (sender `wait_checks`, an upsert with a
  dedup key per waiter and target), so each case produces exactly one entry:
  - **LAUNCHING overdue.** "reviewer is waiting on planner's epic launch (approved
    14:02; no epic after 10m)".
  - **BLOCKED.** It names the reason and includes the resume command. Its action is
    `JumpToAgent` to the waiter.
  - **A followed epic that will never close.** While a target is FOLLOWING, the land
    agent of the followed epic (clan `<epic>`) fails terminally. The entry names the
    waiter, the epic, and the land agent.
  - **Resolved.** Clear or settle the entry when its state resolves, the way existing
    terminal-blocked entries do.
- **Cycle facts.** Add a collector helper, e.g. `_epic_follow_cycle.py`, that fills
  `cycle_epic_ids`. Its best-effort check: an epic is a cycle when any agent in epic
  `E`'s clan already waits on the waiter (`waiting_for`/`wait_for` contains the waiter's
  name or session), for example through the approval "Wait for" field. Never release on
  a timeout.
- **Docs.** In `docs/axe.md`, document the states, the grace and settle windows, the
  notifications, and the deadlock and cycle guards.
- **Tests.** One per notification and its dedup and clear behavior, the cycle detection,
  and the no-timeout-release guarantee.

## Phase: tui

Goal: the hand-off is obvious, calm, and beautiful. Read the `tui` and `tui_perf` memory
notes first.

- **Row** (`ace/tui/wait_status_presentation.py`,
  `widgets/_agent_list_render_agent_status.py`): put the new tokens after the existing
  count tokens and before the `!` and time annotations.

  ```text
  WAITING ▶1              planner running (default follow adds no row noise)
  WAITING ↪ epic…         launch in flight (`epic…` dim teal)
  WAITING ↪ ◐ sase-7k     following one epic
  WAITING ↪ ◐2            following two epics
  WAITING ▶1 ↪ ◐1         one target running, one following
  WAITING ↪ !             blocked (red `!`, WAIT_UNRESOLVABLE style)
  ```

- **Lane** (`widgets/prompt_panel/_agent_wait_section.py`, `[agents]`): copy the tribe
  binding arrow pattern.

  ```text
  Wait: [agents] planner ▶ ↪                       armed, still running (dim teal ↪)
  Wait: [agents] planner ✓ ↪ epic launching… · since 14:02
  Wait: [agents] planner ✓ ↪ sase-7k ◐ in progress · 2/5 phases · since 14:32
  Wait: [agents] planner ✓ ↪ sase-7k ◐ · sase-7m ✓ · since 14:32
  Wait: [agents] planner ✓ ↪ ! epic launch ended without an epic · resume: sase bead work …
  Wait: [agents] research ✓ · agent only            explicit opt-out, shown only while the default is on
  ```

  Followed epics are not repeated in `[beads]`. A canceled or superseded epic shows its
  resolution in the lane. The epic ID jumps to the bead and the target name jumps to the
  agent, through the existing affordances. A cold cache shows a neutral pending token
  and never claims the bead is missing.

- **Phase progress** (`2/5 phases`): extend the off-thread bead warmup
  (`actions/agents/_loading_bead_warmup.py`) to fetch closed/total phase counts for
  followed epic IDs only, using the existing epic-children binding. Render the count
  dim.
- **Toast.** When an agent reload applies, compare each waiter's previous and new
  FOLLOWING sets, off the render path. Follow the `_remote_attention.py` precedent and
  coalesce per epic:
  - "reviewer now waits on epic sase-7k (launched by planner)"
  - "3 agents now wait on epic sase-7k (launched by planner)"

  Never toast on startup loads.

- **Timeline.** Add `↪EPIC | <since> | sase-7k ← planner` to `Agent.timestamps_display`
  (`models/agent.py`), sourced from the mirrored `wait_epic_follows`.
- **Help and docs.**
  - Add the `↪` entries to the "Waiting Badges" legend
    (`modals/help_modal/agents_bindings.py`).
  - Update the wait badges and lanes sections of `docs/ace.md`.
  - Keep the docs-sync assertion in
    `tests/ace/tui/test_agent_wait_dependency_status_counts.py` green.
- **Perf.**
  - Renderers read only the persisted stage and cached bead statuses.
  - A stage change patches only its row through `patch_row()`.
- **Tests:**
  - unit tests for every row and lane state, at narrow width, with a cold cache, several
    epics, and the blocked state;
  - toast coalescing and the no-startup-toast rule;
  - the timeline milestone.

  Add PNG goldens in `tests/ace/tui/visual/test_ace_png_snapshots_agents_waiting.py` for
  the launching, following, multi-epic, and blocked rows and lanes. Run
  `just fix-tui-screenshots` with selectors, through `/sase_monitor` if it is long, and
  inspect the report and every new golden.

## Phase: surfaces

Goal: every other surface tells the same story and edits never corrupt it.

- **Wait modal** (`ace/tui/modals/wait_modal*.py`, `actions/agents/_wait_actions.py`).
  - **Toggle row.** Add a focusable **Follow epics** toggle row directly under Agents.
    It joins the `Ctrl+J`/`Ctrl+K` field cycle, and `Space` toggles it. It shows `on`
    (teal `↪`), `off` (dim), or `mixed` (each target keeps its policy; new agents get
    the default). When every target is a `--plan` row, the row is disabled and shows the
    reason.
  - **Prefill and apply.** Prefill Beads with `authored_wait_beads` only. Applying the
    modal sets `PromptWaitDirective.epic_follow_agents`.
  - **Removing a target, or turning its follow off,** drops its follow entry and its
    `added_bead_ids` from both files under the lock. The pin stays. Turning follow back
    on lets the next evaluation re-promote.
  - **Keymap.** The modal's bindings are code-local, so `src/sase/default_config.yml`
    needs no entry unless you add a configurable key.
  - **Docs.** Update `docs/ace.md` § Wait Modal.
- **Kill-and-relaunch rewrites** (`_wait_actions.py`, `_wait_helpers.prompt_wait_spec`)
  emit correct split `%wait` occurrences and never emit derived beads.
- **CLI.**
  - `sase agent list -j` (`agents/cli_list.py`, `integrations/_agent_list_entry_*`)
    exports `wait_for_epics_of` and `epic_follows`.
  - `sase agent wait` rows (`agents/_wait_live_rows.py` `_why_column`, `_wait_json.py`)
    use the shared phrasing.
  - Update the CLI docs that list these fields.
- **Jinja** (`src/sase/agent/output_variable_context.py`). For each waited target,
  synthesize:
  - `agents["planner"].created_epic`, the first epic, or empty;
  - `agents["planner"].created_epics`, every epic;

  The values come from the waiter's FOLLOWING entry, else from the target's
  `created_epic_ids_from_meta`, so `for_epic=false` users also get the ID. Avoid the
  name `epic_bead_id`. Document both in `docs/macros.md`.

- **Telegram.** Open `sase-telegram` with `sase repo open`. If its agent status text
  renders wait targets, add the shared phrasing there; it must never be a push message.
  If it does not render waits today, make no change.
- **Tests:**
  - the modal tri-state, prefill, apply, and removal flows
    (`tests/ace/tui/test_wait_modal*.py`);
  - relaunch rewrites (`test_agent_wait_resume_*`);
  - the list JSON and wait rows (`tests/test_agent_wait_live.py`);
  - the Jinja variables, which exist only after release.

## Phase: flip

Goal: the user's requested default ships, behind regression proof.

- **Default.** Set `WAIT_FOR_EPIC_DEFAULT = True` for user-authored agent targets, never
  for `--plan` rows. Move the editor's "(default)" suggestion doc to `true` in sase-core
  and move the pin.
- **Generated waits** (`src/sase/bead/work_prompt.py`).
  - **Phase and land segments** become `%w(<agents>, for_epic=false)`. Use the
    parenthesized form, because the colon form takes no keywords. These segments already
    pair with phase bead waits.
  - **Approval "Wait for" extra waits** (`_extra_wait_lines`) keep the default, because
    the user authored them.
  - **`%repeat` chains, mobile and fork prefills, and bare-`%wait` rewrites** keep the
    default too.
- **Regression tests (gate the flip):**
  - an epic's phases and land agent still release under the default;
  - a phase worker that delegated to a child epic releases its dependents only through
    the phase-bead wait, with no deadlock;
  - `%repeat:k` run k follows run k−1's epic;
  - old markers never follow;
  - a waiter launched after the epic already closed releases on the next pass.
- **Docs.**
  - `docs/macros.md`: the default, the opt-out spelling, and the completion matrix.
  - `docs/beads.md`: the generated `for_epic=false`.
  - `docs/axe.md`: the default.
- **Commit message.** Use a `feat(wait):` conventional commit message that names the
  default change. Do not hand-edit `CHANGELOG.md`.
- **Parked runners.** Before flipping, note that following lengthens parked-runner
  lifetimes. If an open host-resource or parked-runner diet epic exists, record a
  `PROPOSED FOLLOW-UP` instead of blocking.
- **Memory follow-up.** Record a `PROPOSED FOLLOW-UP` to update the `%wait` row in
  `sase/memory/macros.md` with the `for_epic` default. Do not edit memory in this epic.

## Verification (every phase)

- Run `sase tool run check` in sase. In sase-core, when it was touched, run
  `sase tool run check` from the sase-core checkout. Never run `just check-full` unless
  a phase's own bead explicitly asks for it.
- Every phase that adds or changes a binding that sase calls moves
  `sase-core-revision.txt`.
- Prefer new modules over growing large files, because of the `toobig` gate. Keep every
  new public symbol used, because of `symvision`.

## Edge Cases (contract-level outcomes)

| Scenario                                              | Outcome                                                             |
| ----------------------------------------------------- | ------------------------------------------------------------------- |
| Target ends with no plan, a tale, or `plan_committed` | NONE: released exactly as today                                     |
| A tale coder later proposes an epic                   | That member's epic is followed                                      |
| Plan in review                                        | AGENT: the session is unresolved                                    |
| Plan rejected                                         | Today's predicate: stays parked                                     |
| Epic approved, monitor path                           | AGENT until `EPIC CREATED`, then FOLLOWING                          |
| Proc fallback                                         | LAUNCHING, then FOLLOWING or BLOCKED                                |
| `skip` mode                                           | BLOCKED `launch_skipped` with a resume command that records         |
| Lost record write                                     | Healed by bead-store attribution                                    |
| Launch fails                                          | Today's terminal blocker for a failed monitor member, or BLOCKED    |
| Several epics (one run or several members)            | Wait for all                                                        |
| Nested child epics                                    | Covered: the top epic can't close before its children               |
| Epic closed `canceled` or `superseded`                | Released; the lane shows the resolution                             |
| Same name re-run after the follow                     | Ignored (pinned)                                                    |
| Phase or land worker as target                        | Inherited epic never followed; a child epic it launched is followed |
| Waiter inside the followed epic's subtree             | Skipped by the guard and recorded                                   |
| `--plan` target                                       | Never armed; explicit `true` is an error                            |
| Epic in another project                               | Full-ID bead routing                                                |
| Marker predates the feature                           | Never follows                                                       |
| Wait edited mid-check                                 | Compare-and-set aborts the stale promotion                          |
| Epic reopened after release                           | No re-park, matching bead waits                                     |
| Run-now                                               | Releases immediately                                                |

## Out of Scope

- `sase agent wait --for-epic/--no-for-epic` (research Q2): `flip` records it as a
  `PROPOSED FOLLOW-UP`.
- An `agent-only=` spelling in the approval "Wait for" grammar (research Q3): add it
  only if users ask.
