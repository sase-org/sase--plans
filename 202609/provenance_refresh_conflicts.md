---
tier: epic
title: Eliminate recurring plans-sidecar provenance merge conflicts
goal: 'Concurrent plan provenance refreshes from different repos and machines converge
  additively instead of fighting, the sync rebase auto-resolves the recurring AGENTS/COMMITS
  conflict class, and finished plans stop being stamped onto unrelated commits — so
  plans-sidecar merge conflicts no longer require manual resolution.

  '
phases:
- id: additive-provenance-merge
  title: Additive provenance section merging
  depends_on: []
  size: medium
  description: 'additive-provenance-merge: make refresh_association_sections and both
    refresh callers union locally derived AGENTS/COMMITS entries with the plan file''s
    existing entries (derived side wins per-key metadata, no entry is ever dropped
    for being outside the local view), with deterministic ordering pinned by idempotence
    and merge-stability tests.

    '
- id: plan-header-conflict-resolver
  title: Plan-header semantic conflict resolver
  depends_on:
  - additive-provenance-merge
  size: medium
  description: 'plan-header-conflict-resolver: add a resolver to the SDD semantic
    conflict chain that claims plans-store month-dir markdown conflicts and, when
    stages differ only in generated AGENTS/COMMITS sections, resolves by unioning
    both sides with the phase-1 merge helper, failing closed on any authored difference.

    '
- id: stale-plan-attribution
  title: Stop stale SASE_PLAN attribution
  depends_on: []
  size: medium
  description: 'stale-plan-attribution: reproduce how launches inherit a stale SASE_PLAN
    env value after a plan finishes, then gate the commit-workflow stamp and/or the
    launch-side propagation so completed plans stop collecting unrelated commits,
    with regression tests keeping legitimate plan-execution flows stamped.'
proposed_by: bbugyi200.athena.0kt
create_time: 2026-09-14 14:03:45
status: wip
bead_id: sase-112
---

- **PROMPT:** [prompts/202609/provenance_refresh_conflicts.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/provenance_refresh_conflicts.md)
- **BEAD:** [sase-112](https://github.com/sase-org/sase--beads/blob/main/pages/sase-112/README.md)

# Eliminate Recurring Plans-Sidecar Merge Conflicts From Plan Provenance Refresh

## Problem

Plan files in the plans sidecar repeatedly develop merge conflicts inside their
generated provenance header (the `**AGENTS:**` and `**COMMITS:**` sections), and the
user has to resolve them by hand. `202609/telegram_receiver_launch_fix.md` is the latest
example: its history contains five `Refresh plan provenance for ...` commits in under an
hour, and adjacent refresh commits (`ee4206e6` at 13:19 and `54e10eba` at 13:21 on
2026-09-14) **completely swapped** the sections — one lists agents
`bbugyi200.athena.0kp/0kr` plus `sase-telegram` commits, the next deletes all of those
and lists `bbugyi200.athena.research.1x/1y.*` agents plus `sase--research` commits.

## Root Cause (three compounding defects)

1. **Each provenance refresh rebuilds from a single repo's local view and replaces the
   sections wholesale.** After every commit carrying a `SASE_PLAN` footer,
   `refresh_committed_plan_header` (`src/sase/sdd/plan_header_refresh.py`) is invoked
   from `src/sase/workflows/commit/workflow_publication.py` with `primary_root=cp.cwd` —
   the repo the commit just landed in. The association index
   (`src/sase/sdd/associations/_build.py`) walks only that repo's `git log` plus
   machine-local agent artifact records, and `refresh_association_sections`
   (`src/sase/sdd/plan_header_writes.py`) then **replaces** the `AGENTS`/`COMMITS`
   sections with exactly that derived set. A plan whose provenance spans multiple repos
   (here: `sase-telegram` commits and `sase--research` commits) therefore gets opposite
   full-section rewrites from refreshes triggered in different repos. The store write
   lock (`src/sase/sdd/_git_contention.py`) is a per-clone flock, so it cannot serialize
   writers in different workspaces or on different machines. Result: guaranteed,
   repeated conflicting commits — and silent provenance deletion even when a merge
   happens to succeed textually.

2. **No semantic conflict resolver claims plan files.** The SDD sync rebase resolves
   conflicts through the chain in `src/sase/sdd/_semantic_conflict_resolver.py`, which
   claims bead files and artifact-link files only. A conflicted plan `.md` is
   "unclaimed", so `_resolve_conflicts` in `src/sase/sdd/_repository_integration.py`
   aborts the rebase back to the starting state. The push worker never succeeds, local
   and remote histories keep diverging, and the user eventually hand-merges purely
   mechanical, re-derivable content.

3. **Stale `SASE_PLAN` attribution pollutes provenance at the source.** Commit
   `b55a005123d9bd02d918e47a7746f85399e08c87` in the `sase--research` sidecar
   ("docs(research): critique AXE/ACE/lumberjack/chop rename proposal") carries the
   footer `SASE_PLAN=[202609/telegram_receiver_launch_fix.md]` even though it has
   nothing to do with the telegram receiver plan. `handle_sase_plan`
   (`src/sase/workflows/commit/plan_hooks.py`, reads `os.environ["SASE_PLAN"]`) stamps
   every commit-workflow commit with whatever plan the process environment names.
   `SASE_PLAN` is set at plan acceptance (`src/sase/axe/run_agent_exec_plan_accept.py`,
   lines ~520-526), popped only at agent-exec finalize
   (`src/sase/axe/run_agent_exec_finalize.py:121`), and propagated into attached family
   launches (`src/sase/agent/_family_attach_launch.py:46`). Some launch chain (the
   research swarm) inherited a stale value after the telegram plan had already been
   archived, so unrelated research commits were attributed to it — which is exactly why
   two unrelated writer populations were fighting over the same plan file at all.

## Fix Strategy

Make the provenance projection **additive and commutative** (a plan file accumulates
provenance; a refresh merges in what its local view derives and never deletes entries it
cannot see), teach the sync rebase to **auto-resolve** plan-header conflicts by unioning
both sides, and **stop the stale-attribution leak** so unrelated runs no longer stamp a
finished plan. Entry removal becomes a deliberate human editorial act (editing the plan
file directly), not a side effect of whichever writer refreshed last.

All work stays in the existing Python modules in this repo; it modifies established
`src/sase/sdd/` and `src/sase/workflows/` behavior in place and crosses no Rust-core
boundary. No SASE memory files are created or edited by this plan.

## Phases

### Phase 1: `additive-provenance-merge` (size: medium)

Depends on: none.

Change `refresh_association_sections` (`src/sase/sdd/plan_header_writes.py`) and its two
callers — post-commit `refresh_committed_plan_header`
(`src/sase/sdd/plan_header_refresh.py`) and tree-wide `refresh_plan_links`
(`src/sase/sdd/plan_links_refresh.py`) — so refreshes merge instead of replace:

- Parse the document's existing `AGENTS`/`COMMITS` entries via `parse_plan_header_block`
  (`src/sase/sdd/plan_header_block.py`) and union them with the derived
  `PlanAssociations` entries.
- Dedup keys: agent entries by label; commit entries by full SHA parsed from the entry
  target URL, falling back to the short-SHA label. When a key exists on both sides, the
  freshly derived entry wins (it carries refreshed target URLs and subjects — e.g. the
  agent-link migration from `families/<name>.md` to `agents/<name>/README.md`).
- Never drop an existing entry merely because the local derivation lacks its key. An
  empty derived set with a populated existing section is a no-op, not a deletion.
- Rendering must be deterministic given (existing entries, derived entries): agents
  sorted by label; commits keep existing file order for retained entries with new
  derived entries appended in derived (commit-time) order. The implementer may refine
  the ordering rule, but tests must pin it and prove idempotence (re-running a refresh
  changes nothing) and merge stability (refreshing with view A then view B yields the
  same entry set as B then A).
- Verify `refresh_bead_plan_section` and `refresh_prompt_plan_section` derive from the
  shared plans/bead store rather than a per-repo view; leave them as-is if consistent
  across writers, otherwise apply the same additive treatment.
- Extend `tests/sdd/test_plan_links_refresh.py` and
  `tests/agents_sync/test_committed_plan_header.py` with the union/no-delete/
  determinism cases, including a two-writer flip-flop regression modeled on the telegram
  example above.

### Phase 2: `plan-header-conflict-resolver` (size: medium)

Depends on: `additive-provenance-merge` (reuses its entry-parsing/union helper so both
code paths merge identically).

Add a plan-header semantic conflict resolver so the SDD sync rebase resolves the
recurring conflict class instead of aborting:

- New module modeled on `src/sase/sdd/_artifact_link_markdown_conflict_resolver.py`,
  using the stage helpers in `src/sase/bead/conflict_resolver_git.py`
  (`unmerged_stages`, `read_git_show`, `upstream_and_local_stages`, `git_add`).
- Claim predicate: the conflicted path is a `.md` file in a plans-store month directory
  (`is_month_dir_name` in `src/sase/sdd/_paths.py`) and every present stage parses with
  `parse_plan_header_block` without an `INVALID` disposition.
- Resolution: when the stages differ **only** inside the generated `AGENTS`/`COMMITS`
  sections (frontmatter, other header sections such as `PARENT`, and the authored plan
  body are byte-identical after normalizing the generated sections away), write the
  union of both sides' entries using the Phase 1 merge helper, then stage the file. Any
  other difference fails closed — return "unclaimed" so the existing abort path and
  human review still own genuine authored conflicts.
- Register the resolver in the chain in `src/sase/sdd/_semantic_conflict_resolver.py`;
  the retry loop in `src/sase/sdd/_repository_integration.py` needs no changes.
- Tests modeled on `tests/sdd/test_artifact_link_conflict_resolver.py`: a real git repo
  fixture reproducing the divergent refresh-commit rebase, asserting the union result,
  plus fail-closed cases (authored-body difference, frontmatter difference, malformed
  header).

### Phase 3: `stale-plan-attribution` (size: medium)

Depends on: none (independent of Phases 1-2; can run in parallel).

Stop finished plans from being stamped onto unrelated commits:

- Reproduce the leak: trace how the research swarm launches inherited
  `SASE_PLAN=202609/telegram_receiver_launch_fix.md` after that plan was archived. Start
  from `src/sase/agent/_family_attach_launch.py:46` (explicit propagation into attached
  family launches), the set/pop lifecycle in
  `src/sase/axe/run_agent_exec_plan_accept.py` and
  `src/sase/axe/run_agent_exec_finalize.py:121` (pop only runs on that finalize path),
  and plain process-environment inheritance for launches that never finalize through
  agent-exec.
- Gate the stamp: `handle_sase_plan` (`src/sase/workflows/commit/plan_hooks.py`) and/or
  the launch-side propagation must stop attributing commits to a plan whose execution is
  over. Candidate gates, to be settled by the reproduction evidence: skip stamping when
  the referenced plan's frontmatter `status` is terminal (done/archived) unless the
  commit is part of the plan's own completion machinery; and/or stop exporting
  `SASE_PLAN` into launches that are not executing that plan (research swarms, unrelated
  helpers).
- Whichever gate lands, legitimate flows must keep working: the plan-executing agent's
  own commits, the archive/completion commits, and provenance refresh commits for an
  active plan must still carry or resolve the plan reference.
- Add regression tests for the chosen gate (commit workflow stamping and launch env
  propagation), extending the existing commit-workflow tests
  (`tests/test_commit_workflow_publication.py` or the closest plan-hooks test module).

## Verification

Each phase runs the repo's standard verification (`just check` at minimum) plus its new
tests. End-to-end confidence check after Phases 1-2: in a scratch clone pair, produce
the two-writer divergence (refresh from two different local views), sync, and confirm
the rebase auto-resolves to the union with no human conflict and no lost entries.

## Out Of Scope

- Cleaning up already-polluted provenance in existing plan files (e.g. the research
  entries misattributed to `202609/telegram_receiver_launch_fix.md`): entry removal is a
  human editorial edit of the plan file once Phase 1 makes it stick.
- Any SASE memory note or decision-record changes; if the additive-provenance decision
  deserves a decision record, file it as a follow-up memory task bead instead.
- Moving any of this logic into the Rust core.
