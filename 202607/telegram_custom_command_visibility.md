---
tier: tale
title: Keep configured Telegram commands visible
goal: 'Newly configured Telegram slash commands appear near the top of the bot menu,
  and failed command registrations are retried instead of being cached as successful.

  '
create_time: 2026-07-16 16:53:08
status: done
prompt: 202607/prompts/telegram_custom_command_visibility.md
---

# Plan: Keep configured Telegram commands visible

## Context

The `/tasks` feature is deployed end to end: the merged SASE configuration contains a valid `telegram.commands.tasks`
entry, `tg_cmd_tasks` resolves on `PATH`, the installed `sase-telegram` package loads the command, and Telegram's
server-side default command list contains all eight expected commands. The screenshot nevertheless shows only the seven
built-in commands because configured commands are appended after every built-in and Telegram's mobile command sheet
initially exposes a limited viewport. `/tasks` is therefore registered but sits just below the visible fold.

The registration path has a related reliability gap: it writes the hourly fingerprint cache whenever `set_my_commands`
returns normally, even if Telegram returns `False`. A rejected registration can consequently be treated as current for
an hour.

## Implementation

Make the focused change in the `sase-telegram` repository.

1. Change the command-list composition in `src/sase_telegram/scripts/sase_tg_inbound.py` so valid configured commands
   are registered first in deterministic name order, followed by the stable built-in list. This keeps personalized
   commands such as `/tasks` in the initially visible portion of Telegram's menu without changing dispatch precedence:
   built-ins and their aliases must remain reserved and must continue to win during message handling.
2. Keep the existing fingerprint-based invalidation tied to the exact ordered payload. The ordering change should
   naturally invalidate the deployed cache and trigger immediate re-registration on the next inbound poll, with no
   manual cache deletion.
3. Treat a false result from `telegram_client.set_my_commands` as a failed registration. Log it and leave the cache
   untouched so the next poll retries; only persist the timestamp and fingerprint after an affirmative result. Preserve
   the current exception retry behavior.
4. Update the custom-command documentation to state that configured commands are placed before built-ins for menu
   discoverability and that Telegram clients may present longer command lists in a scrollable sheet.

## Tests and validation

- Update registration tests in `tests/test_inbound.py` to assert configured-first deterministic ordering, built-in
  retention, and the fingerprint written for the exact submitted order.
- Add regression coverage showing that `False` and exceptions do not create or refresh the registration cache, while
  `True` does.
- Retain coverage that a changed ordered fingerprint forces registration before the hourly interval expires and that
  built-in dispatch cannot be shadowed by configured commands.
- Run the focused inbound/custom-command tests, then the repository's full `just check` validation.

## Deployment smoke check

After installing the updated `sase-telegram` package and restarting axe, verify that SASE doctor still resolves
`tg_cmd_tasks`, the registration cache fingerprint matches the configured-first payload, Telegram's `getMyCommands`
result begins with `/tasks`, and the mobile command sheet exposes `/tasks` without scrolling. Invoke `/tasks` once to
confirm it still returns the expected PDF; do not alter the existing host configuration or dashboard helper unless that
independent execution check fails.

## Scope

No SASE configuration-schema, chezmoi command definition, dashboard rendering, or core dispatch change is required.
Those layers already work and should remain unchanged unless implementation-time evidence contradicts the verified
diagnosis.
