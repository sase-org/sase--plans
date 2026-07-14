---
create_time: 2026-07-14 07:51:36
status: done
prompt: 202607/prompts/artifact_attachment_repo_attribution.md
tier: tale
---
# Fix artifact attachment misattribution for commits made outside the primary workspace repo

## Problem

The ACE "Agent Artifacts" modal for agent `research.b.cld` (run `20260714071915`) shows:

- A **bogus image artifact**: `tests/ace/tui/visual/snapshots/png/notification_question_summary_120x40.png` — a
  pre-existing visual-snapshot golden from the sase repo that the agent never touched.
- The **research report markdown is missing**: the agent's actual deliverable
  (`202607/sase_beads_improvement_opportunities.md`, committed to the `sase--research` linked repo) never became an
  artifact (normally it would be rendered to PDF and attached).

Both symptoms share one root cause: default-artifact discovery assumes every commit an agent makes happens in the
**primary workspace repo**, but the commit finalizer can (and here did) commit to a **different repo** — the
`sase--research` linked-repo checkout materialized under the workspace at `sase/repos/research`.

## Root cause (verified against the real run)

Evidence from `~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202607/14/20260714071915/`:

- `commit_result.json` / `commit_results.json`: the finalizer created commit `e145743` ("docs(research): add sase beads
  improvement opportunities analysis") with `cwd = <workspace>/sase/repos/research` (the linked `sase--research`
  checkout) and `diff_path = .../commit_diffs/001.diff`.
- `done.json`: `image_paths` contains exactly the golden PNG under the workspace checkout; `markdown_pdf_paths` is
  empty.
- The workspace repo HEAD at finalize time was `b5198f7e1` ("feat(notifications): summarize agent questions in
  notification modal") — the **master tip commit, not an agent commit** — and that commit is what added the golden PNG
  (`git log` confirms it introduced `tests/ace/tui/visual/snapshots/png/notification_question_summary_120x40.png`).

Discovery happens in `_collect_default_artifacts()` in `src/sase/axe/run_agent_exec_finalize.py`, which calls the
collectors in `src/sase/axe/image_attachments.py`. Three defects:

1. **False positive (the golden PNG).** `include_head_commit` is set to `True` whenever the step output has
   `meta_new_commit` or `meta_pr_url` (`run_agent_exec_finalize.py:239`). The flag then makes
   `_head_commit_paths(workspace_dir)` diff `HEAD~1..HEAD` of the **primary workspace** — even though the only commit
   the agent made was in the research repo. Since the agent never committed to the workspace, `HEAD~1..HEAD` is someone
   else's commit (`b5198f7e1`), and its added PNG got attributed to the agent, persisted into the global artifact store
   (`~/.sase/artifacts/agents/gh_sase-org__sase/20260714071915/`), shown in the modal, and attached to completion
   notifications.

2. **False negative (the missing research markdown).** The commit's diff file _does_ list the research markdown, but the
   paths inside a diff are relative to the repo the commit was made in
   (`202607/sase_beads_improvement_opportunities.md`). `_resolve_existing_attachment_path()`
   (`image_attachments.py:395`) resolves relative candidates against `workspace_dir` only; the path does not exist under
   the workspace root, so `resolve(strict=True)` fails and the candidate is silently dropped. No markdown → no PDF
   render → no artifact.

3. **Latent gap (multi-commit runs).** Only the single top-level `diff_path` (effectively the last commit's diff) is
   scanned. `commit_results.json` records _every_ commit with its own `diff_path` **and** `cwd`, but those records are
   ignored by attachment discovery. (`_sdd_repo_scans()` exists but only covers the SDD store repo — never linked repos
   or any other commit target.)

The per-commit `cwd` + `diff_path` recorded in `commit_results.json` (surfaced as `meta_commits` on the step output by
`read_commit_results_metadata()` in `src/sase/axe/run_agent_helpers_state.py`) is exactly the attribution signal the
collectors need — it is currently collected but unused.

## Fix design

All changes are in this repo's Python axe runner (no `sase-core` change: no wire/API or `done.json` schema change —
discovery already lives here in `src/sase/axe/image_attachments.py`).

### 1. Per-diff base directories in `image_attachments.py`

- Add a small frozen dataclass, e.g. `DiffScan(diff_path: str, base_dir: str | None = None)`.
- Extend `collect_agent_image_paths` / `collect_agent_video_paths` / `collect_agent_markdown_paths` /
  `_collect_agent_attachment_paths` with a `diff_scans: Iterable[DiffScan] = ()` parameter. Candidates extracted from
  each diff are resolved against that diff's `base_dir` (falling back to `workspace_dir` when `base_dir` is `None`).
- Keep the existing `diff_path` parameter working as today (equivalent to `DiffScan(diff_path, workspace_dir)`) so other
  tests/back-compat paths stay valid; there is only one production call site.

### 2. Build diff scans from commit records in `_collect_default_artifacts()`

- Read the per-commit records from `step_output["meta_commits"]` (each record already carries `diff_path` and `cwd`).
  For each record, emit `DiffScan(record["diff_path"], record.get("cwd"))`.
- Fall back to the current single `diff_path` (resolved against the workspace) when no commit records exist, so legacy
  runs and non-commit workflows behave exactly as before.
- Dedupe: if the top-level `diff_path` equals a record's `diff_path`, don't scan it twice (existing
  `append_unique_paths` dedupe already guards the output; just avoid redundant scanning).

### 3. Gate the head-commit scan on where the commits actually happened

- Replace the blanket `include_head_commit = bool(meta_new_commit or meta_pr_url)` with:
  - `meta_pr_url` set → keep `True` (a PR implies workspace commits), or
  - some commit record's `cwd` belongs to the **same git repo** as `workspace_dir` → `True`, else
  - commit records exist but none are in the workspace repo → `False`, else
  - legacy fallback (`meta_new_commit` set but no `cwd`/`meta_commits` info at all) → keep current behavior (`True`).
- IMPORTANT: "same git repo" must be decided by comparing `git rev-parse --show-toplevel` of the commit `cwd` vs the
  workspace dir — **not** by path-prefix checks — because linked-repo checkouts live _inside_ the workspace directory
  tree (`<workspace>/sase/repos/<name>`) yet are separate repos.
- Add the toplevel-comparison helper to `image_attachments.py` (it already owns the `_run_git` helper).

### 4. Tests

- `tests/test_agent_attachment_discovery.py` (unit level):
  - a diff scan whose `base_dir` is a second git repo outside/inside the workspace tree resolves markdown/image paths
    against that repo (the current failure: relative diff paths against a non-workspace repo).
  - multiple diff scans from multiple commits are all collected.
- `tests/test_axe_run_agent_exec_finalize_attachments.py` (finalize level) — reproduce this incident end-to-end with two
  temp git repos (primary workspace whose tip commit adds a PNG the agent didn't make; a "research" repo where the
  finalizer-style commit adds a markdown file, with `commit_results.json`/`commit_result.json` markers recording `cwd` +
  `diff_path`):
  - the workspace tip-commit PNG is **not** collected into `image_paths`;
  - the research markdown **is** collected and fed to the PDF render step (`markdown_pdf_paths` non-empty; assert via
    the existing render seam/mocking used by `test_axe_run_agent_exec_finalize_markdown_pdfs.py`);
  - legacy case: `meta_new_commit` with no `cwd` info still scans the workspace head commit (current behavior
    preserved).
- Existing head-commit tests (`test_collect_agent_*_paths_from_head_commit`) keep passing since the unit-level
  `include_head_commit` parameter semantics are unchanged — only the finalize-level gating changes.

### 5. Out of scope

- Retroactively repairing already-persisted artifacts of past runs (the bogus PNG row for run `20260714071915` can be
  dismissed manually in the TUI).
- Scanning linked-repo checkouts for _uncommitted_ working-tree files (`opened_linked_workspaces.json` records no base
  SHA; commits are the reliable attribution signal, and the commit finalizer sweeps agent writes into commits).
- Prompt-media discovery (`_discover_prompt_media_candidates`) — uninvolved in this bug.

## Verification

- `just install` then `just check` (lint + mypy + full test suite).
- Targeted:
  `pytest tests/test_agent_attachment_discovery.py tests/test_axe_run_agent_exec_finalize_attachments.py tests/test_axe_run_agent_exec_finalize_markdown_pdfs.py tests/test_axe_run_agent_exec_finalize_sdd_attachments.py`.
