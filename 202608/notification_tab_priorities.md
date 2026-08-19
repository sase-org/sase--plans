---
tier: tale
title: Notification tab priorities with a one-glyph deviation mark
goal:
  Every notification-panel tab has a default priority that reproduces today's tab order
  exactly, `ace.notification_tabs.<key>.priority` reorders any tab against that ladder,
  the shipped `beads` default drops the Beads tab below every ordinary tab, and any tab
  whose priority differs from its default says so with a single-cell `▴`/`▾` mark.
size: medium
proposed_by: bbugyi200.athena.07v
create_time: 2026-08-19 11:05:15
status: wip
---

# Notification Tab Priorities

## Goal

Give every notification-panel tab a **priority**: an integer that decides where the tab
sits in the tab strip, in the top-bar indicator badge, and in the indicator tooltip.
Higher sorts earlier. Every tab gets a sensible default, so nothing has to be
configured; a tab whose effective priority differs from its default renders one extra
cell — `▴` or `▾` — so the deviation is never invisible knowledge.

First use case: the `Beads` tab drops below every ordinary tab.

## Current Shape

Tab order is decided once, in Rust, and rendered in three places from one Python list.

**Rust** (`sase-core`, `crates/sase_core/src/notifications/tabs.rs`): `ordered_tab_keys`
hand-rolls the order — `hitl`, then every `panel`-kind key sorted by lowercased label,
then `errors` / `general` / `done` if present, then every remaining key sorted by
lowercased label, then `__snoozed__`, then `__muted__`. `classify_notification_tabs` and
the store snapshot both return tabs in exactly that order.

**Python**: `notification_tabs_from_core`
(`src/sase/ace/tui/modals/notification_modal_tags.py:161`) is the **only** place a
`NotificationTagTab` is ever constructed — verified by grep: `NotificationTagTab(`
appears at exactly one source line. Every consumer flows through it:

- `classify_notification_modal_tabs` → `NotificationModal._tag_tabs` →
  `NotificationTagStrip`
- `notification_snapshot_from_direct`
  (`actions/agents/_notification_provider_direct.py:41`) →
  `AceNotificationSnapshot.tabs` → `_notification_polling.py:148` →
  `NotificationIndicator.set_tabs`
- `actions/lifecycle.py:129`

That single construction site is the whole reason this feature is small and reliable:
one sort there reorders every surface at once, and no surface can disagree with another.

**Styling precedence** already lives in
`src/sase/ace/tui/widgets/notification_tab_style.py`: color and icon each resolve
configured → sender-declared → built-in → fallback, with the config read cached per
`current_config_token()`. `ace.notification_tabs.<key>` today accepts `color` and `icon`
only.

## Design

### 1. Priority is one integer; higher sorts earlier

Every tab has a **default priority** from a ladder, and an **effective priority** that
is the configured override when one is set and the default otherwise. Tabs render sorted
by effective priority descending; ties keep the order the core returned.

The ladder is the core's existing `ordered_tab_keys` restated as numbers:

| Tab                          | Resolved from          | Default priority |
| ---------------------------- | ---------------------- | ---------------- |
| `Gates` (`hitl`)             | kind `hitl`            | `60`             |
| any declared panel           | kind `panel`           | `50`             |
| `Errors`                     | kind `errors`          | `40`             |
| `General`                    | kind `general`         | `30`             |
| `Done`                       | key `done`, kind `tag` | `20`             |
| any other tag / unknown kind | kind `tag`, fallback   | `10`             |
| `Snoozed` (`__snoozed__`)    | key                    | `-10`            |
| `Muted` (`__muted__`)        | key                    | `-20`            |

Steps of 10 are deliberate: they leave nine slots between any two landmarks, so "put my
`review` tag just above `Errors`" is `priority: 45` and nothing else has to move.

Resolution order inside `default_notification_tab_priority` matters, and each rung
exists because the core has a matching rung:

1. **Key first** for `__snoozed__` / `__muted__`. A notification tagged literally
   `__muted__` but _not_ muted arrives with kind `tag`, and the core still pins it to
   the muted slot (`ordered_tab_keys` filters those keys out of the "remaining" bucket
   and appends them at the end by key). Checking the key first keeps Python agreeing
   with the core on that adversarial input.
2. **`done` only when the kind is `tag`.** `done` is not in the core's
   `RESERVED_PANELS`, so a gate may legally declare `panel: "done"`; the core then ranks
   it kind `panel` and sorts it in the panel bucket, _not_ in the `done` slot. Gating
   the `20` on kind `tag` reproduces that exactly.
3. **Kind**, then `10` as the fallback for a tab arriving with no kind at all — which is
   where the core's own "remaining, sorted by label" bucket puts it.

### 2. Where the ordering lives, and why not in Rust

The ordering _policy_ stays in the core: `notification_tabs_from_core` receives the
core's list and applies a **stable** sort by effective priority. With no overrides
configured, every tab keeps its default, all comparisons tie, and a stable sort of an
already-correct list is the identity — today's order is preserved by construction, not
by a table that happens to agree.

The default ladder is nonetheless mirrored in Python, and that mirroring is a real cost
worth stating plainly. Two facts decide it:

- **The core cannot produce the final order.** Priorities come from
  `ace.notification_tabs`, and `sase_core` never loads merged sase config — it only
  _validates_ config Python hands it. Teaching `classify_notification_tabs` about
  priorities means a new binding parameter, which breaks Python against the published
  minimum core (`sase-core-rs>=0.29.0,<0.30.0`) and forces a core release plus a
  version-window bump. The final sort would still end up in Python either way.
- **This codebase already mirrors core tab vocabulary, under test.**
  `notification_modal_tags.py` mirrors `HITL_ACTIONS` as `_GATE_TAB_ACTIONS`, the
  tab-key constants, and the general-tab key; `notification_tab_style.py` mirrors the
  shipped colors and icons. The established discipline is _mirror plus a parity test_,
  and §7 adds one that calls the real core and fails the moment the two disagree.

Do **not** add a speculative `getattr(tab, "priority", None)` read for a core field that
does not exist. If a future core release stamps priorities on `NotificationTabWire`,
that is a follow-up that deletes the Python ladder; the parity test is what will notice.

### 3. Config: `ace.notification_tabs.<key>.priority`

A third sibling of `color` and `icon`, keyed the same way (user-facing names — `snoozed`
and `muted`, never `__snoozed__` / `__muted__`; `_notification_tab_config_key` already
does that translation).

- Type: integer, `-1000 <= priority <= 1000`.
- **Absent means inherit.** Unlike `color: ""` and `icon: ""`, there is no empty-string
  reset sentinel, because `0` is a legitimate priority and any sentinel would collide
  with a real value. A lower config layer is overridden by writing the number you want —
  `priority: 50` puts a panel tab back where the ladder had it, and correctly stops
  rendering the mark.
- Python sanitizes exactly as it does color and icon: a non-`int`, a `bool` (Python's
  `bool` is an `int` — reject it explicitly, as `_indicator_max_counts_for_token`
  already does), or an out-of-range value resolves to "unset", never to a crash and
  never to a clamped value that silently reorders the strip.

**Shipped default** in `src/sase/default_config.yml`, alongside the `beads` color and
icon that already live there:

```yaml
beads:
  color: "#AF87FF"
  icon: "◈"
  priority: 0
```

`0` is the slot between any custom tag (`10`) and `Snoozed` (`-10`): below every tab
that is asking for attention, above the two put-away tabs. It also reads as the natural
neutral landmark on the ladder. Because the panel-kind default is `50`, this is a
genuine deviation and the Beads tab renders `▾` — the flagship use case demonstrates the
mark rather than hiding behind a built-in.

This is a config field, not a feature flag: per `sase/memory/sase_flags.md`, a value
users are meant to choose forever was never a flag.

### 4. The mark

One cell, immediately after the count, no separating space — `◈ Beads 3▾`. The mark hugs
the count like a superscript rather than floating between tabs, and one cell is the
whole budget a strip that already sheds labels under width pressure can afford.

| Direction                    | Glyph | Color     |
| ---------------------------- | ----- | --------- |
| effective **above** default  | `▴`   | `#FFAF00` |
| effective **below** default  | `▾`   | `#8A8A8A` |
| effective **equals** default | none  | —         |

`▴` U+25B4 and `▾` U+25BE are both covered by the bundled snapshot fonts (verified
against `tests/ace/tui/visual/fonts` with `fontTools`), and `▾` is already in the
codebase (`widgets/prompt_panel/_fold_language.py`). Amber `#FFAF00` is the existing
"this key/thing wants your eye" register; `#8A8A8A` is a receding grey that stays
legible against the active tab's `bold #00D7AF` label and `bold #87D7FF` count.

The glyph, not the color, carries the direction — up is sooner, down is later — so the
mark still reads on a monochrome terminal.

Expose it as one value so both call sites cannot drift:

```python
class NotificationTabPriorityMark(NamedTuple):
    glyph: str
    color: str

def notification_tab_priority_mark(
    tab: NotificationTagTab,
) -> NotificationTabPriorityMark | None: ...
```

**Tab strip** (`_render_tabs`, `notification_modal_tags.py`): after
`append(str(tab.count), count_style)` and before the trailing space, append the glyph
styled `mark.color` when the tab is active and `f"dim {mark.color}"` when it is not —
the same active/inactive rule the icon already uses two lines above, so the mark obeys
the strip's existing grammar instead of inventing one.

Two things fall out of using the existing `append` helper and must not be worked around:
the mark lands inside the tab's `_tab_ranges` entry, so clicking it selects that tab;
and `column` advances in terminal _cells_, so no range to its right is skewed. The
compact render keeps the mark after shedding the label (`◈ 3▾`) — a tab that has been
deliberately pushed down is exactly the tab whose position most needs explaining, and
`_build_content` measures the compact render too, so the extra cell is already inside
the overflow decision.

**Indicator tooltip** (`_tab_detail`, `notification_indicator.py:157`): the tooltip is
the legend that teaches what the strip's glyph means, so it shows the glyph _and_ the
number, adjacent, in the existing dim trailing detail:

```
 ◈ Beads    3   oldest 14m ago · ▾ priority 0
 ☾ Snoozed  4   next wakes in 43m
```

Join the existing detail and the new `{glyph} priority {n}` fragment with `" · "`, and
emit only the priority fragment when the existing detail is empty (Muted always, Snoozed
with no wake time). No new aligned column, so lines without a mark are unchanged.

**Indicator badge chips**: deliberately unchanged. A chip is `<icon><count>` — two or
three cells, at most `ace.notification_indicator_max_counts` (4) of them, on the shared
top bar. A mark there costs 33–50% more width to restate something the badge's own
left-to-right order already says, and the tooltip one hover away spells out. The badge
shows _what is waiting_; the strip and tooltip explain _why it sits where it does_.

**`sase doctor -C config.notification_tabs`** needs no change: it reads
`entry.get("icon")` only, and a priority cannot "collide" the way a duplicate glyph can.
Leave it alone.

## Implementation

### `src/sase/ace/tui/widgets/notification_tab_style.py`

Add, next to the existing color and icon chains and documented in the same
precedence-chain voice as the module docstring:

- `_KEY_TAB_PRIORITIES = {SNOOZED_TAB_KEY: -10, MUTED_TAB_KEY: -20}`
- `_KIND_TAB_PRIORITIES = {"hitl": 60, "panel": 50, "errors": 40, "general": 30, "tag": 10}`
  — kind strings mirror the core's `TAB_KIND_*` constants
- `_DONE_TAB_PRIORITY = 20`, `_UNKNOWN_KIND_TAB_PRIORITY = 10`
- `MIN_NOTIFICATION_TAB_PRIORITY = -1000`, `MAX_NOTIFICATION_TAB_PRIORITY = 1000`
- `_RAISED_PRIORITY_MARK` / `_LOWERED_PRIORITY_MARK` as `NotificationTabPriorityMark`
- `default_notification_tab_priority(tab) -> int` — pure, key rung then `done` rung then
  kind rung then fallback, per §1. Uses `_notification_tab_key(tab)` (the raw core key,
  which spells the internal `__muted__` / `__snoozed__`), **not**
  `_notification_tab_config_key`.
- `resolve_notification_tab_priority(tab) -> int` — configured override else default.
  Reads via `_notification_tab_config_key(tab)` so users write `snoozed` / `muted`.
- `notification_tab_priority_mark(tab) -> NotificationTabPriorityMark | None`

Extend `_ConfiguredTabStyle` with `priority: int | None = None` and parse it inside the
existing `_configured_tab_styles_for_token` loop, so a render still pays exactly one
config read for all three attributes. `_EMPTY_TAB_STYLE` becomes
`_ConfiguredTabStyle(color="", icon="", priority=None)`; the existing
`if style != _EMPTY_TAB_STYLE` guard then keeps working unchanged.

Add a `_sanitize_priority(raw: object) -> int | None` beside `_sanitize_color` /
`_sanitize_icon`, rejecting non-`int`, `bool`, and out-of-range.

Export the four new public names in `__all__` (symvision requires every public symbol to
have a call site; all four do).

### `src/sase/ace/tui/modals/notification_modal_tags.py`

- In `notification_tabs_from_core`, after the list is built, stable-sort it:
  `tabs.sort(key=lambda tab: -resolve_notification_tab_priority(tab))`. Import
  `resolve_notification_tab_priority` **lazily inside the function**, exactly as
  `_render_tabs` already lazily imports from `notification_tab_style` —
  `widgets/__init__` is loaded from inside this package's own import and a module-scope
  import cycles. Add a docstring line saying this is the single place tab order is
  finalized.
- In `_render_tabs`, add the mark per §4 (lazy-import `notification_tab_priority_mark`
  alongside the two resolvers already imported there).

No change to `NotificationTagTab`: priority is resolved on demand from the cached config
read, the same way color and icon already are, so hand-built test doubles across the
suite keep working and there is no second, staler source of truth for a tab's priority.

### `src/sase/ace/tui/widgets/notification_indicator.py`

Extend `_tab_detail` per §4. It already lazy-imports from `notification_tab_style` in
the enclosing methods; follow the same pattern.

### `src/sase/default_config.yml`

Add `priority: 0` under `ace.notification_tabs.beads`. Extend the block comment above
`notification_tabs` to mention priority in one clause.

### `src/sase/config/sase.schema.json`

Add `priority` to the `notification_tabs` `additionalProperties.properties` object
(`additionalProperties: false` is already set there, so the key is otherwise rejected):

```json
"priority": {
  "type": "integer",
  "description": "Sort weight for this tab in the notification panel's tab strip and the top-bar indicator; higher sorts earlier. Omit to inherit the default for the tab's kind (Gates 60, panel 50, Errors 40, General 30, Done 20, tag 10, Snoozed -10, Muted -20). A tab whose priority differs from that default renders a compact up/down mark.",
  "minimum": -1000,
  "maximum": 1000
}
```

No `default` key: the schema must not imply that an omitted priority is `0`, because `0`
is a real value distinct from "inherit".

## Verification

### Unit tests

`tests/test_notification_tab_style.py` (extend):

- `default_notification_tab_priority` for every kind in the ladder, plus a tab with an
  empty `kind`.
- `done` as a tag → `20`; `done` as a declared panel → `50`.
- A tab keyed `__muted__` with kind `tag` → `-20` (key rung beats kind rung).
- Configured override wins; configured `50` on `beads` cancels the shipped `0`.
- Sanitization: `"5"`, `True`, `1001`, `-1001`, `None`, a float → all fall back to the
  default and, critically, produce **no mark** (a rejected value must not look like a
  deviation).
- `notification_tab_priority_mark` returns `▴` above, `▾` below, `None` equal.

`tests/test_notification_tab_priority_order.py` (new):

- **Core parity.** Build `Notification` rows producing every tab kind at once — a
  `PlanApproval` gate, two declared panels, an error row, an untagged row, a
  `done`-tagged row, two other tag rows, a snoozed row, and a muted row — call the real
  `classify_notification_tabs` from `sase.core.notification_store_facade`, and assert
  that with **no** config overrides `notification_tabs_from_core` returns the core's key
  order unchanged. Then assert the same list is non-increasing in
  `default_notification_tab_priority`. Together those pin the ladder to the core's
  actual behavior; either drifting fails the test. Mark it with whatever marker the
  suite uses for tests that require the compiled binding, matching the nearest existing
  core-calling test.
- With the shipped `beads: priority: 0`, `Beads` sorts after every custom tag and before
  `Snoozed`.
- A tag configured to `70` sorts ahead of `Gates`.
- Two panel tabs with equal priority keep the core's label order (stability).

`tests/test_notification_modal_tag_strip.py` (extend):

- The strip renders `▾` for a lowered tab and nothing for the others.
- Click ranges stay correct with a mark present — put a marked tab **first** so a skewed
  range would misroute a click on a later tab; assert clicking the last tab still
  selects the last tab.
- Compact mode keeps the mark after shedding labels.

`tests/test_notification_indicator.py` (extend): chip order follows priority, and the
tooltip line for a deviating tab contains `▾ priority 0` while an unmarked tab's line is
unchanged.

`tests/test_config_schema_notification_tabs.py` (extend): `{"beads": {"priority": 0}}`
and `{"x": {"priority": -1000}}` validate; `"priority": "high"`, `1001`, `-1001`, and
`1.5` are rejected. The existing `test_the_bundled_defaults_validate_against_the_schema`
covers the new shipped value for free.

`tests/doctor/test_checks_config_notification_tabs.py`: add one case proving a tab
configured with `priority` only (no `icon`) is still ignored by the duplicate-icon
check.

### Visual

`tests/ace/tui/visual/test_tab_icon_glyphs.py`: add `▴` and `▾` to the audited glyph set
so the mark gets the same bundled-font-coverage and actual-ink audit every tab icon
gets. The cleanest seam is a small `_priority_mark_glyphs()` helper feeding
`_AUDITED_ICONS`, alongside the existing `_shipped_config_tab_icons()` and
`_artifact_tab_icons()`.

PNG goldens **will** change — Beads moves in the strip and gains `▾`. At minimum
`notification_beads_tab_120x40.png`, `notification_beads_typed_gates_120x40.png`, and
the `notification_indicator_*` snapshots. Run `just test-visual`, inspect the
actual/expected/diff artifacts under `.pytest_cache/sase-visual/`, confirm the only
deltas are the reorder and the one new glyph, then accept with
`just test-visual --sase-update-visual-snapshots` (or
`uv run pytest -m visual --sase-update-visual-snapshots`).

### Docs

`docs/configuration.md`, `ace.notification_tabs`: add a `priority` row to the field
table (default column: `inherit`), then a short paragraph carrying the §1 ladder table,
the "omit to inherit; there is no empty reset" rule, the `-1000..1000` bound, and the
shipped `beads: 0`.

`docs/notifications.md`, "Tabs and Ordering": state that tabs sort by priority
descending with the core's order as the tiebreak, add a `Priority` column to the
existing tab table, and note that the shipped `beads` priority puts `Beads` between
custom tags and `Snoozed` with a `▾` mark. Document the `▴`/`▾` vocabulary and that the
top-bar chips deliberately omit it. Fix the `Panel` row's "sorted alphabetically after
`Gates`" wording, which is now true only at equal priority.

### Gates

```bash
just install
just check
```

`just check-full` before landing (this touches the shipped config, the config schema,
and PNG goldens), run through `/sase_monitor` with a `--next` action, never inline.

## Out of Scope

- **Sender-declared priority.** A gate can declare its `panel`, `color`, and
  `panel_icon`, but not how far up the strip it sits. Priority is the reader's ranking
  of their own attention; letting senders bid on it is an escalation spiral that ends
  with every gate declaring itself urgent. If a specific sender's tab deserves a
  different slot, that is a config line the user owns.
- **Moving the ladder into `sase_core`.** Deferred deliberately (§2). Revisit when
  something other than the TUI renders these tabs; the parity test is the tripwire.
- **Named levels** (`priority: low`). Integers with documented landmarks are one type to
  validate, one type to render, and strictly more expressive; a name/int union would buy
  nothing but a second validation path.
- **Per-priority row ordering inside a tab.** Rows stay newest-first by activity time.
- **Reordering the `Snoozed`/`Muted` special cases** in the indicator badge.
