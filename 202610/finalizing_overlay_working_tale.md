---
tier: tale
title: Show FINALIZING for coder rows working an approved plan or tale
goal: Coder rows labeled WORKING TALE / WORKING PLAN (and the session and clan rows
  that mirror them) show the FINALIZING status word and a non-interrupted finalizer
  chip while host-owned finalizers run, matching plain RUNNING agents.
size: small
proposed_by: bbugyi200.athena.0w9
status: done
---

# Plan: Show `FINALIZING` for coder rows working an approved plan or tale

## Problem

When a coder agent works an approved tale (or a regular approved plan), its Agents-tab
row and its plan-session root row show `WORKING TALE` (or `WORKING PLAN`) for the whole
run, finalization included. They never show the `FINALIZING` word that every other
running agent shows while host-owned finalizers (for example `commit`) execute. The
finalizer chip next to the word is also wrong: it shows the **interrupted** glyph
(`⊛! commit · just fix`) while finalization is still live.

## Diagnosis (confirmed)

The suspicion is correct. The finalizer data is present. Coder turns record
`agent_meta.json["finalizer_status"]` with a `commit` instance that moves through
`planned` → `executing` → `settled`, exactly like plain agents. The bug is purely in TUI
presentation, and it has two layers:

1. **Status relabel.** `_apply_status_overrides`
   (`src/sase/ace/tui/models/_agent_status_apply.py`) calls
   `active_approved_plan_handoff_status`
   (`src/sase/ace/tui/models/_agent_status_agent_session_policy.py`). That function
   rewrites the live coder child's raw `RUNNING` status to `WORKING TALE` /
   `WORKING PLAN`. The plan-session root then copies that label through
   `_mirror_root_from_child`, and a clan container copies it through
   `apply_clan_container_status`.
2. **The overlay only matches the literal string `"RUNNING"`.** Everything in
   `src/sase/ace/tui/models/finalizer_row_state.py` and the row renderer checks for
   exactly `"RUNNING"`, so the relabeled statuses never qualify:
   - `row_status_is_finalizing`: `if agent.status != "RUNNING": return False`.
   - `_finalizer_row_state`:
     `is_finalizing = ... and agent.display_status == "RUNNING"`.
   - `_session_finalizer_row_state`: `container.display_status == "RUNNING" and ...`.
   - `_member_candidates`: `turn_terminal = agent.status != "RUNNING"`. This treats a
     live `WORKING TALE` turn as ended, so its `running` instance becomes `interrupted`.
     That is the cause of the bogus `⊛!` chip.
   - `append_agent_row_status`
     (`src/sase/ace/tui/widgets/_agent_list_render_agent_status.py`): only the
     `agent.status == "RUNNING"` branch consults `row_status_is_finalizing`. The
     `WORKING_PLAN_STATUS` / `WORKING_TALE_STATUS` branches always print the raw label.

Reproduction (before the fix): a tale `.plan` session root (`agent_session_role="root"`,
`plan_action="tale"`) plus a `RUNNING` `.code` child with an `executing` finalizer
summary. After `_apply_status_overrides`, both rows render as
`(WORKING TALE) ⊛! commit · just fix`. A plain `RUNNING` row with the same summary
renders `(FINALIZING) ⊛ commit · just fix`.

There is no Rust-core counterpart. Neither the `WORKING TALE` relabel nor the
`FINALIZING` overlay exists in `sase-core`; both are TUI presentation in this repo. So
the fix stays here and does not move the `sase-core-revision.txt` pin.

## Fix

Teach the finalizer glance state that the coder-specific working labels are live
`RUNNING` turns under another name. Keep the D11 rule: the overlay only replaces the
status **word**. It never changes `agent.status`, buckets, ordering, filters, or
in-flight logic.

### 1. `src/sase/ace/tui/models/finalizer_row_state.py`

- Import `WORKING_PLAN_STATUSES` from `sase.agent.status_buckets`. Add a module-level
  constant with a short comment, for example:

  ```python
  #: Live-turn status words the FINALIZING overlay may replace: raw ``RUNNING``
  #: plus the coder relabels an approved plan/tale handoff applies to it.
  _FINALIZING_OVERLAY_STATUSES = frozenset({"RUNNING", *WORKING_PLAN_STATUSES})
  ```

  Deliberately leave out `EPIC APPROVED` / `PLAN COMMITTED`. They are also settled
  handoff statuses (`HANDOFF_SETTLED_STATUSES`) on non-running gate and planner rows,
  and their running-child producers (`epic` / `commit` roles) are legacy.

- `row_status_is_finalizing`: replace `agent.status != "RUNNING"` with
  `agent.status not in _FINALIZING_OVERLAY_STATUSES`. Move this cheap check above the
  function-local `status_display_agent` import, so non-eligible rows return before the
  import statement runs. The function is now called for more rows; see step 2.
- `_finalizer_row_state`: `agent.display_status in _FINALIZING_OVERLAY_STATUSES`.
- `_session_finalizer_row_state`:
  `container.display_status in _FINALIZING_OVERLAY_STATUSES`.
- `_member_candidates`:
  `turn_terminal = agent.status not in _FINALIZING_OVERLAY_STATUSES`, so a live coder
  turn's `running` instance stays `running` and does not become `interrupted`. Update
  the docstring wording ("non-RUNNING turn") to match.
- Update the `_finalizer_row_state` docstring ("overlays the `RUNNING` word only") and
  the module docstring so they say the overlay covers `RUNNING` and the coder working
  labels.

### 2. `src/sase/ace/tui/widgets/_agent_list_render_agent_status.py`

In `append_agent_row_status`, hoist the overlay into its own branch so that every
eligible status uses one code path. Put it right after the `STARTING` branch and before
`agent.status == "RUNNING"`:

```python
    elif row_status_is_finalizing(agent):
        text.append("FINALIZING", style=f"bold {RUNNING_COLOR}")
    elif agent.status == "RUNNING":
        text.append(display_status, style=f"bold {RUNNING_COLOR}")
```

Leave the existing `WORKING_PLAN_STATUS` / `WORKING_TALE_STATUS` branches unchanged;
they still handle the non-finalizing case. The gate/monitor `presentation` branch and
the named-proc `SETTLING` branch stay ahead of the new branch, so their behavior does
not change. A `FINALIZING` coder row uses the same style as every other `FINALIZING`
row, as requested.

The clan identity `Status:` line and compact clan header
(`prompt_panel/_agent_display_clan_identity.py`) already go through
`presented_status_label`, so step 1 fixes them with no further change. Render cache keys
already include the row's own `finalizer_summary_token`, and through
`followup_signature` the member turns' tokens too (`_agent_list_render_cache.py`), so no
cache-key change is needed. Everything stays pure and in memory, with no I/O on the
render path.

## Tests

Add regression tests (extend `tests/ace/tui/widgets/test_finalizer_glance_surfaces.py`,
or add a sibling module if that file's size or lint limits push back):

1. **Plain coder rows.** For each of `WORKING TALE` and `WORKING PLAN` with the
   `_EXECUTING` summary:
   - `row_status_is_finalizing` is `True`.
   - `presented_status_label` returns `"FINALIZING"`.
   - `_render_row` contains `(FINALIZING)` and `⊛ commit · just fix`, and does not
     contain `⊛!`.
   - `agent.status` is unchanged.
2. **Declaring phase.** A `WORKING TALE` row with a `declaring` summary also presents
   `FINALIZING`.
3. **Not finalizing.** A `WORKING TALE` row whose summary is `planned` (or `None`) still
   renders `(WORKING TALE)`, so the non-finalizing label is preserved.
4. **Terminal coder.** A `TALE DONE` row with an `executing` summary is not finalizing,
   and its chip derives `interrupted` (`⊛!`). This keeps the terminal semantics.
5. **End-to-end through status normalization.**
   - Setup: a tale plan-session root (`AgentType.RUNNING`, `role_suffix=".plan"`,
     `plan_action="tale"`, `agent_session="s"`, `agent_session_role="root"`,
     `status="DONE"`) plus a `RUNNING` `.code` child (`agent_session_role="code"`,
     `parent_timestamp` pointing at the root) that carries the `_EXECUTING` summary.
   - Run `_apply_status_overrides` (`sase.ace.tui.models.agent_loader`). Both rows'
     `status` should be `WORKING TALE`.
   - Then assert that both rows render `(FINALIZING)` with a non-interrupted chip. The
     root goes through `_session_finalizer_row_state`.
   - Use the same setup without `plan_action` for the `WORKING PLAN` variant.
6. **Clan container.** Optional, if cheap: add a case next to
   `test_clan_row_shows_finalizing_for_lone_running_session_member`
   (`tests/ace/tui/models/test_agent_tree_clan_status_finalizing.py`) where the lone
   running member is a `WORKING TALE` session root. The clan row should start with
   `tmp (FINALIZING`.

Existing tests that pin the plain `RUNNING` behavior must keep passing unchanged.

## Verification

- Before finishing, read the `lint_and_test.md` reference memory and follow it
  (`just fix` / `just check` via the guarded flow it describes).
- Run the targeted tests above, plus `tests/test_agent_loader_status_override_tale.py`,
  `tests/ace/tui/models/test_agent_tree_clan_status_finalizing.py`, and
  `tests/ace/tui/widgets/test_agent_list_runtime_rendering_status.py`.
- If any agents-row visual snapshot that contains a finalizing coder row changes,
  inspect it and update it intentionally. Don't regenerate snapshots blindly.
