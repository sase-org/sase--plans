---
create_time: 2026-07-12 16:11:56
status: wip
prompt: 202607/prompts/sdd_prompt_attachment_dedup.md
tier: tale
---
# Fix Duplicate Markdown Attachments on Telegram Agent Completion Messages

## Problem

Agent completion messages in Telegram attach what looks like the same markdown file twice. Example from today's run
`ace(run)-260712_152745` (notification `f3e6d5e3`):

- `sase__repos__plans__202607__amd_agents_template.md.pdf` (76.4 KB — the plan)
- `sase__repos__plans__202607__prompts__amd_agents_template.md.pdf` (13.1 KB — the SDD prompt snapshot, same basename)

Scanning `~/.sase/notifications/notifications.jsonl` shows this is systemic: **108 notifications since 2026-07-06 attach
two files with the same basename, and 99 of them are exactly this plan + `prompts/` companion pair.** The remaining
cases attach SDD store scaffolding (`sase__sdd__README.md.pdf`, `sase__sdd__plans__README.md.pdf`) and even unrelated
plan documents from other months (e.g. `sase__sdd__plans__202605__...` attached to a 202607 run), which arrive via
store-init/migration commits.

The prompt snapshot is pure noise for the user: the completion message body already shows the prompt in its "📝 Prompt:"
section, and the plan itself was already delivered with the earlier Plan Review message.

## Root Cause

The bug is in completion-attachment discovery in the sase repo (not in the sase-telegram plugin — the plugin faithfully
sends whatever is in `notification.files`).

The chain:

1. `send_completion_notification()` (`src/sase/axe/run_agent_runner_finalize.py`) builds the notification `files` list
   from `markdown_pdf_paths` (among others).
2. `markdown_pdf_paths` come from `_collect_default_artifacts()` (`src/sase/axe/run_agent_exec_finalize.py`), which
   calls `collect_agent_markdown_paths()` with `extra_repo_scans=_sdd_repo_scans(ctx)` — a scan of the
   separate/companion SDD store repo.
3. `_attributed_commit_paths()` (`src/sase/axe/image_attachments.py`) attributes **every file touched by any SDD-store
   commit tagged `SASE_AGENT=<agent>`** to the agent. The SDD plan flow commits both the plan document
   (`plans/<yyyymm>/<name>.md`) _and_ its prompt snapshot (`plans/<yyyymm>/prompts/<name>.md`) in agent-tagged commits
   (e.g. "Add SDD files for ...", "Complete SDD plan for ..." with `SASE_TYPE=sdd`, `SASE_AGENT=<n>` trailers).
4. Both markdown files therefore pass the `.md` filter, get rendered to PDFs by `render_markdown_pdf_attachments()`, and
   are attached. Existing dedup (`append_unique_paths`) is by absolute path, so two distinct files with the same
   basename sail through.
5. The same mechanism pulls in store scaffolding: companion-repo init/adoption/migration commits tagged with the agent
   touch `README.md` files and old plan documents, all of which become attachments.

Verified empirically: the affected run's SDD store commits (`git log` in the plans companion repo) show the plan and
`202607/prompts/amd_agents_template.md` in agent-tagged commits, and the prompt snapshot's content is exactly the prompt
text shown in the message body.

## Fix

Filter SDD-internal companion files out of attachment discovery for SDD store scans. The knowledge of _what_ is
SDD-internal belongs in the SDD layer; the scan mechanics stay in `image_attachments.py`.

### 1. SDD-layer classification helper (`src/sase/sdd/_paths.py`)

Add a helper (exported via the usual SDD facade) that decides whether a repo-relative path inside an SDD store is a
user-facing document worth attaching, e.g. `is_sdd_internal_path(rel_path: str) -> bool`. It should classify as
internal:

- Prompt snapshots: any path under a `*/prompts/` month companion dir (`plans/<yyyymm>/prompts/`, `<yyyymm>/prompts/`
  for stores whose root is the plans root) and legacy `prompts/` / `specs/` kind roots — reuse the layout knowledge
  already encoded in `sdd_prompt_roots()` / `_SDD_PROMPT_KINDS` rather than duplicating globs.
- Bead files: anything under `beads/`.
- Scaffolding: `README.md` at the store root and at kind roots (`plans/README.md`, `research/README.md`,
  `sdd/README.md`, ...).

Plan documents (`plans/<yyyymm>/<name>.md`) and research documents remain attachable.

### 2. Apply the filter to SDD repo scans

Extend `ExtraRepoScan` (`src/sase/axe/image_attachments.py`) with an optional path-exclusion predicate (e.g.
`exclude: Callable[[str], bool] | None = None`, applied to the repo-relative candidate path in
`_extra_repo_changed_paths()` before resolution). `_sdd_repo_scans()` (`src/sase/axe/run_agent_exec_finalize.py`) passes
the SDD classifier. Non-SDD scans are unaffected, and the primary-workspace scan is untouched.

Note: `ExtraRepoScan` is a frozen dataclass used only within this repo's Python discovery pipeline; no Rust core
(`sase-core`) or wire changes are needed since attachment discovery lives entirely in this repo today. No sase-telegram
change is needed either — once `notification.files` is clean, the plugin output is clean.

### 3. Scope decision: denylist, not allowlist

Prefer excluding known-internal paths over allowlisting plan/research docs, so genuinely agent-authored markdown in an
SDD store keeps attaching. The unrelated-old-plan noise (case: migration commit tagged with the agent) is largely
init/adoption commits touching `README.md`s and moved files; renames are already reported as the new path, so the
denylist plus the existing "attributed commits only" rule covers the observed cases. If we later see more migration
noise, a follow-up could skip commits whose subject matches store-init/migration patterns — out of scope here.

## Tests

- `tests/test_agent_attachment_discovery.py`: new cases for `ExtraRepoScan` exclusion — agent-tagged SDD commit touching
  `plans/202607/foo.md` + `plans/202607/prompts/foo.md` yields only the plan; legacy `prompts/202607/foo.md` and
  `specs/` layouts excluded; `beads/*.md` excluded; root and kind-root `README.md` excluded; a non-SDD extra scan
  without the predicate still returns everything.
- SDD helper unit tests (wherever SDD path helpers are tested) for `is_sdd_internal_path()` across canonical and legacy
  layouts.
- `tests/test_axe_run_agent_exec_finalize_sdd_attachments.py`: end-to-end style regression — completion attachment
  collection for a run with an SDD plan + prompt snapshot commit produces exactly one `foo.md` source (and thus one
  rendered PDF).

## Verification

- `just check` (lint + full test run).
- Manual spot check: rerun a plan-producing agent and confirm the completion notification in
  `~/.sase/notifications/notifications.jsonl` lists a single `markdown_pdfs/...<name>.md.pdf` for the plan, with no
  `...prompts__<name>.md.pdf` sibling, and the Telegram message shows the plan attached once.

## Out of Scope (noted for follow-up)

- The plan is also attached to the earlier Plan Review message, so completion still re-delivers a document the user has
  seen. That duplication is cross-message and arguably intentional (the plan may have been revised between review and
  completion); leaving as-is.
- Basename-collision handling in the Telegram plugin (e.g. disambiguating display names) — unnecessary once the source
  list is correct, and legitimate collisions (two real `README.md`s) should keep their real names.
