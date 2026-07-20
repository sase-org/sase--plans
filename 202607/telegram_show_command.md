---
tier: tale
title: Telegram /show command for agents, clans, families, and tribes
goal: 'A single /show Telegram command resolves an agent, clan, family, or tribe reference
  and replies with a richly formatted, kind-specific status view that is intuitive
  to invoke from a phone, reliable under partial data, and visually consistent with
  the existing /list surface.

  '
create_time: 2026-07-20 16:40:59
status: done
prompt: 202607/prompts/telegram_show_command.md
---

# Plan: Telegram `/show` command for agents, clans, families, and tribes

## Context

All file changes in this plan land in the **sase-telegram linked repo** (open it with the `/sase_repo` skill:
`sase repo open sase-telegram`). No changes are needed in the sase repo or the Rust core: every data source is an
already-published `sase` API, and everything new here is Telegram presentation plus a thin composition of those existing
domain lookups (the same composition style `sase/scripts/agent_chat_from_name.py` uses for `#fork`).

Today the Telegram bot answers "what is running?" via `/list`, and `/list <name>` shows a detail card for one agent. But
SASE's kinship concepts — **clans** (named parallel containers, members `<clan>.<suffix>`), **families** (sequential
chains, members `<family>--<suffix>`), and **tribes** (cross-entity labels displayed as `@name`) — have no Telegram
surface at all. From a phone there is no way to ask "how far along is clan `review`?", "which phase of family `migrate`
is active?", or "what is tribe `@perf` doing overall?".

`/show <ref>` fills that gap with one intuitive command that accepts any kinship reference and renders the right view.

### Existing building blocks (all published `sase` APIs)

- `sase.integrations.agent_list_entries.agent_list_entries(include_recent=True)` — rich presentation-neutral
  `AgentListEntry` projections with `tribe` (already the _effective_ clan-aware tribe), `clan_tribe`, `agent_clan`,
  `agent_clan_generation`, `agent_family`, `agent_family_role`, `parent_agent_name`, status buckets, wait/retry info,
  children summaries, prompts, and activity. Already consumed by `/list`.
- `sase.agent.names.find_agent_clan(name)` — newest generation of a clan: `AgentClan(name, generation, members)` with
  per-member `outcome`, `artifacts_dir`, `timestamp`, archived completions, and `is_complete`.
- `sase.agent.names.find_agent_family(base)` — newest generation of a family: ordered `AgentFamilyMember`s with outcomes
  and artifact dirs.
- `sase.agent.names.find_named_agent(name)` — exact agent lookup across all projects.
- `sase.core.agent_tribe.parse_tribe_reference("@x")` / `validate_tribe_name` — `@tribe` parsing (raises
  `InvalidTribeError` on bad grammar; the handler must catch it).
- `sase.core.agent_clan_tribe.resolve_clan_tribe` / `resolve_clan_summary` — Rust-backed effective clan tribe/summary
  from member `agent_meta.json` declarations (mirror the `_resolve_clan_tribe` helper in
  `sase/scripts/agent_chat_from_name.py`, which shows the exact member-meta wiring).
- sase-telegram's own `/list` machinery in `src/sase_telegram/scripts/sase_tg_inbound.py`: `_format_agent_list_block`,
  `_format_status_token`, `_format_header_status_counts`, `_detail_rows`/`_format_detail_grid`, `_pack_html_blocks`,
  `_send_html_chunks`, `_build_agent_action_keyboard`, and the persisted-selection-key callback pattern used by `/kill`
  (`generate_key()` + `pending_actions.add`, see `_show_kill_selection`).

## Product design

### Command surface

- `/show <ref>` — resolve `<ref>` and render the matching view.
- `/show @<tribe>` — force tribe interpretation.
- `/show` (no args) — send a **kinship index**: a compact overview of the clans, families, and tribes that currently
  exist (from live + recent entries), each with a tap-to-open inline button, plus a one-line usage hint. This makes the
  command discoverable on mobile without memorizing names.
- Register `("show", "Show an agent, clan, family, or tribe")` in `_SLASH_COMMANDS` so Telegram's command menu lists it
  (the registration fingerprint re-registers automatically) and so the name is reserved against custom-command shadowing
  like the other built-ins.

### Resolution precedence (deterministic, documented in the command docs)

1. `@name` → tribe, always. Unknown tribe → not-found flow. Invalid grammar → friendly usage reply.
2. Exact agent name (`find_named_agent`, falling back to an exact `name` match over
   `agent_list_entries(include_recent=True)`) → **agent view**. Clan bare names are reserved-never-agents and family
   bare names are pure containers, so bare-name collisions with groups cannot occur by construction; clan members
   (`clan.suffix`) and family members (`family--suffix`) are plain agent names and correctly land here.
3. `find_agent_clan(name)` → **clan view**.
4. `find_agent_family(name)` → **family view**.
5. Bare name equal to a known tribe (casefold compare against effective `entry.tribe` values) → **tribe view**.
6. Otherwise → **not-found reply**: "No agent, clan, family, or tribe named `<ref>`" plus up to ~6 close-match
   suggestions (casefold/prefix/substring over known agent names, clan names, family bases, and tribes) rendered as
   tap-to-open buttons.

When a bare ref resolves to a non-tribe entity but the same name is also a known tribe, append a footer hint:
`Also a tribe — /show @<name>`.

### The four views (Telegram HTML, `/list` visual language)

All views use `parse_mode="HTML"`, escape every dynamic string, humanize names via `display_cl_name` /
`display_cl_names_in_text` / `display_project_name`, and split oversized output with the existing `_pack_html_blocks` +
`_send_html_chunks` chunking. Fixed kind glyphs keep the family of views recognizable: agents keep their provider badge,
clans use `⛺`, families use `🧬`, tribes render as `🏷️ @name`.

**Agent view** — the existing `/list <name>` detail card, enriched with kinship rows in the `<pre>` grid: `Clan`
(`⛺ name · gen <short>`), `Tribe` (`@name`), `Family` (`name · role`), `Parent` (fork parent), and `Children` (count +
status rollup) when present. Keyboard: the standard Fork/Wait/Kill/Retry controls plus context-jump buttons `⛺ Clan` /
`🧬 Family` when the agent belongs to one (callback re-renders the group view). `/list <name>` keeps its current
behavior and picks up the new kinship rows for free since both share the detail renderer.

**Clan view**

- Header: `⛺ <clan> — clan · @tribe · gen <short>` with a progress token: `N/M done`, or `✓ complete`.
- Effective clan summary paragraph (from `resolve_clan_summary` over member metas) when one is declared, as italic text
  under the header.
- Status rollup line reusing `_format_header_status_counts` over the clan's live entries.
- Member blocks: members present in `agent_list_entries(include_recent=True)` reuse `_format_agent_list_block` verbatim;
  members known only from artifacts (e.g. archived completions) render a compact fallback line `✓ <name> · done` from
  `AgentClanMember.outcome`.
- Keyboard: `🔄 Refresh`; `🍴 Fork clan` copy-text button with `#fork:<clan> ` (only when `clan.is_complete`, since clan
  forks require completeness); per-member drill-down buttons (capped, with an `…and N more` note when truncated).

**Family view**

- Header: `🧬 <family> — family · N members` with an overall progress token.
- The sequential chain in launch order, one line per member: `1. ✓ <family>--planner · done · 14m`, with the active
  member marked `▶` and non-success outcomes shown verbatim (mirrors the fork flow's excluded-member statuses). Role
  labels come from `agent_family_role` when the member has a live entry.
- The newest active member's activity line and prompt snippet as a `<blockquote>` so "what is it doing right now" is
  answered at a glance.
- Keyboard: `🔄 Refresh`; `🍴 Fork` copy-text `#fork:<family> `; member drill-down buttons.

**Tribe view**

- Header: `🏷️ @<tribe> — tribe` with entity and agent counts.
- Membership = entries whose effective `entry.tribe` equals the tribe. Grouped by entity kind: clans first
  (`⛺ <clan> · 3/4 done`), then families (`🧬 <family> · phase 2/3`), then standalone agents (one
  `_format_agent_list_block`-style compact line each: name · model · status token).
- Status rollup line across all member agents.
- Keyboard: `🔄 Refresh` plus drill-down buttons per entity (clans/families open their group views, agents open the
  agent view).

### Kinship index (bare `/show`)

One message: `🧭 <b>Agents by kinship</b>` header, then short sections for clans, families, and tribes that currently
have live or recent entries (name + member count + progress token per line). Buttons open each group (capped at ~24
buttons; truncation is stated explicitly, never silent). If nothing is grouped, reply with a friendly pointer to `/list`
and the `/show <ref>` usage line.

## Technical design

### New modules (pure logic, no Telegram API calls)

Follow the repo's established split (pure logic modules + IO-only scripts):

- `src/sase_telegram/agent_format.py` — extract the pure HTML formatting helpers that `/show` must share with `/list`
  out of `sase_tg_inbound.py` (~`_html`, `_entry_name`/`_entry_display_name`/`_entry_model_label`,
  `_format_compact_duration`, `_format_wait_token`/`_format_wait_until`/`_format_finished_time`, `_format_status_token`,
  `_entry_micro_badges`, `_format_agent_list_block`, `_entry_context_parts`, `_entry_prompt_snippet`,
  `_format_header_status_counts`, `_detail_rows` + grid helpers, `_pack_html_blocks`). The inbound script imports them
  back so `/list` behavior is unchanged. This is a mechanical move-and-reimport refactor — keep names and behavior
  identical so existing `/list` tests keep passing with only import-path updates.
- `src/sase_telegram/show_entities.py` — reference resolution: a small frozen dataclass result
  (`ShowTarget(kind, name, ...)` with kind-specific payloads for agent entry, `AgentClan`, `AgentFamily`, tribe members)
  plus `resolve_show_reference(ref, entries) -> ShowTarget | ShowNotFound`, the precedence rules, the also-a-tribe
  detection, close-match suggestion computation, and the kinship-index model (known clans/families/tribes with counts).
  Also the effective clan tribe/summary resolution over member metas (modeled on
  `agent_chat_from_name._resolve_clan_tribe`). All inputs (entries, clan/family lookups, meta readers) are passed in or
  injected so the module stays unit-testable without artifact fixtures.
- `src/sase_telegram/show_format.py` — pure renderers: `format_agent_show`, `format_clan_show`, `format_family_show`,
  `format_tribe_show`, `format_show_index`, `format_show_not_found`, each returning HTML block lists ready for
  `_pack_html_blocks`, plus keyboard _specs_ (label + action tuples) that the script converts into
  `InlineKeyboardButton`s. Builds on `agent_format` helpers.

### Wiring in `scripts/sase_tg_inbound.py`

- `_handle_show_command(args, message)` — trim args; empty → index; otherwise load
  `agent_list_entries(include_recent=True)`, resolve, render, send via `_send_html_chunks` with the built keyboard.
  Catch `InvalidTribeError` (usage reply) and unexpected exceptions (log via `log.exception`, send a short "Failed to
  build /show view" message) so one bad artifact never breaks the inbound tick.
- Dispatch: add `show` to `_handle_command`, accepting the command from text messages exactly like `/list`.
- Callbacks: new `show` action type in `_handle_callback`. Buttons carry `encode("show", <selection_key>, <choice>)`
  where `<selection_key>` is a `generate_key()` short key persisted via
  `pending_actions.add(f"show-{key}", {"action": "show", "ref": <full ref>})` — same pattern as the `/kill` selection
  keyboard, which keeps arbitrary-length names inside Telegram's 64-byte callback limit and inherits the 24h stale
  cleanup. Choices: `open` (render the stored ref as a fresh message) and `refresh` (re-resolve and edit the origin
  message in place when the rendered view fits one chunk, else send fresh chunks — mirror `_handle_list_callback`).
  Expired/unknown keys answer the callback with `Selection expired — run /show again`.
- Fork copy-text buttons reuse the existing `CopyTextButton` pattern and the `_COPY_TEXT_MAX` guard.
- Add `("show", ...)` to `_SLASH_COMMANDS`.

### Design decisions and rationale

- **sase-telegram only, existing APIs only.** Resolution primitives, effective-tribe projection, and clan tribe/summary
  resolution already live in `sase`/`sase-core` and are consumed by other frontends; nothing new crosses the
  core-backend boundary. This also avoids coupling the plugin to an unreleased `sase` version.
- **Agent > clan > family > tribe precedence** matches how references are already resolved for `#fork`
  (`agent_chat_from_name`), and name-grammar reservations make agent-vs-group collisions impossible, so the order is
  effectively collision-free except for tribes — which get the explicit `@` escape hatch and the footer hint.
- **HTML over MarkdownV2** matches `/list` and avoids MarkdownV2 escaping fragility inside dynamic names.
- **Persisted selection keys for all callbacks** — clan/family/agent names have no length bound, so raw names in
  callback data would be a latent 64-byte crash; short keys are the already-proven fix.

## Testing

Follow the existing test conventions (`tests/test_inbound.py` `TestHandleListCommand` style: patch `telegram_client`,
`credentials`, and `sase.integrations.agent_list_entries.agent_list_entries`; build entries with the `_list_entry`
helper).

- `tests/test_show_entities.py` — resolution unit tests: precedence order for every kind; `@tribe` forcing; invalid
  tribe grammar; clan/family member names resolving as agents; also-a-tribe hint detection; suggestion computation;
  kinship-index model; behavior when clan/family lookups return `None` and when entries are empty.
- `tests/test_show_format.py` — renderer unit tests: golden-ish assertions on each view's key lines (headers, progress
  tokens, rollups, member lines, archived-member fallback lines, blockquotes), HTML escaping of hostile names
  (`<alpha>&`), chunking of a large clan, keyboard specs (fork button only for complete clans, drill-down caps and the
  explicit truncation note).
- `tests/test_inbound.py` additions — handler-level tests: `/show` dispatch from `_handle_command`; bare `/show` index;
  `/show <agent>` sends the enriched detail with kinship rows and jump buttons; `/show <clan>`; callback `open` and
  `refresh` flows including the expired-key answer; error path sends the friendly failure message; `/list <name>` still
  renders (guarding the `agent_format` extraction).
- Update any existing tests whose imports move with the `agent_format` extraction.

## Documentation

- `docs/inbound.md` — add `/show` to the slash-command list with the resolution precedence, the four views, the bare
  `/show` index, and the callback behavior.
- `README.md` — add `/show` to the built-in command summary line and the inbound feature bullets.

## Risks and mitigations

- **Artifact scans on every invocation** (`find_agent_clan`/`find_agent_family` walk all projects): acceptable — the
  inbound chop already performs comparable scans for `/list` and launches; no caching needed initially.
- **Huge clans/tribes** overflow message limits: chunking handles text; button caps with explicit truncation notes
  handle keyboards.
- **Partially missing metadata** (no meta file, archived members, unnamed agents): every reader tolerates `None` and
  falls back to compact lines; renderers never raise on missing optional fields.
- **`/list` regression risk from the formatter extraction**: mitigated by a mechanical move (no behavior edits) and the
  existing `/list` test suite running against the new import layout.

## Non-goals

- No hood-prefix queries (`/show foo.` for arbitrary hoods) — the four requested kinds only.
- No new `sase` CLI subcommands, no sase/sase-core changes, no TUI changes.
- No mutation actions beyond the existing copy-text/kill/retry buttons already offered on agent cards.

## Completion checklist

- In the sase-telegram repo: `just install`, then `just check` (lint + tests) passes.
- Manual smoke via `sase_chop_tg_inbound --once` with a live clan/family if available.
