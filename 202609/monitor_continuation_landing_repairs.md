---
tier: epic
title: Finish monitor continuation correctness and production acceptance
goal: 'Complete the missing capture, replay, evidence, delivery, recovery and budget
  paths required by sase-zl, preserve intervening launch and capacity changes, and
  prove the integrated feature before resuming its interrupted landing.

  '
parent_bead: sase-zl
phases:
- id: capture
  title: Preserve local provenance and durable exact handoffs
  size: medium
  depends_on: []
  description: 'capture: exclude injected ancestry from canonical local records, accept
    authored checkpoints, and publish immutable recoverable handoffs with exact parent
    references.'
- id: replay
  title: Hydrate ancestry and retain protected context
  size: medium
  depends_on:
  - capture
  description: 'replay: traverse persisted exact parents from a single source, render
    materialized constraints and checkpoints, and refuse incomplete automatic context.'
- id: evidence
  title: Materialize selected diagnostics from one frozen result
  size: medium
  depends_on:
  - capture
  description: 'evidence: feed frozen results and bounded stage diagnostics through
    every projection with configured UTF-8 limits and complete command identity.'
- id: policies
  title: Execute validated frozen outcome policies
  size: medium
  depends_on:
  - capture
  description: 'policies: validate and persist full branch policies before startup,
    apply overrides consistently, and preserve stopped/lost cancellation.'
- id: adoption
  title: Reserve and adopt ordinary continuation deliveries
  size: medium
  depends_on:
  - replay
  - evidence
  - policies
  description: 'adoption: add Rust-owned delivery transitions and exact receiver adoption
    through existing launch admission before provider invocation.'
- id: recovery
  title: Reconcile terminal delivery and implement manual resume
  size: medium
  depends_on:
  - adoption
  description: 'recovery: recover terminal monitors without rerunning commands and
    atomically admit immutable manual-resume revisions with explicit ownership outcomes.'
- id: completion
  title: Repair real host-finalizer recovery and receipts
  size: medium
  depends_on:
  - recovery
  description: 'completion: repair recovery launcher arguments, serialize host adoption,
    and verify all finalizer actions with real controller and multi-repository receipt
    coverage.'
- id: budgets
  title: Apply checkpoint projections and provider-aware budgets
  size: medium
  depends_on:
  - replay
  - evidence
  - recovery
  description: 'budgets: select and render safe checkpoint reductions, use actual
    provider and transport limits, and publish recoverable refusals through delivery
    state.'
- id: experience
  title: Complete monitor controls and visual contracts
  size: medium
  depends_on:
  - completion
  - budgets
  description: 'experience: expose working recovery and evidence controls, synchronize
    docs and skill templates, and inspect narrow/wide monitor-state snapshots.'
- id: acceptance
  title: Prove the complete route and compatibility rollout
  size: medium
  depends_on:
  - experience
  description: 'acceptance: run production-path crash and efficiency evaluations,
    finish the approved flag rollout, and verify coordinated released Rust and Python
    delivery.'
proposed_by: bbugyi200.athena.sase-zl.land
create_time: 2026-09-11 23:43:23
status: wip
bead_id: sase-zl.13
---

- **PROMPT:** [prompts/202609/monitor_continuation_landing_repairs.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/monitor_continuation_landing_repairs.md)
- **PARENT:** [202609/monitor_continuations.md](https://github.com/sase-org/sase--plans/blob/main/202609/monitor_continuations.md)
- **BEAD:** [sase-zl.13](https://github.com/sase-org/sase--beads/blob/main/pages/sase-zl/sase-zl.13.md)

# Finish the remaining monitor continuation work

This is a remaining-work child of `sase-zl`, whose twelve closed phases did not complete
the approved production contracts. The governing requirements remain
`plan:202609/monitor_continuations.md`. Read the audited landing evidence
`file:explicit:b356c51cb45677f60961e909`; it identifies actual code paths, executed
probes, reviewed commits and all twenty proposed-follow-up dispositions. Preserve those
outcomes in the later landing review.

Primary inspection was at `9202146ca`, core at `acab7b4`. Open core using the
`sase_repo` skill; if configured-name opening still hits existing task `sase-zo`, use
`sase repo open gh:sase-org/sase-core -r '<reason>'` and only its printed path. Paths
beginning `crates/` below are relative to that opened repo; other code paths are
relative to SASE. Recheck intervening commits before implementation.

The existing Rust schema/replay/evidence/budget/conditional-completion code, detached
supervisor, stage diagnostic capture, CLI result layout and Python package splits are
useful foundations. Extend them. Do not recreate an entire protocol or scheduler. Shared
validation, identity/digest rules, graph ordering, policies, delivery transitions,
checkpoint selection and completion eligibility belong in Rust and PyO3 bindings. Python
owns I/O, processes and frontend projection.

No phase closes `sase-zl`, runs its post-close Symvision step, or updates the parent
plan status. The `parent_bead` relationship resumes that landing after this child lands.
Do not force either landing or interpret closed old phase beads as acceptance.

## 1. capture

`continuation_capture/segments.py::complete_prompt_segments` currently stores the whole
expanded prompt as `local_materialized`; this includes replayed ancestry. Correct the
preprocessing/workflow provenance stream before serialization. Preserve authored local
requests and materialized local xprompt/artifact/gate instructions, mark injected
ancestry separately, and exclude it from canonical local deltas without parsing Markdown
delimiters. Keep expanded prompt archives as debug artifacts only.

Resolve exact execution parents at launch, retaining their identities while a starter is
running. During monitor handoff, publish the starter delta/checkpoint and connect the
monitor result to that exact node before dispatch is eligible; the currently captured
start-time metadata can precede publication of the starter node. Cover retry, ordinary
completion, failure and interruption through the existing LoopState seams.

Add the approved `-k/--checkpoint FILE` optional monitor-start control. Parse bounded
YAML/JSON with objective, constraints, findings, unresolved decisions, remaining work,
source refs and explicit coverage. Preserve author attribution separately from host
workspace/check/artifact facts. Store the next action only in the intent. Bind the
checkpoint content digest, resolved parents and intent to the start fingerprint and
rollback transaction, not a mutable pathname.

Extend current storage rather than add a database: immutable content-addressed writes
must accept identical retries and reject conflicting replacement. Use a recoverable
publication journal for record/blob/pointer updates; persist content before pointers.
Register portable content using existing artifact mechanisms and protect active
checkpoints/evidence from cleanup. Best-effort debug capture may remain, but failure to
persist essential enabled-route context must prevent successful automatic dispatch and
produce an explicit recovery disposition.

Acceptance: hostile headings/directives remain literal; repeated handoffs do not store
inherited prompt bodies as local material; same-content retries are idempotent;
conflicting writes and injected disk failures cannot publish incomplete success.

## 2. replay

`history/chat_fork/continuation/replay.py` currently loads only the supplied source set.
`_render.py` ignores materialized segments and checkpoint bodies and replaces legacy
content with locators. Implement bounded parent hydration through immutable refs from a
single exact node. Resolve local/portable refs with digest checks and explicit
missing-source errors; do not rediscover mutable family aliases.

The current direct-child probe creates a stored base containing
`ORIGINAL_CONSTRAINT_SENTINEL` and a child naming that base. Rendering just the child
prints `missing_parent` and drops the sentinel although the base exists. Convert this to
a regression using the production capture pipeline and normal source resolver. Add a
100-handoff test that starts from only the final exact monitor-result target; prove
every constraint/decision and local event appears once and new canonical storage grows
linearly. Do not satisfy this by feeding all family members into the renderer.

Render referenced local material and checkpoints with attribution. Protect user/gate
instructions and unresolved decisions. An opaque legacy boundary must carry its
protected content within budget or refuse automatic launch; `none`/`file` must use
provenance-backed evidence-free content or report `legacy_evidence_policy_conflict`.
Missing essential parents and failed starters without checkpoints are nonlaunchable.
Keep explicit historical inspection and legacy multi-parent behavior available.

Route monitor successors from the exact frozen result, separately from family
attachment. Use Rust's stable block ordering and branch attribution; display prefix
resets/omissions when projection changes. Test shared ancestry, alias reuse, delayed
starter settlement, direct/family/manual forks and literal disabled regions.

## 3. evidence

`monitor/followup_prompt.py` currently rebuilds result facts from mutable arguments,
forks the starter, and embeds only an aggregate tail. Rust-selected diagnostic refs are
not materialized there. Consume the frozen monitor result and one centralized selection
on successor, direct, family and versioned-resume paths. Include structured stage/count
summaries on success, failed-stage diagnostics on failure, and bounded fallback
progress/tail for incomplete reports or timeout.

Use the existing diagnostic sink and retained-range metadata. Read selected bounded
diagnostic artifacts outside rendering/UI code; preserve capture errors and omission
markers. Honor `auto|tail|file|none` and latest/historical aging identically everywhere.
No builder may independently append logs after policy selection. `file`/`none` must
inject zero diagnostic/raw bytes; explicit human inspection remains available.

Move the 8/4/12 KiB and 200-line defaults to typed documented configuration. Enforce
UTF-8 byte boundaries as well as lines across combined excerpts. Preserve full canonical
command facts: `_wire_command_part` currently abbreviates arguments over 1024 bytes; use
the appropriate Rust command bound or a resolvable immutable command-content ref, not a
digest marker in place of the command. Keep deliberate display abbreviation local to the
UI. Reject unsupported producer versions/path escapes, and preserve command exit status
when evidence capture fails.

Acceptance: outcome/policy/age/entry-path matrix with unique sentinels; diagnostics
survive aggregate log rotation; hostile output is literal; Unicode limits are exact;
long commands and required diagnostics remain retrievable without repeated payloads.

## 4. policies

`main/monitor_handler.py::_policy_file_digest` hashes an arbitrary object but discards
its policy; `StartMonitorRequest` carries only the digest. Replace this with the
versioned Rust policy contract, complete validation before claim changes, immutable
resolved policy persistence, and a fingerprint covering its content.

Implement CLI override > policy/profile > shared next/model > inherited route
precedence, including effort and evidence controls. Ordinary terminal settlement must
execute the frozen branch. Reject incomplete/contradictory actions and simultaneous
profile/policy. `complete` is success-only and requires a bound prepared intent.
`verify` alone never authorizes completion. Omitted `--next` stays fire-and-forget
unless policy explicitly requests continuation or completion; attaching completion
provides the approved recovery action for failure/timeout. Stopped/lost never dispatch
automatically even if a custom policy requests it. Preserve historical tail decoding.

Acceptance must exercise the CLI parser through start and actual settlement using fake
effects, including a policy selecting `none` despite shared next, a per-branch route,
invalid JSON/YAML shapes, changed-file retries, cancellation and legacy records.

## 5. adoption

There is currently no ordinary continuation delivery consumer: `delivery.py` is used
only by host completion, and `followup.py` spawns before recording the launched name.
Implement the missing Rust transition contract and use the existing typed launch
admission/journal to reserve an exact successor before spawn. Key delivery by monitor,
immutable result and branch, and atomically adopt the same key in the receiving runner
**before provider invocation**. Repeated/concurrent dispatch must discover the reserved
receiver, not allocate a second identity.

Carry exact parent context and weighted capacity/priority/weight, workspace and remote
targeting through the current launch wires. Preserve continuous workspace/family-slot
ownership until acknowledgment or durable terminal disposition. Keep normal fallback
claims with visible workspace provenance; host completion must require the verified
original workspace. Use bounded locks with a documented order and no waits inside store
locks. Put pure transitions and validation in Rust, process effects in Python.

Acceptance: deterministic failures before/after reservation, spawn, receiver adoption
and acknowledgment; concurrent dispatch; at most one fake-provider invocation; no
claim/slot release gap; existing admission/remote-target/capacity contracts still pass.

## 6. recovery

Extend reconciliation beyond `monitor_state == running`. Terminal monitors with
pending/reserved/dispatching delivery must recover from their existing result/receiver
without rerunning the command. Integrate both proc-owned and dead-supervisor paths.
Serialize stop versus reservation/adoption: stop before reservation cancels; after
acknowledgment it reports the existing handoff without killing the successor. Reboot
lost state remains inspection-only; ambiguous ownership refuses another launch.

Implement `sase monitor resume ID [-k FILE] [-m MODEL] [-j]`. Reconcile first, return an
acknowledged successor identity on repeats, and otherwise resume only a conclusively
undelivered requested action. A changed checkpoint/model creates an immutable intent
revision and numbered manual-recovery branch, admitted atomically against the old
disposition. Concurrent resumes cannot spawn twice. Reject stopped/lost,
fire-and-forget, ambiguous ownership and attempts to retry conditional finalization. Do
not rerun the monitored command. Show the precise eligible resume command in errors and
details.

Acceptance: terminal-supervisor crash, simultaneous reconcilers/resumes, stop races,
duplicate successful resume, intent revisions and refusal paths with explicit durable
reasons and ownership checks.

## 7. completion

Fix the confirmed `_recover` integration error: it supplies only artifacts/meta to the
real launcher, omitting monitor_state, exit_code, elapsed_seconds, capture and
project_name. Route recovery through the newly durable continuation branch, carrying
complete frozen result/context. Do not adapt tests to an artificial two-argument
callback that no production caller implements.

Host delivery must atomically adopt under original verified workspace ownership.
Recompute obligations, authenticated finalizer plan and stable repository fingerprints
at bind and immediately before execution. Use a dedicated validated conditional context;
verify the real declaration submission path rather than stubbing it. Preserve explicit
bead-action requirements if intervening sase-zq work has landed.

Use existing controller and repository/finalizer ledgers. `_commit_succeeded` currently
checks whether _any_ receipt is successful; recover all required repository actions and
later finalizers independently. Never skip outstanding work because one repository
committed, and never retry an ambiguous external action. Detect relevant mutations or
new obligations caused during finalization and transfer to recovery instead of publish
success under stale verification. Publish the prepared message and actual family/task
completion only after all required actions succeed, without fake LLM usage.

Acceptance: real controller/declaration adapters over temporary repositories and fake
effect executors; valid complete success uses zero model calls; stale/missing checks,
missing/unsupported finalizers and tree drift use one recovery; two-repository partial
success and crashes after each action retain receipts and never falsely complete.

## 8. budgets

`llm_provider/continuation_budget.py::_budget_request` currently calls every prompt byte
essential and supplies no reductions. Provider/model/effort are only telemetry. Build
the budget request from the replay manifest and protected local/user/gate blocks.
Provide real reduction candidates and apply Rust's selected projection to the prompt; do
not treat a `compact` decision as permission to invoke with unchanged bytes.

Use actual selected provider/model capacity and transport contracts, reserves and
versioned uncertain estimates. Validate persistent config defaults; account for
argv/environment bounds or use supported stdin/file transport. Recheck fallback routes
before invocation. Keep objective, constraints, gate decisions, unresolved work, next
action and newest result. Apply an authored checkpoint only at thresholds and only over
explicitly covered material; preserve refs and a visible omissions manifest.

Persist `context_budget_exceeded` through the adopted delivery when essential content
cannot fit. An uninvoked receiver is not successful model delivery; prevent repeated
automatic retries. Recover via `monitor resume` with a checkpoint or suitable route,
without rerunning verification. No extra summarization model invocation.

Acceptance: below-threshold prefix preservation, real compactable ancestry, oversize
protected content, larger/smaller model budgets, fallback, Unicode and argv limits,
refusal ownership, then successful checkpoint resume of the same command result.

## 9. experience

Finish CLI/help/JSON and existing ACE monitor details for the implemented policy,
checkpoint, evidence, host-completion and recovery states. Keep command outcome,
delivery and task completion distinct. Use existing folds, words plus glyphs, bounded
wrapping and exact resume guidance; do not invent unrelated keymaps. Preserve the
unified Agents list, current query/search keys and selection identity.

No UI-thread file reads, JSON decoding or subprocess work. Extend cached/off-thread
projections and refresh tokens, respecting the detail debounce and current artifact
index optimizations. Update default_config, cli/monitor/family/config/finalizer docs and
`src/sase/xprompts/skills/{sase_monitor,sase_final}.md` as needed. Preview generated
skills only; deploy after clean canonical landing through the normal managed workflow.

Add and visually inspect 90x40 and 120x40 running, host-completed, failed-diagnostics,
timeout, lost, degraded and needs-attention snapshots. Exercise keyboard folds, no-color
output, long Unicode and narrow layouts. Read TUI performance memory and capture
prescribed navigation/idle-refresh evidence; preserve the original p95 and
no-repeated-idle-read acceptance targets.

## 10. acceptance

Run the integrated production capture -> command -> freeze -> dispatch -> adopt ->
provider/host-completion path with deterministic providers and injected crashes. Keep
the detached supervisor's existing acknowledgment, kill-group, timeout, stop and claim
tests. In particular prove direct final-node 100-handoff ancestry and storage growth,
every output policy, legacy protection, one successor per delivery, budget recovery,
partial finalization and real stale-success recovery. Cover launch/gate decisions and
weighted admission added since the original epic started. Do not implement independent
command reservation/monitor-weight work owned by the blocked sase-zm.5.

Complete the originally approved `monitor_continuation_records` sunset rollout through
`sase flag new` and its authored policy sentences. Both states must have explicit
coverage; disabling affects new writer/launcher selection while existing v1 records
retain their immutable semantics. Do not expose newly unfinished controls in earlier
phases without the mandated epic beta scaffold. Remove any beta scaffold before this
child lands; retain or retire the sunset only on its documented evidence and caller
migration conditions, using the feature-flag workflow.

Create an indexed evaluation artifact with reproducible commands and corpus bounds.
Compare normalization, diagnostics, evidence, branching and checkpoints separately;
record prompt bytes, retrieval volume, successor count, elapsed time, correctly
completed work, and provider/cache/reasoning usage where available (otherwise unknown).
Cover success, test/lint/type/validation failure, timeout, long chains and legacy
boundaries. State a measured go/no-go on later provider-session experiments without
making sessions a dependency. Do not substitute synthetic full-family rendering or
skipped installed-plugin tests for the production acceptance matrix.

Coordinate released core/bindings before Python requires new APIs; release-plz owns
versions. Verify a local built wheel and the actual published compatible floor in a
clean environment. Current primary requires >=0.34.15; preserve newer launch, queue,
agent-scan and Grok bindings, plus the core long-command fix at acab7b4. Rebase the
integration evidence on changes landing while this repair plan runs.

Every phase changing tracked SASE files must run `just check`; core changes require its
full `just check` including PyO3 tests with Python >=3.12. Use sase_monitor for long
checks. Before this child lands, complete `just check-full` through sase_monitor with
TESTING/TESTED and a failure-handling continuation, plus the dedicated visual
verification for changed goldens. Reproduce and triage concrete failures from the three
historical broad-suite proposals in the audit; do not claim them resolved from lint-only
checks. Child phase workers record discoveries as PROPOSED FOLLOW-UP notes. Its land
agent owns normal child landing and the linked parent landing resumption.
