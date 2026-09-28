---
tier: tale
title: Give the commit finalizer's discarded-work guard a real "before" HEAD
goal:
  The finalizer's discarded-work guard compares each repo against the HEAD it had when
  the dirty state was scanned, not a HEAD read after dispatch. Shared-checkout races
  stop failing agents whose work landed, and main/sibling discards are still caught.
size: medium
proposed_by: bbugyi200.athena.0tj
create_time: 2026-09-28 08:37:57
status: wip
---

# Give the commit finalizer's discarded-work guard a real "before" HEAD

## Incident

`sase-1bu.5--2` (2026-09-28) finished its phase. Both of its commits landed and were
pushed: main `f79a391b53` ("feat(goals): add sase goal CLI …") and sase-core `d2d9ec7f`.
Bead `sase-1bu.5` was closed at 10:38Z. The run was still marked FAILED, and workspace
#11 stays held:

```text
Commit finalizer failed: dirty work vanished without an attributable commit.
- agents prompt archive: ~/.sase/projects/gh_sase-org__sase/repos/agents (HEAD did not advance)
  - HEAD: 4de7d4a092 -> 4de7d4a092
  - changed files (2):
    - prompts/202609/README.md
    - prompts/202609/land_agent_tabs_scope_repairs.md
```

Two separate defects combined here:

1. **Foreign scratch counted as this run's work.** The two files were orphaned
   agents-sync scratch from another agent (`0tf--plan`). Its publication failed on
   `index.lock`, and so did the cleanup. The finalizer offered them to `sase-1bu.5--2`
   as a required obligation, so the agent had to declare a commit for them. That defect
   is **already being fixed** by the approved plan
   `plan:202609/prompt_archive_finalizer_contention.md`, which agent `0th--code` is
   implementing now. **This plan does not touch it.**
2. **The guard misreports a HEAD that did advance (this plan).** The finalizer ran from
   06:39:18 to 07:16:38 local. The long dispatch included a main-repo conflict repair.
   During that window three agents-sync publications committed to the agents checkout:
   - `7e2ddccc3c` at 06:58:34
   - `835ba7b93f` at 07:09:16
   - `4de7d4a092` at 07:09:51

   Each publication starts with `clean_prompt_archive_worktree`, which removed the
   orphaned scratch. So HEAD **did** advance (`3d1a109db4` → `4de7d4a092`). If the guard
   had seen that, the new commits would have gone through
   `_new_commits_are_attributable` and then `_published_store_state_is_exempt`. For a
   `kind == "sdd"` checkout that classifies the change as a shared-clone race or
   publication, and the run does not fail. Instead the guard reported "HEAD did not
   advance".

## Root cause (regression from `2f9c4ae295`, 2026-08-21)

`discarded_dirty_work_evidence(before, after, *, fingerprint_before=None, …)` in
`src/sase/llm_provider/commit_finalizer_git_progress.py` gets the "before" HEADs from
`fingerprint_before`. When that is omitted, it falls back to
`progress_fingerprint(before)`, which runs `git rev-parse HEAD` **at call time**, i.e.
after dispatch. So `before_head` always equals `after_head`.

Before the pluggable-finalizer rewrite, the legacy loop captured
`fingerprint_before = progress_fingerprint(dirty_state)` before each pass and passed it
in. `2f9c4ae295 feat(finalizers)!: make pluggable finalizers the only completion path`
dropped that. The only production caller, `reject_discarded_dirty_work` in
`src/sase/finalizers/commit_validation.py`, never passes `fingerprint_before`. It has
two call sites, both in `execute_commit_finalizer` in `src/sase/finalizers/commit.py`:

- the `already_clean` check;
- the post-dispatch check. This is the one that failed `sase-1bu.5--2`, at `commit.py`
  ~line 381.

Consequences in production:

- Every repo that went clean without a run-owned commit marker is reported as
  `head_not_advanced`, even when HEAD moved.
- The shared-clone race/published exemption (`_published_store_state_is_exempt`, the
  sase-p5 "Cluster B" design) cannot be reached. Neither can
  `_new_commits_are_attributable` / `missing_agent_provenance`.
- Existing tests don't catch this. Every exemption test calls
  `discarded_dirty_work_evidence` directly with an explicit `fingerprint_before`.
  Examples: `tests/llm_provider/test_commit_finalizer_sdd_publication_exempt.py`,
  `tests/llm_provider/test_commit_finalizer_shared_clone_audit.py`,
  `tests/llm_provider/test_commit_finalizer_hidden_agents_sidecar.py`, and
  `tests/test_plan_approval_launch_reliability_integration.py`. So the helper is tested
  but the production path is not.

This keeps recurring. In each past "dirty work vanished" failure on a shared store, the
reported HEAD is a commit that landed _inside_ that finalizer's own window:

| Run                             | Repo                      | HEAD at failure, landed mid-finalize             |
| ------------------------------- | ------------------------- | ------------------------------------------------ |
| `0ak--2` (08-22)                | `plans`                   | `9b2c4855` at 09:36:52, window 09:36:46–09:36:55 |
| `sase-zl.13.11.land--1` (09-13) | `plans`                   | many plan-archive commits during 18:37–19:16     |
| `0gl--code` (09-05)             | `sase-research-artifacts` | `abdaf1f` inside its 19:12–19:13 window          |
| `sase-1bu.5--2` (09-28)         | `agents prompt archive`   | the three publication commits above              |

## Changes

### 1. Capture pre-dispatch HEADs next to `dirty_before_decisions`

`src/sase/finalizers/commit.py` — `execute_commit_finalizer`:

- Right where `dirty_before_decisions = state.dirty_state` is assigned, capture
  `fingerprint_before_decisions = progress_fingerprint(dirty_before_decisions)`.
  `progress_fingerprint` is already exported from
  `sase.llm_provider.commit_finalizer_git`.
- Keep the two paired: later `state = prepare_commit_dirty_state(...)` re-assignments in
  the checkpoint-recovery and `already_clean` branches do not change
  `dirty_before_decisions`, so they must not change the fingerprint either.
- Pass `fingerprint_before=fingerprint_before_decisions` to the post-dispatch
  `_reject_discarded_dirty_work(dirty_before_decisions, state.dirty_state, …)` call.

`src/sase/finalizers/commit_validation.py` — `reject_discarded_dirty_work`:

- Add a **required** keyword parameter
  `fingerprint_before: tuple[tuple[str, str, tuple[str, ...]], ...]`, with no default.
  Forward it to
  `discarded_dirty_work_evidence(..., fingerprint_before=fingerprint_before)`. Making it
  required means a future call site cannot silently fall back to the lazy capture again.
- Add a short docstring saying the HEADs must be captured when `before` was scanned, and
  why: a lazy capture compares HEAD with itself and disables the shared-clone exemption.

`src/sase/llm_provider/commit_finalizer_git_progress.py`:

- Leave the `fingerprint_before=None` fallback in `discarded_dirty_work_evidence` in
  place for its direct callers.
- Extend its docstring: omitting `fingerprint_before` makes before and after HEADs
  identical, so finalizer callers must always pass a snapshot taken at scan time.

### 2. Give the `already_clean` check a declaration-time HEAD

The `already_clean` repos were dirty when the agent's declaration context was built and
were clean by the time the finalizer started. Their "before" HEAD is the HEAD at context
time, which nothing records today.

`src/sase/finalizers/declaration_store.py`:

- Add `head: str | None = None` to `HostRepositoryRecord`.
- `host_repository_records(dirty_state)` fills `head = git_head_commit_id(repo.path)`,
  storing `None` when the result is `UNKNOWN_HEAD_SENTINEL`. The import comes from
  `sase.llm_provider.commit_finalizer_git` or its status module.
- `write_host_repository_file` writes `"head"`.
- `read_host_repository_file` accepts a missing or `null` `head`. Legacy files load with
  `head=None`. A non-string `head` is rejected with the same
  `malformed_host_repositories` error the other fields use.
- **Do not** add `head` to `_host_repository_record_set` in
  `src/sase/finalizers/declaration.py`. That set decides context staleness at submit. A
  foreign commit landing on a shared store between `sase final context` and
  `sase final submit` must not make the declaration stale. Add a one-line comment there
  saying so.

`src/sase/finalizers/commit.py` — `already_clean` branch:

- Build the `fingerprint_before` passed to that `_reject_discarded_dirty_work` call from
  the accepted host records. The `host_records` are already loaded by
  `_load_accepted_commit_declaration`, and `accepted_repos_from_host` in
  `src/sase/finalizers/commit_declaration.py` maps obligation id → record.
- For each already-clean repo, the entry is
  `(repo.path, <before head>, tuple(sorted(repo.changed_files)))`. `<before head>` is
  the matching record's `head`. If the record has no `head` (declaration files written
  before this change), use the repo's current `git_head_commit_id`, which is exactly
  today's behaviour. Keep that fallback in one small helper next to the call site.

### Behaviour this restores (no policy change)

- `kind == "sdd"` / `"external"` shared checkouts that went clean because HEAD advanced
  with commits not attributable to this run are exempt. They are classified and logged
  as `race` or `published` by the existing `_record_shared_clone_classification`.
- `kind == "main"` / `"sibling"` stay strict:
  - HEAD advanced with unattributable commits → `missing_agent_provenance` failure.
  - HEAD did not move → `head_not_advanced` failure.
- The `proven` filter in `reject_discarded_dirty_work` (new run-owned commit markers,
  and unpushed markers) is unchanged.
- A shared checkout whose dirt vanished while HEAD truly stayed put still fails as
  before. Stopping the agents archive from claiming foreign scratch at all is the
  in-flight plan's job.

## Tests

Use real temporary git repos, as the neighbouring tests do.

- **Unit, `reject_discarded_dirty_work`.** Add these to a new focused test module (e.g.
  `tests/test_finalizers_discard_guard_before_head.py`). Each captures the fingerprint
  while the repo is dirty, then:
  - `kind="sdd"` repo: a foreign commit with body trailer `SASE_AGENT=other-agent`
    cleans it. Expect no raise.
  - `kind="main"` repo: the same foreign commit. Expect `BuiltinCommitFinalizerError`
    with code `dirty_work_discarded`, where the message does **not** say "HEAD did not
    advance". This proves the reason is now `missing_agent_provenance`.
  - `kind="main"` repo: `git checkout -- .` with HEAD unchanged. It still raises, and
    the message does say "HEAD did not advance".
- **Finalizer-level regression (the incident).** Use the live controller harness in
  `tests/test_finalizers_live_e2e_cycles.py` /
  `tests/finalizers_live_e2e_test_helpers.py` (`init_live_repo`, `mark_opened_external`,
  `submit_from_context`, `run_controller`).
  - Setup: dirty main repo plus a dirty opened external repo (`kind="external"`), both
    declared `commit`.
  - The stitch runner for main commits main and records its marker as usual. As a side
    effect it simulates a foreign agent committing the external repo's dirty file with a
    `SASE_AGENT=other-agent` trailer.
  - Expect the controller to finish without `dirty_work_discarded`. Before the fix this
    fails with "HEAD did not advance".
  - Check how the dispatcher treats a declared repo that is already clean at its turn,
    and assert on the observable result.
- **Finalizer-level, `already_clean` path.** After `submit_from_context`, and before
  `run_controller`, a foreign `SASE_AGENT=other-agent` commit cleans the external repo.
  Expect the controller not to fail with `dirty_work_discarded`.
- **Negative control at finalizer level.** A main repo whose dirty file is reverted with
  no commit during another repo's stitch still fails with `dirty_work_discarded`.
- **Host records**, next to existing declaration-store tests:
  - `host_repository_records` records `head` for a real repo.
  - Write/read round-trips `head`.
  - A legacy file without `head` reads as `None`.
  - A non-string `head` is rejected as `malformed_host_repositories`.
  - Two record sets that differ only in `head` compare equal under
    `_host_repository_record_set` (no false staleness).
- Existing direct `discarded_dirty_work_evidence` tests and
  `tests/test_finalizers_commit_reconciliation.py` must keep passing unchanged. If a
  test that mocks `collect_dirty_state` / `git_head_commit_id` now needs a HEAD stub,
  give it one rather than loosening the new required parameter.

Run `just check` before finishing, per the lint/test memory.

## Out of scope

- **Agents prompt-archive ownership and `index.lock` contention.** Covered by the
  approved, in-flight `plan:202609/prompt_archive_finalizer_contention.md`. Don't edit
  what it changes: `llm_provider/commit_finalizer_state/_dirty_repos.py`,
  `llm_provider/commit_finalizer_git_autocommit.py`,
  `llm_provider/commit_finalizer_git_status.py`, `src/sase/agents_sync/**`,
  `tests/llm_provider/test_commit_finalizer_hidden_agents_sidecar.py`. The two changes
  touch different files and can land in either order.
- **Planner prompt-archive loss after a failed publication.** Tracked by task bead
  `sase-1c0`.
- **Any change to the shared-clone exemption policy itself.** This plan only makes the
  existing, tested policy reachable again.
