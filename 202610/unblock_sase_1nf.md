---
tier: epic
title: Unblock the stalled sase-1nf epic
goal: 'Both stalled sase-1nf tracks move again without anyone waiting on a question.
  gc-measure closes, so the waiting gc-fix agent starts. Agent-run Apollo pushes authenticate
  as bbugyi200. Publication requests name the commit their agent actually created,
  and existing stale-revision rows are repaired explicitly. recovery-proof (sase-1nf.10.1.9)
  closes with evidence, so the waiting sase-1nf.10.1 land agent and then pub-recover
  (sase-1nf.11) resume on a backlog that can drain.

  '
decisions:
  check1_canary:
    ask: How should recovery-proof prove acceptance check 1 once the fixes are installed?
    choices:
      fresh_canary: One new limit-1 canary per host/project proves it before recovery-proof
        closes
      defer: recovery-proof closes without canaries; sase-1nf.11's first bounded batch
        proves it
    default: fresh_canary
    why: Three limit-1 calls give sase-1nf.10.1's land agent real check-1 evidence
    answer: fresh_canary
  gc_measure_scope:
    ask: How should gc-measure resolve its missing busy-hour workload evidence?
    choices:
      close_now: Close on the r5 four-hour evidence; gc-fix acceptance measures busy-hour
        j/k
      active_hour: You relaunch both TUIs with flags and use them for an hour before
        gc-measure closes
    default: close_now
    why: gc-fix must measure busy-hour j/k on fixed code anyway; Athena fails targets
      even idle
    answer: close_now
  ssh_fix:
    ask: How should Apollo stop pushing to GitHub with Athena's forwarded deploy key?
    choices:
      apollo_identity: Apollo's github.com uses only Apollo's own key; agent forwarding
        stays on
      no_forward: Athena stops forwarding its ssh-agent to tailnet hosts
    default: apollo_identity
    why: Fixes the push identity without changing forwarding for anything else
    answer: apollo_identity
phases:
- id: gc-closeout
  title: Close gc-measure and hand its findings to gc-fix
  depends_on: []
  size: medium
  description: 'gc-closeout: finish sase-1nf.13 under the chosen scope, append the
    closure and handoff to the GC research report, leave a handoff note for sase-1nf.14,
    and close only sase-1nf.13.'
- id: ssh-identity
  title: Apollo GitHub pushes authenticate as bbugyi200
  depends_on: []
  size: small
  description: 'ssh-identity: change the chezmoi-managed tailnet SSH config, get it
    applied on athena and apollo through a gate, verify the GitHub identity, and close
    sase-1ni.'
- id: request-revision
  title: Publication requests name the committed revision
  depends_on: []
  size: medium
  description: 'request-revision: record the pre-dispatch HEAD separately, publish
    with the post-commit SHA, give linked-repo rows page-only fulfillment in sase-core,
    and add regression tests that fail on the old code.'
- id: stale-row-repair
  title: Explicit repair for existing stale-revision outbox rows
  depends_on:
  - request-revision
  size: medium
  description: 'stale-row-repair: add a report-first, audited `sase agent sync` repair
    that maps each stale requested revision to the agent''s committed revision, or
    to linked scope, without dropping any request.'
- id: pub-closeout
  title: Prove recovery on both hosts and close recovery-proof
  depends_on:
  - ssh-identity
  - stale-row-repair
  size: medium
  description: 'pub-closeout: install the fixes on both hosts, run the repair, prove
    checks 1 and 5 under the chosen canary policy, update the acceptance report, then
    close sase-1nf.10.1.9.1 and sase-1nf.10.1.9.'
proposed_by: bbugyi200.athena.13m
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-24 07:01:31
status: wip
bead_id: sase-1nv
---

- **PROMPT:** [prompts/202610/unblock_sase_1nf.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/unblock_sase_1nf.md)
- **BEAD:** [sase-1nv](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1nv/README.md)

# Plan: Unblock the stalled sase-1nf epic

## Why the epic stalled (verified 24 October 2026)

`sase-1nf` (plan `plan:202610/athena_apollo_audit_fixes.md`) has ten closed phases and
four open ones. No agent is working on any of them. Four agents are alive and `WAITING`,
each on bead closure:

| Waiting agent        | Wakes when                                            |
| -------------------- | ----------------------------------------------------- |
| `sase-1nf.14`        | `sase-1nf.13` closes                                  |
| `sase-1nf.10.1.land` | `sase-1nf.10.1.9` closes                              |
| `sase-1nf.11`        | `sase-1nf.10` closes (`sase-1nf.10.1.land` closes it) |
| `sase-1nf.land`      | all 14 phases close                                   |

Do not kill, relaunch, or clear the waits of these agents. Closing the right beads is
what restarts them.

### GC track: `sase-1nf.13` (gc-measure) has no live agent

- The work is essentially complete. `research:202610/tui_gc_measurement_20261023.md`
  contains:
  - verified four-hour, single-instance windows on both hosts (r5: Athena `6666b3ec` on
    Python 3.14.7, Apollo `ec51eded` on 3.12.3);
  - completed producer experiments, census type counts, and fault/swap correlation.

  Athena fails the idle-p95, GC-share, hitch, late-hitch, RSS, and swap targets even
  under a quiet workload.

- The one missing item is representative busy-hour interaction. Athena has one j/k
  sample in the r5 window, and Apollo has none.
- Repeated continuation agents (`sase-1nf.13` through `sase-1nf.13--6`) have looped on
  that one item. Each asked the user to relaunch both TUIs with
  `SASE_TUI_GC_CENSUS=1 SASE_TUI_PERF=1`. The flags survive in-app exec restarts, but
  every full relaunch loses them. Each exec restart after an update also starts a new
  `app_instance_id`.
- At 10:03Z both TUIs were relaunched without the flags, and Athena's still is.
- The latest continuation, `sase-1nf.13--2`, crashed at 06:26 EDT with a `GateError`
  (notification `26ca1955-9561-46b3-a56e-ba49c78de9eb`, report
  `~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202610/24/20261024055752/error_report.md`).
  Its question received an answer with no selection and no text, which failed schema
  validation and killed the agent.
- Nothing will resume `sase-1nf.13`. Until it closes, `sase-1nf.14` (gc-fix) stays
  parked. Meanwhile `wait_checks` repeats "Wait dependency can never self-resolve" every
  two minutes.

### Publication track: the recovery-proof land agent ended waiting on a human decision

- The child epic `sase-1nf.10.1.9.1` (plan `plan:202610/publication_recovery_proof.md`)
  has all phases closed. Its land agent recorded a blocker on 22 October and ended. Note
  #3 on that bead says:
  - Checks 2, 3, and 4 pass, check 5 is partial, and check 1 is missing.
  - The canary budget is spent.
  - It asked for Option A (fix `sase-1ni`, then authorize new limit-1 canaries) or
    Option B (defer check 1 to `sase-1nf.11`).
- Nobody answered, so `sase-1nf.10.1.9.1` and `sase-1nf.10.1.9` are still open.
- It named three blockers. Fresh tracing shows two of them have a single root cause:
  1. **`sase-1ni`.** Agent-run `ssh apollo` sessions forward Athena's ssh-agent. Its
     first key is the `bobs-org/bob` deploy key (`~/.ssh/id_bob_vault`). Apollo's
     `github.com` config has no `IdentitiesOnly`, so GitHub authenticates the deploy key
     and denies every push. Clean commits `a090a3a2c` (bob-cli agents) and `f2d32a476b`
     (sase agents) are stranded ahead by 1 on Apollo.
  2. **`sase-1nj`: requests name the wrong revision.** Since commit `0c6b59b12cb`
     (2026-09-08), every `create_commit` / `create_pull_request` publication records the
     HEAD from _before_ the commit was created:
     - `src/sase/workflows/commit/workflow.py:333-346` stores
       `resolve_head_commit_sha()` in `CommitCheckpoint.primary_revision` before
       dispatch.
     - `:441` uses that value to detect whether dispatch created a commit.
     - The real post-commit SHA goes to `cp.commit_sha` (`:405-406`).
     - `src/sase/workflows/commit/workflow_publication.py:85` only resolves HEAD when
       `primary_revision` is empty, which it never is now. So both
       `publish_prompt_archive` (`:114-116`) and `publish_committed_agent_hood`
       (`:168-171`) enqueue the pre-commit or pre-rebase base.

     Since `1a69c8947d` (10-20) the commit-drain acknowledgement gate checks that
     revision. sase-core's `revision_associated`
     (`agent_publication_recovery/complete.rs`) requires an exact SHA in the run's
     `commit_shas`, so these rows never acknowledge. About 2,300 Athena/sase rows
     created after 09-08 carry such a base, plus about 329 rows carrying sase-core SHAs.
     Those can never match: run commits only come from primary roots
     (`inventory_history.py`, `inventory_sources.py`, `targets.py`), yet linked-repo
     commits enqueue under the host project (`_commit_publication_queue.py:242-270`).

  3. **"prompt archive is unresolved"** follows from 2. The deferred restore
     (`src/sase/agents_sync/prompt_archive/deferred.py`) looks up runs by
     `(local_agent, primary_revision)` against the run's commit SHAs. A stale revision
     never matches, so the local source looks absent and sase-core's
     `classify_deferred_prompt_obligation` refuses. The test
     `tests/test_commit_publication_inline.py::test_tagged_commit_publishes_and_pushes_every_sidecar_before_returning`
     now passes only because `5639375bfd` stubbed both gates (`:278-285`).

- Without these fixes, `sase-1nf.11`'s bounded drains would spend their budget on rows
  that can never acknowledge, and Apollo pushes would keep failing.

## Shared rules for every phase

- **Repos.** Open other repos with `sase repo open <name> -r "<why>"` (sase-core,
  chezmoi, research, plans, agents sidecars). Use only the printed path, and read that
  repo's `AGENTS.md` before editing. A modified opened repo is a `/sase_final`
  repository obligation.
- **Artifacts.** Read sidecar artifacts with `sase artifact read <ref> "<why>"`, never
  directly.
- **Rust boundary.** Shared decisions belong in sase-core
  (`rust_core_backend_boundary`). New wire fields are additive, with serde defaults that
  reproduce today's behavior, and Python reads them by name. These are bug fixes, so no
  feature flags.
- **Core pin.** A sase phase that needs a landed sase-core commit runs
  `just ratchet-core-revision` and then `just rust-install`. Use `/sase_monitor` for the
  cold build.
- **Verification.**
  - Run `sase tool run check` in every changed repo (for chezmoi, follow its
    `AGENTS.md`). Never run `just check-full`.
  - Read the `lint_and_test` reference memory before finishing a sase change.
  - Every behavior change needs a regression test that fails on the old code. Never
    weaken an assertion to get green.
  - Read `cli_rules` before adding CLI options and `docs_writing` before editing
    `docs/`.
- **Hosts.**
  - Reach Apollo with `ssh apollo 'zsh -lic "<command>"'` (`tailnet` reference memory).
  - Get landed code onto hosts only by requesting `sase update` (or `chezmoi update`)
    through `/sase_gate`, then continue in the gate's follow-up. Never run
    `just install` / `just install-dev` (`global-install-is-human-only`).
  - Use `/sase_monitor` for anything long; never wait inline.
- **Production publication state.** Use the existing CLI, recovery, and validation APIs.
  Never hand-edit outboxes, manifests, or progress files. Never retry everything, drop
  requests, raise the 120 s drain bound, or bulk-drain; bulk draining belongs to
  `sase-1nf.11`.
- **Beads and closure.**
  - Close only the beads a phase names, after verifying them. Record discovered work as
    `PROPOSED FOLLOW-UP: <summary — detail>` notes on your own phase bead.
  - Never close `sase-1nf`, `sase-1nf.10`, `sase-1nf.10.1`, `sase-1nf.11`,
    `sase-1nf.14`, `sase-10x`, or `sase-1f3`.
  - The ancestor decision `gc_soak = no` stays final.
- **Questions.** Ask the user only when a step genuinely needs them. When an answer is
  empty, ambiguous, or contradicted by live evidence, take the documented fallback and
  record it. Do not re-ask the same question in a loop.

## Phase `gc-closeout` (operational; closes only `sase-1nf.13`)

Read `sase-1nf.13`'s notes #16–#21 and the GC report. Do not repeat producer experiments
or the four-hour capture; both are complete.

> [!decision] gc_measure_scope = close_now Close gc-measure on the existing evidence.
> Representative busy-hour interaction, the j/k p95 distribution, other-tab latency, and
> the prompt-active comparison move to `gc-fix`'s acceptance, the same way
> `gc_soak = no` moved the 24-hour soak. That acceptance already has to capture j/k p95
> below 16 ms on the fixed code.
>
> 1. Read checkpoints `file:explicit:b5661e950a3aad343a5afdef` (r6 interruption) and
>    `file:explicit:4c5a448c914d52aca463ecab` (r7 probes) for the latest facts.
> 2. Append a "Closure and handoff" section to
>    `research:202610/tui_gc_measurement_20261023.md`. It states:
>    - What is measured and closes this phase: r5 four-hour windows on both Python
>      versions, ranked producer mechanisms, census types, and fault/swap correlation.
>    - What is not measured, and why: no representative busy hour; the TUIs were
>      repeatedly relaunched or exec-restarted, and flags were lost on full relaunch.
>    - The pre-fix j/k evidence that exists, labeled as non-representative:
>      - corrected r4 Athena Agents j/k p95 215.26 ms (n=52);
>      - Apollo's largest historical hour, 12 actual j/k actions;
>      - the short r6 windows, 3 Athena and 31 Apollo next/prev actions.
>    - That these requirements are now part of gc-fix acceptance.

> [!decision] gc_measure_scope = active_hour Make one bounded attempt, then close either
> way.
>
> 1. Ask once through `/sase_questions`. Have the user fully quit the TUI on each host
>    and run `env SASE_TUI_GC_CENSUS=1 SASE_TUI_PERF=1 /home/bryan/.local/bin/sase tui`
>    from that host's normal shell (no in-app restart). Ask them to use the Agents tab
>    normally, including j/k, for an hour, and to avoid `sase update` during that time.
> 2. Verify from live evidence on both hosts: the new instance IDs, both flags in
>    `/proc/<pid>/environ`, and current-instance census rows.
> 3. After each instance's 20-minute warmup, run a short monitor that confirms real
>    next/prev rows arrive on both hosts. Then monitor a busy hour of at least 60
>    minutes per host with the collector and corrected analyzer in setup bundle
>    `file:explicit:aa1235595f6ee5ae64a9ec41`.
>    - A separate instance from r5 is acceptable for the busy hour. Never pool instances
>      within one window.
> 4. Publish and audit the capture archive. Append the busy-hour results to the report.
> 5. Fall back to the `close_now` steps, with the limitation recorded, if any of these
>    happen: the answer is empty or contradicted by live probes, the user declines, an
>    instance changes mid-hour, or the hour still has no representative navigation. Do
>    not ask a second time.

Under either choice, finish with:

1. Add a handoff note to `sase-1nf.14` (`sase bead note`). It points to the report's
   ranked producers: Agent graph replacement / `copy_agent_graph` cycles, Textual
   Strip/FIFOCache render self-cycles, and the large archive/event/completion
   populations. It also lists what gc-fix's acceptance must handle:
   - **Report-tool defects.** These `tools/tui_freeze_report` defects from `sase-1nf.13`
     notes #4, #5, #13, and #17 must be fixed or worked around before its acceptance
     output can be trusted:
     - unsorted idle p95;
     - PASS despite incomplete coverage or late hitches;
     - endpoint-only swap;
     - paired-heartbeat and float-boundary accounting;
     - mixed action types counted as j/k.
   - **Launch protocol.** Flags exist only when the TUI is launched with
     `env SASE_TUI_GC_CENSUS=1 SASE_TUI_PERF=1 … sase tui` from a shell. They survive
     exec restarts, but each update restart creates a new instance. gc-fix should
     consider a durable opt-in, or verify the flags and early j/k rows before any long
     window.
   - **Perf-row identity.** Perf rows carry no PID or instance, so attribute them by
     wall-clock bounds with one TUI per host.
   - **Moved requirements.** The busy-hour, j/k, and 24-hour soak requirements now
     belong to gc-fix's acceptance.
2. Run `sase bead epic-symbols sase-1nf.13` and resolve any entries. Then run
   `sase bead close sase-1nf.13 --note "<what was measured, what moved to gc-fix, report ref>"`.
   This wakes the waiting `sase-1nf.14` agent. Leave `sase-1f3` open; `sase-1nf`'s land
   agent re-verifies it.
3. Commit the research report through `/sase_final`. Make no source or memory edits.

## Phase `ssh-identity` (chezmoi; closes `sase-1ni`)

Read `sase-1ni`, the `tailnet` reference memory, and chezmoi's `AGENTS.md`. The managed
file is `home/private_dot_ssh/private_tailnet.conf`. A `run_onchange` script includes it
at the top of `~/.ssh/config`, so its blocks take precedence over the unmanaged
`Host * ForwardAgent yes`. That script hashes the file by its source path
(`home/.chezmoiscripts/run_onchange_after_ensure_ssh_tailnet_include.tmpl`). If you
rename the file or make it a template, update that include so the hash still tracks it.

> [!decision] ssh_fix = apollo_identity On Apollo only, make `github.com` use
> `IdentitiesOnly yes` with Apollo's own key (`~/.ssh/id_rsa`, which authenticates as
> bbugyi200). Gate the rule to the host through chezmoi templating, or an equivalent
> check that cannot match Athena or the MacBook. Athena's effective `github.com`
> settings must not change; compare `ssh -G github.com` on Athena before and after.

> [!decision] ssh_fix = no_forward Add `ForwardAgent no` to the tailnet host blocks used
> for agent SSH (at least `apollo` and `apollo-do`). Verify that
> `ssh apollo 'ssh-add -l'` no longer lists Athena's keys.

Steps:

1. Make the change, and add or extend a chezmoi test if its test suite covers SSH
   config. Run chezmoi's checks and commit through `/sase_final`.
2. Once the commit has landed, request `chezmoi update` on Athena and Apollo through one
   `/sase_gate`.
3. In the follow-up, verify from Athena:
   - `ssh apollo 'zsh -lc "ssh -T git@github.com"'` greets `bbugyi200`, under the
     default forwarding.
   - `git push --dry-run` from Apollo's sase and bob-cli agents clones passes
     authentication. A non-fast-forward rejection is fine. Locate the clones through
     `sase repo open agents -p <project>` on Apollo.
   - Athena's own `ssh -T git@github.com` is unchanged.
4. Close `sase-1ni` with that evidence. Do not run publication retries here.

## Phase `request-revision` (sase, plus sase-core)

Goal: a newly finalized agent's publication request names the commit the agent created,
so the existing exact-match ack gate and deferred prompt restore succeed. Read
`sase-1nj` first.

1. **Checkpoint.**
   - Add `pre_dispatch_revision: str | None = None` to `CommitCheckpoint`
     (`src/sase/workflows/commit/checkpoint.py`).
   - `workflow.py:333-346` writes the pre-dispatch HEAD there instead of
     `primary_revision`.
   - The push-failure recovery check at `:441` compares against
     `cp.pre_dispatch_revision or cp.primary_revision`, so checkpoints written before
     this change still resume correctly.
   - Old checkpoint JSON must keep loading.
2. **Publication revision.** In `workflow_publication.py`, publish with `cp.commit_sha`
   (falling back to resolving HEAD, as today, when it is absent). Persist it as the
   checkpoint's publication revision before either publish step.
   - Both steps share one logical key, so they must use the same revision. If a resumed
     legacy checkpoint already completed `publish_prompt_archive` under the old
     revision, keep that revision for the remaining step; `stale-row-repair` handles it
     later. Do not create two rows.
3. **Linked repos.** A commit in a linked (non-primary) repo enqueues under the host
   project, but its SHA never appears in run `commit_shas`. Give such requests an
   additive scope field, defaulting to today's revision-bound behavior.
   - sase-core's `decide_publication_request_completion` fulfills a linked-scope request
     when the agent's required run page is published, with no primary-revision
     association. Do not add git ancestry to core.
   - Update the wire structs, binding round-trip tests, and core unit tests. Then update
     the Python outbox model, serialization, and the enqueue in
     `_commit_publication_queue.py`.
   - A turn that commits only to a linked repo must still publish and acknowledge.
4. **Tests.**
   - In `tests/test_commit_workflow_publication.py`, drive `CommitWorkflow.run()` with
     `provider.revision_id.side_effect` returning `"0"*40` before dispatch and `"1"*40`
     after. Assert that both publish calls receive `"1"*40`; this fails on the old code.
   - Update `tests/test_commit_workflow_checkpointing.py:191` on purpose to assert
     `pre_dispatch_revision`.
   - Cover push-failure recovery for new and legacy checkpoints, the resumed-legacy
     single-row case, and a linked-only commit that acknowledges once its page is
     published.
   - Remove the stubs in `tests/test_commit_publication_inline.py:278-285` if the real
     gates now pass with the fixed revision. Otherwise keep them and say why in the
     test.
5. **Docs.** Update `docs/reference/agents/hood-sync.md` and
   `docs/contribute/agents-sidecar-sync.md` wherever they describe which revision a
   request names or how fulfillment matches it.
6. Leave `sase-1nj` open; `pub-closeout` closes it after production repair. The
   planner-path gap (`sase-1c0`) is out of scope.

## Phase `stale-row-repair` (sase, plus sase-core)

Goal: rows already in production outboxes become fulfillable through an explicit,
audited, report-first repair. No request is silently dropped or rewritten.

1. **Decision (sase-core, pure).** Python gathers the facts for each row in one project:
   - the requested revision;
   - the agent's local run records and their recorded commit SHAs (the same inventory
     the drain uses);
   - for each of those commits, whether the requested revision is a primary-repo
     ancestor;
   - whether the revision belongs to a linked repo.

   A new core function classifies each row:
   - `unchanged`: the requested revision is already among the agent's run commits.
   - `linked`: the revision is a linked-repo commit. Apply `request-revision`'s linked
     scope.
   - `repaired(S)`: S is the earliest recorded run commit of that agent that descends
     from the requested revision. This covers both the parent case and the
     base-before-rebase case (`16d63cf9b0` requested as `50ef6ba498`).
   - `unresolved(reason)`: no evidence, or an ambiguous earliest candidate. Keep the row
     exactly as it is.

2. **Representation.** Preserve the original requested revision on the row, either
   through a recorded fulfilling revision or through a rekey that keeps the original
   value; choose whichever fits the outbox model. The original must stay visible in
   doctor and JSON output.
   - The ack gate and the deferred prompt restore must use the repaired revision.
   - Merge logical-key collisions deterministically and record the merge.
   - Lifecycle, attempts, and prior errors are kept. The repair never revives, retries,
     or drops a row.
3. **CLI.** Add a repair option to `sase agent sync`, following `cli_rules` and the
   existing `--repair-manifest` conventions.
   - It reports by default and needs an explicit apply flag to write.
   - It supports `--json` and runs one project per call.
   - It holds the agents sync lock and writes only through the outbox mutation APIs.
   - Every applied change goes to an audit JSONL beside the outbox: old key, new
     revision or scope, evidence commit, and time.
4. **Doctor.** `state.agent_publication_outbox` reports how many rows still have a
   requested revision that is not among their run's commits.
5. **Tests.** Build a fixture outbox, with real git, that covers:
   - an `unchanged` row;
   - a parent-case and a rebased-case `repaired` row;
   - a linked row;
   - an `unresolved` row, which stays byte-identical;
   - a collision.

   Also test that:
   - report mode writes nothing;
   - apply is idempotent on a second run;
   - after apply, the completion decision fulfills a repaired row against a published
     run carrying S, and the deferred prompt restore finds its local source.

6. **Docs.** Document the repair and the runbook order (report, apply, then bounded
   retry) in `docs/contribute/agents-sidecar-sync.md` and the outbox section of
   `docs/reference/agents/hood-sync.md`.

## Phase `pub-closeout` (operational; closes `sase-1nf.10.1.9.1`, `sase-1nf.10.1.9`, `sase-1nj`)

Read `sase-1nf.10.1.9.1` notes #1–#3, `sase-1nf.11`'s note, the closure section of
`plan:202610/publication_recovery_proof.md`, and
`research:202610/publication_recovery_acceptance.md`. Follow that plan's evidence
contract: UTC times, exact commands and exit codes, host/code/core revisions, selected
keys, before/after row fields, remote SHA, and validation, all saved as explicit
artifacts on this phase.

1. **Install.**
   - Confirm `sase version --json` on both hosts includes `request-revision` and
     `stale-row-repair`. If it does not, request `sase update` for both through one
     `/sase_gate`.
   - Confirm the Apollo identity check from `ssh-identity` still greets `bbugyi200`.
2. **Snapshot.** Take a lock-free outbox snapshot on Athena/sase, Apollo/bob-cli, and
   Apollo/sase. Back up each outbox file under a dated directory in `~/.sase/` before
   any repair.
3. **Repair.**
   - Run the repair in report mode for each host/project, then apply it.
   - Save both outputs and the audit JSONL.
   - Expect most post-09-08 Athena/sase rows to come back `repaired` and sase-core-SHA
     rows to come back `linked`. Explain `unresolved` counts; do not force them.
4. **Stranded commits.** Confirm whether `a090a3a2c` and `f2d32a476b` have pushed. They
   are expected to go out with the first normal drain or canary below. If they have not
   pushed by the end of the phase, record why.

> [!decision] check1_canary = fresh_canary This plan authorizes a new canary budget: one
> limit-1 call per host/project, plus at most one batch of up to 25 per host/project,
> used only when the limit-1 call selected an active row instead of the historical
> class. Use the bounded commands recorded in the acceptance report (`timeout` on
> Athena/sase, `rebase_conflict` on Apollo/bob-cli, `manifest_mismatch` with
> `--retry-retired` on Apollo/sase).
>
> Prove check 1: a named, formerly failing, stopped request publishes within 120 s,
> including its exact requested (repaired) revision and page, its prompt obligation, and
> remote manifest validation at a fetched revision. Unselected rows must keep their
> lifecycle and attempts. Stop on any regression.

> [!decision] check1_canary = defer Spend no canary budget. Record check 1 and its
> bounded commands as deferred to `sase-1nf.11`'s first bounded batch. That phase must
> see the repair results first.

5. **Check 5.** Find an agent that completed after the install and published through its
   automatic finalizer path, with an acknowledged request whose revision equals its
   stitch commit. Correlate the drain timing-log row with the fetched run, page, and
   prompt. If no natural completion exists, launch one minimal publication-eligible
   canary through `/sase_run`. Never spoof a marker.
6. **Report.**
   - Append a dated follow-up section to
     `research:202610/publication_recovery_acceptance.md` with:
     - the updated five-check table;
     - repair counts per host/project;
     - remaining backlog;
     - the bounded commands for `sase-1nf.11`, now run after the repair.
   - Attach the evidence artifacts to `sase-1nf.10.1.9`.
   - Commit through `/sase_final`.
7. **Close.**
   1. Close `sase-1nj` with production repair and acknowledgement evidence.
   2. Close `sase-1nf.10.1.9.1` with a note covering this phase's verified acceptance
      and the chosen canary policy.
   3. Set `status: done` in `plan:202610/publication_recovery_proof.md`, through the
      plans sidecar and `/sase_final`.
   4. Run `sase bead epic-symbols sase-1nf.10.1.9` and resolve any entries.
   5. Close `sase-1nf.10.1.9` with the verified acceptance and the research reference.
      This wakes `sase-1nf.10.1.land`, which closes `sase-1nf.10` and releases
      `sase-1nf.11`.

   Do not close anything higher.

## Landing (for this epic's land agent)

1. Confirm all five phase beads are closed and that the waiting agents woke:
   - `sase-1nf.14` after `gc-closeout`;
   - `sase-1nf.10.1.land` after `pub-closeout`.

   Check with `sase agent list -a -j`. If one did not wake, record it as a follow-up; do
   not relaunch it yourself.

2. File each phase's `PROPOSED FOLLOW-UP:` notes through `/sase_new_task`, after its
   duplicate check. Also file the defect that killed `sase-1nf.13--2`, using
   notification `26ca1955-9561-46b3-a56e-ba49c78de9eb` and its error report as evidence:
   a UserQuestion answer with no selection and no feedback raises a `GateError` and
   fails the agent, instead of being treated as no answer. `sase-1nf.13`'s own
   follow-ups (notes #4, #5, #13, #15, #17) belong to `sase-1nf`'s land agent; do not
   duplicate them.
3. Do not close `sase-1nf`, `sase-1nf.10`, `sase-1nf.10.1`, `sase-1nf.11`,
   `sase-1nf.14`, `sase-10x`, or `sase-1f3`. Do not run `just check-full`.

## Out of scope

- The bulk backlog drain and `sase-10x` closure (`sase-1nf.11`).
- GC fixes and their acceptance (`sase-1nf.14`).
- The planner prompt-archive path (`sase-1c0`).
- Apollo job/tailnet "Repository not found" fetch failures (carried as an existing
  follow-up).
- A durable telemetry opt-in for the TUI. That is a recommendation in the handoff note
  for gc-fix to decide.
