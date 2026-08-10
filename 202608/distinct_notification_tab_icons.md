---
tier: tale
title: Guarantee distinct icons for every notification-panel tab
goal:
  Every tab the ACE notification panel renders carries an icon distinct from every other
  tab in the same render, except where a human or a sending gate explicitly chose the
  same glyph twice — and `sase doctor` reports that case.
size: medium
proposed_by: bbugyi200.athena.x3
create_time: 2026-08-10 10:01:49
status: wip
---

# Plan

## Problem

The notification panel's tab icon is load-bearing, not decorative:

- `NotificationIndicator._build_content`
  (`src/sase/ace/tui/widgets/notification_indicator.py:65-104`) renders label-free
  `<icon><count>` chips. Icon plus color is the _only_ discriminator a reader has.
- `NotificationTagStrip._render_tabs(compact=True)`
  (`src/sase/ace/tui/modals/notification_modal_tags.py:258-302`) sheds the label from
  every inactive tab when the strip overflows. Its own docstring says tabs are then
  "identified by the icon the resolution chain guarantees every tab has".

Color does not rescue an icon collision: `_AUTO_PALETTE`
(`src/sase/ace/tui/widgets/notification_tab_style.py:85-92`) has six entries, so tag-tab
colors themselves collide once seven tag tabs exist.

The six tabs ACE ships knowing about _do_ have pairwise-distinct icons today (`hitl` ⚑,
`errors` ✖, `beads` ◈, `general` ✉, `snoozed` ☾, `muted` ⊘ — in both
`src/sase/default_config.yml:158-176` and `_BUILTIN_TAB_ICONS` in
`notification_tab_style.py:56-63`). Three collision classes are nonetheless real, all
verified in a live workspace:

1. **Every tag tab renders `#`.** `_KIND_TAB_ICONS["tag"]`
   (`notification_tab_style.py:74`) is the terminal rung for all of them. In-repo
   producers that can be live at the same time include `done`
   (`src/sase/axe/run_agent_runner_finalize.py:395`), `axe`
   (`src/sase/axe/orchestrator.py:288`), and `file-hooks`
   (`src/sase/file_hooks/runner.py:149`), plus `plan` / `epic` / `launch` rows that are
   not gate actions. Two live tag tabs are indistinguishable chips.

2. **Every unrecognized panel tab renders `◆`** (`_KIND_TAB_ICONS["panel"]`). Gate
   validation requires `presentation.panel_icon` whenever `presentation.panel` is set
   (`src/sase/notification_gates/validation.py:170-178`), so this mostly bites raw
   `sase notify create` rows and legacy stored rows.

3. **`panel_icon` leaks across tabs.** `accumulate()` in
   `crates/sase_core/src/notifications/tabs.rs:173-178` (repo `sase-core`) donates a
   row's `panel_icon` to whatever tab the row lands in, unconditionally. A BeadSnooze
   gate declares `panel: "beads"` + `panel_icon: "◈"`
   (`src/sase/bead/snooze_gate.py:168-170`, `src/sase/bead/_task_gate_spec.py:105-107`);
   once muted and snoozed it is reclassified into `__snoozed__` and donates `◈` _there_.
   The live snapshot in this workspace shows exactly that:

   ```
   key='beads'       kind='panel'   n=1  icon='◈'
   key='__snoozed__' kind='snoozed' n=2  icon='◈'
   ```

   It is invisible today only because `default_config.yml` ships `snoozed: {icon: "☾"}`
   and the configured rung outranks the declared one. Empty that config block and two
   tabs wear one glyph.

## Design constraint that must be respected

The icon chain deliberately refuses the hashed auto-palette the color chain uses. That
decision is stated in three places — `notification_tab_style.py:9-13`,
`docs/notifications.md` ("an arbitrary glyph would teach the reader something false, so
the chain always bottoms out at a meaningful or honestly generic mark"), and
`docs/configuration.md#acenotification_tabs`. **Do not** solve this by hashing tab keys
into a glyph palette. Any derived glyph must be derived from something true about the
tab.

## Decision

Take the two fixes together:

**A. Scope `panel_icon` donation at its source (Rust core).** `panel_icon` means "the
icon for the panel this gate declared". A row donates it only to the tab its own
declared `panel` names. Kills collision class 3 outright and gives `panel_icon` a single
meaning.

**B. Collision-aware resolution over the whole tab set (Python).** Resolve every visible
tab's icon in one pass, tracking _which rung_ produced each. Then guarantee distinctness
for the icons **SASE chose**, and never touch an icon a human or a sender chose:

| Rung                             | Chosen by | Re-derived on collision? |
| -------------------------------- | --------- | ------------------------ |
| `ace.notification_tabs.<k>.icon` | human     | no                       |
| sender `presentation.panel_icon` | sender    | no                       |
| `_BUILTIN_TAB_ICONS[<k>]`        | SASE      | no (see below)           |
| `_KIND_TAB_ICONS[<kind>]`        | SASE      | **yes**                  |
| `_LAST_RESORT_TAB_ICON`          | SASE      | **yes**                  |

Built-ins are exempt because they are pairwise distinct by construction (a new test
enforces that), so a built-in can only collide with an _explicit_ icon someone chose for
another tab — and moving `General` off ✉ because a user configured ✉ elsewhere would be
the surprising outcome, not the helpful one.

This yields the guarantee in the plan's `goal`: the only way two tabs share a glyph is
that a human or a gate explicitly asked for it, and option **C** below makes that case
visible rather than silent.

**C. Report, don't override, explicit duplicates.** A `sase doctor` config check warns
when two tabs would render the same explicitly chosen glyph.

Alternatives weighed and rejected: a hashed glyph palette (contradicts the documented
decision above); keeping labels in compact mode for ambiguous tabs only (does not help
the indicator, which has no labels at all, so it cannot be the whole fix).

### Derivation rule for a re-derived icon

For a tab at the kind/last-resort rung whose glyph is already taken, walk the tab's own
**key** (`axe`, `file-hooks`, `deployments`) and take the first ASCII alphanumeric
character, lowercased, that is not already claimed in this render:

- `axe` → `a`; if `a` is taken → `x`; then `e`
- `file-hooks` → `f` (hyphens skipped, they are not alphanumeric)
- `123-deploy` → `1`

An ASCII letter or digit is honest — it is derived from the name the tab displays, not
invented — and is guaranteed single-cell in every font including the pinned Fira Code
the visual snapshot suite uses.

If the whole key is exhausted (every alphanumeric in it already claimed), keep the
original generic mark and accept the duplicate. Degrading honestly beats inventing a
glyph that means nothing. Document this residual; it requires many simultaneous tabs
with heavily overlapping names.

### Stability requirement

Disambiguation **must** be computed over the full ordered tab list exactly as the
snapshot carries it, before any `count > 0` filtering, and both call sites must feed it
the same list. Otherwise the indicator (which filters in `set_tabs`) and the panel strip
(which does not) could assign different glyphs to the same tab, breaking
`tests/test_notification_indicator.py::test_indicator_chips_correspond_one_to_one_with_the_panel_tabs`
and, worse, teaching the reader two different glyphs for one tab.

## Work

### 1. Scope `panel_icon` donation (repo `sase-core`)

Open the repo with `/sase_repo` (`sase repo open sase-core -r "<reason>"`) and use only
the path it prints. This is core backend logic per this repo's
`rust_core_backend_boundary` memory, so the behavior change belongs there, not in a
Python workaround.

In `crates/sase_core/src/notifications/tabs.rs`:

- Make `declared_tab_icon` (or `accumulate`'s call to it) donate only when the row's own
  normalized `panel` action-data value equals the tab key the row was classified into by
  `tab_key_for`. A row reclassified into `__snoozed__`, `__muted__`, `errors`, or a tag
  tab therefore donates nothing.
- No wire/schema change is needed — `NotificationTabWire.icon` keeps its shape, only the
  population rule narrows. Confirm this before touching `notifications/wire.rs`.
- Keep the existing newest-row-wins tiebreak (`donation_wins`) for rows that _do_
  qualify.
- Rust tests: extend the neighbours of
  `a_tab_wears_the_newest_declared_icon_and_ignores_junk` (tabs.rs:683) and
  `a_resurfaced_row_donates_its_icon_over_a_newer_sent_row` (tabs.rs:723) with a case
  proving a muted+snoozed `panel: "beads"` row donates `◈` to neither `__snoozed__` nor
  any tab but `beads`.
- Rebuild the Python binding after the change. **Note:** this workspace's `sase_core_rs`
  is currently stale —
  `RuntimeError: sase_core_rs content-layout wire is stale: expected schema >= 5 for ref sources, got 1`
  — so a rebuild is required before any runtime check that touches merged config.

The Python parity mirror `_notification_modal_tab_key`
(`src/sase/ace/tui/modals/notification_modal_tags.py:106-128`) classifies rows into tabs
but computes no icons, so it needs **no** matching change. State that explicitly in the
commit message so the next reader does not go looking.

### 2. Collision-aware icon resolution (this repo)

In `src/sase/ace/tui/widgets/notification_tab_style.py`:

- Split the existing chain in `resolve_notification_tab_icon` (lines 141-158) so it
  returns both the glyph and the rung that produced it — e.g. a private
  `_resolve_tab_icon(tab) -> tuple[str, _IconRung]` with a small enum or string literal
  for the rung. Keep `resolve_notification_tab_icon` as the public single-tab call with
  its current signature and behavior; other callers and tests depend on it.
- Add
  `resolve_notification_tab_icons(tabs: Sequence[NotificationTagTab]) -> dict[str | None, str]`,
  keyed by `tab.tag` exactly as the call sites index today. Two passes:
  1. Resolve every tab, collecting the glyphs claimed at a non-re-derivable rung
     (configured, declared, built-in) into a claimed set.
  2. Walk the tabs in list order; for each tab at the kind/last-resort rung whose glyph
     is already claimed, re-derive per the rule above and add the result to the claimed
     set.
- Export the new function in `__all__`.
- Any derived glyph still passes through `_sanitize_icon`'s width bound conceptually —
  ASCII alphanumerics are single-cell by construction, so assert it in a test rather
  than adding a runtime branch.

### 3. Thread the whole tab set through both call sites

- `src/sase/ace/tui/widgets/notification_indicator.py`: `_build_content` and
  `_build_tooltip` currently call `resolve_notification_tab_icon(tab)` per tab (lines
  86, 95, 132). Resolve once over the full list and index the result. Note `set_tabs`
  (line 52) filters `count > 0` before storing; keep the resolution input consistent
  with the strip's — in practice the core never emits a zero-count tab, since
  `accumulate` creates a tab only when a row lands in it, so filtering is a no-op, but
  the implementation must not _depend_ on that.
- `src/sase/ace/tui/modals/notification_modal_tags.py`: `_render_tabs` (lines 258-302)
  resolves per tab inside the loop. Resolve once before the loop. Preserve the lazy
  import (the module-scope import would cycle) and preserve the cell-accurate `column`
  accounting that backs `_tab_ranges`, since `on_click` compares those ranges against
  `event.x`.

### 4. `sase doctor` check for explicit duplicates

Add a check in `src/sase/doctor/checks_config.py` (or a small sibling module following
the existing `checks_config_*.py` split) that reads merged `ace.notification_tabs` and
warns when two tab keys configure the same icon, naming both keys. Keep it read-only and
fast — it is a plain config read, so it belongs in the default (non-`--deep`) set.
Follow whatever registration pattern the neighbouring config checks already use.

Sender-declared duplicates are out of scope for this check: they are a property of live
store contents, not configuration, and a doctor check should not depend on the
notification store's current rows.

### 5. Tests

- `tests/test_notification_tab_style.py`
  - **New guard that does not exist today:** `_BUILTIN_TAB_ICONS.values()`,
    `_KIND_TAB_ICONS.values()`, and `_LAST_RESORT_TAB_ICON` are **pairwise distinct**.
    Lines 277-284 only assert they are single-cell and line 213 asserts one single pair
    differs, so a future edit could silently introduce a duplicate. This test is what
    makes the built-in exemption in the Decision table sound.
  - Two tag tabs (`axe`, `done`) resolve to different glyphs; the first keeps `#` and
    the second derives from its key, or both derive — assert whichever the implemented
    rule produces, and assert only that they differ plus that each is single-cell.
  - Shared-initial tags (`axe`, `agents`) still resolve distinctly.
  - A configured icon and a gate-declared icon are never re-derived, even when they
    collide with each other.
  - A built-in is never re-derived when an explicit icon on another tab matches it.
  - Key exhaustion falls back to the generic mark rather than inventing a glyph.
  - Resolution is order-stable: the same tab list yields the same mapping across calls,
    and a tab's glyph does not change when an unrelated later tab is appended at a lower
    rung.
- `tests/test_notification_indicator.py`: keep
  `test_indicator_chips_correspond_one_to_one_with_the_panel_tabs` (line 259) green, and
  add a case with two tag tabs asserting the two chips differ.
- `tests/ace/tui/visual/test_tab_icon_glyphs.py`: `_AUDITED_ICONS` (lines 46-58)
  enumerates every glyph ACE can pick with nothing configured. Derived alphanumerics are
  now in that set, so extend `_AUDITED_ICONS` to include the ASCII alphanumeric alphabet
  the derivation rule can emit. This is the `visual` mark, so run `just test-visual`;
  coverage should pass trivially for ASCII, and a failure there means the pinned font
  stack is not what the suite assumes.
- `sase-core`: the Rust cases in step 1.

### 6. Documentation

Both docs currently describe the colliding fallbacks as intended behavior, so both must
change.

- `docs/notifications.md`, "Tab icons" (~lines 613-643): keep the five-rung precedence
  list, then add the distinctness guarantee, the derivation rule, the built-in
  exemption, and the honest residual (explicit duplicates, and key exhaustion). Correct
  the closing sentence about `panel_icon` donation to state the new scoping: a
  `panel_icon` is donated only to the tab the gate's own `panel` names, so a muted or
  snoozed row no longer carries its panel's glyph into `Snoozed` or `Muted`.
- `docs/configuration.md#acenotification_tabs` (~lines 841-870): same precedence
  correction, plus a note that a configured icon is never overridden and that
  `sase doctor` reports two tabs configured with the same glyph.

## Out of scope

- Changing the color chain, `_AUTO_PALETTE`, or its six-entry size. The color collision
  noted under "Problem" is real but is a separate concern; file a task bead with
  `/sase_new_task` rather than widening this plan.
- Overriding or rejecting a duplicate icon that a human configured or a gate declared.
  Step 4 reports it; it does not change it.
- Any change to tab _classification_ (which rows land in which tab). Only icon
  resolution and icon donation move.

## Verification

1. `just install` first — this workspace is ephemeral and may be stale.
2. Rebuild `sase_core_rs` after the step 1 Rust change; confirm the stale-wire
   `RuntimeError` quoted above is gone before running config-touching checks.
3. `cargo test` in the `sase-core` checkout for the notification tab tests.
4. `just test-visual` for the PNG snapshot suite (step 5 touches it).
5. `just check-full` here — not `just check`. This change touches ACE widgets, the
   doctor check registry, `default_config.yml`-adjacent resolution, docs, and the visual
   suite, which is exactly the broadening set `just check-full` exists for.
6. Manual confirmation, since the payoff is visual: with at least two tag tabs live,
   open the notification panel, narrow the terminal until the tab strip enters compact
   mode, and confirm the inactive tabs are still tellable apart; then confirm the
   top-bar indicator chips differ too.
