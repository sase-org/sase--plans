---
tier: tale
title: Stop bead-less family members from permanently wedging sase bead work
goal:
  A `sase bead work` retry no longer aborts when a phase family contains members that
  carry no bead metadata (monitor `--next` follow-ups and their monitors), the
  association guard still refuses to touch genuinely unrelated agents, and when it does
  refuse it reports every blocker at once with the artifact path and a concrete next
  step instead of a single opaque line.
size: medium
proposed_by: bbugyi200.athena.04o
create_time: 2026-08-17 08:35:35
status: wip
---

# Stop bead-less family members from permanently wedging `sase bead work`

## Symptom

`sase bead work ns.6.6.6 -Y` fails, and keeps failing identically on every retry:

```
❯ sase bead work ns.6.6.6 -Y
Epic sase-ns.6.6.6 is already ready; retrying remaining non-closed phases.
Epic sase-ns.6.6.6 — Task backlog top five ...: 1 phase agent(s) in 1 wave(s) plus 1 land agent (sase-ns.6.6.6.land).
  Clan: sase-ns.6.6.6 · Tribe: @epic
  Wave 0: sase-ns.6.6.6.1 → sase-ns.6.6.6.1
  Land waits on: sase-ns.6.6.6.1
Error: agent owner 'sase-ns.6.6.6.1--1' is not associated with expected bead sase-ns.6.6.6.1; observed bead association(s): none
```

The epic is wedged: the offending agents are durable artifacts, so every subsequent
retry aborts at the same point, and the CLI offers no override, no diagnosis, and no
remediation. `--dry-run` does not help — the abort happens inside the preview, before
anything is rendered.

## Reproduction (confirmed against live state, not hypothesized)

The failing check is `_require_record_bead` in
`src/sase/bead/cli_work_cleanup_targets.py:304`. Reproduced in-process against the real
registry:

```
$ .venv/bin/python -c "
from sase.bead.cli_work_cleanup_selection import select_bead_work_launch
from sase.bead.cli_work_cleanup_types import BeadWorkSlot
slot = BeadWorkSlot(slot_id='sase-ns.6.6.6.1', owner_name='sase-ns.6.6.6.1',
                    expected_bead_id='sase-ns.6.6.6.1', launch_name='sase-ns.6.6.6.1')
select_bead_work_launch(slots=(slot,), bead_assignees={})"
ForcedReuseCleanupError: agent owner 'sase-ns.6.6.6.1--1' is not associated with expected bead sase-ns.6.6.6.1; observed bead association(s): none
```

The family behind the slot has eight members, five of which carry no bead attribution at
all:

```
sase-ns.6.6.6.1--plan     role=root     bead_ids=['sase-ns.6.6.6.1', 'sase-ns.6.6.6.1', 'sase-ns.6.6.6']
sase-ns.6.6.6.1--code     role=code     bead_ids=['sase-ns.6.6.6.1', 'sase-ns.6.6.6.1', 'sase-ns.6.6.6']
sase-ns.6.6.6.1--mon      role=monitor  bead_ids=['sase-ns.6.6.6.1', 'sase-ns.6.6.6.1', 'sase-ns.6.6.6']
sase-ns.6.6.6.1--1        role=code     bead_ids=[]
sase-ns.6.6.6.1--mon-0    role=monitor  bead_ids=[]
sase-ns.6.6.6.1--2        role=code     bead_ids=[]
sase-ns.6.6.6.1--mon-1    role=monitor  bead_ids=[]
sase-ns.6.6.6.1--3        role=code     bead_ids=[]
```

`--1` is simply the first offender in scan order, which is why the error names it.

## Root cause

Two correct behaviors combine into a wedge.

**1. Nested family-attach launches deliberately carry no bead attribution.**

`sase bead work` launches the phase agent with `SASE_EPIC_BEAD_ID` /
`SASE_PHASE_BEAD_ID` in its environment. `epic_work_metadata_from_env()`
(`src/sase/axe/run_agent_directive_metadata.py:121`) **pops** those variables into
`agent_meta.json`, with an explicit rationale:

> These values describe only the current child. Popping them prevents a phase or land
> agent from accidentally attributing its own nested launches to the same epic role.

`_add_family_metadata()` (same file, line 282) then propagates `agent_family`,
`agent_family_role`, `agent_clan`, `parent_timestamp`, `cl_name`, and the workspace pair
to a family-attach child — but deliberately not `bead_id`, `phase_bead_id`, or
`epic_bead_id`.

So when the phase agent `sase-ns.6.6.6.1--code` handed `just check-full` to a monitor
with a `--next` action, the monitor's follow-up agent launched as `sase-ns.6.6.6.1--1` —
a real member of the phase family, joined to the epic clan, in the same workspace and
patch — with an empty bead association. Its own monitors (`--mon-0`, `--mon-1`)
inherited the same emptiness. Confirmed in the artifacts: `--1` has
`agent_family=sase-ns.6.6.6.1`, `agent_clan=sase-ns.6.6.6`,
`wait_for=['sase-ns.6.6.6.1--code']`,
`monitor_member_agent_name=sase-ns.6.6.6.1--mon-0`, and no bead fields.

This is working as designed and must not change. Reversing it would misattribute every
nested launch to the epic role across ACE bead views, `agents_sync`, and the commit
workflow.

**2. The relaunch classifier requires bead attribution from every family member.**

`classify_slot_owner()` sees `container_kind == "family"` and calls
`_classify_family_owner()`, which finds members by matching each record's `agent_family`
(or `workflow_name` when `agent_family_role` is set) against the slot's owner name — see
`load_agent_owner_view()` at `cli_work_cleanup_targets.py:31`. It then runs every member
through `_classify_artifact_record()` → `_require_record_bead()`, which demands that
`slot.expected_bead_id` appear in the record's `{bead_id, phase_bead_id, epic_bead_id}`
and raises `ForcedReuseCleanupError` otherwise.

That guard is over-applied. Its real purpose — _do not kill an agent that is not the one
this slot names_ — is already satisfied structurally on the container path: the member
was **found by** its `agent_family` matching the slot owner. The bead check adds nothing
there, and produces a false negative for every by-design bead-less member. On the direct
path (`_record_for_owner()`, which dereferences an arbitrary `artifacts_dir` from the
registry) the guard is genuinely load-bearing and must stay.

Corroborating evidence that the strict check was over-generalized: the older
`preview_legacy_bead_work_force_reuse()` (`src/sase/bead/cli_work_legacy_preview.py:49`)
wipes family members with no bead check at all.

**3. Any single bad member aborts everything, unrecoverably.**

`_classify_family_owner()` builds `member_targets` with a tuple comprehension, so the
first offender raises and the remaining seven members are never classified.
`select_bead_work_launch()` does not catch it, `preview_bead_work_launch_selection()`
does not catch it, and `handle_bead_work()` converts it straight to `BeadWorkError` →
`exit 1`. `--dry-run` dies at the same point, so the operator cannot even see the
picture. `_record_current_state()` (line 349) and `_classify_clan_owner()` (line 180)
have the same one-shot-abort shape.

**Scope note:** the same module backs `sase bead work <task>` via `_task_work_slots()`
(`src/sase/bead/cli_work_task.py:65`), so a task agent that starts a monitor with
`--next` wedges its own task bead the same way. One fix covers both.

## Fix

### 1. Distinguish "no association" from "conflicting association"

In `src/sase/bead/cli_work_cleanup_targets.py`, replace `_require_record_bead()` with an
association check that takes how the record was reached:

```python
type _OwnerMembership = Literal["registry", "family", "clan"]
```

Add a lineage helper — bead IDs are dotted, so an epic ID is an ancestor of its phase
IDs:

```python
def _bead_ids_related(left: str, right: str) -> bool:
    return left == right or left.startswith(f"{right}.") or right.startswith(f"{left}.")
```

Then `_require_record_association(slot, record, *, owner_name, membership)`:

- `slot.expected_bead_id` in the record's observed IDs → accept (unchanged).
- Any observed ID related to `slot.expected_bead_id` by `_bead_ids_related` → accept.
  This covers a member that recorded only `epic_bead_id` for a phase slot.
- Observed IDs empty and `membership` is `"family"` or `"clan"` → **accept**; container
  membership is the association. This is the fix for the reported failure.
- Observed IDs empty and `membership` is `"registry"` → accept only when the record's
  `agent_meta.name` matches `owner_name` under
  `current_owner_agent_name_key(..., view.identity)`; otherwise block. This keeps the
  direct path's real invariant (the registry entry must point at the agent it names)
  while no longer relying on bead metadata to express it.
- Observed IDs non-empty and all unrelated → **block**, naming the conflicting IDs. This
  is the case the guard actually exists for and it stays fatal.

Thread `membership` through `_classify_artifact_record()`; `_classify_family_owner()`
passes `"family"`, `_classify_clan_owner()` passes `"clan"` (if it ever classifies
records), and `classify_slot_owner()`'s direct branch passes `"registry"`.

When a target is accepted with no bead metadata, say so in its `detail` so the preview
stays honest, e.g.:

```
REMOVE   (FAILED) sase-ns.6.6.6.1--1 bead=sase-ns.6.6.6.1  for bead sase-ns.6.6.6.1 at /home/.../20260817065032 (no bead metadata; matched by family membership)
```

### 2. Report every blocker at once instead of aborting on the first

Add a non-destructive `BLOCKED` action so classification failures become data:

- `cli_work_cleanup_types.py`: extend `CleanupAction` with `"BLOCKED"`; add
  `CleanupTarget.blocked` (`action == "BLOCKED"`). `destructive` and `preserved` must
  both stay `False` for it. Add `BeadWorkLaunchSelection.blocked_targets`.
- `cli_work_cleanup_selection.py`: in `select_bead_work_launch()`, wrap the per-slot
  `classify_slot_owner()` call so a `ForcedReuseCleanupError` becomes a `BLOCKED` target
  carrying the exception text as `detail`, instead of propagating. A slot with a blocker
  contributes no launch name and no destructive target.
- `_classify_family_owner()`: classify each member independently and collect per-member
  blockers, so all five offending members in the reported case surface in one run rather
  than one per retry.
- `_record_current_state()`'s "unknown live state" raise flows through the same
  collection and becomes a blocker.

Safety is unchanged, only the reporting: **any blocker still aborts a real launch before
any bead mutation, agent wipe, or spawn.**

- `cli_work_handler.py`: after `preview_bead_work_launch_selection()`, if
  `selection.blocked_targets` and not `dry_run`, call `render_cleanup_preview()` and
  then raise `BeadWorkError` with an aggregate message that contains **each blocker's
  reason text verbatim** (the existing populated-clan test asserts on that wording),
  followed by remediation guidance.
- `cli_work_task.py`: same treatment against `TaskBeadWorkError`.
- Under `--dry-run`, render the `BLOCKED` rows, print a line stating how many blockers
  would abort a real launch, and keep the existing exit code (`0`) and
  `launch_state="dry_run"` result so scripted previews do not change contract.
- `cli_work_cleanup_apply.py`: `revalidate_bead_work_launch_selection()` must raise if
  the rescan produces any blocked target, so a blocker appearing between preview and
  wipe cannot be ignored.

### 3. Make the message actionable

The current text tells the operator nothing to do. Each blocker should name the agent,
its artifact directory, and the family/slot it was reached through; the aggregate error
should end with a concrete next step — rerun with `--dry-run` for the full picture, and
dismiss the listed agent(s) in `sase ace` before retrying. Do not invent a CLI
subcommand: there is no `sase agent dismiss`; `sase agent` exposes
`archive/artifacts/index/kill/list/names/...`, and dismissal is an ACE action.

### 4. Preview rendering

`render_cleanup_preview()` and `render_task_cleanup_preview()`
(`src/sase/bead/cli_work_plan.py:87` and `:107`) both sort by a literal
`action_order = {"PRESERVE": 0, "KILL": 1, "REMOVE": 2, "RELEASE": 3}` and will
`KeyError` on the new action. Add `"BLOCKED"` to both dicts, sorted first so blockers
lead the preview.

## Explicitly out of scope

- **Do not** propagate `bead_id`/`phase_bead_id`/`epic_bead_id` to family-attach
  children. `epic_work_metadata_from_env()`'s popping is intentional and load-bearing;
  changing it would misattribute nested launches across ACE, `agents_sync`, and the
  commit workflow. The fix belongs in the classifier, which is the layer that already
  knows the structural association.
- No new CLI flags or subcommands, and no feature flag: this repairs a hard-failing gate
  rather than routing new user-facing behavior, so per `sase/memory/sase_flags.md` it is
  a config/flag non-candidate.

## Tests

`tests/test_bead/cli_work_helpers.py` — make `write_bead_agent_meta()`'s `bead_id`
parameter accept `None` and, when it is `None`, omit `bead_id`, `phase_bead_id`, and
`epic_bead_id` entirely. Today it always writes all three, which is precisely why no
existing test caught this.

Add to `tests/test_bead/test_cli_work_epic_launch_cleanup.py` (or a sibling module):

1. **Regression for the reported failure.** Family `<phase>` with `--plan` and `--code`
   carrying bead IDs plus `--1` and `--mon-0` carrying none. `sase bead work <epic> -Y`
   succeeds, classifies all four members, and launches the phase agent. Assert the
   `not associated with expected bead` text does **not** appear.
2. **Conflicting association still blocks.** A member whose `bead_id` belongs to an
   unrelated epic yields a `BLOCKED` target, exit 1, no launch, no bead mutation, and a
   message naming both the member and the conflicting ID.
3. **Ancestor ID accepted.** A member carrying only `epic_bead_id` under a phase slot is
   accepted via `_bead_ids_related`.
4. **All blockers reported in one run.** Two unrelated-bead members both appear in
   stderr from a single invocation.
5. **`--dry-run` renders blockers.** `BLOCKED` rows print, exit code stays 0, and
   nothing is mutated.
6. **Direct-path guard preserved.** A registry owner whose artifact record has a
   different `agent_meta.name` and no bead IDs still blocks — the guard's real purpose
   survives the relaxation.
7. **Task path.** `sase bead work <task> -Y` with a bead-less family member under the
   task name succeeds (covers `_task_work_slots`).
8. **Unit level.** `select_bead_work_launch()` returns blocked targets rather than
   raising; `revalidate_bead_work_launch_selection()` raises when a blocker appears
   after preview.

The existing `test_work_expected_name_container_conflict_aborts_before_mutation` and
`test_work_family_cleanup_failure_aborts_before_mutation` must keep passing unchanged —
verify the populated-clan message still reaches stderr verbatim through the new
aggregate error.

## Verification

1. `just install` first — this repo's workspaces are ephemeral.
2. `.venv/bin/python -m pytest tests/test_bead -q` for the focused suite.
3. `just check` for the whole-repo gates plus the scoped test lane. If it runs long,
   hand it to `/sase_monitor` with a `--next` action.
4. **Live acceptance check.** Re-run the read-only reproduction from the Reproduction
   section against the real registry. It must now return classified targets — the five
   bead-less members as `REMOVE`/`KILL` under the family slot — instead of raising:

   ```
   .venv/bin/python -c "
   from sase.bead.cli_work_cleanup_selection import select_bead_work_launch
   from sase.bead.cli_work_cleanup_types import BeadWorkSlot
   slot = BeadWorkSlot(slot_id='sase-ns.6.6.6.1', owner_name='sase-ns.6.6.6.1',
                       expected_bead_id='sase-ns.6.6.6.1', launch_name='sase-ns.6.6.6.1')
   for t in select_bead_work_launch(slots=(slot,), bead_assignees={}).targets:
       print(t.action, t.name, t.current_state)"
   ```

   Then confirm `sase bead work ns.6.6.6 --dry-run` renders a full preview rather than
   erroring. Do not run the destructive form as verification.

## Files

- `src/sase/bead/cli_work_cleanup_targets.py` — association check, membership plumbing,
  per-member blocker collection (primary change)
- `src/sase/bead/cli_work_cleanup_types.py` — `BLOCKED` action, `blocked` /
  `blocked_targets`
- `src/sase/bead/cli_work_cleanup_selection.py` — collect blockers instead of raising
- `src/sase/bead/cli_work_cleanup_apply.py` — refuse a post-preview blocker
- `src/sase/bead/cli_work_plan.py` — `action_order` entries for `BLOCKED`
- `src/sase/bead/cli_work_handler.py` — aggregate blocker error for epic launches
- `src/sase/bead/cli_work_task.py` — aggregate blocker error for task launches
- `tests/test_bead/cli_work_helpers.py` — optional `bead_id` in the meta writer
- `tests/test_bead/test_cli_work_epic_launch_cleanup.py` — regression and guard tests
