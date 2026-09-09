---
tier: epic
title: Replace toobig_split revision dedupe with conditional admission
goal:
  AXE retries oversized-file maintenance without repository-HEAD dedupe keys, and each
  queued toobig_split proposal uses durable %if admission to allocate no agent or
  workspace unless its target file is still at least 700 lines long.
phases:
  - id: axe-chop-typed-admission
    title: Durable typed admission for AXE chop proposals
    depends_on: []
    size: medium
    description:
      "axe-chop-typed-admission: route flag-enabled AXE proposals containing %if or
      %proc through the existing durable typed coordinator, preserve per-proposal chop
      ownership across detached waits, and settle skipped/error/launched units correctly
      in chop lifecycle state."
  - id: toobig-if-guard
    title: Admission-gate toobig_split at the configured line floor
    depends_on:
      - axe-chop-typed-admission
    size: medium
    description:
      "toobig-if-guard: remove repository-HEAD and proposal-dedupe behavior from
      bugyi-chops, emit a safe per-file %if Bash fence that rechecks the configured
      minimum line limit, and prove stale files skip while eligible files still launch."
  - id: conditional-admission-rollout
    title: Remove revision-key guidance and roll out the guarded chop
    depends_on:
      - axe-chop-typed-admission
      - toobig-if-guard
    size: small
    description:
      "conditional-admission-rollout: replace the prior HEAD-key documentation and
      dotfiles wording with the conditional-admission contract, deploy compatible SASE
      and bugyi-chops builds, and verify a live toobig_split run end to end."
proposed_by: bbugyi200.athena.0c1
bead_id: sase-sk
create_time: 2026-09-09 19:51:53
status: wip
---

- **PROMPT:**
  [prompts/202608/toobig_split_conditional_admission.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/toobig_split_conditional_admission.md)
- **BEAD:**
  [sase-sk](https://github.com/sase-org/sase--beads/blob/main/pages/sase-sk/README.md)

# Plan: Replace toobig_split revision dedupe with conditional admission

## Problem and verified current state

Yesterday's `bugyi-chops` commit `644583e6763dd9ced5429d310407c5bfe76f36cb` changed each
`toobig_split` proposal key from a file-content digest to:

```text
toobig_split:{workspace}:{path}:{repository HEAD}
```

That repaired an inert chop: AXE intentionally retains an accepted `dedupe_key` after a
successful agent, so an oversized file that an agent deliberately left unchanged had
been suppressed forever. The revision key reopens the file after any repository commit,
but it is the wrong work identity. An unrelated commit invalidates every file's key, the
chop executes Git only to manufacture dedupe churn, non-Git behavior diverges, and the
key says nothing about whether a queued split is still necessary.

The new `%if` directive provides the correct late-bound decision. It attaches an opaque
Bash or Python fence to one typed launch unit; after waits settle but before any runner,
workspace, agent identity, or model request is allocated, exit `0` makes the unit
eligible and exit `1` records a durable resource-free skip. The `typed_launch_units`
beta flag already exists and is enabled on this host.

There is one necessary integration gap that makes this more than a plugin-only edit. AXE
currently sends clan proposal batches directly to
`sase.agent.launcher.launch_agents_from_cwd`. That agent-only API deliberately raises
`TypedAdmissionRequiredError` when an enabled `%if` or `%proc` reaches it. Direct ACE /
`sase run` submissions and LaunchApproval requests use the durable typed coordinator,
but chop proposals do not. Merely adding `%if` to `bugyi_chops.toobig_split` would
therefore turn the scheduled chop into `action_failed`, not conditionally admit it.

## Intended behavior

For the configured `limits: [1000, 850, 700]` scan:

1. The chop scans and emits one proposal per path reported by `toobig`, preserving the
   existing deterministic clan/member IDs, shared summary, `@medium` model, priority,
   and sequential `wait_on` chain.
2. It supplies no proposal `dedupe_key`. Cadence and the `toobig-` clan inhibit guard
   prevent overlapping scans; every later scan may reconsider a file that remains
   oversized.
3. Each proposal contains `%if::` followed by one Bash fence. The predicate runs in the
   target project's source working directory after the preceding logical unit settles.
   It exits `1` when the file is absent or below the configured floor and `0` only when
   it is still at least that floor (700 lines in the production configuration).
   Read/count failures are condition errors rather than accidental eligibility.
4. A false predicate creates no agent, workspace, runner claim, or model request. The
   next logical unit remains ordered after that terminal skip and evaluates its own
   predicate.
5. An eligible unit launches through the unchanged agent path with the same chop
   ownership metadata. AXE keeps the chop run active until both admission and every
   launched agent have settled. Expected predicate skips count as successful no-op
   admission; condition or launch errors fail the action.

## Design boundaries

- Reuse the Rust-owned typed plan, wait graph, condition classification, and admission
  journal delivered by `sase-s6`; do not add a second condition parser/evaluator to AXE
  or `bugyi-chops`.
- Keep the bridge in SASE's Python AXE/launch glue. This is ownership and lifecycle
  integration, not new shared backend semantics, so it does not require a parallel
  implementation in `sase-core`.
- Preserve the legacy fast path byte-for-byte for proposal sets without active typed
  directives and while `typed_launch_units` is disabled. Explicit `%if`/`%proc` while
  the flag is disabled must fail before model dispatch with the existing actionable flag
  message.
- Use `/sase_repo` before every read or write in `bugyi-chops` and the linked `chezmoi`
  repository. Do not use the host's installed package files as editable sources.
- Do not clear the historical `seen.json`: after proposal keys are removed, its old
  content- and revision-shaped entries cannot match anything and bounded retention can
  age them out without a destructive state mutation.

## Phase 1: Durable typed admission for AXE chop proposals

Implement the SASE side first so no deployed chop can emit an unsupported prompt.

### Route typed proposals without changing legacy launches

- In `src/sase/axe/chop_proposal_launch.py` and focused sibling modules as needed,
  detect active typed directives only after proposal scaffolding and inline xprompt
  expansion have produced the exact launch batch. Plain proposal batches continue
  through `launch_agent_from_cwd` / `launch_agents_from_cwd` unchanged.
- For a typed batch, build the same immutable `LaunchPlan` and durable request bundle
  used by `dispatch_direct_typed_launch`, with a distinct source surface such as
  `axe_chop`. Resolve the plan's selected project and condition `source_cwd` from the
  proposal workspace/project, never from the AXE daemon's ambient current directory.
  Reject mixed-project typed batches explicitly rather than evaluating a predicate in
  the wrong repository.
- Extend the direct typed-bundle/admission adapter only as far as necessary to carry
  host-owned per-logical-unit dispatch metadata. Map `LaunchUnitWire.source_order` to
  the corresponding `PlannedChopProposal`; do not infer ownership from agent names or
  mutable prompt text.
- When an eligible Agent unit dispatches, merge the admission fingerprint/logical ID
  with that proposal's existing `build_chop_launch_env` output. Persist enough logical
  unit/proposal identity in `agent_chops.json` to let a later lumberjack process match
  agents launched by a detached coordinator. Do not expose those host fields in
  `SASE_CONDITION_CONTEXT`, inherit ambient chop variables, or allow a prompt-authored
  environment override.
- Suppress the generic direct-launch completion notification for AXE-owned admission;
  AXE's run history/lifecycle remains the single owner of this action's result.

### Make chop lifecycle own the whole admission

- Add a backward-compatible optional typed-admission record to `ChopRunEntry` (bundle
  identity/path, plan digest, and logical-unit-to-proposal metadata). Older history
  remains readable through the existing known-field filtering.
- When initial dispatch blocks on logical waits, persist the admission record and leave
  the chop in active `launched` state even if no Agent unit exists yet. Do not fabricate
  an `AgentLaunchResult`, fake PID, agent artifact, or placeholder launch row.
- Teach `finalize_launched_chop_runs` to reconcile typed admissions before applying the
  legacy agent-only matcher:
  - while the receipt is incomplete and its coordinator is live, keep waiting;
  - if an incomplete coordinator died, restart the existing bundle idempotently so the
    journal proves settled predicates and dispatch fingerprints are not replayed;
  - after admission completes, map launched logical units to their real chop-agent
    records and wait for those agents' ordinary completion artifacts;
  - treat `skipped` as a successful, resource-free unit outcome, but treat
    `condition_error`, `launch_error`, cancellation, missing receipt/linkage, and an
    unrecoverable coordinator as action failures;
  - finalize `on_action_success` checkpoints only when admission has no errors and every
    launched agent succeeds.
- Release once-per keys for units that admission skipped or could not launch, matching
  the existing rule that a proposal which never starts does not reserve its key.
  Successful launched units retain keys and failed launched units release them as they
  do today. This remains important for other conditional chops even though
  `toobig_split` will stop supplying a key.
- Render typed admission counts/reasons in chop logs and previews without labeling an
  exit-1 predicate as an AXE once-per duplicate or name collision. All-skipped
  conditional batches end as `action_succeeded`, with an explicit admission summary,
  rather than remaining active or becoming a generic launcher error.

### SASE tests and verification

- Add focused AXE tests covering a plain legacy batch, flag-off rejection, condition
  true, all-skipped, mixed skipped/launched, condition error, a skipped predecessor
  followed by an eligible dependent, delayed coordinator completion, coordinator
  restart, correct target `source_cwd`, per-unit chop registry linkage, checkpoint
  behavior, once-per key release, and absence of duplicate completion notifications.
- Include an end-to-end regression that submits a clan batch shaped like `toobig_split`:
  the second unit must not evaluate until the first settles; changing its file below the
  floor while it waits must produce a skip and no second agent.
- Run `just install`. Because this broadens a launch entry point and durable lifecycle,
  run `just check-full` only through `/sase_monitor` with `-s TESTING -S TESTED` and a
  concrete follow-up action. Run the focused typed-launch and AXE suites before that
  handoff for faster diagnostics.

## Phase 2: Admission-gate toobig_split at the configured line floor

Open `gh:bbugyi200/bugyi-chops` through `/sase_repo` and change only that checkout.

### Replace HEAD dedupe with a generated `%if` fence

- In `src/bugyi_chops/toobig_split.py`, delete `_repo_revision`, `_dedupe_key`, the
  per-scan Git call, and the `dedupe_key=` proposal argument. Keep `_path_digest`; it is
  the stable proposal-ID/name component and is unrelated to once-per filtering.
- Add one focused prompt builder that takes a normalized relative path and the
  configured floor (`min(limits)`, which is 700 for the production values). Use
  `shlex.quote` even though whitespace paths are rejected, because metacharacters can
  otherwise turn a scanner-controlled path into Bash syntax.
- Emit a readable directive-owned fence before `#split_file:<path>`, retaining `%auto`
  and `%wait(priority=20)`. Its Bash program must have these exact outcome classes:
  - missing/non-file target: `exit 1` (stale work, skip);
  - successful line count below the floor: `exit 1`;
  - successful line count at or above the floor: `exit 0`;
  - inability to read/count an existing file: an exit other than `0` or `1`, so the
    coordinator records a visible condition error instead of silently launching or
    skipping.
- Derive the threshold from the validated limits rather than duplicating a hidden 700
  constant. The production prompt must visibly contain the `>= 700` check, while custom
  test limits continue to gate at their configured floor.

### Update package compatibility, tests, and docs

- Raise the `sase` dependency floor in `pyproject.toml` to the first released series
  that contains the completed `sase-s6` typed-directive and AXE-admission contract, keep
  the upper bound on the next breaking series, update package version metadata if
  required by this repository's release convention, and refresh `uv.lock`. Do not claim
  compatibility with SASE 0.13, which rejects the emitted directive.
- Remove the Git-initialized fixture machinery and HEAD/non-Git dedupe tests introduced
  by `644583e`. Replace them with assertions that proposals omit `dedupe_key`, preserve
  IDs/clan/waits/model, and contain one parseable `%if` whose opaque code names the
  exact path and configured floor.
- Execute the extracted Bash body in temporary repositories at 699, 700, and 701 lines,
  after deletion, and for an injected/metacharacter path. Assert exit classifications,
  no command injection, and that the model-facing `#split_file` work prompt does not
  contain the directive or its code.
- Add an integration test against the SASE AXE bridge from Phase 1 (using the supported
  public/test surface, not internal source copies) proving a queued proposal whose file
  shrinks below the floor settles skipped with no agent launch.
- Update `README.md` examples and the `toobig_split` contract: proposals are
  condition-gated rather than content/revision-deduped, the condition runs after the
  sequential wait, and `typed_launch_units` plus the compatible SASE version are runtime
  prerequisites.
- Run `just install` and `just check` in the opened `bugyi-chops` checkout. Verify the
  lock resolves the intended SASE/core versions rather than silently exercising the old
  0.13 parser.

## Phase 3: Remove revision-key guidance and roll out the guarded chop

### Correct SASE and host configuration documentation

- In `docs/axe.md` and `docs/configuration.md`, remove the advice added by SASE commit
  `e2056bddebf0898dbf67f8aa8420a057dab42712` that tells retryable chops to include the
  repository revision in a content key. Retain the accurate warning that successful keys
  remain reserved, but describe `dedupe_key` as durable work identity, not a retry
  clock.
- Document the new flag-gated AXE behavior near structured proposal launching: `%if`
  uses typed admission after `wait_on`, false conditions allocate no agent resources,
  AXE owns the detached admission through final action settlement, and plain prompts
  keep the legacy path. Update `docs/architecture.md` so AXE joins ACE, `sase run`, and
  LaunchApproval as a typed-admission source.
- Open the linked `chezmoi` repo through `/sase_repo` and update
  `home/dot_config/sase/sase_athena.yml` descriptions. Replace "content-deduped" wording
  with the hourly/inhibit/sequential `%if` contract and state that every member rechecks
  the 700-line floor immediately before admission. Do not edit generated SASE memory or
  provider instruction files.

### Deployment and live verification

- Land/deploy SASE with Phase 1 before installing the new plugin. Confirm
  `typed_launch_units` is enabled in the effective host flag snapshot; do not create a
  second flag or depend only on an ambient test variable.
- Land the `bugyi-chops` change, then reinstall/sync it into the same managed uv tool
  environment as SASE. Verify with that interpreter that `_repo_revision` is absent and
  a dry-run proposal contains `%if::`, `-ge 700`, and no `dedupe_key`.
- Run `sase axe chop doctor` and a verbose manual dry run for `toobig_split[sase]`. Then
  perform a controlled live run with no active `toobig-` clan: capture its run ID,
  admission bundle/summary, and launched identities.
- Prove the race the change is for: arrange or select a later queued target, reduce it
  below 700 lines before its predecessor completes, and verify admission records
  `skipped`, creates no agent/workspace for that logical unit, continues the remaining
  chain, and lets the AXE run reach `action_succeeded`. Restore any temporary test-only
  content through the normal VCS workflow; do not mutate production run-history or
  `seen.json` by hand.
- Run `just install` and `just check` for SASE documentation/config changes. If the
  combined epic tree still touches launch broadening code at land time, repeat
  `just check-full` through `/sase_monitor`. Run the linked repo's own validation (or
  the narrowest available config validation) and keep all unrelated dirty changes
  untouched.

## Acceptance criteria

- `bugyi_chops.toobig_split` does not call Git, compute repository revisions or file
  content digests for once-per identity, or attach any `dedupe_key` to proposals.
- Every production proposal contains exactly one valid Bash `%if::` fence whose
  path-safe predicate exits eligible only when the target is still at least 700 lines;
  it runs after the proposal's logical wait and before any agent resource allocation.
- AXE uses durable typed admission for flag-enabled typed chop prompts, preserves the
  legacy path for plain prompts, and rejects flag-off typed prompts before an LLM sees
  them.
- Detached admission remains owned by the originating chop run across process boundaries
  and restarts. Predicate skips, condition errors, launch errors, real agents, once-per
  release, checkpoints, logs, and terminal run status are all reconciled without fake
  agents or permanently active runs.
- A stale queued file produces a durable skipped logical-unit result, no corresponding
  agent/workspace/runner/model use, and no failure of later dependents; a still-eligible
  file launches the unchanged `@medium #split_file` agent.
- SASE, `bugyi-chops`, and chezmoi documentation no longer recommend or describe HEAD-
  scoped dedupe. Package metadata requires a SASE release that actually implements the
  emitted contract, focused regressions pass, repository checks pass, and live AXE
  evidence confirms the end-to-end behavior.

## Risks and mitigations

- **AXE lifecycle is longer than the initial dispatch call.** Persist the admission
  bundle and logical mapping in the chop run; never infer completion from the first
  returned agent list.
- **Conditions could run in the daemon's wrong directory.** Resolve `source_cwd` from
  the selected proposal project and test a daemon cwd outside that repository.
- **Detached dispatch could lose chop ownership.** Carry only host-authored per-unit
  metadata in the request and persist logical IDs in the chop-agent registry; test a
  fresh coordinator process, not only an injected in-process dispatcher.
- **Removing dedupe permits repeated review of a file agents decline to split.** This is
  intentional: hourly cadence and the active-clan guard bound retries, while `%if`
  removes only stale queued work. A future cooldown would be a separate policy, not a
  revision disguised as work identity.
- **The beta flag can be disabled.** Fail visibly before model dispatch and document the
  prerequisite. Do not fall back to passing `%if` text to an agent.
- **Cross-repository rollout can briefly mismatch versions.** Deploy SASE first, pin
  `bugyi-chops` to the first compatible release series, then update the plugin and host
  descriptions. Dry-run and doctor checks precede the live run.
