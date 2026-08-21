---
tier: tale
title: Retire stale generated xprompt skills safely
goal: "SASE records every generated skill file it owns in chezmoi, removes retired skill
  files from both chezmoi source and live provider locations, previews those deletions
  before confirmation, and cleans obsolete SASE skills from this machine.

  "
size: medium
proposed_by: bbugyi200.athena.098
create_time: 2026-08-21 09:11:37
status: wip
---

# Plan: Retire stale generated xprompt skills safely

## Current behavior and constraints

`sase skill init` renders only the currently discovered xprompt skills. Its existing
`.sase-skills-manifest.json` records source provenance and a source-set hash, but not
the generated files owned by SASE, so a skill that disappears from the source catalog
never produces a delete operation. `chezmoi apply` also does not remove a live target
merely because its source file disappeared.

The stale state is present on this machine: `/sase_artifact_file` was removed from
`src/sase/xprompts/skills/`, but its `SKILL.md` remains in the chezmoi source tree and
the live skill directories for the seven currently registered providers. The chezmoi
tree also contains a legacy Gemini/Jetski provider skill root that is not part of the
current generated target matrix. The cleanup must cover current and legacy provider
roots without treating arbitrary user-authored skills as SASE-owned.

Keep the existing commit-first deployment rule, source-integrity and monotonic
provenance guards, exclusive chezmoi deployment lock, provider filtering, and
`--no-commit`/`--no-push`/`--no-apply` semantics. Do not edit generated `SKILL.md` files
by hand as the implementation mechanism, and do not edit SASE memory files as part of
this work.

## Implementation

### 1. Add a durable, safe ownership inventory to the skill manifest

- Extend `src/sase/main/_init_skills_manifest.py` so the version-controlled chezmoi
  manifest records normalized managed-file entries in addition to the existing source
  commit, source-set hash, and deployment time. Each entry must carry enough durable
  information to identify both the chezmoi source path and its live home target after a
  provider or deployment subpath is renamed; include provider and skill identity for
  filtering and display.
- Represent current files and retired files separately (or with an explicit state).
  Retired entries are deletion tombstones: keep them after successful removal so an
  interrupted apply or later stale checkout can be cleaned idempotently. If a path
  becomes current again, reconcile it back to active ownership rather than deleting it.
- Store paths relative to the declared chezmoi source root and home root. Reject
  absolute paths, `..` traversal, duplicate/conflicting source or target mappings, and
  any resolved path outside those roots before planning or mutating files. Never delete
  an untracked path based only on its name.
- Migrate the existing manifest without losing current ownership. Seed active entries
  from the complete rendered target set that is already present, including existing
  non-`sase_*` xprompt skills such as `bob_query`, rather than tracking only files
  created after this feature lands. For the one-time legacy sweep, recognize existing
  `sase_*` `SKILL.md` files under actual agent-provider skill roots in the chezmoi tree
  as SASE namespace files, including roots left by a retired provider integration; mark
  entries not present in the current rendered set as retired. Ignore unrelated files and
  skill-source directories that are not provider deployment roots.
- Make provider-filtered runs reconcile only the selected provider's active entries and
  deletions while preserving all other manifest state. A full run reconciles every
  registered provider plus legacy tombstones. Do not advance the manifest past a skipped
  or declined file operation, so the next run can retry it.

### 2. Plan creates, overwrites, and retirements through one model

- Introduce a small internal deployment/ownership model shared by
  `init_skills_handler.py`, `_init_skills_rendering.py`, the manifest code, and the
  read-only inventory. It should pair every chezmoi source path with its home target and
  produce current writes plus stale managed-file removals deterministically.
- Remove the `No skill source entries found` early return as a terminal condition. An
  empty current source set must still plan and execute deletion of previously managed
  files, which is required when the last skill or the last skill for a provider is
  retired.
- Add `delete` actions for retired source files and include the paired live target path
  in structured action data or clear action detail. Count deletes in the plan summary,
  preserve the existing unchanged counts, and keep `--provider` scoping exact.
- Keep `--check` read-only. Under chezmoi, report pending deletions through the existing
  deferred-until-land warning path just like creates and overwrites; outside check mode,
  expose them as actionable plan items.

### 3. Delete source and live targets transactionally within deployment semantics

- In `run_init_skills`, apply accepted retirements by unlinking only validated,
  manifest-owned chezmoi source files and pruning only now-empty generated skill
  directories. Include missing tracked paths in the git pathspec set so deletions and
  the updated manifest are staged and committed together. Missing files are a successful
  idempotent no-op.
- Give direct interactive runs a delete confirmation parallel to overwrite confirmation.
  `--yes` and `--force` may accept planned managed deletions; non-interactive runs
  without either flag must skip them safely and leave manifest ownership retryable.
  `--dry-run` and `--diff` must never write or delete anything.
- Extend `ChezmoiDeployBehavior`, deferred bare-`sase init` aggregation, and the shared
  deployment path to carry validated live targets scheduled for removal. Delete those
  targets only when the workflow reaches the actual apply stage (after successful source
  commit/push and before `chezmoi apply --force`), while the same exclusive lock is
  held. Consequently `--no-commit`, `--no-push`, `--no-apply`, a failed commit/push, or
  a no-upstream early stop must leave live targets untouched; the retained tombstone
  makes a later full apply retry the cleanup.
- Refuse and report a target deletion that escapes the home root or cannot be completed.
  After deleting retired targets, run the ordinary full chezmoi apply so active files
  are refreshed. Preserve existing behavior for callers of the generic chezmoi deploy
  helper that do not supply deletion targets.

### 4. Surface deletions everywhere a user reviews or confirms skill changes

- Update `plan_init_skills`, the shared init inventory/diff renderer inputs, and bare
  `sase init` confirmation text so the user sees the number of retired skills and both
  the chezmoi source and live target paths that will disappear before approving the run.
  A delete diff should show the removed source content when it exists.
- Update direct `sase skill init --dry-run`, `--diff`, and per-file confirmation output
  to label removals as deletions rather than generic refreshes. Avoid duplicate prompts
  for the source/target pair: one decision authorizes the already-disclosed pair.
- Extend `sase skill list`'s inventory/dashboard to report tracked retired targets as
  deletion drift even though they have no current xprompt source, with clear counts,
  provider/skill identity, and source/live paths. Keep current, stale, and missing
  active-target reporting intact.
- Refresh relevant CLI/xprompt documentation and command help to explain that the
  manifest is an ownership registry and that a full chezmoi deployment removes retired
  generated skills. Do not change the generated-skills memory note without separate
  explicit permission.

### 5. Cover migration, safety, UI, and deployment with tests

- Add manifest tests for legacy-manifest migration, current-file backfill, discovery of
  legacy `sase_*` provider skills, stable ordering/serialization, active-to-retired and
  retired-to-active reconciliation, provider-filter preservation, malformed/path-escape
  rejection, and no-op reruns.
- Add planning/apply tests for rename, deletion, an entirely empty source set, declined
  and non-TTY deletion prompts, `--yes`/`--force`, missing files, directory pruning, and
  unchanged unrelated/user-authored skills.
- Add deploy tests that prove ordering and failure semantics: source deletion is staged;
  live deletion happens only immediately before apply; skip flags, missing upstream,
  commit/pull/push failures, and apply failures remain retryable; deferred bare-init
  deployment carries the same delete targets.
- Add inventory/rendering tests showing deletion counts and source/live paths in
  `sase skill list`, bare-init inventory, confirmation text, dry-run output, and full
  diffs. Retain coverage for current/stale/missing skills and provider filters.
- Run `just install`, focused skill-init/inventory/deploy tests while iterating, and
  `just check` for the final SASE tree. Use temporary home and chezmoi roots in tests;
  no test may touch the user's real provider directories or chezmoi repository.

### 6. Deploy and verify the one-time machine cleanup

- After the SASE implementation is committed and landed on the canonical branch, run the
  guarded full `sase skill init --force` from that clean merged tree. Let the normal
  workflow update, commit, push, and apply the chezmoi repository; do not deploy from an
  uncommitted workspace or bypass provenance guards merely to finish cleanup.
- Confirm the migrated manifest includes every current generated xprompt skill target,
  not just newly written skills, and retains tombstones for retired targets.
- Verify that every chezmoi-source and live-provider copy of
  `sase_artifact_file/SKILL.md` is gone. Scan all configured current provider skill
  roots and discovered legacy provider roots (including the existing Gemini/Jetski root)
  for other `/sase_*` directories absent from the current skill catalog; remove those
  through the same tracked retirement flow and verify no obsolete source or live copy
  remains.
- Re-run `sase skill list` and an explicit filesystem/chezmoi-managed inventory check.
  The result must show only current SASE skills (or separately explained active content
  drift), no retired deletion drift, and no untracked changes in either the SASE or
  chezmoi repository.

## Acceptance criteria

- The chezmoi manifest durably identifies all current SASE-generated skill source/live
  pairs and safely carries retired-path tombstones across interrupted deployments.
- Deleting or renaming an xprompt skill, changing its provider set, or removing the last
  source plans and performs deletion of both the owned chezmoi source and live target on
  the next successful full apply.
- No untracked or path-escaping file can be deleted, and provider-filtered or skipped
  runs cannot silently change ownership outside their accepted scope.
- Every interactive preview or confirmation for chezmoi skill changes clearly discloses
  retired-file deletions, including the paired source and live paths.
- Existing xprompt skills are backfilled into ownership on migration, and future skills
  are added automatically.
- `/sase_artifact_file` and every other obsolete `/sase_*` provider skill found on this
  machine are absent from both chezmoi source and live agent-provider locations after
  deployment, while current SASE and unrelated user skills remain intact.
