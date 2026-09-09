---
tier: epic
status: done
title: Drain stranded agents when an LLM provider is disabled
goal:
  Disabling a provider — automatically on a usage limit or by hand in Launch Control —
  relaunches the agents that provider stranded, and says exactly what moved and what did
  not.
phases:
  - id: engine
    title: Provider-drain planning and execution engine
    depends_on: []
    size: medium
    description:
      "engine: select the agents a disabled provider stranded, classify each one's
      reroute or refusal through the existing restart and launch-guard machinery, and
      execute the moves."
  - id: cli
    title: sase agent drain command and durable operation
    depends_on:
      - engine
    size: medium
    description:
      "cli: add the `sase agent drain` subcommand with preview, confirmation, receipt,
      and JSON envelope, and register it as the `agent.drain` durable operation."
  - id: auto
    title: Automatic drain on a usage-limit disable
    depends_on:
      - cli
    size: medium
    description:
      "auto: gate the feature behind a beta flag, submit a drain proc when a usage-limit
      disable wins its first-writer window, and make that drain own the single enriched
      usage-limit notification."
  - id: ace
    title: Launch Control relaunch prompt after a manual disable
    depends_on:
      - cli
    size: medium
    description:
      "ace: after a manual hard disable in the Models panel, offer a single-keypress
      relaunch chooser and submit the drain as a durable tracked proc."
  - id: soak
    title: End-to-end drill and reference documentation
    depends_on:
      - auto
      - ace
    size: small
    description:
      "soak: prove the whole loop through the fakey provider end to end and document
      draining in the LLM provider reference."
proposed_by: bbugyi200.athena.0ce
bead_id: sase-su
create_time: 2026-09-09 19:51:17
---

- **PROMPT:**
  [prompts/202608/provider_drain.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/provider_drain.md)
- **BEAD:**
  [sase-su](https://github.com/sase-org/sase--beads/blob/main/pages/sase-su/README.md)

# Plan: Drain stranded agents when an LLM provider is disabled

## 1. Context

SASE already detects provider usage limits and writes a temporary machine-wide provider
disable. `sase.llm_provider.usage_limit_disable.handle_possible_usage_limit()` matches
the provider's error text, wins a first-writer disable window through
`try_disable_provider*`, and sends one notification through
`sase.notifications.senders.notify_provider_usage_limit_disabled()`. New launches then
route around the provider: `sase.agent.launch_guard` refuses a unit whose only route is
a hard-disabled provider, and model-alias resolution skips disabled pool members.

Nothing happens to the agents that are already alive. The agent that hit the limit
raises and dies. Every other RUNNING, STARTING, and WAITING agent on that provider keeps
going and fails on its next LLM call. The user learns a provider went away and is left
to find and relaunch each stranded agent by hand.

The relaunch machinery to fix this already exists and is well factored:

- `sase.agent.restart` — `plan_agent_restart()` is read-only and discovers every refusal
  (`not_found`, `no_prompt`, `multi_segment`, `fanout`, `container`, `identity`,
  `name_not_reusable`, `preflight`) before anything is killed; `execute_agent_restart()`
  snapshots a recovery bundle, stops the row (kill when live, dismiss when done),
  releases the name, and relaunches the rewritten prompt from the home directory.
- `sase.agents.cli_restart` — the `sase agent restart` CLI with preview, confirmation,
  receipt panel, JSON envelope, and the 0/1/2 exit contract.
- `sase.agent.launch_guard.plan_launch_units()` — resolves a prompt's fan-out slots to
  concrete `(provider, model)` candidates against the live disable snapshot, which is
  exactly the question "where would this agent land if I relaunched it right now?".
- `sase.agent.running_listing.list_all_agents()` — one snapshot scan carrying live rows
  and recent DONE/FAILED rows, including `llm_provider`, status, monitor role, and
  `done.error` / `done.finished_at`.
- `sase.procs.submit_proc_request()` — durable, supervisor-owned background commands
  with `concurrency_keys`, visible and killable in ACE's Procs tab.
- `sase.ops` — typed durable operation requests and result envelopes, already used by
  `sase agent revert` / `persist-cleanup` / `persist-directive`.

This epic composes those parts into one named capability instead of adding a new
subsystem: **provider drain**. Draining a provider relaunches every agent it stranded
and reports every agent it could not move.

## 2. Facts a phase worker must not rediscover the hard way

1. `handle_possible_usage_limit()` runs **inside the process that just failed**. It must
   not perform the drain itself: that process is mid-teardown, would have to kill
   itself, and still holds a workspace claim. The drain runs out of process.
2. Only a **hard** disable strands anything. A soft disable spares a provider in pools
   but still allows launches, so it never drains. The soft-enable path in
   `sase/ace/tui/actions/agent_workflow/_launch_provider_guard.py` writes soft disables
   and must not trigger a prompt.
3. `execute_agent_restart()` deletes the previous run's artifacts (the chat transcript
   under `~/.sase/chats` survives). Draining a RUNNING agent therefore loses in-flight
   progress. That cost is real, is the point of the feature, and must be stated in every
   surface that offers it.
4. Monitor member rows (`RunningAgentInfo.is_monitor`) are supervising a shell command,
   not burning provider quota. Killing one kills the monitored command. Never drain
   them.
5. A row in `QUESTION` / `ANSWERED` status is holding a pending user interaction that a
   restart would destroy. Never drain it; report it.
6. Alias resolution already routes around a hard-disabled provider, so most relaunches
   land on a different provider with no extra work. A prompt pinned to `provider/model`
   has nowhere to go — it is **stranded**, and the honest answer is to report it, not to
   guess a substitute model on the user's behalf.
7. Cascades terminate on their own: a drained agent that hits a limit on its new
   provider disables that provider too, and the next drain finds no available route and
   reports stranded rows. Do not add a cross-provider cascade counter.
8. This epic introduces **no new durable state**. At-most-once is already provided by
   the first-writer disable window plus the proc `concurrency_keys` fence; results live
   in the proc result envelope and the notification.

## 3. Decisions a phase worker must not silently revert

1. **The relaunch is `sase.agent.restart`.** Do not fork the restart flow, do not
   reimplement prompt rewriting or name reuse, and do not reach for
   `sase.axe.run_agent_retry_spawn` (it is only reachable from inside a failing runner).
   Every refusal `plan_agent_restart()` already raises becomes a reported skip reason.
2. **Route classification is `plan_launch_units()`**, not a hand-rolled model parse. If
   the rewritten prompt's units are all blocked, the agent is stranded; otherwise the
   resolved candidate provider is the reported destination.
3. **One notification, owned by the drain.** When a usage-limit disable submits a drain,
   the drain sends the (enriched) usage-limit notification and the failing process sends
   none. When submission fails, the failing process sends today's notification inline.
   Exactly one notification per disable window, in both paths.
4. **The CLI is the only executor.** Both the automatic path and ACE submit
   `sase agent drain`; neither calls the engine in-process. One code path gets
   exercised, debugged, and replayed.
5. **`sase agent drain` refuses a provider with no active hard disable** (exit 2, with a
   hint pointing at Launch Control). Draining an enabled provider would relaunch agents
   straight back onto it.
6. **Ship behind `sase flag new provider_drain -k beta`.** The flag gates the automatic
   drain and the ACE prompt. The `sase agent drain` CLI is always available so the
   capability is usable and testable while the automation bakes. Test both states from
   `auto` onward.
7. **Never drain the calling agent.** `sase agent drain` invoked from inside an agent
   skips that agent (`sase.agent.identity`); it must not kill its own caller.
8. **No silent caps.** A drain bounded by `--limit` reports what it dropped, in the
   receipt, the JSON envelope, and the notification.

## 4. Implementation

### Provider-drain planning and execution engine

Add `src/sase/agent/provider_drain.py` as the seam callers import, with the work split
across private siblings the way `sase.agent.restart` splits its own (`_drain_types.py`,
`_drain_selection.py`, `_drain_planning.py`, `_drain_execute.py`). Keep every module
under the repository's line cap.

Types (`_drain_types.py`):

- `ProviderDrainError(reason, message, hint)` — a refusal raised before any mutation,
  mirroring `AgentRestartError`.
- `DrainRoute` — `kind: Literal["reroute", "stranded"]`, `target_provider`,
  `target_model`.
- `ProviderDrainMove` — `name`, `presented_name`, `project`, `status`, `route`,
  `restart_plan: AgentRestartPlan`.
- `ProviderDrainSkip` — `name`, `presented_name`, `status`, `reason`, `detail`. Reason
  slugs are a closed set: `monitor`, `pending_question`, `caller`, `stranded`, `capped`,
  plus every `AgentRestartError.reason` value passed through verbatim.
- `ProviderDrainPlan` — `provider`, `disable: TemporaryProviderDisable`, `moves`,
  `skips`, `model_override`, `limit`.
- `ProviderDrainOutcome` — `plan`, `results: tuple[AgentRestartOutcome, ...]`, and
  derived `relaunched` / `failed` counts.

Selection (`_drain_selection.py`) takes one `list_all_agents()` snapshot and returns
ordered candidates for a provider:

1. Effective provider is `exec_llm_provider` when `agent_meta.json` records one, else
   the listed `llm_provider`. Read the artifacts JSON only for rows that are otherwise
   candidates; do not widen the scan wire.
2. Live candidates: status in `{"STARTING", "RUNNING", "WAITING"}`.
3. Recently-failed candidates: status `FAILED` whose `done.finished_at` is at or after
   `disable.created_at - _FAILED_GRACE_SECONDS` (module constant, 300) and whose
   recorded `done.error` matches that provider through
   `sase.llm_provider.usage_limit_config.detect_usage_limit()`. This reuses the matcher
   that caused the disable, so a manual disable naturally selects nothing unless the
   user disabled the provider because they watched agents fail on it.
4. Drop and record: monitor members (`monitor`), `QUESTION` / `ANSWERED` rows
   (`pending_question`), and the calling agent (`caller`).
5. Order least-progress-first — `WAITING`, `STARTING`, `FAILED`, `RUNNING` — so the
   cheapest moves happen first and a `--limit` truncation costs the least. Ties break on
   the listing's existing most-recent-first order.

`plan_provider_drain(provider, *, model_override=None, limit=..., now=None)`
(`_drain_planning.py`) is read-only:

1. Resolve the active disable with `get_active_provider_disable()`. Raise
   `ProviderDrainError(reason="not_disabled")` when absent and `reason="soft_disabled"`
   when the record is soft.
2. For each candidate, call `plan_agent_restart(name, model_override=model_override)`.
   Convert an `AgentRestartError` into a `ProviderDrainSkip` carrying that error's
   reason, message, and hint — planning never raises for one bad row.
3. Classify the route by calling `plan_launch_units(plan.rewritten_prompt)`. All units
   blocked (`unit.blocked`) → `DrainRoute(kind="stranded", ...)` recorded as a skip with
   reason `stranded` and a detail naming the pinned `provider/model`. Otherwise take the
   first unit's resolved candidate as `target_provider` / `target_model`.
4. Apply `limit` after classification and record each dropped row as a `capped` skip.

`execute_provider_drain(plan, *, progress=None)` (`_drain_execute.py`) runs the moves
sequentially through `execute_agent_restart()`, emitting the same
`(step, status, detail)` progress triple `ProgressFn` already defines, and never raising
for one failed move: a `kill_failed` / `wipe_failed` / `partial` outcome is collected
and the drain continues. Sequential is deliberate — launch admission and runner slots
throttle the relaunches, and a parallel storm would fight them.

Tests (`tests/test_agent_provider_drain_selection.py`,
`tests/test_agent_provider_drain_plan.py`, `tests/test_agent_provider_drain_execute.py`)
cover: effective-provider precedence, the failed-agent window and grace boundary,
monitor and pending-question exclusions, caller exclusion, ordering, each pass-through
restart refusal, reroute versus stranded classification, `--limit` truncation
accounting, and a mid-drain move failure not aborting the rest.

### `sase agent drain` command and durable operation

Register the subcommand in `src/sase/main/parser_agent_lifecycle.py`
(`register_agent_drain_parser`), wire it alphabetically in
`src/sase/main/parser_agent.py`, and dispatch it in `src/sase/main/agent_handler.py`
(including the usage string). Positional `PROVIDER`; every option optional,
short-aliased, and listed alphabetically per `sase/memory/cli_rules.md`:

- `-j, --json` — one stable envelope on stdout and nothing else; implies no
  confirmation.
- `-l, --limit N` — most agents to move (default 20).
- `-m, --model MODEL` — relaunch every moved agent on this model, accepting the same
  `[provider/]model[@effort]` spelling `sase agent restart -m` accepts. This is the
  answer for stranded agents: pointing the drain at a reachable model turns them into
  ordinary moves.
- `-n, --dry-run` — print the preview and exit 0 without killing or launching.
- `-y, --yes` — skip the confirmation.

Also call `sase.ops.cli.add_operation_io_flags()` on the parser so ACE and the automatic
path can drive the same command as a durable operation, exactly like
`sase agent revert`.

Add `AGENT_DRAIN = "agent.drain"` to `src/sase/ops/names.py` and a `drain` runner to
`src/sase/ops/commands/agent.py` that returns a typed result payload (`provider`,
`relaunched`, `failed`, `skipped` counts plus per-agent rows) through
`run_and_finish()`.

Handler `src/sase/agents/cli_drain.py` mirrors `cli_restart.py`'s shape and injection
seams (`plan_fn`, `execute_fn`, `confirm_fn`, `is_tty_fn`) so tests drive it without
processes. Rendering lives in `src/sase/agents/_drain_render.py` and must be beautiful,
not merely correct:

- A preview panel titled with the provider's display name, its remaining disable window
  (`remaining_label` from `sase/ace/tui/modals/models_panel_provider_state.py`) and its
  provenance (`provider_disable_provenance_label`), then a table of moves — status badge
  (reuse `sase/agents/status_style.py`), agent name, project, and a
  `claude/opus → codex/gpt-5` route arrow — followed by a dim skip table grouped by
  reason.
- A live progress line per move while executing, then a receipt panel summarizing
  `N relaunched · M left alone · K failed`, listing any recovery directories
  `execute_agent_restart()` produced.
- Confirmation is required when the plan moves at least one live row and neither `-y`
  nor `-j` was given; the prompt states plainly that in-flight progress is discarded.

Exit contract, mirroring restart: `0` drained or previewed, `2` refused with nothing
changed (unknown provider, not disabled, soft disable, declined confirmation, nothing to
drain), `1` when at least one move failed after its agent was stopped.

Update `docs/cli.md` with the new row.

Tests (`tests/test_agent_drain_cli.py`, `tests/ops/`-style coverage for the operation)
cover: the refusal exits, dry-run, declined confirmation, the JSON envelope shape, the
`--model` path, `--limit` reporting, and the durable-operation result envelope.

### Automatic drain on a usage-limit disable

Create the flag first, at the start of this phase — it is the one bead this epic may
create, and `sase bead create` / `/sase_new_task` must not be used for it:

```bash
sase flag new provider_drain -k beta \
  --when-enabled "Hard-disabling an LLM provider drains it: a usage-limit disable submits a durable 'sase agent drain' proc that relaunches the agents that provider stranded and sends one enriched usage-limit notification naming what moved and what did not, and a manual disable in Launch Control offers the same relaunch." \
  --when-disabled "No drain is submitted from either path; a usage-limit disable sends today's notification unchanged and Launch Control shows no relaunch prompt. The 'sase agent drain' CLI stays available for a manual drain." \
  --remove-when "Automatic drains have run for a full minor release without a wrong relaunch, the stranded and skip reporting has needed no reinterpretation, and no path still depends on the un-enriched notification text."
```

Config, under `llm_provider.usage_limit` in
`src/sase/llm_provider/usage_limit_config*.py` and `UsageLimitSettings`:

- `relaunch` (bool, default `true`) — whether an automatic disable submits a drain.
- `relaunch_limit` (int, default `20`) — `--limit` for that drain.

Both are consulted only when the flag is enabled. Mirror them into
`src/sase/default_config.yml`'s commented block, `src/sase/config/sase.schema.json`,
`docs/configuration.md`, and the `llm_provider.usage_limit` bullets in `docs/llms.md`.

In `sase/llm_provider/usage_limit_disable.py`, after `outcome.inserted` is true and the
existing logging and metric, replace the unconditional
`_notify_usage_limit_disabled(...)` call with an ownership decision:

1. If the flag is off, or `relaunch` is false, or the written record is not hard, notify
   inline exactly as today.
2. Otherwise submit the drain through `sase.procs.submit_proc_request()`:
   `argv = ["sase", "agent", "drain", provider, "--yes", "--json", "--limit", str(limit)]`
   (plus `--model` never; the automatic path never guesses a model), `operation` =
   `AGENT_DRAIN`, `operation_payload` carrying the trigger context the notification
   needs (`matched_pattern`, `raw_message`, `disable_seconds`, `expires_at`,
   `used_reset_hint`, `trigger_agent`, `trigger_model`, `notify: true`), `label` =
   `Drain <PROVIDER> (usage limit)`, `cwd` = the home directory, `origin="usage_limit"`,
   `tags=["llm", "usage-limit", provider]`,
   `concurrency_keys=[f"provider-drain:{provider}"]`, and a generous `timeout_seconds`.
   On success this process notifies nothing.
3. If submission raises, log at warning and fall back to the inline notification, so a
   proc-layer failure can never leave the user silent. The whole block stays inside the
   module's existing never-raise contract.

In the drain command, when the operation payload carries `notify: true`, the drain owns
the notification and sends it from a `finally` block so a mid-drain failure still
reports. Before planning, settle the trigger agent: resolve it with
`sase.agent.wait_watch.resolve_wait_targets()` and wait through `watch_wait_targets()`
with a bounded timeout (60 s) so the agent that just failed is restarted as a completed
row rather than killed during its own teardown. A timeout is not an error; the drain
proceeds.

Extend `notify_provider_usage_limit_disabled()` to accept an optional drain report and
append notes after the existing trigger line, keeping every note one scannable line:

```
⛔ CLAUDE disabled for 4h 12m — 5-hour limit reached
Re-enables at 8:00 PM EDT (Sun Aug 24), as reported by the provider.
Triggered by agent sase-mf on opus@high.
Relaunched 4 agents on CODEX and GEMINI: sase-mf, ace-01, bd-7, kt.2
Left alone: 1 pinned to claude/opus (nine), 1 waiting on a question (rq.3)
Provider said: "..."
Launches now route to the next enabled provider in each alias.
```

Name at most a handful of agents per line and summarize the remainder as `+N more`;
group skips by reason. When nothing was drained, say so in one line rather than omitting
the report. Keep `action="OpenLaunchControl"` and the existing tags.

Tests: flag on and flag off for both the submission decision and the notification text;
`relaunch: false`; a soft-disable write not submitting; the submission-failure fallback;
exactly one notification per window in every path; the settle wait timing out without
failing the drain.

### Launch Control relaunch prompt after a manual disable

In `src/sase/ace/tui/modals/models_panel_provider_modal.py`, the disable write already
runs in the `provider-routing-write` worker. Extend that worker's `ProviderWriteOutcome`
with a drain preview computed off the event loop — count and route summary only, via
`plan_provider_drain()` — and only when the write was a **hard** disable that changed
state and the flag is on. `_on_write_worker` stays thin and synchronous per
`sase/memory/tui_perf.md`.

When the preview has at least one candidate, push a new single-keypress panel
`src/sase/ace/tui/modals/provider_drain_prompt_modal.py`, modeled directly on
`disabled_provider_launch_modal.py` (same row dataclass, same dim informational rows,
same escape-is-the-safe-choice posture):

```
CLAUDE disabled for 3h · 5 agents depend on it

 r  Relaunch 4 agents now              3 → CODEX · 1 → GEMINI · in-flight work is lost
 m  Relaunch them on a model I pick…
 l  Leave them alone                   1 pinned to claude/opus cannot move either way
```

`r` submits the drain, `m` opens the existing model picker
(`sase/ace/tui/modals/model_picker_modal.py`) and then submits with `--model`, and `l`,
`escape`, and `q` all leave everything alone. When the preview is empty the panel never
appears — a disable with no dependent agents stays as quiet as it is today.

Submission goes through `sase.ace.tui.durable_submit.submit_durable_proc_request()` with
`operation=AGENT_DRAIN`, `concurrency_keys=("provider-drain:<provider>",)`, and a label
matching the CLI's, so the drain appears in the Procs tab, dedups against an automatic
drain already in flight, and returns a typed result. Register the site in the ACE
proc-producer inventory (`_proc_producer_sites_*`) with that concurrency key. Toast the
outcome from the result envelope (`Relaunched 4 agents; 1 left alone`), and toast the
failure path with the proc id to inspect.

Tests: the modal's rows and key map, the empty-preview silent path, soft disable not
prompting, flag off not prompting, the submitted argv and concurrency key, and the
result toast. Add a PNG snapshot for the panel alongside the existing
`tests/ace/tui/visual/test_ace_png_snapshots_disabled_provider_launch.py` goldens.

### End-to-end drill and reference documentation

Add `tests/fakey/test_provider_drain_e2e.py` on top of `FakeyRetryHarness` and
`usage_limit_failure()`: run a real fakey agent into a usage-limit failure with the flag
enabled and a second fakey agent alive, and assert through the production path that the
disable is written once, the drain proc is submitted once with the expected argv and
concurrency key, the second agent is stopped and relaunched, and exactly one
notification carries the drain report. Assert the flag-off variant leaves both agents
alone with today's notification.

Document the capability in `docs/llms.md` as a `### Draining a Disabled Provider`
subsection under Usage-Limit Auto-Disable: what a drain selects, the reroute versus
stranded distinction, what is never drained and why, the progress-loss cost, the
`relaunch` / `relaunch_limit` settings, the beta flag, and the manual `sase agent drain`
escape hatch. Cross-link it from Temporary Provider Disables and from the ACE
Models-panel section of `docs/ace.md`.

## 5. Verification

Every phase runs `just install` first, then `just check`. The final phase and the
combined tree run `just check-full` through `/sase_monitor`, never inline.

## 6. Acceptance

- Hard-disabling a provider with the flag on relaunches the agents that provider
  stranded, from both the usage-limit path and Launch Control.
- Exactly one usage-limit notification is sent per disable window in every path, and it
  names what was relaunched, where it went, and what was left alone with the reason.
- A pinned agent is reported as stranded and never relaunched onto a hard-disabled
  provider; `sase agent drain -m` is the documented way to move it.
- Monitor rows, rows holding a pending question, and the calling agent are never
  drained.
- `sase agent drain` refuses a provider that is not hard-disabled, previews with
  `--dry-run`, confirms before discarding live progress, and reports anything a
  `--limit` dropped.
- With the flag off, both disable paths behave exactly as they do today.
- No new durable state files are introduced.
