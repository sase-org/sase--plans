---
tier: tale
title: Show a model-usage statistics strip in the Alias History panel
goal:
  The Launch Control Alias History panel ends with a compact, provider-themed usage
  strip that ranks which models the currently-shown runs actually used, marks
  configured-but-unused members, and recomputes off-thread on every load, Ctrl+K
  load-more, refresh, and hidden toggle.
size: medium
proposed_by: bbugyi200.athena.06n
create_time: 2026-08-18 15:17:39
status: wip
---

# Plan: Model-usage statistics strip for Alias History

## 1. Outcome

Press `H` on an alias, bucket, or alias-backed launch setting. Below the per-run detail
strip, above the key footer, the panel gains a fifth region that answers "which models
did this alias actually use?" for exactly the runs currently shown:

```
──────────────────────────────────────────────────────────────────────────────
Model usage · 40 runs · 2 of 3 members used
  GROK(grok-4.6) @ xhigh    █████████████████████▍       33  82%
  CLAUDE(sonnet) @ xhigh    ████▌                         7  18%  ✗2
  CLAUDE(opus)   @ xhigh    ░░░░░░░░░░░░░░░░░░░░░░░░░░    0   0%  unused

  enter=Prompt  y=Copy  ctrl+k=Load more  r=Refresh  .=Hidden (excluded)
```

Bars are share-of-total, colored with the same provider palette already used by the
`PROVIDER(model)` badges in the rows above, so the strip reads as one system with the
rest of the panel. `Ctrl+K` (load more), `r` (refresh), and `.` (hidden toggle) all
reload the view and therefore repaint the strip with the new window; `j`/`k` never
recompute it.

## 2. Design decisions

These are decisions this plan makes; the implementer should follow them rather than
re-litigate them, and should flag (not silently change) any that measurement disproves.

1. **Statistics describe the shown window, never the archive.** The user asked for stats
   over "the historical agent runs that are currently shown". The summary is derived
   purely from the loaded `AliasHistoryView`, so it is correct by construction after
   every re-query. The title line already carries `160 recorded · 40 shown`, so the
   usage header says `40 runs` and does not repeat the recorded total.

2. **Group by `(provider, model)`, not by effort.** The question is "which model", so
   effort must not fragment a model into several rows. Effort is still shown in the row
   label: when every counted run for that model shares one effort, render `@ <effort>`;
   when they differ, render `@ mixed` in a dim style. A model with no recorded effort
   renders no effort suffix.

3. **The "pool" includes configured members with zero runs.** A pool member that never
   ran is the most interesting cell in the table (a weight that never wins, a member
   added yesterday, a provider that is always disabled). The entry request therefore
   snapshots the alias's selector members at `H`-press time, exactly like it already
   snapshots the effective provider/model/effort, and unused members render as a real
   `0  0%  unused` row rather than being invisible.

4. **Observed-but-not-configured models are kept and tagged `off-pool`.** History
   outlives config: an alias that used to point at `claude/opus` still has `opus` runs.
   Dropping them would make the percentages lie. They are counted, ranked normally, and
   carry a dim `off-pool` tag when a pool is known.

5. **Runs with no recorded model become one `unrecorded` row**, never a silent drop, so
   the shares always sum over every shown run.

6. **Dedupe by `artifact_dir` before counting.** In a bucket, one run can legitimately
   appear under two member aliases (this is exactly why `alias_history_run_key`
   namespaces by alias). Counting it twice would overstate a model. When dedupe actually
   removed rows, the header appends a dim `(deduped)` so the count difference against
   the title's `shown` is explained rather than mysterious.

7. **Percentages are apportioned by largest remainder so the displayed integers sum to
   exactly 100.** Three even models must not read `33% 33% 33%`. Unused members are a
   true `0%` and take no share.

8. **Bar color is the provider's model color, not a per-row categorical color.** Two
   Claude models share one hue on purpose: provider identity is the established visual
   language of this panel, and the badge text already disambiguates the model. Bars use
   eighth-block partials (`▏▎▍▌▋▊▉`) over a dim `░` track, matching the Statistics pane
   and telemetry bar conventions.

9. **Bounded height with an honest overflow row.** At most four rows render. When more
   models exist, three render and a fourth `+N more` row carries the combined count and
   the remaining percent, so nothing is silently truncated and the shares still total
   100%.

10. **No new keybinding, no toggle, no hidden state.** The strip is always present. An
    expand/collapse key would add footer noise and a second state machine for a region
    that is five lines tall at most. (See §8 for what this defers.)

11. **The summary is computed in the load worker, not on the UI thread.** `Ctrl+K`
    doubles the limit without bound, so the counted set can grow large; per
    `sase/memory/tui_perf.md` rule 1 the aggregation rides along with the load that is
    already off-thread, and the UI thread only paints a cached, immutable summary.
    `j`/`k` highlight moves must not recompute anything (rule 7).

12. **Uniform behavior for every entry kind.** A single alias, a bucket (union of its
    members' targets), a selector alias, and an alias-backed launch setting all produce
    the same strip. A one-model alias simply renders one 100% row; conditional
    appearance would be less predictable than a boring uniform region.

## 3. Boundary and policy decisions

- **Rust core boundary.** No `../sase-core` change. The wire query already returns every
  field this needs; the new work is aggregation over an already-returned result set, and
  its frontend-neutral home is `sase/llm_provider/alias_history*.py` — the module family
  whose stated purpose is being "the seam between the core query and any surface that
  renders it", and which already owns the analogous `done/failed/running` rollup
  (`_rollup_for_runs`). A CLI or web frontend gets identical numbers by calling the same
  function. Do not move status/usage aggregation into Rust as part of this tale.
- **Feature flag.** None. Per `sase/memory/sase_flags.md`, a flag is for behavior that
  reaches users _before it is ready_ (`beta`/`wip`), for a deprecation (`sunset`), or a
  permanent operational switch (`ops`). This lands complete and unconditional inside one
  modal, and is not a choice users make forever. Do not run `sase flag new`.
- **Config.** No new config key. The strip's size is a layout constant, not a user
  preference; `llm_provider.model_alias_history_limit` (default `10`) already controls
  how many runs the window holds.

## 4. Implementation

### 4.1 New frontend-neutral summarizer

Create `src/sase/llm_provider/alias_history_usage.py` (pure, no Textual, no IO):

- `AliasHistoryPoolMember` — frozen slots dataclass: `provider: str | None`,
  `model: str`, `effort: str | None = None`, `weight: int = 1`,
  `available: bool = True`. This is the snapshot of one configured selector member (or
  of an alias's single effective target).
- `AliasHistoryModelUsage` — frozen slots dataclass for one rendered row:
  `provider: str | None`, `model: str | None`, `effort_label: str | None` (`None`, a
  concrete effort, or the `mixed` sentinel — expose it as `effort_is_mixed: bool` plus
  `effort: str | None` so rendering does not string-match), `count: int`,
  `share: float`, `share_percent: int`, `done: int`, `failed: int`, `running: int`,
  `in_pool: bool`, `is_unrecorded: bool`.
- `AliasHistoryUsageSummary` — frozen slots dataclass: `rows: tuple[...]`,
  `counted_runs: int`, `duplicate_runs: int`, `pool_total: int`, `pool_used: int`.
- `summarize_alias_history_usage(view: AliasHistoryView, *, pool: Sequence[AliasHistoryPoolMember] = ()) -> AliasHistoryUsageSummary`

Behavior:

- Iterate `view.groups[*].runs`, skipping any `artifact_dir` already seen and
  incrementing `duplicate_runs` for each skip.
- Key each run on `(provider.strip().casefold() or None, model.strip().casefold())`; a
  run with no model goes to the single `is_unrecorded=True` bucket.
- Pool matching: a member matches an observed key when both providers are known and both
  provider and model match casefolded; when either side's provider is unknown, match on
  model alone. Document this fallback in the function docstring — it is what keeps a
  bare `sonnet` pool member from rendering as a phantom unused row next to its own runs.
- Ordering: counted rows first by `(-count, is_unrecorded, label_casefold)`, then
  zero-count pool members in configured pool order.
- `share = count / counted_runs` (0.0 when nothing counted); `share_percent` via largest
  remainder over counted rows so the shown integers total 100 (or 0 when nothing
  counted).
- Display casing: rows carry the first observed non-normalized `provider`/`model`
  spelling for a counted row, and the configured spelling for a zero-count member, so
  the badge matches the run rows above verbatim.
- `pool_total = len(pool)`, `pool_used` = pool members with `count > 0`.

### 4.2 Capture the pool at `H`-press time

- `src/sase/llm_provider/alias_view.py`: rename `_selector_member_provider_model_effort`
  to public `selector_member_provider_model_effort`, keep its in-file call site, and add
  it to the module's exports. Its new non-test consumer is the Launch Control history
  builder below (satisfies `sase/memory/symvision.md`: a public symbol needs a real
  non-test consumer).
- `src/sase/ace/tui/modals/alias_history_state.py`: add
  `pool: tuple[AliasHistoryPoolMember, ...] = ()` to `AliasHistoryEntryRequest`
  (defaulted, so every existing construction site and test keeps working).
- `src/sase/ace/tui/modals/models_panel_history.py`: add a private
  `_pool_members_for(...)` helper and use it in all three builders:
  - `_alias_history_request(AliasView)` — `view.selector_members` when present (both
    `round_robin` and `fallback` modes), otherwise a single member built from
    `view.provider` / `view.model` / `view.effort`.
  - `_bucket_history_request(BucketView)` — the same derivation over every
    `bucket.members`, concatenated and deduped on `(provider, model, effort)` with
    stable first-seen order.
  - `_launch_setting_history_request(LaunchModelSettingRow)` —
    `row.snapshot.selector_members` when present, otherwise the snapshot's
    provider/model/effort.

### 4.3 Rendering

Create `src/sase/ace/tui/modals/alias_history_usage_rendering.py` (pure; `rich.text`
only, mirroring `alias_history_rendering.py`):

- `alias_history_usage_text(summary: AliasHistoryUsageSummary | None, *, error: str | None = None) -> Text`
  - `summary is None` → one dim italic line `Model usage · loading…` (or the error
    text), so the region never changes height between load states.
  - `summary.counted_runs == 0` → one dim italic line
    `Model usage · no runs in this window`.
  - Otherwise: a header line then at most four rows.
- Header: `Model usage` in the panel's existing label style (`bold #87D7FF`), dim `·`
  separators, `<n> runs`, `(deduped)` in dim when `duplicate_runs`, and
  `<used> of <total> members used` only when `pool_total > 1`.
- Row: two-space indent; the `provider_model_text(provider, model, effort)` badge padded
  to the widest badge among the rendered rows (cap at 38 cells, using `Text.cell_len`
  and a trailing space run, the same technique `models_panel_rendering_rows` uses for
  its provider/model column); a 26-cell bar; right-aligned count (width 4) and percent
  (width 4); then optional `✗<n>` / `▶<n>` chips reusing the panel's existing failed and
  running colors; then a dim `unused` or `off-pool` tag when applicable.
- Bar: `math.floor(share * 26 * 8)` eighths → full `█` cells plus one partial glyph from
  `("", "▏", "▎", "▍", "▌", "▋", "▊", "▉")`, filled in `provider_bar_style(provider)`,
  remainder as `░` in a dim track style. A non-zero share always renders at least one
  partial cell so a 1% row is visible.
- Overflow row (when `len(rows) > 4`): render rows 1–3, then a fourth row labeled
  `+<n> more` in dim, with the summed count, `100 - sum(shown percents)`, and a dim
  track-colored bar.
- `src/sase/ace/tui/provider_styles.py`: add public
  `provider_bar_style(provider: str | None) -> str` returning
  `_provider_style_for(provider).model_style` (the un-bolded model hue).

### 4.4 Modal wiring

`src/sase/ace/tui/modals/alias_history_modal.py`:

- `compose()`: yield `Static(self._usage_text(), id="alias-history-usage")` between
  `#alias-history-detail` and `#alias-history-footer`.
- `__init__`: `self._usage: AliasHistoryUsageSummary | None = None`.
- `_start_load()`: the worker task returns `(view, summary, error)`; the summary is
  computed inside the thread with
  `summarize_alias_history_usage(view, pool=self._entry.pool)` (the entry is immutable,
  so the closure reads no UI state). Update the worker's type annotations accordingly.
- `_on_load_worker()`: store `self._usage` on success; clear it to `None` on error so
  the strip shows the loading/error line instead of stale numbers.
- `_update_context()`: update `#alias-history-usage` alongside title/footer/detail, so
  every existing re-query path repaints it and no new path is introduced (tui_perf rule
  5).
- `on_option_list_option_highlighted()` stays untouched: highlight moves repaint only
  the detail strip.

### 4.5 Layout budget (`src/sase/ace/tui/styles.tcss`)

The modal must not get taller — `#alias-history-container` is already capped at
`max-height: 42` on 40-row terminals, so new lines must be paid for:

- Add
  `#alias-history-usage { height: auto; max-height: 5; border-top: solid $secondary; padding-top: 1; margin-top: 1; }`
  (5 = header + 4 rows).
- `#alias-history-footer`: drop `border-top` and `padding-top`, keep `margin-top: 1` —
  the usage strip's rule now separates both regions, so the two share one rule instead
  of stacking two.
- Reclaim the rest: `#alias-history-list` `max-height: 18` → `14`, and
  `#alias-history-detail` `max-height: 14` → `13`.
- Worst-case arithmetic: `+2` chrome `+5` content `−2` footer chrome `−4` list `−1`
  detail = `0`. The list scrolls, so taking lines from it degrades more gracefully than
  clipping the detail strip.
- The implementer must verify this by measurement (§6) at 120×40 and at a small
  terminal, and may re-divide the reclaim between list and detail — the invariant to
  hold is _the panel's worst-case height does not grow and the footer stays visible_.

### 4.6 Docs

`docs/ace.md`, "Alias History" section: describe the usage strip after the detail-strip
paragraph — what it counts (the shown window, deduped), that bar color follows the
provider palette, the `unused` / `off-pool` / `unrecorded` / `mixed` vocabulary, the
`+N more` overflow, and that `Ctrl+K` / `r` / `.` all update it while `j`/`k` do not. No
key-table change (no new binding). No `docs/llms.md` or `default_config.yml` change (no
new config).

## 5. Tests

- `tests/llm_provider/test_alias_history_usage.py` (new): dedupe across bucket groups;
  rank order and tie-break; zero-count pool members present and last; `off-pool`
  classification; `unrecorded` bucket; percent apportionment summing to exactly 100 for
  a 3-way even split and for a 7-model split; effort single vs `mixed`; provider-unknown
  fallback matching; empty view; `pool_used`/`pool_total`; weights do not affect counts.
- `tests/test_alias_history_usage_rendering.py` (new): header variants (with/without
  members segment, `(deduped)`); row column alignment; a 1% share still paints a visible
  cell; `✗`/`▶` chips appear only when non-zero; `unused` and `off-pool` tags; overflow
  row math (`+N more` count and remaining percent); loading and empty single-line
  states.
- `tests/test_alias_history_modal.py` (extend): `#alias-history-usage` exists and shows
  the loading line before the worker completes; shows counts after it lands;
  `action_load_more` / `action_refresh` / `action_toggle_hidden` each repaint it with
  the new view's numbers; `j`/`k` leave `modal._usage` identical (same object) — the
  perf guard for decision 11.
- `tests/test_models_panel_history.py` (extend): an alias row with a `|` pool captures
  every member; a `||` fallback alias captures its candidates; a plain alias captures
  one member from its effective target; a bucket captures the deduped union of its
  members; an alias-backed launch setting captures the snapshot's members.
- `tests/_alias_history_helpers.py` (extend): a `make_pool_member(...)` builder;
  `make_entry` keeps its current default of an empty pool.
- Visual: extend
  `tests/ace/tui/visual/_ace_models_panel_alias_history_png_snapshot_fixtures.py` with a
  three-member-pool view (one member unused, one off-pool observed model, one failed
  run) and add a `models_panel_alias_history_usage_120x40` case to
  `tests/ace/tui/visual/test_ace_png_snapshots_models_panel_alias_history.py`. The five
  existing alias-history goldens change and must be rebaselined intentionally.

## 6. Verification

```bash
just install                       # ephemeral workspace may have drifted deps
just check                         # whole-repo lint gates + diff-scoped tests
just test-visual tests/ace/tui/visual/test_ace_png_snapshots_models_panel_alias_history.py
# inspect .pytest_cache/sase-visual/ diffs, confirm each change is intended, then:
just test-visual tests/ace/tui/visual/test_ace_png_snapshots_models_panel_alias_history.py -- --sase-update-visual-snapshots
```

Then, before landing, run the exhaustive lane through `/sase_monitor` (it outruns a
turn), with a `--next` action so the follow-up agent acts on the result:

```bash
sase monitor start --command 'just check-full' --next '<follow-up action>'
```

Layout check (decision in §4.5): open the panel in a real 120×40 and a ~100×30 terminal
against an alias with a loaded-more window, and confirm the footer is fully visible and
the list still scrolls. `sase ace` is the surface; the `H` key on a Launch Control alias
row opens it.

## 7. Risks

- **Height regression.** The largest risk. Mitigated by the explicit budget in §4.5, the
  `max-height: 5` cap, the four-row limit, and the real-terminal check.
- **Golden churn.** Every alias-history PNG shifts because the modal's internals move.
  Rebaseline deliberately and eyeball each diff artifact; do not blanket-run
  `just update-visual-snapshots` for the whole corpus.
- **Provider/model spelling drift** between what a run recorded and what config lists
  (e.g. `claude/sonnet` vs a bare `sonnet`). Handled by the provider-unknown fallback
  match in §4.1; anything still unmatched degrades to a visible `off-pool` row rather
  than a wrong count.
- **Bucket double counting** is real today in the title rollup as well; this plan only
  fixes it inside the usage strip and deliberately does not change the title line.
  Mention it in the completion notes rather than expanding scope.

## 8. Non-goals / deferred

- No expand/collapse key or full-breakdown sub-panel for pools with many members; the
  `+N more` row covers the tail. Worth revisiting if buckets with 6+ distinct models
  turn out to be common.
- No per-alias breakdown inside a bucket (the strip is one aggregate across the view).
- No time-window controls, no per-project split, and no success-rate ranking; the ✗/▶
  chips are texture, not a second ranking axis.
- No CLI surface for the same numbers, though `summarize_alias_history_usage` is written
  so one could be added without touching the TUI.
