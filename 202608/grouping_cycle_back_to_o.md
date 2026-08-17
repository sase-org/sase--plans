---
tier: tale
title: Move ACE grouping-cycle back to o/O and re-key the Artifacts open verb to E
goal:
  cycle_grouping_mode / cycle_grouping_mode_reverse default to `o` / `O` again on every
  surface that has a grouping mode, and the two Artifacts `o` actions (beads_open_bug,
  files_open_external) move to a single shared `E` key with docs, help, footer, palette,
  and snapshot coverage all agreeing.
size: medium
proposed_by: bbugyi200.athena.04j
create_time: 2026-08-17 07:33:00
status: wip
---

# Plan: Move ACE grouping-cycle back to `o` / `O`

## Goal

Restore the pre-`sase-m6.9` muscle memory: `o` cycles the grouping strategy forward and
`O` cycles it in reverse, on the Agents tab and on every Artifacts pane that declares
the `GROUPING` capability. Free `o` by moving the two Artifacts "open the entry's target
outside sase" actions onto one shared new key, `E` ("open **E**xternally").

## Background: how the keys got where they are

- `sase-m6.9` unified six Patch-vs-siblings key collisions. As part of that, grouping
  cycle was moved off `o` / `O` onto `B` / `I`, and `o` was described as "reserved
  app-wide for the unified open verb".
- The unified `artifacts_open_external` action described in the `sase-m6.9` notes and in
  `docs/ace.md` was **never implemented**. `o` is still owned by two separate,
  pane-scoped actions: `beads_open_bug` (Beads pane) and `files_open_external` (Files
  pane). Grep confirms `artifacts_open_external` exists only in a doc sentence and a
  config comment, not in code.
- So this change is a straight key swap, not an action redesign.

Nothing about this work crosses the Rust core boundary: keymaps, help text, and footer
hints are presentation-only Textual/Python state that lives in this repo.

## Current state (verified on `master` in a fresh workspace)

Default keys (`src/sase/default_config.yml`):

| Line | Entry                                 |
| ---- | ------------------------------------- |
| ~317 | `beads_open_bug: "o"`                 |
| ~325 | `files_open_external: "o"`            |
| ~405 | comment claiming `o`/`O` are reserved |
| ~409 | `cycle_grouping_mode: "B"`            |
| ~410 | `cycle_grouping_mode_reverse: "I"`    |

`AppKeymaps` (`src/sase/ace/tui/keymaps/app_keymaps.py`) deliberately declares **no**
field defaults, so `src/sase/default_config.yml` is the single source of truth for
default keys. `src/sase/ace/tui/bindings.py` holds a separate, stale fallback
`DEFAULT_BINDINGS` list; `_state_init_late.py` replaces `self._bindings` wholesale with
`build_app_bindings(registry.app)` at startup, so `DEFAULT_BINDINGS` never decides real
behavior — but it still lies today (`Binding("o", "cycle_grouping_mode")` at line 38,
`Binding("o", "beads_open_bug")` at 160, `Binding("o", "files_open_external")` at 170,
and no binding at all for `cycle_grouping_mode_reverse`).

Per-pane action availability, measured with `check_app_action` against every resolved
Artifacts pane contract:

| Pane       | `GROUPING` capability | owns `o` today        |
| ---------- | --------------------- | --------------------- |
| `patches`  | yes                   | nothing               |
| `stitches` | yes                   | nothing               |
| `files`    | yes                   | `files_open_external` |
| `beads`    | **no**                | `beads_open_bug`      |
| `ref:plan` | **no**                | nothing               |

That table is why the swap is safe: the only pane where grouping-cycle and an `o` action
would both be live is `files`, and both must move for the pane-uniqueness conformance
check to pass.

Everything else that renders these keys derives them from the registry, so it updates
for free: the help modal (`modals/help_modal/patches_bindings.py`, `agents_bindings.py`,
`patches_artifact_bindings.py` all use `d(a.<action>)`), the `[group: <label> (<key>)]`
badges (`widgets/agent_info_panel.py:387`, `widgets/patch_info_panel.py:215`), the Files
footer hint (`widgets/artifacts/files_rendering.py:184`), the Beads footer entry
(`widgets/artifacts/beads_navigation.py:266`), and the command palette
(`commands/_app_metadata.py`, key display is derived).

## Decision: the new key is `E`, shared by both open actions

`E` for "open **E**xternally". Both actions launch something outside the TUI —
`beads_open_bug` opens the issue URL via `webbrowser.open`, `files_open_external` opens
text in `$EDITOR` and media via `xdg-open` — so they are one verb and keep one key, the
way they do today on `o`.

Why `E` is safe despite `edit_panel` already defaulting to `E`:

- `edit_panel`'s palette scope is `AGENTS_AXE` (`commands/_app_metadata.py:426`) and it
  only appears in the Agents and AXE help sections (`help_modal/agents_bindings.py:104`,
  `help_modal/axe_bindings.py:62`).
- `action_edit_panel` (`actions/agents/_panel_detail.py:327`) returns immediately unless
  `current_tab` is `agents` or `axe`. On the Patches pane `E` is already a **silent
  no-op**; the only reason `check_app_action` still reports it as available there is
  that `edit_panel` is absent from the non-PR allowlist gate. It is unavailable on every
  other Artifacts pane because it is not in `NON_PRS_ARTIFACT_ACTIONS`.
- So `E` is effectively unused across the whole Artifacts tab in every user-visible
  surface, and the swap introduces no new Patch-vs-siblings divergence of the kind
  `sase-m6.9` set out to remove.

Simulating the post-change registry (`dataclasses.replace` on the loaded `AppKeymaps`,
then re-running the conformance rule from
`tests/ace/tui/artifacts_contract/harness.py::check_declared_keys_resolve_to_named_actions`
over every resolved pane) reports **no problems** and resolves as:

| Pane       | `o`                   | `O`                           | `E`                   |
| ---------- | --------------------- | ----------------------------- | --------------------- |
| `stitches` | `cycle_grouping_mode` | `cycle_grouping_mode_reverse` | —                     |
| `patches`  | `cycle_grouping_mode` | `cycle_grouping_mode_reverse` | `edit_panel` (no-op)  |
| `files`    | `cycle_grouping_mode` | `cycle_grouping_mode_reverse` | `files_open_external` |
| `beads`    | —                     | —                             | `beads_open_bug`      |
| `ref:plan` | —                     | —                             | —                     |

Rejected alternatives, recorded so this is not re-litigated:

- **`B`, `I`, or `P`** — the only letters with no app-level owner at all (`B`/`I` become
  free through this very change). Zero ambiguity, but no mnemonic for either action, and
  reusing `B` immediately after moving grouping off it means a stray `B` on Files
  suddenly launches `$EDITOR`.
- **`ctrl+e`** — also globally unused with a fine mnemonic, but it downgrades a
  frequently used single-keystroke action to a chord.
- **bang mode `!o` / `!O`** — `!o` is already `mark_pr_origin`, and a two-key sequence
  for a common action is worse than either option above.

If the reviewer prefers a globally unique key over the mnemonic, `P` is the drop-in
substitute: change only the two `E` values in step 1 and the matching assertions, and
skip the `edit_panel` note in step 6.

## Implementation steps

### 1. `src/sase/default_config.yml` — the only source of default keys

- `beads_open_bug: "o"` → `beads_open_bug: "E"`.
- `files_open_external: "o"` → `files_open_external: "E"`.
- `cycle_grouping_mode: "B"` → `"o"`; `cycle_grouping_mode_reverse: "I"` → `"O"`.
- Replace the stale comment above the grouping keys. The new comment should say that `o`
  / `O` own grouping-cycle on every surface with a grouping mode, that the Artifacts
  open-externally verb (`beads_open_bug`, `files_open_external`) shares `E`, and that
  bang-mode `!o` still owns `mark_pr_origin`. Do not reintroduce the
  `artifacts_open_external` name — it does not exist.
- Leave every prefixed-mode `o` alone: copy-mode `artifacts_other.source: "o"` and
  `axe.visible: "o"` (`%` prefix), and bang-mode `mark_pr_origin: "o"` (`!` prefix).
  None of them collide with an app-level key.

### 2. `src/sase/ace/tui/bindings.py` — make the fallback list honest

- Keep `Binding("o", "cycle_grouping_mode", "Cycle Grouping", show=False)` (it is
  already `o`, and is correct again after step 1).
- Add the missing
  `Binding("O", "cycle_grouping_mode_reverse", "Cycle Grouping Rev", show=False)`.
- `Binding("o", "beads_open_bug", ...)` → `"E"`.
- `Binding("o", "files_open_external", ...)` → `"E"`.

No test asserts `len(DEFAULT_BINDINGS)`; `tests/test_keymaps_app_bindings.py` only
indexes it by action name, so adding one entry is safe.

### 3. `src/sase/ace/tui/keymaps/registry.py` — allow the deliberate shared key

Add `frozenset({"beads_open_bug", "files_open_external"})` to
`_CONTEXTUAL_APP_DUPLICATES` (line ~85) with a one-line comment: the two actions are
pane-disjoint by construction (`check_app_action` gates `BEADS_ARTIFACT_ACTIONS` to the
Beads pane and `FILES_ARTIFACT_ACTIONS` to the Files pane), so a user who rebinds one of
them onto the pair's shared key must not have the override silently reverted. This gap
exists today for `o`; the pair sharing a less obvious key makes it worth closing.

### 4. `docs/ace.md`

Line numbers are from the pre-change file and will drift; match on the quoted text.

- Beads pane table (~371): `` `o` `` → `` `E` `` for "Open a linked external issue".
- Beads prose (~378): "When a bead has several issue links, `o`, `% u`, and the `b`-mode
  …" → `` `E` ``.
- Files pane table (~562): `` `o` `` → `` `E` `` for "Open text in `$EDITOR` …".
- Patches Navigation table (~649): `` `B` / `I` `` → `` `o` / `O` ``.
- The note under that table (~654-659): rewrite. It currently claims `o`/`O` are
  reserved for `artifacts_open_external` / bang-mode `mark_pr_origin` and cites
  sase-m6.9. It must now say that `o` / `O` cycle the L0 grouping bucket forward /
  reverse on the Agents tab and on every Artifacts pane that has a grouping mode (each
  surface keeps its own in-session mode), that Beads and Plans have no grouping-mode
  data so the keys are a silent no-op there, that the same is true on the AXE tab, and
  that the Artifacts open-externally verb moved to `E` while bang-mode `!o` still marks
  PR origin.
- PR Grouping and Folding prose (~691): "`B` cycles `BY_PROJECT → BY_DATE → BY_STATUS`"
  → `` `o` ``.
- Agents Navigation table (~895): `` `B` / `I` `` → `` `o` / `O` ``.
- The note under it (~902-906): same rewrite as above, keeping the existing `g`/`G`
  sentence intact.
- Grouping Modes prose (~1713): "Press `o` on the Agents tab to cycle the L0 grouping
  bucket" is already correct but stale-by-accident; extend it to mention `O` for the
  reverse cycle.
- Verify the `[group: <label> (o)]` badge sentence (~1755) needs no edit — it is already
  `o` and becomes correct again.
- Leave the Preview Reader table's `o` (~492) alone. That is a modal-local binding on
  `PreviewPanelModal` (`modals/preview_panel_modal.py:87`); the focused screen's
  bindings win over app bindings, so it does not collide with app-level `o`.
- `docs/artifacts_pane_visual_grammar.md` needs no change — it describes grouping
  behavior without naming these keys.
- Run `just fmt-md` afterwards; the tables are width-aligned and `just fmt-md-check`
  gates them.

### 5. Tests to update

- `tests/test_keymaps_patch_grouping_binding.py` — this file exists specifically for
  this collision. Rewrite the module docstring (it currently narrates the m6.9 `B`/`I`
  state) and the assertions:
  - `reg.app.cycle_grouping_mode == "o"`, `reg.app.cycle_grouping_mode_reverse == "O"`.
  - `reg.app.beads_open_bug == "E"`, `reg.app.files_open_external == "E"`.
  - `reg.app.mark_pr_origin == "unbound"` and
    `reg.bang_mode.keys["mark_pr_origin"] == "o"` (unchanged; keep the catalog
    assertions about `app.mark_pr_origin` having no key display and
    `bang.mark_pr_origin` displaying `!o`).
  - Rename/repurpose `test_patch_pane_o_reaches_neither_grouping_nor_mark_pr_origin`: on
    the Patches pane, `o` must now resolve to exactly `["cycle_grouping_mode"]` and `O`
    to exactly `["cycle_grouping_mode_reverse"]`.
  - Add a table-driven case over `patches`, `stitches`, `beads`, `ref:plan`, `files`
    asserting the post-change resolution table above, reusing the existing
    `_action_enabled` helper (extend it to pass `active_artifacts_contract` so the
    contract-gated grouping actions resolve correctly for non-Patches panes).
- `tests/test_keymaps_registry_loading.py::test_g_and_o_default_bindings_do_not_collide`
  (~line 280) — the docstring's premise inverts. Keep the real guard
  (`scroll_to_top == "g"`, so the old `cycle_grouping_mode: g` binding never returns)
  and update the grouping assertions to `"o"` / `"O"`, rewriting the docstring
  accordingly.
- `tests/ace/tui/widgets/test_changespec_info_panel_grouping.py:77` —
  `"[group: by project (B)]"` → `"[group: by project (o)]"`.
- `tests/ace/tui/widgets/test_agent_info_panel.py` — no change needed; it derives the
  key through `key_display_name(...)` into `_DEFAULT_GROUPING_KEY`.
- `tests/test_keymaps_validation.py` — add a case modeled on
  `test_contextual_query_actions_may_share_a_custom_key` proving that overriding
  `beads_open_bug` and `files_open_external` onto the same custom key keeps both
  overrides (covers step 3).
- `tests/ace/tui/artifacts_contract/` — no source edit expected. The reachability probe
  and `check_declared_keys_resolve_to_named_actions` should pass unchanged; if either
  fails, that is a real signal, not an expectation to relax.

### 6. PNG visual snapshots

The changed keys are rendered into several goldens, so `just test-visual` will fail
until they are refreshed. Expected to move:

- `changespec_initial`, `changespec_selected_row`, `patch_filter_bar_*` — the Patches
  info-panel `[group: by project (B)]` badge becomes `(o)`.
- `artifacts_files_*` — the Files footer hint `o external` becomes `E external`.
- `artifacts_beads_*` — the Beads footer `open issue` key.
- `help_keymaps_changespecs`, `help_guide_changespecs` — help modal key columns.

Procedure: run `just test-visual`, inspect `.pytest_cache/sase-visual/` diffs, and
confirm every diff is confined to the changed key glyphs before accepting with
`just test-visual -- --sase-update-visual-snapshots` (or the equivalent direct pytest
invocation). Do **not** blanket-accept: if a golden differs anywhere other than a key
glyph, stop and investigate. A prior epic recorded a broad, unrelated visual-drift
failure class in this suite; if failures show up that are unrelated to these keys,
confirm they reproduce on a stashed tree and record them separately rather than
absorbing them into these goldens.

### 7. Verify

1. `just install` first — workspaces are ephemeral and dependencies may have moved.
2. `just check` (whole-repo lint gates + diff-scoped tests). If it escalates or reports
   an unusual selection, note it and rely on step 4.
3. `just test-visual` per step 6.
4. `just check-full` through `/sase_monitor`
   (`sase monitor start --command 'just check-full' …` with a `--next` action) — never
   inline. This change touches `default_config.yml` and the keymap registry, which is
   exactly the broadening set that warrants the full suite.
5. Sanity-check the live TUI if convenient: on Patches/Stitches/Files, `o` and `O` cycle
   the grouping badge; on Beads, `E` opens a linked issue and `o` does nothing; on
   Files, `E` opens the selected artifact in `$EDITOR`; `?` help and the Files/Beads
   footers show the new keys; `!o` still opens the PR-origin modal on Patches.
6. Confirm no **user-level** override pins these actions. The default change is inert if
   `keymaps.app.cycle_grouping_mode` (or either open action) is set in the user's own
   sase config; if one is, say so in the completion report rather than editing the
   user's config.

## Out of scope

- Implementing the unified `artifacts_open_external` action that `sase-m6.9`'s notes
  describe but never landed. Both open actions stay distinct; only their key changes.
- Giving Beads or the Plan pane a real `GROUPING` capability. They have none today, so
  `o` / `O` are a silent no-op there, exactly as `B` / `I` are now.
- Tightening `edit_panel`'s `check_app_action` availability to agents/axe so `E` is
  unambiguous on the Patches pane too. It is already a runtime no-op there and is
  already scoped correctly in the palette and help, and touching availability would
  churn footer/help/palette output and more goldens for no user-visible gain.
- Adding a new one-shot post-update toast. The existing `_keymap_unification_notice.py`
  marker is specific to the sase-m6.9 `y`/`R` flip and should not be reused or reset. If
  the reviewer wants a heads-up toast for this swap, say so and it can be added as its
  own marker.

## Follow-up to record (do not fix here)

`commands/_app_metadata.py` scopes `cycle_grouping_mode` and
`cycle_grouping_mode_reverse` to `AGENTS_ONLY`, so the command palette does not offer
grouping-cycle on the Patches, Stitches, or Files panes even though the keybinding works
there. This predates and is independent of this key swap. Record it as a
`PROPOSED FOLLOW-UP:` note rather than fixing it inside this plan.

## Risks

- **Missed rendered key.** Mitigated by the fact that nearly every surface derives keys
  from the registry; the exceptions are the two doc files and the goldens, all
  enumerated above.
- **Modal-local `o` handlers.** `PreviewPanelModal` (`o` = open in editor) and the
  models-panel override cards (`o` = choose override) bind `o` on their own screens.
  Screen bindings resolve before app bindings, and app-level `o` already exists today,
  so behavior is unchanged — but spot-check the preview reader while verifying.
- **Muscle memory.** `B` and `I` become unbound at the app level. That is intentional;
  nothing should claim them in this change.
