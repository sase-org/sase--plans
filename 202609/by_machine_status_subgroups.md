---
tier: tale
title: Status sub-groups under BY_MACHINE machine groups
goal: In the Agents-tab BY_MACHINE grouping mode, every machine group (here and each
  remote alias) nests its agents under priority-ordered status sub-group banners,
  with name-root/name-prefix grouping intact inside each bucket, all other grouping
  modes byte-for-byte unchanged, and BY_MACHINE still on the in-place patch fast paths.
size: medium
proposed_by: bbugyi200.athena.0iw
status: done
---

# Status Sub-Groups Under BY_MACHINE Machine Groups

## Goal

In the Agents-tab `BY_MACHINE` grouping mode, each machine group (`here` and every
remote `<machine>` alias) currently renders its agents as a flat name-root/name-prefix
layout directly under the machine banner. Change the hierarchy so every machine group is
sub-grouped **by status**, reusing the established status-bucket semantics from
`BY_STATUS` mode:

```text
here ━━━━━━━━━━━━━━━━━━━━━━━━━━━  7 agents · 3 running · 1 failed
  ▎ ▲ Stopped ─────────────────  1 agent · 1 awaiting
  │   sase.fix-flake
  ▎ ✗ Failed ──────────────────  1 agent · 1 failed
  │   sase.perf-audit
  ▎ ▶ Running ─────────────────  3 agents · 3 running
  │   standalone-agent
  │   ▸ epic-worker ───────────  2 agents · 2 running
  │   │   epic-worker.1
  │   │   epic-worker.2
  ▎ ✓ Done ────────────────────  2 agents
  │   ...
apollo ━━━━━━━━━━━━━━━━━━━━━━━━  2 agents · 2 running
  ▎ ▶ Running ─────────────────  2 agents · 2 running
      ...
```

New `BY_MACHINE` hierarchy: **machine (L0) → status bucket (L1) → name-root (L2) →
dotted name-prefix (L3)**. All other grouping modes are byte-for-byte unchanged.

## Design (decided — implement as specified)

1. **Reuse the existing L1-subgroup plumbing, don't add parallel machinery.** `BY_DATE`
   already threads an L1 subgroup through `GroupingKeys.date_subgroup`,
   `subgroup_indices` in the tree builder, and a subgroup sort component in
   `walk_order`. Rename that field to the mode-neutral `subgroup` and populate it under
   `BY_MACHINE` with the agent's status bucket. A second field plus a second
   banner-emission block was rejected as duplicate plumbing.
2. **Status buckets and their order come from the existing catalog.** Use
   `status_bucket_for()` (which wraps the shared `agent_status_bucket()` in
   `src/sase/agent/status_buckets.py`) for bucketing, and the existing priority order
   `Stopped, Failed, Running, Queued, Waiting, Done, Starting` for sorting.
   `bucket_sort_index()` in `src/sase/ace/tui/models/agent_groups/_buckets.py` already
   selects `_STATUS_BUCKETS` for every non-`BY_DATE` mode, so
   `bucket_sort_index(GroupingMode.BY_MACHINE, subgroup)` works unmodified.
3. **Every status subgroup emits a banner.** Unlike `BY_DATE`'s synthetic `(no time)`
   label there is no synthetic status bucket — `status_bucket_for()` always returns a
   real bucket — so the emit predicate for `BY_MACHINE` is simply "subgroup is
   non-empty". No singleton suppression: this matches `BY_DATE`'s always-emit rule for
   real labels, keeps fold keys stable, and makes the layout predictable as agents
   change status.
4. **Name grouping nests within (machine, status).** A family with one running and one
   done member must appear once under each status bucket rather than as a single group
   spanning buckets. This falls out of extending the grouping "parent key" with the
   subgroup (details below).
5. **Stay on the fast in-place patch paths.** `BY_MACHINE` participates in
   `try_remove_rows` and the panel row-patch path (unlike `BY_STATUS`); keep it there.
   Extend `machine_grouping_signature()` with the status bucket so the per-row guard
   falls back to a rebuild exactly when a row would move between status subgroups, and a
   same-bucket badge-only change still patches in place.
6. **Visual register (the "beautiful" part).** Status subgroup banners use the existing
   middle-tier register (the `▎` bar + light `─` rule that `BY_DATE` hour subgroups and
   STANDARD Patch banners use), with the status glyph from `AGENT_STATUS_BUCKET_GLYPHS`
   between the bar and the label (e.g. `▎ ▶ Running`), echoing the glyphs `BY_STATUS` L0
   banners already use. The `Queued` bucket keeps its `QUEUED_STATUS_COLOR` accent on
   the glyph, mirroring the existing `BY_STATUS` L0 special case. Status banners
   contribute a tier-guide gutter segment so agent rows visibly indent beneath them.

## Non-goals

- No change to `STANDARD`, `BY_DATE`, or `BY_STATUS` mode output (must stay
  byte-for-byte identical — see "Ordering neutrality" below).
- No change to the Patches-tab grouping models (`src/sase/ace/tui/models/patch_groups/`,
  `src/sase/ace/tui/models/changespec_groups/`) even though they contain a sibling
  `date_subgroup` concept; the rename is scoped to `agent_groups` only.
- No new grouping mode and no change to the `o`-key cycle order.
- No Rust-core work: this is presentation-only Textual tree/render state, which stays in
  Python per the core-boundary rule.

## Implementation steps

All paths are repo-relative.

### 1. `src/sase/ace/tui/models/agent_groups/_keys.py`

- Rename the `GroupingKeys.date_subgroup` field to `subgroup` and update every
  in-package and test reference. Keep the BY_DATE-specific helper names
  (`date_subgroup_bucket_for`, `date_subgroup_sort_key`, `NO_HOUR_LABEL`) unchanged —
  only the dataclass field becomes mode-neutral.
- In `grouping_keys_for()`, populate `subgroup` for `BY_MACHINE` as
  `status_bucket_for(target)` (note: `target`, the presentation anchor, so a family/clan
  subtree stays intact under its root's bucket, exactly like `BY_DATE` anchors).
  `BY_DATE` keeps `date_subgroup_bucket_for(target, l0)`; other modes keep `""`.
  `anchor` stays `BY_DATE`-only.
- **Parent-key nesting:** in `walk_order()`, change the non-patch parent key from
  `(k.project, "")` to `(k.project, k.subgroup)`. This single change scopes
  `root_counts`, `prefix_counts`, and the standalone-vs-subgroup partition machinery to
  each (machine, status) pair. It is neutral for every existing mode: `subgroup` is `""`
  outside `BY_DATE`, and `BY_DATE` suppresses name-root/name-prefix grouping entirely.
- **Subgroup sort placement:** generalize `_date_subgroup_sort_key` to a mode-aware
  `_subgroup_sort_key(mode, l0, subgroup, anchor)`:
  - `""` subgroup → neutral `(0, 0)` (unchanged);
  - `BY_DATE` → existing `date_subgroup_sort_key` behavior (unchanged);
  - `BY_MACHINE` → `(0, bucket_sort_index(mode, subgroup))` so buckets sort in priority
    order with unknown labels last. Move this component **before** `status_sort_keys` in
    the `sorted(...)` key tuple so rows group by status bucket first and the
    recency/partition machinery orders rows _within_ a bucket.
  - _Ordering neutrality argument (verify while implementing):_ under `BY_DATE` every
    `status_sort_keys` entry is the neutral tuple, under `STANDARD`/`BY_STATUS` every
    subgroup component is the neutral `(0, 0)`, and under current `BY_MACHINE` the
    subgroup was always `""`. Therefore swapping the relative order of these two
    components cannot change any existing mode's output.
- **Row-patch guard:** extend `machine_grouping_signature()` to include
  `status_bucket_for(agent)` (update the return type annotation to the 6-tuple) and
  document it the way `status_grouping_signature()` is documented: the signature
  captures everything that decides bucket, subgroup, and position, so a status-bucket
  move forces a rebuild while badge-only changes still patch in place.

### 2. `src/sase/ace/tui/models/agent_groups/_tree.py`

- Generalize `_should_emit_date_subgroup_banner` into
  `_should_emit_subgroup_banner(mode, subgroup, count)`: `BY_DATE` keeps the current
  rule (real labels always; `NO_HOUR_LABEL` needs ≥2), `BY_MACHINE` emits whenever
  `subgroup` is non-empty.
- In `build_agent_tree()`:
  - Collect `subgroup_indices` and run the subgroup-banner block for
    `mode in {BY_DATE, BY_MACHINE}` instead of `BY_DATE` only. The subgroup banner keeps
    `level=1` and `group_key=(l0, subgroup)`.
  - For `BY_MACHINE`, set the subgroup banner's `has_child_groups` to whether any
    name-root group with ≥2 members exists under parent `(k.project, k.subgroup)` (this
    drives both the middle-tier register and the tier gutter). `BY_DATE` keeps
    `has_child_groups=False`.
  - For `BY_MACHINE`, set `parent_key = (k.project, k.subgroup)` and `deep_level = 2`,
    so name-root banners render at L2 with fold keys `(machine, status, root)` and
    prefix banners at L3 with `(machine, status, root, prefix)`. Other non-patch modes
    keep `parent_key = (k.project,)` / `deep_level = 1`.
  - Confirm the collapse cascade: collapsing a machine hides its status banners;
    collapsing a status subgroup hides its roots/agents; the existing `cur_subgroup` /
    `cur_root` / `cur_prefix` reset logic already resets on each L0 transition.
- Mirror the same parent-key/subgroup changes in `enumerate_group_keys()` so
  fold-all/expand-all and jump-hint enumeration agree with the rendered tree (subgroup
  key emitted after its L0 key and before its name-root keys).
- `banner_label_for_group_key()` needs no logic change: the status label is the key
  suffix. Status labels pass through `humanize_cl_name()` like `BY_DATE` time labels do;
  Title-case bucket names do not collide with canonical project/Patch keys, so this is a
  no-op (sanity-check in a test).
- Update the `agent_groups/__init__.py` module docstring (the `BY_MACHINE` paragraph)
  and the `GroupingMode` docstring in `_buckets.py` to describe the machine → status →
  name-root hierarchy.

### 3. Rendering

- `src/sase/ace/tui/widgets/_agent_list_render_banner.py` (`format_banner_option`):
  - Add `(group.level == 1 and mode is GroupingMode.BY_MACHINE)` to the
    `is_middle_tier_banner` predicate so status subgroup banners always use the `▎` +
    light-rule register even when they have no name-root children.
  - Inside the middle-tier branch, when the banner is a `BY_MACHINE` L1 status subgroup
    and its label is in `_STATUS_BUCKET_GLYPHS`, render the glyph between the bar and
    the label (`▎ ▶ Running`), and give the `Queued` bucket's glyph the
    `QUEUED_STATUS_COLOR` accent, mirroring the existing `BY_STATUS` L0 special case.
    Account for the extra glyph cells in the rule-padding width math.
  - Update the docstring's per-mode glyph/color inventory.
  - The memoized wrapper needs no cache-key change: `banner_render_key` already keys on
    group key, mode, and member agents, and the group key now includes the status label.
- `src/sase/ace/tui/widgets/_agent_list_build.py` (`compute_tier_styles`):
  - In `descendant_style_for`, return `_PATCH_BANNER_RULE_STYLE` for
    `group.level == 1 and mode is GroupingMode.BY_MACHINE` (all status labels are real;
    there is no synthetic bucket to exclude). Update the docstring.
- Banner summary chips are intentionally unchanged: a `Running` subgroup showing
  `3 agents · 3 running` matches what `BY_STATUS` L0 banners already show today.

### 4. Fast-path integration

- `src/sase/ace/tui/actions/agents/_display_panel_patches.py`: no logic change —
  `_machine_row_patch_is_safe` picks up the strengthened signature automatically. Update
  its docstring to mention the status bucket.
- `try_remove_rows` in `_agent_list_build.py` stays enabled for `BY_MACHINE`: in-place
  removal never reorders surviving rows, and a temporarily stale or empty status banner
  heals on the next full refresh exactly like stale name-root banners do today (the
  function's docstring already documents this contract). No change needed; do not drop
  `BY_MACHINE` from the fast paths.
- Fold-state snapshots persisted from the old key shape (e.g. `("here", "myroot")`)
  simply never match the new keys and are ignored; groups render expanded once and
  re-collapse state self-heals. No migration.

### 5. Help modal

- `src/sase/ace/tui/modals/help_modal/agents_bindings.py` (Grouping section): update the
  `by machine` row description to mention the status sub-grouping, e.g.
  `("by machine", "here + remotes, status subgroups")`. Respect the help modal box rules
  (57-char box width, description ≤ 32 chars). The status glyph legend rows directly
  below already cover the glyphs.

### 6. Tests

Update for the field rename and new hierarchy, and add coverage:

- `tests/ace/tui/models/test_agent_groups_grouping_mode_keys.py`: `BY_MACHINE`
  `GroupingKeys` now carry the status subgroup; the per-mode expected-tuple table and
  the `enumerate_group_keys` expectation (`[("here",)]` → machine + status keys) change
  accordingly. Add a case where one machine holds agents in ≥2 buckets and assert key
  nesting `(machine, status, root)`.
- `tests/ace/tui/models/test_agent_groups_grouping_mode_tree_status.py`: update the
  existing `BY_MACHINE` tree tests and add:
  - buckets render in priority order within each machine, after `here`-first /
    alias-alphabetical machine order;
  - a family with members in two buckets splits into one name-root group per bucket (no
    group spans buckets);
  - standalone-before-subgroups partitioning holds within a bucket;
  - collapsing machine / status / name-root each suppresses exactly its descendants;
    `banner_label` for a status key returns the plain bucket name.
- `tests/ace/tui/models/test_agent_groups_grouping_mode_subgroups.py` and
  `tests/ace/tui/models/test_agent_groups_folds.py`: apply the field rename; extend fold
  tests with the new `BY_MACHINE` key shapes if they enumerate `BY_MACHINE` keys.
- `tests/ace/tui/widgets/test_agent_list_grouping_gutter.py`: status subgroup banners
  contribute a gutter segment; agent rows under `here → Running` carry machine + status
  tiers.
- `tests/ace/tui/widgets/test_agent_list_grouping_buckets.py` (or the closest
  banner-rendering test): `BY_MACHINE` L1 banner renders the middle-tier bar, status
  glyph, and Queued color accent.
- `tests/ace/tui/test_agent_display_diff.py`: `machine_grouping_signature` — a
  status-bucket change fails the in-place guard (rebuild fallback) while a same-bucket,
  badge-only change still patches in place.
- Ordering-neutrality regression: assert `STANDARD` / `BY_DATE` / `BY_STATUS` trees for
  a representative mixed fixture are unchanged by this change (existing mode tests
  largely cover this; keep them green without editing their expectations except for the
  mechanical field rename).

## Verification

- Run the project's standard verification (`just check`) and make it pass; it is the
  agent-default gate per the two-speed verification decision.
- Perf sanity: this feature adds no new refresh paths, keeps `BY_MACHINE` on the
  in-place patch/remove fast paths, and only strengthens the existing signature guard;
  banner rendering stays behind the existing `AgentRenderCache`. If touching render
  paths triggers doubt, compare `SASE_TUI_PERF=1` j/k latency on the Agents tab
  before/after.

## Risks / edge cases

- **Sort-tuple regression risk** is the main hazard: the subgroup component moves
  earlier in `walk_order`'s key. The neutrality argument above must hold; the existing
  mode test suites are the guard.
- An agent whose `status_bucket_for` value changes between refreshes moves between
  subgroups; the strengthened signature makes the row-patch guard force a rebuild in
  that case — verify the fallback trace fires (covered by the display-diff test).
- Machines whose agents all share one bucket still show a single status banner (by
  design — see decision 3).
