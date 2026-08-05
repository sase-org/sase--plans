---
tier: epic
title: Revert async sidecar publication so `sase commit` publishes sidecars inline
  again
goal: '`sase commit` once again publishes to every appropriate sidecar repo (agents,
  beads, plans) before it returns, so the `SASE_AGENT` footer URL resolves as soon
  as the commit lands. The `sidecar_publication` chop and the `publications` lumberjack
  are removed, the durable outbox is narrowed back to agent-hood retries, the agents
  sidecar corruption that is currently blocking all publication is repaired, and this
  project''s agents repo is fully synced.'
phases:
- id: commit
  title: Restore synchronous sidecar publication on the commit path
  depends_on: []
  size: medium
  description: 'commit: turn every enqueue-only writer back into an inline publisher
    so `sase commit`, planner approval, and the bead-store launch push perform their
    agents/beads/plans sidecar work before returning.'
- id: chop
  title: Remove the sidecar_publication chop and publications lumberjack
  depends_on:
  - commit
  size: medium
  description: 'chop: delete the builtin chop, its console script, the `publications`
    axe lane, its tests, and the lock-deadline plumbing that existed only to bound
    that chop.'
- id: queue
  title: Narrow the durable outbox back to agent-hood retries
  depends_on:
  - chop
  size: medium
  description: 'queue: drop the `bead_pages`, `plan_header`, and `sidecar_push` request
    kinds, bump the outbox schema so existing v4 files load without resurrecting them,
    and revert doctor, ACE, status, and prompt-archive validation to agent-hood-only
    semantics.'
- id: repair
  title: Repair the agents sidecar digest corruption blocking all publication
  depends_on: []
  size: medium
  description: 'repair: re-sign the 73 hood-snapshot file digests broken by an out-of-band
    sidecar rewrite, stop the writer that broke them, add a doctor check for payload/snapshot
    drift, and clear the stuck publication residue.'
- id: land
  title: Docs, end-to-end verification, agents-repo sync, and bead bookkeeping
  depends_on:
  - commit
  - chop
  - queue
  - repair
  size: small
  description: 'land: sweep the remaining queue/lane prose out of the docs, prove
    a real commit publishes inline and fast, sync this project''s agents repo until
    the `t2` family page resolves, and close out the sase-ej bead lineage.'
proposed_by: bbugyi200.athena.t4
create_time: 2026-08-05 14:26:20
status: wip
bead_id: sase-fa
---

- **PROMPT:** [prompts/202608/revert_async_sidecar_publication.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/revert_async_sidecar_publication.md)
- **BEAD:** [sase-fa](https://github.com/sase-org/sase--beads/blob/main/pages/sase-fa/README.md)

# Plan: Revert async sidecar publication so `sase commit` publishes sidecars inline again

## Problem

Epic `sase-ej` moved all agents/beads/plans sidecar publication off the `sase commit` path into a durable queue drained
by a new `sidecar_publication` axe chop. That design is being rejected, for a reason the epic did not weigh:

**The `SASE_AGENT` commit footer links to a page that does not exist when the commit is made.** Commit `0e40decdc`
carries `SASE_AGENT=[bbugyi200.athena.t2][2]` pointing at
`https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.t2.md`. That URL 404s today. Deferring
publication to a background lane means every commit's own provenance link is dangling for as long as the lane is behind,
and the commit message is immutable, so a permanently-undrained request leaves a permanently-broken published link.

The queue is not just theoretically behind; it is stuck. Observed on 2026-08-05:

- The agents sidecar's last commit is `49bdd7996`, dated **2026-08-03 15:46**. Nothing has published in two days.
- `~/.sase/projects/gh_sase-org__sase/agents-publication-outbox.json` holds **18 requests, all quarantined at 3
  attempts**: 17 `agent_hood` and one `sidecar_push`.
- Two of the quarantined `agent_hood` requests are exactly the `bbugyi200.athena.t2` publications whose URL the user
  reported as a 404.

The queue also failed silently. Nothing on the commit path surfaced that publication had stopped, because
`queue_sidecar_publication_step` was deliberately built so it "must not be able to fail the commit for publication
reasons any more."

## Root causes found while diagnosing

Two are code-level and one is data-level. All three must be handled for the restored synchronous path to be reliable.

### 1. The commit path cannot report publication failure (design, `sase-ej.4`)

`src/sase/workflows/commit/workflow_publication.py::queue_sidecar_publication_step` only writes JSON requests. Its
single failure mode is an unresolvable primary revision. Restoring the pre-epic `run_agent_publication_step` restores
the loud-failure contract: a publication that cannot even be queued fails the commit with a `sase commit --resume` hint,
and a queued retry is reported as a warning on the commit itself.

### 2. Owner manifests with unknown keys (already fixed in code, not in data)

15 of the 18 quarantined requests failed with `owner manifest has an invalid shape`. Commit `0e40decdc` ("fix: tolerate
forward-compatible owner manifests") fixes that failure, but the affected requests are already quarantined at the
attempt ceiling, so they will never retry on their own.

### 3. The agents sidecar payload no longer matches its signed snapshots (data corruption)

The remaining failures are
`file digest mismatch for 'agents/bbugyi200.athena.chop.refresh_docs.sase.0_753955.2/chat.md'`. Verified directly
against the sidecar checkout:

- Agents sidecar commit `49bdd7996` (`chore(agents): revert stored chat prompt sections`, `SASE <sase@localhost>`,
  2026-08-03 15:46) rewrote **73 `chat.md` files, 3605 deletions**, without re-signing the hood snapshots that reference
  them.
- A full audit of the checkout finds **exactly 73 digest mismatches out of 27,131 signed file references, 0 missing
  files**, all `chat.md`, all under owner `bbugyi200/athena`. Example: the `chop` hood snapshot records
  `size_bytes: 8539, digest: d9e3439f…` for that path; the file on disk is `6124` bytes with digest `e22db755…`.
- Affected hoods span `chop`, `sl`, `sn`, `se`, `sf`, `sw`, `t0`, `sase-ei`, `sase-ej`, `sase-f1`, and more.

This matters far beyond the epic revert: `publication_validation.py::load_validated_publication` walks **every** owner
manifest and **every** hood snapshot and verifies **every** referenced file before publishing **anything**. One
out-of-band rewrite therefore blocks all publication for the whole owner, permanently, in both the synchronous and the
asynchronous design. Reverting `sase-ej` without repairing this leaves `sase commit` failing loudly instead of failing
silently — better, but still broken.

## What "reverted" means here

The epic's phases are not equally implicated. Reverting all of them would reintroduce a separate, real defect.

### Reverted

| Commit      | Phase       | Subject                                                                                  |
| ----------- | ----------- | ---------------------------------------------------------------------------------------- |
| `6e3977945` | `sase-ej.2` | feat: add durable sidecar publication queue                                              |
| `0d6ed1a19` | `sase-ej.3` | feat(axe): drain queued sidecar publications                                             |
| `3ac2b097b` | `sase-ej.4` | feat: queue interactive sidecar publication                                              |
| `1116bccb0` | `sase-ej.5` | feat(sdd)!: keep validation green while publication is pending (**partial** — see below) |
| `671999252` | `sase-ej.6` | feat(sidecars): surface publication queue observability                                  |
| `465676c69` | `sase-ej.6` | refactor(sidecars): rename commit queue publication step                                 |

### Deliberately NOT reverted

1. **`c6bed8236` — `perf: bound agent registry scans during association builds` (phase `sase-ej.1`, "scanfix").** This
   is not part of the async architecture; it is the fix for the stall that `sase-cl` reported. Before it,
   `build_plan_association_index` re-scanned `~/.sase/dismissed_bundles` (17k+ JSON files) once per agent association,
   making `refresh_committed_plan_header` CPU-bound for minutes. Restoring synchronous publication **on top of** the
   unbounded scan would resurrect exactly the multi-minute commit hang that started this whole line of work.
   `sase-ej.6`'s own closing evidence on `sase-cl` records that "Registry lookup was already bounded by the earlier
   scanfix phases" — i.e. scanfix, not the chop, is what fixed the hang. Later commit `d317ab9ce` also builds on it.
   **Keep it, and prove in `land` that commits stay fast.**

2. **The dead-code half of `1116bccb0`.** That commit did two unrelated things. It removed the prompt-to-plan
   counterpart pairing from `src/sase/sdd/_link_validation.py`, which was provably unreachable (`list_sdd_files` only
   ever returns plan files) and emitted a spurious `unpaired-file` warning for every plan, plus it dropped the inert
   `--strict` flag. That is genuine dead-code removal with nothing to do with sync-vs-async; restoring it would restore
   a permanently-firing false warning. **Keep the removal.** Only the queue-aware arm it added to
   `src/sase/agents_sync/prompt_archive/validation.py` gets reverted, in phase `queue`.

   > Flagged for the owner: if you want `--strict` and the pairing checks back regardless, that is a separate change —
   > say so and it can be added to phase `queue`.

### `git revert` will not work

Seven later commits touched the same files and must be preserved: `0f19ffc66`, `028a69b59`, `fc20ba433`, `e6fbb435d`
(publication/outbox module splits), `9162b27e3` (sync-worker launch barrier), `0e40decdc` (forward-compatible owner
manifests), `d317ab9ce` (registry scan split). Hand-author forward changes and use `git show <commit>` /
`git show 3ac2b097b~1:<path>` as the reference for pre-epic bodies. Keep the split module layout — `just _lint-toobig`
enforces 1000/850/700-line limits and the pre-epic single-file shapes would violate them.

## Target behavior

- `sase commit` publishes bead pages, the prompt archive, the plan header, and the committing agent hood inline, and
  pushes each sidecar, before returning. When it returns `OK`, the `SASE_AGENT` footer URL resolves.
- A publication failure is visible on the commit: an unqueueable failure fails the commit with a `--resume` hint; a
  queued retry prints a warning naming the recovery command.
- The agent-hood outbox — which predates `sase-ej` — survives as the retry mechanism, with its quarantine/retire
  lifecycle, doctor check, ACE provenance, and `sase agent sync -q` / `-d` recovery intact.
- No axe lane, chop, or console script for sidecar publication exists.
- The agents sidecar's signed snapshots match its payload, and a future out-of-band rewrite is detected instead of
  silently poisoning all publication.

## Scope decisions to preserve

1. **Do not delete the agent-hood publication outbox.** It is pre-`sase-ej` machinery (schema v3), and
   `publish_committed_agent_hood` has always been enqueue-then-immediately-drain. Restore that composition; do not
   replace it with a bare direct publish.
2. **Do not edit `sase/memory/*.md`, `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, or `QWEN.md`.** If a memory
   file describes the queue/lane, file a task bead through `/sase_new_task` proposing the update.
3. **Do not edit `CHANGELOG.md`**; release-please generates it.
4. **Do not run a full `sase agent sync` against `bob-cli`.** Ready bead `sase-f6` documents that a full sync there
   re-synthesizes retired identities from immutable commit footers. The bob-cli publication residue described in the
   `sase-ej` notes stays where it is; it belongs to `sase-f6`.
5. **Keep every module split.** Reverting content, not file layout.

---

## commit: Restore synchronous sidecar publication on the commit path

Reference: `git show 3ac2b097b` and `git show 3ac2b097b~1:<path>`.

### Changes

1. `src/sase/workflows/commit/workflow_publication.py`: rename `queue_sidecar_publication_step` back to
   `run_agent_publication_step` and restore the inline sequence in pre-epic order:
   1. `sase.bead_pages.publication.publish_committed_bead_pages` (guarded by the `publish_bead_pages` checkpoint),
   2. resolve `cp.primary_revision` when there is a publication agent (preserving both `--resume` error paths),
   3. `sase.agents_sync.prompt_archive.publish_prompt_archive` (guarded by `publish_prompt_archive`),
   4. `sase.sdd.plan_header_refresh.refresh_committed_plan_header`,
   5. `sase.agents_sync.commit_publication.publish_committed_agent_hood` (guarded by `publish_agent_hood`).

   Restore the pre-epic failure contract: `RunResult.FAILED` on an unresolvable revision, on an exception out of
   `publish_committed_agent_hood`, and on `outcome.error and not outcome.queued and not outcome.skip_reason`. Keep
   `_agent_publication_deferred_message` for the queued/quarantined/retired cases, restoring its pre-epic wording. Drop
   `_print_publications_lane_status`.

2. `src/sase/workflows/commit/workflow.py`: rename `_queue_sidecar_publication_step` back to
   `_run_agent_publication_step`, update the import and docstring. Leave its position in `_run_tracking_steps`
   unchanged.

3. `src/sase/agents_sync/commit_publication.py`: restore
   `publish_committed_agent_hood(local_agent, primary_revision, *, project=None, commit_cwd=None, git_runner=run_git)`
   as a single exported function that enqueues and then drains under the bounded agents lock — the composition currently
   living in the test shim `tests/agents_sync/commit_publication_fixtures.py`. Keep `_publish_queued_locked` and the
   `commit_publication_transaction` hooks exactly as they are. `enqueue_committed_agent_publication` and
   `drain_agent_publications` may stay as module-private helpers if `publish_committed_agent_hood` is their only caller
   after phase `chop`; symvision will flag anything left unused, so delete rather than whitelist.

4. `src/sase/bead_pages/publication.py`: restore
   `publish_committed_bead_pages(commit_message, *, primary_root, project=None)` as the synchronous entry point wrapping
   the existing `_publish_bead_lineage` body. Delete `mark_committed_bead_pages` and `drain_bead_pages_publication`.

5. `src/sase/sdd/plan_header_refresh.py`: delete `mark_committed_plan_header` and `drain_plan_header_publication`.
   `refresh_committed_plan_header` is already synchronous and stays.

6. `src/sase/sdd/_commit_store.py`: delete the `SDD_STORAGE_SIDECAR_REPOS and store.sidecar_role` short-circuit added to
   `push_sdd_store_after_commit`, so sidecar stores push again through `push_bead_work_launch` /
   `push_bead_work_launch_async`. Delete `drain_sidecar_push_publication`.

7. `src/sase/axe/run_agent_exec_plan_accept.py::_publish_planner_prompt_archive`: restore the direct
   `publish_prompt_archive(...)` call with `agent_artifacts_dir`, `prompt_content`, `plan_ref`, `prompt_name`, and
   `yyyymm`, returning `outcome.prompt_path`. Remove the `_ = state, prompt_content, plan_name, yyyymm` discard.

8. `src/sase/bead/cli_work_from_plan_store.py::push_store_after_launch`: with the `_commit_store` short-circuit gone,
   confirm the beads sidecar is actually pushed. Keep the `sdd_commit_targets` loop if it resolves the right store;
   otherwise restore the pre-epic direct `push_sdd_store_after_commit(store, push_after_commit=True)`. Whichever you
   keep, pin it with a test that asserts a push against the beads sidecar remote.

9. Delete the `publish_committed_agent_hood` shim from `tests/agents_sync/commit_publication_fixtures.py` and import the
   real function in `tests/agents_sync/test_commit_publication_queue.py` and
   `tests/agents_sync/test_commit_publication_target_resolution.py`.

10. Update the tests that were rewritten to assert queueing: `tests/test_commit_workflow_publication.py`,
    `tests/test_commit_workflow_resume.py`, `tests/test_commit_workflow_checkpointing.py`,
    `tests/test_bead/test_bead_page_publication.py`, `tests/test_bead/test_cli_mutation_push.py`,
    `tests/test_sdd_commit_store.py`, `tests/agents_sync/test_committed_plan_header.py`,
    `tests/test_axe_run_agent_exec_plan_followup_approvals.py`. Rewrite them to assert the published result, do not
    delete coverage.

### Acceptance

- A `sase commit` test with a full footer (`SASE_BEAD`, `SASE_PLAN`, publication agent) asserts that when the command
  returns, the agents sidecar contains the agent/family page and prompt archive, the beads sidecar contains the rendered
  lineage, the plan header is rewritten, each sidecar has been pushed, and the publication outbox is empty.
- A test asserts an agent publication that cannot be queued fails the commit with the `--resume` message.
- A test asserts a queued retry warns but returns `RunResult.OK`.
- A `sase commit --resume` test asserts publication is not performed twice.
- `just check` passes.

---

## chop: Remove the sidecar_publication chop and publications lumberjack

Reference: `git show 0d6ed1a19`.

### Changes

1. Delete `src/sase/scripts/sase_chop_sidecar_publication.py` and `tests/test_axe_chop_sidecar_publication.py`.
2. Remove `sase_chop_sidecar_publication` from `[project.scripts]` in `pyproject.toml`, keeping the block sorted.
3. Remove the `publications` lane from `axe.lumberjacks` in `src/sase/default_config.yml`.
4. Revert the lock-deadline plumbing that `0d6ed1a19` threaded through `src/sase/bead/sync.py`,
   `src/sase/bead/sync_worker.py`, `src/sase/sdd/_git_contention.py`, `src/sase/sdd/_commit_store.py`,
   `src/sase/sdd/plan_header_refresh.py`, and `src/sase/bead_pages/publication.py` — **only where it becomes unused**.
   `store_git_write_lock(timeout=...)` and friends have other production callers; keep every parameter that still has
   one. Let symvision decide: run `just _lint-symvision` and delete what it reports.
5. Docs: `docs/axe.md` — restore "five default lumberjacks", remove the `publications` cell from the lane diagram,
   delete the `### publications (30-second interval)` section, and delete the
   `sase axe chop run sidecar_publication -L publications` examples. `docs/configuration.md` — remove the lane from the
   `axe.lumberjacks` example.
6. Confirm no orphaned runtime state is left behind: after the change, `sase axe lumberjack list` and
   `sase axe chop list` must not mention `publications` / `sidecar_publication`, and any
   `sidecar_publication_backoff.json` under the axe state dir is dead — remove it as part of `repair`'s residue cleanup
   and note the path in the phase result.

### Acceptance

- `sase axe lumberjack list` and `sase axe chop list` show five lanes and no `sidecar_publication` chop.
- `sase axe chop run sidecar_publication` fails as an unknown chop.
- No reference to `sidecar_publication` or the `publications` lane remains outside `CHANGELOG.md` and
  `sase/repos/plans/` history.
- `just check` passes.

---

## queue: Narrow the durable outbox back to agent-hood retries

Reference: `git show 6e3977945` and `git show 671999252`.

### Changes

1. `src/sase/agents_sync/publication_outbox_models.py`: remove the `bead_pages`, `plan_header`, and `sidecar_push`
   members of `PublicationKind`, the `bead_pages_publication_request` / `plan_header_publication_request` /
   `sidecar_push_publication_request` constructors, `PUBLICATION_KIND_RANK`, `ordering_rank`, and the kind-specific
   fields (`bead_id`, `lineage_root`, `plan_ref`, `commit_message`, `sidecar_kind`). `AgentPublicationOutboxItem` keeps
   its v3 fields and its `(global_agent, primary_revision)` logical key. Decide whether the `SidecarPublicationRequest`
   alias is still worth keeping; if nothing imports it, delete it.

2. **Schema migration matters here — the real outbox on this machine currently holds a `sidecar_push` record.** Bump
   `PUBLICATION_OUTBOX_SCHEMA_VERSION` to **5** (do not reuse 4) and keep readers for versions 1–4. A v4 file must load
   cleanly with every non-`agent_hood` record **dropped**, not resurrected and not raising. Emit a diagnostic naming how
   many records were dropped and why, so the drop is visible in `sase doctor` rather than silent.

3. `src/sase/agents_sync/publication_outbox_operations.py`, `_serialization.py`, `_store.py`, `_diagnostics.py`, and the
   `publication_outbox.py` facade: remove the multi-kind enqueue helpers, the rank-based ordering, and the kind
   discriminator from the exports. Preserve every pre-epic invariant: `flock` around read-modify-write,
   `atomic_write_json`, per-item `attempts` / `last_error` / `quarantined` / `terminal`, duplicate-logical-key rejection
   at read time, and a lock-free, non-mutating `snapshot_agent_publications_from_path`.

4. `src/sase/agents_sync/git_sync.py`: drop the now-tautological `item.kind == "agent_hood"` filter.

5. `src/sase/doctor/checks_agent_publication.py`: revert the `671999252` multi-kind diagnostics and the "queue is
   non-empty but not draining / axe daemon is not running" diagnostic, which describes a failure mode that no longer
   exists. **Preserve the unreadable-local-owner-manifest diagnostic added by `0e40decdc`** — it is not part of this
   epic and it is load-bearing for the corruption described in `repair`.

6. `src/sase/history/chat_catalog_provenance/sidecars.py`, `src/sase/ace/tui/agents_sync_format.py`,
   `src/sase/ace/tui/widgets/agents_sync_indicator.py`, `src/sase/agents_sync/status.py`: revert to agent-hood-only
   presentation without changing pre-epic behavior.

7. `src/sase/agents_sync/prompt_archive/validation.py`: delete `_queued_agent_hood_publications` and the "manifest run
   publication is queued" arm of `prompt-unpublished`, restoring the pre-`1116bccb0` message. `prompt-unpublished` and
   `plan-unresolved` stay at `warning` severity.

8. Out of scope, by design: `src/sase/sdd/_link_validation.py`, `src/sase/sdd/_link_support.py`,
   `src/sase/main/parser_plan.py`, `src/sase/main/plan_links_handler.py`, and the `--strict` flag. See "Deliberately NOT
   reverted".

9. Tests: `tests/agents_sync/test_publication_outbox.py`, `tests/agents_sync/test_status.py`,
   `tests/doctor/test_checks_agent_publication.py`, `tests/history/test_chat_catalog_publication.py`,
   `tests/ace/tui/test_agents_sync_indicator.py`, `tests/agents_sync/test_prompt_archive_validation.py`,
   `tests/test_committed_plan_validation.py`, `tests/ace/tui/visual/_ace_config_center_plugins_helpers.py`.

### Acceptance

- A test writes a **version-4 outbox file containing one `agent_hood`, one `bead_pages`, one `plan_header`, and one
  `sidecar_push` record**, loads it, and asserts the agent-hood request survives with an unchanged logical key while the
  other three are dropped with a diagnostic.
- Version 1–3 round-trip tests still pass unchanged.
- The concurrency test (two processes enqueueing at once loses and duplicates nothing) still passes.
- `sase doctor` reports a healthy empty queue and still reports quarantined/retired/stalled agent-hood requests.
- `just check` passes.

---

## repair: Repair the agents sidecar digest corruption blocking all publication

This phase is independent of the revert and can run in parallel with `chop` and `queue`. Nothing else in this epic
matters if publication still cannot succeed.

### Investigate first

Find the writer that produced agents sidecar commits `49bdd7996` (`chore(agents): revert stored chat prompt sections`)
and `a93a9660e` (`chore(prompts): revert stored prompt sections`). They rewrote published payload files in place without
re-signing the hood snapshots and owner manifest that sign them. The likely source is the work around `376a3b1bb`
(`feat(history)!: restore single-prompt chat markdown`), `92b31a1b4`
(`fix(prompt-archive)!: publish archived prompt bodies verbatim`), and `1239c5f5c`
(`feat(cli)!: remove stored prompt rendering surfaces`). Confirm before changing anything; do not guess.

### Changes

1. **Stop the bleed.** Make that rewrite path either re-sign the affected hood snapshots and owner manifest as part of
   the same transaction, or refuse to write and report. An in-place payload rewrite that leaves signatures stale must
   not be reachable.

2. **Provide a first-class repair, not a one-off script.** The current state cannot self-heal:
   `load_validated_publication` verifies every owner snapshot before publishing anything, so the stale `chop` hood
   blocks publication of `t2`, and the repair path is itself gated by the thing it needs to repair. Add a supported
   recovery — a `sase agent sync` repair mode is the natural home — that, for **locally owned** hoods only, re-derives
   each hood snapshot from the on-disk payload and re-signs the owner manifest, then commits and pushes one repair
   commit. It must:
   - only touch hoods owned by the local owner identity (`bbugyi200/athena` here); never re-sign a foreign owner's
     snapshot,
   - be idempotent and a no-op when digests already match,
   - run under the existing bounded agents lock,
   - report exactly which files it re-signed.

3. **Add a doctor check for payload/snapshot drift** in the local owner's hoods, with the repair command as its
   remediation. This is the missing signal: the drift existed for two days with nothing reporting it.

4. **Clear the residue** (operational, record what you did):
   - Re-verify that `0e40decdc` fixes `owner manifest has an invalid shape` before retrying anything.
   - Repair the digests, then clear the 18 quarantined requests in
     `~/.sase/projects/gh_sase-org__sase/agents-publication-outbox.json` via `sase agent sync -q -p sase`.
   - The quarantined `sidecar_push` record fails on
     `git rebase failed: ... could not apply 0d274ff6... chore(beads): checkpoint approved epic graph sase-ey` /
     `duplicate issue_created event for sase-ey`. Resolve the beads sidecar divergence explicitly and confirm the beads
     store is consistent; the `sidecar_push` record itself is dropped by phase `queue`'s v4 reader, so the underlying
     unpushed beads commits are what must actually land.
   - Delete any `sidecar_publication_backoff.json` left in the axe state dir.
   - **Do not** run a full sync against `bob-cli` — see scope decision 4.

### Acceptance

- A test seeds a sidecar whose payload was rewritten out of band, asserts publication fails before repair, runs the
  repair, and asserts publication then succeeds.
- A test asserts the repair refuses to re-sign a foreign owner's snapshot.
- A test asserts the repair is a no-op on a healthy sidecar.
- The doctor check fires on drift and is silent when clean.
- Live: the digest audit over the real agents sidecar reports **0 mismatches** (it reports 73 today), and
  `sase agent sync -p sase --json` reports no quarantined requests.
- `just check` passes.

---

## land: Docs, end-to-end verification, agents-repo sync, and bead bookkeeping

### Changes

1. Docs sweep for the prose the earlier phases did not own — every place that describes commit-time publication as
   queued or lane-owned: `docs/agents_sidecar.md` (lines ~72, ~116, ~238–240, ~257, ~272–274, ~322–329, ~358–360),
   `docs/beads.md` (~165, ~183, ~472–473, ~534, ~1085), `docs/commit_workflows.md` (~227), `docs/sdd.md` (~64, ~86),
   `docs/configuration.md` (~618–624, ~1263). Describe the restored inline behavior and the agent-hood retry outbox.
   Grep for `publications`, `sidecar_publication`, and "queued" afterwards to catch stragglers.
2. Do **not** touch `sase/memory/*.md`, `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`, or
   `CHANGELOG.md`. If a memory file describes the queue/lane, file a task bead through `/sase_new_task`.
3. End-to-end verification, captured as a SASE artifact via `/sase_artifact_file`:
   - Make a real commit carrying `SASE_BEAD`, `SASE_PLAN`, and a publication agent. Record wall-clock time.
   - Assert that when `sase commit` returns, the agents, beads, and plans sidecars have each been committed **and
     pushed**, and that the commit's own `SASE_AGENT` footer URL resolves over HTTPS.
   - **Timing guard:** the commit must stay in the seconds range. `sase-ej.6` measured 10.190s with 17,279
     dismissed-bundle files once scanfix was in place. If the restored synchronous path takes minutes, scanfix is not
     doing its job and that is a defect to fix in this phase, not an accepted cost. Confirm `~/.sase/dismissed_bundles`
     still holds a comparable archive so the measurement means something.
4. **Sync this project's agents repo.** After `repair` has cleared the corruption, run `sase agent sync -p sase` (with
   `-q` while quarantined requests remain) until:
   - `https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.t2.md` resolves,
   - the publication outbox for `gh_sase-org__sase` is empty,
   - `sase doctor` reports no agent-publication diagnostics,
   - the agents sidecar's `origin/main` has advanced past `49bdd7996`.

   Report the resulting agents-sidecar HEAD. Watch for `sase-f6`'s failure mode: if the full sync re-synthesizes retired
   identities from commit footers **in this project**, do not paper over it — record it and leave it to `sase-f6`.

5. Bead bookkeeping. Read `/sase_memory_read sase/memory/sase_beads.md` first for close, resolution, and note semantics.
   - Close `sase-ej` with a resolution stating plainly that the async design was reverted because it left `SASE_AGENT`
     footer URLs dangling, and that its `scanfix` phase was deliberately retained.
   - `sase-cl` stays closed; add a note recording that `scanfix`, not the chop, is what resolved it, so a future reader
     does not conclude the revert reopened it.
   - Confirm `sase-f6` (blocked on `sase-ej`) becomes unblocked.

### Acceptance

- `just check` passes.
- The recorded evidence shows a real commit publishing all three sidecars inline, within seconds, with a resolving
  footer URL.
- The agents repo for this project is fully synced and the `t2` family page resolves.
- `sase doctor` is clean.
- `sase-ej` is closed with the revert rationale; `sase-f6` is unblocked.

## Risks

- **Commits get slower again.** Publication is back on the interactive path, so a slow sidecar push is felt by the
  agent. This is the tradeoff the owner has explicitly chosen: a correct, resolving provenance link is worth the wait.
  Mitigated by retaining `scanfix` and by the phase `land` timing guard.
- **A hard sidecar outage now fails commits.** The pre-epic contract is restored on purpose: a publication that cannot
  be queued fails the commit with a `--resume` hint. The agent-hood outbox still absorbs transient failures as a warned,
  retryable queue. Do not soften this back into silence.
- **The corruption is worse than the 73 files found.** The audit covered signed file references in the sase project's
  agents sidecar only. Phase `repair` should re-run the same audit after repair, and should sanity-check whether other
  projects' agents sidecars carry the same out-of-band rewrite before declaring the class of bug closed.
- **On-disk v4 outboxes elsewhere.** Other projects' outboxes may hold non-`agent_hood` records too. The v4 reader must
  drop them everywhere, not just for this project, and the drop diagnostic is how that stays visible.
