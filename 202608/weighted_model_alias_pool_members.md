---
tier: tale
title: Weighted members in load-balanced model alias pools
goal:
  A load-balanced model alias pool member can carry a leading positive-integer weight
  (`3 grok/grok-4.6@xhigh`), and pool rotation hands out selections in the configured
  ratio while spreading each member's turns as evenly as possible across the cycle.
size: medium
proposed_by: bbugyi200.athena.05w
create_time: 2026-08-18 07:54:05
status: wip
---

# Weighted members in load-balanced model alias pools

## 1. Goal and user-visible behavior

Today a `|` model alias value is a flat round-robin pool: every member gets an equal
share of launches. This plan adds an optional per-member weight so one member can be
favored over the others.

Given:

```yaml
llm_provider:
  model_aliases:
    custom:
      foobar:
        model: "claude/sonnet@xhigh | codex/gpt-5.5@xhigh | 3 grok/grok-4.6@xhigh"
        description: "Grok-heavy pool."
```

then across every 5 consuming launches of `@foobar`, grok is selected 3 times, sonnet
once, and codex once — and those 3 grok launches are spread through the cycle rather
than fired back to back.

Everything else about pools is unchanged: availability filtering, the machine-global
cursor, single-consumption-per-launch, fail-closed resolution, temporary overrides, and
ordered `||` fallback chains.

## 2. Current implementation (read this first)

- `src/sase/llm_provider/load_balancing.py` owns the selector grammar
  (`parse_model_alias_selector`), the `ModelAliasSelector` dataclass (`mode`, `members`,
  `normalized`, `fingerprint`), the machine-global rotation state in
  `~/.sase/llm_lb.json` (`_read_entries_unlocked` / `_write_entries_unlocked`, schema
  `version: 1`, one entry per pool-owning alias with `alias` / `fingerprint` /
  `cursor`), and the two selection entry points `select_model_alias_pool_member` (round
  robin, cursor-advancing when `consume=True`) and `select_model_alias_fallback_member`.
- `src/sase/llm_provider/model_alias_resolution.py` resolves alias chains, builds
  `ModelAliasSelectorMember` display records via `model_alias_selector_details`, and
  produces actionable authoring errors via `validate_model_alias_selector_value`.
- `src/sase/llm_provider/model_launch_settings.py` re-implements the resolve-members →
  availability → select path for a raw selector typed directly into a launch model
  field, and builds its own `ModelAliasSelectorMember` tuple.
- `src/sase/doctor/checks_config_model_aliases.py` reports selector validation errors
  and availability notes.
- ACE surfaces: `src/sase/ace/tui/modals/models_panel_selector.py` (compose/parse glue),
  `.../models_panel_selector_builder.py` (guided builder modal),
  `.../models_panel_rendering_descriptions.py` (member list under the alias row),
  `.../models_panel_rendering_rows.py` + `src/sase/ace/tui/model_alias_styles.py`
  (`pool <available>/<total>` chip).

Two properties of the existing code make this change small and safe, and both MUST be
preserved:

1. `ModelAliasSelector.fingerprint` is asserted in
   `tests/llm_provider/test_load_balanced_aliases.py` to equal
   `sha256(json.dumps(members, ensure_ascii=False, separators=(",", ":")))`. Keep that
   exact spelling for pools whose weights are all `1`, so existing cursors survive.
2. `cursor` is a plain non-negative `int` in state schema `version: 1`. The design below
   keeps it an `int` and keeps the schema at `version: 1` — no migration, no state reset
   for existing unweighted pools.

## 3. Grammar

A pool member may be prefixed with a positive integer weight and at least one space:

```
<weight><whitespace><member>
```

Rules:

- Applies only inside a `|` load-balanced pool. A weight prefix in a `||` ordered
  fallback chain is an error (fallback is deterministic; a weight there is meaningless
  and would silently mislead).
- The weight is matched only as a leading `\d+` token followed by whitespace, on the
  already-stripped member text. Model identifiers never contain spaces, so this cannot
  shadow a real target.
- Valid range is `1` to `MAX_POOL_MEMBER_WEIGHT = 99` inclusive. `0`, a value above the
  max, and a leading `-`/`+`/decimal token are authoring errors (see §6). Leading zeros
  are accepted and normalized (`03 x` → weight 3).
- Weight `1` is the default and never needs writing. Canonical/normalized spelling omits
  it, so `1 claude/sonnet | 3 grok/grok-4.6` normalizes to
  `claude/sonnet | 3 grok/grok-4.6`.
- The weight prefix is stripped before every existing member rule applies, so
  `3 @medium@high` is a weight of 3 on the alias reference `@medium@high` and per-member
  trailing effort keeps working untouched.

`ModelAliasSelector.members` keeps holding the **bare** member strings (weight already
stripped), so every existing consumer of `.members` — resolution, validation, the
launch-settings path, the builder, the doctor — keeps working unchanged. The weights
ride alongside in a new parallel tuple.

## 4. Selection algorithm

Use smooth weighted round robin (the nginx `current_weight` algorithm), pre-expanded
into a fixed cycle schedule, so the persisted state stays a single integer cursor.

### 4.1 Schedule

For weights `w[0..n-1]` with `T = sum(w)`, build `schedule: tuple[int, ...]` of length
`T` holding member indices:

```python
cw = [0] * n
schedule = []
for _ in range(T):
    for i in range(n):
        cw[i] += w[i]
    pick = max(range(n), key=lambda i: cw[i])  # first max wins → lowest index on ties
    schedule.append(pick)
    cw[pick] -= T
```

Properties this must satisfy (assert them in tests):

- Each member index `i` appears exactly `w[i]` times per cycle.
- `cw` returns to all zeros after `T` steps, so the schedule tiles cleanly.
- **Equal weights produce the identity permutation**: `[1,1,1] → [0,1,2]`,
  `[1,1] → [0,1]`, `[1,1,1,1] → [0,1,2,3]`. This is what makes the change inert for
  every pool that exists today.

Worked examples to pin in tests:

| weights                                   | schedule                            |
| ----------------------------------------- | ----------------------------------- |
| `[1, 1, 3]` (`sonnet \| codex \| 3 grok`) | `[grok, sonnet, grok, codex, grok]` |
| `[3, 1, 1]`                               | `[0, 1, 0, 2, 0]`                   |
| `[2, 1]`                                  | `[0, 1, 0]`                         |
| `[1, 2, 3]`                               | `[2, 1, 0, 2, 1, 2]`                |
| `[5, 1]`                                  | `[0, 0, 0, 1, 0, 0]`                |

Cache the schedule with an `lru_cache`-decorated module-level helper keyed by the
weights tuple; it is a pure function of the weights and each call is at most a few
thousand cheap operations.

### 4.2 Cursor semantics

`cursor` becomes "the next position in the pool's weighted schedule" instead of "the
next member index". For an unweighted pool `T == n` and `schedule[i] == i`, so the two
readings coincide and every stored cursor keeps its exact current meaning.

`select_model_alias_pool_member` keeps its signature
(`alias, selector, availability, *, consume`) and keeps returning a **member index**:

1. If `not any(availability)`: return member index `0` without consuming —
   byte-identical to today's all-down behavior, which
   `test_pool_member_snapshot_marks_available_skip_and_all_down_fallback` and the doctor
   note both depend on.
2. Otherwise scan `offset in range(T)`, `position = (cursor + offset) % T`; the first
   `position` whose `schedule[position]` is available wins. Return `schedule[position]`,
   and when `consume=True` store `cursor = (position + 1) % T`.
3. Every existing failure mode is preserved: a fingerprint mismatch resets the cursor to
   `0`; a stored cursor out of range is tolerated by the `% T`; and any state/lock
   exception falls back to selecting from cursor `0` without raising.

Unavailable members simply lose their slots for that pass — they do not bank credit and
burst on return. With `[1,1,3]` and grok down, selection alternates sonnet, codex
evenly.

### 4.3 Fingerprint

`ModelAliasSelector.fingerprint` must change whenever weights change (so re-weighting a
pool restarts its cycle instead of landing mid-schedule at a stale position):

- All weights `1` → the exact legacy payload, unchanged.
- Any weight `> 1` → a distinct payload that includes the weights, e.g.
  `json.dumps([[w, m] for w, m in zip(weights, members)], ensure_ascii=False, separators=(",", ":"))`.

## 5. Implementation

### 5.1 `src/sase/llm_provider/load_balancing.py`

- Add `MAX_POOL_MEMBER_WEIGHT = 99`.
- Add `weights: tuple[int, ...] = ()` to `ModelAliasSelector`. In `__post_init__`
  (`object.__setattr__`, the dataclass is frozen + slots), fill an empty tuple with
  `(1,) * len(members)` and raise `ValueError` on a length mismatch or an out-of-range
  weight, so the invariant "`len(weights) == len(members)` and every weight is in range"
  holds for every construction site, including the TUI's `compose_selector`.
- Add a `weighted` property (`any(weight > 1 ...)`) used by `normalized` and
  `fingerprint`.
- `normalized`: render `f"{weight} {member}"` when `weight > 1`, else the bare member;
  join as today.
- `parse_model_alias_selector`: after splitting on the operator and rejecting empty
  members, peel the weight prefix off each member. In `fallback` mode a weight prefix
  raises `ModelAliasSelectorError`. In `round_robin` mode an out-of-range or malformed
  numeric prefix raises `ModelAliasSelectorError`. Do **not** change behavior for values
  with no `|` at all (see §6).
- Add the cached `_weighted_schedule(weights) -> tuple[int, ...]` helper and rework
  `_selection_index` / `select_model_alias_pool_member` per §4.2. `_selection_index`'s
  contract becomes "first available _schedule position_ at/after cursor"; the
  all-unavailable short-circuit moves to (or stays in) the caller so the member-`0`
  behavior is explicit.
- Update the module docstring: the state cursor is now a schedule position, and pools
  may be weighted.

### 5.2 `src/sase/llm_provider/model_alias_resolution.py`

- Add `weight: int = 1` to `ModelAliasSelectorMember`.
- In `model_alias_selector_details`, populate each member's `weight` from
  `selector.weights[index]`.
- In `validate_model_alias_selector_value`, add weight-specific authoring errors:
  - When `parse_model_alias_selector` raises, the existing single-error return already
    surfaces the message — no change needed.
  - When the value is **not** a selector (`selector is None`) but the trimmed value
    carries a leading weight prefix, return an error explaining that weights only apply
    to `|` load-balanced pool members.
- No change is needed to the two selector branches inside `_resolve_model_alias_result`:
  they iterate `selector.members` (already weight-free) and pass `selector` straight to
  `select_model_alias_pool_member`.

### 5.3 `src/sase/llm_provider/model_launch_settings.py`

In the `raw_selector` branch, thread `raw_selector.weights[index]` into each constructed
`ModelAliasSelectorMember`. Everything else (availability, selection, `selected_index`)
already flows through `select_model_alias_pool_member` and needs no change.

### 5.4 `src/sase/doctor/checks_config_model_aliases.py`

The alias loop currently computes `selector_errors` for every alias but only reports
them under `if "|" in target:`. Report them whenever `selector_errors` is non-empty, so
a weight prefix on a non-pool value (`model: "3 claude/opus"`) is surfaced instead of
silently failing at launch time. Keep the existing availability notes exactly where they
are, gated on the value actually being a selector.

### 5.5 ACE surfaces

- `src/sase/ace/tui/model_alias_styles.py`: add `append_pool_weight(text, weight)` that
  appends a ` ×<n>` chip only when `weight > 1`, and export it. One spelling shared by
  both render sites below.
- `src/sase/ace/tui/modals/models_panel_rendering_descriptions.py`: in the `pool: …`
  member list, append the weight chip for each member. Because the chip is suppressed at
  weight `1`, every existing rendering — and every existing PNG golden — is
  byte-identical unless a pool is actually weighted.
- `src/sase/ace/tui/modals/models_panel_selector.py`: extend `compose_selector` with an
  optional `weights` argument passed through to `ModelAliasSelector`.
- `src/sase/ace/tui/modals/models_panel_selector_builder.py`:
  - Track `self._weights: list[int]` alongside `self._members`; `_seed_selector` returns
    the parsed weights (all `1` for a non-selector seed).
  - Keep the two lists in lockstep in `_append_member` (append `1`),
    `action_remove_member` (delete the same index), and `_reorder` (swap the same pair).
  - Add bindings `w` = increase weight and `W` = decrease weight for the highlighted
    member, clamped to `1..MAX_POOL_MEMBER_WEIGHT`, refreshing the view (the header
    shows the live normalized expression, so the effect is immediately visible). Extend
    `_KEYS_HINT` with `w/W=weight`.
  - Weights only exist in `round_robin` mode: `action_toggle_mode` switching to
    `fallback` resets all weights to `1` and `self.notify(...)`s once when any weight
    was dropped.
  - `_member_option` renders the weight chip via `append_pool_weight`.
  - `compose_selector` / `_validation_errors` / `action_confirm` pass the weights
    through so the confirmed value is the canonical normalized expression.
- `src/sase/ace/tui/modals/models_panel_alias_edit.py`: the custom-value hint text
  should mention the weight prefix (e.g. `A | 3 B` weights B three-to-one). The member
  loop there iterates `parsed.selector.members`, which stays weight-free, so its
  rejection checks need no change.

### 5.6 Docs and schema

- `docs/llms.md`:
  - In the selector paragraph (~line 1149), document the weight prefix, its range, that
    it is pool-only, and that the `llm_lb.json` cursor is a position in the pool's
    weighted cycle.
  - In the "Both selectors accept two or more members…" paragraph, add the weight
    grammar and the "weights are invalid in `||`" rule.
  - Update the pool-fairness audit paragraph (~line 1183): member counts should track
    the configured weight ratio over a full cycle, not stay within one launch of each
    other.
  - Mention the builder's `w`/`W` keys where the guided builder is described.
- `src/sase/config/sase.schema.json`: extend the two prose descriptions for
  `llm_provider.model_aliases.builtin` (`additionalProperties`) and
  `llm_provider.model_aliases.custom.*.model` with the weight grammar. There is no regex
  constraint on these values, so no pattern change is needed.
- `src/sase/default_config.yml`: add a weighted example to the commented
  `model_aliases.builtin` block so the syntax is discoverable from the shipped config.
- Do **not** hand-edit `CHANGELOG.md` (release-please owns it) and do **not** change the
  shipped `src/sase/llm_provider/model_alias_defaults.yml` targets — this plan adds the
  capability, it does not re-weight the shipped size aliases.

## 6. Error messages

Match the existing actionable, lowercase, no-trailing-period style used by
`ModelAliasSelectorError` and `validate_model_alias_selector_value`:

| Input                     | Where                                 | Message (wording may be refined, meaning must match)                                        |
| ------------------------- | ------------------------------------- | ------------------------------------------------------------------------------------------- |
| `A \| 0 B`                | parser raises                         | `load-balanced pool weights must be between 1 and 99; got '0'`                              |
| `A \| 100 B`              | parser raises                         | `load-balanced pool weights must be between 1 and 99; got '100'`                            |
| `A \| -2 B`               | parser raises                         | same range message, quoting `-2`                                                            |
| `A \|\| 2 B`              | parser raises                         | `ordered fallback chains cannot weight candidates; remove the '2 ' prefix from candidate 2` |
| `3 claude/opus` (no `\|`) | `validate_model_alias_selector_value` | `weights only apply to '\|' load-balanced pool members; remove the '3 ' prefix`             |

A malformed numeric prefix must be reported as a weight problem, not left to fail later
as an unknown provider — that is the whole point of catching `-2`/`0`/`100` in the
parser.

Runtime behavior on a raise is unchanged and already correct:
`_resolve_model_alias_result` catches `ModelAliasSelectorError` and fails closed to the
raw input, the doctor reports the message, and the TUI shows it inline.

**Deliberate non-change:** `parse_model_alias_selector` must keep returning `None` (not
raising) for a weight-prefixed value with no `|`, because it is called on ordinary
single-target values all over the launch path. That case is caught by validation
(doctor + ACE edit preview) instead, which keeps the blast radius of this change at zero
for non-pool values.

## 7. Tests

Add to the existing suites; do not create parallel structures.

- `tests/llm_provider/test_load_balanced_aliases.py`
  - Parser: `"claude/opus | 3 codex/gpt-5.5"` →
    `members == ("claude/opus", "codex/gpt-5.5")`, `weights == (1, 3)`,
    `normalized == "claude/opus | 3 codex/gpt-5.5"`.
  - `"1 claude/opus | 3 codex/gpt-5.5"` normalizes with the explicit `1` dropped.
  - `"3 @medium@high | claude/opus"` keeps the alias reference and its effort intact.
  - Weight range errors and the `||`-with-weight error (`pytest.raises`).
  - Fingerprint: unchanged legacy sha256 when all weights are `1`; different from the
    unweighted fingerprint of the same members when any weight is `> 1`.
- New `tests/llm_provider/test_weighted_alias_pools.py`
  - Schedule invariants: per-member counts equal the weights; equal weights give the
    identity permutation for 2/3/4 members; the table in §4.1 reproduces exactly.
  - End-to-end consumption: configure `@pool` as
    `claude/sonnet | codex/gpt-5.5 | 3 grok/grok-4.6` (reuse
    `tests/llm_provider/_load_balanced_alias_helpers.configure_pool`'s `value`
    argument), then 10 consuming `resolve_model_alias("@pool", consume=True)` calls
    yield exactly 6 grok / 2 sonnet / 2 codex, in the two-cycle repetition of the §4.1
    schedule.
  - Availability: with grok unavailable, consumption alternates the two weight-1 members
    evenly and never selects grok; with everything unavailable, member 0 is selected and
    the cursor does not advance.
  - `model_alias_selector_details("pool")` reports `weight` per member and marks the
    right `selected` member before and after a consume.
- `tests/llm_provider/test_load_balanced_alias_state.py`
  - A weighted pool still writes state `version: 1` with an `int` cursor, and the cursor
    walks `0..T-1` then wraps.
  - Re-weighting an existing pool (same members, new weights) resets selection to the
    start of the new cycle via the fingerprint mismatch.
  - Unweighted pools keep their existing cursor assertions unchanged.
- `tests/doctor/test_checks_config_model_aliases.py`
  - `model: "3 claude/opus"` (no `|`) produces the weight problem.
  - `model: "claude/opus || 2 codex/gpt-5.5"` produces the fallback-weight problem.
- `tests/test_models_panel_selector_builder.py`
  - Seeding from a weighted pool populates weights; `w`/`W` increment/decrement and
    clamp at `1` and `MAX_POOL_MEMBER_WEIGHT`; remove/reorder keep weights aligned with
    members; toggling to fallback drops weights; confirm emits the canonical weighted
    expression.
- Visual snapshots: the builder's footer hint string changes, so
  `tests/ace/tui/visual/test_ace_png_snapshots_models_panel_edit.py::test_models_panel_selector_builder_png_snapshot`
  needs its golden refreshed. Run `just test-visual` and re-accept with
  `--sase-update-visual-snapshots`, then confirm no other golden moved (nothing else
  should, because weight chips are suppressed at weight `1`).

## 8. Decisions and non-goals

- **No feature flag.** The syntax is purely additive and inert: a pool with no weight
  prefix parses to all-`1` weights, whose schedule is the identity permutation and whose
  fingerprint is byte-identical to today's, so no existing configuration changes
  behavior and there is no half-ready path to gate. Weights are a config field users
  choose permanently, which the flag guidance explicitly excludes.
- **No Rust core work.** The whole model-alias selector subsystem is Python-only today
  (`sase_core_rs` exposes no alias/load-balancing surface); this plan extends existing
  Python in place rather than opening a cross-repo migration. Moving the subsystem
  behind the core boundary is a separate, larger question.
- **Weights stay config-only**, exactly like the `|` and `||` operators themselves:
  `%model` values, launch-scoped alias overrides, and temporary overrides remain single
  targets.
- **No weight support in `||` fallback chains** — ordered fallback is deterministic, so
  a weight there can only mislead.
- **State schema stays `version: 1`** with an integer cursor. No migration, no reset for
  existing pools.
- **Not in scope:** re-weighting the shipped size aliases, a fractional/percentage
  weight syntax, per-provider global weighting, and any change to how
  `pool <available>/<total>` counts members (it stays a member count, not a weight sum).

## 9. Verification

```bash
just install
just check
```

Then, because this touches the ACE Models panel rendering and the visual suite:

```bash
just test-visual
```

`just check-full` must be run through `/sase_monitor` (never inline) before landing,
since the change touches shared llm_provider resolution used across the launch path.
