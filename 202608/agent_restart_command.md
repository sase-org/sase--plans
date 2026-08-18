---
tier: tale
title: Add a sase agent restart command
goal:
  "`sase agent restart <NAME>` stops the named agent and immediately relaunches its
  stored prompt under the same name, with a preview/confirm/receipt output that makes
  the whole operation legible before and after it happens."
size: medium
proposed_by: bbugyi200.athena.05t
create_time: 2026-08-18 07:34:10
status: wip
---

# Plan: `sase agent restart <NAME>`

## 1. What This Is

`sase agent restart NAME` is the CLI twin of the ACE Agents-tab `,x` (leader
`kill_and_edit`) flow, minus the interactive edit step. `,x` reads the selected agent's
stored prompt, rewrites it so the agent's name is force-reused, kills or dismisses the
row, and seeds the prompt bar so the user can submit it again. The CLI command performs
the identical stop-then-relaunch cycle without the edit pause: same prompt, same name,
new run.

The ACE behavior this mirrors lives in
`src/sase/ace/tui/actions/agent_workflow/_entry_relaunch.py` (`_kill_and_edit_agent` →
`_finish_kill_and_edit_agent` → `_edit_and_relaunch_agent`) and
`src/sase/ace/tui/actions/agent_workflow/_entry_name_prompts.py`
(`prepare_kill_and_edit_prompt`). Read both before writing code.

Design intent, in priority order:

1. **Reliable.** Every reason the restart could be refused is discovered _before_
   anything is killed. The command never leaves two live agents holding one name, and
   never leaves a stopped agent with no relaunch and no recovery path.
2. **Intuitive.** One positional argument does the obvious thing. Optional flags follow
   the conventions already set by `sase agent-cli install` (`-j/--json`, `-n/--dry-run`,
   `-y/--yes`).
3. **Beautiful.** A preview panel, a live step ledger, and a receipt panel — rendered in
   the same Rich idiom as `monitor_detail()` in `src/sase/main/monitor_render.py` and
   `sase agent show`.

## 2. Command Surface

```
sase agent restart [-h] [-j] [-m MODEL] [-n] [-y] NAME
```

`NAME` is positional because `sase/memory/cli_rules.md` requires that a value the
command cannot run without be a positional, not an option. (`sase agent kill -n NAME`
predates that rule; do not copy it.) Flags are declared in alphabetical order and each
long option has a short alias:

| Flag            | Behavior                                                                                                    |
| --------------- | ----------------------------------------------------------------------------------------------------------- |
| `-j, --json`    | Emit one stable JSON envelope on stdout and nothing else.                                                   |
| `-m, --model M` | Relaunch under a different model. Accepts `[provider/]model[@effort]`, the same spelling `%model:` accepts. |
| `-n, --dry-run` | Print the preview and exit 0. Nothing is killed, dismissed, wiped, or launched.                             |
| `-y, --yes`     | Skip the interactive confirmation for a live agent.                                                         |

Register the subparser in `src/sase/main/parser_agent.py` between the `prompts` and
`retire-v1` blocks so the source order stays alphabetical, with a `description` and a
`RawDescriptionHelpFormatter` `epilog` holding examples:

```
examples:
  sase agent restart 02p
  sase agent restart 02p --dry-run
  sase agent restart sase-mf.1 -m opus@high
  sase agent restart 02p -j
```

Dispatch from `src/sase/main/agent_handler.py` (`sub == "restart"`) and extend the
fallback usage string that command prints. Add
`(("agent", "restart"), "name"): ValueKind.AGENT` to `_build_path_overrides()` in
`src/sase/completion/kinds.py` so shell completion offers agent names, exactly as it
already does for `agent kill`, `agent revert`, and `agent show`.

## 3. Pipeline

New module `src/sase/agent/restart.py`, split into a pure planning half and a mutating
execution half. This mirrors `plan_force_reuse_launch()` / `apply_force_reuse_launch()`
in `src/sase/agent/force_reuse_launch.py`, whose docstring states the same rationale:
all parsing and validation must finish before anything destructive runs.

### 3.1 `plan_agent_restart(name, *, model_override=None) -> AgentRestartPlan`

Reads only. Raises `AgentRestartError(reason=..., message=..., hint=...)` on refusal.

1. `find_named_agent(name)` (`sase.agent.names`). `None` → `reason="not_found"`. This is
   the same resolver `sase agent kill` and `sase agent show` use, so owner-prefixed and
   workflow-name spellings resolve identically.
2. Read `agent_meta.json` and `done.json` from `agent.artifacts_dir`, and derive the
   project via `parse_agent_artifact_path()` (`sase.core.agent_artifact_paths`), falling
   back to the path walk `kill_named_agent()` uses when that returns `None`.
3. Read `raw_xprompt.md`. Missing or blank → `reason="no_prompt"`. This file is written
   at launch by `write` in `src/sase/axe/run_agent_runner_setup.py` and is the
   authoritative relaunch source — the same file `get_restartable_prompt_content()`
   prefers. Do **not** port that function's `*_prompt.md` reconstruction fallback: it
   needs a live TUI `Agent` row. The refusal message must say so and point at ACE's `,x`
   as the surface that can still reconstruct such a prompt.
4. `parse_multi_prompt(raw)` (`sase.agent.multi_prompt`). More than one segment →
   `reason="multi_segment"`. A per-agent `raw_xprompt.md` is a single segment in
   practice; this is a cheap guard against relaunching a fan-out under one name.
5. Build the relaunch prompt through the same boundary ACE uses:
   `prepare_kill_and_edit_prompt(raw, meta["name"], family_name=..., role_suffix=..., phase_bead_id=...)`.
   Take `role_suffix`, `agent_family`, `agent_family_parallel`, and `phase_bead_id` from
   `agent_meta.json` (all four are stored fields — see
   `src/sase/integrations/_agent_list_entry_builder.py` and
   `src/sase/agent/running_listing.py`, which already read them). Apply the same
   condition `prepare_kill_edit_agent_prompt()` applies: pass
   `family_name`/`role_suffix` only when the agent has a family, is **not**
   `agent_family_parallel`, and has a role suffix — otherwise a family member would
   collapse to its base name.
6. When `--model` is given, apply the override (§5).
7. `plan_force_reuse_launch(prompt)`. `None` means no `!` force-reuse marker survived
   the rewrite → `reason="name_not_reusable"`. Any exception it raises (name collision,
   clan/family collision, `_ForceReuseLaunchFanoutError`) → `reason="preflight"`
   carrying the original message verbatim; those messages are already actionable.
8. Collect the display facts the renderer needs (§6) and return the plan. The plan owns
   the `NamedAgent`, the artifacts dir, the project name, the meta/done dicts, the
   original and rewritten prompts, and the validated `_ForceReuseLaunchPlan`.

### 3.2 `execute_agent_restart(plan) -> AgentRestartOutcome`

Mutates, in exactly this order:

1. **Stop.** Live agent (`not agent.is_done` and a resolvable PID) →
   `kill_named_agent(plan.name, exact_name=True)` from `src/sase/agent/running.py`. If
   it returns `success=False`, abort here: nothing has been wiped or launched, so the
   restart is a clean no-op and exits 2. Aborting on a failed kill is what keeps a
   restart from ever producing two live agents under one name.
2. **Dismiss** instead, when the agent is already done or already stopped: call the new
   `dismiss_named_agent()` (§4) so the old row leaves the inbox the way `,x` leaves it.
3. **Wipe the name.** `apply_force_reuse_launch(plan.force_reuse_plan)`.
4. **Launch.**
   `launch_agents_from_cwd(plan.rewritten_prompt, segment_extra_env=plan.force_reuse_plan.segment_envs)`,
   run inside `contextlib.chdir(Path.home())`.

   The `chdir` is deliberate and must be kept. `launch_agents_from_cwd_impl()` resolves
   project context from the process CWD and falls back to home mode when the CWD is not
   a project. ACE's relaunch always builds a **home-mode** `PromptContext`
   (`_mount_edit_relaunch_prompt_bar()` sets `project_name="home"`), so the stored
   prompt's own leading VCS tag is what targets the project. Pinning CWD to home
   reproduces that exactly, and stops `sase agent restart` from silently retargeting an
   untagged prompt at whatever workspace the operator happens to be standing in.

5. **Partial failure.** If the launch raises after step 1/2 succeeded, return
   `AgentRestartOutcome(status="partial", ...)` carrying the cause and a recovery
   command. The old agent's artifacts (including `raw_xprompt.md`) still exist after
   dismissal, and its name is now free, so the recovery command is literally:

   ```
   sase run "$(cat <artifacts_dir>/raw_xprompt.md)"
   ```

   Do not re-record the prompt in history here: `launch_agents_from_cwd_impl()` already
   stashes failed launch prompts through `record_failed_launch_prompt()`, and successful
   launches are recorded by `add_or_update_prompt()`.

## 4. `dismiss_named_agent()`

Add to `src/sase/agent/running.py`, returning the same `KillResult` shape as
`kill_named_agent()`. It exists because `kill_named_agent()` deliberately refuses a
completed agent with `reason="already_completed"`, but a restart of a DONE agent must
still retire the old row.

It reuses the private helpers already in that module: `_release_workspace_claim()` for a
non-home agent, `_remove_agent_state_markers(remove_running=True, remove_waiting=True)`,
`_record_dismissal()`, and `_dismiss_agent_notifications()`. Follow
`kill_named_agent()`'s existing contract that index and notification bookkeeping
failures are best-effort and never flip a successful dismissal into a failure.

## 5. `--model` Override

Add `set_prompt_model(prompt, model)` in a new
`src/sase/xprompt/_directive_edit_model.py` and export it from
`src/sase/xprompt/directive_edit.py`, matching the existing `_directive_edit_identity` /
`_directive_edit_wait` split. Build it on `set_prompt_directive()` from
`_directive_edit_core.py`, formatting the replacement the way
`_restart_model_directive()` in `src/sase/ace/tui/models/artifact_files.py` already
does: `%model:<value>` normally, `%model("<value>")` when the value contains whitespace
or `,()=`.

Effort handling, which must be documented in the flag's help text:

- `-m opus` replaces `{"m", "model"}` only. Any standalone `%effort:`/`%e:` directive in
  the stored prompt is left alone, because the user asked to change only the model.
- `-m opus@high` replaces `{"m", "model"}` **and** removes `{"e", "effort"}`, so the
  single `%model:opus@high` directive is the only source of truth and cannot conflict
  with a stale `%effort:`.

Split the value with `split_model_effort()` from `sase.xprompt.effort` to decide which
case applies.

## 6. Output

Errors go to stderr; everything else to stdout. Under `-j` the JSON envelope is the only
thing written to stdout.

### 6.1 Preview panel

Rendered before acting, and the entire output of `--dry-run`. A rounded `Panel` titled
`Restart · <presented name>` with `title_align="left"`, wrapping a
`Table(box=None, show_header=False, pad_edge=False, expand=True)` field/value grid — the
shape `monitor_detail()` uses. Rows, omitting any that do not apply:

`Status`, `Project`, `Patch`, `Workspace`, `PID`, `Model`, `Started`, `Elapsed`,
`Family`, `Bead`, `Prompt`, `Target`, `Name reuse`, and `Model override` (only with
`-m`, rendered as `old → new`).

Rules the rows must honor:

- `Status` uses the status colors and glyph `sase agent list` already uses
  (`_STATUS_COLORS` in `src/sase/agents/cli_list.py`). Promote that mapping to a shared
  helper rather than copying the dict.
- `Project` renders `project_display_name_for(...)`, never the ProjectSpec key — this is
  the "Show Project Names, Never ProjectSpec Keys" rule in `CLAUDE.md`.
- `Model` uses `append_model_field()` from `sase.llm_provider.model_label`, the same
  helper `sase agent show` uses, so the model/provider/effort line reads identically
  across both commands.
- `Prompt` shows a humanized excerpt via `humanize_vcs_refs_in_text()` (about 160 chars,
  ellipsized).
- `Target` shows the humanized leading VCS tag resolved with
  `extract_vcs_workflow_tag()` / `find_vcs_workflow_tag()` from `sase.xprompt`, or
  `home (prompt has no VCS tag)`.
- `Name reuse` shows `forced (%id(!<name>))`.

Warning lines print below the panel, in yellow, when they apply:

- live target: `Restarting a running agent discards its in-flight work.`
- `has_file_changes` (derived like `_agent_list_entry_builder.py` does, from
  `done["diff_path"]` or `meta["commit_diff_path"]`):
  `This run produced file changes; the restart starts from the current workspace state.`
- project agent whose prompt carries no VCS tag:
  `The stored prompt has no VCS tag; the restart will run as a home agent.`

### 6.2 Step ledger

Streamed as execution proceeds (not in dry-run), two columns, `✓` green / `✗` red / `•`
dim:

```
  ✓ stopped    killed PID 481920 · workspace #12 released
  ✓ name       released '02p' for reuse
  ✓ launched   PID 492011 · workspace #14
```

### 6.3 Receipt panel

A second rounded panel titled `Restarted · <name>` with the new PID, workspace, and
artifacts dir, followed by a dim `Next` block of copy-pasteable follow-ups:

```
  sase agent show 02p
  tail -f <artifacts_dir>/live_reply.md
  sase chat show 02p
```

### 6.4 Failure output

Every refusal names what was checked, what failed, and one concrete next step:

- `not_found` → include `sase agent list -a` and up to three near-miss suggestions from
  `difflib.get_close_matches()` over live agent names (`list_running_agents()`). Only
  compute suggestions on this error path.
- `no_prompt` → explain that `raw_xprompt.md` is absent and that ACE's `,x` can still
  reconstruct a prompt for historical rows.
- `preflight` / `name_not_reusable` → print the underlying message verbatim.
- kill failure → print `kill_named_agent()`'s message and state plainly that nothing was
  changed.
- `partial` → print the cause, state that the old run was stopped and the name released,
  and print the `sase run "$(cat ...)"` recovery command from §3.2.

### 6.5 JSON envelope

One object, `schema_version` as an int constant (the pattern
`INSTALL_AGENT_CLI_JSON_SCHEMA_VERSION` sets in `src/sase/agent_clis/cli_install.py`):

```json
{
  "schema_version": 1,
  "ok": true,
  "dry_run": false,
  "name": "02p",
  "project": "gh_sase-org__sase",
  "project_display": "sase",
  "prompt": {
    "source": "raw_xprompt.md",
    "vcs_tag": "#gh:sase",
    "name_reuse": "forced",
    "model_override": null
  },
  "stopped": {
    "action": "killed",
    "pid": 481920,
    "status": "killed",
    "artifacts_dir": "..."
  },
  "launched": { "pid": 492011, "artifacts_dir": "...", "workspace_num": 14 },
  "warnings": ["..."],
  "error": null
}
```

`--dry-run` sets `"dry_run": true` and leaves `stopped`/`launched` as `null`. Failures
set `"ok": false` and populate `"error"` with a `{reason, message}` object.

## 7. Confirmation And Exit Codes

Confirm only when the target is live, stdin is a TTY, and neither `-y` nor `-n` was
given. Render the preview panel, then ask `Restart agent '<name>'? [y/N] `, following
`_confirm_interactively()` / `_stdin_is_tty()` in `src/sase/agent_clis/cli_install.py`
(including its `EOFError` guard). A non-TTY invocation proceeds without prompting so
scripts and agents keep working; the warning lines still print. This is a weaker gate
than `sase agent-cli install`'s deliberately, because restarting your own agent is not
the same risk class as executing a remote install script.

Exit codes, documented in the epilog and in `docs/cli.md`:

- `0` — restarted, or a dry-run preview printed.
- `2` — refused before any mutation: not found, no prompt, multi-segment, preflight
  failure, declined confirmation, or a failed kill. Nothing changed. This matches
  `sase agent kill` and `sase agent show`, which already exit 2 on resolution failure.
- `1` — partial: the old run was stopped and the name released, but the relaunch failed.
  This is the only code that means "state changed and needs attention".

## 8. Files

Create:

- `src/sase/agent/relaunch_prompt.py` — move `prompt_facing_agent_name`,
  `rewrite_retry_prompt_name`, `force_name_reuse_in_prompt`, and
  `prepare_kill_and_edit_prompt` here from
  `src/sase/ace/tui/actions/agent_workflow/_entry_name_prompts.py`, and leave that
  module as a thin re-export so existing TUI imports and tests keep working unchanged.
  None of these four functions import Textual or any TUI module today; they only reach
  into `sase.agent.*`, `sase.plan_chain`, and `sase.xprompt.*`. The move exists so the
  CLI never imports from `sase/ace/tui/**`.
- `src/sase/agent/restart.py` — `AgentRestartPlan`, `AgentRestartError`,
  `AgentRestartOutcome`, `plan_agent_restart()`, `execute_agent_restart()`.
- `src/sase/agents/cli_restart.py` — argparse handler, confirmation, exit codes.
- `src/sase/agents/_restart_render.py` — panels, ledger, and JSON envelope, if
  `cli_restart.py` would otherwise grow past ~400 lines. `just lint` runs `toobig` with
  1000/850/700 thresholds; keep every new module comfortably under them.
- `src/sase/xprompt/_directive_edit_model.py` — `set_prompt_model()`.

Modify:

- `src/sase/main/parser_agent.py` — the `restart` subparser.
- `src/sase/main/agent_handler.py` — dispatch and usage string.
- `src/sase/agent/running.py` — `dismiss_named_agent()`.
- `src/sase/xprompt/directive_edit.py` — export `set_prompt_model` (`__all__` is
  alphabetized; keep it that way).
- `src/sase/completion/kinds.py` — the `agent restart` name slot.
- `src/sase/agents/cli_list.py` — promote `_STATUS_COLORS` to a shared helper.
- `docs/cli.md` — a `sase agent restart` row in the Daily Operation table, placed to
  keep the existing agent-command grouping readable, describing the flags and the three
  exit codes.
- `docs/ace.md` — one cross-reference from the `,x` / kill-and-edit description noting
  the CLI equivalent, if that section exists.

## 9. Tests

- `tests/_agent_restart_helpers.py` — a fixture that materializes a fake agent artifacts
  dir. Extend the existing `make_agent()` in `tests/_agent_names_fixtures.py` with a
  `raw_prompt` argument instead of writing a second builder; it already emits
  `agent_meta.json` with `agent_family` and `role_suffix`.
- `tests/test_agent_restart_plan.py` — planning is pure and refuses correctly: not
  found; missing `raw_xprompt.md`; multi-segment refusal; a family member keeps its
  `family--role` spelling and `phase_bead_id`; an `agent_family_parallel` member does
  **not** take the family branch; `--model` with and without `@effort`; a preflight
  failure surfaces the underlying message. Every one of these asserts that no marker,
  claim, name registry entry, or process was touched.
- `tests/test_agent_restart_execute.py` — execution ordering: the force-reuse preflight
  runs before the stop; a failed kill aborts before the name wipe and before any launch;
  a DONE agent takes the dismiss path rather than the kill path; a launch failure after
  the stop produces the `partial` outcome carrying the recovery command; a successful
  run wipes the name exactly once and launches exactly once.
- `tests/test_agent_restart_cli.py` — surface behavior: `--dry-run` prints the preview,
  changes nothing, exits 0; the JSON envelope shape and `schema_version`; a declined TTY
  confirmation exits 2 with nothing changed; `-y` skips the prompt; a non-TTY run does
  not prompt; not-found exits 2 and lists near-miss suggestions; a partial failure
  exits 1.

Assert on structure and substrings, not on exact panel geometry — Rich box drawing is
not a stable contract.

## 10. Decisions Already Made

Do not relitigate these while implementing.

- **No Rust core change.** `CLAUDE.md`'s backend-boundary rule points shared domain
  behavior at `../sase-core`. This command introduces no new domain rule: it composes
  services that already exist in Python and that ACE already calls —
  `kill_named_agent()`, `plan_force_reuse_launch()`, `launch_agents_from_cwd()`. The
  Rust crate today carries pure planners and wire types (`agent_launch`,
  `agent_cleanup`), and has no force-reuse or relaunch-prompt rewriting at all. The one
  genuinely shared piece this work creates — relaunch-prompt preparation — is promoted
  out of `sase/ace/tui/**` into `sase/agent/` so the CLI stays off TUI code. Migrating
  that text rewriting into `sase_core` is a separate follow-up, not part of this plan.
- **No feature flag.** Per `sase/memory/sase_flags.md`, a flag is a temporary route for
  behavior that reaches users _before_ it is ready. This command lands complete, tested,
  and unconditional.
- **No ACE/TUI behavior change.** `,x` keeps its edit-before-relaunch pause; restart is
  `,x` without the edit. Do not add a new keybinding.
- **No bulk restart.** One named agent per invocation. No `--all`, no marked-set
  support.
- **No `sase.ops` durable wiring.** `sase agent restart` is an operator command, not an
  ACE-submitted durable proc, so it takes no `add_operation_io_flags()` request/result
  plumbing. If ACE ever wants to submit it, that is a follow-up.

## 11. Verification

Run `just install` first — workspace virtualenvs go stale. Then `just check` inline.
Because this touches the launcher and agent-name machinery, hand `just check-full` to
`/sase_monitor` (`sase monitor start --command 'just check-full' --next ...`) before
landing; never run it inline.
