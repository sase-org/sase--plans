---
tier: epic
title: Complete project onboarding and sase project add
goal: Deliver the convergent add-existing CLI and reusable onboarding engine required
  by sase-1jx.7, then close that phase with verified evidence.
parent_bead: sase-1jx.7
phases:
- id: plan-documents
  title: Observed facts, ownership journals, and onboarding documents
  depends_on: []
  size: medium
  description: 'plan-documents: implement actor detection, read-only facts, atomic
    journals, versioned events, and plan documents over the existing Rust planner;
    define and test the reusable engine interfaces.'
- id: managed-recovery
  title: Commit-and-push setup with recoverable postconditions
  depends_on:
  - plan-documents
  size: medium
  description: 'managed-recovery: implement the management effect and its journal
    evidence; extend Rust onboarding facts and planning only where necessary to preserve
    setup obligations and resume after marker-write, commit, or push failures, with
    binding tests and a pin update.'
- id: convergent-apply
  title: Serialize and execute project onboarding
  depends_on:
  - managed-recovery
  size: medium
  description: 'convergent-apply: implement identity and checkout locking, revalidation,
    ordered add-mode step effects, postcondition checks, truthful result events, and
    failure recovery without duplicate effects.'
- id: add-command
  title: CLI, progress rendering, documentation, and end-to-end verification
  depends_on:
  - convergent-apply
  size: medium
  description: 'add-command: expose sase project add with preview, confirmation, NDJSON,
    durable-operation IO and interactive init chaining; update docs and the generated
    skill source, verify the inherited glossary decision, and run scenario and fake-provider
    CLI tests.'
proposed_by: bbugyi200.athena.sase-1jx.7
create_time: 2026-10-11 19:14:20
status: done
bead_id: sase-1jx.7.1
---

- **PROMPT:** [prompts/202610/project_add_engine.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/project_add_engine.md)
- **PARENT:** [202610/new_project_wizard.md](https://github.com/sase-org/sase--plans/blob/main/202610/new_project_wizard.md)
- **BEAD:** [sase-1jx.7.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1jx/sase-1jx.7.1.md)

# Complete phase sase-1jx.7

This child epic implements the `cli-add` phase of `plan:202610/new_project_wizard.md`.
Read that artifact through `sase artifact read` for the approved contract. The original
phase is reserved and in progress. Its description and history were checked twice during
planning; it has no prior work or remaining-work notes. The missing implementation spans
observation, persistent ownership, Git mutation and recovery, serialized execution, and
a public CLI. Four bounded coding phases keep each worker's scope small enough to finish
directly.

The child's land agent finishes and closes the child epic and then closes **sase-1jx.7
only**, after all descendants and verification are complete. Phase workers close only
their own child phase. No worker or land agent closes **sase-1jx**, another ancestor, or
sibling phases.

## Fixed decisions and scope

The parent epic's reviewer decisions remain final and are inherited:

- `human_only_record = no`: the human-only decision record is declined.
- `glossary_project = yes`: only `glossary:sase-project` is authorized for memory
  changes, including the generated outputs of its publication. Do not ask these
  decisions again. Its current text already names add/create and the wizard, and says
  onboarding leaves `PROJECT_STATE` unset. Preserve this completed part; correct only
  its ambiguous suggestion that `add` itself creates a repository, if still present,
  using the inherited authorization and `/sase_memory_write`.

The standalone validator may warn that the memory publication below lacks a decision in
this child file. Coverage comes from the parent's accepted `glossary_project` decision,
not a new review question. Retain that authorization in the approved child execution
context.

Shared policy belongs in `sase-core`, reached through
`src/sase/core/project_onboarding_facade.py`. Python gathers observations and runs
effects; it must not reproduce identity allocation, setup resolution, blockers, or step
derivation. Open linked repositories using `/sase_repo` and use its printed path. Follow
their `AGENTS.md`. Changes to bindings require the corresponding
`sase-core-revision.txt` pin update and the checkout's supported binding build.

The private-sidecar prerequisite is present: `src/sase/sdd/_sidecar_visibility.py`
exists, and `explicit_sidecar_config_update` accepts `private_primary`. Recheck these
before implementation. Delegate sidecar handling to the existing init flow.

`sase-1jx.8` owns `project create`, strict creation-form validation, remote creation and
adoption, and the refusal guard; this child provides the reusable actor, journal, plan
and executor seams it needs. `sase-1jx.10` owns the wizard frontend. Do not implement
those phases here. Do not deploy generated skills globally.

## Starting points and common interfaces

Already available:

- `project_onboarding/refs.py`: `parse_user_ref`, `UserRefError`.
- `project_onboarding/discovery.py`, `preferences.py`, and suggest rendering.
- `workspace_provider` wrappers `parse_repo_ref`, `peek_ref`, `probe_repo`,
  `resolve_ref`, provider metadata and typed hook results.
- Rust `project_onboarding/{wire,plan,plan_tests}.rs`, its Python binding, and the
  schema-checked Python facade `plan_project_onboarding(request, facts)`.
- Existing locked lifecycle, alias and rename services; current-project MRU services;
  `project_management.enable_sase_management`; durable IO helpers in `ops/cli.py` and
  operation envelopes in `ops/models.py`.

Use the following public interfaces, allowing compatible optional keywords for creation
later. Consumers must pass the original intent, not reconstruct intent from a derived
plan:

- `human_only.detect_actor(...) -> "human" | "agent"`.
- `facts.gather_facts(target, mode, *, actor, ...) -> dict` matching the Rust facts
  wire; `target` is the request's target mapping.
- `planning.build_plan_document(request, *, facts=None) -> dict`; omitted facts are
  gathered. A supplied observation supports local-only TUI previews.
- `journal`: tolerant identity lookup, atomic intent/step updates, and
  `list_incomplete()`. Provide a journal path/lock helper for execution.
- `events.emit_event(event, ...)` and pure `parse_event_line(line)`.
- `apply.apply_plan(request, *, emit, proc_id="", ...) -> result_event`.

Validate untrusted documents and emit literal provider strings in Rich `Text`, stripping
control characters. Keep files below the repository's line limit.

## Phase plan-documents

Implement `facts.py`, `planning.py`, `journal.py`, `events.py`, and the actor portion of
`human_only.py`. This phase owns their initial interfaces and tests.

Actor detection recognizes `SASE_AGENT`, `SASE_AGENT_NAME`, an identity discovered by
`sase.agent.identity.discover_agent_identity`, or `SASE_MONITOR_ID`. The approved
gate-command marker `SASE_GATE_COMMAND=1` or a nonempty `SASE_PROJECT_CREATE_BYPASS`
reason permits human behavior; the bypass emits its named reason to stderr once. Do not
drop agent identity to make setup available.

Gather actual facts without writes: provider probe, matching local record, collision
records in all states, checkout absent/empty/matching/mismatch/not-git, origin identity,
local management config, config dirtiness and default-branch status. Use provider
parsing for origin comparisons. Handle ordinary `.git` directories and worktree gitfiles
rather than treating a gitfile as a conflict. Reuse or expose a genuinely shared
observation helper from discovery rather than cross-importing private functions. An
unavailable probe retains its explanatory message and must not be invented as
found/not-found. Check mode never resolves a ref, creates a directory, writes
preferences, prunes journals, or materializes SDD. All observation subprocesses are
bounded and noninteractive.

Journals live under `$SASE_HOME/project_onboarding/journals/`; their filename is a
readable display stem plus `sha1(identity_key)[:12]`. Store version 1, canonical
identity/ref/display/provider/mode, intent fingerprint, timestamp, running/failed/
complete status, optional proc and actual project key, remote ownership evidence,
per-step status/timestamps/messages, stopped step and update timestamp. Use atomic
writes with private permissions. Preserve failed evidence; prune only complete records
older than 30 days, only on write. Match identity and original fingerprint before using
a journal as ownership evidence. Prepare management substep evidence without assuming a
written marker proves a completed push. Make discovery consume the shared
incomplete-journal reader, preserving its wire. Derive the expected fingerprint with the
Rust planner from the original request before attaching ownership facts, then derive the
final plan with only matching evidence. Do not independently implement the fingerprint
algorithm in Python.

The request matches the existing core wire: mode, actor, provider target including
storage policy and key/name bases, requested name/aliases/setup auto|on|off/current, and
optional creation. Core produces all policy. `build_plan_document` adds:

```text
schema_version=1, kind=project_onboarding_plan, mode, status,
intent_fingerprint, provider{workflow_type,display},
target{ref,display,identity_key,web_url,visibility,default_branch,
       checkout_dir,can_push,sase_managed},
project{key,name,aliases,accent,state=enabled,existing},
setup, steps, warnings, blockers, apply_argv, next_argv
```

`apply_argv` uses canonical ref, every requested option, `--yes --json`, and no
request/result file paths or preview flags. Omitted setup stays auto so the intent
fingerprint remains stable. `next_argv` is `sase init -p KEY` only when init is a
pending handoff; use the actual key after resolution. Accent uses the same
existing-record key set as the Projects list. New records leave `PROJECT_STATE` unset;
re-enabling a disabled record uses the existing lifecycle service.

Events match the parent design: `plan` carries the document; `step` carries id,
running/done/failed, label and UTC start time or elapsed milliseconds and message/ hint;
`result` carries ok|partial|failed|blocked, project key/name, next argv and message. The
parser distinguishes supported events from raw log lines and tolerates malformed/unknown
lines without crashing a future TUI tailer.

Verify scenario documents for fresh GH, matching clone, disabled record, already
registered/current, checkout conflict, no push rights, and agent actor. Test no writes
in preview, truthful unknown facts, identity/fingerprint association, atomic journal
updates, retention and malformed journals/events. Test each actor source and allowed
override. New unconsumed APIs may be temporarily whitelisted against a still-open
consumer phase or this child epic; remove entries as real consumers land.

## Phase managed-recovery

Implement `manage.py` and finish the narrowly required recovery facts in Python and
Rust. This phase owns management substep journal extensions and Rust changes.

There is a concrete gap in the starting planner: `plan.rs::is_managed` treats either the
remote or local marker as completion, and auto setup turns off once managed. If a marker
write succeeds and commit/push fails, retry can skip setup. Also, refreshing facts after
a successful manage step must not discard this operation's outstanding init handoff.

Add defaulted journal facts in the Rust wire for an active matching intent's resolved
setup obligation, incomplete management effect, and proof that config changes are owned
by that effect. Keep their decisions in Rust. Auto setup for a human must retain the
same intent's unfinished setup; an incomplete push remains a pending management step
even if the local marker is true. Agent auto/off and agent explicit-on rules remain
authoritative on retries. Unrelated or stale journals cannot grant ownership or suppress
blockers. A config-dirty blocker may be waived only for byte-for-byte changes proven to
be this journal's marker write; default-branch protections remain. Ordinary plans
without new facts keep their current behavior. Test the additive wire and PyO3 round
trips, and update the Python facade/schema validation and CI pin if necessary.

Persist before each effect enough evidence to recover a response lost after the effect:
baseline and expected config hashes, marker-write stage, intended commit parent/content
and actual commit ID, and push verification. Reobserve Git state to recognize a
completed write/commit/push; do not blindly trust step status. Never create a second
marker commit just because the previous result was lost.

Run `enable_sase_management(checkout/sase/sase.yml)` preserving comments, stage that
path, commit `chore: mark repository as SASE-managed`, and push the expected default
branch to origin. Preserve unrelated worktree and index changes; avoid including
unrelated staged files or pushing unrelated preexisting commits. Refuse unexpected
branch/origin changes and return actionable failure. Use bounded timeouts,
`stdin=DEVNULL`, `GH_PROMPT_DISABLED=1` and `GIT_TERMINAL_PROMPT=0`. Never reset, clean,
force push or delete a remote.

Test with disposable local bare remotes and explicit fixture Git identity:
marker/commit/push success, add or commit failure, hook-driven push rejection,
failure/lost response between substeps, retry after rejection clears, already managed
repos, and unrelated staged edits. Verify exact commit contents and no duplicate commit
after recovery. A rejected push retains partial evidence and names protected-branch/PR
or human `sase init -M` remedies.

## Phase convergent-apply

Implement `apply.py` over the established interfaces; this phase owns serialized
execution and result semantics. Add-mode order is checkout, reenable, identity, manage,
current, init handoff. `create_remote` remains the next sibling phase's extension, not a
side effect of add.

Acquire bounded cooperating locks for canonical repository identity and checkout path,
in deterministic order, covering both direct CLI and durable invocations. The future
TUI's supervisor keys are `project-onboard:{identity_key}` and
`checkout:{checkout_dir}`; supervisor dedupe does not alone serialize a raw CLI racing
it. Re-gather facts and derive the plan under the locks before any effect. A changed or
blocked plan must be reported before touching the checkout; do not execute stale review
rows.

Write active intent before effects. Execute pending non-handoff steps in order, emitting
running then done/failed with timings, updating journal evidence and checking the actual
postcondition. Reobserve between effects where a preceding effect changes facts
(especially checkout and manage), using the same requested intent/fingerprint and active
setup obligation. Stop at the first failed step.

- Checkout: recheck origin and filesystem state immediately before
  `workspace_provider.resolve_ref(parsed.ref, workflow_type)`. Remove only an observed
  still-empty checkout directory if the provider needs it absent. Preserve matching
  dirty clones. The provider allocates key/name; subsequent updates and result use its
  actual returned record and checkout.
- Reenable: `set_project_state_locked(key, "enabled")`; never create a duplicate record
  for an existing disabled project.
- Identity: existing locked alias services and a public locked rename adapter over the
  existing core `apply_project_name_update`, preserving collision checks and display/tag
  cache invalidations. Omitted options preserve existing names and aliases. Do not use
  private cross-file imports or hand-edit specs.
- Manage: the management helper and ownership-aware Rust plan.
- Current: `set_current_project(actual_key)`, accepting both set and unchanged; failure
  stays visible. Repeating an already-current request causes no MRU write.
- Init: a handoff only, never run from the executor; populate the exact next argv.

End with exactly one result event. Distinguish blocked before effects, failed before
useful registration, partial after registration or other completed effects, and ok after
all non-handoff postconditions. Mark journals complete only when those postconditions
are satisfied, not when init/sidecar work has run. A failure keeps the project enabled
and preserves enough evidence for the same argv to converge. Init cancellation does not
undo registration.

Test injected failure before/after each add step, retry and actual postconditions,
allocation changed at apply, dirty matching clone, blocked revalidation, and
simultaneous submissions in separate processes sharing a temporary SASE_HOME. Prove one
checkout/record/marker commit, deterministic contention behavior, no duplicate alias/MRU
write, correct event order and truthful partial results.

## Phase add-command

Register `add` in alphabetic command order in `main/parser_project.py` and both the
metavar and handler usage string. Add a focused handler module if needed; keep suggest
intact. Public flags, sorted by long name, are:

```text
sase project add REF
  -a/--alias ALIAS (repeatable)  -c/--check  -j/--json
  -m/--make-current  -n/--name NAME  -S/--no-setup
  -Q/--request-path PATH  -R/--result-path PATH
  -s/--setup  -y/--yes
```

Setup flags are mutually exclusive; default is auto. References go through
`parse_user_ref` (provider prefixes, owner/repo, URLs/SSH and supported paths). Help
explains remote setup writes, actor restrictions, enabled defaults, resumption and init
handoff, with preview/apply examples. Bare project still defaults to list. Parse bad
input as usage exit 2.

Check is read-only: one document with JSON, otherwise a colored plan using init glyphs
`+ ~ − ●`, provider/target header, project and remote summary, step rows,
warnings/blockers and exact would-run argv. It exits 0 for ready/current and 1 for
blocked. Apply without yes on a TTY renders and asks `Apply? [y/N]`; declining writes
nothing. Without a TTY, render and refuse with `re-run with --yes`, exit 1. Yes applies
directly. JSON apply writes NDJSON exclusively on stdout; send human and
provider/subprocess diagnostics to stderr or capture them into step messages. Human
apply renders a live checklist/timings and an accented Ready summary.

Register `PROJECT_ADD = "project.add"` in `ops/names.py`; use `add_operation_io_flags`,
`load_request` and `emit_operation_result` with the existing versioned private IO
contract. Validate sidecar operation and original intent; a request must not change the
target reviewed in argv. The terminal result's payload is exactly the result event, for
success and all failure paths. `ace/tui/durable_submit.py::coerce_operation_request` is
already generic; prove the new operation works without adding a second envelope or an
unused registry.

After successful setup on a TTY without JSON, offer default-yes interactive
`sase init -p ACTUAL_KEY` through the existing init coordinator with terminal IO. Keep
sidecar confirmation there; yes for onboarding does not authorize remote sidecars.
Otherwise convey `next_argv` in the result and human output. Init failure/cancellation
exits 1 with a partial outcome but preserves registration; retry init directly.
Usage/human-only refusal is exit 2; applied/current is 0; blocked/failed/partial is 1.
Actor explicit setup must not launch a management subprocess; exercise the core's
blocker and the documented refusal code.

Update `docs/cli.md`, `docs/project_spec.md`, `docs/workspace.md`,
`docs/getting_started.md`, and `docs/init.md`. Explain registration vs setup, identity
allocation, enabled defaults, agent preview/add behavior and recovery. Update only the
source `src/sase/macros/skills/sase_project.md`: preview with check first, agents omit
setup, and use `/sase_gate` for human setup when needed. Preview generation with
`sase skill init --diff` or `--dry-run`; global deployment is for a clean landed
revision. Re-read `glossary:sase-project` through audited memory reads; apply only the
inherited correction described above if needed and publish with `sase memory init`.

Verify parser/help, one-document previews for the scenario matrix, TTY yes/no, non-TTY
refusal, stdout/stderr separation, NDJSON final event, durable result files including
errors, init offer/decline/failure, actor gates, and all required short aliases. Add
fake-gh-on-PATH end-to-end tests with temp SASE_HOME and disposable repos; real
providers must perform no remote writes. Run read-only
`sase project add gh:sase-org/sase --check` against real gh and record its actual
outcome; an unavailable/auth-limited provider is evidence, not permission to create live
test repositories.

## Verification, symbol ownership, and completion

Every coding phase reads `lint_and_test.md`, runs `just fmt` then `sase tool run check`
in each modified repo, and runs its targeted meaningful tests. Do not run check-full
without explicit instruction. Use `/sase_monitor` for commands that outlast a turn; wait
for the handoff command itself to exit. Observe scoped checks' real results. Reproduce
uncertain failures on the clean base before labeling them preexisting. Record identical
clean-base failures as `PROPOSED FOLLOW-UP:` notes with run IDs and existing tracking
beads, then close the completed child phase. Do not create task beads directly.

At planning time `sase bead epic-symbols sase-1jx.7` reports:

```text
plan_project_onboarding
parse_user_ref
UserRefError
```

Real consumers in planning and the CLI should consume all three; remove their
`sase-1jx.7(...)` lines when consumed. Before closing each child phase, run
`sase bead epic-symbols CHILD_ID`; remove resolved entries or re-key remaining ones to a
still-open consumer/child epic/original phase. Future TUI/create APIs with no consumer
yet may be keyed to still-open `sase-1jx.8` or `sase-1jx.10` as appropriate. Tests alone
do not satisfy Symvision. Never key leftovers to a bead being closed, and do not drop
needed public APIs merely to satisfy lint.

The child land agent verifies all four phases, the cross-repo pin/binding, fake provider
smoke, docs/skill, results and required checks; reviews discovered follow-up notes for
the parent epic's land agent; and confirms there are no new failures. It closes its
child epic only after every descendant is closed. It then runs
`sase bead epic-symbols sase-1jx.7`, resolves or re-keys every leftover to a still-open
bead, and closes **only sase-1jx.7** with a note naming behavior, test/check evidence
and any proven clean-base failures. Never hand-set status, and never close sase-1jx or
other ancestors. Submit `/sase_final` as required by the host, without creating commits,
branches, or PRs manually.
