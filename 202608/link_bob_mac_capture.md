---
tier: tale
title: Link Bob Mac Capture to the bob-cli SASE project
goal:
  Configure bob-mac-capture as a documented, lazily materialized linked repository of
  bob-cli and verify that SASE exposes the relationship correctly.
size: small
proposed_by: bbugyi200.athena.00k
create_time: 2026-08-14 07:49:51
status: wip
---

# Plan

## Context and decisions

Recent `bob-cli` work introduced `capture-parse` and `capture-complete` as shared,
versioned JSON interfaces over the authoritative capture grammar. The new
`bob-mac-capture` Swift app immediately consumed those commands together with
`capture-targets` and `capture --dry-run --format json`, then hardened the subprocess,
preview, cancellation, packaging, and CI paths. The repositories therefore form a
backend/frontend pair: `bob-cli` owns capture semantics and vault mutation, while
`bob-mac-capture` owns the native macOS menu-bar UI and process orchestration.

Declare the relationship in the project-local `sase/sase.yml`, beside the existing
`bob-plugins` linked-repository entry. Keep the new entry lazy by omitting
`auto_clone: true`: not every CLI task needs the macOS app checkout, and a lazy entry
keeps its purpose visible in generated agent instructions while remaining available
through `/sase_repo` on demand.

Use this configuration shape and relationship-focused description:

```yaml
- name: bob-mac-capture
  path: ~/projects/github/bobs-org/bob-mac-capture
  description: >-
    Native macOS menu-bar frontend for Bob capture. It delegates capture grammar,
    completion, live preview, and vault mutation to bob-cli's versioned `bob`
    subprocess/JSON interfaces, so coordinate capture-contract changes across both
    repositories.
```

## Implementation

1. Open the `bob-cli` repository through `/sase_repo`, confirm the checkout is clean,
   and re-read the current `repos.linked` block before editing so concurrent changes are
   preserved.
2. Add the `bob-mac-capture` entry above to `repos.linked` in `sase/sase.yml`. Preserve
   the existing `bob-plugins` configuration and do not add `auto_clone`; the default
   lazy behavior is intentional.
3. Run `sase memory init --no-commit` from the `bob-cli` checkout. Let SASE regenerate
   `sase/memory/sase.md`, the root agent-instruction files, and any related memory index
   output; do not hand-edit generated memory or provider shims. Review the diff and keep
   only changes caused by the new linked-repository declaration.

## Validation

1. Run `sase memory init --check` and require a clean pass, proving the generated memory
   surface matches `sase/sase.yml`.
2. Run `sase doctor -C config.repos` and resolve every repository-configuration error.
3. Inspect `sase repo list --json` from the `bob-cli` checkout and verify that
   `bob-mac-capture` is reported as a lazy `linked` repository with environment name
   `BOB_MAC_CAPTURE`, the configured path, and the authored description.
4. Open it by the linked name with
   `sase repo open bob-mac-capture -r "Verify the bob-cli linked-repository configuration"`,
   then confirm the resolved checkout's `origin` is
   `git@github.com:bobs-org/bob-mac-capture.git` and that its recent history matches the
   repository reviewed during planning.
5. Run `git diff --check` and inspect the final diff. Confirm the generated instructions
   present `bob-mac-capture` by its user-facing name and accurately explain the
   frontend/backend contract without unrelated source or configuration changes.

No Rust or Swift production behavior changes are in scope; configuration, generation,
repository diagnostics, and resolution checks are the relevant verification surface.
