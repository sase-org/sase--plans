---
tier: epic
title: Compact, configurable usage window indicators
goal: Make each provider usage window independently configurable and show compact,
  truthful capacity and reset countdowns with clear ten-bucket colors in ACE.
phases:
- id: window-policy
  title: Define shared usage window identity and visibility policy
  depends_on: []
  size: medium
  description: 'window-policy: implement Rust classification, policy validation and
    selection, time-aware projection, bindings, and contract tests.'
- id: config-and-cache
  title: Integrate configuration and time-aware cached display data
  depends_on:
  - window-policy
  size: medium
  description: 'config-and-cache: integrate Python adapters, schema and defaults,
    live config invalidation, documentation, and the Usage Window glossary strand.'
- id: compact-display
  title: Render and verify the compact usage window display
  depends_on:
  - window-policy
  - config-and-cache
  size: medium
  description: 'compact-display: implement icon badges, countdowns, ten-color themes,
    bounded layout, accessible disclosure, integration tests, and visual verification.'
proposed_by: bbugyi200.athena.0hy
create_time: 2026-09-10 07:06:03
status: done
bead_id: sase-z7
---

- **PROMPT:** [prompts/202609/usage_window_indicators.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/usage_window_indicators.md)
- **BEAD:** [sase-z7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-z7/README.md)

# Compact, configurable usage window indicators

## Outcome and scope

Replace ACE's single leading usage warning with independently selected usage window
indicators. With no user configuration, every observed weekly window covering all models
is eligible for display at every percentage; other windows appear only when remaining
capacity is strictly below 20%. Show several windows from the same provider when
applicable. Actual on-screen capacity is bounded by available terminal space.

The standard entries are:

```text
🎭 62% 3d4h    🤖 81% 5d2h    🛰️ 44% 1d7h
🎭 5h/all 18% 2h9m    🎭 wk/fable 7% 1d6h
```

These percentages mean capacity remaining. Times mean time until the reported reset. A
weekly window covering all models never has a window description specifier, including
`wk/all`, `wk`, or a vendor allowance label. Other windows keep a short, unambiguous
specifier. The detailed Usage view remains the comprehensive inspection surface.

This is an epic because it has three substantial seams with independently verifiable
outcomes: shared Rust semantics and bindings, configuration/cache integration, and
Textual presentation. All phase dependencies are explicit. Activate the completed SASE
feature together; core contracts can land first to satisfy the pinned dependency, and
the first two phases add contracts and plumbing without activating a partial new UI.
Permanent preferences are ordinary config, not feature flags.

Keep provider collection, usage-limit auto-disable, routing priority, and model picker
warning semantics intact. The existing `warn_percent` and `critical_percent` remain
percentages **used** for those attention classifications. New indicator policies use
percentages **remaining** and are independent of them. This plan adds no CLI options,
new collector transports, polling network calls, or routing decisions.

## Findings that shape the design

- `src/sase/llm_provider/usage/hints.py::indicator_usage_items` currently picks one
  provider-level attention hint per provider. Enumerating those hints cannot implement
  independent selection of healthy weekly and several constrained windows.
- `src/sase/ace/tui/widgets/_provider_usage_indicator.py` currently renders one
  attention item, an uppercase provider name, `left`, and a scope label, then rolls the
  rest into a count. A current test expects `! GROK 14% left · wk/all +2`.
- `ProviderDisablesIndicator` shares space with routing controls. Its current budget
  reserves only one cell for the tab bar and its smallest candidate can exceed the
  requested budget. The current 80-column PNG demonstrates severe top-bar crowding.
- Public windows already contain stable `key`, percentage, `resets_at`, duration,
  applicability, observation time, freshness, and reset-passed fields. Core owns their
  semantics under `crates/sase_core/src/provider_usage/` in `sase-core`.
- Claude uses `session`, `weekly`, and model-specific keys such as
  `weekly:claude-fable-5`. Its session and weekly observations have `product: claude`
  applicability and currently omit duration. The existing generic model matcher
  deliberately treats a product without model IDs as unknown. Do not change that matcher
  to fix this display.
- Codex emits `<limit_id>:primary` and `<limit_id>:secondary`; only `limit_id == codex`
  is known account scope. The period comes from `windowDurationMins`, not the word
  `secondary`. Other Codex buckets deliberately retain unknown scope.
- Grok's known included windows use `included_weekly`, `included_monthly`, or
  `included`, with account applicability and optional period timestamps.
- The usage peek is already loaded in a worker and the indicator has a 30-second timer.
  Its reload token currently tracks only the usage file. Time and configuration changes
  must also affect display without repeatedly reopening the store.

## Product contract

### Configuration

Put display preferences under the existing usage domain, grouped separately from
collection controls:

```yaml
llm_provider:
  usage_metrics:
    indicator:
      enabled: true
      default: { below_remaining_percent: 20 }
      weekly_all: always
      providers: {}
```

Every policy value has exactly one of three forms:

- `always`: select each observed matching window at any percentage.
- `never`: suppress matching window entries.
- `{below_remaining_percent: N}`: select when remaining percentage is strictly less than
  finite numeric `N`, with `0 <= N <= 100`. Zero selects no numeric windows; 100 still
  excludes an exactly full window. Use `always` to include full capacity.

A practical customization is:

```yaml
llm_provider:
  usage_metrics:
    indicator:
      providers:
        claude:
          windows:
            session: always
            "weekly:claude-fable-5": { below_remaining_percent: 35 }
        codex:
          windows:
            "codex:secondary": { below_remaining_percent: 10 }
        grok:
          windows:
            included_monthly: always
```

Per-provider objects accept optional `default` and `windows` fields. Resolve a window's
policy in this order, with the first applicable value winning:

1. `indicator.providers.<provider>.windows.<exact-window-key>`.
2. `indicator.providers.<provider>.default` when explicitly supplied.
3. `indicator.weekly_all` when the window is positively classified weekly/all-model.
4. `indicator.default`.

Thus a provider `default: never` suppresses its windows except explicit overrides;
`default: always` shows all of that provider's observed windows. To use thresholds for
all providers' weekly windows too, set `weekly_all: {below_remaining_percent: 20}`.
There is no implicit inheritance via `null`, glob syntax, ordered rule list, or matching
against display labels. Use SASE's existing mapping merge and scalar replacement
semantics; each resolved policy has one meaning, without list concatenation or
rule-order surprises. Do not add a custom global configuration merger.

Provider names and exact window IDs are open-ended, allowing third-party providers and
windows not yet observed. Unmatched entries are inert, not errors. Document how to find
IDs using the existing `sase usage list -p claude --json` output's `windows[].key`, and
include IDs in indicator tooltips. Never derive config selectors from shortened labels.

`indicator.enabled: false` hides the entire usage indicator, including its health
status, while leaving collection and Usage inspection active. The existing global and
per-provider collection opt-outs still prevent the indicator from surfacing an
ineligible provider. Routing-disabled providers remain eligible if they can collect.
Window policies govern numeric window entries, including vendor-rejected windows;
rejected status does not secretly bypass `never` or a numeric threshold. Preserve the
existing independent collector-failure signal as described below.

Schema validation rejects malformed policies, extra policy keys, booleans used as
numbers, nonfinite numbers, out-of-range numbers, and null policies. Runtime loading
must survive hand-edited invalid config: emit a path-specific diagnostic once per config
generation, ignore only the invalid override, and resolve its inherited policy. An
invalid global field falls back to its built-in default. Never reset unrelated valid
collection settings or provider overrides because one display policy is invalid.

### Window identity and truthful data

Use structured facts and explicit, tested provider identities to identify the default:

- A known account-wide window with a seven-day duration is weekly/all-model.
- Recognize Claude's exact `weekly` key with its known Claude product scope, and Grok's
  exact `included_weekly` with account scope, even when duration is absent in the cache.
- Recognize Claude `session` and known `weekly:<model-id>` keys for period display;
  model-specific windows remain model-specific. Existing Fable parsing supplies the key
  `weekly:claude-fable-5`; do not assume `weekly:fable` is a stored key.
- For Codex, use known account scope plus the observed weekly duration. Never assume
  every secondary slot is weekly or that a bucket mentioning a model is all-model.
- Prefer valid explicit duration when present; conflicting metadata must not silently
  receive the default classification. Use a documented small duration tolerance, not the
  time remaining until reset, to recognize a period.
- Unknown scope stays explicit as `scope?`. Product scope for unrelated plugins does not
  become all-model merely because it lacks model IDs. Free-form label heuristics may
  improve display text but must never change policy matching or scope semantics.

Preserve each window separately; never sum or average percentages across windows. Select
from observed windows, not the provider's summary or `attention.window_key`. No cached
window means no invented weekly window or fabricated percentage.

One explicit clock drives all derived state for each render. Recompute freshness and
whether reset has passed from cached timestamps using core policy, rather than trusting
flags frozen when the store was loaded. Keep using the established age thresholds.

For stale or unknown-age samples, retain their last reported percentage, append `~`, and
use neutral percentage styling. The tooltip says this is the last observation, with its
timestamp and age. A future reset still has a valid countdown. If reset has passed, show
`?% 0h0m↻`, explain "reset passed; awaiting a new observation", and retain last reported
capacity in the tooltip. Never infer a new 100% allowance or continue a negative
countdown. Keep visibility based on the last observed percentage and the configured
policy until the next observation, so an expired low entry does not silently disappear.
Fresh post-reset data clears that state normally.

A missing/nonfinite reset displays `?` in the time slot and says "reset time unknown" in
the tooltip. Do not manufacture a reset from observation time plus window duration.
Invalid numeric data never becomes a healthy colored percentage; isolate a malformed
entry and retain any legitimate provider health information.

### Countdown and percentage text

For a positive reset delta, use exactly two units without spaces:

- At least 24 hours: whole days and residual whole hours, e.g. `1d0h`, `3d4h`.
- Less than 24 hours: whole hours and residual whole minutes, e.g. `23h59m`, `2h9m`,
  `0h5m`.
- Use floor at the displayed precision; for a still-positive delta below one minute,
  show `0h1m`. Reserve `0h0m` for a passed reset.

Inject `now` for deterministic tests. Compute elapsed time from epoch timestamps, so
local timezone and DST do not alter arithmetic. The full local reset timestamp belongs
in the tooltip. Do not change the generic age/duration formatter used by unrelated
surfaces just to obtain this compact syntax.

Display whole remaining percentages by flooring a clamped value, except `0 < p < 1` uses
`<1%`, exactly zero uses `0%`, and full uses `100%`. Preserve the precise number and any
exceeded allowance in the tooltip. Match policies and choose colors using the unrounded
numeric value. Do not round 19.99% to an apparently non-triggering 20%.

### Appearance, ordering, and disclosure

Render one complete badge per selected window:

```text
<provider-icon> [<specifier> ]<remaining-percent> <reset-countdown>
```

Reuse `provider_emoji_badge` from the Agents tab's shared badge system: Claude `🎭`,
Codex `🤖`, Grok `🛰️`. Do not create a duplicate provider-to-icon table. For an unknown
plugin without an icon, use a short provider ID fallback with full identity disclosed in
the tooltip; cap it without conflating it with a known provider icon.

Weekly/all-model entries omit the specifier completely. Other examples are `5h/all`,
`wk/fable`, `mo/all`, and `Fast/5h/scope?`. Shorten model names only using known model
metadata/aliases and preserve distinctions; when shortening could collide, retain the
unique full model token. Keep unknown scope visible. A very long specifier makes a badge
less likely to fit; it never causes clipping of a partially displayed badge.

Use the normal top-bar background instead of a filled warning-colored pill for routine
capacity. Explicitly isolate usage styling when appending it after a colored routing
pill, so it cannot inherit that pill's warning background. Color and lightly bold the
percentage; use readable secondary foreground for the specifier and countdown. Separate
badges with two spaces. Healthy badges do not need `!`, the word `left`, or a bullet
between every field. Retain `!` for a displayed vendor rejection and an explicit `⚠` for
collector failure. Neither relies on color.

Show all selected badges that fit. Rank candidates by existing core attention severity
(rejected, very low, collector failure, low, ordinary), then provider ID, then
weekly/all-model first within a tie, then window key. Do not reorder every minute or
sort continuously by changing percentage. Selection remains independent of this rank.

Preserve collector-failure behavior: when a provider would currently produce the
`collection_problem` signal, retain it even if it has no selected numeric window. Add
`⚠` to its first displayed selected window when possible, otherwise show a standalone
`<icon> ⚠`. Include streak, failing-since time, reason, and last success in the tooltip.
Do not emit one duplicate collector warning for each window. Changing display policy
must not change the collector-failure threshold or routing.

Reserve the tab bar's natural text width (including padding) and the existing sibling
controls before allocating usage space. Additionally cap usage at half the top-bar
width, so a provider with many windows cannot dominate the row. Derive geometry from
stable content measurements, not the already-shrunk tab region. The usage feature must
not further squeeze navigation when existing controls already fill the row.

Pack a deterministic prefix of complete badges while reserving the correct overflow
count. Show `+N` for omitted display entries: each window counts once, and a standalone
collector warning counts once. An attached health mark does not count again. Recompute
the count and separator widths when choosing the prefix. If no badge fits, show a
count-only entry; if that cannot fit, show a single `…`, or nothing for a zero-cell
budget. Use Rich terminal-cell widths throughout, including emoji/variation selectors
and multi-digit counts. The final text must never exceed its budget, even at zero to
three cells. Never crop percentage, time, or half an emoji to make a badge fit.

The tooltip includes every selected window, including overflow: full provider and window
label, exact key, precise remaining capacity, full scope, effective policy, reset
countdown and timestamp, freshness, and any collector issue. Explain `~`, `↻`, and
overflow. Any rendered usage badge or overflow affordance opens Providers · Usage;
retain the existing command-palette/Launch Control access when there is no display
space. A healthy `always` window must be clickable, not just attention hints. Avoid
altering the existing routing-only click behavior when no usage entry exists.

### Ten-bucket capacity palette

Use ten distinct colors, bucket `min(9, floor(clamp(p, 0, 100) / 10))`; 100 belongs in
the final bucket. The first bucket is urgent red; then coral, orange, amber, yellow,
yellow-green, green, teal, cyan, and calm blue. The gradient conveys diminishing
capacity without treating a healthy 100% window as an alarm. Fresh numeric windows use
this scale; uncertain data uses neutral styling and its explicit state marker.

| Remaining | Dark theme | Light theme | Meaning                   |
| --------- | ---------- | ----------- | ------------------------- |
| 0–<10%    | `#FF5F6D`  | `#A22534`   | Nearly exhausted, red     |
| 10–<20%   | `#FF805F`  | `#A03620`   | Low, coral                |
| 20–<30%   | `#FFA552`  | `#8C480E`   | Limited, orange           |
| 30–<40%   | `#EBC04F`  | `#775800`   | Watchful, amber           |
| 40–<50%   | `#CED44C`  | `#5F6500`   | Midrange, yellow          |
| 50–<60%   | `#AADC64`  | `#456C1B`   | Comfortable, yellow-green |
| 60–<70%   | `#78DB8D`  | `#206F3C`   | Healthy, green            |
| 70–<80%   | `#4CD4B0`  | `#006E56`   | Ample, teal               |
| 80–<90%   | `#48CCD0`  | `#006C6C`   | Abundant, cyan            |
| 90–100%   | `#65C3ED`  | `#006381`   | Nearly full, blue         |

The proposed colors were calculated to exceed 4.5:1 against the canonical dark `#1E1E1E`
and light `#E0E0E0` surfaces (minimum approximately 5.64 and 4.66 respectively). Verify
against the actual chosen indicator surface in both themes. Use one presentation palette
helper, repaint on theme change, and preserve numeric/text meaning on terminals that
downsample colors. Do not add ten configurable color fields in this feature.

## Window policy — phase `window-policy`

1. Open `sase-core` using `/sase_repo` with a task-specific reason and use only the
   printed checkout path. Read its `AGENTS.md`. Work in its
   `crates/sase_core/src/provider_usage/` module, preferably a focused indicator-policy
   submodule rather than enlarging the main module.
2. Add typed policy settings and validation with normalized diagnostics. Add pure
   classification and selection/projection functions implementing the contract above.
   Inputs are cached public provider windows, eligible provider IDs, resolved policy,
   cadence, and injected `now`. Outputs retain provider/window identity, effective
   policy and reason, default-window classification, period/scope facts, raw remaining
   percentage, current sample/reset state, seconds until reset when known, and existing
   attention/health facts needed for deterministic ordering and disclosure.
3. Keep the display projection additive and separate from persisted observations and the
   existing public snapshot. Do not rewrite caches or change the generic
   applicability/model matcher. Add a versioned output contract for this new API; avoid
   an unnecessary existing store/public-schema version bump.
4. Export through `crates/sase_core_py` with both function registration and binding
   tests. Use the repository's established JSON/wire conversion, limits, and error
   patterns. Python must not duplicate policy resolution, weekly classification, or
   freshness/reset semantics as a fallback.
5. Test defaults, same-provider multiwindow selection, exact overrides and provider
   defaults, unrounded threshold boundaries, unknown future IDs, invalid policy
   diagnostics, and deterministic order. Include actual Claude product-shaped cached
   windows, Claude Fable, Codex shared and unknown buckets, Grok weekly/monthly, and
   synthetic third-party account windows. Exercise unchanged cached data at fresh,
   stale, unknown-age, and reset-crossing clocks.
6. Run `just check` from the core checkout root (or `./scripts/check.sh`), which
   includes the workspace and PyO3 tests. Never substitute only
   `cargo test -p sase_core`. Use `/sase_monitor` if this becomes long-running. Leave
   verified core changes for host-owned completion, with the new API contract available
   to the next phase.

Acceptance: core and binding tests prove the configured selected set and time-derived
states without filesystem or provider I/O; current usage summary/matcher APIs retain
existing behavior.

## Configuration and cache — phase `config-and-cache`

1. Reopen `sase-core` through the repo skill and consume phase one's API. Build/install
   the local binding via the SASE `just install`/Rust-install workflow. Follow the
   existing core revision-pin and dependency-floor process for coordinated landing: core
   changes must be available before SASE's pinned CI build calls new symbols. Update
   `sase-core-revision.txt` to a verified landed core revision through the existing
   ratchet workflow at integration time. Let release-plz own crate versions; never
   invent a future version or SHA. Ensure the published SASE dependency floor includes
   the binding when preparing the coordinated release.
2. Add thin Python dataclasses/adapters in `src/sase/llm_provider/usage/` and expose the
   necessary bindings through `_facade.py` and its normal export path. Extend
   `config.py` with separate indicator settings. Memoize normalized settings by the
   existing config generation/token and keep diagnostics bounded.
3. Add the documented policy shape to `src/sase/config/sase.schema.json`, using a
   reusable policy definition and open-ended provider/window-key mappings. Add actual
   defaults plus commented customization examples in `src/sase/default_config.yml`.
   Verify agreement among schema, runtime, and shipped defaults; distinguish the inner
   display `providers` map from collection's sibling `providers` map.
4. Extend `usage/peek.py` and the indicator's worker scheduling to capture immutable
   settings/eligibility alongside cached windows off the event loop. A config-token
   change must reload display settings and collection eligibility even if the usage file
   is unchanged. Preserve existing cache consumers through a thin compatible accessor
   where needed. Keep collection-disabled gates authoritative.
5. Provide a memory-only selection/projection path over that snapshot. The current
   30-second tick and resize reflow may call the bounded pure core function with a
   shared `now`; they must not read JSON, take the usage store lock, load config files,
   invoke a provider, or rebuild unrelated UI. Keep file/token probes time-gated and
   workers coalesced. Countdown/freshness updates alone must not reload the store.
6. Keep existing picker/header capacity hints on their established attention path;
   replace only the top-bar's provider-summary-based selection at phase-three
   integration. Preserve content-signature caching, and include style changes in that
   signature so bucket/theme transitions repaint even when the text is unchanged.
7. Update `docs/configuration.md` and `docs/llms.md` with complete defaults, precedence,
   three policy forms, IDs, strict remaining-percent semantics, invalid-config behavior,
   config reload, and display-versus-collection distinction.
8. Use `/sase_memory_write` before memory changes. The user explicitly authorized adding
   the glossary term. Create `sase/memory/glossary/usage-window.md` with keyword
   `usage-window`, title `Usage Window`, and the established strand frontmatter. Define
   it as a provider-reported capacity allowance over a time interval, scoped to all
   models or a model/product subset, independently identified and measured, with a reset
   time when known. Include Claude's five-hour all-model, weekly all-model, and weekly
   Fable examples. Explain that windows can overlap, percentages are independent, and
   the duration is distinct from time remaining to reset. Keep this a concise glossary
   entry, not a configuration manual. Run `sase memory init`; inspect generated glossary
   roster and instruction changes and never hand-edit the generated files.
9. Read `lint_and_test.md` through `/sase_memory_read` and run `just check`. Target
   configuration/schema/inventory tests and cache tests, including config changes
   without store writes, opt-out/re-enable, unknown provider keys, and error isolation.

Acceptance: a consumer can obtain the correct selected window records and live clock
state entirely from memory; real layered configuration is validated and discoverable;
the requested glossary term is published. The old top bar remains functional until phase
three switches it to the completed display.

## Compact display — phase `compact-display`

1. Replace the top-bar usage presentation path in `widgets/_provider_usage_indicator.py`
   and integrate it into `widgets/provider_disables_indicator.py`. Use focused modules
   for formatting, palette, and fitting if needed to respect repository size/lint
   limits. Consume the new typed records instead of requiring `CapacityHint` or provider
   `attention` for every window. Keep routing pill construction intact.
2. Thread one `now` through building text and tooltip. Implement the specified two-unit
   countdown, percentage display, status markers, provider icon reuse, default specifier
   omission, and ten-bucket palette. Use known model metadata for compact Fable labels
   without changing underlying IDs.
3. Replace the single-item disclosure ladder with whole-entry packing and the strict
   terminal-cell budget. Update the relevant `styles.tcss` rules and small tab-bar
   measurement helper if needed. Keep all layout calculations memory-only and coalesced;
   prevent resize oscillation and reuse unchanged rendered content.
4. Make click-to-Usage work for healthy always-visible records and overflow, preserve
   routing-only behavior, and build full tooltips for selected and collapsed items.
   Treat collector failures once per provider and preserve their complete disclosure.
5. Update `docs/ace.md` and any existing help-popup usage wording affected by this
   behavior. Include compact examples, `~`/`↻`/`?`, remaining percentage, weekly
   default, overflow count semantics, and how to open the complete Usage view.
   Cross-link the config documentation instead of duplicating the full schema. Do not
   change generic usage CLI output formats or other countdowns.
6. Update `tests/test_provider_usage_indicator_presentation.py`, relevant cases in
   `tests/test_provider_disables_indicator.py`, and applicable hint/cache tests. Add
   behavioral integration coverage for healthy weekly defaults, multiple windows from
   one provider, custom high thresholds, hidden windows, existing collector failures,
   and eligibility. Verify provider routing/picker regressions remain green.
7. Extend the dedicated provider usage visual tests under `tests/ace/tui/visual/`:
   normal 120/160-column multi-provider weekly entries; Claude's three windows; crowded
   80/60-column rows with existing controls; no-observation/unknown/reset states; and a
   ten-bucket palette scene in both dark and light themes. Freeze all clocks and use
   synthetic snapshots, never live subscriptions. Inspect actual PNGs, not only test
   exit codes, before accepting intentional goldens. Confirm entries are visibly smaller
   and quieter than the old warning pill.

Acceptance: the full product contract works in a real Textual app, with no partial badge
clipping, stale countdown, phantom recovered quota, hidden healthy click target, config
reload delay beyond the normal refresh cadence, or new provider I/O on display.

## Verification and landing criteria

In addition to each phase's checks, explicitly cover:

- 19.99/20/20.01 remaining against the default threshold, every exact decile, zero,
  positive subpercent, 100%, exceeded usage, malformed/nonfinite input, and all three
  policy forms across layered config. Assert 10 distinct styles for fresh buckets.
- Countdown deltas just below/at/above one minute, one hour, 24 hours, and reset;
  multi-day values and missing resets. Assert an unchanged store advances the text and
  freshness on later ticks and does not reset the percentage speculatively.
- Weekly/all-model scope omission for every supported collector shape; weekly Fable,
  monthly Grok, Codex unknown buckets, and third-party unknown scope retain specifiers.
- Width budgets from zero upward; variation-selector emoji; long IDs; one/two-digit
  overflow counts; provider warnings and routing controls; deterministic selection after
  resize. Assert final `Text.cell_len <= budget` and complete visible entries.
- No additional store read/network call on pure timer/resize renders, one coalesced
  reload on actual config/store changes, and no unnecessary `update()` when rendered
  text and spans are unchanged. Theme changes and bucket changes must update styles.

Run SASE `just check` after tracked changes and the focused visual lane with
`just test-visual -- tests/ace/tui/visual/test_ace_png_snapshots_provider_usage_indicator.py`
(or the final split files). Use `--sase-update-visual-snapshots` only after reviewing
intentional output. Run the core repository's full `just check` with its PyO3 lane.
Before landing the combined epic, run SASE `just check-full` through `/sase_monitor`
with the required TESTING/TESTED handoff, following the reference-memory instructions.
Record exact verification results and core-pin/release coordination in the completion
handoff. All implementation changes and opened-repo obligations use host-owned
finalization; agents do not create commits, branches, or PRs manually.
