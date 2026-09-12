---
tier: epic
title: Harden every GitHub network interaction against slow and degraded transports
goal: "A slow, congested, or degraded GitHub transport degrades SASE gracefully instead
  of failing an agent run: redundant bulk transfers are eliminated, every network git
  and `gh` call retries transient failures under a shared deterministic classifier,
  wall-clock timeouts are replaced by stall-aware deadlines, and sustained degradation
  is visible in telemetry before it becomes an agent failure.

  "
phases:
  - id: ref-reuse
    title: Borrow local objects when materializing sidecar SDD clones
    depends_on: []
    size: small
    description: "ref-reuse: pass a validated `reference_repo` down the sidecar-kind
      clone path so workspace materialization of the plans store stops re-downloading
      the full pack from GitHub on every launch.

      "
  - id: clone-retry
    title: Make remote clone timeouts retryable instead of fatal
    depends_on: []
    size: small
    description: "clone-retry: close the hole where `SddGitCommandTimeout` bypasses the
      clone retry loop whenever no reference repo is in play, and give retries an
      escalating, deadline-aware timeout budget.

      "
  - id: classifier
    title: Deterministic retryability classifier in the Rust core
    depends_on: []
    size: medium
    description: "classifier: port transient-failure detection for git and `gh` output
      into sase_core as a pure deterministic classifier with a PyO3 binding, replacing
      the ad-hoc substring tuple that only covers git clone stderr.

      "
  - id: stall-deadline
    title: Replace fixed wall-clock timeouts with stall-aware deadlines
    depends_on:
      - clone-retry
    size: medium
    description: "stall-deadline: stream git progress output so a slow-but-advancing
      transfer is distinguished from a genuinely stalled one, bounding idle time rather
      than total duration.

      "
  - id: gh-runner
    title: Single retrying chokepoint for gh CLI calls
    depends_on:
      - classifier
    size: medium
    description: "gh-runner: add one bounded, non-interactive `gh` execution boundary
      that applies the shared classifier, honors rate-limit and Retry-After signals, and
      reports structured failures.

      "
  - id: adoption
    title: Migrate ad-hoc GitHub call sites onto the shared runners
    depends_on:
      - classifier
      - gh-runner
    size: medium
    description: "adoption: move the scattered direct `gh` subprocess calls and the
      remaining network git paths in agents sync and bead sync onto the shared retrying
      runners so resilience is uniform rather than per-call-site.

      "
  - id: materialization
    title: Bound clone concurrency and stop stranding workspaces on transient failure
    depends_on:
      - clone-retry
    size: medium
    description: "materialization: cap concurrent remote clones so parallel launches
      stop saturating the uplink, and let the agent runner degrade or requeue instead of
      hard-failing and holding a numbered workspace.

      "
  - id: observability
    title: Surface transport degradation before it fails a run
    depends_on:
      - classifier
      - stall-deadline
    size: small
    description:
      "observability: record near-miss margin and retry outcomes on network git
      operations and add a doctor check that warns when durations trend toward the
      configured ceiling."
proposed_by: bbugyi200.athena.0k6
create_time: 2026-09-12 09:44:47
status: wip
---

- **PROMPT:**
  [prompts/202609/github_network_resilience.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/github_network_resilience.md)

# Plan: Harden every GitHub network interaction against slow and degraded transports

## Why: what actually failed

The triggering failure was agent `bbugyi200.kellys_mbp.05.f1.f0.f0` (workflow
`ace(run)-260912_090457`, 2026-09-12 09:09:46 EDT) on the MacBook:

```
SddMaterializationError: timed out cloning SDD store
  git@github.com:sase-org/sase--plans.git into <workspace>/sase/repos/plans
subprocess.TimeoutExpired: Command '['git', ..., 'clone',
  'git@github.com:sase-org/sase--plans.git', ...]' timed out after 120.0 seconds
```

The working hypothesis was a network or GitHub outage. The evidence only partly supports
that, and the dominant causes are in this repository.

**GitHub was not in an incident.** The GitHub status API
(`https://www.githubstatus.com/api/v2/incidents.json`) reports no incident on either
2026-09-11 or 2026-09-12; the most recent entry before the failures is 2026-09-04.

**The transport was capable of the work.** A cold, no-reference clone of the same repo,
from the same machine, over the same SSH path, run during this investigation, completed
in **35 s** and transferred 82 MB. The failing clone was allowed 120 s.

**The margin had already been consumed.** From the machine's own
`~/.sase/logs/tui_git_ops.jsonl`, every `sdd.clone.remote` record:

| Time (EDT)     | Result  | Duration | Reference |
| -------------- | ------- | -------- | --------- |
| 09-08 13:29:52 | ok      | 1.5 s    | yes       |
| 09-08 13:30:08 | ok      | 7.6 s    | yes       |
| 09-11 11:55:08 | ok      | 7.6 s    | no        |
| 09-11 12:02:36 | ok      | 7.3 s    | no        |
| 09-11 12:20:17 | ok      | 19.2 s   | yes       |
| 09-11 12:26:11 | timeout | 120.0 s  | no        |
| 09-11 12:45:07 | ok      | 67.5 s   | no        |
| 09-12 06:30:31 | ok      | 69.9 s   | no        |
| 09-12 07:20:24 | ok      | 84.2 s   | no        |
| 09-12 07:27:09 | ok      | 115.8 s  | no        |
| 09-12 09:00:15 | ok      | 114.8 s  | no        |
| 09-12 09:01:48 | ok      | 14.8 s   | yes       |
| 09-12 09:04:18 | ok      | 110.9 s  | no        |
| 09-12 09:09:46 | timeout | 120.0 s  | no        |

Two facts dominate this table:

1. **Reference-backed clones are an order of magnitude cheaper.** At 09:00:15 a
   no-reference clone took 114.8 s; 93 seconds later, on the same machine and the same
   network, a reference-backed clone of the same store took 14.8 s. Whatever the network
   was doing, it was not what separated success from failure.
2. **No-reference clones had been riding the ceiling for hours.** They climbed from ~7 s
   to 110–116 s against a hard 120 s cap. The 09:09:46 timeout is not an anomaly; it is
   the trend crossing the line. The identical 09-11 12:26:11 timeout was the same
   failure 21 hours earlier.

**Why the expensive path is taken at all.** `ensure_sdd_kind_clone` in
`src/sase/sdd/_store_workspace.py` calls `ensure_sidecar_sdd_clone` **without**
`reference_repo`, so the plans sidecar — at ~90 MB of `.git`, across 7,135 commits, the
largest store SASE materializes — does a full cold download for every numbered
workspace. Every sibling path already does the opposite:

| Call site                                          | Passes `reference_repo`? |
| -------------------------------------------------- | ------------------------ |
| `sdd/_store_workspace.py` beads path (~line 192)   | yes                      |
| `_linked_repo_workspaces.py` (~line 279)           | yes                      |
| `sdd/_artifact_link_machine_store.py` (~line 235)  | yes                      |
| `sdd/_store_workspace.py` sidecar-kind path (~134) | **no**                   |

`clone_sdd_store` already accepts `reference_repo` and already implements
`--reference-if-able ... --dissociate`; the plans path simply never supplies one.

**Why one slow clone is fatal.** In `src/sase/sdd/_store_clone_ops.py`, the
`except SddGitCommandTimeout` arm retries only when `reference is not None` (to retry
_without_ the reference). When `reference` is already `None` — exactly the plans case —
it returns `handle_failed_sdd_clone(...)` immediately. The `_REMOTE_CLONE_RETRY_DELAYS`
backoff schedule is reachable only from the non-timeout branch, gated on
`_is_transient_remote_clone_failure(detail)` matching git's stderr. A timeout produces
no stderr to match, so **a timed-out clone gets zero retries**, fails the setup
transaction, fails the agent, and strands the numbered workspace.

**Self-inflicted congestion is a real contributor.** In the 09:00–09:09 window,
materializations for three different numbered workspaces overlapped, and cheap
`sdd.clone.fetch` operations inflated from a 0.3 s baseline to 12.2 s, 16.3 s and 33.9
s. That is a saturated uplink, driven largely by concurrent ~90 MB redundant clones.

**Verdict.** The network was moderately degraded (≈35 s baseline versus ≈115 s observed,
roughly 3×), but GitHub was not in an incident and degradation alone was not sufficient.
The failure required SASE to be downloading ~90 MB it already had on disk, with a fixed
120 s wall-clock cap it had already been brushing for hours, and no retry on the one
failure mode that cap produces. This plan fixes the code first and then makes the whole
GitHub surface tolerant of the degradation that will recur.

## Scope note: the Rust core boundary

`sase_core`'s existing git surface (`crates/sase_core/src/git_query/`) is deliberately
pure: deterministic parsers over git output, no subprocess execution and no policy. This
plan keeps that split. The **deterministic classification** of a git or `gh` failure
into a retryability verdict is core backend behavior — a CLI, the TUI, and any future
frontend must agree on whether a failure is retryable — and belongs in `sase_core`
alongside `git_query`. Subprocess execution, backoff sleeping, deadline arithmetic and
call-site wiring stay in Python. Only the `classifier` phase crosses into the sibling
core repo.

## Borrow local objects when materializing sidecar SDD clones

Give the sidecar-kind clone path the same local-object reuse every other clone path
already has.

- In `ensure_sdd_kind_clone` (`src/sase/sdd/_store_workspace.py`), resolve the primary
  workspace's clone directory for the same kind and pass it as `reference_repo` to
  `ensure_sidecar_sdd_clone`, mirroring how the beads path derives `reference_dir` and
  guards against `reference_dir == clone_dir`.
- Rely on the existing `_matching_clone_reference` validation: it already requires a
  real `.git` directory and a matching remote, and `--dissociate` already prevents the
  new clone from depending on the reference surviving. Unpublished commits cannot leak
  in, because refs still come from the recorded remote.
- Handle the cold-start case explicitly: when no primary clone exists yet, the reference
  is `None` and behavior is unchanged.

Tests: extend `tests/sdd_store/` to assert that materializing a sidecar kind with an
existing primary clone invokes `git clone` with `--reference-if-able` and
`--dissociate`, that a missing or foreign-remote primary falls back to a plain clone,
and that a reference equal to the clone target is refused.

Expected effect, from the table above: the failing operation moves from the 110–116 s
band to the ~15 s band, restoring roughly 8× of headroom against any ceiling.

## Make remote clone timeouts retryable instead of fatal

Treat a timeout as the transient failure it usually is.

- In `clone_sdd_store`, let `SddGitCommandTimeout` fall through to the same bounded
  backoff schedule that non-timeout transient failures use, instead of returning
  immediately when `reference is None`. Preserve the existing "retry once without the
  reference" behavior as a distinct, still-bounded step.
- Escalate the per-attempt timeout across retries rather than reusing one fixed value: a
  clone that needed 130 s will never succeed in another 120 s. Clamp the escalation to
  the caller's `deadline` when one is supplied, and keep returning the existing
  "deadline expired" diagnostic when no budget remains.
- Always clear the partial clone between attempts via the existing
  `_remove_partial_sdd_clone`, so a retry never inherits a half-written tree.
- Keep the total work bounded: the sum of attempt budgets must remain predictable, and
  `strict=True` callers must still fail with a `SddMaterializationError` once the
  schedule is exhausted.

Tests: extend `tests/sdd_store/test_sidecar_clone_retry.py` with a timeout-then-success
case proving retries happen with no reference in play, an escalating-budget assertion,
and a deadline-exhaustion case proving the loop still terminates.

## Deterministic retryability classifier in the Rust core

Replace `_TRANSIENT_REMOTE_CLONE_ERRORS` — a 12-entry substring tuple in
`_store_clone_ops.py`, reachable only from clone — with one shared classifier.

- In the sibling core repo (`crates/sase_core`), add a pure module beside `git_query`
  that maps a failure observation (stderr/stdout text, exit status, and an operation
  kind) to a verdict: retryable-transient, retryable-after-delay, or permanent. Keep it
  free of PyO3 types, matching the `git_query` convention, and cover at minimum:
  transport faults already listed today, plus HTTP 5xx, `TLS`/`SSL` handshake failures,
  proxy errors, GitHub primary and secondary rate limiting, and `gh` authentication and
  not-found cases that must **not** be retried.
- Add the PyO3 binding in `crates/sase_core_py`, the wire types, and Rust unit tests
  plus a Python parity test in the same style as the existing `git_query` parity tests.
- Add the thin Python adapter under `src/sase/core/` and route
  `_is_transient_remote_clone_failure` through it, keeping the existing public Python
  symbol as the call-site-facing name.
- Follow the required sequence for a boundary-crossing change: land the Rust wire, API,
  bindings and tests in the core repo first, then update the Python adapter and callers
  here, and bump `sase-core-revision.txt` as that workflow requires.

Explicitly out of scope: moving subprocess execution, sleeping, or deadline arithmetic
into Rust.

## Replace fixed wall-clock timeouts with stall-aware deadlines

A 120 s cap cannot distinguish "transferring steadily, needs 130 s" from "dead socket at
2 s". Bound idle time instead of total time.

- Add a streaming execution mode to the `run_sdd_git` boundary in `src/sase/sdd/_git.py`
  for long transfers: pass `--progress`, read git's stderr incrementally, and refresh a
  last-progress timestamp on each update. Abort when no progress arrives within a stall
  window, or when a generous absolute ceiling is exceeded — whichever comes first.
- Keep the stall window and absolute ceiling configurable through the existing
  `_float_env` pattern in `_git.py`, alongside `SASE_SDD_GIT_NETWORK_TIMEOUT`, and add
  the corresponding keys to `src/sase/default_config.yml` if the knobs are surfaced
  there.
- Apply the streaming mode to clone, fetch and push; leave short metadata operations on
  the existing simple bounded path so the change stays low-risk.
- Report the distinction in the raised error: a stall and a ceiling overrun must produce
  different messages, because they call for different operator responses.
- Preserve `SddGitCommandTimeout` as the raised type so existing `except` arms,
  including the retry logic from the `clone-retry` phase, keep working unchanged.

Tests: drive the streaming reader with a fake process emitting steady progress past the
old fixed timeout (must succeed), then one that goes silent (must abort at the stall
window, not the ceiling).

## Single retrying chokepoint for gh CLI calls

Network git has one boundary (`run_sdd_git`); `gh` has none. Roughly two dozen modules
shell out to `gh` independently — among them `external_repos.py`, `main/patch_sync.py`,
`repo_open_external.py`, `updates/incoming_commits.py`, `axe/run_agent_retry_spawn.py`,
`doctor/checks_external_pr_mirror.py` and several `xprompt/` completion helpers — each
with its own timeout and error handling, or none.

- Add one `gh` execution boundary in `src/sase/` that mirrors `run_sdd_git`: bounded
  timeout, non-interactive environment, captured output, structured result, and the same
  `log_tui_git_operation`-style telemetry so `gh` failures become as visible as git
  ones.
- Apply the `classifier` verdict to decide retries, with bounded backoff. Honor GitHub's
  rate-limit signals when present — `Retry-After` and `x-ratelimit-reset` — rather than
  retrying blindly into a secondary rate limit, and never retry authentication or
  not-found failures.
- Provide a JSON convenience wrapper for `gh api`, since most call sites parse JSON and
  today each re-implements that parsing and its failure mode.
- Keep the runner injectable, following the `GitRunner` Protocol precedent in
  `src/sase/agents_sync/git.py`, so tests stay deterministic and offline.

Tests: transient-then-success retry, rate-limit response honoring the advertised delay,
permanent failure retried zero times, and timeout behavior.

## Migrate ad-hoc GitHub call sites onto the shared runners

Resilience only counts where it is actually used.

- Migrate the direct `gh` subprocess call sites enumerated in the previous section onto
  the new boundary. Convert the existing `gh_api` in
  `src/sase/plugins/_github_source_gh.py` to delegate to it, preserving its paging
  constants and per-page budget semantics.
- Route the remaining network git paths through the retrying helpers:
  `agents_sync/git.py` (`run_git` with `network=True`), `agents_sync/git_sync_ops.py`,
  `bead/sync_worker.py` and `bead/_sync_git.py`, and `artifact_ref_sync.py`. Respect
  each caller's existing `deadline` parameter — these paths already thread one, and
  retries must consume it rather than ignore it.
- Interactive and TUI-adjacent call sites must not acquire new blocking retry loops that
  could stall the UI; where a call site is on a UI path, keep the retry budget small or
  leave it on the existing single-shot path and record that choice in the code.
- Leave behavior identical where a call site deliberately wants a fast, non-retrying
  probe, such as completion helpers; make that explicit rather than incidental.

Tests: per-call-site tests asserting the shared runner is used, plus an audit test that
fails when a new direct `gh` subprocess invocation appears outside the boundary module,
so this does not silently regress.

## Bound clone concurrency and stop stranding workspaces on transient failure

Two independent problems surfaced in the 09:00–09:09 window.

**Concurrency.** Materializations for three numbered workspaces overlapped and inflated
cheap fetches from 0.3 s to as much as 33.9 s. `materialization_lock` serializes per
primary store but does nothing across workspaces.

- Add a host-level bound on concurrent _remote_ clones, so parallel launches queue
  rather than compete for the uplink. Waiting a few seconds for a 15 s clone beats three
  simultaneous 115 s clones.
- Make the bound configurable and default it conservatively. Note that after the
  `ref-reuse` phase most materializations stop being bulk transfers at all, so this is a
  safety net for the genuine cold-start case.

**Stranded workspaces.** The failure notification recorded: _"Workspace #14 is held ...
Inspect or commit its changes, then dismiss this agent to release it."_ A transient
network failure during setup — before the agent ever ran and before any work existed to
preserve — cost a held numbered workspace requiring manual dismissal.

- Distinguish a setup-phase materialization failure from a failure after the agent has
  produced work. In `axe/run_agent_runner_launch.py` and
  `axe/run_agent_runner_setup.py`, a pre-agent transient failure should release the
  workspace automatically rather than hold it for inspection.
- Prefer requeuing the launch over failing it outright when the classifier reports a
  transient cause and the retry budget is exhausted, so a degraded window delays a run
  instead of losing it. Keep this bounded: a repeatedly failing launch must still
  terminate with a clear diagnostic rather than loop.
- Keep the operator-facing error text specific about which store failed, how long it was
  given, and whether it was retried.

Tests: a pre-agent transient materialization failure releases the workspace; a
post-agent-work failure still holds it; a permanent failure (bad remote, auth) fails
fast with no requeue.

## Surface transport degradation before it fails a run

This failure was fully predictable from data SASE already collected. `sdd.clone.remote`
durations climbed from ~7 s to 116 s against a 120 s cap over about 24 hours, and
nothing reported that until a run died.

- Extend the `log_tui_git_operation` record in `src/sase/sdd/_git.py` with the fields
  needed to spot this: the ratio of duration to the applicable limit, the retry attempt
  index and total attempts, the classifier verdict, and whether a reference repo was
  used. The `cmd` array is already logged, but "was this clone reference-backed?" should
  not require string-matching argv after the fact.
- Add a `sase doctor` check that reads the existing git-operations JSONL and warns when
  recent network operations trend toward their ceiling — for example, when a meaningful
  share of recent samples exceed a high fraction of the limit — and names the store and
  the observed durations. Follow the existing check conventions in `src/sase/doctor/`.
- Make the check cheap and offline: it reads local telemetry only and must not perform
  network calls of its own.

Tests: synthesized JSONL fixtures for a healthy profile (no warning), a
degrading-toward-ceiling profile (warning naming the store), and an empty or absent log
(no crash, no warning).

## Verification

Every phase must pass `just check` before landing, per this project's two-speed
verification convention; `just check-full` gates the epic's landing. The `classifier`
phase additionally runs the sibling core repo's own check script for the Rust side.

Beyond the automated suites, the epic is successful when a cold materialization of the
plans store on a laptop-class connection is reference-backed and completes in seconds
rather than approaching two minutes, a deliberately stalled transport produces a bounded
retry sequence and a stall-specific diagnostic instead of an agent failure, and a
transient setup-phase failure leaves no numbered workspace held.
