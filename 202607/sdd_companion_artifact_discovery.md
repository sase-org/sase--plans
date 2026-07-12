---
create_time: 2026-07-09 13:04:12
status: done
prompt: 202607/prompts/sdd_companion_artifact_discovery.md
tier: tale
---
# Plan: Discover agent artifacts & attachments from the separate SDD companion repo

## Problem / product context

Agents that produce `sdd/research/**.md` files (and images that accompany them, e.g. research infographics) **no longer
list those files as artifacts** on their completion entry. The same root cause also breaks **`sase-telegram`**: images
an agent adds/changes are no longer attached to the Telegram agent-completion message.

This regression started when SDD artifacts migrated from **in-tree** storage (`<workspace>/sdd/…`, inside the main code
repo) to a **separate companion repository** (`separate_repo` storage), where SDD content lives at
`<workspace>/.sase/sdd/…` — a _distinct_ git repo that is also git-ignored by the main repo (`.gitignore` contains
`.sase/`).

### Root cause (confirmed)

Completion-time artifact/attachment discovery is git-scoped to **only the main workspace repo**:

- `src/sase/axe/run_agent_exec_finalize.py` → `_collect_default_artifacts()` calls `collect_agent_markdown_paths()`,
  `collect_agent_image_paths()`, `collect_agent_video_paths()` with `workspace_dir = ctx.workspace_dir` (the main code
  repo).
- `src/sase/axe/image_attachments.py` → `_collect_agent_attachment_paths()` finds files from four sources, all run with
  `cwd=workspace_dir`:
  - `git diff --name-status -z HEAD` (local tracked changes)
  - `git ls-files --others --exclude-standard -z` (untracked; honors `.gitignore`)
  - a saved main-repo diff file
  - optionally `git diff HEAD~1..HEAD` (last commit), gated on a main-repo commit/PR
- Markdown found this way is rendered to PDF (`markdown_pdf_paths` on `done.json`) and later synthesized into `pdf`-kind
  artifacts (`src/sase/core/agent_artifact_defaults.py`, `synthesize_default_agent_artifacts`, whose only route for
  research `.md` is via `markdown_pdf_paths`). Images/videos found this way populate `done.json` `image_paths` /
  `video_paths`, which flow into `Notification.files` → the mobile/Telegram attachment manifests
  (`send_completion_notification` → `notify_workflow_complete` →
  `_mobile_notification_snapshot`/`_mobile_notification_attachments`).

Because `.sase/sdd/**` is (a) inside a nested/separate git repo and (b) excluded by `.sase/` in the main repo's ignore
rules, none of the four sources see research files or SDD images. So no PDF is rendered, no `pdf` artifact is listed,
and no SDD image is attached to Telegram. Fixing `_collect_default_artifacts` therefore repairs **both** the
artifact-listing bug and the `sase-telegram` attachment bug with one change (as suspected in the request).

### Critical timing detail (confirmed)

The separate-repo SDD store is **auto-committed before finalization**: `src/sase/llm_provider/commit_finalizer.py` →
`_auto_commit_separate_sdd_store_if_possible()` ("chore(sdd): sync uncommitted SDD store changes") runs inline in the
LLM invoke path (`run_commit_finalizer` is called after every `provider.invoke`), which completes **before** the exec
loop breaks and `finalize_loop()` runs. It fires on **every** agent completion, not just when there are dirty code
changes.

Consequence: by the time artifacts are collected, the companion repo's **working tree is clean** (research files are
already committed). A plain `git status`/`ls-files --others` scan of the SDD repo would therefore find **nothing**. We
must diff a **commit range** in the SDD repo, not just the working tree.

To bound that commit range to _this run's_ commits, we capture the SDD companion repo's `HEAD` SHA at **agent start** (a
"base") and, at finalize, collect files changed in `base..HEAD` (plus any still-uncommitted working-tree changes, to
cover cases where the auto-commit is disabled, e.g. `SASE_DISABLE_COMMIT_STOP_HOOK`).

## Goals

1. Research `sdd/research/**.md` files produced by an agent are once again listed as artifacts (via the existing
   markdown → PDF → `pdf`-artifact pipeline).
2. Images (and videos) an agent adds/changes in the SDD companion repo are attached to completion notifications,
   restoring `sase-telegram` attachments.
3. No change to behavior for in-tree SDD storage or non-SDD files.
4. Robust to the pre-finalization SDD auto-commit (must catch _committed_ research, not just uncommitted).

## Non-goals

- Fixing every other main-repo-scoped change detector the investigation surfaced (TUI "Deltas" pane `get_agent_diff`,
  `commit_finalizer_state.collect_dirty_state`, commit instructions). Those are separate concerns and out of scope here.
- Surfacing SDD _plans/beads_ as generic artifacts — plans are already surfaced via `plan_path`; beads are intentionally
  not artifacts.
- Any Rust/`sase-core` change. Attachment discovery and SDD storage-policy resolution are already pure Python in this
  repo; this fix stays in Python, consistent with where the logic lives. (Noting the Rust-core boundary explicitly: this
  is presentation/notification glue over already-Python helpers, not shared domain logic being reimplemented.)

## Design

### A. Capture the SDD companion base SHA at agent start

Add a small helper that, given `workspace_dir` + `workspace_num`, resolves the SDD store (`resolve_sdd_store`) and, when
storage is `separate_repo` or `local` and the resolved `repo_root/.git` exists, returns
`git -C <repo_root> rev-parse HEAD` (else `None`). Reuse the bounded git runner already in `image_attachments.py` (or a
tiny local subprocess with a short timeout); failures must degrade to `None`, never raise.

Persist the SHA so finalize can read it. Record it into the run's `agent_meta` dict as `sdd_base_sha` (analogous to how
`linked_repos` is recorded), and re-persist with `write_agent_meta(...)`.

**Capture point (ordering matters):** capture _after_ workspace preparation and _before_ the agent's first turn. In
`src/sase/axe/run_agent_runner.py`, the start sequence is: `enter_agent_workspace()` (sets `SASE_SDD_DIR`) →
`extract_directives_and_write_meta()` (writes `agent_meta.json`, but does not have `workspace_num`) →
`apply_retry_chain_to_meta()` → `prepare_workspace_if_needed()` (materializes/refreshes the SDD clone) → … →
`AgentExecContext(...)` (receives `agent_meta`) → `run_execution_loop()`.

Capture **after** `prepare_workspace_if_needed()` (so any start-of-run refresh / "Initialize SDD store" commit is
already in the base and excluded from `base..HEAD`) and **before** `AgentExecContext` construction, then
`write_agent_meta(artifacts_dir, agent_meta)` so both the in-memory `ctx.agent_meta` and the on-disk marker carry
`sdd_base_sha`. (`ctx.agent_meta` is the same dict passed to `AgentExecContext`, so finalize can read it directly; also
readable from disk via the existing `_read_transcript_agent_meta()` helper as a fallback.)

Guard the capture so it is skipped cleanly for in-tree storage, missing clone, or any error — the common bare-git /
in-tree path must be a no-op.

### B. Extend attachment discovery to scan extra repos with a commit range

In `src/sase/axe/image_attachments.py`:

- Add a small value type describing an extra repo scan, e.g. `ExtraRepoScan(repo_dir: str, base_sha: str | None)`.
- Extend the three public collectors (`collect_agent_image_paths`, `collect_agent_video_paths`,
  `collect_agent_markdown_paths`) and `_collect_agent_attachment_paths` with an
  `extra_repo_scans: Iterable[ExtraRepoScan] = ()` parameter (default empty → **byte-for-byte identical behavior to
  today** for the primary path).
- For each extra scan, gather candidates from:
  - `git diff --name-status -z <base_sha>..HEAD` when `base_sha` is set and differs from HEAD (files committed during
    the run — the primary case here), **plus**
  - `git diff --name-status -z HEAD` (tracked-but-uncommitted), **plus**
  - `git ls-files --others --exclude-standard -z` (untracked; covers the auto-commit-disabled case).
  - Do **not** apply the main repo's `include_head_commit`/`diff_path` logic to extra repos — the `base..HEAD` range is
    the precise, multi-commit-safe equivalent.
- Resolve each extra-repo candidate **relative to its own `repo_dir`** (the existing `_resolve_existing_attachment_path`
  already takes the workspace/root arg — reuse it per scan), keep the same extension filtering and the `artifacts_dir`
  exclusion, and dedupe via the existing `append_unique_paths` (absolute-path keyed), so any overlap with the primary
  scan is harmless.
- Preserve the exact existing ordering for the primary root; append extra-repo results after it.

### C. Wire it into finalize

In `src/sase/axe/run_agent_exec_finalize.py` → `_collect_default_artifacts()`:

- Build `extra_repo_scans` via a new helper `_sdd_repo_scans(ctx)`: resolve
  `resolve_sdd_store(ctx.workspace_dir, ctx.workspace_num)`; if storage is `in_tree` → return `[]`; else if
  `repo_root/.git` exists → return `[ExtraRepoScan(str(repo_root), ctx.agent_meta.get("sdd_base_sha"))]`; guard all
  errors → `[]`.
- Pass `extra_repo_scans=...` to the three `collect_agent_*` calls. Everything downstream is unchanged: markdown →
  `_render_markdown_pdfs` (absolute source paths already returned by the collectors, so PDF rendering works for SDD-repo
  files), `markdown_pdf_paths`/`image_paths`/ `video_paths` flow into `persist_default_agent_artifacts`,
  `build_done_marker`, and the completion notification exactly as before.

No change needed in `send_completion_notification`, the notification senders, the mobile-bridge attachment manifests, or
the `sase-telegram` plugin — they consume `image_paths`/`Notification.files`, which are now correctly populated.

## Files to change

- `src/sase/axe/image_attachments.py` — add `ExtraRepoScan`; add `extra_repo_scans` param to the three collectors and
  `_collect_agent_attachment_paths`; add per-repo `base..HEAD` gathering.
- `src/sase/axe/run_agent_exec_finalize.py` — add `_sdd_repo_scans(ctx)`; pass `extra_repo_scans` into the three
  collectors.
- `src/sase/axe/run_agent_runner.py` (+ possibly a helper in `run_agent_runner_setup.py`) — capture `sdd_base_sha` after
  workspace prep and persist into `agent_meta`.
- (If a shared helper is cleaner) a small function to resolve the SDD companion repo root + capture its HEAD SHA, placed
  near the SDD env/store helpers (e.g. reuse `resolve_sdd_store`).

## Testing

- `tests/test_agent_attachment_discovery.py`:
  - New: `extra_repo_scans` with a second git repo that has a **committed** markdown + image in `base..HEAD` → both
    discovered; a committed file _before_ base → not discovered; an untracked file → discovered. Assert primary-only
    behavior is unchanged when `extra_repo_scans=()`.
- `tests/test_axe_run_agent_exec_finalize_attachments.py`:
  - New integration test simulating `separate_repo`: create a `<workspace>/.sase/sdd` git repo, capture its HEAD as
    base, write + commit `research/example.md` and an image on top, set `ctx.agent_meta["sdd_base_sha"]` to the base,
    and patch/return a `resolve_sdd_store` that reports `separate_repo` with `repo_root` at that dir. Assert the
    research PDF and image appear in `result.markdown_pdf_paths` / `result.image_paths`, in `done.json`, and (via
    `send_completion_notification`) in `notify.call_args.kwargs["extra_files"]`.
  - Keep the existing in-tree test green (in-tree → `_sdd_repo_scans` returns `[]`).
- Start-side: a focused test that `sdd_base_sha` is captured/persisted for a separate-repo/local store and omitted (or
  `None`) for in-tree / missing clone.

## Rollout / verification

- Run `just install` (workspace may be stale), then `just check` (ruff + mypy + tests).
- Manual smoke: an agent that writes `sdd/research/*.md` (+ an image) under separate-repo storage should show the
  research file as a `pdf` artifact on its completion entry, and the image should be attached to the Telegram completion
  message.

## Risks & mitigations

- **Wrong base (refresh moves HEAD):** capture _after_ `prepare_workspace_if_needed` so start-of-run refresh/init
  commits are in the base and excluded. If `rev-parse HEAD` fails (no commits), base is `None` and we fall back to the
  working-tree scan only (rare; a materialized store always has the "Initialize SDD store" commit).
- **Surfacing unrelated SDD content:** `base..HEAD` bounds discovery to this run's commits; plans are surfaced
  elsewhere; only image/markdown/video extensions are collected, so bead/db files are ignored.
- **Performance:** a few extra short, timeout-bounded git calls in one companion repo at finalize only; negligible and
  off the hot TUI path.
- **`local` storage:** handled uniformly (repo_root under the primary workspace); `in_tree` is a no-op.
