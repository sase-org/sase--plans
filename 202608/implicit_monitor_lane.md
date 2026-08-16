---
tier: tale
size: medium
title: Resolve the implicit `sase monitor start` caller from its own artifacts
goal:
  An agent inside a promoted agent family or an epic phase lane can run `sase monitor
  start` with no `--agent`/`--lane` and get a monitor attached to its own lane,
  workspace, and durable family, with regression coverage for both reported failure
  shapes and docs that match the behavior.
proposed_by: bbugyi200.athena.sase-ns.1
bead: sase-ns.1
create_time: 2026-08-16 17:28:00
status: done
---

- **PARENT:** [202608/top_task_bead_sweep.md](top_task_bead_sweep.md)
- **BEAD:**
  [sase-ns.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ns/sase-ns.1.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-ns.1](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-ns.1.md)
- **COMMITS:**
  - [2605324](https://github.com/sase-org/sase/commit/2605324cb2c47e43809de822ae78db120905faa2)
    — fix(monitor): resolve implicit start/show/stop caller from its own artifacts

# Plan

Fix the implicit-lane derivation behind `sase monitor start` so an agent that has been
promoted to an agent family — which every agent that ran `/sase_plan` or started a
previous monitor has been — can hand a command to `/sase_monitor` with no explicit
`--agent`/`--lane`. This closes task bead **`sase-ll`** and phase bead **`sase-ns.1`**
(epic `sase-ns`, plan `202608/top_task_bead_sweep.md`).

## Why this bead keeps coming back

`sase-ll` has 9 independent reporters and has been closed and reopened twice. It has had
two distinct failure shapes, and the second one is a regression introduced by the fix
for the first:

1. **`FamilyAttachError` (fixed 2026-08-15, do not regress).** From an epic phase lane,
   implicit start collapsed `SASE_AGENT_NAME` through `agent_family_base()` to a
   family-wide lane and then picked that lane's _newest_ member — a sibling phase —
   producing
   `Cannot create agent family 'sase-ky': resolved parent is named 'sase-ky.5'`.
   Reported by `sase-ku.10`, `sase-ky.4`/`sase-ky.5`, `sase-lz.4`, `sase-m6.3`,
   `sase-mc.4`. A related silent variant (reporter `02q`) selected a newer settled
   monitor member `02i--mon-6` instead of the caller `02i--code`, so `just check-full`
   ran in the primary checkout instead of the caller's workspace and the follow-up
   `#fork:` pointed at a member with no chat history.

   That was fixed by pinning implicit start to the caller's _exact_ `SASE_AGENT_NAME`
   (`store.default_caller()` + `store.resolve_exact_agent()`).

2. **`no agent artifacts found for agent '<name>'` (live today).** The exact-name pin is
   too strict. Reported after that fix by `sase-m6.6.1.land`, `sase-m6.8`, and `046`.

## Verified root cause of the live failure

`SASE_AGENT_NAME` names the caller's _sase agent_, which for a promoted family is the
family container. The concrete shell that is actually running carries the member name in
its own `agent_meta.json` — `<base>--plan`, `<base>--code`, `<base>--1`. This is already
documented in `sase.agent.identity.resolve_local_agent_name()`:

> family members can replace one another inside a single process, leaving
> `SASE_AGENT_NAME` set to the family/container while this run's `agent_meta.json`
> carries the concrete agent shell.

`store.resolve_exact_agent()` matches only `agent_meta.name == caller`, so no record
matches a family container and `MonitorLaneError` is raised before launch.

Reproduced against the live artifact store on clean master (`83e2ceea6`), with the
project's real records:

```
exact  046               -> ERR no agent artifacts found for agent '046' ...
lane   046               -> 046--code   .../202608/16/20260816151733
exact  sase-m6.8         -> ERR no agent artifacts found for agent 'sase-m6.8' ...
lane   sase-m6.8         -> sase-m6.8--code
exact  sase-m6.6.1.land  -> ERR no agent artifacts found for agent 'sase-m6.6.1.land' ...
lane   sase-m6.6.1.land  -> sase-m6.6.1.land--code
```

The on-disk metadata for each of those reporters confirms the shape — for example
`046--plan` (`agent_family: "046"`, role `root`) and `046--code` (`agent_family: "046"`,
role `code`), with no record named `046` at all.

So the two fixes must be combined rather than traded against each other:

- never collapse the caller's name _upward_ into a broader family (shape 1), and
- do resolve _downward_ into the caller's own concrete member (shape 2),
- while never selecting a monitor member as the parent (the `02q` variant).

## Design

Resolve the implicit caller **metadata-first**, mirroring `resolve_local_agent_name()`:

1. **The caller's own artifacts dir.** `SASE_ARTIFACTS_DIR` is set for every agent shell
   and points at exactly the record that is running. This is the most precise identity
   available and needs no name reasoning at all.
2. **An exact `agent_meta.name` match** for `SASE_AGENT_NAME` (bare agents, and callers
   whose env already carries the member name, e.g. `02i--code`).
3. **The newest non-monitor member of the caller's _own_ family** — records whose
   `agent_meta.agent_family` equals the caller exactly. This restores what the pre-fix
   family lookup got right for `046` → `046--code` without ever matching a sibling,
   because the match is on the durable `agent_family` field and never on a name
   collapsed through `agent_family_base()`.
4. Otherwise a clear, actionable error naming `-a/--agent`.

Monitor members are excluded at step 3 (`sase.monitor.models.is_monitor_member_record`),
which is what keeps the `02q` variant fixed: a settled `--mon` row is usually the newest
member of the family, and inheriting from it drags in `workspace_num=0` and the primary
checkout.

## Implementation

### 1. `src/sase/monitor/store.py`

Add the caller resolver next to `default_caller()`:

```python
def caller_artifacts_dir(env: Mapping[str, str] | None = None) -> str | None:
    """Return the calling agent shell's own artifacts dir, if the env names one."""


def resolve_caller_agent(
    project_name: str,
    caller: str,
    *,
    artifacts_dir: str | None = None,
) -> LaneContext:
    """Resolve the artifact record of the agent shell calling right now."""
```

- Scan `_project_records(project_name)` once and apply the four steps above in order.
- Step 1 compares `record.artifact_dir` to _artifacts_dir_ as normalized paths
  (`Path(...).expanduser().resolve(strict=False)`, trailing slash tolerated), and
  accepts the pinned record only when it belongs to the caller: `meta.name == caller`,
  or `meta.agent_family == caller`, or `agent_family_base(meta.name) == caller`. A stale
  or foreign `SASE_ARTIFACTS_DIR` therefore falls through to steps 2–3 instead of
  hijacking the start.
- Step 3 matches `meta.agent_family == caller` exactly — no `agent_family_base()` on
  either side — and skips `is_monitor_member_record(record)` rows. Ties break on newest
  `record.timestamp`, same as the existing resolvers.
- The final `MonitorLaneError` must name the flag to pass and, when there are near-miss
  records (a name that starts with the caller, or an `agent_family` equal to it), list
  up to five of them newest-first. Shape:
  `no artifacts found for the calling agent 'X' in project 'P'; pass -a/--agent explicitly (nearest artifacts: X--plan, X--code)`.
- Keep `resolve_exact_agent()` and `resolve_lane()` as they are: both remain in use
  (`resolve_lane()` still backs explicit `--agent`, which must keep selecting the newest
  member of the named lane) and both stay covered by their existing tests.
- Export the new names from `store.__all__` and from `src/sase/monitor/__init__.py`.

**Delete `default_lane()`** together with its `__all__`/`__init__` exports and its test
(`test_default_lane_still_collapses_family_and_legacy_suffixes`) once step 3 below
replaces its last production caller. Its `agent_family_base()` collapse is precisely the
defect this bead is about, and Symvision will flag it as unused otherwise. Do not
silence that with a pragma. (If some caller outside this plan's files still needs it,
keep it and say so in the bead note instead.)

### 2. `src/sase/monitor/start.py`

- `_StartIdentity` currently carries `exact: bool` and is resolved twice — once in
  `_resolve_start_identity()` (before the lane lock) and again in
  `_resolve_lane_start()` (under it). Replace `exact` with the resolved
  `context: store.LaneContext | None`, set for implicit starts only, and have
  `_resolve_lane_start()` reuse it instead of re-resolving. `_project_records()` is a
  full-history index query; one scan per start is enough.
- `_resolve_start_identity()` for an implicit start becomes:

  ```python
  caller = store.default_caller()
  if not caller:
      raise MonitorLaneError(...)  # unchanged message
  ctx = store.resolve_caller_agent(
      request.project_name, caller, artifacts_dir=store.caller_artifacts_dir()
  )
  target = resolved record's own agent_meta.name, falling back to caller
  lock_lane = store.durable_lane_for_record(ctx.record, fallback=target)
  ```

- Setting `target` to the resolved record's **own name** matters:
  `_resolve_lane_start()` passes it to
  `promote_agent_to_family(selected.artifact_dir, identity.target)` when the record has
  no `agent_family` yet, and that function raises `FamilyAttachError` whenever the base
  name it is handed differs from the record's own `name`. Deriving the base from the
  record makes that mismatch structurally impossible on the implicit path.
- The explicit `--agent`/`--lane` path keeps passing the user's string to
  `promote_agent_to_family()` and keeps failing loudly on a mismatch: an explicit target
  that resolves to a differently-named record is a real user error, not something to
  silently reinterpret.
- Behavior that must not change: the durable lane still comes from the selected record's
  `agent_family` metadata; the lane lock, replay lookup, conflict detection, and request
  fingerprint all still key on that durable lane.

### 3. `src/sase/main/monitor_handler.py`

- `_agent_workspace_dir()` (used by `_resolve_cwd()` when no `-C/--cwd` is given) must
  use `resolve_caller_agent()` on the implicit path and `resolve_lane()` on the explicit
  one. Today it fails the same way and silently degrades to `Path.cwd()`, which is how a
  monitor ends up running against the wrong checkout.
- `_resolve_ref_or_active()` (implicit `sase monitor show`/`stop` with no ref) currently
  calls `default_lane()`, which collapses `sase-m6.6.1.5` to `sase-m6.6.1` and can
  resolve **another agent's** active monitor — i.e. `sase monitor stop` with no ref can
  stop a parent's or sibling's monitor. Replace it with `resolve_caller_agent()` plus
  `durable_lane_for_record(record, fallback=caller)`. When the caller has no artifacts,
  raise a `MonitorRefError` telling the caller to pass an explicit id; do **not** fall
  back to the collapsed lane.

### 4. Docs and skill source

- `docs/monitors.md`: under "Starting a monitor", document how the implicit agent is
  resolved (own artifacts dir → exact name → newest non-monitor member of its own
  family), that `--agent` is only needed outside an agent or to target a different
  agent, and that an unresolvable caller is a clear error naming `-a/--agent`. Note that
  `show`/`stop` with no ref resolve against the caller's own durable family.
- `src/sase/xprompts/skills/sase_monitor.md`: keep the guidance that no `--agent` is
  needed inside an agent, and state explicitly that this holds inside an epic phase lane
  and inside a promoted agent family. Read `sase memory read generated_skills.md` first
  and follow whatever regeneration/deploy step that memory requires after editing the
  skill source.
- Do not edit `sase/memory/*.md`, `AGENTS.md`, or the generated provider instruction
  shims (`CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`). No user permission for
  that exists in this chain.

## Tests

`tests/_conftest_environment.py` already scrubs `SASE_AGENT_NAME` and
`SASE_ARTIFACTS_DIR` from every test, so tests set them explicitly with
`monkeypatch.setenv`. Use the existing `tests/monitor/_fixtures.py` helpers
(`make_starter_agent`, `patch_project_records`, `write_project_file`, `wait_for_done`).

Add to `tests/monitor/test_monitor_store.py`:

- the caller's own `SASE_ARTIFACTS_DIR` wins over a newer non-monitor member of the same
  family;
- a caller that is a family container (`046`, no record of its own) resolves to the
  newest non-monitor member (`046--code`), not to a newer settled `046--mon-6` and not
  to any record outside `agent_family == '046'`;
- an exact-name record still wins for `02i--code` and for a bare numeric phase name
  (`sase-m6.6.1.5`), and neither resolves to a sibling (`sase-m6.6.1`) or land
  (`sase-m6.10`) record;
- a foreign/stale `SASE_ARTIFACTS_DIR` is ignored and resolution falls through to the
  name/family steps;
- the unresolvable case raises `MonitorLaneError` whose message names `-a/--agent`.

Add to `tests/monitor/test_monitor_start.py` (these are the acceptance criteria from the
bead's `+1 EVIDENCE`):

- `test_implicit_start_from_a_promoted_family_container_pins_the_live_member`:
  `SASE_AGENT_NAME='046'` with `046--plan` (done), `046--code` (live, workspace 12) and
  a newer settled `046--mon-6` (workspace 0). Assert the start succeeds,
  `record.lane == '046'`, and the monitor member's meta inherits `parent_timestamp` of
  the `--code` row, `workspace_num == 12`, the caller's workspace dir, and the caller's
  model.
- `test_implicit_start_pins_the_callers_artifacts_dir_over_a_newer_member`: same family
  with `SASE_ARTIFACTS_DIR` pointing at the older member; assert the parent is the
  pinned row.
- Keep `test_implicit_start_pins_numeric_phase_caller_not_sibling_or_land` and
  `test_implicit_start_pins_family_member_not_newer_settled_monitor` passing unchanged —
  they are shape 1's regression tests.

Add to `tests/main/test_monitor_handler_start.py`: an implicit start by a
family-container caller with no `-C/--cwd` derives the cwd from the caller's own
member's `workspace_dir`.

Add to `tests/main/test_monitor_handler_stop.py` (or the matching show module): a caller
whose durable family is `sase-m6.6.1.5` resolves its _own_ running monitor and not a
running monitor belonging to family `sase-m6.6.1`.

## Verification

1. `just install` first — workspaces are ephemeral and dependencies drift.
2. Focused first:
   `.venv/bin/python -m pytest tests/monitor tests/main/test_monitor_handler_start.py tests/main/test_monitor_handler_stop.py tests/main/test_monitor_handler_show.py -q -p no:randomly`
   (drop any of those paths that does not exist).
3. `just check` (whole-repo lint gates + the diff-scoped test lane).
4. **Live acceptance test — required.** The `sase` on `PATH` is a uv tool editable
   install pointing at the _primary_ checkout, so it does **not** pick up this
   workspace's source. Prove the fix with the workspace CLI instead, from the workspace
   root:

   ```bash
   .venv/bin/sase monitor start \
     -r 'Exhaustive verification for the implicit monitor lane fix' \
     -t 60m --next-output tail \
     -n '<closing instructions, see "Bead bookkeeping" below>' \
     -- just check-full
   ```

   Pass **no** `--agent`/`--lane` and **no** `-C/--cwd`: the implicit resolution and the
   implicit cwd derivation are exactly what is under test. By then the implementing
   agent is itself a promoted family member, so this invocation is the reported failure
   reproduced end to end. If it still fails, that is the fix not working — record the
   exact error and iterate rather than working around it with an explicit `--agent`.

   `just check-full` outruns a single agent turn and must never be run inline. The
   monitor hands the turn off, so everything after this point happens in the follow-up
   agent.

5. If `just check`'s scoped run escalates or the change touches the broadening set, the
   monitor-backed `just check-full` above already satisfies that requirement.

## Bead bookkeeping

Per the epic plan's rules, and carried in the monitor's `--next` action because the
start kills the current turn:

1. Claim the task bead before implementing: `sase bead update sase-ll -s in_progress`.
   (`sase-ns.1` is already `in_progress`; never set a phase bead's status by hand.)
2. Leave a note on `sase-ll` describing what changed, the commands run, and the
   evidence: `sase bead note sase-ll '<...>'`. The note must state plainly whether the
   live acceptance test in step 4 passed.
3. Close both beads once verified:
   `sase bead close sase-ll --note "<what you verified>"` and
   `sase bead close sase-ns.1 --note "<what you verified>"`.
4. Do **not** close the parent epic `sase-ns` or any ancestor plan bead, and do not
   create task beads. Record anything discovered as
   `sase bead note sase-ns.1 'PROPOSED FOLLOW-UP: <one-line summary — detail>'` for the
   epic's land agent to triage.
5. If any part cannot be finished, leave the beads open with a note that says plainly
   what was tried, what was found, and what the next agent should pick up.

## Out of scope

- `store._record_in_lane()`'s `agent_family_base(meta.name) == lane` clause, which is
  why an explicit `--agent sase-ku` can still match a numeric phase `sase-ku.4`. No
  reporter hit this on an explicit target, and changing it would move explicit-lane
  semantics that `test_explicit_family_target_still_selects_newest_lane_member` pins. If
  the implementation shows this is reachable from the implicit path after the change,
  record it as a `PROPOSED FOLLOW-UP:` instead of widening the fix.
- Falling back to `agent_meta.json`'s name when `SASE_AGENT_NAME` is unset entirely
  (what `discover_agent_identity()` does). No reporter hit it; the current "pass
  -a/--agent explicitly" usage error is correct for that case.
- The claim-transfer defect tracked by `sase-lj`, which surfaces after this one is
  worked around with an explicit `--lane`.
