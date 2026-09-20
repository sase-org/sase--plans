---
tier: epic
title: Muse Code subscription usage windows
goal: "SASE collects Muse Code's two subscription usage windows on the normal background
  cadence at zero model cost, and the TUI header shows Muse's weekly window by default
  whenever Muse is an eligible provider.

  "
phases:
  - id: core-normalizer
    title: Rust normalizer and weekly classification
    depends_on: []
    size: medium
    description: "core-normalizer: add the sase_core Muse subscription-usage normalizer,
      its PyO3 binding, and the indicator arm that classifies Muse's weekly window as a
      weekly all-model window.

      "
  - id: collector
    title: Free echo-mint usage probe
    depends_on:
      - core-normalizer
    size: medium
    description: "collector: add the Python MSP probe that mints a Muse usage
      observation with no model call, wire the provider usage hooks, and move the pinned
      core revision and sase-core-rs floor forward so the new binding is guaranteed
      present.

      "
  - id: indicator-default
    title: Default header indicator policy
    depends_on:
      - collector
    size: small
    description:
      "indicator-default: ship default config that shows only Muse's weekly window in
      the TUI header usage cluster, document it, and verify the rendered result."
proposed_by: bbugyi200.athena.0o6
create_time: 2026-09-20 12:41:06
status: wip
---

- **PROMPT:**
  [prompts/202609/muse_usage_windows.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/muse_usage_windows.md)

# Plan: Muse Code subscription usage windows

## Goal

Muse is the only registered provider with no subscription-usage hooks. This epic adds
them, using a probe that costs no tokens and no model turns, and makes Muse's weekly
window visible in the TUI header usage indicator by default when Muse is installed and
in use.

## Background

The design follows the consolidated research report
`research:202609/muse_subscription_usage_windows/muse_subscription_usage_windows.md`.
Its central findings were re-verified while authoring this plan against
`Muse Code 1.3.0 (1.3.0-R3401.1)`:

- `muse schema generate-json-schema --out DIR` emits `msp.schema.json` plus a
  `manifest.json` whose fingerprint is
  `sha256:7469c9e352e67def4a59df7e439984d7194fa351e1c8b7abb34060fd977ced81`, matching
  the report exactly.
- `usage/read` is on the stable, non-experimental method surface and is documented as
  answering "without a model call". Its result is `{usage?: SubscriptionUsage}` — the
  member is omitted, never null, when the host has observed nothing.
- `SubscriptionUsage` requires `observedAtMs`, `tier`, `weekly`, and `window`. There is
  no partial payload: when `usage` is present, both windows are present.
- `window` requires `windowDurationMins` (observed `300`); `weekly` deliberately omits a
  duration. `usedPercent` is an integer >= 0 and over-quota values above 100 are valid.
- `session/start` accepts `providerId` and returns a `Session` carrying `providerId` and
  `modelId`. `turn/start` requires `commandId` (UUIDv7), `sessionId`, and a non-empty
  `input` array, and acks with `disposition`, `status`, and `turnId`.
- `usage/changed` is a notification, edge-triggered on value change, and the
  absent-to-present first observation emits.

The behavioural finding this epic depends on is that a cold `muse serve` host reports
`{}` forever, and that `turn/start` against a session started with `providerId: "echo"`
causes the host to mint credentials and learn its usage roughly 2.5 s later, with
`modelId: null`, no `session/tokenUsage`, and no model call. That is an observed
behaviour, not a documented contract, which is why the probe guards it (see
"Echo-session guard" below).

The rest of the subscription-usage subsystem — store, staleness, backoff, attention,
`sase usage list`, the Providers / Usage modal, the header indicator, doctor — is
already provider-generic. Only a normalizer and a collector are genuinely new.

## Decisions

These are the judgement calls this plan makes. A reviewer who disagrees with one should
say so at the plan gate rather than leave it to the implementing agent.

1. **Weekly window carries no `duration_seconds`.** The vendor omits the weekly duration
   on purpose, and two samples 42 minutes apart carried identical anchored reset stamps,
   ruling out a rolling `now + 7d`. Fabricating `604800` would write a wrong number into
   the store. Classification is fixed in Rust instead (phase `core-normalizer`).
2. **Muse's 5-hour window is hidden by default.** The user asked for the weekly window
   "only". Default config therefore sets `session: never` for Muse rather than letting
   the generic `below_remaining_percent: 20` fallback surface the 5-hour block when it
   runs low. The cost is that a nearly-exhausted 5-hour block is not advertised in the
   header; it remains visible in the Providers / Usage modal and in `sase usage list`,
   and a user restores header attention with one config line. This differs from the
   Claude precedent in the same file, which pins one window `always` and leaves the rest
   on the generic fallback.
3. **`tier` does not become `observation.plan`.** The observed value is a 17-digit
   opaque account id, not a plan name, and persisting it would put a plausibly
   account-scoped identifier into the machine-local store. `plan` stays `None`.
4. **No feature flag.** The user-facing on/off is the existing generic config field
   `llm_provider.usage_metrics.providers.muse`, and each phase lands a complete,
   coherent surface. Per `sase/memory/sase_flags.md`, a `beta` flag is epic scaffolding
   for a phase that would otherwise expose an unfinished feature; no phase here does.
5. **The probe polls `usage/read` rather than awaiting `usage/changed`.** The shared
   `JsonLineSession` transport silently discards server notifications (`read_response`
   skips any frame carrying a `method`), so awaiting `usage/changed` would require
   changing transport shared with the Grok collector. Polling `usage/read` — measured at
   ~1 ms per call — after the turn ack reaches the same observation with no
   shared-transport change. This is a deliberate deviation from the research's preferred
   sequence, which listed polling as its own fallback.

## Phase 1: Rust normalizer and weekly classification

All of this lands in the `sase-core` repo. Open it with the `/sase_repo` skill; do not
edit it through any other path.

### Normalizer

Add `crates/sase_core/src/provider_usage/muse.rs`, modelled closely on the sibling
`grok.rs`: same versioned request wire, same `schema_version` check against
`PROVIDER_USAGE_OBSERVATION_SCHEMA_VERSION`, same `validate_usage_observation` exit.

```rust
pub struct ProviderUsageNormalizeMuseUsageRequestWire {
    pub schema_version: u32,
    pub payload: Value,      // the `usage/read` result object
    pub provider: String,
    pub context_id: String,
    pub account_generation: u64,
    pub request_started_at: f64,
    pub now: f64,
}

pub fn normalize_muse_usage(
    request: ProviderUsageNormalizeMuseUsageRequestWire,
) -> Result<ProviderUsageObservationWire>
```

`payload` is the whole `usage/read` result, so the normalizer — not Python — owns the
absent-`usage` decision.

One `SubscriptionUsage` produces one observation with two windows:

| Observation field  | 5-hour window                  | Weekly window                |
| ------------------ | ------------------------------ | ---------------------------- |
| `key`              | `session`                      | `weekly`                     |
| `label`            | `Muse 5-hour session`          | `Muse weekly all models`     |
| `used_percent`     | `window.usedPercent`           | `weekly.usedPercent`         |
| `resets_at`        | `window.resetsAtMs / 1000.0`   | `weekly.resetsAtMs / 1000.0` |
| `duration_seconds` | `windowDurationMins * 60.0`    | `None`                       |
| `period_start`     | `resets_at - duration_seconds` | `None`                       |
| `applicability`    | `Account`                      | `Account`                    |
| `observed_at`      | `observedAtMs / 1000.0`        | same                         |
| `source`           | `Probe`                        | same                         |
| `vendor_state`     | `Allowed`                      | same                         |

Envelope for the populated case: `outcome: Ok`, `completeness: Complete`,
`account_mode: Some("subscription")`, `plan: None`, `authoritative_empty: false`.

Rules the implementation must honour:

- **Never clamp `usedPercent`.** Values above 100 are valid vendor data. Confirm what
  `validate_used_percent` in `provider_usage/mod.rs` accepts before assuming a bound,
  and if it rejects over-100 values, resolve that there rather than by clamping in the
  Muse arm.
- **`Account` applicability, not `Product`.** `is_all_model_scope` returns `true`
  unconditionally for `Account`, while its `Product` arm is hard-coded to Claude. A
  `Product`-scoped Muse window would be silently excluded from all-model classification.
- **Absent `usage` is not an error.** A result object with no `usage` member becomes a
  windowless non-error observation: `outcome: Ok` with `authoritative_empty: true` and a
  `muse_usage_not_yet_observed` diagnostic. If it becomes `error`, three such ticks trip
  `USAGE_COLLECTOR_FAILING_THRESHOLD` and doctor reports Muse as broken. Under no
  circumstance may absence render as `0 %`.
- **Malformed vendor data** (a `usage` member that is not an object, or a required
  member missing or of the wrong type) becomes a structured error observation with a
  named diagnostic, matching how `grok.rs` handles `MISSING_CONFIG_DIAGNOSTIC`. A
  malformed binding _request_ still returns `ProviderUsageError::Validation`.

### Weekly classification

In `crates/sase_core/src/provider_usage/indicator.rs`, `is_weekly_window` resolves
duration first and otherwise falls through to a per-provider allowlist. Muse reports no
weekly duration, so add the Muse arm to that fallthrough:

```rust
|| (provider == "muse"
    && key == "weekly"
    && matches!(window.applicability, UsageApplicabilityWire::Account))
```

Without this the Muse weekly window misses the `weekly_all: always` policy, falls back
to the generic default, and stays invisible until the week is already 80 % spent. This
is core classification logic and belongs in Rust per the `rust_core_backend_boundary`
core memory; do not work around it by fabricating a duration.

### Exports and binding

- Re-export `normalize_muse_usage` and its request wire from `provider_usage/mod.rs` and
  from `crates/sase_core/src/lib.rs`, following the `normalize_grok_billing` precedent.
- Add `py_provider_usage_normalize_muse_usage` in `crates/sase_core_py/src/lib.rs`
  exposed as `provider_usage_normalize_muse_usage`, and add the matching line to that
  file's module-level binding inventory doc comment (near the existing
  `provider_usage_normalize_grok_billing` entry).

### Tests

Add unit tests beside the normalizer and, where the existing layout puts them, in
`provider_usage/tests.rs`. Cover at least:

- the verified live payload — `window`
  `{usedPercent: 0, windowDurationMins: 300, resetsAtMs: 1789935797000}`, `weekly`
  `{usedPercent: 0, resetsAtMs: 1789948800000}`, `tier` `"27681631238169137"`,
  `observedAtMs: 1789921255705` — asserting both windows, the exact `period_start`
  derivation, `duration_seconds: None` on weekly, and `plan: None`;
- `usedPercent` above 100 surviving unclamped;
- an absent `usage` member producing the authoritative-empty non-error observation;
- a malformed `usage` member producing a structured error observation;
- a wrong `schema_version` producing a validation error;
- `is_weekly_window` returning `true` for a Muse `weekly`/`Account` window and `false`
  for Muse's `session` window and for a `weekly` key under non-`Account` applicability.

Store fixtures under `crates/sase_core/src/provider_usage/fixtures` if that is how the
existing Grok fixtures are organized; follow whatever the directory already does.

### Verification and landing

Run `just check` from the `sase-core` repo root (it wraps `./scripts/check.sh all`). Per
that repo's `AGENTS.md`, never verify with `cargo test -p sase_core` alone — it excludes
the `sase_core_py` binding tests. Do not hand-edit any crate version; release-plz owns
versions and will publish a new `sase-core-rs` once this lands on master.

## Phase 2: Free echo-mint usage probe

This phase lands in the `sase` repo and consumes the phase 1 binding.

### Collector

Add `src/sase/llm_provider/usage/muse.py` exporting
`collect_muse_usage(context: UsageProbeContext, executable: str | None = None)`. Use
`src/sase/llm_provider/usage/grok.py` as the structural model: `JsonLineSession` for
transport, `ProbeStrategy` / `run_probe_strategies` for attempt handling,
`classify_probe_failure` for JSON-RPC error mapping, and `validated_status_observation`
for the non-`ok` paths.

Spawn:

```
muse serve --no-session-log --disable-shell
```

with `MUSE_NO_AUTO_UPDATE=1` in the child environment — the `~/.local/bin/muse` launcher
otherwise swaps the real binary hourly, and `src/sase/llm_provider/muse.py` already sets
this variable for runs. Resolve the executable the same way the provider does
(`context.executable` first, then the provider's own resolver, which honours
`SASE_MUSE_PATH`). Use `context.working_directory` as cwd, and keep the child killable
through the existing `JsonLineSession` teardown. Note that `probe.py` strips any
environment variable matching `KEY|TOKEN|SECRET|PASSWORD|CREDENTIAL|AUTH` from the
worker; the probe must not need any of them.

Sequence:

1. `initialize` with `clientInfo.name: "sase"` — the name must match `^[a-z0-9_]+$`.
2. Send the `initialized` **notification** (no `id`, no `params`). Skipping it makes
   every later call fail `-32600 {"kind": "notInitialized"}`.
3. Optionally compare `InitializeResult.schema.fingerprint` against the fingerprint
   recorded in this plan's Background. A mismatch is a **warning, proceed** condition,
   per the schema's own wording and because the launcher self-updates; never hard-pin.
4. `session/start` with a **UUIDv7** `commandId` (the server rejects UUIDv4),
   `workspaceRoot` set to the probe's scratch cwd, and `providerId: "echo"`.
5. **Echo-session guard.** Assert the returned `session.providerId == "echo"` and
   `session.modelId is None` _before_ sending any turn. If either fails, abort to a
   `vendor_drift` status and send nothing further. Without this guard, a future Muse
   that ignores or rejects `providerId: "echo"` would silently spend a real model turn
   on every refresh tick. Name this assumption in a comment so a future Muse bump is
   diagnosable.
6. `turn/start` with a fresh UUIDv7 `commandId`, the session id, a single minimal text
   part (`input: [{"type": "text", "text": "hi"}]`), and `reasoningEffort: "none"`. Read
   the ack and check its `status` / `disposition` — the research's single
   non-reproduction was a run that never checked the turn ack.
7. Poll `usage/read` until it returns a `usage` member or the deadline expires. The mint
   lands 2.4–2.9 s after `turn/start` and `usage/read` itself costs ~1 ms, so a short
   sleep between polls (on the order of 250 ms) is appropriate. Bound the whole probe
   well inside `USAGE_REFRESH_PROVIDER_DEADLINE_SECONDS` (20 s).
8. Hand the `usage/read` result to
   `require_rust_binding("provider_usage_normalize_muse_usage")` exactly as
   `grok.py::_observation_from_payload` does, and return the observation.

Issue no speculative methods on the probe connection. Keep the sequence minimal: the one
trial that failed to populate had interleaved an unknown method that errored `-32601`.

Status mapping:

- `FileNotFoundError` → `error` / `not_installed`.
- JSON-RPC `-32600` / `-32601` / `-32602` → `vendor_drift`, via
  `classify_probe_failure`.
- Deadline exhausted with `usage/read` still empty → the **absent** status, not an
  error. Use the same authoritative-empty shape phase 1 produces for an absent `usage`
  member, so a logged-out or never-minted host reports truthful absence instead of
  tripping the collector-health counter.
- Transport errors, including the transport's `unsolicited_request` guard firing on an
  approval-style server request, map to the existing transport status path.
- Map auth and transport failures onto the existing `unauthenticated` / `logged_out` /
  `api_mode` outcomes when the observed error text supports it. Logged-out behaviour was
  not verifiable without a destructive logout; do not invent a mapping for a shape you
  have not seen — leave it as absence.

### Provider hooks

In `src/sase/llm_provider/muse.py`, add alongside the existing hooks:

```python
@hookimpl
def llm_usage_capabilities(self) -> dict[str, object]:
    return {"probe": True, "passive_events": False}

@hookimpl
def llm_usage_probe(self, context: UsageProbeContext) -> dict[str, object] | None:
    from .usage.muse import collect_muse_usage

    return collect_muse_usage(
        context, executable=context.executable or _resolve_muse_executable()
    )
```

`{"probe": True}` is what enrols Muse in the background refresh loop:
`eligible_usage_providers()` skips any provider whose `usage_capabilities.probe` is not
`True`, then requires CLI readiness and either a model-alias reference or an explicit
per-provider enable. Because the probe is free, enrolling at the global 300 s cadence
costs about 2.7 s of wall clock per tick and nothing else. There is a single global
`refresh_seconds` and the per-provider map is boolean-only; no per-provider cadence
override exists and none is added here.

### Core version floor and pin

The `tools/check_sase_core_rs_bindings` gate verifies that the **minimum published**
`sase-core-rs` accepted by `pyproject.toml` exposes every binding `require_rust_binding`
names. Local `just install` builds the core from the sibling checkout, so local dev will
not notice skew — CI will. This phase must therefore:

- wait for phase 1 to be on `sase-core` master and released by release-plz;
- raise the `sase-core-rs>=…` floor in `pyproject.toml` to the published version that
  contains `provider_usage_normalize_muse_usage`, keeping the existing upper bound;
- move `sase-core-revision.txt` forward with `just ratchet-core-revision` (use `--check`
  or `--report-only` first; exit 0 means already current, 2 means a ratchet is pending,
  reported, or applied, 3 means it could not be determined safely).

### Tests

- Normalizer-boundary tests driving `collect_muse_usage` against a scripted fake `muse`
  executable (a stub script speaking JSON-lines on stdout), following whatever pattern
  the existing Grok collector tests use. Cover: the happy echo-mint path; a host that
  never populates before the deadline yielding absence rather than error; a
  `session/start` result with a non-echo `providerId` or a non-null `modelId` aborting
  to `vendor_drift` **without** a `turn/start` ever being sent; a missing executable;
  and a `-32601` from a method returning `vendor_drift`.
- A test asserting `MuseProvider.llm_usage_capabilities()` returns
  `{"probe": True, "passive_events": False}`.
- The guard test is the important one: assert on the absence of a `turn/start` frame,
  not merely on the returned status.

### Docs

Update `docs/configuration.md`'s `llm_provider.usage_metrics` section and the Muse entry
in `docs/agent_providers.md` to record that Muse collects subscription usage, how (a
local MSP probe against `muse serve`, no model call, no tokens), and that the
user-facing switch is `llm_provider.usage_metrics.providers.muse`. `CHANGELOG.md` is
release-please generated — do not hand-edit it.

## Phase 3: Default header indicator policy

The TUI header already docks the usage cluster to the top right: `UsageHeader`
(`src/sase/ace/tui/widgets/usage_header.py`) composes `ProviderUsageIndicator` with
`dock: right` and reserves a symmetric budget so the title stays centered. No layout or
widget change is needed to satisfy "top-right", and `usage_metrics.indicator.enabled`
already defaults to `true`. **Confirm this before changing anything, and change no
layout code if it holds.** This phase is configuration, documentation, and verification.

### Default config

In `src/sase/default_config.yml`, under
`llm_provider.usage_metrics.indicator.providers`, add the Muse entry next to the
existing `claude` one:

```yaml
providers:
  claude:
    windows:
      "weekly:claude-fable-5": always
  muse:
    windows:
      session: never
```

The weekly window needs no entry: phase 1 makes it classify as a weekly all-model
window, which the existing `weekly_all: always` policy selects. The explicit
`session: never` implements decision 2 — weekly only. Keep the neighbouring commented
examples accurate.

Mirror the new default into the `providers` node's `default` block in
`src/sase/config/sase.schema.json`; `tests/test_config_schema.py` validates
`default_config.yml` against the schema, and this repo's convention is that a schema
`default` matches what `default_config.yml` actually ships.

### Installed-only behaviour

"Only if Muse is installed" needs no new mechanism, but the implementing agent must
verify the chain rather than assume it:

- `cached_usage_indicator_projection` passes `eligible_providers` into the Rust
  projection, which drops any provider not in that set
  (`indicator.rs::provider_is_eligible`).
- `eligible_usage_providers()` requires `_provider_cli_ready`, which for Muse resolves
  `SASE_MUSE_PATH` or the provider's `autodetect_cli_name` and returns `False` when
  neither is a file nor on `PATH`.

So with no Muse binary there is no eligibility, no probe, no observation, and no header
entry. Note the gate is slightly stricter than "installed": a provider must also be
referenced by a model alias or explicitly enabled in `usage_metrics.providers`. That is
existing generic behaviour and is correct here — it means "installed and actually in
use". Add a test that pins the installed-only outcome so a future change to the
eligibility chain cannot silently start showing Muse on a machine without it.

### Config tests

Add coverage that the shipped defaults select Muse's weekly window and reject its
`session` window, at the config/projection level rather than by asserting on rendered
pixels.

### Render check

With a real observation present, confirm the Muse group renders correctly at 60, 80, and
140 columns in both light and dark themes, that the `♾️` Muse badge
(`src/sase/integrations/provider_badges.py`) appears, and that attention ordering treats
Muse like the other providers. Read `sase/memory/tui.md` with the `/sase_memory_read`
skill before touching anything TUI-adjacent. If rendered TUI output changes, run
`just fix-tui-screenshots` explicitly — `just check` does not run PNG snapshots — and
inspect the report and golden diffs before accepting them. Generation is not approval.

## Verification

Every phase that changes git-tracked files in the `sase` repo runs `just check` (prefer
`sase tool run check`) before finishing, per `sase/memory/lint_and_test.md`; run
`just install` first if the workspace's virtualenv is stale, and `just fix` before
handing a long run to a monitor. Do not run `just check-full` — no phase here names it.
Phase 1 runs `just check` in the `sase-core` repo instead, which is a different recipe
wrapping `./scripts/check.sh all`.

Phase 2 and phase 3 should also do one live exercise against the real Muse binary, which
is installed on this host: run a real refresh and confirm the observation lands with
both windows, a populated `resets_at`, and no token spend. `sase usage list` and the
Providers / Usage modal are the inspection surfaces.

## Risks

1. **Echo-mint durability.** The mint-on-echo-turn behaviour is observed, not
   contracted. If a future Muse defers the mint to provider dispatch, the probe degrades
   to `{}` and the collector reports absence — a safe failure, provided phases 1 and 2
   both handle absence as non-error. The `providerId` / `modelId` guard is what keeps
   the _unsafe_ failure (silently spending a real turn every tick) impossible.
2. **Schema drift.** The fingerprint churns on every Muse release because the launcher
   self-updates. Treat a mismatch as a warning; a hard pin would break on every update.
3. **Weekly reset interval unconfirmed.** Two samples agree on a UTC-midnight-aligned
   stamp but do not establish the interval. This is exactly why the weekly window
   carries no duration and is classified by the allowlist arm instead.
4. **Provider allowlists in `indicator.rs`.** This change adds a third `provider == "…"`
   string allowlist arm. That accumulation is worth a follow-up task; it is not a
   prerequisite for this work and must not expand this epic's scope.

## Out of scope

- Moving `MuseProvider.invoke` from `muse exec --json` to an MSP run transport for
  passive `usage/changed` collection and server-derived `session/tokenUsage`. The
  research makes a real case for it — it would retire `_muse_session_usage.py`'s
  session-log glob and its double-counting hazard — but it is a full session host with
  an approval flow to reconcile against SASE's sandbox posture. It is a separate epic,
  and the free probe removes usage windows as a reason to do it.
- Per-provider `refresh_seconds` overrides. Only needed if the probe ever regresses to a
  paid one.
- Hashing `tier` into the opaque auth fingerprint for tier-change detection.
- Tightening `muse.py::llm_default_usage_limit_config`'s unverified limit-wording
  patterns.
- Refactoring the accumulated per-provider allowlists in `indicator.rs`.
