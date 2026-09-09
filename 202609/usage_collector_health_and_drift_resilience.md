---
tier: epic
title: Usage collector health and vendor-drift resilience
goal: "Provider usage collection classifies and survives vendor CLI drift, exposes an
  honest per-collector health state derived from failure streaks, and ACE shows a calm,
  distinct visual indicator whenever a collector is failing consistently.

  "
phases:
  - id: health-core
    title: Collector health domain model in the Rust core
    depends_on: []
    size: medium
    description:
      "health-core: track failure-streak start in the refresh schedule, project a
      per-provider collector_health block into the public snapshot, gate and re-rank
      CollectionProblem attention on consistent failure, add the vendor_drift reason
      code, and coordinate the sase-core release plus the sase-side floor bump."
  - id: drift-probes
    title: Drift-classifying probe strategies for all collectors
    depends_on:
      - health-core
    size: medium
    description:
      "drift-probes: add a shared bounded probe-strategy runner with drift
      classification, adopt it in the Claude, Codex, and Grok collectors, annotate
      fallback recoveries on ok observations, and extend the live-mirroring fixtures to
      cover strategy chains and drift classification."
  - id: health-cli
    title: Collector health in the usage CLI and doctor
    depends_on:
      - health-core
    size: small
    description:
      "health-cli: render collector health in sase usage list status cells and verbose
      records, pass it through --json, add shared pure health label/style helpers, and
      flag consistently failing collectors in sase doctor."
  - id: health-tui
    title: Failing-collector indicator across ACE surfaces
    depends_on:
      - health-core
      - health-cli
    size: medium
    description:
      "health-tui: give the failing-collector state one visual identity (glyph, color,
      wording) and render it in the Providers Usage view rows and detail header, the
      top-bar usage indicator and tooltip, and picker/models-panel capacity hints, with
      updated visual goldens and no new refresh paths."
  - id: health-verify
    title: Integrated verification, live smoke, and docs
    depends_on:
      - health-core
      - drift-probes
      - health-cli
      - health-tui
    size: small
    description:
      "health-verify: run the combined acceptance checks including check-full through a
      monitor, perform a bounded live smoke plus an induced-failure walkthrough on an
      isolated SASE_HOME, and document collector health and the indicator."
proposed_by: bbugyi200.athena.0hd.f1
bead_id: sase-yz
create_time: 2026-09-09 19:52:53
status: wip
---

- **PROMPT:**
  [prompts/202609/usage_collector_health_and_drift_resilience.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/usage_collector_health_and_drift_resilience.md)
- **BEAD:**
  [sase-yz](https://github.com/sase-org/sase--beads/blob/main/pages/sase-yz/README.md)

# Usage collector health and vendor-drift resilience

## Outcome and scope

Two intertwined gaps remain after the sase-y5 subscription-capacity epic and the
probe-drift fix (`plan:202609/fix_claude_codex_usage_probe_drift.md`):

1. **Vendor drift is undetected and unclassified.** Probes speak empirical CLI contracts
   (Claude print-mode flags, Codex app-server JSON-RPC, Grok ACP). When a vendor
   tightens or changes that contract, every failure collapses into a generic
   `probe_failed`, and nothing distinguishes "the vendor changed the wire shape" from
   "the network hiccuped". The fix plan added bounded diagnostics; this epic adds
   classification and, where a legitimate alternate wire shape exists, bounded fallback.
2. **Consistent failure is invisible.** `consecutive_failures` and `last_success_at`
   exist only in the private Rust refresh schedule
   (`crates/sase_core/src/provider_usage/refresh.rs`); the public snapshot cannot
   distinguish one blip from a week of breakage. In ACE, `collection_problem` is the
   lowest-ranked attention kind, shares the `?` glyph with "unknown" and the yellow of
   "low", labels itself with a bare "usage", and a provider whose windows still carry
   stale numbers renders no failure text at all in the Usage view list. Passive Claude
   stream events mask a dead probe pipeline with fresh windows — exactly why the
   original breakage looked intermittent.

Ship: a per-provider `collector_health` block in the public usage snapshot derived from
probe-pipeline failure streaks; a `vendor_drift` reason code; a shared strategy/fallback
runner adopted by all three collectors; health-aware CLI and doctor output; and one
coherent failing-collector visual identity across every ACE usage surface.

Excluded (do not build): notifications or task beads on collector failure, automatic
provider disables or routing/admission changes based on health, feature flags (all
changes are additive default-on behavior with no old branch that must stay reachable),
vendor CLI version pinning or auto-update, multi-account handling changes, new CLI
subcommands, historical failure analytics, and any fallback that weakens a cost guard or
adds auth side effects. Health data must never change routing eligibility or round-robin
state.

## Grounding and preconditions

- The probe-drift fix (`plan:202609/fix_claude_codex_usage_probe_drift.md`) is
  implemented in a sibling lane but not yet landed on master. `drift-probes` builds
  directly on its collector changes (paramless Codex request with legacy-params retry,
  positive Claude probe budget, bounded diagnostics). If it has not landed when
  `drift-probes` starts, coordinate with the host before proceeding; do not reimplement
  or conflict with it.
- Shared domain behavior belongs in `sase-core`; there is no Python fallback. Open the
  core checkout with `sase repo open sase-core -r '<specific reason>'` and work only in
  the returned path. Start from `crates/sase_core/src/provider_usage/` (`mod.rs`
  snapshot wire + attention, `store.rs` persistence + projection, `refresh.rs`
  schedule/backoff).
- Python seams: `src/sase/llm_provider/usage/` — `_facade.py`/`_wire.py` (binding
  surface), `probe.py` (isolated worker runtime), `codex_collector.py`, `claude.py` +
  `_claude_support_*.py`, `grok.py`, `refresh_runner.py` (records refresh attempts with
  the observation outcome), `presentation.py` (shared status labels used by CLI and
  TUI), `hints.py`/`peek.py` (capacity hints + lock-free peek cache).
- ACE seams: `src/sase/ace/tui/modals/models_panel_usage_modal.py` /
  `models_panel_usage_rendering.py` (Providers · Usage view),
  `src/sase/ace/tui/widgets/provider_disables_indicator.py` +
  `_provider_usage_indicator.py` (top bar), `model_picker_rows.py` /
  `model_picker_options.py` / `models_panel_rendering_descriptions.py` (hints).
- Before phase work, read the mandated memory notes with `/sase_memory_read`:
  `lint_and_test.md` (every phase), `tui_perf.md` (health-tui), `cli_rules.md`
  (health-cli, only if an option is added — the design adds none), `symvision.md` (on
  symbol-lint failures). No phase creates task beads; record discovered work as
  `PROPOSED FOLLOW-UP:` notes on the phase bead.

## Design

### Collector health model (Rust)

Health describes the **probe/refresh pipeline**, not window freshness. It derives from
refresh-schedule state, which only probe refresh attempts update
(`record_provider_usage_refresh_attempt` via `refresh_runner._finish_job`); passive
stream-event observations keep windows fresh but must not mask a dead pipeline.

- Extend `ProviderUsageRefreshScheduleWire` with `first_failure_at: Option<f64>`
  (`#[serde(default)]` so existing state files decode). Set it when
  `consecutive_failures` transitions 0 to 1; clear it on success. Verify that an old
  binding reading a newer state file degrades through the existing corruption-isolation
  path rather than failing every read, and note the observed behavior in the phase bead.
- Project an additive `collector_health` object into each public snapshot provider row
  by joining the schedule for that provider's current
  `(context_id, account_generation)`:
  `{"state": "ok" | "degraded" | "failing", "consecutive_failures": <u32>, "last_success_at": <ts|null>, "failing_since": <ts|null>}`
  — `failing` at or above a named `USAGE_COLLECTOR_FAILING_THRESHOLD: u32 = 3` constant,
  `degraded` for one or two consecutive failures, `ok` otherwise; the whole block is
  `null` when no schedule exists for the current generation (never attempted).
  `failing_since` echoes `first_failure_at` only while a streak is active. Additions
  stay within public schema version 1 (additive-within-version is the established
  contract); Python's `_wire.py` passes the snapshot through opaquely, so no exact-field
  updates are needed there.
- Re-derive attention from consistency, not single blips. `CollectionProblem` fires only
  when (a) `collector_health.state == "failing"`, or (b) a problem outcome exists with
  zero cached windows (the user would otherwise see nothing at all), or (c) the existing
  silent-failure heuristic (ok outcome, no summary, only unknown-freshness windows). A
  single failed attempt with cached numeric data no longer trips attention. Re-rank
  `UsageAttentionKind` to `Rejected > VeryLow > CollectionProblem > Low > None`: a
  consistently blind collector undermines every other number, so it outranks a mere
  low-capacity warning, while true capacity constraints stay on top. Update `rank()`,
  the ordering tests, and mirror the new order in Python `hints.py::_ATTENTION_RANK`
  (owned by health-tui; coordinate the constant, do not edit that file in this phase).
- Add `vendor_drift` to the observation reason-code validation (additive within
  observation schema v1): the provider CLI actively rejected our request shape
  (unknown/invalid flag or option value, invalid params, unsupported method) — as
  distinct from `timeout`, auth states, and `parse_error`.
- Tests in `provider_usage/tests.rs`: streak start/clear across success, failure runs,
  generation advance, and mixed contexts; health classification at the boundaries (0, 1,
  2, 3, saturation); schedule-to-row join misses (no schedule, stale generation);
  attention gating for all three trigger arms and a blip-with-cached-data non-trigger;
  rank ordering; `vendor_drift` accepted and unknown codes still rejected; old-state
  decode with the field absent.
- Release and floor: release-plz owns core versions — never hand-bump Cargo versions.
  After the core change lands and a release exists, raise the sase repo's `sase-core-rs`
  floor in `pyproject.toml` (currently `>=0.32.54,<0.33.0`) to the release that carries
  the new capabilities, update the lock, extend `usage/types.py`'s `UsageReasonCode`
  literal with `vendor_drift`, and confirm `tools/probe_core_floor` and the binding
  capability checks agree. Core-side verification is the core checkout's own
  `just check` (cargo test alone is insufficient); the sase-side bump runs SASE
  `just check`.

### Drift-classifying probe strategies (Python)

- New small module `src/sase/llm_provider/usage/_strategy.py`: a `ProbeStrategy` record
  (slug name plus attempt callable) and a runner that executes an ordered strategy list
  inside the probe's existing single deadline and transport bounds. Rules: try the next
  strategy only when the failure is drift-shaped; stop immediately on terminal outcomes
  (unauthenticated, not installed, timeout, deadline); never spawn beyond existing
  transport primitives. A shared `classify_probe_failure(...)` helper maps vendor
  rejection markers to `vendor_drift`: JSON-RPC `-32600`/`-32601`/`-32602`, CLI
  option/argument errors (for example `error: option`/`unknown option` on stderr with a
  nonzero exit), and ACP method-not-found. Everything else keeps its current reason
  code.
- When a non-primary strategy succeeds, the resulting `ok` observation carries a bounded
  diagnostic naming what drifted and what recovered it (for example
  `primary no_params failed (-32602); recovered via legacy_params`), and the collector
  logs one WARNING. This makes silently auto-healed drift visible in
  `sase usage list -v` without failing collection. Safety invariants, stated and tested:
  no strategy may drop or weaken the Claude cost guard (`--max-budget-usd` stays;
  zero-cost markers stay authoritative), add authentication side effects, or exceed the
  probe deadline.
- Codex (`codex_collector.py`): restructure the fix plan's inline retry into declared
  strategies `no_params` then `legacy_params` for `account/rateLimits/read`; classify
  RPC shape rejections as `vendor_drift`.
- Claude (`claude.py` + `_claude_support_command.py`): single strategy (no argv guessing
  — the cost guard forbids speculative flag fallbacks), but classify CLI option/flag
  rejection as `vendor_drift` with the bounded stderr diagnostic; preflight marker or
  version mismatches stay `unsupported_cli_version`.
- Grok (`grok.py`): classify ACP extension-method rejection as `vendor_drift`; preserve
  current probe behavior otherwise.
- `refresh_runner.py` needs no flow change: `vendor_drift` observations carry
  `outcome=error`, so schedules still count them as failures and health accrues.
- Tests: extend the strict live-mirroring fixtures
  (`tests/llm_provider/fixtures/usage_probe/`) and provider tests to cover the strategy
  chain order, drift classification per provider, the recovered-via-fallback diagnostic
  on ok observations, terminal-outcome short circuits, deadline sharing across
  strategies, and a fixture comment noting which live CLI version each strict mode
  mirrors. Keep the fixtures' env-driven-mode conventions.

### CLI and doctor (Python)

- `presentation.py` gains pure helpers `collector_health_label()` and
  `collector_health_style()` shared by CLI and TUI. Table `Status` cells for a non-null
  unhealthy block read like `failing · vendor drift · 5x` (label, reason family, streak
  count); healthy providers keep today's labels. Verbose plain records add `health=`,
  `consecutive_failures=`, `last_success=`, and `failing_since=` fields; relative ages
  use the existing `duration_label`/`timestamp_label` helpers. `--json` output already
  passes the snapshot through — assert `collector_health` survives filtering in
  `_filtered_usage_snapshot`.
- `sase doctor` (`src/sase/doctor/checks_providers.py`): a consistently failing
  collector is reported with its streak, last-success age, and bounded diagnostic;
  degraded is informational; no probes are run (doctor stays offline).
- No new subcommands or options. If implementation reveals a genuine need for one, stop
  and read `cli_rules.md` first.
- Tests: status-cell and verbose rendering for ok/degraded/failing/null blocks, JSON
  passthrough, doctor classification, and non-TTY/no-color output.

### ACE failing-collector indicator (TUI)

One visual identity everywhere: glyph `⚠`, style `bold #FFAF5F` (the established
"unavailable" orange — distinct from capacity yellow `#FFAF00`, capacity red `#FF5F5F`,
and the `?` unknown glyph), and the word `failing` wherever width allows so color is
never the only carrier of meaning. Do not use `▲` (reserved as the agent-list attention
arrow).

- `hints.py` (owned here): mirror the new attention rank order; give
  `collection_problem` its own marker `⚠` and style, and replace the bare `usage` label
  with `usage failing` plus the provider name where the surface already names providers.
  `peek.py` stays memory-only.
- Providers · Usage view: list rows append a ` ⚠ failing` badge even when a numeric
  remaining meter renders (closes the stale-but-numeric silent gap); the detail header
  gains a `Collector:` line (for example
  `Collector: failing — 5 failures since 2d · last success 3d ago`) under `Status:` when
  health is not ok, alongside the existing yellow diagnostic line; the footer legend
  explains `⚠`. Degraded (1–2 failures) renders only in the detail header, never as a
  row badge — the row badge is reserved for consistent failure.
- Top bar (`provider_disables_indicator.py` + `_provider_usage_indicator.py`): the usage
  attention pill uses `⚠` and `<PROVIDER> usage failing` at the normal disclosure tier,
  keeps the existing narrower tiers' rollup counts, and the tooltip adds
  failing-since/last-success lines while keeping the click-through to the Usage view.
  Because attention now requires consistency, the pill appears less often and means more
  when it does.
- Picker and models-panel hints render the `⚠ usage failing` hint through the shared
  `capacity_hint_marker`/`capacity_hint_style` path.
- Performance: pure rendering changes only. All data flows through the existing peek
  cache, snapshot workers, and change tokens; add no timers, no refresh paths, and no
  UI-thread disk reads (tui_perf rules 5 and 8). Update the existing visual goldens
  (`test_ace_png_snapshots_provider_usage_indicator.py`,
  `test_ace_png_snapshots_model_picker_usage.py`, and the usage-modal snapshots at
  120/80/60 columns) plus interaction tests; inspect actual/expected/diff artifacts
  before accepting new goldens.

## Phase execution and acceptance

1. **health-core** owns `crates/sase_core/src/provider_usage/` in the sase-core checkout
   plus the sase-side floor bump (`pyproject.toml`, lock, `usage/types.py` reason-code
   literal). Acceptance: Rust tests above pass; core `just check` passes in the core
   checkout; after release, sase `just check` passes with the raised floor; the live
   snapshot on this host shows `collector_health` blocks; no Python domain logic added.
2. **drift-probes** owns `_strategy.py`, the three collector modules, `probe.py`
   diagnostics touchpoints, and their fixtures/tests. Acceptance: strategy and
   classification tests pass; targeted suites
   (`tests/llm_provider/test_codex_usage_probe.py`, `test_claude_usage.py`,
   `test_grok_usage_probe.py`, `test_usage_probe.py`) pass; a live `run_usage_probe` for
   all three providers on this host reports `ok`; safety invariants have explicit tests;
   SASE `just check` passes.
3. **health-cli** owns `presentation.py` additions and doctor checks with their tests.
   Acceptance: rendering/doctor tests pass; `sase usage list`, `-v`, and `--json` show
   health on a synthesized failing store without probing; SASE `just check` passes.
4. **health-tui** owns `hints.py`, the usage modal/rendering modules, the top-bar
   indicator modules, and picker/description hint call sites, with interaction and
   visual tests. Acceptance: goldens reviewed and updated; j/k and open/close
   responsiveness unchanged per existing TUI instrumentation; no new refresh code paths
   introduced; SASE `just check` passes.
5. **health-verify** owns integrated verification and documentation (`docs/llms.md`,
   `docs/ace.md`: what collector health means, how the indicator behaves, that health
   never affects routing). Acceptance: full `just check-full` through `/sase_monitor`
   with required TESTING/TESTED statuses; a bounded live smoke on this host
   (`sase usage refresh` then `list -v`; never starts an inference turn; no private
   readings recorded in fixtures); an induced-failure walkthrough in an isolated
   `$SASE_HOME` with a deliberately broken fake provider CLI demonstrating degraded then
   failing states, the CLI output, and the ACE badge; honest recording of anything not
   verified.

Shared-file ownership prevents merge contention: only health-core touches sase-core;
only drift-probes touches collectors; only health-cli touches `presentation.py` and
doctor; only health-tui touches `hints.py` and ACE modules. Cross-phase constants
(attention rank order, the `⚠`/color identity, threshold value) are defined once in
health-core (Rust) and consumed, not redefined, elsewhere.

## Final verification and release criteria

Done means: a vendor wire-shape change on any of the three providers produces a
`vendor_drift`-classified observation (or an auto-recovered ok with a visible fallback
diagnostic) instead of a generic `probe_failed`; three consecutive probe failures
surface as `failing` in the snapshot, the CLI, doctor, and every ACE usage surface with
one consistent `⚠` identity; a single blip with cached data stays quiet everywhere
except detail views; passive stream events can no longer mask a dead probe pipeline; and
all documented checks pass with host-owned commits, releases, and landing through the
normal finalizer workflow.
