---
tier: tale
title: Zero-weight wait monitors
goal:
  Agents can start pure-wait monitors with `sase monitor start -w 0` that hold no runner
  capacity, and the /sase_monitor skill teaches when to use them.
size: medium
decisions:
  monitor_row_badge:
    ask:
      Should ACE agent rows show a `w0` badge while their live monitor holds zero
      weight?
    choices:
      skip: No TUI change; the header load count already shows the freed slot
      badge:
        Monitor rows render w0; adds a TUI change, PNG snapshot checks, and a docs line
    default: skip
    why: the header load already shows freed capacity; skipping keeps this off TUI paths
    answer: badge
proposed_by: bbugyi200.athena.13u
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-24 14:53:19
status: wip
---

# Zero-weight wait monitors: `sase monitor start -w/--weight 0`

## Goal

Let an agent start a **pure-wait** monitor (`sleep N`, `gh run watch`,
`gh pr checks --watch`, a script that only polls something external) that holds **no
runner capacity** while it waits. Its `--next` follow-up then queues for capacity again
at the starter's normal weight. Then teach agents when to use it.

Source: the research report `research:202610/zero_weight_wait_monitors.md` (read it with
`sase artifact read`; do not open sidecar files directly). On this host, 192 pure-wait
monitors held about 76 capacity-hours at the starter's weight. In the motivating
snapshot, two of the eight occupied slots were monitors that were only waiting.

## What already exists (verified, do not rebuild)

- `StartMonitorRequest.queue_weight_override` in `src/sase/monitor/request.py`. It is
  already part of `monitor_request_fingerprint`, so an identical replay still matches.
- `create_monitor_member` (`src/sase/monitor/member.py`) records an override as an
  explicit `queue_weight` and sets `monitor_queue_weight_overridden`. It keeps the
  starter's original weight in `monitor_inherited_queue_weight` and
  `monitor_inherited_queue_weight_explicit`.
- `inheritable_queue_weight` (`src/sase/core/runner_slots/_queue_weight.py`) gives
  successors the pre-override weight. `queue_launch_prefix`
  (`src/sase/monitor/continuation_delivery.py`) builds the follow-up's live
  `%queue(...)` prefix from it. `sase monitor resume` uses the same path.
- Runner capacity is computed from live member metadata
  (`src/sase/core/runner_slots/_admission_capacity_records.py`). A monitor member with
  an explicit `queue_weight: 0` therefore occupies nothing.
- In sase-core, `claim_is_reusable` is `occupied_capacity != 0.0`
  (`crates/sase_core/src/runner_capacity/claims.rs`). Its tests
  `zero_weight_claim_is_not_reusable_by_a_serial_successor` and
  `explicit_zero_weight_monitor_occupies_nothing_at_a_full_limit` pin the rule. A
  follow-up behind a zero-weight monitor goes through `acquire_capacity` at its own
  weight and parks `QUEUED` when the host is full. **No sase-core change and no
  `sase-core-revision.txt` bump.**
- Follow-up launch (`src/sase/monitor/followup.py`) spawns the child and returns. The
  child adopts its continuation delivery from `src/sase/llm_provider/_invoke.py` only
  after runner admission. So a follow-up parked behind a full host is an ordinary waiter
  with no adoption deadline. Known cost: it gets a fresh `slot_requested_at`, so it
  queues behind earlier waiters with the same priority.
- Today only the host epic launch (`src/sase/bead/epic_launch.py`) sets the override.
  `sase monitor start` has no way to set it.

## Changes

### 1. CLI flag: `src/sase/main/parser_monitor.py`

Add the flag to `start_parser` after `-t/--timeout` and before the
`monitor_command_words` positional:

```python
start_parser.add_argument(
    "-w",
    "--weight",
    default=None,
    metavar="W",
    help=(
        "Queue weight for this monitor while it runs. Only 0 is accepted: a pure "
        "wait (sleep, CI watch, external poll) then holds no runner capacity, and "
        "the -n/--next follow-up queues again at your normal weight. Cannot be "
        "combined with -p verify, -f/--completion, -J/--join, or a `sase tool run` "
        "command"
    ),
)
```

Keep it a string, not `type=float`, so the handler can print its own message. Add an
epilog example after the deploy example:
`"  sase monitor start -w 0 -s 'WATCHING CI' -S 'WATCHED CI' -n 'check the CI result' -- gh run watch 123 --exit-status\n"`.
The short alias `-w` matches `%q(w=...)`. We accept that `sase proc run -w` means
`--wait`, as the research recommends.

### 2. Handler: `src/sase/main/monitor/start.py`

All new checks are pure argument checks. They exit `2` and must run **before** the
`will_handoff_monitor_to_agent_runner()` block that writes the in-flight handoff marker.

- **Parse the value** right after `refuse_finalizer_owned_monitor_start()`. When
  `args.weight` is not `None`, `float()` it. Accept it only if it is finite and
  `== 0.0`, so `0`, `0.0`, and `-0` pass. Reject anything else (`1`, `0.5`, `-1`, `nan`,
  `inf`, `abc`) with:
  `sase monitor start: -w/--weight accepts only 0 -- a zero-weight monitor holds no runner capacity while it waits; omit -w to keep your normal queue weight`.
  Only `0` is allowed because a monitor never passes through admission. Lowering its
  weight is always safe. Raising it would take capacity without queueing.
- **`-J/--join`**: in `_validate_join_start`, next to the existing `-f`/`-a` conflicts,
  return `JoinRefusal(exit_code=2, ...)` with
  `sase monitor start -J cannot be used with -w/--weight; a joined ToolRun is verification and keeps the starter's queue weight`.
  Pass the "weight given" fact in, or read `args.weight` there.
- **Verification conflicts**, once `profile`, `completion_ref`, and `command` are known
  (next to the existing `-p`/`-P` exclusivity check). When a weight was given, refuse:
  - `profile == VERIFY_MONITOR_PROFILE_NAME` → flag text `-p/--profile verify`
  - `completion_ref` set → `-f/--completion`
  - `command_invokes_tool_run(command)` (see item 3) → ``a `sase tool run` command``

  Use one message shape:
  `sase monitor start: -w/--weight 0 cannot be used with <flag text> -- verification keeps the starter's queue weight`.

- **Mapping**: pass `queue_weight_override=0.0` into `StartMonitorRequest` when the flag
  was given. Otherwise pass `None`, so existing behavior is byte-identical.
- **Summary**: in the human (non-`--json`) output, after the `timeout:` line, print
  `  queue weight: 0 (holds no runner capacity)` only when the flag was given. Leave the
  `--json` envelope unchanged, because `test_start_json_envelope_is_stable` pins it.

Leave `-P/--policy`, `-a/--agent`, and `-k/--checkpoint` unrestricted.

### 3. Tool-run detection: `src/sase/monitor/tool_wrap.py`

Add a public `command_invokes_tool_run(command: str) -> bool` and list it in `__all__`.
It returns `True` when `_split_command(command)` parses and `_has_wrapped_shape(parts)`
holds. That matches any `sase tool run` in the command, including compound commands like
`'sleep 60 && sase tool run check'`. An unparseable command returns `False`. Do not
import the `_`-prefixed helpers from another module, because symvision flags private
misuse.

### 4. Reference docs (budget: `docs/reference/automation/monitors.md` is about 5,490 of 6,000 words; add at most about 180 words there)

- **`docs/reference/automation/monitors.md`, flag list** (the bullets near `-r/--reason`
  / `-t/--timeout`): add a bullet after the `-r`/`-t` bullet: "`-w/--weight 0` makes a
  pure wait (a `sleep`, CI watch, or external poll) hold no runner capacity; see
  [Runner slots](#runner-slots). Only `0` is accepted."
- **`monitors.md` › Runner slots**:
  - Open with "By default a monitor does not free runner capacity." in place of "A
    monitor is not a way to free runner capacity."
  - Replace the paragraph that starts "The host-owned monitor that launches an approved
    epic is the exception" with one that names **two** zero-weight monitors:
    1. the host epic-launch monitor (its phase agents claim their own capacity), and
    2. an agent-authored pure wait started with `sase monitor start -w 0`.

    The paragraph should also say three things. Only `0` is accepted, because a monitor
    never passes through admission. The flag is refused with `-p verify`,
    `-f/--completion`, `-J/--join`, and a `sase tool run` command. Either zero applies
    to the monitor member only: successors inherit the starter's weight (or the default
    `1.0`). Keep the closing sentence "Only a user-authored `%q(w=0)` propagates zero
    weight to successors."

  - Add that a zero-weight monitor's follow-up has no positive claim to continue. It
    enters admission at the inherited weight and parks `QUEUED` when the host is full,
    behind earlier waiters of the same priority.
  - Amend "Serial session members — the monitor and any ordinary follow-up included —
    ride the claim the session already holds." so it names the exception: the follow-up
    of a zero-weight monitor queues instead.

- **`docs/reference/runner-slots.md`** (the paragraph ending "...a user-authored
  `%q(w=0)` is inherited by successors as an explicit zero weight."): say that both the
  epic-launch monitor and an agent's `sase monitor start -w 0` pure-wait monitor record
  an explicit zero and occupy no capacity. Also say that a wait monitor's follow-up
  re-enters admission at the starter's weight. Change "Unlike that host-set zero" to
  cover both monitor zeros.
- **`docs/reference/macros/directive-launch.md`** (the sentences starting "The
  host-owned monitor that launches an approved epic records an explicit zero
  weight..."): make the same two-monitor update. Change "while the monitor's host-set
  zero never leaks to them" to "while a monitor's zero never leaks to them".
- **`docs/reference/cli/monitor.md`** is generated. Regenerate it with `just fix`; never
  hand-edit it.

Do not edit Guide pages and do not move pages.

### 5. Skill source: `src/sase/macros/skills/sase_monitor.md`

Teach one rule: use zero weight only for a command that does nothing but wait on
something outside the host.

- **Decide Before You Start**: after "...or when it waits on CI, a deploy, a release, or
  a rate limit." add "A pure wait takes `-w 0`; see Sleep Or Wait."
- **Sleep Or Wait**: replace the opening paragraph and example with:

  ````markdown
  Use `sleep` as the monitored command when you need to wait for CI, a deploy, a rate
  limit, or a scheduled time. A pure wait uses no host resources, so pass `-w 0`: the
  monitor then holds no runner capacity while it waits, and the follow-up queues for
  capacity at your normal weight when the wait ends. The same applies to `gh run watch`,
  `gh pr checks --watch`, and scripts that only poll something external (a release tag,
  PyPI, another bead). Pair the wait with labels that name what is being waited on:

  ```bash
  sase monitor start \
    --reason 'Wait for the CI run on PR #412 to finish' \
    --timeout 6m \
    --weight 0 \
    --start-status 'SLEEPING FOR 300s' \
    --stop-status 'SLEPT FOR 300s' \
    --next 'Check the CI status for PR #412 with `gh pr checks 412`.' \
    -- sleep 300
  ```
  ````

  Keep the existing `--idle-timeout` sentence after it.

- **Useful Flags**: append "- `-w, --weight 0` frees this monitor's runner capacity for
  a pure wait (sleep, CI watch, external poll). The follow-up queues again at your
  normal weight. Only `0` is accepted, and it cannot be combined with `-p verify`, `-f`,
  `-J`, or a `sase tool run` command."
- **Hazards**: add "- Never pass `-w 0` for a command that does heavy local work
  (`just check`, builds, installs, screenshot regeneration, renders) or that polls and
  then builds. A zero-weight monitor is invisible to the capacity budget, so the host
  overcommits. Waiting on a human is not a monitor at all: use `/sase_gate` or
  `/sase_questions`."
- Leave the frontmatter `description` unchanged.
- **Do not deploy the skill**. Do not run `sase skill init --force`, because generated
  skills deploy only from a landed tree. `sase skill init --diff` is fine as a read-only
  preview.

### 6. Tests

Keep every touched test file under 700 lines (the toobig tiers are 700/850/1000). Use
new files where an existing one is near that limit.

- `tests/main/test_parser_monitor.py`: add `("-w", "--weight")` to
  `test_monitor_short_options_have_the_documented_long_aliases`. Add a parse test:
  `-w 0` gives `args.weight == "0"`, and omitting it gives `None`.
- New `tests/main/test_monitor_handler_start_weight.py`. Reuse
  `tests/main/monitor_handler_helpers.py` and follow the patterns in
  `test_monitor_handler_start_validation.py` and `test_monitor_handler_start_launch.py`.
  - Each of `1`, `0.5`, `-1`, `nan`, `abc` exits `2` with the "accepts only 0" message.
  - `-w 0 -p verify`, `-w 0 -f REF`, and `-w 0 -- sase tool run check` each exit `2`
    with the matching conflict message. So does a compound
    `-w 0 -- 'sleep 5 && sase tool run check'`.
  - `-w 0 -J RUN` inside an agent (`SASE_AGENT_NAME` set) exits `2` with the `-J`
    message. The conflict check runs before `inspect_join_target`.
  - A rejected `-w` start inside an agent leaves no in-flight handoff marker. Mirror
    `test_monitor_handler_start_inflight.py`.
  - `-w 0` and `-w 0.0` reach `start_monitor` with
    `request.queue_weight_override == 0.0`. Omitting `-w` gives `None`. Capture the
    request by monkeypatching the `start_monitor` that the handler module imported.
  - The human summary prints `queue weight: 0 (holds no runner capacity)` only when `-w`
    was given.
- `tests/monitor/test_monitor_tool_wrap.py` (620 lines; add a few small cases or a
  sibling file): `command_invokes_tool_run` is true for `sase tool run check`,
  `/usr/bin/sase tool run -- x`, `python -m sase tool run check`, and the compound
  command. It is false for `sleep 300`, `gh run watch 1 --exit-status`, `tool run x` (no
  `sase`), and an unbalanced quote.
- `tests/main/test_init_skills_sources.py`, `sase_monitor` entry: add the substrings
  `"--weight 0"`, `"-w, --weight 0"`, `"Never pass \`-w 0\`"`, and `"holds no runner
  capacity"`.
- **New fakey end-to-end test**:
  `tests/fakey/test_monitor_capacity_zero_weight_wait_e2e.py`. Do not add it to
  `test_monitor_capacity_e2e.py` (621 lines). Move `_FakeSupervisorPid` and
  `_make_bootstrap_popen` into a new shared `tests/fakey/_monitor_capacity_helpers.py`
  with public names, following the `tests/fakey/_gate_capacity_helpers.py` precedent.
  Point the existing file at it. Model the test on
  `test_weight_two_land_agent_session_retains_one_claim_through_real_dispatch_and_delayed_child_bootstrap`:
  1. Use harness `cap=2`. Create a starter `wait--0` (agent session `wait`,
     `queue_weight=2.0`, explicit), with `workflow_name` and `continuation_node_id` set
     as in that test. Start it and wait until it starts.
  2. Call
     `start_monitor(StartMonitorRequest(..., lane="wait", inherit_lane_workspace_claim=False, next_action="...", queue_weight_override=0.0))`.
     Assert the monitor meta has `queue_weight == 0.0` with `queue_weight_explicit`,
     `monitor_queue_weight_overridden` is true, and
     `monitor_inherited_queue_weight == 2.0`.
  3. Release and join the starter. Then a weight-2 `filler` agent must be **admitted**
     (`wait_started`), which proves the live monitor occupies nothing. The host is now
     full.
  4. Patch `spawn_agent_subprocess` with a fake that parses the follow-up prompt's
     `%queue` prefix with `extract_prompt_directives` and asserts weight `2.0`. The fake
     then creates the successor (`parent_timestamp` = the monitor artifacts dir name,
     agent session `wait`, weight and priority from the directives), starts it, asserts
     `wait_parked`, saves `extra_env`, and returns an `AgentLaunchResult`.
  5. Call `settle_named_proc(... status="success" ...)`, then
     `wait_for_done(...)["monitor_state"] == "completed"`. The successor must still be
     parked (`assert_parked_not_started`).
  6. Release the filler. The successor must now start. Only then call
     `adopt_ordinary_continuation_delivery(env=saved_env, agent_name=...)` and assert
     `disposition == "acknowledged"`. Production adopts after admission, and this order
     matches it.
  7. A late weight-2 competitor parks while the successor runs, which proves the
     successor holds a real 2.0 claim. Release the successor, and the competitor starts.
     Clean up the dummy supervisor in `finally`.

  Annotate any fixed sleep with the `# sase-test-wait: <why>` comment that the lint
  requires. If the harness shows behavior that differs from this sequence (for example,
  the successor is admitted while the filler still runs), stop. Record the observed
  behavior as a `PROPOSED FOLLOW-UP` in your final report rather than weakening the
  assertions. That would mean the research's admission claim is wrong.

### 7. Optional TUI badge (only if `monitor_row_badge = badge`)

> [!decision] monitor_row_badge = skip Make no TUI change. Skip this section.

> [!decision] monitor_row_badge = badge Before editing, read `tui.md`, `tui_perf.md`,
> and `tui_screenshot.md` with `/sase_memory_read`.
>
> - In `src/sase/ace/tui/widgets/_queue_weight_badge.py`,
>   `_append_agent_queue_weight_badge` returns early for `agent.is_monitor` and for
>   non-parallel child rows. Find which `Agent` backs the row that shows a live
>   monitor's start status (the monitor member itself, or the session row through
>   `wait_display_source`). Then render `w0` only when that live monitor member records
>   `queue_weight == 0` with `queue_weight_explicit`. Keep inherited positive weights
>   hidden on monitor rows, because they are the session's one shared claim and the
>   starter row already shows it. Epic-launch monitors will also show `w0`, which is
>   correct. Leave the capacity (`cN`) badge logic alone.
> - Add unit tests next to
>   `tests/ace/tui/widgets/test_queue_capacity_multiplier_display.py`: `w0` shows on an
>   explicit-zero monitor and stays hidden on a monitor with an inherited 1.0 or 2.0.
> - Run targeted PNG snapshot checks for monitor-bearing visual tests (for example
>   `tests/ace/tui/visual/test_ace_png_snapshots_agents_agent_session_panel_monitor.py`)
>   with `just fix-tui-screenshots -- <selectors>`, through `/sase_monitor` if it can
>   outlast your provider's limit. Inspect every changed golden before accepting it.
> - Update the badge sentence in `docs/reference/tui/agents-tab-grouping.md` ("...an
>   explicit zero renders `w0`") to say that a zero-weight monitor row shows `w0` too.

## Verification

1. `just fix`. This formats code, regenerates `docs/reference/cli/monitor.md`, and runs
   the docs-budget ratchet. Confirm the regenerated CLI page lists `-w, --weight W`.
2. `sase tool run check`. Run it inline. Hand it to `/sase_monitor` with `-p verify`
   only if it escalates, using the printed `-J` command. Do **not** run
   `just check-full`.
3. Make sure the new fakey test and the new handler test file actually ran, either in
   the scoped lane or by an explicit `.venv/bin/pytest <files>`. Report their result.
4. `sase skill init --diff`, as a read-only preview that the skill renders.

## Out of scope (deliberate)

- **No bead macro edits** (`bd/work_phase_bead`, `bd/land_epic`, `bd/work_task` in
  `src/sase/default_config.yml`). None of them mention monitors, and agents learn flags
  from `/sase_monitor`.
- **No start-time hint for bare `sleep`/`gh run watch` without `-w`.** After a
  successful start, `sase monitor start` hands off and kills the starter's runner group
  before the agent reads the command's stderr. The hint could never change the agent's
  behavior and would only reach humans reading logs. Revisit as a follow-up-prompt nudge
  or auto-zero only if adoption measurement shows agents skip the flag.
- **No Muse directive edit.** Muse loads `/sase_monitor` like the other providers, and
  the directive is paid on every Muse turn.
- No auto-zero by command shape, no `-p wait` profile, no weights above zero, no
  `--json` envelope change, no sase-core change, and no memory changes. No memory note
  states the old "monitors never free capacity" rule.

## After landing (human-only; the coder must not run these)

1. Install the landed `sase` globally (`just install`). Global installs are human-only.
2. Then, from the clean merged tree, deploy the skill: `sase skill init --force` (plus
   `chezmoi apply` if it was skipped). This order keeps the skill from teaching a flag
   the installed CLI lacks.
3. After one to two weeks, rerun the research's monitor-history classification to see
   what share of wait-shaped monitors used `-w 0`.
