---
status: done
tier: epic
title: Canonical producer fleet migration
goal:
  Every active configuration, prompt, editor integration, automation, and plugin
  producer emits canonical SASE forms, and the landed sources are deployed and verified
  on athena, mac, and apollo without removing compatibility readers needed by later
  cutover phases.
parent_bead: sase-x7.3
phases:
  - id: host-producers
    title: Canonicalize authoritative SASE producers
    depends_on: []
    size: medium
    description:
      "host-producers: land canonical host workflows, automation, skill and memory
      sources, completion discovery filtering, and portable ownership stamps."
  - id: editor-producers
    title: Canonicalize the Neovim integration
    depends_on:
      - host-producers
    size: medium
    description:
      "editor-producers: migrate Neovim filetype, schema, catalog, reference, and
      fallback-completion surfaces to canonical names."
  - id: plugin-producers
    title: Canonicalize plugin prompts and callers
    depends_on:
      - host-producers
    size: medium
    description:
      "plugin-producers: remove available plugin prompt and import facades while
      preserving later bridge-owned wire and persisted-data readers."
  - id: chezmoi-authority
    title: Regenerate canonical chezmoi sources
    depends_on:
      - editor-producers
      - plugin-producers
    size: medium
    description:
      "chezmoi-authority: update configuration and memory sources, then regenerate
      skills and completions from clean landed revisions."
  - id: fleet-deploy
    title: Deploy and verify the canonical fleet
    depends_on:
      - chezmoi-authority
    size: medium
    description:
      "fleet-deploy: apply exact landed revisions on all three hosts, reconcile Mac
      drift, exercise integrations, and certify the producer census."
proposed_by: bbugyi200.athena.sase-x7.3
bead_id: sase-x7.3.1
create_time: 2026-09-09 19:52:17
---

- **PROMPT:**
  [prompts/202609/canonical_producers.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/canonical_producers.md)
- **PARENT:**
  [202609/canonical_only_fleet_cutover.md](https://github.com/sase-org/sase--plans/blob/main/202609/canonical_only_fleet_cutover.md)
- **BEAD:**
  [sase-x7.3.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-x7/sase-x7.3.1.md)

# Canonical producer fleet migration

## Context and boundaries

The parent design is `plan:202609/canonical_only_fleet_cutover.md`. The authoritative
census and bridge ledger are the reports attached to `sase-x7.1`; the migration kit
landed under `sase-x7.2`. Re-read those artifacts through `sase artifact read` before
implementation and revalidate the fleet baseline at each phase boundary rather than
assuming the planning-time observations are still current.

Planning-time facts that determine this plan:

- athena, mac, and apollo run the same SASE/core build, and all three report only the
  retired `medium_worker`, `small_worker`, and `xsmall_worker` model-alias warning;
- Apollo's config overlay is already reconciled and must not be rewritten without new
  evidence;
- generated provider skills and shell completions are stale on mac and apollo, while the
  Mac completion stamps incorrectly name `/home/bryan` targets;
- the Mac-only `~/sase/memory/sase_beads.md` contains stale guidance and must be
  reconciled without discarding unique prose;
- plugin hook signatures, pending-action fields, persisted legacy data, old roots, and
  explicit migration readers still depend on the bridge and cutover work in later
  `sase-x7` phases.

This epic therefore updates active emitters, generated discovery surfaces, and their
source-owned documentation, but it does not delete runtime aliases or compatibility
readers before the later removal certificate. Raw archived chats, immutable historical
plans, changelogs, migration fixtures, and quoted examples remain evidence rather than
rewrite targets. A textual match is not sufficient proof that something is a SASE
producer; in particular, classify unrelated `.gp` paths such as GAI/Google tooling and
leave them unchanged.

Use `sase repo open` before reading or changing every linked, sidecar, or external
repository. Never edit generated provider skill files, shell completion scripts, managed
instruction shims, or generated memory output directly. Use the owning source and
generator. Preserve every user's concurrent worktree change. Phase workers do not create
task beads: record any genuinely separate discovery on `sase-x7.3` as
`PROPOSED FOLLOW-UP: ...`, and do not close `sase-x7.3`, `sase-x7`, or another ancestor.

## Phase 1: host-producers

Update the authoritative SASE repository surfaces and land them before anything tries to
regenerate chezmoi outputs.

1. Reconcile the census against active xprompts, scripts, parser registrations,
   completion generation, skill sources, and memory authoring guidance. Maintain an
   explicit changed/deferred/no-op list so compatibility readers and historical text are
   not accidentally swept into this producer phase.
2. Change the active `commit` and `pr` workflows to consume and emit canonical patch
   result fields (`patch_name` and `meta_patch`) rather than ChangeSpec names. Update
   their focused tests and render both workflows to prove the canonical data path is
   live.
3. Change `src/sase/scripts/sase_bug` and any other confirmed active automation emitter
   to invoke canonical `sase patch` commands. Do not rewrite unrelated `.gp` or `task`
   text merely because it matches a search term.
4. Retire the generator source for the compatibility-only `sase_changespecs` skill and
   update `sase_patches` plus directly affected generated-skill guidance so newly
   generated skills teach only canonical commands. Preview the eventual provider-file
   prune, but do not deploy generated copies from an unlanded source revision.
5. Invoke `/sase_memory_write` before changing memory sources. Canonicalize current
   authoring guidance from `short|long` to `core|reference`, including the affected
   xprompt/generated-skill/bead guidance and glossary prose, while retaining deliberate
   reader and migration documentation. Run `sase memory init` through its source-owned
   workflow so managed instructions remain consistent.
6. Give completion generation a source-local way to omit retired command aliases, option
   aliases, and other compatibility-only discovery entries without deleting their
   runtime parsers. Mark the known bridge-ledger entries at their registrations and test
   that canonical entries remain discoverable while the old spellings no longer appear
   in generated bash, fish, or zsh specs.
7. Make chezmoi-owned completion stamps home-independent (for example, canonical `~/...`
   targets) while keeping actual write paths resolved and validated. Cover ownership and
   drift detection on differing home directories so Mac cannot inherit a Linux absolute
   path.

Update `src/sase/default_config.yml` whenever a user-reaching configuration default is
renamed. Run focused tests while iterating, then follow the repository's required
verification memory: `just install` when the ephemeral clone needs it and `just check`
before handoff. The phase note records the landed revision and the exact skill and
completion previews that `chezmoi-authority` must regenerate from it.

Acceptance: active commit/pr previews and automation contain canonical patch names;
completion fixtures expose canonical commands but no ledgered compatibility aliases;
portable stamp tests cover a non-Linux home; canonical memory and skill sources render
cleanly; no runtime reader needed by later phases was removed; `just check` passes.

## Phase 2: editor-producers

Open `sase-nvim` through `sase repo open` and migrate its active integration contract to
canonical naming while preserving the useful picker fallback capability.

1. Associate current project specs only with `.sase`, remove `.gp` file-detection and
   obsolete `.xprompts`/legacy schema globs, and replace legacy-only filetype
   identifiers with a neutral canonical ProjectSpec name across source, syntax
   registration, tests, and documentation. Do not broaden the match to arbitrary YAML.
2. Rename the user-facing completion backend/value and implementation identifiers from
   `legacy` to a capability name such as `picker`. Preserve `auto` behavior: use the LSP
   when available and the picker implementation when it is not. This is a terminology
   and producer cleanup, not permission to delete the fallback.
3. Update canned catalogs and reference examples to canonical `patch` kinds and current
   reference syntax. Remove old discovery examples from the README while retaining only
   clearly labeled compatibility-reader facts that users still need before the later
   cutover.
4. Confirm the chezmoi Neovim setup has no caller of a removed public name; if a source
   config change is necessary, document it for `chezmoi-authority` rather than editing
   generated or deployed config in this phase.

Run every headless Lua test in `tests/` with the documented
`nvim --headless -u NONE -c "set rtp+=." -l ...` harness, plus focused smoke tests for
the renamed filetype, picker fallback, project catalog, and reference completion. Record
the landed `sase-nvim` revision for fleet deployment.

Acceptance: `.sase` buffers receive the canonical filetype and integrations; `.gp` and
legacy xprompt roots are no longer advertised or auto-associated; `auto` still falls
back successfully when LSP completion is unavailable; all headless tests pass.

## Phase 3: plugin-producers

Open `sase-github`, `sase-telegram`, and `sase-research-artifacts` separately and use
the bridge ledger to distinguish current caller cleanup from later wire/data migration.

1. In `sase-github`, remove fallback imports through `sase.ace.changespec` where the
   canonical API already exists and remove redundant legacy workflow metadata such as
   `wraps_all` from the `gh` xprompt source. Test the canonical workspace and prompt
   path directly.
2. In `sase-telegram`, remove fallback imports or command construction only where the
   current landed host exposes a canonical equivalent. Do not rename frozen host hook
   arguments, pending-action keys, delivery cursor/bundle fields, or persisted wire data
   ahead of their owning bridge phases. Add focused tests that exercise the canonical
   import path instead of silently accepting the facade.
3. Audit `sase-research-artifacts` end to end. If it has no active legacy producer,
   record a verified no-op rather than manufacturing a change. If a real producer is
   found, update its authoritative source and generated-content test, not a built wheel
   or deployed copy.
4. Search the three resulting trees for ledgered legacy forms and classify every
   remaining match as compatibility reader, persisted schema, fixture/history, or
   later-phase owner. Treat an unexplained active emitter as a phase failure.

Run each changed repository's `just check`; if research artifacts change, also run its
wheel test lane. Land each repository through the host finalizer and record its exact
revision and install method for `fleet-deploy`.

Acceptance: plugin prompts and available canonical call paths no longer emit or import
legacy forms; later bridge-owned fields remain readable; every surviving search hit has
an explicit owner; all applicable repository checks pass.

## Phase 4: chezmoi-authority

Open the landed SASE and chezmoi repositories and regenerate only from clean,
published/landed source revisions. The generator integrity and provenance checks are
release barriers, not warnings to bypass.

1. In the chezmoi config source, remove `medium_worker`, `small_worker`, and
   `xsmall_worker`. Preserve the already intentional canonical `medium` and `small`
   precedence exactly, and add canonical `xsmall` with the former xsmall customization
   because no canonical override currently replaces it. Recheck every overlay and keep
   Apollo's already reconciled overlay unchanged unless fresh evidence proves drift.
2. Update any source-owned home/project prompt, snippet, query profile, script,
   scheduled command, or Neovim config that still actively produces an old form. Keep
   the `sase.yml` snippet and `_snip_utils.lua` counterparts synchronized as required by
   the chezmoi repository instructions. Classify non-SASE lookalikes instead of
   rewriting them.
3. Invoke `/sase_memory_write` for memory changes. Change the chezmoi-owned project
   memory source from legacy `short` to canonical `core`, preserve existing canonical
   `reference` notes, and regenerate managed instructions with
   `sase memory init --no-commit`. Do not hand-edit its generated README or instruction
   output.
4. From the clean landed host revision, preview `sase skill init --diff` and
   `sase skill init --dry-run`; verify the ownership manifest proposes removal only of
   the retired generated `sase_changespecs` copies. Then write the reviewed provider
   skill set into the chezmoi source with the non-committing generator path, without
   `--allow-dirty` or a provenance override.
5. Preview and then run `sase completion deploy-chezmoi --no-commit` from that same
   landed host source. Verify the three generated shells omit compatibility-only
   discovery entries and every stamp stores a portable home-relative target.
6. Review the full chezmoi diff, including generated deletions and mode changes. Run
   `just check` in chezmoi and generator `--check` modes against the pending source.
   Land the source revision, but do not apply it to a machine until the finalizer has
   made that revision available.

Acceptance: the source of truth contains only canonical model aliases and authoring
forms; generated skills and completions have trustworthy provenance and expected
pruning; no hand-edited generated file exists; `just check` and generator checks pass;
the phase note records the landed chezmoi revision.

## Phase 5: fleet-deploy

Deploy only the exact revisions recorded by the preceding phases. Re-probe all hosts
before each write and refuse to mix in a dirty checkout, an unlanded source, or a newer
concurrent deployment. Mac is required evidence: if it is temporarily unreachable, wait
through `/sase_monitor`; do not substitute Linux-only results and close anyway.

1. Review each host's pre-apply `chezmoi diff`, then run the repository-required
   `chezmoi update -a --force` on athena, mac, and apollo from the landed chezmoi
   revision. Confirm applied file hashes and ownership stamps match the source tree.
2. Install/update the landed SASE, `sase-nvim`, and changed plugin revisions through
   their supported deployment paths on each applicable host. Record package versions,
   source revisions, editable-versus-wheel mode, entry points, and plugin load paths; do
   not treat a matching version string alone as proof.
3. Reconcile the Mac-only `~/sase/memory/sase_beads.md` through the audited memory
   workflow. Compare it with the canonical source, preserve any genuinely unique prose
   in an appropriate canonical note, and remove it from active discovery only after
   proving it is stale/duplicated and checking VCS ownership. Run
   `sase memory init --check` on all three hosts and verify rendered instructions
   contain no legacy `short|long` authoring guidance.
4. Run `sase skill init --check` and `sase completion list` on all three hosts. Compare
   generated skill/completion hashes with the landed chezmoi source. Mac must report
   native Mac targets, every provider must lack the retired `sase_changespecs` skill,
   and interactive/bash/fish/zsh discovery must omit the compatibility-only commands and
   options while those runtime readers still work when called explicitly.
5. Run config doctor and effective-layer inspection on each host. The retired
   model-alias warning must be gone, intentional canonical precedence must match the
   pre-change behavior, and unrelated provider-availability notices are recorded rather
   than misrepresented as migration regressions. Render the active commit, PR, GitHub,
   and other touched workflows and inspect their effective canonical fields.
6. Verify the installed Neovim integration at the recorded revision with headless
   `.sase` filetype, schema, LSP, and picker-fallback smoke tests. Restart or otherwise
   invalidate every long-lived editor instance that could retain the old plugin code,
   then prove newly started instances load the canonical revision.
7. Run a final active-producer census across the host, chezmoi, Neovim, and plugin
   sources plus deployed outputs. Every remaining legacy hit must be a named
   compatibility reader, migration fixture, persisted-data field, or archive owned by a
   later `sase-x7` phase. Publish or attach a concise fleet verification receipt with
   commands, revisions, hashes, and per-host outcomes. Run `just check-full` for the
   combined host change through `/sase_monitor` and retain its passing result.

Acceptance: all three hosts use the landed source revisions; doctor is free of retired
alias warnings; effective config behavior is preserved; provider skills, completions,
prompts, plugins, and new editor processes expose only canonical authoring forms; the
Mac path and memory drift are resolved; the final producer census has no unexplained
emitter; full verification passes.

## Refusals and handoff

Stop and report rather than forcing deployment if a generator rejects provenance, a host
has moved past the recorded source, a worktree contains concurrent changes, a Mac-only
memory note contains unresolved unique content, or a required host cannot be verified.
Do not weaken guards, overwrite another agent's deployment, or expand this phase into
reader removal or persisted-data conversion.

The child epic's land agent reviews all phase notes and the fleet receipt. It then runs
`sase bead epic-symbols sase-x7.3`, resolves every result or rekeys it to its actual
later owner, and closes only `sase-x7.3` with a verification-specific note. It does not
close `sase-x7` or any other ancestor. If unresolved work remains, it adds a
`PROPOSED FOLLOW-UP:` note to `sase-x7.3` and leaves the phase open rather than creating
a bead itself.
