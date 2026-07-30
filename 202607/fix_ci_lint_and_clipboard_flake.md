---
tier: tale
title: Fix master CI - symvision private-import lint and the racy gate-debug clipboard test
goal:
  "Master CI is green: `just lint` passes symvision with no cross-module private imports in the clipboard palette
  package, and `tests/ace/tui/test_gate_debug_modal.py::test_tabs_and_copy_actions_use_prebuilt_snapshot` asserts
  deterministically instead of racing on thread-pool completion order."
create_time: 2026-07-30 07:16:22
status: done
---

- **PROMPT:** [202607/prompts/fix_ci_lint_and_clipboard_flake.md](prompts/fix_ci_lint_and_clipboard_flake.md)

# Plan: Fix master CI - symvision private-import lint and the racy gate-debug clipboard test

## Context

Master CI has been red for the last several commits. `actstat -n 8 --repo sase-org/sase` shows three distinct failing
jobs across runs; investigation reduces them to two live defects plus one that is already fixed.

| Failing job                                       | Root cause                                                                                                                       | Status                                                                                                             |
| ------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| `lint` (step 12, `just lint`)                     | symvision: private symbols imported across files in `src/sase/ace/tui/actions/clipboard/`                                        | **Live** — fix in this plan                                                                                        |
| `test (3.12/3.13/3.14)` (step 4, `just test-cov`) | `test_tabs_and_copy_actions_use_prebuilt_snapshot` asserts positional order over two concurrently scheduled clipboard deliveries | **Live** — fix in this plan                                                                                        |
| `published-core-minimum-smoke` (step 7)           | `sase-core-rs` floor `0.12.17` lacked the `AtReferenceInventory` binding                                                         | **Already fixed** by `c135dcbd6` (floor raised to `0.12.18`); the job is green on the latest run. Do not touch it. |

Both live defects were verified by reproducing them locally in this repo, not just by reading CI logs.

## Defect 1 - symvision cross-module private imports

### Evidence

`just lint` reproduces the CI failure exactly:

```
Error: Private functions/classes should not be imported. Make these public if they need to be
imported by non-test files!:
  _artifact_target_state in src/sase/ace/tui/actions/clipboard/_palette_artifact_previews.py
  _axe_item_label in src/sase/ace/tui/actions/clipboard/_palette_helpers.py
  _build_artifacts_context in src/sase/ace/tui/actions/clipboard/_palette_artifacts.py
  _context_from_registry in src/sase/ace/tui/actions/clipboard/_palette_registry.py
  _marked_count_hint in src/sase/ace/tui/actions/clipboard/_palette_helpers.py
  _marked_hint in src/sase/ace/tui/actions/clipboard/_palette_helpers.py
  _number_from_url in src/sase/ace/tui/actions/clipboard/_palette_helpers.py
  _output_hint in src/sase/ace/tui/actions/clipboard/_palette_helpers.py
  _short in src/sase/ace/tui/actions/clipboard/_palette_helpers.py
  _size_hint in src/sase/ace/tui/actions/clipboard/_palette_helpers.py
  _warm_agent_file_path in src/sase/ace/tui/actions/clipboard/_palette_helpers.py
  _warn in src/sase/ace/tui/actions/clipboard/_palette_helpers.py
error: recipe `_lint-symvision` failed on line 273 with exit code 1
```

Introduced by `df18f44f6 refactor(ace): split clipboard palette module`, which split `_palette.py` into
`_palette_helpers.py`, `_palette_registry.py`, `_palette_artifacts.py`, and `_palette_artifact_previews.py` while
keeping the moved symbols `_`-prefixed. Symbols that were file-local before the split are now imported across module
boundaries, which is exactly what symvision forbids.

### Established convention to follow

The same package already solves this correctly. `_helpers.py` and `_representations.py` are themselves private modules
but export **public** names that siblings import:

- `src/sase/ace/tui/actions/clipboard/_helpers.py` defines `cap_copy_content`, `format_multi_copy_content_capped`,
  `capture_tmux_pane`, `format_markdown_link`, `format_changespec_for_clipboard`.
- Those are imported by `_artifact_targets.py`, `_changespec.py`, `_core.py`, `_agents.py`, `_axe.py`,
  `_representations.py`.

Module privacy is carried by the leading `_` on the _module_ name, not on every symbol inside it. The `_palette_*` split
simply missed this step.

### Fix

Rename each of the 12 flagged symbols to a public name in its defining module and update every importer and call site.
Keep symbols that are genuinely file-local (for example `_dispatch_winners`, `_artifact_pane`, `_entry_targets`,
`_plan_values`, `_commit_message`) private and untouched — only rename what symvision flagged.

Proposed names (drop the underscore, except where the bare name is too generic to read well at a call site in another
module):

| Current                    | New                       | Defining module                 |
| -------------------------- | ------------------------- | ------------------------------- |
| `_artifact_target_state`   | `artifact_target_state`   | `_palette_artifact_previews.py` |
| `_build_artifacts_context` | `build_artifacts_context` | `_palette_artifacts.py`         |
| `_context_from_registry`   | `context_from_registry`   | `_palette_registry.py`          |
| `_axe_item_label`          | `axe_item_label`          | `_palette_helpers.py`           |
| `_marked_count_hint`       | `marked_count_hint`       | `_palette_helpers.py`           |
| `_marked_hint`             | `marked_hint`             | `_palette_helpers.py`           |
| `_number_from_url`         | `number_from_url`         | `_palette_helpers.py`           |
| `_output_hint`             | `output_hint`             | `_palette_helpers.py`           |
| `_size_hint`               | `size_hint`               | `_palette_helpers.py`           |
| `_warm_agent_file_path`    | `warm_agent_file_path`    | `_palette_helpers.py`           |
| `_short`                   | `shorten`                 | `_palette_helpers.py`           |
| `_warn`                    | `notify_copy_warning`     | `_palette_helpers.py`           |

`shorten` and `notify_copy_warning` get descriptive names because bare `short`/`warn` read poorly once they are public
and cross-module; this also matches the verb-phrase idiom already used in `_helpers.py`.

Every symbol in `_palette_helpers.py` ends up public. That is fine and expected — the module exists purely as a shared
helper module for the other `_palette_*` modules, exactly like `_helpers.py`.

### Explicitly out of scope for this defect

- Do **not** add symvision pragmas or `--epic-symbol` entries. The memory note (`sase memory read symvision.md`) is
  explicit that whitelisting is a last resort; here there is a real non-test consumer for every renamed symbol, so
  making them public is the sanctioned fix.
- Do **not** patch or vendor symvision.
- Leave `_DISPATCH_ORDER` alone. It is a module-level dict, not a function/class, so symvision does not flag it, and
  `_palette.py` already re-exports it (`from ._palette_registry import _DISPATCH_ORDER as _DISPATCH_ORDER`) for
  `tests/ace/tui/test_copy_targets.py`. Renaming it is unnecessary churn.

## Defect 2 - racy positional assertion in the gate-debug clipboard test

### Evidence

CI failure (identical on 3.12, 3.13, and 3.14 legs, and on every run since at least `df18f44`):

```
FAILED tests/ace/tui/test_gate_debug_modal.py::test_tabs_and_copy_actions_use_prebuilt_snapshot
  - AssertionError: assert ('/var/tmp/sa...issing-gate',) == ('Status     ...errors    0',)
```

The current test body (`tests/ace/tui/test_gate_debug_modal.py`, around line 184) does:

```python
with patch(
    "sase.ace.tui.actions.clipboard._delivery.copy_to_system_clipboard",
    return_value=True,
) as copy:
    modal.action_copy_tab()
    modal.action_copy_bundle_path()
    await pilot.pause()

assert copy.call_args_list[0].args == (modal._snapshot.overview.raw_text,)
assert copy.call_args_list[1].args == (str(tmp_path / "missing-gate"),)
```

### Root cause

`GateDebugModal.action_copy_tab` and `action_copy_bundle_path` both call `schedule_copy_delivery`
(`src/sase/ace/tui/actions/clipboard/_delivery.py`), which spawns an independent event-loop task via
`spawn_pump_free_task`. Each task's `deliver_copy` then reaches the clipboard through
`await asyncio.to_thread(copy_to_system_clipboard, resolved)`.

Two `asyncio.to_thread` calls submitted back to back run on **different threads of the default executor**. The order in
which those threads actually call the patched mock is decided by the OS scheduler, not by submission order, so
`copy.call_args_list` is not deterministically ordered. The test asserts an ordering the implementation never promised.

This was confirmed empirically:

1. A standalone probe that spawns two tasks each awaiting `asyncio.to_thread(fake_copy, ...)` and checks which ran first
   inverted the order in **1294 of 3000 trials** on an idle local machine.
2. Copying the exact test body into a 60x parametrized stress file inside `tests/ace/tui/` reproduced the real CI
   assertion failure locally (**1 failure in 60**). A single run of the test in isolation passes, which is why this
   reads as "flaky" rather than "broken" — CI's loaded, xdist-parallel runners just lose the race far more often.

The same block has a second latent flake: one `await pilot.pause()` is not guaranteed to let _both_ thread hops finish,
so `call_args_list` can have fewer than two entries and the assertion raises `IndexError` instead.

### Fix

Rewrite `test_tabs_and_copy_actions_use_prebuilt_snapshot` to patch the `schedule_copy_delivery` seam in the modal's own
namespace instead of the transport function, and assert synchronously. This is the dominant convention in this repo for
"which value does this action copy" tests — see
`tests/ace/tui/modals/test_report_modal.py::test_copy_path_uses_system_clipboard`, `tests/test_copy_agent_name.py`,
`tests/test_mentor_review_modal_actions.py`, and `tests/test_notification_modal_action_bindings.py`, all of which patch
`schedule_copy_delivery` at the importing module.

`src/sase/ace/tui/modals/gate_debug_modal.py` line 19 does
`from sase.ace.tui.actions.clipboard import schedule_copy_delivery`, so the patch target is
`sase.ace.tui.modals.gate_debug_modal.schedule_copy_delivery`.

Shape of the replacement (the implementer should match surrounding style; this is intent, not literal text):

```python
with patch(
    "sase.ace.tui.modals.gate_debug_modal.schedule_copy_delivery"
) as schedule:
    modal.action_copy_tab()
    modal.action_copy_bundle_path()

assert schedule.call_args_list[0].args[1] == modal._snapshot.overview.raw_text
assert schedule.call_args_list[1].args[1] == str(tmp_path / "missing-gate")
```

Assert the `copied_label` / `task_name` keywords too, matching how `test_report_modal.py` uses
`assert_called_once_with`, so the test still pins which action produced which delivery.

Because `schedule_copy_delivery` is now a mock, no task is spawned, no thread hop happens, and the two calls land in
invocation order by construction. The `await pilot.pause()` in that block becomes unnecessary — drop it. Keep the rest
of the test (snapshot readiness polling, `action_next_tab`/`action_previous_tab` tab-index assertions) exactly as it is;
only the copy block changes.

### Coverage is not lost

The real transport path — `deliver_copy`, `asyncio.to_thread`, OSC 52 fallback, failure policies — is already covered
directly by `tests/ace/tui/actions/test_clipboard_delivery.py`. The gate-debug modal test is about _which value each
action copies_, so patching at the seam is the correct altitude and removes the thread race from a test that never meant
to exercise it.

### Considered and deliberately not done: serializing clipboard deliveries

An alternative fix is to serialize `schedule_copy_delivery` per app (an `asyncio.Lock` or FIFO queue) so back-to-back
copies always land in invocation order. That would make the existing assertion valid _and_ fix a real, if minor,
user-visible race: pressing copy-tab then copy-bundle-path in quick succession can leave the wrong value on the
clipboard, since the clipboard is last-write-wins.

This plan does **not** do that, because:

- It is a behavior change to a hot TUI path that is not required to make CI green.
- Serializing means a slow or wedged copy (the callable-value path does disk I/O in a thread) would block every
  subsequent copy, which is a responsiveness regression the current design deliberately avoids.

The implementer should file this as a separate bead rather than folding it into the CI fix. Note it in the final report
so it is not silently dropped.

## Files to change

- `src/sase/ace/tui/actions/clipboard/_palette_helpers.py` — rename 9 defs to public.
- `src/sase/ace/tui/actions/clipboard/_palette_registry.py` — rename `_context_from_registry`; update its `_short`
  import and call sites.
- `src/sase/ace/tui/actions/clipboard/_palette_artifacts.py` — rename `_build_artifacts_context`; update its
  `_artifact_target_state`, `_warn`, `_context_from_registry` imports and call sites.
- `src/sase/ace/tui/actions/clipboard/_palette_artifact_previews.py` — rename `_artifact_target_state`; update its
  `_marked_count_hint`, `_marked_hint`, `_short`, `_size_hint` imports and call sites.
- `src/sase/ace/tui/actions/clipboard/_palette.py` — update its `_build_artifacts_context`, `_palette_helpers`, and
  `_palette_registry` imports and call sites.
- `tests/ace/tui/test_gate_debug_modal.py` — rewrite the copy block of
  `test_tabs_and_copy_actions_use_prebuilt_snapshot`.

Sweep the whole repo (`src/` and `tests/`) for each old name before declaring the rename done; do not rely on the file
list above being exhaustive.

## Verification

Run `just install` first — this repo uses ephemeral `sase_<N>` workspaces whose virtualenvs drift.

1. `just _lint-symvision` — must pass with no private-import errors. This is the exact failing CI step.
2. `just check` — required by repo policy after any file change; covers `fmt`, `lint`, `mypy`, and tests.
3. Confirm the flake is actually gone, not just passing once. Temporarily add a parametrized stress copy of
   `test_tabs_and_copy_actions_use_prebuilt_snapshot` (60+ repetitions in one process) under `tests/ace/tui/`, confirm 0
   failures, then **delete the stress file** before committing. On the unfixed code this reproduces the CI failure
   locally, so it is a real regression check rather than a formality.
4. `git status --short` must be clean of stray scratch files (stress test, probes) before finishing.

## Success criteria

- `just check` passes locally.
- `just _lint-symvision` reports no `Private functions/classes should not be imported` errors.
- The stress reproduction of `test_tabs_and_copy_actions_use_prebuilt_snapshot` passes 60/60.
- No symvision pragmas, `--epic-symbol` entries, or `Justfile` lint-invocation changes were added.
- No production behavior change in `src/sase/ace/tui/actions/clipboard/_delivery.py` or
  `src/sase/ace/tui/util/pump_tasks.py` — the palette work is a pure rename, and the test fix is test-only.
- `pyproject.toml`'s `sase-core-rs>=0.12.18,<0.13.0` floor is left untouched.
