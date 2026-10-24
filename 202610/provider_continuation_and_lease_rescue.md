---
tier: tale
title:
  Fix provider continuation crash, Muse post-declaration wait-guard misfire, and
  lease-reset data loss
goal:
  Provider continuations survive a second cycle, declared Muse turns end cleanly, and
  leases never silently discard uncommitted work.
size: medium
decisions:
  rescue_failure:
    ask:
      When a dirty leased checkout cannot be rescued, should the lease refuse to reset
      it?
    choices:
      proceed: Warn and reset anyway, as agent prep does; the lease stays available
      refuse:
        Raise and leave the tree; leases of that workspace fail until it is cleaned
    default: proceed
    why: Rescue failures are rare IO errors, and a wedged lease chop retries every ~40s
    answer: proceed
proposed_by: bbugyi200.athena.13t
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-24 14:52:01
status: wip
---

# Fix the provider continuation crash, the Muse post-declaration wait-guard misfire, and lease-reset data loss

## Incident

Agent `sase-1nf.14.1.3` (Muse, `muse-spark-1.3-contributor`) finished its phase. It
closed its bead, and the host accepted its final declaration (commit five files plus
bead close) at 18:20:25Z. Then three defects chained together, and the work was lost:

1. **Wait-guard misfire.** The reply's last line was a factual status report: the full
   `sase tool run check` "was still running under host load". `ends_with_wait_claim`
   matched "still running" in the 700-char tail, so `MuseProvider.invoke` started a
   `stranded_wait_claim` continuation, even though the turn had already been declared.
2. **Continuation crash.** The continuation re-entered `timer_context`. That object is
   built once before the loop from `provider_timer(...)`, which is a single-use
   `@contextmanager`. The second `__enter__` raised
   `AttributeError: '_GeneratorContextManager' object has no attribute 'args'`. The run
   was marked FAILED and the accepted declaration was never finalized. Auto-restart
   declined it as a "provider error".
3. **Silent reset.** The failed run held workspace #16 until it was dismissed at
   14:31:19 local. Fourteen seconds later the `bead_claim_checks` chop took an
   operational lease on #16. `prepare_from_primary_remote` ran
   `git checkout --force -B master origin/master`, which threw away the uncommitted
   tracked changes. No rescue entry was written. The code exists nowhere now, but the
   bead is CLOSED.

The same chain (accepted declaration, a "check run … is still running" bullet, the same
`AttributeError`) also failed agent `13l` earlier today. Both Muse wait-guard firings
this month crashed, so the Muse continuation path has never worked in production. Tests
missed it because every multi-cycle provider test patches `provider_timer` with a
reusable `MagicMock` and passes `suppress_output=True`.

## Changes

### 1. Make `provider_timer` reusable (`src/sase/output.py`)

Every caller except `src/sase/llm_provider/_grok_invocation.py` builds the timer once
before its continuation loop and re-enters it on each cycle:

- `muse_provider.py`
- `codex.py`
- `claude.py` (passes it into `_invoke_loop` as `timer_context`)
- `agy.py`
- `qwen.py`
- `opencode.py`
- `fakey.py`

So any second cycle (a user interrupt or a wait-guard continuation) crashes whenever
`suppress_output=False`, which is how real agent runs invoke providers.

Fix this centrally so the whole bug class goes away and no call site changes:

- Move the current generator body, unchanged, into a private one-shot helper.
- Make `provider_timer(message)` return a small reusable context-manager object:
  - `__enter__` builds a fresh one-shot timer and enters it, so each `with` block shows
    and times only its own cycle.
  - `__exit__` delegates to the active one-shot timer and returns its result, so
    exception propagation is unchanged. It then clears the active timer.
  - Entering the same object while it is already active raises `RuntimeError`. No caller
    nests, and this makes misuse loud.
- Keep the public name and call signature. Change the return annotation to
  `contextlib.AbstractContextManager[None]`.
- State in the docstring that one object may be reused across sequential `with` blocks.

Leave the provider call sites as they are.

### 2. Muse wait guard: an accepted declaration ends the turn (`src/sase/llm_provider/muse_provider.py`)

An accepted final declaration is the turn-ending contract (decisions
`single-turn-agents` and `host-owned-completion`). Continuing the model after acceptance
is wrong for two reasons:

- Muse has no headless resume, so the continuation restarts Muse from scratch with a
  rebuilt prompt.
- The continuation can change a tree whose content digest the host has already accepted.

Changes:

- Share the declaration signature. `_subprocess_plain.py` already identifies one
  acceptance of `final_submission.json` by inode plus mtime (`_file_signature`, used by
  `start_completion_watchdog`). Expose a small shared helper, for example
  `final_submission_signature() -> tuple[int, int] | None`, that resolves the path from
  `SASE_ARTIFACTS_DIR` and `FINAL_SUBMISSION_FILENAME`. Make the watchdog use it too.
- In `MuseProvider.invoke`, take the signature once before the `while True` loop.
- When `ends_with_wait_claim(content)` fires, first check whether the signature has
  changed since that snapshot, meaning a declaration was accepted during this invoke. If
  it has:
  - log `_log_wait_guard("wait_claim_after_accepted_declaration", cycle)`;
  - return the `InvokeResult` with no continuation and no raise.
- A `final_submission.json` that already existed before the invoke (from an earlier
  provider run in the same artifacts dir) must not count. The before/after comparison
  covers that.
- With no accepted declaration, keep today's continuation and budget behavior exactly.
- Update the comment above the guard to describe the exemption.

Leave Claude's guard alone. `_classify_claude_wait_state` only fires when there is
structural evidence (an outstanding background task or a `ScheduleWakeup` call), so
status prose alone can't trigger it.

### 3. Rescue tracked changes before the lease's forced checkout (`src/sase/workspace_provider/_lease_git.py`)

`prepare_from_primary_remote` is the only workspace-reset path with no stash or rescue
step. Agent prep stashes through `run_sase_hg_clean` and rescues when that fails.
Re-creation and sidecar eviction rescue through `rescue_git_repo`.

- Add a keyword-only `workspace_num: int` parameter. `acquire_operational_lease` in
  `src/sase/workspace_provider/lease.py` passes the claimed number. Update the existing
  callers in `tests/workspace_provider/test_workspace_lease_prepare.py` mechanically.
- After the `upstream is None` early return and just before the
  `checkout --force -B <branch> <upstream>` mutation, run
  `git status --porcelain --untracked-files=no`. That set (staged and unstaged changes
  to tracked files) is exactly what a forced checkout destroys; untracked files survive
  it.
- If that output is non-empty, or the status command itself fails, call
  `sase.workspace_provider.rescue.rescue_git_repo` with:
  - `checkout` as the repo root and as `workspace_dir`;
  - `workspace_num`;
  - label `lease-checkout`;
  - a reason naming operational lease preparation;
  - `include_worktree=True`.

  This writes `worktree.patch` (plus a bundle of local-only refs and stashes) to the
  durable rescue store outside the workspace and sends the rescue notification.

- Never call rescue when the tracked tree is clean. `rescue_git_repo` counts any
  existing stash as worktree changes, and pooled workspaces carry old stashes (#16 has
  five). An ungated call would write a rescue entry on every lease, and
  `bead_claim_checks` leases roughly every 40 seconds.
- If the tree is dirty and `rescue_git_repo` returns `None` (rescue failed):

> [!decision] rescue_failure = proceed Log a warning naming the checkout and the dirty
> paths, then run the forced checkout as today.

> [!decision] rescue_failure = refuse Raise `OperationalLeaseError("preparation", ...)`
> naming the checkout and the dirty paths, and skip the forced checkout.
> `acquire_operational_lease` already releases the claim on that error.

## Tests

- `tests/test_output.py`, with `sase.output.time.sleep` patched:
  - one `provider_timer(...)` object entered twice in sequence works;
  - an exception raised inside the block propagates, and the timer thread stops;
  - nested entry raises `RuntimeError`.
- `tests/llm_provider/test_muse_wait_guard.py`:
  - **Regression:** a wait-claim reply followed by a clean reply completes both cycles
    with the real `provider_timer` (not patched) and `suppress_output=False`. Patch only
    `_run_subprocess` and `sase.output.time.sleep`. Confirm this test fails with the
    `AttributeError` before change 1.
  - With `SASE_ARTIFACTS_DIR` pointing at a tmp dir, a fake `_run_subprocess` writes
    `final_submission.json` during cycle 1 and returns a wait-claim reply. `invoke`
    returns after one subprocess call, and `wait_guard_log.jsonl` records
    `wait_claim_after_accepted_declaration`.
  - A `final_submission.json` written before `invoke` starts, plus a wait-claim reply,
    still triggers exactly one continuation.
- One non-Muse provider with an existing two-cycle interrupt test (for example in
  `tests/test_llm_provider_qwen.py`): add a variant that keeps the real `provider_timer`
  and uses `suppress_output=False`, so the central fix stays pinned for the other loops.
- `tests/workspace_provider/test_workspace_lease_prepare.py`, using a real tmp repo with
  an origin remote:
  - a modified tracked file triggers a rescue (assert the `rescue_git_repo` call or the
    rescue entry with the store root redirected into tmp), and the checkout ends reset
    to upstream;
  - a clean tree with an existing stash triggers no rescue;
  - untracked-only changes trigger no rescue and leave the untracked file in place;
  - the chosen `rescue_failure` branch when `rescue_git_repo` returns `None`.

Run the lint and test lanes as the `lint_and_test` memory note directs (`just check`
through `sase tool run`).

## Non-goals

- Running an accepted declaration's finalizers when the provider fails after acceptance.
  That would have saved this work, but it changes host-owned-completion semantics and
  needs its own decision record. File it as a follow-up if wanted.
- Reclassifying SASE-internal exceptions wrapped in `LLMInvocationError`, which
  auto-restart currently declines as provider errors.
- Tightening `WAIT_SIGNAL_RE`. The declaration check fixes the observed misfires without
  new regex heuristics.
- Recovering `sase-1nf.14.1.3`'s lost code. That is operational and handled outside this
  plan.

## Operator note (not for the implementer)

`sase-1nf.14.1.3` is CLOSED, but its graph-allocation change (`CachedGraphNeeds`,
`classify_cached_graph_needs`, `own_required_apply_snapshot`) is not on any ref. Phase
`.6` and the epic's land agent will go ahead without it unless the bead is reopened.
There are two ways to get the work back:

- Reopen the bead and rerun the phase.
- Rebuild the diff from the Muse session log for session
  `9780b2e4-4d07-4634-8ab8-26ac075796a5`. It records 80 `edit_file` and 5 `write_file`
  calls (find/replace arguments), and the run started from base commit `602ff33c1b`.
