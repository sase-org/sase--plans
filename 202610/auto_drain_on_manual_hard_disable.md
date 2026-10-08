---
tier: tale
title: Auto-drain a provider after a manual hard disable
goal:
  A manual hard disable in Provider Routing starts the provider drain automatically in a
  durable background proc, with a start toast and a completion toast, and never blocks
  the TUI.
size: medium
decisions:
  transition_only:
    ask:
      Drain only when a provider newly becomes hard-disabled, never on hard-disable
      window changes?
    default: true
    why:
      An automatic drain discards in-flight work, so it should follow only the stranding
      event.
    answer: true
  noop_success:
    ask: Record an automatic drain that finds nothing to do as a successful no-op proc?
    default: true
    why:
      Avoids a red failed drain proc in Procs every time an idle provider is
      hard-disabled.
    answer: true
proposed_by: bbugyi200.athena.0y5
decided_by: auto
create_time: 2026-10-08 07:49:46
status: wip
---

# Auto-drain a provider after a manual hard disable in Provider Routing

## Problem

In sase's TUI, Admin Center → Launch tab → `p` (Provider Routing) → `d`/`enter` on a
provider hard-disables it. With the `provider_drain` beta flag on, the user is then
asked whether to drain the agents that provider stranded (`ProviderDrainPromptModal`:
`r` relaunch / `m` pick model / `l` leave alone). That prompt can take a long time to
appear.

Root cause: `ProviderRoutingWorkersMixin._submit_disable()`
(`src/sase/ace/tui/modals/models_panel_provider_modal_workers.py`) runs
`_provider_drain_preview()` → `plan_provider_drain()` inside the same
`provider-routing-write` thread worker as the disable write. That preview does a full
`list_all_agents()` scan (~0.4s on athena with ~110 rows) and then a
`plan_agent_restart()` plus `plan_launch_units()` replan for every candidate. Only after
all of that returns does `_on_write_worker()` show the "disabled" toast and push the
prompt. While it runs:

- the user sees no confirmation that the disable even landed;
- `action_back` refuses to close the modal ("A provider-routing update is still in
  progress.");
- the planning is CPU-bound Python on a TUI thread worker, so it competes with the
  Textual event loop for the GIL and makes the whole TUI sluggish.

The planning is wasted work too: `sase agent drain` replans from scratch when the drain
runs.

## Goal

When a manual hard disable newly lands on a provider (and `provider_drain` is on), sase
starts the drain **automatically**, with no prompt:

1. The disable write returns as fast as it does for a soft disable (no in-thread drain
   planning), and the usual "disabled" toast shows right away.
2. sase submits the existing durable `agent.drain` proc
   (`sase agent drain <provider> --yes --json`) right away and shows a clear **start
   toast**.
3. All planning and relaunching happen in that separate durable proc, so nothing touches
   the event loop or the TUI's GIL. The user can close the modal and keep working.
4. When the proc settles, a **completion toast** summarizes what happened, using the
   durable result envelope.

## Design decisions

- **Reuse the durable drain proc. Do not add a TUI-side worker.**
  `submit_provider_drain()` (`src/sase/ace/tui/actions/agent_durable.py`) already
  submits through `app._submit_durable_proc()`. That call registers a pending row in
  memory and submits on a thread worker. Completion arrives later through proc
  observation as a UI-thread `on_complete(TrackedProcCompletion)` callback that carries
  the decoded JSON envelope in `completion.payload`. That gives us the Procs-tab row,
  the proc indicator, dedup with an automatic usage-limit drain through the shared
  `provider-drain:<provider>` concurrency key, a proc that outlives the TUI, and no
  event-loop work. The `on_complete` closure must capture only `app` and the provider
  string, never the modal, because the user may have closed the modal by then.
- **When to drain** (decision `transition_only`). Today the prompt fires on any
  `changed` hard write, which includes merely changing the window of an already-hard
  disable.

> [!decision] transition_only Drain only on a transition into hard-disabled. An
> automatic action that discards in-flight work should follow only the event that
> actually strands agents. So drain only when the write is a hard disable, changed
> routing, and `previous_mode != PROVIDER_DISABLE_MODE_HARD`. That covers: no disable →
> hard, soft → hard, and an expired disable → hard. Re-hard-disabling or extending an
> existing hard disable never re-drains. `sase agent drain` is still the manual escape
> hatch for that case.

> [!decision] transition_only = no Keep today's prompt trigger: drain after every
> changed hard-disable write, including a window change on a provider that was already
> hard-disabled. A re-drain usually finds nothing and ends with the "nothing to
> relaunch" toast.

- **Delete the prompt. Do not keep it behind a config option.** The user asked for
  automatic. `ProviderDrainPromptModal`, its `m` model-picker path, and the `model=`
  parameter of `submit_provider_drain()` lose their only consumers, so they are deleted
  (Symvision would flag them anyway). Draining onto a chosen model is still available as
  `sase agent drain <provider> -m <model>`.
- **Empty automatic drains** (decision `noop_success`). `sase agent drain --yes --json`
  exits 2 (`success=False`) for `nothing_to_drain`. It does the same for `not_disabled`
  and `soft_disabled` (the user re-enabled or softened the provider before the proc
  planned). The completion toast reads `error.reason` from the payload either way, so
  the toast stays friendly whatever this decision says.

> [!decision] noop_success Make an automatic drain that finds nothing to do a successful
> no-op, not a failed proc. Without this, every hard disable of an idle provider leaves
> a red failed proc in Procs. The ACE submission adds `"automatic": true` to its request
> payload. Inside `sase.ops.commands.agent._run_drain()` only, an automatic request
> whose result payload `error.reason` is one of those three is returned as
> `success=True, exit_code=0`. The envelope payload is unchanged. The `sase agent drain`
> CLI contract (exit 2) and the usage-limit path (no `automatic` key) do not change.

> [!decision] noop_success = no Leave the ops layer alone. An empty automatic drain
> stays a failed (exit 2) proc in Procs, and only the toast reports it as benign. Skip
> the `"automatic"` payload key, the `_run_drain()` change, and its tests (section 3's
> ops-layer bullet and section 6's `test_ops_commands.py` bullet).

- **Keep it flag-gated.** `provider_drain` is a beta flag whose Off branch must stay
  explicit. Flag off means no drain is submitted and no drain toast is shown. That is
  the same as today's flag-off behavior, which shows no prompt. Evaluate the flag on the
  write-worker thread, as today.
- **The completion toast is ACE-local presentation over the stable envelope.**
  `sase.agents._drain_render.usage_limit_drain_report_notes()` produces verbose,
  name-listing notification lines. Importing `_drain_render` also pulls in
  `sase.agent.provider_drain`, which costs ~0.5s on a cold import, and the import would
  run on the UI thread at completion time. So keep a small, pure, dict-only toast
  formatter in the TUI drain mixin. It extends today's `_drain_completion_message()`.
  This change also removes the heavy `sase.agent.provider_drain` import from the
  provider-routing modal chain, since the workers mixin and the deleted prompt modal
  were its only TUI importers there.

## Implementation

### 1. Write worker: stop planning, just flag the drain

`src/sase/ace/tui/modals/models_panel_provider_state.py`

- In `ProviderWriteOutcome`, replace `drain_preview` and `drain_preview_error` with
  `drain_requested: bool = False`. Drop the `TYPE_CHECKING` import of
  `ProviderDrainPlan`.

`src/sase/ace/tui/modals/models_panel_provider_modal_workers.py`

- Delete `_provider_drain_preview()`, `_PROVIDER_DRAIN_PREVIEW_LIMIT`, and the
  `plan_provider_drain` / `ProviderDrainError` imports.
- Add a cheap, pure helper, e.g.
  `_provider_drain_requested(provider, *, mode, changed, previous_mode, snapshot, captured_now) -> bool`.
  It returns True only when all of these hold: `changed`,
  `mode == PROVIDER_DISABLE_MODE_HARD`, `_provider_drain_flag_enabled()`, and the
  reloaded snapshot's `active_disable(...)` for the provider is hard. Under
  `transition_only` (above) it also requires
  `previous_mode != PROVIDER_DISABLE_MODE_HARD`. It must do no agent scans and no
  planning.
- In `_submit_disable()`'s `task()`, call that helper on the worker thread and set
  `drain_requested=` on the outcome. `previous_mode` is already captured before the task
  starts.
- In `_on_write_worker()`, on a changed `disable` outcome, keep the existing
  `_disable_success_toast` and `_changed = True`. Then call
  `self._start_provider_drain(outcome.provider)` only when `outcome.drain_requested`.
  This replaces `_maybe_prompt_provider_drain(outcome)`. Update the `TYPE_CHECKING` stub
  accordingly.

### 2. Drain mixin: submit automatically, toast start and finish

`src/sase/ace/tui/modals/models_panel_provider_modal_drain.py`. Rewrite the module
docstring to "Automatic provider drain after a manual hard disable".

- Delete `_maybe_prompt_provider_drain`, `_on_provider_drain_decision`, and
  `_on_provider_drain_model`. Also delete the `ModelPickerModal`,
  `ProviderDrainPromptModal`/`Decision`, and `ProviderDrainPlan`-related imports.
- Add `_start_provider_drain(self, provider: str) -> None`:
  - `app = self.app`. Call
    `submit_provider_drain(app, provider=provider, on_complete=...)` inside
    `try/except Exception`. On an exception, show an error toast:
    `Could not start the {P} drain: {exc}`. `P` is `provider.upper()`.
  - When it returns True, show the **start toast** with `app.notify(...)`, using a title
    and a slightly longer timeout (about 10s) because the text is longer than a
    one-liner. Suggested copy:
    - title: `Draining {P}`
    - message:
      `Relaunching agents stranded on {P} onto enabled providers in the background. In-flight work on those agents restarts; chat transcripts are kept. Track it in Procs — another toast follows when it finishes.`
  - When it returns False, `_submit_durable_proc` has already shown its duplicate
    warning. Show no start toast.
- Completion callback (runs on the UI thread; pure dict formatting only). Use
  `app.notify` with a title in every case:
  - `completion.collision`: warning,
    `A {P} drain is already running; check Procs for its result.`
  - Payload missing or not a mapping, and not success: error, title `{P} drain failed`,
    `Proc {completion.proc_info.proc_id} failed; inspect Procs: {completion.message}`.
  - Payload `error.reason` in `{not_disabled, soft_disabled}`: information,
    `{P} is no longer hard-disabled; nothing was drained.`
  - `counts.failed > 0`: error, title `{P} drain finished with failures`. Body: the
    relaunched summary, the failed count, and the left-alone summary, plus
    `Inspect proc {proc_id} in Procs.`
  - `counts.relaunched > 0`: information, title `{P} drained`. Body example:
    `Relaunched 3 agents on CODEX and GEMINI; 1 left alone (waiting on a question).`
  - No relaunches with skips: warning, title `{P} drain: nothing could move`. Body
    example: `2 agents left alone (1 waiting on a question, 1 pinned to claude/opus).`
  - Nothing moved and nothing skipped: information, title `{P} drain done`,
    `No agents depended on {P}; nothing to relaunch.`
- Formatter helpers stay private and pure, and read only the stable envelope (`counts`,
  `moves[].route.target_provider`, `results[].name/status`, `skips[].reason/detail`,
  `error.reason`). Target providers are the upper-cased distinct `target_provider`
  values of moves whose result status is `ok`, joined with "and" or commas. Group skip
  reasons the same way the deleted prompt's `_skip_summary` did: `stranded` →
  `pinned to <target>` taken from the detail prefix, `pending_question` →
  `waiting on a question`, `monitor` → `monitor row`, `caller` → `current agent`,
  `capped` → `over the drain limit`, and anything else → the reason with `_` replaced by
  spaces. Keep or replace `_drain_completion_message` and `_payload_count`. Delete
  anything that ends up unused.

### 3. Durable submission and the ops-layer no-op

`src/sase/ace/tui/actions/agent_durable.py`

- `submit_provider_drain()`: drop the now-unused `model` parameter and its `--model`
  argv and payload branch. Under `noop_success`, add `"automatic": True` to the request
  payload, next to `"origin": "ace_manual_disable"`. Keep the argv
  (`agent drain <provider> --yes --json`), operation, concurrency key, proc type, label,
  `cwd=Path.home()`, `reload_on_complete=False`, and `notify_on_complete=False`
  unchanged.
- Check that `src/sase/ace/tui/_proc_producer_sites_actions.py`'s
  `submit_provider_drain` producer-site entry still matches. It should need no change.

`src/sase/ops/commands/agent.py` (only under `noop_success`)

- In `_run_drain()`, after `run_agents_drain(...)`, add this: if
  `request.payload.get("automatic") is True` and the result payload's `error.reason` is
  in `{"nothing_to_drain", "not_disabled", "soft_disabled"}`, return
  `OperationCommandResult(success=True, exit_code=0, payload=result.payload, message=...)`.
  Use a short "nothing to drain" message, for example the error's own message.
  Everything else passes through unchanged. Put the reason set in a module-level
  frozenset constant with a one-line comment explaining why automatic drains treat these
  as no-ops.

### 4. Delete the prompt modal

- Delete `src/sase/ace/tui/modals/provider_drain_prompt_modal.py`.
- Remove its two entries from `src/sase/ace/tui/modals/_export_table.py` and
  `src/sase/ace/tui/modals/__init__.pyi`.
- In `src/sase/ace/tui/styles.tcss`, remove `ProviderDrainPromptModal,` from the shared
  duration-choice selector list and remove the `#provider-drain-prompt-container` and
  `#provider-drain-prompt-title` rules.
- Delete `tests/ace/tui/test_provider_drain_prompt_panel.py`,
  `tests/ace/tui/visual/test_ace_png_snapshots_provider_drain_prompt.py`, and the golden
  `tests/ace/tui/visual/snapshots/png/provider_drain_prompt_panel_120x40.png`.
- Grep the repo for `ProviderDrainPrompt`, `provider_drain_prompt`,
  `_maybe_prompt_provider_drain`, and `drain_preview`. No references may remain outside
  `site/`, `node_modules/`, and `__pycache__/`.

### 5. Flag text, docs, and the flag bead

- `src/sase/feature_flags/registry.py`: in the `provider_drain` description, replace
  "and a manual disable in Launch Control offers the same relaunch" with wording like
  "and a manual hard disable in Launch Control submits the same drain automatically,
  toasting when it starts and when it finishes". Then run
  `just sync-feature-flags-schema` to regenerate the flag block in
  `src/sase/config/sase.schema.json`. Do not hand-edit the schema.
- `docs/ace.md`: rewrite
  `### Provider-drain relaunch prompt {#provider-drain-relaunch-prompt}` as
  `### Automatic provider drain {#automatic-provider-drain}`. Document the trigger
  conditions (manual hard disable; flag on; transition into hard only under
  `transition_only`). Document that the drain runs as the durable `agent.drain` proc
  with the shared `provider-drain:<provider>` concurrency key, so it dedups against an
  automatic usage-limit drain. Document the start and completion toasts and the Procs
  row. Under `noop_success`, document that an empty or no-longer-disabled automatic
  drain records a successful no-op. Document that
  `sase agent drain <provider> -m <model>` replaces the old `m` choice. Document that if
  the TUI exits first, the proc still finishes and its result stays in Procs /
  `sase proc show`.
- `docs/llms.md` (around the "provider-drain relaunch prompt" link near line 2758):
  update the link text and anchor to `ace.md#automatic-provider-drain` and describe the
  manual path as automatic. Under `noop_success`, wherever the text says the drain exits
  2 for `nothing_to_drain`, note the automatic-request no-op.
- `docs/configuration.md`: update the `provider_drain` row of the flag table so it
  covers the manual Launch Control hard disable as well as `llm_provider.usage_limit`.
- Flag bead `sase-sx`: append a note so the bead's On-branch record matches the code. Do
  not rewrite the bead description. For example:
  `sase bead update sase-sx -n "On branch changed: a manual hard disable in Launch Control now submits the drain automatically (start and completion toasts) instead of prompting; the prompt modal was deleted."`

### 6. Tests

Rewrite `tests/test_models_panel_provider_modal_drain.py`. Keep its `ModelsPanelTestApp`
/ `wait_for` style and monkeypatch `submit_provider_drain` on the drain mixin module.

- Hard disable from no disable, flag on: `submit_provider_drain` is called exactly once
  with the provider. No `ProviderDrainPromptModal` or any other screen is pushed. The
  start toast is shown with a `Draining CLAUDE` title.
- Assert that `plan_provider_drain` is never called during the write. Monkeypatch it in
  `sase.agent.provider_drain` to raise, or assert it is absent from the workers module.
- Soft → hard transition drains. Hard → hard (window change, `changed=True`) does not
  under `transition_only`; it does drain if that decision is answered no.
- Soft disable does not drain. Flag off does not drain. An unchanged write does not
  drain. These cover both flag states.
- Submission returning False shows no start toast. Submission raising shows the error
  toast.
- Completion callback, driven with fake `TrackedProcCompletion`-like objects: relaunched
  with providers and skips; failures; skips only; nothing at all; `not_disabled`;
  collision; missing payload with failure. Assert severity, title, and key phrases.
- Formatter unit tests for skip-reason grouping and the provider join.

`tests/ace/tui/test_durable_submit.py`: update
`test_submit_provider_drain_uses_cli_and_provider_concurrency_key`. Assert there is no
`--model`, and assert the payload is
`{"provider": "claude", "origin": "ace_manual_disable", "automatic": True}` (without
`"automatic"` if `noop_success` is answered no).

`tests/main/test_ops_commands.py` (only under `noop_success`): add tests next to the
existing `agent.drain` operation tests:

- An automatic request whose fake drain result has `error.reason == "nothing_to_drain"`
  writes a successful result and exits 0. Also cover `not_disabled`.
- A non-automatic request with the same result keeps `success=False` and exit 2.
- An automatic request with `move_failed` still fails.

### 7. Verify

- `sase tool run check` (the agent-default gate). It covers ruff, mypy, Symvision, flag
  lint (including schema drift), and the scoped tests.
- If any `tests/ace/tui/visual` selection fails only because the deleted golden is
  referenced, fix the reference. Do not run `just check-full` or
  `just fix-tui-screenshots`. No new PNG golden is needed because this change adds no
  new screen.
- Manual smoke, optional, only if a TUI session is available: with `provider_drain`
  enabled, hard-disable an idle test provider from Provider Routing. Confirm the
  "disabled" and "Draining …" toasts appear immediately, the modal closes with `esc`
  right away, a `Drain …` row shows up in Procs, and the "nothing to relaunch" toast
  follows. Then re-enable the provider.

## Out of scope

- Changing the usage-limit automatic drain path or its notification.
- Changing `sase agent drain` CLI exit codes or the JSON envelope schema.
- A config option to restore the prompt.
