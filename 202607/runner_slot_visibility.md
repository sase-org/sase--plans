---
tier: tale
title: Surface runner-slot-holding family children in agent listings
goal: 'Every agent that the runner-slot admission layer counts as occupying a slot
  is visible in `sase agent list` and counted in every surfaced runner_slots_in_use
  value (CLI, integrations, and ACE TUI), so the system never again looks idle while
  a hidden epic-phase child holds the only claimed slot.

  '
create_time: 2026-07-17 18:47:44
status: done
prompt: 202607/prompts/runner_slot_visibility.md
---

# Plan: Surface runner-slot-holding family children in agent listings

## Problem and root-cause diagnosis

Bryan observed "no sase agents running" while ~22 agents sat WAITING, some with all of their `%wait:` dependencies
already completed. Investigation showed the scheduler is behaving correctly and the system is **not** stuck — the
observability layers are reporting a false picture:

1. An agent **was** running the whole time: `sase-6n.2`, a parallel epic-phase family child (`agent_meta.json` has
   `parent_timestamp` set and `agent_family_parallel: true`). Family children intentionally bypass the runner-slot queue
   (`wait_for_runner_slot` in `src/sase/axe/run_agent_wait.py` returns `claim()` immediately when `parent_timestamp` is
   set) so an epic can hand its slot from phase to phase, and they **do** count toward occupancy in the admission layer
   (`_is_runner_slot_user_agent_record` in `src/sase/core/runner_slots/_admission.py` includes parallel children).
2. The WAITING agents whose dependencies were complete were all `%wait(runners=0)` waiters (run-only-when-idle
   semantics, confirmed in `resolve_wait_runners_args`). They were correctly parked because occupancy was ≥ 1
   essentially all day (epic phases, hourly audit agents, interactive agents). The remaining WAITING agents were
   dependency-blocked on genuinely unfinished agents (`sase-6n` waits on phases that have not finished;
   `sase-6n.w0`/`sase-6n.f0` wait on `sase-6n`; the `split_file.*` chain waits on its head, itself a `runners=0`
   waiter).
3. Every user-facing surface hid the slot holder and undercounted occupancy:
   - `list_running_agents` (`src/sase/agent/running_listing.py`) filters rows with `is_root_user_agent_record`, which
     drops **all** records with `parent_timestamp` — so the running phase child never appears in `sase agent list`,
     which therefore showed zero RUNNING rows.
   - `agent_list_entries` (`src/sase/integrations/agent_list_entries.py`) computes `runner_slots_in_use` by summing
     `holds_runner_slot` over those root-only rows — it reported 1 slot in use when the admission layer counted 2 (and 0
     when the admission layer counted 1), directly contradicting the scheduler the waiters actually obey.
   - The ACE TUI slot context (`refresh_runner_slot_context` in `src/sase/ace/tui/models/agent_runner_slots.py`) also
     counts only non-child rows (`_is_ace_run_root` rejects `is_child_row`), so TUI waiter rows show the same wrong
     count. Family child rows are additionally folded by default on the Agents tab, so nothing on screen says "running".

The listing layers and the admission layer each reimplement "who holds a runner slot" and disagree. The admission layer
is the source of truth.

## Design

Establish and test one invariant:

> Every live record counted by `running_root_agent_count` (the admission layer's occupancy) appears as a row in
> `list_running_agents` with `holds_runner_slot=True`, and every surfaced `runner_slots_in_use` equals the number of
> such rows.

No scheduler/admission behavior changes. All changes are projection-side.

### 1. Make the admission predicate the shared source of truth

In `src/sase/core/runner_slots/` promote the private predicate to a public export (e.g.
`is_runner_slot_user_agent_record`) alongside the existing `is_root_user_agent_record`, `running_root_agent_count`, and
`live_runner_slot_waiters` exports. Listing code must use this predicate rather than re-deriving "counts toward
occupancy" locally, so any future change to family/clan semantics (the in-flight sase-6n epic) automatically propagates
to every surface.

### 2. `sase agent list` shows slot-holding family children

In `src/sase/agent/running_listing.py`, extend `_running_from_snapshot` to also emit rows for live records that pass
`is_runner_slot_user_agent_record` but are not roots (i.e. parallel family children), when they either hold a slot
(`run_started_at` set, no pending question) or are queued for one (`waiting.slot_requested_at` set). Plain
dependency-waiting phase children stay hidden — their family root already renders the WAITING state. Reuse
`_running_info_from_running_record` (lift its unconditional `meta.parent_timestamp` skip into the root-only call path);
`workspace_num` resolution may return None for children whose claim is not in the project RUNNING field — that is
acceptable.

Completed children remain excluded from `-a` output (unchanged `_done_info_from_record`). A side effect worth keeping:
`sase agent show -n` / `kill -n` resolve running phase children by name.

### 3. Correct `runner_slots_in_use` in integrations

With slot-holding children now present as rows carrying accurate `holds_runner_slot`, the existing sum in
`agent_list_entries` becomes correct by construction. Additionally:

- Append `parent_agent_name` and `agent_family` to the `sase agent list -j` row payload (new keys at the end of the
  dict; existing keys untouched) so consumers can distinguish family-child rows.
- In `_attach_runner_slot_context`, attach the list of slot-holder names to waiting rows (new `runner_slot_holders`
  field in `_AgentWaitInfo` and the JSON payload) so a parked waiter row directly answers "blocked by whom?".

### 4. Correct the ACE TUI slot context

In `src/sase/ace/tui/models/agent_runner_slots.py`, align `refresh_runner_slot_context` with admission semantics: a row
counts as occupying (and may appear in the waiter queue) when it is either an ace-run root **or** a family-member child
row with `agent_family_parallel=True` (field already populated by both meta enrichment loaders). Rename the helper
(`_counts_as_running_root` → e.g. `_holds_runner_slot`) to match. Do **not** touch Agents-tab tree structure, folding,
or rendering — that is bead sase-6n.6's scope.

### 5. Documentation

Update the `sase_agents_status` skill source template (under `src/sase/xprompts/skills/`) to document that slot-holding
family children appear as rows and to mention the new `parent_agent_name`, `agent_family`, and `runner_slot_holders`
JSON fields. Read `sase/memory/generated_skills.md` via the long-memory read procedure before editing skill sources. No
new CLI subcommands or options are added.

## Testing

- `tests/test_running_agents_snapshot.py`: a live parallel family child with `run_started_at` appears in
  `list_running_agents` with `holds_runner_slot=True` while its root is WAITING; a non-parallel child (e.g. a `--code`
  helper) and a dependency-waiting parallel child stay hidden; a slot-queued parallel child (has `slot_requested_at`)
  appears.
- `tests/test_agent_list_entries.py`: `runner_slots_in_use` on waiter rows counts the running child; new JSON keys
  present; `runner_slot_holders` names the child.
- `tests/ace/tui/test_agent_runner_slots.py`: a parallel family child row with a run start time raises
  `runner_slots_in_use` for waiter rows; non-parallel child rows do not.
- `tests/test_runner_slots.py`: cover the newly public predicate export (parity cases: root, parallel child,
  non-parallel child, done-marker).
- Invariant test: for a synthetic snapshot mixing roots, parallel children, non-parallel children, and done records,
  assert `running_root_agent_count == sum(row.holds_runner_slot for row in list_running_agents rows)`.
- Run the full gate (`just check`) including the PNG visual suite; no visual goldens are expected to change since
  Agents-tab rendering is untouched.

## Non-goals and risks

- No change to admission/queue semantics: `%wait(runners=0)` still means "start only when nothing is running", family
  children still bypass the queue, and default-threshold roots may still overtake threshold-0 waiters. Long-term
  starvation of `runners=0` waiters on a busy machine is inherent to those semantics and is out of scope (worth a future
  bead if it bothers us).
- No Agents-tab tree/rendering changes (reserved for sase-6n.6) and no Rust `sase-core` changes: `agent_family_parallel`
  is already on the wire meta and the admission logic consumed here lives in Python `sase/core/runner_slots`.
- Conflict risk with the in-flight sase-6n epic (clans/families/tribes): keep every change keyed off the shared
  admission predicate and re-verify against master at implementation time; if clan records replace parallel families,
  the predicate — not the listing code — is where semantics will move.
- Symvision: promoting the private predicate to a public export must follow the symvision rules; consult
  `sase/memory/symvision.md` if lint flags the new export.
