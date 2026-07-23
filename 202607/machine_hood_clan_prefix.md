---
tier: tale
title: Hide the local machine hood on clan rows that contain a family
goal: On the hood-owning machine, a clan containing an agent family renders as one
  container with the local machine prefix stripped, instead of splitting into a qualified
  and an unqualified row.
create_time: 2026-07-23 08:11:09
status: done
---

- **PROMPT:** [202607/prompts/machine_hood_clan_prefix.md](prompts/machine_hood_clan_prefix.md)

# Hide the local machine hood on clan rows that contain a family

## Problem

On the machine that owns a machine agent hood, the TUI must present agent names with the local machine prefix stripped
(e.g. on `athena`, show `sase-8t`, not `athena.sase-8t`). This works for plain clan members but **fails for a clan that
contains an agent family**: the clan's running row keeps the machine hood prefix.

Observed in the `sase ace` Agents tab on host `athena`: the `@epic` tribe panel shows two rows for the _same_ clan
`sase-8t` (generation `20260723073722`):

- `Running` → `(RUNNING) ×1 [R1] athena.sase-8t` ← machine hood NOT hidden (the bug)
- `Waiting` → `(WAITING) ×3 [W3] sase-8t` ← machine hood correctly hidden

The clan is being split into two synthetic containers, and one of them leaks the `athena.` prefix.

This regression was introduced by the `sase-8k` epic (machine agent hoods; epic commit `9a835e1d`).

### Do not collide with the in-flight `is.cdx` agent

A separate agent (`is.cdx`, plan `fix_slow_agent_launch_naming`) is fixing a _performance_ regression from the same
epic: repeated `MachineHoodIdentity.current()` calls during `rebuild_name_registry()`. Its scope is strictly
`src/sase/agent/names/_registry.py` and `src/sase/agent/names/_registry_scan.py`, and it explicitly states "no Rust core
API or TUI presentation change is needed."

This plan is the complementary, non-overlapping piece: a **TUI presentation** fix in `src/sase/ace/tui/models/agent.py`.
It touches none of the registry files, and — critically — it introduces **no new machine-identity resolution** (it
reuses an already-normalized field), so it does not reintroduce or worsen the per-record identity-discovery cost that
`is.cdx` is removing.

## Root cause

`Agent.presented_clan_reference_name()` in `src/sase/ace/tui/models/agent.py` computes the local-display
(machine-hood-stripped) clan name by _subtracting the member's own suffix_ from the member's presented name:

```python
def presented_clan_reference_name(self) -> str | None:
    raw_clan = self.agent_clan                       # e.g. "athena.sase-8t"
    if not raw_clan:
        return None
    if self.is_clan_container:
        return self.presented_agent_name or raw_clan
    raw_name = self.agent_name or ""                 # e.g. "athena.sase-8t.1--plan"
    presented_name = self.presented_agent_name or raw_name
    prefix = f"{raw_clan}."                           # "athena.sase-8t."
    if raw_name.startswith(prefix):                  # True
        suffix = raw_name[len(raw_clan) :]           # ".1--plan"
        if suffix and presented_name.endswith(suffix):
            return presented_name[: -len(suffix)]
    return raw_clan                                  # ← falls through to the QUALIFIED name
```

The subtraction assumes `presented_agent_name` is the full member name with only the machine hood removed. That
assumption holds for plain members, but **for a family-root row it does not**:

- `refresh_raw_presented_agent_name()` sets a family-root row's `presented_agent_name` to the _family container_
  reference (`family_reference_name()` → `agent_family`), then `refresh_presented_agent_name()` strips the hood,
  yielding `presented_agent_name = "sase-8t.1"` — the **shortened family name**, not `sase-8t.1--plan`.
- So `presented_name.endswith(suffix)` becomes `"sase-8t.1".endswith(".1--plan")` → **False**, and the method returns
  the raw, still-machine-qualified `raw_clan` = `"athena.sase-8t"`.

Downstream, `project_clan_tree()` → `_clan_for_row()` (`src/sase/ace/tui/models/_agent_tree.py`) uses this return value
both as the **clan grouping key** `(clan_name, generation)` and as the synthetic container's `agent_clan`. The
family-root member therefore groups under `athena.sase-8t` while the plain members group under `sase-8t`, splitting one
clan into two containers — and the family-root container renders the machine hood prefix.

### Reproduction (string-level, confirmed)

With machine `athena` and a stubbed hood-strip helper, for a clan `athena.sase-8t`:

| member                                 | `presented_agent_name` | `presented_clan_reference_name()` |
| -------------------------------------- | ---------------------- | --------------------------------- |
| `athena.sase-8t.2` / `.3` / `.land`    | `sase-8t.2` etc.       | `sase-8t` (correct)               |
| `athena.sase-8t.1--plan` (family root) | `sase-8t.1`            | `athena.sase-8t` (**bug**)        |

This matches the screenshot exactly: the running row is the family-root container (qualified) and the waiting row is the
plain-member container (stripped).

## Fix

Use the field that already exists for exactly this purpose. `Agent` (via `AgentState` in
`src/sase/ace/tui/models/_agent_state.py`) declares `presented_identity_name` with the documented contract:

> Precomputed identity used for hood/neighbor relationships. Unlike `presented_agent_name`, this retains a concrete
> family member suffix; only the owning machine hood is removed during snapshot normalization.

That is the correct source for the subtraction: it retains the full concrete member suffix (`sase-8t.1--plan`) with only
the local machine hood stripped. Change the one line in `presented_clan_reference_name()`:

```python
    raw_name = self.agent_name or ""
    presented_name = self.presented_identity_name or raw_name   # was: self.presented_agent_name
    prefix = f"{raw_clan}."
    if raw_name.startswith(prefix):
        ...
```

With `presented_identity_name = "sase-8t.1--plan"`, the guard `"sase-8t.1--plan".endswith(".1--plan")` is True and the
method returns `sase-8t` — correct, and identical to the plain-member result, so the clan no longer splits.

Why this is the right fix (not a workaround):

- **Correct field by design.** `presented_identity_name` is documented as the hood/neighbor identity that keeps the full
  suffix; `presented_agent_name` is the display-shortened form that intentionally collapses family roots.
  `presented_clan_reference_name()` was reading the wrong one.
- **Performance-safe / no collision with `is.cdx`.** `presented_identity_name` is already normalized once per snapshot
  inside `refresh_presented_agent_name()` (called from `_agent_status_apply.py`). The fix adds **no**
  `MachineHoodIdentity.current()` call and no config I/O, so it cannot reintroduce the per-record identity cost `is.cdx`
  is removing, and touches none of its files.
- **Foreign hoods preserved.** For a foreign clan (e.g. `zeus.review` on `athena`), `strip_local_agent_name` leaves the
  name unchanged, `presented_identity_name` keeps the foreign prefix, the `endswith` subtraction still succeeds, and the
  foreign clan name is returned intact.

### Also harden the sibling method (same latent defect)

`presented_family_reference_name()` (directly below, in the same file) uses the identical `self.presented_agent_name`
subtraction pattern. It currently returns the correct value only because its `is_family_root_entry and presented_name`
fallback happens to return the already-shortened `presented_agent_name`. For consistency and to remove the same
fragility, apply the same change there — `presented_name = self.presented_identity_name or raw_name`. This is a
secondary hardening step, gated on the existing family/display tests continuing to pass (see Validation); if any
existing family test regresses, revert only this sibling change and keep the primary clan fix.

## Files to change

- `src/sase/ace/tui/models/agent.py`
  - `presented_clan_reference_name()`: read `self.presented_identity_name` instead of `self.presented_agent_name` for
    the `presented_name` local (primary fix).
  - `presented_family_reference_name()`: same one-line change (secondary hardening, gated on tests).

## Tests

Add a focused regression test. `tests/ace/tui/models/test_agent_tribe_summary.py` already has the right fixtures
(`_agent(...)` supports both `clan=` and `family=`/`role=`, and there is an existing
`MachineHoodIdentity("athena", ("athena", "zeus"))` pattern). Add a test that:

1. Builds a machine-qualified clan `athena.sase-8t` (generation e.g. `"gen"`) containing:
   - plain members `athena.sase-8t.2` (WAITING), `athena.sase-8t.land` (WAITING), and
   - a family-root member `athena.sase-8t.1--plan` (RUNNING, `family="athena.sase-8t.1"`, `role="plan"`) so it exercises
     the family-root `presented_agent_name` shortening.
2. Runs the **production order**: call `agent.refresh_presented_agent_name(identity)` on every row first (mirroring
   `_agent_status_apply.py`), then `project_clan_tree([...])`. (Note: the existing
   `test_machine_qualified_children_compact_against_presented_containers` projects before refreshing; do it in the
   production order here so the regression is actually exercised.)
3. Asserts:
   - Exactly **one** synthetic clan container is produced for this clan (the clan does not split).
   - The container's presented clan name is `sase-8t` — i.e. `presented_clan_reference_name()` / the tribe-summary unit
     label contains no `athena.` prefix.
   - Directly on a family-root `Agent` built with `agent_clan="athena.sase-8t"` and refreshed with the `athena`
     identity, `presented_clan_reference_name() == "sase-8t"` (unit-level assertion that pins the fixed method).
   - A foreign clan control (`zeus.review` on `athena`) still returns `zeus.review` unchanged.

Prefer asserting through `build_agent_tribe_summary_snapshot(...)` unit labels (as the neighboring tests do) plus the
direct method-level assertion, so both the user-visible symptom and the root cause are locked in.

## Validation

Run from the coding agent's own ephemeral workspace checkout (never a sibling workspace dir):

1. `just install` first — ephemeral workspaces may have stale dependencies.
2. Focused tests:
   `just test tests/ace/tui/models/test_agent_tribe_summary.py tests/ace/tui/widgets/test_agent_display_clan.py` plus
   the new test; confirm the new regression fails before the fix and passes after.
3. If the sibling `presented_family_reference_name()` change is kept, confirm the family display tests still pass
   (`tests/ace/tui/widgets/test_agent_display_clan.py`, `tests/ace/tui/models/test_agent_tribe_summary.py`, and any
   `family` tests under `tests/ace/tui/`). If any regress, drop only the sibling change.
4. `just check` and resolve every failure before completion (required for any file change in this repo).

## Acceptance criteria

- On the machine that owns a hood, a clan that contains an agent family renders a single container with the local
  machine hood stripped (e.g. `sase-8t`, never `athena.sase-8t`); the clan is no longer split into a qualified and an
  unqualified container.
- Plain-member clans, family display, and foreign (cross-machine) clan names are unchanged.
- No new `MachineHoodIdentity.current()` / config-discovery calls are introduced on the render path.
- A regression test reproduces the family-in-clan machine-hood leak and passes with the fix.
- `just check` passes.
