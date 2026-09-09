---
tier: tale
title: Restore canonical plan identity through neutral plan gates
goal: "Approved plans keep their durable, uniquely-named plan files after passing
  through the sase-6e neutral gate bundles: SDD archives stop colliding on the literal
  name `plan.md`, `sase bead work <bundle plan>` archives and launches correctly, `sase
  plan list` shows durable ~/.sase/plans paths instead of ~/.sase/interaction_requests
  paths, the plans archive damaged by the collisions is repaired, and the approved
  custom-notification-gates epic is launched.

  "
create_time: 2026-09-09 19:53:04
status: wip
---

# Plan: Restore canonical plan identity through neutral plan gates

## Context and root cause

The sase-6e epic replaced legacy in-place plan review with neutral gate bundles under
`~/.sase/interaction_requests/<kind>/<request-id>/`. Gate creation
(`src/sase/plan_gate.py`) copies the proposed plan into the bundle as the fixed resource
name `plan.md` (`PLAN_RESOURCE_PATH`) and records the durable source path in the request
envelope as `payload.original_plan_file` (proposals are always archived to
`~/.sase/plans/<yyyymm>/<stem>.md` by `sase plan propose` before the gate is created).

After the gate resolves, every downstream consumer derives the plan's durable identity
from the reviewed file's path/basename, which is now always `plan.md`:

1. `handle_plan_approval` (`src/sase/llm_provider/_plan_utils.py`) returns
   `plan_file = <bundle>/plan.md`.
2. `handle_accepted_plan` (`src/sase/axe/run_agent_exec_plan_accept.py`) computes
   `sdd_plan_name = basename(plan_file)` → `"plan"`, so SDD prompt/plan files are
   written and committed as `<plans-root>/<yyyymm>/plan.md` and
   `<yyyymm>/prompts/plan.md`. Every gated plan overwrites the previous one.
3. Epic launches run `sase bead work <bundle>/plan.md`; `plan_archive_destination`
   (`src/sase/sdd/plan_archive.py`) uses `source.name` →
   `<plans-root>/<yyyymm>/plan.md`. With `preserve_existing=True` it found an earlier
   tale plan already at that path, reported "already archived", and the deterministic
   re-validation in `create_and_launch_epic_from_plan`
   (`src/sase/bead/epic_from_plan.py`) then read the wrong (tale) plan:
   `[tier-mismatch] authored tier 'tale' ... [required-missing] required field 'phases'`.
   This is exactly the observed `sase bead work` failure.
4. `agent_meta.json` `plan_path` records the bundle path, so `sase plan list` Approved
   rows display `~/.sase/interaction_requests/...`, and the rejected-inference in
   `src/sase/main/plan_inventory_collectors.py` misclassifies the archived proposals
   (e.g. `~/.sase/plans/202607/custom_notification_gates.md` shows as Rejected even
   though it was approved as an epic).

Observed collision timeline in the SASE project's plans sidecar repo (all commits carry
the collided stem "plan"):

- `4815027` "Add SDD files for plan" (agent `sase-6e`): tale _Finish unified
  notification-gate integration and land sase-6e_ written to `202607/plan.md` +
  `202607/prompts/plan.md`.
- `c5fe6b1` "Complete SDD plan for plan": marks that tale done.
- `df692e6` "Add SDD prompt for plan" (agent `sase-6e.f7`): epic prompt snapshot for
  _First-class custom notification gates_ overwrites `202607/prompts/plan.md`. The epic
  launch then failed with the tier-mismatch error above (recorded in that run's
  `epic_launch_error` meta field) because `202607/plan.md` still held the earlier tale.
- `af6c711` + `60537dd` (agent `sase-6g` tale _Finish and land xprompt agent families_):
  overwrote both `plan.md` files again.

All fixes below are Python orchestration/glue in this repo; plan validation itself stays
in Rust core and is not touched.

## Design

### 1. Thread the durable plan file through gate resolution

- `src/sase/plan_gate.py`:
  - Add `original_plan_file` to the gate's `presentation.action_data` (in
    `_plan_action_data`) so host contexts carry it without re-reading the envelope.
  - In `plan_context_from_envelope`, also merge `payload.original_plan_file` into
    `host_action_data` so pre-existing bundles (created before this fix) resolve
    correctly.
- `src/sase/plan_approval_actions.py`:
  - Add a helper that resolves the durable plan file for a `PlanApprovalActionContext`:
    prefer `host_action_data["original_plan_file"]`, falling back to reading the
    bundle's `request.json` payload when the context points at a neutral bundle; return
    `None` for legacy in-flight requests.
  - In `run_plan_side_effects`, for approving choices, sync the bundle's (possibly
    user-edited) `plan.md` content back to the durable proposal file before archive side
    effects (create parent dirs / recreate the file if missing; best-effort with
    logging). This restores the legacy semantic that review edits land in the durable
    pending file.
  - `_archive_plan_for_approval` archives from the durable file when available (falling
    back to `host_files[0]`), so the SDD archive destination keeps the canonical
    `<stem>.md` name. The legacy TUI worker in
    `src/sase/ace/tui/actions/agents/_notification_modals.py` shares this function and
    is covered automatically.
- `src/sase/llm_provider/_plan_utils.py` `handle_plan_approval`: after a responded gate,
  return `plan_file = <original_plan_file>` (read from the gate envelope payload; the
  side-effect sync above has already refreshed its content) instead of
  `<bundle>/plan.md`, falling back to the bundle copy only if the durable path is
  unknown. This single change fixes, via existing code paths: meta `plan_path` (and
  therefore `sase plan list`), `sdd_plan_name` in `handle_accepted_plan` (SDD files
  regain real names, commit messages regain real stems), `SASE_PLAN` for the coder
  follow-up, the plan chat preview, and the runner-owned epic launch command.
- `src/sase/notification_gates/registry.py` `GateAdapter.apply_side_effects`: pass the
  durable plan file (not `bundle_path / "plan.md"`) to `prepare_epic_launch` for
  host-owned epic launches, falling back to the bundle copy when unknown.

### 2. Make `sase bead work <plan-file>` bundle-aware and collision-safe

- `src/sase/sdd/plan_archive.py`: accept an optional destination stem/name on
  `plan_archive_destination` and `archive_plan_file` (default remains `source.name`).
- `src/sase/bead/cli_work_from_plan.py` `work_from_plan_file`: when the target is a
  neutral gate bundle resource (sibling `request.json` whose payload carries
  `original_plan_file`), archive under that original stem. Manual invocations against
  `~/.sase/interaction_requests/epic_plan/<id>/plan.md` then archive to
  `<plans-root>/<yyyymm>/<original-stem>.md`.
- Mismatch guard: when `archive_plan_file` preserves an existing destination
  (`written=False`), compare the existing file's authored tier and title against the
  source plan. On mismatch, raise `PlanFileWorkError` naming both paths and suggesting
  the correct resume command, instead of letting deterministic re-validation fail later
  with the confusing "approved epic plan failed deterministic validation" message. A
  matching destination (true resume) keeps today's behavior, including `bead_id`-based
  resume.

### 3. Normalize historical bundle paths in the plan inventory

In `src/sase/main/plan_inventory_collectors.py` (and/or `plan_inventory_paths.py`): when
a recorded `plan_path` sits under the interaction-requests root, resolve it to the
sibling `request.json` payload's `original_plan_file` for the plan key, display path,
and title/tier reads. This fixes display and rejected-inference for the already-recorded
approvals without rewriting history, and keeps working for any straggler metas written
by old code.

### 4. Repair the collision-damaged plans archive

Open the SASE project's plans sidecar repo via the `/sase_repo` skill (do not touch
checkouts by raw path), pull it up to date, then restore canonical names in `202607/`
using the commits listed in the timeline above (locate by subject and timestamp if
hashes are unavailable):

- `git mv 202607/plan.md 202607/finish_xprompt_agent_families.md` and
  `git mv 202607/prompts/plan.md 202607/prompts/finish_xprompt_agent_families.md`
  (current contents are the completed sase-6g tale).
- Restore `202607/finish_unified_notification_gates.md` from `c5fe6b1` (completed
  content) and `202607/prompts/finish_unified_notification_gates.md` from `4815027`.
- Restore `202607/prompts/custom_notification_gates.md` from `df692e6` (the epic's
  prompt snapshot).
- Commit the repair in the sidecar with a message explaining the collision, and push so
  every store materialization agrees before the relaunch.

Best-effort meta backfill (optional, display-only once fix 3 is in): update
`plan_path`/`sdd_plan_path` in the affected planner runs' `agent_meta.json` under
`~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202607/16/` to the repaired
durable paths. The epic planner run is `20260716192139` (agent `sase-6e.f7`).

### 5. Launch the approved epic

With the fixes installed in the working checkout (`just install`), launch the epic
exactly as the user originally attempted:

```bash
sase bead work ~/.sase/interaction_requests/epic_plan/11e06ae8-1cd3-4524-a67d-5c71aa9c5686/plan.md --yes
```

Expected: the plan archives and commits to
`<plans-root>/202607/custom_notification_gates.md` (tier epic), an epic bead plus its 8
phase beads are created with the declared dependency edges, `bead_id` is linked back
into the archived plan, and the first wave (`rust_wire`) launches. Verify with
`sase bead show <epic-id>` and confirm `sase plan list` now resolves both recent
approvals to `~/.sase/plans/202607/...` paths with no false Rejected duplicates.
Back-fill `epic_bead_id` into the `20260716192139` planner meta after a successful
launch (mirroring what `update_epic_launch_metadata` records for detached launches).

## Testing

- `tests/test_plan_gates.py`: gate action_data/envelope now carry `original_plan_file`;
  `plan_context_from_envelope` merges it for old bundles.
- `tests/test_plan_approval_actions.py` (+ `tests/test_plan_approval_responses.py` as
  needed): approving choices sync edited bundle content back to the durable file;
  `_archive_plan_for_approval` archives under the canonical stem; epic host launch
  targets the durable file.
- `tests/test_axe_run_agent_exec_plan_followup_approvals.py` /
  `tests/test_sdd_commit_plan_accept.py`: `handle_plan_approval` returns the durable
  path; `sdd_plan_name` regains the real stem for tale and epic flows.
- `tests/test_bead/test_cli_work_from_plan.py`: bundle-sourced targets archive under the
  original stem; the preserved destination mismatch guard raises the new actionable
  error; true resume (matching content / linked `bead_id`) still works.
- `tests/test_plan_inventory.py` / `test_plan_inventory_scanning.py` /
  `test_plan_inventory_paths.py`: bundle `plan_path` metas normalize to the original
  proposal path for display, dedup, and rejected inference.
- Run `just check` before finishing (run `just install` first in a fresh workspace).

## Risks and notes

- Legacy in-flight requests and bundles without `original_plan_file` must degrade to
  today's behavior (bundle path) rather than fail; every new consumer needs that
  fallback.
- The durable proposal file may have been deleted between propose and approval; the
  sync-back recreates it from the bundle content.
- Stem collisions between _different_ plans remain possible in principle
  (`sase plan propose` dedups within `~/.sase/plans`, but the SDD store has independent
  history); the bead-work mismatch guard turns those into clear errors instead of silent
  consumption.
- The relaunch really starts epic worker agents; do it only after `just check` passes
  and the sidecar repair is pushed.
- Do not modify `sase/memory/*.md` or generated instruction shims; no new CLI options
  are introduced.
