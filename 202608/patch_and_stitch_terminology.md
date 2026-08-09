---
tier: epic
title: Rename ChangeSpec to Patch and introduce stitches
goal: Make Patch and stitch the canonical SASE vocabulary everywhere without changing
  workflow semantics or breaking existing data, commands, configuration, integrations,
  or machine consumers
phases:
- id: rust-core-contract
  title: Establish Patch and stitch terminology in the Rust core
  depends_on: []
  description: 'rust-core-contract: add canonical Rust domain names and parser/wire
    compatibility while preserving serialized and binding contracts.'
  size: medium
- id: python-domain-storage
  title: Migrate the Python domain and ProjectSpec storage layer
  depends_on:
  - rust-core-contract
  description: 'python-domain-storage: introduce canonical models, modules, parsing,
    formatting, persistence, and wire adapters with legacy aliases.'
  size: large
- id: workflows-cli-contracts
  title: Rename workflow, automation, CLI, and metadata contracts
  depends_on:
  - python-domain-storage
  description: 'workflows-cli-contracts: migrate lifecycle, automation, command, and
    machine-facing call sites while keeping old entry points and data compatible.'
  size: large
- id: tui-config-surface
  title: Rename the ACE TUI and configuration surface
  depends_on:
  - workflows-cli-contracts
  description: 'tui-config-surface: present Patches and stitches throughout ACE and
    normalize legacy keymap, config, completion, and saved-state identifiers.'
  size: large
- id: linked-integrations
  title: Update linked GitHub, Telegram, and Neovim integrations
  depends_on:
  - workflows-cli-contracts
  description: 'linked-integrations: adopt canonical APIs and labels in GitHub, Telegram,
    and Neovim while retaining mixed-version compatibility.'
  size: medium
- id: docs-memory-skills
  title: Update documentation, glossary, demos, and generated-skill sources
  depends_on:
  - tui-config-surface
  - linked-integrations
  description: 'docs-memory-skills: rewrite maintained explanatory surfaces, regenerate
    memory shims, and prepare generated skills from authoritative sources.'
  size: medium
- id: compatibility-audit
  title: Reconcile compatibility and verify the complete rename
  depends_on:
  - docs-memory-skills
  description: 'compatibility-audit: classify every remaining legacy token and run
    exhaustive cross-repository compatibility and regression verification.'
  size: large
proposed_by: bbugyi200.athena.vu
create_time: 2026-08-08 13:05:29
status: done
bead_id: sase-hn
---

- **PROMPT:** [prompts/202608/patch_and_stitch_terminology.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/patch_and_stitch_terminology.md)
- **BEAD:** [sase-hn](https://github.com/sase-org/sase--beads/blob/main/pages/sase-hn/README.md)

# Patch and Stitch Terminology Migration

## Outcome and semantic contract

Make these the canonical definitions in code, UI, documentation, and agent context:

- A **Patch** is SASE's local unit of change. Every PR created or managed by SASE is
  associated with exactly one Patch, but a Patch may exist without a PR, represented by
  an absent `PR:` field / `pr_url`. Automatically importing externally created PRs by
  creating local Patches is intentionally out of scope.
- A **stitch** is the lightweight, ordered change record currently represented by a
  `CommitEntry` in a Patch's `COMMITS:` section. Every VCS commit made through the
  tracked workflow has an associated numeric stitch. A stitch need not have a commit:
  proposals retain their numeric-plus-letter IDs such as `(2a)` and all existing
  proposal behavior.
- “Commit” remains correct for actual Git/Mercurial commits, SHAs, VCS logs, commit
  statistics, the `sase commit` command, and the act of committing. Rename only the
  commit-like Patch entry and references to its ID/state to “stitch.”
- Lifecycle states, dependency rules, suffix semantics, hook/mentor/comment behavior,
  proposal acceptance/rejection/rewind behavior, archive placement, branch mapping, and
  PR submission behavior must not change.

This is an additive compatibility migration, not a blind search-and-replace. New code
and user-facing output use Patch/stitch. Old spellings may remain only in explicit,
documented compatibility adapters, migration fixtures, immutable historical records, or
stable public paths retained to prevent broken links. Each such occurrence must be
covered by the final allowlist audit.

## Compatibility policy

| Surface                          | Canonical contract                                                           | Compatibility requirement                                                                                                                                                                                                                                                                                                                                             |
| -------------------------------- | ---------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ProjectSpec text                 | `STITCHES:` containing numeric or proposal stitch IDs                        | Read both `STITCHES:` and legacy `COMMITS:`. Preserve the existing header during unrelated in-place edits; new Patches and newly created sections use `STITCHES:`. Continue accepting legacy `## ChangeSpec` headers if present.                                                                                                                                      |
| Python domain                    | `Patch`, `Stitch`, `patch.stitches`, stitch-ID helpers, and `sase.ace.patch` | Keep `ChangeSpec`, `CommitEntry`, `.commits`, `sase.ace.changespec`, and old function imports as thin aliases/adapters with one source of truth and regression tests.                                                                                                                                                                                                 |
| Rust/core wire                   | Canonical Patch/Stitch Rust symbols and Python-facing records                | Tolerantly deserialize legacy type/field spellings and supported schema versions. Preserve old exported bindings or provide wrappers; use a schema bump and dual/aliased fields only where needed so old payloads and installed consumers do not fail. Do not hand-edit release-plz-owned versions.                                                                   |
| CLI and skills                   | `sase patch ...`, `--patch`, and `/sase_patches`                             | Keep `sase changespec`, `--changespec`, and `/sase_changespecs` as tested aliases/shims. Canonical help and examples advertise the new forms. Alias use must perform the same operation and preserve exit codes and machine output semantics.                                                                                                                         |
| Configuration and saved UI state | Patch/stitch keys and labels                                                 | Normalize legacy `changespec*` action names, provider kinds, tab IDs, default-query keys, and saved values. Existing user configuration and state must load without warnings that corrupt output or require manual edits.                                                                                                                                             |
| Persisted/machine metadata       | Patch/stitch names in the next canonical representation                      | Readers accept legacy `changespec_name`, `commit_changespec_name`, `meta_changespec`, ChangeSpec-tag payloads, bead columns/JSONL fields, notification fields, event names, and completion kinds. Where old consumers still read emitted data, dual-write or keep a stable serialized alias while exposing canonical API properties; never silently drop attribution. |
| Docs and URLs                    | Patch/stitch prose, headings, navigation, and examples                       | Preserve public URLs such as an established guide filename or blog slug with a redirect/compatibility page or stable path; do not break inbound links merely to remove a token from a filename. Historical changelog entries and archived plans are data, not current vocabulary.                                                                                     |

## Phase 1: Rust core contract (`rust-core-contract`)

Use `/sase_repo` before working in `sase-core`.

1. Inventory `ChangeSpecWire`, `CommitWire`, parser state/sections, query/status/search
   types, agent scan/stats, bead records and schemas, editor completion kinds, gateway
   contracts, PyO3 functions, fixtures, examples, README, and crate changelogs. Classify
   each “commit” occurrence as a real VCS commit or the lightweight stitch abstraction.
2. Introduce canonical `PatchWire` / `StitchWire` and patch/stitch helper names through
   the reusable Rust core. Rename internal fields such as commit-entry IDs to stitch IDs
   where they identify ProjectSpec entries, including hook and mentor references.
3. Teach the parser and section code to accept `STITCHES:` and `COMMITS:` with identical
   behavior. New canonical formatting/fixtures use `STITCHES:`; legacy golden fixtures
   remain to prove old project files still parse byte-for-byte equivalently.
4. Preserve the published boundary with aliases, tolerant serde rules, and binding
   wrappers. If canonical payload keys require a wire-version bump, keep old-version
   deserialization and explicitly test old producer/new consumer and new producer/old
   compatibility paths rather than changing a key in place.
5. Update Rust comments, errors, examples, tests, and current docs. Keep genuine VCS
   commit APIs (`VcsCommitWire`, git-log parsing, SHA/statistics fields) named “commit.”
6. Run formatting, clippy, `cargo test --workspace`, and Python-wire/golden parity
   tests. Confirm no crate/package version or local dependency version was edited
   manually.

Acceptance: legacy `.sase` fixtures and legacy wire JSON still round-trip; canonical
Patch/Stitch records are available to Python; proposal IDs remain strings such as `2a`;
and no behavior beyond terminology/compatibility changes.

## Phase 2: Python domain and storage (`python-domain-storage`)

1. Create the canonical `sase.ace.patch` package and `Patch` / `Stitch` models. Move
   parsing, discovery, caching, locking, archive, validation, raw-text, section-order,
   refs, and persistence implementations under canonical names. Leave the former package
   as a deliberately small compatibility facade rather than maintaining two
   implementations.
2. Give `Patch` a canonical `stitches` collection and constructors/properties that
   preserve old `commits=` / `.commits` behavior. Rename `CommitEntryDict`, entry-ID
   parsers, hook status fields, and related methods to stitch vocabulary, retaining
   compatibility aliases at public import boundaries.
3. Update Python wire records/conversion/facades to consume the Rust contract from Phase
   1. Validate every supported old schema and field spelling, and keep deterministic
      JSON order/shape wherever it is already contractual.
4. Update all ProjectSpec writers and mutators: new sections use `STITCHES:`, readers
   accept both headers, and edits to legacy records neither lose entries nor cause
   unrelated whole-file churn. Cover mixed files, archives, two-blank-line boundaries,
   section ordering, drawers, multiline bodies, and atomic-write locking.
5. Rename canonical source/test modules, functions, variables, errors, docstrings, and
   fixtures. Use aliases only in designated compatibility modules/tests. Do not conflate
   VCS commit objects or commit hashes with stitches.

Acceptance: old and new ProjectSpec text parse to equivalent Patch objects; formatting
round-trips stitch drawers and proposal suffixes; legacy Python imports and attributes
still work; and canonical callers contain no ChangeSpec/CommitEntry vocabulary outside
the compatibility boundary.

## Phase 3: Workflows, CLI, and machine contracts (`workflows-cli-contracts`)

1. Migrate status transitions, archive/revert/restore/rebase/rename, hook and mentor
   scheduling, comments, deltas, queries, workspace providers, agent loading/cleanup,
   history, stats, beads, notifications, mobile helpers, xprompts, and commit workflows
   to Patch/stitch APIs and identifiers.
2. Ensure numeric stitches continue to be created for actual commits and letter-suffixed
   proposal stitches continue to work without commits. Update rewind/renumber, proposal
   acceptance/rejection, hook eligibility, mentor matching, timestamps, and display
   builders together so every cross-reference uses the same stitch ID.
3. Add `sase patch` as the documented command group with `current`, `search`, `ref`,
   `sync-deltas`, and migration behavior matching the old group. Retain
   `sase changespec` as an alias. Add canonical `--patch` options while accepting old
   option names and normalizing them to one destination.
4. Introduce canonical metadata/property names across agent markers, xprompt outputs,
   bead APIs, query/completion records, events, notifications, and JSON. Add migrations,
   aliases, or dual-read/write adapters for durable `changespec_*` and `meta_changespec`
   data; prove old beads, agent archives, result markers, saved queries, and mobile
   payloads remain readable and attributable.
5. Update generated CLI/config reference sources, shell completions, help, diagnostics,
   telemetry names, tests, and golden output. Legacy commands must retain exit status,
   selection, mutation, and machine-output behavior even when human labels now say
   Patch/stitch.
6. Add explicit invariant tests: a Patch without `PR:` is valid; every local PR-create
   path records its URL on the Patch; no external-PR discovery/import is added; a real
   commit creates a numeric stitch; and a proposal creates a commitless lettered stitch.

Acceptance: all lifecycle and workflow suites pass through both canonical and legacy
entry points, and persistent metadata written by existing releases still drives the same
behavior.

## Phase 4: ACE TUI and configuration (`tui-config-surface`)

1. Rename ChangeSpec-specific TUI models, widgets, actions, mixins, modal classes,
   grouping/index/load code, copy targets, help sections, command-palette entries,
   styles, IDs, fixtures, and tests to Patch. Rename commit-entry presentation within a
   Patch to stitch while leaving the separate VCS Commits artifact pane unchanged.
2. Update visible labels, empty/onboarding states, tooltips, errors, banners, tab text,
   statistics drilldowns, and accessibility/help prose. Ensure narrow/wide layouts,
   selection, folding, navigation, and focus order are unchanged.
3. Add canonical Patch action/config/provider/tab keys and normalize legacy
   `changespec*` values. Preserve saved grouping, selection, tab state, keymap
   overrides, default queries, mentor-provider configuration, and completion payloads
   from older installations.
4. Update `src/sase/default_config.yml` and the JSON schema together, plus config and
   keymap tests. Audit the special legacy artifacts-tab ID and similar persisted values;
   retain an adapter when changing the stored value would invalidate user state.
5. Update Textual and PNG expectations only after inspecting source/expected/diff
   artifacts. Run targeted TUI tests, keymap/config tests, performance checks where
   affected, and the visual snapshot suite.

Acceptance: ACE consistently shows Patches and stitches, actual repository commits are
still called commits, old saved state/config loads identically, and visual changes are
limited to the requested terminology.

## Phase 5: Linked integrations (`linked-integrations`)

Use `/sase_repo` before each linked repository.

1. In `sase-github`, migrate imports, workspace-provider names, PR-description context,
   xprompts, errors, tests, and docs to Patch. Continue importing/calling through a
   compatibility path when the plugin may run against an older supported SASE release.
   Verify recorded `PR:` submission behavior and Patch-without-PR handling.
2. In `sase-telegram`, update Patch-tag helpers, `/changes` presentation, agent/bead
   formatting, docs, and tests. Preserve the slash command and accept legacy helper
   payload/attribute names so existing clients and notifications still work.
3. In `sase-nvim`, present Patch completion rows and accept both `patch` and legacy
   `changespec` completion kinds during the compatibility window. Update README and
   headless LSP smoke fixtures without altering insertion text or completion behavior.
4. Run `just check` in the Python integrations and the relevant headless Neovim/LSP
   smoke tests against the locally updated SASE/core builds.

Acceptance: integrations display the canonical term, retain their existing user commands
and provider behavior, and interoperate with both sides of the compatibility window.

## Phase 6: Documentation, memory, demos, and skills (`docs-memory-skills`)

1. Rewrite maintained README, architecture, CLI/configuration/query/workflow/project
   guides, demos, tapes, current blog prose, image prompts/critiques, acknowledgements,
   code examples, and navigation to Patch/stitch. Keep historical changelog statements
   factual; update links and supply a stable compatibility page/redirect when a public
   filename or slug changes.
2. Replace the main guide with a Patch-focused guide that documents `STITCHES:` and its
   legacy `COMMITS:` reader compatibility. Explain precisely which stitches have commits
   and which do not. Update all examples and cross-links without expanding the scope to
   external PR synchronization.
3. Concisely update `sase/memory/glossary.md` with separate Patch and Stitch entries.
   The Patch entry must state PR -> Patch but not Patch -> PR; the Stitch entry must
   state commit -> stitch but not stitch -> commit and give `(2a)` as the proposal
   example. Remove the ChangeSpec entry rather than keeping duplicate definitions.
4. Run `sase memory init` immediately after the canonical memory edit and review the
   regenerated `AGENTS.md`, provider instruction shims, and memory README. Do not
   hand-edit generated instruction files.
5. Rename the authoritative skill source to `/sase_patches`, update related skill and
   xprompt sources, and keep `/sase_changespecs` as a small compatibility shim. Follow
   the generated-skills workflow: use `sase skill init --diff` while the source tree is
   unlanded and never edit generated chezmoi skill copies directly.
6. Update handwritten linked chezmoi configuration/scripts/snippets that use the old
   vocabulary through `/sase_repo`; keep their functional triggers compatible where
   changing them would break personal workflows.
7. Run strict docs/PDF checks, Markdown formatting checks, memory-init tests, skill
   source/rendering tests, and link validation.

Acceptance: all current explanatory surfaces teach one concise vocabulary, generated
memory files match the glossary, old public links still resolve, and generated skill
diffs contain the intended Patch/stitch wording with a compatibility skill.

## Phase 7: Compatibility audit and exhaustive verification (`compatibility-audit`)

1. Build a checked-in or test-owned terminology audit covering the main repository,
   `sase-core`, linked integrations, and handwritten chezmoi sources. Search case and
   separator variants (`ChangeSpec`, `changespec`, `change_spec`, `change-spec`, plural
   forms) and stitch candidates (`CommitEntry`, `commit_entry`, `COMMITS:`). Classify
   every remaining hit as:
   - required compatibility code/test/fixture;
   - immutable history or stable public path;
   - a genuine VCS commit concept; or
   - a defect to rename before completion.
2. Make the compatibility allowlist narrow and explanatory. It must not excuse current
   UI prose, canonical APIs, ordinary locals/docstrings, or maintained documentation.
   Add regression tests so old aliases cannot accidentally become canonical again or be
   removed without an explicit migration.
3. Exercise mixed-version/data matrices: old/new section headers; old/new Python names;
   canonical/legacy CLI and options; old/new wire payloads; legacy config and saved TUI
   state; bead/agent/notification metadata; completion kinds; and plugin consumers.
4. Re-run `just install` before verification. Because this touches the core boundary and
   the broadening set, run `just check-full`, `just rust-check`, `just test-visual`,
   strict docs/PDF checks, and every linked integration's full check/smoke suite.
   Inspect failures and visual diffs rather than accepting bulk changes blindly.
5. Verify clean generated state: `sase memory init` is idempotent and
   `sase skill init --diff` shows only expected not-yet-deployed provider copies. After
   the SASE source change is committed and landed on the canonical branch, perform the
   required generated-skill deployment from that clean tree with
   `sase skill init --force` and apply chezmoi; do not use dirty/provenance bypasses.
6. Confirm `git diff` in every touched repository contains no behavioral change beyond
   terminology, compatibility adapters/migrations, and tests. Record any intentionally
   retained legacy tokens in the handoff.

Acceptance: all verification lanes pass, both compatibility directions work, the
terminology audit has no unexplained hit, memory/skill generation is reproducible, and
the final handoff distinguishes intentional legacy aliases from canonical vocabulary.

## Explicit non-goals

- Discovering remote PRs and auto-creating local Patches for them.
- Changing Patch lifecycle states, dependency semantics, branch naming, proposal IDs,
  hook/mentor eligibility, or submission policy.
- Renaming real VCS commit concepts, the `sase commit` command, SHAs, or repository
  commit-history/statistics views.
- Removing compatibility aliases or rewriting immutable historical artifacts merely to
  reach a literal zero-match search result.
