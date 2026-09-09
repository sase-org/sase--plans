---
tier: epic
title: Snoozed task beads and a per-tab notification indicator
goal: "A task bead can be snoozed with a wake time, an optional +1 target, and a reason
  that every SASE surface displays; snoozing a task bead always snoozes its
  notification; the wake raises a gate whose primary action closes the bead with an
  overridable preset reason; and the top-bar notification indicator shows one accurately
  colored count per notification-panel tab, with a snoozed count that appears only when
  nothing else is pending, a rich hover briefing, and a structural guarantee that no
  notification can appear in two tabs.

  "
phases:
  - id: notif-tab-model
    title: One tab per notification, counted in the core
    depends_on: []
    size: medium
    description: "notif-tab-model: make tab ownership a single-valued core rule that
      splits Snoozed from Muted, publish ordered per-tab counts on the notification
      snapshot, and reduce the Python tag helpers to a thin adapter.

      "
  - id: notif-tab-style
    title: Notification tab colors from senders and config
    depends_on:
      - notif-tab-model
    size: medium
    description: "notif-tab-style: add an optional sender-declared notification color,
      an ace.notification_tabs config block, and a deterministic resolver that gives
      every tab a stable accessible color.

      "
  - id: notif-indicator
    title: Per-tab notification indicator and hover briefing
    depends_on:
      - notif-tab-style
    size: medium
    description: 'notif-indicator: render one colored count per panel tab with the
      snoozed-only "<N>z" rule, bounded overflow, and a multi-line tooltip briefing,
      wired through every existing indicator refresh path.

      '
  - id: bead-snooze-core
    title: Snoozed task bead status in the Rust core
    depends_on: []
    size: medium
    description: "bead-snooze-core: add the snoozed status, its embedded snooze record,
      snooze/cancel mutations, +1 target wake, and time-based wake selection to the bead
      store, then mirror the model and presentation in Python.

      "
  - id: bead-snooze-cli
    title: sase bead snooze and snooze-aware detail surfaces
    depends_on:
      - bead-snooze-core
    size: medium
    description: "bead-snooze-cli: add the snooze command and its time, +1, and reason
      arguments, extend status filters and query tokens, and show snooze metadata
      everywhere bead detail is rendered.

      "
  - id: snooze-wake-gate
    title: BeadSnooze wake gate
    depends_on:
      - bead-snooze-core
    size: medium
    description: "snooze-wake-gate: register the bead_snooze gate kind with a
      close-primary, ready and re-snooze secondaries, and let a gate be born already
      snoozed so the wake needs no second timer.

      "
  - id: snooze-gate-reconciler
    title: One pending gate per task bead
    depends_on:
      - snooze-wake-gate
    size: medium
    description: "snooze-gate-reconciler: extend the bead task-gate chop to own triage
      and wake gates together, keep each snoozed bead's notification snoozed to its wake
      time, and add triage-time snoozing to the TaskTriage gate.

      "
  - id: bead-snooze-surfaces
    title: Snoozing from ACE, Telegram, and mobile
    depends_on:
      - bead-snooze-cli
      - snooze-gate-reconciler
    size: medium
    description: "bead-snooze-surfaces: add an ACE Beads-pane snooze action with a
      bead-aware duration modal, and confirm the Telegram and mobile gate paths carry
      the new options and metadata.

      "
  - id: snooze-verification
    title: Cross-surface verification and documentation
    depends_on:
      - notif-indicator
      - bead-snooze-surfaces
    size: small
    description:
      "snooze-verification: exercise the whole snooze lifecycle and the indicator end to
      end, check both repositories' gates, update user documentation, and record the
      memory updates that need owner approval."
proposed_by: bbugyi200.athena.uh
status: done
bead_id: sase-gn
create_time: 2026-09-09 19:50:04
---

- **PROMPT:**
  [prompts/202608/bead_snooze_and_notification_indicator.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/bead_snooze_and_notification_indicator.md)
- **BEAD:**
  [sase-gn](https://github.com/sase-org/sase--beads/blob/main/pages/sase-gn/README.md)

# Snoozed task beads and a per-tab notification indicator

## Context

Two related gaps motivate this epic.

**The notification indicator under-reports.** `NotificationIndicator`
(`src/sase/ace/tui/widgets/notification_indicator.py`) collapses everything into one
badge fed by four core counters (`priority`, `errors`, `rest`, `muted`) plus a bare `·`
for a muted backlog. Meanwhile the notification panel groups rows into tabs
(`build_notification_tag_tabs` in `src/sase/ace/tui/modals/notification_modal_tags.py`).
The two vocabularies are unrelated, so the badge cannot tell you what kind of attention
is waiting.

**Tab membership is multi-valued today, and that is a real bug.**
`_notification_modal_tab_keys` returns _every_ display tag for an untagged-panel
notification, so one row is counted in and owned by several tabs at once. Confirmed
empirically against this checkout:

```python
n = Notification(id="x", timestamp="...", sender="s", tags=["alpha", "beta"])
build_notification_tag_tabs([n])   # -> [("alpha", 1), ("beta", 1)]
notification_matches_tag_tab(n, "alpha") and notification_matches_tag_tab(n, "beta")   # -> True
```

Dismissing that single row makes two tabs disappear, which matches the behavior reported
in the prompt. This epic fixes it structurally rather than by patching the display.

**Snooze exists for notifications but not for beads.** Notifications already carry
`muted` + `snooze_until` + `resurfaced_at` (`src/sase/notifications/models.py`), with
atomic expiry reconciliation and a `next_snooze_deadline` the TUI arms a timer against.
Task beads have no way to say "not now, ask me in three days" — a ready task raises a
`TaskTriage` gate every reconciliation until it is launched or closed, and the only way
to quiet it is to close it.

The epic adds a `snoozed` task-bead status that reuses the notification snooze machinery
instead of inventing a second timer, and rebuilds the indicator on the same tab
vocabulary the panel already uses.

## Design decisions

These decisions are binding for every phase; do not relitigate them mid-implementation.

**D1 — Tab ownership is single-valued and lives in the Rust core.** Per the repository's
Rust-core boundary rule, "which bucket does this notification belong to" is backend
domain logic: the panel, the indicator, and the mobile snapshot must agree. The core
assigns each notification exactly one tab key by this precedence:

1. `__snoozed__` — `muted` **and** `snooze_until` is set
2. `__muted__` — `muted` with no `snooze_until`
3. the declared gate `panel` key (e.g. `beads`)
4. `hitl` — `action` in the HITL action set
5. `errors` — `is_error`
6. the **first** declared display tag, in stored order
7. `general` — everything else

Only labels, glyphs, colors, widths, and ordering _within a rendered chip_ stay in
Python. Making tab ownership single-valued in one place is the fix for the multi-tab
bug: it becomes unrepresentable, not merely unlikely.

**D2 — Snoozing a bead creates its notification already snoozed.** A snoozed bead gets
exactly one pending `bead_snooze` gate whose notification is created with
`muted=True, snooze_until=<wake time>`. The existing notification snooze expiry
resurfaces it at the wake time. There is no second scheduler, the bead's row is visible
in the panel's Snoozed tab the whole time, and "snoozing a task bead always snoozes the
corresponding sase notification" holds by construction rather than by convention.

**D3 — One pending gate per task bead, owned by one reconciler.**
`sase_chop_bead_task_triage.py` already reconciles `TaskTriage` gates with per-bead
generations and a presentation fingerprint. It is extended to own `bead_snooze` gates
too, in the same lane state and under the same lock, rather than adding a second chop
that could race it. The chop keeps its registered name so no config migration is needed;
its description in `src/sase/default_config.yml` is updated to match its widened scope.

**D4 — The wake time and the +1 target are separate wake conditions, and the first one
wins.** Reaching the +1 target promotes the bead to `ready`, where the _existing_ triage
path raises the `TaskTriage` gate. Reaching the wake time raises the new `bead_snooze`
gate. A bead never has both gates pending.

**D5 — `sase bead update -s snoozed` is refused.** Snoozing requires a wake time, so
status alone cannot express it. The update path fails with a message pointing at
`sase bead snooze`, mirroring how `sase bead close` is the preferred close path.

**D6 — Duration input reuses `parse_duration`.**
`sase.xprompt._directive_time.parse_duration` already backs the notification snooze
modal (`src/sase/ace/tui/modals/snooze_duration_modal.py`). Every new snooze time
argument accepts the same relative vocabulary (`30m`, `2h`, `1h30m`, `3d`) plus an
ISO-8601 absolute form.

**D7 — Memory files are not edited by this epic.** No phase may modify
`sase/memory/*.md`, `AGENTS.md`, or the generated provider shims. The `sase_beads.md`
status list will be stale once this lands; the final phase records that as a proposal
for the owner to approve, and nothing more.

**Cross-repository note.** Several phases change the sibling Rust core crate. Open it
with the `/sase_repo` skill and use the path that command prints; never locate or clone
it another way. Changes to the crate need their own commit and their own `cargo` checks
in that repository, in addition to this repository's `just check`.

---

## One tab per notification, counted in the core

**Goal**: every notification belongs to exactly one panel tab, Snoozed is a first-class
tab distinct from Muted, and the notification snapshot carries ordered per-tab counts so
the indicator never has to re-derive them.

### Rust core (`crates/sase_core/src/notifications/`)

Add to `wire.rs`:

```rust
#[derive(Debug, Clone, Default, PartialEq, Eq, Serialize, Deserialize)]
pub struct NotificationTabWire {
    pub key: String,                          // "hitl" | "errors" | "general" | "__muted__" | "__snoozed__" | panel | tag
    pub kind: String,                         // "hitl"|"panel"|"errors"|"general"|"tag"|"muted"|"snoozed"
    pub count: u64,
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub oldest_activity_at: Option<String>,   // min activity timestamp in the tab
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub next_wake_at: Option<String>,         // snoozed tab only: min snooze_until
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub color: Option<String>,                // sender-declared color, if any (see notif-tab-style)
}
```

- Add `pub tabs: Vec<NotificationTabWire>` to `NotificationStoreSnapshotWire` with
  `#[serde(default)]` so older readers and fixtures keep deserializing.
- In `store.rs`, add `fn tab_key_for(n: &NotificationWire) -> (String, &'static str)`
  implementing the D1 precedence, and extend the existing `counts_for` pass into a
  `tabs_and_counts_for` pass that fills both `NotificationCountsWire` and the ordered
  `tabs` vector in one iteration over the same rows `counts_for` already visits (skip
  `read || silent`; dismissed rows are already excluded upstream). Do not add a second
  traversal.
- Ordering must match the panel exactly: `hitl`, then panel keys sorted by their
  casefolded label, then `errors`, `general`, `done`, then remaining tag keys sorted by
  casefolded label, then `__snoozed__`, then `__muted__`. Port the existing ordering
  from `build_notification_tag_tabs` and keep `__snoozed__` immediately before
  `__muted__`.
- Add a pure classification entry point used by the modal for in-memory rebuilds:
  `pub fn classify_notification_tabs(rows: &[NotificationWire]) -> NotificationTabClassificationWire`,
  returning both the ordered `tabs` and a `row_tab_keys: BTreeMap<String, String>`
  mapping notification id to its single owning tab key. Returning the per-row map
  matters: the Python caller must resolve membership with a dict lookup, never with one
  FFI call per row.
- Reuse `mobile::mobile_notification_priority_from_wire` / `..._error_from_wire` for the
  error branch; do not add a second error rule.
- Tests in the crate: precedence for every branch; a two-tag row lands in exactly one
  tab and is counted once; a snoozed row is excluded from its panel/HITL/error tab;
  `sum(tab.count) == priority + errors + rest + muted`; ordering is stable;
  `next_wake_at` is the minimum future `snooze_until`.

### Python binding and facade

- Extend `src/sase/core/notification_store_wire.py` with a `_NotificationTabWire`
  dataclass, add `tabs` to `NotificationStoreSnapshotWire`, and decode it with the
  existing `known_field_kwargs` tolerance so an older core binary still loads.
- Add `classify_notification_tabs(notifications) -> NotificationTabClassification` to
  `src/sase/core/notification_store_facade.py`, following the shape of the existing
  `require_rust_binding` helpers.
- Expose `snapshot.tabs` through `src/sase/notifications/store.py` unchanged in shape.

### Python adapter (`src/sase/ace/tui/modals/notification_modal_tags.py`)

- Add `SNOOZED_TAB_KEY = "__snoozed__"` beside `MUTED_TAB_KEY`, and `"Snoozed"` to
  `_SYNTHETIC_TAB_LABELS`.
- Replace the body of `_notification_modal_tab_keys` with a single-key function
  `notification_modal_tab_key`, and delete the list-returning form.
  `notification_matches_tag_tab(notification, tag)` becomes `tab_key == tag`.
- `build_notification_tag_tabs(notifications)` now calls `classify_notification_tabs`
  **once** and returns `NotificationTagTab` rows built from the core result. Keep the
  dataclass, add `kind: str`, `oldest_activity_at`, `next_wake_at`, and `color` fields
  so downstream phases do not need another read. Keep `shorten_notification_tag` and
  `NotificationTagStrip` as-is apart from consuming the new tab list.
- `NotificationModal` (`notification_modal.py`) keeps a `dict[str, str]` of row id to
  tab key alongside `self._notifications`, refreshed whenever `_tag_tabs()` is rebuilt,
  and uses it for per-row membership. Rebuild frequency is unchanged (dismiss, mute,
  snooze, tab switch); the page limit is 100 rows (`DEFAULT_NOTIFICATION_PAGE_LIMIT`),
  so one FFI call per rebuild is well inside budget and no keystroke path gains an FFI
  call.

### Reserved panel names

In `src/sase/notification_gates/presentation.py`, extend `RESERVED_GATE_PANELS` to
`{"errors", "general", "muted", "snoozed", "hitl"}`. `hitl` is a synthetic tab key today
and a gate declaring it would silently collide with the HITL tab; `snoozed` becomes
reserved for the same reason. Leave `done` unreserved — it is an ordinary tag that
merely receives priority ordering, not a synthetic key. Update the error message and the
existing reserved-panel tests.

### Tests in this repository

- `tests/test_notification_modal_tags.py` (or the existing equivalent): the two-tag
  regression — one row, one tab, count of 1; a muted row with `snooze_until` lands in
  Snoozed and not Muted; a muted row without it lands in Muted; a `panel: beads` gate
  that is snoozed leaves the Beads tab.
- A parity test asserting the Python adapter's tab list equals the snapshot's `tabs` for
  a shared fixture, so the two entry points cannot drift.

---

## Notification tab colors from senders and config

**Goal**: every tab has a stable, accessible color that the user can override in config
and a sender can suggest, with a deterministic fallback so a brand-new tag tab never
renders colorless.

### Sender-declared color

- Add `color: str | None = None` to `Notification` (`src/sase/notifications/models.py`)
  and to `NotificationWire` in the Rust core,
  `#[serde(default, skip_serializing_if = "Option::is_none")]`.
- Validate as `^#[0-9A-Fa-f]{6}$` — reject anything else at write time rather than
  storing junk that renders as an unstyled chip. Reuse the pattern already used for
  `ace.tribes[].color` in `src/sase/config/sase.schema.json`.
- Accept it on gate creation: `presentation.color` in the gate spec, validated in
  `src/sase/notification_gates/validation.py` next to `normalize_gate_panel`, and
  applied in `_build_notification` (`src/sase/notification_gates/service.py`).
- Accept it on `sase notify create` JSON input, alongside `icon`.
- In the core's tab pass, a tab's `color` is the declared color of the first row in that
  tab's activity sort order that declares one; `None` when no row does. Deterministic,
  no scanning at render time.

### User configuration

Add an `ace.notification_tabs` block to `src/sase/default_config.yml` and
`src/sase/config/sase.schema.json`, modelled directly on the existing `ace.tribes`
block:

```yaml
ace:
  # Per-tab notification indicator styling. Keys are notification-panel tab
  # keys: the synthetic "hitl", "errors", "general", "snoozed", and "muted"
  # tabs, a gate-declared panel name such as "beads", or a notification tag.
  # An empty color string restores the built-in default for that tab.
  notification_tabs:
    hitl:
      color: "#FF8700"
    errors:
      color: "#FF5F5F"
    beads:
      color: "#AF87FF"
    general:
      color: "#FFD700"
    snoozed:
      color: "#6C6C6C"
    muted:
      color: "#5FAFAF"
  # Maximum per-tab counts rendered in the top-bar indicator before the
  # remainder collapses into a single dim "+N" chip.
  notification_indicator_max_counts: 4
```

Config keys use the user-facing tab names (`snoozed`, `muted`), not the internal
`__snoozed__` / `__muted__` keys; the resolver maps between them.

### Resolver

New module `src/sase/ace/tui/widgets/notification_tab_style.py`:

```python
def resolve_notification_tab_color(tab: NotificationTagTab) -> str: ...
def notification_tab_label(tab: NotificationTagTab) -> str: ...
```

Precedence, highest first:

1. `ace.notification_tabs.<user-facing key>.color`, when non-empty
2. `tab.color` — the sender-declared color carried on the snapshot tab
3. the built-in default for a synthetic key or a known panel key
4. a stable auto-color: `_AUTO_PALETTE[fnv1a32(tab.key) % len(_AUTO_PALETTE)]`

`_AUTO_PALETTE` is a fixed list of six 256-color-safe hexes chosen for legibility on the
ACE top bar and visually distinct from the built-in defaults, so two different tag tabs
are unlikely to collide and the same tag always gets the same color across restarts.

Config reads must go through the cached config token path used elsewhere in the TUI
(`current_config_token()`), never a fresh read per render — the indicator repaints on
every notification poll.

### Tests

- Each precedence rung wins over the ones below it; an empty config string falls through
  to the default.
- The auto-palette is stable for a given key across processes and never returns a
  built-in default color.
- An invalid `presentation.color` is rejected at gate creation with a `GateError`, and
  an invalid stored color degrades to the resolver's next rung instead of raising in the
  render path.

---

## Per-tab notification indicator and hover briefing

**Goal**: the top-bar indicator shows one colored count per panel tab, in panel order,
with the snoozed special case, an overflow guard, and a tooltip that actually briefs the
user.

### Rendering contract (`src/sase/ace/tui/widgets/notification_indicator.py`)

Replace `set_counts(priority, rest, muted)` with
`set_tabs(tabs: Sequence[NotificationTagTab])`. Keep `set_counts` and `set_count` as
thin deprecated shims that synthesize equivalent tabs, so nothing outside the refresh
paths breaks mid-epic; remove them in this same phase once every caller is migrated.

`_build_content(tabs)` renders:

- **Nothing pending** — `" ✉ 0 "`, `dim`. Unchanged.
- **Snoozed only** — `" ✉ 4z "` where `4` and the `z` suffix both use the resolved
  snoozed color at `dim` weight (default `#6C6C6C`). This is the only case where a
  suffix letter appears.
- **Anything else** — `" ✉ "` then one chip per visible tab, each chip being the count
  in that tab's resolved color at `bold` weight, joined by a `·` separator styled
  `#3A3A3A`. Example: `" ✉ 2·3·1 "`.
- **Snoozed suppression** — when any non-snoozed tab exists, the snoozed tab contributes
  no chip. The tooltip still reports it.
- **Overflow** — at most `ace.notification_indicator_max_counts` chips, taken in panel
  order; if more tabs remain, append a chip `+K` styled `dim` where `K` is the number of
  suppressed tabs. Suppressed tabs are still fully described in the tooltip.

Chip order is exactly the panel's tab order, so the leftmost count is the leftmost tab.
That correspondence is the whole reason the counts are readable without labels, and it
must be asserted in tests.

### Tooltip

`_build_tooltip(tabs)` returns a `rich.text.Text` (verified accepted by `Widget.tooltip`
on the pinned Textual 8.0.1; `textual.content.Content` also works if preferred). Shape:

```
6 unread · 3 tabs
 HITL      2   oldest 14m ago
 Beads     3   oldest 2h ago
 Errors    1   oldest 5m ago
 Snoozed   4   next wakes in 43m
 Muted     2
Click to open the notification panel
```

- Header counts only non-snoozed, non-muted rows as "unread"; the snoozed and muted
  lines are informational.
- Labels are colored with each tab's resolved color so the tooltip teaches the chip
  colors.
- Relative times use the existing `format_relative_time` and `format_relative_until`
  (`src/sase/notifications/models.py`); do not write new formatters.
- Empty state stays `"No unread notifications"`.
- Label column is padded to a fixed width computed from the longest visible label,
  capped by `shorten_notification_tag`.

### Refresh wiring

Every site that currently calls
`indicator.set_counts(counts.priority + counts.errors, counts.rest, counts.muted)` now
passes `snapshot.tabs`:

- `src/sase/ace/tui/actions/lifecycle.py` — `_read_notifications_for_startup` returns
  the tab list in its `NotificationStartupState` tuple instead of three ints, and
  `_initialize_agent_tracking` unpacks it. Update the type alias and the docstring
  describing the tuple.
- `src/sase/ace/tui/actions/agents/_notification_polling.py` — all three call sites
  (lines around 149, 235, and 281).
- `src/sase/ace/tui/actions/agents/_notification_provider_models.py` — carry `tabs`
  through the provider count snapshot next to the existing `counts`.

No new disk read, no new refresh path, and no work added to a keystroke path: the tab
list arrives on the snapshot the poll already fetches.

### Tests (`tests/test_notification_indicator.py`)

Rewrite around the new contract, keeping the existing empty-state and click-dispatch
tests:

- one tab renders one chip; three tabs render three chips joined by `·`
- chip order equals input tab order
- snoozed-only renders `4z`; adding any other tab removes the snoozed chip entirely
- muted keeps its own chip (it is a normal tab now, only snoozed is special-cased)
- overflow beyond the configured maximum renders `+K` and the tooltip still lists every
  tab
- tooltip content for mixed, snoozed-only, and empty states
- an indicator-vs-panel invariant test: for a shared fixture, the chips the indicator
  would render correspond one-to-one with the tabs `build_notification_tag_tabs`
  produces, with equal counts

---

## Snoozed task bead status in the Rust core

**Goal**: `snoozed` is a real task-bead status with durable, validated snooze metadata
and two wake conditions.

### Status and record (`crates/sase_core/src/bead/`)

- `wire.rs`: add `Snoozed` to `StatusWire` (serde `snake_case` gives `"snoozed"`), and a
  new record:

```rust
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub struct BeadSnoozeWire {
    pub until: String,                  // ISO-8601 with timezone
    pub snoozed_at: String,
    pub snoozed_by: String,
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub plus_one_target: Option<u32>,   // absolute total +1 count that wakes the bead
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub plus_one_baseline: Option<u32>, // +1 count when the snooze started, for display
    #[serde(default = "empty_string", deserialize_with = "deserialize_string_default_empty")]
    pub reason: String,
}
```

and
`#[serde(default, skip_serializing_if = "Option::is_none")] pub snooze: Option<BeadSnoozeWire>`
on `IssueWire`.

- `mutation.rs`: extend `status_from_str` / `status_to_str` with `"snoozed"`.
- Validation in `IssueWire::validate`, mirrored in Python:
  - only `Task` issues may be `snoozed` (same shape as the existing ready-status rule
    and its error message)
  - `snooze.is_some()` **iff** `status == Snoozed`
  - `until` must parse as an RFC-3339 timestamp with an offset
  - `plus_one_target`, when present, must be strictly greater than `plus_one_baseline`
  - `snoozed_at` and `snoozed_by` must be non-blank

The single embedded record, cleared on wake, is deliberate: no flat field can drift out
of sync with the status, and `sase bead history` already replays the event stream when
past snoozes matter, so no separate history list is needed.

### Mutations (`crates/sase_core/src/bead/mutation.rs`)

```rust
pub fn snooze_task(
    beads_dir: &Path, issue_id: &str, until: &str, plus_ones: Option<u32>,
    reason: &str, actor: &str, now: Option<String>,
) -> Result<BeadMutationOutcomeWire, BeadError>;

pub fn cancel_task_snooze(
    beads_dir: &Path, issue_id: &str, actor: &str, now: Option<String>,
) -> Result<BeadMutationOutcomeWire, BeadError>;

pub fn wake_due_task_snoozes(
    beads_dir: &Path, now: &str,
) -> Result<BeadSnoozeWakeOutcomeWire, BeadError>;
```

- `snooze_task` takes the standard mutation lock, rejects non-task beads, rejects
  `Closed` and `InProgress` and `Claimed` sources (only `Open` and `Ready` may be
  snoozed), rejects an `until` at or before `now`, computes
  `plus_one_baseline = plus_one_count()` and `plus_one_target = baseline + plus_ones`,
  sets the status and record, and appends a `TaskSnoozed` event. Re-snoozing an
  already-snoozed bead is allowed and replaces the record (that is the "snooze for
  longer" path) — append a fresh event rather than mutating in place silently.
- `cancel_task_snooze` clears the record and returns the bead to `Ready`, appending
  `TaskSnoozeCanceled`.
- `wake_due_task_snoozes` is a read-mostly selector for the chop: it returns the ids
  whose `until <= now` **without** mutating them. Time-based wake does not change the
  bead's status by itself; the status changes only when the human answers the wake gate.
  This keeps the store honest about what the user actually decided.
- `add_task_plus_one` gains one branch: when the bead is `Snoozed` and the new
  `plus_one_count()` reaches `snooze.plus_one_target`, clear the record, set `Ready`,
  and append a `TaskSnoozeWoken { cause: "plus_one" }` event plus an attributed note
  using the preset text
  `"Reopened by +1 threshold: reached {target} +1s while snoozed until {until}."`. When
  the target is not yet reached, the bead stays `Snoozed` — critically, the existing
  `Open | Closed -> Ready` promotion must **not** fire for a snoozed bead.

### Events, projection, and reads

- `events.rs`: add `TaskSnoozed`, `TaskSnoozeCanceled`, `TaskSnoozeWoken` to
  `BeadEventOperationWire` and their payloads to `BeadEventPayloadWire`; apply them in
  the reducer so a replay reconstructs `snooze` and `status` exactly.
- `jsonl.rs` / `read.rs` / `search.rs`: carry `snooze` through the generated
  `issues.jsonl` projection and accept `snoozed` in status filters.
- `schema.rs`: add snoozed-task fixtures to the schema tests alongside the existing
  `task-ready` rows.

### Python mirror

- `src/sase/bead/model.py`: `Status.SNOOZED = "snoozed"`; a frozen `SnoozeRecord`
  dataclass with a `validate()` matching the Rust rules; `snooze: SnoozeRecord | None`
  on `Issue`; the same constraints in `Issue.validate()`.
- `src/sase/bead_status_presentation.py`: add `"snoozed"` to `BeadStatusValue` and a
  `_BeadStatusPresentation` entry — `cli_glyph="◈"`, `tui_glyph="◈"`,
  `cli_style="\x1b[90m"`, `rich_color="#6C6C6C"`, `label="Snoozed"`. The grey matches
  the notification indicator's snoozed color on purpose: one visual language for
  "deferred" across both subsystems. Place it between `ready` and `in_progress` in
  `BEAD_STATUS_PRESENTATIONS` so `bead_status_display_order()` reads as a lifecycle.
  Everything that iterates that mapping — `filter_query.py`, `cli_query.py`,
  `cli_dep_render.py`, the Artifacts bead filter bar — picks the new status up for free;
  verify each and fix any hard-coded status list found.
- `src/sase/bead/model.py` codecs and `src/sase/core/bead_read_facade.py`: decode
  `snooze` from the wire.

### Tests

Crate tests for every validation rule, both wake conditions, event replay round-trips,
re-snooze replacing the record, and the +1 branch (below target stays snoozed; at target
promotes with the preset note). Python tests for the model mirror, the presentation
entry, and the status-list consumers.

---

## sase bead snooze and snooze-aware detail surfaces

**Goal**: the CLI can snooze, cancel, filter, and display snoozes, and every bead-detail
surface shows the wake conditions and reason.

### `sase bead snooze`

Register in `src/sase/main/parser_bead_lifecycle.py`:

```
sase bead snooze <ID>... -u <time> [-p <count>] [-r <reason>]
sase bead snooze <ID>... --cancel
```

- `-u/--until` (**required** unless `--cancel`): a duration (`30m`, `2h`, `1h30m`, `3d`)
  parsed by `sase.xprompt._directive_time.parse_duration`, or an ISO-8601 absolute
  timestamp. Reject non-positive durations and past absolute times with a message
  showing accepted forms.
- `-p/--plus-ones` (optional, positive int): wake when this many **additional** +1s
  arrive.
- `-r/--reason` (optional free text).
- `--cancel`: return the bead to `ready` and clear the record.
- Accepts multiple ids and applies atomically, matching `sase bead update`'s batch
  semantics.
- Handler in `src/sase/bead/cli_crud.py` beside the close handler; commit through the
  existing `bead_store_mutation` / `auto_commit_bead_store` path with a `snooze`
  mutation commit message.
- Print a confirmation naming the resolved absolute wake time in the configured timezone
  plus its relative form —
  `Snoozed sase-a1 until Aug 9 09:00 (in 3d) · wakes early at 3 +1s`.

### Refusing the status shortcut

In the update path, reject `--status snoozed` with
`"snoozed requires a wake time; use: sase bead snooze <id> -u <time>"`. Add `"snoozed"`
to the `--status` choices for `sase bead list` and `sase bead search`
(`src/sase/main/parser_bead_queries.py`) but **not** to `sase bead update`'s `--status`
choices, so argparse rejects it before the handler.

### Query and filter tokens

`src/sase/bead/filter_query.py` derives statuses from `bead_status_display_order()`, so
`status:snoozed` and `-status:snoozed` work once the presentation entry exists — verify,
do not duplicate. Consider whether `DEFAULT_BEAD_FILTER_QUERY` (`-status:closed`) should
also hide snoozed beads by default: it should **not**, because a snoozed task is still
live work the user chose to defer, and hiding it would make the status feel like a black
hole.

### Detail rendering

- `src/sase/bead/cli_detail_prose.py` and `cli_detail_style.py`: a `Snooze` property
  block on a snoozed bead — `until` as absolute + relative, `+1 target` as
  `2 more (3 total)` when set, and the reason. Use the existing
  `bead_time_presentation.py` helpers.
- `src/sase/bead/cli_detail_json.py`: a `"snooze"` object with `until`, `snoozed_at`,
  `snoozed_by`, `plus_one_target`, `plus_one_baseline`, `reason`, and a derived
  `plus_ones_remaining`. `null` when not snoozed.
- `src/sase/ace/tui/widgets/artifacts/beads_detail.py`: a snooze chip in
  `bead_properties_header` next to the status chip, a `Snooze` row in the properties
  table, and a `_readiness_label` branch reading `Snoozed · wakes in 3d` (mirror the
  existing `Status.READY` / `Status.CLAIMED` branches, and extend the `_readiness_chip`
  status set).
- `src/sase/ace/tui/widgets/artifacts/beads_rendering.py`: the list row shows the
  snoozed glyph and a dim relative wake time so a snoozed bead reads correctly without
  opening detail.
- `src/sase/bead_pages/rendering.py` and `rendering_tables.py`: include snooze metadata
  in the generated bead pages.

### Tests

Argument parsing (durations, absolute times, rejections), the batch path, `--cancel`,
the refused `update -s snoozed`, JSON detail shape, and rendering snapshots for each
surface.

---

## BeadSnooze wake gate

**Goal**: a registered gate kind whose primary action closes the bead with an
overridable preset reason, with secondaries to make it ready or to snooze longer — and
which is born already snoozed.

### Gate spec being born snoozed

Add an optional `presentation.snooze_until` (ISO-8601 with offset) to the gate contract:

- validate it in `src/sase/notification_gates/validation.py` beside
  `normalize_gate_panel`
- apply it in `_build_notification` (`src/sase/notification_gates/service.py`) by
  setting `muted=True` and `snooze_until` on the constructed `Notification`

This makes gate creation a single atomic append: there is no window in which the
notification is briefly unread, and no crash between "create gate" and "snooze
notification" can leak a premature alert. It is a general capability — any producer that
knows a gate is not actionable until a future time can use it.

### Adapter and kind

Register in `src/sase/notification_gates/adapters.py`:

```python
GateAdapter(
    kind="bead_snooze",
    display_title="Snoozed Task",
    action="BeadSnooze",
    pending_action_kind="bead_snooze",
    sender="bead",
    request_filename="request.json",
    response_filename="response.json",
    legacy_directory_key="bead_snooze_dir",
    auto_policy="forbidden",
    neutral_only=True,
    generic_form=True,
)
```

Add `"BeadSnooze"` to `PRIVILEGED_GATE_ACTIONS` so the notification stays unread until
the human answers it, and to the priority action sets in **both**
`src/sase/notifications/priority.py` and `crates/sase_core/src/notifications/mobile.rs`
— those two lists are parallel today and drifting them would misclassify the gate on one
surface. Do **not** add it to `_HITL_ACTIONS`: the gate declares `panel: "beads"`, which
outranks the HITL rung in the D1 precedence, so it lands in the Beads tab alongside
`TaskTriage` once awake and in the Snoozed tab while it sleeps.

### Options

New module `src/sase/bead/snooze_gate.py`, mirroring the structure of
`src/sase/bead/task_gate.py`:

- query `close OR ready OR snooze`, primary branch `("close",)`
- **close** (primary, icon `✕`, `feedback: "optional"`) — closes the bead. Empty
  feedback uses the preset reason
  `"Snoozed until {until} with no new evidence; closing as stale."`; any feedback text
  replaces it verbatim. Resolution `canceled`.
- **ready** (icon `◇`, `feedback: "optional"`) — sets the bead to `ready` with a preset
  note `"Woken from snooze and returned to triage."`. The existing `bead_task_triage`
  reconciliation then raises the `TaskTriage` gate; this option deliberately does not
  raise it directly.
- **snooze** (icon `◈`, `feedback: "required"`) — re-snoozes. The feedback text is the
  new duration or absolute time, parsed with the same `parse_duration` vocabulary.
  Unparsable input fails the option command with a message listing accepted forms and
  leaves the gate pending, so a typo cannot lose the bead.

The generic gate form supports one free-text feedback field per branch and no structured
input widgets; carrying the new duration in the required feedback field is why `snooze`
uses `feedback: "required"` rather than an input schema.

### Validation, payload, preview, side effects

- `src/sase/notification_gates/kind_validation/bead_snooze.py` and
  `bead_snooze_payload.py`, structured exactly like the `task_triage` pair: an
  exhaustive field set, option-by-option comparison against the registered adapter,
  resource checks, and a preview check. Register both in `kind_validation/__init__.py`
  and dispatch from `validation.py`, with
  `expected_primary["bead_snooze"] = ("close",)`.
- Payload fields: `bead_id`, `project`, `title`, `created_at`, `size`, `refs`,
  `plus_one_count`, `plus_one_evidence`, `close_history`, `snooze` (`until`,
  `snoozed_at`, `snoozed_by`, `plus_one_target`, `plus_one_baseline`, `reason`).
- Preview `task.md` reuses `render_task_triage_preview` and prepends a snooze block: who
  snoozed it, when, why, the wake time, and +1 progress. Do not fork the renderer.
- Side effects in `GateAdapter.apply_side_effects`: a `bead_snooze` branch calling
  `translate_bead_snooze_response` / `close_bead_snooze` / `ready_bead_snooze` /
  `resnooze_bead_snooze`, following the `task_triage` branch's shape including its
  response-file rewrite for ids produced by the action.
- Command scripts registered like `TASK_TRIAGE_COMMAND_PATHS`, executed through the same
  `execute_*_gate_command` entry point pattern, with `result_schema` per option.

### Tests

Spec validation (each malformed shape rejected), preview rendering, each option's
translation and side effect, the preset-reason-versus-override behavior for `close`,
duration rejection for `snooze`, and a test that a gate created with
`presentation.snooze_until` produces a notification with `muted=True` and the expected
`snooze_until`.

---

## One pending gate per task bead

**Goal**: one reconciler owns every task-bead gate, a snoozed bead's notification stays
snoozed to its wake time, and a ready task can be snoozed straight from its triage gate.

### Extend the reconciler (`src/sase/scripts/sase_chop_bead_task_triage.py`)

Keep the registered chop name and its lane placement; widen its scope and bump
`_STATE_SCHEMA_VERSION` to 3.

- `_ProjectState` gains a `kinds: dict[str, str]` mapping bead id to the gate kind
  currently pending for it, so the existing `gates` / `generations` / `fingerprints`
  maps stay one-entry-per-bead. Tolerate a version-2 state file by treating every
  recorded gate as `task_triage`.
- Read both `Status.READY` and `Status.SNOOZED` tasks in one pass. A bead's **expected**
  gate kind is `task_triage` when ready and `bead_snooze` when snoozed.
- Reconciliation per bead, in this order:
  1. pending gate of the wrong kind for the current status → cancel it (reason
     `bead_status_changed`) and drop the state entry, so the correct kind is created in
     the same tick
  2. no bead → cancel a pending gate (existing `task_bead_no_longer_ready` path)
  3. pending gate of the right kind with a changed presentation fingerprint → cancel and
     recreate (existing behavior; extend `_presentation_fingerprint` to include the
     snooze record)
  4. no pending gate → create the expected kind. `bead_snooze` gates are created with
     `presentation.snooze_until = <bead wake time>`.
  5. pending `bead_snooze` gate whose notification's `snooze_until` no longer matches
     the bead's wake time → re-snooze the notification with
     `mark_snoozed(notification_id, until)`. This is the self-healing step that keeps
     the D2 invariant true after a crash, a manual unmute, or a re-snooze.
- Post-condition asserted in tests: for every task bead, **at most one** pending gate
  exists across both kinds. This is the bead-side counterpart to the notification-side
  single-tab guarantee.
- Update the chop's description in `src/sase/default_config.yml` to describe both gate
  kinds. Its `checks` lane runs every 300s; a five-minute wake granularity is
  appropriate for a snooze measured in hours or days, and the notification itself
  resurfaces on its own snooze deadline independently of the chop.

### Snooze from the TaskTriage gate

Triage time is when the user is actually looking at a ready task, so it is the most
valuable place to defer one.

- Add a third option `snooze` to the TaskTriage spec in `src/sase/bead/task_gate.py`:
  icon `◈`, `feedback: "required"` carrying the duration, `result_schema` following the
  existing per-option schema helper.
- Update `TASK_TRIAGE_OPTION_IDS`, `TASK_TRIAGE_QUERY` (`launch OR close OR snooze`),
  `TASK_TRIAGE_COMMAND_PATHS`, the `branches` tuple, the expected-feedback map, and
  every assertion in `src/sase/notification_gates/kind_validation/task_triage.py`.
  `primary_branch` stays `("launch",)`.
- `translate_task_triage_response` gains a `snooze` decision that calls the core snooze
  mutation; the bead leaves `ready`, and the reconciler replaces the answered triage
  gate with a snoozed `bead_snooze` gate on its next tick.
- The optional +1 target is not expressible in a single feedback field. Accept a compact
  `"<duration> [+<N>]"` form — `3d`, `3d +2` — parsed by one shared helper used by both
  this option and the wake gate's `snooze` option, with the accepted forms named in the
  option label and in the parse-error message.

### Tests

Kind-swap on status change, no-double-gate invariant, fingerprint refresh including
snooze fields, notification re-snooze self-healing, version-2 state migration, and each
new triage decision including the `+N` parse.

---

## Snoozing from ACE, Telegram, and mobile

**Goal**: the user can snooze a task bead from the surfaces where they actually meet
beads, and every surface shows the snooze metadata.

### ACE Beads pane

- New action `action_beads_snooze` in
  `src/sase/ace/tui/actions/_artifacts_beads_mutations.py`, beside `action_beads_close`.
  It opens a bead-flavored duration picker, then submits the mutation through the
  existing `_submit_bead_mutation` path (optimistic UI, off-thread worker, typed
  outcome) so it obeys the TUI's background-task rules.
- New `BeadSnoozeModal` in `src/sase/ace/tui/modals/`, built on the same
  `DurationChoiceModal` base as `SnoozeDurationModal`, with presets `4 hours`,
  `Tomorrow morning`, `3 days`, `1 week`, a custom duration field accepting the shared
  `"<duration> [+<N>]"` form, and an optional reason field. On an already-snoozed bead
  the modal opens in "re-snooze" mode showing the current wake time, and offers a
  cancel-snooze choice.
- Keymap: add `beads_snooze` to `src/sase/ace/tui/keymaps/app_keymaps.py`,
  `metadata.py`, the Beads help bindings, and `src/sase/default_config.yml`'s keymap
  block. Bind `z` — it is unbound in the Beads pane and matches the `z` suffix in the
  indicator, keeping one mnemonic for "deferred" across the TUI. Verify against the
  current Beads-pane bindings before committing to it and pick another free key if `z`
  is taken.
- `next_bead_status` (`src/sase/ace/tui/actions/_artifacts_beads_common.py`) must
  **not** include `snoozed` in the blind status cycle — snoozing needs arguments. Add
  `Status.SNOOZED: Status.READY` so cycling _out_ of snoozed works and cancels the
  snooze through the proper mutation, and leave nothing cycling _into_ it.

### Telegram

`sase-telegram` renders gates generically through `adapter_for_kind`, so `BeadSnooze`
options become buttons once the adapter is registered, and the `snooze` option's
required feedback becomes a reply prompt. Verify rather than assume:

- open the plugin repository with `/sase_repo` and check `gate_flow.py` for any per-kind
  allowlist that needs the new kind added
- `bead_format.py` parses `sase bead` CLI output with regexes keyed on `[status]`;
  confirm the snoozed status and the new `Snooze` detail block survive that parsing, and
  extend the regexes and formatters if they do not
- confirm the `/bead` command surface can reach `sase bead snooze`

### Mobile

Check `src/sase/integrations/_mobile_notification_actions.py` and `mobile_gateway.py`
for kind or action allowlists that need `BeadSnooze`, and confirm
`crates/sase_core/src/notifications/mobile.rs` contract snapshots still pass with the
new notification `color` field and the new gate kind — update the snapshot fixtures
deliberately if they change.

### Tests

An ACE modal test per mode (fresh snooze, re-snooze, cancel), a keymap registration
test, the status-cycle change, and whatever Telegram/mobile tests the verification above
shows to be needed.

---

## Cross-surface verification and documentation

**Goal**: the whole feature is exercised end to end, both repositories pass their gates,
and the documentation debt is either paid or explicitly recorded.

- Walk the full lifecycle against a scratch project: create a task, mark it ready,
  confirm the `TaskTriage` gate, snooze it from the gate, confirm the triage gate is
  replaced by exactly one snoozed `bead_snooze` gate, confirm the row appears in the
  panel's Snoozed tab and nowhere else, confirm the indicator shows `<N>z` when nothing
  else is pending and hides that count as soon as anything else arrives, force the wake
  time forward, and exercise each of the three wake options.
- Separately exercise the +1 wake: snooze with `-p 2`, add one +1 (stays snoozed), add
  the second (promotes to ready with the preset note), and confirm the `bead_snooze`
  gate is canceled and a `TaskTriage` gate replaces it.
- Confirm the multi-tab regression is gone: a notification with two tags occupies one
  tab, and dismissing it removes exactly one tab.
- Inspect the indicator visually at one, three, and overflow tab counts, and in the
  snoozed-only state; hover for the tooltip. Check the ACE PNG snapshot suite
  (`just test-visual`) and accept intentional visual changes with
  `--sase-update-visual-snapshots` only where the change is genuinely intended.
- Run `just check-full` in this repository (the change touches broad surfaces, so the
  scoped lane is not sufficient) and the Rust crate's own checks in the core repository.
  Land the crate change and its binding bump before the Python callers depending on it.
- Update user documentation under `docs/` for the new status, the `sase bead snooze`
  command, the notification tabs, and the indicator's reading rules.
- **Do not edit any memory file.** `sase/memory/sase_beads.md` documents the status list
  and will be stale; record a `PROPOSED FOLLOW-UP:` note on this phase's bead describing
  the exact edit needed, and leave the decision to the owner.
