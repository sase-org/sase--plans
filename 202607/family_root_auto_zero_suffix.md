---
create_time: 2026-07-08 14:11:08
status: wip
prompt: 202607/prompts/family_root_auto_zero_suffix.md
tier: tale
---
# Plan: Auto-assign `--0` suffix to the bare family-root member

## Problem

The Agents tab enforces an implicit invariant: **whenever a root/family entry has two or more child agent rows, each
child should display a distinct `--<id>` suffix.** This is naturally true for plan-chain families (`foo--plan`,
`foo--code`, …) because every member is born with a suffix.

It is violated in one dynamic case:

1. An agent is launched with the plain naming directive `%n:foo` (or `%name:foo`) → it is named `foo` with **no**
   `--<id>` suffix.
2. Later, a new family member is attached dynamically with the two-argument form `%n(foo, bar)` (or `%name(foo, bar)`) →
   a new agent `foo--bar` is created whose `agent_family` is `foo` and whose `parent_timestamp` points at the first
   agent.

Both agents now share the family name-root `foo`, so the TUI draws a `foo` banner with the two as sibling rows. But the
first row still renders its bare stored name `foo`, while the second renders `foo--bar`. The result (see
`sdd/research/`-style repro in the reported screenshot) is a family whose members are `foo` and `foo--bar` — the first
lacks the required `--<id>` suffix.

The desired behavior: **as soon as the second member is added, the first (bare) member should take the `--0` slot**, so
the family renders as `foo--0` and `foo--bar` — two distinct, suffixed rows.

## Root cause

- The main-tree row shows the agent's stored identity verbatim (the `agent_name` annotation in the row renderer,
  `src/sase/ace/tui/widgets/_agent_list_render_agent.py`, and `display_name` which is derived from `cl_name`). There is
  **no** normalization step anywhere that rewrites a bare family root to `foo--0` for display.
- The name-root banner is emitted only when a base name has **≥2 members**
  (`src/sase/ace/tui/models/agent_groups/_tree.py`). With one member (`foo` alone) the bare row is correct; the
  invariant only "turns on" once a second member appears.
- The `--0` slot is already **reserved** for exactly this member: `_reserved_agent_family_names`
  (`src/sase/plan_chain.py`) reserves both `{base}--plan` and `{base}--0`, and the `--@` child allocator skips them. So
  `--0` is the intended identity of the bare/root member; nothing ever assigns it today.

## Key design constraint discovered during research

Renaming a **running** agent's on-disk name is unsafe here. The first agent (`foo`) is typically still RUNNING when the
second member is attached (it is the parent). A running agent caches its own name (`SASE_AGENT_NAME`, set at its
startup) and its finalizer writes that name into `done.json` when it completes. An external rewrite of its
`agent_meta.json` name to `foo--0` would therefore be **reverted/contradicted** when the parent finishes (the registry
rebuild reads `done.json` too), resurfacing `foo`. On-disk metadata mutation of a live agent from another process also
races with the parent runner's own meta writes.

This constraint drives the recommendation below toward an **in-memory display normalization** rather than an on-disk
rename.

## Recommended approach — in-memory display normalization (no disk mutation)

Normalize the bare family-root member's **display identity** to `{base}--0` in the TUI model layer, computed on every
load/refresh from the already-loaded agent set. This mirrors the existing pattern where the codebase derives a family
member's display identity rather than storing it (`apply_workflow_child_identity_from_meta` derives a plan-root main
step's name as `{base}--plan`; `ensure_synthetic_planner_children` synthesizes logical family children). The
stored/registry name stays `foo`, so `%wait:foo` / `@foo` references and the name registry are untouched.

### Where

Add a post-load pass in the family-status apply stage, alongside the existing family wiring:

- New helper in `src/sase/ace/tui/models/_agent_status_family.py` (next to `ensure_synthetic_planner_children` /
  `has_family_followup_child` / `children_by_parent_timestamp`).
- Invoke it from `src/sase/ace/tui/models/_agent_status_apply.py`, in the same function that clears and rebuilds
  `followup_agents` and calls `ensure_synthetic_planner_children` — after the `children_by_parent` map is available so
  sibling awareness is present.

### What the pass does

For each loaded agent `a` that is a **bare family root**:

- `a` is a top-level row (not itself a family-member child) whose stored `agent_name` equals its own family base
  (`agent_family_name(a) == a.agent_name`), and
- `a` has **no** canonical plan-chain `role_suffix` and is not already a plan-chain root, and
- the family (same name-root) has **≥1 other member with a real `--<id>` suffix** — in practice detectable via
  `has_family_followup_child(a, …)` (a family-member follow-up exists) and/or a sibling sharing the name-root; this is
  exactly the ≥2-member condition that triggers the banner.

…assign, **in memory only**:

- `a.agent_name = f"{base}{AGENT_FAMILY_SEPARATOR}0"` (i.e. `foo--0`; equivalently
  `agent_family_phase_name(base, "--0")`),
- `a.role_suffix = f"{AGENT_FAMILY_SEPARATOR}0"`,
- `a.agent_family = base`,
- `a.agent_family_role = "root"` (the family root; see role note below).

Grouping is preserved: `_grouping_name` prefers `agent_family` (`foo`), and even by name
`agent_family_base("foo--0") == "foo"`, so the banner label stays `foo` and both members still sit under it. The only
visible change is the first row now reads `foo--0`.

### Guards / edge cases

- **Single-member family** (only `foo`, no attached member): do nothing — the bare row is correct. The trigger requires
  an existing suffixed sibling / follow-up.
- **Explicit `{base}--0` already present** (someone used `%n(foo, 0)`): do not create a duplicate `foo--0`. Skip
  normalization for that family (leave the bare root as-is) to avoid two `foo--0` rows; this is rare because `--0` is a
  reserved slot the `--@` allocator skips.
- **Plan-chain families** (`foo--plan`, `foo--code`): unaffected — a plan-chain root is promoted to a workflow and its
  main step already derives `foo--plan`, so no bare-base member exists. The "no canonical `role_suffix` / not
  plan_chain_root" guard prevents mis-normalizing these.
- **Uniqueness**: at most one agent can literally be named `foo` (registry names are unique), so there is never more
  than one bare-base candidate per family.
- **Role label note**: `--0` parses (in `plan_chain.py`) as a `root_question` → role `"q"`, so the _context/tools_ side
  panels' compact label may read `q`. This is cosmetic, confined to secondary panels, and pre-existing to how `--0` is
  classified; the main row simply shows `foo--0`. If this is undesirable, the implementer should confirm the
  compact-label path (`src/sase/ace/tui/agent_context_members.py`) renders acceptably and adjust only if needed.

### Consistency across views

Because the normalization sets `a.agent_name` on the in-memory model, the row badge, and any view that reads
`agent.agent_name` (e.g. the detail panel's Name field), should all show `foo--0` consistently. Verify the detail panel
does not separately re-read the on-disk `agent_meta.json` name; if it does, route it through the same normalized model
value so the display is coherent.

## Alternatives considered (and why not chosen)

- **Option A — on-disk rename `foo` → `foo--0` at attach time** (in the family-attach flow,
  `src/sase/agent/family_attach.py` / `src/sase/axe/run_agent_directives.py`). Most literally "assigns" the suffix and
  would make `%wait:foo--0` / `@foo--0` resolve. **Rejected as the default** because renaming a _running_ parent is
  unsafe: the parent's finalizer rewrites `done.json` with its cached startup name (`foo`), reverting the change and
  risking a `foo`/`foo--0` split after a registry rebuild; it also requires coordinated updates to the name registry
  (claim new + delete old under the allocation lock), the parent's `%name` prompt directive (`set_prompt_name`),
  `done.json`, and the artifact index, plus a read-modify-write race with the live parent. Viable only for
  already-completed parents and only if reference-resolution to `foo--0` is explicitly required.
- **Option B — on-disk `role_suffix` mutation (keep name `foo`) + display derivation.** Mirrors the `sase questions`
  precedent (`update_meta_suffix(dir, "--0")` for the first family agent). Lighter than A but still mutates a live
  parent's meta from outside (racy; the parent runner may clobber the injected `role_suffix`), and the detail/Name view
  still shows `foo` unless display also derives. The in-memory Option C is strictly safer and needs no disk write.

If the user specifically wants `foo--0` to be a _real_ reference (resolvable by `%wait`/`@` and shown in the detail
panel as the canonical name), we escalate to a hardened Option A limited to completed parents (with an explicit
mechanism to also update a running parent's in-process name); otherwise Option C fully satisfies the reported display
invariant.

## Testing

- **Unit (model/apply pass)** — new tests near `tests/ace/tui/models/` and `tests/test_dynamic_agent_family_attach.py`
  (which already has `_artifact_record`, `_patch_attach_snapshot`, `_write_agent_artifact` fixtures):
  - Bare `foo` (running) + attached `foo--bar` ⇒ after the pass, `foo`'s display identity is `foo--0`; both group under
    name-root `foo`; banner shows `foo` with children `foo--0`, `foo--bar`.
  - Single member `foo` only ⇒ stays bare `foo` (no `--0`).
  - Explicit `foo--0` already present ⇒ no duplicate; bare member left unchanged.
  - Plan-chain family (`foo--plan` + `foo--code`) ⇒ unchanged (no spurious `--0`).
  - Reference identity unchanged: the on-disk/registry name stays `foo` (the pass does not write to disk).
- **Grouping/render** — extend `tests/ace/tui/widgets/test_agent_list_grouping*.py` /
  `tests/ace/tui/models/test_agent_groups_*` to assert the two rows render with distinct `--<id>` suffixes under the
  `foo` banner.
- **Visual/PNG snapshot** — if a snapshot covers a multi-member family tree, add/update one showing `foo--0` +
  `foo--bar` (use `--sase-update-visual-snapshots` only for the intentional change).

## Risks / open questions

- **Role classification of `--0`** (`q`) surfacing in secondary panels — cosmetic; confirm during implementation (see
  role note).
- **Detail-panel Name coherence** — confirm the detail view reflects the normalized `foo--0` (see Consistency section).
- **Scope**: this is display-only. It intentionally does **not** make `foo--0` a resolvable reference. Flagging so the
  user can redirect to Option A if a real rename is required.

## Out of scope

- Changing how `%n(parent, suffix)` resolves the parent, or the `--@` allocation sequence.
- Any Rust-core change: family/name-root grouping is pure Python; the fix stays in the Python TUI model layer.
