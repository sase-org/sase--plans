---
tier: tale
title: Fix approved tale plans never launching their coder agent
goal:
  Approving a tale plan launches its coder follow-up again; empty identity-field
  sentinels no longer abort family-attach resolution.
size: small
proposed_by: bbugyi200.kellys_mbp.t
create_time: 2026-09-04 12:57:33
status: wip
---

# Fix approved tale plans never launching their coder agent (EmptyAgentName regression)

## Problem

Approving a tale plan from the ACE TUI settles the plan gate shell and should launch a
`<lane>--code` coder follow-up agent. Since the typed-owner-identity wiring landed
(commit `f4032c55d`, 2026-09-04), every tale-plan approval on a machine with a
configured owner identity fails to launch its coder. The approval itself "succeeds"
(plan archived, gate settled), but the shell records:

```
gate_followup_outcome: not-launchable
gate_followup_error: agent name must not be empty; follow-up prompt saved to .../gate_followup_prompt.md
```

Two real approvals hit this (plans `procs_pane_kill_feedback.md` and
`family_runtime_monitor_starter.md`); tale approvals from the previous day, running the
pre-`f4032c55d` build, launched their coders fine.

## Root cause

`resolve_family_attach_plan` (`src/sase/agent/_family_attach_resolution.py:200`)
computes `sase_plan=_candidates.family_sase_plan(snapshot.records, parent_base)` — but
only when the resolved role is `"code"`. That is why exactly the coder follow-up breaks
while feedback/epic follow-ups survive.

`family_sase_plan` (`src/sase/agent/_family_attach_candidates.py:218-246`) filters the
full agent snapshot with:

```python
current_owner_agent_name_key(record.agent_meta.agent_family or "")
or current_owner_agent_name_key(record.agent_meta.workflow_name or "")
or current_owner_agent_name_key(record.agent_meta.name or "")
```

It deliberately passes `""` for absent fields. Roughly 40% of snapshot records have no
`agent_family`, so the first such record always feeds `""` into
`current_owner_agent_name_key`.

Before `f4032c55d`, `foreign_agent_owner_root` and `normalize_owned_agent_name` in
`src/sase/core/agent_identity_facade.py` were pure-Python string manipulations that
passed empty input through harmlessly (`current_owner_agent_name_key("") == ""`).
`f4032c55d` replaced their bodies with strict `sase_core_rs` Rust bindings whose
validation (`normalize_agent_archive_name` → `validate_historical_semantic_name`) raises
`AgentIdentityError::EmptyAgentName` → Python `ValueError: agent name must not be empty`
for empty input. The guard only engages when `snapshot.owner` is configured, which it
now is.

The `ValueError` escapes `resolve_family_attach_plan` → `spawn_family_successor` and is
caught by `launch_shell_followup` (`src/sase/shells/followup.py:186`), which records the
follow-up as not-launchable — silently, from the approving user's point of view.

Reproduced deterministically: calling
`resolve_family_attach_plan(FamilyAttachDirective(parent="p", suffix="--code"), project_name="gh_sase-org__sase")`
raises `ValueError: agent name must not be empty` from
`_family_attach_candidates.py:227`.

## Fix

Two layers: fix the offending call site semantics, and restore the facade's historical
empty-input totality so the other ~130 facade callers cannot reintroduce the same silent
launch-killer.

### 1. `src/sase/agent/_family_attach_candidates.py` — `family_sase_plan`

Do not key empty sentinel values at all. An empty field can never match the non-empty
`parent_key`, so skip falsy values instead of calling
`current_owner_agent_name_key(... or "")`. For example, introduce a small local helper:

```python
def _key_matches(value: str | None, parent_key: str) -> bool:
    return bool(value) and current_owner_agent_name_key(value) == parent_key
```

and rewrite the `family_records` comprehension to use
`_key_matches(record.agent_meta.agent_family, parent_key) or _key_matches(record.agent_meta.workflow_name, parent_key) or _key_matches(record.agent_meta.name, parent_key)`.

### 2. `src/sase/core/agent_identity_facade.py` — restore empty-input totality

Add early returns for empty names before consulting the Rust binding, mirroring the
existing `if owner is None:` Python-side guards in the same functions:

- `foreign_agent_owner_root`: `if not name: return None` (an empty name cannot carry a
  foreign owner root).
- `normalize_owned_agent_name`: `if not name: return name`.

This automatically makes `current_owner_agent_name_key("")` return `""` and keeps
`current_owner_agent_name_lookup_candidates` total, restoring the exact pre-`f4032c55d`
contract for empty input.

Rust-core boundary note: do NOT relax the validation in `sase-core`'s
`agent_identity/identity.rs`. The Rust primitives are shared identity boundaries
(imports, archives, launch validation) where rejecting empty names loudly is correct.
Empty-input tolerance here is Python-facade compatibility glue for optional/absent
record fields, the same class of guard as the existing `owner is None` early returns;
real agent launches still reject empty names via launch validation.

## Regression tests

1. `tests/test_agent_identity_facade.py`: with a configured
   `AgentIdentitySnapshot(AgentOwnerIdentity(...))` (see existing tests around lines
   176-200 for the pattern), assert:
   - `foreign_agent_owner_root("", identity) is None`
   - `normalize_owned_agent_name("", identity) == ""`
   - `current_owner_agent_name_key("", identity) == ""`

2. `tests/test_dynamic_agent_family_attach_resolution.py` (helpers in
   `tests/_dynamic_agent_family_attach_helpers.py`): a test that resolves a coder-role
   directive (`FamilyAttachDirective(parent=..., suffix="--code")`) against a snapshot
   whose records include at least one record with `agent_family=None` (and ideally one
   with an empty `name`), while `AgentIdentitySnapshot.current` is monkeypatched to a
   configured owner. Without the fix this raises
   `ValueError: agent name must not be empty`; with the fix it resolves and returns a
   launch plan. The owner-identity monkeypatch is required — with owner `None` the old
   lenient path is taken and the test would pass even without the fix.

3. Optionally a direct `family_sase_plan` unit test: records with mixed empty/populated
   `agent_family`/`workflow_name`/`name` fields under a configured owner return the
   newest matching record's plan path without raising.

## Verification

- `just install` first if the workspace virtualenv is stale (ephemeral clones).
- `just fmt` and `just check` (agent default lane) must pass before finishing.
- Confirm the end-to-end repro is fixed: in the repo virtualenv, run
  `resolve_family_attach_plan(FamilyAttachDirective(parent=<any existing lane>, suffix="--code"), project_name=<project>)`
  against the live snapshot and confirm it returns a plan instead of raising (read-only;
  it does not spawn anything).

## Out of scope

- Surfacing `gate_followup_error` to the user as a notification (the failure was silent;
  tracked separately as task bead `sase-wp`).
- Re-launching the two affected plans' coders — already remediated manually.
- Any change to `sase-core` Rust validation.
