---
tier: tale
title: Enable automatic SASE artifact pruning
goal:
  Opt the chezmoi-managed global SASE configuration into automatic artifact retention while preserving the shipped
  retention thresholds and verifying the deployed effective policy.
proposed_by: bbugyi200.athena.r8
create_time: 2026-08-01 09:50:41
status: done
---

- **PROMPT:** [202608/prompts/enable_artifact_pruning.md](prompts/enable_artifact_pruning.md)

# Plan: Enable automatic SASE artifact pruning

## Context and decisions

- SASE added the opt-in `artifacts.retention` policy in commit `6999e31a3` on 2026-07-30. Automatic retention runs after
  agent finalization only when `artifacts.retention.enabled` is `true`.
- The built-in policy already supplies `keep_per_label: 3`, `max_age_days: 90`, and `trash_grace_days: 14`. The
  chezmoi-managed user config currently has no `artifacts` override, so its effective value is `enabled: false` with
  those thresholds.
- Add only the opt-in boolean. Do not copy the numeric defaults into user configuration: inheriting them keeps the
  override minimal and lets future SASE default changes remain effective unless the user deliberately pins a value.
- A read-only `sase artifact stats --json` preflight found no unavailable protection sources, no existing trash, and
  zero artifacts selected by the current default policy. Repeat that check immediately before activation because the
  artifact store can change between planning and implementation.

## Implementation

1. Open the linked `chezmoi` repository with `sase repo open chezmoi` and use the returned checkout for all reads and
   writes. Confirm that `home/dot_config/sase/sase.yml` has no overlapping work before editing it.
2. Add this minimal top-level configuration block to `home/dot_config/sase/sase.yml`, preserving the file's existing
   formatting and unrelated settings:

   ```yaml
   artifacts:
     retention:
       enabled: true
   ```

   Do not add `keep_per_label`, `max_age_days`, or `trash_grace_days`; they should continue to resolve to the built-in
   values `3`, `90`, and `14`.

3. Before applying the chezmoi source, rerun the read-only retention preview and require
   `protections.sources_unavailable` to remain empty. Inspect a target-scoped chezmoi dry run for
   `~/.config/sase/sase.yml` and confirm it contains only the intended retention opt-in plus any rendering context
   required by chezmoi.
4. Apply the target config through chezmoi so this machine is actually opted in. If the change is committed during the
   normal SASE completion workflow, follow the chezmoi repository instruction to run `chezmoi update -a --force`
   afterward so the committed source and home-directory target remain synchronized.

## Validation

1. Parse the source with `yq` and assert that `.artifacts.retention` contains exactly `enabled: true`; this catches YAML
   structure or indentation errors and accidental pinning of the inherited thresholds.
2. Run `git diff --check` and inspect the repository diff, confirming that the only source change is the new block in
   `home/dot_config/sase/sase.yml` and that no machine overlay or unrelated configuration changed.
3. Run `sase config layers` and `sase doctor -C config.layers -s` after applying the target. Confirm the user layer now
   exposes `artifacts` and that SASE reports no config-layer errors or warnings.
4. Run `sase config show -k artifacts` and verify the deployed merged policy is exactly:

   ```yaml
   artifacts:
     capture:
       max_history_scan: 20
       max_stored_per_agent: 50
     retention:
       enabled: true
       keep_per_label: 3
       max_age_days: 90
       trash_grace_days: 14
   ```

5. Run `sase artifact stats --json` once more and confirm the protection scan succeeds and the reported default policy
   still uses `keep_per_label: 3` and `max_age_days: 90`. Do not invoke `sase artifact prune --apply`; ongoing pruning
   is intentionally delegated to the newly enabled post-finalization retention pass, with discarded rows remaining
   restorable for the inherited 14-day grace period.

## Acceptance criteria

- The chezmoi source opts in with only `artifacts.retention.enabled: true`.
- The rendered user config is applied on this machine and the merged SASE config reports retention enabled with the
  unpinned built-in `3 / 90 / 14` policy.
- Protection-source checks remain healthy, YAML/config diagnostics pass, and the diff contains no unrelated changes.
