---
tier: tale
title: Restore Codex weekly usage visibility with NVM executable resolution
size: medium
goal: Restore the Codex weekly all-model usage indicator when Codex is installed through
  NVM and its bin directory is absent from the TUI's PATH.
proposed_by: bbugyi200.athena.0jv
status: done
---

# Restore Codex usage visibility with NVM executable resolution

## Outcome and scope

Make usage collection eligibility recognize the same Codex executable that SASE's Codex
launcher and usage collector already resolve. A configured, enabled Codex provider with
an executable available through `NVM_BIN/codex` must remain eligible when `codex` is
absent from `PATH`. Its cached weekly all-model window must reach the header under the
existing `weekly_all: always` policy.

This is one bounded implementation for one coding agent: a provider integration fix and
regression coverage. Reuse the existing executable resolver as Python OS glue; keep
usage validation, period/scope classification, and policy selection in Rust core. No new
backend policy, binding, configuration option, or usage schema is needed.

## Diagnosis and evidence

The user screenshot is available through the audited reference
`file:/home/bryan/tmp/screenshots/20260912_050506.png`, also supplied as
`.sase/artifacts/pool/ee06c365aed1-file-ref.png`. It shows Claude and Grok usage but no
Codex usage, with ample header space and no overflow disclosure.

Read-only investigation on September 12, 2026 established the following:

- `sase usage list -p codex --json` contains an account-scoped `codex:primary` window
  with `duration_seconds: 604800`, 1% used, and a future reset. Collection succeeded
  with a complete observation. A weekly window can occupy the primary slot; do not
  rename it or assume that weekly means `codex:secondary`.
- The effective indicator configuration has `weekly_all: always` and no Codex override.
  The real Rust projection correctly classifies this window as weekly, all-model, and
  selected. A fresh cache projection under the agent's environment renders the Codex
  badge.
- The running ACE process's `PATH` cannot resolve `codex`. Its `NVM_BIN` points to a
  directory containing an existing executable `codex`, and `SASE_CODEX_PATH` is unset.
  `resolve_codex_executable()` correctly resolves that NVM executable.
- `eligible_usage_providers()` calls `_provider_cli_ready()` in
  `src/sase/llm_provider/usage/refresh.py`. That helper checks the explicit provider
  override or declared CLI name against a file/`PATH` lookup, bypassing Codex's NVM
  fallback. It therefore excludes Codex. `refresh_usage_peek_cache()` passes this
  reduced eligible set to Rust, which correctly filters Codex out of the indicator.
- Replaying only the live process's `PATH` and `NVM_BIN` in a disposable Python process
  reproduced eligibility `claude, grok` and header `🛰️ 59% 6d4h`. An in-memory
  substitution that changed only Codex readiness to use its existing resolver produced
  `claude, codex, grok` and header `🤖 99% 6d22h  🛰️ 59% 6d4h`. Other providers retained
  the original readiness check. This changed no source files, saved usage data, or live
  TUI state.

The root cause is inconsistent executable discovery between usage eligibility and Codex
invocation. Neither weekly classification nor width allocation needs a fix.

## Implementation

1. Add a focused regression for public `eligible_usage_providers()` in
   `tests/llm_provider/test_usage_refresh.py`, or a dedicated sibling eligibility test
   module if that keeps the suite readable. Supply deterministic provider metadata,
   references, configuration, and a temporary executable under `NVM_BIN`; ensure `PATH`
   cannot resolve Codex and the explicit override is absent. Leave the actual readiness
   helper and Codex resolver active. Demonstrate the failure before implementing the
   fix.

2. Update `_provider_cli_ready()` to obtain the Codex command from
   `sase.llm_provider.codex.resolve_codex_executable()` before applying readiness
   checks. Keep the existing generic metadata/override path for other providers. Reuse
   the resolver instead of duplicating NVM probing or changing static plugin metadata to
   contain a process-specific path. Preserve its precedence: `SASE_CODEX_PATH`, then
   `PATH`, then `NVM_BIN/codex`, then the unresolved command name. An explicit invalid
   override must not silently fall through to NVM. Preserve existing readiness semantics
   rather than broadening this fix into an executable-permission or path-normalization
   rewrite.

3. Cover the availability boundaries: NVM fallback succeeds; ordinary PATH lookup
   succeeds; an explicit path wins over PATH/NVM; an explicit missing path stays
   unavailable; missing PATH/NVM candidates stay unavailable. Preserve exclusion for
   hidden, unreferenced, non-probing, or collection-disabled providers and inclusion
   through the existing explicit collection enablement. Keep a generic plugin case to
   protect the metadata-driven behavior for other providers.

4. Add an integration regression in `tests/llm_provider/test_usage_peek.py` or the
   existing indicator presentation integration tests. Supply a deterministic public
   snapshot with the observed account-scoped `codex:primary` weekly shape at 99%
   remaining, plus a normal second provider. Exercise the real eligibility check,
   peek-cache loading, Rust projection, and header text builder. Mock only external
   inputs such as metadata, configuration, filesystem candidates, snapshot loading, and
   the clock; do not mock eligibility, projection, or the selected entries. Assert Codex
   is selected through `weekly_all`, has period `weekly` and scope `all_models`, and
   renders its badge and reset countdown with a sufficient budget. Verify an explicit
   `never` window policy still hides it. Clear process caches between cases and isolate
   all state using the test sandbox.

5. Keep the widget, cache token, refresh cadence, and Rust usage projection intact. The
   corrected helper already runs on the worker/refresh path. Add no filesystem work,
   subprocess probing, or network calls to render paths or the message pump. Keep normal
   agent invocation and the Codex collector using their existing resolver.

## Verification and acceptance

Before implementation, read `lint_and_test.md` and `tui_perf.md` using
`sase memory read`. Establish the workspace test environment with `just install` if
necessary. During planning the workspace virtualenv reported a core distribution but
could not import its editable extension; the successful reproduction used the installed
SASE interpreter with the working tree's Python source. Do not mistake that environment
issue for a failing product regression or bypass Rust with mocks.

Run the focused eligibility/cache regressions, existing Codex executable-resolution and
collector tests, and the existing usage indicator presentation/widget/header tests. Use
the repository's test runner and isolated virtualenv. Then run `just check` as required
by the repository. Use `sase_monitor` if verification becomes long-running; run
`just check-full` through that skill if the repository's escalation rules require it. No
visual golden updates should be necessary because rendering is unchanged.

Acceptance requires an automated reproduction that fails before the fix and passes
afterward, successful normal verification, and the weekly Codex badge selected in a
read-only replay of the TUI environment without an ad hoc indicator override or a PATH
edit. Existing provider opt-outs and display policies must still be respected. Report
that the running TUI needs to reload the updated Python code through the normal
update/restart workflow; a usage refresh alone cannot replace already-loaded code.

No source implementation or live configuration changes occur before this plan is
submitted and approved. Broader registry discovery changes, shell startup changes, and
provider protocol changes are outside this fix.
