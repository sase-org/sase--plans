---
tier: epic
title: Fix in-tree bead root escape that aborts epic plan launches
goal: 'An epic launch archives and commits its approved plan in the repository that owns
  the in-tree SDD store, an in-tree beads root that cannot belong to that repository is
  rejected loudly instead of silently redirecting bead writes, and a commit target that
  is not a git repository is reported as such instead of as "nothing changed".

  '
phases:
  - id: bound
    title: Bound in-tree beads discovery to the owning checkout
    depends_on: []
    size: small
    description: "bound: stop the in-tree beads walk-up at the checkout that owns the
      cwd so a stale ~/sdd/beads outside every repository can no longer be adopted, and
      bound the legacy no-workspace fallback at its VCS root.

      "
  - id: invariant
    title: Derive the plan commit repository from the SDD store
    depends_on:
      - bound
    size: medium
    description: "invariant: make the epic launch commit its archived plan in the git
      repository that owns the store's plans root instead of in the beads root, reject a
      beads root that is not inside that repository, and make the in-tree store health
      preflight stop silently no-opping.

      "
  - id: diagnose
    title: Fail loudly when a commit target is not a git repository
    depends_on:
      - bound
    size: small
    description: 'diagnose: separate "target is not a git repository" from "nothing to
      commit" in the SDD commit helper, accept a .git file for worktree and submodule
      checkouts, and name the directory the launch actually tried to commit in.

      '
  - id: guard
    title: Doctor check and recovery for stores already redirected
    depends_on:
      - invariant
      - diagnose
    size: medium
    description:
      "guard: add a doctor check that fails when the resolved in-tree beads root sits
      outside the SDD store's repository, and produce an owner-confirmed recovery for
      the beads and archived plans this defect already misplaced on athena."
proposed_by: bbugyi200.athena.yf
create_time: 2026-09-09 20:00:10
status: wip
---

- **PROMPT:**
  [prompts/202608/in_tree_bead_root_escape.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/in_tree_bead_root_escape.md)

# Plan: Fix in-tree bead root escape that aborts epic plan launches

## Symptom

The most recent epic launch (`axe_chop_overrun_indicator`, 2026-08-12 09:04) exited 1
after passing validation and store resolution:

```
$ sase bead work /home/bryan/.sase/plans/202608/axe_chop_overrun_indicator.md \
    --yes-to-all --artifacts-dir ... --cl-name gh_sase-org__sase --expect-prompt-snapshot
✓ Validated       tier: epic · 4 phases · 3 dependency edges
✓ Store           in_tree · beads at /home/bryan/sdd/beads
Error: failed to commit archived epic plan
  /home/bryan/projects/git/gh_sase-org__sase/sdd/plans/202608/axe_chop_overrun_indicator.md
exited with code 1
```

This is not a one-off. `/home/bryan/projects/git/gh_sase-org__sase` reports
`?? sdd/plans/202608/` — five plans (`bead_wait_store_diagnostics`, `fix_sase_core_ci`,
`xprompt_properties_preview`, `sase_core_ci_parity_gate`, `axe_chop_overrun_indicator`)
were archived into the checkout today and none were ever committed.

The two printed lines are the tell: the store banner names a beads root under `$HOME`
while the error names an archived plan inside the project checkout. Those are two
different directories, and only one of them is a git repository.

## Reproduction

Run against the live checkout with the installed sase interpreter
(`/home/bryan/.local/share/uv/tools/sase/bin/python3`), cwd
`/home/bryan/projects/git/gh_sase-org__sase`:

```python
from sase.bead.cli_location import resolve_beads_location, _resolve_workspace_context
cwd = Path.cwd().resolve()
_resolve_workspace_context(cwd)
# _WorkspaceContext(root=.../gh_sase-org__sase, primary=.../gh_sase-org__sase,
#                   workspace_num=1, project_name='gh_sase-org__sase')
loc = resolve_beads_location(cwd=cwd, materialize=False)
# BeadsLocation(root=PosixPath('/home/bryan'), beads_dirname='sdd/beads',
#               storage='in_tree',
#               store=SddStore(storage='in_tree',
#                              sdd_dir=.../gh_sase-org__sase/sdd,
#                              repo_root=.../gh_sase-org__sase/sdd, ...))
loc.store.kind_root("plans")   # .../gh_sase-org__sase/sdd/plans   (correct)
loc.root                       # /home/bryan                        (wrong)
(loc.root / ".git").is_dir()   # False
```

Supporting filesystem state:

- `/home/bryan/sdd/beads/` exists (`beads.db` from 2026-07-12, `config.json` with
  `"issue_prefix": "bryan"`, 16 issues in `issues.jsonl` of which 14 are epic plan beads
  `bryan-2` … `bryan-f`).
- `/home/bryan/projects/git/gh_sase-org__sase/sdd/beads/` does **not** exist.
- `/home/bryan/.git` does **not** exist; `git -C /home/bryan rev-parse --show-toplevel`
  fails with "not a git repository".

## Root cause

All paths below are relative to the sase source repository. That repository is not the
project checkout in this workspace — open it with the `/sase_repo` skill before editing
and use the path it prints.

1. `src/sase/bead/cli_location.py:303` `_select_in_tree_beads_root()` walks
   `[cwd, *cwd.parents]` with no upper bound, returning the first ancestor that contains
   an `sdd/beads` directory. The project checkout has `sdd/` but no `sdd/beads`, so the
   walk leaves the repository, climbs to `$HOME`, finds the stale personal store created
   on 2026-07-12, and returns `/home/bryan`.

2. `src/sase/bead/cli_location.py:112-124` returns that root paired with the _correct_
   in-tree `SddStore` (whose `sdd_dir` is `<checkout>/sdd`). The resulting
   `BeadsLocation` is internally inconsistent: its beads root and its store's plans root
   live in different places, and nothing checks that they agree.

3. `src/sase/bead/cli_work_from_plan_store.py:95` computes
   `workspace_dir = location.root if store.is_in_tree else cwd`, so the plan-commit
   repository is taken from the _beads_ root — `/home/bryan`.

4. `archive_plan_file()` writes correctly, because `plan_archive_destination()` uses
   `store.kind_root("plans")`. The plan lands in
   `<checkout>/sdd/plans/202608/<name>.md`.

5. `src/sase/bead/cli_work_from_plan_store.py:389` `_plan_commit_store()` then rewrites
   the store's `repo_root` to `workspace_dir` (`/home/bryan`) for in-tree stores, and
   `src/sase/sdd/_commit_store.py:54` short-circuits:
   `if not (sdd_dir / ".git").is_dir(): return False`.

6. `commit_sdd_store_files()` therefore returns `False`,
   `src/sase/bead/cli_work_from_plan.py:309` sets `archive_committed = False`, and line
   317 raises `failed to commit archived epic plan <archived_path>`.

The error names the archived plan in the correct repository, so it points the reader at
a file that is fine. The directory that is actually broken (`/home/bryan`) never appears
in the message.

## Contributing defects

- **Silent no-op commit.** `commit_sdd_files()` returns `False` for "not a git repo",
  for "`.git` is a file" (git worktrees and submodules), and for "nothing to commit".
  Three very different conditions collapse into one boolean, and only the third is
  benign.
- **Dead health preflight.** `require_plan_store_health()`
  (`cli_work_from_plan_store.py:99-103`) returns immediately when
  `(store.repo_root / ".git").exists()` is false. For in-tree stores `repo_root` is
  `<checkout>/sdd` (`src/sase/sdd/_store_resolution.py:132` sets `repo_root=sdd_dir`),
  which never contains `.git`. The preflight that exists to catch a poisoned plans repo
  has therefore never run for any in-tree store, which is why the mismatch went
  undetected on both sides of the archive.
- **Bead writes were already misrouted.** Because the walk-up always found `~/sdd/beads`
  first, `resolve_plan_file_context()`'s
  `if not location.beads_dir.is_dir(): init_beads(...)` never fired for the checkout.
  Fourteen epic beads carrying the `bryan` prefix are sitting in the home store instead
  of the project store. That failure was silent — launches "succeeded".
- **Why it surfaced now.**
  `archive_committed = not archive_result.written or _commit_plan_file(...)` skips the
  commit entirely when the plan was already inside the plans root. Launches whose plan
  file was already archived never reached the broken commit. Only launches from
  `~/.sase/plans/...` — an external source that must be copied in — hit it, which is why
  the failure looks intermittent.
- **Same unbounded walk in the legacy path.** `_resolve_legacy_beads_location()`
  (`cli_location.py:321`) repeats the pattern for the no-workspace-context case.

## Bound in-tree beads discovery to the owning checkout

Change `_select_in_tree_beads_root()` so the ancestor walk cannot leave the checkout
that owns the cwd.

- Pass `context.root` (the checkout `_resolve_workspace_context()` resolved for this
  cwd) into the helper alongside `primary`, and stop the walk once a candidate is no
  longer `context.root` or a descendant of it.
- Keep the existing fallback order after the bounded walk: `context.root`, then
  `primary`. Under `require_existing=True` return `None` when neither has an `sdd/beads`
  directory; under `require_existing=False` (the materializing path) return the root so
  the caller's `init_beads()` creates the store in the right repository.
- Apply the same bound to `_resolve_legacy_beads_location()` using the VCS root of the
  cwd (`git_toplevel()` from `src/sase/sdd/_commit_bare_git.py:87`, or the equivalent
  provider-neutral helper). When no VCS root is discoverable, keep today's behaviour so
  non-repository usage is unaffected.

Tests in `tests/test_bead/test_cli_resolution.py`:

- A checkout with no `sdd/beads` whose _parent_ directory has one resolves to the
  checkout, not the parent — the regression this plan fixes.
- The existing `test_find_beads_location_in_tree_prefers_current_checkout` still passes
  (cwd's checkout wins over `primary` when both have `sdd/beads`).
- A numbered workspace with no `sdd/beads` still falls back to `primary` when `primary`
  has one.
- `require_existing=True` with no store in either location returns `None`.

## Derive the plan commit repository from the SDD store

Stop deriving the plan-commit repository from the beads root.

- In `resolve_plan_file_context()`, compute the in-tree `workspace_dir` from the store:
  the git repository that owns `store.kind_root("plans")`. Resolve it through the VCS
  root of `store.sdd_dir` rather than assuming `sdd_dir.parent`, so worktrees and nested
  checkouts resolve correctly. Non-in-tree stores keep using `cwd`.
- Add the missing invariant. When the store is in-tree and the resolved beads root is
  not inside the store's repository, raise with both paths named and with the remedy
  (the beads root belongs at `<repo>/sdd/beads`). Silently writing beads to a foreign
  store is the worse outcome; this is the check that would have caught the bug on day
  one.
- Fix `require_plan_store_health()` so it stops no-opping for in-tree stores: resolve
  the repository root for the health probe the same way (VCS root of the store's plans
  root) instead of testing `store.repo_root / ".git"`. Confirm no in-tree caller starts
  failing for an unrelated pre-existing reason before landing.
- Once `workspace_dir` is store-derived, re-check whether `_plan_commit_store()`'s
  `replace(store, repo_root=workspace_dir)` is still needed or whether `repo_root` for
  in-tree stores should be corrected at construction
  (`src/sase/sdd/_store_resolution.py:132`). Prefer the smaller change; if correcting
  construction, audit every `repo_root` reader for in-tree assumptions first and record
  the finding either way.

Tests in `tests/test_bead/test_cli_work_from_plan_store.py` and
`tests/test_bead/test_cli_work_from_plan.py`:

- An in-tree launch from a plan outside the plans root archives _and commits_ into the
  checkout, with the archived plan tracked afterwards — the end-to-end regression test
  for the reported failure.
- A beads root outside the store repository raises the new invariant error naming both
  paths.
- `require_plan_store_health()` actually probes an in-tree store's repository.

## Fail loudly when a commit target is not a git repository

Make the commit helper stop conflating unrelated conditions.

- In `commit_sdd_files()` (`src/sase/sdd/_commit_store.py:54`), treat "the target is not
  a git repository" as an error rather than a `False` return. Raise
  `SddRepositoryHealthError` naming the directory, so the caller cannot mistake it for
  "nothing changed". Keep `False` for the genuine no-op cases at lines 78 and 95.
- Accept a `.git` **file** as a valid repository marker (git worktrees and submodules).
  Today `.is_dir()` rejects them and they take the silent-`False` path.
- Audit callers of `commit_sdd_files()` / `commit_sdd_store_files()` for ones that rely
  on the current tolerant `False` for a non-repository directory — in particular any
  best-effort or optional commit path — and keep those tolerant explicitly rather than
  by accident.
- Extend `cli_work_from_plan.py:317` so the message names the directory the commit was
  attempted in as well as the archived plan path.

Tests in `tests/test_sdd_commit_store.py`:

- A non-repository directory raises with the directory named.
- A `.git` file worktree commits successfully.
- A clean repository with no changes still returns `False`.

## Doctor check and recovery for stores already redirected

Close the loop for machines already in the bad state.

- Add a check to `src/sase/doctor/checks_beads.py` that fails when the resolved in-tree
  beads root is not inside the SDD store's repository, reporting both paths and the
  expected `<repo>/sdd/beads` location. Cover it in the doctor test suite. This is the
  detector for the class of failure, independent of the specific walk-up bug.
- Produce a written recovery procedure for athena's current state and **obtain the
  project owner's confirmation before running any part of it**. It must cover:
  - the fourteen `bryan-*` epic beads in `/home/bryan/sdd/beads` that belong in the
    project store — decide per bead whether to migrate or abandon, and state how ids and
    the `bryan` prefix are reconciled against the project store's prefix;
  - the five untracked plans in
    `/home/bryan/projects/git/gh_sase-org__sase/sdd/plans/202608/`, which need
    committing through `/sase_git_commit` once the fix is in;
  - what happens to `/home/bryan/sdd/beads` afterwards (leave in place, or move aside so
    the bounded walk cannot be tempted by it again).
- Do not mutate `/home/bryan/sdd/beads` or the project bead store without that
  confirmation. Recovery is destructive and reversible only from backups.

## Verification

1. The reproduction snippet above resolves `loc.root` to
   `/home/bryan/projects/git/gh_sase-org__sase`, and `(loc.root / ".git").is_dir()` is
   true.
2. A fresh `sase bead work` on an epic plan sourced from `~/.sase/plans/...` completes,
   and `git -C <checkout> status --porcelain sdd/plans` is clean afterwards.
3. `sase doctor` reports the new beads-root check as passing on a healthy checkout and
   failing on a synthesized mismatch.
4. The full test suites for `tests/test_bead/` and the SDD commit tests pass.

## Out of scope

- Changing SDD storage policy selection, or how a project chooses in-tree versus sidecar
  storage.
- Reworking `SddStore.repo_root` semantics beyond what the `invariant` phase concludes
  is the smaller correct change.
- Backfilling plan-to-bead links for epics that were launched while beads were being
  written to the home store, beyond what the `guard` recovery decides.
