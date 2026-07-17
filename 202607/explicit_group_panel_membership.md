---
tier: tale
title: Remove implicit name-based agent panel assignment
goal: Agent names no longer influence Agents-tab panel membership; ordinary launches
  join a tag panel only through an explicit `%group` assignment.
create_time: 2026-07-17 14:37:29
status: done
prompt: 202607/prompts/explicit_group_panel_membership.md
---

# Plan: Remove Implicit Name-Based Agent Panel Assignment

## Context

Agents-tab side panels are keyed by the tag loaded onto each agent. An ordinary launch with `%group:<tag>` records that
tag in `AgentInfo`, `agent_meta.json`, and (when the launch has a ChangeSpec identity) `agent_tags.json`, which gives
the panel model one consistent assignment to render. TUI tag edits follow the same contract by rewriting the persisted
prompt with `%group` and synchronizing the metadata and tag store.

The launcher currently has a second, implicit source of panel membership. After resolving the final agent name, it reads
existing tag values and treats an exact name or dotted name prefix as a group match. For example, if `foo` is an
existing tag, an agent named `foo.child`, `foo.w0`, or `foo.f0` is assigned to the `foo` panel without `%group:foo`. The
longest matching dotted tag wins, and that inferred result is written into all of the same metadata and persistence
surfaces as an explicit directive. This couples agent naming and dependency-derived names to panel placement.

## Implementation

### Make `%group` the ordinary-launch source of truth

Update `src/sase/axe/run_agent_directives.py` so the effective launch tag comes directly from `directives.tag` and is
never inferred from the resolved agent name or the current contents of `agent_tags.json`.

- Preserve final name allocation, name templates, wait/fork-derived names, family metadata, and name claiming exactly as
  they are; these values must no longer participate in tag selection.
- Continue writing the tag to `AgentInfo` and `agent_meta.json` when `%group` is present, and omit tag metadata when it
  is absent.
- Keep the existing `cl_name` guard and atomic explicit-tag persistence path, but call it only for a real `%group`
  directive. A launch without `%group` must not read or mutate the tag store merely because its name resembles an
  existing panel tag.

This is a launcher-boundary cleanup. Directive parsing, tag validation, panel construction, tag-store loading, TUI tag
editing, and intentional non-name-based system tags (such as specialized review-runner tags) remain unchanged.

### Remove the obsolete name-matching API

Delete the exact/dotted-prefix matcher and the atomic "update from existing name group" helper from
`src/sase/ace/agent_tags.py`, along with imports that only support them. Retain the general load/save/set/unset and
explicit atomic update APIs used by `%group`, TUI edits, and system-managed tags.

No migration should rewrite existing `agent_tags.json` entries or historical `agent_meta.json` files: implicit and
explicit assignments were deliberately persisted in the same shape, so their provenance cannot be recovered safely. The
behavioral guarantee applies to new launches after the change; existing user-managed panel assignments remain intact and
can still be cleared through the normal TUI tag action.

## Tests

Update `tests/test_agent_tags.py` to remove unit tests for the deleted name matcher and implicit persistence helper
while retaining coverage for explicit atomic tag updates and the tag store's validation/round-trip behavior.

Update `tests/test_axe_run_agent_phases_tags.py` around the launcher boundary:

- Keep an explicit `%group` regression proving that it writes `AgentInfo.tag`, `agent_meta.json["tag"]`, and the
  identity's persisted tag even when the agent name resembles a different existing tag.
- Replace the current positive auto-grouping cases with negative regressions for explicit dotted names, wait-derived
  names, fork-derived names, and planned/template-resolved names. Seed matching (including nested) tags, assert name
  resolution itself is unchanged, and assert every no-`%group` launch returns `tag is None`, omits the metadata tag, and
  leaves the seeded tag store unchanged.
- Retain a no-existing-tag case if it still adds distinct coverage; otherwise consolidate it into the stronger seeded
  regressions so the tests emphasize that the absence of `%group`, not the contents of the tag store, controls panel
  membership.

## Verification and acceptance criteria

Run focused coverage first:

```bash
uv run pytest tests/test_agent_tags.py tests/test_axe_run_agent_phases_tags.py
```

Then install the workspace dependencies as required and run the repository-wide gate:

```bash
just install
just check
```

The change is complete when explicit `%group` behavior is unchanged, all tested name-allocation paths remain intact, and
a launch without `%group` produces no new tag metadata or tag-store assignment regardless of any exact or dotted
relationship between its name and an existing agent-panel tag.
