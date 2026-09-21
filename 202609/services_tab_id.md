---
tier: tale
title: Canonicalize the Services tab id
goal:
  The internal ACE tab id for the Services tab is `services`; `axe` survives only as a
  normalized legacy alias for persisted session state and CLI input, and no module,
  widget id, CSS class, keymap config key, query token, state path, or PNG golden
  changes.
size: medium
proposed_by: bbugyi200.athena.sase-11y.10.1.4
bead: sase-11y.10.1.4
create_time: 2026-09-20 21:08:44
status: wip
---

- **PARENT:**
  [202609/service_host_sunset.md](https://github.com/sase-org/sase--plans/blob/main/202609/service_host_sunset.md)
- **BEAD:**
  [sase-11y.10.1.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-11y/sase-11y.10.1.4.md)

# Plan: Canonicalize the Services tab id

Phase `tab-id` of epic `sase-11y.10.1` (Sunset legacy supervision paths, docs, and
glossary). Bead: `sase-11y.10.1.4`.

## Goal

The internal ACE tab id for the Services tab is `services`. `axe` survives only as a
normalized legacy alias for persisted session state and CLI input, exactly the way
`changespecs`/`patches` already survive for the Artifacts tab. No module, widget id, CSS
class, keymap config key, query token, state path, or display label is renamed.

Nothing here is a behavior change. A user who types `sase ace --tab axe`, or whose
session hands the old id forward, must land on the Services tab — never on the `agents`
fallback.

## Context an implementer needs

`src/sase/ace/tui/tab_order.py` is the single source of truth. Today it reads
`SERVICES_TAB: TabName = "axe"` with `SERVICE_TAB_ALIASES = frozenset({"services"})`
pointing the wrong way. `SERVICES_TAB` currently has no consumer outside its own module;
after this plan it is imported widely.

`src/sase/main/parser_ace.py` already accepts both `services` and `axe` for `-t/--tab`
and already lists `axe` in `set_completion_compat_choices`, so the parser needs only a
help-text touch-up.

Read these before starting:
`sase memory read lint_and_test.md symvision.md tui.md cli_rules.md -r "<why>"`.

## Step 1 — Invert `tab_order.py`

In `src/sase/ace/tui/tab_order.py`:

- `TabName` becomes `Literal["artifacts", "agents", "services"]`.
- `LegacyTabName` gains `"axe"` (with a comment saying it is the legacy Services tab id
  persisted by older sessions) and loses `"services"`.
- `SERVICE_TAB_ALIASES` is replaced by
  `LEGACY_SERVICES_TABS: frozenset[str] = frozenset({"axe"})`, mirroring the existing
  `LEGACY_ARTIFACTS_TABS` shape. Delete `SERVICE_TAB_ALIASES`; it has no other consumer.
- `SERVICES_TAB: TabName = "services"`.
- `TAB_ORDER` becomes `("agents", "artifacts", "services")`.
- `normalize_tab_name` maps `LEGACY_SERVICES_TABS` forward to `SERVICES_TAB`.

## Step 2 — Rename the tab id everywhere it is a tab id

This is the bulk of the diff. Two kinds of site:

**(a) Local `TabName` / `CommandTab` protocol aliases.** 33 modules redeclare
`TabName = Literal["artifacts", "agents", "axe"]` as a structural stub for the mixin
protocols. Every one becomes `"services"`. The two wider variants
(`actions/marking.py:18`, `actions/clipboard/_base.py:12`, which also carry
`"patches"`/`"changespecs"`), `commands/types.py:36` (`CommandTab`),
`modals/help_modal/binding_common.py:13`, and `modals/jump_all_modal.py:39` change the
same way. Find them with:

```bash
grep -rn 'Literal\[.*"axe"' --include=*.py src/
```

Do **not** touch the `AxeViewType = Literal["axe"] | int` aliases in the same grep
output — that is the AXE pane's view selector, not a tab id.

**(b) Tab-valued `"axe"` comparisons and dict keys.** Sweep with:

```bash
grep -rn '"axe"' --include=*.py src/
```

and classify each hit against the leave-alone list in Step 4. In the files the epic plan
names — `_app_layout.py`, `_app_watchers.py`, `_app_action_availability.py`,
`actions/axe.py`, `testing/ace_page.py`, `testing/ace_page_group.py` — import
`SERVICES_TAB` from `tab_order` rather than reintroducing a literal;
`_app_action_availability.py` already imports `ARTIFACTS_TAB` for exactly this reason.
Elsewhere a plain `"services"` literal is fine, matching how `"agents"` literals are
already written throughout `actions/`.

Dict keys typed by the tab move; their **values do not**:

| Site                                                                       | Key becomes | Value stays      |
| -------------------------------------------------------------------------- | ----------- | ---------------- |
| `widgets/tab_bar.py` `_TAB_COLORS`, `_TAB_DISPLAY_NAMES`                   | `services`  | `"Services"`     |
| `widgets/agent_onboarding.py` `_TAB_ROWS`                                  | `services`  | `"AXE"` row copy |
| `modals/help_modal/modal.py` `_TAB_ACCENTS`, `_TAB_CLASSES`                | `services`  | `-tab-axe`       |
| `modals/help_modal/binding_common.py` `TAB_DISPLAY_NAMES`, `COLUMN_SPLITS` | `services`  | `"Axe"`          |
| `commands/_tabs.py` `ALL_TABS`, `AXE_ONLY`, `AGENTS_AXE`, `CL_AXE`         | `services`  | names unchanged  |

`styles.tcss` is not edited: `#axe-dashboard` and `.-tab-axe` keep their names.

The two display labels the epic plan explicitly authorizes changing:

- `modals/command_palette_modal.py:34` `_TAB_BADGE` — this dict is keyed by
  `CommandTab`, so the **key must** become `"services"` or the context badge silently
  falls back; set the label to `"Services"` at the same time.
- `modals/jump_all_modal.py:58` `_TAB_STYLES["axe"]` — keep the `"axe"` key (its entries
  are section keys fed by `_Entry("axe", ...)`, and `normalize_tab_name` already carries
  them home through `current_tab`); change only the label to `"Services"`.

Neither label is captured by a PNG golden —
`tests/ace/tui/visual/test_ace_png_snapshots_command_palette.py` and
`..._jump_action.py` contain no `axe` reference — so both stay pixel-neutral. Leave
every other `"AXE"` / `"Axe"` display string alone; changing them would move goldens and
belongs with the `docs` phase.

## Step 3 — Compat funnels, each of which needs a test

Every entry point that can receive the old id must normalize it. Five of these are
load-bearing:

1. `normalize_tab_name` (Step 1) — already the funnel for `AceApp.__init__`
   (`src/sase/ace/tui/app.py:329`) and `validate_current_tab` (`:285`).
2. `src/sase/ace/tui/commands/types.py` — add `"axe"` to `LegacyCommandTab` and give
   `CommandContext.__post_init__` an `axe` → `services` arm next to the existing
   `patches`/`changespecs` arm.
3. `src/sase/ace/tui/commands/context.py:40` `_normalize_tab` — map `"axe"` to
   `"services"` before the membership check.
4. `src/sase/ace/tui/widgets/_keybinding_modes.py:556` — `key_group = tab` feeds
   `copy_mode.keys.get(key_group, {})`, and the copy-mode keymap group is still named
   `axe` in `src/sase/default_config.yml:901`. Without a `services` → `axe` mapping here
   the Services-tab copy footer renders empty. **This is the one silent breakage in the
   sweep; write the regression test first.**
5. `src/sase/ace/testing/ace_page.py:34` `_LEGACY_STATE_VALUE_ALIASES` gains
   `("tab", "axe"): "services"`, so the ~40 existing `expect_state("tab", "axe")` calls
   keep passing and keep proving the alias works. The `initial_tab` Literals in
   `ace_page.py:190` and `ace_page_group.py:83` gain `"services"` and keep `"axe"`.

Also:

- `src/sase/ace/tui/commands/_mode_commands.py:183` `tab_to_command_tab` — the key stays
  `"axe"` (it is a keymap group name) and the value becomes `"services"`. Command ids
  stay `copy.axe.*`, which `commands/_availability_axe.py:83` matches on.
- `src/sase/ace/tui/actions/refresh_panel.py:246` `_this_tab_surface` branches on the
  tab but returns the freshness **surface** name, which stays `"axe"`. Change the branch
  condition only.
- `src/sase/main/parser_ace.py:113` — extend the `-t/--tab` help so it names `axe` as
  the accepted legacy alias for `services`, alongside the existing
  `changespecs`/`patches` sentence. Keep every current choice. Per `cli_rules`, the help
  must stay scannable; no other parser change is needed.

## Step 4 — Leave these alone (they are not tab ids)

Changing any of these is a behavior change, not a rename:

- Query token `source:axe` — `ace/agent_query/tokenizer.py:49`,
  `ace/query_profile/profiles/_agents_live.py:39`.
- Agent-launch source values — `ace/scheduler/mentor_runner.py:104`,
  `ace/scheduler/workflows_runner/starter.py:254,382,487`,
  `ace/tui/models/agent_source.py`.
- State paths — `ace/tui/bgcmd.py:37` `BGCMD_STATE_DIR = sase_subdir("axe") / "bgcmd"`.
- Config segment names — `ace/tui/modals/axe_entry_editor_types.py:179`,
  `config/core.py:537`, `config/loading.py:73`.
- The `axe` **keymap config group** in `src/sase/default_config.yml:901` and every
  lookup of it: `actions/clipboard/_core.py:201`,
  `modals/help_modal/axe_bindings.py:29`, `actions/clipboard/_palette_registry.py:100`,
  `actions/clipboard/_palette.py:181,184`, `_copy_target_standard.py:132,140,148`. This
  is a user-editable config key; renaming it would break existing keymap overrides. No
  `default_config.yml` change is needed by this plan.
- `runners_modal.py:71` `section` literal and its `:205` `jumpable.section == "axe"`
  branch; `jump_all_modal.py` `_Entry` tab values.
- `AxeViewType`, `widgets/bgcmd_list.py:27` `ItemType`,
  `modals/process_select_modal.py:25` `process_type`.
- Freshness surface names — `actions/event_refresh/_freshness.py:13`,
  `actions/refresh_panel.py:90`.
- `widgets/_keybinding_status.py:160` signature tag; `stats/_perf_view_latency.py:56`
  metric group.
- `doctor/checks_*.py` `group="axe"`, `notifications/senders.py` `sender="axe"`,
  `bead/epic_launch.py` `EpicLaunchOrigin`, `llm_provider/usage/refresh.py` origin.
- Module and file names under `ace/tui/actions/axe*`, the `#axe-dashboard` widget id,
  and the `-tab-axe` CSS class. A file rename would collide with the `docs` phase and
  explode the diff.

## Step 5 — Tests

Update existing tests that assert the id directly:

- `tests/test_command_catalog_build.py` — eleven
  `spec.tabs == ("artifacts", "agents", "axe")` asserts become `"services"`.
- `tests/test_command_availability_scope.py:74` — the `("changespecs", "agents", "axe")`
  tuple.
- Roughly 64 lines across ~30 test modules set `current_tab` on a hand-rolled stub app
  (`self.current_tab: Any = "axe"`, `app.current_tab = "axe"`,
  `_TabNavigationApp(current_tab="axe")`) or assert `app.current_tab == "axe"`. Those
  stubs bypass the reactive validator, so each must become `"services"`. Find them with
  `grep -rn 'current_tab' --include=*.py tests/ | grep axe`.
- `tests/ace/tui/test_agent_fold_transitions_llm_calls.py:35`
  `Literal["axe", "patches"]`.

Deliberately **keep** the existing `page.expect_state("tab", "axe")` calls and
`AcePage(initial_tab="axe")` fixtures as-is: they are now the alias's regression
coverage.

Add new tests:

1. `normalize_tab_name("axe") == "services"`, `"axe" not in TAB_ORDER`,
   `TAB_ORDER == ("agents", "artifacts", "services")`, and `SERVICES_TAB == "services"`.
2. **Restart carries the old id forward.** An `AcePage(initial_tab="axe")` (and an
   `AceApp` constructed the same way) lands with `current_tab == "services"` and the
   Services view visible — explicitly assert it is _not_ the `agents` default fallback.
   This is the case the epic plan calls out by name.
3. `CommandContext(tab="axe").tab == "services"`, and a Services-only spec from
   `AXE_ONLY` is available for that context.
4. The Services-tab copy footer resolves the `axe` keymap group: drive
   `update_copy_bindings(tab="services", ...)` (or the footer entry point the existing
   `tests/test_keybinding_footer_status.py` uses) and assert the visible copy bindings
   are non-empty and match `copy_mode.keys["axe"]`. Guards Step 3 item 4.
5. `create_parser().parse_args(["ace", "--tab", "axe"])` still parses, and the resulting
   `initial_tab` normalizes to `"services"`.

## Step 6 — Verify

```bash
just install          # ephemeral workspaces can have drifted deps
just fix              # inline, before anything long
sase tool run check
```

`just check` must pass. Do **not** run `just check-full`.

Then, because this touches rendered TUI code paths:

```bash
just fix-tui-screenshots
```

The tab label already renders as "Services", so this should report `updated=0`. Inspect
the report at `.pytest_cache/sase-visual/latest-report.json` before applying anything.
If a group did move, expand it and confirm the diff is explainable before accepting;
generation is not approval. The 18 goldens that went stale earlier in this epic (epic
bead `sase-11y` notes #3 and #4) were refreshed during the `sase-12z.5` landing, so a
dirty golden here is new, not inherited.

Symvision: follow the `symvision` memory's hierarchy if the sweep orphans a symbol —
delete dead symbols and their tests, do not whitelist. This plan is not expected to
create or remove any `--epic-symbol` entry; `sase bead epic-symbols sase-11y.10.1.4`
currently reports none, and the `sase-11y` entries in the `Justfile` are keyed to the
top-level epic, not to this phase.

## Step 7 — Close

```bash
sase bead epic-symbols sase-11y.10.1.4    # must report no entries for this phase
sase bead close sase-11y.10.1.4 --note "<what was verified>"
```

Close **only** `sase-11y.10.1.4`. Do not close `sase-11y.10.1`, `sase-11y.10`, or
`sase-11y`. `sase-11y.10.1.5` (docs) and `sase-11y.10.1.6` (glossary) are in progress
and blocked on this phase; they are other agents' work.

Do not create beads. Record any discovered follow-up with
`sase bead note sase-11y.10.1.4 'PROPOSED FOLLOW-UP: <one-line summary — detail>'`.

## Out of scope

- Any file under `sase/repos/` or any linked/sidecar repo.
- `docs/` and `mkdocs.yml` — the `docs` phase owns them.
- `sase/memory/` — the `glossary` phase owns it.
- Renaming modules, widget ids, CSS classes, keymap config groups, state paths, or query
  tokens.
- The remaining `AXE` / `Axe` **display labels** in `widgets/agent_onboarding.py:51`,
  `modals/help_modal/binding_common.py:292`, and `modals/help_modal/modal.py` accents.
  They render into PNG goldens; record them as a `PROPOSED FOLLOW-UP:` note so the
  epic's land agent can triage them alongside the `docs` phase.
