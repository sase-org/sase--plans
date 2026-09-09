---
tier: epic
title: Every notification-panel tab wears an icon
goal: "Every notification-panel tab renders a meaningful icon in the panel's tab strip
  and in the top-bar indicator's per-tab chips, resolved through a chain that can never
  come up empty; the Snoozed count sheds its `z` suffix for a moon glyph; and any gate
  that declares a new panel tab must declare that tab's icon.

  "
phases:
  - id: core-icon
    title: Rust core carries a per-tab icon
    depends_on: []
    size: medium
    description: "core-icon: add `icon` to the core's notification tab record, donated
      by the newest member row that declares `action_data.panel_icon`, mirroring the
      existing color-donation rule, with defensive validation and tests; land and
      release it in sase-core.

      "
  - id: icon-chain
    title: Icon resolution chain and configuration
    depends_on: []
    size: medium
    description: "icon-chain: add `resolve_notification_tab_icon` to the tab-style
      module with its four-rung resolution chain, the built-in key and kind glyph
      tables, the `ace.notification_tabs.*.icon` setting, its JSON-schema entry, and the
      bundled defaults.

      "
  - id: gate-contract
    title: Gates must declare their panel's icon
    depends_on: []
    size: medium
    description: "gate-contract: add the required `presentation.panel_icon` gate field,
      project it into notification `action_data` as a protected key, and update both
      bead gate producers that declare `panel: beads`.

      "
  - id: render
    title: Render icons in the tab strip and indicator
    depends_on:
      - icon-chain
    size: medium
    description: "render: draw the icon in the modal tab strip and in every indicator
      chip, replace the snoozed `z` suffix with its glyph, and make every width
      measurement cell-aware so a two-cell icon cannot desync tab click ranges or
      tooltip alignment.

      "
  - id: core-floor
    title: Adopt the released core and verify end to end
    depends_on:
      - core-icon
      - icon-chain
      - gate-contract
    size: small
    description: "core-floor: raise the `sase-core-rs` floor to the release carrying the
      tab icon and verify a gate-declared panel icon reaches the tab strip and the
      indicator through the real core.

      "
  - id: docs-skill
    title: Documentation and the sase_gate skill contract
    depends_on:
      - render
      - gate-contract
    size: small
    description:
      "docs-skill: document tab icons, the reshaped indicator badge, and the new
      configuration field, and teach the bundled sase_gate skill source that a declared
      panel requires a panel icon."
proposed_by: bbugyi200.athena.ui.w1
status: done
bead_id: sase-gz
create_time: 2026-09-09 19:50:58
---

- **PROMPT:**
  [prompts/202608/notification_tab_icons.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/notification_tab_icons.md)
- **BEAD:**
  [sase-gz](https://github.com/sase-org/sase--beads/blob/main/pages/sase-gz/README.md)

# Plan: Every notification-panel tab wears an icon

## Why

Today a notification-panel tab is identified by its name in the modal's tab strip and by
nothing at all in the top-bar indicator, where each tab contributes a bare count
distinguished only by color (`✉ 2·3·1`). Color alone is a weak identifier: it is
invisible to anyone reading quickly, it collides once more than a handful of tabs exist,
and it degrades to a hashed auto-palette entry for tabs nobody named. The one place a
glyph does appear — the snoozed-only badge — spells it as an ASCII `z` suffix (`✉ 4z`),
which reads like a typo rather than a symbol.

Giving every tab an icon makes the badge self-describing at a glance, ties the indicator
and the panel together into one visual language, and gives brand-new gate-declared
panels a real identity instead of a random color.

## Design

### The resolution chain

Every tab resolves exactly one icon, by this precedence, highest first. It deliberately
mirrors the existing color chain in
`src/sase/ace/tui/widgets/notification_tab_style.py`, so a reader who knows one knows
the other:

1. **User configuration** — `ace.notification_tabs.<key>.icon`. An empty string restores
   the built-in default, exactly as `color` already behaves.
2. **Sender-declared** — the icon carried on the snapshot tab record (`tab.icon`), which
   the Rust core donates from the newest member row declaring `action_data.panel_icon`.
3. **Built-in key default** — for the tabs ACE ships knowing about (`hitl`, `errors`,
   `beads`, `general`, `snoozed`, `muted`).
4. **Kind default** — keyed by the core's own `tab.kind` (`hitl`, `panel`, `errors`,
   `general`, `tag`, `snoozed`, `muted`), so a tab ACE has never heard of still gets a
   glyph that means something about what it is.
5. **Last resort** — `•`, only reachable when a tab arrives with no kind at all.

Rung 4 is the important design difference from color. Color's last rung is a _hashed_
palette entry, which is fine because an arbitrary color is still a usable identifier. An
arbitrary _glyph_ would be worse than none — an icon that means nothing teaches the
reader nothing. So icons never hash: they are either meaningful or an honest generic
mark for that kind of tab.

### The glyph set

Chosen for meaning first, then for being single-cell and broadly available so the top
bar stays dense and nothing renders as tofu:

| Tab / kind     | Glyph | Reading                            | Alternates if it fails the audit |
| -------------- | ----- | ---------------------------------- | -------------------------------- |
| `hitl` (Gates) | `⚑`   | a flag raised for you to answer    | `⌸`, `!`                         |
| `errors`       | `✖`   | something failed                   | `✗`, `x`                         |
| `beads`        | `◈`   | a bead                             | `◆`                              |
| `general`      | `✉`   | the plain inbox                    | `✻`                              |
| `snoozed`      | `☾`   | asleep until later — replaces `z`  | `⏾`, `z`                         |
| `muted`        | `⊘`   | silenced                           | `⊗`                              |
| kind `panel`   | `◆`   | a named place                      | `▪`                              |
| kind `tag`     | `#`   | a tag; ASCII, so it always renders | —                                |
| no kind        | `•`   | last resort                        | —                                |

`✉` is already rendered by today's indicator, so it is proven against the pinned
visual-test font. Every other glyph must pass the audit described under **Verification**
before it is committed; if one fails, take its listed alternate rather than inventing a
new glyph, and record the substitution in the commit message.

### Two decisions worth reviewing

**The indicator drops its `✉` anchor when it has chips to show.** Today every populated
badge is prefixed `✉ `. Once `general` owns `✉` as its own tab glyph, keeping the prefix
would render `✉ ✉2` — the same glyph twice, meaning two different things. Each chip is
now self-identifying, so the anchor has no work left to do. The empty state keeps it:
`✉ 0` still means "your inbox, nothing in it". This also keeps the badge's width roughly
neutral — every chip grows by its glyph, but the badge sheds the two-cell prefix and the
`·` separators (see below).

**The tab strip colors the icon, not the label.** The modal's tab strip styles labels by
active state (teal active, grey inactive) and does not use per-tab colors at all.
Rendering just the _icon_ in the tab's resolved color, with the label and count keeping
today's active/inactive styling, makes the strip a legend for the indicator chips
without turning it into a rainbow.

### Badge and strip shapes

- **Indicator, empty** — `✉ 0`, dim. Unchanged.
- **Indicator, snoozed only** — `☾4`, in the Snoozed tab's resolved color at dim weight.
  The `z` suffix is gone. The rule that a snoozed chip drops out entirely as soon as any
  other tab has a count is unchanged.
- **Indicator, anything else** — one `<icon><count>` chip per visible tab in panel
  order, each in the tab's resolved color at bold weight, joined by a single space. The
  dim `·` separator is removed: a glyph already delimits each chip, and the separator
  now reads as clutter. The overflow `+K` chip and the
  `ace.notification_indicator_max_counts` budget are unchanged; `+K` keeps its dim style
  and is joined by the same single space.
- **Tab strip** — `<icon> <Label> <count>` per tab, `|`-separated as today.
- **Tooltip** — each tab line gains its icon before the label; the header, the per-tab
  counts, and the trailing time phrases are unchanged.

### The gate contract

A gate declares its notification's tab with `presentation.panel`. It will now also have
to declare that tab's icon with `presentation.panel_icon`, which is **required whenever
`panel` is set**, for every gate kind — not only `kind: "custom"`. A gate that names a
tab is the thing introducing that tab to the user, so it is the thing responsible for
saying what the tab looks like.

`panel_icon` is a separate field rather than a reuse of the existing
`presentation.icon`, because `presentation.icon` is the _row's_ icon and rows in one
panel legitimately differ: the two gates in this repo that declare `panel: "beads"`
carry `◈` (snooze) and `✦` (triage). Donating the row icon to the tab would make the
Beads tab's glyph flip depending on which row arrived most recently. `panel_icon` is a
property of the tab, and gates sharing a panel are expected to agree on it.

This is a **breaking change** to the schema-version 3 gate request contract. The blast
radius has been surveyed and is small: the only producers that declare a panel are
`src/sase/bead/snooze_gate.py` and `src/sase/bead/_task_gate_spec.py`, both in this
repo; neither `sase-github` nor `sase-telegram` authors a gate request with a panel.
Stored notifications written before this change keep working — a `panel` with no
`panel_icon` simply falls to rung 4 (`◆`). No migration is needed.

## Phases

### Rust core carries a per-tab icon

**Repository: `sase-core`.** Open it with the `/sase_repo` skill; it is a linked repo,
not part of this workspace checkout. All work here is in
`crates/sase_core/src/notifications/`.

The tab record already carries a sender-declared `color` donated by the newest member
row, in `tabs.rs`. Add `icon` alongside it, following that code exactly:

- `wire.rs`: add `icon: Option<String>` to `NotificationTabWire`, serialized only when
  present (the existing `an_absent_color_stays_absent_on_the_wire` test documents that
  convention for `color`).
- `tabs.rs`: add `icon` and `icon_cursor` to `TabAccumulator`; add a
  `declared_tab_icon(row)` helper that reads `action_data["panel_icon"]`; in
  `accumulate`, apply the same newest-activity-cursor wins rule the color donation uses,
  and copy the value through in `ordered_tabs`.
- `declared_tab_icon` is a _defensive_ reader, not a validator — Python validates at
  gate-write time, so this only has to refuse stored junk: reject empty-after-trim, more
  than 32 codepoints, more than 128 bytes, or any control character. Those bounds match
  `validate_icon` in `src/sase/notification_gates/model_validation.py` on the Python
  side.

Icon donation is deliberately not restricted to panel-kind tabs. Any row may donate,
exactly as any row may donate a color; keeping the two rules identical is worth more
than a restriction nobody needs.

Tests in `tabs.rs`, mirroring the existing color tests one for one: newest declared icon
wins regardless of input order, a resurfaced row outranks a newer sent row, junk is
ignored, and an absent icon stays absent on the wire.

Land the change and get it released (the repo releases through release-please;
`crates/sase_core` is currently at 0.19.1). Record the released version in this phase's
completion notes — the `core-floor` phase needs it.

### Icon resolution chain and configuration

All in `src/sase/ace/tui/widgets/notification_tab_style.py` unless noted.

- Add `_BUILTIN_TAB_ICONS` (the six built-in keys) and `_KIND_TAB_ICONS` (the seven core
  kinds) from the glyph table above, plus the `•` last resort.
- Add `resolve_notification_tab_icon(tab: NotificationTagTab) -> str` implementing the
  four-rung chain, and export it in `__all__`. Reuse `_notification_tab_config_key` so
  config keys stay the user-facing `snoozed` / `muted` spellings rather than the
  internal `__snoozed__` / `__muted__`.
- Replace `_configured_tab_colors_for_token` with a single cached parse that returns
  both the color and the icon per key, so a config read is not paid twice. Keep the
  existing `lru_cache(maxsize=1)` on `current_config_token()` and keep
  `_configured_tab_colors()` working for the color path.
- Sanitize a configured icon by calling `validate_icon` from
  `sase.notification_gates.model_validation` inside a `try/except GateError`, returning
  `""` on failure. That keeps one definition of "a legal icon" in the codebase instead
  of duplicating its grapheme-counting logic. Add one extra guard the gate path does not
  need: reject an icon whose `rich.cells.cell_len` exceeds 2, so no stored value can
  blow out the top bar.
- `src/sase/default_config.yml`: add an `icon` to each of the six
  `ace.notification_tabs` entries, matching `_BUILTIN_TAB_ICONS`.
- `src/sase/config/sase.schema.json`: add `icon` to the `notification_tabs`
  additionalProperties object — `"type": "string"`, `"maxLength": 32`, `"default": ""`,
  with a description that says an empty string restores the built-in default.

Tests: extend `tests/test_notification_tab_style.py` with one case per rung (config wins
over declared, declared wins over built-in, built-in wins over kind, kind wins over the
last resort), plus junk-config fallthrough and the empty-string reset. Add a parity test
asserting `_BUILTIN_TAB_ICONS` matches the bundled `default_config.yml` icons, mirroring
however the color parity is asserted today. Extend
`tests/test_config_schema_notification_tabs.py` for the new field.

### Gates must declare their panel's icon

- `src/sase/notification_gates/presentation.py`: add
  `GATE_PANEL_ICON_ACTION_DATA_KEY = "panel_icon"` and
  `normalize_gate_panel_icon(value)`, which returns `None` for `None` and otherwise
  delegates to the shared `validate_icon` bounds, raising
  `GateError("invalid_presentation", "presentation.panel_icon", ...)` on junk. Export
  both.
- `src/sase/notification_gates/validation.py`: normalize `presentation.panel_icon` next
  to the existing `normalize_gate_panel` call, and raise
  `GateError("missing_presentation", "presentation.panel_icon", ...)` when a panel is
  declared with no panel icon. The message must say what to do, in the voice the
  existing `missing_presentation` errors use: a gate that declares a panel names a
  notification-panel tab, so it must also declare that tab's icon as one emoji or glyph.
  Add `panel_icon` to the protected `action_data` keys that producers may not write
  directly through `presentation.action_data`, alongside `panel`, `origin_agent`, and
  `gate_title`.
- `src/sase/notification_gates/service.py`: project the normalized value into
  `action_data[GATE_PANEL_ICON_ACTION_DATA_KEY]` next to where `panel` is projected
  (around line 347).
- `src/sase/bead/snooze_gate.py` and `src/sase/bead/_task_gate_spec.py`: both declare
  `"panel": "beads"` and must now also declare `"panel_icon": "◈"` — the same glyph in
  both, since they share the tab, and matching the built-in `beads` default so the tab
  looks identical whichever rung supplies it.

Tests: extend `tests/test_notification_gate_presentation.py` and
`tests/test_notification_gates.py` for the required-when-panel-declared rule, the
invalid-icon rejection, the projection into `action_data`, and the protected-key
rejection. Any existing fixture that declares a `panel` without a `panel_icon` now fails
by design — update it rather than relaxing the rule. Grep for `"panel":` under `tests/`
to find them; there are roughly seven files.

### Render icons in the tab strip and indicator

`src/sase/ace/tui/widgets/notification_indicator.py`:

- `_build_content`: emit `<icon><count>` per chip, joined by a single space, dropping
  the `·` separator and `_CHIP_SEPARATOR_STYLE` with it. Drop the leading `✉ ` prefix
  from the populated branches; keep `✉ 0` exactly as it is for the empty state. The
  snoozed-only branch renders `<icon><total>` in `dim <color>`, replacing the
  `f"{total}z"` spelling.
- `_build_tooltip`: prefix each tab line with its icon, and compute the label column
  width with `rich.cells.cell_len` instead of `len`, so a two-cell icon or a non-ASCII
  tag label cannot skew the column.

`src/sase/ace/tui/modals/notification_modal_tags.py`,
`NotificationTagStrip._build_content`:

- Render `<icon> <Label> <count>` per tab, with the icon styled in the tab's resolved
  color (and dimmed when the tab is inactive) while the label and count keep today's
  active/inactive styles. Import the resolvers lazily, the way
  `notification_indicator.py` already does, to avoid the widgets/modals import cycle
  called out in that module's comment.
- **Fix the click-range arithmetic.** `_tab_ranges` currently records offsets from
  `len(text.plain)` — _character_ counts — while `on_click` compares them against
  `event.x`, a _terminal column_. Every tab label is ASCII today so the two happen to
  agree. The moment a two-cell icon appears in the strip (any emoji a gate or a user
  configures), every range to its right shifts and clicking a tab selects the wrong one.
  Accumulate the ranges with `rich.cells.cell_len` over the appended segments instead.

Tests: extend `tests/test_notification_indicator.py` for icons in chips, the
snoozed-only glyph, the absent anchor when chips are present, the retained `✉ 0` empty
state, overflow, and tooltip alignment with a deliberately two-cell icon. Add a
`NotificationTagStrip` test that places a two-cell icon on the first tab and asserts a
click on a later tab still resolves to that tab — a direct regression test for the bug
above.

Visual snapshots: `notification_beads_tab_120x40` covers the tab strip and must be
regenerated; review the PNG, do not blind-accept it. Add one new snapshot covering the
top-bar indicator with several icon chips (there is no dedicated indicator snapshot
today; follow the naming of `updates_indicator_*_120x40`). This is also the **glyph
audit**: every glyph in the table above must render as a real mark, not tofu, in the
pinned Fira Code fixture, and `rich.cells.cell_len` must report 1 for each. Substitute
the listed alternate for any glyph that fails, and update the built-in tables,
`default_config.yml`, and the docs together so all three stay in agreement.

### Adopt the released core and verify end to end

Raise the `sase-core-rs` floor in `pyproject.toml` from `>=0.19.0,<0.20.0` to the
release produced by the `core-icon` phase, following the pattern of the recent
floor-raise commits (for example `5b3f3494b`), and refresh `uv.lock`.

Then verify the one rung that cannot be tested without the real core: create a gate
declaring a panel and a `panel_icon`, and confirm the icon reaches both the modal tab
strip and the indicator chip through `classify_notification_tabs`. Add that as a test
against the real binding if the suite has a place for one; otherwise verify it by hand
in `sase ace` and record what you saw.

If the core release is not published yet, do not fake the bump or pin a prerelease —
leave the floor alone, say so plainly in the phase notes, and stop. Every other rung of
the chain works without it; only gate-declared icons wait.

### Documentation and the sase_gate skill contract

`docs/notifications.md`:

- **Tabs and Ordering**: add an icon column to the tab table.
- **Top-Bar Indicator**: rewrite the three bullets. `✉ 4z` becomes `☾4`; `✉ 2·3·1`
  becomes `⚑2 ✖3 ◈1`; state that the `✉` prefix now appears only in the empty state and
  why; keep the overflow bullet, adjusting the separator description.
- Add a **Tab icons** section beside the existing **Tab colors**, giving the four-rung
  chain and the bundled glyph set.
- In the gate presentation prose (around the `presentation.panel` and
  `presentation.color` paragraphs), document `presentation.panel_icon`: required
  whenever `panel` is declared, one emoji or glyph, projected into `action_data` as a
  protected `panel_icon` key, and why it is separate from the row's `presentation.icon`.

`docs/configuration.md`, under `ace.notification_tabs`: add the `icon` row to the field
table and describe its resolution chain, mirroring how the `color` row and its
precedence paragraph read.

`src/sase/xprompts/skills/sase_gate.md`: extend the `presentation.panel` bullet to say a
declared panel requires `presentation.panel_icon`, add it to the list of fields that are
required, and add `"panel_icon": "🚀"` next to `"panel": "deployments"` in the worked
example. `tests/main/test_init_skills_sources.py` asserts the example's exact
`"panel": "deployments"` string (line 181) — extend that assertion list rather than
leaving it stale.

Do **not** run `sase skill init` in this phase. Per the generated-skills workflow, a
chezmoi deploy happens only from a clean tree whose `HEAD` is already an ancestor of the
canonical branch, which is not true mid-epic. The source template is the deliverable;
deploying it is a separate step after the epic lands.

## Verification

`just check` after any file change, per the repo's two-speed rule, and `just check-full`
before the epic's combined tree lands. `just test-visual` for the snapshot phases,
reviewing the artifacts in `.pytest_cache/sase-visual/` and only then accepting with
`--sase-update-visual-snapshots`. In `sase-core`, run that repo's own gates.

The `gate-contract` phase changes a published contract, so its commit message must be
marked breaking (`feat(gates)!:`), and the `render` phase's badge reshaping is
user-visible enough to deserve the same treatment (`feat(ace)!:`).
