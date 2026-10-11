---
tier: epic
title: Retire every current feature flag while preserving enabled behavior
goal: Remove all 23 registered feature flags and their disabled implementations across
  sase and sase-core, preserve today's all-enabled behavior, and leave an empty, usable
  flag registry through strictly sequential implementation phases.
decisions:
  macro_memory:
    ask: Update the macro memory note to describe typed Proc launches as unconditional?
    memory:
    - macros.md
    default: false
    answer: false
  refresh_memory:
    ask: Update the TUI performance memory note to describe refresh tokens as unconditional?
    memory:
    - tui_perf.md
    default: false
    answer: false
phases:
- id: fixture-foundation
  title: Make flag infrastructure tests independent of production flags
  size: medium
  depends_on: []
  description: 'fixture-foundation: Follow the shared contracts and phase 1 below.
    Replace real rollout keys in generic feature-flag framework, CLI, state, doctor,
    checker, and Flags-pane fixtures with test-only beta/sunset definitions. Cover
    an empty registry and stale saved/environment overrides. Preserve product behavior
    and the current registry in this phase. Run focused tests and the required check
    before completion; do not launch helpers.'
- id: typed-launch
  title: Make typed Agent and Proc launches unconditional
  size: medium
  depends_on:
  - fixture-foundation
  description: 'typed-launch: Follow phase 2 and the shared removal checklist. Retire
    typed_launch_units and close sase-s7. Remove Python and Rust/LSP opt-in checks,
    disabled diagnostics, hidden-completion paths, and obsolete environment transport
    while preserving mixed-unit dispatch, static and script conditions, recovery,
    and code-fence safety. Keep queue_capacity_budget functional for phase 3. Verify
    both repositories sequentially and include the core revision pin in host finalization.'
- id: queue-budget
  title: Make queue capacity budgets unconditional
  size: medium
  depends_on:
  - typed-launch
  description: 'queue-budget: Follow phase 3 and the shared removal checklist. Retire
    queue_capacity_budget and close sase-zv across Rust parsing/admission/editor behavior
    and Python launch, bead, and TUI adapters. Delete the Off threshold semantics
    and finished launch/editor flag plumbing, including already-retired constant shims
    where proven unused. Preserve supported persisted queue records and percent-hold
    behavior. Verify the Rust core and Python checkout sequentially.'
- id: macro-contracts
  title: Stabilize macro aliases and strict input types
  size: medium
  depends_on:
  - queue-budget
  description: 'macro-contracts: Follow phase 4 and the shared removal checklist.
    Retire legacy_xprompt_syntax and strict_macro_input_types; close sase-1fj and
    sase-1g9. Preserve the On branch accepting xprompt aliases, legacy discovery and
    plugin names, while removing rollout rejection policy and unknown-type-to-line
    fallback. Make Rust config and LSP normalization follow the same unconditional
    semantics. Keep duplicate-name and malformed-input errors and durable readers.'
- id: identity-aliases
  title: Stabilize agent-session and turn compatibility aliases
  size: medium
  depends_on:
  - macro-contracts
  description: 'identity-aliases: Follow phase 5 and the shared removal checklist.
    Retire legacy_agent_family_syntax and legacy_sase_shell_syntax; close sase-18l
    and sase-1ar. Keep all currently enabled alias normalization, including Rust''s
    family keyword alias, and delete the flag-off rejection branches. Preserve canonical
    output, conflict validation, and old durable records. Do not interpret legacy
    flag names as permission to remove their enabled compatibility behavior.'
- id: agents-query
  title: Remove the legacy live Agents query implementation
  size: medium
  depends_on:
  - identity-aliases
  description: 'agents-query: Follow phase 6 and the shared removal checklist. Retire
    agents_unified_query and close sase-zg. Make the Rust agents-live profile and
    FilterBar unconditional. Delete the legacy live parser/evaluator, QueryEditModal,
    and dependent dead state after checking every consumer. Preserve current saved-query
    compatibility, history scoping, Rust pushdown, and asynchronous refresh performance.
    Verify behavior and affected visual fixtures.'
- id: tui-refresh
  title: Stabilize refresh tokens, refresh gestures, and the Flags pane
  size: medium
  depends_on:
  - agents-query
  description: 'tui-refresh: Follow phase 7 and the shared removal checklist. Retire
    ace_refresh_tokens, admin_center_flags, ref_sync_gesture, and refresh_panel; close
    sase-wr, sase-rx, sase-qu, and sase-105. Preserve token-based refresh, Flags-pane
    availability, the reference-sync gesture, and Refresh-panel key behavior. Delete
    disabled paths and special rollout self-disable UI. Keep dirty/sanity recovery
    and perform targeted TUI verification without parallel workloads.'
- id: publication-services
  title: Stabilize publication formats and service contracts
  size: medium
  depends_on:
  - tui-refresh
  description: 'publication-services: Follow phase 8 and the shared removal checklist.
    Retire slim_agents_manifest, agents_session_manifest_compat, bgcmd_legacy_slots,
    and axe_routine_job_contract; close sase-11p, sase-1ft, sase-13w, and sase-11f.
    Write only slim manifests and canonical routine/job output; retain enabled explicit-file-set
    compatibility and legacy slot reading/actions. Preserve data-integrity checks
    and existing accepted input normalization.'
- id: monitor-records
  title: Remove legacy monitor-start rollout paths
  size: medium
  depends_on:
  - publication-services
  description: 'monitor-records: Follow phase 9 and the shared removal checklist.
    Retire monitor_continuation_records and close sase-102. Every new monitor uses
    versioned records and capture. Remove the selectable legacy writer/start path,
    while keeping existing monitor settlement and recovery governed by persisted protocol
    and sentinel fields. Exercise success/failure delivery, idempotency, recovery,
    and older persisted monitor records.'
- id: provider-instructions
  title: Stabilize provider instruction and execution channels
  size: medium
  depends_on:
  - monitor-records
  description: 'provider-instructions: Follow phase 10 and the shared removal checklist.
    Retire muse_synchronous_shell, grok_rules_delivery, claude_helper_channel, and
    instruction_shadow_render; close sase-178, sase-1gv, sase-1gw, and sase-1h4. Keep
    synchronous Muse execution, root Grok rules, supported Claude helper channels/guards,
    and fail-open shadow rendering. Delete flag resolution and disabled variants without
    converting shadow rendering into a new delivery cutover.'
- id: runtime-controls
  title: Stabilize sudo requests, provider drains, and autonomy records
  size: medium
  depends_on:
  - provider-instructions
  description: 'runtime-controls: Follow phase 11 and the shared removal checklist.
    Retire agent_sudo_requests, provider_drain, and autonomy_record_only; close sase-111,
    sase-sx, and sase-1j0. Remove beta refusal/no-drain/legacy-write branches. Preserve
    typed sudo review safeguards, automatic durable drains and current toasts, canonical
    autonomy writes, legacy read projection, and the plan-flow marker. Finish with
    an empty production registry.'
- id: clean-slate
  title: Verify the empty registry and complete retirement cleanup
  size: medium
  depends_on:
  - runtime-controls
  description: 'clean-slate: Follow phase 12. Audit all 23 flags, closure evidence,
    residual wrappers and literal-key checks across the changed repositories. Finish
    docs, synthetic examples, empty-state visuals, schema and managed config cleanup;
    verify upgrade startup with stale overrides. Apply each memory edit only if its
    corresponding decision is accepted, otherwise record the prescribed proposed follow-up.
    Run final verification sequentially and supply the land agent a complete flag-to-evidence
    ledger.'
proposed_by: bbugyi200.athena.0zb
decided_by: auto
create_time: 2026-10-09 22:28:21
status: done
bead_id: sase-1jc
---

- **PROMPT:** [prompts/202610/retire_all_feature_flags.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/retire_all_feature_flags.md)
- **BEAD:** [sase-1jc](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1jc/README.md)

# Retire the current feature flags

## Outcome and execution contract

The user requests a clean slate after using every feature enabled, and explicitly
requires phase agents to run one after another to protect the shared machine. This is an
epic because its independent correctness boundaries include Rust and Python launch
semantics, editor assistance, TUI performance, durable monitor protocols, and provider
execution. Every phase is bounded direct implementation work, sized `medium`; no phase
requires a nested planning epic.

**Concurrency is exactly one phase at a time.** The frontmatter forms a single chain,
with no independent roots or parallel waves. A successor may begin work only after its
predecessor has completed verification and host finalization. Workers must not spawn
helper agents, launch parallel swarms, or detach verification that overlaps the next
phase. The land agent runs after the final phase. Use this graph rather than changing
Bryan's global scheduler capacity or stopping unrelated work. A required monitor
continuation remains part of its phase until settled. Run Rust builds, Python checks,
and visual captures one at a time. Request one Python test worker and one Rust build job
through the supported recipe/environment controls where applicable; use
`SASE_PYTEST_WORKERS=1` and `CARGO_BUILD_JOBS=1` for the corresponding checks. No stress
tests or new multi-machine soak is needed.

The target is **zero registered rollout flags**, with the existing flag machinery
available for future work: registry type, resolver, CLI, schema generator, saved state
reconciliation, flag beads/triage, doctor/checker, and Config Flags pane. Permanent
configuration choices, ordinary CLI options, Cargo build features, and runtime state
booleans are not rollout flags and remain. Do not replace removed flags with permanent
config toggles or helpers that merely return `True`.

The user's explicit retirement request supersedes dossier waiting dates, release-count
criteria, and old notes saying to wait. Closing a dossier must cite this authorization
and actual regression evidence; do not claim unperformed soaks or releases. All 23 flags
resolved enabled in the planning session, including the three beta flags. All 23
associated task beads were open when audited.

## Inventory and semantics

The source of truth is `src/sase/feature_flags/registry.py`, cross-checked with audited
`sase bead read` calls for every dossier. Apply the uniform rule: delete the Off branch
and make the On branch unconditional.

| Phase                 | Flag                             | Dossier    | Behavior that survives                                                                       |
| --------------------- | -------------------------------- | ---------- | -------------------------------------------------------------------------------------------- |
| typed-launch          | `typed_launch_units`             | `sase-s7`  | Mixed Agent/Proc launch units, script conditions and code inputs, matching editor assistance |
| queue-budget          | `queue_capacity_budget`          | `sase-zv`  | Per-launch runner-capacity budget, including multiplier/weight semantics                     |
| macro-contracts       | `legacy_xprompt_syntax`          | `sase-1fj` | Accepted xprompt aliases, discovery paths, plugin and environment spellings                  |
| macro-contracts       | `strict_macro_input_types`       | `sase-1g9` | Unknown input types cause per-macro load errors with suggestions                             |
| identity-aliases      | `legacy_agent_family_syntax`     | `sase-18l` | Family spellings normalize to agent-session spellings                                        |
| identity-aliases      | `legacy_sase_shell_syntax`       | `sase-1ar` | Shell spellings normalize to turn spellings                                                  |
| agents-query          | `agents_unified_query`           | `sase-zg`  | Shared agents-live Rust query engine and FilterBar                                           |
| tui-refresh           | `ace_refresh_tokens`             | `sase-wr`  | Stat-only token gating with dirty and sanity-interval recovery                               |
| tui-refresh           | `admin_center_flags`             | `sase-rx`  | Config always offers its Flags pane, including its empty state                               |
| tui-refresh           | `ref_sync_gesture`               | `sase-qu`  | Empty-payload second colon refreshes the reference catalog                                   |
| tui-refresh           | `refresh_panel`                  | `sase-105` | R opens Refresh; leader-y opens it on Full history                                           |
| publication-services  | `slim_agents_manifest`           | `sase-11p` | All publication/repair writers omit per-hood file lists                                      |
| publication-services  | `agents_session_manifest_compat` | `sase-1ft` | Core validates canonical and precisely supported legacy explicit file sets                   |
| publication-services  | `bgcmd_legacy_slots`             | `sase-13w` | Existing legacy slots remain readable and actionable beside durable proc rows                |
| publication-services  | `axe_routine_job_contract`       | `sase-11f` | Public output uses routine/job names; existing accepted inputs normalize                     |
| monitor-records       | `monitor_continuation_records`   | `sase-102` | New starts use durable versioned continuation records                                        |
| provider-instructions | `muse_synchronous_shell`         | `sase-178` | Muse uses `--enable-shell-tool` and its synchronous single-turn directive                    |
| provider-instructions | `grok_rules_delivery`            | `sase-1gv` | Root Grok invocations receive rules through the current channel                              |
| provider-instructions | `claude_helper_channel`          | `sase-1gw` | Supported helper prompt channel and PreToolUse settings/guard                                |
| provider-instructions | `instruction_shadow_render`      | `sase-1h4` | Root shadow bundle/manifest/meta capture and scoped environment export                       |
| runtime-controls      | `agent_sudo_requests`            | `sase-111` | Typed sudo request/review workflow is available                                              |
| runtime-controls      | `provider_drain`                 | `sase-sx`  | Hard disables submit durable automatic drains                                                |
| runtime-controls      | `autonomy_record_only`           | `sase-1j0` | Canonical autonomy-only writes and no legacy auto-approve environment export                 |

Several flags have misleading retirement names. The enabled branch of the three
`legacy_*_syntax` flags **accepts** aliases. Likewise `bgcmd_legacy_slots` enables
reading old slots, and `agents_session_manifest_compat` enables supported legacy file
sets. These are live behavior and remain. The registry's sentence suggesting removal of
Rust's hidden `family` alias conflicts with the actual On branch and dossier's removal
rule; preserve that alias. Alias removal or deleting old user data would be a separate
behavior change, outside this request.

Persisted-version dispatch is also distinct from rollout dispatch. The current On branch
still reads old monitor protocols, manifests, queries, and autonomy records. Keep that
supported recovery behavior while deleting obsolete _new-write_ choices.

## Shared implementation and verification checklist

Every removal phase must finish a coherent, passing intermediate tree:

1. Read its assigned dossiers with `sase bead read ... -r ...`, and trace both direct
   `FeatureFlag` references and indirect helper calls. Search literal keys, environment
   transport, bindings, fixtures, doc examples, and configuration too. Inspect actual
   current source if it has advanced since this plan was written.
2. Inline the On path; delete Off branches, dead-only helpers/modules, alternative
   metadata, stale imports/exports, and disabled-only fixtures. Preserve validation,
   security checks, failure handling and data readers that the On path uses. Retain
   enabled behavior tests as unconditional regression tests. Do not delete assertions
   simply to make tests pass or remove useful shared APIs by name alone.
3. Remove assigned enum members and registry definitions. Update any test roster
   assertions, feature-specific fixture overrides and managed YAML overrides in the same
   phase. Regenerate `src/sase/config/sase.schema.json` with
   `tools/sync_feature_flags_schema --write`; never hand-delete its generated rows.
4. Update current product documentation and help that describe these flags or an opt-in
   command. Leave historical plans, immutable decisions, event logs and archived records
   intact. Update `src/sase/default_config.yml` when relevant keymap/config
   documentation changes, preserving current enabled bindings.
5. Run meaningful focused regressions, then close assigned flag dossiers with
   `sase bead close <id> --note '<authorization and verified removal evidence>'` after
   their definitions/branches are gone, before the whole-repo integrity gate. This
   avoids a live orphan bead or a closed bead with a surviving definition. Do not launch
   23 removal-task agents or create duplicate tasks. If retirement is reverted, restore
   the matching dossier lifecycle rather than leave it closed.
6. Read the current `lint_and_test.md` memory. Format, run the feature-flag/schema
   checks using their documented local entry points, and run `sase tool run check` in
   every modified code repository. This is the required `just check` gate. Fix failures
   caused by the change; use evidence-based triage for other failures. `just check-full`
   is not requested. Do not globally install or restart Bryan's running services as
   verification.
7. For rendered TUI changes or altered visual fixtures, use targeted
   `just fix-tui-screenshots -- <selectors>` with the supported single-worker option,
   inspect its report and golden diff, and address partial/skipped results. If stale
   goldens need deletion, follow the memory's required full-capture evidence workflow,
   once and sequentially. Existing enabled-screen appearance should generally stay
   unchanged. Use `/sase_monitor` for commands requiring handoff; completion and golden
   review belong to this phase, not its successor.
8. Record removed paths, tests/results, retained compatibility readers and closed
   dossiers in the phase evidence. Finalize through `/sase_final`; let host-owned
   finalization commit. Phase workers record out-of-scope discoveries as
   `PROPOSED FOLLOW-UP:` notes on their own phase, as required by bead memory.

### Rust and linked repository coordination

Open `sase-core` with `/sase_repo` in each worker's workspace and use only its printed
path. Read its `AGENTS.md`. Shared parser, admission, validation and editor semantics
belong in `sase_core`; Python remains an adapter. Changes include the PyO3 bindings, LSP
callers and tests as necessary. Use the repo's recipes, not bare Cargo, and test the
built extension from the checkout rather than a stale globally installed package. Never
silently substitute Python backend behavior.

Within phases 2-4, update both repositories coherently. Preserve exported binding names
and optional argument acceptance where needed for the published compatibility window; an
old optional rollout argument may be accepted and ignored at the binding boundary, but
must not select an Off branch. Remove internal boolean/string flag plumbing once unused.
Record any intentionally retained wire compatibility fields. If a binding/wire really
must break, follow the linked repo's breaking-change and schema-version conventions and
update all consumers/fixtures explicitly.

Declare both repositories for host finalization. As documented in `docs/rust_backend.md`
under the CI source revision pin, the host commits core first and updates
`sase-core-revision.txt` before the primary commit. Verify the resulting pin includes
the core changes before the next phase relies on them. Do not invent a SHA, manually
commit, or pin uncommitted code. The same rule applies if other phases discover a
necessary core edit.

The planning audit also opened `sase-nvim`: no current flag keys or environment toggles
were found there; its queue LSP smoke test exercises the core contract. Reopen it
through `/sase_repo` if needed for a real consumer update. Do not edit it speculatively.
Other linked repos are in scope only where an actual flag consumer is found; always open
them through the skill before reading or changing files.

## Phase details

### 1. fixture-foundation

Generic framework tests currently depend on live keys, for example
`tests/feature_flags/test_consumers.py`, `test_cli_journeys.py`, `test_state.py`, and
the Flags-pane tests/PNG fixture builders. Reuse `tests/feature_flags/_helpers.py`'s
synthetic definitions and isolate patched registry/snapshot caches per test. Generic
tests should use explicit synthetic beta/sunset keys; production behavior tests remain
with their owning phase. Do not install fake members into the production enum or weaken
registry integrity.

Cover resolution precedence, saved preference mutation, child-process transport, CLI
unknown-key errors, orphan/closed-bead checks, and empty list/pane behavior. Add or
adapt a meaningful empty-registry startup test containing old saved keys and inherited
`SASE_FEATURE_FLAGS` values. Exercise the existing Rust-backed reconciliation, including
preservation of unrelated state and corrupt-state error handling. Existing unknown-key
diagnostics are acceptable; startup must succeed and child snapshots must stop exporting
unregistered keys. No real home-state mutation, scheduler restart, or new production
flag is necessary.

Completion: fixture coverage works with a deliberately empty registry; ordinary
production behavior is unchanged; later removal phases can delete their keys without
breaking generic framework assumptions.

### 2. typed-launch

Trace `src/sase/macro/code_value.py`, `_directive_collect_code.py`,
`_directive_extract.py`,
`src/sase/agent/{launch_request*,direct_typed_launch.py, launch_cwd_guards.py,launch_hold_preview.py}`,
the AXE proposal launch adapter, and `src/sase/core/agent_launch_facade.py`. Delete
`typed_launch_units_enabled`, disabled-use rejection and opt-in messages after updating
callers. Always validate owned code fences and preserve their shielding from ordinary
macro expansion. Keep static `%if(should_run=...)` behavior alongside script `%if::`,
`%proc(...)`, `%proc::` and `type: code`.

In core inspect `fenced_code.rs`, `agent_launch/typed_units.rs`,
`editor/{wire, directive,completion,diagnostics}` and PyO3
`agent_launch`/`editor_completion`. In `sase_macro_lsp/src/server` remove the
typed-launch setting, environment reader, disabled diagnostics, and conditional
recipes/hover/actions. Remove host export of `SASE_TYPED_LAUNCH_UNITS` in
`src/sase/integrations/macro_lsp.py`. Both direct LSP startup and host-started LSP
expose the enabled feature without a flag list. Keep queue-specific flag transport until
phase 3.

Verify mixed/all-agent launches, code opacity and malformed fences, true/false
static/script admission, skipped units, working-directory validation, Proc recovery and
idempotency, plus completion/hover/diagnostics and Python binding round trips. Use
subprocess fakes/hermetic fixtures, not live agents or real Proc side effects.

### 3. queue-budget

Trace `macro/queue_directive.py`, `bead/work_queue_capacity.py`, TUI queue/weight badges
and wait/header sections, core `queue_directive.rs`, launch admission, typed plans,
editor metadata and LSP settings. Make capacity budget semantics unconditional: keep
multiplier handling, weights, priority/FIFO, and the enabled validation of invalid/zero
capacities. Delete the old pre-admission threshold, zero-drain branch and Off
completion/help metadata. Preserve whatever historical queue record decoding is required
by today's enabled path.

Remove `SASE_QUEUE_CAPACITY_BUDGET`, corresponding LSP initialization switches, and
obsolete host flag-list transport after both typed/queue consumers are gone. Audit
existing vestiges `AGENT_HOLDS_FLAG`/`SASE_AGENT_HOLDS` and `QUEUE_DIRECTIVE_FLAG`: core
hold collection already ignores its feature list and the old queue directive gate is
already unconditional. Remove dead transport and no-op internal wrappers without
disabling `%hold` or changing holds' durable semantics. Keep genuinely generic
compatibility signatures at external boundaries only as described above.

Verify direct core calls with omitted/empty old flag arguments as well as Python calls;
neither may restore Off behavior. Cover capacity vs global limit, weighted admission,
multiplier serialization, queued Proc dispatch, user display, and epic `--capacity`
composition. This epic's serial execution is enforced by dependencies, not by relying on
the flag being retired.

### 4. macro-contracts

Trace `legacy_xprompt_syntax.py`, config bootstrap/loading/inventory/edit adapters,
content layout and discovery, frontmatter/workflow loaders, plugin discovery, macro
skill adapters, TUI catalogs/keymap normalization, CLI entry normalization, doctor's
retired-name reporting, and `integrations/macro_lsp.py`. Retain enabled alias mappings
and conflict errors, canonical write locations, legacy directory/plugin discovery
precedence, environment conflict detection and the LSP binary fallback. Remove rollout
checks and disabled-only diagnostics.

Make the On policy unconditional in core `config/macro_syntax.rs` and relevant
editor/catalog/LSP normalization. Remove `SASE_ACCEPT_LEGACY_XPROMPT_NAMES` as a
behavior switch along with the LSP initialization option; old optional binding fields
must no longer change policy. Avoid recursing from raw config bootstrap back into
feature-flag resolution. Preserve presence-based collision checks, even when the legacy
value is null, false, empty text or an empty mapping.

In `macro/_loader_parsing_inputs.py`, remove unknown-input-type fallback to `line`. Keep
valid scalar/domain/plugin type resolution, suggested corrections, and per-macro error
isolation. Verify discovery, config startup, both-name conflicts, legacy
environments/plugin entry points, canonical writes, strict invalid types and editor
parity. Read generated-skill memory if actual skill source changes prove necessary; do
not edit installed generated skills by hand.

### 5. identity-aliases

Simplify `agent/legacy_agent_family_syntax.py` and `agent/legacy_sase_shell_syntax.py`
plus callers in attach, directive editing, Agents query normalization, gate/proc parser
handlers, notification-gate models, and gate config normalization. Remove only
flag-driven rejections and persisted vs authored splits that become redundant. Retain
family/session and shell/turn dual-input conflict errors, canonical environment
precedence, query AST rewriting, limit tokens and canonical output. Keep Rust's hidden
`%id(..., family=...)` normalization; test that Python and core agree.

Verify authored aliases and persisted gates/continuations/procs still load; canonical
inputs and unrelated words such as free-text `family` remain untouched. Delete
rejection-only tests and update current docs to describe accepted aliases without a
temporary flag contract.

### 6. agents-query

Start with `ace/tui/models/agent_live_query_engine.py` and all callers of
`agents_unified_query_enabled`, including `agent_loader`, loading compute/finalize,
query persistence, prospective clans, node jumps, machine filters, help and availability
checks. Use the Rust profile/corpus and FilterBar everywhere. Remove the legacy live
evaluation path, its history digest, modal exports/stubs and dead query-only state.

`sase.ace.agent_query` currently still has imports in saved-query canonicalization and
machine/project term builders. Trace these before deleting its package: migrate live
utility consumers to the shared profile, and preserve the enabled saved-query
compatibility contract without retaining a second live evaluator. An existing
`agents-legacy` saved record must receive the same safe acceptance or warning/rejection
as it does with On today; do not silently reinterpret queries or erase the user's saved
file. Do not delete shared `sase.ace.query` code.

Verify boolean filters, aliases from phase 5, invalid filters and last-good state,
history keys/dialect handling, limits, recent/full-history reconciliation, index
pushdown and query completeness. Keep corpus construction and disk work off the event
loop/pump and preserve incremental row updates. Read TUI performance memory.

### 7. tui-refresh

Remove token-mode alternatives in `ace/tui/actions/event_refresh` and
`proc_observer.py`, preserving watcherless token probes, dirty events, exact agent
deltas, initial load and sanity-interval refresh. Tests must assert unchanged tokens
skip expensive reads and changed/dirty/sanity states still reload.

Make the Flags catalog entry and config-hub routing unconditional. Remove the
`admin_center_flags` self-disable special case in pane rendering; generic disable
confirmation for future flags remains. Migrate populated pane fixtures to synthetic
flags and keep the empty pane reachable.

Remove the feature guard in `widgets/_artifact_ref_sync.py`, keeping the precise
empty-payload/mode trigger, in-flight coalescing, failure recovery, catalog refresh and
newly arrived badges. Nonmatching colons still insert literally. Remove only the
flag-off dispatch in `actions/refresh_panel.py` and its callers: underlying
tab/full-history refresh methods remain the panel's actions. Verify R/leader-y, focus
restoration, help and keybinding hints. Run affected widget/performance regressions and
targeted visuals sequentially.

### 8. publication-services

In `agents_sync/publication_planning.py` and `publication_repair.py`, remove fat writer
branches, `slim` arguments and computations used only for those writes. All
fresh/digest-repair/manifest-repair outputs omit per-hood file lists. Continue reading
old explicit lists and validating actual digests, sizes, paths and allowed file sets. In
`publication_validation.py`, always use core `classify_session_manifest_files`; delete
the Off strict-only comparison while retaining required canonical file-set helpers used
elsewhere.

In `ace/tui/bgcmd.py`, inline legacy slot reading alongside durable oneshot rows.
Preserve identity, kill/dismiss/output/rerun behavior and avoid resurrecting any legacy
slot writer. Do not delete actual slot directories.

In `axe/{cli,config_backend}.py` and `config/{core,inventory}.py`, always request
canonical routine/job projections. Keep accepted legacy input normalization and internal
domain names where still used; this is not an internal rename project. Remove unused
legacy-output-only projection code, in core if owned there.

Verify all manifest write routes, supported old explicit sets and rejection of
missing/extra/tampered files; test canonical CLI/config JSON, accepted config inputs,
and mixed legacy-slot/durable-proc presentation/actions using temp stores.

### 9. monitor-records

`continuation_capture/rollout.py` separates new-start flag selection from
`monitor_continuation_records_enabled_for_meta`, which derives stored protocol. Make new
starts and capture unconditional in monitor start flow, continuation intent/result
adapters and their callers. Remove only new-start legacy writer selection and
rollout-only capture environment fallback where now redundant.

Retain persisted protocol/sentinel classification for settlement, follow-up,
reconciliation and restart recovery. A monitor explicitly stored as legacy must not
suddenly be settled as `records_v1`; malformed/unknown protocol handling must remain as
strict as the enabled implementation. Delete a legacy launch function only if no
supported stored-monitor recovery can reach it.

Verify records and frozen policy are written on every new start; success, failure,
checkpoint/evidence capture, delivery/adoption, completion and budget recovery remain
correct and idempotent. Test stored v1 and legacy records independent of removed flag
values. Use mocked monitor processes rather than launching workloads.

### 10. provider-instructions

Simplify `llm_provider/_muse_directive.py`, `muse_provider.py` and
`instructions/directives.py` to the synchronous shell argument and directive; delete the
Off managed-bash wording and rollout helper. Keep actual single-turn wait protection,
ceilings and continuation limits.

In `llm_provider/grok.py`, always apply today's root rules delivery where that provider
context requires it. In `_claude_helper_channel.py`, `claude.py`, instruction assembly
and doctor, remove rollout checks while retaining the CLI capability probe, root/helper
role separation and required PreToolUse guard. Availability of a provider option is not
a feature flag and still matters.

In `_instruction_boundary.py`, remove the flag helper and skipped-render variant.
Preserve artifacts-dir requirements, scoped `SASE_INSTRUCTIONS_FILE`, metadata and
manifest naming, non-root exclusions, recorded errors and fail-open rendering. Shadow
rendering stays shadow rendering; do not deliver the shadow bundle or claim E3's
instruction-channel cutover has happened.

Verify argv and instruction text with provider fakes, supported/unsupported Claude
capabilities, retries, root/helper calls, shadow success/error, and environment
restoration. No live model invocations are needed.

### 11. runtime-controls

Delete `sudo/feature.py` and its `require_sudo_requests_enabled` calls/opt-in hints
after tracing CLI and gate callers. Preserve typed payload validation, reviewed
approval, runner policy, single-use authority, cancellation, SSH relay and persisted
gate inspection. Verify the workflow with fake privileged execution.

Remove the provider-drain flag helper in `llm_provider/usage_limit_disable.py` and calls
from models-panel workers/drain handling. Keep automatic drain submission for
usage-limit and manual hard disables, deduplication, stranded/skipped outcomes, one
enriched notification and current start/completion toasts. The latest dossier note says
the manual prompt modal was already deleted; do not recreate it. Preserve soft-disable
semantics and no-op/failure handling. Test with fake provider/process state, without
disabling a real provider.

In `autonomy/record.py` remove `record_only` and legacy write projection branches, plus
its callers in runner environment, scan projection, TUI retune/revive/directive
persistence. Keep Rust-owned live record evaluation and legacy read translation needed
for existing records. Preserve the dual-use `plan` flow marker: it is not simply
obsolete auto state. New launch/mutation writes must omit legacy autonomy keys and
`SASE_AGENT_AUTO_APPROVE`; old records and mixed-version scan projections must still
behave as today's On path. Verify toggles, successor inheritance, revive/retune, and
host checkpoint evaluation in the autonomy contract suite.

### 12. clean-slate

Audit the inventory table against source and dossier state. Confirm an empty
`FeatureFlag` enum/definition mapping is valid; no dummy production member is allowed.
Search both code repositories for every removed key and wrapper name, including direct
string lookups, `_enabled` helpers, boolean wire selectors, LSP initialization options
and environment exports. Function names such as `plan_typed_launch_units` or
`provider_drain` can legitimately remain: classify hits semantically rather than
deleting features by substring. Only historical records, retirement regressions or inert
external compatibility arguments may retain retired rollout identifiers; no active
choice of Off behavior remains.

Finish the current documentation inventory in `docs/configuration.md`, `editor.md`,
`macros.md`, `ace.md`, `architecture.md`, `axe.md`, `cli.md`, provider docs and help.
Use explicitly illustrative synthetic names for framework examples, explain that the
registry is currently empty, and remove stale enable-this-feature instructions. Keep the
flag creation/lifecycle documentation for future flags. Remove managed overrides only
where found in source-owned configuration; inspect any external config repo through
`/sase_repo` before changing it. Do not hand-edit personal state or rewrite old
artifact/event history.

Verify fresh-process CLI/TUI startup with no override transport, empty transport, and
old transport with both true and false retired values. None may restore a disabled path.
Exercise actual empty-registry saved-state reconciliation using temporary state roots,
including the three former beta preferences and an unrelated preserved field. The next
normal startup uses this same cleanup for Bryan's saved preferences. Verify
`sase flag list --json` gives an empty flags array and that CLI/Flags-pane empty states,
schema/checker/doctor remain usable. Test future flag creation using an isolated
fixture/store, without adding a real flag or dossier to the project.

> [!decision] macro_memory

If accepted, use `/sase_memory_write` and update only `sase/memory/macros.md`'s
temporary beta/rollout wording for the now-unconditional typed features.

> [!decision] refresh_memory

If accepted, use `/sase_memory_write` and update only `sase/memory/tui_perf.md` rule
14's flag condition while preserving its performance requirements. Republish with
`sase memory init` after accepted edits; never edit generated instruction files
directly. Under automatic approval, default-false memory decisions were not reviewed by
a human: record one `PROPOSED FOLLOW-UP:` per skipped note on this phase for the land
agent to deduplicate/file through `/sase_new_task`, as `/sase_memory_write` requires.
Memory choices do not alter the implementation graph or delay flag removal.

Run the final required check for any modified repositories in sequence; reuse prior
phase evidence for unchanged repositories rather than rerunning expensive checks without
cause. Perform any remaining required visual verification and inspect its report before
finalization. Give the land agent the full 23-row closure/evidence ledger, retained
compatibility rationale, changed-repository list and final core pin. The land agent
verifies predecessor completion and handles final epic closure through the normal host
lifecycle.

## Acceptance criteria

- All 23 inventory flags are absent from the production registry and generated flag
  properties; all 23 dedicated flag dossiers are closed with evidence.
- The enabled behavior in every inventory row works without saved preferences, CLI
  overrides, environment opt-in, or LSP initialization switches.
- Off-only implementations and their tests/docs are deleted, not left behind a hardcoded
  condition. Enabled compatibility aliases and persisted-data readers remain explicitly
  tested.
- Python, the Rust extension and LSP agree on unconditional behavior. The pinned core
  revision contains every required core change.
- Empty-registry startup and stale state/transport handling work, and generic
  feature-flag infrastructure remains usable for the next feature.
- Each phase and its verification completed sequentially, with no helper swarm or
  overlapping build/test workloads; required checks passed or any independently
  witnessed pre-existing failures are reported with precise evidence.
