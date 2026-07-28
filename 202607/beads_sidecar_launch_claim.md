---
tier: tale
title: Materialize the beads sidecar before the launch-time bead claim
goal: Agents launched with a bead claim succeed on projects whose bead state lives
  in a dedicated beads sidecar repo, because the runner materializes the workspace's
  beads clone before opening the bead store.
create_time: 2026-07-28 06:56:01
status: done
---

- **PROMPT:** [202607/prompts/beads_sidecar_launch_claim.md](prompts/beads_sidecar_launch_claim.md)
- **AGENTS:**
  - [bbugyi200.athena.mq](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.mq/README.md)
  - [bbugyi200.athena.mq--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.mq.md#member-code)
  - [bbugyi200.athena.mq--plan](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.mq.md#member-plan)
- **COMMITS:**
  - [86dd439](https://github.com/sase-org/sase/commit/86dd439409b08005fe24758419e7d669d4c808c7) — fix: materialize beads sidecar before launch claim

# Plan: Materialize the beads sidecar before the launch-time bead claim

## Problem

Every launch of the `sase-a8.land` epic lander fails immediately, before the model ever runs:

```
RuntimeError: Failed to claim bead 'sase-a8' for agent 'sase-a8.land':
No beads/ directory found at <workspace>/sase/repos. Run 'sase bead init' first.
```

Two consecutive attempts failed this way (`ace(run)-260728_063921` and `ace(run)-260728_064421`), each burning a
workspace claim and leaving the workspace held. The `sase-a8` epic bead is still `open`, so the lander cannot land the
epic until this is fixed.

The failure is not specific to that agent. It hits **any** agent launched with a bead directive (`%id(..., bead=...)`,
`#bd/land_epic:<epic>`, `#bd/work:<bead>`, JIT epic phase launches) on a project that has migrated to a dedicated beads
sidecar.

### Evidence

- `~/.sase/logs/tui_toasts.jsonl` — `Axe: CLAUDE(opus) @sase-a8.land failed: ace(run)-260728_063921` and `...064421`.
- Artifact `error_report.md` / `done.json` under
  `~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202607/28/20260728064421/` — full traceback,
  `"outcome": "failed"`, duration 55s.
- Runner logs `~/.sase/workflows/202607/gh_sase-org__sase_ace-run-260728_0639*.txt` and `...0644*.txt` — the failure
  lands right after `Claimed workspace #N` and before any workflow step output.
- The traceback bottoms out in `BeadProject.__init__` (`src/sase/bead/project.py:64`) via
  `open_bead_project_for_beads_dir` (`src/sase/bead/store_locator.py:28`) via `claim_bead_for_agent_launch`
  (`src/sase/axe/run_agent_runner_bead.py:31`).

## Root cause

This is a gap left by the `sase-a8` epic ("Split bead state into a dedicated beads sidecar repository", plan
`plans:202607/beads_sidecar_repo.md`). Under schema v3 the project's beads live in their own sidecar repo cloned to
`<workspace>/sase/repos/beads`, and that clone is deliberately **lazy** — see the epic's "Auto-clone gating" and "Lazy
beads sidecar materialization" sections, delivered as phase `sase-a8.7`.

Phase `sase-a8.7` wired lazy materialization into exactly one funnel: `resolve_beads_location(materialize=True)` in
`src/sase/bead/cli_common.py:107-115`, which is what `sase bead` uses. The axe launch-time claim does **not** go through
that funnel. `claim_bead_for_agent_launch` (`src/sase/axe/run_agent_runner_bead.py:22-31`) resolves the path directly
and opens the store:

```python
store = resolve_sdd_store(workspace_dir, workspace_num)
beads_dir = store.kind_root("beads")          # <workspace>/sase/repos/beads — may not exist yet
with bead_store_write_lock(beads_dir) as already_locked:
    with open_bead_project_for_beads_dir(beads_dir) as project:   # FileNotFoundError here
```

Nothing materializes that clone before this point:

- `prepare_launch_workspace_repos` (`src/sase/axe/runner_workspace.py:253-278`) calls `ensure_workspace_sdd_clone`,
  which for sidecar storage syncs the **plans** sidecar only — see `src/sase/sdd/_store_workspace.py:46-56` and the test
  `test_ensure_workspace_sdd_clone_syncs_plans_sidecar_only` in `tests/sdd_store/test_workspace_clone.py`.
- The `beads` sidecar _is_ reported as an `auto_clone: true` linked repo once the store record names it, but for
  VCS-provider projects the linked-repo/sidecar materialization runs inside the provider's workflow steps (`gh__setup` /
  `gh__checkout` / `gh__prepare`), which execute inside `run_execution_loop`.
- `_promote_bead_claim` runs **before** `run_execution_loop` — see `src/sase/axe/run_agent_runner_launch.py:180-212`.

Ordering proof from the run artifacts: a comparable successful run in the same project claimed its workspace at
`06:46:5x`, while the beads clone's git reflog records `clone: from github.com:sase-org/sase--beads.git` at `06:48:38`,
i.e. ~100s later, during the workflow steps. A bead-claiming agent never gets that far.

Before `sase-a8`, `kind_root("beads")` for a schema-2 store resolved _inside_ the plans sidecar, which workspace prep
does clone eagerly — so this path worked. Phase `sase-a8.10` migrated the enabled projects to schema v3, which is what
exposed the gap.

Two aggravating details worth fixing at the same time:

1. `bead_store_write_lock` (`src/sase/bead/_sync_git.py:29-31`) **silently yields `False`** when `beads_dir` does not
   exist. So the current code does not just fail — it fails while holding no store lock at all. Materialization must
   therefore happen _before_ the lock is taken, not inside it: the lock is derived from the git root that only exists
   once the clone does.
2. The surfaced error is the raw `FileNotFoundError` telling the user to `Run 'sase bead init' first`, pointing at
   `<workspace>/sase/repos` (a gitignored container directory, not a bead store). The epic plan explicitly required the
   opposite: _"Failure must be a clear, actionable error naming the repository and the remote, not a `FileNotFoundError`
   from `BeadProject`."_

## Fix

### 1. Materialize the beads clone in `claim_bead_for_agent_launch`

In `src/sase/axe/run_agent_runner_bead.py`, resolve the store, and when the resolved beads root is missing, materialize
it through the existing helper before taking the write lock:

```python
from sase.sdd.store import ensure_sdd_kind_clone, resolve_sdd_store

store = resolve_sdd_store(workspace_dir, workspace_num)
beads_dir = store.kind_root("beads")
if not beads_dir.is_dir():
    # A dedicated beads sidecar is materialized lazily, and the provider
    # workflow steps that clone it run after this claim. Clone it here so the
    # claim, its store lock, and its commit all target a real repository.
    ensure_sdd_kind_clone(workspace_dir, workspace_num, "beads", strict=True)
    store = resolve_sdd_store(workspace_dir, workspace_num)
    beads_dir = store.kind_root("beads")
```

Everything after that (`bead_store_write_lock` → `open_bead_project_for_beads_dir` → `commit_sdd_store_files` →
`publish_bead_claim`) is unchanged.

`ensure_sdd_kind_clone` is the right primitive because it already dispatches correctly for every store shape
(`src/sase/sdd/_store_workspace.py:91-147`):

| store shape                                                   | behavior                                                                                                              |
| ------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| in-tree / local / separate-repo                               | no sidecar work; returns the resolved root                                                                            |
| sidecar, schema 2 (no split beads, `store.beads_dir is None`) | ensures the **plans** clone, which is where bead state lives — correct and unchanged                                  |
| sidecar, schema 3 (`store.beads_dir` set)                     | delegates to `ensure_beads_sidecar_clone`, which takes the shared materialization lock and clones the recorded remote |

Guard on `not beads_dir.is_dir()` rather than calling unconditionally: this keeps the warm path free of any new network
work, preserving current claim latency and offline behavior. Only the cold path — which today is a hard failure — does
any extra work.

### 2. Make the failure actionable

`ensure_beads_sidecar_clone` already raises `SddMaterializationError` naming the repository, the remote, the target
path, and a credentials hint. Let that message propagate through the existing `RuntimeError` wrapper instead of the
`FileNotFoundError`. If `ensure_sdd_kind_clone` returns without producing a usable directory, raise an explicit error
that says the beads sidecar could not be materialized for this workspace — do not fall through into `BeadProject` and
re-emit `Run 'sase bead init' first`.

## Tests

Extend `tests/test_run_agent_runner_bead.py` (it already builds `SddStore` values directly and patches
`sase.sdd.store.resolve_sdd_store`, so follow that shape). Use a local bare repo as the sidecar remote, matching the
fixtures in `tests/sdd_store/test_workspace_clone.py`.

1. **Cold split-beads store claims successfully.** Schema-3 sidecar store, seeded beads remote, `sase/repos/beads`
   absent. Assert the claim materializes the clone, returns an `IN_PROGRESS` issue assigned to the agent, and that the
   claim was committed and published (the existing managed-store assertions).
2. **Materialization precedes the store lock.** Assert `bead_store_write_lock` observes an existing directory — i.e. the
   claim is serialized rather than silently running unlocked. A call-order assertion against a patched
   `ensure_sdd_kind_clone` / `bead_store_write_lock` pair is sufficient.
3. **Warm clone does no extra materialization.** Existing beads clone → `ensure_sdd_kind_clone` is not called. This pins
   the no-added-latency property.
4. **In-tree and schema-2 stores are unchanged.** The two existing tests must still pass untouched; add a schema-2
   sidecar case asserting no dedicated beads clone is created.
5. **Failure surface.** Materialization raises `SddMaterializationError` → the resulting `RuntimeError` mentions the
   beads repository/remote and does **not** contain `Run 'sase bead init' first`.

## Verification

```bash
just install    # ephemeral workspaces need this before anything else
just check
```

Then confirm the real scenario end to end: relaunch the epic lander for `sase-a8`
(`%id(land, clan=sase-a8, bead=sase-a8)` + `#bd/land_epic:sase-a8`) into a workspace with no `sase/repos/beads`, and
confirm it claims the bead and proceeds into its workflow steps instead of failing in under a minute. Check the run's
`done.json` shows a non-failed outcome and that no new `error_report.md` names `claim_bead_for_agent_launch`.

Stale state to clean up: workspace #12 is still held by the failed run — dismiss the failed `sase-a8.land` agent in
`sase ace` to release it. The bead claim itself was already released
(`Released bead claim on sase-a8 from waiting agent sase-a8.land`), so no manual bead surgery is needed.

## Out of scope

- **The waiting-agent claim path** (`src/sase/bead/claims.py:144-165`, `:244-260`). It resolves through
  `canonical_beads_dir_for_project`, which reads from the _primary_ checkout and returns `None` when the store is
  unavailable, so it degrades instead of crashing. The launch claim is the only path that resolves a beads dir from the
  agent's own workspace. Leave `claims.py` alone.
- **Making provider workspace prep clone the beads sidecar eagerly.** That would undo the epic's deliberate auto-clone
  gating (an un-migrated project's beads remote does not exist, and eager cloning would fail workspace preparation for
  every project). The lazy-on-demand model is correct; this plan only adds the missing on-demand call site.
- **Deduplicating the bead-store path classification.** The same "which parts make a bead root" logic exists in
  `store_locator.open_bead_project_for_beads_dir`, `agent/bead_display._bead_project_location`, and
  `integrations/_mobile_helper_beads`. Only `bead_display` learned the `sase/repos/beads` → `BEADS_DIRNAME_ROOT` branch
  during the epic. This is a real duplication smell, but for the split layout all three resolve to the same `beads_dir`,
  so it is cosmetic today. Do not fold it into this fix.
