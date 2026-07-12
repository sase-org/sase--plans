---
create_time: 2026-07-11 08:10:42
status: done
prompt: 202607/prompts/is_sase_managed.md
tier: tale
---
# Generalize Project Initialization Ownership with `is_sase_managed`

## Goal

Replace the recently introduced project-local `memory.enabled` opt-in with a top-level `is_sase_managed` boolean that
represents SASE ownership of the repository as a whole. Preserve the memory initializer's current safety boundaries, and
use the same local marker to prevent `sase sdd init` (including its planner and compatibility alias) from materializing
storage or generating files for repositories that are not SASE-managed main projects.

## Current Behavior and Constraints

- `memory.enabled` is read only from the current repository's own `sase.yml`; missing or false leaves project memory and
  the root `AGENTS.md` unmanaged. Home memory initialization and byte-for-byte provider copies for existing project
  `AGENTS.md` files intentionally continue independently.
- `-M, --enable-project-memory` can create or update that local opt-in before planning. The resulting config change is
  included in the normal project commit path.
- `sase sdd init` currently considers any VCS checkout eligible. Its apply path can materialize provider storage before
  generating SDD guide files, and its read-only planner can propose companion-repository and generated-file work.
- `sase init --all` limits its inventory to registered active main projects, but direct and ordinary bare init still
  need a repository-local ownership guard. The guard must never be inherited from merged user/default configuration.
- `sase sdd init --path` must evaluate the target repository's `sase.yml`, not the caller's working directory.

## Proposed Semantics

- `is_sase_managed: true` in the target repository's own `sase.yml` is the sole local authorization marker. A missing
  file, missing key, or explicit false means unmanaged. Invalid YAML, a non-mapping document, or a non-boolean marker is
  an error because SASE cannot safely determine ownership.
- This is a direct migration, not a compatibility alias: `memory.enabled` no longer opts a repository in. Update this
  repository's checked-in config and all documented examples to the new key so stale memory-specific authorization
  cannot silently retain broader SDD ownership.
- Project memory generation, managed root `AGENTS.md` generation, project-memory validation, title derivation, and
  linked-repository memory loading remain gated exactly as they are today, but by `is_sase_managed`. Home memory and
  provider instruction copies retain their existing independent behavior.
- Keep the existing `-M, --enable-project-memory` CLI spelling for compatibility, but make it write
  `is_sase_managed: true` and explain that it marks the current repository as SASE-managed, thereby enabling managed
  project memory. Continue rejecting it with `--check` and `--all` and preserve comment-aware YAML edits and commit
  ownership.
- For a valid unmanaged target, both `sase sdd init` and `sase sdd init --check` short-circuit before provider
  detection, store resolution/materialization, legacy import inspection, or generated-file planning. They report an
  informative skipped/current summary and succeed with no actions. Invalid marker configuration produces a planner
  blocker or apply error and performs no provider or filesystem work.

## Implementation Plan

1. Introduce a shared project-management config boundary used by memory and SDD initialization. Centralize local-only
   YAML loading, boolean validation, and comment-preserving writes for `is_sase_managed`, with structured results that
   distinguish unmanaged repositories from invalid configuration. Keep memory-specific naming and linked-repository
   parsing in the memory module.
2. Migrate configuration surfaces from `memory.enabled` to top-level `is_sase_managed`: update the packaged defaults,
   JSON schema and descriptions, this repository's `sase.yml`, and schema regression coverage. Explicitly verify that
   booleans are accepted, other types are rejected, and the retired nested key is not treated as authorization.
3. Rewire memory initialization and onboarding to the shared marker. Rename internal state/helpers away from
   memory-specific ownership terminology where they now represent repository management, update the `-M` preparatory
   write and planned action text, and preserve the existing ordering that writes the opt-in before the bare init
   coordinator plans memory and SDD work.
4. Add the SDD ownership guard to both `plan_sdd_init` and `run_sdd_init`, using the existing target config resolver so
   `--path` is handled correctly. Return an action-free, non-blocking plan and exit zero for valid unmanaged targets;
   return a blocker/error for invalid config. Place the guard before every provider/store/import/generated-file call so
   a skipped command is demonstrably side-effect free.
5. Expand focused tests across the shared config helper, memory opt-in/validation/commit behavior, SDD planning and
   apply behavior, parser/help text, and bare/batch onboarding integration. Cover missing, false, true, invalid-type,
   malformed-YAML, retired-key, merged-global, and `--path` cases; assert provider materialization and file creation are
   never invoked for unmanaged or invalid targets, while managed targets retain current SDD behavior.
6. Update user documentation in the initialization, configuration, and memory guides to describe the repository-wide
   marker, the unchanged home/provider-copy exception, the SDD no-op behavior for unmanaged repositories, the `-M`
   migration behavior, and the one-time config change from `memory.enabled` to `is_sase_managed`.
7. Install the workspace dependencies as required, run focused memory/SDD/onboarding/schema tests during development,
   then run the repository-mandated `just check` suite to validate formatting, linting, types, unit tests, and visual
   regressions.

## Acceptance Criteria

- A repository is eligible for project memory and explicit SDD initialization only when its own `sase.yml` contains
  `is_sase_managed: true`; global/default values and the retired `memory.enabled` key cannot authorize it.
- Unmanaged `sase sdd init` and `--check` invocations succeed without provider calls, storage materialization, config
  writes, SDD writes, or proposed actions, including when targeting another repository via `--path`.
- Invalid local ownership configuration fails safely and before any project or provider mutation.
- Managed repositories preserve current memory, onboarding, commit/deploy, and SDD initialization behavior.
- Documentation, schema/defaults, checked-in config, help text, and tests consistently use `is_sase_managed`.
