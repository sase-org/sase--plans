---
tier: tale
title: Keep deferred prompt-archive sources reachable
goal:
  Monitor-turn commits restore their prompt archive from the starter turn. Dismissing,
  killing, or self-healing an agent no longer destroys a prompt source that an active
  publication row still needs. Together these stop the false
  prompt_archive_unrecoverable digest errors.
size: medium
proposed_by: bbugyi200.athena.1fj
proposed_with: claude/opus
create_time: 2026-11-23 09:45:36
status: wip
---

# Keep deferred prompt-archive sources reachable (monitor turns and dismissals)

## Problem

The `publication/agent_publication` chop reported a digest error on 2026-11-23 at 08:18:

```
gh_bbugyi200__tailnet: could not restore deferred prompt archive for
bbugyi200.athena.tailnet-1e.7.1.4.1.7.1.7.1.3@de476054ab9e: prompt archive is unresolved
and a missing local source does not prove there was no prompt obligation
```

One minute later the row was retired as `prompt_archive_unrecoverable` and recorded in
`~/.sase/agent_publication/terminal_obligations.jsonl`. The terminal machinery from
`sase-1tz.2.1.6` behaved as designed. The real problem is that the prompt source was
lost. Every terminal row recorded since 2026-11-20 (the current regime, where rows
retire roughly 30 minutes after creation) comes from one of two source-loss gaps that
earlier work did not cover:

### Cause A: dismissal deletes a dir that an active outbox row still needs (today's digest)

- At 07:48:32 the agent `tailnet-1e.7.1.4.1.7.1.7.1.3` (run `20261123053834`) committed
  into a linked repo. That commit enqueued a hood-publication row whose `artifacts_dir`
  is that run's dir. Every queued row also owns a deferred prompt-archive obligation;
  `git_sync` and `_commit_publication_transaction_steps` pass every pending row to
  `restore_deferred_prompt_archives`.
- The agent's finalizer then failed on a different repo, so the agent ended FAILED.
- At 07:49:29 the user dismissed it from ACE. `persist-cleanup` reported "artifact index
  sync failed", and the TUI loader self-heal (`compute_loader_cleanup`) deleted the
  artifacts dir. Every dismiss, kill, and self-heal path ends in
  `delete_agent_artifacts` (`src/sase/ace/tui/actions/agents/_killing_utils.py`), which
  calls the Rust `delete_agent_artifacts` binding. That binding removes the whole tree,
  including `raw_prompt.md`.
- The row then had no local source. It stayed "pending" during the 30-minute grace
  window, failed loudly at 08:18, and retired at 08:19.
- The prompt-artifact pool GC already refuses to collect files for revisions with
  non-terminal outbox rows (`src/sase/core/prompt_artifact_staging_pool.py`,
  `_active_publication_revisions`). Agent-artifact deletion has no equivalent guard.

### Cause B: monitor-turn host completions point the row at a dir that never has a prompt

Four of the other five recent terminal rows were committed by a monitor member's host
completion: `1f5--mon-2`, `1eu--mon`, `sase-1tz.3.1.5--mon-0`, and
`sase-1tz.4.1.10--mon-0`. The fifth, `sase-1ux.2--mon-2`, is also a monitor member, but
its dir was later deleted.

- The commit finalizer runs with `SASE_ARTIFACTS_DIR` set to the monitor member's dir.
  That dir has `agent_meta.json` with `turn_kind: monitor` and
  `monitor_starter_artifacts_dir`, but it never has `raw_prompt.md`. The prompt lives in
  the starter agent turn's dir; all four starter dirs still exist and hold
  `raw_prompt.md`.
- `deferred.py::_prompt_source_candidates` probes only the row's `artifacts_dir` and the
  inventory run's `source_label`. It never follows `monitor_starter_artifacts_dir`, so
  the restore finds no source.
- No session member had an archive (`sidecar archive present=False`), and the published
  snapshot shows a prompt file. The obligation therefore can never resolve, and it goes
  terminal after the grace window every time.
- The foreground commit-time path (`prompt_archive/publish.py::_publish_prompt_archive`)
  has the same blind spot. It skips with "agent raw_prompt.md is unavailable" instead of
  using the starter's prompt.

The result is about one false digest error per day and permanently missing prompt
archives for sessions that end in a monitor-driven commit.

## Goal

1. A row whose committing turn is a monitor member restores its prompt archive from the
   monitor's starter turn, at commit time and in the deferred/quarantine paths.
2. Dismissing, killing, or self-healing an agent never destroys a prompt source that an
   active publication row still needs. Deletion is deferred until the row is
   acknowledged, retired, or quarantined.
3. Archives that really cannot be reconstructed still retire as
   `prompt_archive_unrecoverable`. That path does not change.

## Design

### 1. Shared prompt-source resolver (new module)

Add `src/sase/agents_sync/prompt_archive/source.py` with:

- `prompt_source_dirs(artifacts_dir: str | Path) -> tuple[Path, ...]`
  - Returns the given dir first.
  - If that dir's `agent_meta.json` names a non-empty `monitor_starter_artifacts_dir`,
    appends that dir too.
  - Follows the same key from each appended dir for at most 3 hops, with a seen-set, so
    it stays cycle-safe and bounded.
  - Paths are `expanduser()`'d and deduplicated by `resolve(strict=False)`.
  - An unreadable or missing `agent_meta.json` simply ends the chain. It never raises.
  - Reuse `sase.legacy_xprompt_names` / an existing agent-meta JSON reader if one fits.
    `history/chat_fork/continuation/_source.py` follows the same key and shows the
    pattern.
- `first_prompt_source_dir(artifacts_dir) -> Path | None`
  - Returns the first dir from `prompt_source_dirs` for which
    `resolve_raw_prompt_path(dir)` is not `None`.

Key the resolver on the presence of `monitor_starter_artifacts_dir` rather than on
`turn_kind`. Only monitor members carry that key.

Gate turns are out of scope. They never commit, and their planner inputs already flow
through `prompt_archive_inputs.json`.

### 2. Use the resolver in the deferred, quarantine, and terminal paths

In `src/sase/agents_sync/prompt_archive/deferred.py`, rewrite
`_prompt_source_candidates` to expand each existing candidate through
`prompt_source_dirs`. The order is:

1. row `artifacts_dir`
2. its starter chain
3. run `source_label`
4. the starter chain of the `source_label`

Deduplicate while preserving order. Return `list[str]` as today.

This one change covers:

- `prepare_deferred_prompt_archive`
- `_local_prompt_source_present`
- `classify_quarantined_prompt_archives`. That function's `needs_runs` shortcut calls
  `_local_prompt_source_present(None, request)`. It must still see the starter, which it
  will because the starter expansion happens on the row dir.
- the deterministic `_terminal_prompt_failure` evidence, which will now list the starter
  dir it probed

`prepare_prompt_archive` then runs against the starter dir. That is the correct source
for every input it reads from that dir: `agent_meta.json` (workspace root and
`plan_ref`), the archive month derived from the path, the manifest records filtered by
artifacts dir, and `prompt_archive_inputs.json`.

### 3. Use the resolver in the foreground commit-time path

In `src/sase/agents_sync/prompt_archive/publish.py::_publish_prompt_archive`:

- Keep `artifacts_dir` (the committing turn's dir) as the outbox row's `artifacts_dir`.
  Other consumers key on that dir: the `done.json` gating in `_publication_job_shared`
  and the planner proof in `_publication_planner_scope_repair_classification`.
- Compute `source_dir = first_prompt_source_dir(artifacts_dir)`.
- Skip with "agent raw_prompt.md is unavailable" only when `source_dir` is `None`.
- Pass `source_dir` as `agent_artifacts_dir` to `prepare_prompt_archive`.

### 4. Defer artifact deletion while an active publication row needs the source

Add a pin-set helper, either in the same new module or a sibling such as
`prompt_archive/source_pins.py` if `source.py` grows.

- `active_prompt_source_pins() -> frozenset[str] | None`
  - Scans `sase_projects_dir().glob("*/agents-publication-outbox.json")` with the
    lock-free `snapshot_agent_publications_from_path`.
  - Considers only rows that are **not terminal and not quarantined**; parked rows
    count. Quarantined rows are excluded because they are legacy backlog (thousands of
    Aug–Oct rows on athena). They have their own quarantine-sweep path, and pinning them
    would keep their dirs for no benefit.
  - Adds every `prompt_source_dirs(row.artifacts_dir)` entry, normalized with
    `resolve(strict=False)`, so a monitor row also pins its starter.
  - Returns `None` if any outbox read fails.
  - Memoizes in-process on the tuple of `(path, st_mtime_ns, st_size)` for all outbox
    files. The sase outbox is about 3.5 MB, and the loader self-heal and bulk dismiss
    call deletion once per dir.
- `artifacts_dir_has_pending_prompt_obligation(artifacts_dir) -> bool`
  - Returns `True` when the normalized dir is in the pin set.
  - Fails closed: also returns `True` when the pin set is `None`, logged at debug level.
    This matches the pool-GC precedent, and month-based ACE-run retention still bounds
    disk use.

In `_killing_utils.delete_agent_artifacts`, after `before_delete` and
`_resolve_waiters_before_artifact_delete` and **before** `try_delete_agent_artifacts`:

- If the dir is pinned, log at info level and return `False`. Return before both the
  Rust delete and the loader-file strip fallback. The dir is left intact, the agent
  stays hidden through the dismissed-identity tombstone, and dismissal semantics such as
  waiter resolution are unchanged.
- Import the helper lazily, as the module already does, to keep TUI startup lean.
- The loader self-heal (`_loading_compute.compute_loader_cleanup`) already treats
  `completed is False` as "deferred, retry next load". The dir is therefore deleted on a
  later pass once the row is acknowledged, retired, or quarantined. No new sweeper is
  needed.
- Other callers ignore the return value. Check that each one tolerates a surviving dir:
  - `_dismiss_persistence.py` (three call sites)
  - `_kill_persistence.py`
  - their callers in `_kill_transactions.py`

  A failed `persist-cleanup` already produces exactly this state today, so they should.

- Confirm `_resolve_waiters_before_artifact_delete` is idempotent, because it will run
  again on the later self-heal deletion. If it is not, move the guard ahead of the
  pre-delete hooks and note why in a comment.

### Rust boundary note

The publication outbox store, the prompt-archive pipeline, and the analogous pool-GC pin
are Python-owned today. The Rust classifier `classify_deferred_prompt_obligation` keeps
the same inputs; it just sees `local_source_present=True` more often. No sase-core
change or `sase-core-revision.txt` bump is expected. If you find that the Rust
`delete_agent_artifacts` binding or `plan_agent_cleanup` already owns an equivalent
"retain" hook, use it instead and update the binding in the linked `sase-core` checkout
per the `rust_core_backend_boundary` memory.

## Tests

### Deferred prompt archive

Add `tests/agents_sync/test_deferred_prompt_archive/test_monitor_source.py`, reusing
`_helpers.py`:

- A row whose `artifacts_dir` is a monitor member (meta has
  `monitor_starter_artifacts_dir`, no raw prompt) and whose starter holds
  `raw_prompt.md`: `restore_deferred_prompt_archives` writes the archive and returns no
  failure.
- Same setup with the starter dir missing: the failure is still pending inside grace,
  and past grace with attempts exhausted it is terminal. The terminal evidence lists
  both the monitor dir and the starter dir.
- `classify_quarantined_prompt_archives` does not report a monitor row as unrecoverable
  when its starter still has the prompt.
- Cycle and hop-limit safety for `prompt_source_dirs`, and unreadable meta ends the
  chain.

### Commit-time publish

In `tests/agents_sync/test_prompt_archive_publish.py`, foreground
`publish_prompt_archive` from a monitor member dir publishes the starter's prompt
instead of skipping. The queued row keeps the monitor dir as `artifacts_dir`.

### Dismissal and kill

In `tests/test_agent_dismiss_artifact_cleanup.py` (or a new sibling if it would exceed
the toobig thresholds):

- A dir named by an active row survives `delete_agent_artifacts`, keeps `raw_prompt.md`,
  and the call returns `False`.
- A dir named by a terminal row is deleted. So is a dir named by a quarantined row, and
  a dir with no row.
- A monitor row pins its starter dir.
- An unreadable outbox defers deletion (fail-closed).
- The memo invalidates when the outbox file changes.

### Loader self-heal

In `tests/test_agent_loader_self_heal_cleanup.py`, a dismissed loader-sourced agent with
a pinned dir is not added to `cleaned_dirs`. After the row is removed or made terminal,
a later pass deletes it.

### End-to-end regression

Enqueue an active row, dismiss the agent through `delete_agent_artifacts`, then run
`restore_deferred_prompt_archives`. The archive is written and nothing goes terminal.

## Docs

Before editing, read the `docs_writing` reference memory. In
`docs/reference/agents/hood-sync.md`, section "Unrecoverable prompt archives", add two
points:

- The restore also probes a monitor turn's starter directory.
- Dismissing or killing an agent defers deleting its artifacts directory while a
  non-terminal, non-quarantined publication request still needs it as a prompt source.

Mirror the sentence in `docs/contribute/agents-sidecar-sync.md` if it describes the
probe set.

## Verification

- Run the repo's standard check recipe per the `lint_and_test` memory.
- Read-only sanity check against live state: call
  `prompt_source_dirs("~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202611/22/20261122111618")`.
  It should yield the starter `.../20261122110913`, which has `raw_prompt.md`.

## Out of scope / follow-ups

- **Recovering already-retired rows.** This is human-run after the fix is installed.
  Four retired monitor rows still have their starter prompts on disk:
  - `bbugyi200.athena.1f5@8cfdf6c2ee1488a312d29af8596957ee2eda5386`
  - `bbugyi200.athena.1eu@dfc086636041669f393f232df4cad06836b2ed58`
  - `bbugyi200.athena.sase-1tz.3.1.5@f9b88003413260bc1c85ecb39ea1d794a6d17db5`
  - `bbugyi200.athena.sase-1tz.4.1.10@335170e1cb0b5b2d597fd43994f30eff784ac126`

  They can be retried one at a time, if the Rust retry policy selects
  prompt-unrecoverable rows:

  ```bash
  sase agent sync --retry-retired --retry-limit 1 --retry-key '<key>' -p sase
  ```

  Two rows cannot be recovered:
  - The tailnet `.3` row: its source dir was deleted.
  - `sase-1ux.2`: its monitor dir was deleted, so its starter can't be resolved.

- **The dismiss failure.** The `persist-cleanup` "dismissed-agent artifact index sync
  failed" error at 07:49:39 is a separate issue. It did not cause the loss; the loader
  self-heal would have deleted the dir anyway.
