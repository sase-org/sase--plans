---
tier: tale
title: Complete the Patch and stitch compatibility audit
goal:
  Make every maintained SASE surface Patch/stitch-first, retain only explicit and tested
  legacy compatibility boundaries, and prove the complete migration across the main,
  core, integration, and handwritten configuration repositories.
proposed_by: bbugyi200.athena.sase-hn.7
bead: sase-hn.7
create_time: 2026-08-08 22:42:31
status: done
---

- **PARENT:**
  [202608/patch_and_stitch_terminology.md](https://github.com/sase-org/sase--plans/blob/main/202608/patch_and_stitch_terminology.md)
- **BEAD:**
  [sase-hn.7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-hn/sase-hn.7.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-hn.7](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-hn.7.md)
- **COMMITS:**
  - [ba9bb17](https://github.com/sase-org/sase-nvim/commit/ba9bb178ef151294e5aa63ee1e2ee110fc348f7d)
    — test: update xprompt LSP smoke fixtures

# Complete the Patch and stitch compatibility audit

## Goal

Finish phase `sase-hn.7` by reconciling the Patch/stitch migration delivered by the
preceding epic phases. Build an executable terminology audit, use it to remove ordinary
legacy ChangeSpec/CommitEntry vocabulary from canonical code and maintained prose,
preserve the intentionally supported old data and entry points, exercise both sides of
the compatibility window, and run every exhaustive verification lane required by the
epic design.

This is a terminology and compatibility hardening pass. It must not change Patch
lifecycle behavior, dependency rules, branch mapping, proposal suffix semantics,
hook/mentor/comment behavior, archive placement, PR submission behavior, or the meaning
of real VCS commits.

## Current state and constraints

- The prior phases have landed canonical `Patch` / `Stitch` Rust and Python records,
  `sase.ace.patch`, `STITCHES:`, `sase patch`, Patch-oriented ACE/configuration, updated
  linked integrations, and Patch/stitch documentation and skill sources.
- Static inspection still finds legacy tokens across ordinary main-repository and Rust
  internals in addition to legitimate compatibility adapters, fixtures, serialized field
  names, generated copies, stable public paths, and immutable history. The final phase
  must distinguish and resolve those categories instead of accepting a blanket
  repository-wide exclusion.
- `ChangeSpec`, `changespec`, `change_spec`, `change-spec`, plural/case variants,
  `CommitEntry`, `commit_entry`, and `COMMITS:` are the audited legacy candidates.
  “Commit” remains canonical for Git/Mercurial commits, SHAs, VCS logs/statistics, the
  `sase commit` command, and the act of committing.
- Preserve and test `sase changespec`, `--changespec`, `/sase_changespecs`,
  `sase.ace.changespec`, `ChangeSpec`, `CommitEntry`, `.commits`, old wire/schema keys,
  legacy ProjectSpec headings/sections, old configuration/saved-state values, completion
  kinds, and durable agent/bead/notification metadata wherever the approved
  compatibility policy requires them.
- Do not edit release-plz-owned Rust versions. Do not deploy generated skills from an
  unlanded tree or edit generated chezmoi skill copies directly. Record any genuinely
  separate discovery as a `PROPOSED FOLLOW-UP:` note on `sase-hn.7`, never as a new
  bead, and close only the phase bead.

## Implementation

### 1. Add a narrow, executable terminology audit

- Add a checked-in audit implementation and contract tests in the main repository that
  scan tracked text in the main repo plus explicit paths supplied for `sase-core`,
  `sase-github`, `sase-telegram`, `sase-nvim`, and handwritten chezmoi sources.
- Report every candidate with repository, path, line, matched spelling, classification,
  and explanatory rule. Keep classification rules reviewable and narrow: explicit
  compatibility adapters or legacy-data tests/fixtures; immutable history or a stable
  public path; genuine VCS-commit concepts; otherwise a defect.
- Reject unclassified hits, stale allowlist entries, overbroad directory exemptions, and
  legacy vocabulary in current UI prose, canonical APIs, ordinary locals, docstrings, or
  maintained documentation. Mark the main-repository guard as a contract test and
  refresh the generated contract manifest so every scoped check exercises it.
- Cover the audit itself with positive and negative fixtures, including case and
  separator variants, compatibility comments, legacy serialized keys, stable paths, and
  a deliberate ordinary-source regression.

### 2. Reconcile canonical code and explicit compatibility boundaries

- Use the audit findings to migrate remaining ordinary main-repository callers,
  arguments, locals, type aliases, docstrings, errors, tests, and canonical filenames to
  Patch/stitch vocabulary. Route them through `sase.ace.patch`, canonical core facades,
  canonical CLI/parser handlers, Patch metadata properties, and stitch-ID helpers.
- Keep old Python packages/modules, command/parser registrations, option destinations,
  serialized fields, and helper names only in deliberately small shims or adapters.
  Where an old module currently contains mixed implementation and compatibility logic,
  move the implementation behind the canonical module and leave a one-way re-export or
  normalization layer with focused identity/behavior tests.
- Apply the same rule in `sase-core`: make
  parser/state/query/status/agent/editor/gateway internals Patch/Stitch-first, while
  retaining legacy Rust/PyO3 exports and serde/wire spellings as explicit wrappers or
  tolerant aliases. Preserve old schema versions and JSON shapes, proposal stitch
  strings such as `2a`, and byte-equivalent parsing of legacy fixtures.
- Reconcile linked integrations and handwritten chezmoi sources against the audit. Keep
  public slash commands, stable completion aliases, mixed-version plugin adapters, and
  generated skill copies where required; remove any ordinary legacy implementation
  vocabulary or current prose missed by earlier phases.

### 3. Exercise the complete compatibility matrix

- Add or consolidate regression coverage for `## Patch` / `## ChangeSpec` and
  `STITCHES:` / `COMMITS:` parsing, legacy-header preservation during unrelated edits,
  canonical new-section emission, numeric committed stitches, and commitless lettered
  proposal stitches.
- Prove canonical and legacy Python imports/classes/properties/helpers remain identical
  or equivalent as contracted, and prove canonical and legacy CLI commands/options have
  matching selection, output shapes, exit codes, and mutations.
- Exercise old/new Rust and Python wire payloads, config/keymap/saved-tab normalization,
  bead/agent/notification/result metadata, editor completion kinds, and GitHub,
  Telegram, and Neovim mixed-version consumers. Retain attribution, Patch-without-PR
  support, and local PR URL recording; do not introduce external PR discovery.
- Ensure the audit passes against every in-scope repository and produces a concise
  retained-token report suitable for the phase handoff.

### 4. Verify generated state and all repositories exhaustively

- Run `just install` in the main repository before verification, then run focused audit
  and compatibility suites while iterating.
- Run `just check-full`, `just rust-check`, `just test-visual`, `just docs-check`, and
  `just docs-pdf-check`. Inspect failures and visual artifacts; do not bulk-accept
  snapshot changes. Run `cargo fmt --all -- --check`, clippy with warnings denied, and
  `cargo test --workspace` in `sase-core` if the main wrapper does not already cover an
  equivalent lane.
- Run full `just check` suites in `sase-github` and `sase-telegram`, the relevant
  headless Neovim/LSP completion smoke tests in `sase-nvim`, and the appropriate
  handwritten-source checks in chezmoi. Use the locally updated main/core builds where
  integration setup supports them.
- Run `sase memory init --check` (or initialize and confirm a clean second pass if the
  command requires regeneration) and `sase skill init --diff`. Confirm memory generation
  is idempotent and the skill diff contains only expected provider copies pending the
  post-land deployment; do not use `--force` or provenance bypasses in this phase.
- Run `git diff --check`, review tracked diffs and status independently in every touched
  repository, and confirm there is no behavior change outside terminology, compatibility
  adapters/migrations, tests, and the audit.

## Completion

- Record any separate follow-up as `sase bead note sase-hn.7 'PROPOSED FOLLOW-UP: ...'`.
- Close `sase-hn.7` with
  `sase bead close sase-hn.7 --note "<exact checks and retained compatibility boundaries verified>"`
  only after all required lanes pass.
- Do not close the parent epic. The final handoff must identify the audit location,
  summarize intentionally retained legacy categories, list verification commands, and
  call out any proposed follow-up notes.

## Acceptance criteria

- The terminology audit covers all six in-scope repositories/surfaces, has no
  unexplained candidate, and cannot silently grow or retain stale exemptions.
- Canonical code, tests, UI text, errors, docstrings, and maintained documentation use
  Patch/stitch; remaining old names exist only at explicit, tested compatibility,
  fixture/history, stable-path, or genuine-VCS boundaries.
- Both compatibility directions work for ProjectSpec text, Python/Rust wire and public
  APIs, CLI/options, saved configuration/state, durable metadata, completion payloads,
  and linked integrations.
- All exhaustive verification and generation-idempotence checks pass, repository diffs
  contain only the intended migration hardening, and only `sase-hn.7` is closed.
