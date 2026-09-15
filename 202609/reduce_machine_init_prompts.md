---
tier: tale
title: Prompt for machine initialization only when review is needed
goal:
  Offer machine initialization at most once per sase init batch, and after a completed
  local review offer it only for newly discovered, unreviewed machines.
size: medium
proposed_by: bbugyi200.athena.0lh
create_time: 2026-09-15 14:51:49
status: wip
---

# Plan: Reduce repeated machine initialization prompts

## Scope and outcome

Implement both requested behaviors together:

- `sase init -a/--all` offers `sase machine init` at most once for the entire
  invocation, including when the user declines or machine initialization fails. Apply
  the same rule to repeatable `-p/--project`, which uses the same coordinator.
- A completed `sase machine init` saves which discovered candidates were reviewed on
  this local machine. Later interactive onboarding offers it only when discovery returns
  a new enrollment candidate outside that saved review set.
- Reviewing includes consciously skipping candidates. Explicit `sase machine init` and
  its `sase init machine` alias remain available to reconsider them at any time.

This is one medium tale: the Rust policy, Python persistence and CLI integration are one
bounded behavior change that one coding agent can implement and test together. Do not
introduce commands, flags, a background discovery service, or a new TUI flow. Keep all
work in implementation, tests, and ordinary documentation; this plan does not call for
editing SASE memory or generated agent instructions.

## Findings and relevant code

In the primary `sase` repository:

- `src/sase/main/init_onboarding.py::run_init_onboarding_all` runs the complete
  initializer registry separately in every available project. Each call receives a fresh
  namespace from `_init_onboarding_batch.py::project_args`.
- `src/sase/main/init_registry.py` orders initializers as config, machine, memory, repo,
  skills. Config establishes local owner identity before machine enrollment.
- `src/sase/main/_init_onboarding_apply.py::run_changed_plans` owns generic
  confirmations, `--yes`, skipped/failed status, and application argument injection.
- `src/sase/main/init_machine_handler.py::plan_init_machine` currently turns the offline
  `MachineInitService.plan` offer into a changed, TTY-required action. `_init_stdin` is
  normally injected only during apply, so planning must be given the coordinator's
  actual/injected stdin when adding discovery gating.
- `src/sase/dispatch/machine_init.py::MachineInitService.plan` offers enrollment on
  every interactive non-check run with discovery providers configured, regardless of
  earlier reviews. `apply` rescans, reconciles, lists candidates, prompts for a
  selection, enrolls selected machines, and verifies activation. Nothing persists a
  review, including the successful empty-result and blank-selection paths.
- `DiscoveryCandidate` has provider, endpoint, optional installation pin, and display
  metadata. `DiscoveryResult` has candidates and diagnostics but no durable cache.
  `MachineService.discover_detailed` already runs enabled discovery providers with
  bounded timeouts. Tailnet discovery may invoke Tailscale and public health probes.
- `_machine_init_reconcile.py` already delegates classification against enrolled
  machines to `src/sase/core/machine_setup_facade.py` and Rust. Existing classification
  separates `new`, `enrolled`, and `repair` candidates.
- `src/sase/dispatch/credentials.py` provides the local locked/atomic JSON-store
  pattern. `src/sase/core/paths.py::sase_home` provides the machine-local state root and
  respects `SASE_HOME`; test writes must use the existing state-write guard.

Open `sase-core` using the `sase_repo` skill and `sase repo open sase-core -r ...`; use
only the returned path. Paths below are relative to that checkout:

- `crates/sase_core/src/machine_setup/{mod,wire,reconcile}.rs` owns the shared
  machine-setup policy; its current wire schema is 1.
- `crates/sase_core_py/src/lib.rs` exposes the machine-setup PyO3 functions and includes
  `machine_setup_bindings_round_trip_json_shapes` tests.

Shared identity matching, review-state validation/merging, and offer decisions belong in
Rust. Python owns filesystem operations, provider execution, prompts, and the
invocation-local batch context. Do not implement a Python fallback for the policy.

## Behavioral contract

### Review and offer rules

| Situation                                                                         | Onboarding behavior                                                                   |
| --------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| No usable saved review yet, discovery configured, interactive apply               | Offer initial review, preserving the initial opt-in behavior.                         |
| Saved review exists; discovery finds an unreviewed `new` candidate                | Offer machine init.                                                                   |
| Saved review exists; candidates are reviewed, already enrolled, or require repair | Do not offer enrollment; retain appropriate diagnostics and explicit repair guidance. |
| Saved review exists; discovery returns no candidates                              | Do not offer; do not erase past reviews.                                              |
| Saved review exists; discovery fails without a new candidate                      | Warn, continue other initialization, and leave review state unchanged.                |
| Some providers fail but others return unreviewed new candidates                   | Offer review for the returned candidates and retain diagnostics.                      |
| No configured discovery providers                                                 | No onboarding offer.                                                                  |
| Check/JSON mode or non-interactive onboarding, including non-interactive `--yes`  | No discovery, no machine prompt/enrollment, no review-state writes.                   |
| Explicit machine init apply                                                       | Always discover and show the current candidates, regardless of saved review.          |

`--yes` on a TTY skips the outer confirmation only when an offer is due; candidate
selection and enrollment remain interactive. It must not repeatedly run machine init in
a batch or enroll candidates automatically.

Persist a review only after machine init finishes normally with exit code zero and
without cancellation. This includes selecting none, selecting a subset, and discovering
no candidates. Save all candidates actually presented by that apply run, not merely
selected/enrolled candidates and not a prior onboarding discovery snapshot. An empty
successful discovery still records that initial review has completed.

Do not update the record for outer-prompt decline, EOF/interrupt, invalid selection,
discovery-only commands, previews, enrollment/quarantine/activation failure, or a failed
empty discovery. Partial discovery followed by successful review records only returned
candidates. Keep earlier entries so temporarily absent machines and failed or unselected
providers do not cause forgotten reviews. A successful provider-restricted explicit run
must preserve entries from other providers.

The record is an acknowledgment through the last completed review, not a fingerprint of
the current inventory. Inventory order, duplicates, display-name/detail changes, and
removal/reappearance of a previously reviewed machine must not trigger prompts.
Declining the outer offer leaves new candidates unreviewed for a future invocation.

### Candidate identity

Reuse the existing reconciliation result first: only candidates classified `new` can
trigger the follow-up enrollment offer. Existing enrollment pin-change cases continue
through deliberate repair and must never be repinned by this feature.

Define and test review matching in Rust using non-secret observations:

- A matching nonempty installation pin identifies the same reviewed installation even if
  its endpoint/display name changes, consistent with existing reconciliation.
- If a pin is unavailable on either observation, match the provider/endpoint pair, using
  the same exact endpoint convention as existing reconciliation. Built-in Tailnet
  candidates commonly have no installation pin; empty pins must never make unrelated
  candidates equal.
- Two different nonempty pins on an otherwise matching endpoint are different
  installations. An unenrolled replacement may need review; an enrolled replacement
  remains a repair case.
- Ignore display name, selector labels, details, ordering, and duplicate observations.
  When merging the same endpoint observation, preserve known pin information rather than
  replacing it with an empty hint. Unpinned endpoints inherently cannot reveal an
  installation replacement; do not infer identity from unauthenticated labels.

These are prompt-suppression observations only. They convey no enrollment trust and must
never supply or overwrite machine pins or credentials.

### State, migration, and failures

Use a versioned JSON record at `sase_home() / "fleet" / "machine_init_review.json"`,
with a completion marker and compact reviewed identity entries. Keep it local to this
user's SASE state root, shared across projects, outside synced config and chezmoi. Do
not persist bootstrap material, tokens, diagnostics, or display payloads.

Missing state means no recorded review. Pre-change installations need one catch-up
review because enrolled records cannot reconstruct which other candidates were skipped.
Do not fabricate a review from the current registry or silently mark a scan reviewed.

Validate the record through Rust. Missing/unreadable/malformed/unsupported records must
not crash unrelated initialization. Treat unusable state as requiring an initial review
and show a concise warning when a record exists but cannot be used. Reads must not
create directories or lock files. Use guarded writes, a bounded lock, read-modify-merge
under that lock, and atomic replacement so concurrent completed reviews do not discard
each other's acknowledgments. For invalid existing content, preserve it for diagnosis
before replacing it during a successful explicit review; never rewrite it from an
onboarding scan. A read/backup/write failure leaves existing state intact and emits a
warning. Metadata persistence failure must not misreport a successfully activated
enrollment as failed; explain that future init may offer review again. Do not swallow
Rust binding/schema errors as if they were optional discovery.

## Implementation steps

### 1. Add Rust review policy and binding

Add a focused module under `machine_setup` and additive wire types/functions for
assessing a review and merging a completed review. Inputs should explicitly represent
saved state, observed candidates, enrolled records, and whether initial review is
complete; return the offer decision/unreviewed candidates and normalized persisted state
as appropriate. Keep discovery and filesystem access outside the Rust policy. Use the
existing reconcile function instead of copying its classification algorithm.

Export through the Rust public module and PyO3, then add thin facade calls and typed
Python conversion where needed. Preserve existing machine-setup requests and results;
prefer additive APIs without changing their schema-1 contract. Add binding round-trip
coverage as well as Rust behavior tests. The implementation may refine function/type
names, but the rules above must remain one shared policy.

### 2. Persist successful explicit reviews

Add a small Python review-store adapter following the existing credential-store I/O
pattern, with injectable paths and test isolation. Implement all state normalization and
merge decisions through the new Rust API.

Integrate persistence in `MachineInitService.apply` at the successful terminal paths (or
one factored success finalizer). Include blank selection and empty successful results;
exclude every failed/cancelled path. Reuse this for canonical machine init, the
compatibility alias, and accepted onboarding. Keep activation, hidden bootstrap prompts,
quarantine, and repair behavior intact. Surface persistence diagnostics once through
both human output and the existing machine-init JSON diagnostics contract.

### 3. Gate interactive offers using discovery

Keep `MachineInitService.plan` and standalone check/preview calls offline. Introduce an
explicit interactive-onboarding assessment path that reads the review and, when a review
exists, calls `discover_detailed` to assess current candidates through Rust. The initial
no-record offer can remain offline until accepted. Do not attach network work to
general-purpose preview/JSON rendering.

Pass the effective stdin and explicit onboarding intent into machine planning using
copied arguments/context, before `plan_specs` runs. Only interactive non-check
onboarding may select the new path. Render the resulting current/offer/warning state
accurately; absent enrollment work must not create perpetual check drift or prevent
other initializers from running.

Reuse the existing provider timeouts. Within a batch, cache assessment observations for
an equivalent effective discovery/config context to avoid probing the same fleet per
project. Do not reuse a result across differing project provider settings or a changed
owner/config context; config init can invalidate the assessment. Before actually
prompting/running, refresh an assessment if preceding config initialization changed that
context. Preserve the initializer ordering and do not consume the batch machine
opportunity before config work succeeds.

Reassessment after config changes must also handle an originally empty machine plan:
regenerate the machine action before deciding whether to skip that registry slot. For
example, completing owner setup may select an overlay that enables discovery. Keep this
refresh narrowly scoped to the machine step rather than replanning all initializers or
treating a cached pre-config decision as authoritative.

An accepted machine init performs its normal fresh discovery; save only that run's
presented candidates. Thus a machine appearing between assessment and apply is reviewed
normally, and no unseen candidates are acknowledged by the preflight scan. Explicit
invocation bypasses onboarding suppression entirely.

### 4. Share one machine-offer opportunity per batch

Add an invocation-local context shared explicitly across project passes. It should track
whether the machine confirmation/application has been handled and hold the bounded
assessment cache. Allocate it once in `run_init_onboarding_all`; allocate a fresh
context for a single-project invocation. Avoid module-global state and avoid setting a
boolean only on namespaces that `project_args` subsequently copies.

Mark the machine opportunity handled immediately before presenting its outer
confirmation, or before applying it under `--yes`. This covers accept, decline, EOF, and
machine-init failure. Subsequent projects must neither prompt nor run the machine
initializer. Filter/suppress its apply action and repeated assessment at the coordinator
boundary so injected specs cannot bypass the batch rule. Keep check-mode planner rows
and JSON shape unchanged.

If an earlier project is unavailable, fails before the machine step, or has no due
machine offer, later projects remain eligible. Project-local config differences can
therefore expose an offer in a later project, but after an actual offer the batch does
not offer again. Repeat `d` or an invalid answer stays within the same confirmation
interaction. KeyboardInterrupt retains the existing batch-abort behavior.

Retain normal per-project initialization, current-directory restoration, deferred
chezmoi deployment, and error continuation. A declined first offer retains the existing
`needs_attention` batch outcome; suppressing later offers must not erase a prior
failure/decline. Use the canonical `sase machine init` spelling in the outer prompt.

### 5. Documentation and verification

Update the initialization and remote-machine sections of `docs/init.md` and the relevant
explanation in `docs/remote_dispatch.md`. Explain the one-off catch-up review, local
scope, skipped candidates, explicit reconsideration, and that interactive onboarding may
discover/probe while check/JSON modes remain offline. Keep ordinary `--diff` semantics:
it displays diffs and is not itself a check-mode guarantee. Update relevant docstrings
whose promise currently says discovery occurs only during explicit machine-init apply.
No new CLI/config keys are needed.

Add focused behavior regressions using fake discovery, temporary SASE roots, and
injected input; do not contact actual providers or mutate real machine state:

1. Rust: missing versus completed-empty review; unchanged/duplicate/reordered entries;
   skipped candidates; one new candidate; enrolled/repair exclusion; pin and endpoint
   matching; harmless display changes; partial-provider merges; invalid schema/state.
2. PyO3/facade: new APIs round-trip persisted state and assessment; existing machine
   setup API fixtures still pass without requiring new fields.
3. Apply/store: successful subset, blank selection, empty discovery, both entrypoints;
   cancellation/invalid selection and every existing activation failure preserve state;
   partial-provider success merges only presented candidates; atomic concurrent merge;
   malformed/missing/unwritable state and injected state-root isolation.
4. Single init: repeat invocation after explicit review causes no machine prompt; adding
   one unreviewed candidate produces an offer; declining repeats only in a subsequent
   invocation; a reviewed machine disappearing/reappearing stays quiet.
5. Batch: three projects with accept, decline, EOF, failing apply, and TTY `--yes` each
   offer/apply at most once; named-project batches behave the same; an unavailable or
   earlier-failing project does not consume the opportunity; later differing discovery
   config can expose an offer; a second invocation gets a fresh context.
6. Offline: single/batch check and JSON, canonical machine check, and non-TTY init
   with/without `--yes` neither discover nor write review state. Mock
   `discover_detailed` itself to fail if called, not just the older `discover` method.
   Preserve schema-1 JSON and existing exit/status rules.
7. Performance/control flow: equivalent batch contexts reuse assessment; config changes
   invalidate it; accepted apply rescans; discovery diagnostics do not block unrelated
   work; existing config-before-machine ordering and single deferred chezmoi deploy
   still hold.

Extend the existing `tests/dispatch/test_machine_init*.py`, relevant
`tests/main/test_init_onboarding*.py`, and `tests/main/test_parser_machine.py`,
splitting new tests by responsibility if size limits require it. Update shared fixtures
to isolate all new persistence paths. Use existing fake activation coverage rather than
rebuilding gateway behavior in a new end-to-end harness.

Before finishing implementation, read applicable verification memory and run the primary
repo's `just check`. Build/install the edited linked Rust binding using the existing
`just --set sase_core_dir <opened-core-path> rust-install` workflow so Python tests
actually exercise the new API. Run `just check` in the opened Rust repository; its gate
includes PyO3 and requires Python >=3.12. Testing only `cargo test -p sase_core` is
insufficient. Run longer commands through the SASE monitor skill as required. Follow
existing core revision/dependency release tooling for availability of the new binding
before the Python consumer lands; do not manually bump Rust crate versions. Use the
host-owned multi-repository finalization flow for both changed repositories.

## Acceptance criteria

A user can complete machine init, including choosing to enroll none, and subsequently
run `sase init --all` without repeated enrollment offers for the same discovered
machines. A newly discovered unreviewed machine produces one offer for the entire batch.
Accepting and completing that review silences subsequent offers for it; declining leaves
it eligible next time. Explicit machine init remains a full review, and offline checks,
enrollment trust, activation recovery, and other project initialization behavior retain
their existing contracts.
