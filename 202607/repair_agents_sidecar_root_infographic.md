---
tier: tale
title: Repair the populated agents-sidecar root infographic
goal:
  Repository and onboarding initialization regenerate a populated agents sidecar's
  canonical root index so its packaged infographic appears immediately on GitHub without
  waiting for a later agent sync.
create_time: 2026-09-09 19:53:21
status: wip
---

# Plan: Repair the populated agents-sidecar root infographic

## Context and verified failure

The live `sase-org/sase--agents` repository demonstrates a split generated-file state
after `sase init`:

- `origin/main` at `6a40c970` contains the packaged 1600x900
  `assets/agents-directory-map.png`.
- Its 277-byte root `README.md` is still the manifest-derived owner index and contains
  no `assets/agents-directory-map.png` Markdown reference.
- The initialization commit added only the PNG, so GitHub has no instruction to render
  it.

The source already puts the image reference in both the fresh scaffold template and
`sase.agents_sync.rendering_index_pages.render_root_page()`. The missing lifecycle is
repository initialization for an already-published v2 sidecar:
`plan_sdd_sidecar_init_actions()` detects any `users/*/machines/*/manifest.json`, skips
`README.md` unconditionally, and therefore repairs the asset without regenerating the
stale root index. A later agent publication would eventually rebuild the root page, but
`sase repo init` and bare `sase init` do not fulfill their advertised generated-guide
refresh behavior immediately.

This is a Python generated-guide lifecycle bug. It does not change the agents-sidecar
wire format, owner authority, privacy policy, or Rust core behavior.

## Canonical populated-root generation

Teach `src/sase/sdd/_init_files.py` to select the correct expected agents README instead
of excluding every populated README from drift management:

- Keep using `src/sase/sdd/templates/sidecar-agents-README.md` when no v2 owner manifest
  exists, so a newly created sidecar retains its full privacy and onboarding guide.
- When v2 owner manifests exist, read all of them with the strict existing
  `sase.agents_sync.v2_io.read_all_owner_manifests()` path and render the expected root
  index with the same `render_root_page()`/Markdown normalization used by agent
  publication.
- Compare that derived text through the existing `planned_text_operation()` machinery.
  Remove the unconditional populated-README skip so `sase repo init --check`, `--diff`,
  and apply mode can report and repair root drift.
- Continue treating the packaged PNG as an independently drift-managed binary file. Do
  not add it to owner manifests, hood snapshots, agent-sync payload authority, or
  numbered workspaces.
- Do not regenerate nested user, machine, hood, family, or agent pages during repository
  initialization. Those require the complete validated snapshot graph and remain owned
  by agent synchronization; the root index alone is completely determined by owner
  manifests.

Use one canonical populated-root renderer rather than surgically inserting Markdown into
the current README. That preserves all live owner/machine/hood/run counts and links,
repairs any stale deterministic root content at the same time, and remains
byte-identical with the next agent sync.

Strict validation must remain fail-safe. If an existing owner manifest is malformed,
initialization must surface the existing agents-sync format error before writing
generated files rather than replacing the current root with partial or guessed content.
A valid, already-current populated root and asset must remain an idempotent no-op.

## Repository-init and onboarding behavior

Keep the existing sidecar transaction in `src/sase/sdd/_sidecar_init.py`: once the
planner returns README and/or asset drift, initialization writes those generated paths,
commits them together as the agents-sidecar initialization update, and pushes the
sidecar. No new direct GitHub mutation path is needed.

Confirm that:

- `sase repo init --check` reports both the stale populated `README.md` and a
  missing/stale infographic without writing.
- `sase repo init` repairs and pushes whichever of those paths drift.
- Bare `sase init`, which already delegates to the same repository initializer, gets the
  same behavior.
- A subsequent `sase agent sync` renders byte-identical root Markdown, so ownership does
  not oscillate between the two commands.

Keep the user-facing plan summary classified as a sidecar-guide refresh when the
populated README, the PNG, or both need repair.

## Tests

Update the focused fixtures to model valid v2 publication state rather than using an
invalid `{}` owner manifest:

- In `tests/sdd_store/test_sidecar_init_files.py`, reproduce the live regression with a
  valid owner manifest, an old manifest-derived root that lacks the image link, and a
  missing or stale PNG. Assert that planning includes both root-relative paths, apply
  mode writes the canonical derived root with the exact relative image link and correct
  counts, the packaged asset is restored, and the next plan is empty.
- Add the complementary cases where only the README drifts and only the asset drifts.
  Confirm fresh unpopulated sidecars still receive the privacy-forward scaffold and
  image, while populated sidecars never fall back to that scaffold.
- Cover malformed owner-manifest input: planning/apply must fail before changing the
  existing README or creating the asset.
- In `tests/main/test_repo_init_plan.py`, change the populated-sidecar regression from
  “asset only” to the exact README-plus-asset drift observed on `sase-org/sase--agents`;
  assert paths, operations, diff content, and the sidecar-guide summary. Also cover an
  already-current derived README so `--check` remains clean.
- In `tests/sdd_store/test_sidecar_init_creation.py`, extend the
  materialized/bare-remote lifecycle test to verify the derived README and asset are
  committed and visible in the remote after initialization, not merely written in the
  local clone.
- Retain `tests/agents_sync/test_rendering.py` coverage for the exact root image
  Markdown and add a byte-equality assertion between the populated README expected by
  repository initialization and the root emitted by agent sync for the same manifests.
- Exercise the existing bare-`sase init` delegation test with populated agents-sidecar
  drift so the extension path is covered rather than inferred only from source.

Use small, deterministic owner-manifest fixtures and the real packaged PNG only where
image metadata matters. Tests must not contact or mutate the live GitHub repository.

## Documentation

Correct the lifecycle wording in `docs/agents_sidecar.md` and `docs/init.md`:

- `sase repo init` owns both the static asset and the canonical root README appropriate
  to the sidecar's state.
- On a populated sidecar it rebuilds only the root index from validated owner manifests,
  preserving live counts and links while applying static presentation changes such as
  the infographic.
- Agent synchronization continues to rebuild the complete root and nested browsing tree
  from validated manifests and snapshots.

Review `docs/sdd.md` for consistency with its existing claim that repository
initialization refreshes generated READMEs and infographic assets; change it only if
clarification is needed.

## Verification and acceptance

Run `just install` before repository checks, as required for an ephemeral SASE checkout.
Run the focused generated-file, repository-init plan/apply, onboarding delegation,
sidecar-creation, and agents-rendering tests, then run the mandatory `just check`.
Review `git diff --check`, the final diff, and status.

As a read-only production acceptance check after installation, run
`sase repo init --check` for the SASE project and confirm it identifies the live
agents-sidecar root README drift (and no PNG drift). Applying `sase repo init` should
then produce an agents-sidecar commit containing the regenerated `README.md`; once the
user runs that mutating command or otherwise authorizes the live repair, verify that the
GitHub root page renders the relative PNG and that its owner, machine, hood, and run
totals remain unchanged.

The work is complete when fresh and populated agents sidecars both show the infographic
immediately after repository/onboarding initialization, populated indexes remain
canonical and idempotent, invalid manifests cannot cause partial guide writes, later
agent syncs do not churn the root page, all focused tests pass, and `just check`
succeeds.
