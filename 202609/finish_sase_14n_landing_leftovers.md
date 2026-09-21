---
tier: epic
title: Finish the sase-14n landing leftovers
goal:
  "`just check` is green on master again: symvision is clean and the TUI app import
  count is strictly under its unchanged 3290 cap. Both defects behind sase-14g are
  closed end to end: a dismissed notification can be found and restored from the ACE
  notification modal, and a shell-block custom gate always registers its gate-shell row
  through the real creation path."
parent_bead: sase-14n
phases:
  - id: check_green
    title: Return just check to green on master
    depends_on: []
    size: medium
    description:
      "check_green: privatize the three test-only notification footer helpers that
      symvision flags, and cut at least 20 modules from the sase.ace.tui.app import
      closure so the import-budget node passes under its unchanged cap."
  - id: dismissed_view
    title: Reach and restore dismissed notifications from the notification modal
    depends_on:
      - check_green
    size: medium
    description:
      "dismissed_view: give the ACE notification modal a way to show dismissed rows, so
      its existing u binding can restore one, then advertise both keys in the footer."
  - id: gate_row
    title: Prove the gate-shell row through the production gate creation path
    depends_on:
      - check_green
    size: medium
    description:
      "gate_row: find how a shell-block custom gate can reach create_gate without
      registering a row, stop that path from succeeding silently, and replace the
      fabricated row test with an end-to-end one."
proposed_by: bbugyi200.athena.sase-14n.land
create_time: 2026-09-21 15:11:19
status: wip
---

- **PROMPT:**
  [prompts/202609/finish_sase_14n_landing_leftovers.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/finish_sase_14n_landing_leftovers.md)
- **PARENT:**
  [202609/fix_triaged_bug_and_ci_beads.md](https://github.com/sase-org/sase--plans/blob/main/202609/fix_triaged_bug_and_ci_beads.md)

# Plan: Finish the sase-14n landing leftovers

## Problem

Epic `sase-14n` ("Fix the bug and CI task beads that survived 2026-09-20 triage") was
being landed at master `87c604833` when its land agent found four unresolved issues. All
14 of its phases are closed and their commits are intact. After `just install` against
the linked sase-core, which the pin in `sase-core-revision.txt` already contains, a
230-node batch of every phase's tests passes. `just test-visual` fails only on
`sase-151`, which a bisect pins to the unrelated `4255afbb0`. Ten of the epic's task
beads are closed. `sase-13p` and `sase-14g` stay open because of the issues below.

1. `just lint` is red at the symvision stage. Epic phase `sase-14n.8` (`03fff9dcb`, the
   notification footer tier ladder) added three public symbols that only tests use.
2. `tests/ace/tui/test_app_import_budget.py::test_tui_app_import_stays_under_startup_budget`
   fails with `assert 3309 < 3290`. Phase `sase-14n.4` (`5963c52e8`) brought the closure
   from 3290 down to 3282. Commits that landed later then re-grew it by 27 modules.
   `sase-13p` stays open for this.
3. The undismiss half of `sase-14g` works from the CLI but not from the TUI. Phase
   `sase-14n.13` (`98bf83a38`) added `u` to the notification modal, but the modal only
   ever receives non-dismissed rows, so `u` can never reach a dismissed one.
4. The gate-shell-row half of `sase-14g` is fixed only at the model layer. Phase
   `sase-14n.12` (`c04171670`) derives `continuation_mode="gate_shell"` for a shell
   block. But its "row" test builds the row itself, and a direct `create_gate()` call
   with a shell block still produces a gate with no row.

This plan finishes those four items. The `sase-14n` land agent resumes after this child
epic lands, so this plan does not include closing `sase-14n`.

## Conventions for every phase

- Verify your phase with `sase tool run check` (the recorded form of `just check`). Do
  not run `just check-full`: nothing here asks for it. A `just check` pass with a
  `just check-full` failure is a test-infrastructure bug, not phase work.
- Your workspace may have a stale `sase_core_rs`. If notification or tool-run tests fail
  with `unknown variant` or missing-binding errors, run `sase repo open sase-core` and
  then `just install` before debugging anything else.
- Do not create task beads. Record anything out of scope as a
  `PROPOSED FOLLOW-UP: <summary — detail>` note on your own phase bead.
- Do not loosen an assertion, raise a budget, or regenerate a golden just to make a gate
  pass. Phases that change rendered TUI output run `just fix-tui-screenshots` and
  inspect the report and every update group before accepting it. Read
  `sase memory read tui.md` before changing the TUI.
- Known unrelated reds you may hit: `sase-151` in `just test-visual`, and the load
  flakes `sase-t6`, `sase-120`, `sase-15i` to `sase-15m`. Neither is phase work.

## Return just check to green on master

Owns `sase-13p`. Close it with `sase bead close sase-13p --note "..."`, quoting the
measured module count and the passing command.

**Symvision.** `just _lint-symvision` reports these symbols in
`src/sase/ace/tui/modals/notification_modal_constants.py`:

- `NotificationHintFragment`
- `notification_hint_fragments`
- `notification_hint_tier`

All three are used only inside that file, apart from the test module
`tests/ace/tui/modals/test_notification_hint_footer.py`, which imports
`notification_hint_tier`. Read `sase memory read symvision.md` first. Make each one
private (`_` prefix), update the in-file callers, and point the test at the private
name. Test files may import private names; test references never keep a public symbol
alive. Add no pragmas and no Justfile `--epic-symbol` entries. Keep public the constants
and functions that non-test code imports: `notification_hint_text`,
`NOTIFICATION_HINT_FALLBACK_WIDTH`, and the `*_HINT_TEXT` / `*_HINT_FRAGMENTS`
constants.

**Import budget.** The guard caps the `sase.ace.tui.app` import closure at
`_MAX_MODULE_COUNT = 3290` with a strict `<`. At `87c604833` the closure is 3309 modules
on every run, so you need to cut at least 20. Raising the cap or relaxing the comparison
is not an option. Keep the `deferred_modules` assertion and the `sase-136` CPU-time
check exactly as they are.

Re-measure first with
`python -c "import sys, sase.ace.tui.app; print(len(sys.modules))"`. Then find each
eager edge with `python -X importtime -c "import sase.ace.tui.app"`. These modules
arrived after `5963c52e8` and are the leads:

- The `sase.update_progress` package (9 modules) plus `rich.spinner`, from `d9a1de8cc`
  and `f64bd3ac2`.
- `sase.dev_update.*` (about 20 modules). It is reached eagerly through the
  plugins-browser and update-toast modules and `sase.uv_tool.versions`; most of those
  imports only want `sase.dev_update.models` types, and
  `sase.ace.tui.actions.update_toast` imports `sase.dev_update.prebuild` at module
  level.
- The bead-touch modules `sase.ace.tui.bead_touches`, `sase.bead.touch_glyphs`, and
  `sase.core.bead_touch_index_facade`, from `e1ba4851c`, `319fe6b24`, `821a21c49`, and
  `da766b87a`.

Follow the `5963c52e8` precedent: move runtime imports to their use sites, and use
`TYPE_CHECKING` imports with quoted annotations where only typing needs the name. Do not
change behaviour. Once a package is fully out of the closure (for example
`sase.update_progress`), add it to the test's `deferred_modules` probe list so it cannot
creep back. That only makes the existing assertion stronger. Record in the commit
message which edges you cut and the before and after counts.

Done when:

- `just _lint-symvision` is clean.
- The import-budget node passes, with a measured count strictly below 3290.
- `sase tool run check` passes.
- `sase-13p` is closed.

## Reach and restore dismissed notifications from the notification modal

This is the TUI half of `sase-14g`. Do not close that bead; the land agent closes it
once this phase and `gate_row` are both verified.

What exists today:

- `sase notify apply-state <id> undismiss` works.
- `mark_undismissed` and `mark_many_undismissed` live in
  `src/sase/notifications/store.py`, backed by the sase-core `MarkUndismissed` wire
  variants.
- The modal binds `u` to `action_undismiss_notification` in
  `src/sase/ace/tui/modals/notification_modal_undismiss_actions.py`.

The gap: the modal is only ever populated from
`_read_unread_notification_page_from_provider()` with `include_dismissed=False` (see
`src/sase/ace/tui/actions/agents/_notification_modal_flow.py` and
`_notification_provider_direct.py`). Pressing `x` also drops the row from the list
immediately. So in production `u` never has a dismissed row to act on. The tests in
`tests/test_notification_modal_undismiss_actions.py` construct modals that already hold
`dismissed=True` rows, a state production cannot produce.

Required:

- Give the modal a way to show dismissed rows. The recommended shape is a modal key that
  switches between the normal inbox and a dismissed view. The dismissed view reloads the
  page with `include_dismissed=True` (the provider already accepts it) and marks
  dismissed rows visibly in the list or title. Pick a key that no modal binding uses
  today and that doesn't clash with the uppercase/`shift+` twin convention the modal
  already follows. A tag tab for dismissed rows is also acceptable if it fits the
  existing tab strip better. Record the choice and the reason.
- `u` restores the highlighted row, or every marked row, in that view. Once restored,
  the row behaves like any other row when you return to the normal inbox.
- Decide what happens to a live gate notification that was read and then dismissed. The
  modal lists unread rows only. `_notification_modal_flow.py` keeps some gate actions
  unread until they are answered (`read_protected_actions`), but check whether a
  `CustomGate` row can end up both read and dismissed. If it can, the dismissed view
  must still surface it, or undismissing a live gate must make it reachable some other
  way. Record the decision.
- Add the new view key and `u: undismiss` to all three footer fragment lists in
  `notification_modal_constants.py` (default, question, gate) at a priority that follows
  the existing ladder. The invariant tests in
  `tests/ace/tui/modals/test_notification_hint_footer.py` must still hold: every variant
  fits 108 cells at 120 columns, with `q: close` and `+: +1` always present. If the
  `notification_*_120x40` goldens move, refresh them through `just fix-tui-screenshots`
  and inspect the report.
- Tests:
  - A production-path test that seeds the store with a dismissed live gate notification,
    opens the modal through the real app flow, switches to the dismissed view, presses
    `u`, and asserts two things: the store row is undismissed, and after returning to
    the normal inbox the gate notification is listed and Enter reaches its gate handler.
  - Replace or re-seed the existing undismiss tests so none of them depend on the
    impossible pre-populated dismissed state.
- If you add or change a keymap that is configurable, update
  `src/sase/default_config.yml`. Update `docs/notifications.md` wherever it describes
  dismissal as one-way or lists the modal keys.

Done when:

- A dismissed notification can be found and restored from the modal in the real app
  flow.
- The production-path test pins that, including gate reachability.
- The footer advertises the keys and still fits.
- `sase tool run check` passes.

## Prove the gate-shell row through the production gate creation path

This is the gate-list half of `sase-14g`. Do not close that bead.

Background. The reporting case was gate `custom-fcaeb8b6`, filed by `sase-13t.1.f1`. Its
`request.json` declared a shell block, but `.creation_result.json` recorded
`continuation_mode="none"`, and `sase gate list --all` did not list it. Its artifacts
are no longer on athena.

`c04171670` changed the model layer:

- `GateSpec` in `src/sase/notification_gates/model_request.py` now derives `gate_shell`
  when a request has a shell block and no mode, and rejects a shell block paired with
  `none`.
- `src/sase/notification_gates/validation.py` repeats that check.

Two things remain unproven:

- `sase gate create` (`src/sase/main/gate_handler.py`) routes a request with a `shell`
  dict to `create_gate_shell` in `src/sase/gate_shell/transaction.py`. That function
  registers the member row _before_ it calls `notification_gates.service.create_gate`,
  so a CLI-created shell gate already had a row before `c04171670`. The only thing the
  old default wrote wrong was the mode, and `continuation_mode` plays no part in whether
  the gate is listed. So why the reported gate had no row is still unexplained.
- `tests/gate_shell/test_model_shell.py` builds its own row with
  `make_gate_shell_member`, so it never proves that production registers one. A direct
  `create_gate(spec)` call with a shell block still creates a gate with no row, and now
  records `gate_shell`.

Required:

- Enumerate every path that can reach `notification_gates.service.create_gate` with a
  shell-bearing spec: grep the callers and `GateSpec` producers, including request-file
  ingestion, plugins, and the command the `/sase_gate` skill documents. Reproduce with
  the real CLI at `c04171670^` to see whether a shell-block request can end up without a
  row.
- Also rule in or out the listing side. `handle_gate_shell_list` calls
  `list_gate_shells(project=...)` and hides terminal rows unless `--all` is passed, so a
  gate created from another project's workspace, or settled in a state the list filters
  out, would be missing even with a row. Record what you find in the commit message and
  the phase note.
- Make it impossible to create a shell-bearing gate without a row silently. Either have
  `create_gate` refuse a shell-bearing spec that did not come through the gate-shell
  transaction, with a clear `GateError` (the transaction passes an explicit internal
  marker), or route such a spec through `create_gate_shell`. Keep every built-in
  shell-backed kind working, including sudo, plan, question, HITL, and launch.
- Replace the fabricated-row assertion with an end-to-end test through the gate CLI
  handler or `create_gate_shell`, run in a temporary SASE home and project. For a
  request that declares a shell block and omits `continuation_mode`, assert all three
  of:
  - the recorded continuation mode is `gate_shell`;
  - a gate-shell member row exists;
  - `list_gate_shells(project=...)`, and therefore `sase gate list --all`, returns it.

  Add a test that the bypass path now fails loudly.

Done when:

- A shell-block custom gate provably registers and lists its row through the production
  path.
- The rowless path fails with a clear error instead of succeeding silently.
- The investigation's finding is recorded.
- `sase tool run check` passes.
