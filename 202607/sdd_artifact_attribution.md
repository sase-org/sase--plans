---
create_time: 2026-07-10 19:26:14
status: done
prompt: 202607/prompts/sdd_artifact_attribution.md
tier: tale
---
# Fix SDD Companion-Repo Artifact Attribution on Agent Completion

## Problem

When an agent completes, its Telegram completion message can include images (and Markdown PDFs / videos) that live in
the SDD repo but were **not** created or modified by that agent. The user's suspicion is confirmed: SDD artifacts
produced by _other_ agents (or left over in the store) are being attributed to the completing agent.

## Diagnosis (confirmed)

Commit `28168ad05` ("fix: include companion SDD artifacts in finalization", 2026-07-09 — the day the spurious Telegram
images started) added companion-SDD scanning to agent finalization:

- `src/sase/axe/run_agent_runner_setup.py` — `capture_sdd_base_sha()` records the companion SDD repo HEAD at launch into
  `agent_meta.json` (`sdd_base_sha`).
- `src/sase/axe/run_agent_exec_finalize.py` — `_sdd_repo_scans()` builds an `ExtraRepoScan(repo_root, base_sha)` for any
  out-of-tree SDD store, and `_collect_default_artifacts()` feeds it to `collect_agent_markdown_paths` /
  `collect_agent_image_paths` / `collect_agent_video_paths`.
- `src/sase/axe/image_attachments.py` — `_extra_repo_changed_paths()` attributes to the completing agent:
  1. **every** commit in `sdd_base_sha..HEAD`,
  2. **all** dirty tracked files (`git diff HEAD`), and
  3. **all** untracked files in the SDD repo — with no check of who produced them.

The collected paths land in the done marker (`build_done_marker(... image_paths=...)`) and in
`send_completion_notification()` → `notify_workflow_complete(extra_files=...)` → notification `files` → mobile bridge
`host_files` → Telegram image attachments.

### How foreign files enter the scan window

1. **`git pull --rebase` during SDD pushes** (`src/sase/bead/sync.py` — `push_bead_work_launch`, and the async
   `git push || (git pull --rebase && git push)` variant): any SDD auto-commit + push during the run pulls _other
   agents'_ SDD commits into the local clone. Those commits now sit inside `sdd_base_sha..HEAD` at finalize time.
2. **Shared `local` SDD stores** (`_sdd_dir_for_storage`: `local` storage resolves to the _primary_ workspace's
   `.sase/sdd`, shared by every numbered workspace of the project): concurrent agents commit directly into the same repo
   (inside the base..HEAD window), and their in-progress dirty/untracked files are visible to whichever agent finalizes
   first.
3. **Stale working-tree leftovers**: dirty/untracked files remaining in a reused workspace's SDD clone (e.g. from a
   crashed earlier run) are swept in as the next agent's artifacts.

## Fix Design

SDD auto-commits already carry agent provenance: `commit_sdd_files()` routes every message through
`apply_auto_commit_tags_with_runtime()` (`src/sase/workflows/commit/runtime_tags.py`), which appends trailing
`AGENT=<name>` and `MACHINE=<host>` tags, and `parse_trailing_commit_tags()` already parses them. Use those trailers to
attribute commits, and stop attributing working-tree state that cannot be attributed.

### 1. Attribute base..HEAD commits by AGENT/MACHINE trailers (`src/sase/axe/image_attachments.py`)

- Extend `ExtraRepoScan` with `agent_name: str | None = None` and `include_working_tree: bool = False`.
- Rework `_extra_repo_changed_paths()`:
  - Commit-range portion: instead of one `git diff base..HEAD`, enumerate commits in `base..HEAD` with their full
    messages (single `git log` invocation with NUL/unit separators), parse each message's trailing tag block with
    `parse_trailing_commit_tags` (lazy import, matching the module's existing style), and keep only commits whose
    `AGENT` tag belongs to the completing agent's hood/family: tag equals the agent's base name, or starts with
    `<base>.` (hood) or `<base>--` (family). This deliberately groups plan-chain/step commits (`foo--plan-0`, `foo.2`)
    with their root agent `foo`, matching the glossary's hood/family semantics.
  - When a commit carries a `MACHINE` tag that differs from the current hostname, exclude it (same agent name on another
    machine is a different run).
  - Commits with no `AGENT` tag are excluded — conservative: sase-produced SDD commits always carry the tag when an
    agent identity exists; untagged commits are human/manual or foreign.
  - If `scan.agent_name` is `None`, skip the commit-range portion entirely (nothing can be attributed).
  - Collect changed paths only from the matching commits (`git diff-tree --no-commit-id --name-status -z -r <sha>`),
    reusing `_paths_from_name_status`.
  - Dirty tracked + untracked working-tree scanning becomes conditional on `include_working_tree`.
- The main-workspace scan (`_local_changed_paths` / `_untracked_paths` on `workspace_dir`) is unchanged — the workspace
  clone is owned by the run.

### 2. Pass attribution context from finalize (`src/sase/axe/run_agent_exec_finalize.py`)

In `_sdd_repo_scans()`:

- Set `agent_name` from `ctx.agent_name`, falling back to the transcript `agent_meta.json` `name` (helper already
  exists: `_read_transcript_agent_meta`).
- Set `include_working_tree=True` only for per-workspace clones (`store.storage == SDD_STORAGE_SEPARATE_REPO`), where
  the only writers during the run are this workspace's agent processes. For shared `local` stores the working tree is
  cross-agent and must not be attributed.
- Accepted residual risk (documented in code): a `separate_repo` clone can, rarely, hold uncommitted leftovers from a
  previous crashed run in the same workspace. The commit finalizer's separate-store sweep
  (`_auto_commit_separate_sdd_store_if_possible`) normally commits the agent's own SDD writes (with its `AGENT=` tag)
  before completion, so this window is small; full attribution of working-tree files is not possible in git.

### 3. Tests

Extend `tests/test_agent_attachment_discovery.py` (and the finalize-time coverage added by `28168ad05`):

- A base..HEAD commit tagged `AGENT=<other-agent>` is **excluded**; commits tagged with the agent's own name, a hood
  member (`<base>.foo`), and a family member (`<base>--plan-0`) are **included**.
- An untagged commit in base..HEAD is excluded.
- A commit whose `MACHINE` tag differs from the current hostname is excluded.
- Dirty tracked and untracked files are excluded when `include_working_tree=False` and included when `True`.
- `_sdd_repo_scans()` passes the agent name and sets `include_working_tree` per storage kind (`separate_repo` → True,
  `local` git-backed store → False).
- Keep/adjust the existing `test_collect_agent_paths_from_extra_repo_base_range_and_untracked` to the new attribution
  semantics (its fixture commits must now carry the agent's tags).

## Non-goals / Notes

- No change to the Telegram plugin or the mobile notification bridge — they faithfully render whatever the done marker
  and completion notification claim; the bug is attribution at collection time.
- No Rust-core change: this is Python agent-runtime finalize behavior in `axe`, not shared domain logic.
- The markdown-PDF pipeline is fixed by the same change since all three collectors share `_extra_repo_changed_paths()`.
- Reverting `28168ad05` is not proposed: attaching the agent's _own_ companion-SDD artifacts is desired behavior.

## Verification

- `just install`, then `just check` (lint + mypy + tests).
- Manual sanity check: with two overlapping agent runs writing images into a shared SDD store, confirm each completion
  notification only lists images from commits tagged with that agent's own name.
