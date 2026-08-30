---
tier: tale
title: Settled handoff gates bucket as Running, not Done
goal:
  An approved tale gate shell shows TALE APPROVED and stays in the Running agent group
  instead of dropping into Done the moment the gate settles.
size: small
proposed_by: bbugyi200.athena.0gd
---

- **AGENTS:**
  - [bbugyi200.athena.0gd](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0gd.md)
  - [bbugyi200.athena.chop.refresh_docs.sase.4_310058.1](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.chop.refresh_docs.sase.4_310058.1/README.md)
- **COMMITS:**
  - [fdb962c](https://github.com/sase-org/sase/commit/fdb962c13ab6827a5ca3b7c3aca3c0d94a5a261c)
    — fix(tui): keep approved gate shells in the Running bucket
  - [341d177](https://github.com/sase-org/sase/commit/341d17739bf0d1ac7baae2871728788801cecf7c)
    — docs: refresh current behavior reference

# Settled handoff gates must bucket as Running, not Done

## Goal

An approved tale gate shell shows `TALE APPROVED` and stays in the **Running** agent
group. Today the settled gate row publishes the right status label but forces the
**Done** bucket, so approving a tale drops the gate shell (and the family container
mirroring it) into `Done` until the coder follow-up agent registers.

## Evidence

Live gate shell for family `sase-vk.land.w2.f0` after its tale was approved
(`agent_meta.json` / `done.json`):

```json
{
  "gate_state": "answered",
  "gate_start_status": "TALE",
  "gate_stop_status": "TALE APPROVED",
  "gate_followup_agent": "sase-vk.land.w2.f0--code"
}
```

The projection chain that turns that into a `Done` row:

1. `src/sase/gate_shell/state.py:17` — `GATE_STATE_BUCKETS` maps `"answered": "Done"`.
2. `src/sase/ace/tui/models/_loaders/_meta_enrichment_common.py:244` (`apply_gate_meta`)
   and `:273` (`apply_gate_done`) set `agent.status_bucket = gate_state_bucket(state)`
   on every gate member row.
3. `sase.agent.status_buckets.agent_status_bucket` treats a valid `status_bucket` as an
   override that wins over the status-derived mapping, so the row buckets `Done` even
   though `status_bucket_for_values("TALE APPROVED")` is `Running`.
4. `src/sase/ace/tui/models/agent_groups/_keys.py:196` groups the `BY_STATUS` L0 bucket
   off exactly that value, and `_mirror_root_from_child`
   (`src/sase/ace/tui/models/_agent_status_apply.py:43`) copies the gate's
   `status_bucket` onto the family container.

So `status_buckets.py:108-113` already documents the intended semantics —
"`PLAN APPROVED` / `TALE APPROVED` ... are actively executing states and read as
**Running**" — and the gate member row is the one place that contradicts it.

## Design

A gate that settles into a **handoff status** has not finished the work; it moved the
work to a successor agent. Those settled statuses must keep the bucket their own status
implies instead of the gate-state bucket:

| settled status  | gate branch                | successor             |
| --------------- | -------------------------- | --------------------- |
| `TALE APPROVED` | tale gate `approve+commit` | `--code` coder        |
| `PLAN APPROVED` | tale gate `approve`        | `--code` coder        |
| `EPIC APPROVED` | epic gate `approve`        | epic launch           |
| `ANSWERED`      | question gate settled      | question continuation |

Every other settled status keeps today's gate-state bucket. That distinction has to be
an explicit allow-list, **not** "let the status decide whenever the gate is terminal":
`status_bucket_for_values` falls through to `Running` for unrecognized labels, so a
blanket rule would wrongly promote `PLAN CANCELLED`, `PLAN TIMED OUT`, `PLAN FAILED` and
the default `GATED` label into `Running`.

`GATE_STATE_BUCKETS` itself must not change. `answered` is a genuinely successful
terminal gate state and `tests/test_done_outcome_classification.py:29` pins
`GATE_SUCCESS_STATES` to the map's `Done` entries; the fix belongs in the member-row
projection that combines gate state with the settled status.

## Changes

### 1. Name the handoff statuses — `src/sase/agent/status_buckets.py`

Next to `APPROVED_PLAN_STATUSES` (line 52), add:

```python
ANSWERED_STATUS = "ANSWERED"
#: Settled shell statuses that mean the shell handed its work to a successor
#: agent rather than finishing it.  A shell row carrying one of these must keep
#: the bucket its status implies (``Running``) instead of the terminal bucket
#: its shell state implies, so approving a plan never files the family under
#: ``Done`` while the follow-up agent is still coming up.
HANDOFF_SETTLED_STATUSES: frozenset[str] = APPROVED_PLAN_STATUSES | frozenset(
    {EPIC_APPROVED_STATUS, ANSWERED_STATUS}
)
```

Use `ANSWERED_STATUS` for the existing `"ANSWERED"` literal in
`status_bucket_for_values` (line 208) so the two definitions cannot drift. Export the
new names if the module has an `__all__`; otherwise leave the module surface as is.

### 2. Add the member-row bucket helper — `src/sase/gate_shell/state.py`

```python
def gate_member_status_bucket(gate_state: str | None, status: str | None) -> str:
    """Return the display bucket for a gate member row showing *status*.

    A gate that settled into a handoff status (an approved plan, an answered
    question) launched a successor instead of finishing, so the row keeps the
    ``Running`` bucket its status implies.  Every other state -- pending,
    settling, rejected, cancelled, timed out, failed -- keeps the bucket its
    gate state implies.
    """
    if status in HANDOFF_SETTLED_STATUSES:
        return status_bucket_for_values(status)
    return gate_state_bucket(gate_state)
```

Import `HANDOFF_SETTLED_STATUSES` and `status_bucket_for_values` from
`sase.agent.status_buckets` at module level, and add `gate_member_status_bucket` to
`__all__`.

**That import is safe here and only here.** `sase/agent/__init__.py` eagerly imports
`launcher`, `running`, and friends, so importing any `sase.agent.*` submodule executes
that whole package init. A static walk of module-level imports shows the init reaches
`sase.shells.state` (via `sase.agent.running` -> `running_listing` ->
`sase.monitor_state` -> `sase.shells.state`) but never reaches `sase.gate_shell`. So:

- `sase/gate_shell/state.py` may import `sase.agent.status_buckets` — no cycle.
- `sase/shells/state.py` **may not**; putting the helper in the shared shell substrate
  instead would deadlock the import (`shells.state` -> `sase.agent` init ->
  `monitor_state` -> partially-initialized `shells.state`). Do not "generalize" it down
  there without breaking that edge first.

Re-run the check if the import graph looks different by the time this lands; parsing
`src/` with `ast` and walking module-level `sase.*` imports out of `sase.agent` is
enough.

### 3. Use it in the loaders — `src/sase/ace/tui/models/_loaders/_meta_enrichment_common.py`

- `apply_gate_meta`, end of the `gate_member` branch (lines 244-245): swap the two
  statements so the status is resolved first, then bucket from it.

  ```python
  agent.status = pair.stop if gate_state_is_terminal(state) else pair.start
  agent.status_bucket = gate_member_status_bucket(state, agent.status)
  ```

- `apply_gate_done` (lines 271-291): today line 273 buckets from `state` _before_ the
  `status_label` block at 286-291 resolves `agent.status`. Drop the assignment at 273
  (keep `agent.gate_state = state`) and set the bucket once, after the status is final:

  ```python
  if agent.gate_state:
      agent.status_bucket = gate_member_status_bucket(agent.gate_state, agent.status)
  ```

  Keep the existing guard shape: when `gate_state` resolves to nothing, neither
  `agent.gate_state` nor `agent.status_bucket` is touched.

Both done-marker loaders (`_done_snapshot_loaders.py:193`,
`_done_filesystem_loaders.py:237`) route through `apply_gate_done`, so they need no edit
of their own.

### 4. Keep the non-TUI gate surfaces consistent — `src/sase/gate_shell/models.py`

`GateShellRecord.status_bucket` (line 79) feeds `gate_shell_runtime_json`
(`src/sase/gate_shell/projection.py:50`), the shape shared by
`sase gate list`/`show`/`cancel` and the ACE `GATE` section. Compute it the same way,
from the record's own effective status:

```python
@property
def status_bucket(self) -> str:
    """Return this gate shell's display status bucket."""
    pair = gate_status_pair(self.start_status, self.stop_status)
    status = effective_gate_status(
        pair, gate_state=self.gate_state, settled=self.is_terminal
    )
    return gate_member_status_bucket(self.gate_state, status)
```

`models.py` already imports `gate_status_pair` from `sase.gate_shell.status` and
`gate_state_bucket` / `gate_state_is_terminal` from `sase.gate_shell.state`, so this
only adds `effective_gate_status` to the first import and `gate_member_status_bucket` to
the second — no new package edge, and `gate_state_bucket` becomes unused here (drop it
from the import or symvision will flag it).

### 5. Keep the visual fixture a faithful mirror — `tests/ace/tui/visual/_ace_agents_png_snapshot_family_panel_fixtures.py:191`

The fixture hand-builds gate rows with `status_bucket=gate_state_bucket(state)`. Switch
it to `gate_member_status_bucket(state, <the status expression already on line 190>)` so
it cannot drift from the loader. None of its four gate rows uses a handoff status
(`ANSWERED` appears only on a `pending` row; the `answered` row settles to `APPROVED`),
so this produces byte-identical rows and **no PNG golden churn is expected**. If
`just test-visual` does report a diff, stop and re-check the fixture rather than
accepting the golden.

## Tests

Add to `tests/ace/tui/models/test_gate_rows.py` (wire + filesystem + done-snapshot
loader paths already have helpers there):

1. An `answered` gate member with `stop_status="TALE APPROVED"` gets
   `status_bucket == "Running"` and `agent_status_bucket(agent) == "Running"`.
2. Same for `PLAN APPROVED`, `EPIC APPROVED`, and a question gate's `ANSWERED`.
3. Regression guards that the rest of the ladder is untouched: an `answered` gate with
   `stop_status="PLAN REJECTED"` stays `Done`; a `stopped` gate with
   `stop_status="PLAN CANCELLED"` stays `Done`; `timeout`/`failed` stay `Failed`;
   `pending` stays `Stopped`; `settling` stays `Running` **and still projects the start
   label** (`test_settling_gate_meta_projects_start_label_and_bucket` at line 44 must
   keep passing unchanged).
4. The done-marker path: a `gated` done marker with `status_label="TALE APPROVED"` and
   `gate_state="answered"` buckets `Running` — this is the one that would regress if the
   `apply_gate_done` reordering is done wrong.

Add to `tests/test_agent_loader_status_override_gate_shell_family.py`, beside
`test_settled_approve_and_commit_gate_projects_tale_approved` (line 194): assert that
after `_apply_status_overrides` the **container** row's
`agent_status_bucket(root) == "Running"`, i.e. the family leaves the `Done` group the
moment the tale is approved. The existing status assertions in that module must not
change.

Add to `tests/gate_shell/` a case pinning `GateShellRecord.status_bucket == "Running"`
for an answered handoff gate, next to the existing pending/`"Stopped"` assertion at
`tests/gate_shell/test_member_store.py:93`.

Check `tests/ace/tui/models/_agent_family_members_helpers.py:111`, which hardcodes
`status_bucket="Running" if gate_state == "settling" else "Done"`. If any test built
from it uses a handoff stop status, make the helper use `gate_member_status_bucket`;
otherwise leave it.

## Verification

`sase/memory/lint_and_test.md` governs this repo. This workspace is an ephemeral clone,
so:

```bash
just install          # may be required before anything else in a fresh workspace
just check            # agent default: all lint gates + diff-scoped tests
just test-visual      # this change touches a PNG snapshot fixture
```

Run `just check-full` through `/sase_monitor` (never inline) before landing, and read
`sase memory read symvision.md` if the unused/misused-symbol gate flags the new
constants.

## Non-goals

- **`GATE_STATE_BUCKETS` stays as it is.** `answered` remains a `Done` gate _state_;
  only the member row's _display_ bucket becomes status-aware.
  `tests/test_done_outcome_classification.py` must keep passing untouched.
- **The `settling` window keeps showing the pending label.** `settle_gate_shell`
  (`src/sase/gate_shell/settlement.py:100-107`) writes `gate_stop_status` and
  `gate_state="settling"` in the same `_write_meta`, so the decided label _is_ already
  on disk during settling and could be shown then. That window is a sub-second internal
  step between the decision and the done marker, it already buckets `Running`, and
  publishing the settled label there would flip the deliberately pinned contract in
  `test_settling_gate_meta_projects_start_label_and_bucket` for every gate kind. Not
  worth it for this fix; easy to revisit as its own change if the blip is ever visible.
- **Row styling is unchanged.** `gate_status_style`
  (`src/sase/gate_shell/status.py:82-88`) keeps rendering a settled gate's label grey
  with a `✓` glyph, so an approved tale will read `TALE APPROVED ✓` in grey inside the
  `Running` group. That is a defensible mixed signal (the gate did settle OK) but it is
  a separate presentation call; leave it alone here.
- **`sase agent list` / the integrations listing are out of scope.**
  `src/sase/integrations/_agent_list_entry_builder.py` has monitor-aware status and
  bucket derivation (`record_status_bucket` at line 62, `build_agent_list_entry` at
  line 99) but no gate equivalent at all, so a settled gate row there reports plain
  `DONE` rather than any gate status. That is a separate, larger parity gap. File it
  with `/sase_new_task` rather than widening this tale.

## Risks

Once a gate settles into a handoff status its row buckets `Running` for good — nothing
later retires that label. In practice the family container stops mirroring the gate as
soon as the successor row exists (`_agent_status_apply.py` mirrors the newest child), so
the sticky value is confined to the gate member row, and it matches the sticky
`TALE APPROVED` that legacy no-gate-shell planner rows already carry
(`approved_followup_planner_status`,
`src/sase/ace/tui/models/_agent_status_family_policy.py:98`). The one case where it is
visible is a settled gate whose follow-up failed to launch (`gate_followup_error` set):
that family will sit in `Running` instead of `Done`. That is accepted — a gate whose
successor never started is stuck work, not finished work, and the row already surfaces
the follow-up error in its detail panel.
