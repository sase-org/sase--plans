---
tier: tale
title: Restore Ctrl+I as the pager's advertised forward-history key
goal: The pager advertises Ctrl+I for forward history everywhere while retaining the
  working Tab compatibility binding.
size: small
proposed_by: bbugyi200.athena.0ii
status: done
---

# Restore Ctrl+I as the pager's advertised forward-history key

## Context

The pager's forward-history fix in commit `0523874af` made the action respond to both
`tab` and `ctrl+i`, which is necessary because terminals and Textual hosts may deliver
that physical chord under either key name. The same change also promoted `<tab>` in the
pager's binding metadata, sticky footer, expanded trail band, help sheet, and user
documentation. The dual binding works and must remain intact, but the user-facing UX
should advertise Ctrl+I again.

## Outcome

Every pager-owned description of forward-history navigation presents Ctrl+I, using the
surface's established notation (`^I` in compact chrome, `ctrl+i` in the key sheet, and
`Ctrl+I` in prose), while both `tab` and `ctrl+i` continue to invoke `trail_forward`.
Pressing Tab must still be consumed by the pager rather than leaking to a priority host
binding, including when no forward history exists.

## Implementation

1. In `src/sase/pager/screen.py`, retain the compound `tab,ctrl+i` binding for
   `trail_forward`, but change its explicit `key_display` from `<tab>` to `<ctrl+i>` so
   binding-derived UX names Ctrl+I. Do not change action dispatch, focus handling,
   modal/search guards, or the priority relationship with host bindings.
2. Restore Ctrl+I-oriented labels on every pager-owned runtime surface:
   - use `^I forward` in the availability-driven footer in `src/sase/pager/_chrome.py`;
   - use `^I forward <count>` in the full trail band in
     `src/sase/pager/_trail_chrome_band.py`, leaving compact arrow/count rendering
     unchanged;
   - use `ctrl+i` for “Walk forward” in the Trail & keys sheet in
     `src/sase/pager/_trail_chrome_help.py`.
3. Update the forward-history row in `docs/pager.md` to list `Ctrl+I` as the documented
   key again. Do not advertise Tab in that UX row; its binding remains an internal
   compatibility path for terminals and hosts that deliver the same chord as `tab`.
4. Update focused assertions in `tests/pager/test_app_history.py`,
   `tests/pager/test_bead_live_links.py`, `tests/pager/test_chrome.py`,
   `tests/pager/test_help.py`, and `tests/pager/test_trail_chrome.py` so they require
   the restored Ctrl+I labels and reject the superseded `<tab>` label where useful. Add
   a direct binding-metadata assertion near the pager tests to lock in both halves of
   the contract: the registered keys remain `tab,ctrl+i`, while the displayed key is
   Ctrl+I.
5. Preserve the behavior-focused coverage added by the original fix. In particular, do
   not rewrite Tab-driven navigation tests to use Ctrl+I: the existing standalone and
   ACE-hosted tests should continue proving that Tab advances history, is harmless with
   no forward entry, respects pager modal/input states, and cannot escape to ACE's
   priority next-tab binding. Retain the existing Ctrl+I alias test as the complementary
   proof that the advertised chord works.

## Verification

1. Run the focused pager suites that cover the changed renderers, help content,
   documentation-adjacent behavior, and embedded-host regression:

   ```bash
   uv run pytest \
     tests/pager/test_app_history.py \
     tests/pager/test_bead_live_links.py \
     tests/pager/test_chrome.py \
     tests/pager/test_help.py \
     tests/pager/test_trail_chrome.py \
     tests/ace/tui/actions/test_view_files_pager_screen.py
   ```

2. Search the pager source, tests, and pager documentation for stale forward-history
   `<tab>` labels, distinguishing intentional Tab behavior tests and the compound
   binding from user-facing text.
3. Run `just check` for the repository-required whole-tree lint gates and diff-scoped
   tests. If dependency drift prevents the check from starting, run `just install` and
   retry `just check`.

## Non-goals

- Do not remove or demote the working Tab alias.
- Do not add a configurable pager keymap or modify `src/sase/default_config.yml`; this
  pager binding is hard-coded and the requested change is presentation-only.
- Do not change forward-history stack semantics, the backward-history keys, compact
  trail arrows, or unrelated Tab labels elsewhere in ACE.
