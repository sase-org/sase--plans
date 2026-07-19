---
tier: tale
title: Restore SASE Admin Center numbered keymaps
goal: 'Admin Center digit shortcuts are handled by the foreground modal even when
  an Agents-tab clan or family container remains selected behind it, while numbered
  member jumps continue to work on the main Agents screen.

  '
create_time: 2026-07-19 07:25:19
status: done
prompt: 202607/prompts/restore_admin_center_digit_keymaps.md
---

# Plan: Restore SASE Admin Center numbered keymaps

## Context and root cause

The Admin Center owns `1`–`7` as numbered tab shortcuts and reserves the remaining digits as no-ops. Those bindings are
intentionally non-priority so focused inputs can still accept digits when appropriate; the XPrompts filter, for example,
reserves digits while empty but allows values such as `bug2` once filter text exists.

The `sase-6w.3` member-roster work added an app-level digit handler in `EventKeyboardMixin`. Its selected-container
check only considers the hidden ACE tab state, so it can still see a clan or family selected behind a modal. Textual
lets the unhandled key bubble from the Admin Center to the app before dispatching the modal's non-priority binding. The
member-jump handler then claims and stops the digit, preventing the Admin Center action from running. Existing Admin
Center tests miss the collision because they open the modal without a selected member-roster container. A
production-shaped reproduction confirmed that pressing `3` over such a container invokes the member handler, returns
`True`, and leaves the Admin Center on its original tab.

## Implementation

Make numbered member navigation respect foreground screen ownership in the member-jump dispatch path. When a Textual
modal is active, the member-jump handler must return without consuming the key so the modal or its focused widget can
apply its own binding. If a two-digit member jump was pending when a modal became active, clear that buffer without
trying to refresh the hidden Agents footer; otherwise it could survive the modal and consume a later main- screen key.

Keep the guard local to member-jump navigation rather than promoting Admin Center digit bindings to priority. That
preserves the established XPrompts input contract and prevents this background Agents feature from interfering with any
modal that legitimately owns digits, not only Admin Center. The main Agents screen's one- and two-digit roster behavior,
stale-map validation, fold reveal, and jump history remain unchanged. The guard must stay entirely in-memory and perform
no work beyond inspecting the active screen and clearing session state.

## Regression coverage

Add an integration regression around the real `AceApp` and `ConfigCenterModal` that installs a selected clan or family
container behind the modal, presses a numbered Admin Center key, and verifies that the modal switches tabs instead of
the member-jump handler consuming the key. Cover the pending two-digit state as part of this scenario and assert it is
cleared so no stale member-jump state remains after modal input.

Retain and run the existing contracts that prove:

- valid and out-of-range Admin Center digits behave correctly;
- an empty XPrompts filter reserves numeric tab keys while a non-empty filter accepts digits as text;
- member-roster one- and two-digit jumps still work when the main Agents screen owns input.

## Validation

Run `just install` before repository checks, then exercise the focused suites for Admin Center tabs, XPrompts browser
key handling, and member-jump navigation. Re-run those focused tests after any correction. Finish with `just check` as
the repository-wide required gate, including formatting, lint/type checks, unit tests, and visual snapshot validation.
No default keymap configuration or Rust-core changes are expected because this is a presentation-only event-ownership
correction and does not change the bindings themselves.
