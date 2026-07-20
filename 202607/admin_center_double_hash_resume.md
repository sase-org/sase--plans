---
tier: tale
title: Resume the last Admin Center section with a repeated opener key
goal: 'Preserve the Admin Center''s lightweight home-first opening while letting users
  repeat its opener key (## by default) to resume the last section they used in the
  current ACE session, with clear discovery, safe key routing, and no I/O.

  '
create_time: 2026-07-20 13:36:27
status: done
prompt: 202607/prompts/admin_center_double_hash_resume.md
---

# Plan: Resume the last Admin Center section with `##`

## Context and product contract

The Admin Center deliberately opens on its lightweight landing page. That keeps the first `#` fast and gives the seven
sections a stable discovery surface, but it adds friction for users who repeatedly leave the modal to work in Agents,
Artifacts, or Axe and then return to the same Admin Center section.

Add a repeat-to-resume interaction with this precise contract:

- The first Admin Center opener key always opens a fresh home modal and mounts no working pane, even when a previous
  section is known.
- Repeating that same key while the home page is visible opens the last section that was successfully active in the
  current ACE process. With the default keymap this is `##`; if `ace.keymaps.app.open_config_center` is rebound, the
  repeated configured key remains the coherent equivalent.
- The remembered value is only a validated catalog tab ID. Pane instances, filters, selections, and loaded data remain
  scoped to one modal lifetime and are never retained across closes.
- History is in memory only. A new ACE process starts without a resume target; there is no startup read, background
  write, migration, or recreation of the removed `admin_center_tab.json` persistence.
- Until a section has been active, repeating the opener leaves the user on the landing page without constructing a pane.
  The inline landing guidance makes that state understandable instead of silently implying a target exists.

This is a shortcut into the existing lazy navigation path, not a second pane creation path. Direct-entry commands (Logs,
Tasks, Statistics, Updates, and so on) continue to open their requested section immediately and establish that section
as the next resume target once it is actually active.

## Interaction and visual design

Turn the landing page's existing single-line hint into a context-aware resume affordance without adding another focus
stop or changing the card hierarchy:

- With history, show the effective opener as a keycap followed by a concise, catalog-derived target such as
  `# resume Tasks`, styled with that section's accent, followed by the existing numbered/click, tab-cycle, and close
  hints.
- Without history, keep the same footprint but explain briefly that the opener resumes a section after the first visit.
  Do not invent a Config fallback, because that would blur “last active” semantics and unexpectedly load data.
- Derive the target label and accent from the immutable Admin Center catalog and format the key through the existing
  key-display helpers so custom bindings, aliases, and named keys are rendered consistently.
- Keep the hint non-focusable and one row at both 120x40 and 100x24. If the full wording cannot remain unclipped at the
  compact width, shorten the secondary navigation clauses rather than wrapping, truncating the target, or adding a
  scrollbar.

The local resume binding must not steal literal `#` input from a focused text editor or filter in a working pane. It is
a landing-page command: on home it must win over the app-level opener so the second key does not stack another Admin
Center modal; away from home the app-level opener must be suppressed for the already-open Admin Center while focused
inputs keep their normal character handling.

## State ownership and lifecycle

Keep the resume identity on the `AceApp` instance and initialize it explicitly to no target with the rest of the app's
in-memory presentation state. Pass a validated snapshot of that target, plus the effective opener binding, into each new
generic or direct-entry `ConfigCenterModal`.

Have the modal return its final successfully active catalog tab when it closes. The existing dismissal callback should
validate and record a non-null result, retain the older target when the user only viewed home or a requested pane failed
to mount, and still perform its current updates-indicator revalidation. This keeps ownership outside the modal and makes
all close routes share one state transition.

Within the modal, add a home-scoped resume action that delegates to `_schedule_switch()` and therefore inherits the
navigation lock, lazy factory, mount caching, visibility notifications, rollback, error toast, and focus behavior. Do
not set the app's last-tab state when a tab is merely requested; only the active tab returned after a successful switch
is eligible. Repeated or autorepeated opener events must remain idempotent and mount the target at most once.

Build the modal-local binding from the already-loaded effective `open_config_center` key. This reuses the existing app
keymap field rather than adding a second configurable action or duplicating `number_sign` in `default_config.yml`.
Retain a safe `number_sign` fallback for isolated modal tests or hosts without an ACE keymap registry. Add the minimal
app/screen action gating needed to prevent the underlying global opener from pushing a nested Admin Center while the
current screen is the Admin Center.

## Implementation surfaces

Update the focused surfaces only:

- `src/sase/ace/tui/actions/_state_init.py` for the session-local resume target.
- `src/sase/ace/tui/actions/base.py` for passing the resume context, consuming the modal result, and preserving
  update-indicator revalidation.
- `src/sase/ace/tui/app.py` only if global action gating is needed to prevent a nested modal after the screen-local
  binding declines an event.
- `src/sase/ace/tui/modals/config_center_modal.py` for the typed modal result, effective local binding, context-aware
  hint, and resume action routed through the existing serialized switcher.
- The existing Admin Center behavioral/action tests, visual fixtures, and user documentation in `docs/ace.md` and
  `docs/configuration.md`.

No Rust-core change is appropriate: this is presentation-only session state and key dispatch. Do not add a config schema
field, persistence module, durable file, extra home tab, alternate pane factory, or eager pane import.

## Behavioral and regression coverage

Extend the focused tests to lock down the complete lifecycle:

1. A fresh app has no remembered Admin Center section and initializes that state without disk, subprocess, or
   data-scaled work.
2. With no history, the first opener shows home and the repeated opener keeps the same single modal on home with zero
   concrete panes.
3. After visiting a section and closing, one opener still shows home; the immediate repeat opens exactly that section,
   mounts only its pane, preserves the underlying ACE tab, and never stacks a second modal.
4. Switching sections changes the next session's resume target, while closing from home does not clear an existing
   target.
5. Direct-entry actions establish their successfully mounted section as the next target. Construction/mount/switch
   failures retain the prior target and remain retryable through the existing rollback path.
6. Rapid repeated resume events are serialized and idempotent. The tab's factory runs once and the final header,
   switcher, focus, and visibility state agree.
7. A custom `open_config_center` binding is used both to open home and to resume, and the landing copy displays its
   effective key. No new keymap field or bundled-default entry appears.
8. `#` remains typeable in focused text-entry/filter widgets on working panes, and an unhandled opener key elsewhere in
   an already-open Admin Center cannot open a nested copy.
9. Home copy tests cover both no-history and resume-ready variants and confirm the target label/accent comes from the
   catalog.

Regenerate and inspect PNG coverage for both 120x40 and 100x24 home layouts in the no-history and resume-ready states.
Confirm the keycap, target, and remaining hints are legible, aligned, unwrapped, and free of clipping or scrollbars; all
non-home Admin Center snapshots should remain unchanged.

## Performance, validation, and documentation

Before implementation, capture the existing Admin Center open benchmark for empty and populated fixtures. After
implementation, rerun `tests/ace/tui/bench_admin_center_open.py` and require its structural guarantees: the first opener
mounts no panes, performs no store-dependent work, and remains fixture-size independent. The added state lookup, key
formatting, and hint render must be bounded in-memory work; no I/O or slow await belongs in the key, action, compose, or
render paths.

Update the Admin Center and global-keybinding documentation to explain the single-opener home / repeated-opener resume
model, its current-process lifetime, the no-history behavior, direct-entry participation, and custom-keymap equivalent.
Replace statements that currently say the previous section never reopens, while retaining the guarantee that the first
opener never does so automatically.

Run formatting and the focused Admin Center/action tests, the two-state home visual tests, the complete visual suite,
the focused benchmark, and finally the repository-required `just check` after `just install`. Inspect all changed PNGs
at native resolution and audit the final diff to ensure no persistence or unrelated keymap/config changes slipped in.

## Acceptance criteria

- Default `##` returns to the last successfully active Admin Center section in the same ACE session; a custom opener
  repeats equivalently.
- A single opener remains home-first and zero-pane, and a fresh process has no remembered target.
- No-history, failed-navigation, rapid-repeat, direct-entry, and close-only-home cases behave deterministically without
  nested modals or stale state.
- Working-pane text entry is not broken by the screen-local opener binding.
- The landing page makes the shortcut and exact destination obvious at both supported snapshot sizes without weakening
  its polished composition.
- Targeted tests, visual tests, benchmark invariants, and `just check` pass.
