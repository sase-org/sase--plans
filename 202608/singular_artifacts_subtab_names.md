---
tier: tale
title: Rename the Artifacts sub-tabs to singular labels
goal:
  The Artifacts tab strip and every surface that echoes a sub-tab's name read Stitch,
  Patch, Bead, Plan, and File instead of their plural forms, with no change to sub-tab
  ids, pane ids, keymaps, or persisted state.
size: medium
proposed_by: bbugyi200.athena.sase-kv.5.w1
create_time: 2026-08-13 10:53:37
status: wip
---

# Plan: Rename the Artifacts sub-tabs to singular labels

## Problem

The Artifacts tab strip renders plural sub-tab names — `Stitches`, `Patches`, `Beads`,
`Plans`, `Files`. They should be singular: `Stitch`, `Patch`, `Bead`, `Plan`, `File`.
Provider-backed document tabs are pluralized by code, not by config, so `Research`
becomes `Researchs`-safe today only because its kind already ends in `s`; the pluralizer
must go too.

## Naming rule (apply this consistently)

Two categories of text exist and only the first one changes:

1. **Name surfaces** — text that _names the sub-tab_: the tab strip, the pane header
   badge, command-palette entries, Textual binding descriptions, the tab quickstart
   jump/cycle rows, the copy-mode context label, and Help-modal section titles. These
   become **singular**.
2. **Prose** — sentences that use the word as a common noun for the pane's contents
   ("Bulk status change (marked Patches)", "bead work items", "Plans have not loaded
   yet", list-panel border titles that title a list of rows). These **stay plural**.

The one wrinkle: prose that _interpolates_ a descriptor label breaks grammatically once
the label is singular. Reword those sentences so they read correctly with a singular
label (`f"{provider_label} documents have not loaded yet."`), rather than re-pluralizing
the label.

## Non-goals — do not change these

- **Sub-tab ids stay plural**: `"stitches"`, `"patches"`, `"beads"`, `"files"`,
  `"ref:plan"`. They are identity and storage — `FIXED_ARTIFACTS_SUBTAB_ORDER`,
  `FIXED_ARTIFACTS_PANE_IDS`, `ARTIFACTS_ACCENTS`, `ARTIFACTS_ICONS`,
  `LEGACY_ARTIFACTS_SUBTABS`, `app.current_artifacts_subtab`, persisted session state.
- **Pane element ids and CSS** (`artifacts-beads-pane`, `#plans-list`, …).
- **Keymap group names and config keys** (`artifacts_beads`, `artifacts_plans`,
  `artifacts_other` copy groups; `default_config.yml` keymap keys and comments).
- **Query language tokens and filter facets.**
- **Icons, accent colors, digit-shortcut assignment, tab order.**
- **List-panel border titles** — `beads_pane.py` `"Beads"`, `files_pane.py` `"Files"`,
  `plans_pane.py` `"Plan pipeline"`. These title a _list of rows_, not the tab.
- **Explicitly configured provider labels.** If a sidecar `ref:` policy sets `label:`
  (or `ref.label`), that string is used verbatim, exactly as today. Do not strip a
  trailing `s` from user configuration.
- `ArtifactPlaceholderPane` in `widgets/artifacts/panes.py` — `ArtifactsView` no longer
  composes it, so its `self.subtab.title()` badge is not user-visible. Leave it alone.

## Implementation

### 1. Descriptor labels — `src/sase/ace/tui/artifact_tabs.py`

- `_fixed_descriptor` (~line 302): the `labels` dict becomes
  `{"patches": "Patch", "stitches": "Stitch", "beads": "Bead", "files": "File"}`.
- `_provider_label` (~line 531): keep the configured-`label` lookups unchanged, then
  drop the pluralizing tail. The derived branch returns the title-cased kind as-is, and
  the empty fallback becomes `"Document"`. Delete the
  `if label.casefold().endswith("s")` / `f"{label}s"` pair.

Every descriptor consumer picks the new labels up for free — the tab strip
(`widgets/artifacts/view.py::_panel_tabs`), command palette
(`tui/commands/catalog.py::_iter_artifacts_subtab_commands`, which yields
`Show Artifacts: File` and the `artifact File` alias), Textual bindings
(`tui/keymaps/bindings.py::_artifact_subtab_bindings` → `Show File`), and the quickstart
rows (`widgets/tab_quickstart.py`, the `Jump:` and `Cycle Artifacts:` lines).

### 2. The second pluralizer — `src/sase/ace/tui/widgets/artifacts/plans_data.py`

`_provider_label` (~line 270) is a duplicate of the same helper and feeds
`PlansSnapshot.provider_label`. Apply the identical change: no `f"{label}s"`, empty
fallback `"Document"`.

### 3. Pane header badges

Each document/artifact pane renders a name badge at the top left, mirroring the sub-tab
label. Singularize:

- `widgets/artifacts/commits_rendering.py:65` — `" Stitches "` → `" Stitch "`
- `widgets/artifacts/beads_rendering.py:48` — `" Beads "` → `" Bead "`
- `widgets/artifacts/files_rendering.py:60` — `" Files "` → `" File "`
- `widgets/artifacts/plans_rendering.py:28` — `" Plans "` is hardcoded and is therefore
  already wrong for non-plan document panes (a Research pane shows `Plans` today). Add a
  `provider_label: str = "Plan"` keyword parameter to `build_plans_scope`, render
  `f" {provider_label} "`, and pass `self.provider_label` from
  `widgets/artifacts/plans_options.py::_scope_text` (~line 258), which lives on the pane
  that already stores it. Keep the `ARTIFACTS_ACCENTS['plans']` accent lookup as is.

### 4. Prose that interpolates a label

- `widgets/artifacts/plans_list.py:71-75`: default `"Plans"` → `"Plan"`, and reword so a
  singular label is grammatical — `f"Loading {provider_label.casefold()} documents…"`
  and `f"{provider_label} documents have not loaded yet."`
- `widgets/artifacts/plans_data_models.py:87`: `provider_label: str = "Plans"` →
  `"Plan"`.
- `widgets/artifacts/plans_pane.py:57`: the `provider_label: str = "Plans"` default
  argument → `"Plan"`.
- Leave the hardcoded prose in `plans_rendering.py` (`"Plans have not loaded yet"`,
  `"# Plans"`, `"# Plans unavailable"`) alone — it is category-2 prose.

### 5. Copy-mode context label

`src/sase/ace/tui/actions/clipboard/_artifacts.py:38-44`
(`_copy_label_for_artifacts_subtab`) feeds the `Unknown copy key (Plans: …)`
notification. Replace `subtab.title()` with an explicit singular map
(`stitches → Stitch`, `patches → Patch`, `beads → Bead`), return `"Plan"` for `ref:` ids
and `"File"` for `files`. Do not resolve descriptors here — this runs on a keypath that
must not trigger provider discovery, and the harness in
`tests/ace/tui/test_artifacts_copy_mode.py` has no projects configured.

### 6. Help modal section titles

Recommended for coherence — the Help modal is the in-app reference for these panes, and
leaving it plural means the app names the same tab two ways.

- `tui/modals/help_modal/patches_artifact_bindings.py:54,96,156` — `"Stitches Pane"`,
  `"Beads Pane"`, `"Files Pane"` → `"Stitch Pane"`, `"Bead Pane"`, `"File Pane"`.
- `tui/modals/help_modal/patches_copy_bindings.py:27,69,111` — `Copy Mode · Stitches`,
  `· Beads`, `· Plans` → `· Stitch`, `· Bead`, `· Plan`. Leave `Copy Mode · Other`.
- `tui/modals/help_modal/filter_model.py:59` — the docstring example cites `Beads Pane`;
  update it to the new title.

**Consequence to accept knowingly:** the Help filter matches a token against section
name, key display, and description, so typing `beads` will no longer match the section
_name_ (`bead` still will, as a substring of `Bead Pane`). This is the only behavioral
side effect of the rename. If the user prefers to keep the Help modal plural, skip this
step entirely — nothing else in the plan depends on it.

### 7. Test fixtures and assertions

- `src/sase/ace/testing/_startup.py:51` — fast-startup descriptor `label="Plans"` →
  `"Plan"`.
- `tests/ace/tui/widgets/test_tab_quickstart.py:39` and its expected quickstart text.
- `tests/ace/tui/widgets/test_changespec_onboarding.py:53-63` — the ordered
  `Stitches/Patches/Beads/Files` strip assertions.
- `tests/ace/tui/test_artifacts_copy_mode.py:61,74` — the
  `"Plans" if subtab == "ref:plan" else subtab.title()` derivation and the `"Plans:"`
  assertion.
- Help-modal tests, only if step 6 is done: `tests/ace/tui/modals/test_help_modal.py`
  (lines 25-35), `tests/ace/tui/test_help_modal_filter.py` (the `"Beads Pane"` waits),
  and `tests/ace/tui/visual/test_ace_png_snapshots_help_panel.py:100-103` (the
  `assert_page_svg_contains(page, "Beads")` check and its explanatory comment).
  `tests/ace/tui/test_help_modal_filter_model.py` builds its own section fixtures and
  needs no change.
- Run the rest of the suite and fix whatever else asserts a label;
  `test_artifact_tab_icons.py`, `test_artifact_tab_digits.py`, and
  `test_keymaps_app_bindings.py` read labels from the resolver and should pass
  untouched.
- **Do not touch** `tests/agents_sync/test_publication.py` (`"Files"` is an agents
  sidecar README section) or the notification-tag tests (`"Beads"` there is a
  notification tag, not an Artifacts sub-tab).

### 8. New regression test

Add a focused test — `tests/ace/tui/test_artifact_tab_labels.py` is a good home — that
locks the rule in:

- every descriptor from `resolve_artifacts_subtabs()` has a label that does not end in
  `s` for the four fixed panes, and the fixed labels are exactly `Stitch`, `Patch`,
  `Bead`, `File`;
- `_provider_label("plan", {})` is `"Plan"` and `_provider_label("", {})` is
  `"Document"`;
- a configured `{"label": "Plans"}` spec still wins verbatim, proving user config is not
  rewritten.

### 9. Documentation — `docs/ace.md`

Update only where the docs enumerate or quote the strip labels, so they stop describing
a UI that no longer exists. Leave prose that refers to pane contents:

- line 71 — the Artifacts row of the tab table.
- lines 98-105 — the digit-order paragraph
  (`Stitches, Patches, and Beads are always 1, 2, 3`;
  `Files, which always renders last`; `Press p in Stitches, Beads, …`).
- lines 370-373 — "The fixed tabs are Stitches, Patches, Beads, and Files;
  provider-backed document tabs such as Plans and Research …".
- lines 462-463 — "The pane is one of the four fixed Artifacts views: …".

Section headings such as `### Beads Pane` and `### Files Pane` should follow step 6's
decision — rename them with the Help modal, or leave them if step 6 is skipped. Do not
sweep the remaining ~30 plural mentions in `docs/ace.md`; those are prose.

### 10. Visual snapshots

Nearly every full-app PNG snapshot renders the Artifacts strip behind whatever it is
testing, so expect large, mechanical churn — the comparable icon commit `7e4ac6d7c`
regenerated 213 PNGs. The labels also get shorter, which can shift `PanelTabStrip`'s
reflow tier on narrow captures (`artifacts_stitches_persistent_filter_80x24`).

Regenerate and then verify clean, both under `/sase_monitor` since the suite is slow:

```bash
just test-visual --sase-update-visual-snapshots
just test-visual
```

Inspect a couple of regenerated Artifacts PNGs (`artifacts_beads_populated_120x40.png`,
`artifacts_plans_populated_120x40.png`) before committing, to confirm the strip reads
`Stitch  Patch  Bead  PLAN  File` and nothing else moved.

## Verification

1. `just install` first — workspace virtualenvs go stale.
2. `just check` inline while iterating.
3. `just test-visual --sase-update-visual-snapshots`, then `just test-visual` clean
   (both via `/sase_monitor`).
4. `just check-full` via `/sase_monitor` before landing, with a `--next` action.
5. Manual smoke: open `sase ace`, press `Tab` to Artifacts, confirm the strip, cycle
   with `[` / `]`, open the command palette and search `artifact` to see
   `Show Artifacts: Bead`, and press `?` for the Help modal.

## Risks

- **Snapshot churn hides a real regression.** Mitigate by reviewing the diff of a few
  Artifacts PNGs by eye rather than trusting the bulk update.
- **A missed label consumer.** After the source edits, grep for the plural forms across
  `src/sase/ace/` and triage each hit against the two categories above before deciding
  to leave it.
- **Reflow tier changes on narrow terminals** are expected, not a bug, but confirm the
  80x24 snapshot still shows a usable strip.
