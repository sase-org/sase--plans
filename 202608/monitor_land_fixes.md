---
tier: tale
size: medium
title: Fix the five monitor defects sase-kp.12 found, then land epic sase-kp
goal:
  The five monitor defects that sase-kp.12 found are fixed — finished starter agents no
  longer render as running monitors, in-workspace monitors record and hold their real
  workspace number, signal-terminated monitors stop showing a bare -15 exit code, the
  sase monitor JSON renders the configured project name instead of the ProjectSpec key,
  and the monitor claim-release test is no longer flaky — and epic sase-kp is closed,
  its symvision whitelist cleaned up, and its plan file marked done.
proposed_by: bbugyi200.athena.sase-kp.land
bead: sase-kp
create_time: 2026-08-13 07:56:45
status: wip
---

- **PARENT:**
  [202608/sase_monitor.md](https://github.com/sase-org/sase--plans/blob/main/202608/sase_monitor.md)
- **BEAD:**
  [sase-kp](https://github.com/sase-org/sase--beads/blob/main/pages/sase-kp/README.md)

# Fix the monitor defects sase-kp.12 found, then land epic sase-kp

## Goal

Epic `sase-kp` shipped `sase monitor` (long-running commands as first-class agent family
members). Its final phase, `sase-kp.12`, ran real end-to-end smoke exercises and
recorded five defects as `PROPOSED FOLLOW-UP:` notes. All five are caused by the epic
itself, so they are epic work and must be fixed before `sase-kp` can close.

The land agent re-verified every one of them against live artifacts and current
`master`; the root causes below are confirmed, not hypotheses. Fix all five, then land
the epic.

## Background: how to reproduce the live symptoms

`sase agent list --all --json` currently reports finished starter agents (agents that
handed a command to a monitor and exited) like this:

```
smoke-fail--0      is_monitor=True   status=MONITORING   bucket=Running
smoke-timeout--0   is_monitor=True   status=MONITORING   bucket=Running
smoke-stop--0      is_monitor=True   status=MONITORING   bucket=Running
z2--code           is_monitor=True   status=MONITORING   bucket=Running
```

Every one of those agents is long finished (its `done.json` has `outcome: "completed"`),
yet each renders as a perpetually running monitor. That is defect 1 below.

`sase monitor list --all --format json` currently reports:

```
"project_name": "gh_sase-org__sase",   <- defect 4: ProjectSpec key, not "sase"
"monitor_state": "timeout", "exit_code": -15,   <- defect 3: bare SIGTERM number
"monitor_state": "stopped", "exit_code": -15,   <- defect 3
```

and monitor members whose `monitor_cwd` is a real numbered workspace checkout record
`workspace_num: 0` in their `agent_meta.json` — that is defect 2.

## Defect 1 — starter agents are misclassified as running monitors

**Severity: highest. This is a live, user-visible wrong state in both the CLI and the
TUI, and it affects every agent that has ever handed off to a monitor.**

### Root cause

`src/sase/axe/run_agent_exec_monitor.py` (in `handle_monitor_marker`) writes the
monitor's id into the **starter's** `agent_meta.json` as a back-pointer:

```python
if monitor_id:
    update_meta_field(state.current_artifacts_dir, "monitor_id", monitor_id)
```

Every Python monitor predicate keys on that field, so the starter is then treated as a
monitor member:

- `src/sase/integrations/_agent_list_entry_builder.py`, `_is_monitor()` —
  `bool(meta.monitor_id) or done.outcome == "monitored"`
- `src/sase/ace/tui/models/agent.py`, `Agent.is_monitor` —
  `bool(self.monitor_id) or self.agent_family_role == "monitor"`
- `src/sase/integrations/_editor_helper_agents.py` — both the
  `"kind": "monitor" if getattr(agent, "monitor_id", None) else "agent"` expression and
  `is_monitor=bool(meta.monitor_id)`
- `src/sase/integrations/_mobile_agent_summary.py` —
  `"is_monitor": bool(agent.monitor_id)`

Once `_is_monitor()` is true for a starter, two more things go wrong in
`_agent_list_entry_builder.py`:

- `_derive_status()` takes its monitor branch. The starter's done marker has
  `outcome == "completed"` (not `"monitored"`), so it falls through to the
  `meta.run_started_at` branch and returns `meta.monitor_start_status or "MONITORING"` —
  the starter has no `monitor_start_status`, so it renders as `MONITORING`.
- `record_status_bucket()` returns
  `monitor_state_bucket(done.monitor_state or meta.monitor_state)`; the starter has
  neither, so `monitor_state_bucket(None)` yields the `Running` bucket.

The canonical predicate is **`agent_meta.agent_family_role == "monitor"`**. That is the
documented contract on the Python wire — see the `only_monitors` docstring in
`src/sase/core/agent_scan_wire_records.py` — and it is what the Rust core already
implements in `record_is_monitor()` in
`../sase-core/crates/sase_core/src/agent_scan/index.rs`. Python and Rust currently
disagree; Rust is right.

Monitor members are created by `src/sase/monitor/member.py`, which passes
`agent_family_role="monitor"` to `create_followup_artifacts`, so every real monitor
member already satisfies the canonical predicate.

### Fix

Do both halves. The predicate half repairs artifacts already on disk; the writer half
stops the bug recurring.

1. **Align every Python predicate with the wire contract.** Classify a row as a monitor
   by `agent_family_role == "monitor"`, not by the presence of `monitor_id`. Keep the
   existing `done.outcome == "monitored"` fallback in
   `_agent_list_entry_builder._is_monitor()` for records whose `agent_meta` is missing —
   it is safe, because a starter's done marker records `outcome: "completed"`. Cover all
   four modules listed above.
2. **Stop overloading `monitor_id` on the starter.** In
   `src/sase/axe/run_agent_exec_monitor.py`, record the handoff back-pointer under a
   distinct key (for example `monitor_handoff_id`) next to the
   `monitor_member_agent_name` field that function already writes. Neither field is on
   the agent-scan wire, so this needs no schema bump. Do not remove the back-pointer —
   it is genuinely useful for linking a starter to the monitor it spawned; just stop
   spelling it with the field that means "I am a monitor member".

### Tests

- A finished starter whose meta carries a monitor back-pointer is **not** a monitor and
  keeps its own terminal status and bucket — assert this at the
  `_agent_list_entry_builder` level (both `build_agent_list_entry` and
  `record_status_bucket`) and for `Agent.is_monitor`.
- A real monitor member (`agent_family_role == "monitor"`) is still classified as a
  monitor and still renders its monitor status and bucket.
- The editor-helper and mobile-summary projections agree with the same rule.
- A regression test that the handoff writer no longer sets `monitor_id` on the starter
  while still recording the back-pointer and `monitor_member_agent_name`.

## Defect 2 — monitor members report `workspace_num: 0` for in-workspace commands

### Root cause

In `src/sase/monitor/start.py`, `start_monitor()` only inherits the lane's workspace
claim when **all** of these hold:

```python
if (
    request.inherit_lane_workspace_claim
    and cwd_matches_lane
    and lane_workspace_num is not None
    and runner_pid is not None
):
    resolved_workspace_num = lane_workspace_num
    ...
else:
    resolved_workspace_num = 0
```

`lane_workspace_num` comes from the lane's newest `agent_meta.json` `workspace_num`
field, and that field is commonly `null` even when the same meta's `workspace_dir` names
a real numbered checkout. Confirmed against live artifacts: of the smoke starters, every
one had `workspace_num: null` with a valid numbered `workspace_dir`, and their monitors
all landed on `workspace_num: 0`. The one starter that did carry an explicit
`workspace_num` produced a monitor with the correct number and a real claim transfer.

The design intent — stated in the epic plan and in the
`MONITOR_WORKSPACE_CLAIM_WORKFLOW` comment in the same file — is that an in-workspace
monitor holds and displays the same workspace as the lane it came from.

### Fix

When the lane meta has no explicit `workspace_num` but does have a `workspace_dir`,
derive the number from that directory before deciding which branch to take. Use the
workspace provider rather than parsing the path by hand: `find_marker_from_cwd()` from
`sase.workspace_provider` returns the marker whose `workspace_num` is the value wanted —
see `src/sase/ace/tui/actions/clipboard/_artifact_reference_resolution.py` for an
existing caller.

Keep `resolved_workspace_num = 0` as the honest fallback for a monitor that genuinely is
not running inside a numbered workspace (home mode, or a `--cwd` outside any checkout).
Only take the inherit-and-transfer branch when a real number was resolved **and**
`runner_pid` is available, so the claim transfer stays correct.

### Tests

- A lane whose newest meta has `workspace_num: null` but a numbered `workspace_dir`,
  with `cwd` equal to that directory, produces a monitor member with that workspace
  number and transfers the claim from the runner pid.
- A lane whose `cwd` is outside any numbered checkout still produces
  `workspace_num == 0` and takes the plain-claim path, unchanged.
- The existing claim-transfer coverage in `tests/monitor/` still passes.

## Defect 3 — timeout and stopped monitors display a bare `-15` exit code

### Root cause

`src/sase/monitor/supervise.py` sets `exit_code = child.returncode` unconditionally
after streaming. When the supervisor enforces a timeout or handles a stop it kills the
command's process group with `SIGTERM`, so `returncode` is `-15`. That number is then
rendered as if the command itself had exited with it.

Live examples: the `timeout` monitor and the `stopped` monitor both report
`exit_code: -15`.

### Fix

Keep the raw value in the machine-readable JSON — `-15` is the truthful process result
and scripts may depend on it — but stop presenting it as a command exit status in human
surfaces. In every human-facing renderer, when a monitor was terminated by a signal
rather than exiting on its own (that is, `exit_code` is negative, or `monitor_state` is
`timeout` or `stopped`), render the signal explicitly (for example `killed (SIGTERM)`)
instead of a bare negative integer.

Human surfaces to update:

- `src/sase/main/monitor_render.py` — `_exit_code_text()` (used by the rich table and
  the `monitor show` detail panel) and the exit column in `monitor_list_markdown()`.
- `src/sase/ace/tui/widgets/prompt_panel/_agent_monitor_section.py` — the
  `(exit {agent.monitor_exit_code})` suffix.

### Tests

- A timeout monitor and a stopped monitor render the signal form, not a bare `-15`, in
  the table, the markdown table, the `monitor show` detail panel, and the TUI monitor
  section.
- A monitor whose command exited on its own still renders its plain exit code (`0` dim,
  non-zero red), unchanged.
- The JSON payload still carries the raw `exit_code`.

## Defect 4 — `sase monitor` JSON leaks the ProjectSpec key

### Root cause

`src/sase/main/monitor_render.py`, in `_monitor_json()`, emits
`"project_name": record.project_name`, and `MonitorRecord.project_name` is the
ProjectSpec key taken straight from the artifact scan row. Live output is
`"project_name": "gh_sase-org__sase"` where the configured project name is `sase`.

This violates the repo convention recorded in the `gotchas` memory: user-facing text
must render the configured `PROJECT_NAME`, projecting through
`sase.project_display_names`; keys remain identity and storage only.

`src/sase/main/repo_handler.py` is the precedent to follow — it emits
`project_name=project_display_name_for(host_ctx.project_name)` in its JSON.

### Fix

In `_monitor_json()`, render `project_name` through
`project_display_name_for(record.project_name)` and expose the ProjectSpec key under a
separate `project_key` field so identity is still available to callers. Because the
meaning of an existing field changes, bump `MONITOR_JSON_SCHEMA_VERSION` from `1` to `2`
in the same file.

Audit the rest of the `sase monitor` surface for the same leak while you are here: check
the `scope` dict built in `src/sase/main/monitor_handler.py` and every error and status
message that names a project, and route each through the display-name helper.
`MonitorRecord.project_name` itself should stay the key — it is identity used for
lookups in `src/sase/monitor/store.py`.

### Tests

- `sase monitor list --format json` and `sase monitor show --format json` emit the
  display name in `project_name` and the key in `project_key`, for a project whose
  configured name differs from its key.
- The declared schema version in the envelope is `2`.
- Any project-naming text in `monitor_handler` messages renders the display name.

## Defect 5 — the claim-release race that makes a monitor test flaky

### Root cause

`tests/monitor/test_monitor_start.py::test_start_monitor_promotes_a_bare_lane_and_runs_to_completion`
fails intermittently (reproduced roughly 2 in 15 locally, and it reproduces on plain
`master`). It waits for the done marker to report `monitor_state == "completed"` and
then immediately asserts `get_claimed_workspaces(project_file) == []`.

`_finish_monitor()` in `src/sase/monitor/supervise.py` writes the done marker first and
calls `_release_claim_and_notify()` afterwards, so between those two statements the
claim is still held and the assertion can run in that window.

The implementation ordering is correct and must not change: the done marker is the
terminal signal, and the follow-up-agent path deliberately transfers the claim instead
of releasing it, so releasing before the marker would be wrong.

### Fix

Fix the test, not the ordering. Wait for the claim release as its own bounded observable
instead of assuming it has already happened when the done marker appears. Note that
`tools/check_test_wait_helpers` gates bounded-wait idioms in `tests/`, so extend the
existing `_wait_for_done` style helper in that test module (or use whatever shared
helper the gate sanctions) rather than writing a fresh private polling loop — run
`just _lint-test-waits` to confirm.

Add an explicit regression test for the ordering contract itself: on a terminal monitor
with no follow-up, the done marker is written before the claim is released, and the
claim is released eventually.

Re-run the test at least 15 times to confirm the flake is gone.

## Landing (do this last, after every fix above is committed)

1. Run `just check`. Run `just check-full` through `/sase_monitor` — never inline — and
   hand it a `--next` action so the follow-up agent acts on the result.

   `just check-full` and `just check` currently fail at the patch/stitch terminology
   gate with exactly three unclassified `changespec` defects in
   `tests/test_validate_sase_core_rs_tool.py` and `tools/validate_sase_core_rs`. That
   failure is pre-existing on clean `master`, predates this epic, and is already tracked
   as ready task bead **`sase-kq`** with three `+1` corroborations. Treat only that
   exact failure as an expected unrelated blocker; investigate anything else.

2. Close the epic:

   ```
   sase bead close sase-kp --note "<what was verified and fixed>"
   ```

   The note must record: the twelve phases verified, the five defects fixed here, and
   the follow-up dispositions (see below).

3. **After** closing, run `just symvision`. Epic-symbol whitelist entries for `sase-kp`
   expire at close, so it will report newly stale entries and unused code — remove them.

4. Set `status: done` in the frontmatter of the epic's plan file. Get its path from
   `sase bead show sase-kp` (the `PLAN` line).

### Follow-up dispositions already settled by the land agent

Every `PROPOSED FOLLOW-UP:` note across the twelve phases has already been triaged
through `/sase_new_task`. **Do not re-file any of these.** The close note should record
them, and nothing else here needs action:

- **Patch/stitch terminology audit** (proposed by ten phases: `.2` `.3` `.4` `.5` `.6`
  `.7` `.8` `.9` `.10` `.11`) — already tracked as ready task bead **`sase-kq`**;
  corroborated with a `+1` carrying the ten-phase impact evidence (now +5). Confirmed
  pre-existing and not caused by this epic: the offending token in
  `tools/validate_sase_core_rs` was introduced by commit `2fb4313af`, which landed
  twenty minutes before this epic's first commit `3c37f8e36`, and `sase-kp.1`'s only
  edit to that file was the schema probe version bump.
- **Questions-flow hang under the full xdist lane** (proposed by `sase-kp.7`) — routed
  as a `DISCOVERED ISSUE:` note onto **active epic `sase-j7`** ("Fix the sase-ct flake
  class at its root — process-global state leaking between tests"), which already
  carries an equivalent hang in the same module blocked in
  `notification_gates.poller.wait_for_gate`. Not this epic's work.
- **Safe epic-approval smoke harness** (proposed by `sase-kp.12`) — filed as new ready
  task bead **`sase-kr`** (`medium`). A coverage gap, not a defect.
- **Glossary entry for "Monitor Member"** (proposed by `sase-kp.11`) — **declined**, on
  precedent. Task `sase-hy` was the identical request for a different term and the owner
  closed it `canceled` with the reasoning that a glossary entry is now a `sase.yml` edit
  plus `sase memory init`, is "a cheap edit the owner can request directly in
  conversation when wanted", "does not need to hold a triage slot", and "memory edits
  require explicit user permission anyway". Filing another one would repeat work the
  owner has already declined.
- **Date-hardcoded teardown test** (proposed by `sase-kp.6`) — **already fixed inside
  the epic**. The `sase-kp.9` commit replaced the day-shard glob in
  `test_start_monitor_tears_down_the_member_when_the_supervisor_cannot_spawn` with a
  repo-root glob. Verified passing on a later date than the hardcoded one, so the
  reported failure mode cannot recur.

## Constraints

- Follow the repo's two-speed verification rule: `just check` inline is fine;
  `just check-full` must go through `/sase_monitor`.
- Run `just install` first — workspace directories are ephemeral and dependencies may
  have shifted.
- Do not edit any file under `sase/memory/`, `AGENTS.md`, or the generated provider
  instruction shims. None of the work above requires it.
- The Rust core is correct here and needs no changes; all five fixes are Python-side.
