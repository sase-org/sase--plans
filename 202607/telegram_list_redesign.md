---
create_time: 2026-07-09 03:03:28
status: done
prompt: 202607/prompts/telegram_list_redesign.md
tier: tale
---
# Redesign the `/list` Telegram command

## 1. Problem & goal

The Telegram bot's `/list` command shows currently-running SASE agents, but its output is thin compared to what the ACE
TUI **Agents** tab conveys. Today each agent is three terse lines (name + model + duration, then `project · ws# · PID`,
then a truncated prompt), grouped by status bucket.

**Goal:** make `/list` output _much_ nicer. At minimum it must surface all the information a user gets from (a) the
Agents-tab **row entries** and (b) the top-level **agent metadata panel**. Where it adds value without becoming verbose,
aim higher. The result must be **intuitive**, **reliable**, and **beautiful** on a phone.

Because a chat message is a linear, mobile, tap-friendly medium — not a dense navigable tree — the design does not
reproduce every TUI glyph literally. Instead it surfaces the _information_ those glyphs encode, using Telegram's own
affordances (emoji as semantic color, whitespace/alignment, bold hierarchy, expandable blockquotes, inline keyboards) to
stay compact while making everything reachable.

## 2. Current behavior (baseline)

Handler chain (in the `sase-telegram` plugin), for reference:

- `_handle_command` dispatches `list` → `_handle_list_command()` (no args used).
- `_handle_list_command()` calls `sase.agent.running.list_running_agents()`, groups via the **shared**
  `sase.integrations.agent_status_groups.group_agent_statuses` / `status_bucket_header`, and renders each agent with
  `_format_agent_list_block(agent)`.
- Output is HTML (`<b>`/`<i>` + `html.escape`), no inline buttons. Long messages are auto-split at the last newline
  before 4096 chars by `telegram_client._split_message` (only the last chunk keeps any `reply_markup`), with a silent
  fallback to plain text if the formatted send fails.

Current per-agent block (what a user sees now):

```
<b>3e</b>  gpt-5.5, 14m31s
sase · ws#11 · PID 3861904
<i>#gh:sase Can you help me rename the sdd repo to sase-sdd …</i>
```

Fields used today: `name, model, duration, project, workspace_num, pid, approve→"autonomous", prompt`. The data source
(`RunningAgentInfo`) already carries **unused** fields: `provider`, `started_at`, `duration_seconds`, `status`,
`artifacts_dir`.

**Two structural limits of today's `/list`:**

1. It only lists **root** agents that are RUNNING / STARTING / WAITING. Child agents (workflow steps, family members,
   retry attempts) and terminal agents (DONE / FAILED / STOPPED) never appear, and the actionable "needs your input"
   states the TUI shows (PLAN submitted, QUESTION) are not reachable through `RunningAgentInfo.status`.
2. It shows a small subset of the metadata the Agents tab exposes.

## 3. Target: what the Agents tab conveys (field inventory to match)

### 3a. Agents-tab **row entry** signals

Provider badge (🎭 claude · 🤖 codex · 🐼 qwen · 🐙 opencode · 🪐 agy); auto-approve (⚡, ⚡T tale, ⚡E epic); retry
badge (↻N) and retry/fallback annotation (↻N ▸ model); bead (◆); tag (#tag); display name; **status** (with per-status
meaning) plus WAITING countdown / until-time and RETRYING countdown; children fold summary (×N, +shown/−hidden);
agent-name annotation; embedded-workflow annotation; and a right-aligned suffix of **live activity**, **finish
timestamp**, a run/done/failed/ paused marker, **elapsed duration**, and a file-change mark (✏️). Lineage shown as retry
chains (`↳`), workflow-step children, and family members.

### 3b. Agent **metadata panel** fields

`Name`, `Bead`, `Retry chain`, `Step`/`ChangeSpec`(+cl#)/`Project`, `Workspace #`, `Workflow`, `Auto` (⚡
PLAN/TALE/EPIC), `Model` (PROVIDER(model) @ reasoning_effort), `Xprompts`, `VCS`, `PID`, `BUG`, `Wait` (dep names +
status badges + until/duration + remaining), `Retries` (count/max + per-attempt lines + `Fallback` model), `Activity`,
`Timestamps` (START/RUN/PLAN/FBACK/ QUEST/RETRY/CODE/EPIC/DONE), output variables, commits, deltas + artifacts, workflow
variables, context (memory reads / skill uses / opened workspaces), slow tool calls, `ERROR` + traceback.

### 3c. Status vocabulary & buckets (already shared)

Buckets and glyphs already live in `sase.agent.status_buckets` and are **already reused by the plugin** via
`sase.integrations.agent_status_groups` (pyvision-annotated): `▲ Stopped`, `✗ Failed`, `◐ Starting`, `▶ Running`,
`⏳ Waiting`, `✓ Done`. The full raw-status set (RUNNING, WAITING, STARTING, PLAN, QUESTION, ANSWERED, RETRYING,
PLAN/TALE APPROVED, WORKING PLAN/TALE, PLAN/TALE DONE, EPIC CREATED, STOPPED, FEEDBACK, DONE, FAILED, …) maps into those
six buckets. "Stopped" ▲ means _paused for you_ (PLAN / QUESTION) — the highest-signal state on mobile.

The underlying `AgentMetaWire` (Rust core → Python mirror, already crossing the binding) **already carries every field
above** — `reasoning_effort`, `vcs_provider`, `tag`, `bead_id`, `agent_family`/`agent_family_role`,
`plan`/`plan_approved`/`plan_action`, `wait_for`/`wait_until`/ `wait_duration`,
`retry_attempt`/`retry_of_timestamp`/`retried_as_timestamp`, `output_variables`, `parent_agent_name`, `changespec_name`,
`run_started_at`/`stopped_at`, etc. **No Rust change is needed**; the work is to _surface_ fields the core already
provides.

## 4. Design principles

1. **Mirror the TUI's mental model.** Two surfaces: an **overview list** (≈ row entries) and a **per-agent detail** view
   (≈ metadata panel). Reuse the TUI's status buckets, glyphs, and iconography so the two experiences feel like one
   product.
2. **Progressive disclosure = compact yet complete.** The overview stays scannable; verbose content (full prompt, full
   metadata) is one tap or one expandable blockquote away. "Complete" never means "everything on screen at once."
3. **Emoji as semantic color.** Telegram ignores text color, so meaning is carried by a small, consistent icon set drawn
   from the TUI (provider badges, ⚡ ◆ ↻ ✏️ and the bucket glyphs).
4. **Reliable rendering.** Escape all dynamic text; assemble on agent-block boundaries so chunking never splits a tag or
   blockquote; degrade gracefully when optional fields are missing; keep the existing plain-text fallback intact.
5. **One source of truth.** Field selection and human-facing derivations (lineage, wait ETA, family role, status
   refinement) are **shared backend** — a web/mobile/CLI frontend must match the TUI — so they live in the main `sase`
   repo (`sase.integrations`), reused by the plugin, the mobile summary, and `sase agent list`. The plugin stays a thin
   renderer. (Rust-core boundary: the raw fields already come from `sase-core`; only the Python projection/derivation is
   added here. If a reviewer prefers the derivation move into `sase-core`, that is a clean follow-up — called out as an
   open decision.)

## 5. Proposed output (before → after)

### 5a. Overview — `/list`

Header with live counts, then TUI status buckets, then a rich but compact block per agent. The prompt is a normal
(collapsible) blockquote; optional signals appear only when present.

```
🤖 Agents · 3 active
▶ 1 running · ⏳ 2 waiting

▶ Running (1)

🎭 3e · opus @ high · ▶ 14m31s ✏️
sase · ws#11 · PID 3861904 · GitHub
“Rename the sdd repo to sase-sdd (rename the repo with the `gh` command and rename the local dir…”

⏳ Waiting (2)

🤖 3e.f1 · gpt-5.5 · ⏳ on 3e · ~2m left
sase · ws#0 · PID 3932704 · fork of 3e · GitHub
“Migrate the bob-cli and actstat repos to a separate repo for sase sdd artifacts…”

🤖 3a.f1 · gpt-5.5 · ⏳ 2h
sase · ws#0 · PID 3731686 · fork of 3a · GitHub
“Try again.”

— ✓ 3 done · ✗ 1 failed in the last hour · /list all
```

Per-agent overview block, line by line:

- **Line 1 (identity + live state):** provider badge · **name** (bold, humanized) · `model @ effort` · status token. The
  status token adapts: running → `▶ <elapsed>`; waiting → `⏳ on <dep> · ~<eta>` or `⏳ <duration>` or `⏳ until HH:MM`;
  retrying → `↻N (<countdown>)`; stopped-for-input → `📋 plan ready` / `❓ needs answer`; done/failed (in `/list all`) →
  `✓ <finished>` / `✗ <finished>`. Trailing micro-badges when applicable: `✏️` file change, `⚡`/`⚡T`/`⚡E` autonomous,
  `◆` bead, `#tag`, `↻N ▸ <fallback>` fallback model, `×N` children fold.
- **Line 2 (context):** `project · ws#N · PID · VCS` plus lineage when present (`fork of <name>` /
  `family <base>·<role>` / `retry N`). Omit empty parts (as today).
- **Line 3 (activity, only if set):** the live `Activity:` string, italic.
- **Line 4 (prompt):** VCS-ref-humanized, single-line, in a `<blockquote>` so long prompts collapse.

The header counts and the `— ✓/✗ … · /list all` footer are only shown when relevant.

### 5b. Detail — `/list <name>`

A metadata-panel equivalent for one agent: an aligned key/value grid (rendered in a `<pre>` block for clean monospace
alignment on mobile), the full prompt in an **expandable** blockquote, and that agent's action buttons. Only non-empty
fields render.

```
🎭 3e — details
▶ Running · 14m31s

Model      opus @ high
Project    sase
Workspace  #11
PID        3861904
VCS        GitHub
Started    2026-07-09 02:33:26
Activity   writing tests
Wait       —
Retries    0 / 3
Outputs    plan_path=…
Artifacts  5 files · 2 commits

“full prompt, expandable…”     ← <blockquote expandable>
[🍴 Fork] [⏳ Wait] [🗡️ Kill] [🔄 Retry]
```

Fields map 1:1 to §3b (Name, Bead, Retry chain, ChangeSpec/Project, Workspace, Workflow, Auto, Model@effort, VCS, PID,
Wait, Retries+Fallback, Activity, Timestamps, Output vars, Commits, Deltas/Artifacts, Error). A single agent → single
message → single reliable keyboard.

### 5c. Interaction model

- `/list` — overview (active agents), grouped by bucket, with a compact **global** footer keyboard: `🔄 Refresh` (edits
  the message in place via callback), `🗡️ Kill…`, `🍴 Fork…` (reuse the existing per-agent selection keyboards), and
  `✓ Show finished` / `Hide finished` toggle.
- `/list all` — include DONE/FAILED (and STOPPED) via the existing `list_all_agents()`.
- `/list <name>` — the detail view (§5b) with per-agent `🍴 Fork · ⏳ Wait · 🗡️ Kill · 🔄 Retry` buttons, reusing the
  launch-notification button builder.
- `/list <project>` — filter the overview to one project (matches `sase agent list --project` / mobile filter
  semantics). Name vs. project is disambiguated by matching known agent names first.

This exactly mirrors the TUI split (scan the row list → open one agent's detail panel → act), which is what makes it
intuitive.

## 6. Data-source strategy (shared projection)

Add one **presentation-neutral, richer projection** in the main `sase` repo, reused by every frontend, keeping the
plugin thin:

- New shared module under `sase/integrations/` (sibling to `agent_status_groups.py` and `_mobile_agent_summary.py`)
  exposing an `AgentListEntry` dataclass + builder (e.g. `agent_list_entries(*, include_recent=False, project=None)`),
  plus a small typed **detail** projection for one agent. It composes the existing `list_running_agents()` /
  `list_all_agents()` and enriches each entry by reading the already-scanned markers (`agent_meta.json`,
  `pending_question.json`) — the same low-risk pattern `_mobile_agent_summary.retry_lineage` already uses — to add:
  `reasoning_effort`, `vcs_provider` (display name), `tag`, `bead_id`, `agent_family`/role,
  `plan`/`plan_approved`/`plan_action`, `wait_for`/`wait_until`/remaining, `retry_attempt`/fallback, `output_variables`,
  `parent_agent_name`, `changespec_name`(+cl#), timestamps, activity, and a cheap **children summary** (`×N` +
  per-status counts) where available.
- Expose a **shared provider-emoji accessor** (the badge map currently lives in `sase.ace.tui.provider_styles`; surface
  it from a non-TUI location so the plugin/CLI/mobile can reuse it without importing TUI internals).
- **Actionable "needs you" states:** derive `📋 plan ready` from plan-submitted markers and `❓ needs answer` from the
  authoritative `pending_question.json` marker, mapping into the existing **Stopped** bucket so these high-signal states
  finally reach `/list`. Reuse the constants in `status_buckets.py`; keep the derivation shared to avoid divergence from
  the TUI. (If robust status refinement proves involved, ship the **badge/indicator** first — `📋`/`❓` on the identity
  line driven by the authoritative markers — and treat full status-bucket reclassification as a fast-follow; the marker
  is authoritative, so the indicator is safe.)
- Annotate every new shared symbol the plugin imports with the
  `# pyvision: https://github.com/sase-org/sase-telegram.git` marker (matching existing shared helpers) so the
  cross-repo visibility linter and `sase validate` freshness gates stay green.
- **Bonus alignment (low cost):** point `sase agent list` and `_mobile_agent_summary` at the same projection so the CLI
  and mobile app inherit the richer fields for free and all frontends match.

The plugin keeps importing in-process from the main package (its established pattern) — no shelling out — so `/list`
stays fast and reliable.

## 7. Reliability

- **Escaping:** all dynamic text through `html.escape`; keep VCS-ref/name humanization (`display_cl_names_in_text`,
  `humanize_vcs_refs_in_text`).
- **Chunking:** assemble the overview as a list of **whole agent blocks** and pack them into ≤4096-char messages on
  block boundaries, so a tag/blockquote is never split (today's `_split_message` can cut mid-block). Prefer HTML
  `<blockquote expandable>` (no MarkdownV2 per-line prefix hazard that `formatting._wrap_expandable_blockquote` works
  around). Keep the existing plain-text fallback.
- **Missing/None fields:** every enriched field is optional; blocks render cleanly when data is absent (mirror today's
  conditional `details` assembly).
- **Empty / large lists:** keep the friendly empty message; when there are many agents, the block- boundary packing
  produces several tidy messages rather than one broken one. Consider a soft cap with an explicit "+N more · /list all"
  footer rather than silent truncation.
- **Buttons:** overview uses one **global** footer keyboard (last chunk only, which is fine for global actions);
  per-agent action buttons live in the single-message detail view, avoiding the "one keyboard can't address N agents"
  problem.

## 8. Implementation phases

**Phase 1 — Shared projection (main `sase` repo).** Add the `AgentListEntry` + detail projection and the shared
provider-emoji accessor in `sase/integrations/`; enrich from already-scanned markers; add the `📋`/`❓` actionable
indicators; pyvision-annotate shared symbols; unit tests for the projection (field mapping, lineage, wait ETA, children
summary, indicators, missing-field safety).

**Phase 2 — Overview renderer (`sase-telegram`).** Rewrite `_handle_list_command` to accept args (`all`, `<name>`,
`<project>`) and `_format_agent_list_block` to render the §5a block from the shared entries (provider badge,
`model @ effort`, adaptive status token, micro-badges, lineage line, activity, blockquote prompt, header counts +
finished footer). Add block-boundary-aware sending. Update/extend the existing `TestHandleListCommand` suite.

**Phase 3 — Detail view + actions (`sase-telegram`).** Add the `/list <name>` metadata panel (§5b) with the aligned
`<pre>` grid + expandable prompt, and reuse the launch-notification button builder for `🍴/⏳/🗡️/🔄`. Unify the two
near-duplicate formatters (`_format_agent_list_block` / `_format_agent_description`) onto the shared entry. Tests for
detail rendering + buttons.

**Phase 4 — Global keyboard & refresh (`sase-telegram`).** Add the overview footer keyboard (`🔄 Refresh` in-place edit,
`🗡️ Kill…`, `🍴 Fork…`, finished toggle) and their callback handlers (reuse existing kill/fork selection flows and
`callback_data` codec). Tests for callbacks.

**Phase 5 — Docs & polish.** Update the plugin README command table, the `set_my_commands` description for `list`
(mention `all` / `<name>` / `<project>`), and any help text. Run `just check` in **both** repos (via the linked-repo
workspace-open workflow for `sase-telegram`).

Phases 2–4 each deliver visible value independently; Phase 1 is their shared prerequisite. Scope can stop after Phase 2
and still fully satisfy the "match the row entries" ask, with Phase 3 satisfying "match the metadata panel."

## 9. Testing

- Main repo: projection unit tests (as in Phase 1) — deterministic fixtures for each status/lineage/ wait/indicator
  permutation and for missing-field safety.
- Plugin: extend `tests/test_inbound.py` `TestHandleListCommand` for HTML structure, escaping, bucket order, provider
  badges, adaptive status tokens, lineage, blockquote prompts, header/footer, the `all`/`<name>`/`<project>` arg paths,
  detail-view fields + buttons, and callback handlers. Assert the assembled message is valid (no split tag) at the chunk
  boundary.
- Both repos: `just install` then `just check`.

## 10. Risks & open decisions

- **Rust-core boundary:** the projection/derivation is added in Python `sase.integrations` (consistent with existing
  shared helpers `agent_status_groups` / `_mobile_agent_summary`). Open decision for the reviewer: keep it in Python, or
  push the derivation down into `sase-core` for the mobile/web frontends. No Rust change is required either way for this
  plan.
- **Status refinement vs. indicator:** deriving PLAN/QUESTION into the Stopped bucket must stay consistent with the TUI.
  Mitigation: ship the authoritative-marker **indicator** first; full bucket reclassification as fast-follow if the
  shared derivation is nontrivial.
- **Children/terminal breadth:** the overview stays root-focused with a `×N` children summary and a finished-tail footer
  to avoid verbosity; full child/terminal expansion is reachable via `/list all` and the detail view rather than shown
  inline.
- **Expandable-blockquote support:** requires Bot API ≥7.4 (already relied on elsewhere in the plugin); fall back to a
  plain blockquote if needed.

## 11. Out of scope

Changes to the ACE TUI itself; new Rust wire fields (none needed); non-`/list` Telegram commands except where they are
reused (`/kill`, `/fork` selection flows, launch-notification buttons); and any memory-file edits.
