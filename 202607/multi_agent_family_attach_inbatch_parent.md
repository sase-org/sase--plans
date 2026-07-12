---
create_time: 2026-07-08 14:45:57
status: wip
prompt: 202607/prompts/multi_agent_family_attach_inbatch_parent.md
tier: tale
---
# Plan: Attach `%n(parent, suffix)` to an in-batch sibling named earlier in the same multi-agent prompt

## Problem

In a multi-agent prompt, naming an agent in one segment and attaching a new family member to it from a later segment
fails:

```text
%n:foo Plan the change.
---
%n(foo, reviewer) Review foo's plan.
```

The second segment errors at launch time:

```
_FamilyAttachError: Cannot attach family member with %n(foo, reviewer): parent agent 'foo' was not found in
project '<project>'.
```

The user's intent is that `%n(parent, suffix)` should be able to reference a parent that was just named earlier in the
**same** multi-agent prompt, so custom agent-family members can be composed inline. Today that always fails.

## Root Cause (verified by reading the launch path)

There are **two separate agent stores**, written at different times, and family-attach parent resolution consults the
wrong one for the in-batch case:

1. **Name registry (store A)** — a durable `agent_name_registry.json` written **synchronously** by the launcher when a
   segment's name is planned (`reserve_registered_name(..., reservation_kind="planned")` via `PlannedNameAllocator`).
   This prevents name collisions but is **never consulted** for family-parent resolution.

2. **Agent artifact index / `agent_meta.json` (store B)** — written **asynchronously by the child runner process** after
   it boots (`write_agent_meta` in `src/sase/axe/run_agent_directives.py`). This is the **only** source
   `%n(parent, suffix)` resolution reads.

The multi-agent launch flow (`src/sase/agent/multi_prompt_launcher.py::_spawn_segments_into`) iterates segments
**sequentially**, but each segment is spawned as a **detached subprocess** that returns immediately — the launcher does
**not** wait for the child to boot and write its `agent_meta.json`. It only inserts an inter-segment naming wait for a
**bare `%wait` / `%resume`** in the next segment (`next_segment_needs_name`); a `%n(parent, suffix)` family reference
does **not** trigger that wait.

Family-attach resolution happens per slot inside `execute_launch_plan` (`src/sase/agent/launch_executor.py:99-106` →
`prepare_family_attach_launch` → `_resolve_family_attach_plan` in `src/sase/agent/family_attach.py`).
`_resolve_family_attach_plan` builds candidates only from `_agent_family_snapshot(project)` (a scan of store B). When
segment 2 resolves `%n(foo, reviewer)`, segment 1's `foo` child has not yet written `agent_meta.json`, so the Rust
resolver (`resolve_agent_family_parent`) returns `kind == "absent"` and `_resolve_family_attach_plan` raises
`_FamilyAttachError` (`family_attach.py:277-280`, message from `_resolution_error_message`).

So the parent's **name** is reserved synchronously (store A) and the launcher already knows everything about what it
just launched, but none of that in-memory knowledge reaches family-parent resolution — it only reads the
asynchronously-populated store B.

## Design Decision

**Make family-attach resolution aware of the agents launched earlier in the same launch batch ("in-batch siblings"),
resolving an in-batch parent authoritatively from in-memory launch data instead of from the async artifact index.**

The launcher is the source of truth for what it just launched. As each explicitly-named slot is spawned, we know its
resolved name (`extract_static_name_directive` / the planned name), timestamp, project, workspace, and CL — everything a
family-attach plan needs. We thread a growing list of these in-batch siblings into per-slot family-attach resolution.
When `%n(parent, suffix)` names an in-batch sibling, we resolve against it directly and build a **running-parent queued
plan**: the new member launches as a WAITING child under the parent and starts when the parent's run completes — exactly
the documented semantics for attaching to a still-running parent.

This is deterministic (no polling for the child's async `agent_meta.json`, no race) and needs no new persisted state.

### Why this interpretation

The user explicitly wants the inline composition to _work_. A running parent is the correct model: segment 1 (`foo`) is
live when segment 2 launches, and the documented behavior for attaching to a running parent is already "queue the child
as WAITING and start it when the parent completes successfully." We are only making the parent **resolvable**; the
downstream queue/wait/workspace-reuse machinery is unchanged and already exists for family attaches to running parents.

### Threading approach (single chokepoint)

`execute_launch_plan` is the common point every launch surface funnels through, and it already processes slots
sequentially. It becomes the accumulation point:

- It accepts an optional `pending_family_parents` list (defaulting to a fresh internal list).
- Before spawning each slot it passes that list to `prepare_family_attach_launch`.
- After spawning a slot that carries an **explicit static name** (from `extract_static_name_directive(slot.prompt)`) and
  has a resolved workspace, it appends an in-batch-sibling descriptor built from the spawn record.

For a single `execute_launch_plan` call this covers intra-fan-out references (e.g.
`%{%n:foo ... | %n(foo, reviewer) ...}`). For multi-agent prompts — where each `---` segment is a **separate**
`execute_launch_plan` call — `_spawn_segments_into` creates **one shared** `pending_family_parents` list before the
segment loop and passes the same list object to every segment's `execute_launch_plan`, so segment 1's `foo` is visible
when segment 2 resolves `%n(foo, reviewer)`. This mirrors the existing cross-segment threading of `previous_agent_name`
and `upstreams`.

### Resolving an in-batch parent (reuse the Rust resolver, no Rust change)

Parent resolution stays in the Rust binding `resolve_agent_family_parent`. `_resolve_family_attach_plan` simply adds the
in-batch siblings to the **candidate set** it already builds from the artifact snapshot (each sibling becomes a
candidate dict via a new `_candidate_from_sibling`, marked `is_terminal=False` so it resolves as `running`). In-batch
siblings carry the freshest batch-allocated timestamps, so the resolver's existing "newest wins" rule makes them take
precedence over any persisted agent of the same name — which is the intended behavior (the user just typed `%n:foo`).

After resolution, when the resolved parent's `artifact_dir` belongs to an in-batch sibling,
`_resolve_family_attach_plan` builds the `_FamilyAttachLaunchPlan` directly from the sibling descriptor (running parent)
and **skips** the `_record_by_artifact_dir(...).agent_meta` lookup (there is no scan record for a not-yet-persisted
sibling). Suffix resolution, `@` allocation, and the "member name already exists" check (`_resolve_role_suffix` /
`_ensure_family_name_available`) continue to run, extended to also account for in-batch member names so repeated
`%n(foo, @)` across segments allocate distinct suffixes and collisions are still caught.

### Rust core backend boundary

No change to `../sase-core` is required. The **domain** resolution rule (newest / ambiguous / dismissed / terminal)
stays in the existing Rust binding; we only extend the Python-side **candidate-set construction**, which already lives
in `family_attach.py`. The in-batch sibling set is ephemeral, in-process launch-orchestration state that only the
launching process knows; the launch orchestration loop it lives in (`_spawn_segments_into`, `execute_launch_plan`,
`prepare_family_attach_launch`) is already Python. If a future Rust launch orchestrator subsumes this loop, the
candidate-augmentation moves with it.

### Alternatives considered (not chosen)

- **Make `%n(parent, suffix)` trigger the inter-segment naming wait** (like bare `%wait`) and poll for the parent's
  `agent_meta.json`. Rejected: it is race-based (waits on an async write), adds launch latency, and still fails if the
  child is slow to boot or crashes on startup. The in-batch data is already known deterministically.
- **Consult the synchronous name registry (store A) during parent resolution.** Rejected: store A holds only the name,
  not the timestamp/workspace/artifact-dir/CL a launch plan needs, and it is a collision index, not a family-membership
  source — mixing the two concerns.
- **Synchronously write a placeholder artifact record for the parent at plan time.** Rejected: pollutes persisted store
  B, can orphan records if the child never boots, and duplicates the name-reservation concept.

## Scope of Change

- `src/sase/agent/family_attach.py`
  - Add a `FamilyAttachSibling` descriptor (name, family base, timestamp, artifact_dir, project_name, cl_name,
    workspace_dir, workspace_num) and a small builder that derives it from a launched slot's spawn request + resolved
    name (canonical artifact dir via `canonical_agent_artifact_path(project, "ace-run", <artifacts-format ts>)`).
  - Thread `pending_family_parents` through `prepare_family_attach_launch` and `_resolve_family_attach_plan`.
  - Add `_candidate_from_sibling`; merge sibling candidates into the resolver request; when the resolved parent is an
    in-batch sibling, build the running-parent `_FamilyAttachLaunchPlan` from the descriptor and skip the scan-record
    lookup. Extend `_resolve_role_suffix` / `_ensure_family_name_available` inputs to include in-batch member names.
- `src/sase/agent/launch_executor.py`
  - Add `pending_family_parents` param to `execute_launch_plan`; seed it (or a fresh list), pass it to
    `prepare_family_attach_launch`, and append a sibling descriptor after spawning any slot with an explicit static name
    and a resolved workspace.
- `src/sase/agent/multi_prompt_launcher.py`
  - Create one shared `pending_family_parents` list before the segment loop and pass it to every segment's
    `execute_launch_plan` call.

## Edge Cases & Risks

- **Precedence vs. a persisted same-name agent.** In-batch siblings use fresh batch-allocated timestamps and win the
  resolver's "newest" tie-break. Verify with a test that an in-batch `foo` is preferred over an older persisted `foo`.
- **`%n(foo, @)` and collisions within one batch.** Two segments attaching to the same in-batch `foo` must allocate
  distinct suffixes (`--1`, `--2`) and re-typing an existing suffix must still error. This requires the in-batch members
  (the attached children, not just the named parents) to be visible to suffix allocation / availability checks. Include
  in-batch member names in `_resolve_role_suffix` / `_ensure_family_name_available` inputs and cover with tests.
- **Workspace-override composition (multi-prompt specific).** A running-parent attach flips the child to
  `deferred_workspace=True` and reuses the parent's workspace. Confirm this composes with the multi-prompt segment's own
  VCS/workspace context resolution without double-claiming or leaking a workspace (family-attach-to-running-parent
  already works from single launches; the risk is the multi-prompt segment context path). Add an integration test.
- **Wait-dependency identity fields.** The child's running-parent wait keys on
  `{project_name, timestamp, artifact_dir, name}` (`run_agent_directives.py:113-119`). Ensure the sibling descriptor's
  `timestamp` and `artifact_dir` use the same representation the parent actually registers under (artifacts-format
  timestamp + canonical artifact dir) so the child waits on the correct run.
- **Template / auto-named parents.** `extract_static_name_directive` returns the clean name for `%n:foo` / `%n:!foo` and
  `None` for `%n(...)`. A parent named via a name **template** (`%n:foo@`) or bare/auto `%n` is out of scope for this
  change (its resolved name is not statically known at the accumulation point); such references keep today's behavior.
  State this limitation explicitly.
- **Grandchild attach in the same batch** (attaching to an in-batch child such as `%n(foo--reviewer, ...)`) is out of
  scope; only explicitly-named siblings become parents in this iteration.
- **No behavior change for existing paths.** Single-agent launches and family attaches to already-persisted parents are
  unaffected (the sibling list is empty/irrelevant); the added parameter defaults keep all current callers working.

## Testing

- `tests/test_dynamic_agent_family_attach.py`
  - `_resolve_family_attach_plan` with a `pending_family_parents` in-batch `foo` and an empty snapshot resolves a
    running-parent plan (`parent_is_running is True`, `agent_name == "foo--reviewer"`, parent workspace/timestamp taken
    from the sibling) — this is the previously-failing case.
  - In-batch `foo` takes precedence over an older persisted `foo`.
  - `%n(foo, @)` against an in-batch parent allocates a feedback suffix; a second attach in the same batch allocates the
    next distinct suffix; re-typing an existing member name still raises the collision error suggesting `%n(foo, @)`.
- Multi-prompt launcher integration test: a two-segment prompt (`%n:foo ...` / `---` / `%n(foo, reviewer) ...`) launches
  both agents with no `_FamilyAttachError`; the second carries family metadata (name `foo--reviewer`, family base `foo`,
  a running-parent wait dependency on `foo`) and reuses `foo`'s workspace. Also cover the intra-fan-out variant through
  a single `execute_launch_plan` call.
- Regression: the standalone absent-parent error still fires when neither an in-batch sibling nor a persisted record
  matches.
- Run `just install` then `just check` (lint + mypy + tests) in this repo. No `../sase-core` changes are expected; if
  the candidate wiring surprises the resolver, re-check the binding contract before touching Rust.

## Documentation

- `docs/xprompt.md` (family-attach section): note that in a multi-agent prompt, `%n(parent, suffix)` may reference a
  parent named earlier in the **same** prompt (an in-batch sibling); such a parent is treated as still-running, so the
  member queues as a WAITING child that starts when the parent completes.
- `docs/agent_families.md` ("Extending a Family by Hand"): add the inline multi-segment example (`%n:foo ...` / `---` /
  `%n(foo, reviewer) ...`) and state the template/auto-named-parent limitation.

## Out of Scope

- Attaching to a template-named, bare/auto-named, or in-batch **child** parent within the same batch.
- Any change to `../sase-core` (`resolve_agent_family_parent` stays as-is; only the Python candidate set is extended).
- Changing family-attach queue/wait/workspace-reuse semantics for already-persisted parents.
