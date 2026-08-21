---
tier: tale
title: Align Ctrl+G Ctrl+X with mini-xprompt targeting
goal:
  The control-key alias opens or retargets a mini-xprompt consistently in INSERT and
  NORMAL modes while whole-stack save remains on uppercase X.
size: small
proposed_by: bbugyi200.athena.094
create_time: 2026-08-21 08:41:19
status: done
---

- **PROMPT:**
  [prompts/202608/ctrl_g_ctrl_x_mini_xprompt.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/ctrl_g_ctrl_x_mini_xprompt.md)
- **AGENTS:**
  - [bbugyi200.athena.094](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.094.md)
- **COMMITS:**
  - [e91b9f8](https://github.com/sase-org/sase/commit/e91b9f83acba631f36d0d74b3e8a6e19ae63d51e)
    — fix(ace): route ctrl-g ctrl-x to mini xprompt

# Align Ctrl+G Ctrl+X with mini-xprompt targeting

## Goal

Make `<Ctrl+G><Ctrl+X>` in the ACE prompt input behave exactly like the existing
mini-xprompt targeting commands: `<Ctrl+G>x` in INSERT mode and `gx` in NORMAL mode.
Keep `gX` and `<Ctrl+G>X` as the only prompt-prefix routes to the unified whole-stack
xprompt/snippet save panel.

This deliberately revises the compatibility choice made by closed epic `sase-rl`. That
epic placed the `ctrl+x` continuation on uppercase `X`; the requested behavior instead
groups it with lowercase `x`:

| Prompt state | Mini-xprompt target             | Unified whole-stack save |
| ------------ | ------------------------------- | ------------------------ |
| INSERT       | `<Ctrl+G>x`, `<Ctrl+G><Ctrl+X>` | `<Ctrl+G>X`              |
| NORMAL       | `gx`, `<Ctrl+G><Ctrl+X>`        | `gX`, `<Ctrl+G>X`        |

The remap applies whether the mini-xprompt action opens a fresh name panel or retargets
an existing mini pane. It retains the action's existing prompt-mode guards, auxiliary
draft guard, captured origin pane, and focus/mode restoration behavior.

## Implementation

1. In the prompt input's single declarative `g`-prefix binding table, move the `ctrl+x`
   `Ctrl+G`-only alias from the uppercase `X` whole-stack-save binding to the lowercase
   `x` mini-xprompt binding. Do not add a special case to the INSERT/NORMAL key
   handlers: both modes already canonicalize a real terminal control character as
   `ctrl+x` and dispatch through this table, which also generates the transient hint
   metadata.
2. Update focused keymap tests to lock the complete routing contract:
   - direct dispatch of `ctrl+x` with `via_ctrl_g=True` calls the mini-xprompt request,
     while bare `ctrl+x` remains unclaimed;
   - the `^G` hint entry renders `^Gx / ^G^X` for mini-xprompt targeting, while the
     uppercase whole-stack entry has no `^X` alias and the bare `g` hint surface remains
     unchanged;
   - `<Ctrl+G><Ctrl+X>` posts one mini-xprompt request, never a whole-stack save
     request, from both INSERT and NORMAL modes, preserving the draft, current Vim mode,
     and cleared prefix state exactly as `<Ctrl+G>x` does;
   - `gx`, `<Ctrl+G>x`, `gX`, and `<Ctrl+G>X` retain their existing behavior.
3. Replace the unified-save integration regression built around
   `<Ctrl+G><Ctrl+X><Ctrl+X>` with the still-supported sequence `<Ctrl+G>X`, then
   `<Ctrl+X>` inside the opened panel. This preserves coverage that the panel can switch
   to snippet mode without implying that the remapped control continuation opens that
   panel.
4. Update the prompt help registry and its assertions so the control alias is grouped
   with `gx` / `<Ctrl+G>x`, and the `gX` / `<Ctrl+G>X` row describes only unified save.
   Update every user-facing reference in `docs/ace.md`, `docs/prompt.md`, and
   `docs/xprompt.md`: describe `<Ctrl+G><Ctrl+X>` as the mini-xprompt route, remove the
   obsolete three-key shortcut, and direct users who want snippet mode through `gX` or
   `<Ctrl+G>X` followed by the panel-local `<Ctrl+X>` toggle.

## Boundaries

- Do not change the mini-xprompt modal, pane lifecycle, persistence behavior, or the
  unified save panel's internal `<Ctrl+X>` mode toggle.
- Do not change NORMAL-mode standalone `<Ctrl+X>` number decrementing, bare Vim `gx`, or
  unknown-prefix fallthrough outside the existing declarative routing contract.
- Do not introduce configurable keymap fields or edit `src/sase/default_config.yml`;
  these are hard-coded prompt-local continuations. No Rust-core change or feature flag
  is needed for this focused TUI keymap correction.
- Do not update PNG goldens unless a focused visual test proves a real intentional pixel
  change. The existing snapshot exercises the bare `g` surface, whose aliases do not
  render; the changed `^G` alias presentation is covered by textual hint assertions.

## Verification

1. Run `just install` before tests, as required for an ephemeral SASE workspace.
2. Run the focused prompt-prefix entry, lifecycle, and routing tests; the unified save
   action test containing the panel-local `<Ctrl+X>` toggle; and
   `tests/test_keymaps_display_help.py`.
3. Search the source, tests, and docs for stale `Ctrl+G Ctrl+X`, `^G^X`, and
   `<Ctrl+G><Ctrl+X><Ctrl+X>` descriptions, allowing only statements that identify the
   control continuation as a mini-xprompt alias.
4. Run the repository-required `just check` and resolve any failures caused by this
   change. If selection unexpectedly broadens to the full suite, follow the repository's
   monitored `just check-full` workflow rather than running it inline.

## Acceptance criteria

- In INSERT and NORMAL modes, `<Ctrl+G><Ctrl+X>` opens or retargets one mini-xprompt
  pane through the same request path as `<Ctrl+G>x` / `gx`; it never emits a unified
  whole-stack save request.
- `gX` and `<Ctrl+G>X` still open the unified xprompt/snippet save panel, whose own
  `<Ctrl+X>` toggle still switches to snippet mode.
- Transient `^G` hints, static help, tests, and documentation all group the control
  alias with lowercase `x` and contain no obsolete three-key shortcut.
- Focused tests and `just check` pass without unrelated source, configuration, or PNG
  snapshot changes.
