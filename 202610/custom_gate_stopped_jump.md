---
tier: tale
title: Treat pending custom gates as stopped for ,J and every stopped-agent probe
goal:
  Every Agents-tab row filed under the Stopped group, including a session whose current
  turn is a pending custom gate with a free-form label like CONFIRM, is a ,J target and
  counts toward ,J availability and the startup attention tab.
size: small
proposed_by: bbugyi200.athena.15j
proposed_with: claude/opus
create_time: 2026-10-28 18:13:12
status: wip
---

# Plan: Treat pending custom gates as stopped for `,J` and every stopped-agent probe

## Problem

A row the Agents tab files under the **Stopped** group — including an agent session
whose current turn is a pending custom gate with a free-form pending label such as
`CONFIRM` — must also count as "stopped" for `,J` (jump to next stopped agent), the `,J`
footer/palette availability, and the startup attention-tab choice. Today the `15g`
session in the user's screenshot sits in the Stopped group as `bob-cli (CONFIRM)`, yet
`,J` toasts "No stopped agents" and never lands on it.

## Root cause (verified)

There are two different definitions of "stopped" in the TUI:

1. **Grouping, header counts, clan tallies** use the row's _effective_ bucket,
   `sase.agent.status_buckets.agent_status_bucket(agent)`, which honors an explicit
   `agent.status_bucket` override before falling back to the raw-status mapping.
2. **`,J` and its availability probes** use
   `sase.ace.tui.models.agent_status.is_stopped_agent_status(agent.status)`, which only
   maps the _raw status string_ through the Rust `status_bucket_for_status` binding.

A gate-turn member row gets `status = gate_start_status` (the custom `pending_status`,
for example `CONFIRM`) and `status_bucket = gate_member_status_bucket(state, status)`,
which is `"Stopped"` for a `pending` gate (`GATE_STATE_BUCKETS` in
`src/sase/gate_turn/state.py`; applied in
`src/sase/ace/tui/models/_loaders/_meta_enrichment_gate.py::apply_gate_meta`). The
session root mirrors both fields from that child (`_mirror_root_from_child` in
`src/sase/ace/tui/models/_agent_status_apply.py`). The raw mapping knows nothing about
`CONFIRM`, so `status_bucket_for_values("CONFIRM")` returns `"Running"`. Plan-approval
gates only "worked" by coincidence: their labels (`PLAN`, `TALE`, `EPIC`) are already
Stopped by raw status.

Reproduced in a scratch script against the real test harness
(`tests/ace/tui/_agent_unread_navigation_helpers.UnreadJumpApp`): a row with
`status="CONFIRM"`, `status_bucket="Stopped"` reports `agent_status_bucket == "Stopped"`
but `_has_stopped_agent() is False`, `_jump_to_next_stopped_agent() is False`, and
`commands.context._stopped_agent_count(app) == 0`.

The inverse mismatch also exists and is fixed by the same change: a gate row whose label
is `PLAN`/`TALE`/`QUESTION` but whose bucket override is not Stopped (for example a
`settling` gate, bucket `Running`) is currently a `,J` target even though it is not in
the Stopped group.

## Design

Make every stopped-agent _row_ probe use the same effective bucket the grouping uses.
This is TUI navigation/presentation wiring over an existing Python helper; no
`sase-core` change is needed (the Rust raw-status table stays the fallback, and the
override-wins rule already lives in `sase/agent/status_buckets.py`).

### 1. Row-level predicate — `src/sase/ace/tui/models/agent_status.py`

Add a row predicate next to `is_stopped_agent_status`:

```python
def is_stopped_agent(agent: object) -> bool:
    """Return True for rows displayed in the Stopped agent bucket.

    Honors an explicit ``status_bucket`` override exactly like the grouping
    and header counts (``agent_status_bucket``), so a pending custom gate
    whose display label is free-form (``CONFIRM``) counts as stopped and a
    settling gate still labeled ``PLAN`` does not.
    """
    override = valid_status_bucket(getattr(agent, "status_bucket", None))
    if override is not None:
        return override == "Stopped"
    status = getattr(agent, "status", None)
    return isinstance(status, str) and is_stopped_agent_status(status)
```

- Import `valid_status_bucket` from `sase.agent.status_buckets` (already the module this
  file imports `status_bucket_for_values` from).
- Use `getattr` deliberately: several existing tests pass `SimpleNamespace(status=...)`
  rows with no `status_bucket`/`retried_as_timestamp` attributes
  (`tests/test_command_context_extraction.py`), and those must keep working. This is why
  the predicate does not call `agent_status_bucket(agent)` directly.
- Keep `is_stopped_agent_status(status)` as the raw-status helper the new predicate
  delegates to (so it stays used; no symvision churn).

### 2. `,J` navigation — `src/sase/ace/tui/actions/agents/_unread_navigation.py`

- `_has_stopped_agent`: `any(is_stopped_agent(agent) for agent in self._agents)`.
- `_jump_to_next_stopped_agent`: predicate becomes `is_stopped_agent`. The same
  predicate is re-validated after clan reveal and off-tab reveal
  (`_reveal_and_select_timed_jump_candidate`, `_jump_to_off_tab_timed_candidate`), so
  gate rows inside collapsed clans and on other tabs work with no further change.
- Swap the import from `is_stopped_agent_status` to `is_stopped_agent`.
- Leave `_stopped_agent_jump_time` unchanged: a gate row is neither a pending plan
  review nor `QUESTION` by label, so it orders by `stop_time or start_time`, like any
  other stopped row without plan/question timestamps.

### 3. Palette availability — `src/sase/ace/tui/commands/context.py`

`_stopped_agent_count` counts `is_stopped_agent(a)` instead of
`is_stopped_agent_status(a.status)`. This feeds `CommandContext.stopped_agent_count`,
which gates `leader.jump_to_next_stopped_agent` in
`src/sase/ace/tui/commands/_availability_agents.py`. Update the import.

### 4. Startup attention tab — `src/sase/ace/tui/actions/agents/_agent_tabs_lifecycle.py`

`_attention_status(status)` only sees the root's raw status, so a tab whose newest
attention root is a pending custom gate is not chosen at startup. Change it to take the
root row (rename to `_attention_root(root: object) -> bool`): stopped via
`is_stopped_agent(root)`, or `status == "FAILED"`, or
`is_unread_completed_status(status)` on the raw status (both unchanged). Keep the
existing defensive `try/except` fallback (`status in ("STOPPED", "FAILED")`). Update the
one caller in `_attention_tab` to pass `root` instead of `root.status` (it can drop its
own `root.status` read and the associated `try/except`, or keep it — preserve behavior
for rows whose `status` access raises).

### 5. Leave alone

- `src/sase/ace/tui/actions/agents/_core.py` imports `is_stopped_agent_status` without
  using it in that module; do not touch it unless lint requires it.
- Grouping, header, clan, and rail code already use `agent_status_bucket` and are the
  reference behavior; no change.
- No keymap or `default_config.yml` change (`,J` binding is unchanged).

## Tests

Add focused regression tests; reuse the existing helpers.

1. `tests/ace/tui/test_agent_stopped_navigation.py`
   - A gate-shaped row (`make_agent(status="CONFIRM", ...)` then
     `row.status_bucket = "Stopped"`) makes `_has_stopped_agent()` True on its own.
   - `,J` reaches it: in a list with a `RUNNING` row and a `PLAN` row, repeated
     `_jump_to_next_stopped_agent()` visits both the `CONFIRM`/Stopped row and the
     `PLAN` row (and wraps), ordered by recency as today.
   - Inverse: a row with `status="PLAN"` and `status_bucket="Running"` (settling gate)
     is not a target; with only that row and a `RUNNING` row, `_has_stopped_agent()` is
     False and `_jump_to_next_stopped_agent()` returns False.
2. `tests/ace/tui/test_agent_stopped_navigation_clans.py` (only if the existing helpers
   make it a few lines): a `CONFIRM`/Stopped member hidden by a collapsed clan fold is
   revealed and selected by `,J`, mirroring the existing PLAN/QUESTION collapsed-clan
   test.
3. `tests/test_command_context_extraction.py`
   - Extend or add next to `test_stopped_agent_count_uses_stopped_status_bucket`:
     `SimpleNamespace(status="CONFIRM", status_bucket="Stopped")` counts, and
     `SimpleNamespace(status="PLAN", status_bucket="Running")` does not, while the
     existing attribute-less `SimpleNamespace(status=...)` rows keep their counts.
4. `tests/ace/tui/test_agent_tab_switch.py`
   - Next to `test_startup_selection_attention_tab`: a root on the `sase` tab with
     `status="CONFIRM"` and `status_bucket="Stopped"` makes startup reconcile select the
     `sase` tab. Check how `_row(...)` builds rows; set `status_bucket` on the returned
     row if the helper has no parameter for it.

## Docs

`docs/reference/tui/agents-tab-grouping.md` currently says **Stopped** "keeps the strict
'you need to act' semantics for plan approval, questions, and workflow input." Extend
that sentence to name pending gates (including custom `/sase_gate` gates with their own
pending label), and state in one clause that `,J` targets exactly the rows in the
Stopped bucket. This is a behavior update to an existing Reference page; keep it to that
one sentence (read the `docs_writing.md` reference memory before editing). The `,J` row
in `docs/reference/tui/keymaps.md` is generated from the keymap description and needs no
change.

## Verification

- Re-run the scratch reproduction (a `CONFIRM` row with `status_bucket="Stopped"`):
  `_has_stopped_agent()`, `_jump_to_next_stopped_agent()`, and `_stopped_agent_count`
  must now report the row.
- Run the new and touched test modules directly with the checkout's `.venv` pytest.
- Run `just check` (through `sase tool run`, per the `lint_and_test.md` reference
  memory) before finishing. Do not run `just check-full`.

## Out of scope

- Changing what statuses the raw Rust `status_bucket_for_status` table maps to Stopped.
  Custom gate labels are arbitrary per gate, so the gate-state-derived bucket override
  is the right source of truth; teaching the raw table individual labels would never be
  complete.
- `,J` ordering refinements (for example ordering gate rows by gate creation time).
