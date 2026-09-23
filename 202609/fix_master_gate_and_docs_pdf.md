---
tier: tale
title: Fix the red Master Gate and Deploy Docs workflows on master
goal:
  Master Gate lint and sharded tests pass deterministically on Python 3.12 CI and the
  docs handbook PDF builds well under its 22 MiB gate, so Deploy Docs succeeds again.
size: medium
proposed_by: bbugyi200.athena.0pn
create_time: 2026-09-23 07:00:25
status: wip
---

# Fix the red Master Gate and Deploy Docs workflows on master

## Context

Every `Master Gate` run on `master` has been red for ~a day, and `Deploy Docs` started
failing at `07d867e61` (docs refresh). `actstat` plus the failed-job logs of the last
~12 gate runs show one lint failure, six distinct test failures (four deterministic,
three load/timing dependent), and one docs-PDF size failure. Each root cause below was
reproduced locally (Python 3.12 venv with lockfile deps, CPU-contention loops, or a
scratch docs build) and each fix was prototyped and validated in a scratch copy. None of
these failures is a flake to retry; all need code changes.

| #   | CI job / test                                                                                                             | Frequency         | Root cause (short)                                                                             |
| --- | ------------------------------------------------------------------------------------------------------------------------- | ----------------- | ---------------------------------------------------------------------------------------------- |
| 1   | `lint` → `_lint-symvision`: `delete_paths_in_background` unused public                                                    | every run         | `505934a63` made an in-file helper public                                                      |
| 2   | `tests/test_agent_artifact_directory_operation_audit.py::test_artifact_directory_operation_sites_are_reviewed`            | every run         | `505934a63` moved dir ops into new functions; allowlist not updated                            |
| 3   | `tests/main/test_notify_rules.py::test_help_documents_the_options`                                                        | every run         | asserts the Python ≥3.13 argparse rendering; CI runs 3.12                                      |
| 4   | `tests/ace/tui/modals/test_preview_panel_modal_geometry.py::test_preview_modal_resize_recomputes_geometry`                | every gate run    | CI gets Textual 8.2.8, which delivers screen Resize via an ~8 ms timer; test uses fixed pauses |
| 5   | `tests/ace/tui/test_epic_panel_arrival_frames.py::test_a_cosmetic_epic_change_beside_the_bucket_move_is_patched_in_place` | last ~4 runs      | test harness leaves `_agents_refresh_active_source="watcher"` set after its apply              |
| 6   | `tests/ace/tui/test_usage_header.py::test_usage_only_changes_do_not_move_control_row`                                     | ~half of runs     | fast AcePage harness launches a real usage-refresh subprocess → proc chip appears              |
| 7   | `tests/test_launch_proc_runtime.py::test_proc_dispatch_rebinds_launch_hold_and_settlement_releases_it`                    | ~4 of 9 runs      | product ordering race: terminal proc row published before its hold is released                 |
| 8   | `Deploy Docs` → `just docs-pdf-check`: "PDF is 22.2 MiB, above the 22 MiB limit"                                          | since `07d867e61` | rounded inline-code chips make Chromium emit ~9.8 MiB of Bézier path data                      |

## Implementation

### 1. Symvision: re-privatize `delete_paths_in_background`

`src/sase/_linked_repo_workspaces.py`: `505934a63` renamed `_delete_paths_in_background`
to `delete_paths_in_background`, but its only callers are in the same file
(`move_aside_for_background_delete` and three calls in `clear_workspace_repos`). No test
or other module references it. Rename it back to `_delete_paths_in_background` (the
definition plus the four in-file call sites). Keep `move_aside_for_background_delete`
public, because `workspace_provider/_utils_checkout.py` and
`axe/run_agent_runner_setup_linked_repos.py` import it.

### 2. Directory-operation audit allowlist

`tests/test_agent_artifact_directory_operation_audit.py`,
`_REVIEWED_DIR_OPERATION_CONTEXTS`. `505934a63` moved the corrupt-checkout
`shutil.rmtree` out of `ensure_git_clone_at` into the new `_remove_corrupt_checkout`,
and added the `os.rename` in `move_aside_for_background_delete`. Make three changes:

- Remove the `"src/sase/workspace_provider/_utils_checkout.py:ensure_git_clone_at"`
  entry. That function no longer performs a whole-directory operation directly.
- Add `"src/sase/workspace_provider/_utils_checkout.py:_remove_corrupt_checkout"` with
  an `exemption`. It rescues the sidecar clones of a corrupt numbered workspace
  checkout, then moves the checkout aside for background deletion, falling back to
  `rmtree`. That is a workspace checkout, not an agent artifact directory.
- Add `"src/sase/_linked_repo_workspaces.py:move_aside_for_background_delete"` with an
  `exemption`. It renames a caller-supplied path to a unique same-parent trash sibling
  and deletes it in the background. Its callers pass only numbered-workspace checkouts
  (corrupt-checkout recovery, `recreate_managed_workspace`) or linked-repo clone
  workspaces, never an agent artifact directory.

Keep the dict's existing grouping style and put each entry near its file's siblings.
`test_reviewed_dir_operation_sites_declare_coverage` must still pass: exactly one
coverage kind per entry.

### 3. `sase notify rules -h` help assertion (Python-version portable)

`tests/main/test_notify_rules.py::test_help_documents_the_options` asserts
`"-e, --explain ID"`, which is argparse's 3.13+ rendering. Python 3.12, the CI and
`requires-python` floor, renders `-e ID, --explain ID`. The repo already has a helper
for exactly this case. Import `assert_metavar_option_documented` from
`tests.main.parser_help_helpers` and replace that one assertion with
`assert_metavar_option_documented(out, "-e", "--explain", "ID")`. Keep the `-j, --json`
assertion (it has no metavar), the `--explain` before `--json` ordering check, and the
docstring check.

### 4. Preview modal resize test: wait for the behaviour, not a pause count

Root cause:

- CI's `just install` runs `uv pip install -e ".[dev]"`, which ignores `uv.lock`, so the
  gate installs Textual 8.2.8 and Rich 15.0.0.
- Local venvs and the lockfile have Textual 8.0.1. Only the `visual` extra pins it.
- From Textual 8.2.3 on, `App._on_resize` updates `app.size` immediately, but forwards
  the Resize to the screen from a `set_timer(1/120, ...)` callback.
- The repo patches `Pilot.pause()` to `settle_pilot` (`tests/ace/tui/conftest.py` →
  `src/sase/ace/testing/settle.py`), which never waits on timers.
- So `PreviewPanelGeometryMixin.on_resize` has not run when the test asserts.
- The product code is correct.

Fix in `tests/ace/tui/modals/test_preview_panel_modal_geometry.py`: add
`from sase.ace.testing import wait_for`, the established pattern used by e.g.
`tests/ace/tui/test_panel_tab_strip_compact.py`. In
`test_preview_modal_resize_recomputes_geometry`, replace the three `pilot.pause()` calls
after `resize_terminal(160, 60)`, and the `_last_geometry_screen == (160, 60)` assert,
with:

```python
        await pilot.resize_terminal(160, 60)
        await wait_for(pilot, lambda: modal._last_geometry_screen == (160, 60))  # noqa: SLF001
        container = modal.query_one("#preview-modal-container", Container)
```

Keep the `_geometry_floor == PanelGeometry(150, 58)`, `styles.height.cells == 58`, and
`outer_size.height >= before.height` assertions. `wait_for` backs off with real sleeps,
so it works whether Resize delivery is immediate (8.0.1) or timer-delayed (8.2.8). Also
scan the other `resize_terminal` call sites under `tests/ace/tui/`. Convert any that
assert a resize-driven result after only bare pauses to the same `wait_for` form. The
investigation found none besides this one, but re-check.

### 5. Epic arrival-frames harness: scope the refresh source to its apply

In `tests/ace/tui/_epic_arrival_frames.py`, the `apply()` closure (~line 308) sets
`app._agents_refresh_active_source = "watcher"` and never resets it. Production resets
it in a `finally` right after the apply
(`src/sase/ace/tui/actions/agents/_loading_refresh.py` ~236/271). A background live-hint
pass then patches two more rows, and those trace records read the app-wide source
(`_display_panel_patches.py` ~112). Under CPU load the pass finishes before the fleet
follow-up resets the source, so its two row patches are attributed to `"watcher"`,
giving `3 == 1`. This was reproduced 6/6 with `taskset -c 0` plus busy loops. Mirror
production:

```python
            app._agents_refresh_active_source = "watcher"
            try:
                app._apply_loaded_agents_prepared(...)  # existing arguments unchanged
            finally:
                # Scope the source to this apply, as _loading_refresh.py does, so
                # pump-free follow-ups (live hints, bead warmup) are not attributed to it.
                app._agents_refresh_active_source = "unknown"
```

### 6. Fast AcePage harness: stop launching the real usage-refresh subprocess

Root cause:

- After startup, `_startup_loads.py` (~296-301) schedules
  `AceApp._schedule_usage_refresh_fallback`
  (`src/sase/ace/tui/actions/_usage_refresh_fallback.py`).
- That calls `request_due_usage_refresh(origin="ace")`, which spawns a real
  `python -m sase.llm_provider.usage.refresh_runner` proc.
- `ProcIndicator` then goes from 0 to 1 partway through the test and shifts the control
  row.
- The visual-snapshot harness already suppresses this
  (`tests/ace/tui/visual/_ace_png_snapshot_startup.py`); the fast harness does not.

Fix in `src/sase/ace/testing/_startup.py`:

- Add
  `_ORIGINAL_SCHEDULE_USAGE_REFRESH_FALLBACK = AceApp._schedule_usage_refresh_fallback`
  next to the other `_ORIGINAL_*` captures.
- Add `("_schedule_usage_refresh_fallback", _ORIGINAL_SCHEDULE_USAGE_REFRESH_FALLBACK)`
  to the `for name, original in (...)` list that `_patch_method_if_unchanged(...)`
  replaces with `_noop_startup_service`. `_patch_method_if_unchanged` leaves tests that
  install their own override untouched.

That change alone makes three tests fail every time with
`NoMatches '#provider-usage-detail'` in `ProviderUsageModal.on_mount`:
`test_fallback_clicks_open_usage`,
`test_header_icon_keeps_hit_target_and_palette_works_at_zero_space`, and
`test_explicit_provider_argument_wins_over_header_selection`.

- Each of those tests ends right after `page.expect_modal(...)`.
- `AcePage.expect_modal` returns once `app.screen` is the modal, before the modal has
  finished mounting, so teardown races the mount.
- Harden `AcePage.expect_modal` in `src/sase/ace/testing/ace_page.py` (~line 440) to
  also wait for the mount:

```python
        await self.expect_state("modal", name, timeout=timeout)
        app = self.app
        await _poll_until(
            lambda: app.screen.is_mounted,
            is_success=bool,
            settle=lambda: settle_helpers.settle_pilot(self._pilot),
            timeout=timeout,
            timeout_message=lambda: (
                f"expect_modal({name!r}) timed out after {timeout}s"
                " — the modal was pushed but never finished mounting"
            ),
            clock=app._loop.time if app._loop is not None else None,
        )
```

Follow the style of the neighbouring `expect_screen_contains`. With both changes, the
prototype passed the whole `tests/ace` suite plus `tests/test_ace_testing.py` with no
new failures, and `test_usage_header.py` passed 5/5 under contention.

### 7. Proc settlement: release proc holds before publishing the terminal row (product fix)

Root cause in `src/sase/procs/settlement.py` `settle_proc_shell`:

- `finish_proc(...)` makes the terminal status visible first.
- Only afterwards does `_release_proc_holds(finished)` run: a cold import of
  `sase.core.agent_hold_facade`, a Rust `list_holds`, then the release.
- The gap measured 26-87 ms. Any observer, including `wait_for_proc`, can see `success`
  while the `proc:<id>` hold is still stored.
- This breaks the function's own contract, "Run remaining settlement checkpoints, then
  publish the terminal row".
- A 0.5 s injected delay reproduced the failure deterministically.

Fix:

- Right after `_mark(state, "result_written")` and before `finish_proc(...)`, call
  `_release_proc_holds({"proc_id": proc_id, "status": status})`, with a short comment
  explaining why.
- `release_proc_agent_holds` already accepts a `Mapping`, and `status` is always
  terminal at this point.
- Widen the helper's annotation to
  `def _release_proc_holds(proc: Proc | Mapping[str, Any]) -> None`, importing `Mapping`
  from `collections.abc` to match repo style.
- Keep the existing `_release_proc_holds(finished)` after `finish_proc` as an idempotent
  backstop, and keep the early-return release for already-terminal rows.

Crash safety is unchanged: a crash between release and finish leaves the proc
`settling`, and resumed settlement finishes it.

Add a deterministic regression test near `tests/test_launch_proc_runtime.py`, or a new
focused test file. Arm a `proc:`-kind hold for a proc, and wrap or monkeypatch
`sase.procs.settlement.finish_proc` so it records `list_agent_holds_without_liveness()`
at call time. Drive `settle_proc_shell` to completion and assert the recorded list
contains no hold for that proc. That ordering property is what the flaky end-to-end test
depends on. Do not weaken the existing end-to-end assertion: it deliberately reads the
raw store without liveness pruning.

### 8. Docs PDF size: square inline-code chips in the PDF stylesheet

Findings from a local build of the handbook:

- Output was 21.8 MiB locally; CI builds about 0.4 MiB larger because of runner fonts.
- Page content streams are 72% of the bytes, and about 95% of the path data is inline
  `code` chips.
- Material gives those chips a small `border-radius`. Chromium's PDF backend draws every
  rounded corner as several full-precision Bézier segments, across roughly 37k chips.
- Fonts (~1.9 MiB) and images (~2.2 MiB) are not the driver.

Fix in `docs/stylesheets/pdf.css`, which only `mkdocs-pdf.yml` loads, so the HTML site
is unaffected. Add this rule just above the existing
`.md-typeset pre, .md-typeset .highlight > pre` block:

```css
/* Chromium's PDF backend draws every rounded corner as several full-precision
   Bezier segments; square code chips emit one compact rectangle instead, which
   keeps the handbook's page content streams several MiB smaller. */
.md-typeset code,
.md-typeset kbd {
  border-radius: 0;
}
```

**Gotcha:** do not put a backtick character anywhere in `pdf.css`, not even in a
comment. With one, the mkdocs-exporter/Paged.js build timed out waiting for
`body[mkdocs-exporter="true"]`, reproduced twice.

- Measured result: 21.82 → 11.99 MiB, `tools/validate_docs_pdf` passes at 987 pages, and
  sampled pages render the same (0-2 px differences at 60 dpi).
- Keep `MAX_SIZE_BYTES = 22 * 1024 * 1024` in `tools/validate_docs_pdf`. Cloudflare
  Workers static assets cap a single file at 25 MiB, and the gate is that cap minus
  headroom.
- Add a one-line comment above the constant saying so, so nobody "fixes" a future
  overage by bumping toward 25 MiB.
- Do not add the optional `TJ` text-merging or JPEG-quality postprocess changes; they
  are unnecessary with ~10 MiB of new headroom.

## Follow-up beads

Before creating either bead, use `/sase_new_task`, which checks for duplicates:

- **CI does not install from `uv.lock`**. `just install` uses
  `uv pip install -e ".[dev]"`, so CI fast lanes run Textual 8.2.8 / Rich 15.0.0 while
  local dev runs 8.0.1 / 14.3.3. This is the underlying reason fix #4 was CI-only.
  Options: install from the lockfile in CI, or pin `textual`/`rich` in base deps. Type
  `bug`, size `large`.
- **`dispatch_proc_unit` terminal check uses a stale proc row**. In
  `src/sase/agent/launch_proc_runtime.py` (~73-80), `current` is read before
  `_rebind_launch_hold_to_proc`. A very fast command can finish before the rebind, and
  then the terminal check misses it, leaving the rebound `proc:` hold in the raw store
  until liveness pruning or TTL. Suggested fix: re-read with `get_proc(...)` after the
  rebind. Type `bug`; `small` is justified because the root cause is precise.

## Out of scope (do not change)

- The `release-please--branches--master` PR's `release-core-floor-smoke` job fails
  because published `sase-core-rs==0.34.72` lacks the `project_tag_*` and
  `tool_run_observe` bindings. The fix is release sequencing: publish sase-core v0.34.73
  (its open release PR), then raise the floor. That is a user-driven release action, not
  a code fix here.
- `sase-telegram` CI is also red, but that is a separate repo.

## Verification

1. `just install` if the workspace venv is stale, then `just fix`.
2. Targeted runs: `.venv/bin/python -m pytest` on
   - the eight failing tests above (except the PDF), their files, and the new settlement
     regression test;
   - `tests/test_ace_testing.py`;
   - the three `ProviderUsageModal` tests named in #6;
   - `tests/ace/tui/test_usage_header.py`.
3. If practical, recheck #3 and #4 in a throwaway Python 3.12 venv:
   `uv venv --python 3.12`, `uv pip install -e ".[dev]"`, which pulls Textual 8.2.8 like
   CI. Delete the venv afterwards.
4. `just _lint-symvision` passes.
5. `sase tool run check`: the agent-default whole-repo lint plus scoped tests. Do not
   run `check-full`.
6. Docs PDF: run `just docs-pdf-check` through `/sase_monitor`; it takes several
   minutes. Confirm `[validate_docs_pdf] ok` reports roughly 12 MiB, then remove the
   generated `site/` output if it is untracked. `playwright install chromium` may
   replace browser builds in the shared `~/.cache/ms-playwright`; that is expected.
