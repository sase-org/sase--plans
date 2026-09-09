---
status: done
tier: epic
title: Make the commit finalizer's protection baseline truthful
goal: "A turn's own work is never mistaken for foreign dirt. The dirty-path baseline is
  captured before the agent can write, every consumer reads it through one contract, the
  finalizer refuses to dispatch a stitch that protection has already emptied, and a
  stitch failure reports the reason the VCS provider actually gave instead of spending a
  whole attempt budget on an identical, guaranteed-to-fail retry.

  "
phases:
  - id: scope
    title: One baseline, one answer about who owns a path
    depends_on: []
    size: medium
    description: "scope: give finalizer_baseline.json a single documented read contract
      so the evidence an agent reads and the protection the dispatcher applies can never
      contradict each other for the same repository path, and pin that with an invariant
      test.

      "
  - id: checkouts
    title: Baseline every checkout that exists before the first turn
    depends_on:
      - scope
    size: medium
    description: "checkouts: snapshot every repository checkout already present in the
      workspace at runner start rather than only the dirty ones, and make a later repo
      open unable to rebase an existing record for the same path onto the agent's own
      work.

      "
  - id: attribution
    title: Repair run-written path attribution outside the primary repo
    depends_on: []
    size: small
    description: "attribution: stop discarding the absolute tool-call path before the
      only code that can relativize it against a repository root, so direct writes into
      linked, sidecar, and external repos are visible to both the declaration evidence
      and the deferral counter-evidence check.

      "
  - id: guard
    title: Never dispatch a stitch that protection has already emptied
    depends_on:
      - scope
    size: medium
    description: "guard: detect before dispatch that every changed path in a repository
      is excluded, refuse the doomed sase stitch create, and raise a specific
      non-retryable diagnostic that names the protected paths and the baseline record
      that protected them.

      "
  - id: fidelity
    title: Truthful stitch failures and a retry budget that cannot be wasted
    depends_on: []
    size: medium
    description: "fidelity: carry the VCS provider's real reason through the failure
      message, diagnostics, and error report, and stop spending a second mutating
      attempt on a failure whose inputs are provably identical to the first.

      "
  - id: verify
    title: Replay the failure end to end and land the tree
    depends_on:
      - checkouts
      - attribution
      - guard
      - fidelity
    size: medium
    description:
      "verify: add a live end-to-end regression that reproduces the original
      research.13.cdx sequence and now commits, then run the full landing gate."
proposed_by: bbugyi200.athena.0d9
bead_id: sase-ti
create_time: 2026-09-09 19:50:10
---

- **PROMPT:**
  [prompts/202608/commit_finalizer_protection_truth.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/commit_finalizer_protection_truth.md)
- **BEAD:**
  [sase-ti](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ti/README.md)

# Plan: Make the commit finalizer's protection baseline truthful

## Why this epic exists

Agent `research.13.cdx` (run `20260825070100`, CODEX `gpt-5.6-sol` @ xhigh) produced a
complete 318-line research report, wrote it into the `research` SDD sidecar, submitted a
valid final declaration asking to commit it — and then the run **failed**, discarding
20m56s of work and stranding its workspace. Nothing about the agent's behavior was
wrong. The commit finalizer excluded the only file the agent had produced and then
failed because nothing was staged.

Every claim below was reproduced against that run's artifacts, which are durable at:

```
~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202608/25/20260825070100/
```

### The failure, exactly

`finalizers/commit/attempt-{1,2}.research.stdout`:

```
🔄 Running before commit hook: sase_git_fix
🔄 Dispatching create_commit to VCS provider...
❌ create_commit failed: No staged changes to commit
```

That string comes from `_validate_staged` in
`src/sase/vcs_provider/plugins/_git_commit_dispatch.py:73`. It fires because
`vcs_create_commit` (same file, line 402) staged with `_stage_all_except` (line 140),
and the single exclude it was handed covered the repository's only changed file:

```
git add -A -- :/ ':(exclude,top)202608/remove_direct_git_plugin_installs.md'
```

The exclude came from `src/sase/finalizers/commit_dispatch.py:175-177`:

```python
protected = protected_path_resolver(artifacts, repo.path)
before_markers = load_commit_results(artifacts)
stitch = stitch_runner(repo, message, protected, context)
```

and `run_stitch_create` (`src/sase/finalizers/commit_repair.py:42`) turns each protected
path into a `-x` flag on `sase stitch create`.

### Root cause: the baseline was captured after the agent wrote the file

`finalizer_baseline.json` for that run contains, verbatim:

```json
{
  "fingerprints": {
    "202608/remove_direct_git_plugin_installs.md": [
      "??",
      "727c244a43888a5ac147acfcc54b5699862f777c"
    ]
  },
  "kind": "linked",
  "name": "research",
  "repo_id": "linked:research",
  "scope": "opened_repo"
}
```

`727c244` is the blob of the report the agent wrote. Sequence from `tool_calls.jsonl`
and the sidecar's reflog (UTC):

| Time                | Event                                                                                                                     |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| 11:02:01            | workspace prep clones the `research` SDD sidecar into the workspace                                                       |
| 11:03:38            | agent opens `chezmoi` and `gh:bbugyi200/bugyi-chops`; both baselined empty                                                |
| 11:09:25            | agent `Write`s `202608/remove_direct_git_plugin_installs.md` into the already-present sidecar                             |
| 11:21:06            | agent finally runs `sase repo open sase--research`, which **now** captures the baseline — with its own file already in it |
| 11:21:48            | host accepts the declaration: `action: commit`                                                                            |
| 11:22:01 / 11:22:03 | both stitch attempts excluded that path and failed                                                                        |

`capture_opened_repo_dirty_baseline`
(`src/sase/llm_provider/commit_finalizer_baseline.py:61`) snapshots dirt at **open**
time. Its docstring promises "the first baseline for a repository ID wins, so repeated
opens cannot rebase the already-protected dirt" — but the _first_ open was already
nineteen minutes too late, because SDD sidecar checkouts exist in the workspace before
the agent's first turn and are readable and writable without ever being opened. The
run's `#research(report_target=...)` expansion even instructed the agent to write
directly into that sidecar directory, with no instruction to open it first.

### The contradiction the agent was shown

`final_context.json` reported both of these about the same path:

```json
{
  "path": "202608/remove_direct_git_plugin_installs.md",
  "provenance": "new_since_run_start",
  "written_by_this_run": false,
  "protected": true
}
```

`already_dirty_at_run_start_paths` was `[]` while `protected_paths` listed that file.
Two readers of one artifact disagree, and this reproduces today:

```python
load_dirty_baseline(root)                   # -> {}
_load_baseline_fingerprints(root, repo)     # -> {'202608/...md': ('??', '727c244a...')}
```

- `_load_finalizer_dirty_baseline`
  (`src/sase/llm_provider/commit_finalizer_baseline.py:155`) **skips** any record whose
  `scope != run_start` (line 176). It feeds `collect_dirty_state` and the `provenance`
  field.
- `_load_baseline_fingerprints` (`src/sase/finalizers/commit_validation.py:97`) applies
  **no scope filter**. It feeds `protected_baseline_paths` (line 77), which feeds the
  `-x` excludes and the `protected` flag.

So `provenance` said "this is your work" while protection said "this is not yours", and
protection is the one that touches the index.

### `written_by_this_run: false` was also wrong

The agent wrote that path through a tracked `Write` tool call, recorded in
`tool_calls.jsonl` with the absolute path. Reproduced:

```python
written_paths_from_tool_calls(root)
# -> ('sase/repos/research/202608/remove_direct_git_plugin_installs.md',)
_direct_written_paths(repo_path=<sidecar>, written_paths=..., named_paths=('202608/...md',))
# -> ()
```

`written_paths_from_tool_calls`
(`src/sase/finalizers/declaration_recovery_evidence.py:208`) passes every path through
`_workspace_relative` (line 241), which rewrites the absolute path relative to
`Path.cwd()`. Both `_direct_written_paths` implementations
(`src/sase/finalizers/declaration_context_evidence.py:181` and
`src/sase/finalizers/declaration_deferrals.py:260`) only re-relativize a candidate when
`candidate_path.is_absolute()` — which is now never true. The result: a direct write
into any repository other than the primary workspace checkout is invisible. That is not
only lost evidence in the context; it is a hole in `_reject_run_owned_paths`, whose job
is to refuse a bogus `protected_paths` deferral by proving the run wrote the path
itself.

### Two failures where there should have been one clear one

- `stitch_failure_message` (`src/sase/finalizers/commit_repair.py:267`) returns
  `(result.stderr or result.stdout)`. stderr was non-empty but useless — "Commit message
  preserved at ... re-run with the same -M flag after fixing" — so the actual reason, on
  stdout, was dropped from the diagnostic, from `error_report.md`, from `done.json`, and
  from what the user saw. Diagnosing this run required opening the per-attempt artifact
  files by hand.
- `stitch_failed` is in `RETRYABLE_DIAGNOSTIC_CODES`
  (`src/sase/finalizers/ledger.py:17`), so `controller.py:469` retried. Attempt 2 ran
  two seconds later against a byte-identical repository, exclude set, and message, and
  failed identically. The retry budget existed to survive transient failures; here it
  guaranteed the run died.

### Design intent this epic must preserve

`opened_repo` baselines exist for a real reason (bead `sase-rn.3`): a linked repo like
`chezmoi` can hold the user's own uncommitted edits, and an agent must never sweep those
into its commit. Nothing here weakens that. The bug is _when_ the snapshot is taken and
_who_ is allowed to disagree about it — not that protection exists.

## Cross-cutting constraints

- **Rust core boundary.** Per `sase/memory/rust_core_backend_boundary.md`, ask of each
  change whether another frontend would need the same answer. The baseline artifact's
  read contract and the "is this path this run's own work" predicate are shared domain
  logic; the existing implementation lives in Python under `src/sase/finalizers/` and
  `src/sase/llm_provider/`, and this epic does **not** relocate it. If a phase finds
  itself designing a new wire type rather than fixing a reader, stop and route that
  through `../sase-core` first.
- **Never widen what gets committed.** Every change here must be provably
  protection-preserving for genuinely foreign dirt. Any phase that relaxes a guard must
  add a test that a truly pre-existing edit is still excluded.
- **Baseline capture stays best-effort.** `capture_dirty_baseline` deliberately swallows
  exceptions so a broken snapshot degrades to pre-baseline behavior instead of killing a
  run. Preserve that; do not convert a capture failure into a hard error.
- **Family-attach inheritance is correct and must survive.**
  `_inherit_parent_commit_finalizer_baseline`
  (`src/sase/axe/run_agent_runner_bootstrap.py:103`) copies a parent's baseline for a
  same-lane continuation. Phase `checkouts` must not capture over an inherited baseline.
- **No memory-file edits.** No phase may touch `sase/memory/*.md`, `AGENTS.md`, or a
  generated provider shim. If a phase believes a memory note is now wrong, it records a
  `PROPOSED FOLLOW-UP:` note on its own bead.
- **Related in-flight work.** Epic `sase-rr` (retire the pluggable finalizers beta) has
  all phases closed and is in landing; `sase-sc` wants the conflict-repair turn to get
  the recovery evidence brief. Neither owns these files right now, but rebase before
  landing and check for conflicts in `src/sase/finalizers/`.

---

## Phase `scope`: One baseline, one answer about who owns a path

**Gap.** `finalizer_baseline.json` has two readers with two different scope rules, and
nothing forces them to agree.

Settle the contract explicitly and write it down in module docstrings:

1. Decide and document what each scope means. `run_start` records describe dirt that
   predates the agent's first turn. `opened_repo` records describe dirt that predates
   this run's first contact with a repository whose checkout it did not previously have.
   Both are legitimate "not this run's work" evidence, so **protection must consider
   both** — which means `_load_finalizer_dirty_baseline`'s `run_start`-only filter is
   the reader that is wrong for provenance purposes, not the unfiltered one.
2. Introduce a single loader that returns records with their scope intact, and express
   both existing entry points in terms of it: the provenance/`collect_dirty_state` view
   and the protection view must be derived from the same records, so a path can never be
   `new_since_run_start` and `protected` at once.
3. Define record precedence when more than one record matches a normalized repository
   path. This is reachable today: the same physical sidecar is `sdd:research` to
   `_repo_id` (`commit_finalizer_baseline.py:196`) and `linked:research` to
   `sase repo open`, so one path can hold two records with different ids, and
   `_load_baseline_fingerprints` silently returns whichever it walks first. Pick the
   **earliest-captured** record for a path and make the choice deterministic and tested,
   not list-order-dependent.
4. Keep the legacy `commit_finalizer_baseline.json` reader read-only, as documented.

**Acceptance.**

- An invariant test loads a synthetic baseline containing `run_start`, `opened_repo`,
  and duplicate-path records, and asserts that for every (repo, path) pair the
  provenance view and the protection view give consistent answers — `protected` implies
  a non-`new_since_run_start` provenance, and vice versa.
- A regression test builds the exact `finalizer_baseline.json` from run `20260825070100`
  and asserts the two views agree about `202608/remove_direct_git_plugin_installs.md`.
- Existing finalizer suites stay green, in particular
  `tests/test_finalizers_commit_reconciliation*.py` and
  `tests/test_finalizer_declaration_channel_context.py`.

## Phase `checkouts`: Baseline every checkout that exists before the first turn

**Gap.** `capture_dirty_baseline`
(`src/sase/llm_provider/commit_finalizer_baseline.py:32`) records only
`dirty_state.repos` — repositories that are _already dirty_ at runner start. A clean
sidecar gets no record, so the first `sase repo open` writes the record instead, at
whatever time that happens, capturing whatever the agent has written by then.

1. At runner start, enumerate every repository checkout that already exists in the
   workspace and record a `run_start` baseline for each, dirty or clean. The SDD
   sidecars are the case that actually bit: `_dirty_sdd_store_repos`
   (`src/sase/llm_provider/commit_finalizer_state.py:247`) already knows how to walk
   them via `sdd_commit_targets(store, None)` — reuse that enumeration but drop its
   `if not changed_files: continue` short-circuit. A clean repo yields an empty
   `fingerprints` map, which is exactly the record that makes later work attributable.
   Also cover the agents prompt-archive sidecar and any pre-existing `sase/repos/**`
   checkout with a `.git` entry.
2. Make `capture_opened_repo_dirty_baseline` idempotent by **normalized path** as well
   as `repo_id`. Today the "first write wins" guard is keyed only on `repo_id`, so the
   `sdd:` and `linked:` views of one sidecar do not collide. A later open of a path that
   already has any record must be a no-op, not a second record.
3. Preserve the existing guard that refuses to recapture a `repo_id` under a different
   path, and keep returning a diagnostic string rather than raising.
4. Do not capture when `_inherit_parent_commit_finalizer_baseline` inherited a parent
   baseline.
5. Keep the whole capture best-effort and bounded: a workspace with many sidecars must
   not turn startup into a long `git status` fan-out. If enumeration needs a cap, cap it
   explicitly and log what was skipped rather than silently truncating.

**Acceptance.**

- A test builds a workspace with a **clean** pre-existing sidecar checkout, writes a new
  file into it _without_ opening the repo, then opens the repo, and asserts the file is
  attributed to the run: not protected, not excluded, and committed.
- A test with a sidecar that is **genuinely dirty** at runner start asserts that dirt is
  still recorded, still protected, and still excluded from the commit.
- A test asserts a second `sase repo open` of an already-baselined path leaves the
  record byte-identical.
- A family-attach continuation test asserts the inherited baseline is not overwritten.

## Phase `attribution`: Repair run-written path attribution outside the primary repo

**Gap.** The absolute path recorded in `tool_calls.jsonl` is destroyed by
`_workspace_relative` before the only code that could match it against a repository root
ever sees it, so `written_by_this_run` is structurally `false` for every non-primary
repository.

1. Stop throwing the absolute path away. `written_paths_from_tool_calls`
   (`src/sase/finalizers/declaration_recovery_evidence.py:208`) should preserve the
   original absolute path alongside any workspace-relative rendering, so callers can
   choose. Note that `_workspace_relative` depends on `Path.cwd()` at call time, which
   makes its output a function of where the finalizer happens to be running — another
   reason the raw path must survive.
2. Fix both `_direct_written_paths` implementations
   (`src/sase/finalizers/declaration_context_evidence.py:181` and
   `src/sase/finalizers/declaration_deferrals.py:260`) to relativize any candidate
   against the repository root, absolute or not. The two copies are near-identical and
   drifting; collapse them into one shared helper.
3. Keep the human-readable "Files this run wrote directly" section rendering paths the
   way it does today.

**Acceptance.**

- A regression test replays the real `tool_calls.jsonl` row from run `20260825070100`
  and asserts `written_by_this_run` is `true` for
  `202608/remove_direct_git_plugin_installs.md` against the sidecar root.
- A test asserts `_reject_run_owned_paths` now refuses a `protected_paths` deferral for
  a sidecar path the run wrote directly — the safety hole this bug opened.
- A test asserts the matcher is not fooled by a path that merely shares a prefix with
  the repository root.
- `tests/test_finalizer_declaration_recovery_evidence.py` stays green.

## Phase `guard`: Never dispatch a stitch that protection has already emptied

**Gap.** `dispatch_commit_decisions` computes excludes and dispatches without ever
asking whether anything survives them. When protection covers every changed path, the
stitch is guaranteed to fail, and it fails with a message that explains none of this.

1. Before dispatching, compute the changed paths that remain after protection. If the
   set is empty for a repository whose accepted action is `commit`, do not run
   `sase stitch create`.
2. Fail with a new, **non-retryable** diagnostic code that states the contradiction
   plainly: the declaration accepted a commit for this repository, protection excluded
   every changed path, here are those paths, and here is the baseline record (its
   `scope`, `repo_id`, and capture provenance) that protected each one. An operator
   reading only `error_report.md` should be able to diagnose it.
3. Add the same check on the submit side. `final_context.json` already knows the
   obligation paths and the protected paths; a context whose every obligation path for a
   repository is protected is a host bug, and the declaration flow should say so rather
   than accept a `commit` decision it cannot honor. Emit an actionable diagnostic naming
   the deferral reason `protected_paths` as the agent-side option.
4. Do not consume the mutating attempt budget for a failure the host detected before
   touching the repository.
5. Leave `_validate_excludes` (`_git_commit_dispatch.py:106`) alone; its unmatched-
   exclude refusal is correct and orthogonal.

**Acceptance.**

- A test where protection covers every changed path asserts: no `sase stitch create`
  subprocess is spawned, the diagnostic code is the new one, the message names the paths
  and the protecting baseline records, and `ledger.consumed == 0`.
- A test where protection covers _some_ changed paths asserts the stitch still runs and
  commits the rest.
- A test asserts the new diagnostic is not in `RETRYABLE_DIAGNOSTIC_CODES`.

## Phase `fidelity`: Truthful stitch failures and a retry budget that cannot be wasted

**Gap.** The real reason is discarded, and a deterministic failure is retried anyway.

1. Change `stitch_failure_message` (`src/sase/finalizers/commit_repair.py:267`) to
   include **both** streams, bounded and labelled, so `❌ create_commit failed: ...`
   always reaches the diagnostic. Preserve the existing exit-code fallback for the
   silent case, and respect the existing output-cap handling.
2. Record the dispatch inputs in the per-attempt artifacts written by
   `record_stitch_artifacts`: the exclude list, the resolved message file, the repo
   `HEAD`, and the argv. This run's artifacts contained the stitch's _output_ but not
   its _inputs_, which is why reconstructing the `-x` flag required reading source.
3. Make a retry require evidence that something changed. Before spending a second
   mutating attempt on a `stitch_failed` result, fingerprint the attempt inputs (repo
   path, `HEAD`, dirty-path fingerprints, exclude set, message digest). If the
   fingerprint is unchanged from the failed attempt, do not retry: record a distinct
   terminal diagnostic saying the retry was skipped because the inputs were identical,
   and surface the first attempt's real reason as the failure. Keep genuinely transient
   failures retryable.
4. Make sure the top-level `error_report.md` and the `RESPONSE:` block in `sase.md`
   carry the enriched message; they are what a person actually reads.

**Acceptance.**

- A test whose fake stitch writes the reason to stdout and boilerplate to stderr asserts
  the reason appears in the diagnostic message and in the rendered error report.
- A test asserts a `stitch_failed` whose inputs are unchanged consumes exactly one
  mutating attempt and reports the skipped-retry diagnostic.
- A test asserts a `stitch_failed` whose inputs _did_ change still retries, so the
  transient-failure path is preserved.
- `tests/test_finalizers_execution_ledger.py` and
  `tests/test_finalizers_historical_refusal_corpus.py` stay green.

## Phase `verify`: Replay the failure end to end and land the tree

1. Add a live end-to-end regression in the `tests/test_finalizers_live_e2e*.py` family
   that reproduces the original sequence exactly: a pre-cloned clean sidecar in the
   workspace, a file written into it with no prior repo open, a `sase repo open` after
   the write, a declaration whose action is `commit`. Assert the commit lands, exactly
   one mutating attempt is consumed, and the aggregate finalizer status is `success`.
2. Add the negative twin in the same harness: a sidecar that is dirty before the run
   starts, written to by nobody, must still be protected and left uncommitted, with the
   agent's own separate change still committed.
3. Confirm the diagnostics an operator sees. For a deliberately broken protection setup,
   assert `error_report.md` names the reason and the protected paths.
4. Run the landing gate through a monitor, never inline:

   ```bash
   sase monitor start --command 'just check-full' \
     --start-status TESTING --stop-status TESTED --next '...'
   ```

   `just install` first — this epic's phases may run in workspaces that have been idle.

5. Record any unrelated red found by `check-full` as a `PROPOSED FOLLOW-UP:` note on
   this phase's bead rather than fixing it here.

**Acceptance.**

- Both live e2e regressions pass.
- `just check-full` is green, or every remaining failure is proven pre-existing on
  `master` and recorded as a follow-up note.
