---
tier: tale
title: Put Artifacts ▸ Files behind a default-off beta feature flag
goal:
  The Artifacts ▸ Files sub-tab is hidden by default behind a new beta feature flag,
  artifacts_files_pane, and every route into it explains how to opt in instead of
  silently landing on Commits.
size: medium
proposed_by: bbugyi200.athena.13k
proposed_with: claude/opus
create_time: 2026-10-24 06:07:26
status: wip
---

# Plan: Put Artifacts ▸ Files behind a default-off beta feature flag

## Goal

Hide the Artifacts tab's **Files** sub-tab by default behind a new `beta` feature flag,
`artifacts_files_pane`. While the pane awaits a possible redesign, users who still want
it can opt in with `sase flag enable artifacts_files_pane` (or
`feature_flags: {artifacts_files_pane: true}`) and restart sase's TUI. The flag only
changes sase's TUI presentation. The artifact-file index, `sase artifact list` /
`sase artifact create`, `@file:` references, the Agents-tab artifact-files key (`a`),
and the Agents-tab Files deck all keep working in both flag states.

This is a `beta` flag because the user asked for one: the Files pane is the unproven
behavior, and a user opts in. Removing the flag later means a redesigned Files pane
ships visible by default: delete the Off (hidden) branch and make the On (shown) branch
unconditional.

## Background (verified in the current tree)

- **Precedent.** The sunset flag `artifacts_agent_pane_retired` already hides one
  Artifacts sub-tab (the Agent pane). Mirror its seams, but with the opposite polarity
  (here, flag **on** = pane shown):
  - Helper `artifacts_agent_pane_retired()` in `src/sase/ace/tui/_artifact_tab_model.py`
    resolves the flag lazily through `current_flags()`. There is no import-time
    resolution, which `tools/check_feature_flags` rule 4 requires.
  - `src/sase/ace/tui/artifact_tabs.py`: `_build_artifacts_subtabs()` builds the fixed
    strip, and its `resolve_artifacts_subtabs()` cache token includes the flag value so
    both states resolve deterministically in one process.
  - `src/sase/ace/testing/_startup.py::_fast_artifacts_subtabs()` mirrors the production
    strip for fast pilot tests and the visual suite.
    `tests/ace/tui/visual/test_ace_png_snapshots_artifacts_descriptions.py` carries a
    third hand-rolled copy of that strip. Today the Files descriptor is appended
    unconditionally in all three places.
  - `tests/ace/tui/artifacts_contract/test_agents_pane_conformance.py::test_fast_startup_fixed_pane_order_matches_production`
    asserts the fast strip matches production in each flag state.
- **Digits.**
  `src/sase/ace/tui/_artifact_tab_descriptors.py::assign_artifacts_digit_shortcuts`
  always gives Files the highest digit. It already has a "no `files` descriptor" branch
  that falls back to plain positional numbering. Its docstring calls that branch
  unreachable, which becomes false once the pane can be hidden.
- **Silent fallback hazard.** When Files is not in the strip,
  `normalize_artifacts_subtab("files")` returns `"stitches"`.
  `ArtifactsView.entry_navigator("files")` (`widgets/artifacts/view.py`) therefore
  returns the **Commits** pane. So `_artifacts_entry_navigator("files")`,
  `_request_artifacts_entry(<files target>)` (`actions/artifacts_navigation.py`), and
  link-follow (`actions/_link_follow_targets.py::_follow_artifacts_target`) would
  quietly land on Commits and report a miss. `ref:` panes with no provider are already
  guarded against this by `is_unconfigured_ref_pane`. The hidden Files pane needs an
  equivalent guard.
- `file:` refs route to `ArtifactEntryTarget("files", ...)` through
  `relations/artifact_links.py::target_for_ref_kind`. Link-failure toasts are built by
  `actions/_link_follow_toast.py::format_link_failure`, whose kinds today are
  `dangling | load | unconfigured | missing`. The `unconfigured` wording ("configure its
  provider") is wrong for a flag.
- Things that already follow the live strip and need no logic change: the
  command-palette `artifacts.<id>` digit commands (`commands/catalog.py`), the
  onboarding tabs card (`widgets/patch_onboarding.py`), `[`/`]` cycling, digit keys, and
  the `files_*` action and command availability (gated on the current pane being
  `files`, which cannot happen while the pane is hidden).
- Things that need explicit handling:
  - `action_show_artifacts_files` in `actions/artifacts_navigation.py` would silently
    switch to Commits.
  - The help modal's unconditional "Files Pane" section
    (`modals/help_modal/patches_artifact_bindings.py`; the Agent section above it is
    already gated on `contracts.get("agents") is not None`).
  - `sase artifact pane show files` (`src/sase/artifact_cli/pane.py`) would print
    "unknown Artifacts pane". The Agent pane has a friendly retired message in the same
    place.
- Files-pane data loading is local to the pane (`widgets/artifacts/files_*`,
  `relations/files.py`). It does not mount, so it does no work while hidden. No startup
  prewarm exists to gate.
- Flags resolve once per process. `sase flag enable` already prints a "restart sase's
  TUI" notice. The Config hub **Flags** pane lists registered flags automatically.
- **Rust boundary.** This is presentation-only Python. The flag registry and the
  Artifacts strip composition both live in this repo. No `sase-core` change is needed.
- No `sase/memory/` note describes the Files pane, so no memory edits are needed.

## Implementation

### 1. Create the flag (do this first)

Run `sase flag new` exactly once. It creates the `flag` removal task bead and prints the
registry entry. Do not hand-write the registry member, and do not use
`sase bead create`.

```bash
sase flag new artifacts_files_pane -k beta \
  -d "Show the Artifacts ▸ Files sub-tab, a beta pane hidden by default while it is redesigned." \
  --when-enabled "sase's TUI shows the Artifacts ▸ Files sub-tab as the last pane with the highest digit, and file: link follows land on it." \
  --when-disabled "The Artifacts strip omits Files and numbers its panes positionally; file: link follows, the show-Files action, and sase artifact pane show files explain that Files is a disabled beta pane instead of opening it." \
  --remove-when "A redesigned Files pane is ready to ship visible by default; delete the hidden-pane branch then. If the redesign replaces or drops the pane instead, delete this flag together with the old pane."
```

Paste the printed member and `FeatureFlagDefinition` into
`src/sase/feature_flags/registry.py`, keeping the enum and dict keys alphabetically
sorted. Then run `just sync-feature-flags-schema` to regenerate the `feature_flags`
block of `src/sase/config/sase.schema.json`. Keep the printed both-states checklist; the
tests below must cover every item on it.

### 2. Flag helper and disabled-pane hint (`_artifact_tab_model.py`)

- Add `artifacts_files_pane_enabled() -> bool` next to `artifacts_agent_pane_retired()`,
  with the same lazy `current_flags().enabled(FeatureFlag.artifacts_files_pane)` body.
- Add one shared, user-facing hint constant, for example
  `ARTIFACTS_FILES_PANE_DISABLED_HINT`: "Artifacts ▸ Files is a beta pane that is off by
  default. Run `sase flag enable artifacts_files_pane` and restart sase's TUI to show
  it." The navigation toast, the link-follow toast, and the pane CLI must all use this
  constant so the wording cannot drift.
- Re-export both through `src/sase/ace/tui/artifact_tabs.py` (imports and `__all__`), as
  that module already does for `artifacts_agent_pane_retired`.

### 3. One strip builder for production, fast startup, and visual tests

- In `_artifact_tab_descriptors.py`, add a public builder such as
  `build_artifacts_strip(providers: Sequence[ArtifactsTabDescriptor]) -> tuple[ArtifactsTabDescriptor, ...]`.
  It assembles, in order: `agents` (only while `artifacts_agent_pane_retired()` is off),
  `stitches`, `patches`, `beads`, the given providers, then `files` (only while
  `artifacts_files_pane_enabled()` is on). It then returns
  `assign_artifacts_digit_shortcuts(...)`. Check that the import direction avoids a
  cycle. `_artifact_tab_descriptors` already imports from `_artifact_tab_model`.
- Replace the three hand-rolled strips with calls to it:
  - `artifact_tabs._build_artifacts_subtabs()`
  - `sase.ace.testing._startup._fast_artifacts_subtabs()`, which passes
    `(_plan_test_descriptor(),)` and keeps its docstring accurate for both flags
  - the nested `_fast_artifacts_subtabs` override in
    `tests/ace/tui/visual/test_ace_png_snapshots_artifacts_descriptions.py`, which
    passes `(notes_descriptor,)`
- In `resolve_artifacts_subtabs()`, extend the cache token to
  `(artifacts_agent_pane_retired(), artifacts_files_pane_enabled(), token)` and update
  its docstring.
- Update the `assign_artifacts_digit_shortcuts` docstring: the no-Files branch is now
  the normal path while `artifacts_files_pane` is off. Change no logic. With Files
  hidden, Commits/Patches/Beads are 1/2/3, providers take the next digits, and panes
  past the ninth get no digit.

### 4. Guard every route into the hidden pane

- In `artifact_tabs.py`, add `is_disabled_artifacts_pane(pane_id: str) -> bool`. It is
  true only for `pane_id == "files"` while the flag is off. Leave the retired Agent pane
  on its existing routing.
- In `actions/artifacts_navigation.py`, treat a disabled pane like an unconfigured
  `ref:` pane:
  - `_artifacts_entry_navigator(pane_key)` returns `None` for it, never the
    Commits-normalized pane.
  - `_request_artifacts_entry` returns `LinkRequestState.MISSING` for a target on a
    disabled pane, before any sub-tab switch.
  - `action_show_artifacts_files`: when disabled, `notify` the shared hint with
    `severity="warning"` and do not switch panes. Leave its availability rule as is.
- Link follow (`actions/_link_follow_targets.py::_follow_artifacts_target`): right after
  the existing `agent:` branch, if the target pane is disabled (the ref parses as
  `file`, or the routed target's `pane_id` is `files`), do the following and return
  before any scope switch, tab switch, transaction, or trail hop:
  - call `record_link_follow_outcome("missing")`
  - toast through a new `format_link_failure` kind, `"disabled"`, added to
    `LinkFailureKind` in `actions/_link_follow_toast.py`. Title:
    `Cannot follow <short ref>`. Body: the shared hint. Severity: `warning`. Update the
    function docstring's list of cases.
- `file:` link chips keep rendering in relation and link rails. Only following them
  changes.

### 5. Help modal and pane CLI

- `modals/help_modal/patches_artifact_bindings.py`: emit the "Files Pane" section only
  when `contracts.get("files") is not None`, mirroring the Agent section's gate.
- `src/sase/artifact_cli/pane.py::_handle_show`: when `pane_id == "files"` and the flag
  is off, mirror the retired-Agent branch and return 0:
  - **Text:** a one-line bold header plus the shared hint.
  - **`-j`:**
    `{"schema_version": PANE_SHOW_SCHEMA_VERSION, "pane": "files", "disabled": True, "feature_flag": "artifacts_files_pane", "summary": ..., "hint": ...}`.
  - Keep `PANE_SHOW_SCHEMA_VERSION` unchanged, as the retired payload does.

### 6. Tests (cover both flag states)

Tests run with registry defaults, so after this change Files is **hidden** in every test
unless the test opts in.

**New coverage** (a new module such as
`tests/ace/tui/test_artifacts_files_pane_flag.py`, plus extensions next to the existing
precedent tests):

- With the default (off):
  - `resolve_artifacts_subtabs()` has no `files` descriptor, and digits are positional:
    Commits/Patches/Beads are 1/2/3 with no providers, and providers continue from 4.
  - `normalize_artifacts_subtab("files")` and the legacy `"other"` both resolve to
    `"stitches"`.
  - `is_disabled_artifacts_pane("files")` is true.
- With `override_flags(artifacts_files_pane=True)`: Files renders last with the highest
  digit. Keep the existing `test_artifact_tab_digits.py` expectations under this
  override.
- Extend
  `test_agents_pane_conformance.py::test_fast_startup_fixed_pane_order_matches_production`,
  or add a sibling test, to cover all four combinations of
  `artifacts_agent_pane_retired` × `artifacts_files_pane`.
- Pilot tests:
  - **Disabled:** `action_show_artifacts_files` shows the hint toast and leaves the
    current pane unchanged. Following a `file:` link shows the `disabled` toast, does
    not switch tabs or panes, and records no trail hop. `_request_artifacts_entry` on a
    `files` target returns `MISSING`.
  - **Enabled:** the existing Files link-follow tests keep passing under the override.
- A `format_link_failure("disabled", ...)` wording test next to the existing toast
  tests.
- Help modal: a Files twin of `test_help_modal_agent_pane_section_follows_retired_flag`,
  covering both states.
- `tests/main/test_artifact_pane.py`: `pane show files` text and JSON when off, and the
  normal contract output when on.

**Opt existing Files-dependent tests into the flag.** About 50 test modules reference
the Files pane, for example:

- every `tests/ace/tui/test_artifacts_files_*.py`
- the Files link-follow, copy, marks, relation, limit, and pane-state tests
- `tests/ace/tui/visual/test_ace_png_snapshots_artifacts_files*.py`
- `tests/main/test_artifact_pane.py`
- command-availability and keymap tests that select the `files` pane
- `tests/ace/tui/bench_artifacts_jk.py` and `tests/perf/bench_artifacts_first_paint.py`,
  if they switch to Files

Find the failures by running the affected modules with the flag at its default. Fix each
one by enabling the flag rather than by changing its assertions:

- Prefer explicit `with override_flags(artifacts_files_pane=True):` around the app run,
  as the `artifacts_agent_pane_retired` tests do.
- A module that is entirely about the Files pane may use a module-level autouse fixture
  that enters that override.
- Rewrite an assertion only where it encodes the default strip, for example a test
  asserting Files exists by default. In that case assert the new default and add the
  enabled case.

### 7. Visual goldens

- The Files visual tests (`..._artifacts_files.py`, `..._artifacts_files_empty.py`,
  `..._artifacts_files_view.py`) run with the flag on, so their goldens
  (`artifacts_files_*_120x40.png`) must stay byte-identical.
- Every other golden that renders the Artifacts tab strip will lose its Files tab. Run
  `just fix-tui-screenshots` through `/sase_monitor` (`TESTING` / `TESTED`, generous
  timeout), then inspect the report and the golden diff before finalizing. Expected
  result:
  - updates appear only where the Artifacts strip is visible, with the Files tab gone
  - no `artifacts_files_*` golden changes
  - no unexplained creations or removals

  Investigate anything else rather than accepting it. Treat a `partial` status as
  unfinished.

### 8. Docs (Reference pages only; regenerate generated pages)

- `docs/reference/tui/files-and-preview.md`, `## Files Pane` intro:
  - Files is a beta pane, off by default, enabled with
    `sase flag enable artifacts_files_pane` or
    `feature_flags: {artifacts_files_pane: true}`, then a sase TUI restart.
  - While hidden, `file:` link follows show a toast instead of opening the pane, and
    `sase artifact list` still covers the same index.
  - Fix the "one of the four fixed Artifacts views" sentence accordingly.
- `docs/reference/tui/tabs-and-panes.md`:
  - the Artifacts row of the tabs table
  - the digit paragraph: Commits/Patches/Beads 1-3, providers next, and Files (when
    enabled) last with the highest digit
  - the later "fixed tabs are Commits, Patches, Beads, and Files" sentence
- `docs/reference/config/feature-flags.md`: add the beta flag next to the existing
  `artifacts_agent_pane_retired` paragraph. That paragraph's "One sunset flag is
  currently registered" claim is already inaccurate, so reword it to describe the TUI
  flags without a count.
- `docs/reference/upgrading/tui.md`: add an
  `## Artifacts Files pane {#artifacts-files-pane}` section next to
  `#artifacts-agent-pane`. It says how to restore the pane, and the reference pages
  above link to it.
- Run `just fix`. It regenerates `docs/reference/config/integrations.md`, the docs map,
  and formatting. Do not hand-edit generated pages, and do not move pages.

## Verification

1. `just sync-feature-flags-schema` and `just fix` leave a clean regenerated tree.
2. `sase tool run check` passes. It covers the static rules of
   `tools/check_feature_flags`:
   - the registry entry names its bead
   - the schema block matches the registry
   - the key has a non-test reference
   - no import-time resolution
3. The monitored `just fix-tui-screenshots` run reaches `applied` or `clean`, and the
   report review in step 7 is done.
4. Manual sanity check in a pilot test or a live TUI:
   - default strip: Commits 1, Patches 2, Beads 3, then any providers, with no Files tab
   - with the flag on, the strip is identical to today's

Do not run `just check-full`.

## Non-goals

- No redesign or deletion of Files-pane code, its keymaps (`files_*` entries in
  `src/sase/default_config.yml` stay as they are), its query profile, or its saved
  queries.
- No changes to artifact-file indexing, the `sase artifact` CLIs other than
  `pane show files`, `@file:` reference expansion, or Agents-tab artifact-file actions.
- No one-time "pane moved" toast like the Agent pane retirement's. Nothing replaced the
  Files pane, and the hint appears wherever a user would try to reach it.
- No VHS demo tape regeneration.
