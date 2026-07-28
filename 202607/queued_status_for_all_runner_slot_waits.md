---
tier: tale
title: Use QUEUED for every runner-slot admission wait
goal:
  Agents parked at the runner-slot admission gate always display QUEUED, whether their threshold comes from the global
  max_running_agents cap or an authored %wait(runners=N) directive, and WAITING is reserved for agents still blocked on
  an authored dependency.
create_time: 2026-07-28 17:46:09
status: wip
---

- **PROMPT:**
  [202607/prompts/queued_status_for_all_runner_slot_waits.md](prompts/queued_status_for_all_runner_slot_waits.md)

# Plan: Use QUEUED for every runner-slot admission wait

## Problem

Agents parked at the runner-slot admission gate still render `WAITING` whenever their threshold came from an authored
`%wait(runners=N)` directive. Only _implicit_ global-cap waiters are promoted to `QUEUED`.

The result is a self-contradicting Agents tab: the detail panel shows a four-entry `QUEUE` ladder (`#1 audit_bugs… ≤0`,
`#2 audit_improvements… ≤0`, `#3 chop.refresh_docs… ≤0`, `#4 toobig-0t.split_file… ≤3 p20`), while all four of those
same rows render `(WAITING ▶4→0)` / `(WAITING ▶4→3 p20)` in the list, and the header capacity chip counts zero queued
agents. The selected agent's `Wait:` block shows `[agents] … ✓` — its only authored dependency is already satisfied —
and `[runners] ≤ 3 · 4 runners still running · queue #4 of 4`. It is unambiguously sitting in the admission queue, yet
it is labeled `WAITING`.

`WAITING` should mean "blocked on an authored dependency". `QUEUED` should mean "cleared every authored wait; holding
for runner capacity".

## Decision

**Every live runner-slot waiter displays `QUEUED`, regardless of whether its governing threshold is the global
`max_running_agents` cap or an authored `%wait(runners=N)` value.**

This is the load-bearing interpretation of the request, so state it plainly: the user asked for `QUEUED` "when agents
are waiting because of the configured maximum allowed number of running agents", and in the screenshot _no_ agent is
blocked by the global cap (5/10 running) — every parked row carries an explicit threshold. So the literal
global-cap-only reading would change nothing on screen. The intended semantic is capacity-vs-dependency, and the
explicit/implicit provenance of the threshold is a detail, not a status distinction.

Two facts in the existing runtime make this the correct model rather than a cosmetic relabel:

1. **Dependency waits strictly precede slot admission.** `src/sase/axe/run_agent_runner.py::_run_agent` runs
   `_wait_for_dependencies_phase(...)` and only then `_admit_and_launch(...)` → `wait_for_runner_slot(...)`.
   `slot_requested_at` is written only inside that gate
   (`src/sase/axe/run_agent_wait_slots.py::_try_claim_runner_slot`). Therefore a row with `slot_requested_at` set has,
   by construction, already cleared every agent, bead, time, and fork wait. Explicitness of the threshold does not
   change that.
2. **Both kinds of waiter already share one admission queue.** `live_runner_slot_waiters()` and `may_start()` in
   `src/sase/core/runner_slots/_admission.py` rank all waiters together by priority then request FIFO, with no
   explicit/implicit partition. ACE already displays that unified ladder. Only the _status label_ was partitioned.

Nothing about admission behavior changes. This is a display-derivation change plus the renames, row affordances, counts,
docs, and tests that must move with it.

### What is deliberately preserved

- The explicit threshold stays visible on the row (`▶4→0`) and in the `Wait:` detail (`≤ 0 (drain barrier)`), so a drain
  barrier is still identifiable at a glance — it is now a _qualifier on a queued row_ rather than a different status.
- Queue-ladder "parked" styling (`_PARKED_COLOR` amethyst on earlier entries with a stricter threshold) stays. It now
  means "stricter threshold, therefore not counted as ahead of you" rather than "different status". Its documentation
  must be reworded accordingly.
- The `wait_runners_explicit` field itself stays on markers, wire records, and JSON payloads. It still drives rendering
  (`≤N` vs `R/L`, arrow vs fraction) and the wait-modal prefill. Only its role in _status derivation_ is removed.

## Implementation

### 1. Status derivation

**`src/sase/agent/status_buckets.py`**

- Rename the `runner_slot_display_status()` keyword `globally_queued:` → `slot_queued:`.
- Update the docstring: `QUEUED` is derived for any live agent parked at the runner-slot admission gate; persisted and
  scan-level statuses remain `WAITING`, so the promotion stays in-memory and reversible.

**`src/sase/ace/tui/models/agent_runner_slots.py`**

- Rename `_agent_is_globally_queued()` → `_agent_is_slot_queued()` and drop the `and not agent.wait_runners_explicit`
  clause, so it is exactly `_is_live_slot_waiter(agent)`. If it collapses to a pure alias, inline it at the three call
  sites rather than keeping a redundant wrapper — whichever reads better, but do not leave a function whose name implies
  a filter it no longer applies.
- Rename `RunnerCapacitySnapshot.global_cap_queue_count` → `queued_count`. Every live slot waiter is now promoted, so
  set it from `len(queue_entries)` instead of re-deriving it from a bucket test in the loop.
- Keep the clan-container branch passing `slot_queued=False`; clan containers never hold a slot request of their own,
  and their status comes from `aggregate_clan_status`.

**`src/sase/integrations/agent_list_entries.py`**

- Apply the same change to `_is_globally_queued()` (rename to `_is_slot_queued()`, drop the explicitness clause, or
  inline into `_attach_runner_slot_context`).

**`src/sase/ace/tui/actions/agents/_wait_actions.py`**

- In `_apply_live_runner_wait()`, pass `slot_queued=True` instead of `globally_queued=result.runners is None`. This path
  is only reachable when `agent.slot_requested_at` is set (see the two guards in `_apply_wait`), so the row is a live
  slot waiter whatever threshold the modal just applied.
- Leave the `"runners ≤ N"` / `"global runner cap"` notification label alone — it describes the applied threshold, not
  the status.

### 2. Row rendering

**`src/sase/ace/tui/widgets/_agent_list_render_agent.py`**

Explicit-threshold waiters move from the `WAITING` branch to the `QUEUED` branch, so the branch must carry the
affordances that are still meaningful, otherwise the change silently drops information from the list:

- `QUEUED` rows keep `#N/M` (rank) as today.
- Append ` ▶{runner_slots_in_use}→{wait_runners}` when `wait_runners_explicit` and both values are available. Do **not**
  append the implicit `▶R/L` fraction — that stays omitted, since the header capacity chip already reports `R/L`
  globally and the rank already conveys queue state.
- Append ` p{wait_priority}` when `wait_priority_explicit` and the priority is available (currently only rendered on
  `WAITING` rows).
- Style these suffixes in the queued cornflower blue (`QUEUED_STATUS_COLOR`, dim) rather than the amethyst used by the
  `WAITING` branch.
- Suggested order: `QUEUED #4/4 ▶4→0 p20`.
- Remove the now-dead runner-slot suffix block from the `WAITING` branch. A `WAITING` row can no longer be a live slot
  waiter (`slot_requested_at` implies `QUEUED`), so that block is unreachable for live rows; confirm no persisted-scan
  path renders a `WAITING` row with `slot_requested_at` before deleting it, and if one exists, keep a minimal fallback
  rather than regressing it.

`src/sase/ace/tui/widgets/_agent_list_render_cache.py` already keys on `wait_runners`, `wait_runners_explicit`,
`wait_priority`, `wait_priority_explicit`, `slot_requested_at`, `runner_slots_in_use`, `runner_slot_queue_position`, and
`runner_slot_queue_size` — verify, but no change is expected.

### 3. Counts and detail panels

- `src/sase/ace/tui/actions/agents/_display_detail_info.py` — update the two `global_cap_queue_count` references to the
  renamed field.
- `src/sase/ace/tui/widgets/agent_info_panel.py` — no logic change; the `[R/L · Q queued]` chip now reports every live
  slot waiter, and the `W` metric drops them. Confirm the two counts stay disjoint.
- `src/sase/ace/tui/models/_agent_clan.py` — no logic change. `QUEUED` already outranks `WAITING` in clan aggregation
  and queued members are already excluded from the waiting count; more clans will simply read `QUEUED` now.
- `src/sase/ace/tui/models/agent_tribe_summary.py` — no logic change expected; confirm the queued/waiting split flows
  through.
- `src/sase/ace/tui/widgets/prompt_panel/_agent_queue_section.py` — no logic change. Re-check that the `≤N` threshold
  column and `_PARKED_COLOR` accent still read correctly now that every ladder entry is `QUEUED`.

### 4. Documentation

Every one of these currently states the explicit-threshold carve-out and must be corrected — do not leave a stale
sentence behind, this is the contract users read:

- `docs/ace.md`
  - Status-strip paragraph (~line 1057): `waiting` no longer contains "explicit-runner-threshold lanes"; the capacity
    chip's `queued` count is no longer limited to "ambient global-cap waiters".
  - Capacity chip paragraph (~line 1045): drop "Waits with an explicit `%wait(runners=N)` threshold are intentionally
    excluded from `Q`"; `Q` counts every live slot waiter.
  - Bucket definitions (~lines 1066–1069): `Queued` covers every agent that cleared its authored waits and is holding
    for capacity; `Waiting` no longer lists "an authored `%wait(runners=N)` threshold". Move the arrow-form (`▶7→0`)
    description onto the queued row form and update the example row text.
  - Clan aggregation paragraph (~line 964): the `QUEUED [Q3 W6]` example currently reads "three global-cap waiters and
    six explicit, dependency, bead, or time waiters" — re-split it.
  - Active-status table (~line 2161): update both the `QUEUED` and `WAITING` descriptions.
  - Wait-detail bullet (~line 2373): an explicit runner threshold is now shown on a queued row.
  - Wait-modal bullet (~line 724) mentions "WAITING or QUEUED agent" — still accurate, but re-read it in context.
- `docs/integrations.md` — bucket table row for `Queued` (~line 77) and the "Global-cap waiters are promoted…" paragraph
  (~line 105), which ends with "Explicit `%wait(runners=N)` rows keep `WAITING`…".
- `docs/cli.md` (~line 50) — "`sase agent list -j` reports an implicit global-cap waiter as `status: \"QUEUED\"`".
- `docs/configuration.md` (~line 1777, `max_running_agents`) — re-read for consistency with the new semantic.
- `docs/troubleshooting/runner-slots.md` — the opening definition, the `Q` chip paragraph ("`Q` does not include waits
  with an explicit `%wait(runners=N)` threshold"), and the queue-ladder paragraph that explains stricter drain waits as
  "shown in the `WAITING` amethyst instead of being counted as ahead" (reword as a threshold-based, not status-based,
  distinction). The page title and framing still work.
- `docs/xprompt.md` (~line 1371) — re-read the question-yield paragraph; it already says the answered agent "appears as
  a normal runner-slot `QUEUED` row", which is now simply the general case.
- `docs/agent_families.md` (~line 154) — status-priority ordering is unchanged; re-read only.

### 5. Generated skill source

`src/sase/xprompts/skills/sase_agents_status.md` (~line 23) states "An implicit global-cap waiter is reported as
`QUEUED`; authored `%wait(runners=N)` threshold waits remain `WAITING`." Rewrite it for the unified semantic.

Edit **only** the source template. Do not run `sase skill init` / `chezmoi apply` as part of this change: the chezmoi
destination is global, and deploying from an unmerged tree reverts other agents' deployments. Deployment happens from a
clean, landed tree as a separate step.

## Tests

Update the existing suites that encode the old carve-out — several assert `WAITING` for explicit thresholds _by name_,
so renaming them matters as much as flipping the assertion:

- `tests/ace/tui/test_agent_runner_slots.py`
  - `test_runner_capacity_excludes_non_slot_and_explicit_waits_from_global_queue` — explicit waiters are no longer
    excluded; rename to reflect that non-slot rows are the only exclusion, and keep the non-slot half of the assertion.
  - The `("QUEUED", "WAITING")` expectation around line 418 becomes `("QUEUED", "QUEUED")`.
  - `test_stale_queued_status_demotes_without_a_live_slot_request` must still pass unchanged — a stale `QUEUED` with no
    `slot_requested_at` still demotes to `WAITING`. This is the guard that the promotion stayed reversible.
  - `test_queued_rows_match_chip_header_summary_and_capacity_counts` — extend to cover an explicit-threshold waiter and
    assert the chip count equals the ladder length.
- `tests/test_agent_list_entries.py` — the explicit-threshold entry (~line 155) now projects `QUEUED` with the `Queued`
  bucket and glyph.
- `tests/test_agent_clan.py::test_clan_member_counts_ignores_globally_queued_leaf` — re-read and rename; the
  `wait_runners_explicit = False` setup is no longer what makes the leaf queued.
- `tests/test_agents_tab_apply_boundary.py`, `tests/test_agent_loader_dedup_pid_families.py`,
  `tests/ace/tui/widgets/test_agent_queue_section.py` — follow the `global_cap_queue_count` → `queued_count` rename.
- `tests/ace/tui/widgets/test_agent_list_status_indicators.py`,
  `tests/ace/tui/widgets/test_agent_display_waiting_warning.py`,
  `tests/ace/tui/widgets/test_agent_parallel_family_count_chips.py` — row-rendering expectations for the moved `▶R→T` /
  `p{N}` suffixes.
- `tests/ace/tui/models/test_agent_summary_status_counts.py`, `tests/ace/tui/models/test_agent_tribe_summary.py`,
  `tests/test_agent_status_buckets.py` — queued/waiting split.

Add coverage for the new contract:

- An explicit `%wait(runners=0)` drain barrier with `slot_requested_at` set promotes to `QUEUED` and renders
  `QUEUED #N/M ▶R→0`.
- An explicit-threshold waiter with an explicit priority renders the ` p{N}` suffix on the queued row.
- A row with an unsatisfied `waiting_for` dependency and no `slot_requested_at` stays `WAITING` (the negative case that
  keeps the two statuses meaningfully distinct).

Runtime admission tests (`tests/test_run_agent_runner_slot_capacity.py`, `tests/test_run_agent_runner_slot_priority.py`,
`tests/test_run_agent_runner_slot_markers.py`) assert marker contents and admission order. They must pass **unchanged**
— if any of them needs editing, the change has leaked past display derivation into admission behavior, which is out of
scope.

### Visual snapshots

`tests/ace/tui/visual/_ace_agents_png_snapshot_fixtures.py` (explicit `wait_runners=0` fixture ~line 299, mixed
explicit/implicit ladder ~line 366) and `_ace_agents_png_snapshot_clan_fixtures.py` (~line 212) will re-render. Run
`just test-visual`, inspect the actual/expected/diff artifacts under `.pytest_cache/sase-visual/` to confirm each diff
is the intended relabel and nothing else, then accept with `--sase-update-visual-snapshots`.

## Verification

1. `just install` first (workspace virtualenvs go stale), then `just check`.
2. `just test-visual`, review diffs, accept intentionally.
3. `rg -n "globally_queued|global_cap_queue_count"` returns nothing.
4. `rg -ni "explicit.*(remain|keep|stay).*WAITING|excluded from .Q."` over `docs/` and `src/sase/xprompts/skills/`
   returns nothing.
5. Manual check in `sase ace` against the reported case: a clan of runner-parked agents shows `QUEUED` rows whose ranks
   match the `QUEUE` ladder, and the header reads `[R/L · Q queued]` with `Q` equal to the ladder length.
