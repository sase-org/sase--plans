---
tier: epic
title: Host-owned epic launches stop failing their planner agent
goal: 'An approved epic whose host-owned `sase bead work` launch is still running
  (or already succeeded) never marks its planner agent FAILED, a planner-side SDD
  publication failure is reported as exactly that, and a `sase dev update` source
  swap under a live agent runner is prevented where it can be and named honestly where
  it cannot.

  '
phases:
- id: attribution
  title: Host-owned launches own their own outcome
  depends_on: []
  size: medium
  description: 'attribution: stop converting a planner-side SDD store failure into
    `epic_launch_failed` when the approval response assigned launch ownership to the
    host, degrade to `epic_approved` with a recorded publication error, and carry
    that degradation into the completion notification instead of a bogus resume command.

    '
- id: skew_guard
  title: Agent runners survive mid-run editable source swaps
  depends_on: []
  size: medium
  description: 'skew_guard: preload the post-gate import surface once at agent-runner
    start so a later `sase dev update` cannot tear a deferred import, snapshot the
    source revision the process booted against, and label failures whose cause is
    an import error after a swap as a code swap rather than an unusable store.

    '
- id: swap_visibility
  title: Dev updates name the live runners they may tear
  depends_on:
  - skew_guard
  size: small
  description: 'swap_visibility: register long-lived agent runners as advisory, non-blocking
    code-swap readers and surface them in `sase update` and the ACE update preview,
    without ever letting an agent defer a source swap.'
proposed_by: bbugyi200.athena.tl
create_time: 2026-08-05 18:31:19
status: wip
bead_id: sase-fl
---

- **PROMPT:** [prompts/202608/epic_launch_false_failure.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/epic_launch_false_failure.md)
- **BEAD:** [sase-fl](https://github.com/sase-org/sase--beads/blob/main/pages/sase-fl/README.md)

# Plan: Host-owned epic launches stop failing their planner agent

## The incident

On 2026-08-05 the planner agent `tg` (`gh_sase-org__sase`, artifacts `.../artifacts/ace-run/202608/05/20260805165837`)
proposed the `parallel_suite_contention_reliability` epic plan. The user approved it in ACE at 18:13. ACE showed the
agent as `FAILED` with `ERROR / Epic launch failed`, while the SASE Admin Center simultaneously showed the detached
`sase bead work .../parallel_suite_contention_reliability.md --yes-to-all --artifacts-dir ... --cl-name gh_sase-org__sase --expect-prompt-snapshot`
task still `Working...`.

The launch was never in trouble. It completed and wrote its result back into that same agent's metadata three minutes
after the agent had already been declared failed:

```
epic_launch_error = "cannot import name 'issue_id_for_counter' from 'sase.bead.ids'
                     (/home/bryan/projects/github/sase-org/sase/src/sase/bead/ids.py)"
stopped_at        = "2026-08-05T22:13:47Z"      # agent marked FAILED
epic_bead_id      = "sase-fd"                    # launch succeeded
epic_started_at   = "2026-08-05T22:16:17Z"
```

So the user saw a red FAILED agent, a failure notification carrying a `Resume with: sase bead work ...` command that
would have created a **second** epic, and a successful epic launch, all for the same approval.

## Root cause

Two independent defects compose into the observed contradiction.

### Defect A — the planner claims an outcome it does not own

`handle_accepted_plan` (`src/sase/axe/run_agent_exec_plan_accept.py`) treats any `SddMaterializationError` /
`SddRepositoryHealthError` raised while publishing the planner's own SDD artifacts as an epic launch failure: it records
`epic_launch_error`, calls `_notify_epic_launch_failure` (failure notification plus a `sase bead work` resume command)
and returns the `epic_launch_failed` loop outcome, which `done.json`, `running_listing.py` and the ACE done-loaders all
render as `FAILED / Epic launch failed`.

That attribution is wrong for host-owned launches, and host-owned is the only modern case:

- Every approval surface assigns ownership to the host — `plan_approval_actions.py:166`, `plan_gate.py:279`,
  `notification_gates/adapters.py:154`, `ace/tui/actions/agents/_notification_modals.py:320` — and the response carries
  `epic_launch_owner: "host"` into `PlanApprovalResult.epic_launch_owner` (`src/sase/llm_provider/_plan_utils.py:28`).
  Nothing under `src/sase/axe/` reads that field today.
- The host submits the detached `sase bead work` task in the same call that writes the gate response
  (`plan_approval_actions.py:170-180`), so by the time the planner resumes, the launch is already committed. The comment
  "stops before launcher" on the existing test (`tests/test_axe_run_agent_exec_plan_followup_approvals.py:206`)
  describes a launcher the agent no longer gates.
- The host already ran its own store preflight against the launch cwd (`_plan_approval_epic.prepare_epic_launch` →
  `require_epic_launch_store_health`). The store the planner failed on is a _different_ store: the SDD sidecar clone
  materialized inside the planner's own ephemeral workspace.
- The sibling failure branch eleven lines below already gets this right: a failed prompt-archive commit only logs "the
  host-owned epic launch continues independently" and returns `epic_approved` (`run_agent_exec_plan_accept.py:506-512`).

A second-order effect follows from the wrong outcome: only `epic_approved` defers the planner's completion notification
(`run_agent_runner_finalize.py:370` → `defer_epic_completion`). With `epic_launch_failed` the planner sends an immediate
failure notification, and the real launch's `finish_epic_launch` later finds no pending deferral, so it writes a
`.settled.json` marker nobody folds in and sends a separate success notification. The user gets two contradictory
notifications.

### Defect B — a `sase dev update` swapped the source under a live runner

The error string itself was never a store problem. `~/.local/bin/sase` resolves to a uv tool venv whose
`_editable_impl_sase.pth` points at `/home/bryan/projects/github/sase-org/sase/src`, so every long-lived SASE process —
the axe daemon, its agent runners, ACE — executes the primary checkout's source directly. The timeline:

| time     | event                                                                                                                                                                                                                                                           |
| -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 16:17    | `6b3b46e85` adds `issue_id_for_counter` to `sase/bead/ids.py` and a module-level import of it in `sase/bead/conflict_resolver.py`                                                                                                                               |
| 16:58:37 | runner for `tg` starts; importing `sase.sdd.store` pulls `sase.bead.ids` into `sys.modules` from the _pre-`6b3b46e85`_ source (verified by import probe)                                                                                                        |
| 17:52:50 | primary checkout fast-forwards to `e4fce05b6` (`git merge --ff-only origin/master`), journalled by `sase dev update` at 17:57:43 in `~/.sase/logs/dev_update.jsonl`                                                                                             |
| 18:13:02 | approval resumes the runner; the accepted-plan path performs a _lazy_ import of `sase.bead.conflict_resolver`, which reads the new on-disk source and resolves `from sase.bead.ids import issue_id_for_counter` against the stale cached module → `ImportError` |

`_sidecar_init.py:133` / `:430` (`raise SddMaterializationError(str(exc) or type(exc).__name__) from exc`) wrap that
`ImportError` verbatim, which is why the recorded error is a bare import message with no store context.

`src/sase/dev_update/code_swap_lock.py` exists precisely to prevent this, and its docstring already concedes the
residual race. But only `sase bead work` registers as a reader (`src/sase/bead/cli_work_entry.py:25-27`); a multi-hour
agent runner is invisible to the writer lock, and making it a blocking reader is not an option — with agents running all
day `sase dev update` would never acquire the lock again.

## Host-owned launches own their own outcome

Change `handle_accepted_plan` in `src/sase/axe/run_agent_exec_plan_accept.py` so the planner only reports a launch
outcome it actually owns.

1. Add a small predicate for host ownership — `plan_result.action == "epic"` and
   `plan_result.epic_launch_owner == "host"`. `PlanApprovalResult` already carries the field; no plumbing is required.
2. In both store-unusable branches (the materialization branch at lines 430-441 and the commit branch at 481-492), split
   on that predicate:
   - **Host-owned:** record the detail under a new `sdd_publication_error` metadata key, log a warning worded like the
     existing sibling branch ("the host-owned epic launch continues independently"), do **not** call
     `_notify_epic_launch_failure`, and fall through to the existing `return "epic_approved"`. Falling through is safe:
     every block between those branches and the epic return is `not is_epic`-gated, so `sdd_store=None` /
     `sdd_plan_path=None` is never dereferenced. Confirm that claim while implementing rather than assuming it.
   - **Not host-owned:** keep today's behavior byte for byte — `epic_launch_error` metadata,
     `_notify_epic_launch_failure`, `epic_launch_failed`. That outcome and its resume command stay correct when nobody
     has claimed the launch.
3. Surface the degradation instead of swallowing it. In `run_agent_runner_finalize.py`'s notes builder (around line
   280), append a note when the agent's `agent_meta.json` carries `sdd_publication_error` — the planner's
   prompt-archive/spec publication failed and the epic launch is unaffected. The finalizer already has
   `current_artifacts_dir`; `read_agent_meta` (`src/sase/axe/runner_artifacts.py:310`) only returns model fields today,
   so either widen it deliberately or read the key directly. The note must ride the same payload that
   `defer_epic_completion` hands to the launch, so the user gets one notification that says both "epic launched" and
   "the planner's archive entry is missing".
4. Do not weaken any assertion to make this pass, and do not silence the underlying failure: the planner's prompt
   archive really was not written, and that must remain visible.

Tests (`tests/test_axe_run_agent_exec_plan_followup_approvals.py`): replace
`test_unusable_epic_store_stops_before_launcher_with_home_resume` with a host-owned case (`epic_launch_owner="host"` →
`epic_approved`, `_notify_epic_launch_failure` not called, `sdd_publication_error` recorded, `write_sdd_spec` still not
called) and an unowned case (`epic_launch_owner=None` → the current `epic_launch_failed` behavior, including the
`notify_failure.assert_called_once_with(...)` arguments). Add a finalizer test that the degraded note reaches the
completion payload and that `epic_approved` still defers through `defer_epic_completion`.

## Agent runners survive mid-run editable source swaps

Add a source-skew module (for example `src/sase/axe/source_skew.py`) and wire it into agent-runner startup. Two halves,
prevention and honesty:

**Prevention.** `preload_post_gate_modules()` imports, once per runner before the agent CLI blocks on its first turn,
the module surface the post-gate path reaches lazily: walk `sase.sdd` and `sase.bead`, plus the named modules the
accepted-plan and commit paths defer (`sase.bead.epic_launch`, `sase.bead.epic_launch_handoff`,
`sase.agents_sync.prompt_archive`, `sase.notifications.senders`, `sase.vcs_provider.plugins._git_commit_dispatch`, and
the `sase.workspace_provider` plugin entry points). Walking whole packages beats an allowlist here: there is no list to
keep in sync, and the failing import (`sase.sdd._bead_manifest_repair` / `sase.sdd._repository_integration` →
`sase.bead.conflict_resolver`) is covered by construction. Requirements: best-effort per module so one broken import
never fails a launch, an `SASE_DISABLE_IMPORT_PRELOAD=1` kill switch, a debug log of elapsed milliseconds, and a
measured budget — report the real cost in the phase bead and keep it under ~1.5s, since it is paid once per runner while
the agent CLI is booting anyway.

**Honesty.** Snapshot the git revision of the checkout `sase` was imported from at process start
(`Path(sase.__file__).parents[2]`, `None` when it is not a git checkout), and expose `source_revision_changed()` plus a
human description of the swap. Where `handle_accepted_plan` records a store failure, check whether the exception chain
contains an `ImportError`/`AttributeError` _and_ the source revision moved; when both hold, record and log it as a
mid-run code swap naming both revisions instead of "Approved epic SDD store is unusable". Combined with the
`attribution` phase, that failure then degrades rather than killing the agent, and the next occurrence explains itself.

Preloading cannot cover everything — plugin packages (`sase-github`, `sase-telegram`) are separate editable checkouts
that `sase dev update` swaps too, which is exactly why the classification half exists. Pinning or re-execing long-lived
processes stays out of scope, as `code_swap_lock.py`'s docstring already states.

Tests: the preload leaves each named module in `sys.modules`; a deliberately broken module in the walk does not raise;
revision snapshot/compare works against a temporary git repo and degrades to `None` outside one; the classifier adds the
swap explanation only when both the import-error cause and the revision change are present.

## Dev updates name the live runners they may tear

Extend `src/sase/dev_update/code_swap_lock.py` with an advisory, non-blocking reader registration: a context manager
that writes a holder record marked non-blocking and never takes the `flock`. Agent runners register for their lifetime
at the same startup site the `skew_guard` phase adds.

The blocking semantics must not change. `code_swap_readers_active()` and the writer lock's `blocked_by` text must
consider only blocking holders, so neither `sase update` nor the ACE update preview
(`src/sase/ace/tui/modals/plugins_browser_dev_update.py:178`) can ever be deferred by a running agent — that would
starve updates on a host that always has agents running. Add a separate accessor for advisory holders and use it to
print a warning naming them: N agent runner(s) are running from this checkout and a swap now can break their deferred
imports. Dead-pid holder files are already pruned by `_live_reader_holders()`; keep that behavior for advisory records.

Tests: advisory holders never block `code_swap_writer_lock()`; `code_swap_readers_active()` ignores them; the warning
text lists them; a `sase bead work` reader still blocks and still explains itself.

## Non-goals

- Re-execing or source-pinning long-lived processes (ACE, the axe daemon, runners).
- Letting any agent defer a `sase dev update` source swap.
- Reworking the epic completion handoff protocol in `epic_launch_handoff.py`. If the orphaned `.settled.json` marker
  path (written when a planner never deferred) needs cleanup beyond what the `attribution` phase makes unreachable,
  record it as `PROPOSED FOLLOW-UP:` on the phase bead.
- The `parallel_suite_contention_reliability` epic (`sase-fd`) that this incident happened to be approving. It is
  unrelated work and must not be touched.

## Constraints

- No phase may edit SASE memory files (`sase/memory/*.md`, `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`,
  `QWEN.md`). This plan carries no permission for that; put durable notes in `CONTRIBUTING.md` if any are needed.
- No assertion may be weakened, deleted, or `xfail`-ed to make a node pass.
- Phase agents record discovered work as `PROPOSED FOLLOW-UP:` notes on their own bead rather than filing beads.

## Verification

Each phase runs `just install` first (workspaces are ephemeral and dependencies drift), then its targeted tests, then
`just check` before handing off:

```bash
just install
just test -- tests/test_axe_run_agent_exec_plan_followup_approvals.py   # attribution
just test -- tests/dev_update/                                          # swap_visibility
just check
```

The `attribution` phase additionally re-reads the incident record — the false `FAILED` for a host-owned launch that
succeeded — and states in its commit message which of the two branches now returns `epic_approved` and which still
returns `epic_launch_failed`.
