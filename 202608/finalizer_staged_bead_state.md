---
tier: tale
title: Stop the commit finalizer from failing runs over staged bead state
goal:
  A run whose only outstanding change is machine-managed bead state finalizes cleanly
  instead of failing with "dirty work vanished without an attributable commit".
size: medium
proposed_by: bbugyi200.athena.04b
create_time: 2026-08-16 16:42:05
status: wip
---

# Plan: Stop the commit finalizer from failing runs over staged bead state

## Problem

Agent `047` (run `ace-run-260816_154827`) finished its work successfully — the
deliverable landed as `26c9f9a92 docs: define sase monitor glossary term` and the
outstanding bead note was already published upstream — yet the whole 25-minute run was
reported as **FAILED**:

```
WorkflowExecutionError: Step 'main' failed: Error: Commit finalizer failed: dirty
work vanished without an attributable commit. The finalizer will not treat
discarded, reset, or foreign-agent work as successful finalization.
- beads: <workspace>/sase/repos/beads (no newly reachable commit was attributed to this agent)
  - issues.jsonl
- beads: <workspace>/sase/repos/beads (no newly reachable commit was attributed to this agent)
  - issues.jsonl
```

Artifacts:
`~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202608/16/20260816154827/error_report.md`
and the run log `~/.sase/workflows/202608/gh_sase-org__sase_ace-run-260816_154827.txt`
(lines 156-173 narrate the finalizer pass that tripped the guard).

## Root cause

Bead writes stage bead state without committing it — `git_sync()` in
`src/sase/bead/_sync_git.py:76` is documented as "Stage bead state in git (does not
commit)". A **staged-but-uncommitted** `issues.jsonl` is then a terminal trap, because
the two layers that look at bead state disagree about what "clean" means:

1. Both bead-layer gates call `_list_bead_state_changes()`
   (`src/sase/bead/_sync_git.py:414`), which runs
   `git ls-files --modified --others --deleted --exclude-standard` — **worktree vs index
   only**. A staged-only change is invisible to it, so:
   - `bead_state_is_clean()` (`src/sase/bead/_sync_git.py:350`) returns `True`, and the
     finalizer's machine auto-commit `_auto_commit_bead_state()`
     (`src/sase/llm_provider/commit_finalizer.py:466`) bails out before committing
     anything;
   - `_commit_bead_state()` (`src/sase/bead/_sync_git.py:304-306`) early-returns
     `False`, so no bead-layer commit path ever flushes it either — even though the code
     immediately below that early return (`git diff --cached`, lines 321-346) handles
     staged content perfectly well.
2. The finalizer's dirty scan uses `git status --porcelain=v1`
   (`src/sase/llm_provider/commit_finalizer_git_status.py:59`), which **does** report
   staged-only changes. So the beads sidecar is handed to the LLM as work it must
   commit.

Verified directly (scratch repo, staged-only `beads/issues.jsonl`):

```
bead_state_is_clean(beads dir): True
git_changed_files(repo root)  : ['beads/issues.jsonl']
```

The rest follows mechanically. The agent obeyed the finalizer and ran `sase_git_commit`
in the beads sidecar; the wrapper's sync rebase conflicted on `issues.jsonl` because
another agent had already published the very same note upstream; the replayed commit was
redundant, so it was skipped. The sidecar ended clean and aligned with `origin/main`,
but HEAD had advanced only to foreign upstream commits.
`discarded_dirty_work_evidence()`
(`src/sase/llm_provider/commit_finalizer_git_progress.py:52`) saw a repo that left the
dirty set with no newly reachable commit carrying this agent's provenance, returned
`missing_agent_provenance`, and `_fail_on_discarded_dirty_work()`
(`src/sase/llm_provider/commit_finalizer.py:530`) failed the run.

Two contributing defects made this worse:

- **Bead commits are not attributable.** `_commit_bead_state()` stamps only
  `apply_auto_commit_type_tag` (`TYPE=beads`), unlike `commit_sdd_files()`
  (`src/sase/sdd/_commit_store.py:139-142`) which uses
  `apply_auto_commit_tags_with_runtime` and therefore carries `SASE_AGENT=`. Any
  bead-layer commit that cleans the store during a finalizer pass reads as foreign work
  to the guard, reproducing this failure by a second route.
- **Each sidecar is enumerated twice.** `collect_dirty_state()`
  (`src/sase/llm_provider/commit_finalizer_state.py:81-86`) concatenates main, sibling,
  external and sdd repos with no cross-source dedupe by path, and the `beads`/`plans`
  sidecars resolve as _both_ configured linked repos and SDD store targets. That is why
  the error lists `beads` twice; it also duplicates the sidecar instructions in every
  finalizer prompt. Reproduced with a stubbed git layer: `collect_dirty_state()` returns
  `kind=sibling name='beads'` and `kind=sdd name='beads'` for the same path.

## Goal

A run whose only outstanding change is machine-managed bead state must never fail with
"dirty work vanished without an attributable commit". Machine-managed store state is
committed by the machine before any LLM pass, machine commits are attributable, and
store state that a sync-rebase absorbed upstream is recognized as published rather than
discarded. The guard keeps its full strictness everywhere else.

## Changes

### 1. Bead-state change detection must include staged changes

`src/sase/bead/_sync_git.py`

- Extend `_list_bead_state_changes()` so it also reports index-vs-HEAD changes under the
  beads directory, e.g. by unioning the current
  `git ls-files --modified --others --deleted --exclude-standard -z -- <beads>/` output
  with `git diff --cached --name-only -z -- <beads>/`.
- Preserve current guarantees: stable order, de-duplicated paths, and the existing
  `beads.db` / SQLite-sidecar filtering applied to both sources.
- Handle an unborn HEAD (fresh clone with no commit): `git diff --cached` fails there,
  so detect that case and fall back to the `ls-files` result rather than raising
  `BeadWorkLaunchCommitError`.

Effects, both of which are the actual fix:

- `bead_state_is_clean()` reports staged-only bead state as dirty, so the finalizer's
  `_auto_commit_bead_state()` commits it through `commit_sdd_store_files` — with
  `TYPE=beads` _and_ runtime `SASE_AGENT=` provenance — **before** the first LLM pass,
  and the existing `_unpublished_bead_state_error()` publication check runs against that
  commit.
- `_commit_bead_state()` stops early-returning on staged-only content, so every
  bead-layer commit helper (claims, reconciliation, epic-graph checkpoint, external
  issue mirror) can flush it too.

Tests (`tests/bead/`): staged-only `issues.jsonl` makes `bead_state_is_clean()` return
`False`; a `_commit_bead_state()`-backed helper commits staged-only bead state; clean
store still returns `True`; worktree-modified and untracked cases unchanged; ignored
`beads.db` never reported; unborn-HEAD store does not raise.

### 2. Bead auto-commits must carry agent provenance

`src/sase/bead/_sync_git.py`

- In `_commit_bead_state()` (line ~338-340), replace
  `apply_auto_commit_type_tag(message, auto_commit_type)` with
  `apply_auto_commit_tags_with_runtime(message, auto_commit_type)`, matching what
  `commit_sdd_files()` already does for SDD commits. The `TYPE=` tag is unchanged;
  `SASE_AGENT=` is added when an agent identity is available and omitted otherwise.
- Grep for tests and callers that assert exact bead commit-message text and update them
  to tolerate the provenance footer.

Effect: bead commits SASE itself makes during a run are attributable, so
`_new_commits_are_attributable()` no longer reads them as foreign work.

Tests (`tests/bead/`): commit message carries `SASE_AGENT=` when the agent-identity env
is set, carries only `TYPE=beads` when it is not.

### 3. Dedupe dirty repos by path

`src/sase/llm_provider/commit_finalizer_state.py`

- In `collect_dirty_state()`, dedupe `repos` on `finalizer_git.normalize_path(path)`
  before the baseline split, keeping the most specific kind when the same path is
  reached by more than one source: `main` > `sdd` > `external` > `sibling`. Union the
  `changed_files` of merged entries; keep first-seen order.
- The `sdd` kind must win over `sibling` for a sidecar, because the machine auto-commit
  and prompt-rendering paths key off that kind.

Effect: one entry per repo in the finalizer prompt, in `progress_fingerprint()`, and in
failure messages — no more duplicated `- beads: …` lines and no duplicated sidecar
instructions burning prompt tokens.

Tests (`tests/llm_provider/`): a repo reachable both as a configured sibling and as an
SDD sidecar target yields exactly one `DirtyRepo`, with `kind == "sdd"` and the union of
changed files.

### 4. The guard must not call published store state discarded

`src/sase/llm_provider/commit_finalizer_git_progress.py`

- In `discarded_dirty_work_evidence()`, before recording `missing_agent_provenance`,
  skip the entry when **all** of these hold:
  - the repo is machine-managed store state (`DirtyRepo.kind == "sdd"`);
  - the repo is clean after the pass (it already left the dirty set);
  - the repo has an upstream and is not ahead of it —
    `git rev-list --count   @{upstream}..HEAD` is `0`. That is exactly the
    sync-rebase-absorbed case: the store's content is published and the clone is
    consistent with the canonical remote.
- Log a warning naming the repo, the before/after HEADs, and the changed files when this
  exemption applies, so a genuine discard is still auditable rather than silent.
- Change nothing else: the main repo, siblings, external repos, any repo left ahead of
  its upstream, any repo with no upstream, and any repo whose HEAD did not advance keep
  failing exactly as they do today.

Tests (`tests/llm_provider/test_commit_finalizer_no_progress.py` and neighbours): an
`sdd` sidecar that goes clean with only foreign upstream commits and no
ahead-of-upstream commits finalizes instead of raising; the same sidecar left ahead of
upstream still fails; an `sdd` sidecar with no upstream still fails; the existing
main-repo reset and foreign-agent-commit tests still fail as before.

## Verification

- `just install` first (ephemeral workspace), then `just check`.
- `just check-full` through `/sase_monitor` before landing, since this touches the bead
  store and the commit finalizer, both in the broadening set.
- Targeted regression: build a scratch git repo with `beads/issues.jsonl`, stage a
  change without committing, and assert `bead_state_is_clean()` is now `False` while
  `git_changed_files()` still reports the path — the two layers must agree.

## Risks

- Widening bead cleanliness makes two existing callers stricter: `sase doctor`'s beads
  check (`src/sase/doctor/checks_beads.py:391`) and the epic-graph pre-launch assertion
  in `src/sase/bead/cli_work_from_plan.py:592` ("bead-state changes remain
  uncommitted"). Both become _more_ correct, and change 1 also lets
  `commit_epic_graph_checkpoint()` flush staged-only state, so the assertion should
  still pass. Confirm this with a test that stages bead state and then runs the
  checkpoint path.
- Change 2 alters bead commit-message text; snapshot or exact-match assertions on those
  messages must be updated.
- Change 4 relaxes the guard for machine-managed store state only. It is deliberately
  gated on "clean and not ahead of upstream" so a local discard that leaves work
  unpublished still fails.

## Non-goals / follow-ups

- `issues.jsonl` rebase conflicts themselves. A JSONL-aware merge driver would stop the
  conflict that started this incident, but it is separate work.
- `sase_git_commit --resume` refusing to finish bookkeeping after the agent skipped a
  redundant replayed commit (run log line 165). Worth its own task.
- `sase_core_rs` exposes `bead_sync_is_clean`
  (`crates/sase_core/src/bead/mutation.rs:2057`) with an even narrower definition
  (`git diff --quiet issues.jsonl`, missing staged _and_ untracked state). Nothing in
  this repo calls it today, so it stays out of scope, but it should be aligned before
  anything starts using it.

## Boundary note

Bead reads and codecs live in `sase_core_rs`, but bead git staging/commit gating and the
commit finalizer are Python-side in this repo today, and the Rust `bead_sync_is_clean`
binding is unused. No Rust wire or binding change is required.
