---
tier: tale
title: Split sase_tg_inbound.py into an inbound_handlers package
goal:
  sase-telegram's 5,150-line inbound script becomes a thin entry point plus a layered
  inbound_handlers package in which every file is at most 700 lines, as a pure move with
  no behavior change, and the test suite passes with its current patch semantics.
size: medium
proposed_by: bbugyi200.athena.0q2
create_time: 2026-09-23 10:53:51
status: wip
---

# Split `sase_tg_inbound.py` into an `inbound_handlers` package

## Context

`src/sase_telegram/scripts/sase_tg_inbound.py` in the linked **sase-telegram** repo is
5,150 lines. It holds the inbound entry point (`main`, the `--once` / chop-tick /
`--receiver` run modes) **and** every Telegram update handler: questions, gates,
`/kill`, `/list`, `/show`, `/bead`, `/changes`, `/xprompts`, `/update`, custom commands,
photos and albums, agent launching, completion delivery, keyboard cleanup, and update
dispatch.

All work happens in the sase-telegram repo. Open it with
`sase repo open sase-telegram -r "<reason>"`, work only in the path it prints, and read
its `AGENTS.md` first. The sase repo itself does not change. Nothing outside
sase-telegram imports `sase_telegram.scripts.sase_tg_inbound`. The console scripts reach
the module through `sase_telegram.scripts:inbound_main`, which imports `main` lazily.

**The main risk is the tests, not the move itself.** About 650 test sites patch
`sase_telegram.scripts.sase_tg_inbound.<name>`. Many replace whole modules:
`telegram_client` 188×, `credentials` 101×, `pending_actions` 45×. Others patch shared
helpers such as `_launch_agent` 25× or `_record_project_context` 12×, or even the whole
`time` module. Many tests drive flows that will cross several of the new modules;
`inbound_main(["--once"])` integration tests cross nearly all of them. A plain split
leaves those patches applying to one module while the code under test looks names up in
others. The tests would then silently hit real functions.

## Goals and non-goals

- **Goal:** every new module and the slimmed script stay at **≤700 lines** (`wc -l`).
  The layout below lands between roughly 140 and 470 lines per file, which leaves
  headroom so the files do not grow back past the limit.
- **Goal: a pure move.** Function, class, and constant bodies move **verbatim**, with no
  renames (keep the `_` prefixes), no signature changes, and no logic edits. Only these
  change: import headers, module docstrings, one `log = logging.getLogger(__name__)` per
  module that logs, and comments that point at code by position ("mirroring … above").
  Reviewers should be able to confirm the move with `git diff --color-moved=zebra`.
- **Goal:** the test suite passes with the same semantics it has today.
- **Non-goals:**
  - Renaming private helpers into a public API.
  - Moving logic into sase-core.
  - CLI or behavior changes.
  - Splitting the large test files (`tests/test_inbound.py` is 5.9k lines).
  - Editing `CHANGELOG.md`, which release-please manages.

## Design

### Package location

Create a new package `src/sase_telegram/inbound_handlers/`. It cannot be called
`sase_telegram/inbound/` because that would collide with the existing pure-logic module
`sase_telegram/inbound.py`. `scripts/sase_tg_inbound.py` stays as the entry-point file,
so its import path, `main`, and the `scripts/__init__.py` wrapper do not change.
`inbound_handlers/__init__.py` holds only a docstring. It has no imports and no
re-exports; the docstring states the layering rule below and points at
`tests/inbound_namespace.py`.

### Module map

Modules are listed in **layer order**. A module may import only from modules listed
above it. This order was checked for acyclicity against the real call graph. Within each
module:

- Constants and dataclasses come first, then functions.
- Each group keeps its original relative order from the old file.
- Comment lines directly above a symbol move with it: section banners, the `#:` docs,
  and the comment above `_GATE_KEYBOARD_CLEANUP_DIR`.

Line counts are estimates that include imports.

| #   | Module                       | Responsibility                                                                                         | ~Lines |
| --- | ---------------------------- | ------------------------------------------------------------------------------------------------------ | ------ |
| 1   | `common.py`                  | Shared chat/message/callback primitives, response writing, JSON file helpers                           | 220    |
| 2   | `project_context.py`         | Prompt → project tag parsing, per-chat project context, workspace discovery, bead cwd resolution       | 345    |
| 3   | `agent_launch.py`            | Launching agents from Telegram, launch notifications, agent action keyboard, retry prompts             | 440    |
| 4   | `questions.py`               | User-question callbacks and free-text answers                                                          | 470    |
| 5   | `keyboard_cleanup.py`        | Durable inline-keyboard removal retries and dismissal of resolved actions                              | 145    |
| 6   | `gate_completions.py`        | Delivering gate-answer outcomes after the supervised proc finishes                                     | 140    |
| 7   | `update_command.py`          | `/update` worker start, acknowledgement, and completion delivery                                       | 150    |
| 8   | `xprompt_commands.py`        | `/changes` and `/xprompts`                                                                             | 190    |
| 9   | `beads.py`                   | `/bead` picker, bead show, and `sase bead` subprocess helpers                                          | 365    |
| 10  | `images.py`                  | Photo and document-image messages and media-group (album) staging and flushing                         | 360    |
| 11  | `agent_actions.py`           | `/kill`, the kill-selection picker, Kill/Retry buttons, `/fork`                                        | 350    |
| 12  | `gate_response.py`           | Submitting a gate response, error replies, starting feedback, input prompts, dismissing gate keyboards | 190    |
| 13  | `agent_list.py`              | `/list` overview, detail, and callbacks                                                                | 315    |
| 14  | `gate_input_steps.py`        | Multi-step gate input collection, from both callbacks and text                                         | 260    |
| 15  | `agent_show.py`              | `/show` rendering and callbacks                                                                        | 195    |
| 16  | `gate_callbacks.py`          | Gate button entry point and option selection                                                           | 270    |
| 17  | `commands.py`                | Slash-command router, bot command-menu registration, custom command delivery                           | 280    |
| 18  | `callbacks.py`               | Callback-query router and the already-handled race guard                                               | 150    |
| 19  | `text_messages.py`           | Text-message router and stale awaiting-feedback cleanup                                                | 170    |
| 20  | `dispatch.py`                | Per-update routing, chat authentication, poll-and-dispatch, dispatch-failure replies                   | 190    |
| —   | `scripts/sase_tg_inbound.py` | Entry point: args, run modes, pre/post-poll housekeeping, receiver loop, runtime refresh               | 425    |

Every top-level symbol of the current file is assigned exactly once below.

- **common**: `_COPY_TEXT_MAX`, `_LAUNCH_AGENTS_DISABLED_ENV`,
  `_STALE_AWAITING_FEEDBACK_TEXT`, `_telegram_agent_launches_disabled`,
  `_message_chat_id`, `_configured_chat_id`, `_context_chat_id`, `_message_id`,
  `_action_message_id`, `_callback_origin_message_id`, `_callback_chat_id`,
  `_answer_callback`, `_answer_resolved_callback`, `_write_response`,
  `_resolve_response`, `_send_confirmation`, `_gate_error_answer_text`,
  `_clear_awaiting_feedback_entry`, `_load_json_file`, `_atomic_write_json`,
  `_shorten_home`, `_send_html_chunks`
- **project_context**: `_KNOWN_VCS_WORKFLOWS`, `_VCS_WORKFLOW_PATTERN`,
  `_VCS_PROJECT_PATTERN`, `_VCS_PROJECT_RE`, `_DIRECTIVE_PREFIX_RE`,
  `_PROJECT_CONTEXT_PATH`, `_BEAD_PROJECT_ENV`, `_KnownProjectWorkspace`,
  `_ProjectDiscoveryResult`, `_project_from_vcs_match`, `_extract_project_from_prompt`,
  `_load_project_context`, `_save_project_context`, `_record_project_context`,
  `_workspace_from_context_entry`, `_pending_action_chat_id`, `_iter_pending_prompts`,
  `_workspace_from_project_file`, `_project_spec_path`,
  `_resolve_workspace_from_project_file`, `_iter_known_project_workspaces`,
  `_resolve_workspace_for_project`, `_bead_project_override`, `_resolve_bead_cwd`
- **agent_launch**: `_get_agent_retry_prompt`, `_build_retry_prompt_for_agent`,
  `_launch_agent`, `_prompt_has_pr_xprompt`, `_launch_agents_with_notifications`,
  `_resolve_slot_prompts`, `_resolve_launch_result_agent_name`,
  `_launch_result_artifacts_dir`, `_launch_provider_model_label`, `_agent_vcs_prefix`,
  `_build_agent_action_keyboard`, `_send_launch_notification`
- **questions**: `_edit_question_answered`, `_save_pending_action_message`,
  `_send_next_question`, `_question_response_action`, `_handle_question_decision`,
  `_handle_question_callback`, `_awaiting_key_from_progress`,
  `_clear_question_awaiting`, `_handle_question_text_message`
- **keyboard_cleanup**: `_GATE_KEYBOARD_CLEANUP_DIR`, `_keyboard_cleanup_retry_path`,
  `_persist_keyboard_cleanup_pending`, `_clear_keyboard_cleanup_pending`,
  `_dismiss_button_with_retry`, `_retry_pending_keyboard_cleanups`,
  `_dismiss_resolved_button`, `_find_shared_handled_transports`
- **gate_completions**: `_latest_gate_execution_error`, `_format_gate_response_success`,
  `_format_gate_execution_error`, `_format_gate_proc_failure`,
  `_send_ready_gate_completions`, `_gate_answer_proc_status`. The "Gate answer
  completion delivery" banner comment moves here too; change its "mirroring
  `_send_ready_update_completions` above" to point at `update_command`.
- **update_command**: `_UPDATE_COMPLETION_PENDING_DIR`, `_ChatInstallUnavailableResult`,
  `start_chat_install_worker`, `_handle_update_command`, `_format_update_ack`,
  `_persist_update_completion_pending`, `_send_ready_update_completions`,
  `_format_update_completion`
- **xprompt_commands**: `_CHANGES_BUTTON_CHUNK_SIZE`, `_list_patch_xprompt_tags`,
  `_handle_changes_command`, `_format_patch_skipped_note`, `_changes_button_label`,
  `_format_xprompts_caption`, `_handle_xprompts_command`
- **beads**: `_BEAD_PICKER_LIMIT`, `_BEAD_BUTTON_LABEL_MAX`,
  `_BEAD_LIST_ERROR_SUMMARY_MAX`, `_ACTIVE_BEAD_LIST_ARGS`, `_ProjectBeadEntry`,
  `_run_bead_command`, `_run_active_bead_list`, `_project_bead_token`,
  `_split_project_bead_token`, `_send_bead_subprocess_error`,
  `_send_project_discovery_error`, `_summarize_bead_list_error`,
  `_project_bead_entries`, `_legacy_bead_entries`, `_render_bead_selection`,
  `_show_bead_selection`, `_bead_show_result`, `_handle_bead_callback`,
  `_handle_bead_command`
- **images**: `_MEDIA_GROUPS_PATH`, `_MEDIA_GROUP_QUIET_SECONDS`,
  `_MediaGroupMessageContext`, `_media_group_id`, `_media_group_key`,
  `_load_media_groups`, `_save_media_groups`, `_message_caption`,
  `_safe_document_filename`, `_make_document_image_filename`,
  `_media_group_item_from_message`, `_media_group_items`, `_stage_media_group_image`,
  `_download_media_group_item`, `_launch_media_group`, `_flush_ready_media_groups`,
  `_handle_photo_message`, `_handle_document_image`
- **agent_actions**: `_KILL_SELECTION_PENDING_KEY`, `_KILL_SELECTION_CHOICE`,
  `_build_redo_prompt_for_killed_agent`, `_send_kill_result`,
  `_handle_kill_from_callback`, `_handle_kill_selection_from_callback`,
  `_handle_retry_from_callback`, `_format_agent_description`, `_show_kill_selection`,
  `_handle_kill_command`, `_handle_fork_command`
- **gate_response**: `_dismiss_gate_callback`, `_execute_gate_callback_response`,
  `_send_gate_response_error`, `_begin_gate_feedback`, `_send_gate_input_prompt`
- **agent_list**: `_ListCommandArgs`, `_parse_list_args`, `_active_list_entries`,
  `_load_list_entries`, `_find_entry_by_name`, `_handle_list_command`,
  `_handle_list_callback`, `_render_list_overview`, `_group_list_entries`,
  `_format_list_header`, `_format_list_footer`, `_build_list_overview_keyboard`,
  `_send_list_detail`, `_format_list_detail`, `_get_detail_prompt`
- **gate_input_steps**: `_handle_gate_input_callback`, `_advance_gate_input`,
  `_handle_gate_input_text_message`
- **agent_show**: `_handle_show_command`, `_render_show_reference`,
  `_build_show_keyboard`, `_show_button`, `_handle_show_callback`
- **gate_callbacks**: `_reject_tty_required_selection`,
  `_start_or_submit_gate_selection`, `_handle_gate_callback`
- **commands**: `_COMMANDS_REGISTERED_PATH`, `_COMMANDS_REGISTER_INTERVAL`,
  `_SLASH_COMMANDS`, `_CUSTOM_COMMAND_CAPTION_LIMIT`, `_CUSTOM_COMMAND_STDERR_LIMIT`,
  `_handle_command`, `_custom_command_caption`, `_custom_command_pdf_filename`,
  `_send_custom_command_error`, `_send_custom_markdown`, `_handle_custom_command`,
  `_slash_commands_fingerprint`, `_registered_slash_commands`,
  `_commands_registration_is_current`, `_register_commands_if_needed`. `_SLASH_COMMANDS`
  sits next to the `_handle_command` router so a new command updates both in one file.
- **callbacks**: `_shared_action_resolution`, `_resolve_callback_already_handled`,
  `_handle_callback`
- **text_messages**: `_awaiting_feedback_prefix`, `_pending_action_exists`,
  `_clear_stale_awaiting_feedback_entry`, `_clear_stale_awaiting_feedback`,
  `_send_stale_awaiting_feedback_reply`, `_handle_text_message`
- **dispatch**: `_callback_query_sender_id`, `_update_is_from_configured_chat`,
  `_dispatch_one_update`, `_PollResult`, `_poll_and_dispatch_updates`,
  `_reply_dispatch_failure`, `_dispatch_fetched_updates`
- **scripts/sase_tg_inbound.py (stays)**:
  - Its own `log`.
  - The `_RECEIVER_*` constants (six of them, with their `#:` comments).
  - `_print_inbound_summary`, `_parse_args`, `_run_pre_poll_cleanup`,
    `_run_post_poll_cleanup`, `_notify_receiver_chat_id_missing`, `_run_once`,
    `_run_chop_tick`, `_run_receiver`, `_runtime_requires_refresh`,
    `_refresh_receiver_runtime`, `main`.
  - The `if __name__ == "__main__":` guard.

  Keep the module docstring and add one sentence saying the handlers live in
  `sase_telegram.inbound_handlers`.

### Import rules

These rules are what make the test strategy below exact:

1. **Keep the original binding names and import style.** A moved function must find
   every name through a module-global binding **with the same name** it had in the old
   file.
   - Keep `from sase_telegram import credentials, pending_actions, telegram_client, …`
     module imports.
   - Keep `import time`, `import subprocess`, `import os`, and
     `from pathlib import Path`.
   - Keep aliases such as `clear_progress as clear_gate_progress` and
     `load_progress as load_gate_progress`.
   - Never switch to `import x as y`, attribute-style access, or a different alias.
2. Cross-module references inside the package use absolute name imports, for example
   `from sase_telegram.inbound_handlers.agent_launch import _launch_agent`. This follows
   the AGENTS.md convention and the layer order above.
3. Every symbol is defined in exactly one module. Constants shared by several modules
   live in `common`. The one intentional exception is each module's own
   `log = logging.getLogger(__name__)`. Logger names change from
   `sase_telegram.scripts.sase_tg_inbound` to per-module names; no test or config
   depends on the old name.
4. Import only what the module uses. The script's current dead imports are not carried
   anywhere: `_detail_rows`, `_entry_display_name`, `_entry_model_label`,
   `_format_detail_grid`, `_format_status_token`, `build_fork_copy_text`.
5. Function-local imports, for example `from sase.agent.names import find_named_agent`,
   stay inside their functions unchanged.

### Test strategy: one patchable namespace (`tests/inbound_namespace.py`)

Add a test-only helper module (the `tests/` directory is on pytest's `pythonpath`). It
exposes `INBOUND`, a proxy over **all** inbound modules: the script plus every submodule
of `sase_telegram.inbound_handlers`, discovered with `pkgutil` and cached. The proxy
works like this:

- `getattr(INBOUND, name)` returns the object that the inbound modules bind to `name`.
  It raises `AttributeError` if no module binds the name, or if two modules bind it to
  different objects. `log` and dunders are never shared.
- `setattr(INBOUND, name, value)` rebinds `name` in **every** inbound module that binds
  it.
- A `__dict__` property exposes consistently-bound names, so `unittest.mock` treats a
  patched name as "local" and restores it through `setattr`, which fans out. The class
  uses `__slots__ = ()`, and `__delattr__` raises.

The proxy works as a `patch.object` target and as a dotted string path. It **reproduces
today's single-namespace patch semantics exactly**: patching `telegram_client`,
`_launch_agent`, or even `time` reaches every function that could see it before the
split. The same mechanism works with:

- `@patch(...)` decorators, including pytest's mock-argument counting next to fixtures
- context managers
- `start()` / `stop()`
- `return_value=` / `side_effect=`
- `monkeypatch.setattr` with an object or a string target
- attribute chains such as `INBOUND.telegram_client.send_message`
- typo detection

This design was prototyped and all of those cases passed. Sketch:

```python
_PACKAGE = "sase_telegram.inbound_handlers"
_SCRIPT = "sase_telegram.scripts.sase_tg_inbound"

@cache
def inbound_modules() -> tuple[ModuleType, ...]:
    package = importlib.import_module(_PACKAGE)
    names = [_SCRIPT, _PACKAGE, *(f"{_PACKAGE}.{i.name}" for i in pkgutil.iter_modules(package.__path__))]
    return tuple(importlib.import_module(n) for n in names)

class _InboundNamespace:
    __slots__ = ()
    def _owners(self, name): return [m for m in inbound_modules() if name in vars(m)]
    def _lookup(self, name):  # AttributeError if unbound, dunder, or bound to different objects
        ...
    @property
    def __dict__(self): ...   # mapping whose __getitem__ delegates to _lookup (KeyError on miss)
    def __getattr__(self, name): return self._lookup(name)
    def __setattr__(self, name, value):
        self._lookup(name)
        for module in self._owners(name): setattr(module, name, value)
    def __delattr__(self, name): raise AttributeError(name)

INBOUND = _InboundNamespace()
```

Why not retarget each patch to one owning module? That would take a separate analysis
for each of about 650 sites, across flows that span many modules. It would break again
every time a helper moves, and a missed site would hit real Telegram or credential code
without any error. The proxy keeps tests independent of the internal module layout.

**Migration rules for the tests.** They apply to `test_inbound.py`,
`test_integration.py`, `test_custom_gates.py`, and `test_receiver_exits.py`. Leave
`test_enabled.py` alone: its `…sase_tg_inbound.main` patches target the real wrapper
import path, and `main` stays in the script.

- **R1 (string targets, sed):** replace the prefix
  `"sase_telegram.scripts.sase_tg_inbound.` with `"inbound_namespace.INBOUND.`
  everywhere. This covers `patch(...)`, `@patch(...)`, `monkeypatch.setattr("…")`, and
  chains such as `…telegram_client.send_message`, `…subprocess.run`, `…os.execvp`, and
  `…Path.home`.
- **R2 (module aliases, sed):** replace
  `from sase_telegram.scripts import sase_tg_inbound as <alias>` with
  `from inbound_namespace import INBOUND as <alias>`. The aliases in use are `inbound`
  and `tg_inbound`. After this, `inbound._run_receiver()`, `patch.object(inbound, …)`,
  and `inbound._KnownProjectWorkspace(...)` all resolve through the proxy. Do not touch
  imports of the pure `sase_telegram.inbound` module.
- **R3 (direct imports):** rewrite each
  `from sase_telegram.scripts.sase_tg_inbound import (…)` to import each name from the
  module that owns it in the module map. There are about 120 imported names, 29 of them
  distinct. Two cases keep their import:
  - `from sase_telegram.scripts.sase_tg_inbound import main as inbound_main`
  - `main` imports in general
- Add `tests/test_inbound_namespace.py` with two guard tests:
  1. Every `.py` file under `src/sase_telegram/inbound_handlers/` is in
     `inbound_modules()`.
  2. No top-level name except `log` and dunders is bound to different objects in two
     inbound modules. This enforces import rules 1 and 3, which the proxy depends on.

## Implementation steps

1. Open the sase-telegram repo, read `AGENTS.md`, and run `sase tool run check` for a
   baseline. Note any failures that already exist.
2. **Migrate the tests first, while the code is still one file.**
   - Create `src/sase_telegram/inbound_handlers/__init__.py`, docstring only.
   - Add `tests/inbound_namespace.py` and `tests/test_inbound_namespace.py`.
   - Apply R1 and R2.
   - Run the tests. They must be green now, because the proxy over (script + empty
     package) behaves exactly like the old module. This proves the migration preserves
     semantics before any code moves.
3. **Split the source.** Write a throwaway extraction script outside the repo. It
   should:
   - parse the old file with `ast`;
   - slice each top-level statement verbatim, together with the comment lines directly
     above it;
   - write each module in the map in layer order, with a one-line module docstring,
     `from __future__ import annotations`, the imports it needs, `log` if used, then the
     moved statements;
   - rewrite the script with only what stays.

   To compute each module's imports, collect the `ast.Name` references its moved
   statements make. Resolve names defined elsewhere in the package to that module, and
   external names to their original import lines, keeping the same aliases. Then run
   `ruff format`.

4. **Check the imports strictly.** The repo's ruff config ignores `F401` and `F821`, so
   run `ruff check --isolated --select F401,F811,F821` over the new package and the
   script once. Fix every missing or unused import. Then run `mypy` through the normal
   lint. Any import cycle means a symbol is in the wrong layer: fix the placement, not
   the rules.
5. **Apply R3** to the direct test imports, then run the full test suite. A proxy error
   saying "no inbound module binds X" or "bound to different objects" means an import
   rule was broken in the source. Fix the source; do not weaken the test. If a patch
   target was genuinely unused before the split, check against the original file before
   removing that patch.
6. **Update the docs and positional references:**
   - `README.md` project tree: add the `inbound_handlers/` package with a short
     per-module note, and update the `sase_tg_inbound.py` line.
   - `AGENTS.md` Architecture: add an `inbound_handlers/` entry, and note that tests
     patch inbound modules through `tests/inbound_namespace.py`.
   - `docs/architecture.md`: in the "Pure logic separation" bullet, say that handlers
     live in `inbound_handlers/` and the script only wires the run modes.
   - `src/sase_telegram/receiver.py` module docstring: handlers now live in
     `inbound_handlers`.
   - `src/sase_telegram/inbound.py`: update its three comment references to
     `sase_tg_inbound._send_ready_gate_completions` /
     `_persist_update_completion_pending` to point at the new modules.
   - Moved comments that say "above" or "below" about code that is now in another
     module.
7. **Verify** (see the acceptance criteria), then use `git diff --color-moved=zebra` to
   confirm the source diff is almost entirely moved blocks.

## Acceptance criteria

- `wc -l` is ≤700 for `scripts/sase_tg_inbound.py` and for every file in
  `src/sase_telegram/inbound_handlers/`.
- Every top-level symbol of the original file exists exactly once, in the module the map
  assigns it to, with its body unchanged.
- The package imports cleanly in any order, with no import cycles, and follows the layer
  order.
- `sase tool run check` passes: ruff, mypy, and pytest. The new guard tests are
  included.
- `grep -rn "sase_tg_inbound" tests/` shows only `main` imports and `test_enabled.py`'s
  `main` patches.
- The `sase_job_tg_inbound`, `sase_chop_tg_inbound`, and `--receiver` entry points
  behave the same as before. The `--once` integration tests pass unmodified apart from
  R1–R3.

## Out of scope and possible follow-ups

- A line-count guard test that keeps inbound modules at ≤700 lines. Add it only if the
  user wants that policy enforced in CI.
- Splitting `tests/test_inbound.py` (5.9k lines) along the same module boundaries.
- Promoting cross-module `_` helpers to public names.
