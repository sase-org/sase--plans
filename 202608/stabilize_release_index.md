---
tier: tale
title: Stabilize v0.17.0 release metadata reconciliation
goal:
  The Publish workflow updates the pending v0.17.0 release branch without unrelated uv
  registry-source churn, allowing all release checks to pass.
size: small
proposed_by: bbugyi200.athena.0b2
create_time: 2026-08-22 16:19:43
status: wip
---

# Stabilize release metadata reconciliation for v0.17.0

## Goal

Make the Publish workflow reconcile the pending v0.17.0 release branch without rewriting
unrelated dependency sources, so the workflow can ratchet `sase-core-rs` from 0.30.0 to
0.31.0 and allow `ci_watch` to merge release PR #284.

## Root cause

Publish runs `tools/ratchet_core_window --allow-transitive-lock-refresh` on the
release-please branch. The tracked `uv.lock` records the canonical PyPI registry as
`https://pypi.org/simple/`, matching the developer uv configuration that generated it. A
clean GitHub Actions runner has no repository uv index setting, so uv 0.12.5 uses its
default spelling, `https://pypi.org/simple`, and rewrites the `source.registry` field
for every PyPI package while updating `sase-core-rs`. The semantic ratchet guard
encounters the first direct dependency, Jinja2, and correctly refuses the unexpected
direct-dependency rewrite with exit code 3.

This is reproducible locally with `UV_NO_CONFIG=1`; setting
`UV_DEFAULT_INDEX=https://pypi.org/simple/` makes the same release-version/core ratchet
produce only the intended project version, core constraint, and core artifact changes.

## Implementation

1. In `.github/workflows/publish.yml`, set `UV_DEFAULT_INDEX=https://pypi.org/simple/`
   on the `Reconcile release metadata` step. Keep it scoped to that mutating step so
   both the ratchet's internal `uv lock` and the following idempotence lock use the
   tracked canonical spelling without changing unrelated workflow jobs.
2. Extend `tests/test_github_actions_ci.py` to assert that the reconciliation step
   carries the exact canonical default-index environment value. Preserve the existing
   assertions that the ratchet runs before the final lock refresh and that only
   `pyproject.toml` and `uv.lock` are committed.

## Validation

1. Run the focused GitHub Actions workflow contract test.
2. Reproduce release reconciliation from temporary copies of `pyproject.toml`,
   `uv.lock`, `README.md`, and `LICENSE`, changing only the temporary project version to
   0.17.0. Run the ratchet in a clean uv configuration with `UV_NO_CONFIG=1` and the
   workflow's `UV_DEFAULT_INDEX=https://pypi.org/simple/`; require exit code 2 and
   verify the report contains the intended 0.30.0-to-0.31.0 core ratchet without the
   Jinja2 direct-dependency error or broad registry-source churn.
3. Run `just install`, then `just check` as the required repository-wide gate.
4. After the change reaches master, use `actstat` and the Publish run logs to confirm
   `sync-release-metadata` succeeds, updates the pending release branch idempotently,
   and leaves no remaining failed GitHub Actions jobs blocking release PR #284.

## Non-goals and safety

- Do not weaken `ratchet_core_window`'s protection against direct dependency version or
  metadata changes; the failure is caused by missing workflow input, not an overly
  strict safety policy.
- Do not commit generated v0.17.0 release metadata directly to master; the existing
  release-please and reconciliation jobs remain responsible for the release branch.
- Do not broaden the index setting to install, test, build, or publish jobs.
