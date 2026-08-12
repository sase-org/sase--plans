---
tier: tale
title: Fix the agent_runners axe startup failure and close the core-capability gate gap
goal:
  sase axe starts again with the agent_runners chop guard, the declared sase-core-rs
  floor can validate every guard provider this build advertises, and both the core-floor
  gates and the runtime error detect a stale sase_core_rs binding on their own.
size: medium
proposed_by: bbugyi200.athena.yz
create_time: 2026-08-12 17:28:09
status: wip
---

# Plan: Fix the `agent_runners` axe startup failure and close the core-capability gate gap

## Summary

`sase axe` cannot start on the primary host: fail-closed axe configuration validation
rejects the `agent_runners` chop guard that the host config now uses, because the
installed `sase_core_rs` binding predates the sase-core release that taught the config
validator about that guard provider. Ratchet the declared `sase-core-rs` floor to the
release that supports the guard, teach the core-floor gates to detect _behavioral_
capability gaps (not just missing binding names), and make the runtime error say that
the Rust core binding is stale so the next occurrence diagnoses itself.

## Symptom

`sase axe` fails to start / restart. ACE shows a red `Axe restart failed` notification
with:

```
Could not load axe configuration for restart: Invalid axe configuration:
- axe.lumberjacks.ci_watch.chops.ci_watch.inhibit_if.agent_runners
  (source: overlay:sase_athena.yml:<host>/.config/sase/sase_athena.yml):
  unknown guard provider `agent_runners`; supported providers: patch, agent_hood, agent_clan
```

The message is produced by `src/sase/axe/_process_restart.py:65` after
`load_axe_config()` (`src/sase/axe/config.py:97`) raises `AxeConfigError`
(`src/sase/axe/_config_types.py:42`) for the diagnostics returned by the Rust
`validate_axe_config` binding.

## Root cause

Three changes from the recent `sase-ko` epic landed within twelve minutes of each other
and left the running host install one core release behind what its own configuration
required:

1. **sase-core `a0a6ca4` — "feat(axe-chop): add the agent_runners inhibit_if guard
   provider"**, released as **v0.26.6** (PyPI upload `2026-08-12T20:39:13Z`). This is
   the commit that taught the _config validator_
   (`crates/sase_core/src/axe_chop/config.rs`) about `agent_runners`. At tag `v0.26.5`
   that validator emits exactly the observed string,
   `supported providers: patch, agent_hood, agent_clan`.
2. **sase `7e8f528b2` — "feat(axe): add agent_runners chop guard preflight"**
   (2026-08-12 16:38) added host-side snapshot support (`src/sase/axe/chop_policy.py`),
   JSON-schema support (`src/sase/config/sase.schema.json`), docs, and tests — but left
   the declared dependency floor at `sase-core-rs>=0.26.5,<0.27.0` in `pyproject.toml`.
   `tools/ratchet_core_window --report-only` still reports a pending `0.26.5 -> 0.26.6`
   ratchet.
3. **chezmoi `54de26c3` — "feat(axe): idle-gate bugyi_chop_ci_watch on the agent_runners
   guard"** (2026-08-12 16:50) added `inhibit_if: {agent_runners: {max: 0}}` to the
   `ci_watch` chop in the host overlay `sase_athena.yml`.

The host `sase` is a `uv tool` **dev (editable)** install, so `sase_core_rs` is the
extension built from the host's own sase-core checkout rather than a published wheel.
That build predates `a0a6ca4`: running `validate_axe_config` under the tool
environment's interpreter with the `ci_watch` guard returns

```
{'code': 'unknown_guard_provider', 'path': 'axe.lumberjacks.ci_watch.chops[0].inhibit_if.agent_runners', ...}
```

while the same call in a workspace whose core build includes `a0a6ca4` returns no
diagnostics. So the outage is core-binding skew: config (chezmoi) adopted a v0.26.6-only
capability while the running install still carried a pre-v0.26.6 core build. Validation
is fail-closed by design, so one unknown guard provider stops the whole daemon.

## Why every existing gate stayed green

- **The Python test suite never exercises the failing surface.** `agent_runners` already
  existed in the core _decision_ engine before v0.26.6; only the _validator_ gained it
  in `a0a6ca4`. `tests/test_axe_chop_preflight_policy.py` drives
  `evaluate_chop_decision`, which returns a correct `skip`
  (`inhibited by 1 agent holding runner slots ...`) even on a v0.26.5 build.
  `tests/test_config_schema_automation.py` validates against the bundled JSON schema,
  not against the Rust validator. Both pass with a core that cannot load the config.
- **The advisory floor probe cannot see this class of gap.** `tools/probe_core_floor`
  (run by `just check` / `just check-full`) installs the declared floor wheel into a
  scratch venv and runs `tools/check_sase_core_rs_bindings` plus
  `tools/validate_sase_core_rs`, then extracts _binding names_ from their output.
  `validate_axe_config` already existed at 0.26.5, so the probe reports
  `{"declared_floor": "0.26.5", "status": "ok"}`. Nothing probes behavior _inside_ an
  existing binding.
- **The published-floor CI job only runs on the release branch.**
  `.github/workflows/ci.yml` gates `release-core-floor-smoke` on
  `github.event.pull_request.head.ref == 'release-please--branches--master'`. Feature
  lanes build core from the sase-core checkout, so the declared floor is never exercised
  when the feature lands.
- **Version numbers cannot detect it**, exactly as `tools/validate_sase_core_rs`'s
  module docstring already warns: between releases the Cargo workspace version does not
  move, so only behavior probes can distinguish two cores.

## Goal

1. Make the declared floor able to validate every guard provider this build advertises.
2. Give the core-floor gates a behavior-level probe so the next capability adopted from
   an unreleased/newer core is caught by `just check` and by the release-floor CI job
   instead of by a dead daemon.
3. Make the runtime failure self-diagnosing: say that the installed `sase_core_rs` is
   older than this sase build and name the remedy.

## Non-goals

- Do not relax fail-closed axe configuration validation. Silently dropping an unknown
  guard would let a guarded chop fire unguarded; failing the load is correct.
- Do not change the `agent_runners` semantics, the chop decision engine, or anything in
  the sase-core repo. Core already ships the capability in v0.26.6.
- Do not edit the host overlay (`sase_athena.yml`, chezmoi) as part of this work; the
  config is valid for a current core.
- No sase memory (`sase/memory/*.md`, `AGENTS.md`, provider shims) edits.

## Implementation steps

### 1. Ratchet the published `sase-core-rs` floor to 0.26.6

```bash
just install
just ratchet-core-window            # exits 2 when it applies the ratchet
```

Confirm `pyproject.toml` now declares `sase-core-rs>=0.26.6,<0.27.0` and that `uv.lock`
records `sase-core-rs` 0.26.6 (both the `specifier` entry and the resolved package
version/hashes). Re-run `tools/ratchet_core_window --report-only` and expect exit 0 with
no pending diff. Do not hand-edit `uv.lock`; if the tool leaves it stale, run `uv lock`.

This is a real fix, not bookkeeping: `just install`'s `_setup` treats "checkout ahead of
the published window" as normal precisely because the window is expected to be ratcheted
once the needed release exists — v0.26.6 is published, so the window is simply overdue.

### 2. Add a behavioral guard-provider capability probe to `tools/validate_sase_core_rs`

`tools/validate_sase_core_rs` is the right home: it already probes behavior (e.g.
`_validate_artifact_ref_schemas`, `_validate_skill_reference_contract`,
`_validate_project_lifecycle_contract`), it runs against the _installed_ binding during
`just install` (`tools/validate_test_environment` dispatches it as `core-bindings`), it
runs against the _floor wheel_ inside `tools/probe_core_floor`, and it runs in the
`release-core-floor-smoke` CI job. Its test file
(`tests/test_validate_sase_core_rs_tool.py`) is already in
`tests/contract_manifest.txt`, so no contract-set re-curation is needed.

Add a validator, e.g. `_validate_axe_chop_guard_providers(module)`, that:

- Reads the advertised guard providers from the bundled schema
  `src/sase/config/sase.schema.json` — the `inhibit_if` tagged-array `provider` enum
  (currently `patch`, `changespec`, `agent_hood`, `agent_clan`, `agent_runners`).
- Holds a small provider → minimal-valid-payload table (`patch`/`changespec` →
  `{"name_prefix": "x"}`, `agent_hood` → `{"hood": "h"}`, `agent_clan` →
  `{"name_prefix": "x"}`, `agent_runners` → `{"max": 0}`) and **fails if the schema
  advertises a provider the table does not cover**, so a future provider cannot be added
  to the schema without extending this probe.
- For each provider, calls the extension directly (mirroring the request shape in
  `src/sase/core/axe_chop_facade.py:135`):
  `module.validate_axe_config({"schema_version": module.chop_engine_schema_version(), "config": {"axe": {"lumberjacks": {<one lumberjack with one chop carrying that guard>}}}, "provenance": {}, "require_descriptions": False, "require_description_shape": False})`
  and fails when the returned diagnostics are non-empty.
- Degrades gracefully when `validate_axe_config` or `chop_engine_schema_version` is
  missing entirely (report the capability gap; do not raise `AttributeError`).
- Emits its failure lines in the shape `tools/probe_core_floor` already parses, so the
  advisory probe can name and date the gap. `_VALIDATE_BINDING_RE`
  (`r"\] (?P<name>[A-Za-z_][A-Za-z0-9_]*)\("`) matches a line such as
  `[axe-chop-guard] agent_runners(...) rejected by the installed core`. Confirm the
  extracted capability name is the provider name (`agent_runners`), because
  `_diagnose_capability` then runs `git log -S<name> -- crates/sase_core_py/src/lib.rs`
  in the sase-core checkout; for `agent_runners` that resolves to `a0a6ca4`, whose
  containing release tag is `v0.26.6`, which yields the actionable `stale_actionable`
  verdict rather than `could_not_determine`.

Register the new validator in `main()` alongside the existing checks so it runs in every
mode that already runs them.

Open the sase-core checkout with the `/sase_repo` skill if you need to confirm the core
history or the validator source; do not clone or web-fetch it another way.

### 3. Cover the new probe in tests

Extend `tests/test_validate_sase_core_rs_tool.py` (contract-marked, per the file's
existing convention) with:

- a pass case: a fake module whose `validate_axe_config` returns `[]` for every
  advertised provider;
- a fail case: a fake module that returns an `unknown_guard_provider` diagnostic for
  `agent_runners`, asserting a non-zero result and that the emitted line names
  `agent_runners` in the shape `tools/probe_core_floor` extracts;
- a drift case: a schema stub advertising an unknown provider, asserting the probe fails
  rather than silently skipping it.

Add a focused test asserting the `_VALIDATE_BINDING_RE`/`_extract_capabilities` contract
still holds for the new line shape — either in the same file or in
`tests/test_probe_core_floor_tool.py` — so the two scripts cannot drift apart silently.

### 4. Make the runtime error name the stale binding

In `src/sase/axe/_config_types.py`, have `AxeConfigError.__init__` append one hint line
when the core rejected something this build advertises. Every raise site
(`src/sase/axe/config.py:97` and the five in `src/sase/axe/_config_targets.py`) funnels
through this constructor, so one change covers `sase axe start`, restart, `sase doctor`,
and the ACE notification.

- Trigger only for diagnostics whose `code` is `unknown_guard_provider` or
  `unknown_trigger_provider`, and only when the rejected token (parse it from the
  diagnostic `path` tail or the backticked name in `message`) is advertised by the
  bundled schema resolved through `sase.config.inventory`
  (`src/sase/config/inventory.py:50`).
- Hint text must name the mismatch and the remedy, e.g.:
  `hint: this sase build advertises guard provider 'agent_runners', but the installed sase_core_rs (<version>) rejects it — the Rust core binding is older than this sase build; run 'sase update' (dev installs) or reinstall sase to rebuild sase_core_rs.`
  Read the installed version through `importlib.metadata` and omit it if unavailable.
- Deliberately do **not** compare the installed core version against the declared floor:
  between core releases the Cargo version does not move, and the floor legitimately lags
  master, so a version comparison both misses real skew and fires falsely. The
  schema-advertises-but-core-rejects signal is exact.
- Cache the schema read (module-level `lru_cache`) and fall back to no hint on any
  read/parse failure. The hint must never turn a config error into a crash.
- Cover it in the existing axe config test module(s): hint present for an advertised
  provider, absent for a genuinely bogus provider name, and absent when the schema
  cannot be read.

## Verification

- `just install` then `just check-full`. Use the full lane, not `just check`: this
  change touches `pyproject.toml`, `uv.lock`, `tools/`, and CI-adjacent contracts.
- Prove the new probe actually catches the regression, against the _old_ wheel:

  ```bash
  uv venv /tmp/core-floor-0265
  uv pip install --python /tmp/core-floor-0265/bin/python sase-core-rs==0.26.5
  /tmp/core-floor-0265/bin/python tools/validate_sase_core_rs   # must fail, naming agent_runners
  ```

  Then confirm it passes on the new floor (`sase-core-rs==0.26.6`) in the same way.

- Run the advisory floor probe and record the verdict:
  `.venv/bin/python tools/probe_core_floor --json` — expect `status: ok` with
  `declared_floor: 0.26.6`. (The probe caches by a fingerprint that hashes
  `tools/validate_sase_core_rs`, so editing that script busts the cache automatically.)
- Sanity-check the end-to-end config path with the real host-shaped guard: load a config
  containing `inhibit_if: {agent_runners: {max: 0}}` through `sase.axe.config` in the
  workspace venv and assert no diagnostics.
- Confirm no other advertised capability is missing from the new floor: the probe run
  above passing is the assertion.

## Operator recovery (host action — the implementing agent must not run it)

Restoring axe on the host is a separate, user-run step; `sase update` swaps the code the
live ACE/axe process is running from, so the implementing agent must not trigger it.

- Preferred: run `sase update` on the host so the dev checkouts fast-forward and
  `sase_core_rs` is rebuilt from sase-core ≥ v0.26.6, then start axe (`sase axe start`)
  and confirm the `ci_watch` lumberjack loads.
- Fallback (instant unblock, loses the guard): temporarily remove the
  `inhibit_if: {agent_runners: {max: 0}}` block from the `ci_watch` chop in the chezmoi
  overlay and re-apply, then restore it after the install is updated.

Neither step is required for this plan's code changes to be correct or verifiable.

## Risks

- The ratchet moves the floor for published-wheel installs; 0.26.6 is a patch release
  inside the existing `<0.27.0` window, so no API break is expected. `just check-full`
  plus the release-floor probe cover it.
- The new probe adds one more failure mode to `just install` in workspaces whose linked
  sase-core checkout is stale. That is the intended behavior (it is exactly the skew
  that caused this outage) and `sase repo open sase-core` is the documented remedy
  already printed by `_setup`.
- The runtime hint runs on an error path only; keep it exception-safe so a malformed or
  missing schema cannot mask the underlying config diagnostics.

## Proposed follow-ups (out of scope; file with `/sase_new_task` if wanted)

- The cross-repo ordering hazard itself: an epic adopted a new core capability in host
  configuration eleven minutes after the core release, with nothing verifying that hosts
  had been updated. A `sase doctor` check that probes the installed core for every
  capability the bundled schema advertises would catch this before axe dies.
- `release-core-floor-smoke` only runs on the release branch; consider running the
  capability probe (cheap, wheel-only) on feature PRs too.
