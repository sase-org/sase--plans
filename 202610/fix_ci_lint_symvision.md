---
tier: tale
size: small
title: Turn the CI lint job green by clearing the symvision unused-public findings
goal: "`just _lint-symvision` exits 0 on master, so the `lint` job in Master Gate and PR
  CI (and `full / lint` in Full CI) passes again. The fix privatizes in-module helpers,
  deletes one dead helper, and changes no runtime behavior.

  "
decisions:
  privatize_count_rows:
    ask:
      Also privatize the unflagged count_wrapped_rows geometry helper for consistency?
    default: true
    why: Same in-module-only shape; symvision misses it only through a name collision
    answer: true
proposed_by: bbugyi200.athena.0zm
decided_by: reviewer
decided_via: tui
create_time: 2026-10-11 07:43:27
status: wip
---

# Plan: Turn the CI `lint` job green by clearing the symvision unused-public findings

## Context

The `lint` job has been red on every Master Gate run since the last green one, run
37979427867 on 70c51adbdd (2026-10-09 19:19 UTC). The latest red run is 38130053371 on
8122d44255. Full CI's `full / lint` job fails the same way. The failing step is `Lint`
(`just lint`), which stops at the `_lint-symvision` stage with exit 1. Every lint stage
before symvision passes.

Current symvision output (it reproduces locally with `just _lint-symvision` on
8122d44255):

```text
Unused public functions/classes. Make these private if they are used only within the file they are defined. If the functions/classes are completely unused, you should delete them:
  action_footer_lines in src/sase/ace/tui/modals/update_failure_geometry.py
  choose_outer_width in src/sase/ace/tui/modals/update_failure_geometry.py
  content_cell_width in src/sase/ace/tui/modals/update_failure_geometry.py
  footer_groups_for_width in src/sase/ace/tui/modals/update_failure_geometry.py
  hint_lines in src/sase/ace/tui/modals/update_failure_geometry.py
  is_cramped in src/sase/ace/tui/modals/update_failure_geometry.py
  logical_lines in src/sase/ace/tui/modals/update_failure_geometry.py
  max_cell_width in src/sase/ace/tui/modals/update_failure_geometry.py
  max_outer_size in src/sase/ace/tui/modals/update_failure_geometry.py
  pack_footer_lines in src/sase/ace/tui/modals/update_failure_geometry.py
  project_for_done in src/sase/agent/auto_restart/history.py
  required_outer_width in src/sase/ace/tui/modals/update_failure_geometry.py
```

Two ready `ci` task beads already track these findings, and this tale fixes both:

- **sase-1jp**: `project_for_done`
- **sase-1js**: the 11 `update_failure_geometry.py` helpers

No in-progress epic owns these files. No linked repo references any of these symbols
(checked: sase-telegram, sase-github, sase-research-artifacts, sase-listen).

## Root causes

1. **`src/sase/ace/tui/modals/update_failure_geometry.py`** came from 217a5439f5 (an
   ad-hoc agent with no bead). It declares its sizing helpers public and lists them in
   `__all__`. Its only non-test consumer is `update_failure_modal.py`, which imports
   just five names:
   - `PLACEHOLDER_TEXT`
   - `SEPARATOR_TEXT`
   - `UpdateFailureGeometry`
   - `compute_geometry`
   - `expand_tabs`

   Every other helper is reached only from inside the module or from
   `tests/ace/tui/test_update_failure_geometry.py`. Neither path keeps a public symbol
   alive: symvision ignores test references and does not count `__all__` membership.
   `choose_outer_width` has no production caller at all, because `compute_geometry`
   inlines the same clamp:
   `width = max(1, min(max(needed, PREFERRED_MIN_WIDTH), max_w, max(1, screen_w)))`.

   The sase-1js description mentions other `src` hits for `logical_lines` and
   `hint_lines`. Those are unrelated names, not imports of these functions:
   - a local variable in `pager/syntax.py` and in `modals/_preview_panel_geometry.py`
   - the `UpdateFailureGeometry.hint_lines` dataclass field, which the modal reads

2. **`src/sase/agent/auto_restart/history.py::project_for_done`** lost its only
   cross-file caller in 20ddc1b154 (sase-1j6.10). That commit inlined the call into
   `healer_targets._target_for_artifacts_dir`. The one remaining caller is in
   `history.py` itself, in the `project=project_for_done(done, artifacts_dir)` argument
   that builds a `FailedCandidate`.

## Changes

This is a rename/delete-only change. It does not change behavior.

### 1. `src/sase/ace/tui/modals/update_failure_geometry.py`

Privatize each in-module-only helper. Rename the `def` and every in-file call site:

| Old name                  | New name                   |
| ------------------------- | -------------------------- |
| `is_cramped`              | `_is_cramped`              |
| `max_outer_size`          | `_max_outer_size`          |
| `logical_lines`           | `_logical_lines`           |
| `max_cell_width`          | `_max_cell_width`          |
| `content_cell_width`      | `_content_cell_width`      |
| `count_wrapped_rows`      | `_count_wrapped_rows`      |
| `footer_groups_for_width` | `_footer_groups_for_width` |
| `pack_footer_lines`       | `_pack_footer_lines`       |
| `action_footer_lines`     | `_action_footer_lines`     |
| `hint_lines`              | `_hint_lines`              |
| `required_outer_width`    | `_required_outer_width`    |

> [!decision] privatize_count_rows
>
> **`count_wrapped_rows` is not flagged**, but only because symvision matches by name.
> `_preview_panel_geometry.py` imports the unrelated
> `preview_search.count_wrapped_rows`, which hides this one. It has the same
> in-module-only shape, so privatize it too and the module's public surface stays
> consistent.

> [!decision] privatize_count_rows = no
>
> Leave `count_wrapped_rows` public, in `__all__`, and imported under its current name
> by the test. Skip its row in the table above and its entry in the test-import list
> below.

Notes:

- **`hint_lines` collides with the `UpdateFailureGeometry.hint_lines` dataclass field.**
  Rename only the function `def` and its single call, `hint = hint_lines(inner_width)`.
  Do **not** rename the dataclass field or the `hint_lines=...` keyword arguments in
  `compute_geometry`'s three `UpdateFailureGeometry(...)` constructions. The modal still
  reads `geometry.hint_lines`.

Delete `choose_outer_width` entirely, since it is dead (step 1 of the symvision decision
hierarchy). Do not wire it into `compute_geometry`.

Keep these public: `UpdateFailureGeometry`, `compute_geometry`, `expand_tabs`, and all
module constants.

Rewrite `__all__` to drop every renamed or deleted name. It keeps the constants plus
`UpdateFailureGeometry`, `compute_geometry`, and `expand_tabs`, in the current
alphabetical order.

### 2. `tests/ace/tui/test_update_failure_geometry.py`

- Import the underscored names (`_action_footer_lines`, `_content_cell_width`,
  `_count_wrapped_rows`, `_is_cramped`, `_max_outer_size`, `_pack_footer_lines`) and
  update their call sites. The tests in this repo routinely import private helpers, and
  symvision allows private imports from test files.
- Remove the `choose_outer_width` import and the two `choose_outer_width(...)` asserts
  at the end of `test_cramped_uses_full_rectangle`. Existing tests already cover that
  behavior:
  - `test_compact_failure_uses_preferred_width` checks the 90-column width at 240×70.
  - `test_minimum_terminal_keeps_actions_and_one_row` checks a width of at most 40 at
    40×12.
- Leave every other assertion unchanged.

### 3. `src/sase/agent/auto_restart/history.py`

Rename `project_for_done` to `_project_for_done` and update its one in-file call. It is
not in `__all__`, and no test imports it.

### Out of scope

- No `# symvision:` pragmas, no new `--epic-symbol` or `--exclude-decorator` rows, and
  no change to the installed symvision package or its version constraint.
- No edits to the Justfile, the workflows, other CI jobs (test shards, `visual-test`,
  Full CI test lanes), memory files, or `CHANGELOG.md`.
- No PNG goldens should change. Do not run `just fix-tui-screenshots`.

## Verification

1. Run `just _lint-symvision`. It must exit 0 with no `Unused public`, `Private ...`,
   pragma, or `--epic-symbol` errors. If this workspace's venv has drifted, run
   `sase tool run install-venv` first; it is long-running, so follow the `/sase_monitor`
   guidance if it will not fit inline.
2. Run the targeted tests:
   `sase tool run test -- tests/ace/tui/test_update_failure_geometry.py tests/ace/tui/test_update_failure_modal.py tests/ace/tui/test_update_attempt_state.py tests/test_agent_auto_restart_scan.py tests/test_agent_auto_restart_scan_history.py`.
3. Run `sase tool run check`, which the repo requires after any code change. It runs
   every lint gate except `toobig`, plus the scoped tests. Treat UNKNOWN items as yours.
4. Run the lint-job steps that `just check` does not cover. CI has not reached them
   since symvision went red:
   - `just _lint-toobig`
   - `just validate`
   - `just validate-committed-plans`

   All three passed locally on 8122d44255.

   Do not chase a local `just build-check` failure. CI's fresh install resolves twine
   7.0.0, and the last green gate (37979427867) logged `PASSED` for both artifacts. A
   stale workspace venv that still has twine 6.2.0 rejects hatchling 1.32.4's wheel with
   `'2.5' is not a valid metadata version`. That failure is local environment drift, not
   a repo defect. To check the build anyway, run `just build`, then
   `uvx --from twine twine check dist/*`, then `rm -rf dist`.

5. Master moves quickly, so re-run `just _lint-symvision` on the final tree. Handle any
   finding that appeared after this plan as follows:
   - **Ownerless and mechanical** (owning bead closed or no owning epic): fix it with
     the same symvision decision hierarchy.
   - **Stale `--epic-symbol` rows** (for example, `sase-1jm.3.1(...)` once that
     in-progress epic closes): drop them only if the bead is closed and the symbol has a
     real consumer.
   - **Owned by an in-progress epic**: leave it and report it in the final response.
6. After local verification passes, close both task beads with notes that name the
   change and the passing `just _lint-symvision` run:
   - `sase bead close sase-1jp --note "..."`
   - `sase bead close sase-1js --note "..."`

   Do not wait on CI. The next Master Gate run on the landed commit confirms the green
   `lint` job.
