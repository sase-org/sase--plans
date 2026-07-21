---
tier: tale
title: Models panel pool and effort display
goal: 'The ace Models panel makes load-balanced alias pools and reasoning-effort levels
  first-class: every pool row shows its members, their availability, and the exact
  member the next launch will use, while the panel header shows the configured default
  effort and each alias that overrides it shows its effort inline — all consistent
  with existing provider/effort styling conventions.

  '
create_time: 2026-07-21 10:42:14
status: done
prompt: '[202607/prompts/models_panel_pool_effort_display.md](prompts/models_panel_pool_effort_display.md)'
---

# Plan: Models panel pool and effort display

## Background

The recent load-balanced pool work (`feat!: add load-balanced model alias pools`) taught the model-alias layer two
things the Models panel (`,m` in `sase ace`) barely surfaces today:

1. **Pools** — a configured/implicit alias value may be pipe-separated (`claude/opus@medium | codex/gpt-5.5`). Launches
   pick the next available member round-robin via a machine-global cursor (`src/sase/llm_provider/load_balancing.py`),
   peeking with `consume=False` and advancing with `consume=True`. The panel currently shows pool members only in the
   two-line description strip (`pool: ✓ member · ✓ member`), with no indication of which member the next launch will
   actually use, and the pool-owning row itself is indistinguishable from a plain alias row.
2. **Effort** — alias values may carry a trailing `@<level>` effort suffix (canonical vocabulary in
   `src/sase/xprompt/effort.py`), which takes precedence over the `llm_provider.default_effort` config value
   (`resolve_effective_effort` in `src/sase/llm_provider/config.py`). The panel shows neither the configured default
   effort nor per-alias effort overrides anywhere except incidentally inside pool-member strings.

This plan makes both concepts first-class in the panel while staying inside the panel's existing visual language: the
fixed kind/name columns, the `PROVIDER(model)` badge column, the right-aligned provenance/state tag, and the
fixed-height two-line description strip.

**Rust core boundary note:** the entire model-alias/pool/effort domain lives in this repo's Python
(`src/sase/llm_provider/`), not in `sase-core`; this work extends those existing modules and their display layer only.
No Rust changes.

## UX design

Target layout (110-column modal, same columns as today):

```
                                   Models
                          default effort: @ xhigh

  default    default            CODEX(gpt-5.6-sol)            implicit
  ▸ bucket   coders             7 aliases                     bucket
  role       epic_lander        CODEX(gpt-5.6-sol)            configured
  role       big_epic_lander    CLAUDE(claude-fable-5)        configured → @smartest
  ▸ bucket   phase_worker       4 aliases                     bucket
  role       smartest           CLAUDE(claude-fable-5) @ max  configured
  role       cheapest           CLAUDE(opus) @ medium         implicit · pool 2/2
  ▸ bucket   researchers        3 aliases                     bucket
 ──────────────────────────────────────────────────────────────────────
  Cheap load-balanced pool for high-volume agents.
  pool: → ✓ claude/opus@medium · ✓ codex/gpt-5.5
```

### 1. Panel header: default effort

Add a second line to the centered title (`_title_text()` in `src/sase/ace/tui/modals/models_panel_display.py`):

- When `llm_provider.default_effort` is configured: `default effort: @ xhigh`, rendered with the exact uniform suffix
  styling used by `append_model_field` (`src/sase/llm_provider/model_label.py`): dim `#878787` for the connective text
  and `bold #AF87FF` for the level. The label itself is dim so the level pops.
- When unset: `default effort: provider default` fully dim — truthful, since launches without any effort run at whatever
  the provider CLI defaults to.
- The bucket-drilled-in breadcrumb (`Models › phase_worker`) keeps working; the effort line renders beneath the
  breadcrumb identically in both states.

Expose the value through a new public accessor in `src/sase/llm_provider/config.py` —
`default_reasoning_effort() -> str | None` — that wraps the existing private `_get_default_effort()` (validated,
normalized), and export it from `src/sase/llm_provider/__init__.py`. The panel must not read raw config or duplicate
validation.

### 2. Alias rows: effort suffix in the badge column

Add `effort: str | None` to `AliasView` (`src/sase/llm_provider/alias_view.py`), populated from the same resolution that
already produces the row's effective provider/model — switch `_effective_provider_model` to the existing
`resolve_model_provider_with_effort` (registry) and thread the third element through. Semantics fall out of the resolver
naturally:

- Alias value carries `@<level>` (directly or inherited through an `@alias` chain, e.g. `@smartest@high`) → that level.
- Pool alias → the effort of the member the next launch would use.
- Active temporary override → `None` (the resolver's override path yields no alias-borne effort; the launch falls back
  to the default), so overridden rows show the override chip and no stale effort.

Render it in `render_alias_row` (`src/sase/ace/tui/modals/models_panel_rendering.py`) as a ` @ <level>` suffix _inside_
the provider/model badge column, using the same dim-`@`/purple-level styling as the header line. `_provider_model_text`
must include the suffix so `provider_model_column_width` measures and truncates the combined badge correctly (cap stays
`PROVIDER_MODEL_CELL_MAX`; ellipsis behavior unchanged).

Deliberate scope choice: rows show only _alias-borne_ effort — exactly the "overrides the default" set the user cares
about. Painting the effective default on every row would be redundant with the header and turn signal into noise.

### 3. Alias rows: pool availability chip in the state tag

For a row whose `pool_members` is non-empty and which has no active override, `state_tag` appends a pool chip after the
provenance word:

- `configured · pool 2/2` / `implicit · pool 2/2` — green chip when all members are available.
- `pool 1/2` in the existing warning amber (`#FFD75F` family) when some members are unavailable; red-leaning (`#D78787`
  family) `pool 0/2` when none are (the next launch will fail with normal provider diagnostics).
- Pool aliases never render the `→ @ref` reference suffix (a pool value is not a single reference), which keeps the tag
  length bounded.
- Overridden pool rows keep the existing override chip untouched — the suspension story lives in the description strip.

### 4. Description strip: next-member marker, suspension, effort provenance

The strip stays fixed at two content rows (`#models-panel-title`/CSS budget in `src/sase/ace/tui/styles.tcss` is
untouched). Changes in `description_text_for_view`:

- **Pool line next marker.** The member the next launch will use is prefixed with a bold `→` and rendered at full
  brightness; the other members keep their current styling but slightly dimmed, so the eye lands on the next target:
  `pool: → ✓ claude/opus@medium · ✓ codex/gpt-5.5`. Unavailable members keep `×` and the dim red style; invalid members
  render their raw value with the warning style.
- **Suspension truthfulness.** When the alias has an active temporary override, load balancing is suspended
  (`_resolve_model_alias_result` returns the override before pool selection). Render the label as
  `pool (suspended by override):` and the whole member list dim, with no `→` marker — the panel must never claim a next
  member that a launch would not use.
- **Effort provenance line (non-pool rows).** A non-pool alias with alias-borne effort uses its free second strip line
  for provenance, dim:
  - `effort: medium · overrides default xhigh` when a default is configured and differs;
  - `effort: xhigh · matches default` when equal (still explicit — it pins the level even if the default later changes);
  - `effort: medium · no default configured` otherwise. Pool rows skip this line (their member list already spells out
    per-member effort, and the strip has no third row).
- **Bucket strip model mix.** Extend `BucketView._model_label` so member labels include the `@<effort>` suffix when
  present; the existing `model_counts` aggregation then naturally distinguishes
  `claude/opus@medium ×2 · claude/opus ×1`.

### Considered and rejected

- **Drill-in pool rows (like buckets).** Pool members are not aliases — no per-member override/edit/reset actions exist
  — so a drill-in would create a row type whose every action is disabled. The inline strip handles realistic 2–4 member
  pools cleanly.
- **Effective effort on every row.** Redundant with the header; noise.
- **New keybindings.** None needed; footer and help modal are unchanged.

## Data-layer changes (`src/sase/llm_provider/`)

1. **`config.py`**
   - `default_reasoning_effort()` public accessor (wraps `_get_default_effort`).
   - `ModelAliasPoolMember` gains `is_next: bool = False`.
   - `model_alias_pool_members(name)` computes the peek index with the same
     `select_model_alias_pool_member(alias, pool, availability, consume=False)` call that launches use (availability
     derived from the already-computed member list) and marks that member `is_next=True`. The all-unavailable case marks
     the member the resolver would still select (index-0 fallback semantics), matching real launch behavior.
2. **`alias_view.py`**
   - `AliasView.effort: str | None = None`.
   - `_effective_provider_model` returns the effort triple via `resolve_model_provider_with_effort`.
   - **Consistency by construction:** for a view with non-empty `pool_members` and no active override, derive the row's
     provider/model/effort from the `is_next` member's resolved target/provider/effort instead of issuing a second peek.
     This makes the row badge and the strip's `→` marker agree structurally rather than by coincidence of two
     cursor-state reads.
3. **`__init__.py`** — export the new accessor (and anything the panel needs) through the package facade, matching how
   `build_alias_views` is exported.

## Rendering/display changes (`src/sase/ace/tui/modals/`)

- `models_panel_rendering.py`: effort suffix in `_provider_model_text` + `render_alias_row`; pool chip in `state_tag`;
  next-marker, suspension, and effort-provenance rendering in `description_text_for_view`; bucket mix label change rides
  on `BucketView`.
- `models_panel_display.py`: two-line `_title_text()` with the default-effort header (load the value once per panel
  build alongside `_load_alias_views`, not per keystroke).

## Reliability and performance

- All new data is computed inside the existing panel-open snapshot path (`build_alias_views` /
  `model_alias_pool_members`); rendering stays pure over the snapshot. No resolution, config reads, stats, or globs are
  added to per-keystroke paths (tui_perf rules 8/11).
- The added peek is one tiny locked JSON read per pool-owning alias per panel build (2 implicit pools today);
  `provider_cli_available` results are already process-cached, so availability marks add no PATH scans.
- Peeking never consumes: only authoritative launch lanes pass `consume=True`, so opening the panel cannot skew
  rotation.
- The panel refresh path (`_refresh_rows` after override set/clear) picks the new fields up automatically; no new
  refresh code paths (tui_perf rule 5).
- Rendering handles degenerate input: invalid pool members, pools where every member is down, effort levels at maximum
  width (`@ minimal`), and badge truncation at `PROVIDER_MODEL_CELL_MAX` must all render without pushing the state tag
  off-column.

## Testing

- **Unit — data layer** (`tests/llm_provider/`): `default_reasoning_effort` validation/normalization;
  `model_alias_pool_members` `is_next` marking (fresh cursor, advanced cursor, unavailable-member skip, all-unavailable
  fallback, fingerprint reset after membership change); `AliasView.effort` for direct suffix, chain-inherited suffix,
  pool next-member effort, and override-active `None`; pool-row provider/model/effort derived from the `is_next` member.
- **Unit — rendering** (`tests/test_models_panel*.py`): effort suffix presence/absence and truncation; pool chip states
  (all up / partial / none / overridden); strip next marker, suspension line, and all three effort provenance variants;
  header line configured/unset states; bucket mix labels with effort.
- **PNG snapshots** (`tests/ace/tui/visual/`): the header line changes every existing `models_panel_*` golden —
  regenerate them all with `--sase-update-visual-snapshots` after eyeballing `.pytest_cache/sase-visual/` artifacts. Add
  at least one new scenario exercising the full design in one frame: a configured default effort, a configured pool with
  mixed availability and per-member efforts with the next marker visible, and a non-pool alias showing the effort
  provenance line; plus one snapshot of a pool alias suspended by an active override.
- `just check` (and `just test-visual`) must pass before completion.

## Documentation

- `docs/ace.md` — Models panel section: header effort line, effort suffix, pool chip, next marker, and suspension
  display.
- `docs/llms.md` — pools/effort sections: note that the Models panel shows pool membership, availability, and the next
  selection, and where default vs. alias-borne effort appears.
- `docs/configuration.md` — only if its `default_effort`/pool passages reference the panel display; keep claims in sync.

## Out of scope

- Overriding or editing per-pool-member values, per-member panel actions, or pool drill-in rows.
- Changing pool selection/rotation semantics, cursor storage, or effort resolution precedence.
- Effort support in the temporary-override picker flow (override values keep existing semantics).
- Top-bar alias-override indicator changes.
