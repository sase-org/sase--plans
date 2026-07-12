---
create_time: 2026-07-11 18:00:31
status: done
tier: tale
---

# Migrate actstat SDD Companion Prompt Snapshots into `plans/<YYYYMM>/prompts/`

## Context

sase commit `71effb320` (`feat(sdd): nest prompt snapshots with monthly plans`) changed the canonical SDD
prompt-snapshot location from a top-level `prompts/<YYYYMM>/` directory to `plans/<YYYYMM>/prompts/`, nesting each
prompt snapshot alongside the monthly plans it belongs to. That change shipped with an idempotent, collision-safe
migration that runs as part of `sase sdd init`: it `git mv`s each snapshot, rewrites every reference (all three
link-prefix variants), regenerates the store README guides, validates the result, and commits + pushes the companion
store using the scoped-commit helper (which now correctly includes staged renames, so the migration lands as a single
commit).

The sase repo's own companion store has already been migrated. This plan applies the same migration to **this repo's SDD
companion store**: `bbugyi200/actstat--sdd`.

### Current state (verified read-only)

- The resolved SDD store (`sase sdd path`) is a clean git checkout of `git@github.com:bbugyi200/actstat--sdd.git`,
  synchronized with `origin/master`.
- It contains **8 prompt snapshots** under the legacy layout: 6 in `prompts/202606/` and 2 in `prompts/202607/`.
  Matching plan months `plans/202606/` and `plans/202607/` already exist.
- **8 plan files** reference their snapshots via frontmatter, all using the `sdd/`-prefixed variant
  (`prompt: sdd/prompts/<YYYYMM>/<name>.md`). No other references to `prompts/` exist in the store outside the generated
  `README.md` guides, which describe the old layout and will be regenerated.
- `sase sdd validate --show-warnings` baseline: **16 files, 0 errors, 0 warnings**.
- No `specs/` legacy directory and no name collisions with existing `plans/<YYYYMM>/prompts/` content (that nested
  directory does not exist yet).
- The actstat repo itself contains no references to SDD prompt paths (it is a Rust project; the only hits are inside SDD
  store checkouts).
- **Blocker:** the installed `sase` CLI is an editable install of the sase dev checkout, which is currently one commit
  behind `origin/master` — missing exactly `71effb320`. The installed CLI has no `sase.sdd._prompt_migration` module, so
  running `sase sdd init` today would not migrate anything. A `sase update` dry run confirms it will fetch +
  fast-forward that clean checkout and reinstall the editable tool set; `origin/master` is exactly `71effb320`, which
  already passed the sase repo's full `just check` gate.
- A linked-workspace clone of the **same** companion repo may exist under `.sase/workspaces/*/.sase/sdd` (one exists
  here for the chezmoi linked workspace). It is clean and in sync today, and will be stale after the migration push
  until it next fast-forwards.

## Goal

The `bbugyi200/actstat--sdd` store uses the nested `plans/<YYYYMM>/prompts/` layout with all references rewritten,
guides regenerated, validation clean, and the migration committed and pushed — performed by the supported
`sase sdd init` migration path, not by hand.

## Implementation Steps

### 1. Update the installed sase CLI

Run `sase update` (the sanctioned upgrade path). Confirm afterwards that:

- the sase dev checkout fast-forwarded to `71effb320`, and
- the installed CLI now resolves `sase.sdd._prompt_migration` (e.g. `python -c "import importlib.util; ..."` against the
  uv tool's interpreter, or simply `sase version`).

This updates the machine-wide `sase` install by exactly one already-verified commit. If `sase update` pulls anything
other than a fast-forward to `71effb320`, stop and surface it before proceeding.

### 2. Re-verify the pre-migration baseline

Read-only sanity pass immediately before migrating (guards against drift since planning):

- companion checkout clean and synced with `origin/master`;
- snapshot count under top-level `prompts/` (expected: 8, plus any snapshot added for this plan itself — record the
  exact number);
- reference inventory: every `prompt:` frontmatter line and its prefix variant;
- `sase sdd validate --show-warnings` passes with 0 errors.

### 3. Run the migration

From this repo's workspace root, run:

```bash
sase sdd init
```

This is the approved operation that performs the migration: `git mv` each snapshot into `plans/<YYYYMM>/prompts/`,
rewrite frontmatter/link references, regenerate the README guides for the nested layout, validate, and commit + push the
companion store. It is idempotent, so a re-run after a transient failure (e.g. a concurrent remote update) is safe.

### 4. Post-migration audit

Verify, in the resolved store checkout:

- top-level `prompts/` directory is gone; all snapshots live under `plans/<YYYYMM>/prompts/` with the same total count
  as the step-2 baseline;
- every plan's frontmatter now reads `prompt: sdd/plans/<YYYYMM>/prompts/<name>.md` (same prefix variant as before, path
  nested) and `sase sdd links` / `sase sdd validate --show-warnings` report 0 errors with the same file count as
  baseline;
- the regenerated `README.md` guides describe the nested layout (no lingering top-level `prompts/` wording);
- plan listing/search still returns each plan rather than its nested prompt snapshot (spot-check with `sase sdd list`);
- git history recorded renames (not delete + add), the migration landed as a **single** scoped commit (the staged-rename
  fix makes this the expected outcome — a second commit like the sase store needed would indicate a regression worth
  flagging), and the checkout is clean and synchronized with `origin/master`.

### 5. Refresh sibling checkouts of the same store

If a linked-workspace clone of the companion repo exists under `.sase/workspaces/*/.sase/sdd`, fast-forward it
(`git pull --ff-only`) and confirm it shows the nested layout. Clones in other actstat workspaces fast-forward on their
next sync and need no action.

## Out of Scope

- The chezmoi repo's own in-repo `sdd/tales/` documents (they reference a historical in-repo `sdd/prompts/` path that is
  not part of this companion store) and any chezmoi companion store — migrate those from a chezmoi context if desired.
- The actstat repo's source code — it has no SDD prompt-path references, so no repo commit is expected from this work
  beyond the plan/prompt snapshot bookkeeping SASE itself performs.
- Any changes to sase itself; the migration code is already merged and verified.

## Risks / Notes

- `sase update` changes the machine-wide sase install. Mitigation: the delta is a single commit that already passed the
  sase repo's full test/lint gate, and the dry run confirmed a plain fast-forward.
- The companion push may race a concurrent remote update; the push helper fetch/rebases safely, and the migration is
  idempotent.
- Expected final state: same validated file count as baseline, 0 errors, snapshot count unchanged (modulo the snapshot
  SASE records for this plan), single migration commit pushed.
