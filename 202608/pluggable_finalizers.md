---
tier: epic
title: Host-owned pluggable finalizer protocol
goal:
  SASE agents finish through a beta-gated, host-owned finalizer protocol in which
  trusted configuration activates named built-in or plugin providers, `%final` selects
  those configured instances per launch, `/sase_final` submits one atomic turn-bound
  declaration, and the built-in `commit` instance preserves every current attribution,
  publication, discarded-work, and merge-conflict guarantee while requiring a reason
  whenever an agent refuses to commit attributable repository changes.
phases:
  - id: core-protocol
    title: Rust finalizer protocol and resolution contract
    depends_on: []
    size: medium
    description:
      "core-protocol: add versioned provider, instance, selector, plan, context,
      submission, result, and diagnostic wires to sase-core; own stable
      selection/dependency ordering, canonical digests, envelope coverage checks, and
      aggregate outcomes there; expose focused PyO3 bindings and add `%final` to the
      shared editor directive contract; then publish the core release."
  - id: core-adopt
    title: Adopt the finalizer protocol core release
    depends_on:
      - core-protocol
    size: small
    description:
      "core-adopt: raise the `sase-core-rs` dependency floor, refresh the lockfile, add
      the Python typed adapter and minimum-version smoke coverage, and keep all
      host-side work behind compatibility facades until the released binding is
      available."
  - id: host-foundation
    title: Feature flag, repository baselines, registry, and launch selection
    depends_on:
      - core-adopt
    size: medium
    description:
      "host-foundation: create the beta flag through `sase flag new`, capture
      late-opened repository baselines atomically, add keyed configuration and
      `sase_finalizers` discovery with source provenance, parse ordered `%final`
      operations, reject invalid selected plans at launch, and persist the resolved plan
      without changing the flag-off commit finalizer path."
  - id: declaration-channel
    title: Turn-bound sase final declaration channel and skill
    depends_on:
      - host-foundation
    size: medium
    description:
      "declaration-channel: add `sase final context` and atomic `submit`, opaque
      host-issued repository obligations, nonce and digest validation, retained
      invalid-attempt diagnostics, the generated `/sase_final` source, and beta-only
      end-of-turn prompt instructions with demand-driven one-turn recovery and
      mechanical intentional-handoff exemption."
  - id: extension-runtime
    title: Isolated plugin and configuration finalizer execution
    depends_on:
      - host-foundation
    size: medium
    description:
      "extension-runtime: implement the `sase_finalizers` subprocess protocol with
      sanitized environments and bounded JSON I/O, add constrained `builtin@command`,
      surface activation and provenance through `sase final list`, `show`, and `doctor`,
      and prove with a non-mutating reference plugin that installation alone never
      activates behavior."
  - id: commit-reconciliation
    title: Generic controller and built-in commit parity
    depends_on:
      - declaration-channel
      - extension-runtime
    size: medium
    description:
      "commit-reconciliation: replace the flag-on hard-coded seam with bounded
      plan/declare/execute/verify reconciliation, require exactly one commit-or-refuse
      decision per attributable dirty repository, execute sequential stitches through
      `sase stitch create`, preserve the existing auto-commit, publication, evidence,
      discard, and family rules, and retain the dedicated one-turn `--resume`
      conflict-repair state machine."
  - id: compatibility-soak
    title: Compatibility migration, observability, documentation, and soak gates
    depends_on:
      - commit-reconciliation
    size: medium
    description:
      "compatibility-soak: map the legacy commit-finalizer settings with explicit
      diagnostics, retain compatibility artifacts and reporting, add full
      flag-off/flag-on and adversarial end-to-end coverage, document the
      config/plugin/directive/CLI contracts, preview generated skill deployment safely,
      and define measurable beta-removal criteria while leaving flag deletion to its
      dedicated flag bead."
proposed_by: bbugyi200.athena.08y
bead_id: sase-rn
create_time: 2026-09-09 19:51:06
status: wip
---

- **PROMPT:**
  [prompts/202608/pluggable_finalizers.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/pluggable_finalizers.md)
- **BEAD:**
  [sase-rn](https://github.com/sase-org/sase--beads/blob/main/pages/sase-rn/README.md)

# Plan: Host-owned pluggable finalizer protocol

This plan is derived from
`research:202608/finalizer_protocol_and_extensibility/finalizer_protocol_and_extensibility.md`
and a current-code audit of the invocation seam, commit finalizer, baseline capture,
commit checkpoint/result ledgers, prompt directives, generated skill sources, plugin
inventory, and configuration merge path.

## 1. Problem and desired outcome

`src/sase/llm_provider/_invoke.py` currently calls `run_commit_finalizer()` directly
after every successful provider invocation. That controller is much more than a stop
hook: it identifies the primary and opened repositories, protects pre-run dirt,
auto-commits a few narrowly proven machine-owned changes, prompts for bounded repair,
checks for discarded work, verifies publication, and recognizes commits after rebases
through `commit_results.json` tree evidence. Generalizing it by merely replacing the
function call with an arbitrary command hook would discard the safety properties that
make completion trustworthy.

The target is therefore a host-owned reconciliation protocol:

1. Before the model turn, SASE resolves trusted configuration plus ordered `%final`
   selector operations into an immutable finalizer plan.
2. During a normally completing turn, `/sase_final` asks the host for current typed
   obligations and atomically declares intent through `sase final submit`.
3. After the provider returns, SASE—not the model and not a plugin—drives selected
   executors in deterministic order with bounded retries.
4. SASE independently recomputes triggers, repository state, evidence, and
   postconditions until it reaches a fixed point or fails closed.

The built-in `commit` finalizer remains the bundled default. Omitting `%final` has the
resolved effect of selecting `commit`; SASE must not inject a literal `%final:commit`
into every prompt. With the beta enabled, users can select configured alternatives or
clear the default explicitly. With the beta disabled, current behavior remains the exact
old `run_commit_finalizer()` branch.

## 2. Binding design decisions

### 2.1 Authority and trust boundary

- The host owns lifecycle, selection, dependency closure, ordering, retry budgets, state
  transitions, status aggregation, evidence capture, and failure. A provider cannot
  invoke the LLM, choose its own cwd, mark itself skipped, or report the whole agent
  complete.
- Plugins advertise providers; they do not activate them. A provider is active only when
  a user, machine overlay, or project configuration declares a named instance and that
  instance is selected by defaults or `%final`.
- Plugin-contributed config layers may not add finalizer defaults, required instances,
  or implicitly selectable instances. Installation alone must be inert. The package's
  `sase_finalizers` entry point only makes provider IDs discoverable.
- Every non-builtin `use: distribution@provider` remains subject to `plugins.required`.
  A selected missing, mismatched, disabled, or malformed provider is a launch-time
  error; an unselected broken provider is visible in `doctor` without consuming the main
  model turn.
- Prompt text is selection-only. `%final` cannot provide argv, code, environment
  variables, credentials, timeouts, retry counts, repository paths, or arbitrary
  provider options.

### 2.2 Shared Rust boundary

The frontend-neutral parts belong in a new `sase-core` finalizer module:

- versioned provider/instance/selector/plan/context/submission/result wires;
- ordered selector replay and required-instance policy;
- dependency validation, cycle diagnostics, stable topological ordering, and spec/plan
  digests;
- strict envelope identity and coverage validation;
- normalized per-instance and aggregate outcomes.

Python remains responsible for configuration-layer provenance, entry-point discovery,
repository inspection, locks and artifact files, subprocess execution, LLM recovery
turns, and invoking the VCS workflow. This keeps one implementation of shared domain
rules without moving filesystem- or provider-specific orchestration into Rust.

The provider-specific payload remains JSON in the shared envelope. Core validates its
identity, size, selected-instance coverage, and digest binding. The selected provider's
isolated `validate` operation validates its own payload schema before SASE accepts the
atomic submission; built-ins use in-process host validators. This avoids pretending an
executor's success result is proof while still permitting typed plugin-defined data.

### 2.3 Configuration model

The bundled configuration starts as:

```yaml
finalizers:
  defaults: [commit]
  required: []
  instances:
    commit:
      use: builtin@commit
      after: []
      max_attempts: 2
      refusal: fail
```

`required` is intentionally empty in the bundled default. `commit` is selected by
default, preserving current behavior, but `%final:none` remains a real per-launch
override. A project that must forbid opt-out can set `required: [commit]`.

Instances are keyed lowercase slugs and merge by key across trusted configuration layers
while retaining field-level/source-layer provenance. Lists such as `defaults`,
`required`, and `after` replace rather than concatenate at the layer that owns them.
Unknown keys fail schema validation rather than being ignored. Provider-specific
configuration lives under a dedicated `config:` mapping so common policy fields cannot
be shadowed accidentally.

A configuration-only check uses the constrained built-in provider:

```yaml
finalizers:
  defaults: [local-check, commit]
  instances:
    local-check:
      use: builtin@command
      after: []
      config:
        command: [just, check]
        cwd: primary
        timeout: 10m
        submission: none
    commit:
      use: builtin@commit
      after: [local-check]
      max_attempts: 2
      refusal: fail
```

Commands are argv arrays from trusted config, never shell strings. `cwd` is a closed
host-resolved policy (`primary` initially), not a path. Environment values are not
interpolated from a declaration. Reusable or domain-aware behavior belongs in a
versioned plugin provider.

### 2.4 `%final` semantics

`%final` is a repeatable, comma-aware sequence of operations applied left to right to
configuration-derived defaults:

| Form                      | Resolved operation                                                   |
| ------------------------- | -------------------------------------------------------------------- |
| omitted                   | Use configured defaults; bundled result is `[commit]`.               |
| `%final:commit`           | Add or retain `commit` without duplicating it.                       |
| `%final:lint`             | Add configured instance `lint` after retained defaults.              |
| `%final:!commit`          | Remove `commit` unless it is required.                               |
| `%final:none`             | Clear the current selection unless that removes a required instance. |
| `%final:none %final:lint` | Select exactly `lint`.                                               |
| `%final:lint,commit`      | Replay `lint`, then `commit`, before dependency ordering.            |

Reject bare `%final`, empty comma elements, unknown instances, duplicates that encode a
contradictory remove/add operation in one token, forbidden removal, missing
dependencies, and dependency cycles before the main provider turn. Persist both the raw
operations and resolved order. While `pluggable_finalizers` is off, recognize an
explicit `%final` only to emit a targeted opt-in error; never silently strip it or run a
different finalizer than the prompt requested.

The directive parser, stripped-prompt scanner, fan-out/relaunch preservation,
completion/hover contract, `PromptDirectives`, and `agent_meta.json` projection must all
agree on this syntax. Repeated `%final` is deliberately multi-valued; it is not a single
directive whose last occurrence wins.

### 2.5 Agent artifact protocol

All state is scoped to the current `SASE_ARTIFACTS_DIR`; `sase var` is not used. Files
are schema-versioned, size-bounded, written under an advisory lock, fsynced, and
published with same-directory `os.replace`:

| Artifact                          | Purpose                                                                                                                                       |
| --------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `finalizer_plan.json`             | Raw selectors, resolved order, instance/provider provenance, redacted config metadata, and digests fixed before the model turn.               |
| `finalizer_baseline.json`         | Per-repository start fingerprints plus atomic late-open additions; the legacy baseline file is written/read compatibly during rollout.        |
| `final_context.json`              | Latest host-issued context for the current turn nonce, selected instances, trigger states, opaque repository obligations, and context digest. |
| `final_submission.json`           | The one currently valid atomic envelope; intent only, never proof that an action occurred.                                                    |
| `final_submission_attempts.jsonl` | Bounded diagnostic records for rejected and replaced submissions, including error codes and content digests; sensitive values are redacted.   |
| `finalizer_result.json`           | Aggregate and per-instance outcomes, attempts, refusal reasons, evidence references, timings, and terminal error.                             |

Per-attempt stdout/stderr and structured executor results live below a bounded
`finalizers/<instance>/` directory. The existing `commit_finalizer_result.json`, pass
prompt/response artifacts, `commit_state.json`, and `commit_results.json` remain
available during compatibility migration.

The envelope binds `schema_version`, agent/run identity, turn nonce, plan digest,
context digest, and exactly one payload for every selected instance whose current
trigger requires submission. Unknown/duplicate/extra/missing instance payloads, unknown
repository IDs, stale nonces or digests, unknown fields, oversized input, and provider
validation failures are rejected without replacing the last valid submission.

### 2.6 `/sase_final` end-of-turn behavior

The generated skill instructs an agent to make `/sase_final` its last action on every
normally completing SASE turn:

1. run `sase final context -f json`;
2. inspect selected instances and host-issued obligations;
3. build one complete manifest;
4. run `sase final submit <manifest-file|->`;
5. repair validation errors before returning when possible.

The CLI records declarations only. It never executes a plugin, Git, a command finalizer,
or a commit.

Instruction is universal, but enforcement is demand-driven. A clean commit-only turn has
no declaration-required payload and does not spend a recovery model call if the agent
omitted the skill. An always-run audit provider with a required payload does. When
required input is missing or stale after a normal return, the host issues exactly one
declaration-recovery turn with a fresh nonce; omission or invalid input on that turn
fails closed.

Intentional handoffs are exempt mechanically, not by a growing skill-name allowlist.
`/sase_plan`, `/sase_monitor`, `/sase_pipe`, `/sase_questions`, and any future command
using the same intentional-handoff signal terminate the runner before the normal
post-invocation completion seam. A recovery turn is not launched for a runner that has
already handed off.

### 2.7 Repository obligations and refusal

The host inventories repositories and assigns opaque stable IDs. Context may show a
display name, kind, and attributable relative paths so the agent can reason about its
work; declarations identify repositories only by host-issued ID. A model never submits
an absolute path or expands the inventory.

For every repository with agent-attributable dirty paths, the `commit` payload requires
exactly one decision:

```json
{
  "repositories": [
    {
      "repo_id": "repo-7f4c2a",
      "action": "commit",
      "message": "feat(finalizers): add the declaration protocol"
    },
    {
      "repo_id": "repo-a91d30",
      "action": "refuse",
      "reason": "These generated changes are unrelated to the requested work."
    }
  ]
}
```

`commit` requires one valid conventional message and requests at most one final stitch
for that repository in version 1. `refuse` requires a nonblank, length-bounded reason.
Refusal is diagnostic evidence, not authorization to discard or silently release a dirty
workspace: the version-1 `refusal: fail` policy records and surfaces the reason, then
fails completion while attributable dirt remains. Other refusal policies and
multi-commit file partitioning are explicitly deferred.

Pre-existing dirty paths remain protected. The executor passes them as exclusions to the
existing stitch workflow and fails if it cannot prove that remaining attributable paths
were covered. An agent edit to a path already dirty at baseline remains
ambiguous/protected rather than being swept into a commit.

### 2.8 Commit execution and conflicts

The generic controller processes mutating decisions sequentially in deterministic
repository order. It may run non-mutating providers according to the resolved graph, but
it never executes repository commits concurrently.

For each `commit` decision:

1. Recompute the obligation and verify the context/message digest immediately before
   mutation.
2. Write the declared message to a repo-local ignored `.sase/` file.
3. invoke `sase stitch create -M <file>` with protected baseline paths passed as
   exclusions and with the repository selected only through the host's opaque-ID map;
4. verify the run-owned `commit_results.json` entry, SHA/tree evidence, cleanliness
   relative to protected dirt, publication, and Patch/Stitch bookkeeping;
5. recompute all triggers before advancing.

The controller preserves the current narrow bead-state, prompt-Q&A, and completed-plan
status auto-commits, plus their publication checks. It continues to use the existing
`commit_results.json` ledger so a rebase-altered SHA is recognized by committed tree
identity. It preserves the shared-clone race classification and discarded-work guard.

Exit code 2 is a separate state machine, not a declaration failure:

1. Stop the queue immediately; do not start the next repository.
2. Preserve `commit_state.json`, the paused VCS operation, sealed decision, and all
   artifacts.
3. Give the agent one dedicated conflict-repair turn with the existing recipe: inspect
   unmerged files, resolve every marker, stage, continue the rebase/merge, and complete
   the wrapper's resume path.
4. Resume/verify the same operation through `sase stitch create --resume`; never start a
   second stitch, stash, skip, auto-abort, or guess a resolution.
5. Continue only when the checkpoint, commit ledger, publication, and worktree all
   agree. A second unresolved conflict or contradictory/stale checkpoint fails with
   state intact for diagnosis.

The one declaration-recovery turn and one conflict-repair turn have separate budgets. A
real merge conflict cannot consume the agent's compliance retry.

### 2.9 Fixed point and ordering

Finalizers execute in stable topological order: selector/default order breaks ties and
`after` adds dependency edges. After each mutating executor, the host recomputes every
trigger and postcondition. If a later provider creates new attributable dirt, `commit`
becomes pending again; it does not report stale success. The controller asks for a new
turn-bound declaration only if newly pending work actually needs model input.

The run stops successfully only when every selected instance is terminal-success and all
host postconditions hold. It fails on a provider attempt cap, controller cycle cap,
no-progress fingerprint, refusal, invalid/stale state, timeout, malformed output,
discarded work, unpublished required state, or unresolved dirt. Executors cannot
increase these budgets.

## 3. CLI and plugin surface

The public command group is:

```text
sase final [list]
sase final context [-f|--format pretty|json]
sase final doctor [-f|--format pretty|json]
sase final show <instance> [-f|--format pretty|json]
sase final submit <manifest-file|->
```

Bare `sase final` delegates to `list` through the central default-list mechanism.
Required values are positional and every public long option has a short alias. `list`,
`show`, and `doctor` work outside an agent; `context` and `submit` require a
beta-enabled SASE agent environment and actionable current run metadata.

Pretty output should make the contract pleasant to inspect: instance name and glyph,
selected/default/required badges, provider reference, source layer, dependencies,
submission requirement, timeout/attempt policy, plugin distribution/version, and any
diagnostic. JSON output is stable and includes the same provenance.

External providers use the `sase_finalizers` entry-point group. Discovery reads entry
point metadata without importing third-party code. A selected provider runs through a
dedicated worker subprocess with versioned `describe`, `validate`, `execute`, and
`verify` operations over bounded JSON stdin/stdout. The host supplies immutable context
and a sanitized environment, uses argv without a shell, caps runtime/output, captures
stderr, and rejects unknown fields/schema versions. Provider config may explicitly allow
additional environment variable names; values are never copied into persisted
plan/context artifacts.

`builtin@command` uses the same result envelope but has no model submission in version

1. Its fixed argv, cwd policy, timeout, and environment allowlist come only from trusted
   config. Exit zero is necessary but not sufficient: the host still recomputes
   finalizer triggers, repository state, and aggregate postconditions afterward.

## 4. Compatibility and rollout

Create `pluggable_finalizers` only through `sase flag new`, as a beta defaulting off.
The generated removal bead—not this epic—owns eventual deletion of the old branch. Both
states receive tests:

- **Off:** call the existing `run_commit_finalizer()` at the same seam with byte- and
  behavior-compatible artifacts, prompts, retry counts, errors, and reporting.
- **On:** resolve and run the generic protocol; default configuration selects only the
  built-in commit instance.

Legacy configuration migrates without ambiguity:

- When no new `finalizers` block is present, `commit.finalizer.enabled: true|false` maps
  to defaults `[commit]|[]`, and `commit.finalizer.max_passes` maps to the commit
  instance's `max_attempts`.
- When both old and new settings are present, new settings win and `sase final doctor`
  reports the ignored legacy path and source layer.
- `SASE_DISABLE_COMMIT_STOP_HOOK` remains a compatibility escape hatch during the beta,
  with a deprecation diagnostic. It disables generic finalization only when the new
  configuration did not explicitly choose another policy.

The generic aggregate result is additive. Existing runner reporting continues to read
`commit_finalizer_result.json`; the new controller writes that compatibility projection
for the `commit` instance while new surfaces move to `finalizer_result.json`.

The beta-removal bead may be proposed for closure only after all of these are true:

- the default `commit` instance has soaked on real SASE agents with no unexplained
  parity regressions;
- conflict resume, linked/external/sidecar commits, family continuation, publication,
  and pre-existing dirt telemetry are healthy;
- at least one configuration-only command finalizer and one external reference plugin
  have completed end to end;
- legacy-setting diagnostics have shipped for a documented migration window;
- no live consumer depends exclusively on old commit-finalizer artifacts.

## 5. Explicitly out of scope

- Prompt-supplied finalizer definitions, commands, cwd paths, environment values,
  credentials, timeouts, retries, or arbitrary per-run kwargs.
- More than one requested final stitch per dirty repository, file-to-commit
  partitioning, or parallel repository commits.
- A successful dirty-work refusal policy. Version 1 always preserves the reason and
  fails while attributable work remains.
- Running arbitrary shell strings or interpolating declaration fields into argv.
- Giving plugin executors direct control of LLM recovery turns or terminal agent state.
- Removing `pluggable_finalizers` or closing its flag bead in this epic.
- Editing SASE memory notes or deploying generated skills from a dirty/unmerged tree.

## 6. Rust finalizer protocol and resolution contract

Work in the checkout printed by
`sase repo open sase-core -r "Implement the shared finalizer protocol"`; do not use a
hard-coded path. Add `crates/sase_core/src/finalizer/` with focused `wire`, `selection`,
`digest`, `submission`, and `outcome` modules and export them through
`crates/sase_core/src/lib.rs`.

Define version-1 wires for:

- provider capabilities and provider-specific schema metadata/digest;
- configured instances, `after` edges, common policies, and provider provenance IDs;
- ordered selector operations and resolved plan entries;
- context identity, trigger/submission requirements, opaque obligations, and digests;
- atomic submission envelopes and per-instance payload records;
- executor/verifier attempts, refusal evidence, per-instance status, aggregate status,
  and structured diagnostics.

Core must validate lowercase instance slugs, unique IDs, supported schema versions,
closed status enums, bounded text/list sizes, missing dependencies, cycles, required
instance removal, selector replay, complete submission coverage, exact run/turn/plan/
context identity, duplicate or unexpected payloads, and canonical SHA-256 digests. Use
deterministic canonical JSON and existing digest conventions rather than Python object
ordering. Stable topological ordering preserves configured/selector order among ready
nodes.

Expose small PyO3 functions rather than one oversized binding: validate/digest a
provider or instance spec, resolve selectors, validate/digest context/submission, and
aggregate outcomes. Add Python-visible schema version constants and update
`tools/check_sase_core_rs_bindings` in the later adoption phase.

Also add `final` metadata to the Rust editor directive contract so completion, hover,
diagnostics, and shared launch parsing know it is repeatable, colon/parenthesized,
comma-aware, and selection-only. Do not teach Rust to inspect Python configuration; the
host supplies instance-name completion candidates.

Tests must cover deterministic digests, add/remove/none selector replay, required
policy, stable ordering, cycles, unknown dependencies, all stale identity fields,
coverage errors, unknown fields/statuses, size limits, aggregate failure precedence, and
editor completion/hover for `%final`.

Run `cargo fmt --all -- --check`,
`cargo clippy --workspace --all-targets -- -D warnings`, and `cargo test --workspace`.
Land through the core repo's normal release flow and record the actual published version
for `core-adopt`; do not hand-edit crate versions.

## 7. Adopt the finalizer protocol core release

After the core release is published:

1. Raise the `sase-core-rs` floor in `pyproject.toml` to the actual release while
   preserving the repository's version-window policy, refresh `uv.lock`, and run
   `just install`.
2. Add `src/sase/core/finalizer_wire.py` as the narrow typed adapter. It checks the
   binding schema version before translating dicts into frozen Python records; callers
   must not manipulate raw binding dictionaries.
3. Extend the published-floor binding smoke test to call every new finalizer binding,
   not merely check that names exist.
4. Add adapter parity tests using shared JSON fixtures generated by core, including
   malformed/future versions and digest mismatches.

This phase changes no agent behavior and introduces no finalizer config or directive
handling. Run `just check`; use `just check-full` through `/sase_monitor` if dependency
adoption broadens selection.

## 8. Feature flag, repository baselines, registry, and launch selection

### 8.1 Flag and compatibility scaffold

Create the beta with `sase flag new pluggable_finalizers` and authored enabled,
disabled, and removal-gate prose. The enabled branch routes normal completion through
the new controller; the disabled branch is the untouched old commit reconciler. The
command's dedicated flag bead owns later removal. Add both-state tests at the invocation
seam before any behavior moves.

Introduce `src/sase/finalizers/` as the generic host package. Keep thin compatibility
imports in `sase.llm_provider.commit_finalizer*` until migration finishes so current
tests and consumers do not all move at once.

### 8.2 Complete baseline inventory

Generalize `commit_finalizer_baseline.json` into a versioned per-repository baseline
store while retaining a compatibility reader/writer. Continue capturing the primary and
already-known repositories in `run_agent_runner_bootstrap.py`, including the current
family-attach inheritance rule.

Fix the late-open race in the shared repository-open marker path: when an agent opens a
linked, sidecar, different-project, or external repo, fingerprint that checkout and
atomically add it to the current run's baseline under the artifact lock before the
opened-repo marker becomes visible and before `sase repo open` prints/returns the path.
The first capture for a repo ID wins; repeat opens verify identity without rebasing the
baseline. Failure to capture an agent-opened repository baseline is a visible
finalization safety diagnostic, not a silent assumption that existing dirt belongs to
the agent.

Tests must simulate a repo that is dirty before `sase repo open`, agent edits after
open, repeat opens, two open processes, family inheritance, legacy marker reads, and an
open/capture failure.

### 8.3 Registry and plan resolution

Add the `finalizers` schema and bundled `commit` instance to
`src/sase/default_config.yml` and `src/sase/config/sase.schema.json`. Build a
provenance-preserving loader by replaying config layers rather than reading only the
fully merged dict. Reject plugin-config activation and retain diagnostics for invalid
unselected instances.

Add `sase_finalizers` to plugin inventory, provider-disable handling, version output,
completion, required-plugin diagnostics, and doctor inventory. Discovery returns
metadata without loading third-party code.

Extend Python prompt extraction with repeatable raw `final` operations; preserve them
through fan-out, repeat, relaunch, family attachment, prompt stashing, and editor
rewrites. Resolve the selected plan before the first provider call with the core
binding. Store raw operations and resolved/redacted plan in `finalizer_plan.json` and
project the digest/order/provenance into `agent_meta.json`. An explicit `%final` while
the beta is off raises the targeted flag-required diagnostic.

Tests cover config-layer precedence/provenance, plugin installation inertness, required
plugin enforcement, default commit selection, every selector form, required removal,
unknown/cyclic dependencies, prompt stripping/literal zones, fan-out inheritance, and
launch failure before provider invocation.

## 9. Turn-bound sase final declaration channel and skill

Add `src/sase/main/parser_final.py`, handler modules, and the central parser/entry
registrations for `context` and `submit`. Use the normal CLI sorting/default-list
conventions.

At the start of every provider turn, mint a cryptographically random turn nonce and make
only that nonce plus run identity available to subprocesses. `sase final context` loads
the sealed plan, recomputes current host triggers and repository attribution, issues
opaque repo IDs, validates through core, and atomically publishes the latest context.
Re-reading context in the same turn may refresh its digest; an older submission then
becomes stale by design.

`sase final submit` reads exactly one document from the positional path or stdin,
enforces an input-size cap, performs core envelope validation plus built-in/isolated
provider payload validation, appends a bounded diagnostic attempt record, and atomically
replaces `final_submission.json` only on success. It never mutates a repo or executes a
selected finalizer.

Create `src/sase/xprompts/skills/sase_final.md` as the canonical generated skill source
and update skill rendering/source-content tests for every supported runtime. Add the
beta-enabled end-of-turn rule at launch-time, not to unconditional global generated
instructions, so a project-local flag decision is respected. The prompt explains
commit/refuse coverage, conventional messages, refusal reasons, validation repair, and
that the declaration is the last normal action.

Add one bounded missing/stale-declaration recovery prompt using the same provider,
model, and resolved effort as the original turn. The recovery gets a new nonce and
context; it cannot reuse the initial envelope. Demand-driven tests prove clean
commit-only runs do not spend a recovery call. Signal/handoff tests prove intentional
plan, monitor, pipe, and question termination never enters recovery.

Run skill previews only with `sase skill init --diff` or `--dry-run` in the phase
workspace. Do not deploy shared generated skill files until the source change is
committed and landed canonically.

## 10. Isolated plugin and configuration finalizer execution

Define a small public provider SDK and `sase_finalizers` entry-point contract. The
entry-point target identifies a provider worker; the host invokes it only when a
configured selected instance needs `describe`, `validate`, `execute`, or `verify`. Each
operation receives a versioned immutable JSON request and must return one bounded JSON
result. The host checks provider ID, distribution/version provenance, schema
version/digest, operation, instance, plan/context identity, and allowed status before
using it.

Run workers without a shell using a sanitized base environment, explicit trusted
allowlisted variable names, a host-resolved cwd policy, timeout, process-group cleanup,
and stdout/stderr caps. Never persist resolved secret values. Treat import failures,
timeouts, signals, excess output, malformed JSON, unknown fields, schema drift, or a
failed verify operation as structured finalizer failures.

Implement `builtin@command` on the same executor-result contract. Version 1 accepts no
model payload, uses fixed argv, and runs once whenever selected. A zero exit result is
recorded, then the controller independently rechecks repo and aggregate state.

Finish the read-only CLI:

- `list` shows effective instances, selection/default/required state, dependencies,
  provider provenance, source layer, and health;
- `show` explains the fully merged/redacted instance and provider contract;
- `doctor` diagnoses config, required plugins, missing/mismatched entry points,
  dependency cycles, unsafe command shapes, stale schema versions, and legacy config;
- all three support matching pretty and JSON fields.

Build an installable test fixture plugin exposing a non-mutating reference/audit
finalizer. Prove discovery without activation, explicit project/user activation,
`%final` selection, payload validation, successful verification, bad output, timeout,
disabled plugin environment, distribution-prefix mismatch, and no plugin-controlled
LLM/status mutation.

## 11. Generic controller and built-in commit parity

Add `run_finalizers()` at the existing post-`provider.invoke()` seam and call it only
when `pluggable_finalizers` is enabled. The controller loads the sealed plan and drives
plan/context/declaration/execute/verify cycles through explicit typed states. It records
every transition and fingerprint, enforces per-instance and controller caps, and detects
no progress.

Implement `builtin@commit` as an adapter over the existing commit-finalizer inventory,
auto-commit, evidence, publication, and discarded-work helpers. Do not introduce a
second VCS mutation engine. For each declared commit, map the opaque repo ID internally,
write the message file, and invoke `sase stitch create`; preserve `SASE_COMMIT_METHOD`,
PR metadata, Patch/Stitch tracking, bead auto-close rules, file hooks, push behavior,
and proposal mode.

Execute repository decisions sequentially. Recompute obligation coverage at submit,
before execution, and after execution; any edit after `/sase_final` invalidates the
context. Pass every protected baseline path as an exclusion, verify that no attributable
dirty path was silently omitted, and continue using SHA/tree evidence to recognize
rebased run-owned commits.

Preserve current narrow machine-owned auto-commits and bead publication failure. Model
them as host-owned reconciliation steps, not as implicit agent decisions. Recompute the
agent-facing obligations after they run so a machine-owned path never causes a stale
extra repo decision.

Implement the conflict state exactly as specified in section 2.8. The conflict turn uses
the existing `/sase_git_commit` recovery contract and `commit_state.json`; after
resolution the controller validates `sase stitch create --resume` evidence before
starting another repository. It never counts conflict repair against declaration
recovery.

For a refusal, write its exact reason into per-instance and aggregate results, retain
the dirty workspace under the existing failed-agent recovery policy, and make runner/
notification reporting say which repo was refused and why. Never translate refusal to
clean, skipped, or success.

Port the current commit finalizer test corpus to a parity harness that can run old and
new controllers against identical fixtures. Required cases include:

- clean and ordinary dirty primary workspaces;
- linked, external, sidecar, SDD, and hidden agents-sidecar repositories;
- pre-existing dirt and late-opened pre-existing dirt;
- family baseline inheritance and shared-clone races;
- automatic bead/plan/prompt metadata commits and publication failures;
- run-owned evidence before and after rebase;
- discarded or partially covered work;
- complete commit/refuse coverage and refusal reason preservation;
- missing/stale declaration recovery exactly once;
- first-repo conflict preventing second-repo dispatch;
- successful resume, unresolved second conflict, and contradictory checkpoint;
- provider/model/effort/usage accumulation across recovery turns.

## 12. Compatibility migration, observability, documentation, and soak gates

Add the legacy-config adapter and diagnostics described in section 4. Preserve flag-off
behavior at the invocation seam and write the commit compatibility result projection
from the flag-on controller. Update `runner_reporting.py`, completion notifications,
chat extras, restartable prompt selection, artifact capture, and any tests that assume
only `commit_finalizer_pass_*` artifacts so generic recovery remains visible without
breaking historical agents.

Add structured metrics for selected provider/instance, trigger, attempt, recovery kind,
duration, result, refusal, stale submission, no progress, conflict, and parity branch.
Labels must be bounded provider/instance slugs and status enums—never repository paths,
messages, reasons, or plugin output. Human diagnostics use project/repo display names,
not internal project keys.

Document:

- the `finalizers` configuration and supply-chain boundary;
- plugin author and operator contracts, entry point, worker protocol, and security
  limits;
- `%final` ordered selection semantics and examples;
- `/sase_final` and every `sase final` subcommand;
- commit/refuse behavior and the merge-conflict recovery guarantee;
- feature-flag opt-in, legacy-setting migration, artifacts, and failure recovery.

Update ordinary docs and generated skill sources/tests only. Memory-note edits remain
out of scope without separate current-conversation user approval. After the combined
tree lands canonically, follow the generated-skill commit-first deployment workflow;
until then use read-only previews.

Add adversarial and end-to-end tests for corrupted artifact files, concurrent submits,
oversized manifests/output, nonce replay, plan/context digest swaps, plugin activation
from forbidden layers, env/argv/cwd injection attempts, command timeouts, subprocess
leaks, fixed-point cycles, and old/new config coexistence. Confirm `%final:none` really
clears bundled commit, `%final:lint` retains it, and a project-required commit cannot be
removed.

Run `just install` before validation in each SASE workspace. Every SASE phase runs
`just check`; the combined landing runs `just check-full` only through `/sase_monitor`.
Run focused live beta smoke agents for clean, commit, refusal, command, external plugin,
and forced-conflict scenarios, inspect both generic and compatibility artifacts, and
record soak evidence on the epic/flag bead. Leave the beta default off and leave flag
removal to the dedicated flag bead after the criteria in section 4 are met.

## 13. Acceptance matrix

| Scenario                                       | Required result                                                          |
| ---------------------------------------------- | ------------------------------------------------------------------------ |
| Flag off, no `%final`                          | Exact current commit-finalizer behavior and compatibility artifacts.     |
| Flag off, explicit `%final`                    | Launch-time targeted beta-required error; no stripped override.          |
| Flag on, omitted `%final`                      | Resolved plan selects bundled `commit`.                                  |
| Flag on, `%final:lint`                         | Defaults retained and `lint` added in stable dependency order.           |
| Flag on, `%final:none`                         | Selection cleared unless policy marks an instance required.              |
| Clean commit-only turn omits skill             | No recovery model call; terminal clean.                                  |
| Required declaration omitted                   | Exactly one recovery turn, then fail if still absent/invalid.            |
| Intentional plan/monitor/pipe/question handoff | Runner exits through handoff; no finalizer recovery.                     |
| Dirty repo decision is `commit`                | Exactly one sequential stitch; ledger/tree/publication verified.         |
| Dirty repo decision is `refuse`                | Nonblank reason retained and surfaced; run fails with dirt intact.       |
| Repo opened dirty after bootstrap              | Its dirt is protected by the open-time baseline.                         |
| Submission followed by another edit            | Context is stale; mutation does not start from stale intent.             |
| First repo conflicts                           | Second repo never starts; one agent repair turn resumes the same stitch. |
| Plugin merely installed                        | Listed as available but no instance runs.                                |
| Plugin configured and selected                 | Bounded isolated validate/execute/verify protocol runs.                  |
| Plugin lies, times out, or emits bad output    | Host rejects it and aggregate finalization fails closed.                 |
| Later finalizer creates dirt                   | Triggers recompute; commit becomes pending again before success.         |
