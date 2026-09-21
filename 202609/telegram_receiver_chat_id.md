---
tier: tale
title:
  Give the service-host Telegram receiver its bot identity and fail closed without it
goal:
  Telegram text/photo messages launch agents again on athena, and a missing Telegram
  chat id can never again silently drop messages or let updates from other chats
  through.
size: medium
proposed_by: bbugyi200.athena.0ow
create_time: 2026-09-21 18:31:09
status: wip
---

# Restore Telegram agent launches: give the service-host receiver its bot identity, and fail loudly instead of silently

## Diagnosis (what happened)

A Telegram text message ("#gh:sase The `research.24` sase agent just failed…", update_id
`264361664`, ~17:47 EDT on 2026-09-21) never launched an agent. The inbound receiver did
receive it. The receiver's log (`~/.sase/service/procs/telegram_receiver/output.log`)
shows:

```
Processing text message (update_id=264361664)
Launching agent for prompt: #gh:sase The `research.24` sase agent just failed. …
Failed to process Telegram update_id=264361664
  … _handle_text_message → _launch_agent → _launch_agents_with_notifications
  … credentials.get_chat_id()
RuntimeError: SASE_TELEGRAM_BOT_CHAT_ID environment variable is not set
```

**Root cause.** The Telegram bot identity (`SASE_TELEGRAM_BOT_CHAT_ID`,
`SASE_TELEGRAM_BOT_USERNAME`) only exists as `env:` on the AXE `telegram` routine in the
chezmoi-managed `~/.config/sase/sase.yml`. Before 2026-09-19 the receiver was spawned by
the `tg_inbound` job in that routine, so it inherited the identity. Chezmoi commit
`2f03d015` ("migrate gateway and telegram onto the service host", bead `sase-11y.9`)
dropped the `tg_inbound` job and enabled `service.procs.telegram_receiver` in the athena
overlay (`sase_athena.yml`). It did not give that service proc the identity env. The
service host (`sase.service` → `sase service run`) launches procs with the captured
service env (`~/.sase/service/env`: PATH, SSH agent, one API key) plus the entry's
`env:`, which is empty. You can confirm this on the running receiver:
`/proc/<receiver-pid>/environ` has no `SASE_TELEGRAM_*` variables. `get_chat_id()` reads
only the process env, so every text/photo launch now raises before
`launch_agents_from_cwd` is ever called.

**Why it was silent, and a security gap it exposed (sase-telegram defects):**

1. `_launch_agents_with_notifications` calls `credentials.get_chat_id()` outside its
   `try`. That is exactly the value it would need to send the "Failed to launch agent"
   reply. The exception escapes to `_dispatch_fetched_updates`, which only logs it and
   advances the offset. The message is consumed, nobody is told, and it cannot be
   replayed.
2. `_update_is_from_configured_chat` fails open: when the configured chat id cannot be
   resolved, it lets every update through "unfiltered". Since the migration, the live
   receiver has been accepting updates from _any_ Telegram chat. Handlers that reply to
   the message's own chat (for example `/show`, which uses
   `_message_chat_id(message) or get_chat_id()`) are reachable by strangers. Launches
   only failed by accident.
3. Nothing detects the misconfiguration. The receiver starts, polls, and reports healthy
   with no chat id. Callbacks (gate buttons) kept working because they take the chat id
   from the callback's message, so only free-form launches broke.

Callbacks were unaffected. Only text/photo launches were lost. The log shows one lost
launch: the `research.24` prompt. Telegram cannot redeliver it because the offset
already moved past it, so **the user must resend that prompt after the fix**.

## Goal

1. Restore launches now by giving the `telegram_receiver` service proc the bot identity
   env (chezmoi config).
2. Make the plugin fail closed and loudly when the identity is missing, so this
   regression class can never again silently drop messages or open the bot to strangers
   (sase-telegram).

## Repositories

Open each repo with `sase repo open <name> -r "<reason>"` and use only the printed path:

- `chezmoi` (linked repo): dotfiles source for `~/.config/sase/sase.yml` and
  `~/.config/sase/sase_athena.yml`.
- `sase-telegram` (linked repo): the Telegram plugin.

The sase repo itself needs no changes. Both repos above become commit obligations for
the final declaration.

## Part A: chezmoi config fix (restores launches)

1. `home/dot_config/sase/sase_athena.yml`: under the existing
   `service.procs.telegram_receiver` entry (next to `enabled: true`), add:

   ```yaml
   env:
     SASE_TELEGRAM_BOT_USERNAME: sase_athena_bot
     SASE_TELEGRAM_BOT_CHAT_ID: "8990449281"
   ```

   Values must be strings (the `serviceProc.env` schema requires string values), so keep
   the chat id quoted. Add a short comment explaining why: the service host does not
   inherit AXE routine `env:`, so the receiver needs its own copy. It must stay in sync
   with `axe.routines.telegram.env` in `sase.yml`. Keep the token out of config. It is
   read from `~/.sase/telegram_bot_token`, which already works under the service host.

2. `home/dot_config/sase/sase.yml`: next to `axe.routines.telegram.env`, add a comment
   saying these two values are mirrored in `service.procs.telegram_receiver.env` in
   `sase_athena.yml` and must stay in sync. Do not change the routine's values. The
   outbound job still needs them.
3. Deploy on athena. Per the chezmoi repo gotcha, run `chezmoi update -a --force` after
   the chezmoi commit lands. The service host then relaunches `telegram_receiver` on its
   own, because `env` is part of the entry signature it compares on reconcile. No manual
   service restart is needed. If the commit can only land at finalization, say so
   explicitly in the final report so the deploy step is not forgotten.

## Part B: sase-telegram hardening

All paths below are relative to the sase-telegram repo root.

1. **`src/sase_telegram/credentials.py`**: make `get_chat_id()` raise
   `TelegramCredentialError`, which already subclasses `RuntimeError`, so existing
   `except RuntimeError` / `except Exception` callers keep working. Treat blank or
   whitespace-only values as unset. Make the message actionable: name
   `SASE_TELEGRAM_BOT_CHAT_ID` and say that the service-host receiver needs it under
   `service.procs.telegram_receiver.env`, because it does not inherit AXE routine env.
   Apply the same treatment to `get_bot_username()` for consistency.
   `scripts/__init__.py`'s `_run_cleanly` then turns a missing chat id in the
   outbound/`--once`/job-tick entry points into a one-line stderr message plus exit 1,
   not a traceback.

2. **Fail-closed authentication**, in `src/sase_telegram/scripts/sase_tg_inbound.py`:
   change `_update_is_from_configured_chat` to return `False` (reject) when
   `_configured_chat_id()` is `None`, and log a clear warning that updates are rejected
   because the chat id is not configured. Rewrite its docstring, which currently
   documents the fail-open behavior. The existing "Rejecting Telegram update from an
   unauthorized chat" path in `_dispatch_one_update` stays as is.

3. **Receiver preflight** (`_run_receiver`): after the existing enabled/token checks and
   **before the first `telegram_client.get_updates` call**, resolve the chat id. If it
   is unresolvable:
   - log an error naming the fix (`service.procs.telegram_receiver.env`);
   - upsert one SASE notification (sender `telegram`, a dedicated `dedup_key` such as
     `telegram-receiver-chat-id-missing`, tags `telegram`/`receiver`/`error`). Model it
     on `_notify_receiver_launch_failure` in `src/sase_telegram/receiver.py`, so a
     service-host restart loop cannot pile up duplicate notifications. The outbound job
     still has the chat id, so this notification also reaches Telegram.
   - return a **non-zero** exit code (use `78`, EX_CONFIG) without polling.

   Not polling is deliberate: Telegram keeps unfetched updates for about 24h, so
   messages sent while misconfigured are processed after the fix instead of being
   consumed and dropped. A non-zero exit makes the service host apply its
   `restart: on-failure` backoff and show the proc as failing, instead of a
   healthy-looking receiver. This differs on purpose from the existing clean `0` exit
   for disabled Telegram or a missing token. Keep that path unchanged.

4. **Legacy entry points**: in `_run_chop_tick`, resolve the chat id right after the
   existing `credentials.get_bot_token()` call and before `ensure_receiver_running()`. A
   legacy job tick without a chat id must not spawn a receiver that immediately exits 78
   on every 5-second tick. In `_run_once`, resolve it before polling so `--once` never
   consumes updates it would reject.

5. **No more silent drops**: in `_dispatch_fetched_updates`, when `_dispatch_one_update`
   raises for an update that carries a `message`, best-effort reply in that message's
   chat (`_message_chat_id(update.message)`) with a short one-line notice. It should say
   the message could not be processed, give the one-line error, and ask the user to
   resend. Catch and log any failure while sending that reply. Keep the existing
   per-update offset advance, log line, and callback behavior unchanged (callbacks
   already answer their own queries). This reply is safe because only authenticated
   updates reach dispatch after step 2.

6. **Docs and config comments** (do not edit `CHANGELOG.md`, which release-please
   manages):
   - `README.md`: in the Credentials section, state that when the receiver runs as the
     service-host `telegram_receiver` proc it does **not** inherit AXE routine `env:`.
     Show the `service.procs.telegram_receiver.env` YAML snippet with both variables.
     Update the authentication paragraph ("Updates from any chat other than the
     configured chat … are rejected") to say that an unconfigured chat id rejects
     everything and the receiver refuses to start.
   - `docs/inbound.md` (Long-Poll Receiver section): document the chat-id preflight, the
     exit code `78` with its notification, and the service-host env requirement.
   - `src/sase_telegram/default_config.yml`: add a comment on the `telegram_receiver`
     entry naming the two required env vars and where to set them.

### Tests (sase-telegram)

Add or adjust tests in the existing files: `tests/test_credentials.py`,
`tests/test_integration.py` (the foreign-chat rejection tests and the receiver tests
near `test_unsettled_start_refreshes_without_polling`), and `tests/test_inbound.py` as
appropriate:

- `get_chat_id()` raises `TelegramCredentialError` when the variable is unset, empty, or
  whitespace. It is still catchable as `RuntimeError`, and the message mentions
  `service.procs.telegram_receiver.env`.
- With no configured chat id, text, photo, document-image, and callback updates are all
  rejected before dispatch.
- `_run_receiver()` with Telegram enabled, a valid token, and no chat id: never calls
  `get_updates`, returns `78`, and a second invocation does not create a second
  notification.
- `_run_chop_tick` with no chat id never calls `ensure_receiver_running`. `--once` with
  no chat id never calls `get_updates`.
- A text-message handler that raises produces exactly one best-effort reply to the
  originating chat, and the offset still advances past that update. A failure while
  sending the reply is swallowed and logged.
- Fail-closed auth may break existing tests that silently relied on an unset chat id.
  Fix those tests by configuring a chat id (env or the existing `credentials` mocks). Do
  not weaken the fail-closed behavior.

Run `just check` in the sase-telegram repo (lint + tests) and make it pass.

## Verification on athena (after deploy)

1. `sase service status` shows `telegram_receiver` running with a new pid started after
   the config change.
2. `tr '\0' '\n' < /proc/<pid>/environ | grep SASE_TELEGRAM_BOT_` lists both variables.
3. `~/.sase/service/procs/telegram_receiver/output.log` shows a fresh "Starting Telegram
   long-poll receiver" and no chat-id errors.
4. Tell the user to resend the lost `research.24` prompt from Telegram, since update
   `264361664` cannot be replayed. Its launch confirmation proves end-to-end recovery.

## Out of scope (related findings; do not fix here)

- A durable inbound inbox (claim before offset advance, poison-update handling) is item
  3 of the existing ready bug task `sase-11w`. This plan only makes failures visible.
- The same log shows a separate, non-blocking warning:
  `Failed to resolve bead project 'sase' from launch_prompt`.
  `_extract_project_from_prompt` takes the raw `#gh:sase` ref and passes `sase` to
  `get_workspace_directory`. The registered project is `gh_sase-org__sase`, so the
  `/bead` chat project context ends up with an empty workspace. It did not block the
  launch. Capture it as a task bead (via `/sase_new_task`) only if you reproduce it
  independently.
- Making Telegram identity a first-class `telegram:` config key (removing the duplicated
  values) would need a `sase.schema.json` change in sase plus careful rollout ordering.
  It is not needed to fix this incident.
