---
tier: epic
title: User-defined Telegram slash commands
goal: 'Custom Telegram slash commands can be declared in sase''s config and executed
  by the sase-telegram inbound chop, and a new /tasks command sends a polished PDF
  of the ~/bob/dash.md task queries with a descriptive caption message.

  '
phases:
- id: config-schema
  title: telegram.commands config surface in sase core
  depends_on: []
- id: command-engine
  title: Custom command dispatch and delivery in sase-telegram
  depends_on:
  - config-schema
- id: tasks-command
  title: /tasks report script and config in chezmoi
  depends_on:
  - command-engine
- id: smoke
  title: End-to-end smoke of /tasks delivery
  depends_on:
  - tasks-command
  model: haiku
create_time: 2026-07-16 15:29:35
status: done
bead_id: sase-6f
---

# Plan: User-defined Telegram slash commands (+ first use case: /tasks)

## Product context

The sase-telegram plugin currently supports a fixed set of built-in slash commands (`/kill`, `/list`, `/fork`,
`/changes`, `/xprompts`, `/bead`, `/update`), each hardcoded as an `elif` branch in `_handle_command` in
`src/sase_telegram/scripts/sase_tg_inbound.py`. There is no way for the user to add their own commands without editing
the plugin.

This epic adds **user-defined slash commands, declared in sase's config**. A custom command names an executable; when
the user sends `/name` in the Telegram chat, the inbound chop runs that executable and delivers its output back to the
chat — either as a formatted message or as a rendered PDF document with a caption. Custom commands are registered with
Telegram's `set_my_commands` API so they appear (with descriptions) in the Telegram "/" autocomplete menu, exactly like
the built-ins.

The first shipped use case is `/tasks`: a script in the chezmoi repo runs the Obsidian Tasks queries from the `dash.md`
note of the Bob vault (via `bob query --tasks-note dash.md --format json`), renders a beautiful tasks-dashboard markdown
report, and the framework converts it to a PDF and sends it with a summary caption (e.g. "📋 _Tasks Dashboard_ — 🔨 7
WIP · ⏭️ 6 NEXT · ✅ 23 READY").

## Architecture facts (verified during design)

- **The bot is not a daemon.** `sase_chop_tg_inbound` is a short-lived poll-and-exit chop run by the axe `telegram`
  lumberjack every few seconds. Slash-command dispatch is a manual `elif` chain in `_handle_command`; the
  `_SLASH_COMMANDS` list feeds `_register_commands_if_needed`, which calls `telegram_client.set_my_commands` and
  re-registers whenever a SHA-256 fingerprint of the command list changes.
- **The plugin has no structured config layer today** (env vars + `~/.sase/` dot-files only), but it already imports
  installed `sase.*` modules directly (`sase.notifications.store`, `sase.attachments.markdown_pdf`, `sase.core.paths`),
  so it can read sase's merged config through `sase.config.load_merged_config()`.
- **sase core's config is schema-driven end to end.** `src/sase/config/sase.schema.json` (root
  `additionalProperties: false`) is passed into the Rust `config_inventory`/`config_validate` bindings as data; the
  sase-core Rust repo's `config_parity.rs` uses synthetic fixture schemas only. Therefore adding a new top-level config
  section requires **only** editing the JSON schema + `src/sase/default_config.yml` in the sase repo — **no sase-core
  (Rust) changes**. This was explicitly verified against the Rust core boundary rule.
- **A PDF pipeline already exists.** `sase_telegram.pdf_convert.md_to_pdf(md_path)` renders markdown to a styled PDF
  (delegating to `sase.attachments.markdown_pdf` with `src/sase_telegram/pdf_style.css`), and
  `telegram_client.send_document(chat_id, path, caption=..., parse_mode=...)` sends it. `_handle_xprompts_command` is
  the existing "build PDF, send with caption" template to follow.
- **`bob query` can execute a note's Tasks blocks headlessly.** `bob query --tasks-note dash.md --format json` returns
  `{"blocks": [...]}` where each block has `heading` (e.g. "WIP Tasks"), `index`, and a result tree containing task
  records with clean fields: `descriptionWithoutTags`, `status.type`/`status.name`, `isBlocked`, `priority`,
  `due`/`scheduled`/`created` (`{raw, value}` or null), `file.path`, `blockId`, `urgency`. The frontmatter
  `TQ_extra_instructions` global filters (exclude templates/`#hide`, scheduled ≤ today, group by path, short mode) are
  applied automatically.

## Design

### Config surface (owned by sase core)

New top-level `telegram:` section (precedent for integration-specific top-level sections: `chat_install`,
`mobile_gateway`; core already hosts telegram-specific chop diagnostics in `sase/axe/chop_doctor.py`):

```yaml
telegram:
  commands:
    tasks:
      description: "📋 Obsidian tasks dashboard (WIP/NEXT/READY) as a PDF"
      run: tg_cmd_tasks
      output: pdf # "message" (default) | "pdf"
      timeout: 90s # optional; default 60s
```

Schema rules:

- Command name keys: `^[a-z0-9_]{1,32}$` (Telegram bot-command constraint), enforced via `propertyNames`.
- `description`: required string, 1–256 chars (Telegram menu limit).
- `run`: required non-empty string. Parsed with `shlex.split` (never a shell); the first token may be a bare name
  resolved on `PATH` or an absolute/`~` path; remaining tokens are fixed arguments.
- `output`: enum `message` | `pdf`, default `message`.
- `timeout`: duration string matching the axe `chop_timeout` pattern `^\d+(s|m|h)$`, default `60s`.
- `additionalProperties: false` on the section, the `commands` map values, and the `telegram` object.
- `default_config.yml` gains `telegram: { commands: {} }` so the section is always present in merged config.

### Script contract (the heart of the feature — keep it trivial for script authors)

- The framework runs the command's argv (no shell) with a per-run temporary working directory, captures stdout/stderr,
  and enforces the timeout (kill + report on expiry).
- Anything the user typed after the command (`/tasks foo bar`) is appended as **one** trailing argv element (raw
  remainder — no shell splitting of untrusted chat text) and also exported as `SASE_TELEGRAM_COMMAND_ARGS`.
  `SASE_TELEGRAM_COMMAND` is set to the command name.
- **stdout is markdown**, optionally starting with a YAML frontmatter block:

  ```markdown
  ---
  caption: "📋 *Tasks Dashboard* — 🔨 7 WIP · ⏭️ 6 NEXT · ✅ 23 READY"
  filename: tasks_dashboard_2026-07-16.pdf
  ---

  # 📋 Tasks Dashboard

  ...
  ```

  - `caption` (optional): markdown for the Telegram document caption. Default when absent: "📄 " + the command's
    configured `description`. Captions are converted with the existing formatting helpers and truncated safely to
    Telegram's 1024-char caption limit (ellipsis).
  - `filename` (optional): attachment filename; default `<command>_<YYYY-MM-DD>.pdf`.

- Delivery by configured `output` mode:
  - `message`: body → `formatting.markdown_to_telegram_v2` → `send_message` (existing auto-split handles >4096 chars;
    existing parse-mode fallback handles conversion edge cases). Frontmatter `caption`/`filename` are ignored.
  - `pdf`: send an immediate ack ("⏳ Running `/tasks`…", following the `_handle_xprompts_command` convention), write
    the body to `<tmpdir>/<stem>.md`, convert via `pdf_convert.md_to_pdf`, and `send_document` with the caption. **If
    PDF conversion fails, fall back to sending the markdown body as a message** prefixed with a one-line warning — the
    user always gets their content.
- Failure UX (honest and bounded):
  - Non-zero exit → "⚠️ `/tasks` failed (exit N)" plus a bounded stderr tail (~1000 chars) in an expandable blockquote.
  - Timeout → "⏱ `/tasks` timed out after 90s".
  - Empty stdout on success → "🫙 `/tasks` produced no output."
- Reserved names: the built-in command names (including the `beads` alias) always win. A custom command shadowing a
  built-in is skipped with a warning log and excluded from registration. Unknown commands stay silently ignored
  (unchanged behavior).
- No new trust surface: only the already-gated chat's updates reach `_handle_command`; chat text is passed as a single
  argv element, never through a shell.

### Registration & config loading

- `_register_commands_if_needed` registers `_SLASH_COMMANDS + sorted custom (name, description) pairs`. The existing
  fingerprint mechanism then re-registers automatically whenever the user adds/renames/removes a custom command, so
  commands appear in the Telegram "/" menu within one registration cycle.
- Config is read once per inbound run via `sase.config.load_merged_config()` with
  `sase.config.core.set_include_local_config(False)` so chop behavior never depends on the process cwd. Invalid entries
  (bad name, missing keys, bad timeout) are skipped with a warning log — one bad entry must never break the inbound chop
  or the other commands.

### Doctor check (reliability)

Add a small diagnostics check (pattern: `sase/doctor/checks_integrations.py`, which already hosts mobile-gateway checks)
that iterates `telegram.commands` in merged config and reports, per command, whether the `run` executable resolves (PATH
lookup / expanded path exists + executable bit). Schema-level shape problems are already covered by the generic config
validation; this check covers the runtime concern schema cannot see.

## Phases

### Phase `config-schema` — sase repo

1. Add the `telegram` section to `src/sase/config/sase.schema.json` per the rules above (descriptions on every field —
   they surface in the ACE Config Center).
2. Add `telegram: { commands: {} }` to `src/sase/default_config.yml`.
3. Add the telegram-commands doctor check to `src/sase/doctor/checks_integrations.py` (wired wherever the mobile-gateway
   checks are aggregated).
4. Tests, following `tests/test_config_schema.py` conventions: default config still validates; a representative valid
   `telegram.commands` document is accepted; invalid command names, missing `description`/`run`, bad `output`, and bad
   `timeout` values are rejected. Doctor check unit tests for resolvable/unresolvable `run` values.
5. Run the repo's standard checks before finishing.

### Phase `command-engine` — sase-telegram repo (open it with the `/sase_repo` skill)

1. New module `src/sase_telegram/custom_commands.py` (pure logic, following this repo's inbound.py/scripts split):
   - `CustomCommand` frozen dataclass (`name`, `description`, `argv`, `output`, `timeout_seconds`).
   - `load_custom_commands() -> dict[str, CustomCommand]` — reads merged sase config (local layer disabled),
     validates/normalizes entries, drops invalid or reserved names with warning logs.
   - `run_custom_command(command, args_text) -> CommandResult` — no-shell subprocess execution per the contract (temp
     cwd, env vars, timeout, captured output).
   - Frontmatter parsing of stdout (`caption`, `filename`) tolerant of absent/malformed frontmatter.
2. Dispatch: in `scripts/sase_tg_inbound.py::_handle_command`, add a final fallback branch that looks up custom commands
   and calls a new `_handle_custom_command(...)` implementing the delivery/error UX above (modeled on
   `_handle_xprompts_command` for the PDF path; `formatting.markdown_to_telegram_v2` + `telegram_client.send_message`
   for the message path; `pdf_convert.md_to_pdf` + `telegram_client.send_document` for documents).
3. Registration: extend `_register_commands_if_needed` to append custom `(name, description)` pairs; the fingerprint
   covers change detection with no extra state.
4. Tests per repo conventions (`tests/test_inbound.py` patterns: patch handlers to assert routing; SimpleNamespace
   messages; monkeypatched state paths): config parsing (valid/invalid/reserved), dispatch routing to
   `_handle_custom_command`, registration merge + fingerprint change, execution success/failure/timeout via a tiny fake
   script, frontmatter parsing, PDF-failure fallback to message, caption truncation.
5. Update `README.md` command list and `docs/inbound.md` dispatch description to document custom commands and the script
   contract (this contract doc is what makes the feature intuitive — write it as user-facing documentation).

### Phase `tasks-command` — chezmoi repo (open it with the `/sase_repo` skill)

1. New script `home/bin/executable_tg_cmd_tasks` (deploys to `~/bin/tg_cmd_tasks`) — Python 3, stdlib only,
   `#!/usr/bin/env python3`, following the repo's Python script conventions (docstring + argparse, flake8-clean):
   - Resolve `bob` via PATH with `~/.cargo/bin/bob` fallback (same convention as `executable_bob_sync`).
   - Run `bob query --tasks-note dash.md --format json` (note path overridable via `--note`, default `dash.md`).
   - Render a **beautiful** markdown report:
     - `# 📋 Tasks Dashboard` + human date line (e.g. "Wednesday, July 16, 2026").
     - A summary line mirroring dash.md's chip widget: `🔨 7 WIP · ⏭️ 6 NEXT · ✅ 23 READY`.
     - One `##` section per query block, in note order, with emoji + count (🔨 WIP Tasks / ⏭️ NEXT Tasks / ✅ READY
       Tasks); `###` subsections per source note (grouped by `file.path` stem, matching the dashboard's group-by-path);
       tasks as list items using `descriptionWithoutTags` cleaned of `#task` and `^anchor` artifacts, with compact
       badges for due/scheduled dates and non-normal priority.
     - Graceful empty state ("🎉 No open tasks!") and a small generated-at footer.
   - Emit frontmatter `caption` (the chips summary) and `filename` (`tasks_dashboard_<YYYY-MM-DD>.pdf`), then the body.
   - Fail with a clear stderr message (non-zero exit) when `bob` is missing or the query fails — the framework relays it
     to the chat.
2. Config in `home/dot_config/sase/sase_athena.yml` (athena hosts the bot and the Bob vault):

   ```yaml
   telegram:
     commands:
       tasks:
         description: "📋 Obsidian tasks dashboard (WIP/NEXT/READY) as a PDF"
         run: tg_cmd_tasks
         output: pdf
         timeout: 90s
   ```

3. Follow the chezmoi repo's CLAUDE.md: after committing, run `chezmoi update -a --force` so the script and config
   deploy to the live home directory.
4. Local verification: run the deployed script directly and eyeball the markdown; run it through `sase doctor` / the new
   telegram-commands check to confirm the command resolves.

### Phase `smoke` — end-to-end verification

1. `sase doctor` (or the specific integrations check) reports the `tasks` command as resolvable on athena.
2. Exercise the full delivery path without waiting for a live chat message: run `tg_cmd_tasks`, feed its stdout through
   the plugin's frontmatter parsing and `md_to_pdf`, verify a non-empty styled PDF is produced, then send it to the
   configured chat via `telegram_client.send_document` with the parsed caption — this both proves the pipeline on the
   real host and shows the user the finished product in Telegram.
3. Verify registration: run the inbound chop's registration step and confirm the registered command list now includes
   `tasks` with its description (fingerprint file updated).
4. Report results, including a note asking the user to send `/tasks` in the chat for the final interactive confirmation.

## Testing strategy

- sase: schema accept/reject unit tests + doctor check tests (phase `config-schema`).
- sase-telegram: unit tests for config parsing, routing, registration, execution (success/error/timeout), frontmatter,
  and PDF fallback (phase `command-engine`).
- chezmoi: direct script runs against the live vault; lint per repo tooling (phase `tasks-command`).
- End-to-end: real PDF built and delivered to the chat on athena (phase `smoke`).

## Risks & edge cases

- **`bob query` JSON shape drift**: the renderer must parse defensively (tolerate missing/null date fields, unknown
  status types) and pin only to the fields listed above.
- **Telegram limits**: 1024-char captions (truncate), 4096-char messages (existing auto-split), command descriptions
  ≤256 chars (schema).
- **PDF toolchain**: pandoc/wkhtmltopdf availability on athena is already exercised by the launch-preview flow; the
  message fallback covers transient conversion failures.
- **Config errors must never brick the chop**: every custom-command config problem degrades to a logged skip.
- **Latency**: the telegram lumberjack ticks every few seconds; a `pdf` command posts an ack immediately, so the user
  always gets fast feedback even when the script takes tens of seconds.

## Out of scope (deliberate)

- No sase-core (Rust) changes — verified unnecessary (schema-driven config machinery).
- No new attachment types (photos, videos, arbitrary files) — the contract's frontmatter leaves room to add an
  `attachments:` key later.
- No per-command auth/multi-chat support — the bot remains single-chat, gated exactly as today.
- No changes to built-in commands or to memory files / agent instruction shims in any repo.
