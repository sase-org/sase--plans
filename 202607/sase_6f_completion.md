---
tier: tale
title: Complete and land user-defined Telegram slash commands
goal: 'The missing live Telegram delivery acceptance test is completed with durable
  evidence, then epic sase-6f is closed, its expired Symvision allowances are cleaned
  up, and its original plan is marked done.

  '
create_time: 2026-07-16 16:52:17
status: wip
prompt: 202607/prompts/sase_6f_completion.md
---

# Plan: Complete and land user-defined Telegram slash commands

## Context

Epic `sase-6f` implemented user-defined Telegram slash commands across the `sase`, `sase-telegram`, and `chezmoi`
repositories. Its four phase beads are closed and the implementation has been audited against the linked plan and the
phase commits:

- `0333dcf68` adds the closed `telegram.commands` schema, defaults, doctor check, and tests in `sase`.
- `e2527d0` adds no-shell custom-command loading, execution, dispatch, registration, delivery, failure handling, tests,
  and documentation in `sase-telegram`.
- `1258cd96` adds the defensive `tg_cmd_tasks` report renderer and athena configuration in `chezmoi`.

Current-head verification passes: the focused `sase` schema/doctor suite has 77 passing tests; `sase-telegram` passes
Ruff, mypy, and all 462 tests when installed against the linked current `sase`/`sase-core`; the deployed
`~/bin/tg_cmd_tasks` is executable and byte-identical to the chezmoi source; the doctor reports the command executable
as resolved; and a fresh read-only run produces valid Markdown and a non-empty PDF. Later commits on the direct `master`
histories were reviewed. The only overlapping later commit removed obsolete agent-family config and preserved the
Telegram schema, defaults, and tests; other later commits are unrelated.

The remaining gap is in phase `sase-6f.4`. Its saved transcript says the full pipeline was verified "without live chat",
but the approved epic plan requires calling `telegram_client.send_document` for the configured chat with the parsed
caption. The original smoke bead also has no durable NOTES evidence.

## Phase 1: Live delivery

Open `sase-telegram` and `chezmoi` with `sase repo open` before reading or running them. Follow the required audited
Obsidian memory-read procedure before querying the Bob vault.

1. Re-run the focused `integrations.telegram_commands` doctor check and require an OK result for
   `tasks: tg_cmd_tasks (resolved)`.
2. Load `tasks` through `load_custom_commands()` and exercise the real custom command delivery path against the
   configured Telegram chat. The operation must execute `tg_cmd_tasks`, parse its frontmatter, render a non-empty PDF,
   and successfully call `telegram_client.send_document` with the parsed, safely formatted caption and requested
   filename. Using `_handle_custom_command` is preferred because it exercises the same ack, execution, conversion, and
   delivery code used by inbound dispatch.
3. Confirm the registered command set and cached fingerprint contain `tasks` with its configured description. Do not
   expose bot credentials or private task text in logs or notes.
4. Add concise durable notes to closed bead `sase-6f.4` recording the successful live send, doctor result, non-zero PDF
   size, registration confirmation, and timestamp. If the send fails, leave the parent epic open and diagnose the
   failure instead of proceeding to landing.

## Phase 2: Land

This phase is deliberately last and depends on successful live delivery.

1. Recheck `sase bead show sase-6f` and every child, then close the parent with `sase bead close sase-6f`.
2. After the close, read `symvision.md` through `sase memory read` with a specific reason, then run `just symvision` if
   that recipe is available. Remove stale `sase-6f` epic-symbol whitelist entries and any unused code the post-close run
   reports. Run `just install` before repository checks and run `just check` if source/config files change, as required
   by the repository.
3. Set `status: done` in the YAML frontmatter of the original epic plan at
   `${SASE_SDD_PLANS_DIR}/202607/telegram_custom_commands.md`.
4. Verify the parent bead is closed, all children remain closed, the original plan reports `status: done`, Symvision is
   clean, and every touched repository has only the intended landing changes.

## Safety and acceptance

- The live Telegram send is the only external message authorized by this completion plan; do not send duplicate
  dashboards after one successful send.
- Never print or persist Telegram credentials or the private body of the task report.
- Do not close `sase-6f` until the real `send_document` call succeeds.
- Do not edit memory files or generated provider instruction shims.
- Completion requires both the live-delivery evidence and the ordered landing sequence: close, post-close Symvision
  cleanup, then original-plan status.
