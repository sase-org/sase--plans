---
tier: tale
title: Fix imported registry entries squatting local agent namespaces
goal:
  "Xprompt swarm launches with keyed name templates (e.g. #research_swarm) succeed on
  machines holding agents synced from a sibling machine, existing registries auto-repair
  on upgrade, and any residual blocked-namespace collision fails fast with one clear,
  actionable launch error instead of a raw NameCollisionError traceback or an infinite
  allocation loop."
size: medium
proposed_by: bbugyi200.apollo.7
create_time: 2026-09-02 14:04:37
status: wip
---

# Fix Imported Registry Entries Squatting Local Agent Namespaces (Xprompt Swarm NameCollisionError)

## Problem

Launching `#research_swarm` on this user's machine fails with:

```
Error: NameCollisionError: agent name 'research.0' is inside reserved owner namespace 'research'
```

(Two occurrences recorded in `~/.sase/logs/launch_failures.log`, both with the same
traceback through `prepare_clan_launches` -> `reserve_registered_clan_name` ->
`ensure_local_namespace_available`.)

## Root Cause (verified, full chain)

1. **Synced/imported artifacts carry one unlocalized name field.** Agents imported from
   a sibling machine (same username, different machine; e.g. agents synced from the
   user's `athena` machine onto a machine whose configured identity is a different
   machine name) store a correctly localized `name` (e.g. `athena.research.b.cld.f0`) in
   `agent_meta.json` / `done.json`, but keep `workflow_name` in the _source machine's
   bare spelling_ (`research.b.cld.f0`).

2. **The registry rebuild scan registers the bare spelling.** `names_from_payloads()` in
   `src/sase/agent/names/_registry_scan_payloads.py` collects both `name` and
   `workflow_name` verbatim. `add_owner_names()` in
   `src/sase/agent/names/_registry_scan_entries.py` then registers the bare
   `research.b.cld.f0` as a `claimed` entry and derives a bare `research` `auto_prefix`
   entry. Because the payload carries `imported_source_owner` + `canonical_global_name`,
   `_entry_provenance()` stamps these entries `origin: import_v2`. The same leak floods
   the registry with thousands of other bare imported roots (`00`, `012`, ... — one per
   historical auto-named agent on the source machine).

3. **The namespace guard blocks the whole local subtree.**
   `ensure_local_namespace_available()` in
   `src/sase/agent/names/_registry_mutation_support.py` rejects any local allocation
   whose dotted prefix has an entry with `container_kind == "owner_namespace"` **or**
   `origin in {import_v1, import_v2}`. The bare `research` import entry therefore
   permanently blocks _every_ local `research.*` name — no retry or alternate token can
   ever succeed.

4. **The launch pipeline picks a doomed name anyway, then dies.** `#research_swarm`
   names members with keyed templates (`%id:research.{@1}.cdx`,
   `%clan(research.{@1}, ...)`). Keyed markers are resolved by `_allocate_key_tokens()`
   in `src/sase/agent/agent_name_keys.py`, whose availability oracle
   (`AgentNameNamespaceReservationIndex.candidate_available` in
   `src/sase/agent/names/_templates.py`) only checks exact reserved names and occupied
   dotted namespaces — it does **not** implement the guard's blocked-prefix rule. It
   picks token `0` and bakes concrete names (`research.0.cdx`, clan `research.0`) into
   the segments. Concrete `%id` names skip planned reservation
   (`planned_name_for_prompt` in `src/sase/agent/multi_prompt_reference_allocator.py`
   returns without reserving), so the first registry write is the clan reservation in
   `prepare_clan_launches()` (`src/sase/agent/multi_prompt_launch_plan.py`, the
   `reserve_registered_clan_name` call), which raises a raw `NameCollisionError` that
   kills the whole launch with a traceback.

There is also a **latent hang**: template allocation paths that _do_ reserve
(`PlannedNameAllocator._allocate_template_name`, `allocate_agent_name_template`) catch
`NameCollisionError` per token and advance to the next token of an **infinite** token
iterator. When a whole base is blocked by the prefix rule, every token fails and the
loop never terminates.

## Fix Design

Two production fixes plus a resilience layer, all in Python in this repo (the name
registry and launch planning are existing Python subsystems; no Rust-core surface
changes — the core is only used for template parsing/token iteration, which is
untouched).

### 1. Localize imported payload names during registry scans (root cause)

In `src/sase/agent/names/_registry_scan_entries.py` / `_registry_scan_payloads.py` /
`_registry_scan_collectors.py`:

- Add a helper that localizes every payload-derived name (agent names, `workflow_name`,
  family, and clan names) through the payload's import provenance before any entry is
  registered:
  - No imported provenance (`source_owner`/`imported_source_owner`/
    `imported_from_machine` absent) -> keep the name unchanged (local behavior is
    unchanged).
  - v2 provenance: classify with `classify_imported_agent_owner`. `EXACT_OWNER` ->
    unchanged (a machine re-scanning its own artifacts keeps short spellings).
    Otherwise, if the name is already rooted for its classification (starts with
    `<source machine>.` for same-user-other-machine, or `<source username>.` for foreign
    username) keep it; otherwise globalize the bare spelling with the source owner
    (`<username>.<machine>.<bare>`) and run `localize_imported_agent_name` against the
    current identity.
  - v1 provenance (`imported_from_machine`): qualify bare spellings with the source
    machine prefix (matching `claim_imported_registered_name`'s
    `validate_qualified_name` contract).
  - If a name cannot be localized, drop it (never register a spelling that squats the
    local namespace).
- Apply the helper in `_collect_workflow_artifact_entries` and
  `collect_dismissed_bundle_entries` before `add_owner_names` / `add_owner_family` /
  `add_owner_clan`, so auto-prefix derivation (`extract_auto_name_prefix`) also runs on
  localized spellings and produces qualified prefixes (`athena.research...`), never bare
  ones.
- In `collect_owner_namespace_entries`, also reserve an `owner_namespace` container for
  each _observed_ same-user sibling machine root (derived from entries' `source_owner`
  whose username matches the current owner but whose machine differs), mirroring the
  `sibling_machine` container the v2 import mutation path creates. When the root name is
  currently held by an import-origin `auto_prefix` entry, the container takes precedence
  (same displacement idea as `_promote_container_over_auto_prefix`).

### 2. Auto-repair existing registries (data fix for this machine)

The registry staleness check only fingerprints artifact paths/mtimes, so a code upgrade
alone never triggers a rebuild. In `src/sase/agent/names/_registry_store.py`:

- Add a module constant `SCAN_VERSION = 1`, persist it in the envelope from
  `registry_data()`, and make `registry_file_is_stale()` treat a missing or different
  stored `scan_version` as stale.

This makes every existing registry rebuild exactly once after upgrade, deleting the bare
imported squatter entries (they are re-derived only from localized payload names). No
manual repair steps and no new CLI are needed.

### 3. Launch resilience: consistent oracle + fail-fast, actionable errors

- Add a registry query (in `src/sase/agent/names/_registry_queries.py`, exported through
  the package) returning the _blocked local namespace roots_: names of entries that
  `ensure_local_namespace_available` treats as blocking
  (`container_kind == "owner_namespace"` or `origin in {import_v1, import_v2}`).
- Extend `AgentNameNamespaceReservationIndex` (in `src/sase/agent/names/_templates.py`)
  with those blocked roots: `candidate_available` returns `False` when a candidate name
  or its namespace sits at/under a blocked root, and a new method reports which blocked
  root (if any) covers a template's static dotted base. Wire the blocked roots into the
  registry-backed index builders used by `_allocate_key_tokens`
  (`src/sase/agent/agent_name_keys.py`), `PlannedNameAllocator`
  (`src/sase/agent/multi_prompt_reference_allocator.py`), and
  `allocate_agent_name_template`.
- **Fail fast instead of hanging or picking doomed tokens**: before entering the token
  loop, if the template/keyed-marker shape's static dotted base (per
  `agent_name_template_base` / the shape's prefix) is inside a blocked root, raise a
  typed error (e.g. `AgentNameBaseReservedError`, carrying the base and the blocking
  root) whose message names the blocking root and its recorded source owner and tells
  the user to choose a different base name. This is what prevents the infinite token
  loop once the oracle knows about blocked roots.
- Launch layers surface it cleanly: convert the typed error to `DirectiveError` in the
  launch planning paths (keyed-marker resolution and clan prepass), and in
  `prepare_clan_launches` also wrap any residual `NameCollisionError` from
  `reserve_registered_clan_name` into a `DirectiveError` (keep the existing "already
  exists" special case) so a failed swarm launch reports one clear message instead of a
  raw traceback.

## Steps

1. Implement the payload-name localization helper + apply it in the artifact and
   dismissed-bundle collectors; add the observed sibling-machine owner-namespace
   containers (`_registry_scan_entries.py`, `_registry_scan_payloads.py`,
   `_registry_scan_collectors.py`).
2. Add `SCAN_VERSION` to the registry envelope and staleness check
   (`_registry_store.py`).
3. Add the blocked-roots registry query and thread it through
   `AgentNameNamespaceReservationIndex` and its registry-backed builders; add the
   fail-fast typed error for blocked template bases in `_allocate_key_tokens`,
   `PlannedNameAllocator`, and `allocate_agent_name_template`.
4. Wrap launch-path errors (`agent_name_keys` callers, `prepare_clan_launches`) into
   `DirectiveError` with the actionable message.
5. Tests (see below), then run `just check`; hand `just check-full` to a monitor only if
   the change escalates per the two-speed rule.

## Tests

- `tests/test_agent_name_registry_rebuild.py`: scanning an artifact payload with
  imported v2 provenance and a bare `workflow_name` registers only localized
  (`<machine>.`-rooted) entries — no bare name, no bare auto-prefix; an
  `owner_namespace` container exists for the observed sibling machine; local
  (no-provenance) payloads scan exactly as before; a registry written without
  `scan_version` is stale and rebuilds once.
- `tests/test_agent_name_registry_reservations.py` /
  `test_agent_name_registry_claims.py`: after a rebuild of imported-only sources,
  reserving a local name under the previously squatted base (e.g. `research.0`)
  succeeds; reserving under the sibling-machine root (`athena.anything`) still raises.
- `tests/test_agent_name_key_markers.py`: with a blocked root covering a keyed shape's
  base, resolution raises the typed error (message names the root and source owner)
  instead of returning a doomed token; with only _sibling_ names reserved, allocation
  still picks the next token (no regression).
- Template allocation: a template whose base is inside a blocked root raises the typed
  error immediately (regression test for the infinite-loop hazard) — both via
  `allocate_agent_name_template` and `PlannedNameAllocator`.
- Clan prepass: `prepare_clan_launches` converts a colliding clan reservation into
  `DirectiveError` (no raw `NameCollisionError` escapes).

## Out of Scope

- The broader multi-machine agent-sync presentation/revivability architecture
  (dismissed-by-default sync state, `*--code` shell grouping, hood presentation rules):
  the user has a separate research effort for that. This plan only fixes registry
  keying, the resulting launch failure, and launch-time resilience.
- No new CLI subcommands and no feature flags: the rebuild is automatic and the behavior
  change (not squatting local names with foreign spellings) is a bug fix with no old
  branch worth preserving.
