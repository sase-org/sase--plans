---
create_time: 2026-07-13 09:28:41
status: wip
prompt: 202607/prompts/wait_runners_deferred_workspace.md
tier: tale
---
# Plan: Complete `%wait(runners=)` support for deferred-workspace launches

## Problem

Agent `7s` (prompt: `#gh:gh_sase-org__sase Describe this repo. %w(runners=0)`) failed after 3s with:

```
RuntimeError: SASE_AGENT_DEFERRED_WORKSPACE=1 but extracted wait metadata is empty;
refusing to continue in the placeholder workspace
```

Any **non-home** launch whose `%wait` directive contains _only_ the `runners=` keyword (no named dependencies, no
`time=`, no absolute time) crashes the same way. This includes the drain barriers recently added to bundled workflows
(`xprompts/audit_recent_bugs.yml`, `xprompts/refresh_docs.yml`, `xprompts/toobig_split.yml` all emit `%w(runners=0)`),
so the blast radius is every detached workflow launch into a project workspace, not just one manual prompt.

## Root cause

The `runners=` keyword (epic `sase-5u`, July 2026) was wired into directive parsing, `agent_meta`, the runner-slot gate,
agent-scan wire fields, and the TUI — but never into the runner's deferred-workspace lifecycle, which predates it
(fail-fast guard added May 2026 in `6dac49cb6`):

1. **Launch side** classifies the prompt with `has_deferred_start_directive()` (`src/sase/xprompt/_directive_scan.py`),
   a cheap regex that matches _any_ `%wait`/`%w` form — including `%w(runners=0)`. The launch therefore holds
   placeholder workspace #0 and sets `SASE_AGENT_DEFERRED_WORKSPACE=1`.
2. **Runner side** (`src/sase/axe/run_agent_runner.py`) computes `has_wait` from only `wait_names` /
   `wait_identity_deps` / `wait_duration` / `wait_until` — `info.wait_runners` is missing. For a runners-only wait
   `has_wait` is `False`, so the safety guard ("deferred workspace but no wait metadata") raises and the run dies before
   doing anything, leaving the placeholder workspace held.
3. Even if the guard were passed, the deferred flow has no path for runners-only waits: `wait_for_dependencies()`
   (`src/sase/axe/run_agent_wait.py`) handles names/duration/until only, and the deferred workspace claim currently
   happens _before_ the runner-slot gate (`wait_for_runner_slot()`), which is where a runners-only wait actually parks.

Home-mode launches work today because the guard and the claim block are home-exempt while the runner-slot gate already
receives `wait_runners` — which is why the gap survived the `sase-5u` verification. The docs (`docs/xprompt.md`
"runners=" section, `docs/ace.md`) document runners-only waits as fully supported.

## Design

Fix the **runner side**; leave the launch-side classification as-is (runners-only prompts _should_ defer their
workspace). Rationale:

- A drain barrier can park for hours. Deferring means a parked agent holds only the shared placeholder — not one of the
  bounded axe workspaces — and its checkout is prepared _after_ the drain completes, so it sees the drained/final repo
  state (the entire point of `%w(runners=0)` in the refresh-docs/audit workflows).
- Narrowing the cheap launch scan to parse `%wait` args would reintroduce the classify-vs-extract mismatch class of bugs
  that `6dac49cb6` was written to close.

### Runner lifecycle changes (`src/sase/axe/run_agent_runner.py`)

1. Split the wait predicate:
   - `has_dependency_wait` = names / identity deps / duration / until (the current condition).
   - `has_wait` = `has_dependency_wait or info.wait_runners is not None`. The placeholder-workspace guard keeps using
     `has_wait`, so runners-only launches pass it while a genuinely-empty wait extraction still fails fast.
2. Gate `wait_for_dependencies()` and `detect_repeat_stop()` on `has_dependency_wait` (unchanged behavior for named/time
   waits; skipped for runners-only waits, which have no upstream producer and would otherwise hit the duration-only
   `assert` in `wait_for_dependencies`).
3. **Move the runner-slot gate before the deferred workspace claim.** New order inside the execution branch: dependency
   wait (if any) → repeat-stop handling → `wait_for_runner_slot(...)` → deferred claim + workspace prep + linked-repo
   refresh (if `SASE_AGENT_DEFERRED_WORKSPACE` and not home mode) → prompt ref resolution / sdd base sha /
   `AgentExecContext` build → exec loop. This is the piece that makes runners-only deferral work (the agent must not
   claim a real workspace until the drain gate opens) and it makes the ordering uniform for all deferred waits, matching
   the plan-of-record semantics ("deps → time floor → slot gate" as the _final_ pre-run stage, workspace materialized
   after all waiting). `wait_for_runner_slot` needs nothing from the claim block (it only touches the artifacts dir and
   global scan state), and everything workspace-dependent (`capture_sdd_base_sha`, the exec context) stays after the
   claim.

No changes to `wait_for_runner_slot` / runner-slot admission itself — it already accepts `wait_runners`, writes the
`waiting.json` slot marker (so the TUI `▶n→N` rendering and wait-modal editing keep working), and honors FIFO ordering.

### Intentional behavior deltas (call out in the commit message)

- Named-dependency deferred agents now claim their real workspace _after_ the slot gate instead of before it: agents
  parked at the gate no longer pin a real workspace, and the checkout is fresher when they start. Previously the gate
  rarely parked such agents, so the practical delta is small.
- For deferred agents, `run_started_at` (claimed inside the gate) now precedes workspace prep, so displayed runtime
  includes prep time (a few seconds).
- Non-deferred launches are unaffected: the gate only moves ahead of pure-local computations.

### Out of scope

- **No Rust core changes**: `prepare_agent_launch` just forwards `deferred_workspace`, and the agent-scan wire already
  carries `wait_runners`/`wait_runners_explicit`/`slot_requested_at` (`sase-5u.3`). This is Python runner-lifecycle
  behavior, not shared domain logic.
- **No memory-file edits** (`memory/xprompts.md` doesn't mention `runners=`; updating canonical memory requires explicit
  user permission and is not part of this change).

## Tests

Extend `tests/test_axe_run_agent_runner_deferred_workspace.py` (using the existing
`run_main`/`base_patches`/`AGENT_INFO` helpers from `tests/_axe_run_agent_runner_retry_helpers.py`):

1. **Runners-only deferred launch succeeds** — `SASE_AGENT_DEFERRED_WORKSPACE=1` +
   `AGENT_INFO._replace(wait_runners=0)`: no error report; `wait_for_dependencies` **not** called;
   `wait_for_runner_slot` receives `wait_runners=0`; `claim_deferred_workspace` is called and the exec loop runs against
   the claimed workspace dir.
2. **Gate-before-claim ordering** — record call order across the mocked gate and `claim_deferred_workspace`; assert the
   gate resolves first (covers named-dep deferred waits too, since the ordering is now uniform).
3. **Combined wait** — `wait_names=["x"]` + `wait_runners=1`: `wait_for_dependencies` still runs, then the gate gets
   threshold 1, then the claim.
4. **Guard regression** — the existing `test_deferred_workspace_without_extracted_wait_fails_before_run_loop` (all wait
   fields empty, `wait_runners=None`) must keep failing fast.
5. Audit `tests/test_run_agent_runner_slots.py` / `tests/test_run_agent_runner_wait_queue.py` and the repeat-stop
   deferred test for assumptions pinned to the old gate position; update assertions where they encode claim-before-gate
   ordering rather than user-visible behavior.

## Verification

- Targeted pytest subsets for the four test modules above (full `just test` is unreliable in the agent sandbox; rely on
  targeted runs plus the static gates).
- `just check` static gates (lint, mypy, symvision) before finishing.
- Manual/live check after review: launch a trivial non-home prompt with `%w(runners=0)` while another agent runs,
  confirm it parks as `WAITING ▶n→0` without claiming a workspace, starts after the drain, and completes; confirm
  home-mode runners-only behavior is unchanged.
