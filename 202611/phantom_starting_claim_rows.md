---
tier: tale
title: Stop stale workspace-claim placeholders from inflating the TUI starting count
goal:
  A long-running sase tui never counts or hides agents because of stale RUNNING-field
  claim rows; the Agents headline "starting" count matches a freshly launched TUI, and
  every running agent renders as its own row.
size: medium
proposed_by: bbugyi200.athena.1cn
proposed_with: claude/opus
create_time: 2026-11-15 13:04:31
status: wip
---

# Stop stale workspace-claim placeholders from inflating the TUI "starting" count

## Problem

A long-running `sase tui` shows `N starting` in the Agents headline (for example
`14 [3 running · 2 starting · 2 waiting · …]`) when no agent is actually starting. A
freshly launched TUI over the same disk state shows no `starting` count. The phantom
count never clears; only a TUI restart clears it.

## Root cause (confirmed against a live TUI)

A read-only dump of a live TUI's `_agents` roster found the two phantom rows. Both were
**workspace pre-claim placeholders**:

```
agent_type=RUNNING  status=STARTING  raw_suffix=None  runner_is_live=True
workflow='ace(run)-261115_123045'  pid=<launcher pid, now dead>  workspace_num=16
artifacts_dir=None  index_record_dir=None  agent_name=None
```

How they got there:

1. The regular launch path pre-claims a workspace under the **launcher's** pid and
   writes the RUNNING-field claim **without an artifacts timestamp**
   (`src/sase/agent/launch_executor_workspace.py`: `_preclaim_axe_workspace`,
   `spawn_slot_with_workspace_retry`). After spawning, it moves the claim to the runner
   pid and adds the artifacts timestamp
   (`#16 | <runner pid> | ace(run)-261115_123045 | gh_… | 20261115123045`).
2. If a TUI load runs inside that short window,
   `load_agents_from_resolved_running_claims`
   (`src/sase/ace/tui/models/_loaders/_running_loaders.py`) builds a claim row with
   `status="STARTING"`, `raw_suffix=None`, and `runner_is_live=True`.
3. Later TUI loads are Tier 1 / bounded / artifact-delta patches. They merge over the
   cached roster in `merge_incomplete_load_after_complete_history`
   (`src/sase/ace/tui/actions/agents/_loading_compute_merge.py`). Nothing removes the
   cached placeholder:
   - its `_tier1_merge_key` is `("identity", RUNNING, cl_name, None)`, and no fresh row
     ever has that key again;
   - `_tier1_stable_merge_key` is `None` because `raw_suffix` is `None`, so stable
     replacement never applies;
   - `is_dismissed` returns `False` because `runner_is_live` is a stale `True`, copied
     at load time and never revalidated;
   - `dedup_running_vs_workflow` needs a `raw_suffix`, and `dedup_by_pid` never matches
     because the launcher pid differs from the runner pid.
4. The row is `STARTING`, so `agent_is_rendered_in_agents_panel` hides it. It is still
   counted through `AgentPanelIndex.hidden_starting_indices`
   (`src/sase/ace/tui/models/agent_panel_index.py`) and surfaced as `starting` by
   `_display_detail_info.py`. Because the row is never rendered, it can't be dismissed
   either. Only a complete-history load replaces the roster, and that runs once after
   startup.

### Same mechanism, worse effect (latent)

An offline merge simulation found a second variant. A suffixed claim row can be cached
as `STARTING` before the artifact index has the agent's WORKFLOW record. On the next
patch load, the incoming WORKFLOW row has the same stable key
`("artifact-row", project_file, raw_suffix)` as the cached claim row. The new-root pass
then skips the incoming row. In the cached pass, `_stable_replacement_wins` returns
`False` for a plain agent with no clan, tribe, or session, since agent types differ and
nothing is structural or terminal. Result: the real RUNNING row is dropped. The agent
renders nowhere and is counted as `starting` instead. Clan members escape only because
`_adds_structural_placement` fires on `agent_clan`.

Both variants share one flaw: **a cached RUNNING-field claim projection is treated as
durable history**, though it is only a transient observation that every claim-scanning
load re-derives.

## Fix design

This is TUI roster-cache maintenance (presentation state), so it stays in Python. No
`sase-core` / Rust change is needed.

1. **Claim-projection provenance.** Add an explicit field to the TUI `Agent` model
   (`src/sase/ace/tui/models/agent.py`), e.g. `from_running_claim: bool = False`. Set it
   in `load_agents_from_resolved_running_claims` only. Don't set it on home-mode
   `running.json` rows: they come from the artifact snapshot and every load, deltas
   included, re-derives them. Make sure:
   - when `dedup_running_vs_workflow` / `dedup_by_pid` fold a claim row into a WORKFLOW
     survivor, the survivor does **not** inherit the flag (it is now artifact-backed);
   - any agent-row copy helpers (`copy_agent_rows_relinked`, graph ownership copies)
     preserve the field. Check `__slots__`/dataclass field handling in the model.
2. **Record whether a load rescanned claims.** Add a field to `AgentLoadState`
   (`src/sase/ace/tui/models/_agent_loader_artifact_state.py`), e.g.
   `running_claims_scanned: bool = False`. Set it `True` only on paths that actually ran
   `_load_agents_from_all_sources` (which always calls
   `load_agents_from_running_field`). It must stay `False` for:
   - artifact-delta loads (`load_artifact_delta_agents` uses
     `_load_agents_from_artifact_snapshot_sources`, which has no RUNNING-field source);
   - loads where `load_state_keeps_displayed_roster(state)` short-circuits to an empty
     agent list;
   - any bounded-prefix load whose `requested_limit` truncation in `load_tiered_agents`
     can cut claim rows. Verify whether truncation can drop claim rows. If it can,
     either leave the flag `False` for truncated loads or carry live claim rows past the
     cap. Pick one and state it in a code comment. Explicit state beats inferring
     "claims were scanned" from `artifact_source`.
3. **Treat cached claim projections as non-durable in the merge.** In
   `merge_incomplete_load_after_complete_history`, filter `cached_agents` **before**
   building `cached_keys`, `cached_by_stable_key`, `cached_parent_by_suffix`, and the
   clan-context map. Filtering first lets incoming rows that the stale placeholder used
   to shadow become ordinary new roots:
   - If `load_state.running_claims_scanned`: drop every cached row with
     `from_running_claim=True`. The incoming load is authoritative for claims, and any
     still-live claim is re-emitted, either as its own row or folded into its artifact
     row.
   - Otherwise (artifact delta, or a load that didn't scan claims): drop a cached claim
     projection only when an incoming row supersedes it, meaning the same
     `(project_file, raw_suffix)` with a non-`None` suffix. Keep the rest; the next
     claim-scanning load settles them.
   - Apply the same filtering to the capacity-roster recursion at the end of the
     function. It reuses the merge with `capacity_agents_with_children`.
4. **Belt and braces for stable replacement.** In `_stable_replacement_wins`, return
   `True` when `cached.from_running_claim and not incoming.from_running_claim`: an
   artifact-backed observation always outranks a claim placeholder. Do **not** weaken
   the reverse direction.
   `test_bounded_prefix_shadow_without_turn_state_does_not_clobber_cached_row` pins that
   an incoming placeholder shadow must not clobber a richer cached row, and it must stay
   green.
5. Keep the work off the UI thread. The merge already runs in the prepared-apply worker
   boundary. Add no disk I/O or pid probes to render or headline paths, and leave the
   headline computation in `_display_detail_info.py` unchanged. Read the TUI performance
   memory (`tui_perf.md`) before touching the loading path.

## Tests

Add a focused test module next to the existing merge tests, e.g.
`tests/test_agents_tab_incomplete_merge_claim_placeholders.py`. Reuse
`tests/_agents_tab_incomplete_merge_helpers.py` (`_incomplete_tier1_snapshot`, etc.) and
keep `tests/test_agents_tab_incomplete_merge.py` under the line-count gate. Cover:

1. **Pre-claim placeholder is dropped.** Cached roster: a real WORKFLOW row for suffix S
   plus a claim placeholder (`from_running_claim=True`, `raw_suffix=None`,
   `status="STARTING"`, `runner_is_live=True`, launcher pid). Incoming: a claim-scanning
   Tier 1 load containing only the WORKFLOW row. Assert that no `STARTING` row survives
   and that `build_agent_panel_index(...).hidden_starting_indices == []`.
2. **Live placeholder is not duplicated.** Same as 1, but incoming also re-emits the
   placeholder. Assert exactly one placeholder row.
3. **Suffixed claim-only cache yields to the artifact row (latent variant).** Cached: a
   claim projection for a plain, clan-less agent with suffix S, `status="STARTING"`.
   Incoming: the WORKFLOW row for suffix S, `RUNNING`, with an agent name. Assert that
   the published top-level row for S is the WORKFLOW row with status `RUNNING`, and that
   no `STARTING` row remains. Run it for both a claim-scanning Tier 1 load and an
   artifact-delta load.
4. **Artifact delta keeps unsuperseded claim rows.** Cached: a claim projection for
   suffix T. The delta incoming has no row for T. Assert the claim row is kept, since
   the delta didn't scan claims.
5. **Provenance through dedup.** `load_agents_from_resolved_running_claims` stamps
   `from_running_claim=True`. After `dedup_running_vs_workflow` folds that claim into a
   WORKFLOW row, the survivor has `from_running_claim=False`.
6. **Load-state flag.** `running_claims_scanned` is `True` for a normal tiered load and
   `False` for `load_artifact_delta_agents` and the keeps-displayed-roster
   short-circuit. Monkeypatch the loader sources as the existing loader tests do.

Existing suites that must stay green: `tests/test_agents_tab_incomplete_merge.py`,
`tests/test_agents_tab_artifact_delta_merge.py`,
`tests/test_agent_loader_dedup_cross_project_collision.py`,
`tests/ace/tui/test_starting_agent_poll.py`,
`tests/ace/tui/test_agent_panel_index_integration.py`, and
`tests/ace/tui/test_agents_info_metrics_generations.py`.

## Verification

- `sase tool run check`. Read `lint_and_test.md` first.
- Optional, as a manual sanity check: with a fresh TUI kept open
  (`sase screenshot --keep`), launch a couple of agents. Confirm the headline `starting`
  count returns to 0 once they are running, and that each launched agent renders as its
  own row.

## Out of scope

- Changing the launcher's pre-claim protocol. Claims without a timestamp are legitimate
  during the spawn window; the TUI just must not treat them as durable.
- Periodic complete-history reloads as a blanket cleanup. They are expensive and would
  hide the bug rather than fix it.
