---
tier: tale
title: Telegram receiver registers slash commands and runs inbound housekeeping
goal:
  The service-host Telegram receiver keeps the bot's command menu current (so /usage
  appears) and delivers every tick-only follow-up (usage refreshes, gate/update
  completions, media groups, button cleanup) without an AXE tg_inbound tick.
size: medium
proposed_by: bbugyi200.athena.0tl
create_time: 2026-09-28 09:59:26
status: wip
---

# Plan: Make the Telegram receiver register slash commands and run inbound housekeeping

## Problem

`/usage` works when typed in Telegram, but it is missing from the bot's `/` command
menu. The live bot (`getMyCommands`, default scope; the private-chat and chat scopes are
empty) still advertises an old list that has no `usage` entry and still says `/show` =
"Show an agent, clan, **family**, or tribe". The code changed that wording to "session"
in release 0.4.20, and `/usage` arrived in 0.4.22. So Telegram has not received a new
command list since before 0.4.20.

## Root cause (verified)

All of this is in the linked `sase-telegram` repo (`sase repo open sase-telegram`).

1. The bot menu is set by `_register_commands_if_needed()` in
   `src/sase_telegram/inbound_handlers/commands.py`. It runs **only** from
   `_run_pre_poll_cleanup()` in `src/sase_telegram/scripts/sase_tg_inbound.py`, which is
   called only by the bare AXE job tick (`_run_chop_tick`) and by `--once`.
2. The persistent `--receiver` loop (`_run_receiver`) never calls
   `_run_pre_poll_cleanup()` or `_run_post_poll_cleanup()`.
3. On athena the receiver is now the service-host `telegram_receiver` proc. The user's
   global config dropped the `tg_inbound` job from the AXE `telegram` routine because
   long polling moved to the service proc. So in service-host mode nothing registers
   commands, and the receiver process handles every update. `/usage` works because
   dispatch runs in the receiver; the menu is stale because registration never runs
   there.
4. The same gap silently disables **all** tick-only housekeeping in service-host mode,
   not just the menu:
   - `_finish_ready_usage_refreshes()`: the `/usage` **Refresh** button's final edit
     never lands. The message stays on "⏳ Refreshing…" with the busy keyboard.
   - `_send_ready_gate_completions()`: gate-answer outcome messages are never sent. A
     record from 2026-09-27 18:54 is still sitting undelivered in
     `~/.sase/telegram/gate_completions/`.
   - `_send_ready_update_completions()`: `/update` completion messages.
   - `_flush_ready_media_groups()`: multi-image albums are staged but their agents are
     never launched.
   - `pending_actions.cleanup_stale()`, `_retry_pending_keyboard_cleanups()`, and the
     dismissal of buttons for actions resolved outside Telegram.
5. Separate test-isolation leak, which explains why the on-disk cache is misleading:
   `tests/test_integration.py::_patch_paths` patches `load_custom_commands` to `{}` but
   not `_COMMANDS_REGISTERED_PATH`. Integration tests that call
   `inbound_main(["--once"])` with a mocked `telegram_client` therefore write the
   **real** `~/.sase/telegram/commands_registered_ts`. The current file holds the
   fingerprint of the c386148 built-in list without custom commands, written at
   2026-09-27 16:16 while that commit's tests ran. No real registration produced it. If
   that fingerprint ever matched a real command list, a test run would suppress real
   registration for up to an hour.

Design intent in `receiver.py::ensure_receiver_running` confirms the service-host path
was meant to keep some tick running: it skips only the receiver launch when the service
host owns the receiver. The user's architecture (the receiver owns inbound, and the fast
AXE lane stays outbound-only) is reasonable, so the fix makes the receiver
self-sufficient instead of asking the user to re-add the tick.

## Changes (all in `sase-telegram`)

### 1. Shared housekeeping lock (tick, `--once`, and receiver)

In `src/sase_telegram/scripts/sase_tg_inbound.py`, add a module-level
`_HOUSEKEEPING_LOCK_FILE = Path.home() / ".sase" / "telegram" / "inbound_housekeeping.lock"`.
Add a small non-blocking `fcntl.flock` context manager, modeled on
`outbound.try_acquire_outbound_lock` / `release_outbound_lock`, that yields whether the
lock was acquired. When the lock is busy, skip housekeeping: another process is already
doing the same work. This prevents duplicate completion messages when a legacy AXE tick
and the receiver both run housekeeping. Wrap the pre-poll and post-poll cleanup calls in
`_run_chop_tick` and `_run_once` with it. Keep their existing behavior and summary
output when the lock is acquired. When it is busy, treat the counts as 0.

### 2. Receiver runs housekeeping every iteration

In `_run_receiver`, after the credential checks, the
`_runtime_requires_refresh(baseline)` check, and
`custom_commands = load_custom_commands()`:

- Before `get_updates`, run `_run_pre_poll_cleanup(custom_commands)` under the
  housekeeping lock. This includes `_register_commands_if_needed`, so the menu is
  registered when a receiver first starts (and after every runtime re-exec, because the
  fingerprint differs), then hourly or whenever the list changes.
- After `_dispatch_fetched_updates(...)`, run `_run_post_poll_cleanup()` under the same
  lock.
- Guard both calls with a broad `try/except Exception` that logs
  `log.warning(..., exc_info=True)` and continues. A housekeeping failure must never
  stop the receiver from polling or change its exit codes. The existing credential and
  chat-id exit paths stay as they are.
- Do not print the `tg_inbound:` summary line for housekeeping-only iterations. Keep the
  current rule: print only when updates were dispatched. `ready_completions_sent` and
  `pending_actions_cleaned` may carry the housekeeping counts on those lines.

### 3. Adaptive long-poll timeout while follow-ups are pending

Receiver-run housekeeping would otherwise run only once per 30 s idle long poll. Keep
`_RECEIVER_POLL_TIMEOUT_SECONDS = 30` for idle waits and add
`_RECEIVER_FOLLOWUP_POLL_TIMEOUT_SECONDS = 5`. Add `_has_recent_pending_followups(now)`.
It returns True when any durable follow-up record modified within the last
`_FOLLOWUP_FAST_POLL_WINDOW_SECONDS = 600` exists in:

- `inbound.GATE_COMPLETION_PENDING_DIR` (`*.json`)
- `usage_command._USAGE_REFRESH_PENDING_DIR` (`*.json`)
- `update_command._UPDATE_COMPLETION_PENDING_DIR` (`*.json`)
- `keyboard_cleanup._GATE_KEYBOARD_CLEANUP_DIR` (`*.json`)
- a non-empty staged media-group store (`images._MEDIA_GROUPS_PATH`, via the existing
  `_load_media_groups()`)

Pass the short timeout to `telegram_client.get_updates` when it returns True. The
recency window is required: gate-completion records have no expiry (an unresolved one
can live forever, like the stuck 2026-09-27 record), and without the window the receiver
would stay in 5 s polling permanently. Older records are still processed on every
iteration at the normal cadence. Treat `OSError` while scanning as "no pending
follow-ups". Read the directory and file constants from their owning modules at call
time, not through copied constants, so test patches through `inbound_namespace.INBOUND`
reach them.

### 4. Test isolation for the registration cache

- In `tests/conftest.py`, add an autouse fixture that monkeypatches
  `sase_telegram.inbound_handlers.commands._COMMANDS_REGISTERED_PATH` and the new
  `sase_telegram.scripts.sase_tg_inbound._HOUSEKEEPING_LOCK_FILE` to paths under
  `tmp_path`, so no test can write the real `~/.sase/telegram/` registration cache or
  lock. Existing tests that patch `_COMMANDS_REGISTERED_PATH` themselves keep working,
  because their patch applies on top.
- Also add both paths to `tests/test_integration.py::_patch_paths` if its temp-file
  cleanup (`_cleanup_files`) needs to know about them. Otherwise the conftest fixture is
  enough.
- Any new receiver-loop tests must also redirect the follow-up directories they touch.
  Reuse the existing `_patch_paths` / `inbound_namespace.INBOUND` patching style.

### 5. Tests

Add these to `tests/test_integration.py` (next to `TestReceiverLoop`, reusing its
`_stable_runtime` fixture and its pattern of patching `is_telegram_enabled` /
`credentials` / `telegram_client` so the `while True` loop runs one or two iterations
and then exits through the disabled path):

- The receiver registers slash commands before its first `get_updates`:
  `set_my_commands` is called with sorted custom commands followed by `_SLASH_COMMANDS`
  (which includes `usage`), and the redirected cache file records the matching
  fingerprint.
- The receiver runs pre-poll cleanup before polling and post-poll cleanup after
  dispatch. For example, a ready usage-refresh or gate-completion record is delivered by
  the receiver alone, and a staged media group whose quiet window has elapsed is
  launched.
- If housekeeping raises, the receiver logs it and still calls `get_updates` in the same
  iteration.
- When the housekeeping lock is held by another holder (acquire it in the test), the
  receiver skips housekeeping and still polls.
- `get_updates` receives `timeout=5` when a fresh follow-up record exists and
  `timeout=30` when none exist or only records older than the window exist.
- The tick and `--once` still run cleanup when the lock is free and skip it when the
  lock is held.
- A regression check: running the existing `inbound_main(["--once"])` integration tests
  leaves the (monkeypatched) real-home registration path untouched. The conftest fixture
  makes this structural; one direct assertion is enough.

### 6. Docs

Update these to say the long-poll receiver itself registers the bot command menu and
runs inbound housekeeping (completion delivery, `/usage` refresh finishing, media-group
flushing, stale/handled button cleanup) under a shared lock. Under the service host, the
AXE `tg_inbound` tick is therefore optional. It remains supported for the legacy
durable-proc receiver, where it also re-arms the receiver every tick.

- `docs/inbound.md` ("CLI Usage" and "Long-Poll Receiver" sections)
- `docs/architecture.md` ("Inbound" data flow)
- `README.md` (the service-host `telegram_receiver` paragraph)
- the `receiver.py` module docstring and the `_run_receiver` / `_run_chop_tick`
  docstrings

Mention the adaptive 5 s / 30 s poll timeout in `docs/inbound.md`.

## Out of scope

- The user's chezmoi-managed `~/.config/sase/sase.yml` stays unchanged. Its comment
  ("Inbound Telegram long-polling is the `telegram_receiver` service proc on athena, not
  a job in this lane") remains accurate after this fix.
- No sase core (`sase` / `sase-core`) change. Registration stays at the default
  `BotCommandScope`, which is where Telegram currently serves the stale list; no scoped
  overrides exist.
- Adding an expiry to gate-completion records. The recency window above keeps unresolved
  records from forcing fast polling. File a follow-up bug bead if the coder confirms
  such records can never resolve.
- Broader `$HOME` isolation for the remaining `~/.sase/telegram/` module paths beyond
  the two named above.

## Verification

- In the `sase-telegram` checkout run `sase tool run check` (the guarded lint + test
  recipe; do not run bare `just check`). It must pass.
- Confirm by inspection that `~/.sase/telegram/commands_registered_ts` is not modified
  by the test run (compare its mtime before and after).
- Post-deploy, which is not the coder's job but goes in the final summary for the user:
  once the editable `sase-telegram` install picks up the change, the running
  `telegram_receiver` re-execs on its runtime-generation change, or the user can run
  `sase service proc restart telegram_receiver`. It registers the new list on its first
  iteration. A read-only `getMyCommands` should then list `usage` and the "session"
  wording for `/show`. Telegram mobile clients cache the menu, so reopening the chat or
  restarting the app may be needed before `/usage` appears.
- Expect the one stuck 2026-09-27 gate-completion record to be delivered late (or to
  linger harmlessly) once receiver housekeeping starts. Mention this in the final
  summary so a late message is not a surprise.
