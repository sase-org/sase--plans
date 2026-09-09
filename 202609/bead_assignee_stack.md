---
tier: epic
title: Bead assignee history stack
goal: 'A bead records every agent ever assigned to it as an ordered stack derived from
  its event stream, host-directed launch claims push onto that stack instead of dying
  with "already in_progress and assigned to", and every surface (wire, DB mirror, CLI,
  TUI, mobile) renders the current holder plus the assignment trail.

  '
phases:
  - id: core-assignee-history
    title: Rust core assignee stack derived from the event stream
    depends_on: []
    size: medium
    description:
      "core-assignee-history: add a derived assignees stack to the core bead model —
      reducer + in-memory mutation twins push on every non-empty assignee assignment, no
      event-wire change, schema fixtures and parity tests updated per the
      notes-to-records precedent, additive binding/gateway exposure, and the
      SQLite-mirror migration SQL."
  - id: adopt-core-floor
    title: Adopt the released core binding floor
    depends_on:
      - core-assignee-history
    size: small
    description:
      "adopt-core-floor: ratchet the pinned sase-core revision to the landed stack
      change, wait for the release-plz publish, and move the sase-core-rs version window
      in pyproject to the released floor."
  - id: python-claim-takeover
    title: Python stack adoption and takeover-instead-of-failure claims
    depends_on:
      - adopt-core-floor
    size: medium
    description:
      "python-claim-takeover: thread assignees through the Python Issue model, wire
      decode, and SQLite mirror; replace the launch-claim RuntimeError with a logged
      takeover push for current-owner holders (foreign holders still refuse); align
      relaunch cleanup and preclaim rollback; add the incident regression test."
  - id: surfaces-and-docs
    title: Render the assignment trail on CLI, TUI, and docs surfaces
    depends_on:
      - python-claim-takeover
    size: small
    description:
      "surfaces-and-docs: show the assignment trail in bead detail (CLI text + JSON),
      bead pages, the mobile helper, and the ACE beads detail pane; refresh goldens,
      docs, and the beads memory note."
proposed_by: bbugyi200.athena.04r
bead_id: sase-y4
create_time: 2026-09-09 19:52:15
status: wip
---

- **PROMPT:**
  [prompts/202609/bead_assignee_stack.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/bead_assignee_stack.md)
- **BEAD:**
  [sase-y4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-y4/README.md)

# Bead Assignee History Stack

## Problem

An epic phase agent died before its model ever ran:

```
RuntimeError: Failed to claim bead 'sase-y1.1' for agent 'sase-y1.1': bead 'sase-y1.1'
is already in_progress and assigned to 'sase-y0.1'
```

The launch-claim promotion (`_promote_bead_claim`,
`src/sase/axe/run_agent_runner_launch.py:245`) runs before `run_execution_loop`, so this
failure wastes a prepared workspace and kills the launch outright. A bead's single
scalar `assignee` cannot express "a different agent legitimately takes this bead over",
and nothing records who held the bead before. The fix this epic implements: the assignee
becomes an ordered stack of every agent assigned to the bead (most recent on top),
host-directed launch claims push onto it instead of refusing, and the trail is visible
everywhere a bead is rendered.

## Root Cause (diagnosed, with evidence)

Three cooperating facts produced the failure, verified against the live bead store and
event streams:

1. **The refusal is Python-only.** The Rust core mutation `claim_for_agent_launch`
   (`crates/sase_core/src/bead/mutation.rs:1054` in the sase-core repo) never refuses an
   in-progress bead held by another agent — it overwrites `assignee` (line 1094). The
   fatal guard is `issue_is_in_progress_for_another_agent` at
   `src/sase/axe/run_agent_runner_bead.py:88-96`, and the error string exists only
   there.

2. **The stale holder name came from bead-id relocation, not from a real rival agent.**
   The epic's event stream records
   `"operation":"epic_work_preclaimed","issue_id":"sase-y1.1","payload":{"agent_name":"sase-y0.1"}`:
   two clones minted the same bead id concurrently, the sync conflict resolver relocated
   one side's whole stream (`sase-y0` → `sase-y1`,
   `src/sase/bead/conflict_resolver.py:156-199`, `src/sase/bead/relocation.py`),
   rewriting bead ids inside events — but `assignee` and the preclaim payload's
   `agent_name` are opaque strings and were not rewritten. Since epic worker names are
   deterministic (phase agent name == phase bead id, land agent == `<epic>.land`;
   `src/sase/bead/work.py:213-215`, `src/sase/bead/cli_work_plan.py:33-40`), the
   relaunch's name `sase-y1.1` could never match the stale `sase-y0.1`.

3. **Force-reuse is same-name-only, so no takeover path exists.**
   `issue_retains_force_reuse_owner` (`src/sase/bead/force_reuse.py:21-33`) requires
   `prior_owner ≡ agent_name`; it authorizes one name reclaiming its own bead and
   nothing else. Legitimate differently-named holders also arise from in-process
   successors (`/sase_pipe` and plan-chain hand-offs run inside the same runner and
   never re-claim, so the bead keeps the predecessor's name while a successor works it)
   and from retry descendants (`foo.r0`, `foo.r1`), which needed their own special case
   (`_resolve_assignee_conflict`, `src/sase/bead/cli_work_cleanup_targets.py:506-546`).

A second, smaller defect observed in the same incident: after the launch failed, the
rollback restored only `status` (the bead went back to `open` while still displaying
`Assignee: sase-y0.1`), even though `rollback_work_launch`
(`src/sase/bead/cli_work_cleanup.py:119-123`) intends to restore both.

## Design

### The stack is a derived projection, not a new event kind

The core insight that keeps this change safe: bead event streams already record every
assignee assignment as scalar field values (`BeadIssueUpdateEventFieldsWire.assignee`,
plus the `EpicWorkPreclaimed` payload), so the full assignment history of every existing
bead is already durably stored. The stack is therefore **derived by the event reducer**,
not written by a new event operation:

- `IssueWire` (and the Python `Issue`) gains `assignees: Vec<String>` — chronological,
  last element = most recent assignee, entries non-empty, no adjacent duplicates.
- The reducer and every in-memory mutation twin apply one shared rule: assigning a
  non-empty name pushes it when it differs from the current top; assigning the empty
  string (release, +1 reopen of a closed task) clears only the scalar and leaves the
  stack intact as history.
- The scalar `assignee` keeps its exact current semantics everywhere: it remains "the
  current holder when status is claimed/in_progress, else stale-or-empty text". Every
  existing `(status, assignee)` behavior, guard, and test tuple stays valid, and the
  ~30-signature `bead_assignees: dict[str, str]` threading through the cleanup stack
  keeps its scalar value type.

Consequences that make this reliable and robust:

- **No event-wire change at all.** New writers emit exactly today's events; old readers
  (other machines on older sase versions reading synced streams) keep working, and new
  readers derive full stacks retroactively from old streams. No
  `BEAD_EVENT_SCHEMA_VERSION` bump, no new `BeadEventOperationWire` kinds (which old
  readers would hard-reject).
- **The stack is never directly settable.** No CLI flag, update field, or API input
  writes `assignees`; it is only ever derived from assignment events, so it cannot drift
  from history.
- **Determinism under sync.** Claim mutations already run under the bead mutation lock,
  and cross-clone merges reduce event streams deterministically, so the derived stack is
  identical on every clone that has the same events.

### Claims: push and take over instead of dying

- `claim_for_agent_launch` keeps its Rust mechanics (set scalar → reducer/twin pushes).
  The Python guard at `src/sase/axe/run_agent_runner_bead.py:88-96` changes from a
  RuntimeError to a **takeover**: when the current holder normalizes to a different
  current-owner agent, print a clearly visible takeover line naming the prior holder and
  proceed; the push records the hand-off durably. This fixes relocation staleness,
  successor hand-offs, and epic relaunches in one move, and it matches the existing
  recovery contract ("rerun `sase bead work` to reassign every non-closed bead").
- **Foreign holders still refuse.** When `foreign_agent_owner_root` recognizes the
  holder as another user/machine's agent, the launch claim keeps failing (now with the
  assignment trail in the message). Host-directed takeover authority is per-owner;
  cross-machine safety does not weaken.
- Wait claims (`claim_for_agent_wait`) and releases keep today's semantics: a wait claim
  still only reserves an `open` bead and declines politely otherwise; a release still
  clears the scalar holder — the stack simply retains the history.

### Compatibility and no feature flag

The stored-format change follows the established sase-core scalar→list evolution recipe
(the `notes: String → Vec<BeadNoteWire>` change, commit `bda9efc` there): tolerant
deserialization on the raw wire struct, `skip_serializing_if` so untouched rows
serialize byte-identically, updated `current_schema.jsonl` plus a new
`pre_assignee_stack_schema.jsonl` legacy fixture, and no schema-version bump. No feature
flag is warranted: the behavior change is agent-launch machinery (not user-choosable
behavior), the old refusal branch has no reason to stay reachable, and wire/API changes
are strictly additive.

## Phase: core-assignee-history

- slug: core-assignee-history
- size: medium
- depends: []

All work in this phase happens in the linked `sase-core` repository (open it with the
`/sase_repo` skill). Model it closely on that repo's commit `bda9efc` ("store notes as
structured records"), which is the exact scalar→list precedent. Paths below are relative
to the sase-core repo root.

Steps:

1. **Wire model.** In `crates/sase_core/src/bead/wire.rs`, add `assignees: Vec<String>`
   to `IssueWire` (field `assignee` is at line 763) with
   `#[serde(default, skip_serializing_if = "Vec::is_empty")]`, mirrored on the private
   `IssueWireRaw` deserialize target (line 876) and copied across in the hand-written
   `impl Deserialize` (line 938+). Compat shim in that deserialize: when `assignees` is
   absent/empty and `assignee` is non-empty, derive `assignees = [assignee]` so
   jsonl-only legacy stores load with a consistent singleton stack. Extend
   `IssueWire::validate()` (line 1035): entries non-empty, no adjacent duplicates, and
   when `assignee` is non-empty the last entry equals it.
2. **One shared assignment helper.** Add a helper (suggested:
   `apply_assignee_assignment(issue: &mut IssueWire, value: &str)`) implementing the
   push rule (non-empty and different from top → push + set scalar; non-empty and equal
   to top → set scalar only; empty → clear scalar, stack untouched). Use it in both
   projections — the reducer and the in-memory mutation twins — to preserve the
   dual-projection invariant documented at `mutation.rs:1095-1097`.
3. **Reducer.** In `crates/sase_core/src/bead/events.rs`: `apply_update_event_fields`
   (line 1787, assignee branch at 1801) routes through the helper; `EpicWorkPreclaimed`
   (line 1720) pushes; the `TaskPlusOneRecorded` closed-reopen clear (line 1758) clears
   the scalar only. `IssueCreated` embeds a whole `IssueWire` snapshot, so creation is
   covered once `create_issue` populates the stack.
4. **Mutation twins.** In `crates/sase_core/src/bead/mutation.rs` route every direct
   assignee write through the helper: `create_issue` (line 271, from
   `BeadCreateRequestWire.assignee`), `apply_update_fields` (line 2681),
   `claim_for_agent_launch` (line 1094), `claim_for_agent_wait` (line 1179),
   `release_agent_claim` (line 1236 — scalar clear, stack retained),
   `preclaim_epic_work_plan` (lines 1367 and 1393), and `add_task_plus_one` (line 435).
   Keep every existing guard and idempotency check on the scalar exactly as it is (lines
   1081, 1147, 1159, 1226). Do not add `assignees` to any create/update input type — the
   stack is never directly settable.
5. **Projection drift.** Check how `compute_projection_drift` / `changed_issue_fields`
   (`crates/sase_core/src/bead/read.rs:448/488`) treat the added field on existing
   stores: reduced-from-events stores will rewrite `issues.jsonl` on first save. Follow
   whatever the `is_ready_to_work` derived-field addition did so `bead doctor` does not
   flag every pre-existing store as drifted in the interim.
6. **Search and core CLI.** `crates/sase_core/src/bead/search.rs:329`: include prior
   assignees in the searchable text (join like `refs`/`close_history` at 274-326); the
   `assignee` field key keeps matching the scalar.
   `crates/sase_core/src/bead/cli.rs:216-224`: render the trail under the `Assignee:`
   line when the stack holds more than the current holder (e.g.
   `Previously: sase-y0.1`). `history.rs` is generic over issue JSON and needs no
   change.
7. **Mobile summary (additive).** Add `assignees: Vec<String>`
   (`skip_serializing_if = "Vec::is_empty"`) to `MobileBeadSummaryWire`
   (`crates/sase_core/src/host_bridge.rs:840-856`), declare it in the api_v1 contract
   snapshot (`crates/sase_gateway/src/contract.rs`, assignee is at line 667), regenerate
   `contracts/api_v1/mobile_api_v1.json` via `write_api_v1_contract_snapshot`, and
   update the gateway golden (`crates/sase_gateway/src/wire.rs:1616-1683`) and the LSP
   fixture (`crates/sase_xprompt_lsp/src/server.rs:3411`).
8. **SQLite-mirror migration SQL.** The Python repo's legacy SQLite mirror consumes
   migration SQL through PyO3 getters. Add the `assignees` column (JSON-encoded text,
   like notes) to `BEAD_SQLITE_SCHEMA` in `crates/sase_core/src/bead/schema.rs` plus a
   paired `bead_needs_<x>_migration` probe / `bead_<x>_migration_sql` getter following
   the existing migration pairs, and expose them through `sase_core_py`.
9. **Binding surface.** Bead payloads cross to Python as JSON-shaped dicts
   (`bead_result_to_py`, `crates/sase_core_py/src/lib.rs:7609`), so outbound `assignees`
   is automatically additive. Update the module docstring API reference (lib.rs:327-351)
   and the issue literal in the merge-streams binding test (lib.rs:20279).
10. **Fixtures and tests.** Update
    `crates/sase_core/tests/fixtures/bead/jsonl/current_schema.jsonl` and the inline
    literals (`tests/bead_storage_parity.rs:112/121`, `tests/bead_read_parity.rs:991`,
    `mutation.rs:8129/9092/9886`). Add `pre_assignee_stack_schema.jsonl` with a
    `legacy_jsonl_fixtures_get_python_defaults`-style assertion locking in that old
    single-string rows still read (deriving the singleton stack). Keep
    `event_roundtrip_schema.jsonl` and `events/event_roundtrip/streams/gold-1.jsonl` as
    legacy inputs and assert their replay now derives the expected stacks. Add reducer
    unit tests: push, top-dedupe, release retains history, preclaim push, +1-reopen
    clear-scalar-only, and validate() invariant violations.

Use a Conventional Commit `feat(bead): ...` subject (release-plz derives the version
from it; the change is additive, not breaking).

Verification: `just check` from the sase-core repo root — never
`cargo test -p sase_core` alone, since that skips the `sase_core_py` binding tests.

## Phase: adopt-core-floor

- slug: adopt-core-floor
- size: small
- depends: [core-assignee-history]

Adopt the landed core change in this repo, following the precedent of the
`ratchet_core_pin` plan and the earlier "adopt the released core binding floor" phases.

Steps:

1. Ratchet `sase-core-revision.txt` to the sase-core SHA containing the
   core-assignee-history phase using `tools/ratchet_core_revision` (verify with
   `--check`), so sase CI builds the wheel from that revision.
2. Run `tools/check_sase_core_rs_bindings` to confirm the new migration bindings
   resolve.
3. Wait for release-plz to publish the corresponding `sase-core-rs` release (use the
   `/sase_monitor` skill for the wait; if release automation is stalled, record that on
   the phase bead rather than blocking forever), then move the version window at
   `pyproject.toml:47` (currently `sase-core-rs>=0.32.34,<0.33.0`) to the released
   version's floor and matching ceiling, and refresh the lock.

Verification: `just install` (rebuilds the binding from the linked checkout), then
`just check`.

## Phase: python-claim-takeover

- slug: python-claim-takeover
- size: medium
- depends: [adopt-core-floor]

Thread the stack through the Python model and replace the fatal claim guard with
takeover semantics.

Steps:

1. **Model and decode.** `src/sase/bead/model.py:239`: add
   `assignees: tuple[str, ...] = ()` beside the scalar. `src/sase/core/bead_wire.py:83`
   (`issue_from_dict`, the single production decode point): decode `assignees` with the
   same singleton-derive fallback as the core (absent + non-empty scalar → singleton).
   `src/sase/bead/_project_types.py:24` (`EpicPreclaimRollback`) stays scalar — rollback
   restores the scalar; history is append-only by design.
2. **SQLite mirror and JSONL codec.** Persist `assignees` as a JSON-encoded column using
   the phase-1 migration SQL bindings: `src/sase/bead/_db_schema.py:21`,
   `src/sase/bead/_db_rows.py:34`, `src/sase/bead/db.py:48/64/140`,
   `src/sase/bead/_db_migrations.py`, with codec helpers in `src/sase/bead/_db_codec.py`
   mirroring the notes pattern. Mirror the byte format in `src/sase/bead/jsonl.py`
   (`_issue_to_dict` emits the field only when non-empty, matching the Rust
   `skip_serializing_if`; `_dict_to_issue` applies the singleton fallback). Confirm the
   synthetic rows at `src/sase/core/health.py:259` and
   `tools/smoke_sase_core_rs_bead_resolution:49` still parse.
3. **Takeover instead of RuntimeError.** In
   `src/sase/axe/run_agent_runner_bead.py:88-96`: when the bead is in progress for
   another agent, refuse only when `foreign_agent_owner_root` (from
   `sase.core.agent_identity_facade`) recognizes the holder as a foreign owner — and
   include the assignment trail in that error. Otherwise print a clearly visible
   takeover line (prior holder → this agent) and fall through to
   `claim_for_agent_launch`, which pushes. Keep the force-reuse retain branch
   (no-mutation) exactly as is.
4. **Helpers and messages.** Extend `src/sase/bead/force_reuse.py` with the takeover
   predicate ("held by a different current-owner agent") so the policy lives in one
   place. Harden `src/sase/bead/claims.py:177` (raw `==` on names) to use
   `same_current_owner_agent_name`, and include the trail in the decline message at
   `claims.py:196`.
5. **Relaunch cleanup alignment.** `_resolve_assignee_conflict`
   (`src/sase/bead/cli_work_cleanup_targets.py:506-546`): a stale assignee with no live
   conflicting shell no longer raises `ForcedReuseCleanupError` for current-owner
   holders — align with the takeover policy while preserving the live-retry-descendant
   `PRESERVE` behavior. Update `tests/test_bead/test_cli_work_cleanup_assignees.py`.
6. **Rollback fidelity.** Add a regression test for the observed status-only rollback:
   `rollback_work_launch` (`src/sase/bead/cli_work_cleanup.py:119-123`) must restore the
   captured assignee even when the captured value is the empty string (the incident's
   bead came back `open` yet still displayed the preclaimed holder).
7. **Incident regression test.** In `tests/test_run_agent_runner_bead.py` (the
   `already in_progress` expectations at lines 92-215 change): a bead preclaimed
   in-progress for `sase-y0.1` claimed by launch as `sase-y1.1` succeeds, the issue's
   `assignees` ends as `("sase-y0.1", "sase-y1.1")`, and the takeover line is printed; a
   foreign-owner holder still refuses. Also update the `(status, assignee)` tuple
   assertions in `tests/test_bead/test_claims_lifecycle.py` /
   `tests/test_bead/test_claims.py` only where the new field is asserted — scalar
   behavior is unchanged.

Verification: `just install`, then `just check`.

## Phase: surfaces-and-docs

- slug: surfaces-and-docs
- size: small
- depends: [python-claim-takeover]

Make the trail visible and document the semantics.

Steps:

1. **CLI detail.** `src/sase/bead/cli_detail_render.py:104-108`: under the `Assignee:`
   line, render the prior trail when the stack holds more than the current holder (e.g.
   `Previously: sase-y0.1`); keep the `Claimed by:` phrasing. Add `"assignees"` to
   `src/sase/bead/cli_detail_json.py:112`. Refresh the CLI goldens
   (`tests/test_bead/golden/cli/show_json.stdout` and siblings) and the JSONL goldens
   (`tests/test_bead/golden/jsonl/`, `tests/test_bead/golden/stores/`), including a new
   `pre_assignee_stack` legacy golden if the suite follows the sase-core pattern.
2. **Pages and providers.** `src/sase/bead_pages/rendering_identity.py:295-296` and
   `src/sase/artifact_providers/builtin_entry_bead.py:120-121`: expose the trail
   alongside the current assignee.
3. **Mobile helper.** `src/sase/integrations/_mobile_helper_beads.py:270`: emit the
   additive `assignees` list (the gateway-side field landed in the core phase).
4. **ACE/TUI.** `src/sase/ace/tui/widgets/artifacts/beads_detail.py:122/219-220` (and
   `beads_rendering.py:475-476`): show the trail in the bead detail pane. Filtering
   semantics stay scalar (`assignee:` matches the current holder) — do not touch
   `filter_query.py` key behavior. If the detail pane is covered by PNG snapshots,
   update them via `just test-visual` with `--sase-update-visual-snapshots`.
5. **Docs and memory.** Update `docs/beads.md` (claim lifecycle and the relocation
   section around lines 1014-1019) to describe the assignment stack and takeover
   semantics. Update `sase/memory/sase_beads.md` accordingly — memory files must go
   through the `/sase_memory_write` skill.

Verification: `just install`, then `just check` (plus `just test-visual` if snapshots
changed).

## Follow-ups for the land agent

Beyond the standard landing duties (`just check-full` through a monitor before the
combined tree lands), file these as task beads when the epic closes:

1. **Relocation should rewrite derived agent names.** Bead-id relocation
   (`src/sase/bead/conflict_resolver.py`, `src/sase/bead/relocation.py`, plus the Rust
   merge machinery) rewrites bead ids inside events but leaves `assignee` and
   `epic_work_preclaimed.agent_name` strings stale. Since epic worker names are
   deterministic functions of bead ids, relocation can rewrite matching assignee
   spellings (`<old-id>` / `<old-id>.<suffix>` / `<old-id>.land`). The stack makes the
   stale spellings harmless history, but rewriting stops them from being minted at all.
2. **In-process successors could record their assignment.** `/sase_pipe` and plan-chain
   hand-offs (`continue_as_successor`) never re-claim, so the bead's current holder
   stays the predecessor's name while the successor works. With the stack in place, a
   cheap push-on-succession would make the trail reflect reality.
3. **Decisions record.** Propose a decisions-web record capturing the two invariants
   this epic establishes: the assignee stack is derived from the event stream (never
   directly settable), and host-directed launch claims take over with a push while
   foreign-owner holders still refuse.
