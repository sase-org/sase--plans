---
tier: epic
title: Make research an ordinary filtered custom sidecar
goal:
  Move Bryan's research-sidecar policy into personal configuration, exclude
  double-underscore Markdown swarm drafts from artifact-reference resolution and
  completion, and remove research-role special cases from SASE and sase-core.
phases:
  - id: generic-core-document-corpora
    title: Remove the research-only document fallback from sase-core
    depends_on: []
    size: medium
    description:
      "generic-core-document-corpora: make explicit document corpora the authority for
      custom plan-search kinds, retain only reserved plans behavior in the fallback
      path, update Rust coverage, and publish the compatible core release."
  - id: generic-sidecar-lifecycle
    title: Generalize SASE sidecar configuration and initialization
    depends_on: []
    size: medium
    description:
      "generic-sidecar-lifecycle: remove research defaults and presentation presets from
      repository configuration and initialization while preserving explicitly configured
      roles and custom sidecar-owned content."
  - id: generic-document-consumers
    title: Derive search, protection, environment, and UI behavior from roles
    depends_on:
      - generic-core-document-corpora
    size: medium
    description:
      "generic-document-consumers: consume the released core behavior and derive
      document corpora, retention scans, role environments, and ACE kind completion from
      generic configured and resolved sidecar roles."
  - id: personal-research-policy
    title: Move the research role and filter into Bryan's personal config
    depends_on:
      - generic-sidecar-lifecycle
      - generic-document-consumers
    size: small
    description:
      "personal-research-policy: declare research in the chezmoi-managed user config
      with shared Markdown include/exclude globs, remove the public project-local
      declaration, and verify effective @research completion policy."
  - id: migration-docs-and-verification
    title: Document the generic contract and verify the combined tree
    depends_on:
      - generic-sidecar-lifecycle
      - generic-document-consumers
      - personal-research-policy
    size: medium
    description:
      "migration-docs-and-verification: remove claims of built-in research behavior,
      document explicit custom-role migration, exercise cross-surface filter behavior,
      and run exhaustive SASE, core, and visual verification."
proposed_by: bbugyi200.athena.w7
create_time: 2026-09-09 20:00:07
status: wip
---

- **PROMPT:**
  [prompts/202608/generic_research_sidecar_filters.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/generic_research_sidecar_filters.md)

# Make research an ordinary filtered custom sidecar

## Outcome

Bryan's `research` document role is declared in his chezmoi-managed SASE config, not
seeded or otherwise privileged by the public SASE codebase. Its artifact-reference
policy accepts Markdown documents except for basenames containing two consecutive
underscores:

```yaml
repos:
  sidecar:
    custom:
      research:
        description: Durable SASE research reports and generated media.
        ref:
          filters:
            path_globs:
              - "**/*.md"
              - "!**/*__*.md"
```

The positive glob deliberately preserves the current Markdown-only default. The veto
removes swarm intermediates such as `topic__a.md`, `topic__b.md`, and any other Markdown
basename containing `__`, at any directory depth. Consolidated `topic.md` reports and
root-level Markdown remain available. Because the existing artifact reference contract
applies `path_globs` in the Rust backend, this one policy governs exact resolution,
drift repair, canonicalization, `@research:` completion, and `#ref/research:` argument
completion.

The public implementation continues to support `research` as an ordinary configured
custom role, exactly as it supports `designs`, `notes`, or any other valid document
sidecar. It no longer creates, advertises, scans, or renders `research` merely because
of that name.

## Evidence and current state

- The screenshot at `.sase/artifacts/home/tmp/screenshots/20260808_190525.png` shows
  `@research:` completion offering `glm_5_2_sase_rollout__a.md`,
  `glm_5_2_sase_rollout__b.md`, and similar first-agent swarm drafts beside their
  consolidated report.
- Epic `sase-ho` already delivered the required generic filter contract. Its plan
  specifies positive/negative `repos.sidecar.*.<role>.ref.filters.path_globs`, and
  phases `sase-ho.2`, `sase-ho.4`, and `sase-ho.5` report completed config, resolution,
  TUI/LSP completion, and end-to-end coverage.
- `sase xprompt show ref/research -f json` currently reports only the default
  `ref_path_globs: ["**/*.md"]`, proving the sidecar has no effective veto yet.
- A direct call through `filter_artifact_ref_paths` with `["**/*.md", "!**/*__*.md"]`
  accepts `README.md` and `202608/topic/topic.md`, while filtering `topic__a.md`,
  `topic__b.md`, other double-underscore Markdown, and non-Markdown files.
- The effective `research` declaration currently comes from the public project file
  `sase/sase.yml`; Bryan's chezmoi source does not declare the sidecar. The user layer
  already owns the related research xprompts, tribe display, and research-specific
  file-hook policy, so it is the appropriate owner for this repository convention.
- The audit found name-based behavior in both repositories: SASE repo init seeds a
  `research` custom entry; SASE ships and drift-manages a research README and image;
  retention scans `research` as the only optional document role; legacy plan search and
  ACE kind completion expose it statically; and sase-core's implicit plan corpus is
  hard-coded to `plans` plus `research`.
- The checked-out research sidecar already owns byte-identical copies of the specialized
  README and `assets/research-directory-map.png`, so removing the packaged SASE preset
  does not discard the user-facing repository documentation.

## Design constraints

- Keep the matcher and every allow/filter decision in `sase-core`; do not duplicate glob
  semantics in Python, the TUI, LSP, or personal scripts.
- Do not add a `research` branch, constant, default, implicit repo, or preset elsewhere
  to make this filter work. Custom roles must flow through the existing sidecar role
  registry and artifact-reference context.
- Keep reserved behavior only for `plans`, `beads`, and `agents`. `plans` is a document
  corpus; `beads` and `agents` have non-document artifact kinds.
- Preserve existing explicitly configured research sidecars and recorded sidecar stores.
  The migration removes implicit creation/defaulting; it must not disable, delete,
  rename, or relocate an existing configured repository.
- Existing custom sidecar content is owner-authored. SASE may seed a generic README when
  a new custom repo is empty, but subsequent `sase repo init` runs must not overwrite or
  drift-manage that README. Reserved sidecar scaffolds may remain managed according to
  their contracts.
- Concrete `research` strings may remain in tests and docs as clearly labeled examples
  of an arbitrary custom role, or where they describe Bryan's personal workflow. Remove
  only name-based product behavior and claims that the role is built in, default-seeded,
  or specially presented.
- Do not change the artifact-reference wire or filter semantics delivered by `sase-ho`;
  this work configures and generalizes consumers of that contract.

## Phase 1: Remove the research-only document fallback from sase-core

**Slug:** `generic-core-document-corpora`

### Work

- In the linked `sase-core` repository, remove `research` from the implicit `plan::read`
  corpus and its tier-classification branch. When callers provide `document_corpora`,
  continue to accept any role label and scan its flat, sharded, and one-level bundle
  Markdown layouts exactly as today.
- Narrow the no-corpora compatibility path to the reserved plans corpus, or make the
  absence of explicit corpora mean plans-only, so a custom role is never inferred by
  name. Update comments and wire documentation that currently describe
  `tale`/`epic`/`research` as an intrinsic set.
- Replace Rust tests that assert implicit `sdd/research` discovery with tests proving:
  plans still work without explicit corpora; an arbitrary explicit role such as `notes`
  or `designs` is classified by its supplied label; and an unconfigured `research`
  directory is not special.
- Keep `research` artifact-reference and LSP examples where they exercise arbitrary
  strings rather than role privilege; rename examples only when doing so clarifies the
  distinction.
- Release the backward-compatible `sase-core-rs` patch/minor needed by SASE and record
  the minimum compatible version for Phase 3. No schema bump is needed unless the
  implementation actually changes a serialized wire.

### Exit criteria

- No production Rust branch or constant treats `research` as an implicit document
  corpus.
- Explicit custom corpora retain their current search behavior and kind labels.
- `cargo fmt --check`, `cargo clippy --all-targets --all-features -- -D warnings`, and
  the full core workspace test suite pass before the release is consumed.

## Phase 2: Generalize SASE sidecar configuration and initialization

**Slug:** `generic-sidecar-lifecycle`

### Work

- Remove `DEFAULT_RESEARCH_DESCRIPTION` and its public re-export. Change repository
  initialization to seed only the reserved managed roles (`plans`, `beads`, and hidden
  `agents`), preserve every user-authored `repos.sidecar.custom` entry, and update
  summaries/help so they describe "configured custom sidecars" instead of a fixed
  research sidecar.
- Remove the name-keyed research presentation preset from `sdd._paths` and
  `sdd._init_files`, including the packaged research README, directory-map image, and
  image prompt. Keep generic custom-sidecar initialization, but change it to create a
  README only when absent and never overwrite an existing custom README. Retain drift
  management for reserved sidecar scaffolds. Generalize internal-guide detection where
  necessary rather than replacing the research name with another custom name.
- Remove the public `research` custom declaration from `sase/sase.yml`; Phase 4 will
  provide it from Bryan's user layer. Do not modify any SASE memory file.
- Update focused tests for repository-init config writes/idempotence, generic custom
  README ownership, and path/internal-guide behavior. Use arbitrary custom roles in
  behavior tests so future regressions cannot pass through a renewed research special
  case.

### Exit criteria

- A source audit of repository configuration, initialization, and packaged SDD resources
  finds no research-role default or presentation preset.
- A globally or locally configured `research` role continues to materialize through
  generic role handling.
- `sase repo init --check` neither inserts research into a project without that custom
  role nor overwrites the README of an existing custom sidecar.
- `plans`, `beads`, and `agents` keep their reserved lifecycle and scaffolding behavior.

## Phase 3: Derive document consumers from configured roles

**Slug:** `generic-document-consumers`

### Work

- Raise SASE's `sase-core-rs` lower bound and lockfile to the release from Phase 1, then
  make `plan_search.facade` pass explicit corpora for both split and legacy/local
  stores. Build the role list from the configured sidecar registry plus the resolved
  store, include reserved `plans`, exclude non-document `beads`/`agents`, and stop
  adding `research` in `available_kinds` when no role is configured.
- Generalize artifact retention protection: scan every materialized custom document
  sidecar opportunistically for durable `file:` references, keep plans/beads required
  according to the current fail-safe contract, exclude non-document agents data from
  this document scan, and deduplicate aliases/clones without hard-coding custom role
  names. Add a `designs`/`notes` regression and prove a missing custom sidecar is
  optional.
- Remove the unused `SASE_SDD_RESEARCH_DIR_ENV` symbol; continue generating
  `SASE_SDD_<ROLE>_DIR` uniformly, so a configured research role still receives the same
  environment variable through the generic path.
- Remove `research` from ACE's static plan-kind completion and hint text. Keep only
  intrinsic kinds static and inject all configured/observed document kinds through
  `PlanFilterBar.set_completion_sources`; verify both a configured arbitrary role and
  the absence of unconfigured research. Update visual snapshots if the intentional
  hint/menu text changes.
- Update focused tests for plan corpus discovery, environment variables, retention
  protection, and ACE completion. Use arbitrary custom roles so future regressions
  cannot pass through a renewed research special case.

### Exit criteria

- A source audit of production consumers finds no research-only fallback, optional scan
  list, static UI option, or exported env constant.
- A globally or locally configured `research` role continues to resolve, search, and
  expose `SASE_SDD_RESEARCH_DIR` through generic role handling; an unconfigured role
  receives none of those behaviors.
- `designs` or another arbitrary custom role receives the same search, protection,
  environment, and dynamic-completion behavior as an equivalently configured `research`
  role.

## Phase 4: Move the research role and filter into Bryan's personal config

**Slug:** `personal-research-policy`

### Work

- Open the linked `chezmoi` repository through `sase repo open` and add
  `repos.sidecar.custom.research` to `home/dot_config/sase/sase.yml`, alongside the
  existing user-owned research xprompts, tribe, and file-hook conventions. Preserve the
  existing description and configure exactly:

  ```yaml
  ref:
    filters:
      path_globs:
        - "**/*.md"
        - "!**/*__*.md"
  ```

- Keep the renderer omitted unless Bryan already has user-specific wording to add; the
  generated generic renderer remains appropriate and the config should own only policy
  that differs from the default.
- Validate the chezmoi source YAML and SASE schema before applying it. Materialize the
  targeted config through the normal chezmoi workflow, without overwriting unrelated
  local dotfile changes, and clear/reload SASE's config cache as needed for runtime
  checks.
- Confirm the merged config contains the research role exactly once and that
  `sase xprompt show ref/research -f json` reports both globs in order. Exercise the
  shared resolver/filter API against real sidecar-relative candidates and confirm:
  `topic.md` is allowed; `topic__a.md`, `topic__b.md`, and any Markdown basename with
  `__` are filtered; non-Markdown remains filtered by the positive pattern.
- Verify `@research:` and `#ref/research:` completion return the same inventory and do
  not include the screenshot examples. Verify an exact filtered reference returns the
  stable `filtered` status rather than resolving or drifting to another candidate.
- Leave the research sidecar's existing README and directory-map image in place as
  repository-owned content. Do not introduce a sidecar-local configuration format;
  `repos.sidecar.custom.research.ref` in Bryan's user config is the supported owner of
  this policy.

### Exit criteria

- The durable personal config, not the SASE repository or a built-in default, declares
  the research sidecar and its filter.
- Both artifact-reference completion syntaxes hide every `*__*.md` research draft, while
  consolidated Markdown remains selectable and resolvable.
- The research repository's specialized README/image remain unchanged and future init
  runs do not claim ownership of them.

## Phase 5: Document migration and verify the combined behavior

**Slug:** `migration-docs-and-verification`

### Work

- Update SASE configuration, initialization, SDD/storage, plan-search, editor, and
  artifact-retention documentation to say that only reserved roles are built in; custom
  document roles are explicit configuration. Remove claims that repo init seeds research
  or that SASE ships a research presentation preset.
- Retain `research` snippets only as examples clearly declared under
  `repos.sidecar.custom`, and show the include-plus-veto filter example. Document that
  moving an old implicit `sdd/research` corpus forward requires declaring the custom
  role; recorded/configured repositories continue to work without data migration.
- Update CLI help and generated text that statically list research as an intrinsic plan
  kind. Describe configured document kinds dynamically or use a neutral custom-role
  example.
- Add/retain an end-to-end test that builds two custom roles with different filters and
  proves resolution, `@` completion, `#ref/` completion, and plan-kind discovery all
  consume role-specific configuration without name-based behavior.
- Re-scan production SASE and sase-core sources for behavioral `research` branches.
  Classify remaining hits as personal/example prose or ordinary arbitrary-string test
  data; eliminate any remaining product default or conditional.
- Run `just install` before repository verification. Because this changes broad
  storage/config/search behavior and the linked core, run SASE's focused tests,
  `just check-full`, and `just test-visual` for the ACE completion change. In sase-core
  run formatting, clippy, and the full workspace suite against the released version.
  Validate the chezmoi YAML and re-run the real merged-config/filter checks after all
  repositories are combined.

### Exit criteria

- Public docs and help distinguish reserved sidecars from arbitrary configured document
  roles and contain no default-seeded/preset research claim.
- Cross-role tests fail if any consumer bypasses config or gives `research` behavior
  that an equivalently configured `designs` role does not receive.
- Full SASE, core, and visual verification passes, and the final runtime completion
  inventory matches the requested screenshot cleanup.

## Expected files and repositories

The exact split may evolve during implementation, but the expected ownership is:

- `sase-core`: `crates/sase_core/src/plan/read.rs` and its plan read/search tests.
- SASE: `src/sase/plan_search/facade.py`, `src/sase/main/_repo_init_config.py`,
  `src/sase/main/repo_init_handler.py`, `src/sase/_linked_repo_config.py`,
  `src/sase/linked_repos.py`, `src/sase/sdd/_init_files.py`, `src/sase/sdd/_paths.py`,
  `src/sase/sdd/env.py`, `src/sase/core/artifact_file_protection.py`, the ACE plan
  filter, package resources, `sase/sase.yml`, focused tests, and affected public
  docs/help.
- `chezmoi`: `home/dot_config/sase/sase.yml` for Bryan's explicit custom role and
  `ref.filters.path_globs` policy.
- `research`: no content change is expected; its existing README and image are the
  durable repository-owned copies used to justify deleting SASE's packaged preset.

## Rollback and compatibility

- Removing the personal filter is a one-block config rollback; the default `**/*.md`
  inventory returns immediately after config reload.
- Existing recorded research sidecars are not deleted or rewritten. If a deployment
  needs the old implicit legacy corpus temporarily, it can declare
  `repos.sidecar.custom.research` explicitly before upgrading.
- The core change is consumed only after a released compatible wheel exists. Do not land
  a SASE dependency bound that cannot be installed on supported platforms.
- If exhaustive verification exposes an unrelated failure, follow the project task
  workflow rather than weakening these cross-role assertions.
