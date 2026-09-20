---
tier: tale
title: Hide Claude's Fable usage window until it runs low
goal:
  sase's TUI header stops showing Claude's weekly Fable usage window by default, showing
  it only once its remaining capacity falls below the generic indicator threshold, while
  users can restore the always-visible behavior with an explicit config override.
size: small
proposed_by: bbugyi200.apollo.1b
create_time: 2026-09-20 17:57:02
status: wip
---

# Hide Claude's Fable Usage Window Until It Runs Low

## Problem

sase's TUI header usage indicator always shows Claude's per-model weekly Fable window,
even at `100%` remaining. The header therefore carries a permanently-visible
`fable 100% 5d2h` badge that never tells the user anything actionable, and it competes
for the right-docked usage reserve against windows that do warrant attention.

The user wants that badge gone by default unless the Fable window crosses a low-capacity
threshold, while keeping the current always-visible behavior available as an explicit
opt-in.

## Root Cause

The behavior is not code — it is one line of shipped configuration.
`src/sase/default_config.yml` pins an exact-window display override:

```yaml
llm_provider:
  usage_metrics:
    indicator:
      default:
        below_remaining_percent: 20
      weekly_all: always
      providers:
        claude:
          windows:
            "weekly:claude-fable-5": always # <-- this
```

`resolve_policy` in
`sase/repos/linked/sase-core/crates/sase_core/src/provider_usage/indicator.rs` resolves
display policy in this precedence order:

1. exact `providers.<name>.windows.<key>`
2. `providers.<name>.default`
3. `weekly_all` (only for a positively classified weekly all-model window)
4. global `indicator.default`

The shipped `always` wins at step 1, so the Fable window is selected at every remaining
percentage.

## Approach

Delete the shipped exact-key override and let the Fable window fall through to the
generic `indicator.default` threshold (`below_remaining_percent: 20`). The header then
stays quiet until Fable drops strictly below 20% remaining, and the user restores
today's behavior by adding the exact key back with `always`.

This is the right mechanism because the indicator config already models exactly this
choice, and because the Fable window is **not** a weekly all-model window, so removing
the override cannot accidentally promote it to the `always` `weekly_all` policy.

Verify that claim before relying on it — it is the load-bearing fact in this plan. In
`indicator.rs`, `is_weekly_all_window` = `is_all_model_scope && is_weekly_window`, and
`is_all_model_scope` only returns `true` for `Account` applicability, or for
`Product { product: "claude", model_ids: [] }` whose key is exactly `session` or
`weekly`. The Fable window carries applicability
`{"kind": "models", "model_ids": ["claude-fable-5"]}` (see `_usage_window_identity` in
`src/sase/llm_provider/usage/_claude_support_windows.py`), which hits the `_ => false`
arm. So with the override removed, resolution falls to step 4 and `policy_source`
becomes `default`.

### Rejected Alternative

Replacing the override with an explicit
`"weekly:claude-fable-5": {below_remaining_percent: 20}` was considered and rejected. It
duplicates the global default's value at a higher precedence level, so a user who later
raises `indicator.default` to `30` would silently keep Fable pinned at `20` with no
indication why. Removing the key entirely makes the Fable window an ordinary window that
follows whatever generic threshold the user has set.

### Non-Goals — Do Not Do These

- **No Rust / sase-core change.** `UsageIndicatorConfigWire::default()` in
  `indicator.rs` already ships `providers: BTreeMap::new()` and
  `default: below_remaining_percent 20.0`. The built-in Rust defaults already behave the
  way we want; only the YAML layer disagrees. Do not open sase-core for this work.
- **No feature flag.** Per `sase memory read sase_flags.md`: "Do not flag anything users
  are meant to choose forever; that is a config field." Which usage windows appear in
  the header is a permanent user choice already expressed as config, so a flag would be
  wrong here and would have no removal condition.
- **No new config key.** Everything needed already exists under
  `llm_provider.usage_metrics.indicator`.
- **No PNG golden regeneration.** See Verification.

## Implementation

### 1. `src/sase/default_config.yml`

Under `llm_provider.usage_metrics.indicator.providers`, delete the entire `claude:`
mapping (the `claude:` key, its `windows:` key, and the
`"weekly:claude-fable-5": always` line). Keep the `muse:` entry and its explanatory
comment exactly as-is.

Then rewrite the commented Claude example that follows it. Today it reads:

```yaml
# providers:
#   claude:
#     windows:
#       "weekly:claude-fable-5": never
#       # Or restore the generic fallback:
#       # "weekly:claude-fable-5":
#       #   below_remaining_percent: 20
```

That comment documents escaping _from_ the shipped `always`, which no longer exists.
Invert it so it documents opting back _into_ pinning, and state the new shipped
behavior. Something in this shape (match the surrounding comment style and indentation):

```yaml
# Claude's per-model weekly Fable window ships with no entry of its own, so
# the generic `default` threshold above governs it: the header stays quiet
# until Fable drops below 20% remaining. Add an exact key to change that.
# providers:
#   claude:
#     windows:
#       # Pin it back to always-visible, at any remaining percentage:
#       "weekly:claude-fable-5": always
#       # Or hide it even when it runs low:
#       # "weekly:claude-fable-5": never
#       # Or give it its own threshold, independent of `default`:
#       # "weekly:claude-fable-5":
#       #   below_remaining_percent: 35
```

Leave the `codex`, `grok`, and `muse` commented examples below it untouched.

### 2. `src/sase/config/sase.schema.json`

The `indicator.providers` property carries a JSON Schema `default` annotation that
hard-codes the shipped map (around line 4048). It currently reads:

```json
"default": {
  "claude": { "windows": { "weekly:claude-fable-5": "always" } },
  "muse":   { "windows": { "session": "never" } }
}
```

Drop the `claude` member so the annotation stays truthful; keep `muse`. This annotation
is documentation only (Draft-07 `default` is not enforced), but
`tests/test_config_schema.py` holds the schema and `default_config.yml` in sync as a
pair, and a stale annotation would mislead anyone reading the schema.
`weekly:claude-fable-5` should appear nowhere in the schema after this edit — confirm
with a grep.

Do not change `definitions/usageIndicatorPolicy` or
`definitions/usageIndicatorProvider`; the policy grammar is unchanged.

### 3. `docs/configuration.md`

Three edits in the `llm_provider.usage_metrics` section:

- The example YAML block (~line 2070): remove the `claude:` / `windows:` /
  `"weekly:claude-fable-5": always` lines so the example matches the new shipped file.
- The field table row for
  `llm_provider.usage_metrics.indicator.providers.<name>.windows.<key>`: its Default
  cell reads ``inherit; Fable `always`, Muse `session` `never` `` — change it to
  ``inherit; Muse `session` `never` ``.
- The prose block beginning "The bundled default adds one exact Claude display override
  and one exact Muse override:" (~line 2118) and its code block and following paragraph.
  Rewrite so it describes a single bundled Muse override, and replace the Fable
  paragraph ("That observed weekly Fable window appears at any remaining percentage,
  including `100%` ...") with the new behavior: the Fable window has no bundled entry,
  so the global `indicator.default` threshold governs it and the header shows it only
  below 20% remaining; set the exact key `"weekly:claude-fable-5"` to `always` to
  restore the always-visible behavior, to `never` to hide it even when low, or to a
  `{below_remaining_percent: N}` of its own.

  **Preserve** the existing sentence about recursive config merge ("Because config
  layers merge recursively, an empty user `indicator.providers: {}` does not erase this
  bundled key, while an explicit value for the key does override it."), rewording it so
  it still applies to the remaining Muse key. It is the reason the user-facing opt-in
  works, and it should stay documented.

### 4. `docs/llms.md`

Two edits in the "Subscription Usage" section:

- The example YAML block (~line 2249): remove the `claude:` / `windows:` /
  `"weekly:claude-fable-5": always` lines.
- The paragraph at ~line 2272: the sentence "The default shows every positively
  classified weekly all-model window (including Muse's weekly window), Claude's observed
  weekly `weekly:claude-fable-5` window at any capacity, and any other observed window
  whose remaining capacity is strictly below 20%." must drop its Fable clause, and the
  later sentence "Set the exact Fable key to `never` to hide it, or to
  `{below_remaining_percent: 20}` to restore the generic fallback threshold." must be
  inverted to describe setting it to `always` to restore always-visible behavior.

### 5. `docs/ace.md` — verify only, expect no edit

The header example at ~line 3641 is:

```text
🎭 62% 3d4h · fable 0% 1d8h  🤖 81% 5d2h
```

`0%` remaining is below the 20% threshold, so this example stays accurate under the new
default and should be left alone. Read the surrounding paragraphs and confirm none of
them claim the Fable window is always shown; that section documents layout and color,
not selection policy, and links out to `configuration.md` for which windows appear. Only
edit if you find an actual selection-policy claim.

### 6. New test — `tests/llm_provider/test_claude_fable_usage_indicator_default.py`

Model it closely on the existing
`tests/llm_provider/test_muse_usage_indicator_default.py`, which is the established
pattern for locking a shipped per-provider indicator default. Reuse its
`_shipped_usage_metrics()` helper shape (parse `src/sase/default_config.yml` with
`yaml.safe_load` and return `llm_provider.usage_metrics`) and build snapshots with
`usage_provider` / `usage_window` from `tests/_usage_view_helpers.py`.

**Snapshot construction gotcha:** `usage_window` defaults `duration_seconds` to
`18_000.0` (5 hours). In `indicator.rs`, `is_weekly_window` short-circuits to `false`
when a present duration fails to match a week, so a claude `weekly` window left at that
default would not classify as weekly all-model and the `weekly_all: always` policy would
not select it. Set `duration_seconds = None` and `period_start = None` on both windows,
which is what the real collector emits (`parse_usage_windows` in
`_claude_support_windows.py` writes both as `None`). The Muse test does the same thing
for the same reason.

Build a claude provider with two windows:

- key `weekly`, applicability `{"kind": "product", "product": "claude"}` (no
  `model_ids`), at a healthy remaining percentage such as `62.0` — this is the weekly
  all-model anchor and must stay visible.
- key `weekly:claude-fable-5`, applicability
  `{"kind": "models", "model_ids": ["claude-fable-5"]}`, at a parameterized remaining
  percentage.

Project with
`provider_usage_project_indicator(snapshot, indicator=_shipped_usage_metrics()["indicator"], eligible_providers=frozenset({"claude"}), now=<frozen>)`
and assert:

1. **The shipped config no longer pins Fable.**
   `_shipped_usage_metrics()["indicator"]["providers"]` has no `"claude"` key. This is
   the direct regression guard on `default_config.yml`.
2. **Hidden at healthy and at the boundary.** At Fable remaining `100.0`, `21.0`, and
   `20.0`, the selected `window_key`s are exactly `["weekly"]`. Include `20.0`
   explicitly: the policy is _strictly_ below, so an exactly-20% window is still hidden,
   and that edge is worth pinning.
3. **Shown when low.** At Fable remaining `19.0`, both `weekly` and
   `weekly:claude-fable-5` are selected, and the Fable entry's `policy_source` is
   `"default"` (not `"window"`) — this proves it is riding the generic fallback rather
   than a surviving exact override.
4. **The user can restore today's behavior.** Deep-copy the shipped indicator config,
   add `{"claude": {"windows": {"weekly:claude-fable-5": "always"}}}` into its
   `providers` map (merge, do not clobber the `muse` key), reproject at Fable remaining
   `100.0`, and assert both windows are selected and the Fable entry's `policy_source`
   is `"window"`. This is the "users can still configure the current behavior" guarantee
   and is the most important assertion in the file.
5. **Header text.** Render with
   `build_usage_indicator_segment(usage_indicator_groups(entries, dark=True, now=<frozen>))`
   from `sase.ace.tui.widgets._provider_usage_indicator` and assert `"fable"` is absent
   from `segment.plain` under shipped defaults at `100.0`, and present in the
   restored-`always` projection. This closes the loop from config to the string the user
   actually sees.

Write real docstrings on the module and the test functions describing the shipped
default being locked, matching the Muse test's style.

### 7. Existing tests — expected to pass unchanged, but confirm

Do not edit these; just understand why they are unaffected, and investigate rather than
patch if any of them fails:

- `tests/llm_provider/test_claude_fable_usage_identity.py` passes its `indicator=`
  config inline (including `{"claude": {"windows": {FABLE_CANONICAL_KEY: "always"}}}`)
  and never reads `default_config.yml`. It keeps asserting that an explicit `always`
  works, which is now the opt-in path rather than the shipped default.
- `tests/llm_provider/test_muse_usage_indicator_default.py` asserts
  `indicator["providers"]["muse"] == {"windows": {"session": "never"}}` and separately
  sets `providers = {}`. Neither touches the `claude` key.
- `tests/test_config_schema.py` validates `default_config.yml` against
  `sase.schema.json`; removing a key from both keeps them consistent, and the JSON
  Schema `default` annotation is not validated either way.
- The ACE PNG visual fixtures in
  `tests/ace/tui/visual/_provider_usage_indicator_fixtures.py` call
  `provider_usage_project_indicator(...)` **without** an `indicator=` argument. A `None`
  indicator makes the Rust side use `UsageIndicatorConfigWire::default()`, which has
  never contained the shipped YAML overrides, so no golden can shift.

## Verification

Run the standard agent recipe from the repo root:

```bash
sase tool run check
```

Run `just fix` inline first if the docs or YAML edits need formatting.

Targeted runs while iterating:

```bash
just test -- tests/llm_provider/test_claude_fable_usage_indicator_default.py \
               tests/llm_provider/test_claude_fable_usage_identity.py \
               tests/llm_provider/test_muse_usage_indicator_default.py \
               tests/test_config_schema.py
```

Do **not** run `just check-full` and do **not** run `just fix-tui-screenshots`. No PNG
golden can change, for the reason given in step 7; if a visual golden does move, that is
a signal the change went further than intended — stop and re-read rather than accepting
the new golden.

Grep as a final sweep to confirm no stale claim survives:

```bash
grep -rn "weekly:claude-fable-5" src/ docs/ | grep -v "^tests/"
```

Every remaining hit should be either a commented opt-in example or prose describing the
opt-in — none should assert that the window is shown by default.

## Manual Confirmation

Optional but cheap, and worth doing if a Claude usage cache is present on this machine:

```bash
sase usage list -p claude --json
```

Confirm `windows[].key` still contains `weekly:claude-fable-5` — collection is
unchanged, only display selection moved. The window must remain fully visible in
`sase usage list` and in the ACE **Providers · Usage** view; this change touches the
header indicator only.

## Done When

- `src/sase/default_config.yml` ships no `claude` entry under
  `llm_provider.usage_metrics.indicator.providers`, and its comments document adding the
  exact key back.
- `src/sase/config/sase.schema.json` contains no `weekly:claude-fable-5`.
- `docs/configuration.md` and `docs/llms.md` describe the Fable window as
  threshold-governed and document `always` as the restore path.
- `tests/llm_provider/test_claude_fable_usage_indicator_default.py` exists and locks
  both the new default and the `always` opt-in.
- `sase tool run check` passes.
