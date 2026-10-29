---
tier: epic
title: End agent commit stages at a durable publication enqueue
goal: "A no-conflict sase agent commit stage (provider done to done.json) drops from a
  median of about four minutes to about one, and sase-core pin-only rebase conflicts
  stop costing an LLM repair turn, while every sidecar write the commit path owes today
  is still made. The agent's own declared commits stay synchronous and host-verified.
  Prompt-archive, agent-hood, Referenced By, and artifact-link publication is durably
  queued before the stage ends, then drained by a host-owned background publisher, with
  today's inline drain as an automatic fallback.

  "
decisions:
  convert_backlog:
    ask:
      Should the queued legacy Referenced By requests be replayed as cites link events?
    default: true
    why:
      They are real prompt citations that the link-v2 fence has dropped since September
    answer: true
  decision_record:
    ask:
      Should the prove phase add a decision record for ending commits at durable
      enqueue?
    memory:
      - decisions
    default: false
    answer: false
phases:
  - id: refby-events
    title: Stop the fenced Referenced By loop and carry prompt citations as link events
    depends_on: []
    size: medium
    description:
      "refby-events: check the link-v2 fence before the lock, pull, and snapshot, emit
      prompt cites rows as publishable link events in fenced projects, make
      referenced-by failures charge attempts, and handle the existing queued backlog."
  - id: measure
    title: Truthful commit-stage timing
    depends_on: []
    size: small
    description:
      "measure: emit stitch op_started before the subprocess runs and add per-substep
      publication timing on the finalizer step channel."
  - id: retention
    title: Move artifact retention off the per-run tail
    depends_on: []
    size: small
    description:
      "retention: run enforce_artifact_retention from an hourly housekeeping job and
      keep only a throttled inline fallback in the runner's exec-finalize tail."
  - id: pin-resolver
    title: Resolve revision-pin rebase conflicts mechanically
    depends_on: []
    size: medium
    description:
      "pin-resolver: add a revision-pin resolver to the semantic rebase conflict chain
      that takes the descendant SHA when ancestry proves it, falling back to model
      repair otherwise."
  - id: deferred-parity
    title: Make queued prompt-archive publication equivalent and protected
    depends_on:
      - refby-events
    size: medium
    description:
      "deferred-parity: record the commit context on outbox rows, render deferred prompt
      archives exactly like the synchronous path including its side effects, and stop
      pruning pool files that a pending request still needs."
  - id: publisher
    title: Host-owned background publication job
    depends_on:
      - refby-events
    size: medium
    description:
      "publisher: add a doorbell-triggered scheduler job that drains agent-hood
      requests, hinted link-outbox agents, and unfenced referenced-by requests under the
      existing lock, plus a heartbeat and a real drain bound in the secret scanner."
  - id: enqueue-only
    title: End the commit stage at the durable enqueue
    depends_on:
      - measure
      - deferred-parity
      - publisher
    size: medium
    description:
      "enqueue-only: replace the commit path's prompt-archive publish and both drains
      with a durable enqueue plus publisher hints, ring the doorbell after done.json,
      and keep the inline path as a config- and liveness-gated fallback."
  - id: prove
    title: Prove latency and sidecar reliability on athena
    depends_on:
      - retention
      - pin-resolver
      - enqueue-only
    size: medium
    description:
      "prove: after the installed sase includes the epic, measure commit-stage latency
      against the baseline and audit every sampled run's sidecar obligations end to end."
proposed_by: bbugyi200.athena.15z
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-29 18:11:04
status: wip
---

- **PROMPT:**
  [prompts/202610/commit_stage_async_publication.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/commit_stage_async_publication.md)

# End agent commit stages at a durable publication enqueue

## Why, and what the evidence says

This epic implements the recommendation of the research report
`file:explicit:36faeacb2cd5192c51117b9b` (label
`research:202610/commit_stage_latency_synchronous_publication/commit_stage_latency_synchronous_publication.md`).
Read it with `sase artifact read file:explicit:36faeacb2cd5192c51117b9b "<reason>"`
before relying on any number below. Its findings were measured on athena against sase
`dd59c28f44`; recheck code paths against current master.

- A no-conflict run spends a median **236 s** from `provider_done` to `done.json`, of
  which `git commit` + fetch/rebase + push is **2.1 s**.
- After the push, `run_agent_publication_step` runs three passes of about 50 s each:
  - **A:** bead pages, then the prompt archive. Pass A is dominated by
    `_drain_referenced_by_after_prompt_publish`, which takes the plans-sidecar lock,
    pulls, and builds a full link snapshot. It then raises
    `artifact-link legacy index writes are fenced; use link events` from
    `_upsert_sidecar`. The exception escapes `drain_referenced_by_requests` before
    attempts are charged, so every commit replays the same requests forever.
  - **B:** plan header refresh, the agent-hood enqueue, and the bounded hood drain.
  - **C:** release evidence, then a full parse of the 10k-row artifact-link outbox to
    drain one agent's rows.
- The exec tail then spends about 28 s in `enforce_artifact_retention()`, and 99% of
  those passes trash nothing.
- The cost repeats for every committed repository.
- Rebase-conflict repair turns are 9% of phases but 58% of finalizer hours. Pin-only
  (`sase-core-revision.txt`) repairs are the slowest class (median 60 min).

## Why this is safe

The design rests on existing guarantees that each phase must keep. Verify each one in
code before relying on it.

- **The durable obligation already exists.** `_enqueue_committed_agent_publication`
  (`src/sase/agents_sync/_commit_publication_queue.py`) persists the agent-hood request
  without touching a sidecar. That same request also carries the prompt archive:
  `prepare_deferred_prompt_archive` and `restore_deferred_prompt_archives`
  (`src/sase/agents_sync/prompt_archive/deferred.py`) regenerate the archive inside the
  hood transaction. Acknowledgment happens only after push and manifest verification.
  That deferred path already runs in production whenever the synchronous attempt finds
  the agents lock busy.
- **Link rows do not depend on the drain timing.** Link-outbox eligibility
  (`_entry_is_eligible` in `src/sase/sdd/_artifact_link_outbox_drain.py`) is release
  evidence bound to the exact `(run_id, agent)`. The commit path records that evidence
  synchronously and must keep doing so. Stale-terminal dropping applies only to rows
  _without_ release evidence, so an eligible row is never aged out while it waits.
- **Background writes go to host-owned clones.** The agents-sidecar publication clone is
  machine-level (`ProjectTarget.sidecar_path`), and machine link writes use the hidden
  clones under `~/.sase/projects/<key>/repos/<role>`
  (`decisions:machine-link-writes-off-primary`). None of that work depends on the
  agent's evictable workspace except the prompt-artifact pool, which `deferred-parity`
  protects.
- **The locks are crash-safe.** The agents lock is an `flock` (`bounded_agents_lock`),
  so a killed publisher releases it. Repository-scope failures such as lock busy or a
  timeout do not charge attempts (`decide_publication_failure_accounting`).
- **The host-owned completion contract is unchanged.** Under
  `decisions:host-owned-completion`, the commit finalizer's postcondition is commit +
  push of every declared repository, and it remains so. A durable enqueue failure still
  fails the stitch with the existing `sase stitch create --resume` recovery.
- **Reliability improves.** Today, deferred hood requests are retried only when some
  later commit happens to drain them or a human runs `sase agent sync`. After this epic,
  a scheduler job drains them within seconds, with a five-minute backstop. When the
  scheduler or the job is not running, the commit path automatically falls back to
  today's inline drain.

### Reliability contract

Every sidecar write the commit path owes today, and who owns it after this epic:

| Write                                                 | Sidecar                      | Today                                      | After                                                                                                    |
| ----------------------------------------------------- | ---------------------------- | ------------------------------------------ | -------------------------------------------------------------------------------------------------------- |
| The agent's declared commits (any repo, sidecars too) | research, plans, beads, ...  | `sase stitch create`, sync, host-verified  | Unchanged                                                                                                |
| Bead pages for `SASE_BEAD` commits                    | beads                        | Sync, best-effort                          | Unchanged (sync); timed by `measure`                                                                     |
| Assigned bead close                                   | beads                        | Sync, strict                               | Unchanged                                                                                                |
| Plan header refresh for `SASE_PLAN` commits           | plans                        | Sync, best-effort                          | Unchanged (sync); timed by `measure`                                                                     |
| Prompt archive                                        | agents                       | Sync attempt, falling back to the hood row | Durable hood row enqueued sync; publisher regenerates the archive with full parity                       |
| Agent hood pages and manifest                         | agents                       | Durable row, then a bounded sync drain     | Durable row enqueued sync; publisher drains; 5-minute backstop; `sase agent sync`                        |
| Prompt `cites` rows (Referenced By)                   | plans, research              | Legacy request; never succeeds when fenced | Fenced: link events drained by the publisher. Unfenced: legacy drain run by the publisher                |
| Artifact-link events (reads and the like)             | hidden plans/research clones | Release evidence sync, then a sync drain   | Release evidence sync; publisher drains hinted agents; the hourly `artifact_link_backfill` drain remains |
| Artifact retention (local store, not a sidecar)       | none                         | Every run                                  | Hourly job; throttled inline fallback                                                                    |

Expected result (an estimate from medians, which `prove` must measure):

- **Sase-primary commit:** about 50–60 s. Declaration and scans (~12 s), stitch startup
  (~9 s), `just fix` (~14 s), git (~2 s), the enqueue (<1 s), and the remaining exec
  tail are still paid.
- **Sidecar-only commit:** about 35 s.
- **Multi-repo commit:** each additional repository adds seconds instead of minutes.

## Common implementation rules

- Open any other repository with `/sase_repo` (`sase repo open <name> -r "<reason>"`)
  and use only the printed path. Read sidecar artifacts with `sase artifact read`.
  Shared domain policy belongs in sase-core (see the `rust_core_backend_boundary` core
  memory): retry selection and failure accounting already live there and must stay
  there.
  - Scheduling, hint files, doorbells, and conflict resolvers follow the existing Python
    patterns, and none of the sibling resolvers uses Rust.
  - If a phase must change a Rust-owned producer id, eligibility rule, or wire, make an
    additive change in sase-core and move the `sase-core-revision.txt` pin as described
    in `docs/rust_backend.md`.
- Keep every on-disk format change additive. New outbox fields are optional, and old
  rows read as today. Old `commit_state.json` checkpoints must resume correctly; in
  particular, legacy `publish_prompt_archive`, `publish_agent_hood`, and
  `drain_artifact_link_outbox` steps keep their meaning.
- No feature flag is needed. The phase order lands the background publisher before the
  commit path depends on it. The inline path survives permanently as the fallback branch
  selected by config and liveness, not as a flag.
- Read the `lint_and_test` memory before finishing a sase change and run
  `sase tool run check`. Hand long commands to `/sase_monitor`. Do not run `check-full`,
  raw recipes outside `sase tool run`, manual commits, branches, or global installs
  (`decisions:global-install-is-human-only`; propose installs through `/sase_gate`).
  Finish through `/sase_final`.
- Read `cli_rules` before any CLI change, `docs_writing` before editing `docs/`, `tools`
  before touching `tools/`, and `tui` before touching `src/sase/ace/`. Update the
  generated config and option references through `just fix`.
- Docs that describe today's synchronous behavior and must be corrected by the phase
  that changes it:
  - `docs/reference/agents/hood-sync.md` (the prompt-archive and Referenced By
    paragraphs, and the commit workflow description);
  - `docs/reference/artifact-references.md` (the four-step write-back workflow);
  - `docs/reference/automation/scheduler-housekeeping.md` (new jobs and routine);
  - `docs/reference/disk-cleanup.md` (retention cadence).
- Register a new builtin job everywhere `artifact_link_backfill` is registered:
  - the `pyproject.toml` script entry;
  - the routine entry in `src/sase/default_config.yml`;
  - `src/sase/footprint/catalog.jsonl`;
  - the automation docs table.
- Every behavior change gets a regression test that fails on the old behavior. Do not
  weaken existing tests; update tests that asserted synchronous drains so they assert
  both the background and the inline branch.
- Phase workers close only their own phase and record discoveries as
  `PROPOSED FOLLOW-UP:` notes on their phase bead, not new beads. Before closing, run
  `sase bead epic-symbols <phase>` and re-key any remaining exemption to an open phase.
- A check failure reproduced identically on the clean base is documented with its
  reproduction and closes normally.

## Phase refby-events

Work in:

- `src/sase/agents_sync/prompt_archive/publish.py`;
- `src/sase/agents_sync/referenced_by_publication.py`;
- `src/sase/agents_sync/referenced_by_planning.py`;
- `src/sase/sdd/referenced_by_refresh.py`;
- `src/sase/agents_sync/git_sync.py` (the `sase agent sync` drain).

1. Add one cheap fence probe per project and per drain, reusing
   `legacy_artifact_link_writes_fenced` (`src/sase/sdd/_artifact_link_cutover_state.py`)
   with the machine link store's sidecar roots. Call it before any lock, pull, or
   snapshot. In a fenced project, never call `refresh_referenced_by` with `write=True`
   and never enqueue new `ReferencedByOutboxItem`s.
2. In fenced projects, emit the `cites` rows the prompt archive plans
   (`plan_referenced_by_requests`, `prose_referenced_by_requests`) as artifact-link
   outbox events instead.
   - Build each row with `artifact_link_row(item)`
     (`src/sase/sdd/_artifact_link_projection.py`), then `validate_artifact_link_row`.
   - Turn the row into an event with `observation_or_put_event_from_row` (it already
     maps `prompt_ref` origins to observation events), using a
     `stable_artifact_link_operation_id` over the project, source agent, relation,
     target, and origin.
   - Append with `append_artifact_link_outbox_event`, mirroring
     `_persist_derived_link_candidates_as_events` in
     `src/sase/sdd/artifact_link_derivation.py`.
   - Hard requirement: no appended event may be one the drain can never publish.
     - On the commit path, append under the committing run's
       `(SASE_AGENT_TIMESTAMP, publication agent)` identity, whose release evidence the
       commit path records. The existing pass-C drain then publishes the events in the
       same parse.
     - For a caller with no release evidence, such as the approved-planner archive
       (`src/sase/axe/run_agent_exec_plan_accept_sdd.py`), use a trusted machine lane
       that `_entry_is_trusted_machine_event` accepts. If no existing lane fits without
       losing the `prompt_ref` observation semantics, add an additive trusted producer
       in sase-core.
   - Keep the planner caller synchronous, because it needs `prompt_path`.
3. Make `drain_referenced_by_requests` charge failures. Catch exceptions raised by
   `refresh_referenced_by` per role, then record them through
   `update_referenced_by_requests(..., increment_attempts=True, quarantine_threshold=...)`.
   A request can then quarantine instead of replaying forever. Unfenced projects keep
   today's legacy write-back behavior byte for byte.
4. Handle the backlog of active referenced-by requests in fenced projects. The report
   counted 83 plans-role rows on athena, created Sep 18 to Oct 28.

   > [!decision] convert_backlog
   >
   > Replay each active request once as the equivalent `cites` event through a trusted
   > machine lane (`migrated` origin under an accepted machine producer, as the link-v2
   > cutover import did). Acknowledge each request only after its event is durably
   > appended. This is idempotent through the stable operation id.

   > [!decision] convert_backlog = no
   >
   > Mark each active request terminal with the reason
   > `legacy Referenced By write-back is fenced; superseded by link events`, so
   > `sase agent sync --drop-retired` can clear them. Emit no events for them.

5. Tests:
   - a fenced fixture never takes the plans lock or pulls, and costs O(1) git calls;
   - an unfenced fixture is unchanged;
   - an exception in refresh charges attempts and quarantines at the threshold;
   - emitted events are drained by `drain_artifact_link_outbox` for the committing agent
     and survive `drop_stale_terminal=True`;
   - projections after event publication show the same Referenced By rows the legacy
     path writes in an unfenced fixture;
   - the selected backlog branch.
6. Update the Referenced By workflow in `docs/reference/artifact-references.md` and
   `docs/reference/agents/hood-sync.md`.

## Phase measure

1. Fix the journal defect: `record_stitch_artifacts`
   (`src/sase/finalizers/commit_repair_stitch.py`) calls `OperationRecorder.start` only
   after the stitch subprocess has exited. Emit `op_started` (and the tracker state)
   when the subprocess launches. Do not emit it a second time afterwards. Keep
   `started_at` in the outcome JSON equal to the launch time. Check the other
   `OperationRecorder.start` callers for the same defect.
2. In `run_agent_publication_step`
   (`src/sase/workflows/commit/workflow_publication.py`), emit `start`/`ok`/`fail` step
   records through `sase.finalizers.steps.emit_step` for each sub-step. Use names under
   one `publication.` prefix: bead pages, plan header, prompt archive, hood
   enqueue/drain, release evidence, link drain.
   - Confirm the final deck under `src/sase/ace/tui/widgets/decks/final/` still renders
     sensibly. If it would need a TUI change, keep the names on the step channel only
     and note the follow-up.
3. Tests:
   - `op_started` precedes a slow fake subprocess's completion in the journal;
   - every publication sub-step emits a balanced pair, including on failure paths.

## Phase retention

1. Add a builtin job `artifact_retention` to the hourly `housekeeping` routine. Its
   script calls the existing `enforce_artifact_retention()`. Move that function from
   `src/sase/axe/run_agent_exec_finalize_artifacts.py` into a shared non-axe module if
   imports require it; the job must not import the runner.
   - Write a small stamp under `~/.sase/` with the time and outcome of every completed
     pass, including a skip because `artifacts.retention.enabled` is false or protection
     sources are unavailable.
   - Take a non-blocking exclusive lock so two passes never overlap.
2. In `finalize_loop` (`src/sase/axe/run_agent_exec_finalize.py`), replace the
   unconditional `_enforce_artifact_retention()` with a throttled fallback.
   - Run it inline only when the stamp is missing or older than 24 hours, so a machine
     without a scheduler still enforces retention at least daily.
   - Skip immediately if the lock is busy.
3. Tests:
   - the job runs and stamps;
   - the runner skips retention when the stamp is fresh and runs it when the stamp is
     stale or missing;
   - concurrent passes are excluded;
   - a disabled config still never trashes.
4. Update `docs/reference/disk-cleanup.md` and the automation docs.

## Phase pin-resolver

Extend the semantic conflict chain in `src/sase/sdd/_semantic_conflict_resolver.py`.
`_continue_rebase_resolving_semantic_conflicts` in
`src/sase/vcs_provider/plugins/_git_commit_dispatch.py` already runs that chain before a
rebase conflict escalates to model repair. After a semantic resolution it runs
`complete_rebase_gate_check` with outcome `applied`, so the rebased tree is still
verified.

1. Add `src/sase/sdd/_revision_pin_conflict_resolver.py`, with a path predicate and a
   resolver, and claim it in the chain like the plan-header resolver.
   - A path is claimed only if it equals a `repos.linked[].revision_pin` path from
     `revision_pins_for_project` (`src/sase/finalizers/commit_revision_pin.py`) for the
     repository root.
   - Read both conflict stages (`git show :2:<path>` and `:3:<path>`). Each must be one
     full 40-hex SHA (`_is_pin_sha`).
   - Resolve the pinned linked repo's local checkout through the existing linked-repo
     config. If either SHA is absent there, fetch that repo's default branch once with a
     bounded timeout.
   - Decide with `git merge-base --is-ancestor`, which is symmetric, so rebase's
     reversed ours/theirs does not matter. If one SHA is an ancestor of the other, write
     the descendant and `git add` the path. If the SHAs are equal, keep the shared
     value.
   - Diverged, missing, malformed, or unreadable states return a failed resolution with
     a precise message. The chain then falls through to today's model repair unchanged.
2. Never resolve a file that is not a configured pin, and never touch other conflicted
   paths. The chain's rule that any unclaimed path defeats semantic resolution stays.
3. Tests with real fixture repositories:
   - a linear sibling history in both conflict orientations;
   - equal SHAs;
   - diverged SHAs, which go to model repair;
   - non-SHA content;
   - a missing sibling checkout;
   - a pin conflict plus an unclaimed file, which goes to model repair;
   - the rebase gate still runs after a pin-only semantic resolution.
4. Document the resolver next to the existing semantic resolvers in the VCS reference
   docs.

## Phase deferred-parity

The background publisher will regenerate the prompt archive through the deferred path
for every commit. That path must produce what the synchronous path produces and must not
lose its inputs.

1. Add optional fields to `AgentPublicationOutboxItem` and its serialization
   (`src/sase/agents_sync/publication_outbox_models.py`,
   `publication_outbox_serialization.py`), without bumping the schema version unless old
   readers would reject the new fields:
   - `commit_cwd`: the committing checkout;
   - `artifacts_dir`: the run's artifacts directory;
   - `run_id`: the run's `SASE_AGENT_TIMESTAMP`.

   Populate them in `_enqueue_committed_agent_publication` and in
   `publish_prompt_archive` from what the caller already knows. Re-enqueueing the same
   logical key must fill missing fields without resetting attempts or state.

2. In `prepare_deferred_prompt_archive`, pass the recorded `commit_cwd` when present,
   falling back to `target.primary_checkout`. Today the deferred render uses the primary
   checkout, so `_ArtifactTargetResolver` stops treating the workspace as the primary
   root and builds blob URLs from the workspace's current `HEAD` instead of the
   immutable `primary_revision`.
3. Move the post-publication side effects of `_publish_prompt_archive` into one shared
   helper that runs after a verified publish in both paths:
   - `upsert_ref_file_versions`;
   - the `cites` emission from `refby-events`, or the legacy referenced-by enqueue in
     unfenced projects.

   The deferred path runs them for each acknowledged request after verification, using
   the recorded `run_id` and agent for event identity.

4. Fix the prompt-artifact pool prune in `src/sase/core/prompt_artifact_staging.py`.
   `_run_is_terminal_and_published` currently trusts `publish_prompt_archive` in
   `completed_steps`, and that step is appended even when publication was only queued.
   - A pool file becomes eligible only when every naming run is terminal and the
     project's agent-publication outbox has no active or quarantined row for that run's
     `primary_revision`.
   - Retired rows do not protect.
   - Keep the check cheap, since it runs only when the pool exceeds its budget.
5. Tests:
   - byte-identical prompt archives from the synchronous and deferred paths for one
     fixture run (including a pooled object and a workspace-relative file reference)
     after the workspace `HEAD` has moved;
   - side effects run exactly once per acknowledged request;
   - the pool keeps files for a queued or quarantined request and prunes them after
     acknowledgment;
   - legacy rows without the new fields still publish.

## Phase publisher

Add the host-owned drain. It must be safe to run alongside today's synchronous commit
drains, because it lands first.

1. Add `src/sase/agents_sync/publication_doorbell.py`, owning three things.
   - **Doorbell:** `~/.sase/projects/<project-key>/agent_publication/doorbell`, rung by
     an atomic mtime touch.
   - **Hints:** an append-only JSONL file in the same directory. Each line is a
     non-authoritative hint with `local_agent`, `global_agent`, `primary_revision`,
     `run_id`, the link-outbox agent name, and the time. The outbox stays the source of
     truth; a lost hint only delays work until the backstop.
   - **Heartbeat:** `~/.sase/agent_publication/heartbeat.json`, written by the job on
     every pass with per-project queue stats (active count, oldest active age, last
     outcome).

   Provide `ring_publication_doorbell(project_key, hint=None)` and
   `publisher_is_live(max_age_seconds)`. Hints are claimed by atomic rename, so a crash
   mid-pass re-merges unprocessed hints.

2. Add builtin job `agent_publication`
   (`src/sase/scripts/sase_chop_agent_publication.py`, script
   `sase_job_agent_publication`) in a new `publication` routine:
   - interval 15 s, job timeout 5 m;
   - an `fs` trigger on path `projects` with glob `*/agent_publication/doorbell`, and
     `max_quiet: "5m"` as the backstop.

   Copy the trigger and registration pattern of `agent_auto_restart` and
   `artifact_link_backfill`. Use an internal work budget of about 240 s.

3. Per enabled project, a pass does the following.
   1. Drain hinted active hood requests first through
      `drain_selected_agent_publications`.
   2. Run oldest-first `drain_agent_publications` batches until none are active, the
      lock is busy, a pass makes no progress, or the budget is spent. Each batch keeps
      its existing 120 s alarm. Lock busy is never charged and never quarantines.
   3. Drain the link outbox once for all hinted agents. Add an `agent_names` collection
      parameter to `drain_artifact_link_outbox` so N agents cost one parse, and pass
      `drop_stale_terminal=False` and a deadline.
   4. In unfenced projects with active requests, run `drain_referenced_by_requests`.
   5. If anything remains (active rows, a busy lock, unprocessed hints, or a spent
      budget), re-ring the doorbell so the next tick runs. Report counts through the
      `ChopResultBuilder` summary.
4. Make the drain bound real. `PublicationDrainTimedOut` is a `TimeoutError`, so it can
   be swallowed. In `src/sase/agents_sync/publication_protection.py`, re-raise it ahead
   of the broad handlers in `_scan_file` and `scan_publication_bytes`. Audit the other
   broad handlers reachable from the hood transaction the same way, in the style of the
   existing `except PublicationDrainTimedOut: raise` guards. Do not change the
   exception's base class.
5. Tests:
   - doorbell, hint, and heartbeat round trips;
   - crash recovery of claimed hints;
   - fresh requests are drained before the backlog;
   - lock busy re-rings without charging;
   - the budget stops the loop;
   - one link-outbox parse for several agents;
   - an alarm raised inside the scanner binding unwinds the drain and the request stays
     queued;
   - a SIGKILLed job leaves requests active and releases the lock.
6. Document the routine and job in `docs/reference/automation/scheduler-housekeeping.md`
   and the publication docs.

## Phase enqueue-only

1. Add config `commit.publication.mode` (`background` default, `inline`) to
   `src/sase/default_config.yml` and `src/sase/config/sase.schema.json`. The background
   branch applies only when all of these hold:
   - the mode is `background`;
   - `is_axe_running()` (`src/sase/axe/_process_probe.py`);
   - `publisher_is_live(15 min)`.

   Anything else runs today's inline code unchanged. Print one line naming the selected
   branch and why.

2. Background branch of `_run_agent_publication_inner`:
   1. Keep bead pages, plan header refresh, revision resolution, and
      `_record_real_change_release_evidence` synchronous.
   2. Enqueue the hood row with the new fields. A failed enqueue returns False, which
      keeps the `--resume` recovery.
   3. Append one publisher hint and record a new checkpoint step (for example
      `enqueue_agent_publication`).
   4. Do not publish the prompt archive and do not drain anything.
   - Replace the "queued and will retry automatically" warning with one info line.
   - On resume, an old checkpoint that has `publish_prompt_archive` but not
     `publish_agent_hood` takes the same idempotent enqueue.
3. Ring the doorbell after `done.json`. In `finalize_loop`, after
   `write_done_marker_and_update_index`, ring the project's doorbell when the run
   enqueued any publication. Publishing after `done.json` lets the hood capture terminal
   run state.
4. In the publisher, settle new requests. A request whose recorded `artifacts_dir` lacks
   `done.json` is skipped without charge until `done.json` exists or the request is
   older than 10 minutes. A crashed run therefore still publishes, through the backstop.
5. Audit every in-repo consumer that assumes publication finished before `done.json`,
   and adapt it to the queue. `_run_is_terminal_and_published` was the known one, fixed
   by `deferred-parity`. Grep at least for `publish_prompt_archive`,
   `publish_agent_hood`, and `drain_artifact_link_outbox` step names, and for readers of
   agents-sidecar prompt paths.
6. Tests:
   - both modes, plus the liveness fallback (scheduler down, stale heartbeat);
   - termination right after the enqueue, then a publisher pass that publishes the
     prompt archive and hood with parity;
   - duplicate delivery;
   - multi-repo runs enqueue one row per repository and ring once;
   - push rejection and a missing local source keep the request queued;
   - legacy checkpoint resumes;
   - release evidence is recorded before the stage returns, so the run's link rows are
     eligible;
   - the commit-stage publication step completes in well under a second with fake
     sidecars.
7. Update `docs/reference/agents/hood-sync.md` and the commit workflow docs: the commit
   stage now ends at commit, push, and a durable enqueue, followed by background
   publication, the inline fallback, and how to inspect the queue.

## Phase prove

1. Check that the installed `sase` on athena contains this epic's commits. If it does
   not, request the reinstall with `/sase_gate`; never install it yourself. Then, under
   `/sase_monitor`, wait until at least 100 no-conflict committing runs have finished on
   the new install, or 72 hours have passed. Accept a minimum of 30 runs and report the
   count.
2. Measure with read-only queries over the runs' artifacts, `finalizers/` journals, and
   `~/.sase/logs/agent_publication_drains.jsonl`, compared with the report's October
   baseline:
   - median and p95 time from `provider_done` to `done.json`;
   - finalizer-phase p95;
   - per-substep publication timings;
   - publisher queue age p50/p95.

   Targets: finalizer-phase p95 under 60 s, median model-done-to-`done.json` under 60 s,
   and queue-age p95 under 10 minutes.

3. Audit every sampled run's obligations. A gap is a defect to diagnose, not a
   statistic.
   - Each declared repository pushed.
   - Its hood row was acknowledged, not left active or quarantined.
   - Its prompt archive exists on the agents-sidecar remote, listing the run.
   - Its hood page shows terminal state.
   - No eligible link-outbox rows remain for its `(run_id, agent)`.
   - Any prompt `cites` it planned are published. Also check the outcome of any pin-only
     conflicts and the retention stamp cadence.
4. Store the report with `sase artifact create`, and attach the numbers to the phase
   bead. If a target is missed, record the measured residual and a `PROPOSED FOLLOW-UP:`
   for each cause.

   > [!decision] decision_record
   >
   > Author a new `decisions` strand, through `/sase_memory_write`, recording that the
   > commit stage ends at commit, push, and a durable publication enqueue, with
   > publication host-owned in the background. Include why it beat the synchronous
   > alternatives, its cost (eventual sidecar freshness and a background job to keep
   > healthy), and its reopen condition (publication freshness or loss regressions).
   > Then run `sase memory init`.

## Out of scope (record as follow-ups, do not implement)

- Compacting the artifact-link outbox and aging out its September rows.
- Caching secret-scan verdicts by `(sha256, scanner version)`.
- Querying the index per hood.
- Hood retention (`sase-11n`).
- Draining the 3,501 quarantined hood rows (`sase-1nf.11` owns that).
- Generated-doc and snapshot conflict resolvers.
- A visible "conflict repair" state with a time budget.
- Scoping `just fix` to changed files.
- Caching dirty-state scans.
- Republishing hoods whose runs became terminal after publication (the rest of
  `sase-1cc`).
