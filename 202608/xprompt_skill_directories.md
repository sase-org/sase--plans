---
tier: epic
title: Canonical xprompt skill directories and namespaced invocation
goal: "Xprompt-backed agent skills are defined only in canonical sase/skills sources,
  remain invokable as agent skills with /<name>, and require a skills/ namespace
  whenever they are expanded as xprompts.

  "
phases:
  - id: core-contract
    title: Shared skill layout and editor contract
    depends_on: []
    size: medium
    description:
      "core-contract: define canonical skill sources, dual names, and editor behavior in
      the Rust backend."
  - id: sase-runtime
    title: Python discovery, validation, generation, and bundled migration
    depends_on:
      - core-contract
    size: medium
    description:
      "sase-runtime: consume the core contract, enforce canonical placement, preserve
      slash names, and migrate bundled sources."
  - id: user-surfaces
    title: Catalog, authoring, completion, and documentation updates
    depends_on:
      - sase-runtime
    size: medium
    description:
      "user-surfaces: expose the split xprompt and slash names consistently across every
      user-facing workflow."
  - id: source-migrations
    title: Enabled-project and chezmoi source migration
    depends_on:
      - sase-runtime
    size: small
    description:
      "source-migrations: re-audit enabled projects and move every personal skill source
      and reference to the canonical layout."
  - id: rollout-verification
    title: Cross-repository validation and canonical deployment
    depends_on:
      - user-surfaces
      - source-migrations
    size: small
    description:
      "rollout-verification: prove the hard cutover end to end and deploy generated
      skills only from landed canonical sources."
proposed_by: bbugyi200.athena.vh
create_time: 2026-08-07 22:51:10
status: wip
---

- **PROMPT:**
  [prompts/202608/xprompt_skill_directories.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/xprompt_skill_directories.md)

# Plan: Canonical xprompt skill directories and namespaced invocation

## Context and verified current behavior

The current classifier is the Markdown/config `skill` value, not the directory. Python
parses the value directly into `XPrompt.skill`, and `select_skill_xprompts()` selects
every truthy value. The native Rust catalog independently does the same truthiness
check. The special packaged `src/sase/xprompts/skills/` directory is only an extra
directory scanned by the built-in loader; its location does not itself establish the
skill contract. Config-defined and plugin-defined xprompts can therefore become skills
today without living in any skills directory.

There is also one name serving two incompatible purposes today: `XPrompt.name` is both
the `#name` reference and the generated `/<name>` skill name. The new layout must split
those concepts. A source declaring `name: foo` will have xprompt reference name
`skills/foo` but provider skill name `foo`. Thus `#skills/foo` expands the xprompt and
`/foo` invokes the installed agent skill. No `#foo` compatibility alias will be kept.
For sources that already receive an existing project namespace, the skill segment is
inside it: for example, project `app` exposes `#app/skills/foo`; global/home/package
sources expose `#skills/foo`.

The planning inventory found:

- 16 bundled sources plus `SKILL.frame.template.md` in `src/sase/xprompts/skills/` in
  the `sase` repository.
- One personal source, `bob_query.md`, in `home/sase/xprompts/` in the linked `chezmoi`
  repository.
- No skill-marked file or config definitions in the enabled `actstat` or `bob-cli`
  repositories, and no project-local skill-marked definition in `sase/xprompts/`.

Re-run that inventory during implementation so sources added after this plan was
authored are not missed. Access every non-current repository through `sase repo open`.

## Contract and compatibility decisions

The implementation should establish these invariants in one hard cutover:

1. Canonical user/project sources live at `<project>/sase/skills/` and `~/sase/skills/`;
   chezmoi represents the latter at `home/sase/skills/`. Packaged sources live directly
   in the existing Python package directory `src/sase/skills/`. Plugin resources use a
   sibling `skills/` resource directory rather than an `xprompts/` resource with skill
   frontmatter.
2. The `skill` field remains required because `true` versus a provider list still
   controls deployment targets. A file in a skills source without a valid truthy `skill`
   value is invalid. Conversely, a truthy `skill` field in an ordinary xprompt,
   workflow, config, or old `xprompts/skills/` source is invalid and is excluded with an
   actionable migration diagnostic.
3. The catalog carries both the canonical xprompt reference name and an explicit
   provider skill name. Inline expansion, argument assistance, references, snippets,
   statistics, and `#` completion use the former. Skill generation, `/` completion,
   slash diagnostics, hover/definition lookup, audit instructions, inventory, and
   provider target paths use the latter.
4. There is no silent fallback from `#foo` to `#skills/foo`, and legacy source paths do
   not remain read-compatible. Validation/doctor/init surfaces must identify the old
   source and the exact move required rather than silently dropping it.
5. Ordinary xprompt precedence and existing project namespacing remain intact. Skill
   sources have the analogous first-wins ordering within their own `skills/` namespace,
   so this change cannot make an ordinary `foo` shadow a skill xprompt `skills/foo` or
   vice versa.

## Phase 1: Shared skill layout and editor contract

Work in the linked `sase-core` repository, opened through `/sase_repo`.

- Extend the Rust-owned content-layout wire with canonical project, home, and chezmoi
  skill paths and an ordered skill-source contract. Do not derive `../skills` ad hoc in
  Python. Keep skills source-controlled and first-wins, but do not invent legacy paths
  for this hard cutover.
- Teach the native catalog fallback to scan canonical skill sources and packaged/plugin
  skill resources, require valid skill frontmatter there, reject skill frontmatter from
  ordinary xprompt/config sources, and construct names such as `skills/foo` (or
  `app/skills/foo`) while retaining `foo` as the skill name.
- Add a backward-compatible optional `skill_name` field to the mobile/editor catalog
  wire and the internal `XpromptAssistEntry`. Inline completion and xprompt diagnostics
  continue matching `name`; slash completion, slash diagnostics, hover, definition,
  completion filtering, and insertion match `skill_name` and emit `/foo`.
- Include `sase/skills/`, home skills, and package/plugin skill resources in native
  source classification, definition navigation, document eligibility, and catalog
  invalidation. Ordinary xprompt workflows and shared `steps/` remain under
  `sase/xprompts/` only.
- Add Rust tests for all layout scopes and precedence, the two-name wire contract,
  `#skills/foo` versus `/foo`, rejection of `#foo`, provider-list truthiness, invalid
  placement in both directions, project namespacing, editor diagnostics/navigation, and
  source invalidation. Run the repository's formatter, lints, and full test suite.

## Phase 2: Python discovery, validation, generation, and bundled migration

Work in the `sase` repository after the core contract is available.

- Update Python content-layout wire conversion and loader APIs to consume the Rust
  skill-source descriptors. Add an explicit `skill_name` (or equivalently named typed
  field) to `XPrompt` and preserve it through namespacing and
  xprompt-to-workflow/catalog conversion instead of deriving provider names from string
  splitting at call sites.
- Centralize skill-file parsing so all Python loaders enforce the same placement rules
  as the native fallback. Remove config-based skill support and the old packaged
  `xprompts/skills/` special scan. Record load issues with source, rule, and migration
  destination so `sase doctor`, `sase validate`, `sase xprompt show`, `sase skill list`,
  and `sase skill init` fail or warn consistently according to their existing contracts;
  generation must never proceed from an invalid source set.
- Make selection, rendering, inventory, target grouping, generated frontmatter,
  self-audit directives, compact source display, and provenance hashing use
  `skill_name`, while xprompt lookup uses the canonical reference name.
- Move all 16 bundled Markdown sources and `SKILL.frame.template.md` from
  `src/sase/xprompts/skills/` into `src/sase/skills/`. Render the frame through the
  `sase.skills` package, update package-data tests, and change the source-integrity
  guard from the old xprompt subtree to the canonical packaged skill subtree with
  accurate messages.
- Update tests for file loading, config rejection, skill selection, rendering,
  inventory, manifests, integrity guards, package resources, duplicate precedence, and
  provider-specific lists. Assert that generated provider paths and generated `SKILL.md`
  names remain unchanged despite the xprompt-reference rename.
- Run `just install` before verification. Because this touches the shared Rust binding,
  discovery broadening set, and packaging, run `just check-full` rather than only the
  scoped lane.

## Phase 3: Catalog, authoring, completion, and documentation updates

Complete every SASE-owned surface that consumes or creates xprompt skill definitions.

- Carry `skill_name` through the Python structured catalog, mobile helper bridge, ACE
  assist models, browser/list/show rows, editor helper payloads, syntax highlighting,
  statistics, and test fixtures. `#` completion must show/insert `#skills/foo`; slash
  completion must show/insert `/foo`. Argument hints for both forms must resolve the
  same source definition.
- Update ACE/local authoring and copy-on-edit flows so setting skill frontmatter cannot
  write into `sase/xprompts/`: existing skills edit in place under `sase/skills/`, and
  new project/home skill definitions choose the canonical skill destination. Ordinary
  prompt-save and workflow writers continue using `sase/xprompts/` and must reject a
  request that would smuggle in a skill definition.
- Update CLI help and public documentation, including the xprompt discovery table,
  canonical content layout, skill field, initialization/deployment instructions,
  architecture/development paths, ACE/editor behavior, and examples. Clearly document
  the hard cutover, project-qualified form, and the distinction between source,
  xprompt-reference, and provider-skill names.
- Do not edit `sase/memory/*.md` or generated agent instruction files without separate
  explicit user authorization. If authorization is later supplied, update the
  `generated_skills.md` and `xprompts.md` canonical notes and run `sase memory init` as
  the mandatory regeneration step; otherwise record that documentation follow-up through
  the required `/sase_new_task` workflow rather than changing memory implicitly.
- Add unit/integration tests for TUI completion, catalog JSON, mobile/editor bridges,
  show/list output, authoring destinations, copy-on-edit, highlighting, and definition
  navigation. Update intentional visual snapshots only after inspecting actual/expected
  diffs. Run `just install`, `just check-full`, and `just test-visual` if rendered ACE
  output changes.

## Phase 4: Enabled-project and chezmoi source migration

Use `/sase_project` to obtain the then-current enabled set and `/sase_repo` before
reading or changing every repository outside the implementing workspace.

- Search Markdown frontmatter and config entries, not just directory names, because the
  verified old classifier was `skill`. Move every discovered source to the scope's
  canonical `sase/skills/` directory, retain its declared provider targeting and slash
  name, and update every xprompt-form reference from `#foo` to its canonical
  `#skills/foo` or project-qualified equivalent. Do not rewrite `/foo` invocations.
- For the inventoried snapshot, move chezmoi's `home/sase/xprompts/bob_query.md` to
  `home/sase/skills/bob_query.md`. The `actstat` and `bob-cli` repositories require a
  documented zero-result re-audit rather than a speculative file change; the bundled
  `sase` sources were migrated in Phase 2.
- Search for stale `skill:` declarations outside canonical skill trees, legacy
  `xprompts/skills` paths, and old `#<skill-name>` references across all enabled project
  roots and the chezmoi source. Resolve every true positive while preserving unrelated
  files and user changes.
- Preview chezmoi effects and keep this phase source-only. Do not run
  `sase skill init --force`, use `--allow-dirty`, or publish generated provider files
  from an unlanded SASE implementation; canonical deployment belongs to Phase 5.

## Phase 5: Cross-repository validation and canonical deployment

- Re-run the enabled-project/source audit and prove the source invariant: every truthy
  skill declaration is in a canonical skill source, every canonical skill source has a
  valid truthy provider target, and no legacy packaged/user/project path remains.
- Exercise the public contract from package, home, and project scopes: catalog/show and
  completion resolve `#skills/sase_plan` and `#skills/bob_query` (plus qualified project
  forms where applicable); old `#sase_plan`/`#bob_query` references are unresolved with
  useful diagnostics; `/sase_plan` and `/bob_query` still resolve, navigate, and render
  to unchanged provider target names.
- Run the final full SASE and `sase-core` verification suites, `sase doctor`, and
  `sase validate`. Exercise both Python-helper and native-fallback editor catalogs so a
  missing helper cannot reintroduce the old naming behavior.
- Preview generated changes with `sase skill init --diff` or `--dry-run`. Only after the
  SASE source changes are committed and landed on the canonical branch, run
  `sase skill init --force` from that clean merged tree, then apply chezmoi. Never use
  `--allow-dirty`; verify the provenance manifest records the landed source revision and
  new source-set hash.
- Confirm generated `SKILL.md` content/paths are current for every registered provider,
  the live home targets match chezmoi, and a second init/apply/check pass is idempotent.

## Acceptance criteria

- A new or migrated `foo` source is accepted only from canonical `skills/` storage with
  valid skill frontmatter.
- `#skills/foo` expands it; `#foo` does not; `/foo` continues to invoke it.
- Python and native Rust catalogs, TUI, LSP/editor, mobile helper, CLI, generation,
  doctor, and validation agree on both names and placement diagnostics.
- All 16 bundled skills and `bob_query` are migrated; every enabled project has been
  re-audited; no stale source or xprompt-form reference remains.
- Full tests pass in both repositories, generated skills are deployed only from landed
  sources, chezmoi/live targets are current, and the final run is idempotent.
