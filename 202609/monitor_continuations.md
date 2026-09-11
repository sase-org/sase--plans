---
tier: epic
title: Reliable monitor continuations with clear results and bounded context
goal:
  Monitor chains preserve the user's intent without recursively replaying history,
  deliver one useful result with recoverable evidence, and reliably continue or finish
  through an explicitly prepared host completion action in a clear, polished interface.
phases:
  - id: baseline
    title: Reproduce failures and measure continuation costs
    depends_on: []
    size: medium
    description:
      "baseline: add deterministic replay, evidence and lifecycle fixtures plus
      component-level prompt measurements and shadow budget checks."
  - id: contract
    title: Define the Rust continuation and result contracts
    depends_on:
      - baseline
    size: medium
    description:
      "contract: implement versioned exact-identity records, deterministic replay
      planning, evidence policies and typed budget decisions with PyO3 bindings."
  - id: capture
    title: Persist local deltas and handoff checkpoints
    depends_on:
      - contract
    size: medium
    description:
      "capture: capture provenance during expansion and persist immutable local turns,
      exact parents and checkpoints across normal and interrupted endings."
  - id: replay
    title: Reconstruct ancestry without recursive transcript replay
    depends_on:
      - capture
    size: medium
    description:
      "replay: integrate unique-node serial replay, exact fork targets, stable blocks
      and a conservative immutable-history compatibility reader."
  - id: diagnostics
    title: Preserve structured verification evidence
    depends_on:
      - contract
    size: medium
    description:
      "diagnostics: capture first-party stage results before temporary output disappears
      and record bounded durable diagnostics and precise log-retention metadata."
  - id: results
    title: Deliver each monitor result once
    depends_on:
      - replay
      - diagnostics
    size: medium
    description:
      "results: freeze terminal results and apply auto, tail, file and none policies
      consistently across successor, family and direct-reference projections."
  - id: dispatch
    title: Make outcome delivery durable and deduplicated
    depends_on:
      - results
    size: medium
    description:
      "dispatch: persist outcome policies and delivery identities, deduplicate successor
      adoption, and reconcile crashes while preserving claims and cancellation."
  - id: prepare
    title: Prepare conditional completion declarations
    depends_on:
      - dispatch
    size: medium
    description:
      "prepare: add a host-sealed completion intent with repository decisions,
      verification requirements, state fingerprints and prepared result text."
  - id: completion
    title: Complete eligible verification through the host
    depends_on:
      - prepare
    size: medium
    description:
      "completion: consume valid success intents through existing finalizers without a
      model turn and route stale or failed completion into durable recovery."
  - id: budgets
    title: Bound continuation context without losing instructions
    depends_on:
      - results
    size: medium
    description:
      "budgets: enforce expanded-prompt budgets, reuse explicit checkpoints at
      thresholds, preserve essential context and expose actionable nonlaunchable
      outcomes."
  - id: experience
    title: Present a coherent monitor workflow
    depends_on:
      - completion
      - budgets
    size: medium
    description:
      "experience: expose the profile and evidence controls, implement compact result
      and continuation views in CLI and ACE, and synchronize documentation and skill
      sources."
  - id: release
    title: Validate the combined feature and activate it
    depends_on:
      - experience
    size: medium
    description:
      "release: exercise failure recovery and visual contracts, evaluate task-level
      efficiency, activate the complete feature with a defined compatibility route and
      verify coordinated Rust and Python delivery."
proposed_by: bbugyi200.athena.0j2
create_time: 2026-09-11 06:30:09
status: wip
---

- **PROMPT:**
  [prompts/202609/monitor_continuations.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/monitor_continuations.md)

# Reliable monitor continuations

## Intended experience

A monitor is a command followed by a clear next step. A user should be able to see what
ran, whether it succeeded, what evidence is available, and whether SASE is continuing,
finishing, or needs attention. An agent continuing the work should receive the necessary
decisions and one result, without paying repeatedly for copied history.

Keep the hardened detached supervisor. Build a small continuation protocol on the
existing shell and artifact infrastructure. Make correctness independent of a model
provider's session storage or cache. The ordinary interface remains:

```sh
sase monitor start -s TESTING -S TESTED -t 45m \
  -n 'Investigate any failures, then finish the requested change.' -- just check-full
```

The new common verification form adds one understandable choice:

```sh
sase monitor start -p verify -t 45m \
  -n 'Fix failures and complete the requested change.' -- just check-full
```

`verify` supplies TESTING/TESTED labels and outcome-aware evidence. It does not by
itself authorize host completion. A successful benchmark, research command, or check can
still require another reasoning turn. No automatic selection of a smaller model.

This is an epic because storage, evidence, delivery, host finalization, and presentation
have separate correctness boundaries and can be implemented and reviewed as bounded
phases. All phases above are direct implementation work sized `medium`; no phase is a
placeholder asking another agent to rediscover the design. The diagnostics work can
proceed alongside capture/replay; budgeting can proceed alongside conditional
completion.

## Basis and current implementation

The design basis is the audited artifact
`research:202609/monitor_continuation_design/monitor_continuation_design.md`. The
research's savings estimates are experiment targets, not promised reductions. The
implementation was inspected at SASE `8d9f24833` and sase-core `7d6dfcf`. Reconfirm
these integration points against the implementing checkouts:

| Area            | Existing implementation and implication                                                                                                                                                                                                                            |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Handoff storage | `src/sase/axe/run_agent_exec_monitor.py` saves `state.current_prompt` with a synthetic handoff response. `run_agent_exec_finalize.py` also saves the expanded prompt. Neither is a reliable local-delta source.                                                    |
| Expansion       | `src/sase/xprompts/fork.yml`, `scripts/fork_history.py`, `llm_provider/preprocessing.py`, and workflow expansion currently pass rendered strings. Capture source identities and segment provenance here, before they become indistinguishable text.                |
| Replay          | `src/sase/history/chat_fork/{build,family,proc}.py` and `history/chat_resume.py` replay saved text. Family headings include changing member totals. The separate `previous_history` serialization parameter does not guarantee ancestry reconstruction.            |
| Output bypass   | `src/sase/scripts/_fork_proc_sources.py` reads a monitor tail independently of `monitor/followup_prompt.py`. Suppressing the latter's output does not suppress it from family history.                                                                             |
| Lifecycle       | `src/sase/monitor/{start,transaction,supervise,reconcile,settlement}.py` and `src/sase/shells/{followup,settlement}.py` already provide startup acknowledgments, process-group cleanup, starter barriers, claim fallback, and delivery status. Extend these seams. |
| Verification    | `tools/run_silent` knows stage name and exit code, but deletes successful captures and prints failures into a rotating aggregate log. The `Justfile` has independent formatting, lint, validation, scoped-test and full-test stages.                               |
| Finalization    | `src/sase/finalizers/{declaration,declaration_store,plan,controller,ledger}.py` already authenticate plans, bind turn context, validate repository decisions and drive host completion. A monitor handoff currently skips normal finalizers.                       |
| Presentation    | `main/monitor_render.py`, `ace/tui/widgets/prompt_panel/_agent_monitor_section.py`, existing monitor row models and visual fixtures provide the visual language.                                                                                                   |
| Core            | sase-core has `agent_family.rs`, `agent_scan/`, `text_tail.rs`, `finalizer/`, and PyO3 adapters; add the missing continuation domain here. The inspected Python dependency is `sase-core-rs>=0.33.0,<0.34.0`.                                                      |

Open sase-core with `/sase_repo` and `sase repo open sase-core -r '<specific reason>'`;
use only the returned checkout. Paths beginning `crates/` below are relative to that
repository. Other paths are relative to the SASE checkout. Do not assume sibling paths
or any particular workspace number. Do not edit research artifacts or canonical SASE
memory as part of this plan. Documentation and generated-skill **source templates** are
included; deploying generated copies is a separate post-landing operation.

## Invariants

1. Parent edges identify exact executions or terminal results, never a mutable family
   alias. A terminal record and its referenced content are immutable.
2. New canonical turns contain local content only. Expanded prompt archives are useful
   debug artifacts but are never ancestry inputs for the new reader.
3. A new serial replay includes every reachable event once, preserving constraints and
   attributed decisions. Distinct events with equal text remain distinct.
4. One monitor result owns its execution facts, evidence references and continuation
   intent. Its newest diagnostic excerpt is injected once. Human inspection may still
   show logs explicitly, independently of model-context policy.
5. Host observations determine state and exit status. Command-authored summaries are
   untrusted and cannot choose routing, report authoritative success, or authorize
   finalizers.
6. Stop and lost outcomes do not automatically launch successors or finish work. A
   custom policy cannot override this cancellation rule in this release.
7. Terminal command outcome, delivery state and task completion are separate facts.
   `exit 0` alone never means the user's task is complete.
8. The workspace and family slot retain continuous ownership until the successor or host
   completion worker acknowledges ownership, or a durable terminal disposition records
   why there is no next owner. Unknown ownership is reconciled before release.
9. Budget decisions never silently discard instructions. Unsupported or incomplete
   context produces a recoverable, visible error rather than fabricated ancestry.
10. Existing supervisor acknowledgments, launch barriers, timeout behavior, whole-group
    termination, environment scrubbing, stop semantics and claim fallbacks remain
    covered.

## Data model and responsibility boundary

Add a versioned `continuation` domain to `crates/sase_core/src/`, focused into schema,
replay, evidence, policy and budget modules rather than one large module. Export narrow
operations from `crates/sase_core_py`; consume them through typed Python adapters. Rust
owns validation, hashes over canonical values, parent ordering, deduplication, evidence
selection, policy resolution, budget selection, completion eligibility and delivery
transitions. Python owns file/process effects, artifact registration, provider
invocation, worktree observation, and CLI/Textual rendering.

Use existing artifact identities and per-shell storage; no graph database or new
scheduler. Internal node IDs encode project plus exact run identity and kind, with a
content digest to detect accidental replacement. They are not new user-facing artifact
reference syntax. Use existing indexed-file references for portable content. Do not
embed physical workspace paths in identity; keep execution location as factual metadata.

Define these version-1 records, with explicit version rejection, size/depth bounds, and
round-trip fixtures shared across Rust, PyO3 and Python:

| Record              | Required information                                                                                                                                                                                                                                                                                                                                       |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Continuation node   | Version, exact node ID, kind, ordered exact parent IDs, owner execution identity, immutable content ref/digest, optional checkpoint/intent/workspace refs. Kinds initially include agent delta, monitor result, checkpoint and an attributed legacy boundary. Human gate decisions retain typed attribution or a complete protected legacy decision block. |
| Agent delta         | Authored local request, materialized local prompt segments, final response or handoff checkpoint, source/provenance refs, and explicit interrupted/failed status. Injected parent segments are excluded by provenance, never heading parsing.                                                                                                              |
| Continuation intent | The next action once, authored checkpoint reference, resolved model/effort or inheritance, frozen outcome policy, and optional conditional-completion reference.                                                                                                                                                                                           |
| Monitor result      | Monitor ID, exact starter identity, outcome, exit code or unknown, command/cwd, start/end/elapsed, timeout kind/budget, workspace identity, diagnostic manifest ref and retained-log metadata.                                                                                                                                                             |
| Diagnostic manifest | Producer/version, stage IDs/names, stage status/exit, bounded diagnostic refs, counts with provenance, capture errors, retained ranges and completeness. Claims remain distinct from supervisor facts.                                                                                                                                                     |
| Delivery record     | Key `(monitor_id, result_id, branch)`, selected action, reserved successor or completion identity, attempt history, acknowledgment and typed disposition/reason. Mutable operational state, separate from immutable results.                                                                                                                               |
| Replay manifest     | Ordered node IDs, branch attribution, projection version, selected evidence refs, checkpoint coverage, omissions, rendered-component sizes, budget and prefix-reset reason.                                                                                                                                                                                |

Persist immutable bytes and references before publishing pointers to them. Use atomic
replace and existing locking conventions, with a recoverable journal for multiple-file
publication; test crashes between every stage. A duplicate freeze with the same digest
is idempotent; a conflicting digest is an integrity error, never last-writer-wins. Pin
referenced checkpoints and failure evidence through existing artifact-retention
protection so active/recoverable continuations cannot lose their sources to cleanup.
Archiving debug prompts must not create recursive authoritative snapshots.

## Correct capture and replay

Extend the preprocessing/workflow result with typed segment provenance and resolved
parent identities. Resolve aliases once at launch. Keep ordered parent IDs when a target
is still running, then wait for that exact run's terminal content. Do not resolve the
alias again after another family member starts. Propagate this context through
`LoopState`, ordinary completion, monitor handoff, retry and failure paths; neither
`original_prompt` nor Markdown `New Query` delimiters substitute for this context.

For a serial monitor handoff, the graph is:

```text
previous exact result -> starter's local delta -> monitor's terminal result -> successor delta
                              |
                              +-> checkpoint and the single continuation intent
```

The successor's transport references the exact monitor result; family attachment is
separate scheduling metadata. The graph renderer includes the starter once and the
monitor result once. Do not combine a whole-family fork with a second result body.
Capture initial user requests and follow-up steering faithfully, including constraints
introduced by referenced prompt content and human review gates.

A checkpoint is optional structured YAML/JSON supplied with `-k/--checkpoint FILE`. It
contains objective, constraints, findings, unresolved questions/decisions, remaining
work, source refs and optional explicitly covered node IDs. The host adds repository
identity, changed paths, artifact-read refs and completed-check refs. These two
authorship sources remain distinguishable. Keep `--next` text losslessly in the intent
record; do not duplicate it in the checkpoint schema or synthetic handoff transcript. If
no checkpoint was supplied, preserve the full next action and captured local request; do
not pretend the host recovered the starter's private reasoning or tool results. Apply a
generous defensive file bound with an actionable error, not a tiny prose limit.

Canonical replay is parent-first, appending each unseen node in the frozen parent order.
For a new attributed merge, retain the primary parent's existing ordered blocks, append
unseen secondary-parent nodes in their existing order, then append a merge attribution
block referencing shared nodes. Do not globally sort the union or use changing
`Member i of N` headings. Canonical blocks use stable IDs. Changing projection/evidence
age or introducing a checkpoint records a prefix reset; claim byte-stable prefixes only
when the rendered projection is unchanged.

First activate this reader for serial monitor continuations and direct/family forks into
their versioned ancestry. Preserve the existing legacy multi-parent behavior, including
`test_multi_parent_expansion_preserves_shared_ancestry_per_parent`; do not quietly
rewrite its shared-ancestor semantics. New graph fixtures must demonstrate explicit
branch attribution before any broader multi-parent rollout.

Old transcripts remain immutable. A compatibility reader normalizes only structures with
recoverable parent provenance. Missing parents produce explicit missing-source entries.
An uncertain legacy snapshot is an attributed opaque boundary, never guessed local
turns. Preserve it within budget or report that a checkpoint is needed. Under strict
`none`/`file` evidence policy, an opaque legacy blob that may contain raw logs cannot be
blindly included: use a provenance-backed representation or stop automatic launch with
`legacy_evidence_policy_conflict`. Users retain explicit access to the original
artifact. Missing essential context and a failed starter without a durable checkpoint
must not silently become a fresh, apparently complete conversation.

## Result and evidence policy

Resolve policy once and persist it on start. New continuation-enabled launches default
to `auto`; old records lacking a policy decode with their historical `tail` default.
Explicit controls stay available through `-o/--next-output auto|tail|file|none`.

| Policy/outcome       | Model context                                                                                                               |
| -------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `auto`, completed    | Host execution facts, concise structured stage/count summary if available, refs; no raw log by default.                     |
| `auto`, failed       | Host facts, failed-stage diagnostics, and a small fallback excerpt when structured diagnostics are absent or insufficient.  |
| `auto`, timeout      | Timeout kind, last observed progress, available diagnostics and a bounded tail.                                             |
| `auto`, stopped/lost | Reason, uncertainty and recovery refs for inspection/manual forks; no automatic continuation.                               |
| `tail`               | Facts and the selected retained tail, subject to both line and byte budgets.                                                |
| `file`               | Facts and canonical evidence refs plus valid local log locators; no embedded diagnostics or raw output.                     |
| `none`               | Facts and a bounded-retrieval command/ref; no embedded diagnostics or raw output, on any replay path.                       |
| Older results        | Compact host facts and references; no repeated raw excerpts. Keep human gate decisions and reviewer instructions available. |

Centralize this selection in Rust. Successor composition, direct monitor forks, family
forks, resumed versioned transcripts and prompt previews consume the same projection. No
source builder independently loads a log to append it later. Legacy compatibility cannot
bypass a strict policy. Human `show`, explicit artifact reads, and expanded TUI logs
still allow deliberate evidence inspection. Live monitor forks retain their exact target
wait and do not freeze partial progress as a terminal result.

Use named, configurable defaults in `default_config.yml`, with typed validation and
documentation: 8 KiB of selected diagnostics, 4 KiB fallback tail, 12 KiB total raw
excerpt maximum, and existing 200-line tail limit. Count UTF-8 bytes correctly; preserve
valid boundaries and label excerpt omissions. These are resource defaults to test and
tune, not claims about token counts or diagnostic recall.

Extend `tools/run_silent` using a monitor-owned diagnostic sink and a unique stage ID.
Preserve its console behavior and exact exit status. Before deleting captures,
atomically publish stage metadata and a bounded diagnostic artifact. Parallel stages
need distinct files; a manifest assembled by the host avoids concurrent JSON append
corruption. First-party producers include formatting, ruff, mypy, SASE validation,
scoped tests and full tests; incorporate native reports where available. Do not equate
every check failure with a pytest failure. Record skipped/unfinished stages so an
early-exit run cannot claim all verification completed. Work outside a monitor keeps
current behavior.

Keep raw logs bounded with the existing two-segment policy initially; its default is
approximately 4 MiB retained, not an unlimited archive. Track total observed bytes,
dropped bytes, retained half-open byte ranges and completeness across rotation and
oversized chunks. Snapshot final retained segments/manifests so results never point to
mutable files that can later rotate. A read spanning a gap returns available ranges and
an explicit gap notice. `--all-lines` means all **retained** lines. Completeness
requires a confirmed drain to EOF as well as no discarded observed bytes. If supervisor
loss or a drain failure makes the total unknowable, report unknown loss; do not present
an observed-byte counter as the complete amount the command produced.

Diagnostics have a separate configurable quota: initially 256 KiB per failed stage and 2
MiB total per monitor, with bounded stage count and explicit quota-exceeded metadata.
Capture them before aggregate-log rotation. Report evidence-write failure independently
of command failure and preserve a fallback where possible. Raw and diagnostic retention
are separate: neither claims that discarded evidence can be retrieved. Full raw-log
archival is an optional later resource policy, not required for this release.

Add bounded retrieval to `monitor show`: `-d/--diagnostics`, `-R/--range START:END` (raw
byte offsets), and `-b/--max-bytes N` (64 KiB default, 1 MiB maximum for these bounded
modes). Diagnostics and range are mutually exclusive; range cannot combine with follow
or all-lines. Preserve existing explicit `--all-lines` behavior. JSON returns
completeness, available/missing ranges and continuation offsets as well as text.
Malformed producer JSON, invalid sizes, duplicate IDs, symlink/path escapes or
unsupported versions cannot change execution facts; report an invalid diagnostic record
and use bounded fallback evidence. Retain disabled-region and fence protections for
every command-authored string, including adversarial headings, fence runs, directives
and shell substitution text.

## Outcome policy and durable delivery

Use a small versioned policy object, selectable with `-p/--profile verify` or
`-P/--policy FILE`, rather than a flag for each outcome/model combination. A policy
contains `completed`, `failed`, and `timeout` branches. Actions are `continue`, `none`,
or `complete`; `complete` is legal only on success with a prepared intent. A continue
branch may override model/effort and action text. File parsing never evaluates code.

Precedence is explicit CLI override, selected policy/profile, then shared `--next` and
`--model`, then inherited routing. Reject simultaneous profile/policy selection and
contradictory or incomplete actions before changing claims or handing off. Include the
fully resolved checkpoint, completion intent and branch policy in the start-request
fingerprint, so replays cannot silently attach different actions to an existing command.

For ordinary commands, omitted `--next` remains no-successor behavior unless a policy
explicitly selects a continuation or the user attaches a prepared completion intent.
`verify` plus `--next` continues on all three automatic outcomes by default. Attaching
`-f/--completion REF` changes verified success to host completion; failure/timeout use
`--next` or a built-in recovery instruction to preserve the original objective and
inspect/repair the reported failure. Profile selection alone never authorizes
completion.

Persist the frozen result and pending delivery before dispatch. Reserve the successor's
exact identity before spawn, pass the delivery key, and require the receiver to
atomically adopt that key **before provider invocation**. A supervisor death after spawn
but before acknowledgment must find the reserved receiver, rather than start another
model turn. Reuse the existing launch/admission journal where its contract fits; extend
its typed API instead of maintaining a competing scheduler.

The logical delivery sequence is:

```text
result frozen -> pending -> reserved -> dispatching -> acknowledged -> settled
                                |            |
                                +-- recovery/inspection of the same delivery key --+
```

Attempt records distinguish retryable pre-dispatch failure, acknowledged ownership,
degraded launch, cancelled and nonlaunchable/uncertain delivery. Retries reuse the same
identity; a terminal failure can produce one separately keyed recovery branch. Do not
claim arbitrary external side effects are exactly-once: an executor without idempotency
or a recoverable receipt becomes needs-attention after an ambiguous dispatch.

Extend reconciliation to incomplete deliveries even when `monitor_state` is already
terminal and the supervisor is gone. The current running-only check is insufficient. Use
bounded locks and a documented lock order; never hold store locks while waiting for a
provider, command, user, or child acknowledgment. Preserve claim fallback for ordinary
successors and state which workspace was used. No-model completion requires the original
verified workspace; it may not use degraded workspace fallback.

Serialize stop versus terminal-delivery selection: a stop recorded before delivery
reservation cancels it; after ownership is acknowledged, stopping the settled monitor
does not kill the successor. Stop remains idempotent and reports that handoff already
occurred. Pre-reboot lost state never dispatches automatically. A same-boot dead
supervisor follows existing failed recovery semantics, without rerunning the command.

Provide `sase monitor resume ID` for a requested continuation that is conclusively
undelivered, with optional `-k/--checkpoint FILE`, `-m/--model MODEL` and `-j/--json`.
It resumes the next action, never the monitored command. Reconcile first; an already
acknowledged continuation returns its existing identity, and ambiguous ownership refuses
another launch. A supplied checkpoint/model creates a new immutable intent revision and
an explicitly numbered manual-recovery branch in the delivery key; it never edits the
terminal result or original intent. Admit that revision atomically against the previous
disposition so concurrent resume commands cannot create two successors. This command
cannot retry conditional finalization, override stopped/lost cancellation, or invent an
action for a fire-and-forget monitor. Those monitors remain inspectable and manually
forkable with the existing `F` action. CLI errors and ACE needs-attention details show
the exact resume command when it is eligible. Resume mutations from ACE, if exposed as
an action, use the existing durable launch-proc machinery.

## Prepared host completion

Add `sase final prepare MANIFEST` as a non-terminating preparation command. It uses a
host-issued context and a versioned wrapper containing the existing declaration payload,
prepared success message, and exact required verification command/stage contract. It
validates and stores a host-sealed, single-use intent and prints its artifact reference
and a readable preview; support `-j/--json`. It does **not** submit normal finalization,
commit, publish, or end the turn. `sase final submit` retains its normal semantics.

Example interaction, with an authored declaration file:

```sh
sase final prepare completion.json
# Copy the returned immutable reference into the next command.
sase monitor start -p verify -f '<prepared-intent-ref>' -t 45m \
  -n 'Diagnose failures or stale verification, then finish the requested change.' \
  -- just check-full
```

The preview states the success action, repository decisions, required checks, prepared
message, and failure/timeout routing. Binding to the monitor request is atomic and
single-use. A preparation failure or mismatch leaves the creator alive and creates no
monitor. Starting without a prepared intent remains fully supported.

The host seal binds creator/run identity, resolved finalizer-plan digest, repository IDs
and ownership evidence, declaration requirements and payloads, command identity,
checkpoint/intent, and worktree fingerprint. Observe all relevant opened repositories,
HEAD/index state, tracked changes including deletions/modes/symlinks, new files intended
for the commit, and protected/foreign paths through the existing finalizer evidence
mechanism. Extend observation where existing dirty-only fingerprints cannot establish
the tested state. File mtimes alone are not sufficient. Incomplete or unstable reads
make the intent ineligible.

At monitor start and command completion, verify the fingerprint and required check
contract. Immediately before finalizers run, recompute obligations and state under
exclusive workspace ownership. A zero exit with missing required stages, altered files,
different HEAD/index, new repository obligations, changed finalizer requirements,
unsupported executor semantics, or degraded workspace location routes to one recovery
agent. Do not transplant an old turn nonce into a new normal declaration. Validate a
separate conditional-intent type and mint the host execution context only after its
conditions hold. Command output and diagnostic claims cannot grant eligibility.

Initially support no-model success for registered first-party verification contracts for
`just check` and `just check-full`, with an exact command match and the required
verification level sealed in the intent. Reject an arbitrary shell expression as a
completion prerequisite; it may still run with an ordinary continuation. Stage evidence
can veto eligibility when incomplete or contradictory, but a producer's reported pass
cannot replace the host-observed successful command and verified repository state. Do
not silently lower a required full check to a scoped check. Additional producers need an
explicit host contract before they can use conditional completion.

The host completion worker adopts the delivery key and claim, then calls the existing
finalizer controller with an explicit no-model mode. The commit finalizer consumes the
prepared repository decisions through the established host-owned stitch path. Reuse
receipts/ledgers for completed repository actions. A plugin finalizer must support
headless execution and durable replay or make this path ineligible; do not silently
invoke a model from a supposedly no-model success path. If a finalizer changes relevant
content, needs a new declaration, needs conflict repair, or fails, stop the fast path
and transfer to durable recovery; do not apply unverified changes under the old intent.

Publish the prepared success message only after all required host actions succeed. Allow
only documented literal substitutions for host facts such as duration and evidence ref;
no shell/template execution. Surface `Finalizing` while work is in progress and
`Completed by host` only after success. Preserve existing agent-family completion,
task/bead completion, result publication and artifact bookkeeping without inventing a
fake LLM shell or counting one in usage statistics.

Test the complete crash matrix, including commit succeeds but the supervisor has not
recorded acknowledgment. Observe durable completion receipts and reconcile; do not run
the commit/publish action again merely because a marker is missing. External actions
with an ambiguous outcome require explicit recovery rather than blind retries.

## Budgets and checkpoints

Measure after all SASE expansion and final routing selection, immediately before
provider invocation. Maintain a typed provider budget contract containing known context
capacity, transport byte limits and reserves for instructions, tools, output and
reasoning. Adapters supply known limits; configurable conservative defaults handle
unavailable CLI request details. Record estimator/version, UTF-8 bytes, estimated
tokens, reserves and uncertainty. A SASE-only estimate must never be labeled a guarantee
of total provider tokens. Check argv/environment limits for argv transports; prefer
supported stdin/file transport rather than truncation. Recheck if fallback routing
selects a smaller budget.

Initial order of reduction: identity deduplication, removal of old raw excerpts, bounded
newest diagnostics, then a previously authored checkpoint covering older work. Keep the
active objective, original user constraints and later steering, human gate decisions,
unresolved decisions, current next action and latest result. User/gate instructions
remain protected unless an explicit attributed update supersedes them. Do not infer
semantic irrelevance merely from age.

Use checkpoints only when the prompt crosses a threshold; preserve covered-node/source
refs and a visible omissions manifest. A checkpoint is an alternate projection, not a
replacement or deletion of canonical events. Host-generated checkpoints can summarize
host facts; they cannot fabricate conclusions. No extra summarization model call after
every monitor. If an adequate checkpoint is unavailable or essential content still does
not fit, persist the intended continuation and a `context_budget_exceeded` disposition
with size breakdown and recovery guidance. Release or transfer the claim only through
normal durable settlement. No repeated automatic retries of the identical oversized
prompt. Log retrieval counts toward evaluation of total context cost. Run a cheap
preflight before dispatch where the complete prompt is available; always recheck the
final expanded prompt at the receiving runner before invoking a provider. If that last
check refuses, acknowledge ownership and persist the typed refusal back to the parent
delivery record before releasing the claim. A spawned runner with no provider invocation
is not a successfully delivered model turn. `monitor resume` with an adequate checkpoint
or model budget is the recovery path, without rerunning the check.

## CLI and ACE visual design

Reuse the amber monitor glyph, status-pair accent, failure red, dim metadata, existing
fold controls and file hints. Keep the command's outcome distinct from the next action.
Do not add a row badge for routine savings or expose storage versions on the main view.
Make exceptional continuation state visible using the existing attention flag with plain
text in details; color is supplemental to words and glyphs.

Example settled failure, using the same information hierarchy in CLI and ACE:

```text
⚙ TESTED  ✗ exit 1                                      2m 14s
  just check-full
  Result        Type checking failed
  Next          Continuing · repair--2 · inherited model
  Evidence      mypy · 3 diagnostics · retained output incomplete

  ▸ Diagnostics
  ▸ Retained output
  ▸ Context and delivery details
```

Example eligible success:

```text
⚙ TESTED  ✓ exit 0                                      3m 02s
  just check-full
  Result        Required checks passed
  Next          Finalizing
  Evidence      Stage results available
```

After acknowledgment, `Next` becomes `Completed by host`. If the fingerprint changed,
show `Continuing · verification became stale`; if recovery cannot launch, show
`⚑ Needs attention · <specific reason>` with an inspectable saved continuation ref.
Never render a custom past-tense label as proof of success. Empty, running, stopped,
lost and unknown-exit cases use concrete text rather than blank diagnostics panels.

Keep detailed command, cwd, timing/budget, exact IDs and replay/budget information
available in expanded details. Human logs remain expandable and ANSI-aware. CLI JSON
adds versioned `result`, `evidence`, `continuation` and `context_budget` objects while
preserving existing status and follow-up fields with faithful projections. List/table
output stays compact; show/detail views explain the disposition. Restore focus and
selected identity after async operations.

No file reads, JSON decoding, provider calls, artifact resolution or subprocess work on
the UI thread or serial event pump. Compute projections with existing cached/off-thread
loaders and refresh tokens. Coalesce refreshes, update rows selectively and keep the 150
ms detail debounce. Use existing folds rather than introducing new keymaps unless an
accessibility test demonstrates a need; any keymap/config change must update
`src/sase/default_config.yml`.

All new public long options have the short aliases specified above. Sort help entries
and subcommands, explain mutual exclusions and defaults, and keep options optional. The
generic status-label behavior stays compatible; the verification profile supplies its
own defaults. Fix monitor skill examples to use the documented command after `--`. Teach
one supervised command waiting for an external terminal state with bounded backoff
instead of repeated sleep/agent polling loops. Do not add speculative named wait
adapters. Shorten boilerplate only after checkpoint preservation is implemented.

## Phase implementation and acceptance

### baseline

Add deterministic fixtures near `tests/history`, `tests/monitor` and `tests/fakey` for
recursive family growth, split-only ancestry loss, output-policy bypass and starter
interruption. A small synthetic fixture must demonstrate the old defect without saving a
giant prompt in the repo. Add component measurements around fork rendering and final
provider preprocessing: unique/duplicate node counts where known, local/history/evidence
bytes, total expanded bytes, fallback routing and estimated budget headroom. Shadow
checks record diagnostics without changing current launch behavior. Establish baseline
fixtures for actual failure classes and stage early exit. No production transcript
harvesting or provider launches are needed for these regressions.

### contract

Implement the record and pure planning contracts above in Rust and PyO3. Include
unknown-version/type/size failures, cycles, conflicting duplicate IDs, missing parents,
stable serial order, attributed diamond graphs, evidence policies, and budget result
types. Add shared JSON fixtures and a thin `src/sase/core/continuation_*` adapter. Keep
unintegrated APIs behind internal seams until release; temporary epic symbol allowances
must follow the Symvision memory and be removed when callers exist.

### capture

Add typed provenance to preprocessing and embedded workflow expansion, then propagate it
into execution state and persistence. Cover normal completion, monitor handoff, failure
before final reply, retries and interrupted starter settlement. Store checkpoint and
host workspace facts before the creator can be terminated. Include prepared files in the
existing start transaction/rollback: failed acknowledgment must leave no armed intent or
mistaken terminal starter. Tests put `New Query`, `Prompt`, `Response`, fences and
directives in ordinary user content to prove there is no delimiter recovery.

### replay

Wire Rust replay into versioned serial sources, direct forks and monitor family views;
separate family attachment from history target. Add conservative compatibility decoding
and preserve old readers/artifacts. A 100-turn chain with fixed-size unique deltas must
render each event once; verify incremental storage and rendered bytes grow linearly
within explicitly bounded metadata overhead. Test alias reuse, delayed settlement,
direct/family forks, missing parents, failed starters, opaque legacy boundaries and
legacy multi-parent attribution. Test stable canonical prefix bytes under unchanged
projection and explicit reset markers when projection changes.

### diagnostics

Extend the shell wrapper and first-party report adapters with isolated stage sinks and
durable evidence capture. Extend log capture/rotation with exact byte accounting and
freezeable retained segments. Test formatter, ruff, mypy, validation, pytest, missing
executable, signal, timeout, malformed report, disk-full and quota-exceeded cases. Prove
failed-stage evidence remains retrievable after aggregate log rotation. Parallel stage
completion cannot overwrite another stage's report. Evidence errors must not change the
command's original exit status or falsely claim verification passed.

### results

Freeze results in both `_finish_monitor` and dead-supervisor reconciliation before
follow-up composition. Project the same result through every prompt path. Add a
cross-product test over outcome, output policy, latest/historical age, and direct/family
entry. Use unique raw-log sentinels to prove `none` and `file` inject zero raw evidence
and latest result content appears once. Test continued human inspection, byte/line
budgets, hostile output, Unicode boundaries and active-monitor fork waits.

### dispatch

Implement frozen policies and durable delivery/adoption through current launch APIs.
Extend monitor request fingerprints, meta/done wire fields, reconciliation queries and
claim settlement. Add deterministic crash points before/after result publication,
reservation, spawn, receiver adoption, claim transfer and acknowledgment. Across restart
and simultaneous reconciliation, assert at most one provider invocation per delivery,
one family slot, no premature workspace release, and a durable reason for any action not
delivered. Cover stop races, lost supervisors and existing degraded fallbacks. Cover
concurrent manual resume, repeat resume after successful delivery, refusal under
uncertain dispatch, and resuming an oversized prompt after supplying a checkpoint.

### prepare

Implement the conditional-intent schema and pure Rust validation, host observation and
sealing, preview and preparation handler, plus single-use binding in monitor start. Test
missing repository decisions, stale contexts, altered finalizer plans, unsupported
payloads/executors, foreign/protected paths, duplicate intent use, changed commands and
failed-start rollback. Preparation must perform zero finalizer side effects and keep the
agent running. Hold public exposure until the consumption path is complete.

### completion

Add a host completion receiver that adopts the durable delivery and original claim,
validates the intent and invokes existing finalizers in no-model mode. Integrate actual
task/family completion and prepared message publication. Test all relevant tree changes,
missing/skipped verification stages, new repo obligations, mutation during finalization,
stale/missing intent, model-requiring finalizers and retry receipts. Eligible success
must invoke zero LLM turns; ineligible success gets one attributable recovery attempt.
Use fake finalizer/provider adapters for deterministic tests and local temporary repos
for commit-receipt integration; never exercise real publishing as a test side effect.

### budgets

Implement final preflight enforcement and thresholded checkpoint projections using the
Rust budget planner. Scope initial activation to the new continuation route, including
its fallback provider invocations. Add oversized essential-content refusal, compactable
history, routing-to-smaller-context, argv-byte overflow, unknown CLI reserve and
Unicode-heavy inputs. Prove every omission is disclosed and source material is still
retrievable. Measure retained prompt plus follow-up evidence retrieval, not prompt size
alone. No automatic LLM checkpoint generation in this release.

### experience

Integrate parser/handler/profile configuration and consistent CLI/ACE view models. Add
help and machine-schema tests, checkpoint/profile examples and interaction tests. Update
`docs/monitors.md`, `docs/cli.md`, relevant family/configuration/finalizer docs,
`src/sase/default_config.yml`, and skill sources
`src/sase/xprompts/skills/sase_monitor.md` and `sase_final.md` as necessary. Preview
skills with `sase skill init --diff` or `--dry-run`; do not deploy an unlanded revision.
Add and visually inspect 90x40 and 120x40 snapshots for running, successful host
completion, failed diagnostics, timeout, lost, degraded and needs-attention cases.
Exercise keyboard folds, narrow wrapping, plain/no-color output and long Unicode names.

### release

Integrate all phases before activating any new default. Run the end-to-end matrix below,
inspect actual visual artifacts, and compare the measurements against the frozen
baseline. Activate the complete route using the compatibility procedure below. Remove
temporary integration scaffolding and epic-only symbol allowances. Produce an evaluation
artifact through `sase artifact create`, recording reproducible commands, corpus bounds,
per-intervention results and limits; the report is not canonical memory. Run coordinated
Rust/Python checks and the combined-tree full verification before landing.

## Rollout, compatibility and verification

Earlier phases may add library code and test-only integration seams, but must not route
live users into an unfinished pipeline. At final activation, create a sunset flag
`monitor_continuation_records` through `sase flag new`, with enabled behavior being the
complete new route and disabled behavior being the existing writer/launcher for **new**
runs. Author the three required policy sentences and use the CLI-generated registry
entry/removal bead. Its removal condition is passing ancestry, evidence, delivery,
completion and representative task-evaluation gates plus migration of known callers.
Test both states. Removing it deletes the old writer/launcher branch and makes the new
path unconditional. Readers for already-written v1 records remain available in either
flag state, with their stored policies honored; disabling must never strand records,
reinterpret pending actions, or rearm an intent.

Immutable historical formats remain readable through explicit versioned codecs; they are
not a fallback backend or a second authoritative writer. Existing multi-parent semantics
and explicit tail/file/none choices remain supported. If an intermediate phase must
expose unfinished behavior despite the planned late activation, use the mandated epic
beta scaffold, test both states, and remove its Off branch and flag before the epic
lands. Permanent quotas, profiles and budget controls belong in configuration.

Release core/bindings before Python requires their new APIs. Let release-plz own Rust
versions; do not hand-edit Cargo package versions. Ratchet the Python dependency window
and lockfile to the actually released compatible wheel, with no Python fallback. Use the
built wheel from the opened core checkout for isolated integration verification before
release. Follow host-owned finalization for repository changes. From the clean landed
source, deploy skill templates with `sase skill init --force` and the normal chezmoi
workflow; never deploy from a dirty or unmerged workspace.

Required evaluation:

| Gate                | Acceptance                                                                                                                                                                                                                                                                                                                                        |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Ancestry            | 100-turn unique-content chain; direct/family/legacy/multi-parent fixtures preserve required constraints and attribution; no guessed boundaries; linear new canonical storage.                                                                                                                                                                     |
| Prompt projection   | Each result payload once; explicit no-output policies hold across every route; stable blocks and disclosed prefix resets; no silent clipping or directive execution.                                                                                                                                                                              |
| Evidence usefulness | Stage-specific diagnostics cover test, formatting, lint, type, validation, process and timeout cases. Structured failures survive log rotation; gaps and capture failures are explicit. Required diagnostics in the labeled deterministic corpus are preserved or explicitly retrievable.                                                         |
| Lifecycle           | All outcomes, failed starter, launch-ack failure, same-boot/reboot recovery, stop races, multiple reconcilers and workspace fallbacks preserve ownership and cancellation.                                                                                                                                                                        |
| Completion          | Verified unchanged work plus complete prepared intent finishes without an LLM; changed/missing/unsupported prerequisites do not. Duplicate/crashed dispatch never repeats an acknowledged host action.                                                                                                                                            |
| Budget              | Fully expanded checks include provider reserves and transport bounds; uncertain estimates are labeled; oversized protected content produces one actionable disposition.                                                                                                                                                                           |
| UX/performance      | Reviewed narrow/wide snapshots; readable no-color states; no UI-thread I/O; existing refresh caches and selection remain correct. Use documented TUI navigation and idle-refresh measurements, targeting p95 key-to-paint below 16 ms and no new repeated idle artifact reads.                                                                    |
| Task efficiency     | Compare normalization, evidence selection, structured diagnostics, branching and checkpoints separately. Record prompt bytes, estimated/actual uncached input, cache reads/writes where exposed, output/reasoning usage, retrieval volume, successor count, elapsed time and correctly completed tasks. Missing provider telemetry stays unknown. |

Use deterministic fake-provider tests for crash, claim, budget and completion coverage.
A bounded representative shadow-render corpus must cover success, test/lint/type
failure, timeout, long chains, legacy boundaries and cold/warm-cache observations where
available. Real provider task evaluations use the normal SASE launch-approval workflow
and established usage controls. Compare matched workloads; reject an optimization that
introduces missed failures, unnecessary reruns or unresolved finalizer obligations. Do
not gate on the research's speculative 87% figure.

Provider sessions and cache-specific routing are a later optimization decision, outside
the production implementation here. The evaluation must state whether the portable path
leaves a measured problem worth a bounded experiment, with an explicit go/no-go result.
Any subsequent experiment must verify current official provider support, target exact
session IDs rather than `--last`, survive starter termination and workspace relocation,
and retain the portable fallback. No provider-specific session dependency is needed to
complete this epic.

For each phase that changes tracked SASE files, read the required lint/test memory and
run `just check`. For core changes, run that repo's `just check` or `./scripts/check.sh`
with Python >=3.12 reachable; include PyO3 tests, not only `cargo test -p sase_core`.
For visual changes run the dedicated visual suite and inspect changed images. Before
landing the combined epic tree, run `just check-full` only through `/sase_monitor` with
TESTING/TESTED and a continuation that handles failures. Re-run relevant checks after
fixes and finish the full gate. The planning turn itself creates only this scratch
proposal; validate it with `--explain`, revalidate without it, and submit with
`sase plan propose` before implementation begins.
