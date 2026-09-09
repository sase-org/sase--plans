---
tier: tale
title: Make the updates badge signal a sase-core rebuild
goal: "The ACE top-bar updates badge tells the user not just how many updates are
  pending but how expensive they are, calling out a pending sase-core update — which
  forces a slow Rust recompile — with a distinct glyph, color, and tooltip. The badge
  count agrees with the Updates panel instead of silently undercounting.

  "
create_time: 2026-09-09 19:53:27
status: wip
---

# Plan: Make the updates badge signal a sase-core rebuild

## Product context

`sase update` is not a uniform-cost operation. Updating `sase` or a plugin is a fast
wheel swap. Updating `sase-core` recompiles the Rust crates and takes substantially
longer. That cost difference is the single most decision-relevant fact about a pending
update: it determines whether the user pulls the trigger now or waits until they are
between tasks.

Today the badge cannot express it. It renders `↑ N` and nothing else, so a cheap plugin
bump and an expensive Rust rebuild look identical. The user hits `,U` blind and
discovers the cost only once the compile starts.

Worse, in the reported case the badge read `↑ 1` while the Updates panel simultaneously
listed **two** outdated components (`sase` _and_ `sase-core`). The pending core update
was not merely undecorated — it was absent from the count entirely.

## Root cause

Two defects compound, and their order matters.

**D1 — role is discarded at the widget boundary.** `UpdateStatus.components` carries a
`role` field (`"host" | "core" | "plugin"`, `src/sase/updates/status.py:26,34`). But
`_refresh_updates_indicator` collapses that to `status.count`
(`src/sase/ace/tui/actions/update_toast.py:310`), and the widget's only mutation entry
point is `set_available(count: int)`
(`src/sase/ace/tui/widgets/updates_indicator.py:25`). An `int` is the entire contract.
The badge structurally cannot distinguish a Rust rebuild from a plugin bump, no matter
how it is styled.

**D2 — the badge and the panel never reconcile, and the badge can only shrink.** The
Updates panel computes fresh core state on open (`detect_dev_latest`, which performs a
git fetch) and correctly showed both components. The badge is driven only from the
cached snapshot, and `revalidate_update_status` (`src/sase/updates/cache.py:117`)
_filters_ the cached component tuple — it drops entries that no longer look outdated and
can never add a newly-outdated one. Nothing in `src/sase/ace/tui/modals/` references the
badge or the snapshot cache (verified by grep), so opening the panel and reading "2
updates" does not correct `↑ 1`. Only a full recompute can grow the count.

D2 is load-bearing for this feature, not a separate cleanup: **if the count omits
`sase-core`, no amount of styling will ever fire the core treatment.** D1 and D2 must be
fixed together or the feature is decorative and dead.

The implementing agent should confirm the recompute cadence as part of D2 and fix it if
it is the stale-count culprit. Note the suspicious pairing of
`_AUTOMATIC_UPDATE_CHECK_INTERVAL_SECONDS = 600.0` (the timer fires every 10 minutes,
`update_toast.py:51`) against `_DEFAULT_RECOMPUTE_INTERVAL_SECONDS = 3600.0` (the
default recompute gate, `update_toast.py:53-54`). If the 10-minute tick is gated behind
a 60-minute recompute interval, the badge can lag reality by up to an hour — which fully
explains the screenshot. Establish the actual behavior before changing the constants; do
not tune numbers blind.

One thing already works in our favor: `role` is **already persisted** in the snapshot
(`cache.py:88`) and validated on read (`cache.py:218`) under the current
`SCHEMA_VERSION = 2`. The core signal is therefore available from cache, offline, across
restarts, with **no new data source, no new network call, and no schema bump**. This is
a plumbing fix, not a data-acquisition one.

## Design

### The idea: the badge encodes cost, not just count

The badge's job is upgraded from _"how many updates"_ to _"how many, and how heavy"_. It
gets exactly two states, because the user only makes one decision (now vs. later).

| State   | Condition                           | Render  | Background                    |
| ------- | ----------------------------------- | ------- | ----------------------------- |
| Routine | updates pending, no `core` role     | `↑ N`   | `#AF87FF` (purple, unchanged) |
| Rebuild | any component with `role == "core"` | `↑ N ⚙` | amber (e.g. `#FFAF5F`)        |
| Hidden  | no updates                          | empty   | —                             |

Deliberate choices:

- **Routine is byte-for-byte unchanged.** The common case must not regress; existing
  muscle memory and the existing purple accent survive untouched. The new treatment is a
  strict addition that appears only when it has something to say. A badge that always
  shouts teaches the user to ignore it.
- **Two states, not per-role counts.** `↑ 1 ⚙ 1` is more information and a worse badge.
  The user's decision is binary; the badge should be too. The panel is where breakdowns
  belong, and it already has them.
- **Redundant encoding.** The signal is carried three times — glyph, color, and tooltip
  — so it survives colorblindness, a monochrome terminal, and a font that mangles the
  glyph. Color alone would be pretty and unreliable; color is the _accent_, the glyph is
  the _message_.
- **Amber, not red.** A pending rebuild is a cost, not an error. Amber reads as
  "heavier, plan for it" without the alarm semantics red would import. Purple→amber also
  survives the common (deuteran/protan) confusion lines, since it separates on the
  blue–yellow axis.
- **Two cells of growth.** The top bar is crowded. `↑ N ⚙` costs two cells over `↑ N`,
  only in the state that warrants it.
- **`⚙` means build/compile**, which is what actually costs the time. It is the honest
  glyph: the cost is the recompile, not Rust as a language.

The tooltip states the cost in words rather than making the user infer it:

> `3 updates available — includes sase-core (Rust rebuild, expect a slower update). Click to open Updates, or press ,U to update sase, core & plugins.`

The routine tooltip keeps its current wording.

### Glyph risk and the fallback

`⚙` is U+2699, which some fonts resolve to an emoji presentation with a double-width
cell — that would shear the top bar's alignment. This is the one real aesthetic risk in
the plan and it must be **proven, not assumed**: the PNG snapshot below is the
acceptance gate.

The visual fixtures pin Fira Code and fontconfig for determinism, so the snapshot is a
faithful test. If `⚙` renders as double-width or tofu, fall back in this order and
record which was chosen and why: `⚡` (already used in the Agents tab and known to
render), then `*`. Do not ship a glyph the snapshot has not confirmed.

### Where the policy lives

Add a single derived predicate to the model, alongside the existing `has_updates` /
`count`:

```python
# src/sase/updates/status.py
@property
def has_core_update(self) -> bool:
    """Return whether a pending update requires a Rust core rebuild."""
    return any(component.role == "core" for component in self.components)
```

One definition, consumed by both refresh paths, so the periodic path and the
revalidation path cannot drift apart. This is deliberately _not_ pushed to the sibling
Rust core repo: it is a trivial predicate over a model that is Python-only by design —
`src/sase/updates/` imports no `sase_core_rs`, and `src/sase/uv_tool/__init__.py:10`
documents the update path as having "no `sase-core` (Rust) involvement". Keep it here;
do not open the boundary for a one-line `any()`.

Then widen the widget contract from `int` to count-plus-role:

- `UpdatesAvailableIndicator.set_available(count: int, *, core: bool = False)` — keep
  the unchanged-state early-return by comparing both fields, not just the count.
- Rename `_set_updates_indicator_count` → `_set_updates_indicator_state` and thread
  `core` through both call sites (`update_toast.py:294,302,310`). Both already hold the
  full `UpdateStatus`, so nothing new needs fetching. The `config.indicator`-off path
  passes `(0, core=False)`.
- Keep the amber constant next to `_UPDATES_ACCENT` in the widget module. **Do not**
  refactor the ~10 hardcoded `#AF87FF` sites into a shared palette — real, but out of
  scope and it would bury this change.

### Closing the reconciliation gap (D2)

- When the Updates panel completes its fresh core/plugin check, write the shared
  snapshot and refresh the badge. This heals the disagreement at exactly the moment the
  user is positioned to notice it.
- Fix the recompute cadence so the badge converges **without** requiring the user to
  open the panel — the panel path is a safety net, not the mechanism.
- Leave `revalidate_update_status` drop-only (it is correct: it is the cheap, no-network
  path and must not invent components), but document the asymmetry at the definition so
  the next reader does not mistake it for a general-purpose refresh.

## Testing

- **Unit — model**: `has_core_update` across empty / host-only / plugin-only /
  core-present / mixed component sets (`tests/test_update_status.py`).
- **Unit — widget**: extend `tests/test_updates_indicator.py`. It pins the exact
  rendered string at line 15 (`" ↑ 3 "`) and the accent at line 16, so both **will**
  fail and must be updated deliberately, not blindly. Add coverage for the core render,
  the amber accent, both tooltips, and the `core`-flip early-return.
- **Unit — plumbing**: `tests/ace/tui/test_update_toast.py` — assert `core` survives
  both the periodic path and the cached-revalidation path, and that `indicator: false`
  still clears to `(0, core=False)`.
- **Regression — reconciliation**: a panel fresh-compute that finds `sase-core` must
  raise the badge count and flip it to the core state. This is the exact `↑ 1` bug; it
  gets a named test.
- **Visual — PNG snapshot**: there is currently **no PNG coverage of the header badge at
  all** (every existing badge test is string- or placement-level; the visual suite
  covers only the Updates panel and toasts). Add snapshots for both routine and core
  states under `tests/ace/tui/visual/`. This is what locks in "beautiful" and proves the
  glyph's cell width. Accept new goldens only via `--sase-update-visual-snapshots`.
- **Placement**: re-run `tests/ace/tui/test_top_bar_order.py` and confirm the two extra
  cells do not disturb the top-bar order or overflow a narrow terminal.
- Run `just install` first (ephemeral workspace), then `just check`.

## Risks

- **Glyph width shearing the top bar** — the headline risk. Gated by the new PNG
  snapshot and the documented fallback ladder.
- **Pinned-string test churn** — `test_updates_indicator.py` asserts the exact badge
  string today. Expected and desired; the point is that the render is deliberate.
- **Cadence tuning** — changing the recompute interval trades freshness against
  git-fetch cost. Diagnose the 600s-vs-3600s interaction before touching constants; do
  not "fix" it by simply making checks more frequent.
- **Scope drift into the palette** — `#AF87FF` is duplicated widely and
  `center_tab_accent` exists as a partial canonical source. Explicitly out of scope.

## Non-goals

- No global TUI palette refactor.
- No change to what counts as an available update, or to the update mechanism itself.
- No new network calls, no snapshot schema bump (`role` is already persisted at schema
  v2).
- No changes to the sibling Rust core repo.
- No new user-facing config key; the existing `indicator` toggle continues to govern the
  badge.
