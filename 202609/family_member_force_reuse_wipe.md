---
tier: tale
title:
  Scope forced-reuse wipes of agent-session members so ,x relaunches keep their family
  parent
goal:
  Relaunching a serial agent-session member with ,x (%id(!suffix, family=P)) wipes only
  that member and its own descendants, never the session root or sibling members, so
  family-attach resolution finds the parent and the relaunch succeeds.
size: medium
proposed_by: bbugyi200.athena.0r0
create_time: 2026-09-24 12:47:17
status: wip
---

# Stop `,x` family-member relaunches from wiping their own family parent

## Problem

Relaunching a serial agent-session (family) member with `,x` fails with:

```
Cannot attach family member with %i(code, family=sase-17m.3.1.land): parent agent
'sase-17m.3.1.land' was not found in project 'gh_sase-org__sase'.
```

It also deletes the entire family's artifacts, including the parent root's pinned
workspace hold. This is silent data loss followed by a misleading error.

## Root Cause (verified)

`,x` on a family member (here `sase-17m.3.1.land--code`, whose root was
`sase-17m.3.1.land--plan`) rewrites the prompt to `%id(!code, family=sase-17m.3.1.land)`
(`src/sase/agent/relaunch_prompt.py`, `prepare_kill_and_edit_prompt`). In `sase run`
(`src/sase/main/query_handler/_launch.py`), `apply_force_reuse_launch` wipes the
force-reuse owner name `sase-17m.3.1.land--code` **before** `launch_agents_from_cwd`
runs the family-attach parent resolution. The wipe closure is not scoped to that one
member, so it removes the parent root too. Resolution then finds nothing and reports
"parent ... not found".

Two independent leaks pull the root into the wipe of `P--code`:

1. **`workflow_name` is treated as an owner identity.** Every plan-chain member
   (`P--plan`, `P--gate`, `P--code`, `P--mon`, ...) stores `workflow_name == P`, the
   agent-session container name. `payload_names()` in
   `src/sase/agent/names/_wipe_payload.py` returns `name` and `workflow_name`.
   `build_wipe_plan()` (`_wipe_plan.py`) then links every artifact that shares a name.
   Seeding from `P--code` adds `P`, and the closure pulls in the root and every sibling.
   The name registry scan already knows `P` is a container: it calls
   `names.discard(family)` in `_registry_scan_collectors.py`. The wipe scan and seed
   never do.
2. **`%auto` roots claim the code member's name.** When a `%auto` plan chain continues
   in-process into the code step, `finalize_loop`
   (`src/sase/axe/run_agent_exec_finalize.py`, the
   `if state.current_artifacts_dir != ctx.artifacts_dir:` copy near the end) writes the
   code member's done marker, whose `name` is `P--code`, into the **root** dir too. The
   registry collects names from both `agent_meta.json` and `done.json`, so the live
   registry maps `P--code` to the root artifact dir. On this host,
   `sase-17d.5--code -> .../20260924072618`, which is the `--plan` root. The wipe
   therefore seeds directly from the root.

Evidence from the failed launch:

- Proc `62e5dxffc2j8` ran from 12:07:45 to 12:08:28.
- At 12:07:54 a child of that proc released the root `20260924115043`'s pinned workspace
  #14 hold (`~/.sase/logs/workspace_claims.jsonl`). That is `execute_wipe_plan` →
  `_release_artifact_workspace` running on the root.
- The root, gate and code artifact dirs are gone, with no dismissed bundles.

A read-only `preview_agent_name_wipe("sase-17d.5--code")` on live data lists the whole
family. A temp-HOME repro shows both the non-`%auto` family shape and the `%auto` family
shape losing the root and gate when `P--code` is force-reused. Parent resolution itself
is correct. With the real `sase_core_rs` binding, `resolve_family_attach_plan` picks the
`P--plan` root for surviving families.

The existing test `test_wipe_code_member_preserves_plan_member_and_family_container`
(`tests/test_agent_name_wipe.py`) misses this. Its `_artifact()` helper sets
`workflow_name` to the member's own name, not the family base, and no fixture writes a
`%auto` root `done.json`.

## Changes

### 1. One shared owner-identity name derivation (registry and wipe)

Add a single helper that returns an artifact's **owner identity names** from its
`(agent_meta, done)` payloads. Put it next to `names_from_payloads` in
`src/sase/agent/names/_registry_scan_payloads.py`, or in `_wipe_payload.py` and import
it from both places. Registry and wipe must use the same code so they cannot drift
again.

Rules:

- Start from the same keys used today: `name` and `workflow_name`, plus `agent_name` for
  bundle payloads.
- Discard the agent-session container name.
  - Use `family_from_payload(meta)`, which reads `agent_session` and the legacy
    `agent_family` spelling through `sase.plan_chain.agent_session_value`.
  - Legacy fallback: if the payload has no session key but `workflow_name` equals
    `agent_session_base(name)` and differs from `name`, treat `workflow_name` as the
    container too.
- Discard a `done.json` `name` that names a **different member of the same agent
  session** than meta's own `name`. The test: meta has a non-empty `name`, the session
  `S` is known, the done name is not the meta name, and `agent_session_base(done_name)`
  equals `S`. This targets the `%auto` root copy while keeping the other cases where
  done names differ.
- Parallel sessions (`agent_session_parallel`) keep today's behavior, because
  `family_from_payload` already returns `None` for them.

Use the helper in these places:

- The registry artifact scan, `_collect_workflow_artifact_entries` in
  `_registry_scan_collectors.py`. It replaces `names_from_payloads(meta, done)` plus the
  manual `names.discard(family)`.
- The registry bundle scan, `collect_dismissed_bundle_entries`. Behavior should be
  unchanged, since bundles have no `done`, but route it through the helper for
  consistency.
- The wipe catalog scan, `_scan_artifacts` / `_scan_bundles` in
  `src/sase/agent/names/_wipe_scan.py`, for `ArtifactRecord.names` and
  `BundleRecord.names`.
- The wipe seed, `_seed_owner` / `_seed_payload_path` in `_wipe_plan.py`. When an owner
  has an `artifacts_dir`, derive its names from that dir's meta and done payloads with
  the helper. Do not union raw `payload_names` results.

Keep the relation closure unchanged, so descendants still follow `parent_timestamp` /
`retried_as_timestamp`. Wiping `P--code` must still remove its own `--mon` child and any
retries. Wiping the root `P--plan` must still cascade to children whose
`parent_timestamp` is the root, because a family-container relaunch depends on that.

Bump `SCAN_VERSION` in `src/sase/agent/names/_registry_store.py` from 1 to 2. Existing
registries already map `P--code` to `%auto` roots, and the bump makes them rebuild once.
This matters because `_resolve_wipe_targets` seeds from the registry's recorded owner.

### 2. Stop writing the code member's name into the `%auto` root's `done.json`

In `finalize_loop` (`src/sase/axe/run_agent_exec_finalize.py`), the root-copy branch
(`write_done_marker_and_update_index(ctx.artifacts_dir, done_marker)`) should write a
copy whose `name` is the root's own identity. Read it from the root's current
`agent_meta.json` `name`, which is `P--plan` after `_family_promotion`. If that name is
unavailable, drop `name` from the root copy rather than claiming the member's name. Keep
every other field as it is today, because the root row still reflects the chain's final
outcome and response.

Before changing it, confirm nothing depends on the root `done.json` name being the final
member's name. Check these readers:

- `src/sase/ace/tui/models/_loaders/_done_snapshot_loaders.py` (`agent_name=done.name`)
- `src/sase/core/artifact_file_helpers.py` (done name preferred over meta)
- `src/sase/agents/cli_wait.py`
- `src/sase/agent/wait_watch/_resolve.py`
- `src/sase/core/agent_artifact_run_retention.py`
- `src/sase/completion/candidates/catalog_agents.py`

Adjust any that relied on it. Extend
`tests/test_axe_run_agent_exec_finalize_metadata.py` (or the nearest finalize test
module) so an in-process plan→code continuation leaves the root `done.json` naming the
root (`X--plan`) and the code dir's `done.json` naming `X--code`.

### 3. Fail-closed guard: a member wipe never deletes its session root

In `wipe_agent_names_for_reuse` (`src/sase/agent/names/_wipe.py`), check each target's
plan before `execute_wipe_plan` runs:

- The target name `T` is a non-root member of session `S`, meaning
  `agent_session_base(T) == S` and `T != S`.
- The plan's `artifact_dirs` contain an artifact that is the root of `S`. A root has
  meta session `S` and either `agent_session_role == "root"`, `plan_chain_root`, or no
  `parent_timestamp`.
- That root's identity names (from step 1's helper) share nothing with the batch's
  target names.

If all three hold, execute nothing for the whole batch. Return error results naming the
member, the root name and the root dir, for example "forced reuse of 'P--code' would
remove agent-session root 'P--plan' (<dir>); refusing to wipe".

These errors already flow through `wipe_force_reuse_owners` to
`ForcedReuseCleanupError`, and `sase run` reports them as "Agent name reuse failed:
...", so nothing is deleted. A family-container relaunch
(`_wipe_families_for_forced_reuse`) stays allowed, because the root's own member name is
one of the batch targets. This guard also covers a stale registry that still attributes
`T` to the root.

### 4. Tests

In `tests/test_agent_name_wipe.py`:

- Make the family fixtures realistic. Members should store `workflow_name` equal to the
  family base plus `agent_session`/`agent_session_role`. Either give `_artifact()` an
  explicit `workflow_name` override, or pass it through `meta`.
- Update `test_wipe_code_member_preserves_plan_member_and_family_container` to use those
  fixtures. It must fail on the current code.
- Add a `%auto` variant: the root `P--plan` has `done.json` name `P--code`, the code dir
  has `parent_timestamp` equal to the root, and there is a `P--mon` whose
  `parent_timestamp` is the code dir. Assert:
  - after `rebuild_name_registry()`, `lookup_registered_name("P--code")` points at the
    code dir;
  - `wipe_agent_name_for_reuse("P--code")` removes exactly the code dir and its `P--mon`
    descendant;
  - the root and gate survive;
  - `P`, `P--plan` and `P--gate` stay reserved.
- Add a guard test that forces a closure containing the root. For example, patch
  `build_wipe_plan` or seed a stale owner entry that points at the root. Assert that
  nothing is removed, the result carries the refusal error, and
  `wipe_force_reuse_owners` raises.
- Confirm the existing family-container and forced-reuse tests still pass. That covers
  the tests in `tests/test_agent_name_wipe.py`,
  `tests/agent/test_force_reuse_launch.py`, `tests/test_force_reuse_launch_seam.py` and
  `tests/test_agent_restart_plan.py`.

Add one launch-seam regression. Put it in `tests/test_force_reuse_launch_seam.py` or
`tests/agent/test_force_reuse_launch.py`, whichever already builds temp-HOME artifacts.
Plan and apply `%id(!code, family=P)` against a realistic `%auto` family on a temp HOME,
then run `resolve_family_attach_plan` for
`FamilyAttachDirective(parent="P", suffix="code", force_reuse=True)`. It should resolve
the `P--plan` root, with `agent_name == "P--code"`, instead of raising "parent agent 'P'
was not found". Inject the snapshot and dismissed factories if the Rust binding is not
available in the test environment.

### 5. Docs

In `docs/ace.md`, in the kill-and-edit forced-reuse table section (the paragraph after
the `%id(!<suffix>, family=<family>)` row), add one sentence. It should say that forced
reuse of a family member replaces only that member and its own descendants, and that the
family root and sibling members are left untouched.

## Out of Scope

- The stale-binding Python fallback `_resolve_agent_session_parent_fallback` in
  `src/sase/agent/_family_attach_candidates.py` differs from the Rust resolver: it has
  no `parent_timestamp` filter and uses prefix matching. It is unrelated to this
  failure, because the TUI's install uses the real binding.
- Recovering the deleted `sase-17m.3.1.land` artifacts. They were `rmtree`'d. The user
  already relaunched the land agent, and the plan file and chat transcripts survive.

## Coordination Note

The `sase-17m.3.1` wire-cutover epic (agent-family → agent-session rename) is landing
concurrently. Rebase before starting. Read session metadata only through the
`sase.plan_chain` accessors (`agent_session_value`, `agent_session_parallel_value`,
`agent_session_base`) so both spellings keep working.

## Verification

- Run `sase tool run check` (the guarded `just check`). Do not run `just check-full`.
- Targeted:
  `pytest tests/test_agent_name_wipe.py tests/agent/test_force_reuse_launch.py tests/test_force_reuse_launch_seam.py tests/test_axe_run_agent_exec_finalize_metadata.py tests/ace/tui/test_family_member_relaunch.py`.
- Manual sanity check, read-only against real data:
  `from sase.agent.names._wipe import preview_agent_name_wipe; preview_agent_name_wipe("<some-family>--code")`
  should list only that member's dir and its own descendants, not the `--plan` root,
  `--gate`, or sibling members.
