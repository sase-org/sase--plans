---
tier: epic
title: sase monitor — long-running commands as first-class agent family members
goal: "Agents can hand a slow command (`just check-full`, a CI wait, `sase bead work`)
  to `sase monitor start` and be killed cleanly, while the command keeps running as a
  live, first-class monitor member of their agent family — with streaming output, live
  runtime, custom start/stop statuses, a timeout, and an optional follow-up agent that
  resumes the same workspace and conversation when the command finishes.

  "
phases:
  - id: core-wire
    title: Monitor marker fields on the agent scan wire
    depends_on: []
    size: small
    description: "core-wire: add the monitor marker fields to the Rust `AgentMetaWire` /
      `DoneMarkerWire` and their Python mirrors so monitor members survive the agent
      artifact scan.

      "
  - id: status-bucket
    title: First-class custom agent status labels
    depends_on: []
    size: medium
    description: "status-bucket: give `Agent` an explicit status-bucket override and
      route agent-shaped bucket lookups through one accessor so arbitrary status labels
      bucket correctly.

      "
  - id: engine-run
    title: Monitor member lifecycle and supervisor process
    depends_on:
      - core-wire
      - status-bucket
    size: medium
    description: "engine-run: create the monitor family member, run and stream the
      monitored command from a detached supervisor, enforce the timeout, own the
      workspace claim, and write the terminal marker.

      "
  - id: engine-next
    title: Follow-up agent handoff after a monitor completes
    depends_on:
      - engine-run
    size: medium
    description: "engine-next: compose the command breakdown, resume the starter's
      conversation with `#fork`, and launch the follow-up agent into the same lane and
      workspace.

      "
  - id: handoff
    title: In-agent handoff marker and runner adoption
    depends_on:
      - engine-run
    size: medium
    description: "handoff: add the `.sase_monitor_pending` handoff so `sase monitor
      start` kills the calling agent cleanly, saves its chat for `#fork`, and never
      releases the workspace.

      "
  - id: cli
    title: sase monitor command group
    depends_on:
      - engine-next
      - handoff
    size: medium
    description: "cli: add `sase monitor start|stop|list|show` with multi-format output,
      duration parsing, and monitor id resolution.

      "
  - id: tui-rows
    title: Monitor rows in agent lists and family rosters
    depends_on:
      - status-bucket
      - engine-run
    size: medium
    description: "tui-rows: render monitor members in the Agents tab, family roster, and
      integration agent lists with their own glyph, phase label, live runtime, and
      status.

      "
  - id: tui-detail
    title: Monitor detail panel, live output, and keybindings
    depends_on:
      - tui-rows
      - cli
    size: medium
    description: "tui-detail: render the monitor detail section and live command output,
      and wire the stop action into the footer and help modal.

      "
  - id: epic-launch
    title: Approved-epic launch runs as a monitor
    depends_on:
      - cli
    size: medium
    description: "epic-launch: replace the detached epic-launch task with a generic
      monitor start using `EPIC APPROVED` / `EPIC CREATED`, keeping the host claim and a
      fallback.

      "
  - id: skill
    title: /sase_monitor skill
    depends_on:
      - cli
    size: small
    description: "skill: author the `/sase_monitor` skill source so agents prefer it
      over their own monitor and scheduled wake-up tools.

      "
  - id: memory-docs
    title: Memory and documentation updates
    depends_on:
      - skill
    size: small
    description: "memory-docs: update the build-and-run memory note, regenerate agent
      instruction files, and document monitors.

      "
  - id: smoke
    title: End-to-end monitor exercises
    depends_on:
      - tui-detail
      - epic-launch
      - memory-docs
    size: xsmall
    description:
      "smoke: launch real agents that exercise the sleep, timeout, stop, and follow-up
      monitor paths and report what the surfaces actually showed."
proposed_by: bbugyi200.athena.yy
create_time: 2026-08-12 17:27:54
status: wip
---

- **PROMPT:**
  [prompts/202608/sase_monitor.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/sase_monitor.md)

# Plan: `sase monitor` — long-running commands as agent family members

## Problem

SASE agents are single-turn: a provider turn runs, the runner captures it, and the agent
is done. Agent runtimes ship their own "monitor" / "scheduled wake-up" tools that assume
a multi-turn session, so agents reach for them when a command is slow
(`just check-full`, a CI poll, a deploy). Those tools do nothing useful in SASE — the
turn ends, the wake-up never fires, and the agent either blocks its whole turn on a slow
command or gives up on verifying its work.

SASE already has the right shape for this: an **agent family** is a strictly sequential
chain of members in one lane, and the runner already hands off between members (`--plan`
→ `--code`, question rounds, feedback rounds). What is missing is a member whose work is
a supervised OS command rather than an LLM turn.

`bash` steps in xprompt workflows are the closest existing thing, and they are the wrong
tool: `subprocess.run(..., capture_output=True)` with no streaming, no runtime
accounting, no timeout, no status of their own, no family membership, and no way to hand
the result to a fresh agent.

## Solution overview

Introduce the **monitor member**: a real agent-family member whose work is one
supervised command.

```
lane "acme"          before                     after `sase monitor start`
─────────────────────────────────────────────────────────────────────────────
acme                 (single agent, RUNNING)    acme            (family container)
                                                ├─ acme--0      DONE      ← starter, killed
                                                ├─ acme--mon    MONITORING ← the command
                                                └─ acme--1      RUNNING   ← follow-up agent
```

A monitor member is an ordinary artifacts directory under
`~/.sase/projects/<key>/artifacts/ace-run/<timestamp>/` with `agent_meta.json`,
`live_reply.md`, and `done.json`, exactly like an agent. That single decision is what
buys "first-class support for agent families" nearly for free: the artifact scanner
finds it, the loaders build an `Agent` row for it, family roster / runtime aggregation /
fold state / chat history / `%wait` resolution all already operate on that shape.

The command itself is run by a **detached monitor supervisor** process, not by the
calling agent's runner. This is deliberate:

- the monitored command outlives the agent that started it, which is the entire point;
- the host-owned approved-epic launch has no live runner to lean on and must stay
  durable (today it is a detached task claimed by whichever surface answered the gate);
- the supervisor owns the workspace claim, so the workspace is never released and
  re-claimed between the starter and the follow-up agent — the follow-up sees exactly
  the tree the monitor started with, plus whatever the command itself changed.

The supervisor takes over the calling agent's workspace claim using
`transfer_workspace_claim()` — the same primitive the spawn-on-retry flow already uses
to hand a claim from a dying parent to a fresh detached child without freeing the slot.
When the command finishes and a next action was given, the supervisor transfers the
claim once more to the follow-up agent it spawns.

### Vocabulary used throughout this plan

| Term           | Meaning                                                                    |
| -------------- | -------------------------------------------------------------------------- |
| starter        | The agent that ran `sase monitor start` (absent for host-started monitors) |
| monitor member | The `--mon` family member representing the supervised command              |
| supervisor     | The detached `sase monitor _supervise` process that runs the command       |
| follow-up      | The agent launched into the lane after the command finishes                |
| lane           | Agent lane — a family, or a single agent that is not yet a family          |

### Durable monitor record

There is **no new store**. A monitor's durable record is the monitor member's
`agent_meta.json` plus its `done.json`, and `sase monitor list|show` are queries over
the existing agent artifact index. History, cross-surface consistency, and cleanup all
come from the machinery that already manages agent artifacts.

`agent_meta.json` fields owned by monitors (all optional, all `monitor_*`-prefixed):

| Field                      | Type  | Meaning                                                        |
| -------------------------- | ----- | -------------------------------------------------------------- |
| `monitor_id`               | str   | Short opaque id (same alphabet/length as task ids)             |
| `monitor_command`          | str   | The exact command string handed to the shell                   |
| `monitor_cwd`              | str   | Absolute working directory the command runs in                 |
| `monitor_label`            | str   | Short human label for rows (defaults to the command's head)    |
| `monitor_reason`           | str   | Why this command is being monitored                            |
| `monitor_next_action`      | str   | Follow-up instruction, empty when none                         |
| `monitor_start_status`     | str   | Status label while running (default `MONITORING`)              |
| `monitor_stop_status`      | str   | Status label once finished (default `MONITORED`)               |
| `monitor_timeout_seconds`  | float | Timeout budget                                                 |
| `monitor_state`            | str   | `running` \| `completed` \| `failed` \| `timeout` \| `stopped` |
| `monitor_exit_code`        | int   | Command exit code (absent while running / on timeout)          |
| `monitor_output_path`      | str   | Path to the captured output (`live_reply.md`)                  |
| `monitor_output_truncated` | bool  | Whether retained output elided a middle span                   |
| `monitor_starter_agent`    | str   | Starter agent name, for `#fork` (absent for host-started)      |
| `monitor_followup_agent`   | str   | Follow-up agent name once launched                             |
| `monitor_tail_lines`       | int   | Output tail size handed to the follow-up                       |

`done.json` gains `monitor_state`, `monitor_exit_code`, `monitor_elapsed_seconds`, and
`status_label`, and uses `outcome: "monitored"`.

### Status semantics

The **displayed status** is always the configured label (`MONITORING` → `MONITORED`,
`EPIC APPROVED` → `EPIC CREATED`, `SLEEPING FOR 300s` → `SLEPT FOR 300s`). Because those
labels are arbitrary strings, `status_bucket_for_values()` cannot classify them — every
unknown status falls through to `Running`, so a finished monitor would look live
forever. The `status-bucket` phase fixes that generally by letting a row carry an
explicit bucket.

The **bucket** reflects the command's real outcome, which is predictable and honest:

| `monitor_state` | Bucket  | Displayed status              |
| --------------- | ------- | ----------------------------- |
| `running`       | Running | start status                  |
| `completed`     | Done    | stop status                   |
| `failed`        | Failed  | stop status (+ exit N badge)  |
| `timeout`       | Failed  | stop status (+ timeout badge) |
| `stopped`       | Done    | `STOPPED`                     |

A failing `just check-full` therefore shows up as a Failed member with the exit code
visible, and the follow-up agent still launches to fix it.

### Non-goals

- No new Rust monitor store or monitor domain crate. Monitors are modeled as agent
  family members, and agent-family/status semantics already live in Python next to the
  runner; the Rust core change is confined to the wire/scan layer that must know the new
  marker fields. Splitting the monitor lifecycle across the boundary would fragment one
  model.
- No `monitor:` config section. Behavior is set per invocation by CLI flags.
- No cron/recurring monitors, no monitor restart, no multiple concurrent monitors per
  lane (a lane is sequential by definition — starting a second monitor while one is
  active is an error that points at the active one).

---

## Phase `core-wire`: Monitor marker fields on the agent scan wire

The agent artifact scanner is Rust (`crates/sase_core/src/agent_scan/`) and its wires
are **explicit field lists** — unknown keys in `agent_meta.json` / `done.json` are
dropped, not round-tripped. Monitor fields must therefore be declared on both sides
before anything else can read them.

Open the core repo with the `/sase_repo` skill first:

```bash
sase repo open sase-core -r "Add monitor marker fields to the agent scan wire"
```

Work items:

1. In `crates/sase_core/src/agent_scan/wire.rs`, add the `monitor_*` fields from the
   table above to `AgentMetaWire` (all `#[serde(default)]`), and add `monitor_state`,
   `monitor_exit_code`, `monitor_elapsed_seconds`, `status_label` to `DoneMarkerWire`.
2. Bump `AGENT_SCAN_WIRE_SCHEMA_VERSION` and mirror the bump in
   `src/sase/core/agent_scan_wire_records.py`.
3. Mirror the fields in `src/sase/core/agent_scan_wire_markers.py` (`AgentMetaWire`,
   `DoneMarkerWire`) and in `agent_scan_wire_conversion.py` if the conversion helpers
   enumerate fields.
4. Extend `AgentArtifactIndexQueryWire` with a `only_monitors: bool` filter so
   `sase monitor list` can ask the index for monitor members instead of scanning and
   filtering in Python.
5. Rust tests: round-trip a record carrying every new field; assert an older record
   without them still parses. Python tests: `tests/core/` round-trip parity.

Land the core change first (its own commit in `sase-core`), then the Python mirror. The
Python side must tolerate a core binary that predates the field (fields simply arrive
absent) — `known_field_kwargs` already gives this in the other direction.

## Phase `status-bucket`: First-class custom agent status labels

`status_bucket_for_values()` maps a status string to one of the seven buckets by
membership in hard-coded sets, defaulting to `Running`. That is fine for a fixed status
vocabulary and wrong for monitors, whose labels are user-supplied.

Work items:

1. Add `status_bucket: str | None = None` to the `Agent` model
   (`src/sase/ace/tui/models/agent.py`).
2. Add `agent_status_bucket(agent) -> str` to `src/sase/agent/status_buckets.py` (or a
   thin TUI-side sibling if the import direction requires it), returning
   `agent.status_bucket` when set and
   `status_bucket_for_values(agent.status, agent.retried_as_timestamp)` otherwise.
   Validate the override against `AGENT_STATUS_BUCKETS` and fall back to the derived
   bucket on an unknown value rather than raising — a malformed marker must never break
   the Agents tab.
3. Migrate every `status_bucket_for_values(<agent-ish>.status)` call site to
   `agent_status_bucket(...)`. The known set is in `_agent_completion_wait.py`,
   `_agent_list_render_layout.py`, `_agent_display_clan.py`,
   `_agent_display_header_metadata.py`, `_agent_display_tribe_header.py`,
   `_member_roster.py`, `file_panel/_file_list.py`, `file_panel/_diff.py`,
   `file_panel/_linked_deltas.py`, `agent_family_members.py`, `_agent_status_apply.py`.
   Grep for the symbol; do not rely on this list being exhaustive.
4. `agent_family_members.family_member_status_buckets()` and `concrete_agent_statuses()`
   must respect the override for both settled and final members.
5. Carry the override through the integration wire:
   `src/sase/integrations/_agent_list_entry_builder.py` and `agent_list_entries.py`
   compute buckets from raw statuses; give the entry a `status_bucket` field populated
   from the agent row so Telegram/mobile/`/sase_agents_status` agree with the TUI.
6. `aggregate_agent_group_bucket()` already accepts `(status, effective_bucket)` pairs —
   make sure family/clan aggregation feeds it the overridden bucket so a finished
   monitor does not keep its container Running.
7. Tests: a row with a nonsense status and `status_bucket="Done"` buckets Done
   everywhere; rows without an override are byte-identical to today (regression-guard
   the existing bucket tests).

This phase ships no monitor behavior. It is deliberately separate so its blast radius is
reviewable on its own.

## Phase `engine-run`: Monitor member lifecycle and supervisor process

New package `src/sase/monitor/` (keep every module under the `_lint-toobig` cap; split
early rather than late):

- `models.py` — `MonitorRecord` (the projection of the fields above), `MonitorState`,
  and `monitor_state_bucket()`.
- `naming.py` — suffix/role allocation.
- `member.py` — create the monitor member's artifacts directory.
- `start.py` — the `start` API used by both the CLI and the host epic launch.
- `supervise.py` — the detached supervisor entry point.
- `output.py` — bounded streaming capture.
- `store.py` — index-backed lookup for `list` / `show` / `stop`.

### Naming and role

Add to `src/sase/plan_chain.py`:

- `PLAN_CHAIN_MONITOR_SUFFIX = "--mon"`, registered in `_KNOWN_SUFFIXES` and in
  `_PHASE_SUFFIX_ROLES` with role `"monitor"`, and `"monitor"` added to
  `_EXPLICIT_FAMILY_ROLES`.
- A `--mon-<token>` sequence suffix that parses with role `"monitor"` and kind
  `"phase"`, mirroring the existing `--plan-<token>` handling in
  `_PLAN_FEEDBACK_SUFFIX_RE`.

The first monitor in a lane takes `--mon`; later ones allocate from the template
`--mon-@` via `allocate_agent_family_child_suffix()`, giving `--mon-0`, `--mon-1`, … .

### Starting a monitor

`sase.monitor.start.start_monitor(request) -> MonitorRecord` performs, in this order:

1. **Resolve the lane.** Default to the calling agent's lane (`SASE_AGENT_NAME` →
   `agent_family_base()`, falling back to the bare name), else the explicit `lane`
   argument. Resolve it to the lane's newest member artifacts directory, workspace, and
   project via the agent artifact index.
2. **Reject duplicates.** If the lane already has a `running` monitor, fail with an
   error naming it. If an identical `monitor_command` is already running in this lane,
   return that record unchanged (this is the generic form of today's
   `_active_epic_launch_for_plan` dedupe, and the epic launch depends on it).
3. **Promote the lane to a family** when it is still a single agent, via
   `promote_agent_to_family(parent_artifacts_dir, base_name)` — the starter is renamed
   `<lane>--0` (or `<lane>--plan` when it is a planner) and `<lane>` becomes a pure
   container. This is exactly what `%n(suffix, family=parent)` already does.
4. **Create the monitor member** with `create_followup_artifacts()`-shaped metadata:
   inherit `model`, `llm_provider`, `reasoning_effort`, `workspace_dir`, `cl_name`,
   `bead_id`, `tribe`, and the bead/epic association fields from the lane's newest
   member so the follow-up agent inherits them too; set `role_suffix`, `agent_family`,
   `agent_family_role="monitor"`, `parent_timestamp`, `run_started_at`, and every
   `monitor_*` field; `monitor_state="running"`.
5. **Spawn the supervisor** detached (`setsid`, own process group, stdout/stderr to the
   member's artifacts) running `sase monitor _supervise --artifacts-dir <member-dir>` —
   an internal, hidden subcommand. Record its pid in the member's `agent_meta.json`
   (`pid`).
6. **Take the workspace claim.** When `cwd` is the lane's workspace,
   `transfer_workspace_claim(project_file, workspace_num, from_pid=<runner pid>, to_pid=<supervisor pid>, new_workflow=<monitor workflow name>, new_artifacts_timestamp=<member timestamp>)`.
   Because release matches on `(workspace_num, workflow, cl_name)`, giving the claim a
   new workflow name means the starter's runner cleanup can no longer release it — the
   slot is continuously held. When `cwd` is outside the lane's workspace (the epic
   launch runs in the primary workspace) or there is no calling agent, claim workspace
   `0` instead: the row is visible in the Agents tab and reserves nothing, the same
   trick deferred-workspace (`%wait`) agents use.
7. Return the record. The caller (the `handoff` phase) is responsible for killing the
   starter.

If any step after (4) fails, tear down: mark the member `monitor_state="failed"` with a
clear error, write `done.json`, restore/release claims, and raise. A half-created
monitor must never leave a lane with a phantom running member.

### The supervisor

`sase monitor _supervise --artifacts-dir <dir>`:

1. Re-reads the member's `agent_meta.json` for the full request; refuses to run if
   `monitor_state != "running"`.
2. Installs a SIGTERM handler that terminates the command's process group, sets
   `monitor_state="stopped"`, writes the terminal marker, and exits — this is what
   `sase monitor stop` triggers.
3. Runs the command with
   `subprocess.Popen(command, shell=True, cwd=..., start_new_session=True, stdout=PIPE, stderr=STDOUT, text=True)`.
   A new session means a timeout or stop kills the whole tree, not just the shell — a
   concrete improvement over `bash` steps.
4. Streams merged output line by line into `<dir>/live_reply.md`, flushing after each
   line so the TUI's existing `get_live_reply_content()` shows it live with no new
   plumbing. Capping is enforced by `output.py`: retain at most
   `MONITOR_MAX_OUTPUT_BYTES` (2 MiB) as head + tail with an explicit
   `… <n> bytes elided …` marker between them, and set `monitor_output_truncated`.
   Unbounded capture of a chatty command must not fill the artifacts store.
5. Enforces `monitor_timeout_seconds`. On expiry: `SIGTERM` the process group, wait a
   short grace period, then `SIGKILL`; set `monitor_state="timeout"`.
6. On exit, sets `monitor_state` to `completed` (exit 0) or `failed` (non-zero), records
   `monitor_exit_code`, `stopped_at`, and elapsed seconds, and writes `done.json` with
   `outcome: "monitored"`, `status_label: <stop status>`, and the fields above.
   Refreshes the agent artifact index
   (`update_agent_artifact_index_for_marker_mutation`) and touches
   `<project>/artifacts/.ace_refresh_pulse` so the ACE watcher wakes immediately — the
   marker write is too deep in the tree for the non-recursive watcher.
7. Saves a chat history entry for the monitor member (`save_chat_history` with
   `workflow="ace-run"`, `agent=<member name>`, prompt = the command + reason, response
   = the retained output) so monitors appear in `sase chats` and the chat panel like any
   other member.
8. Hands off to `engine-next`, or — with no next action — releases the workspace claim
   and sends a completion notification via `notify_workflow_complete("monitor", ...)`.

### Stopping

`sase.monitor.stop_monitor(record)` sends `SIGTERM` to the supervisor's process group
and waits (bounded) for `monitor_state` to leave `running`. If the supervisor pid is
already dead, reconcile the record in place: `monitor_state="failed"` with
`error: "monitor supervisor died without reporting"`. This mirrors the task store's
dead-supervisor reconciliation and keeps a crashed monitor from showing RUNNING forever.

### Tests

`tests/monitor/` with a fake command (`sh -c 'echo hi; sleep …'`): member creation and
promotion, streaming into `live_reply.md`, output cap and elision, timeout kills the
whole process group, non-zero exit records the code, stop is idempotent,
duplicate-command start returns the existing record, second-concurrent-monitor start is
rejected, teardown on mid-creation failure. Use the existing artifact/workspace
fixtures; do not add sleeps — poll the marker files.

## Phase `engine-next`: Follow-up agent handoff after a monitor completes

When `monitor_next_action` is set and `monitor_state != "stopped"`, the supervisor
launches one follow-up agent into the same lane.

1. **Wait for the starter to settle.** Poll (bounded, ~60s) for the starter member's
   `done.json`. Two agents must never be live in one lane, and `#fork` needs the
   starter's saved chat. If the wait expires, continue without the `#fork` prefix rather
   than dropping the follow-up.
2. **Allocate the follow-up suffix** from the generic template `--@` via
   `allocate_agent_family_child_suffix()`, reserving the starter and monitor suffixes.
3. **Compose the prompt.** The `#fork:<starter>` prefix gives the follow-up the
   starter's full conversation (this is the same mechanism
   `SASE_CODER_INHERIT_PLANNER_CHAT` and `spawn_retry_agent` already use). Then the
   breakdown — this is the part that decides whether the follow-up can actually debug
   the command, so make it excellent:

   ````markdown
   #fork:acme--0

   # Monitored command finished

   |               |                                                                       |
   | ------------- | --------------------------------------------------------------------- |
   | **Command**   | `just check-full`                                                     |
   | **Directory** | `/home/…/sase_12`                                                     |
   | **Outcome**   | FAILED — exit 1                                                       |
   | **Started**   | 2026-08-12 14:02:11 (local)                                           |
   | **Finished**  | 2026-08-12 14:19:48 (local)                                           |
   | **Elapsed**   | 17m 37s of a 45m budget                                               |
   | **Output**    | 1,284 lines · 96 KiB · full log: `sase monitor show m4kq --all-lines` |

   **Why this was monitored:** verify the refactor before handing back to the user.

   ## Last 200 lines of output

   ```text
   …
   ```
   ````

   ## Your next action

   Fix any failures `just check-full` reported, then reply to the user.

   ```

   Rules the composer must honor:
   - The outcome line is unambiguous for all four terminal states, and a timeout says
     plainly that the command **did not finish**, how long it ran, and what it had produced
     so far, so the follow-up decides whether to re-run or investigate.
   - The tail is `monitor_tail_lines` lines (default 200) fenced as `text`. Fences inside
     the output are escaped by widening the fence, never by mangling the bytes.
   - The full log path and the `sase monitor show <id> --all-lines` invocation are always
     given, so the follow-up can read more than the tail.
   - The next action is the last section, under its own heading, verbatim as given.
   ```

4. **Launch.**
   `spawn_agent_subprocess(..., workspace_dir=<lane workspace>, workspace_num=<same>, retry_transfer_from_pid=os.getpid())`
   so the claim moves straight from the supervisor to the follow-up runner and the
   workspace is never free. Pass the inherited `agent_name_override`, `workflow_name`
   (the family name), `agent_family_role`, and the starter's model/provider/effort so
   the follow-up is the same kind of agent that started the monitor. Record
   `monitor_followup_agent` on the monitor member.
5. If the launch fails, do not silently swallow it: record the error on the monitor
   member, release the claim, and send a failure notification naming the exact
   `sase monitor show <id>` command to inspect.

Tests: prompt composition golden tests for completed / failed / timeout / no-fork cases;
the follow-up inherits model and workspace; the claim never appears free between
supervisor exit and follow-up start; a `stopped` monitor launches nothing.

## Phase `handoff`: In-agent handoff marker and runner adoption

This is what makes `sase monitor start` safe to run from inside an agent. It mirrors
`sase plan propose` / `sase questions` exactly.

1. `sase monitor start`, when `SASE_AGENT` is set, after `start_monitor()` returns:
   writes `<SASE_ARTIFACTS_DIR>/.sase_monitor_pending` containing
   `{"monitor_id", "member_artifacts_dir", "member_agent_name", "timestamp"}`, touches
   `<project>/artifacts/.ace_refresh_pulse`, then calls
   `kill_agent_runner_group(artifacts_dir)`.
2. Add `".sase_monitor_pending"` to `PENDING_HANDOFF_MARKERS` in
   `src/sase/agent/pending_handoff.py`. This is load-bearing:
   `install_workspace_release_sigterm_handler()` skips the workspace release when a
   pending handoff marker exists, which — together with the claim transfer — guarantees
   the slot is never released.
3. In `src/sase/axe/run_agent_exec.py::_handle_killed_iteration`, read and delete
   `.sase_monitor_pending` alongside the plan/question markers, and dispatch to a new
   `handle_monitor_marker()` in `src/sase/axe/run_agent_exec_monitor.py`. Keep the
   existing precedence: an explicit user kill still wins and discards the marker.
4. `handle_monitor_marker()`:
   - `normalize_handoff_interruption_state()` +
     `finalize_handoff_artifacts_as_completed()` on the starter's artifacts, as the
     plan/question handlers do;
   - `update_meta_suffix()` to the promoted starter suffix when this is the first family
     member, so the renamed starter and its artifacts agree;
   - saves the starter's chat with `save_chat_history()` using a synthetic response
     describing the monitor it started (the real provider response was lost to SIGTERM)
     — this is the chat `#fork:<starter>` will resume, so it must include the command,
     reason, next action, and the `sase monitor show <id>` pointer;
   - records `monitor_id` / `monitor_member_agent_name` as relationships on the starter;
   - returns a `"monitored"` loop outcome that breaks the loop.
5. `run_agent_exec_finalize` maps the `"monitored"` outcome to a terminal `DONE` with
   `outcome: "completed"` in `done.json`. The starter finished its turn and delegated;
   it is not a failure and not a kill. Ensure `AGENT_KILLS` is **not** incremented for
   this path.
6. `run_agent_runner_lifecycle` must not release or hold the workspace for a
   `"monitored"` outcome — the claim already belongs to the supervisor.

Tests: extend `tests/_axe_run_agent_exec_helpers.py`-style harnesses — the marker is
adopted, the starter lands `DONE` with a saved chat, the workspace claim is still held
by the supervisor pid afterwards, a marker written after the kill timestamp is ignored
(the existing `_marker_predates_kill` guard), and a user kill discards the marker.

## Phase `cli`: `sase monitor` command group

New `src/sase/main/parser_monitor.py`, `monitor_handler.py`, `monitor_render.py`,
modeled directly on the `task` trio. Register in `src/sase/main/parser.py`; bare
`sase monitor` delegates to `sase monitor list` through the central
`_default_list_subcommands()` wiring (do not re-implement it). Keep subcommands and
options alphabetical, give every public long option a short alias, and make the `-h`
output complete and scannable.

### `sase monitor start`

| Option                    | Required | Meaning                                                        |
| ------------------------- | -------- | -------------------------------------------------------------- |
| `-c, --command CMD`       | yes      | The full command to run and monitor                            |
| `-C, --cwd DIR`           | no       | Working directory (default: the lane's workspace, else `$PWD`) |
| `-j, --json`              | no       | Machine-readable result                                        |
| `-L, --label TEXT`        | no       | Short row label (default: derived from the command)            |
| `-l, --lane NAME`         | no       | Target agent lane (default: the calling agent's lane)          |
| `-n, --next TEXT`         | no       | Next action for the follow-up agent; omit for no follow-up     |
| `-r, --reason TEXT`       | yes      | Why this command is being monitored                            |
| `-s, --start-status TEXT` | no       | Status while running (default `MONITORING`)                    |
| `-S, --stop-status TEXT`  | no       | Status when finished (default `MONITORED`)                     |
| `-T, --tail-lines N`      | no       | Output tail handed to the follow-up (default 200)              |
| `-t, --timeout DURATION`  | yes      | Timeout: bare seconds or `90s` / `45m` / `2h`                  |

`--timeout` accepts a human duration and reports the parsed value back, because a silent
unit mistake on a 45-minute command is expensive. Status labels are validated:
non-empty, no newlines, and capped (48 chars) so rows cannot be broken by a pathological
label.

On success, print the monitor id, the member agent name, the resolved timeout, and the
`sase monitor show <id> --follow` pointer. When run inside an agent, this is the last
output the agent produces before it is killed — say so explicitly, the way
`sase plan propose` does.

### `sase monitor stop [ID]`

Stops a running monitor. `ID` accepts a monitor id or unique prefix, a monitor member
agent name, or a lane name; omitted, it targets the calling agent's lane's active
monitor. No follow-up agent is launched even when a next action was recorded — say so in
the output. `-j/--json`.

### `sase monitor list`

Active monitors by default. `-a/--all` includes history;
`-f/--format {table,markdown,json}` (default `table`), `-j/--json` as the shorthand,
`-l/--lane`, `-n/--limit`, `-p/--project`, `-s/--status` (repeatable: `running`,
`completed`, `failed`, `timeout`, `stopped`). Rich table columns: id, state (colored),
label, lane/member, elapsed, exit, started.

### `sase monitor show ID`

`-A/--all-lines`, `-f/--format {markdown,json}` (default `markdown`), `-F/--follow`,
`-l/--log-lines N` (default 200), `-o/--output-only`. `--follow` streams new output
until the monitor reaches a terminal state, exactly like `sase task show --follow`. The
markdown detail shows command, cwd, reason, next action, state, exit code, timeout
budget vs elapsed, lane/member/follow-up names, and the log tail.

### `sase monitor _supervise`

Hidden internal subcommand (suppressed from help), takes `--artifacts-dir`. It is not a
user surface.

Tests: `tests/main/test_parser_monitor.py` for parsing and help ordering,
`tests/main/test_monitor_handler_*.py` per subcommand, plus JSON envelope stability
tests. Add the new subcommands to whatever command-availability fixtures exist
(`tests/_command_availability_helpers.py`).

## Phase `tui-rows`: Monitor rows in agent lists and family rosters

1. `src/sase/ace/tui/models/agent.py` — surface the monitor projection on `Agent`
   (`monitor_id`, `monitor_state`, `monitor_command`, `monitor_label`,
   `monitor_exit_code`, `is_monitor`), populated in `_loaders/_meta_enrichment*.py` from
   the wire fields and in `_done_loaders.py` from the `outcome: "monitored"` branch.
   Both loaders set `agent.status` to the configured label and `agent.status_bucket`
   from `monitor_state_bucket()`.
2. `_running_loaders.py` promotes a live monitor row from `STARTING` to its start status
   once `run_started_at` is present — the same promotion agents already get.
3. `src/sase/ace/tui/widgets/_agent_list_styling.py` — add a monitor glyph and color (an
   amber `⏱`, sitting beside the existing bash `🐚` / python `🐍` step glyphs so the
   family reads consistently), and render the row title from `monitor_label` with the
   command as the annotation.
4. `_agent_list_render_agent.py` — monitor rows show the live elapsed suffix while
   running and the exit-code badge (`✗ 1`) or timeout badge (`⧖`) when terminal.
5. `prompt_panel/_agent_display_content.py::get_phase_label()` — role `"monitor"`
   returns `MONITOR`.
6. `agent_family_members.py` — monitor members are **family members** (they appear in
   `concrete_family_member_rows()`, contribute to the FAMILY MEMBERS roster, and their
   settled/final bucket logic uses the override), but they are **not agents** for
   counting purposes: exclude them from `_concrete_agent_rows()` the same way
   python/bash workflow steps are excluded, and from agent counts in `sase stats` and
   the tribe/clan summaries. A family with one agent and one monitor is a one-agent
   family that ran one command.
7. Runtime: the monitor member carries `run_started_at` and a terminal marker, so
   `agent_runtime.rs` aggregation and `compute_row_runtime()` include it in the family
   total with no change. Verify `runtime_suffix_ticks()` returns True for a live monitor
   so the row's elapsed actually ticks, and add the case if it does not.
8. Integrations: `src/sase/integrations/_agent_list_entry_models.py` /
   `_agent_list_entry_builder.py` / `agent_list_entries.py` carry the monitor label,
   bucket, and a monitor marker so Telegram, the mobile summary, and
   `/sase_agents_status` describe monitors correctly rather than as mystery agents.

Tests: loader tests for running/terminal monitor rows; family roster ordering and phase
labels; family runtime includes the monitor's interval; agent counts exclude it; a
terminal monitor with a custom label does not keep its container Running.

## Phase `tui-detail`: Monitor detail panel, live output, and keybindings

1. **Detail section.** In the prompt panel, a selected monitor row renders a `MONITOR`
   section above the output: command (syntax-highlighted as shell), working directory,
   reason, next action, state, exit code, timeout budget vs elapsed, monitor id, and the
   `sase monitor show <id> --follow` pointer. Follow the existing fold-section idiom
   (`append_fold_section_heading`) so it folds with everything else.
2. **Live output.** `live_reply.md` for a monitor row must not be rendered as markdown —
   command output is not prose and backtick-heavy output renders as garbage. Render it
   as a plain, ANSI-aware log block (reuse `src/sase/ansi_style.py`), preserving the
   tail view and auto-follow behavior the panel already has for live replies. When
   `monitor_output_truncated` is set, render the elision marker visibly.
3. **Stop action.** The existing agent-kill action, invoked on a monitor row, calls
   `sase monitor stop` semantics rather than the agent kill path (a monitor has no LLM
   process to kill). Per `src/sase/ace/CLAUDE.md`, a keymap belongs in the footer only
   if it is conditionally available — a monitor-stop binding qualifies, so add it to
   `keybinding_footer.py` with a condition of "selected row is a running monitor",
   update the `?` help modal, and add the binding to `src/sase/default_config.yml`.
4. Update the help modal text describing family members to mention monitor members.
5. Visual snapshots: rendering changes will move PNG goldens. Run `just test-visual` and
   accept intentional diffs with `--sase-update-visual-snapshots`; inspect
   `.pytest_cache/sase-visual/` artifacts before accepting anything.

Tests: detail-section snapshot tests for running/completed/failed/timeout monitors; the
output block is not markdown-rendered; the footer binding appears only for running
monitors; the help modal lists it.

## Phase `epic-launch`: Approved-epic launch runs as a monitor

Today an approved epic is launched by whichever surface answered the gate: it calls
`prepare_epic_launch()` → `submit_epic_launch_task()`, which submits a detached
background task running `sase bead work <plan> --yes-to-all …`. The planner shows
`EPIC APPROVED`, and `EPIC CREATED` follows. **Do not break this.**

Change only the mechanism, with no epic-specific knowledge inside the monitor subsystem:

1. `src/sase/bead/epic_launch.py::submit_epic_launch_task()` becomes
   `start_epic_launch_monitor()`, which calls `sase.monitor.start.start_monitor()` with:
   - `command = shlex.join(build_epic_launch_argv(plan_file, artifacts_dir=…, cl_name=…))`
     — the argv builder is unchanged, so `--yes-to-all`, `--artifacts-dir`, `--cl-name`,
     and `--expect-prompt-snapshot` are passed exactly as today;
   - `cwd = resolve_epic_launch_cwd(...)` — still the project's **primary** workspace,
     so the monitor takes a zero workspace claim and does not hold the planner's
     workspace;
   - `lane` = the planner's lane, derived from the approving agent's name / artifacts
     dir in `host_action_data`;
   - `start_status = "EPIC APPROVED"`, `stop_status = "EPIC CREATED"`;
   - `reason = "Launch the approved epic from <plan name>"`;
   - `next_action = None` — `sase bead work` launches the phase agents itself;
   - `timeout` = a generous budget (4h) sized for a large epic launch;
   - `label = f"Epic launch · {Path(plan_file).stem}"`.
2. The `sase bead work` process still calls `finish_epic_launch()` itself
   (`src/sase/bead/cli_work_entry.py`), so approval metadata back-fill, the deferred
   completion payload, and `notify_workflow_complete` are untouched. This is why the
   change is low-risk: only the supervision changes.
3. Dedupe: `_active_epic_launch_for_plan()` is replaced by the generic same-command
   dedupe in `start_monitor()`. Keep the `epic-launch-submit` file lock around the
   check-then-start so two surfaces answering at once still cannot double-launch.
4. `can_claim_epic_launch()` now validates that a monitor can be started: the lane
   resolves and the primary workspace exists. Its contract — refuse the approval rather
   than accept it and silently drop the launch — is unchanged.
5. **Fallback.** If the planner's lane cannot be resolved (a very old artifacts layout,
   a wiped agent), fall back to today's detached-task submission rather than failing the
   approval. Log the fallback. This preserves the existing durability guarantee that an
   epic approved after its planner has been reaped still launches.
6. `src/sase/plan_approval_choices.py` — the epic consequence text currently reads
   "launch beads via `sase bead work` (background task; track it in `sase task list` or
   the ACE Tasks tab)". Update it to name the monitor and `sase monitor list` / the
   planner's family row.
7. The planner keeps its own `EPIC APPROVED` terminal status (the `epic_approved` loop
   outcome and its `done.json` mapping are unchanged); the monitor member is what moves
   `EPIC APPROVED` → `EPIC CREATED`. The lane therefore still ends at `EPIC CREATED`.

Tests: the existing epic-approval tests must pass with the monitor path
(`tests/test_plan_approve_cli.py`, the notification-gate adapter tests, the ACE modal
tests) — update their assertions from "a detached task was submitted" to "a monitor was
started", and keep an explicit test for the lane-unresolvable fallback. Add a test that
two concurrent approvals of the same plan start exactly one monitor.

## Phase `skill`: `/sase_monitor` skill

New source `src/sase/xprompts/skills/sase_monitor.md` (skills are auto-discovered from
that directory; do not hand-edit the generated `SKILL.md` files).

Frontmatter:

```yaml
---
name: sase_monitor
description:
  Run a long command without blocking your turn. Use this INSTEAD of any built-in
  monitor, background-task, or scheduled wake-up tool — those do not work in SASE, which
  runs agents for a single turn. Also use it to sleep/wait (for a CI job, a deploy, a
  rate limit) by monitoring a `sleep` command.
skill: true
log_skill_use: false
---
```

Body requirements:

- State plainly that `sase monitor start` kills the current agent, that the current turn
  will not return normally, and that the agent must not poll or wait for the command
  itself.
- Show the canonical invocation with a next action:

  ```bash
  sase monitor start \
    --command 'just check-full' \
    --reason 'Verify the refactor before replying to the user' \
    --timeout 45m \
    --next 'Fix anything just check-full reported, then reply to the user.'
  ```

- Show the sleep/wait form and its statuses, as requested:

  ```bash
  sase monitor start \
    --command 'sleep 300' \
    --reason 'Wait for the CI run on PR #412 to finish' \
    --timeout 6m \
    --start-status 'SLEEPING FOR 300s' \
    --stop-status 'SLEPT FOR 300s' \
    --next 'Check the CI status for PR #412 with `gh pr checks 412`.'
  ```

- Show the no-follow-up form (fire and forget) and note that the monitor still records
  everything for later inspection.
- Briefly describe the other subcommands: `stop` (stop a monitor; no follow-up agent is
  launched), `list` (active by default, `--all` for history), `show` (details, tail, and
  `--follow`).
- Explain what the follow-up agent receives: the previous conversation via `#fork`, the
  reason, the next action, and a full breakdown of the command run (outcome, exit code,
  elapsed, output tail, and the path to the full log).
- Explain the timeout: the follow-up agent still launches, and is told the command did
  not finish and what it had produced.

The `memory-docs` phase deploys it (`sase skill init`) after the source lands on the
canonical branch — never deploy from a dirty or unmerged tree.

## Phase `memory-docs`: Memory and documentation updates

1. `sase/memory/build_and_run.md` — the user explicitly approved this edit. Add guidance
   in the two-speed verification section:
   - `just check-full` must be run **only** as a sase monitor (`/sase_monitor` →
     `sase monitor start --command 'just check-full' …`), never inline, because it
     routinely outruns a single turn;
   - `just check` may be run inline, and should be handed to a monitor as well whenever
     it is taking a long time;
   - both cases should pass a `--next` action so the follow-up agent acts on the result.
2. Run `sase memory init` to regenerate `AGENTS.md`, the provider instruction shims
   (`CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`), and the memory README. This is
   mandatory and needs no further permission.
3. Deploy the skill from a clean, merged tree: `sase skill init --force`, then
   `chezmoi apply` if it was skipped. Preview with `sase skill init --diff` while
   iterating.
4. Docs: add `docs/monitors.md` (what a monitor member is, the lifecycle, the CLI, the
   family picture, and the epic-launch example) and register it in `mkdocs.yml` nav
   under "Beyond the Basics", next to Agent Families. Cross-reference from
   `docs/agent_families.md` (monitor members in a lane), `docs/cli.md` (the new command
   group), and `docs/ace.md` (the monitor row and detail panel).
5. Record a `PROPOSED FOLLOW-UP:` note on this phase's bead proposing a
   `sase/memory/glossary.md` entry for **Monitor Member** — the glossary is a memory
   file the user has not approved editing in this work, so propose it rather than
   writing it.

## Phase `smoke`: End-to-end monitor exercises

Launch real SASE agents (this is what `xsmall` is for) and observe the surfaces rather
than asserting in a unit test:

1. An agent that runs
   `sase monitor start --command 'sleep 60' --start-status 'SLEEPING FOR 60s' --stop-status 'SLEPT FOR 60s' --next 'Report that the sleep finished.'`
   — confirm the starter lands `DONE`, the monitor member shows `SLEEPING FOR 60s` with
   a ticking runtime in the ACE Agents tab, the family total runtime grows, and the
   follow-up agent launches in the same workspace with the previous conversation.
2. A failing command (`sh -c 'echo boom >&2; exit 3'`) — confirm the exit-code badge,
   the `Failed` bucket, and that the follow-up still launches with the failure in its
   breakdown.
3. A timeout (`--command 'sleep 120' --timeout 10s`) — confirm the timeout badge, that
   the process tree is dead, and that the follow-up is told the command did not finish.
4. `sase monitor stop` on a live monitor — confirm no follow-up launches and the
   workspace is released.
5. `sase monitor list` / `show --follow` in every format, and one real epic approval end
   to end (`EPIC APPROVED` → `EPIC CREATED` on the planner's lane, phases created).

Report exactly what each surface showed, including anything that looked wrong.

## Cross-cutting requirements

- Run `just install` before any other `just` command in a fresh workspace.
- Verify with `just check`; use `just check-full` before landing (and, once this epic
  has landed, do it through `sase monitor start`).
- `_lint-symvision` rejects unused and misused private symbols — read
  `sase/memory/symvision.md` with `/sase_memory_read` before fighting it. `_lint-toobig`
  caps file length: split modules up front.
- Do not hand-edit `AGENTS.md` or the generated provider instruction shims; they come
  from `sase memory init`.
- Everything user-facing must render project **names**, never ProjectSpec keys.
