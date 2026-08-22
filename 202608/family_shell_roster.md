---
tier: tale
title: Family shell roster with monitor navigation
goal: "Agent-family metadata presents every agent and monitor shell as one coherent,
  beautiful FAMILY SHELLS roster whose numeric shortcuts reliably select the exact
  rendered shell.

  "
size: medium
proposed_by: bbugyi200.athena.0ap
create_time: 2026-08-22 12:44:18
status: wip
---

# Plan: Family shell roster with monitor navigation

## Outcome

When a family container is selected in ACE's Agents tab, its metadata panel will show
`FAMILY SHELLS`, not `FAMILY MEMBERS`. The roster will contain every concrete shell in
causal execution order: LLM-backed agent shells and proc-backed monitor shells. Each
visible row will receive the same zero-based numeric chip that selects it, including a
monitor nested beneath the shell that started it. Selecting an individual family shell
will continue to show the other shells in that family, under the same terminology.

The presentation should make shell kinds legible at a glance. Agent rows keep their role
and model treatment. Monitor rows use the monitor gear vocabulary, their effective
monitor status, and a concise monitor-specific descriptor (prefer the label, then a
bounded one-line command summary, then a neutral command fallback) instead of pretending
that a monitor has an LLM model. Counts, status glyphs, durations, folds, and shared
number alignment remain visually consistent with the existing roster language.

## Current behavior and constraints

ACE correctly preserves a monitor's durable `parent_timestamp` and renders the monitor
under the exact family shell that started it. That ownership must not be rewritten.
However, the current family-roster projection only combines the family root's direct
`runtime_children` and `followup_agents`. A realistic mid-family monitor is therefore
reachable in the rendered tree but absent from the family roster, the `Shells:` metadata
lanes, the family conversation, the family-container backlink, and the roster's
published digit map. Direct-root monitor fixtures hide this gap.

The fix must remain a pure in-memory projection. Rendering and digit keystrokes must not
read files, scan stores, invoke subprocesses, or rebuild the agent list. Monitor shells
remain proc shells rather than agents: adding them to the shell roster must not inflate
agent counts, runner counts, family completion counts, or any agent-only projection.

The shared member-roster renderer and its internal `members` section/fold identifier
also serve clan and tribe member lists. Keep that generic machinery and stable fold
identity intact; change the family-specific user language and projection rather than
globally renaming unrelated member concepts or resetting users' fold state.

## Design and implementation

### 1. Establish one canonical family-shell projection

Add an explicitly named, pure projection in
`src/sase/ace/tui/models/agent_family_members.py` for ordered concrete family shells. It
should retain the existing aggregate-plan semantics: a loaded concrete planner step
replaces its workflow aggregate, a rename-on-attach root remains the first real shell,
and synthetic planners, non-agent workflow steps, and parallel-family rows are excluded.

Extend that projection through descendant `runtime_children` and `followup_agents` so a
monitor nested under any family shell is included immediately after its causal starter,
while later continuations retain their established order. Traverse both relationship
collections because loaded shapes expose overlapping but not always identical links;
deduplicate by durable row identity, guard malformed cycles by object identity, and keep
the already-normalized child ordering rather than inventing a second timestamp sort.
This makes the output deterministic, linear in the loaded family subtree, and safe for
the render and keystroke paths.

Keep agent-only consumers explicit. Either retain the current concrete-agent projection
or derive a clearly named agent-only view by filtering monitor shells, but do not let
the new shell projection silently change agent/status/completion counts. Switch only
shell-facing consumers to the canonical shell sequence: family roster entries,
per-family `Shells:` metadata, family conversation phases and hint inputs, family
neighbor suppression where duplicate shell rows would otherwise leak through, and
family-container attachment. The attachment pass must point nested monitors back to the
family container without changing their immediate rendered-tree parent.

### 2. Present a beautiful `FAMILY SHELLS` roster

Rename the family-specific visible heading to `FAMILY SHELLS` everywhere it appears,
including family-container panels and sibling rosters on selected shell panels. Update
family-specific overflow and cross-reference copy such as “see FAMILY SHELLS” and “also
listed under FAMILY SHELLS”; leave clan/tribe `MEMBERS` wording untouched.

Build roster entries from the canonical shell sequence. Continue excluding the selected
shell from its own sibling roster, but include every other agent or monitor shell and
preserve the family-name suffix in the heading. Render a monitor with the established
gear iconography and monitor status bucket. Use a short, normalized monitor label or
command summary in the roster's descriptor slot so collapsed rows remain informative
without allowing multiline or unbounded command text to damage alignment. The expanded
digest may expose existing workspace/timestamp facts, but it must not perform new I/O.

Keep the generic roster's number chips, rule, accent, indentation, fold behavior, and
100-entry cap. The heading count must equal the shell entries actually represented by
the family roster, including monitors and excluding aggregate/synthetic rows.

### 3. Make numeric shell selection exact and self-validating

Publish the family jump map from the exact ordered entries that were rendered. A monitor
therefore consumes the next numeric slot just like an agent shell; single- and
double-digit numbering continue to share one allocator with any visible neighbor rows.
Validate family jump targets against the same canonical shell projection, not the old
direct-member view, so rendering and keystroke acceptance cannot drift.

Reuse the existing reveal pipeline for the selected target. Its monitor-aware fold gate
already climbs through a nested starter to the outer family container, so a digit should
expand only the necessary folds and land on the real monitor row without rerooting it.
Cover navigation from both a selected family container and an individual family shell;
the latter should list siblings, never itself.

Make family-owned numeric affordances say `shell`: the normal footer chip, buffered
two-digit mode, and family-specific stale/not-ready/no-target messages. Preserve
`member` for clan and tribe rosters and `neighbor` for neighbor-only maps. No new
configurable key is required—the feature assigns the existing dynamic `0-9` direct-jump
keys to the expanded set of rendered family shells.

### 4. Prove realistic monitor topology and presentation

Add focused model tests for direct and nested monitors, including a monitor started by a
mid-family continuation, overlapping runtime/follow-up links, stable causal order,
identity deduplication, cycle safety, and monitor exclusion from concrete agent counts.
Assert that family-container attachment reaches nested monitors while the monitor's
immediate starter parent remains unchanged.

Update renderer tests to require `FAMILY SHELLS`, reject stale `FAMILY MEMBERS` family
copy, verify monitor-specific kind/descriptor/status output, confirm the count includes
the monitor, and confirm selected shell panels exclude only themselves. Update numeric
navigation and footer tests to prove the monitor receives the displayed number, the
number selects that exact nested monitor, two-digit allocation still works, and stale
family maps report shell-specific language. Retain clan, tribe, neighbor, fold, and
agent-count regressions to protect shared behavior.

Replace the direct-root visual fixture with the production topology in which the monitor
is parented to the shell that started it. Add or update a PNG snapshot at a viewport or
scroll position that visibly captures the `FAMILY SHELLS` heading, an agent-shell row, a
monitor row with its gear/status/descriptor, and the monitor's numeric chip. Inspect the
golden before accepting it for alignment, wrapping, color hierarchy, and footer wording.

## Verification

Run `just install` before repository checks. Exercise the focused model, family-render,
member-jump, footer, fold/reveal, and monitor-root projection tests while iterating. Run
the intentional snapshot update with
`just test-visual -- --sase-update-visual-snapshots`, inspect the generated family-panel
goldens and diff artifacts, then rerun `just test-visual` without the update flag for
exact equality. Finish with `just check`; if its selector escalates or reports unusual
coverage, use `/sase_monitor` for `just check-full` as required by the repository.

Acceptance is complete when direct and nested monitor shells appear exactly once in
`FAMILY SHELLS`, their displayed numbers navigate to their real rows, family-specific UI
copy consistently says shell, agent-only counts remain unchanged, the monitor stays
nested under its starter, visual snapshots are intentionally accepted, and all focused
and repository checks pass.
