---
tier: epic
title: Complete the AXE routine/job landing contracts
goal:
  General configuration, automation tribe identity, and public output satisfy the
  routine/job compatibility contract without changing stored identities or user data.
parent_bead: sase-11e
phases:
  - id: shared_config
    title: Share structural AXE normalization with general configuration
    size: medium
    depends_on: []
    description:
      "shared_config: complete Rust normalization, projection, provenance, and
      source-preserving edit support for general config consumers."
  - id: config_consumers
    title: Connect canonical config views and editors
    size: medium
    depends_on:
      - shared_config
    description:
      "config_consumers: wire runtime loading, config show and inventory, schema
      catalogs, and both editors to the shared contract in both rollout states."
  - id: tribe_identity
    title: Resolve automation tribe aliases and collisions in Rust
    size: medium
    depends_on:
      - config_consumers
    description:
      "tribe_identity: preserve stored chop identity while exposing job consistently and
      rejecting independent job tribe collisions through shared core behavior."
  - id: public_output
    title: Preserve data while completing routine and job presentation
    size: medium
    depends_on:
      - tribe_identity
    description:
      "public_output: replace whole-string substitutions with explicit projections and
      canonical diagnostic templates, retaining exact names, paths, payloads, and legacy
      contracts."
  - id: acceptance
    title: Prove the repaired upgrade contract and integration
    size: medium
    depends_on:
      - public_output
    description:
      "acceptance: complete isolated end-to-end, visual, generated-output, binding, and
      repository checks against the combined tree and intervening changes."
proposed_by: bbugyi200.athena.sase-11e.land
create_time: 2026-09-16 01:04:08
status: wip
---

- **PROMPT:**
  [prompts/202609/axe_routine_job_landing_repairs.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/axe_routine_job_landing_repairs.md)
- **PARENT:**
  [202609/axe_routines_jobs.md](https://github.com/sase-org/sase--plans/blob/main/202609/axe_routines_jobs.md)

# Complete the AXE routine/job landing contracts

## Scope and handoff

This is remaining work discovered while landing `sase-11e`, whose approved design is
`plan:202609/axe_routines_jobs.md`. Read that artifact and the parent bead's landing
audit before implementing. The parent has seven completed phase records, but the
implementation does not yet meet its compatibility contract. This child epic repairs
those gaps; it does not repeat the completed command, executable, SDK, documentation, or
operator-config migrations.

The `parent_bead: sase-11e` relationship hands completion back to the interrupted parent
landing. Parent closure, its post-close Symvision run, and setting its plan status to
done are not child implementation phases. Do not force-close anything. Keep AXE,
`sase axe`, the AXE tab, the existing `sase tui` interface, scheduling and admission
semantics, runtime paths, `.chop.` agent names, graph identities, and accepted legacy
inputs unchanged. Keep sunset flag `axe_routine_job_contract` and its independent
removal bead `sase-11f`; this rollout explicitly retains it.

All paths below are repository-relative. Use `/sase_repo` to open `sase-core`,
`sase-telegram`, `chezmoi`, and any other non-primary repository before access. Rust
owns shared backend behavior; Python may marshal, cache, perform file I/O, and render
Textual widgets. Read `tui_perf.md` and `src/sase/ace/AGENTS.md` before changing TUI
paths. No live AXE starts/stops, job execution, Telegram sends, operator-config
application, or deployment is needed for these tests.

## Audited baseline and concrete failures

The landing audit examined all seven child notes and epic-associated commits in SASE
(`e41f651eb9` through `a7029f02c8`), core (`a68ee7d`, `6be757c`), Telegram (`ab9d985`),
and chezmoi (`8b28cb16`, `fe65414d`). SASE HEAD and its local `origin/master` both named
`a7029f02c8`. The opened core additionally contains hold/queue work through `a7d5882`;
Telegram is released at `bedb649`.

1. `config/axe.rs` normalizes each AXE layer and returns both internal and public data,
   but its helper is private to that subsystem. General core `config/provenance.rs` and
   `config/plan.rs` still call generic `merge_layers`. Python `config/loading.py`,
   `config/inventory.py`, `config/_edit_plan.py`, and `main/config_handler.py` do not
   select the new public projection. An isolated legacy default
   `axe.lumberjacks.checks.interval: 5` plus canonical user
   `axe.routines.checks.interval: 19` produces runtime interval 19 through
   `compose_axe_config`, while general inventory reports two separate collections with
   intervals 5 and 19. Generic merge retains both alias trees. This violates
   effective-view parity and risks edits writing a second alias subtree.
2. `core/agent_tribe.py` implements `job -> chop` in Python without consulting
   configuration. Given one user layer containing distinct `ace.tribes.chop` and
   `ace.tribes.job` records, each with a description and a different icon, general
   inventory reports no collision. `tribe_display_for("chop")` silently selects the
   custom job icon. Shared collision handling promised by phase 4 is absent.
3. `axe/status_render.py::_public_status_value` recursively substitutes text in every
   string. Rendering a fixture with routine name `chop-watch`, configured job
   `chop-test`, and path `/tmp/lumberjacks/chop-watch.log` changes all three values to
   `job-watch`, `job-test`, and `/tmp/routines/job-watch.log`.
   `axe/chop_doctor.py::chop_check_to_public_dict` similarly changes an exact missing
   executable `/tmp/sase_chop_test` to `/tmp/sase_job_test` in its summary and details.
   `axe/cli.py::_public_axe_text` applies the same unsafe strategy to ambiguity errors
   containing user identities.
4. Original backend diagnostics remain visible with old vocabulary, including
   `config/axe.rs` generated-entry and duplicate-identity errors, `axe_chop/config.rs`
   validation messages, and `axe/status_collector.py` errors. Update the templates at
   their owner; a final rendered-text replacement cannot preserve embedded user data.
   Audit related script/context and TUI errors too.

The existing focused tests all pass despite these gaps: 107 tests covering AXE config
backend, status CLI, doctor, SDK, script runner, artifact-run retention, and git-object
sharing. Preserve those successes and add meaningful regression coverage for the
counterexamples above. In standalone probes, clear the inherited `SASE_FEATURE_FLAGS`
transport before using test overrides, and assert the resolved flag state; otherwise
inherited state can mask the Off branch. With isolated transport, the current
AXE-specific projection correctly switches both ways.

## Phase shared_config

In `sase-core/crates/sase_core/src/config/`, extract and reuse the established
structural normalization and exact source-path mapping from `axe.rs`. Integrate it into
the general config merge/inventory/edit backend. Expose the minimum versioned
wire/binding support needed by frontend adapters; do not implement a second mapping in
Python. Keep schema-v1 internal identities unless an actual wire shape change requires
explicit versioning.

Normalize each input layer before merge. Cover all AXE structural aliases from the
parent contract, including collections, script directories, timeouts, logs, and
diagnostic settings. Preserve list replacement/concatenation rules, keyed identity
merge, sparse overrides, disables/deletes, target expansion, dotted entity keys as exact
segments, and authored source provenance. Conflicting leaves within one layer must
identify both authored paths and the source layer; disjoint/equal synonyms may merge.
Arbitrary `vars`, env mappings, descriptions, paths, and routine/job names are opaque
data.

Provide an explicit public projection selector for general effective views, with
canonical On output and legacy Off output, while internal runtime consumers keep their
internal model and raw/source views keep their actual authored bytes. Avoid requiring
flag-dependent config resolution to resolve the flag itself.

General and AXE edit plans must target an existing contribution's real source subtree
for set/reset/delete and use canonical paths for new entries. Test cross-alias edits,
both source spellings, mixed list/map representations, exact dotted keys, and additions
with inherited-only fields. Do not lose comments or create a parallel alias subtree.
Ensure candidate validation sees the same effective semantics as runtime. Run full core
`just check`, including Python

> =3.12 PyO3 binding tests, using `/sase_monitor` when long.

## Phase config_consumers

Wire the shared API into SASE's config load, effective `sase config show`, general
inventory/catalog and completion, config validation/doctor, general edit previews, and
AXE entry editor. Start with `config/{loading,core,inventory,_edit_plan}.py`,
`main/config_handler.py`, `axe/{config,config_backend,_config_layers}.py`, and the
config pane and AXE editor modules under `ace/tui/`.

Use the Rust result for normalization and source-path resolution. Retain cached and
worker-based behavior; do not add I/O or subprocess work to the UI event loop. Audit all
actual consumers of the generic merged config before changing its shape. Effective UI
fields and labels should be canonical in normal operation; raw layer views must still
show actual old source names where they were authored. Resolve existing schema
definitions and compatibility aliases without suggesting two independent settings. Keep
`src/sase/default_config.yml` and the public schema consistent, preserving all values,
cadences, enablement, and keybinding chords.

Test the concrete default=5/user=19 counterexample through runtime, general show,
inventory, preview, and applied temporary-file edit. Exercise both flag states, legacy
and canonical inputs, adding a new entry, modifying/resetting an old one, and
conflicting aliases. Assert both exact target bytes and effective behavior. Refresh the
local binding through the repository's supported workflow before Python verification; do
not manually edit core release versions or claim a local build proves a released
dependency floor. Run focused config/TUI checks and `just check` after tracked changes.

## Phase tribe_identity

Move the shared automation-tribe resolution and collision contract into Rust, using the
appropriate config and agent-identity domain and binding entrypoints. Replace Python
alias logic with thin adapters. Built-in public `job` and historical `chop` refer to the
existing stored `chop` identity. Preserve historical assignment bytes and `.chop.` agent
names; do not globally rewrite the assignment store.

Distinguish ordinary customization of the built-in automation tribe from an
independently configured pre-existing `job` tribe. Detect ambiguous/conflicting authored
identities and report an actionable, source-aware diagnostic before any
assignment/filter/display can silently merge the two. Use layer provenance and explicit
resolution rules, not an assumption that every `job` entry is the built-in alias. Keep
user-selected unrelated tribe names unchanged.

Audit assignment input, CLI lists, query filtering, wait/fork references where
applicable, completion, config views, and TUI panels. All consumers must agree on stored
identity and canonical display while preserving glyph, color, collapse defaults, and
grouping. Starting files include `core/agent_tribe.py`, `ace/agent_tribes.py`,
`ace/agent_query/evaluator.py`, and
`ace/tui/models/{tribe_display,agent_panels,agent_live_query}.py`.

Add Rust/binding and frontend tests for historical assignment reads, canonical
assignment writes to the existing identity, legacy/canonical filtering, display
customization, and an independent custom `job` collision. Verify no metadata read
silently reassigns historical user data. Run required core and SASE checks.

## Phase public_output

Remove unrestricted rendered-string substitutions in status, doctor, and CLI ambiguity
handling. Rename only contract-owned envelope keys and authored labels; all names, IDs,
script values, paths, free text, arbitrary payload keys/values, stored diagnostics, and
historical output must survive verbatim. Public status and doctor projections must be
explicit about their fields and owned diagnostic templates. Place shared
projection/resolution/diagnostic behavior in Rust with thin Python adapters; Textual
layout remains in Python.

Update the live diagnostic templates at their source to routine/job wording and
canonical remediation commands, keeping raw authored config paths accurate. Audit
status/restart errors, AXE validation, generated-entry errors, script SDK and context
errors, logs/notifications, and visible editor fields. Do not rename private types,
modules, state paths, operation IDs, raw kind values, or immutable history just because
their spelling is old.

Preserve schema-v1 output for hidden `chop` CLI aliases. Canonical `job` JSON and
unchanged `axe status --json` use v2 On and v1 Off, with valid JSON-only stdout. Retain
the current artifact-ref canonicalization to stored `chop:` and expose `job:` only on
the intended public surfaces; round trips must not duplicate edges or rewrite an actual
agent name. Ensure public human errors are canonical without corrupting exact names or
paths even for legacy invocation spelling.

Tests must exercise names containing old words, absolute old-only executable paths,
descriptions mentioning both vocabularies, keys named `chops` inside user payloads,
duplicate job names across routines, and actual legacy state paths. Prove both public
envelopes and preserved data, rather than only checking that the new words occur. Run
focused CLI, diagnostics, links, and status suites and the required repository checks.

## Phase acceptance

Build one reproducible isolated integration matrix using the repaired production
entrypoints. Compare equivalent legacy and canonical config through routine/job list,
doctor, an exact source edit, a harmless fixture-script run, history and JSON
inspection, and job/agent link navigation. Cover both flag states, old-only plugin
entrypoints, equal/conflicting aliases, expanded targets, dotted names, duplicate job
names across routines, stable deduplication, timeouts, and the custom tribe collision.
Mock external launches/network effects and use temporary config/state directories.
Include env alias propagation/scrubbing and SDK parity in the existing regression
suites.

Recheck post-start drift before deciding acceptance. The original audit reviewed
intervening disk cleanup/wire-v3 changes (`95ac39fc7c`, `90e95fbd26`, `6c76f29d75`,
`6fca91cdc7`) and queue admission (`b6b11f2155`), plus xprompt, TUI cache, dev-update
timeout, and module-split work. Keep preview-only artifact retention, borrower
protection, and newer queue/hold semantics intact. These changes are not authorization
to restore unsafe deletion or rename internal `.chop.` admission identities. Repeat the
log/changed-file audit for drift since this plan was authored, including linked repos
and any PR base branch.

Telegram aliases and receiver argv are already implemented and its intervening commit is
release metadata. Chezmoi structural config and generated completions are already
migrated; preserve exact `bugyi_chop_*` custom executable values. Recheck `sase-github`,
`sase-nvim`, and `sase-research-artifacts` only for actual new integration matches. This
landing audit found none; newer Neovim highlighting and research-role changes were
unrelated.

Run affected AXE narrow/wide visual snapshots and inspect actual/expected/diff images
before accepting goldens. Verify current docs/examples, schema/defaults, shell
completions and syntax, glossary aliases, generated memory output, and currently
embedded diagrams agree with the final contracts. Do not rewrite historical docs or
redeploy generated skills from an unlanded tree. If a skill source actually needs
changing, follow `generated_skills.md`; if memory actually needs changing, invoke
`/sase_memory_write` first.

Run required Rust `just check` including binding tests, relevant mocked Telegram checks,
and SASE `just check` after tracked changes. Before combined landing run
`just check-full` only through `/sase_monitor` with TESTING/TESTED labels and a
mechanical continuation. Do not substitute passing focused tests for the full landing
gate. Resolve any remaining epic-caused issue before declaring acceptance.

## Follow-up disposition inherited by the parent land agent

- `sase-11e.4` note 2 (wire alignment): resolved in the current tree by the
  object-sharing v3 adoption/recovery commits and phase 5's artifact-retention v3
  update. Repository open works and the corresponding focused tests pass. Decline a new
  task for this now-resolved proposal.
- `sase-11e.4` note 3 (shared tribe collision handling): accepted as unfinished parent
  epic work, implemented by this child plan, not an independent task.
- `sase-11e.5` note 2 (missing busted): independently reproduced and recorded as small
  CI task `sase-11m`. Evidence is `file:explicit:604a46eb5672a96d747773f2`. Chezmoi
  `just check` passes lint and fails deterministically at `Justfile:test-nvim` with
  missing `busted`; this prerequisite predates and is not caused by the AXE rename.
  Recheck that task before reporting final integration verification; do not claim
  chezmoi's full check passed or silently skip its tests.
- Evidence attachment hit the existing hidden-plans-clone write-lane defect `sase-10y`;
  the artifact itself was created and its ref retained in prose. Record corroboration
  there without creating a duplicate or changing foreign hidden-clone files. Active
  `sase-112` handles provenance merge conflicts, not this missing-test-prerequisite or
  the routine/job defects.

`sase bead epic-symbols sase-11e` reported no entries during this audit. Neither the
parent epic nor its linked plan was marked complete. Its final land agent must carry
these dispositions into the eventual close note and recheck symbols at the actual close
boundary.
