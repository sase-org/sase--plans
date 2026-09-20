---
tier: tale
title: Show which LLM providers update in the `,U` panel and the completion toast
goal:
  The `,U` Update panel names every captured provider with its installed-to-latest
  version transition and marks manual-only providers by name, and the in-session
  completion toast reports each provider's outcome with the same per-provider fidelity
  the post-restart receipt toast already has.
size: medium
proposed_by: bbugyi200.apollo.18
create_time: 2026-09-20 17:01:50
status: wip
---

# Show Which LLM Providers Update — In The `,U` Panel And In The Completion Toast

## Problem

Two places in the comprehensive-update flow under-report provider (agent CLI) work:

1. **The `,U` Update panel** (`UpdatePanel`, projected by
   `src/sase/ace/tui/update_panel_state.py`) shows the providers row as a single
   truncated line of up to four display names plus a manual-steps count — e.g.
   `claude, codex · 1 needs manual steps`. It never shows the version transition each
   provider will make, never says _which_ provider needs manual steps, and the
   `Everything` row — the default highlight and the one `,E` applies — carries no detail
   at all unless a source failed.
2. **The in-session completion toast** raised by `_on_scoped_update_complete`
   (`src/sase/ace/tui/actions/update_run.py:214`) renders
   `comprehensive_update_summary(result)`, which is only bucketed counts:
   `Agent CLIs: 2 updated, 1 current`. This is the toast the user sees for a
   providers-only update (no SASE code change → no restart), so it is the _only_ report
   of that work. It names no provider and shows no versions.

The post-restart receipt toast is not in scope and must not regress: it already renders
good per-provider lines via `_provider_result_lines()` in
`src/sase/ace/tui/actions/post_update_toast.py:157`. That renderer is the quality bar
and the reuse target for item 2.

## Goal

The panel tells the user exactly which providers will be updated and from which version
to which version, and the completion toast tells them exactly which providers were
updated, to what, and what failed or needs manual steps — with the same per-provider
fidelity the post-restart toast already has.

## Evidence already in memory — no new I/O

`,U` dispatch is allocation-only by design (`action_update_sase_shortcut`,
`src/sase/ace/tui/actions/base.py:228`): it projects the cached `UpdateStatus` snapshot
and pushes the panel; preview I/O only starts after a row is chosen. Everything this
plan adds to the panel already exists in that cached snapshot: `ProviderUpdateCandidate`
(`src/sase/updates/status.py:50`) carries `display_name`, `installed_version`,
`latest_version`, and `manual_only`. **Do not add any I/O, config load, or live
inventory read to the keystroke path.**

Likewise `ComprehensiveUpdateResult.provider_results` is a tuple of
`AgentCliUpdateResult` (`src/sase/agent_clis/models.py:179`) carrying `display_name`,
`status`, `old_version`, `new_version`, `reason`, `command`, `suggested_command`, and
`docs_url`. The completion toast needs no new data either.

## Hard constraints

- `src/sase/ace/tui/modals/update_panel.py` must not import `sase.updates` or
  `sase.agents_sync`; `test_update_panel_module_does_not_import_update_backends`
  (`tests/ace/tui/test_update_panel.py:161`) enforces it. All provider projection and
  string formatting therefore stays in `src/sase/ace/tui/update_panel_state.py`; the
  modal only renders the already-projected strings.
- Do not import `plugins_browser_comprehensive_update_preview` (or any other modal /
  `uv_tool` module) from `update_panel_state.py` just to reuse its private
  `_version_transition`. Keep a tiny local formatter in the state module.
- This is presentation-only Textual/Python work. It does not cross the Rust core
  boundary and nothing here belongs in `sase-core`.
- No feature flag: this is not a deprecation, not an unfinished feature, and no old
  branch must stay reachable.
- No new config fields. The existing `ace.updates.post_update_toast_*` knobs gate the
  _post-restart_ toast only and keep their current meaning; the two surfaces changed
  here use module constants for their caps.

## Design

### 1. Panel rows carry multiple detail lines

In `src/sase/ace/tui/update_panel_state.py`, replace
`UpdateOptionRow.detail: str | None` with `details: tuple[str, ...] = ()` and update
every producer and consumer:

- `_row(...)` takes `details: tuple[str, ...]` instead of `detail: str | None`.
- `_sase_row`, `_restart_row`, `_single_source_kind`-driven error paths: wrap their
  existing single string in a one-tuple (or `()` when `None`). Their rendered output
  must not change.
- `_everything_row`'s failed-source branch keeps de-duplicating the child rows' error
  strings, now reading `row.details` instead of `row.detail`.

In `src/sase/ace/tui/modals/update_panel.py`, `_row_prompt` loops over `row.details`,
appending each on its own line with the existing `dim {accent}` style. Rendering of a
one-element tuple is byte-identical to today's single-detail rendering.

### 2. Providers row gets one line per provider

Add a `_provider_details(status)` projection to `update_panel_state.py` replacing
`_providers_detail`. For each `ProviderUpdateCandidate` in `status.provider_candidates`,
emit:

```
• claude-code   2.1.0 → 2.2.0
• codex         0.9.1 → 1.0.0 · manual steps
• gemini-cli    unknown → 1.4.0
```

- Version text: `f"{installed} → {latest}"`, substituting `unknown` for an empty or
  missing side (small private helper local to this module; do not import the modal
  package's version-transition formatter).
- Append ` · manual steps` to a candidate whose `manual_only` is true, so the user
  learns _which_ provider needs manual work instead of only a count. The old trailing
  `N needs manual steps` clause is removed — it is now redundant and per-name.
- Pad the display-name column to the widest shown name so the arrows align, bounded by a
  name-truncation cap so one long name cannot push a row past the panel's usable width
  (container width is 78 with `padding: 1 2`, so keep lines ≤ ~70 cells; see
  `UpdatePanel > #update-panel-container` in `src/sase/ace/tui/styles.tcss:1392`).
- Cap the list at a module constant `_PROVIDER_DETAIL_LIMIT = 6`; when more candidates
  exist, append a final `• +N more providers` line. Keep the existing
  `_DETAIL_NAME_LIMIT` constant only if it still has a caller; otherwise remove it so
  symvision does not report it unused.
- A failed provider source keeps its existing single error line and shows no per-
  provider lines. An unknown or already-current providers row still shows no details.

The list max-height is 24 (`UpdatePanel #update-panel-list`) and the container is
`height: auto`, so up to 7 provider lines fit without redesigning the modal; the
`OptionList` scrolls if the restart row is also present.

### 3. Everything row gets a real summary line

`_everything_row` currently has `detail=None` unless a source failed. Give it one
compact combined line when work is available, so the default-highlighted row that `,E`
applies actually describes the work:

```
sase 1 · sase-core 1 · plugins 2 · core rebuild · providers 2 (1 manual)
```

Build it by reusing the SASE row's existing breakdown string and appending a provider
clause derived from `status.agent_cli_count` and `status.manual_agent_cli_count` (omit
the `(N manual)` parenthetical when zero, omit each half when that leg has no work). Do
**not** repeat the full per-provider list on the Everything row — that is what the
providers row is for, and duplicating it would push the panel into scrolling for the
common case. The failed-source branch keeps precedence over this new line.

### 4. Shared per-provider toast renderer

Extract the existing renderer so the two toasts cannot drift:

- Move `_provider_result_lines` out of `src/sase/ace/tui/actions/post_update_toast.py`
  into a new module `src/sase/ace/tui/actions/_update_provider_toast_lines.py` as
  `provider_result_lines(results, *, overflow=0) -> str`, taking a sequence of
  `ProviderUpdateReceiptResult` and returning the same Textual markup it produces today
  (heading `[bold]Agent CLIs[/]`, `•` bullets, `updated` version transition,
  `already current`, `failed — reason`, `manual — reason` plus the indented `shlex.join`
  command, `skipped — reason`, and the `…and N more provider results` overflow line).
  `post_update_toast.py` imports and calls it with `receipt.provider_results` /
  `receipt.provider_overflow`. Its rendered output must be unchanged —
  `tests/ace/tui/test_post_update_toast.py` and the `post_update_toast_120x40` golden
  both pin it.
- Promote `_provider_receipt_result` in `src/sase/ace/_update_receipt_builders.py` to a
  public
  `provider_receipt_result(result: AgentCliUpdateResult) -> ProviderUpdateReceiptResult`,
  keep the existing internal caller pointing at it, and re-export it from
  `src/sase/ace/update_receipt.py` (add to `__all__`). This is the single supported
  `AgentCliUpdateResult → ProviderUpdateReceiptResult` conversion; do not write a second
  one, and do not reach into a private name from another module (symvision flags private
  misuse).

### 5. Rich in-session completion toast

Add `src/sase/ace/tui/actions/_update_completion_toast.py` with

```python
def completion_toast(result: ComprehensiveUpdateResult) -> CompletionToast
```

returning a small frozen dataclass of `title`, `message` (Textual markup), and
`severity`. It must:

- Include a SASE-leg line when `UpdateLeg.SASE in result.selected_legs`, carrying
  `result.sase.message` (escaped) under a `[bold]SASE, core & plugins[/]` heading.
- Include the provider block when `UpdateLeg.PROVIDERS in result.selected_legs`, by
  mapping `result.provider_results` through `provider_receipt_result` and rendering with
  `provider_result_lines`, capped by a module constant (reuse `MAX_PROVIDER_LINES` from
  `src/sase/ace/_update_receipt_models.py` so both toasts cap identically) and passing
  the remainder as `overflow`.
- Append `result.provider_error` as a failed `Agent CLIs` entry, exactly as
  `_build_comprehensive_receipt` already does, so a planning/execution failure is not
  silently dropped.
- Render an empty provider result set as the current truthful `no captured work` wording
  rather than an empty block.
- Derive `title` as `✓ Providers updated` / `✓ Update complete` on success,
  `⚠ Update finished with issues` when `result.has_failures`, and `✕ Update failed`
  when `result.fully_failed`; derive `severity` from the same three-way split the
  current code computes (`information` / `warning` / `error`).
- Escape every interpolated name, version, and reason with `textual.markup.escape`,
  matching `post_update_toast.py`.

Wire it in `src/sase/ace/tui/actions/update_run.py`:

- `_on_scoped_update_complete` keeps `comprehensive_update_summary(result)` for the
  `TrackedProcResult` message and for `_restart_after_update(message)` (that path is a
  one-line pre-restart notice and the post-restart receipt toast already reports in full
  — do not change it), and uses `completion_toast(result)` for the `self._notify` call
  on the non-restart path.
- Extend `UpdateRunActionsMixin._notify` with optional `title: str | None = None`,
  `markup: bool = False`, and `timeout: float | None = None`, forwarding only the
  arguments that were supplied so existing callers and test doubles that accept just
  `(message, severity=...)` keep working. Use a ~10s timeout for the rich toast, as
  `post_update_toast.py` does.
- `comprehensive_update_summary` in
  `src/sase/ace/tui/modals/plugins_browser_comprehensive_update_execution.py` is
  unchanged; `tests/ace/tui/test_plugins_browser_pane_comprehensive_update_execution.py`
  must still pass untouched.

## Tests

Add or extend, in the repo's existing style:

- `tests/ace/tui/test_update_panel_state.py`
  - Per-provider detail lines carry `installed → latest` for each candidate.
  - A `manual_only` candidate carries the per-name `manual steps` marker, and the old
    aggregate `N needs manual steps` clause is gone.
  - More than `_PROVIDER_DETAIL_LIMIT` candidates cap the list and append
    `+N more providers`.
  - A missing installed version renders `unknown → <latest>`.
  - The Everything row's new combined summary line, including the manual parenthetical
    and its omission at zero.
  - A failed provider source still projects the error line and no per-provider lines
    (update `test_failed_provider_source_uses_error_as_detail` and
    `test_manual_only_providers_append_caveat_and_truncate_names` for the new shape).
  - Existing sase/restart-row detail assertions migrate from `detail` to `details` with
    unchanged strings.
- `tests/ace/tui/test_update_panel.py`
  - `_row_prompt` renders each entry of `details` on its own line.
  - The import-weight test must keep passing unchanged.
- `tests/ace/tui/test_post_update_toast.py` — unchanged assertions must pass after the
  renderer extraction; add a direct unit test of `provider_result_lines` covering
  updated / already-current / failed / manual-with-command / skipped / overflow.
- New `tests/ace/tui/test_update_completion_toast.py`
  - Providers-only result names each provider with its version transition.
  - Failure sets `warning`/`error` severity and the matching title.
  - `provider_error` appears as a failed `Agent CLIs` entry.
  - A SASE-leg-selected result includes the SASE line.
  - Empty provider results render the truthful no-work wording.
  - Names/reasons containing markup-significant characters are escaped.
- `tests/ace/tui/visual/test_ace_png_snapshots_update_panel.py` — update
  `_row`/`_pending_state` for `details`, and make the pending fixture exercise the new
  multi-line provider rows and the Everything summary line. The `wait_for_state` /
  `wait_for_svg_contains` predicates must be updated to strings the new projection
  actually emits.

## Verification

1. `just install` if the workspace venv is stale, then `just fix` inline.
2. `sase tool run check` (preferred over raw `just check`). Hand it to `/sase_monitor`
   if it runs long. Do **not** run `just check-full` — it is not requested by this plan.
3. The panel's rendered output changes, so the PNG golden must be refreshed:
   `just fix-tui-screenshots -- update_panel` (targeted), run through `/sase_monitor`
   with the `TESTING` / `TESTED` status pair. Inspect the report under
   `.pytest_cache/sase-visual/` and the golden diff before accepting — generation is not
   approval. `post_update_toast_120x40` must be **unchanged**; if it moves, the renderer
   extraction was not behavior-preserving and that is a bug to fix rather than a golden
   to accept.

## Docs

- `docs/configuration.md` (the `,U` **Update panel** paragraph near line 430): say the
  providers row lists each captured provider with its version transition and marks
  manual-only providers by name, and that the Everything row summarizes both legs.
- `docs/ace.md` (the `,U` paragraph near line 7448): same, plus one sentence that a
  provider update that does not change SASE code reports per-provider results in the
  completion toast instead of restarting.
- `docs/plugins.md` (the Admin Center bullet near line 289): one clause on the richer
  providers row.
- No `src/sase/default_config.yml` or `src/sase/config/sase.schema.json` change — no new
  config fields and no keymap change.

## Out of scope

- The post-restart receipt toast's content and layout (only its renderer moves).
- The Admin Center Updates pane's own `A` / `u` batch toasts
  (`_agent_cli_update_summary`) — a follow-up could adopt the shared renderer, but
  changing it here would widen the diff and the golden surface.
- Adding new update evidence, network calls, or provider metadata.
- The top-bar update indicator and its tooltip.
