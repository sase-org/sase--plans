---
tier: tale
title: Make the after-rebase import gate certify inside its timeout, then land sase-1vn
goal:
  The after-rebase import gate certifies real publications inside its hook timeout, epic
  sase-1vn is closed with its plan marked done, and the final backlog epic (Backlog Epic
  2) is proposed as a top-level epic that ends the chain.
size: medium
proposed_by: bbugyi200.athena.sase-1vn.land
proposed_with: claude/opus
bead: sase-1vn
status: done
---

- **BEAD:**
  [sase-1vn](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1vn/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-1vn.land](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.sase-1vn.land.md)
- **COMMITS:**
  - [4010ecd](https://github.com/sase-org/sase/commit/4010ecd89aec66a64f78b6591f4157d214f994aa)
    — fix(rebase-gate): certify committed tree from a single ls-tree index

# Make the After-Rebase Import Gate Certify Inside Its Timeout, Then Land sase-1vn

This tale finishes the landing of epic **sase-1vn** ("Backlog Epic 1: landing gates,
test cost, and epic liveness"). Its approved plan is `plan:202611/backlog_epic_1.md`.
The sase-1vn land agent verified everything else. Read its landing review note on
sase-1vn first: `sase bead read sase-1vn -r "Need the landing review and triage note"`.

That review established the following, so do not redo it:

- All 4 phases, both nested child epics (sase-1vn.3.1, sase-1vn.4.1), and their 7 phases
  are closed.
- Target beads sase-1d9, sase-1un, sase-j0, sase-1sh, and sase-12u are closed.
- Drift since the epic started needed no integration.
- `sase bead epic-symbols sase-1vn` is empty.
- Follow-up triage is done and recorded on sase-1vn. Do not file any more follow-ups for
  sase-1vn's children.

sase-1vn has no `parent_bead`. This tale has no land agent: the coder must finish the
closeout and the Backlog Epic 2 proposal in the same turn as the code change. None of
those steps depend on this tale's own commit SHA, push, or CI result.

Governing decisions: `decisions:check-full-is-explicit` (verify with
`sase tool run check`, never `just check-full`),
`decisions:handoffs-end-through-host-completion` (the Backlog Epic 2 proposal is staged
and adopted after this turn's host completion), and `decisions:host-owned-completion`.

## Remaining Work: The Gate Still Fails Open by Timeout

Phase sase-1vn.1 (commit f079628697) stopped gitignored virtualenv `.py` files from
counting as dirty. That fix is correct, but the live gate still fails open:

- Since that fix, after-rebase hook records show 2 certifications in 9 publications:
  f079628697 at 1.8s and dcd6c726c3 at 3.0s. The records are
  `commit_hooks/*after_rebase*.json` under the agents' ace-run artifact dirs.
- The other 7 publications (324d63e8ef, 1b461588eb, 51fc0d8cb2, 1cde333808, f6dd22d4dd,
  c783119ffd, 96e87d5984) hit `sase/sase.yml` `after_rebase_timeout_seconds: 10` at
  9.6s. Each logged "after-rebase hook timed out; allowing publication".
- On a quiet host (load ~4 on 64 cores), `tools/check_rebase_imports` took 18.9s, 10.4s,
  and 7.1s for the changed paths of 96e87d5984, a5fde6cd11, and 8610438838, measured
  against tree 8610438838.
- In a cProfile of the 18.9s case, about 13s is `CommittedTree._entry` in
  `tools/_check_rebase_imports.py`. It runs one `git ls-tree` and one
  `git cat-file blob` subprocess for every module the import walk loads: 4,412
  subprocesses for about 2,200 modules. `compile()` is another 3.4s.

### 1. Read each committed tree once, without per-file subprocesses

All changes are in `tools/_check_rebase_imports.py`, inside `CommittedTree`.

- Replace the per-path `_entry` subprocess pair with a lazily built index. On first use,
  run one `git ls-tree -r -z <tree>` from the repository root. Use `--full-tree` or the
  repo-root cwd so paths are root-relative. Bound that one call by
  `_GIT_TIMEOUT_SECONDS`. Parse it into `posix path -> (mode, object type, object id)`.
  Keep the existing `CheckerError` messages for these cases:
  - a path that is absent or does not resolve to exactly one entry;
  - a non-blob entry, such as a submodule `commit`;
  - a malformed entry;
  - a failed `git ls-tree`.
- Verify worktree identity without reading the blob. Compute the git blob object id of
  the worktree bytes: hash `b"blob %d\0" % len(data) + data`, with SHA-1 for a 40-hex
  object id and SHA-256 for a 64-hex id. Compare it with the indexed object id. For a
  committed symlink (`120000`), hash the `os.readlink` bytes the same way. This is
  byte-for-byte equivalent to today's comparison of raw worktree bytes with the raw
  committed blob; git filters were never applied there either.
- Keep everything else in `source_bytes` unchanged:
  - the path validation;
  - the per-path cache;
  - the mode checks (`100644`/`100755`/`120000`);
  - the worktree type checks;
  - symlink escape and cycle handling;
  - the "differs from committed tree" errors;
  - the return value. It stays the verified source bytes, which are the worktree bytes
    once their hash matches.
- Stay stdlib-only. `test_helper_never_imports_sase` must keep passing. Do not change
  `_python_paths_not_matching_tree`, compile/exec, or bytecode behavior.
  `test_stale_pyc_is_never_used_for_committed_source` must keep passing.

### 2. Tests

In `tests/test_check_rebase_imports.py`:

- Add a regression test showing that the git subprocess count does not depend on how
  many first-party modules are loaded. Build a fixture repo with many modules imported
  transitively (for example 30). Count `subprocess.run` calls made while running
  `check_repository`, for example by wrapping the helper module's `subprocess.run` with
  a counter. Assert that the count is a small constant (the four dirty-tree queries plus
  one `ls-tree`), not proportional to the number of modules.
- Unit-test `CommittedTree.source_bytes` directly, because the dirty-tree pre-check
  (`git diff <tree>`) normally rejects a modified worktree file before `source_bytes`
  runs. This is the hash path's defense in depth:
  - Commit a `.py` file, build `CommittedTree(repo, tree)`, then rewrite the file with
    different bytes of the same length. `source_bytes` must raise the "differs from
    committed tree" `CheckerError`.
  - The same must happen for a committed symlink retargeted in the worktree.
  - An unchanged file must return its exact bytes.
- Every existing test in the file must pass unchanged, especially these: the venv test,
  the ignored and untracked first-party tests, the dirty-file tests, the symlink tests,
  the stale-pyc test, and the stale installed-package test.

### 3. Measure against real changesets and settle the timeout

From the repo root, after the change, time the checker on the same three changesets.
Build each context like this:

```bash
for c in 96e87d5984 a5fde6cd11 8610438838; do
  python3 - "$c" <<'EOF'
import json, subprocess, sys
c = sys.argv[1]
repo = subprocess.check_output(["git", "rev-parse", "--show-toplevel"], text=True).strip()
head = subprocess.check_output(["git", "rev-parse", "HEAD"], text=True).strip()
tree = subprocess.check_output(["git", "rev-parse", "HEAD^{tree}"], text=True).strip()
paths = subprocess.check_output(["git", "show", "--name-only", "--format=", c], text=True).split()
ctx = {"schema_version": 1, "repository_root": repo, "current_head": head,
       "current_tree": tree, "changed_paths": [{"path": p} for p in paths]}
json.dump(ctx, open(f"/tmp/rebase_ctx_{c}.json", "w"))
EOF
done
```

Then time
`SASE_REBASE_CONTEXT=/tmp/rebase_ctx_<c>.json SASE_REBASE_RESULT=/tmp/rebase_res_<c>.json ./.venv/bin/python tools/check_rebase_imports`
for each changeset, for example with a small `time.monotonic()` wrapper. Run on a
committed, clean tree. If your edits are still uncommitted, first stash them, or run the
measurement from a scratch `git worktree` of your change, so the checker's dirty-tree
guard does not refuse. Every run must report `status: clean`.

Rule for the timeout: if the slowest of the three after-change runs exceeds 5.0s (half
of today's budget), raise `commit_hooks.after_rebase_timeout_seconds` in `sase/sase.yml`
from 10 to 30, the documented maximum. Otherwise leave it at 10. Either way, record the
before and after numbers.

### 4. Verify

Run `just fix`, then `sase tool run check`. Do not run `just check-full`. Treat UNKNOWN
triage as yours.

### 5. Record the evidence on sase-1un

Run
`sase bead note sase-1un "<before/after timings, subprocess-count test, timeout decision, check run id>"`.
sase-1un stays closed; do not reopen it.

## Final Step A: Close Epic sase-1vn

1. Run `sase bead epic-symbols sase-1vn`. It is empty today. If entries appear, resolve
   each symbol: wire it up, privatize it, add a non-test pragma, or delete it, per
   `symvision.md` (read it with `/sase_memory_read`). Re-key a Justfile line only to a
   still-open later bead that needs the exemption.
2. Run `sase bead close sase-1vn --note "<verification>"`. The note must summarize:
   - the land agent's landing review on sase-1vn: phases, child epics, and target beads
     closed; drift reviewed; triage outcomes;
   - this tale's checker fix, with before and after timings, the timeout decision, and
     the check run id.

   Never use `--force` to make the close succeed. If the close is rejected for
   epic-symbol entries, clean them up and close again.

3. Run `just symvision` and confirm it is clean.
4. Open the plans sidecar with `/sase_repo` (`sase repo open plans -r "..."`). In the
   printed path, set `status: done` in the frontmatter of `202611/backlog_epic_1.md`.
   That is the PLAN path `sase bead read sase-1vn` shows. Change nothing else in that
   file.
5. sase-1vn has no `parent_bead`, so there is no ancestor to close.

## Final Step B: Plan Backlog Epic 2, the Last Round

`plan:202611/backlog_epic_1.md` (section "Land: Close This Epic, Then Plan Backlog Epic
2") gives this tale ownership of the next-epic steps, because this tale is the child
plan of sase-1vn's landing. The sase-1vn land agent does not also plan it.

The user's note #1 on sase-1vn caps the chain at two rounds: "The epic lander for this
epic should create a new epic but let's stop it at two rounds. Make sure the epic lander
for the next epic is aware of this stop point." Backlog Epic 2 is therefore the last
backlog epic.

Order matters. Propose only after sase-1vn is closed (Final Step A). `sase plan propose`
makes an epic top-level when the proposing agent's active bead is already closed.
Proposing earlier would nest the chain under sase-1vn and recreate the sase-12u zombie
problem.

Follow the plan's Land steps 1-3, with the overrides below.

1. **Count.** Run
   `sase bead list -t task -s open -s ready -s snoozed -s claimed -s in_progress -n 0`.
   If it lists no task beads, say so and finish normally.
2. **Close duplicates and obsolete beads.** This is a scoped sweep, not a full
   re-triage. Check:
   - beads linked to, or in the same subsystems as, sase-1un, sase-1d9, sase-j0,
     sase-1sh, and sase-12u;
   - task beads created or +1'd since 2026-11-27;
   - every candidate you are about to pick.

   Close only with concrete evidence: a fixing commit plus current code that no longer
   has the defect, or a target that no longer exists. Use
   `sase bead close <id> -R done|canceled|superseded -r "<reason>" -n "<evidence>"`. A
   newer duplicate closes as `superseded` in favor of the older bead, and its evidence
   moves over as a note. A flake passing once is not evidence. Re-check the plan's close
   candidates first: sase-1uv and sase-1kq.

3. **Pick and plan.** Choose the five most important remaining task beads. Rank by
   severity: red master or CI gates first, then data loss or corruption, then
   user-facing breakage, then corroboration (+1 count and recency). Skip beads an active
   epic already owns. Start from the plan's "Starting queue for Backlog Epic 2" and
   re-verify each bead before picking it. Two updates since triage:
   - **sase-1vg** (the leak-detector thread poisons the CI Telemetry gate) is now the
     only red on the CI Telemetry `test-cost` job and on `just check-full`. sase-j0's
     budgets are green; see the evidence note on sase-1vg. Rank it first.
   - New flake beads filed since triage: sase-1vv, sase-1vw, sase-1vx.

   Use `/sase_plan` to author "Backlog Epic 2" with these rules:
   - at most five phases and at most five task beads;
   - phases run strictly in sequence (each depends on the previous one);
   - a phase may work more than one bead;
   - few `decisions:`, used mainly to let the user opt out of questionable feature or
     memory work;
   - a "Land" section that keeps the normal land-agent verification, integration,
     follow-up triage, and close of Backlog Epic 2, but **does not plan a Backlog Epic
     3**. It must say, in its own words, that the user capped the backlog chain at two
     rounds (sase-1vn note #1), that Backlog Epic 2 is the final round, and that its
     land agent finishes normally after closing the epic. If its landing needs a child
     plan, that child plan inherits the same stop. It should list the remaining clusters
     from the Backlog Epic 1 plan (bead-event actor attribution, bead-scale performance,
     stale memory notes) as candidates the user may pick up by hand. It must not
     instruct anyone to start another epic.

   Validate the plan (`sase plan validate ... --explain`, then without `--explain` until
   it passes), then propose it with `sase plan propose`. When it reports **Staged**,
   finish through `/sase_final`. Never wait on the proposal.
