---
tier: tale
title: Hint-selected agent panel folding
goal: 'Agents-tab users can press ,H, select one or more numbered panel hints with
  the TUI''s familiar single/range syntax, and atomically toggle those panels between
  expanded and collapsed states with stable focus, durable state, and polished visual
  feedback.

  '
create_time: 2026-07-16 16:56:06
status: done
prompt: 202607/prompts/hint_selected_agent_panel_folding.md
---

# Plan: Hint-selected agent panel folding

## Product intent

Plain `H` and `L` remain the fast, directional commands for collapsing or expanding the focused Agents panel. Add
configurable leader command `,H` as the set-oriented companion: it reveals numeric chips on every eligible panel title,
opens a compact `Panels:` hint input, and toggles exactly the panels named by the submitted hints.

This command should feel like the existing hint-driven file and hook actions, not like a second navigation mode. It uses
numeric hints because users need to submit sets and ranges, while the apostrophe jump mode keeps its optimized
single-keystroke alphabet. It is available only on the Agents tab when tag panels are split and at least two panels
exist; otherwise it explains why there is nothing to select without altering fold state.

## Interaction contract

1. Register an Agents-only leader action with default key `H`, surface it as `,H` in the leader footer, Agents help, and
   command catalog, and preserve the normal configurable-keymap and repeat-last-command behavior.
2. On entry, snapshot the current rendered panel order and assign `1..N` to stable `PanelKey` values, including the
   untagged `None` key. Paint a bright, compact `[N]` chip before each panel's existing chevron/name/count title.
   Expanded and collapsed panels are both selectable. The snapshot order is the order visible when the mode opens, even
   though applying a toggle may move collapsed panels into or out of the collapsed-last partition.
3. Mount the established `HintInputBar` in the Agents detail column with a `Panels:` label, a concise example such as
   `1-3 or 1 3 (toggle panels)`, and the standard `[Esc] cancel` treatment. Re-entering the command while the bar exists
   focuses the existing input rather than mounting a duplicate.
4. Accept the same whitespace-delimited grammar used by other numeric TUI hints: individual positive numbers (`2`),
   inclusive ascending ranges (`2-5`), and any combination (`1-3 5 7-8`). Preserve first-seen order while deduplicating
   overlaps. Do not introduce comma syntax. Empty, malformed, descending, or out-of-range input is rejected clearly and
   atomically so the user can correct it; no panel changes on a partially invalid submission.
5. A valid submission toggles each selected panel according to its own current state. Therefore a mixed selection can
   expand collapsed panels and collapse expanded panels in one operation. Show a short confirmation summarizing how many
   panels expanded and collapsed, then remove the input and transient chips. Escape removes only the transient UI and
   leaves all fold state alone.

## State and mutation design

- Give panel-fold selection dedicated hint-to-`PanelKey` and inverse mappings; do not overload file-path mappings or
  apostrophe jump targets. Use the shared transient hint-mode activity gate so background refreshes cannot silently
  renumber titles while the user types, and centralize teardown for submit, cancel, tab/layout changes, and failed
  mounting.
- Factor the common numeric single/range expansion into a reusable parser in the existing hint utilities, then have
  panel selection and current numeric consumers share it without changing their action-specific suffix semantics. The
  parser should return ordered unique selections plus enough diagnostics to distinguish malformed tokens from
  valid-but-unavailable hint numbers.
- Before mutation, resolve the complete selection against both the entry snapshot and the live split-panel collection.
  If layout or membership has changed despite the activity gate, abort the whole operation with a retry message. Once
  validated, compute the new collapsed-key set in memory, then invalidate panel caches and synchronize/render once
  rather than calling the single-panel `H`/`L` actions repeatedly.
- Preserve panel focus by stable key across the collapsed-last reorder. If the focused panel is toggled closed, retain
  its backing agent/detail context as plain `H` does; if it is toggled open, land on its first rendered navigation stop
  as plain `L` does. Toggling only non-focused panels must not move the cursor, clear the current group/attempt, or
  acknowledge unread agents. Nested group/workflow folds remain untouched in every case.
- Record the final state of every changed panel through the existing fold intent journal and coalesced off-thread
  persistence path, including the pre-load replay case. A multi-panel operation should produce one visual refresh and no
  synchronous disk work on the key or submission path.

## Rendering and lifecycle

Generalize the panel-title hint plumbing just enough to render the numeric selection chips on all panel headers while
retaining the existing collapsed-only apostrophe jump hints. Keep the established yellow chip styling, title metrics,
focus border, two-row collapsed height, and dynamic width calculation intact, including multi-digit hints.
Selection-mode entry and cancellation should use the selective panel refresh path so showing or removing chips does not
rebuild unrelated rows; the submitted fold transition may use the established single coalesced list-change refresh
because panel contents, heights, and order change.

Ensure transient state cannot leak: teardown clears maps and activity flags before repainting, removes the input safely
if mounted, restores the normal footer/detail presentation, and lets deferred auto-refresh work resume. Keep apostrophe
jump mode and panel selection mutually exclusive so one title never receives competing hint namespaces.

## Verification

- Add parser coverage for singles, disjoint and overlapping ranges, ordered deduplication, whitespace,
  malformed/descending tokens, and unavailable numbers; retain existing file/hook hint behavior.
- Cover default and overridden keymap loading, `,H` dispatch, repeat behavior, Agents-only catalog availability, leader
  footer/help copy, and no-op messaging for merged or single-panel layouts.
- Exercise selection-mode entry/cancel/refocus and numeric title rendering for expanded, collapsed, untagged, and
  multi-digit panels. Verify stable mapping uses the visible entry order and that title/width state is restored on exit.
- Exercise single, multiple, range, overlapping, and mixed-state submissions. Assert atomic rejection for invalid/stale
  selections; exactly one fold/display transition for valid bulk input; stable focused key across reorder; correct
  focused-panel collapse/expand anchoring; no movement for non-focused changes; preserved nested folds/unread state; and
  correct persistence intents before and after fold-state loading.
- Add or update an ACE PNG interaction snapshot showing the polished selection state with numbered title chips and the
  `Panels:` input, and assert the final mixed toggle layout. Run `just install`, focused unit/integration tests,
  `just test-visual` (updating the intentional golden through the sanctioned snapshot flag), and the required
  `just check` quality gate.

## Boundaries

Do not change plain `H`/`L`, apostrophe jump semantics, the collapsed-last ordering rule, or merged-panel behavior. Do
not add mouse-only affordances, comma-separated hint syntax, per-panel animation, or a second persistence format. This
tale ends when the keyboard workflow, transient visuals, atomic state transition, durability, and regression/visual
coverage all agree.
