---
tier: epic
title: Stateless release steward with an observe-only workflow
goal: Complete sase-1n6.7 with a restart-safe release reconciler, durable escalation,
  and a verified observe-only GitHub Actions host.
parent_bead: sase-1n6.7
phases:
- id: policy
  title: Typed release decisions in the Rust core
  size: medium
  depends_on: []
  description: 'policy: define the fact and decision wire contracts and implement
    the pure release state machine, daily cadence, bounded recovery, and escalation
    policy in sase-core.'
- id: train-facts
  title: Read release trains and publication evidence
  size: medium
  depends_on:
  - policy
  description: 'train-facts: read PyPI, tags, draft releases, manifests, release PR
    checks, and workflow history; reuse the candidate evaluator and expose a reusable
    read-only reader.'
- id: health-facts
  title: Read core, canary, and escalation evidence
  size: medium
  depends_on:
  - policy
  - train-facts
  description: 'health-facts: compose core-ensure, typed core-cut outcomes, scheduled
    install-canary results, and issue history into the complete per-tick snapshot.'
- id: effects
  title: Execute idempotent train and escalation actions
  size: medium
  depends_on:
  - policy
  - train-facts
  - health-facts
  description: 'effects: add precondition-checked action execution and issue sinks,
    proving one train action and one escalation operation per tick across restarts.'
- id: cli-host
  title: Replace the watcher CLI and install the observe-only host
  size: medium
  depends_on:
  - policy
  - train-facts
  - health-facts
  - effects
  description: 'cli-host: switch release_watch to the new reconciler, migrate legacy
    tests, and add the scheduled observe-only workflow with pinned Rust runtime and
    structured output.'
proposed_by: bbugyi200.athena.sase-1n6.7
proposed_with: codex/gpt-6-astra
create_time: 2026-10-19 19:56:20
status: wip
bead_id: sase-1n6.7.1
---

- **PROMPT:** [prompts/202610/release_steward_observe.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/release_steward_observe.md)
- **PARENT:** [202610/e2_release_train_steward.md](https://github.com/sase-org/sase--plans/blob/main/202610/e2_release_train_steward.md)
- **BEAD:** [sase-1n6.7.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1n6/sase-1n6.7.1.md)

# Stateless release steward

This is a child epic of `sase-1n6.7`, implementing the `steward` section of
`plan:202610/e2_release_train_steward.md`. It is deliberately split into five bounded
coding phases. The existing implementation has about 2,800 lines, with process-local
retry sets, dispatch timestamps, target-version polling, install proofs, and
bead-closing finalization. Replacing that implementation, adding the missing remote
facts and mutation paths, and deploying the host is too much for one direct
implementation task.

The phase was `in_progress` with no earlier-work notes when inspected. Its three
prerequisites are closed, and it had no `--epic-symbol` entries. The current tree
already has `tools/_release_core.py`, `tools/release_candidate`, and the initial
`release.yml`. The candidate evaluator is still deliberately fail-closed until the
qualification and publication phases fill in their evidence. Build against the landed
version of those interfaces, without duplicating their policy.

Read the parent plan and
`research:202610/pypi_release_reliability_design_and_work_split/pypi_release_reliability_design_and_work_split.md`
through `sase artifact read`. Relevant design sections are “The releasable promise,” “E2
Exact-candidate release train and steward,” “File ownership,” and acceptance cases 2–5,
8–10, and 14. The parent plan's finalized choices supersede older research suggestions
about the host and release cadence.

## Fixed scope and ownership

- The host is a GitHub Actions workflow. Scheduled ticks are observe-only. The daily
  cadence allows merges after 09:00 UTC and at most one release PR merge per UTC day,
  except when that day's candidate was abandoned.
- This child epic changes no memory notes. The parent epic's authorized memory work
  remains with its owner. It changes no chezmoi settings, PyPI settings, repository
  variables, rulesets, tags, versions, or trusted publishers.
- Keep `tools/release_watch` as the entry point. Python owns transport, CLI,
  orchestration, and output. New shared state, eligibility, precedence, idempotency-key,
  and escalation policy belongs in the `sase_core` crate and is called through
  `sase_core_rs`; no Python policy fallback.
- Open `sase-core` with `/sase_repo`, then use only the printed checkout. Read its
  `AGENTS.md`. Its new module is `crates/sase_core/src/release_steward/`, with a binding
  domain under `crates/sase_core_py/src/release_steward/`. Existing
  `_release_core.ensure_core` and `release_candidate evaluate` remain the single owners
  of their already-approved rules. Feed their results to the new policy. Do not port or
  fork either function in this child epic.
- Each phase owns its named new files. Only `cli-host` replaces the existing
  `_release_watch.py`, `_release_watch_api.py`, `_release_watch_engine.py`, and legacy
  tests. Serial dependencies avoid concurrent edits to shared helpers.
- Do not edit `release.yml`, `publish.yml`, `ci.yml`, `build-core.yml`,
  `install-smoke.yml`, or E1's tools. Qualification/publication owners may still be
  changing their contracts. Consume their final contract when available; absent required
  evidence produces a refusal, never an optimistic pass.
- Help and docstrings document this phase. Documentation pages, including old watcher
  examples in `docs/contribute/rust-backend.md`, belong to `sase-1n6.9` handover. Record
  the obsolete examples in the completion evidence for it.
- `sase-1n6.8` will reuse the fact reader. Expose documented read-only methods for runs,
  releases, tags, and publication history, with explicit time bounds and pagination.
  Implement no release metrics here.
- Phase workers close only their assigned child phase, never `sase-1n6.7` or `sase-1n6`.
  The child epic's land agent owns completion of `sase-1n6.7`, after its own children
  and child epic are complete. The outer epic remains open.

## Shared contract

### Facts and result

The pure Rust entry point is `release_steward_decide(request)`. Define versioned serde
request/result wires and register the PyO3 binding with a real round-trip test. The
request contains the observation time in UTC, explicit runtime policy inputs, and a
complete snapshot. Represent absent evidence separately from an unreadable API response.
The latter cannot authorize an action.

The snapshot carries:

1. Repository and master identity; Master Gate runs at that exact SHA, including jobs,
   attempts, and timestamps; expected required PR checks and their source.
2. Release PR identity, head, source pin, exact public core pin, labels, checks, merge
   history, and stale-head/pin evidence. A changed head is a new identity.
3. Tags and peeled SHAs; draft/non-draft releases; immutable manifests and observation
   assets by candidate ID and generation; `release.yml` and `publish.yml` runs with
   workflow, ref, SHA, attempt, jobs, and timestamps.
4. PyPI file inventories for sase and core, including yanks and SHA-256 digests;
   core-ensure's pick/lag result; candidate evaluator results and refusal clauses.
5. Core master CI, open release PR, core-cut runs and typed failure evidence; scheduled
   canary results and package/platform identity.
6. Existing escalation issues and their machine-readable identity markers, comments,
   recovery history, and timestamps.

Every tick reconstructs these facts. Within-tick request deduplication is fine; no
success, cooldown, retry count, or blocked-since value survives in Python object state.
Sort remote collections deterministically. Never infer completeness from a first API
page or infer green from an empty check set.

The result has a schema version, one primary typed state with details, evidence
URLs/identities, blocker timestamps, at most one proposed train action, and at most one
proposed escalation operation. Include secondary findings so an install incident and a
safe recovery candidate can coexist. Proposed actions include stable keys and exact
preconditions. Output distinguishes proposed, suppressed, already-satisfied, and
executed actions; observe output never says an action occurred. `WATCHER_BROKEN`
includes the failing source and remedy and has no executable action.

### Policy and precedence

Use this order, keeping the primary state and train action separable from
install-canary/advisory findings:

1. Invalid, unknown, inconsistent, or unreadable required facts yield `WATCHER_BROKEN`,
   no remote writes, and a non-zero process exit. A known red canary, including its
   contracted missing-artifact result, is an install finding, not a general transport
   success or a promotion veto.
2. For the selected in-flight candidate, a PyPI digest/extra-file conflict is
   `PUBLISH_CONFLICT`; an absent, yanked, or incomplete exact public dependency is
   `DEPENDENCY_UNPUBLISHED(pkg)`; a release-hold label or active candidate veto is
   `VETOED`. These prohibit train publication actions.
3. Reconcile the newest generation of an existing train before starting another. A
   train's latest generation remains active until confirmed or abandoned, including when
   a partial upload has already made its version visible on PyPI. Discover newer tags
   relative to the last confirmed release, accounting explicitly for fully published
   pre-train releases without manifests; do not mistake every historical legacy tag for
   an unfinished train. Tag/manifest/ evidence mismatches fail closed; a previous
   generation's pass never qualifies a rebuilt generation. An ambiguous set of active
   trains prevents another merge and names the facts.
4. For a matching, publishable retained candidate with missing PyPI files,
   `PUBLISH_PARTIAL` proposes `release.yml` in `resume` mode. Partial matching files
   never relax the evaluator's full promotion predicate. A release run already
   pending/in flight suppresses dispatch. A confirmed, matching public release yields
   `RELEASED(v)`. Public files without confirmation remain work for the release
   workflow, rather than fabricated success.
5. `QUALIFY_RED(signature, recovered?)` and `EXERCISE_RED(platform)` propose one
   failed-job rerun only when the shared evaluator says `rerun_eligible` and the
   recorded attempt permits it. Otherwise propose abandonment only when the version has
   zero PyPI files. A missing retained asset, a workflow broken at its frozen tag, or
   exhausted unrecoverable evidence burns an unpublished candidate; no tag is moved or
   rebuilt automatically. If any public file exists, preserve the candidate and escalate
   instead of abandoning it.
6. With no train to advance, core CI/cut failures yield `CORE_CI_RED` or
   `CORE_CUT_RED(check)`. The core-cut workflow owns its one retry; the steward must not
   multiply it. Otherwise `CORE_LAG(pin)` proposes one core cut when no cut or
   applicable earlier attempt already covers the pin. A red published-core PR smoke
   yields `CORE_INCOMPATIBLE`: propose a metadata refresh once for the current PR
   identity, then escalate if unchanged.
7. `GATE_RED(sha)` proposes one rerun of failed Master Gate test jobs at that HEAD only;
   never rerun deterministic lint failures, cancelled or absent checks, or an exhausted
   attempt. Do not implement E4's revert behavior.
8. A stale/missing release PR or changed exact pin may need `publish.yml` to refresh
   release-please. A merged PR without its tag/draft needs the same workflow. Use its
   normal metadata mode, never `publish_existing`. Existing applicable runs and durable
   timestamps suppress repeat dispatch.
9. `RELEASABLE` can propose merge only if authority is enabled, no train is in flight,
   Master Gate is green at HEAD, every required PR check is exactly successful, the PR
   pin equals core-ensure's complete pick, there is no veto, and daily cadence permits
   it. A tag/draft ready for its initial train run instead proposes
   `release.yml --ref <tag> -f mode=release`.
10. Otherwise report `NOTHING_TO_RELEASE`, or the applicable typed waiting state with a
    concrete reason and no action. A red public install adds `INSTALL_BROKEN(member)`
    and immediate severity-1 escalation. It never removes an independently safe
    recovery-candidate action; if no higher priority train state applies it can be the
    primary state.

Required evidence accepts `success` only: skipped, neutral, cancelled, stale,
zero-collected, and missing results are not green. Consume qualification's N1 policy
through the candidate evaluator, rather than copying a second inventory. For PR checks,
determine the expected set from authoritative requirements and applicable workflow
contracts; an unreadable/unknown expected set cannot merge.

The selected daily policy has boundary fixtures for 08:59/09:00 UTC, UTC midnight,
prior-day merges, same-day merges, abandoned same-day candidates, and any remaining
active train. Do not add the rejected `eligible` cadence or chop host.

### Idempotency and escalation

Keys are derived from remote identity: merge uses PR number plus head SHA;
release/resume uses tag, SHA, candidate generation and matching run/evidence; rerun uses
run ID and the initial attempt; core cut uses source pin and core cut history; metadata
refresh uses PR head/pin or the merged PR; abandonment uses candidate ID/generation and
its deterministic observation asset.

Before executing a mutation, re-read its critical facts. If they changed, return a
suppressed action and let the next tick recompute. Use argv arrays, never shell
interpolation. Do not retry a mutation after an uncertain acknowledgement; surface the
uncertainty and reconcile remote facts before any later attempt. Treat a
pending/queued/waiting run as in flight. The workflow's single no-cancel concurrency
group serializes live ticks; no local file is an authoritative lock or action ledger.

Escalation is one issue per state family, with a fixed title and the `release-steward`
label; advisory qualification issues also use `ci`. Store machine-readable identity,
candidate/pin/SHA, signature, and evidence timestamps in issue/comment markers. Derive
blocked duration from the current continuous fact lineage, resetting it after recovery
or a changed identity. Never use the age of an unrelated old failure or the process
start time. If history cannot prove duration, report that uncertainty rather than
inventing 24 hours.

At 24 hours, open once, then comment as +1 on later distinct ticks; close on recovery.
`INSTALL_BROKEN`, `PUBLISH_CONFLICT`, and `DEPENDENCY_UNPUBLISHED` escalate immediately.
Advisory qualification failures produce one `ci` escalation per candidate without
blocking publication. Use a tick identity (workflow run/attempt or observation identity)
to avoid duplicate comments during re-execution. Prioritize immediate escalation, then
aged blocking escalation, then recovery/advisory bookkeeping deterministically; deferred
operations remain discoverable next tick. Observe mode proposes issue operations but
creates no issue, label, comment, bead, or notification. Live code never launches repair
agents. The issue mirror owns task-bead creation.

## policy

**Owner:** new Rust domain, binding registration, wire fixtures/tests, and a thin Python
`_release_watch_decide.py` adapter plus binding integration test. Update the source pin
through the host's paired-repository completion, or the supported ratchet for an
already-landed core commit. Do not edit core package versions or sase's public exact
dependency pin to make a source binding available.

Implement the shared contract above as free functions over serde wires. The clock is an
explicit input. Keep side effects, HTTP, `gh`, and sleeping out of Rust policy. The
Python adapter requires the binding and maps a missing/stale extension to
`WATCHER_BROKEN`; it does not calculate a substitute verdict.

Fixtures must cover every state, simultaneous-condition precedence, strict required
checks, multiple generations/trains, all daily cadence boundaries, core lag and
in-flight cuts, a spent retry, public-file abandonment refusal, 24-hour boundaries and
recovery, advisory de-duplication, and install-red plus a publishable recovery
candidate. Supply complete fixture snapshots that the transport and action phases can
reuse, with unambiguous error/absent shapes.

**Done:** Rust unit tests and real PyO3 round-trip tests pass, and the Python adapter
consumes the registered binding. The source revision pin contains the binding's landed
commit before a Python consumer is finalized. Leave the old CLI operational until
`cli-host` switches it.

## train-facts

**Owner:** new `tools/_release_watch_transport.py`, `tools/_release_watch_facts.py`, and
dedicated reader/fixture tests.

Build an injectable, read-only transport and train reader. Reuse robust existing `gh`
environment and timeout handling where appropriate, but do not import old policy such as
`GOOD_CONCLUSIONS` (which currently treats skipped as good). Implement pagination,
bounded timeouts, validated timestamps, annotated tag peeling, drafts, assets, and
JSON/API error attribution. Use read-only API calls for all source content; for local
agent research, follow `/sase_repo` rules.

Collect the release PR and expected checks at its exact head, master gate at master
HEAD, merge history for cadence, tags/releases, and all relevant workflow attempts/jobs.
Match runs using workflow identity, ref and SHA, not newest-run position alone.
Distinguish absence of a newly created workflow on a historical tag from a missing
workflow on the expected current branch.

Read full PyPI inventories and candidate assets. Invoke the existing candidate CLI's
`evaluate`/`verify` interface through a small adapter, validate its output, and preserve
clauses, candidate identity, generation and `rerun_eligible`. Required-evidence-missing
is a valid refusal; malformed output is an error. Keep candidate evaluator integration
localized because later E2 phases extend it. Fetch retained bytes when the evaluator
requires hashing; never turn a seal-only placeholder into permission to publish. The
owner of the evaluator fixes any missing promotion clause; record such integration
issues on the assigned bead.

Expose reusable read-only history methods for the metrics phase, independent of the CLI
and executor. Fact parsing performs shape/transport conversion; shared eligibility
decisions stay in Rust.

**Done:** offline transport fixtures prove pagination, empty versus failed reads,
head/tag/generation mismatch rejection, stale attempts, matching and conflicting PyPI
files, and strict candidate evaluator handling. No test uses live writes.

## health-facts

**Owner:** new `tools/_release_watch_health.py`, complete snapshot composition in
`_release_watch_facts.py`, and health/source-contract tests. Reuse the transport from
`train-facts`.

Call `_release_core.ensure_core` using freshly read PyPI metadata and the source pin
read at the relevant candidate, PR head, or master SHA. Do not substitute the local
checkout's pin when those identities differ. Preserve `Pinned`, `CoreLag`, and errors.
Never maintain a copy of `EXPECTED_DIST_SUFFIXES`: the shared helper reads it at the
core version's tag. Preserve its complete-file and source-containment requirements. Read
core master CI, the open core release PR and cut runs.

Consume core-cut failure evidence from the actual supported API surface.
`release_plz_merge.py` writes a typed annotation/summary plus job outputs; arbitrary job
outputs are not fields in the Actions runs API. Read the relevant check annotations or
retained log line, bind it to run/attempt/job, and parse
`CORE_CUT_RED check=... run=... attempt=...` and `CHECKS_NEVER_REGISTERED` without
guessing from a generic failure. A failed cut with unavailable required diagnostic
evidence fails closed.

Read the newest scheduled `install-smoke.yml` results, not arbitrary manual probe runs.
E1's artifacts are `install-canary-<os>` containing schema-v1 JSON with run ID, attempt,
SHA, verdict, packages, failing step, and reason. Require the expected Linux and macOS
results. Missing/corrupt/expired contracted results are install-health findings with
evidence, not a pass; preserve the distinction from an authentication or API outage.
Never edit the E1 producer.

Read existing steward issues/comments including closed history needed to prove recovery
and prevent duplicate incidents. Compose a fresh complete snapshot per tick, including
historical facts sufficient to measure continuous blocked duration. Add a read-only core
Actions-access probe using the same credential as the host. A failed probe is
`WATCHER_BROKEN`. A successful read proves only read access: do not claim it proves
workflow-dispatch write permission. Missing or uncertain dispatch scope is explicit
handover evidence for the human gate; never use a production dispatch just to test
permissions.

**Done:** complete fixture snapshots feed the real Rust binding. Tests cover core
completeness/lag/errors, typed cut outcomes, stale and missing canary artifacts, manual
probe exclusion, issue recovery lineage, and permission errors. A read-only live fact
collection can be used for debugging; it cannot mutate production or satisfy the later
scheduled-tick proof by itself.

## effects

**Owner:** new `tools/_release_watch_actions.py`, `tools/_release_watch_sinks.py`, and
executor/restart tests.

Implement exactly the action variants selected by Rust: head-matched
`gh pr merge --merge --match-head-commit`, metadata dispatch, tagged release or resume
dispatch, one failed-job rerun, core cut with `dry_run=false`, and abandonment. No
publisher upload or fresh dist build belongs in this executor. Apply `live` at a single
outer mutation boundary so read-only modes cannot leak issue or label writes.

Abandonment is a resumable multi-step logical action: recheck zero PyPI files, append
the generation-specific `abandoned` observation using the established candidate
observation contract, and prefix the draft title once. If killed between writes, a later
tick repairs the missing title without rewriting a seal or adding another observation.
Do not use clobber to replace an immutable asset.

Implement the issue sink using fixed titles, identity markers, labels, comments, and
closure. Never close an unrelated issue that merely shares a title. Local
stdout/JSON/step-summary writes are separate from remote escalation mutations.

For every action, tests run tick one, apply the resulting remote fact changes, discard
all process objects, and run tick two. Assert no duplicate merge, dispatch, rerun,
abandonment asset, issue, or comment. Also cover termination after each abandonment
substep, a changed PR head before merge, files appearing before abandonment, pending
runs, uncertain acknowledgements, and two eligible actions competing in one tick.
Read-only runs issue zero mutating subprocesses.

**Done:** fixture integration proves at most one train action and one escalation
operation, with revalidation and recovery. Production mutations remain off.

## cli-host

**Owner:** `tools/release_watch`, existing `_release_watch*.py` entry/legacy modules,
legacy watcher tests, new CLI/workflow contract tests, and the new
`.github/workflows/release-steward.yml`.

Replace the old watcher with one read/decide/execute/report tick. Delete the unused
stateful engine, cooldown sets, install proof, ci_watch file reads, and bead-closing
behavior. Preserve useful transport fixture helpers by migrating them; remove obsolete
imports/re-export aggregators as their tests disappear.

CLI contract, with alphabetized help and short aliases:

- `--once/-o`: one read-only decision; the safe default when no mode is selected.
- `--tick/-t`: one decision; `--live/-l` permits its actions. `--live` is valid only
  with `--tick`.
- `--json/-j`: machine-readable result only on stdout; diagnostics go to stderr.
- `--until-released/-u [VERSION] --deadline/-d D`: repeat read-only snapshots until that
  exact version is confirmed, or until the active/new target is confirmed when omitted.
  Define the omitted-target baseline in help, so an unrelated historical release does
  not end the wait. A monitor may invoke it.
- `--repo/-r` and `--poll/-p`, if retained, have explicit safe defaults and validated
  inputs. Mutually exclusive modes reject contradictory arguments.

Drop `--close-bead`, `--fixed-sha`, their lifecycle semantics, and all old retry knobs.
Do not silently reinterpret removed flags. Exit zero for every well-formed state,
including blocked states; non-zero for `WATCHER_BROKEN`, bad arguments, or deadline
expiry. A fatal execute error must also produce the typed failure, rather than printing
success for the planned action. Ctrl-C stops polling cleanly and never closes a bead.

The new workflow has schedule `*/15 * * * *`, manual dispatch with boolean `live`
defaulting false, concurrency group `release-steward`, and `cancel-in-progress: false`.
It checks out master. Schedule always calls `--tick` without `--live`; only an
explicitly true manual input may pass it. Keep merge authority separately disabled until
the `sase-1n6.9` handover sets the repository variable `RELEASE_STEWARD_MERGE_ENABLED`
to the literal `true`. Pass that variable as an explicit policy input; missing or other
values mean disabled. Do not create/change that variable in this phase. Use
`SASE_RELEASE_TOKEN` for GitHub reads/actions and append the typed report to
`GITHUB_STEP_SUMMARY`, including failures. Never print credentials.

The policy binding must be available even before its core version is on PyPI. Within
this new workflow, resolve `sase-core-revision.txt` from the checked-out master, check
out that exact core SHA, and use the established maturin build invocation to
build/install its wheel into an isolated Python environment with `packaging`. Cache the
resulting wheel only under an exact source SHA, platform/ABI, and build-contract key.
Install no whole sase application merely to run the standalone tool. The thin adapter
can import `sase_core_rs` directly and fail closed when its binding is missing. Do not
assume the currently published exact dependency already exports this new binding or
introduce a Python fallback. Keep these setup steps inside the new workflow, without
changing shared workflow ownership.

Contract tests assert schedule versus manual mutation behavior, master checkout, exact
core pin, credential routing, no-cancel concurrency, step summary, and the disabled
merge-authority default. CLI tests invoke the entry point against fixture
transport/binding adapters, including a real binding round trip, unknown flags, mode
conflicts, JSON purity, deadline, and fatal errors.

**Done:** the replacement CLI passes its tests and a live read-only `--once --json`
prints a typed state plus evidence. Record its output as an artifact or bead note.
Record permission limitations for handover. The workflow is ready to land in
observe-only mode; scheduled proof belongs to the land agent below.

## Verification and land-agent responsibilities

Every coding phase reads `lint_and_test.md` before finishing and runs the required
`sase tool run check` in each changed repo. In sase-core follow its `AGENTS.md`: no bare
cargo, include the binding tests, use `sase tool run check` as the final gate. Use
`/sase_monitor` for long final checks, with the required handoff and completion
discipline. Run formatting first. Do not run `check-full` merely because this is a child
epic.

Use focused fixtures for iteration. Where new tests belong in the contract manifest,
refresh it with `just refresh-contract-manifest` and honor its cost budget. Do not rerun
broad suites without a change or unresolved failure that warrants them. A failure
reproduced identically on the clean base is recorded as `PROPOSED FOLLOW-UP:` on the
assigned phase with both pieces of evidence and an existing task ID when known; it does
not leave that phase open. Do not assume a failure is pre-existing just because
prerequisites reported it.

Before each phase closure run `sase bead epic-symbols <assigned-phase>`, resolve every
entry or re-key it to a still-open owner, then close only that phase with verification
evidence. Discovered work is a `PROPOSED FOLLOW-UP:` note, never a new bead created by a
phase worker.

The child epic's land agent must:

1. Verify all five phases, recheck the final candidate/core interfaces against landed E2
   work, and confirm the pin exposes the binding. Run the required repository checks and
   collect the restart, install-recovery, daily-cadence, read-only, and escalation
   acceptance evidence.
2. Arrange host-owned landing of the observe workflow. No agent manually creates a
   commit, branch, PR, or release tag. Run a read-only `--once` against real GitHub/PyPI
   on the landed tree and retain the evidence.
3. Observe two actual scheduled runs of the deployed workflow, record their URLs, and
   inspect typed step summaries and the absence of remote writes. If landing or the
   schedule has not happened yet, use `/sase_monitor` for the wait and follow-up; do not
   end with a promise to resume or replace scheduled proof with two fixture/manual
   ticks. Timeouts and access errors remain explicit pending acceptance work, not
   claimed completion.
4. Give outer handover `sase-1n6.9` the authority-switch name, scope probe evidence,
   observed states, scheduled URLs, reader API for metrics, and the old watcher
   documentation examples that need updating. Keep production merge authority and
   scheduled mutation disabled throughout this child epic.
5. Once its own descendants are closed and the child epic is complete, run
   `sase bead epic-symbols sase-1n6.7`, resolve/re-key leftovers, and close **only the
   delegated outer phase** with
   `sase bead close sase-1n6.7 --note "<verification and scheduled-run URLs>"`. Do not
   close `sase-1n6` or any higher ancestor. Submit the normal root final declaration as
   the last action when not handing off to a monitor.

This plan makes no claim that observation alone enables production release automation.
The parent `handover` phase owns that later cutover.
