---
tier: epic
title: Singular xprompt skill references and builtin source exception
goal: 'Xprompt-backed skills expand through the singular #skill/ namespace while user
  and plugin sources remain in plural skills/ directories and bundled Markdown skill
  sources live only under src/sase/xprompts/skills/.

  '
phases:
- id: core_skill_reference_contract
  title: Shared skill reference and directory contracts
  depends_on: []
  size: medium
  description: 'core_skill_reference_contract: separate physical skill directories
    from singular xprompt references across the Rust catalog, bindings, editor, and
    gateway contracts.'
- id: builtin_skill_source_exception
  title: Python builtin source layout and loading
  depends_on:
  - core_skill_reference_contract
  size: medium
  description: 'builtin_skill_source_exception: move only bundled Markdown skill assets
    beneath xprompts and update Python discovery, placement, packaging, and deployment
    guards.'
- id: skill_surface_cutover
  title: User-facing cutover and end-to-end verification
  depends_on:
  - core_skill_reference_contract
  - builtin_skill_source_exception
  size: medium
  description: 'skill_surface_cutover: align CLI, ACE, LSP-facing fixtures, documentation,
    visual snapshots, and full repository checks with the singular reference contract.'
proposed_by: bbugyi200.athena.sase-hf.land.w2
create_time: 2026-08-08 11:49:49
status: wip
bead_id: sase-hi
---

- **PROMPT:** [prompts/202608/xprompt_skill_singular_namespace.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/xprompt_skill_singular_namespace.md)
- **BEAD:** [sase-hi](https://github.com/sase-org/sase--beads/blob/main/pages/sase-hi/README.md)

# Plan: Singular xprompt skill references and builtin source exception

## Context

The recent canonical skill-source cutover correctly requires project and home skills to
be authored in `sase/skills/` directories and plugin skills in plugin `skills/`
resources. It also coupled that plural physical directory segment to the xprompt
reference namespace and moved bundled Markdown sources into `src/sase/skills/`, where
they now share a package with the Python implementation of the `sase skill` commands.

This follow-up deliberately separates those concerns:

- A skill named `foo` is expanded inline as `#skill/foo`. A project-qualified skill is
  expanded as `#<project>/skill/foo`.
- Provider-native invocation remains `/foo`, and generated provider targets such as
  `~/.codex/skills/foo/SKILL.md` remain unchanged.
- Project, home, project-specific-home, and plugin source directories remain plural
  `skills/` directories. The singular spelling is a reference namespace, not a
  filesystem migration.
- Bundled Markdown skill sources and `SKILL.frame.template.md` are the one source-layout
  exception: they live in `src/sase/xprompts/skills/`. Python command modules remain in
  `src/sase/skills/`; do not move or duplicate those `.py` files.
- This is a hard reference cutover. Do not retain a `#skills/foo` alias or silently
  rewrite the old namespace. Tests and diagnostics should demonstrate that only
  `#skill/foo` resolves, while `/foo` continues to resolve the same definition.

The shared naming behavior belongs in `sase-core`. Open that linked repository through
`/sase_repo` before working in it. Because its release versions are managed by
release-plz, do not hand-edit crate versions. Land and release the breaking core
contract before updating the Python package to require the compatible released binding
series; use breaking-change commit metadata in the core repository so release-plz can
choose the version.

## Phase 1: Shared skill reference and directory contracts

In `sase-core`, split the current overloaded `SKILL_NAMESPACE_SEGMENT` concept into
explicit contracts for the plural physical source directory (`skills`) and the singular
xprompt reference namespace (`skill`). Use the physical segment when constructing
project, home, project-specific-home, and plugin/package source locators. Use the
reference segment only in `skill_reference_name` and `split_skill_reference_name`,
yielding `skill/foo` and `<project>/skill/foo`. Bump the content-layout schema because
the serialized package source locator and reference contract change, and advertise the
builtin symbolic source as `package:xprompts/skills` while leaving plugin resources at
`entrypoint:sase_xprompts/skills`.

Carry the singular reference through the native catalog, structured/mobile wire
documentation, Python binding documentation, editor completion, diagnostics, hover,
definition lookup, and the xprompt LSP. The catalog's package loader should receive the
actual builtin directory from the host rather than deriving it from the reference
namespace. When a misplaced skill is found in the package xprompt root, its migration
destination must be the configured package skill directory `src/sase/xprompts/skills/`;
project/home/plugin diagnostics must still point to their plural canonical `skills/`
locations.

Update Rust fixtures to cover all of these distinctions:

- `skill_reference_name` and its inverse return singular global and project-qualified
  references, while the physical layout records still use plural directories.
- Native catalogs load bundled sources from an `xprompts/skills` fixture and local
  sources from `sase/skills`, applying the existing first-wins precedence.
- Hash completion, hover, navigation, diagnostics, structured catalog output, and the
  gateway contract use `#skill/foo`; slash completion remains `/foo`.
- `#skills/foo`, bare `#foo`, malformed singular references, and nested invalid forms do
  not resolve as skills.

Regenerate checked-in API contract artifacts through the repository's supported
generator rather than editing generated JSON by hand. Run focused tests while iterating,
then `cargo fmt --check`, strict workspace Clippy, and the relevant core,
Python-binding, gateway, and LSP test suites. Finish with the repository's full
workspace test gate before handing the released binding contract to the next phase.

## Phase 2: Python builtin source layout and loading

After the compatible `sase-core` binding is available, update the Python dependency
constraint and adapters for its singular reference contract. Rename only the bundled
skill Markdown files and `SKILL.frame.template.md` from `src/sase/skills/` to
`src/sase/xprompts/skills/`; retain `src/sase/skills/__init__.py`, CLI helpers,
inventory, and use-log implementation in their current Python package.

Make `get_sase_package_skills_dir()` resolve the nested package resource with
`importlib.resources`, and make every builtin consumer use that one helper. This
includes package skill loading, authoring destinations, catalog source roots,
`sase skill list`, rendering, source-integrity pathspec checks, provenance manifest
resolution, and frame-template loading. Keep the normal canonical placement rules
unchanged for project, home, project-home, and plugin definitions. Add the builtin-only
migration-destination special case so a bundled `skill: true` file misplaced directly
under `src/sase/xprompts/` is directed to `src/sase/xprompts/skills/`, not to the Python
module directory.

Update package and loader tests to prove that wheels contain
`sase/xprompts/skills/SKILL.frame.template.md` and representative builtin sources,
ordinary package xprompt scanning does not double-load the nested skill files, and the
builtin destination is offered without making `src/sase/skills/` a canonical source
directory. Adjust source-integrity and manifest fixtures to expect the nested path and
retain their clean-tree, canonical-branch, and monotonic-provenance protections.

Exercise `sase skill list` and `sase skill init --dry-run`/`--diff` against the moved
sources, including the installed-wheel path. Do not run `sase skill init --force` or
apply generated chezmoi changes from this feature tree: the generated-skills workflow
requires the source commit to be landed first.

## Phase 3: User-facing cutover and end-to-end verification

Replace stale plural references in Python comments and display contracts, xprompt list
and show fixtures, ACE completion/highlight/jump/preview paths, editor-helper fixtures,
catalog HTML, and visual test setup. Preserve the dual-name model on every surface:
`name`/`reference`/hash insertion is `skill/<name>`, `skill_name` is the bare provider
name, and slash insertion is `/<name>`. Add explicit regression assertions that global
and project-qualified singular references select the same backing source as slash
invocation and that `#skills/` produces no compatibility candidate.

Update `docs/xprompt.md`, `docs/editor.md`, `docs/content_layout.md`, `docs/init.md`,
`docs/development.md`, `docs/architecture.md`, and any generated/reference prose that
still describes `#skills/` or `src/sase/skills/` as the bundled Markdown source path.
Documentation must distinguish the builtin exception from normal plural authoring
directories and from the unchanged provider installation targets. The existing
`generated_skills` memory already names `src/sase/xprompts/skills/`; do not edit memory
notes or their generated instruction files as part of this implementation.

Run a final stale-string audit that permits plural `skills/` only for physical source or
provider target paths and rejects `#skills/`, `<project>/skills/<name>` reference
examples, and claims that bundled Markdown lives directly in `src/sase/skills/`. Inspect
and intentionally regenerate only the ACE PNG goldens whose visible insertion text
changes, then rerun `just test-visual`. In the primary repository, run `just install`
before checks, followed by `just docs-check`, `just build-check`, and `just check-full`;
use the local compatible core checkout/binding for integration and repeat against the
released dependency before landing. The final smoke test should expand a builtin
`#skill/sase_plan`, expand a project/home fixture through the singular namespace, verify
`/sase_plan` completion and definition lookup, and confirm that `#skills/sase_plan` is
rejected.

## Landing order and acceptance criteria

Land the breaking `sase-core` contract first and allow release-plz to publish the
compatible binding. Then land the primary-repository package move, dependency update,
surface cutover, documentation, and reviewed visual goldens. The feature is complete
when source discovery still reads plural external `skills/` directories, builtin
Markdown is shipped only from `src/sase/xprompts/skills/`, every inline skill reference
uses singular `#skill/`, slash-provider behavior is unchanged, old plural references do
not resolve, deployment safety guards still block dirty/unlanded sources, and all core
and primary-repository gates pass.
