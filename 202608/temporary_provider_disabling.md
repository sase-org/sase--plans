---
tier: epic
title: Temporarily Disable LLM Providers
goal:
  Let users temporarily disable one or more registered LLM providers from the ACE Models
  panel, with durable machine-wide expiry state, routing semantics that remove disabled
  providers from alias pools and ordered fallbacks, preserved-but-suspended alias
  overrides, actionable failures for direct explicit requests, and a polished
  provider-management/countdown experience that stays responsive and visible.
phases:
  - id: provider-disable-core
    title: Add the Rust-owned temporary provider-disable state contract
    depends_on: []
    size: medium
    description:
      "provider-disable-core: add a versioned, lock-bounded, atomic multi-provider
      disable store and PyO3 bindings in the linked sase-core repo, then add the strict
      Python facade and lock-free display peek in sase. Cover concurrent entries,
      replacement, exact expiry, until-cleared state, partial corruption cleanup, and
      binding parity."
  - id: routing-semantics
    title: Make every model-selection path honor disabled providers
    depends_on:
      - provider-disable-core
    size: medium
    description:
      "routing-semantics: introduce one routing-availability seam and apply it to alias
      pools, ordered fallbacks, temporary alias overrides, default autodetection,
      provider dispatch, model pickers, and completion catalogs. Preserve
      direct-selection diagnostics, selector cursor invariants, and automatic
      restoration on expiry."
  - id: models-panel-ux
    title: Build the Provider Routing experience in the Models panel
    depends_on:
      - routing-semantics
    size: medium
    description:
      "models-panel-ux: add a provider-routing modal, duration and exact-time flows,
      live countdown/status rendering, background refreshes, affected-alias
      presentation, a top-bar disabled-provider pill, keyboard help, focused interaction
      tests, and PNG snapshots at normal and narrow terminal sizes."
  - id: integration-and-docs
    title: Document, stress, and land the combined provider-disable feature
    depends_on:
      - models-panel-ux
    size: small
    description:
      "integration-and-docs: reconcile cross-phase behavior, update the
      LLM/ACE/Rust-backend docs, run the full sase-core and sase verification lanes
      including visual snapshots, and exercise expiry, fallback, direct-request, and
      multi-process state transitions in an end-to-end smoke matrix before landing."
proposed_by: bbugyi200.athena.02f
bead_id: sase-mc
create_time: 2026-09-09 19:51:50
status: wip
---

- **PROMPT:**
  [prompts/202608/temporary_provider_disabling.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/temporary_provider_disabling.md)
- **BEAD:**
  [sase-mc](https://github.com/sase-org/sase--beads/blob/main/pages/sase-mc/README.md)

# Plan: Temporarily Disable LLM Providers

## Why this is an epic

This feature crosses three boundaries that should be owned and verified separately:

1. a shared, machine-wide, multi-process state contract in `sase-core`;
2. provider/model routing policy used by launches, alias previews, pickers, and
   completions; and
3. a stateful Textual workflow with countdowns, background writes, top-bar visibility,
   and pixel-level layout requirements.

Each boundary is substantial but bounded. Keeping them as dependent medium phases lets
one agent establish a stable wire/state API before another changes launch routing, then
lets a UI-focused agent build against settled semantics. A final small phase owns the
combined verification and documentation rather than asking an earlier phase to predict
the final tree.

## Product contract

### User-facing behavior

- The Models panel keeps its existing alias list and gains fixed `p = Providers` in its
  context footer. `p` opens a **Provider Routing** modal listing every user-facing
  registered provider using the existing provider colors.
- Each provider row distinguishes:
  - **available**: registered and its declared CLI is present (or no CLI is required);
  - **CLI unavailable**: already excluded from automatic selector routing because the
    declared CLI is missing; and
  - **disabled · <time> left**: explicitly disabled by the user, regardless of whether
    its CLI is currently installed.
- Hidden/test-only providers that opt out of model pickers remain absent from this
  human-facing manager. The underlying state API stays provider-name-generic.
- On an enabled row, `d` or `enter` opens a provider-specific duration picker. It
  retains the familiar `15m`, `30m`, `1h`, `2h`, `4h`, **until cleared**, **until a
  specific time**, and custom-duration paths used by model alias overrides. Copy says
  “Disable CLAUDE” / “Route new launches around CLAUDE,” not “model override.”
- On a disabled row, `d`/`enter` replaces its duration and `x` enables it immediately.
  On any enabled row, `x` is a harmless warning rather than a state mutation.
- The exact-time modal is generalized just enough to accept feature-specific title and
  submit copy while retaining the timezone/DST parsing and Back behavior already used by
  overrides.
- A successful write reports both the duration and the effect, for example:
  `CLAUDE disabled for 1h; alias routing refreshed.` Clearing reports
  `CLAUDE enabled for new launches.` Failure to acquire/read/write state is an error
  toast and leaves the current UI snapshot unchanged.
- The Models title gains a conditional, compact disabled-provider line only while at
  least one disable is active, such as
  `disabled providers: CLAUDE 42m · GROK until cleared`. The ordinary no-disable title
  stays as calm as it is today.
- ACE gains a compact coral/amber top-bar pill parallel to the gold default-override and
  violet alias-override pills:
  - one provider: `CLAUDE off 42m`;
  - several: `CLAUDE +2` (stable alphabetical order);
  - none: zero width. Its tooltip lists every provider and expiry and points to `,m`,
    and clicking it opens the Models panel.
- The provider manager and pill explain that already-running provider processes are not
  killed. The disable applies to new launches, follow-ups, and later retry/fallback
  resolution.

### Routing precedence and edge cases

Temporary provider disabling is an availability layer, not a mutation of plugin
registration or user configuration:

1. Raw registry metadata and registered-provider discovery remain intact so the Models
   panel can show and re-enable the provider, doctor can identify it, and configuration
   diagnostics do not misclassify a temporary state as an unknown plugin.
2. One routing-availability predicate combines raw registration, physical CLI
   availability, and the captured active-disable set. Alias selector resolution reads
   the active set once per top-level operation rather than once per member.
3. Round-robin pools skip disabled members without selecting or invoking them. The
   existing cursor advances from the member actually selected; disabling a member does
   not rewrite membership/fingerprints, and re-enabling naturally brings it back into
   future rotations.
4. Ordered fallbacks select the first physically available, non-disabled member. When a
   higher-priority provider returns, it becomes the winner again on the next resolution.
5. When every selector member is unavailable, preserve today's fail-diagnostic policy:
   retain member zero rather than silently rerouting its model through a default
   provider. Dispatch then raises an actionable temporary-disable or ordinary
   provider-unavailable error.
6. A stored per-alias temporary model override whose resolved provider is disabled is
   **preserved but suspended**. Resolution falls through to that alias's configured or
   implicit target, including its pool/fallback. If the provider disable expires or is
   cleared while the alias override is still active, the override resumes automatically;
   neither timer rewrites the other state file.
7. The alias view distinguishes a stored override from an applied override. Suspended
   rows render an amber state such as `override paused · CLAUDE disabled`; selector
   details show the live fallback/pool winner rather than claiming the selector itself
   is suspended. Bucket counts and descriptions use applied-vs-paused semantics
   consistently.
8. `@default` follows the same rule; remove the special-case shortcut that would let a
   disabled provider in a temporary default override bypass alias resolution. Its
   configured/shipped fallback is allowed to run while the override is paused.
9. Provider autodetection ignores disabled providers. An explicitly configured
   `llm_provider.provider` remains configuration, but it cannot bypass the dispatch
   gate; alias-backed defaults can still route around it through their selectors.
10. A direct explicit request (`%model:claude/opus`, an explicit `provider_name`, or
    `SASE_LLM_EXEC_PROVIDER=claude`) does not silently change providers. It fails before
    provider creation with a dedicated diagnostic naming the provider and formatted
    expiry. Bare known models retain their raw provider identity for the same reason.
11. Model pickers and completion catalogs omit concrete models/provider scopes from
    currently disabled providers. Alias entries remain visible and show their current
    effective alternate target. Free-form explicit input is validated at submission and
    receives the same actionable disabled-provider feedback.

### Storage and wire contract

- Shared state belongs in `../sase-core/crates/sase_core`, following the existing
  `effort_override.rs` and `runner_limit_override.rs` patterns rather than adding
  another Python-owned mutating store.
- Use a dedicated state file under the resolved SASE home (proposed name
  `llm_provider_disables.json`) and a dedicated lock. The schema is versioned and holds
  a map/list of independent provider records with: `provider`, `created_at`,
  `expires_at`, and `source`.
- Multiple providers can be disabled concurrently. Setting one provider replaces only
  that provider's record; clearing one preserves all others. All list/snapshot output is
  deterministically sorted by provider name.
- Relative duration `None` means until cleared. Finite durations must be positive; exact
  expiries must be finite and strictly later than the captured creation time. Expiry is
  exclusive at `now >= expires_at`, matching the existing override stores.
- Mutations hold a process-shared exclusive lock for the complete read/modify/write
  cycle and persist with temp-file + flush + fsync + atomic replace. Lock wait is
  bounded, following the core's 250 ms precedent.
- Authoritative reads prune expired and malformed records independently, preserving
  valid siblings and deleting the file when empty. A malformed envelope/version fails
  closed to no disables and self-cleans; a valid steady-state read performs no rewrite.
- Expose schema-version, get-active-snapshot, set-relative, set-until, and clear-one
  PyO3 bindings. Add a strict Python dataclass facade under
  `sase.llm_provider.provider_disable` that rejects stale or malformed binding payloads.
- Keystroke/display callers use a read-only, time-gated, mtime/size-keyed peek cache
  parallel to `temporary_override_peek.py`. It never locks, prunes, rewrites, prompts,
  or invokes provider code. Launches and write workflows use the authoritative Rust
  facade.

## Non-goals

- No permanent `disabled_providers` configuration key or config-edit workflow.
- No new CLI command in this epic; the reusable state/routing APIs intentionally make a
  future CLI or web surface possible.
- No automatic disabling based on rate limits, auth failures, health checks, or retry
  telemetry.
- No cancellation or signalling of provider processes that are already running.
- No change to plugin installation, provider entry points, retry policy definitions, or
  model-alias selector grammar.
- No migration or merging of alias overrides, effort overrides, runner limits, and
  provider disables into one omnibus state file.
- No memory-file changes. Documentation updates are limited to checked-in product and
  architecture docs.

## Cross-phase rules

- Before touching the linked core repo, run
  `sase repo open sase-core -r "Implement temporary LLM provider disabling"` and use
  only the printed path. Do not assume a particular numbered workspace path.
- Start each phase by checking both relevant worktrees and preserve unrelated changes.
- In the sase repo, run `just install` before focused tests or verification. Every phase
  that changes sase files runs `just check` before handoff.
- In sase-core, never edit release-plz-owned versions or path-dependency pins. Run the
  core repo's `just check`, which includes workspace and PyO3 binding tests.
- Slow state I/O, provider availability recomputation, and alias-view rebuilding never
  run in a Textual render/action/timer callback. Use threaded workers; timer callbacks
  only compare captured deadlines and schedule guarded work. Re-capture selected row
  identity before applying worker results and preserve it across refresh.
- Selector resolution captures one active-disable snapshot per operation. Do not put a
  state-file read behind the process-lifetime `provider_cli_available()` cache and do
  not read state once per selector member.
- Provider-disable state is machine-global and wall-clock based. Tests isolate
  `SASE_HOME`, pin clocks, and clear any process-local peek/registry caches.
- Epic phase workers do not create task beads. Record discoveries as
  `PROPOSED FOLLOW-UP:` notes on the phase bead.

---

## Phase `provider-disable-core`: Rust state contract and Python facade

### Core implementation

Open `sase-core` through `sase repo open` and add a focused provider-disable module to
`crates/sase_core/src/`:

- Define stable wire records for one disable and an ordered active snapshot, a schema
  version constant, state/lock filename constants, and a typed error enum separating
  validation, bounded lock timeout, I/O, and serialization failures.
- Implement:
  - authoritative `get` at an injected Unix timestamp;
  - set/replace by relative duration;
  - set/replace until an exact timestamp; and
  - clear one provider idempotently.
- Keep the full map update under one lock. On read, validate records independently so a
  damaged `claude` row cannot erase a valid `codex` row. Canonical rewrites sort keys,
  prune expired/invalid entries, and include a final newline.
- Add module exports in `crates/sase_core/src/lib.rs`.
- Bind the schema version and four operations in `crates/sase_core_py/src/lib.rs`,
  document them in the binding inventory, map validation to `ValueError` and state
  failures to `RuntimeError`, and add them to module registration.

### Python facade and peek

- Add `src/sase/llm_provider/provider_disable.py` with:
  - `TemporaryProviderDisable` and strict snapshot rehydration;
  - `get_active_provider_disables(now=None)`;
  - `get_active_provider_disable(provider, now=None)`;
  - `disable_provider(provider, duration_seconds, source, now=None)`;
  - `disable_provider_until(provider, expires_at, source, now=None)`; and
  - `enable_provider(provider)`.
- Validate UI/API provider ids against raw registered provider names before mutation,
  but keep wire parsing independent of current registration so an uninstall does not
  make state unreadable/unclearable.
- Add a lock-free `provider_disable_peek.py` cache for high-frequency presentation
  paths. It parses only the stable on-disk wire, filters expiry on every call, and
  degrades to an empty mapping on any read/parse problem.
- Export the supported facade from `sase.llm_provider.__init__`, isolate it in the test
  runtime fixture, and extend Rust-binding health/parity validation if the current
  validator enumerates public bindings.
- Add the new operation to `docs/rust_backend.md`'s Rust-owned state inventory in the
  final docs phase; do not document the UI before it exists.

### Tests and acceptance

Rust unit/PyO3 tests must cover:

- two providers set concurrently and returned in deterministic order;
- replacing one duration without changing its sibling;
- clear-one and idempotent missing clear;
- exact boundary expiry and until-cleared persistence;
- invalid provider/source/duration/expiry/clock values;
- malformed envelope deletion and per-entry malformed/expired pruning;
- no rewrite for a canonical active file;
- lock timeout bounded well below two seconds; and
- binding wire keys, Python exception mapping, and round-trip parity.

Python tests must cover strict payload shape/version/type validation, `SASE_HOME`
redirection, public facade calls, and the peek cache's mtime, time-floor, expiry,
missing, corrupt, and read-only behavior.

Verification:

```bash
# in the opened sase-core checkout
just check

# in sase
just install
pytest tests/test_provider_disable.py tests/llm_provider/test_provider_disable_peek.py
just check
```

Handoff: the next phase consumes only the Python facade/dataclass and does not know the
state filename, lock, or JSON layout.

---

## Phase `routing-semantics`: Effective provider availability everywhere

### Central availability seam

- In `src/sase/llm_provider/registry.py`, keep raw plugin discovery APIs raw and add a
  routing predicate/snapshot helper that combines: raw registration,
  `provider_cli_available()`, and an injected/captured disabled set.
- Add a typed `ProviderTemporarilyDisabledError` (or equally explicit domain error)
  carrying the active record. `get_provider()` checks it before plugin creation so an
  explicit `provider_name`, `SASE_LLM_EXEC_PROVIDER`, stale resolved selection, or
  caller bypass cannot invoke the disabled plugin.
- Format errors with `until cleared` or a timezone-neutral remaining/expiry value that
  remains useful outside ACE. TUI-specific local clock copy stays in the UI layer.
- Autodetection filters disabled candidates; raw provider/model/color/advisory metadata
  remains available for management and diagnostics.

### Alias resolution and view semantics

- Thread one captured disable snapshot through `model_alias_resolution.py`'s top-level
  resolution, recursive selector members, `resolved_target_is_available()`, and
  `model_alias_selector_details()`.
- Preserve the existing monkeypatch/test seam by making availability snapshot injection
  explicit rather than hiding a new disk read in each call.
- For pools/fallbacks, availability is false when a member's provider is disabled.
  Preserve the all-false selector diagnostic behavior and round-robin fingerprint/state
  format.
- When a temporary alias override points to a disabled provider, mark it suspended and
  continue through configured/implicit alias resolution. Carry enough metadata out of
  resolution to distinguish:
  - no stored override;
  - stored and currently applied override; and
  - stored but provider-disabled override.
- Update `AliasView`/`BucketView` builders and completion projections to consume an
  injected disable snapshot. A paused override retains its record/countdown but the
  effective provider/model, pool availability count, selected fallback, and bucket model
  summary come from live underlying resolution.
- Route `@default` through the same resolver; remove duplicate shortcuts in
  `temporary_override_defaults.py`, `registry.py`, and `launch_selection.py` that would
  force a disabled provider from the default override.
- A direct concrete provider/model or bare known model keeps its raw provider identity.
  It never silently turns into an unknown model on another provider; the dispatch gate
  supplies the disable diagnostic.

### Pickers and completions

- Parameterize `build_model_rows()` with the panel's captured disable snapshot and omit
  disabled providers' concrete model groups. Keep aliases visible with rerouted
  effective targets.
- Apply a cheap live provider-disable overlay to the cached `%model` catalog:
  - remove concrete model and provider-scope entries owned by disabled providers;
  - rebuild/overlay alias target metadata from the injected snapshot; and
  - keep the static registry/config catalog cache independent of wall-clock state.
- ACE directive completion passes `peek_active_provider_disables()` alongside its
  existing alias-override peek. The LSP launch payload captures an authoritative
  provider-disable snapshot once at materialization time; it is a launch snapshot, not a
  cross-process live subscription.
- Validate free-form custom targets just before an override/edit submission. A disabled
  explicit provider receives a warning/error and is not stored as a new alias override
  through the normal UI.

### Tests and acceptance matrix

Add focused tests proving:

- round-robin skips disabled members, advances from the actual winner exactly once, and
  admits a provider again without changing the selector fingerprint;
- ordered fallback chooses the next member and immediately restores priority after
  clear/expiry;
- all members disabled retains member zero and produces the specific dispatch error;
- a temporary alias override pauses, reveals its configured selector/fallback, and
  resumes only while its own expiry remains active;
- `@default` has the same pause/fallback/resume behavior;
- direct `%model`, explicit `provider_name`, and execution-provider env override cannot
  bypass the gate;
- autodetection skips disabled candidates while explicit configuration remains visible
  and cannot execute through `get_provider()`;
- picker/completion concrete entries disappear, provider scopes stop matching, aliases
  remain and show effective alternatives, and cache reuse does not serve stale disable
  overlays;
- physical CLI missing + disabled, unknown provider, hidden provider, and unaffected
  provider cases stay distinct; and
- display/preview (`consume=False`) never advances pool state.

Relevant test homes include:

- `tests/llm_provider/test_load_balanced_aliases.py`
- `tests/llm_provider/test_ordered_fallback_aliases.py`
- `tests/llm_provider/test_alias_overrides.py`
- `tests/llm_provider/test_registry_resolution.py`
- `tests/test_xprompt_model_completion_catalog.py`
- `tests/test_models_panel_selector_builder.py`
- new provider-disable routing/dispatch modules as needed.

Verification:

```bash
just install
pytest tests/llm_provider/test_load_balanced_aliases.py \
       tests/llm_provider/test_ordered_fallback_aliases.py \
       tests/llm_provider/test_registry_resolution.py \
       tests/test_xprompt_model_completion_catalog.py
just check
```

Handoff: expose a pure provider-routing view snapshot for the UI containing raw provider
name, model count, physical CLI availability, active disable, and any lightweight
affected-alias metadata. The UI must not reconstruct routing policy.

---

## Phase `models-panel-ux`: Provider Routing workflow and visibility

### Models panel integration

- Add fixed `p` binding/action to `ModelsPanel`; keep `ctrl+p` as Previous. Extend every
  alias/bucket footer variant with `p=Providers` without dropping current actions.
- Add a focused mixin/module (for example `models_panel_providers.py`) rather than
  growing `models_panel.py`. The facade owns dependency indirections and worker handles
  so tests keep stable monkeypatch points.
- Load provider routing snapshots and all state mutations through threaded workers.
  Provider rows and alias views are rendered from immutable in-memory snapshots.
- The periodic 5-second clock callback only formats captured countdowns and detects a
  crossed expiry. A crossed deadline schedules one coalesced worker to reload state and
  alias views. The completion handler re-captures the selected row/provider and
  preserves cursor identity; it never rebuilds from a stale pre-worker selection.
- Closing is blocked while a provider write is active, matching other Models-panel
  override lanes. Cancel provider workers on unmount.
- Extend `ModelsPanelResult` so the caller can distinguish routing changes when needed.
  On provider changes, invalidate/re-resolve `LLMOverrideIndicator`'s cached default and
  refresh the default, alias, and disabled-provider pills.

### Provider Routing modal

- Create a compact centered modal with a provider-colored `OptionList`, stable
  alphabetical ordering, a short explanation that this affects new launches, and a
  fixed-height description strip to avoid layout jumps.
- Suggested row grid:
  `PROVIDER    <model count>    <available | CLI missing | disabled · 42m left>`. Use
  provider plugin colors for identity; use green for available, dim neutral for physical
  CLI absence, and coral/amber for user-disabled state. Do not use color alone: every
  state has text.
- Footer: `d/enter=Disable or change duration  x=Enable  j/k=Navigate  esc=Back`.
- Description copy for an active row states that new launches and fallbacks route around
  it, existing provider processes continue, and the exact end time/duration. CLI-missing
  copy explains that selector routing already excludes it.
- After successful disable/enable, refresh the modal row, the Models title line, and
  alias rows from one returned snapshot. Do not close the provider manager, so the user
  can manage several providers in one session.

### Shared duration/time presentation

- Refactor `models_panel_duration.py` so the same result types/parser/presets can render
  feature-specific title and subtitle copy. Existing alias, effort, and runner-limit
  snapshots must remain unchanged unless intentional copy improvements are explicitly
  accepted in their own updated goldens.
- Parameterize `OverrideUntilModal`'s title and submit label while preserving its DOM
  ids, time parser, DST ambiguity/nonexistent-time errors, preview, and Back sentinel.
- Provider disable notifications use the captured result type to distinguish relative,
  until-cleared, and exact-time copy exactly as alias overrides do.

### Alias impact rendering

- Add a clear paused-override state style and description. A stored override targeting a
  disabled provider does not paint as the active effective target; the row shows the
  fallback target and explains when the override can resume.
- Pool/fallback descriptions mark disabled members unavailable and keep the selected
  arrow on the live alternate member. Bucket override counts distinguish active and
  paused overrides if both are surfaced.
- The conditional Models title summary is compact, deterministic, countdown-aware, and
  disappears after expiry without a synchronous state read.

### Top-bar pill

- Add `ProviderDisablesIndicator` beside the two existing model override indicators in
  `_app_layout.py`, styles, lazy widget exports/stubs, and leader-mode refresh handling.
- Reuse/generalize `_override_pill.py` formatting instead of duplicating countdown and
  tooltip grammar. Add a disable-lane palette with accessible dark foreground contrast.
- Poll via the lock-free peek at the established 30-second top-bar cadence; filter
  expiry every render. Clicking opens the Models panel.

### Interaction, performance, and visual acceptance

Add tests for:

- `p`/footer/keymap behavior at top level and inside buckets;
- provider row ordering, hidden provider omission, all three availability states, and no
  cursor jump on programmatic refresh;
- disable, replace duration, exact time, Back, cancel, custom duration, until cleared,
  enable, idempotent enable, and worker error flows;
- closing/unmount during writes and stale worker completion protection;
- local countdown-only ticks versus one guarded expiry reload;
- default-indicator cache invalidation and all top-bar indicator refreshes;
- paused alias override rendering and pool/fallback winner descriptions;
- provider pill single/multiple/expired/tooltips/click behavior; and
- 120x40 plus narrow 70x32 geometry with footer, descriptions, and focused rows visible.

Add deterministic PNG snapshots for at least:

1. Provider Routing with mixed available/CLI-missing states;
2. Provider Routing with one active disable;
3. provider-specific duration picker;
4. Models panel with disabled-provider title and aliases rerouted;
5. a paused alias override falling through to a pool/fallback;
6. single and multiple top-bar disable pills; and
7. the provider manager at narrow width.

Run `just test-visual`, inspect actual/expected/diff artifacts for every changed golden,
and accept only intentional pixels. Run the TUI navigation benchmark or focused
`SASE_TUI_PERF=1` exercise if the implementation changes the existing j/k render path;
the target remains p95 under 16 ms.

Verification:

```bash
just install
pytest tests/test_models_panel_provider* tests/test_provider_disables_indicator.py \
       tests/test_models_panel_navigation.py tests/test_models_panel_keymaps.py
just test-visual
just check
```

---

## Phase `integration-and-docs`: Combined contract and landing

### Documentation

Update:

- `docs/ace.md`: `p=Providers`, provider modal states/actions, duration flow, title
  summary, top-bar pill, and already-running-process boundary;
- `docs/llms.md`: storage/precedence, temporary override suspension, pool/fallback
  availability, direct explicit failure, expiry restoration, and examples;
- `docs/configuration.md`: clarify that the feature is temporary runtime state and does
  not edit `llm_provider.provider` or alias configuration;
- `docs/rust_backend.md`: list the Rust-owned provider-disable state contract/bindings;
  and
- `src/sase/default_config.yml` comments only if needed to keep pool/fallback
  availability documentation accurate. Do not add a config key.

Include a concise scenario table:

| Request                  | Disabled provider present? | Result                                             |
| ------------------------ | -------------------------- | -------------------------------------------------- |
| round-robin alias        | one member                 | next available member; cursor advances from winner |
| ordered fallback         | preferred member           | next available candidate                           |
| temporary alias override | override target            | override pauses; underlying alias resolves         |
| direct provider/model    | target provider            | actionable failure; no silent provider change      |
| every selector member    | all                        | member zero retained for diagnostic; launch fails  |
| running provider process | disabled after start       | process continues; future resolution changes       |

### Final verification and smoke matrix

- Re-run core `just check` on the exact linked-core tree consumed by sase.
- Run the sase exhaustive lane only through `/sase_monitor`, with a `--next` action that
  inspects the result and fixes failures:

  ```bash
  sase monitor start --command 'just check-full' --label provider-disable-check-full \
    --next '@medium_worker inspect the completed provider-disable full-check monitor, fix any failures, rerun the appropriate verification, and finish the phase'
  ```

- Run `just test-visual` and inspect all provider-disable goldens.
- In isolated `SASE_HOME` state, exercise public APIs across fresh Python processes:
  disable Claude and Grok concurrently, confirm `@smarter`/`@smartest` route to Codex,
  confirm a temporary Claude alias override is paused, clear Claude and observe it
  re-enter fallback priority/rotation, then cross exact expiry and verify self-cleanup.
- Exercise a direct disabled-provider launch far enough to prove no provider subprocess
  is created and the error includes provider plus expiry context; use mocks/fakey-safe
  harnesses rather than real paid provider calls.
- Recheck `git status --short` in both repos, confirm no generated memory files or
  release-plz-owned versions changed, and land only after both verification suites are
  green.

## Definition of done

- A user can disable and re-enable multiple providers entirely from the Models panel,
  with relative, exact-time, custom, and until-cleared durations.
- Every new launch path either routes an alias around disabled providers or fails
  explicitly when the user directly requested one; no path invokes a disabled plugin.
- Alias overrides pause/resume predictably, selectors retain cursor/fallback invariants,
  and expiry restores providers without restart or config edits.
- Active global state is visible in the Models panel and top bar, including countdowns
  and multiple-provider summaries.
- The state store is multi-process safe, atomic, bounded under contention,
  self-cleaning, and owned/tested in Rust with strict Python bindings.
- No state-file I/O or provider recomputation runs in TUI render, keypress, or serial
  timer callbacks.
- Focused, full, and visual verification pass in both repositories, and docs describe
  the exact precedence and already-running-process boundary.
