---
tier: tale
title: Hide machine operational-lease claims from the Agents tab
goal:
  Operational workspace leases taken by chops and other host work no longer render as
  phantom STARTING agent rows in the ACE Agents tab.
size: small
proposed_by: bbugyi200.athena.03w
create_time: 2026-08-16 12:20:22
status: wip
---

# Plan: Hide machine operational-lease claims from the Agents tab

## Problem

Rows like `[agent] bead_claim_checks:gh_sase-org__sase (STARTING)` appear in the ACE
Agents tab for a few seconds and then vanish. They are not agents. They are
machine-owned operational workspace leases that a chop takes to get a writable bead
store, and the TUI's claim loader turns every live `RUNNING:` claim into an agent row.

### Root cause chain

1. The `bead_claim_checks` chop calls
   `writable_bead_store_for_machine(project_name, workflow="chop:bead_claim_checks", holder=f"bead_claim_checks:{project_name}")`
   (`src/sase/scripts/sase_chop_bead_claim_checks.py:306-310`).
2. That delegates to `operational_workspace_lease`
   (`src/sase/bead/background_store.py:94-101`), which claims a unified-pool workspace
   in the ProjectSpec `RUNNING:` field via
   `claim_next_axe_workspace(spec, workflow=workflow, pid=..., cl_name=claim_name)`
   where `claim_name = cl_name if cl_name else holder`
   (`src/sase/workspace_provider/lease.py:246-254`, `:477-493`). No
   `artifacts_timestamp` is passed.
3. The TUI loader `load_agents_from_running_field`
   (`src/sase/ace/tui/models/_loaders/_running_loaders.py:120-181`) builds an `Agent`
   from every live claim, with `status="STARTING"` and `cl_name=claim.cl_name`. The only
   claims it skips are `axe(hooks)*` hook processes (`:133-136`). The lease claim
   therefore renders as `[agent] <holder> (STARTING)`, and the holder embeds the
   ProjectSpec key (`gh_sase-org__sase`) rather than the project name.
4. The row is visible for the lease's lifetime and disappears when
   `release_operational_lease` runs in the context manager's `finally`
   (`lease.py:643-666`) — exactly the "shows up randomly, disappears quickly" behavior.

### Why the existing STARTING-row hiding does not catch it

`agent_is_rendered_in_agents_panel` → `_agent_is_starting`
(`src/sase/ace/tui/models/agent_panels.py:105-131`) hides a `STARTING` row only for
`STARTING_ROW_HIDE_GRACE_SECONDS` (120s) **after its `start_time`**, and returns `False`
immediately when `agent.start_time is None`:

```python
if agent.status != "STARTING":
    return False
if agent.start_time is None:
    return False
```

A lease claim has no `artifacts_timestamp`, and
`parse_timestamp_from_workflow_name("chop:bead_claim_checks")` finds no embedded
`YYMMDD_HHMMSS`, so `start_time` is `None` and the grace window never applies. That
`start_time is None` branch is deliberate: commit `0083d1e10` added the grace window as
defense in depth so an _unmapped agent_ claim surfaces as a real selectable row instead
of a silent phantom count. It should stay as-is — these lease claims are not unmapped
agents, they are non-agents, and they belong in the loader-level skip list instead.

### Existing hiding infrastructure surveyed

Three distinct mechanisms exist; only the first fits this case.

1. **Loader-level skip** (`_running_loaders.py:133-136`) — `axe(hooks)*` claims are
   dropped before an `Agent` is ever constructed, with the comment "Skip hook
   processes - they're not agents". Covered by
   `tests/test_agent_loader.py::test_load_all_agents_filters_hook_processes`. This is
   the exact precedent for a non-agent claim.
2. **Panel render predicate** (`agent_panels.py` + `agent_panel_index.py`) — the
   120-second STARTING grace window. Does not apply (see above), and widening it would
   regress the anti-phantom guarantee from `0083d1e10`.
3. **`Agent.hidden` flag + `.` toggle** (`_agent_state.py:280-281`,
   `_agent_list_render_agent.py:200-201`, `_loading_compute.py:289-302`). This does
   **not** remove a row: it adds a `◌` icon and auto-dismisses the agent once it
   completes. The `.` toggle (`hide_non_run_agents`) only hides agents that are not
   `is_always_visible`, and a live `STARTING` claim with `runner_is_live=True` is always
   visible. So mechanism 3 alone cannot hide these rows.

### Sibling claims with the same defect

Every `operational_workspace_lease` caller produces the same phantom row, so the fix
must cover the family rather than one label:

| Call site                                             | `workflow`                   | `holder` (becomes `cl_name`)      |
| ----------------------------------------------------- | ---------------------------- | --------------------------------- |
| `src/sase/scripts/sase_chop_bead_claim_checks.py:306` | `chop:bead_claim_checks`     | `bead_claim_checks:<project>`     |
| `src/sase/external_mirror/issues.py:318`              | `chop:external_issue_mirror` | `external_issue_mirror:<project>` |
| `src/sase/bead/claims.py:158`                         | `bead_claim`                 | `wait-claim:<agent>`              |
| `src/sase/bead/claims.py:242`                         | `bead_claim`                 | `wait-retain:<agent>`             |
| `src/sase/bead/claims.py:302`                         | `bead_claim`                 | `wait-release:<agent>`            |
| `src/sase/_plan_archive_approval.py:98`               | `plan-archive`               | `plan-archive`                    |

The `bead/claims.py` sites pass `prefer_existing_claim=True`, so they only take a lease
when they cannot reuse a live machine-owned sidecar claim — but they do take one then.

## Approach

Make an operational-lease claim self-identifying on disk, then skip it in the loader.

The single write chokepoint is `acquire_operational_lease`, which is also the single
source of `OperationalLease.workflow` — the value echoed into the persisted settlement
policy (`_operational_lease_settlement_policy`, `lease.py:407-421`), the claim transfer
(`_transfer_operational_lease`, `lease.py:360-382`, `new_workflow=lease.workflow`), and
every release path (`release_operational_lease`, `_release_acquired_claim`). Normalizing
the label there keeps acquire, transfer, settlement, and release consistent with what is
actually written to the `RUNNING:` field.

The reserved wrapper is `lease(<workflow>)`, mirroring the `workflow(<name>)` convention
this same loader already parses (`_running_loaders.py:138-141`) and making `RUNNING:`
lines self-documenting for a human reading a ProjectSpec.

Rejected alternatives:

- **Read-side allow/deny list of known labels** (`chop:*`, `bead_claim`,
  `plan-archive`): the label set is open-ended — any future
  `operational_workspace_lease` caller silently reintroduces the bug — and `bead_claim`
  / `plan-archive` are too generic to match safely.
- **Widening the STARTING grace window to cover `start_time is None`**: reverts the
  anti-phantom guard added by `0083d1e10` and hides genuinely stuck agent claims.
- **Setting `Agent.hidden = True` on lease rows**: does not remove the row (see
  mechanism 3 above).

### Verified non-impacts

- No Python code matches on the raw lease labels for claim identification. Everything
  that branches on `claim.workflow` matches agent/hook/monitor labels
  (`stale_running_cleanup.py:90`, `hooks_runner.py:50,66,106`, `mentor_checks.py:257`,
  `processes.py:518`, `change_actions.py:530`, `accept/workflow.py:210`,
  `_runners_data.py:272-274`) or just displays it (`inventory.py:432,439,506`,
  `display_helpers.py:110`).
- `WorkspaceClaim.from_line` parses the workflow column as `(\S+)`
  (`running_field/_model.py:68-70`); `lease(chop:bead_claim_checks)` contains no
  whitespace and round-trips.
- The Runners modal (`_runners_data.py:255-315`) already drops these claims because it
  requires a prompt preview (`if not prompt_preview: continue`). No change needed.
- The `sase agent list` CLI path (`agent/running_listing.py`) is artifact-driven, not
  claim-driven, and never built these rows. No change needed.
- The Rust core's `sase_core::workspace_lease` treats `workflow` as an opaque non-empty
  string (`validate_operational_lease_policy` only checks `!workflow.trim().is_empty()`)
  and has no agent-vs-lease claim classification at all. No Rust change is required.
- Claims written by a pre-upgrade process keep their old bare label and are released
  with that same stored label (in-memory `OperationalLease` or persisted settlement
  policy), so no in-flight lease is orphaned by the upgrade. Such a claim can still
  render one last phantom row until it is released; that is a one-time transient.

### Rust core backend boundary note for the reviewer

Per `CLAUDE.md`'s litmus test, "is this `RUNNING:` claim an agent?" is behavior a second
frontend would need to match. This plan keeps it in Python because (a) the Rust core has
no RUNNING-field claim-label vocabulary today, (b) the label is _constructed_ in the
pure-Python lease chokepoint, (c) it is a string-prefix test on the TUI's hot agent
loader path where a binding hop is not warranted, and (d) the directly analogous
`MONITOR_WORKSPACE_CLAIM_WORKFLOW` constant already lives in Python
(`src/sase/monitor/claims.py`). If the reviewer prefers the vocabulary in
`../sase-core/crates/sase_core` instead, say so at approval time — that changes the
shape of step 1 only.

## Implementation Steps

### 1. Add the reserved claim-label vocabulary

Create `src/sase/running_field/_claim_labels.py`. This package already owns the
`RUNNING:` line format (`_model.py`) and is already imported at module scope by both the
TUI loader and `workspace_provider/lease.py`, so it adds zero import cost to the hot
loader path (unlike `sase.workspace_provider`, whose `__init__` pulls in pluggy, the
hookspec, ownership, and the registry).

Module docstring: explain that the `RUNNING:` field records both agent runs and
machine-owned operational leases, that only the former are agents, and that leases wrap
their caller-supplied workflow identity in a reserved `lease(...)` label so any reader
can tell them apart from the claim line alone — the same way `workflow(...)` already
marks a workflow claim.

Contents:

- `OPERATIONAL_LEASE_CLAIM_PREFIX = "lease("`
- `operational_lease_claim_workflow(workflow: str) -> str` — returns
  `f"lease({workflow})"`. Must be **idempotent**: return `workflow` unchanged when it is
  already an operational-lease label, so a normalized label fed back in is never
  double-wrapped.
- `is_operational_lease_claim_workflow(workflow: str | None) -> bool` — true when
  `workflow` is truthy, starts with the prefix, and ends with `")"`.

Re-export all three from `src/sase/running_field/__init__.py` and add them to `__all__`
(which is alphabetically sorted). Extend that module's docstring so the documented
`RUNNING:` format mentions the reserved `workflow(...)` and `lease(...)` label forms.

### 2. Normalize the claim label at the lease chokepoint

In `src/sase/workspace_provider/lease.py`:

- Add `operational_lease_claim_workflow` to the existing top-level
  `from sase.running_field import (...)` block.
- In `acquire_operational_lease`, after the existing `workflow` / `holder` validation,
  compute `claim_workflow = operational_lease_claim_workflow(workflow.strip())`.
- Use `claim_workflow` — not `workflow` — for `_claim_pool_workspace(...)`, for both
  `_release_acquired_claim(spec, workspace_num, ..., claim_name)` calls in the two
  `except` handlers, and for the returned `OperationalLease(workflow=...)`.

Because `OperationalLease.workflow` is what the settlement policy persists, what
`_transfer_operational_lease` passes as `new_workflow`, and what every release path
matches on, storing the normalized value is what keeps all four consistent. Do not
change `OperationalLease.holder` — the holder stays the caller's identity and is what
`cl_name` falls back to.

Update the `acquire_operational_lease` docstring to state that the RUNNING-field label
is the reserved `lease(<workflow>)` form and that `OperationalLease.workflow` reports
that on-disk label rather than the caller's argument.

### 3. Skip lease claims in the TUI agent loader

In `src/sase/ace/tui/models/_loaders/_running_loaders.py`:

- Add `is_operational_lease_claim_workflow` to the existing top-level
  `from sase.running_field import (...)` import.
- In `load_agents_from_running_field`, immediately **after** the existing `axe(hooks)`
  skip, add:

  ```python
  # Skip machine-owned operational leases (chops, bead-claim reconciliation,
  # plan archiving). They hold a pool workspace but are not agents, and their
  # cl_name is a lease holder identity rather than an agent or Patch name.
  if is_operational_lease_claim_workflow(claim.workflow):
      continue
  ```

Placement matters: the skip must stay **below** the
`if not _claim_pid_is_live(claim.pid): _release_stale_running_claim(...)` gate, exactly
like the `axe(hooks)` skip, so a dead lease claim is still reaped by the TUI's
best-effort stale-release path.

Also extend the module docstring's existing note about which claims become rows to say
that operational-lease claims are excluded because they are not agents.

### 4. Tests

- `tests/test_agent_loader.py` — add a sibling to
  `test_load_all_agents_filters_hook_processes` proving a live
  `lease(chop:bead_claim_checks)` claim with `cl_name="bead_claim_checks:demo"` and
  `artifacts_timestamp=None` produces **no** agent row. Add a second test proving a
  _dead_ lease claim still reaches `_release_stale_running_claim` (skip ordering
  regression guard).
- `tests/running_field/` (create if absent) or a focused new module — unit-test the
  label helpers: wrapping, idempotence on an already-wrapped label, `None`, empty
  string, and negatives for real agent labels (`ace(run)-260816_112109`,
  `workflow(refresh_cl_desc)`, `axe(hooks)-1`).
- `tests/workspace_provider/test_workspace_lease.py` — extend the existing acquire
  coverage to assert (a) the claim written to the `RUNNING:` field carries
  `lease(<workflow>)`, (b) `OperationalLease.workflow` equals that on-disk label, (c)
  `lease.settlement_policy()["workflow"]` equals it too, (d) `holder` is unchanged, and
  (e) `operational_workspace_lease` releases the claim it wrote (round-trip through
  `get_claimed_workspaces`, ending empty). Reuse the module's existing fixtures rather
  than inventing new ones; `tests/workspace_lease_helpers.py::fake_operational_lease`
  builds `OperationalLease` directly and does **not** go through
  `acquire_operational_lease`, so it needs no change.
- Confirm `_transfer_operational_lease` coverage in that file still passes with the
  normalized `new_workflow`.

### 5. Docs

In `docs/project_spec.md`, extend the `RUNNING` field description (around the "Active
workspace claims written and released by SASE while agents or workflows are running"
bullet) to note that machine-owned operational leases appear with a reserved
`lease(<workflow>)` label in the workflow column and are not agent runs.

### 6. Verify

Run `just install` first (ephemeral workspace), then `just check`. This touches the
running-field and lease modules, so also run `just check-full` through `/sase_monitor`
(`sase monitor start --command 'just check-full' …` with a `--next` action) before
landing, rather than inline.

## Acceptance Criteria

- A running `bead_claim_checks` chop no longer produces any Agents-tab row, and the
  Agents-tab "starting" headline count does not include it either.
- The same holds for `chop:external_issue_mirror`, the three `bead_claim` holders, and
  `plan-archive` leases.
- `RUNNING:` lines for operational leases read
  `#<N> | <pid> | lease(chop:bead_claim_checks) | bead_claim_checks:<project>`.
- Acquire → (optional transfer) → release round-trips cleanly: no lease claim is left
  behind in the `RUNNING:` field, and persisted settlement policies release the claim
  they describe.
- A _dead_ lease claim is still released by the TUI's stale-claim path.
- Genuine agent claims are unaffected: `ace(run)-*`, `workflow(*)`, monitor
  (`ace-monitor`), mentor, and fix-hook rows all still render, and the 120-second
  STARTING grace window in `agent_panels.py` is unchanged.
- `just check` passes; `just check-full` passes via a monitor before landing.

## Out of Scope

- Changing the holder strings themselves. `bead_claim_checks:gh_sase-org__sase` embeds a
  ProjectSpec key rather than the configured `PROJECT_NAME`, which would violate the
  "Show Project Names, Never ProjectSpec Keys" convention _if_ it were user-facing — but
  once the row is hidden it is an internal lease identity only, and holder strings are
  also matched by `bead/claims.py` release logic. Not worth churning here.
- The Runners modal, the `sase agent list` CLI, and `sase workspace` inventory output,
  which either already exclude these claims or intentionally show raw claim state for
  diagnostics.
- Any change to `STARTING_ROW_HIDE_GRACE_SECONDS` or the `start_time is None` branch of
  `_agent_is_starting`.
- Promoting the claim-label vocabulary into `sase_core` (see the boundary note above).
