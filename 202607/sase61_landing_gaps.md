---
tier: tale
title: Finish landing sase-61 (core release pin, wheel probe, skill deployment)
goal: 'The sase-61 plan-validation feature works outside dev checkouts — published
  sase resolves a sase-core-rs release that ships the plan_validate bindings, stale
  core wheels fail fast at setup, and planning agents receive the deployed tier-aware
  /sase_plan skill — and epic sase-61 is closed with its plan file marked done.

  '
phases:
- id: core-release-pin
  title: Release sase-core v0.4.0 and require it from sase
  depends_on: []
- id: skill-deploy
  title: Regenerate and deploy the updated /sase_plan skill
  depends_on: []
- id: land
  title: Close epic sase-61, run symvision, mark the epic plan done
  depends_on:
  - core-release-pin
  - skill-deploy
create_time: 2026-07-14 14:53:11
status: done
prompt: 202607/prompts/sase61_landing_gaps.md
---

# Plan: Finish landing sase-61 (core release pin, wheel probe, skill deployment)

## Context

Epic sase-61 ("Agent-Facing `sase plan validate` + Structured Epic Frontmatter", plan
`202607/plan_validate_command_1.md` in the plans sidecar) is functionally complete. Land-agent verification confirmed
all six phases are implemented, committed, and working:

- `crates/sase_core/src/plan/validate.rs` + `plan_validate`/`plan_frontmatter_schema` bindings (sase-core `717300e`,
  currently sase-core master HEAD).
- `sase plan validate` CLI + facade (`4881a04bf`), propose gate (`d2e9613a8`), approval gates (`bc32fb844`),
  deterministic epic bead creation with `bd/new_epic` retired (`9ef9688c8`), and cutover-aware committed-plan
  enforcement (`b33ef206c`). The CLI, gates, and `just validate-committed-plans` sweep (2681 legacy files) were all
  exercised locally and behave as designed. Master CI failures are pre-existing and unrelated (commit-workflow-resume
  tests, a sidecar README `sase init --check` drift, and an xprompt-highlight test predate the epic).

Verification uncovered three landing gaps that this plan finishes. Approving this plan authorizes the release-PR merge
in phase 1.

1. **The published core dependency can never satisfy the feature.** The epic plan required Phase 2 to pin "the Phase 1
   release", but no such release existed when 61.2 landed: the latest sase-core release is v0.3.4, which predates
   `717300e` and ships no `plan_validate` binding. The pin bumped to `sase-core-rs>=0.3.4,<0.4.0` — and because
   release-plz's pending release PR (sase-org/sase-core#18) is a breaking v0.4.0, the `<0.4.0` cap permanently excludes
   the first release that contains the bindings. Dev checkouts are unaffected (editable builds from the linked sase-core
   checkout; CI builds core from source), and no broken sase version has been published (Publish skipped — version
   0.10.2 unchanged), but the next sase release to PyPI would resolve sase-core-rs 0.3.4 and crash with `AttributeError`
   on every propose/approve/commit path.

2. **The core-wheel probe does not know about the new bindings.** `tools/validate_sase_core_rs` exists so stale
   installed wheels are rebuilt during setup instead of failing mid-suite (see `01babf3a8` for the precedent). Its
   `REQUIRED_BINDINGS` list was not extended, so a wheel without `plan_validate` passes the probe today.

3. **Phase 3's skill deployment never happened.** The `/sase_plan` skill source
   (`src/sase/xprompts/skills/sase_plan.md`) teaches the tier-aware author → validate → propose loop, but the chezmoi
   source and the deployed `~/.claude/skills/sase_plan/SKILL.md` still carry the old three-step flow with no
   frontmatter, tier, or validation guidance. Every planning agent currently relies on the propose gate's self-teaching
   failure output instead of authoring correctly the first time.

## Phase 1: Release sase-core v0.4.0 and require it from sase (`core-release-pin`)

**Goal**: A published sase-core-rs release ships the plan-validation bindings, and sase's dependency floor requires it.

- Open the core repo with `sase repo open sase-core`. Merge the pending release-plz release PR (sase-org/sase-core#18,
  "chore: release v0.4.0") — this is the outward-facing release action this plan authorizes. Never hand-edit crate
  versions; release-plz owns them. Wait for the tag/publish workflow to put sase-core-rs 0.4.0 on PyPI and verify the
  published wheel exposes `plan_validate` and `plan_frontmatter_schema` (e.g. install it into a scratch venv and check
  the attributes).
- Only after the release is live, in the sase repo: bump `pyproject.toml` to `sase-core-rs>=0.4.0,<0.5.0` and refresh
  `uv.lock` (bumping earlier breaks dependency resolution everywhere).
- Extend `REQUIRED_BINDINGS` in `tools/validate_sase_core_rs` with `plan_validate` and `plan_frontmatter_schema`, and
  update `tests/test_validate_sase_core_rs_tool.py` coverage accordingly.

## Phase 2: Regenerate and deploy the updated /sase_plan skill (`skill-deploy`)

**Goal**: Deployed provider skills teach the tier-aware author → validate → revalidate → propose loop.

- Read the generated-skills long-term memory first (`sase memory read generated_skills.md`) and follow its procedure
  exactly: `sase skill init --force`, then `chezmoi apply`. Skill files under chezmoi are generated — never hand-edit
  them.
- Verify the deployed `~/.claude/skills/sase_plan/SKILL.md` (and other provider variants) now document tier choice, the
  required frontmatter shapes, and the `sase plan validate` loop.
- Commit the regenerated files in the chezmoi repo.

## Phase 3: Close epic sase-61, run symvision, mark the epic plan done (`land`)

**Goal**: Epic sase-61 is closed and its plan file records completion.

- `sase bead close sase-61`.
- Run `just symvision` after the close (epic-symbol whitelist entries for sase-61 expire then) and remove any stale
  entries or unused code it reports. Read `memory/symvision.md` via `sase memory read` before fixing any failures. Note:
  symvision passed clean immediately before the close and no `sase-61` pragmas exist in `src/`, so this is expected to
  be a quick confirmation.
- Set `status: done` in the frontmatter of the epic's plan file — the PLAN path shown by `sase bead show sase-61`
  (`202607/plan_validate_command_1.md` in the plans sidecar; open it with `sase repo open`) — and commit that sidecar
  change.

## Out of Scope

- The pre-existing master CI failures (commit-workflow-resume tests, sidecar README `init --check` drift,
  xprompt-highlight test). They predate the epic and are unrelated to plan validation.
- Backfilling structured frontmatter into legacy plan files (cutover is 202608; earlier months stay legacy-lenient).

## Risks

- **Release contents**: sase-core master also carries other unreleased breaking changes (e.g. conditional agent-ID
  separators, legacy plan-directory retirement); v0.4.0 releases them together. Phase 1 should sanity-check the
  release-PR changelog against sase master compatibility — sase already builds against sase-core master in CI, which is
  strong evidence of compatibility.
- **Pin/publish ordering**: bumping the pin before sase-core-rs 0.4.0 is resolvable breaks installs; the phase order
  above (release, verify on PyPI, then bump) avoids this.
- **Skill regeneration scope**: `sase skill init --force` regenerates all provider skills, so the chezmoi diff may
  include drift beyond sase_plan; review the diff before committing.
