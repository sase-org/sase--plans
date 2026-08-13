---
tier: epic
title: Complete the sase-research-artifacts repository rename
goal: 'The renamed research-artifacts plugin has one coherent repository, distribution,
  module, release, catalog, and linked-repository identity; SASE still exposes the
  existing research artifact-reference, hook, and xprompt contracts; obsolete warnings
  that distinguish the plugin from the sase--research content sidecar are gone; and
  the renamed plugin is installable and verified under its new name.

  '
phases:
- id: plugin-identity
  title: Rename the plugin's package and repository-facing identity
  depends_on: []
  size: medium
  description: 'plugin-identity: migrate the renamed repository''s distribution, import
    package, entry-point targets, release metadata, automation, tests, documentation,
    and GitHub description while preserving its research-facing feature IDs.'
- id: host-wiring
  title: Rewire SASE to the renamed linked repository and plugin
  depends_on:
  - plugin-identity
  size: small
  description: 'host-wiring: update SASE''s linked-repository configuration, provider
    provenance fixtures, artifact-reference documentation, and generated memory outputs
    to use the new identity without the obsolete sidecar-name warning.'
- id: integration-cutover
  title: Verify the catalog cutover and restore the plugin
  depends_on:
  - plugin-identity
  - host-wiring
  size: small
  description: 'integration-cutover: refresh the live catalog, exercise linked-repository
    and install resolution, install the renamed plugin from Git, and prove the old
    distribution identity is absent while the existing research contracts work.'
proposed_by: bbugyi200.athena.zt
create_time: 2026-08-13 14:11:56
status: done
bead_id: sase-l2
---

- **PROMPT:** [prompts/202608/research_artifacts_rename.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/research_artifacts_rename.md)
- **BEAD:** [sase-l2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-l2/README.md)

# Plan: Complete the `sase-research-artifacts` repository rename

## Context and findings

The GitHub repository has already moved from `sase-org/sase-research` to
`sase-org/sase-research-artifacts`, and its `origin` uses the new URL. SASE's live
GitHub-topic catalog therefore exposes it as plugin `research-artifacts`, with
repository and full-name keys `sase-research-artifacts` and
`sase-org/sase-research-artifacts`. The old plugin is currently uninstalled.

The cutover is incomplete in two repositories:

- The renamed plugin repository still declares the distribution `sase-research`, keeps
  its code/resources in `sase_research`, points project URLs at the old GitHub slug,
  brands its release component and automation with the old name, and explains at length
  that it is not `sase--research`. Its live GitHub description repeats that warning.
- The SASE repository still registers a lazy linked repo named `sase-research` at
  `../sase-research`, renders the old name and warning into its canonical/generated
  memory, names the old plugin in artifact-reference documentation, and uses the old
  distribution in provider-provenance tests.

This is not merely cosmetic. For a catalog entry, the default index install requirement
is the repository slug. Today `sase plugin install research-artifacts` therefore
resolves to `sase-research-artifacts`, while the repository actually builds
`sase-research`. Neither name has a PyPI project yet, so there is no released
compatibility contract to preserve and the identities can be aligned before the first
release.

No stale rename references were found in the configured `chezmoi` repository, and there
is no separate enabled, disabled, or sibling SASE project lifecycle record to migrate.
The `sase--research` sidecar name itself remains correct.

## Identity contract

Use these names consistently after the migration:

| Layer                                         | Final identity                                                                       |
| --------------------------------------------- | ------------------------------------------------------------------------------------ |
| GitHub repository and SASE linked-repo name   | `sase-research-artifacts`                                                            |
| Python distribution / install requirement     | `sase-research-artifacts`                                                            |
| Python import and packaged-resource namespace | `sase_research_artifacts`                                                            |
| Short catalog/plugin name                     | `research-artifacts`                                                                 |
| Artifact-ref provider and document kind       | `research` / `@research`                                                             |
| File-hook provider                            | `research-highlights`                                                                |
| Xprompts and model/tribe concepts             | `#research*`, `research_a`, `research_b`, `research_lead`, `researchers`, `research` |
| Durable content sidecar                       | `sase--research`                                                                     |

The first four rows are repository/package identity and must be renamed. The remaining
rows are functional user contracts and must not change. It is still useful to explain
where durable reports live when architecture or configuration requires that fact, but
remove warning-style prose about confusing two similarly named repositories, including
"not the content repo", "note the double hyphen", and equivalent agent guidance.

## Phase `plugin-identity` — Rename the plugin's package and repository-facing identity

Open `gh:sase-org/sase-research-artifacts` through `/sase_repo`; use only the path that
command returns.

1. Change `[project].name` and the `Repository`/`Issues` URLs in `pyproject.toml` to
   `sase-research-artifacts` and the renamed GitHub URL. Rename the source/resource
   package from `src/sase_research` to `src/sase_research_artifacts`, then update
   imports, module docstrings, hatch build configuration, and all four entry-point
   targets and resource-module values. Keep the provider IDs and research-facing
   entry-point names unchanged.
2. Rename repository/distribution identity in the test fixtures and contract tests.
   Strengthen the wheel contract so it asserts the built distribution metadata and
   archive/resource paths use `sase-research-artifacts` / `sase_research_artifacts`,
   while still proving that SASE discovers the `research` ref provider,
   `research-highlights` hook, default config, and all five xprompts.
3. Update release and development surfaces: the release-please component, workflow step
   labels and smoke-test imports, Justfile branding, task comments, override/resolved
   environment-variable prefixes, and any references in tests that assert those names.
   Keep the existing release manifest version unless the release tooling itself requires
   a rename-specific change; there is no published package/version to deprecate.
4. Rebrand `README.md`, `AGENTS.md` (and any repo-local instruction shim that mirrors
   it), architecture/configuration docs, and source docstrings. Installation examples
   must say `pip install sase-research-artifacts`. Replace the old disambiguation
   warning with concise positive descriptions of the plugin; mention `sase--research`
   only where the actual content-sidecar relationship is relevant.
5. Update the live GitHub repository description to a positive description of the
   research artifact-reference, highlights, config, and xprompt plugin without the old
   parenthetical warning. Preserve the `sase--plugin` topic, visibility, default branch,
   Actions configuration, and repository name already established by the user.
6. Audit the complete tree for `sase-research`, `sase_research`, and old GitHub URLs.
   Every remaining match must be either the new longer identity or an explicitly
   justified historical record; do not alter `sase--research` or the functional IDs in
   the identity table.

Verification for this phase:

- Run `just install`, `just check`, and the real build/install lane `just test-wheel` in
  the plugin repository.
- Build artifacts must be named for `sase_research_artifacts`, report distribution
  metadata `sase-research-artifacts`, import `sase_research_artifacts`, and expose the
  existing SASE entry points/resources.
- Query the GitHub repository metadata and confirm the new URL/name and cleaned
  description, with the plugin topic still present.

## Phase `host-wiring` — Rewire SASE to the renamed linked repository and plugin

Work in the SASE repository after `plugin-identity` has landed.

1. In `sase/sase.yml`, rename the linked repository to `sase-research-artifacts`, change
   its sibling path to `../sase-research-artifacts`, and replace the old warning-heavy
   description with a direct description of the plugin. Simplify the `research` sidecar
   description so it describes durable research reports/media and their workflow role
   without contrasting repository names.
2. Update `docs/artifact_references.md` and any other current documentation that names
   the plugin/distribution to `sase-research-artifacts`. Preserve examples and links
   that correctly refer to the `sase--research` content repository.
3. Update synthetic provider-discovery and file-hook fixtures whose `package` provenance
   models this plugin so assertions expect `sase-research-artifacts`. Do not rename the
   `research` provider, `research-highlights` hook, `@research` document kind, xprompts,
   sidecar role, or unrelated historical examples.
4. The user's request explicitly authorizes the canonical SASE memory update needed to
   remove the obsolete agent warning. After changing repository configuration, run
   `sase memory init` as required so `sase/memory/sase.md`, `AGENTS.md`, the provider
   instruction shims, and the memory README are regenerated from their canonical inputs;
   do not hand-edit generated shims independently.
5. Audit the SASE tree for stale old repository/package identities. Remaining
   `sase--research` references are expected; remaining exact `sase-research` references
   must be either part of `sase-research-artifacts` or an intentionally preserved
   historical changelog record.

Verification for this phase:

- Run `just install` before repository checks, then run `just check` as required for all
  SASE file changes. Escalate to `just check-full` through `/sase_monitor` if scoped
  selection requests it or reports unusual coverage.
- Run `sase repo list --json` and confirm the linked row is named
  `sase-research-artifacts`, has the expected sanitized environment name, and no old
  linked row is rendered from current configuration.
- Open the renamed linked repo through
  `sase repo open sase-research-artifacts -r "Verify the completed research-artifacts rename"`
  and confirm it resolves to the renamed checkout.
- Confirm regenerated instructions describe `sase-research-artifacts` and
  `sase--research` directly without any similarity/double-hyphen warning.

## Phase `integration-cutover` — Verify the catalog cutover and restore the plugin

Perform this only after both repository changes are committed and available at their
canonical remotes, so installation exercises the same source users will receive.

1. Refresh the live catalog and assert there is exactly one built-in entry for this
   plugin: short name `research-artifacts`, repo `sase-research-artifacts`, full name
   `sase-org/sase-research-artifacts`, cleaned description, and no stale `sase-research`
   catalog entry.
2. Dry-run both catalog resolution modes. The default/index form must resolve to the
   `sase-research-artifacts` distribution name, even though it remains unavailable until
   a first PyPI release; `--git` must resolve to the renamed GitHub URL. Creating or
   publishing that first PyPI release is outside this rename.
3. Because the user uninstalled the old plugin as part of this cutover, install the
   renamed plugin from Git with the catalog's `research-artifacts` name. Run this as the
   final environment mutation because SASE restarts the axe daemon after a real plugin
   change.
4. In the restarted environment, verify `sase plugin list/show` reports the renamed
   plugin as installed, its live distribution provenance is `sase-research-artifacts`,
   and the old `sase-research` distribution is absent from both installed metadata and
   the uv tool receipt. Run `sase doctor -C config.repos` and a focused provider/xprompt
   discovery smoke check to prove the configured `research` sidecar regains `@research`,
   `research-highlights`, and `#research*` behavior.

## Completion criteria

- Repository, distribution, module, release, catalog, and linked-repo identities agree
  on `sase-research-artifacts` (with underscore normalization for Python imports).
- No current user or agent guidance warns about confusing the plugin repository with
  `sase--research`; the content sidecar retains its correct name and behavior.
- Both repositories pass their required checks, including the plugin's real wheel
  contract and SASE's post-memory-init `just check` lane.
- Catalog lookup, Git installation, installed metadata, provider discovery, and SASE
  repository diagnostics all succeed under the new name, with no installed old
  distribution left behind.

## Non-goals and safeguards

- Do not rename the `research` sidecar, provider IDs, reference syntax, hook ID,
  xprompts, model aliases, bucket, or tribe.
- Do not add compatibility aliases for the unreleased old distribution/module unless a
  concrete external consumer is discovered during implementation; record that evidence
  on the phase bead before expanding scope.
- Do not publish a PyPI release, create a PyPI project, or change repository visibility,
  branch policy, topics, secrets, or environments as part of this rename.
- Preserve unrelated dirty changes in either repository and use `/sase_repo` before all
  cross-repository reads or writes.
