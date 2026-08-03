---
tier: epic
title: Publish agents and beads sidecars asynchronously from an axe chop
goal: "`sase commit` never blocks on agents/beads sidecar publication. It records durable publication requests and
  returns, while a dedicated axe lumberjack drains those requests, and `just check` stays green while requests are still
  pending.

  "
phases:
  - id: scanfix
    title: Bound the agent-name registry source scan
    depends_on: []
    size: medium
    description: "scanfix: eliminate the per-lookup rescan of ~17k dismissed bundles and every agent artifact directory
      that makes plan-association resolution CPU-bound, which is the concrete stall reported by sase-cl.

      "
  - id: queue
    title: Durable sidecar publication queue
    depends_on: []
    size: medium
    description: "queue: generalize the agents publication outbox into a workspace-independent, per-project queue that
      also records pending bead-page and plan-header publication work, with enqueue, drain, quarantine, and retire
      semantics.

      "
  - id: chop
    title: publications lumberjack and sidecar_publication chop
    depends_on:
      - queue
    size: medium
    description: "chop: add the new axe lumberjack plus the builtin chop that drains the queue for every project, in
      agents -> beads -> plan-header order, under bounded per-project locks, work budgets, and exponential backoff.

      "
  - id: commit
    title: Rewire commit and other writers to mark instead of publish
    depends_on:
      - queue
      - chop
    size: medium
    description: "commit: convert `sase commit` and every remaining synchronous agents/beads sidecar writer into
      enqueue-only callers so no interactive command performs sidecar git work.

      "
  - id: validate
    title: Keep validation green while publication is pending
    depends_on:
      - commit
    size: small
    description: "validate: remove the dead prompt-to-plan link validation and make every remaining prompt-archive and
      plan check tolerant of a non-empty publication queue so `just check` never depends on sidecar publication.

      "
  - id: land
    title: Observability, docs, and sase-cl closure
    depends_on:
      - scanfix
      - chop
      - commit
      - validate
    size: small
    description:
      "land: surface queue health in doctor and ACE, refresh the axe and configuration documentation, verify the stall
      is gone end to end, and close the sase-cl bead with that evidence."
proposed_by: bbugyi200.athena.sh
create_time: 2026-08-03 06:18:50
status: wip
bead_id: sase-ej
---

- **PROMPT:**
  [prompts/202608/async_sidecar_publication.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/async_sidecar_publication.md)
- **BEAD:** [sase-ej](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ej/README.md)

# Plan: Publish agents and beads sidecars asynchronously from an axe chop

## Problem

`sase commit` performs all agents-sidecar and beads-sidecar publication inline, after the primary commit already exists.
That work is expensive (whole-tree association indexes, full sidecar git transactions, network pushes) and it sits
directly in front of every agent's commit. When it stalls, the agent is stuck even though its real work is done.

`sase-cl` is a concrete instance with four independent reproductions. The recorded `py-spy` hot path is:

```
run_agent_publication_step
  -> refresh_committed_plan_header
    -> build_plan_association_index
      -> resolve_agent_association_url
        -> get_reserved_family_names
          -> registry_file_is_stale
            -> source_signature_paths
              -> dismissed_bundles.rglob("*.json") / iter_agent_artifact_dirs
```

### Root cause of the stall

`src/sase/sdd/associations/_build.py::build_plan_association_index` builds one `HostedLinkResolver` and then calls
`resolve_agent_association_url` once per agent association, across **every** plan in the plans store.

`HostedLinkResolver` already has the mitigation: `snapshot_agent_name_registry()` caches `get_reserved_family_names()`
into `self._reserved_family_names`, and `agent_url()` passes that snapshot to `lane_ref_for_lane_name`. The bead-page
builder calls it — see `src/sase/bead_pages/associations/_build.py:125`
(`_snapshot_agent_name_registry(resolver, diagnostics)`). The plan builder never does.

So every unsnapshotted lookup falls through `agent_lanes._is_reserved_family_name` into `get_reserved_family_names()` ->
`load_name_registry()` -> `_cached_registry()` -> `_registry_file_is_stale(_CACHE_DATA)` ->
`_registry_store._source_signature()` -> `_registry_scan.source_signature_paths()`, which does a full `rglob("*.json")`
over `~/.sase/dismissed_bundles` plus an `iter_agent_artifact_dirs` walk of every project's artifacts, and then
`stat()`s every path found.

On this machine `~/.sase/dismissed_bundles` holds **17,225** JSON files. The cost is therefore
`O(number_of_agent_associations x 17k+ filesystem entries)` per commit, entirely CPU/syscall bound, matching every
reproduction in the bead (clean `git status`, commit already on `origin/master`, minutes of CPU, `Ctrl-C` landing in
`rglob` or `iter_agent_artifact_dirs`).

### Why the architecture must change too

Fixing the scan alone is not enough. `HostedLinkResolver.agent_url()` decides between an agent's family page and its
solo page by testing `self._agents_sidecar_path / f"families/{global_name}.md"` in the **local agents sidecar
checkout**. Once agent-hood publication is deferred, that file does not exist yet at commit time, so a plan header
refreshed during the commit would be written with the wrong link. The plan-header refresh is genuinely downstream of
agents/beads publication and has to move with it.

## Current behavior being replaced

`src/sase/workflows/commit/workflow_publication.py::run_agent_publication_step` runs, inline, after the primary commit:

1. `sase.bead_pages.publication.publish_committed_bead_pages` — takes the beads-store write lock, opens the bead
   project, builds a whole-store bead association index, renders every page in the committed bead's lineage, commits the
   beads root, and pushes.
2. `sase.agents_sync.prompt_archive.publish_prompt_archive` — publishes the agent's prompt document and artifacts into
   the agents sidecar.
3. `sase.sdd.plan_header_refresh.refresh_committed_plan_header` — takes the plans-store write lock, builds the whole
   plan association index, rewrites the plan header, commits and pushes the plans sidecar. **This is the stall.**
4. `sase.agents_sync.commit_publication.publish_committed_agent_hood` — enqueues into the existing agents publication
   outbox and then immediately drains it: ensure clone, bounded lock, `pull --rebase`, integrate incoming imports, build
   the project hood inventory, publish hoods, regenerate queued prompt archives, commit, push, with one non-fast-forward
   retry.

Step 4 already has exactly the durable-request machinery this epic needs (`src/sase/agents_sync/publication_outbox.py`):
schema-versioned, `flock`-guarded, atomically written, idempotent by logical key, with attempts/quarantine/retired
states, a `sase doctor` check (`src/sase/doctor/checks_agent_publication.py`), an ACE/chat-provenance reader
(`src/sase/history/chat_catalog_provenance/sidecars.py`), and CLI recovery via `sase agent sync --retry-quarantined` /
`--drop-retired`. **Reuse and extend it; do not invent a second parallel store.**

## Target behavior

- Every interactive command that would publish to the agents or beads sidecar instead writes a durable publication
  request and returns. The only remaining synchronous cost is a small `flock`ed JSON write.
- A new `publications` lumberjack runs one builtin chop, `sidecar_publication`, which owns _all_ automatic agents and
  beads sidecar git work: hood publication, prompt archives, bead pages, sidecar pushes, and the downstream plan-header
  refresh.
- Bead _state_ commits stay synchronous and local (see "Scope decisions"), but every sidecar **push** becomes chop-owned
  and retried.
- A pending queue never fails `just check`, never blocks an agent launch, and is visible in `sase doctor` and ACE.

## Scope decisions to preserve

These are deliberate; keep them and do not silently widen or narrow them.

1. **Local bead-state commits remain synchronous.** `sase bead` mutations write the canonical bead store that other
   agents read directly from the local checkout and that `bead_store_refresh` reconciles. Deferring the local commit
   would risk bead claims and `%wait(bead=...)` dependencies. What moves to the chop is bead **page rendering** and the
   **push**. Callers that today pass `push_after_commit=True` or `"async"` for a beads sidecar must enqueue instead.
2. **The plan-header refresh moves to the chop** even though it writes the _plans_ sidecar, because its `AGENTS`,
   `BEAD`, and `PROMPT` link targets are all resolved against published agents/beads sidecar state, and because it is
   the step that stalls commits today.
3. **Requests must be workspace-independent.** `sase commit` runs in an ephemeral `sase_<N>` workspace that may be
   recycled before the chop drains. A request may record a project key, a primary revision, a plan reference, a bead id,
   an agent lane, and a commit message, but must never depend on the committing workspace path still existing. The chop
   resolves work from `ProjectTarget.primary_checkout` (`sase.agents_sync.targets.resolve_sync_targets`).
4. **The chop is the only automatic drainer.** Do not add opportunistic drains to interactive commands. Explicit manual
   drains (`sase agent sync`, `sase axe chop run sidecar_publication -L publications`) remain available.
5. **Do not edit `sase/memory/*.md`, `AGENTS.md`, or the generated provider shims.** If a memory file becomes stale,
   file a task bead through `/sase_new_task` proposing the update.

---

## scanfix: Bound the agent-name registry source scan

Independent of the async work, and the phase that actually resolves what `sase-cl` reports.

### Changes

1. In `src/sase/sdd/associations/_build.py::build_plan_association_index`, snapshot the reservation registry exactly
   once before any `resolve_agent_association_url` call, mirroring `src/sase/bead_pages/associations/_build.py`. Reuse
   that module's `_snapshot_agent_name_registry` helper by promoting it to a shared location (a small helper next to
   `sase.association_agents` is a good home) rather than duplicating it. It must stay best-effort: a registry failure
   degrades to an empty snapshot and a diagnostic, never an exception.
2. Wrap the association-index build in `sase.agent.names._registry.name_registry_load_session()` so any lookup that
   still reaches the registry reuses one validated snapshot instead of revalidating.
3. Memoize `_source_signature()` in `src/sase/agent/names/_registry_store.py` for the duration of an active
   `name_registry_load_session()`. Outside a session, behavior is unchanged.
4. Bound the scan itself in `src/sase/agent/names/_registry_scan.py::source_signature_paths`. Both the recursive
   `dismissed_bundles.rglob("*.json")` and the per-project `iter_agent_artifact_dirs` walk are O(archive size) on every
   call. The bead explicitly asks for "caching or bounding". Pick one approach and justify it in the code: either fold
   the bundle archive into a cheaper directory-level signature (the archive is month-sharded — see
   `sase/ace/dismissed_agents*.py`) or cache the enumeration keyed by the mtimes of the shard directories. Whatever is
   chosen must still make the registry go stale when a bundle is added, removed, or rewritten. Add a test proving both
   the staleness contract and the bound.
5. `tests/test_dismissed_bundle_persistence.py::test_save_dismissed_bundle_is_fast_with_many_existing_bundles` spends
   ~124s in its 1000-bundle setup loop, meaning `save_dismissed_bundle` is itself O(existing bundles) per save. This is
   the same recursive-scan family the bead names. Fix it if the cause is a shared scan helper; if it turns out to be an
   unrelated write path, leave it and file a task bead through `/sase_new_task` instead of expanding this phase.

### Acceptance

- A regression test builds a plan association index over many plans and many agent associations and asserts that
  `source_signature_paths` (or the registry-load entry point) is invoked a **constant** number of times, not once per
  association.
- A test asserts the bounded scan still reports staleness when a dismissed bundle changes.
- `just check` passes.

---

## queue: Durable sidecar publication queue

### Changes

1. Extend `src/sase/agents_sync/publication_outbox.py` from an agent-hood-only outbox into a typed sidecar publication
   queue. Bump `PUBLICATION_OUTBOX_SCHEMA_VERSION` to 4 and keep readers for versions 1-3 so existing on-disk outboxes
   load unchanged; a version-3 item reads as an `agent_hood` request.
2. Add a request `kind` discriminator with at least:
   - `agent_hood` — the existing shape (`local_agent`, `global_agent`, `primary_revision`, `local_hood`, `hood_digest`).
     Its logical key stays `(global_agent, primary_revision)` so existing dedupe, doctor output, and ACE provenance keep
     working.
   - `bead_pages` — `bead_id` plus the derived lineage root; logical key `(lineage_root, primary_revision or "")`.
     Re-marking the same lineage before a drain must coalesce, not duplicate.
   - `plan_header` — canonical `plan_ref`, `primary_revision`, and the commit message needed by
     `refresh_committed_plan_header`; logical key `(plan_ref, primary_revision)`.
   - `sidecar_push` — a sidecar repo identity that has local commits needing a push; logical key
     `(sidecar_kind, project_key)`, coalescing.
3. Preserve every existing invariant: `flock` around read-modify-write, `atomic_write_json`, per-item `attempts`,
   `last_error`, `quarantined`/`quarantined_at`, `terminal`/`terminal_reason`, `created_at`/`updated_at`, duplicate
   logical keys rejected at read time, and `snapshot_agent_publications_from_path` staying lock-free and non-mutating.
4. Requests carry an explicit ordering rank so a drain always processes `agent_hood` -> `bead_pages` -> `plan_header` ->
   `sidecar_push` for the same revision.
5. Split `src/sase/agents_sync/commit_publication.py::publish_committed_agent_hood` into two exported functions:
   - `enqueue_committed_agent_publication(...)` — resolves the project target and the committing agent's lane, writes
     the request, and returns an outcome. No git, no network, no locks beyond the outbox `flock`.
   - `drain_agent_publications(project_key, ...)` — the existing locked transaction body (`_publish_queued_locked` /
     `_publish_queued_transaction` / `_prepare_publications`), unchanged in behavior. Keep the current
     `_CommitPublicationOutcome` fields so `_agent_publication_deferred_message` and its tests still apply.
6. Add the equivalent enqueue helpers for the other kinds, and drain helpers that wrap the existing publication bodies:
   `sase.bead_pages.publication` for `bead_pages`, `sase.sdd.plan_header_refresh` for `plan_header`, and
   `sase.sdd._commit_store.push_sdd_store_after_commit` for `sidecar_push`.
7. `sase agent sync` (`src/sase/agents_sync/git_sync.py::sync_agents`) must keep working: it already inspects, retries,
   and retires outbox items. Make sure it ignores or correctly handles the new kinds rather than mis-retiring them.

### Acceptance

- Round-trip tests for every kind: enqueue, coalesce, list, update with attempts, quarantine at threshold, retire,
  acknowledge, and drop.
- A test loading a version-3 outbox file and asserting it reads as `agent_hood` requests with unchanged logical keys.
- A concurrency test: two processes enqueueing simultaneously never lose or duplicate a request.
- Enqueue performs no git command. Assert this with a git runner that fails if invoked.

---

## chop: publications lumberjack and sidecar_publication chop

### Changes

1. Add `src/sase/scripts/sase_chop_sidecar_publication.py` following the `sase_chop_bead_store_refresh.py` pattern:
   `@builtin_chop("sidecar_publication")`, a `main()` calling `run_builtin_chop`, and a `runtime.emit_summary(...)` with
   stable counter fields (for example `projects_pending`, `agents_published`, `bead_pages_published`,
   `plan_headers_refreshed`, `pushes_completed`, `requests_failed`, `requests_quarantined`, `projects_backed_off`,
   `projects_deferred`) plus a `reason` when nothing was done.
2. Register the console script in `pyproject.toml` under `[project.scripts]`, keeping the block sorted:
   `sase_chop_sidecar_publication = "sase.scripts.sase_chop_sidecar_publication:main"`.
3. Add the lumberjack to `src/sase/default_config.yml` under `axe.lumberjacks`:

   ```yaml
   publications:
     description: |-
       Publish queued agents and beads sidecar work off the interactive commit path

       Runs every thirty seconds so sidecar pages, prompt archives, and their pushes land promptly without any
       interactive command waiting on sidecar git or network work. Put durable sidecar publication here; bead-store
       reconciliation, ChangeSpec lifecycle, and remote PR polling belong in the other lanes.
     interval: 30
     chop_timeout: "5m"
     chops:
       - name: sidecar_publication
         script: sase_chop_sidecar_publication
         description: |-
           Drain each project's queued agents, beads, and plan-header publication requests

           Publishes queued agent hoods and prompt archives, renders and commits queued bead pages, refreshes the
           plan headers those publications feed, and pushes each sidecar, under bounded per-project locks. Contended
           or failing projects back off exponentially and are retried on a later tick instead of blocking the pass.
   ```

   Line 1 of every `description` must be at most 100 characters, and line 2 must be blank — the axe config schema is
   fail-closed on this.

4. Chop behavior per tick:
   - Enumerate projects with a non-empty queue using the lock-free `snapshot_agent_publications_from_path` reader over
     `sase_projects_dir()`. Do not take locks to discover work.
   - For each project in a deterministic order, take the existing bounded agents lock (`git_sync.bounded_agents_lock`)
     or beads-store write lock (`sdd._git_contention.store_git_write_lock`) as appropriate, and drain that project's
     requests in rank order.
   - Bound the pass exactly like `sase_chop_bead_store_refresh`: a whole-pass work budget derived from the chop timeout,
     an equal share of a lock-wait budget per project with a floor, decline rather than block on a contended lock, and
     defer unattempted projects to the next tick.
   - Persist exponential backoff per project in `runtime.context.state_dir` **before** each attempt, so a SIGKILL at the
     chop timeout leaves the backoff behind rather than erasing it. Prune backoff entries for projects that no longer
     have queued work.
   - Acknowledge on success; on failure record the error, increment attempts, and quarantine at
     `configured_publication_max_attempts()`. Never raise out of a single project's drain — one bad project must not
     stop the pass.
5. The chop must be safe to run concurrently with a manual `sase agent sync`; the sidecar locks are the arbiter.

### Acceptance

- Unit tests for the chop covering: empty queue, one project draining every kind in rank order, a contended lock being
  declined and backed off, a failing request quarantining after the configured attempts, the work budget deferring later
  projects, and backoff persisting across a simulated timeout kill.
- `sase axe chop list` and `sase axe lumberjack list` show the new lane and chop.
- `sase axe chop run sidecar_publication -L publications --dry-run` works.

---

## commit: Rewire commit and other writers to mark instead of publish

### Changes

1. `src/sase/workflows/commit/workflow_publication.py::run_agent_publication_step` becomes enqueue-only:
   - Resolve the primary revision (it is still needed as a request key, and the existing `--resume` error paths for an
     unresolvable revision must be preserved).
   - Enqueue `bead_pages` when the commit footer carries `SASE_BEAD`, `agent_hood` and its prompt archive when there is
     a publication agent, and `plan_header` when the footer carries `SASE_PLAN`.
   - Keep the existing `cp.completed_steps` checkpoint entries so `sase commit --resume` stays idempotent.
   - Remove the inline calls to `publish_committed_bead_pages`, `publish_prompt_archive`,
     `refresh_committed_plan_header`, and the drain half of `publish_committed_agent_hood`.
   - Replace the "queued and will retry automatically" messaging with a single accurate status line naming the
     `publications` lane, and keep `_agent_publication_deferred_message` for the quarantined/retired cases.
   - `run_agent_publication_step` must not be able to fail the commit for publication reasons any more. The only
     remaining `RunResult.FAILED` path is a genuinely unresolvable primary revision.
2. Convert the remaining synchronous sidecar writers. Audit and handle each of these; the list was built by grepping
   `commit_sdd_store_files`, `push_after_commit`, and `agents_sync` callers, so re-run those greps and treat any new hit
   the same way:
   - `src/sase/bead_pages/publication.py::publish_committed_bead_pages` — keep the rendering body as the chop's drain
     implementation; add a thin `mark_committed_bead_pages(...)` enqueue entry point for callers.
   - `src/sase/bead_pages/refresh.py` — the explicit `sase bead pages refresh` command may still render and commit
     synchronously (it is an explicit user action), but it must enqueue a `sidecar_push` instead of pushing.
   - `src/sase/bead/cli_common.py::auto_commit_bead_store` / `bead_store_mutation` / `_push_committed_bead_store` — keep
     the local commit, enqueue the push.
   - `src/sase/bead/cli_work_from_plan_store.py:401` — currently a **synchronous** push (`push_after_commit=True`);
     enqueue instead.
   - `src/sase/bead/cli_admin.py`, `src/sase/sdd/beads.py`, `src/sase/plan_approval_actions.py`,
     `src/sase/llm_provider/commit_finalizer.py`, `src/sase/ace/tui/actions/agents/_notification_plan_background.py` —
     audit each; anything that pushes a beads or agents sidecar enqueues instead.
   - `src/sase/sdd/_commit_store.py::push_sdd_store_after_commit` — for beads/agents sidecar stores, replace the
     fire-and-forget `push_bead_work_launch_async` spawn with an enqueue. Other SDD stores keep today's behavior.
3. Because the plan-header refresh now runs in the chop, it must resolve its store from the project's primary checkout
   rather than the committing workspace. Verify `refresh_committed_plan_header` and
   `sase.sdd.plan_refs.workspace_context_for_plan_resolution` behave correctly when handed
   `ProjectTarget.primary_checkout`, and adjust if they do not.

### Acceptance

- A `sase commit` test with a fully populated footer (`SASE_BEAD`, `SASE_PLAN`, publication agent) asserts that no git
  command runs against any sidecar during the commit, that all expected requests land in the queue, and that the command
  returns promptly.
- A follow-up test runs the chop against that queue and asserts the sidecars end up in exactly the state today's
  synchronous path produces.
- A `sase commit --resume` test asserts requests are not duplicated.
- A test asserts a failing sidecar push cannot fail `sase commit`.

---

## validate: Keep validation green while publication is pending

`just check` must be healthy for agents to work, so it must never depend on sidecar publication having happened.

### Changes

1. Remove the dead prompt-to-plan link validation in `src/sase/sdd/_link_validation.py::validate_sdd_tree`.
   `src/sase/sdd/_link_files.py::list_sdd_files` only ever returns plan files now, so `infer_counterpart` can never find
   a prompt counterpart. The consequences today are a permanent `unpaired-file` diagnostic for every prompt-less plan
   (an error under `--strict`) and an entire unreachable branch covering `missing-link`, `reverse-link`, and prompt/plan
   `link-kind` checks. Delete the prompt-pairing logic and the now-unreachable branches, and drop `infer_counterpart`
   and `expected_link_type`'s prompt arm from `src/sase/sdd/_link_support.py` if nothing else uses them (symvision will
   flag leftovers). Keep: the `prompt-in-plans-store` migration error, `frontmatter-parse`, `plan-tier`,
   `parent-missing-target`, and `link-format` / `link-placement` checks for a plan's own `PROMPT` bullet.
2. In `src/sase/agents_sync/prompt_archive/validation.py`, confirm `plan-unresolved` and `prompt-unpublished` stay
   `warning` severity and add a queue-aware improvement: when the project has a pending `agent_hood` request covering a
   manifest run, report it as "publication queued" rather than "has no matching published prompt". Never promote it to
   an error.
3. Re-check the error-severity diagnostics in that module (`artifact-missing`, `artifact-digest`, `prompt-parse`,
   `prompt-section-sentinel`, `rendered-prompt-fence`, `xprompt-link-target`) and confirm none of them can fire merely
   because publication is pending. A prompt document and its artifacts are published in one transaction, so a partially
   published archive should be impossible; assert that with a test.
4. Confirm `sase validate` (`src/sase/main/validate_handler.py`) and `just validate-committed-plans` pass with a
   non-empty queue. `src/sase/sdd/committed_plan_validation.py` validates plan schema only, so a plan whose `AGENTS` and
   `COMMITS` sections have not been refreshed yet must still validate — add a test pinning that.
5. Update `src/sase/main/parser_plan.py` help text if any of it still describes prompt/plan pairing.

### Acceptance

- A test runs the full `sase validate` check set against a project with a non-empty publication queue and a plan whose
  prompt is not yet published, and asserts exit code 0.
- Tests for removed diagnostics are deleted or rewritten, not left asserting behavior that no longer exists.
- `just check` passes.

---

## land: Observability, docs, and sase-cl closure

### Changes

1. Extend `src/sase/doctor/checks_agent_publication.py` to cover every request kind, not just agent hoods. Keep the
   existing stalled/quarantined/retired classification and remediation commands, and add a diagnostic for a queue that
   is non-empty but not draining, which is the failure mode when the axe daemon is not running. Reuse
   `DEFAULT_PUBLICATION_STALLED_AGE_SECONDS` semantics.
2. Update `src/sase/history/chat_catalog_provenance/sidecars.py` so the new kinds do not corrupt the chat provenance
   backlog view, and surface pending publication in ACE wherever the agents-sync status is already shown
   (`src/sase/ace/tui/agents_sync_format.py`).
3. Documentation:
   - `docs/axe.md`: change "Axe ships with five default lumberjacks" to six, add a
     `### publications (30-second interval)` section with the chop table and a paragraph on bounding, backoff, and
     recovery, matching the depth of the existing `waits` and `checks` sections.
   - `docs/configuration.md`: add the new lane to the `axe.lumberjacks` example.
   - Any page describing `sase commit` as publishing sidecars synchronously (check `docs/beads.md`, `docs/agents.md`,
     and the prompt-archive docs) must describe the queue and the lane instead.
   - Do **not** edit `sase/memory/*.md`, `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, or `QWEN.md`. If those
     describe the old synchronous behavior, file a task bead via `/sase_new_task` proposing the update.
   - Do not edit `CHANGELOG.md`; it is generated by release-please from conventional commit subjects.
4. End-to-end verification before closing the bead:
   - Confirm `~/.sase/dismissed_bundles` still holds a large archive (it held 17,225 JSON files when this plan was
     written) so the measurement is meaningful.
   - Time a real `sase commit` on a commit carrying `SASE_BEAD` and `SASE_PLAN` footers and a publication agent. Record
     the wall-clock time and confirm no `run_agent_publication_step` frame appears in a `py-spy dump` taken during the
     commit.
   - Run the `sidecar_publication` chop and confirm the agents sidecar, beads sidecar, and plan header all reach the
     state the old synchronous path produced.
   - Capture the evidence as a SASE artifact via `/sase_artifact_file`.
5. Close `sase-cl`. First read the beads long-term memory with `/sase_memory_read` (`sase/memory/sase_beads.md`) for the
   close, resolution, and note semantics agents must follow. The close must reference the artifact from step 4 and state
   plainly both halves of the resolution: the registry scan is bounded and snapshotted so the association index no
   longer rescans the bundle archive per lookup, and publication no longer runs on the commit path at all. Do not close
   the bead if either half is unverified.

### Acceptance

- `just check` passes.
- `sase doctor` reports a healthy empty queue, and reports actionable diagnostics for a stalled one.
- The recorded timing evidence shows the commit completing without a publication stall.
- `sase-cl` is closed with that evidence attached.

## Risks

- **Silent non-publication when axe is not running.** The queue grows and nothing drains it. Mitigated by the doctor
  check in `land`, the commit's status line naming the lane, and the manual drain commands. Do not mitigate it by adding
  opportunistic drains to interactive commands.
- **Agent artifacts disappearing before the prompt archive is published.** `publish_prompt_archive` already degrades to
  a skip when `raw_xprompt.md` is unavailable, and the 30-second cadence makes the window small, but the drain must keep
  treating a missing source as a skip rather than a hard failure that quarantines the request.
- **Outbox schema migration.** Version 4 must read versions 1-3 losslessly. `sase doctor` and the chat provenance reader
  both parse the file; ship their updates in the same change.
- **Two drainers.** `sase agent sync` and the chop can run at once. The sidecar locks arbitrate, but both must
  acknowledge idempotently — `sync_agents` currently retires items whose page "did not materialize during full sync",
  which must not fire against a request the chop is concurrently handling.
