---
tier: tale
title: Fix ci_watch fix-proposal starvation with a hood-scoped gate and lane wait_runners
goal:
  The ci_watch chop proposes debounced CI-fix agents whenever a mature red repo has no live ci_fix hood agent, and
  launched fix agents defer to interactive work via wait_runners instead of a never-satisfiable zero-agents gate.
create_time: 2026-07-28 18:03:46
status: done
---

- **PROMPT:** [202607/prompts/ci_watch_fix_gate.md](prompts/ci_watch_fix_gate.md)
- **AGENTS:**
  - bbugyi200.athena.nn--code
- **COMMITS:**
  - [b161e1c](https://github.com/bbugyi200/dotfiles/commit/b161e1c12798e4e1580460b148b2cdd119c918ea) — fix(sase):
    idle-gate ci_watch fix launches

# Fix ci_watch Fix-Proposal Starvation (Hood-Scoped Gate + Lane wait_runners)

## Problem

The `ci_watch` AXE lumberjack and its single `ci_watch` chop (script `bugyi_chop_ci_watch`, installed at v0.3.1,
matching `gh:bbugyi200/bugyi-chops` HEAD) are mechanically healthy: the lumberjack ticks every 5 minutes, the chop has
329 recorded runs over ~27 hours with zero errors, actstat sweeps and repo classification work, and the merge half has
already merged 2 release PRs since `merge_enabled: true` was set.

However, the fix half has NEVER proposed a CI repair. Evidence from the decision ledgers under
`~/.sase/axe/lumberjacks/ci_watch/chops/ci_watch/runs/`:

- `sase-org/sase` has been red with a stable failing-job fingerprint (`bead-backend`, `lint`,
  `published-core-minimum-smoke`) and a mature debounce streak (currently 6).
- Across all 329 ticks, the final reason for `sase-org/sase` was: `agents_busy` × 230, `superseded_or_unsettled` × 72,
  `red_debounce` × 27 — and `fix_proposed` × 0.

Root cause: `build_ci_watch_result()` in `src/bugyi_chops/ci_watch.py` gates fix proposals on `AgentsGate.probe()`
returning ZERO agents from `sase agent list -j`. That command reports every live agent system-wide, including WAITING
agents (queued chop proposals from other lumberjacks, parked `.land`/wait lane members, idle-gated audit agents). On
athena this count was 17–19 at every probed tick, while AXE's own runner slots showed `agents 0/3` in use — the machine
was idle by the scheduler's definition, yet the chop considered it busy. The zero-agents condition is effectively
unsatisfiable, so the fix half is permanently starved.

The correct SASE mechanism for "don't compete with interactive work" already exists at the lane level: `wait_runners: N`
("start a lane agent only once at most this many other agents hold a runner slot") — used by the `code_quality` lane
(`wait_runners: 0`). Proposal-time suppression should only prevent duplicate/overlapping FIX work, not machine-wide
contention.

## Solution Overview

1. In `bugyi-chops`: narrow the proposal-time gate from "zero agents anywhere" to "no live agent in the `ci_fix` hood"
   (agent named `ci_fix` or prefixed `ci_fix.`). Duplicate-proposal protection is already provided by the chop
   `dedupe_key` once-per store (keys are released when a launched agent fails, and a successful fix changes the
   fingerprint), so this gate only needs to prevent overlapping fix agents.
2. In `chezmoi`: add `wait_runners: 0` to the `ci_watch` lumberjack lane so proposed fix agents queue until no other
   agent holds a runner slot, and refresh the stale lane/chop descriptions.

We deliberately do NOT add a config-level `inhibit_if: agent_hood: {hood: ci_fix}` guard on the chop: that would
suppress the entire chop tick (including merge gating, ledger evidence, and streak bookkeeping) while a fix agent is
queued or running, which is undesirable.

Both repos MUST be opened with the `/sase_repo` skill (`sase repo open <name> -r "<reason>"`); use the printed paths for
all reads and writes.

## Changes

### 1. bugyi-chops (`gh:bbugyi200/bugyi-chops`)

`src/bugyi_chops/ci_watch.py`:

- Replace the mature-red gating logic that suppresses all fixes when `probe.count > 0` with a hood-scoped check:
  suppress only when the probe reports a live agent whose name equals `ci_fix` or starts with `ci_fix.`. Keep
  `AgentsGate.probe()` itself generic (returning all names) and filter in `build_ci_watch_result()`.
- Ledger/counter updates:
  - New suppression reason `fix_in_flight` (replacing `agents_busy` in this path); record the in-flight `ci_fix` hood
    agent names in the ledger row (bounded by `MAX_LEDGER_NAMES`, reusing the existing `busy_agents`-style field or a
    clearer `in_flight_agents` name — implementer's choice, but keep it bounded and redacted via `_bounded`).
  - Keep `counters["agents_running"]` as the total live-agent count for observability. Optionally add a
    `fix_agents_running` counter for the hood count; if added, include it in `_new_counters()` and the summary.
  - The probe-failure path must keep failing closed with reason `agents_check_failed`.
- Update the module docstring / `main()` chop description text that promises idle-gating so it now describes hood-scoped
  in-flight gating (machine-idleness is handled at launch time by the lane's `wait_runners`).
- Bump `version` in `pyproject.toml` to `0.4.0` (behavior change).

`tests/test_ci_watch.py`:

- Update existing `agents_busy` expectations to the new behavior.
- Add coverage for: (a) unrelated live agents (e.g. interactive lanes, `toobig-*`, WAITING audit agents) do NOT suppress
  a fix proposal; (b) a live `ci_fix.<slug>` agent DOES suppress with `fix_in_flight`; (c) probe failure still
  suppresses with `agents_check_failed`; (d) the bare name `ci_fix` and prefixed `ci_fix.` names both match, while
  lookalikes such as `ci_fixer` do not.
- Keep the repo's coverage gate (`fail_under = 90`) green; run the repo's own checks (`just` targets or
  `pytest`/`ruff`/`mypy` per its `justfile`) before committing.

### 2. chezmoi (linked repo)

`home/dot_config/sase/sase_athena.yml` — in `axe.lumberjacks.ci_watch`:

- Add `wait_runners: 0` at the lumberjack level (sibling of `interval: 300` / `chop_timeout: "120s"`), so launched
  `ci_fix.*` agents wait until zero other agents hold runner slots before starting.
- Update the lane `description`: replace "launches at most one debounced CI-fix agent per tick and only when zero other
  agents are running" with the new semantics (one debounced fix proposal per tick, suppressed only while a `ci_fix` hood
  agent is live; launched fix agents wait for an idle machine via `wait_runners: 0`). Also remove or rewrite the stale
  sentence "Merging stays disabled until the fix half has soaked" — `merge_enabled: true` is set and release merges have
  already landed.
- Update the chop `description` accordingly (it currently references the "zero-agents gate").
- No changes to `vars` (including `fix_enabled`/`merge_enabled`), `env`, or `actstat-ci-watch.yml`.

Per the chezmoi repo's instructions, after committing run `chezmoi update -a --force` to apply the change to the home
directory.

## Commit / Landing

- Commit each repo's changes with the `/sase_git_commit` skill (one commit per repo).
- bugyi-chops publishes releases via its own flow; the version bump commit is sufficient here.

## Deployment Notes (post-merge, for the user)

- The running system uses the `bugyi-chops` build installed in the sase uv tool environment
  (`~/.local/share/uv/tools/sase/`, currently 0.3.1). The new version must be installed there before the gate change
  takes effect.
- The `ci_watch` lumberjack reads its lane config at startup: restart AXE (or ACE, which restarts AXE at startup) after
  `chezmoi update -a --force` so `wait_runners: 0` and the new chop build are picked up.

## Verification

1. Unit tests in bugyi-chops cover the new gate (see above).
2. After deployment, `sase axe status` should still show the `ci_watch` lumberjack running, and within a few ticks the
   newest `*.result.json` under `~/.sase/axe/lumberjacks/ci_watch/chops/ci_watch/runs/` should show `fix_proposed: 1`
   (assuming `sase-org/sase` is still red and no `ci_fix.*` agent is live), with a queued `ci_fix.sase` agent visible in
   `sase agent list` that starts once runner slots free up.
3. While that fix agent is live, subsequent ledgers should show `fix_in_flight` (not `agents_busy`) for `sase-org/sase`.

## Out of Scope

- The actual CI failure on `sase-org/sase` (`bead-backend`, `lint`, `published-core-minimum-smoke`) — the un-starved
  chop will propose the repair agent itself.
- Any change to merge gating, debounce (`red_debounce_ticks`), actstat config, or the sase repo's axe machinery.
