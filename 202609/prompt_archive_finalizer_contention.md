---
tier: tale
title: Stop foreign prompt-archive scratch from failing agents' commit finalizers
goal:
  An agent's commit finalizer never fails on agents-prompt-archive files it did not
  write, read-only finalizer scans stop contending for index.lock, and prompt-archive
  publication retries transient index.lock failures instead of orphaning scratch.
size: medium
proposed_by: bbugyi200.athena.0th
create_time: 2026-09-28 06:38:05
status: wip
---

# Stop foreign prompt-archive scratch from failing unrelated agents' commit finalizers

## Incident

`research.2v.final` (2026-09-28) finished its work: the research stitch commit and the
artifact-link commit both landed and are on the research sidecar's `origin/main`. It was
still marked FAILED:

```text
Commit finalizer failed: uncommitted changes remain after 1 finalizer pass(es) in
agents prompt archive=~/.sase/projects/gh_sase-org__sase/repos/agents:
prompts/202609/README.md, prompts/202609/land_agent_tabs_scope_repairs.md
```

Neither file came from that agent. Its own prompt archive commit had already landed at
06:14:22. The dirt came from a different agent (`0tf--plan`) whose planner
prompt-archive publication ran at 06:19:01:

1. `publish_prompt_archive` took the agents-sync lock, cleaned the tree, pulled, and
   `prepare_prompt_archive` wrote `prompts/202609/land_agent_tabs_scope_repairs.md` plus
   a new row in `prompts/202609/README.md`.
2. `git add` (op `agents_sync.prompt_archive_stage`) failed with
   `Unable to create '.../agents/.git/index.lock': File exists`.
3. The cleanup in the `finally` block (`clean_prompt_archive_worktree` → `git reset`, op
   `agents_sync.prompt_archive_reset`) failed 23 ms later with the same `index.lock`
   error. Its return value is ignored, so the half-written archive stayed in the shared
   checkout.
4. At 06:20, `research.2v.final`'s commit finalizer ran `collect_dirty_state`.
   `dirty_agents_prompt_archive_repo` reports _any_ dirty `prompts/<YYYYMM>/*.md` in the
   machine-wide agents checkout, and `README.md` matches `is_prompt_archive_path`. The
   finalizer attributed the other agent's orphaned files to this run. The files were not
   Q&A-only edits (one untracked, one index edit), so the Q&A auto-commit declined them
   and the finalizer failed with `dirty_after_commit_decisions`.

Evidence is in `~/.sase/logs/tui_git_ops.jsonl`. In roughly the last day the agents
checkout logged about 14 `index.lock` collisions (`prompt_archive_stage`,
`prompt_archive_commit`, `prompt_archive_reset`, `payload_reset_index`,
`payload_restore`). Three of them (09-27 17:36:34, 09-27 17:49:54, 09-28 06:19:01)
failed a stage or commit and then the cleanup, which orphans scratch exactly as above.
The primary beads checkout shows the same pattern (7 `sdd.clone.rebase` `index.lock`
failures).

The most likely lock holders are the read-only scans. Every agent's run-start baseline
and final dirty scan runs plain `git status --porcelain=v1 --untracked-files=all` on
each shared checkout (agents archive, primary beads/plans/research SDD checkouts,
sibling primaries). Without `--no-optional-locks`, `git status` holds `index.lock` for
the whole scan so it can write back refreshed stat data. On the agents checkout (about
115k tracked files, 15 MB index, 662 KB `README.md`) a status takes about 0.3 s, and
many agents do this concurrently.

## Root causes

1. **Ownership (primary).** The commit finalizer treats the machine-wide agents prompt
   archive as this run's dirty work. The only uncommitted edit a run intentionally
   leaves there is a question-gate Q&A snapshot. `followup.py` writes Q&A into the
   canonical prompt file and relies on the finalizer's Q&A auto-commit. Everything else
   there is agents-sync publication scratch:
   - It is written only under the agents-sync lock.
   - It is always regenerable from the local artifact pool.
   - It is durably queued in the publication outbox before any attempt.

   An agent can never meaningfully "commit" another agent's publication scratch. The
   existing `test_foreign_agent_commit_in_shared_sidecar_is_a_race_not_a_discard` and
   the sase-p5 "Cluster B" note already treat this checkout as machine-wide shared
   state.

2. **Reader contention.** The finalizer's read-only status helpers take git's optional
   index lock on shared checkouts.
3. **Non-resilient publication.** The agents-sync git runner has no `index.lock` retry.
   The shared `sase.git_lock_retry.run_with_git_lock_retry` policy exists but is not
   used there, and the `finally` cleanup failure is silently discarded.

Fixing only #1 stops the false failures. #2 and #3 remove the contention that orphans
scratch (and that sometimes defers publications), so all three belong in one change.

## Changes

### 1. Finalizer claims only provable Q&A edits in the agents prompt archive

`src/sase/llm_provider/commit_finalizer_state/_dirty_repos.py` —
`dirty_agents_prompt_archive_repo`:

- Keep resolving the archive checkout exactly as today.
- From its status records, keep only paths that are a tracked, modified (`" M"`)
  canonical prompt file whose worktree-vs-`HEAD` diff is provably Q&A-only. Reuse the
  existing prover `_has_only_sdd_prompt_qa_diff` in
  `src/sase/llm_provider/commit_finalizer_git_autocommit.py`. Promote it to a
  non-underscore shared helper (e.g. `has_only_prompt_qa_diff`) exported via
  `commit_finalizer_git.py` rather than duplicating it.
- Drop the current "every changed file must be a prompt path, otherwise report nothing"
  rule. Report a `DirtyRepo` named `agents prompt archive` whose `changed_files`
  contains only the proven Q&A paths, or report nothing when there are none.
- Never report other archive dirt as this run's work: untracked prompt files,
  `prompts/<YYYYMM>/README.md` index edits, non-Q&A modifications, and anything under
  `artifacts/` or `files/objects/`. Log it once at debug level, with repo and paths, as
  "agents-sync publication scratch left for the publisher". Do **not** clean or commit
  it from the finalizer. The next publication's leading `clean_prompt_archive_worktree`
  owns that, under the agents-sync lock.
- Update the docstring to state the ownership rule and why: the checkout is machine-wide
  and host-owned, and publications are durable through the outbox.

`src/sase/llm_provider/commit_finalizer_git_autocommit.py`:

- `sdd_prompt_qa_auto_commit_candidates`: stop requiring the repo's full status-path set
  to equal `repo.changed_files`. Require that every path in `repo.changed_files` has an
  `" M"` record, is a prompt-archive path, and is proven Q&A-only. Unrelated records in
  the same checkout, such as foreign scratch, no longer disqualify the candidate. The
  commit is already pathspec-limited (`git commit -- *paths`), so foreign files are
  never swept in.
- `auto_commit_sdd_prompt_qa_candidate`: after acquiring the agents-sync lock and before
  `git add`, re-prove each path is still `" M"` and Q&A-only. The first proof ran
  outside the lock, and a publication may have committed or rewritten the file since.
  Skip paths that are no longer dirty. Return `False` without committing if any
  remaining path fails the proof.

Leave `is_prompt_archive_path` and `_baseline_agents_prompt_archive_repo` unchanged. The
Q&A prover already excludes `README.md`, and baseline fingerprints stay harmless. Leave
`dirty_sdd_store_repos` (plans/beads/research SDD stores) unchanged: those stores keep
their existing strict behavior.

### 2. Read-only finalizer status scans stop taking git's optional index lock

`src/sase/llm_provider/commit_finalizer_git_status.py`:

- `git_changed_files` and `git_status_records` run
  `git --no-optional-locks -C <repo> status --porcelain=v1 --untracked-files=all`.
  `--no-optional-locks` is a top-level git option and must come before the subcommand.
- These two helpers back run-start baseline capture (`dirty_path_fingerprints`),
  opened-repo baselines, every dirty scan, and the Q&A candidate check, so this one
  change covers every per-agent scan of the shared checkouts.
- Extend the module docstring: read-only queries must pass `--no-optional-locks` so
  scans of shared checkouts never contend with writers for `index.lock`.

Do not sweep other modules' `git status` callers in this change. See Follow-ups.

### 3. Agents-sync git survives transient `index.lock` contention and reports orphaned scratch

`src/sase/agents_sync/git.py` — `run_git`:

- For `network=False`, wrap the `run_sdd_git` call in
  `sase.git_lock_retry.run_with_git_lock_retry(lambda: ..., cwd=cwd)`. This uses the
  default delays and existing stale-lock policy, and `CompletedProcess` works with the
  default result adapter.
- Git performs no work when it cannot create `index.lock`, so re-running any local
  agents-sync command (add/commit/reset/checkout/clean/ls-files/rebase --abort) after a
  lock failure is safe.
- Leave `network=True` (pull --rebase / push) unwrapped. Rebase failures already go
  through `abort_agents_rebase`.

`src/sase/agents_sync/prompt_archive/publish.py` — `_publish_prompt_archive`:

- In the `finally` block, capture `_clean_prompt_archive_worktree`'s return value. If it
  is an error string, `log.warning` it with the sidecar path and a note that
  prompt-archive scratch may remain until the next publication. Today the failure is
  silently dropped.
- Do not change the returned outcome's success fields.

## Tests

Update or add tests next to the existing ones. Use real temporary git repos as the
existing helpers do.

- `tests/llm_provider/test_commit_finalizer_hidden_agents_sidecar.py`:
  - Replace `test_agents_prompt_archive_dirty_state_has_single_agents_entry` with the
    new ownership rule. An untracked `prompts/<YYYYMM>/<name>.md` plus a modified
    tracked `prompts/<YYYYMM>/README.md` in the archive produce **no** agents-archive
    `DirtyRepo`.
  - Add a case where a tracked prompt carries a Q&A-only edit (via `set_prompt_qa`)
    alongside that foreign scratch. Assert exactly one `agents prompt archive` entry
    whose `changed_files` is only the Q&A path.
- `tests/llm_provider/test_commit_finalizer_auto_sdd_qa.py`:
  - A Q&A-only archive edit coexisting with an untracked foreign prompt and a README
    index edit is auto-committed. The commit contains only the Q&A path, and the foreign
    scratch is still present and uncommitted afterwards.
  - The re-proof under lock rejects a path whose content became non-Q&A between the
    candidate scan and the commit (simulate by editing the file before calling
    `auto_commit_sdd_prompt_qa_candidate`).
  - Keep `test_unsafe_external_sdd_changes_use_normal_finalizer_prompting` passing
    unchanged. It exercises a configured external SDD store, not the agents archive.
- Incident regression at the finalizer entry point used by existing finalizer tests
  (e.g. the `prepare_commit_dirty_state` / `_prepare` harness): main workspace clean,
  agents archive dirty only with foreign scratch → dirty state is clean and no
  `dirty_after_commit_decisions` failure is produced.
- `commit_finalizer_git_status` helpers: monkeypatch `subprocess.run` and assert
  `git_changed_files` and `git_status_records` pass `--no-optional-locks` before
  `status`.
- `tests/agents_sync/`: `run_git` with a monkeypatched `run_sdd_git`.
  - It returns an `index.lock` "File exists" failure (returncode 128) once, then
    succeeds. Assert success and two calls, with `SASE_GIT_LOCK_RETRY_DELAYS=0` to keep
    it fast.
  - The same failure with `network=True` is not retried.
- `tests/agents_sync/test_prompt_archive.py`: when the final
  `clean_prompt_archive_worktree` returns an error (inject a failing `git_runner` for
  the `reset` op after a failing stage), `publish_prompt_archive` still returns its
  queued outcome and a warning is logged (use `caplog`).

Run `just check` per the lint/test memory before finishing.

## Out of scope / follow-ups

- **Planner prompt-archive loss on failed publication.** For planner archives
  (`publish_planner_prompt_archive`), `primary_revision` is the workspace `HEAD` at plan
  acceptance, not a commit of the planner run. So `prompt_runs_by_request` in
  `src/sase/agents_sync/prompt_archive/deferred.py` cannot match the queued request.
  Even when it matches, `prepare_deferred_prompt_archive` drops
  `plan_ref`/`prompt_name`/`yyyymm`. A failed planner publication (like `0tf--plan`'s
  above) is therefore effectively lost once the next publication cleans the worktree.
  Tracked separately as task bead `sase-1c0`; not part of this tale.
- **Q&A snapshot committed at write time.** `question_gate_turn/followup.py` could
  commit its canonical-archive Q&A edit under the agents-sync lock, as the legacy
  plans-sidecar branch already does. Today any concurrent publication's leading cleanup
  can wipe an uncommitted Q&A edit before a finalizer commits it. After this tale the
  finalizer path still works as the fallback.
- **Broader `--no-optional-locks` sweep.** Other read-only `git status` callers over
  shared primary SDD checkouts (e.g. `sdd/_repository_transaction.py`,
  `sdd/_link_parent.py`, artifact-link publication retry) likely contribute to the beads
  `index.lock` rebase failures.
- Current state: the orphaned `0tf--plan` scratch was still uncommitted in the agents
  checkout at planning time. The implementer must not touch it. It is live shared state
  for the user to decide on.
