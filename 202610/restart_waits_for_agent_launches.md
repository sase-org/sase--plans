---
tier: tale
title: Hold update restarts for in-flight agent launches
goal:
  ACE update, feature-flag, and Restart ACE restarts wait (up to a bounded 5-minute cap)
  for agent launches this TUI accepted or submitted, instead of silently cancelling an
  accepted swarm launch into the stash, while independent commands still never block.
size: medium
proposed_by: bbugyi200.apollo.50
create_time: 2026-10-04 08:38:30
status: wip
---

# Plan: Hold update restarts for in-flight agent launches

## Diagnosis (confirmed from on-disk evidence on apollo, 2026-10-04)

- 12:24:29Z: the prompt bar accepted a
  `+sase #research_swarm(gemini=true,grok=true,muse=true,...)` prompt. The toast log
  (`~/.sase/logs/tui_toasts.jsonl`) shows "Launching agent for gh_sase-org__sase...".
- 12:25:11Z: the `,E` update completed ("SASE, core & plugins: Updated sase dev checkout
  in 5.9s ... — restarting ACE to load new code."). There was **no** "restart queued
  until ..." toast, so `collect_restart_blockers` returned nothing.
- 12:25:11.63Z: the swarm prompt was written to `~/.sase/prompt_stash.jsonl` with
  `source: "failed_launch"`. That comes from `flush_pending_launch_stashes`, which
  `_flush_then_do_quit` runs on every controlled exit, restarts included.
- `~/.sase/procs/procs.jsonl` has no `launch sase` row between 11:55Z and 12:28Z. The
  swarm never reached `sase run`, and the post-restart TUI said nothing about it.

The suspicion is right, with one correction. The swarm was not yet a running proc. It
was an accepted **pending launch**: prompt bar retired, a `PREPARING` launch record, and
a pending placeholder row `launch sase · <stage>`. It was still in post-acceptance
preflight. MUSE has an active _hard_ provider disable, so `_preflight_provider_disables`
ran `plan_launch_units` in a worker. For this exact prompt that takes 5–12 s standalone
on apollo. The time goes to `require_latest_agent_name_template` → name-registry
staleness checks, plus ~20k `match_agent_name_template` calls. Inside the busy TUI
process it takes longer. That stage shows no modal, so `,E` stayed usable.

**Root cause:** commit `458dfe59dc` ("restart TUI without waiting on independent
commands", already in the running build) narrowed restart blockers. They are now
session-overlay rows, `_session_workers`, `_durable_submit_workers`, active
install-mutation rows, and local legacy TUI rows. Before that commit,
`running_background_procs` waited on every active projection row except monitor turns
and service daemons. That included the pending-launch placeholder (`proc_type="launch"`,
status `pending`) and the durable `sase run` launch row. Neither blocks now, so:

1. An accepted-but-unsubmitted launch is cancelled by the restart's exit flush and
   silently dumped into the stash (this incident).
2. An in-flight `sase run` launch proc survives, because it has a detached supervisor.
   Its TUI-side completion handling is lost: the "Started N agent(s)" toast,
   launch-record results that power `,X`, deferred-kill application, and dispatch
   outcome recording.

Separately, even when launches are counted, the 60 s `_RESTART_WAIT_SECONDS` cap is too
short for a swarm. Preflight takes 10–40 s. `sase run` took ~42 s for the 11:55Z
research swarm, and epic launches take up to ~90 s.

## Goal

Every restart that goes through `restart_after_update_when_ready` waits for agent
launches this TUI owns: `,E`/`,U` update restarts, Config > Flags restarts, and the `,U`
"Restart ACE" row. Owned launches are pending launches plus this session's in-flight
`sase run` launch procs. The wait gets a longer but still bounded cap and visible copy.
Independent work keeps its current non-blocking behavior: tool runs, ordinary durable
commands, oneshots, monitor turns (including monitor-run epic launches), service
daemons, and other sessions' procs.

## Changes

### 1. Launch-row predicate — `src/sase/ace/tui/_proc_observer_models.py`

- Add `ACE_LAUNCH_SCOPE_PREFIX = "ace:launch:"` and `is_ace_launch_row(row) -> bool`. It
  returns True when either:
  - `row.proc_type == "launch"`, which covers pending-launch placeholders from
    `begin_pending_launch` and durable-submit placeholders from `submit_agent_launch`;
    or
  - any `row.exclusive_scopes` entry starts with the prefix. This covers store-backed
    `sase run` rows, whose `proc_type` is the store kind `command` and whose concurrency
    key comes from `launch_concurrency_key`.

  Return False for monitor-turn and service-daemon rows (same guard as
  `is_install_mutation_row`). Callers still gate on active status.

- Export the constant and helper next to `is_install_mutation_row`, including the
  re-export list in `src/sase/ace/tui/proc_observer.py`.
- Make `launch_concurrency_key` in `src/sase/ace/tui/durable_ops.py` build from that
  constant so the prefix has one source of truth. If the import would create a cycle or
  break the import budget, keep the literal and add a test asserting
  `launch_concurrency_key("x").startswith(ACE_LAUNCH_SCOPE_PREFIX)`.

### 2. Launch blockers — `src/sase/ace/tui/update_restart.py`

- Extend `RestartBlockerKind` with `"launch"`. Add a `"launch"` group:
  - in `_KIND_GROUP`;
  - in `_GROUP_NOUNS` as `("agent launch", "agent launches")`;
  - in `_GROUP_ORDER`, after `"tui"` and before `"submission"`.
- In `collect_restart_blockers`, first add every live entry of `app._pending_launches`.
  - It is a dict. Skip entries whose `cancelled` or `submitted` is true, with the same
    defensive `getattr` style as `quit_impact._pending_launch_count`.
  - Identity is `placeholder_id or launch_id`, with `launch_id` as an alias.
  - Label is `launch <context.display_name>`, or `bulk N Patches` for bulk launches,
    followed by the stage message. Example: `launch sase · checking providers`. Expose
    the stage text with a small public helper in
    `src/sase/ace/tui/actions/agent_workflow/_pending_launch.py`, e.g.
    `pending_launch_stage_message(launch)`, backed by `_STAGE_ROW_MESSAGES`. Fall back
    to the bare label.
  - Kind is `"launch"`.
  - Stay I/O-free and import lazily inside the function. `update_restart.py` is in the
    app import closure, so `tests/ace/tui/test_app_import_budget.py` must stay green.
- In the `_durable_submit_workers` loop, classify a worker as `"launch"` instead of
  `"submission"` in either case:
  - its cached row satisfies `is_ace_launch_row`;
  - `getattr(app, "_proc_pending_scopes", {}).get(placeholder_id)` contains an
    `ace:launch:` scope. This covers projection lag.

  The kind has to stay `"launch"` through the whole pending → submitting → durable
  handoff. Otherwise an expired generic deadline could restart mid-submit and discard
  the submission.

- In the cached-rows loop, after the install-mutation and legacy-TUI checks, add active
  `is_ace_launch_row` rows owned by this TUI as `"launch"`. Owned means either:
  - a non-store-backed placeholder; or
  - a store row whose `session_id` equals `_current_session_id(app)`.

  Generalize the session test inside `_is_local_legacy_tui` into a shared helper, e.g.
  `_row_is_local(row, session_id)`. Rows from other sessions must not block, and neither
  must monitor-originated epic launches.

- Rely on the existing `add()` identity/alias dedup so one launch is reported exactly
  once as it moves pending → submitting → durable.

### 3. Longer bounded wait for launches — `src/sase/ace/tui/update_restart.py`

- Add `_LAUNCH_RESTART_WAIT_SECONDS = 300.0`. Track a `launch_deadline` (queue time +
  300 s) next to the existing `deadline`. Store it in `_pending_restart_chain` and
  thread it through the deferred `set_timer` callback.
- On each tick, the effective deadline is `launch_deadline` when any current blocker has
  kind `"launch"`, and `deadline` otherwise.
- The restart proceeds only when there are no blockers or the effective deadline has
  passed. A coalesced second request (`existing` chain) keeps the original deadlines.
- Recompute the published `restart_by` (`_publish_pending_restart`, which feeds the
  yellow-gear tooltip) from the effective deadline on every tick instead of freezing it
  at queue time. Keep `queued_at` stable.
- Expiry copy. If launch blockers remain at a forced restart and `_pending_launches`
  still holds unsubmitted entries, the existing "restart wait expired" warning must say
  those prompts go to the stash. For example:
  `... restarting with 1 agent launch still active: launch sase · checking providers. Unsubmitted launch prompts are saved to the stash (press @ after restart).`
- Normal queued copy then reads, e.g.,
  `... - restart queued until 1 agent launch finishes.`

### 4. Docs

Update the restart-wait wording in two places:

- `docs/ace.md`: the update-restart paragraph (~L9102), the "Restart ACE" row (~L9036),
  the yellow-gear description (~L5181), and the Config > Flags confirmation paragraph
  (~L3578).
- `docs/configuration.md` (~L5213).

New wording: agent launches the TUI accepted or submitted also hold the restart, for up
to 5 minutes instead of 60 seconds, and independent commands still keep running. Leave
the Feature Flags confirmation-modal string in `feature_flags_pane_rendering.py`
unchanged ("TUI tasks" already covers launches). That avoids PNG-golden churn.

## Tests

In `tests/ace/tui/test_update_restart.py`, extend the `_App`/`_PendingApp` fixtures with
optional `_pending_launches` and `_proc_pending_scopes` attributes. Use lightweight
`SimpleNamespace` launches carrying `launch_id`, `placeholder_id`,
`context.display_name`, `stage`, `bulk_patches`, `cancelled`, and `submitted`. Then add:

- A live pending launch blocks with kind `"launch"`, and its label includes the stage
  text. Cancelled or submitted entries don't block.
- A pending placeholder row (`proc_type="launch"`) plus a registry entry with the same
  placeholder id yields exactly one blocker.
- A durable submit worker whose placeholder is a launch has kind `"launch"`, including
  the case where only `_proc_pending_scopes` identifies it. A non-launch submit worker
  still has kind `"submission"`.
- An active store row (`proc_type="command"`, `exclusive_scopes={"ace:launch:..."}`,
  `store_backed=True`, `session_id="session-a"`) blocks. The same row with
  `session_id="session-b"`, a terminal status, or monitor origin does not. The existing
  `test_ordinary_durable_work_does_not_block_restart` and the tool-run tests stay green.
- Deadline, with `time.monotonic` monkeypatched:
  - Only a launch blocker: a deferred tick at queue+61 s re-arms the timer with no
    restart. At queue+301 s it restarts with the expiry warning that mentions the stash.
  - Only a session-worker blocker: queue+61 s still restarts (existing behavior).
- The published `PendingUpdateRestart.restart_by` uses the 300 s horizon while a launch
  blocks and drops back to the 60 s horizon once only non-launch blockers remain.
- Wait copy: `"restart queued until 1 agent launch finishes."`, plus a mixed-kind join
  such as `"1 TUI task and 1 agent launch"`.

Unit-test `is_ace_launch_row` next to the existing `is_install_mutation_row` tests (e.g.
`tests/ace/tui/test_proc_gear_lanes.py` or the proc-observer model tests). Cover the
pending placeholder, the durable-submit placeholder, the store row with a launch scope,
and a monitor-origin row (→ False).

## Verification

- `just fmt`, then `just check`. Do **not** run `just check-full`.
- Optional manual check: with a hard provider disable active, submit a prompt and
  immediately choose `,U` → Restart ACE. Expect the "restart queued until 1 agent launch
  finishes" toast and the yellow gear, then the restart right after "Started N
  agent(s)".

## Out of scope / follow-ups

- `plan_launch_units` takes 5–12 s for an 8-unit research swarm. Name-registry staleness
  checks and template matching repeat per unit, and it runs on every launch while any
  provider is hard-disabled. That is what widened this window. File it as a task bead
  via `/sase_new_task` (performance; it likely belongs in the name-registry /
  launch-guard code and possibly sase-core), and don't fix it here.
- Manual quit and quit-menu restart behavior is unchanged: pending launches are stashed,
  with the existing "will be stashed for @" summary line.
- Whether the update _install_ step should also wait for in-flight `sase run` launches
  is a separate question (it swaps code under a running launcher). Don't change it here.
