---
tier: tale
title: Build the sase-research plugin
goal:
  The sase-research repository ships a tested installable plugin whose artifact
  providers, xprompts, and defaults are discoverable from a clean wheel.
size: medium
proposed_by: bbugyi200.athena.sase-js.8
bead: sase-js.8
create_time: 2026-08-11 16:29:29
status: wip
---

- **PARENT:**
  [202608/artifact_ref_contract.md](https://github.com/sase-org/sase--plans/blob/main/202608/artifact_ref_contract.md)
- **BEAD:**
  [sase-js.8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-js/sase-js.8.md)

# Plan: Build the `sase-research` plugin

## Goal

Turn the one-commit `sase-org/sase-research` repository into an installable,
release-ready SASE plugin that owns the `research` artifact-reference provider, the
`research-highlights` file-hook template, and the five live `#research*` xprompts. Prove
that all four entry-point groups and every package resource work from a built wheel in a
clean environment.

## Context and constraints

- Implement only phase bead `sase-js.8`; the later adoption phase owns linking the repo
  from SASE, installing it into Bryan's environment, changing project/home
  configuration, and removing the old chezmoi definitions.
- Work in the repository path returned by `sase repo open gh:sase-org/sase-research`.
  Use `sase-telegram` as the packaging/release structure reference and `sase-github` as
  the entry-point and coordinated-source CI reference.
- Match the artifact-provider contract already landed on SASE `master`, including
  `expansion_format`, `inventory.globs`, per-property `source`, and file-hook
  `required: [command]`. The reference inventory intentionally includes swarm drafts
  while the highlights hook intentionally excludes them.
- Lift the current `research`, `research/image`, `research/more`, `research/prompt`, and
  `research_swarm` definitions from chezmoi without deleting or editing chezmoi. Do not
  package `old_research_swarm`.
- Package defaults for the three model aliases, the `researchers` bucket, and the
  `research` tribe so `#research_swarm` is self-contained on a fresh install while
  retaining normal user-config override precedence.
- Disambiguate `sase-research` (plugin code) from `sase--research` (content sidecar) in
  the README and GitHub repository description. The later adoption phase owns the
  corresponding descriptions in SASE's `sase.yml`.
- Keep Python support at 3.12+ with ruff, strict mypy, and pytest configured with strict
  markers/config. Depend on the minimum SASE version that introduces the two artifact
  provider groups, and coordinate CI against SASE and sase-core source revisions until
  that release is available from the package index.

## Implementation

1. Scaffold the package and developer workflow.
   - Add `pyproject.toml` using hatchling and a `src/sase_research` layout.
   - Register `research` in `sase_artifact_refs`, `research-highlights` in
     `sase_file_hooks`, and `sase_research` in both `sase_xprompts` and `sase_config`.
   - Add package metadata, typed package markers where appropriate, a Justfile,
     `.gitignore`, MIT license, changelog, and contributor-facing agent/build
     instructions.

2. Implement immutable provider specifications.
   - Export `RESEARCH_REF_PROVIDER` and `RESEARCH_HIGHLIGHTS_HOOK` from
     `sase_research.provider` as pluggy-compatible provider objects.
   - Define the schema-version-1 `research` document spec with the exact inventory,
     expansion, frontmatter properties, detail ordering, VCS permalink policy, and
     Referenced By projection required by the epic design.
   - Define the schema-version-1 highlights template with its distinct sidecar/path/
     agent/operation filters, 120-second timeout, no command value, and
     `required: [command]`.

3. Package prompts and defaults.
   - Add four Markdown resource prompts for the config-backed research helpers, using
     explicit frontmatter names for slash-qualified prompts.
   - Add the four-segment `research_swarm.md` resource with its clan names, wait/fork
     dependencies, report consolidation rules, and image handoff.
   - Add `default_config.yml` with the three research model aliases, shared bucket, and
     tribe presentation configuration.

4. Add focused contract tests.
   - Validate both provider specs through SASE's registry and test installed
     discovery/provenance, duplicate-provider and missing-provider diagnostics,
     required-command resolution, list-replacing inline overrides, and inline-vs- `use:`
     normalized-spec/digest parity.
   - Exercise the shared artifact path-filter semantics against real temporary fixtures
     to prove the ref/hook glob divergence.
   - Parse representative Markdown frontmatter into the declared typed property fields.
   - Load all packaged xprompts and defaults through SASE's public/plugin loaders; prove
     that the swarm has four top-level segments and that its wait and fork targets
     preserve the intended dependency graph.
   - Build sdist and wheel, install the wheel into a clean temporary environment,
     enumerate all four entry-point groups, and assert every xprompt/default-config
     resource is discoverable without relying on the source tree.

5. Add CI and release automation.
   - Run lint on Python 3.12 and tests across supported Python versions, with a separate
     minimum-supported-SASE compatibility lane.
   - Build both distributions and run the clean-wheel contract before publishing.
   - Add release-please metadata and trusted PyPI publication gated on the wheel smoke
     test, plus a conventional PR-title check.

6. Document the plugin.
   - Make the README's opening paragraph distinguish plugin code from the
     `sase--research` content repository, document installation, the four entry points,
     provider configuration, xprompts/defaults, and the intentional glob divergence.
   - Add concise architecture, configuration, xprompt, and development/release docs.
   - Update the GitHub repository description to the same unambiguous positioning.

## Verification

1. Run the repository formatter, then its complete `just check` gate.
2. Build sdist and wheel and inspect their member lists for provider code,
   `default_config.yml`, and all five Markdown xprompt resources.
3. Install the wheel into a fresh temporary virtual environment alongside the
   coordinated SASE/sase-core sources; enumerate and load all four entry points,
   validate both provider specs, load the defaults, and enumerate all five xprompts.
4. Re-run `just check` after any packaging or documentation corrections.
5. Confirm the Git worktree contains only this phase's intended files and confirm the
   GitHub description distinguishes the plugin from `sase--research`.
6. Close only `sase-js.8` with a note summarizing the passing checks and wheel smoke
   results; record any out-of-scope discovery as a `PROPOSED FOLLOW-UP:` note instead of
   creating a bead.
