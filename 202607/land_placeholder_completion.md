---
tier: tale
title: Land placeholder completion after the sase-core release
goal: 'SASE consumes the released sase-core placeholder APIs, the feature is verified
  through the published artifact and all editor surfaces, and epic sase-6b is closed
  with its Symvision and plan-state cleanup complete.

  '
create_time: 2026-07-16 09:51:29
status: wip
prompt: 202607/prompts/land_placeholder_completion.md
---

# Plan: Land placeholder completion after the sase-core release

## Context

Epic `sase-6b` implemented document-local `<placeholder>` completion across three closed child beads:

- `sase-6b.1` added the single-source-of-truth Rust engine, Python bindings, and xprompt LSP support in `sase-core`
  (`b90ffdc479287215a18ae1aac50e0dfe2f6e5772`).
- `sase-6b.2` added ACE completion, snippet retriggering, highlighting, caching, tests, and PNG snapshots in `sase` (the
  recorded `6b8ad332f` was rebased to reachable commit `b74adbf4cad66da4435017a41df074965d53a694`).
- `sase-6b.3` added the Neovim LSP smoke test and documentation in `sase-nvim`
  (`161f2f13be673ea3cd6f48cc1fbcff718299d43e`).

The implementation audit and source-based verification are clean: the full Rust format/clippy/workspace-test gate, SASE
`just check`, and `tests/lsp_placeholder_smoke.lua` all pass. The remaining issue is release integration. While the epic
was running, release-plz landed `sase-core` commit `754f40c6717a682ea309017286f23ea82180677b` and tag `v0.5.0`; its
changelog explicitly includes `sase-6b.1`. The SASE repo still declares `sase-core-rs>=0.4.1,<0.5.0`, so its version
validator rejects the released linked checkout. At plan time the v0.5.0 GitHub release exists but the publish workflow
is queued and PyPI still reports 0.4.1. Do not manually edit release-plz-owned Cargo versions or use the manual recovery
path without explicit user approval.

Two unrelated SASE commits landed after the epic began: `0a910c518` and `ec1a006f5`, both confined to the Artifacts
Plans data/pane redesign (plus its styles and tests). They do not overlap the prompt-input implementation, but preserve
and re-run their coverage when updating the core dependency and lockfile.

## Phase 1: Confirm the released distribution is usable

Open `sase-core` through `/sase_repo` and refresh its base branch. Confirm that `v0.5.0` contains `b90ffdc`, that the
release changelogs include placeholder completion, and that no later base-branch change conflicts with the placeholder
engine or LSP server.

Wait for the existing release workflow to finish successfully and for `sase-core-rs==0.5.0` to appear on PyPI. Do not
close the epic while only the source checkout contains the required Python bindings. In a disposable environment with no
linked-source build, install the published 0.5.0 wheel and assert that `sase_core_rs.placeholder_completion` and
`sase_core_rs.placeholder_spans` exist and return the expected completion/span JSON shapes. If publication fails or the
wheel lacks the APIs, stop and report that external release blocker rather than triggering a recovery publish without
authorization.

## Phase 2: Integrate sase-core 0.5.0 into SASE

Update `pyproject.toml` from the stale 0.4 compatibility window to `sase-core-rs>=0.5.0,<0.6.0`, then regenerate the
corresponding `uv.lock` entry from the published distribution. Do not change the Rust workspace version: release-plz
already produced the canonical release commit.

Verify both installation modes:

1. Use a disposable environment without the linked `sase-core` checkout to prove normal dependency resolution installs
   the published 0.5.0 wheel and the SASE placeholder facade can call both new bindings.
2. Run `just install`, the SASE core-version validator, and `just check` in the normal linked development workspace.
   Re-run the focused ACE placeholder tests and PNG snapshots if a failure needs diagnosis.

Open `sase-nvim` through `/sase_repo` and run the headless `tests/lsp_placeholder_smoke.lua` test against the v0.5.0 LSP
source/binary, covering the `<` trigger, document-order candidates, filtering, text edits, empty results, and the `cbi`
snippet retrigger command. Re-check the post-start Artifacts Plans tests to ensure the dependency/lock update did not
regress the two non-epic commits that landed during the work.

## Phase 3: Close and clean up epic sase-6b

This is the final phase and must run only after Phases 1-2 are green.

1. Re-run `sase bead show sase-6b` and each child show command, then close the epic with `sase bead close sase-6b`.
2. After the close, run `just symvision`. Follow the audited Symvision hierarchy: remove any now-stale `sase-6b(...)`
   epic-symbol entries, delete genuinely unused code, or privatize same-file-only symbols; do not keep production APIs
   alive solely through tests or add replacement whitelists. If cleanup changes SASE files, run `just check` again.
3. Open the plans sidecar through `/sase_repo` and change only the linked epic plan `202607/placeholder_completion.md`
   frontmatter from `status: wip` to `status: done`.
4. Verify the epic now shows closed, all three children remain closed, the linked plan reports `status: done`, every
   touched repository is clean apart from the intended landing changes, and the final SASE and relevant cross-repo tests
   pass.

The landing is not complete if the PyPI artifact is unavailable, the SASE dependency still admits the pre-feature 0.4.1
wheel, Symvision has not been run after bead closure, or the original epic plan remains WIP.
