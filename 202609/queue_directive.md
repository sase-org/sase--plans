---
tier: epic
title: Separate agent queue controls into %queue and %q
goal:
  Move runners and priority from %wait to %queue, support positional runners and p=,
  preserve admission behavior, provide matching ACE and LSP completion, and migrate
  maintained prompt producers and documentation across linked repositories.
phases:
  - id: core
    title: Shared queue grammar and editor contract
    depends_on: []
    size: medium
    description:
      "core: implement shared Rust queue validation, formatting, launch parsing,
      bindings, and completion behind temporary epic scaffolding."
  - id: integration
    title: Python runtime and prompt editing integration
    depends_on:
      - core
    size: medium
    description:
      "integration: connect queue parsing, admission detection, AXE generation, durable
      edits, and ACE completion to the shared contract."
  - id: migration
    title: Repository migration and unconditional cutover
    depends_on:
      - core
      - integration
    size: medium
    description:
      "migration: convert all maintained consumers and documentation, reject retired
      wait keywords, and remove the temporary feature flag."
  - id: verification
    title: Cross-repository acceptance and landing preparation
    depends_on:
      - core
      - integration
      - migration
    size: medium
    description:
      "verification: exercise runtime and editor parity, complete repository gates,
      audit remaining old syntax, and prepare coordinated landing and generated-skill
      deployment."
proposed_by: bbugyi200.athena.09b
bead_id: sase-yj
create_time: 2026-09-09 19:52:40
status: wip
---

- **PROMPT:**
  [prompts/202609/queue_directive.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/queue_directive.md)
- **BEAD:**
  [sase-yj](https://github.com/sase-org/sase--beads/blob/main/pages/sase-yj/README.md)

# Plan: Separate agent queue controls into %queue and %q

## Outcome and scope

Queue admission controls belong to `%queue` / `%q`. `%wait` / `%w` continues to describe
dependencies and time floors. This is a syntax migration with the same scheduling
behavior. It includes handwritten prompts, generated prompts, editing and replay paths,
shared editor assistance, documentation, and personal automation.

An epic is appropriate because Rust owns shared launch/editor behavior, Python owns
integration with the existing runner and Textual, and independently maintained plugins
and personal configuration produce prompts. The four phases are ordered to keep those
interfaces reviewable. Each phase is bounded direct implementation work; authoring this
epic is xlarge planning work under the SASE size guidance.

Only the scratch plan is authored before approval. Implementation workers must open each
other repository with `sase repo open <handle> -r "<specific reason>"` and use its
returned path. Paths below are relative to the repository named with them; no checkout
location is assumed. Re-read applicable repository instructions.

## Final directive contract

| Input                                                | Meaning                                                          |
| ---------------------------------------------------- | ---------------------------------------------------------------- |
| `%q:5`, `%queue:5`, `%q(5)`, `%queue(runners=5)`     | Explicit admission threshold of five already-running agents      |
| `%q(p=20)`, `%q(priority=20)`, `%queue(priority=20)` | Explicit queue priority 20 with the normal global capacity limit |
| `%q(5, p=20)` or `%queue(runners=5, priority=20)`    | Both controls                                                    |
| `%q:5 %queue(p=20)`                                  | The same two controls, supplied in separate occurrences          |
| `%w(builder, time=5m) %q(1, p=20)`                   | Dependencies and time floor, followed by queue admission         |

1. The first and only positional argument is `runners`. Colon syntax supplies that
   argument; keyword arguments require parentheses. `p` canonicalizes to `priority`
   before duplicate checks. Generated prompts use explicit canonical
   `%queue(runners=N, priority=P)` syntax, omitting absent fields; the personal short
   snippet uses `%q:$1`.
2. Each canonical field may occur only once per expanded launch prompt/unit, including
   across repeated directives and aliases. Disjoint occurrences compose. Reject
   duplicates even when values match: `%q(5, runners=5)`, `%q(p=20, priority=20)`, and
   repeated assignments across `%q` and `%queue`. Reject extra positional arguments,
   duplicate literal keywords, unknown keys, bare `%q`, `%q()`, `%q:`, `%q+`, missing
   values, and malformed parentheses. Empty queue syntax must never acquire bare
   `%wait`'s previous-agent meaning.
3. Accept non-negative decimal integers, including zero. Reject negative, fractional,
   signed-plus, boolean, and nonnumeric values. Make overflow behavior identical in
   Python and Rust, with clear errors rather than wrapping or clamping. The existing
   launch wire bounds are `u32` for runners and non-negative `i32` for priority; test
   their boundaries explicitly. Preserve the distinction between omitted values and
   explicit zero or explicit default priority 10.
4. Preserve the current runner semantics: threshold means already-running count,
   `runners=0` is a drain barrier, explicit thresholds may exceed the global cap, lower
   priorities start first, equal priorities use FIFO, and the default priority
   remains 10. Preserve deference, family slot sharing, question-resume admission, and
   dependency/time-before-slot ordering.
5. After cutover, `%wait(runners=...)` and `%wait(priority=...)` fail with targeted
   migration errors showing `%queue` and how to split mixed waits. `%wait(p=...)` is
   unsupported. `%queue` rejects dependency/time keywords. There is no permanent silent
   compatibility alias and no repository-wide rewrite of archived prompts or live agent
   records. Existing stored `wait_runners`, `wait_priority`, explicit flags, and AXE
   `wait_runners` configuration retain their data contract.
6. Honor existing directive boundaries, protected literal/fenced/disabled regions,
   argument quoting, and fan-out scope. New syntax is stripped from model input. A
   queue-only prompt never invents a named dependency. Preserve remote V1's current
   restriction by rejecting `%dispatch` combined with `%q` / `%queue`; do not
   accidentally bypass the old `%wait` restriction. Queue directives are agent-only and
   must be rejected on typed `%proc` units instead of being dropped.

## Survey and implementation map

The planning survey opened every configured linked repository: `chezmoi`, `sase-core`,
`sase-github`, `sase-nvim`, `sase-research-artifacts`, and `sase-telegram`, plus
`gh:bbugyi200/bugyi-chops`. Searches covered bare `runners` and `priority`,
`wait_runners` / `wait_priority`, and old directive strings, including multiline calls
and hidden configuration. Repeat the survey on the implementation revisions, including
any newly configured linked repos.

| Repository              | Findings and required work                                                                                                                                                                                                                                                                                                       |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| sase                    | `_directive_collect.py`, `_directive_extract.py`, `_directive_values.py`, `_directive_types.py`, and `_directive_scan.py` currently associate both controls with wait. `_directive_edit_wait.py`, durable edit operations, and AXE regenerate that syntax. Tests, help text, docs, and a generated skill source also contain it. |
| sase-core               | `crates/sase_core/src/editor/directive.rs` owns shared name/keyword/value completion metadata. `agent_launch/mod.rs` separately parses wait queue fields; `agent_launch/admission.rs` emits old directives. Binding and LSP tests assert the old keyword lists.                                                                  |
| sase-research-artifacts | `src/sase_research_artifacts/xprompts/research_swarm.md` emits `%wait(priority={{ priority }})` for all four members; `tests/test_xprompt_loading.py` asserts it. Preserve the public swarm `priority` input and its null-versus-zero behavior.                                                                                  |
| bugyi-chops             | `src/bugyi_chops/toobig_split.py` emits `%wait(priority={LAUNCH_PRIORITY})`; `tests/test_toobig_split.py` asserts priority 20.                                                                                                                                                                                                   |
| chezmoi                 | `home/dot_config/sase/sase.yml` defines `r: "%w(runners=$1)"`. Seven generated provider copies of `sase_agents_status` mention the old spelling. `sase_athena.yml` has AXE `wait_runners: 3` and `wait_runners: 0`, which remain valid configuration inputs whose generated prompts must change.                                 |
| sase-nvim               | No queue-keyword references were found. Directive assistance comes from the LSP; add a headless completion smoke regression using the existing test style, without a parallel Lua directive catalog.                                                                                                                             |
| sase-github             | No matching queue-keyword references were found. Recheck workflow/template producers before declaring the audit complete.                                                                                                                                                                                                        |
| sase-telegram           | Keyword hits concern unrelated media ordering and dispatch documentation. No queue directive migration found; recheck prompt consumers.                                                                                                                                                                                          |

Historical sidecars are not active prompt producers. Do not bulk-edit archived plans,
chats, research, or beads. If an artifact is needed as context, consume it through
`sase artifact read`; consult canonical memory only through `sase memory read`.

## Phase core

Implement one Rust queue contract and reuse it across runtime paths and editor metadata.
Avoid extending the current discrepancy: Python rejects negative priorities and
duplicate assignments, while Rust's typed wait parser currently accepts negative `i32`
priorities and overwrites repeated values.

- Add a focused Rust module for queue argument normalization, validation, collection
  across occurrences, and canonical formatting. Reuse existing directive occurrence
  scanning, argument parsing, and protected-range machinery. Preserve source spans for
  actionable errors. Expose the needed pure API through `crates/sase_core_py`, its
  public exports, and binding tests. Python must call this API through a thin adapter
  rather than duplicate the queue rules.
- Update `agent_launch/mod.rs` to parse queue arguments into the existing queue fields
  without adding wait targets. Update `agent_launch/admission.rs` prompt generation.
  Test typed launch planning, serial/fan-out units, cleanup, agent-only restrictions,
  and parse/render round trips.
- Extend `editor/directive.rs` with `%queue` and alias `%q`, colon/parenthesized forms,
  non-negative positional runners, and `runners=`, `priority=`, `p=`. Keep numeric
  suggestions `0`/`1` for runners and `10`/`1` for either priority spelling. Suggestions
  are examples, not an allowed-value enumeration.
- Reuse keyword conflict metadata for `p` versus `priority`, and account for a supplied
  positional runners argument when suppressing `runners=`. Conversely, a supplied
  `runners=` suppresses another positional threshold. Preserve editing the active clause
  and suffix clauses when computing availability and ranges. Do not let queue
  positionals enter the dynamic agent target completion path.
- Include shared contract/wire, completion, hover, and known-directive diagnostics
  updates. Use `editor/completion.rs`, `crates/sase_xprompt_lsp/src/server.rs`, and
  JSON-RPC tests to verify both spellings and replacement behavior. Reuse existing
  assistance rather than implement a separate LSP parser or completion catalog.

Because intermediate epic phases can land before all consumers migrate, use one
temporary `queue_directive` beta flag, default off, created only during approved
implementation with `sase flag new`, following `sase_flags.md`. Its On behavior is the
final split contract; Off retains current wait parsing/generation/completion. Record the
removal condition as complete runtime/editor integration and conversion of all
maintained producers. Pass the resolved state explicitly through Rust entry points and
the existing ACE/LSP feature visibility plumbing; do not read global configuration on
keystrokes or create a backend fallback switch. Give disabled explicit new syntax a
useful error. Test both states. This scaffolding must be removed within this epic,
before final landing, rather than become a long-term compatibility feature.

Core validation: repository `just check` / `./scripts/check.sh all`, including PyO3 and
LSP workspace tests. Do not use `cargo test -p sase_core` as the gate or manually edit
release-plz-owned versions. Since flag registration also changes sase, run its required
`just check` lane after preparing the matching environment.

## Phase integration

- Register queue and q in the Python directive tables and stripping paths. Route
  collection and validation through the new core binding. Keep the existing
  `PromptDirectives.wait_runners` / `wait_priority` adapters and durable metadata shape
  unless a concrete binding requirement demands otherwise. Remove queue keywords from
  wait's enabled branch and produce the migration errors above.
- Update deferred-start detection used by `agent/launch_cwd_agents.py`,
  `launch_cwd_bead_work.py`, `multi_prompt_launch_plan.py`, and
  `multi_prompt_launch_execution.py`. Both threshold-only and priority-only queue
  prompts must follow the existing deferred admission path without acquiring a workspace
  prematurely. Add focused queue detection rather than misclassifying all queue
  directives as dependency waits.
- Replace `has_wait_runners_directive`'s regex-specific role with shared detection of an
  explicit queue threshold, supporting colon and parentheses and both aliases. In
  `axe/chop_proposal_models.py`, an authored threshold overrides the lumberjack default;
  priority-only `%q(p=20)` must still receive the configured threshold. The injected
  threshold and existing priority compose without duplicates. Preserve protected-region
  behavior and dry-run/actual-launch parity.
- Split queue payload/formatting from `PromptWaitDirective` using a queue-specific
  adapter. Provide independent wait and queue rewrites and a combined operation for the
  existing modal. Updating only dependencies preserves queue settings; updating only
  queue settings preserves all dependency kinds and `#t` time syntax. Canonical queue
  rewrites replace both aliases and are idempotent. When an explicit edit touches an
  existing prompt with old mixed wait syntax, split those queue fields during that
  rewrite and retain the dependencies; this does not authorize a background conversion
  of archived prompts.
- Wire `ace/tui/actions/agents/_wait_actions.py`, `_wait_helpers.py`, related live
  runner edit helpers, `_directive_persistence.py`, and `ops/commands/agent.py` to the
  combined edit. Preserve the current modal/keymap workflow, explicit-value clearing,
  Run now semantics, failed-operation rollback, and submitted/raw prompt, history, and
  stash synchronization. Queue updates must not leave a stale second directive or
  silently erase `unit=` / `proc=` waits. Put new shared text/domain behavior in Rust,
  keeping Textual state and worker dispatch in Python.
- Connect the shared completion contract to ACE and the LSP wrapper's temporary feature
  state. Check `_directive_completion_tokens.py`, candidate builders, completion panel
  classification, and prompt input interaction handlers for wait-specific assumptions.
  Static queue completion must work without agent or bead catalogs, subprocesses, or
  blocking I/O.
- Update runtime tests alongside these changes: directive extraction/scanning,
  runner-slot priority/admission, AXE prompt production, swarm inheritance, prompt
  editing, durable operation dispatch, and modal/live-resume behavior. Preserve both
  temporary flag states until the migration phase.

Install/rebuild the matching local Rust binding and LSP with the repository's
`just install` / `just rust-dev-install` workflow before integration tests. Verify the
interpreter and LSP binary used by the tests are the rebuilt versions. Run `just check`
in sase; use a SASE monitor for long commands.

## Phase migration

Convert maintained producers in the inventory table and all additional hits discovered
by the repeated sweep. Split mixed forms instead of renaming the whole directive:
`%w(builder, time=5m, runners=1, priority=20)` becomes
`%w(builder, time=5m) %queue(runners=1, priority=20)`.

- Convert bugyi-chops's launcher and assertion, retaining `LAUNCH_PRIORITY=20`. Convert
  all four research swarm emissions and tests while preserving `wait`, optional
  `priority`, explicit zero, and VCS/family inheritance. Check the plugins' README/docs
  for syntax claims.
- Convert the chezmoi `r` snippet to `%q:$1`. Preserve AXE `wait_runners` config keys
  and prove their preview and execution now emit `%queue`. Do not rename unrelated model
  routing priority, keybinding priority, notification priority, runner-limit
  configuration, or data fields solely because the words match.
- Update sase's `docs/xprompt.md` directive table, completion table, queue behavior
  section, and examples; update `docs/ace.md`, `docs/axe.md`, `docs/cli.md`,
  `docs/configuration.md`, `docs/integrations.md` where applicable, and
  `docs/troubleshooting/runner-slots.md`. Update runner panel help such as
  `models_panel_runner_limit_cards.py`, source comments, test fixtures, and public error
  text. Keep default config and help content consistent with any actual presentation
  changes; no new keybinding or CLI option is required.
- Update `src/sase/xprompts/skills/sase_agents_status.md`. Preview with
  `sase skill init --diff` or `--dry-run`. Never hand-edit the seven generated chezmoi
  copies or deploy from unlanded source; the landing procedure below owns their
  regeneration.
- Update the existing directive reference `sase/memory/xprompts.md` to document `%queue`
  / `%q`, positional runners, `p=`, and wait's reduced scope. This is the documentation
  change included in the approved migration. Before editing, use `sase_memory_write` and
  an audited read, then run `sase memory init`; do not hand-edit generated instruction
  shims or add a new core memory note.
- Once the new-path tests and migrated source tree are coherent, remove
  `queue_directive`: delete the Off branches and temporary flag propagation, make the
  split behavior unconditional, remove its registry entry, and close its flag bead
  through supported commands. Keep explicit rejection tests for the old wait spellings;
  remove obsolete positive compatibility expectations. The final completion contract
  advertises queue arguments only under queue.

Do not deploy migrated plugin/config producers to a runtime that lacks `%queue`. Prepare
installation prerequisites from actual release availability: core binding and LSP, then
sase runtime, then producer plugins and personal config. Review the existing dependency
windows (`sase-core-rs` in sase, `sase` in research and bugyi-chops) and use the normal
release process for minimum supporting versions; do not invent future version numbers or
hand-bump Rust workspace versions.

## Phase verification

Verify the final unconditional behavior on the combined revisions, and repair
integration defects before handing it to the land agent.

1. **Parser and scheduler:** exercise every equivalent spelling in the contract,
   zero/default/upper-bound values, all duplicate and malformed cases, mixed wait plus
   queue prompts, fenced/disabled literals, and fan-out. Compare Python extraction to
   Rust typed launch results for the same corpus. Test queue-only threshold and priority
   launches, explicit override of AXE defaults, metadata persistence/restart, drain
   barriers, priority/FIFO order, and workspace deferral. Extend the existing runner
   tests and `tests/fakey/test_runner_slots_e2e.py` where needed; do not change
   scheduling policy to make syntax tests pass.
2. **Real editor parity:** extend `tests/test_xprompt_directive_completion_parity.py`
   and the ACE candidate/extraction/interaction suites. Cover `%`, `%q`, `%que`, `%q:`,
   `%queue(`, positional `5, `, `runners=`, `priority=`, and `p=`; alias
   conflict/positional suppression; numeric partial filtering; accepting a completion
   beside existing text and closing parentheses; and UTF-16 ranges after non-BMP
   characters. Compare insertions, descriptions, and replacement edits against a real
   rebuilt LSP. Assert no agent targets in queue positions and no queue keys in wait
   completion. Check hover and absence of unknown-name diagnostics. Exercise Neovim
   through a small headless LSP completion smoke test modeled on
   `sase-nvim/tests/lsp_snippet_smoke.lua`.
3. **Editing and replay:** assert independent wait/queue rewrites and combined modal
   changes preserve untouched fields, protected text, frontmatter, and branch scope.
   Test priority clearing separately from setting zero, Run now, live queued edits, and
   durable raw/submitted/history/stash round trips. No operation should recreate retired
   wait keywords.
4. **Repository gates:** run sase `just check`, core `just check` including all
   bindings/LSP tests, and `just check` for changed Python plugins. Run appropriate
   Neovim headless tests and validate the chezmoi snippet/config. If visible UI content
   changes, inspect and update only intentional affected PNG snapshots and run
   `just test-visual`. Run sase `just check-full` through `sase_monitor` with
   TESTING/TESTED before landing the combined epic, as required by `lint_and_test.md`;
   never run the exhaustive lane inline.
5. **Final audit:** repeat both a broad keyword search and a multiline old-wait
   directive search in the primary and every opened linked/external code repo, including
   templates, hidden config, tests, and docs. Classify every remaining hit:
   migration-error tests/text, historical material, generated copies awaiting the
   required post-land generation, unchanged structured data/config keys, or unrelated
   uses. There must be no active maintained producer emitting the old syntax. Record
   repositories with no required change as well as converted ones.

## Landing and completion

Use host-owned finalization for every changed repository; phase workers do not create
manual commits or branches. Recheck linked-repo changes and dependency availability
together so the LSP, binding, runtime, and prompt producers advance coherently. The land
agent owns `just check-full` evidence and final deployment sequencing, including any
required SASE monitor continuation.

After the skill source has landed, regenerate provider skills from the clean, merged
source with `sase skill init --force`, following `generated_skills.md` and the
sanctioned repository workflow. Apply chezmoi as required, including
`chezmoi update -a --force` after its host-owned commits. Arrange a host follow-up when
finalization must complete before deployment; do not bypass the clean-source guard.
Confirm all generated `sase_agents_status` copies and the active `r` snippet use the new
directive. This deployment and its final reference audit are part of completion, not an
untracked follow-up.

Acceptance requires all user examples to produce the same admission values as before,
matching prompt-widget and external-editor assistance, and no maintained source or
deployed generated skill still teaching or emitting queue kwargs on `%wait` except
explicit migration guidance.
