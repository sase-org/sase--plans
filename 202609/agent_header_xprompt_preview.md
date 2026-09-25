---
tier: epic
title: AGENT XPROMPT preview in the sticky agent header
goal: 'On the Agents tab, the selected agent''s AGENT XPROMPT moves out of the data
  deck''s Context card and into the sticky header panel above the deck. While collapsed,
  the header shows a dense, syntax-highlighted preview of the prompt. The preview
  fills the full panel width and as many rows as a height budget derived from the
  detail column allows. `d` expands the header to the complete xprompt alongside the
  full identity fields. Header and body always come from the same document, j/k never
  flickers on agents you have already visited, and the collapsed header no longer
  wastes a blank row.

  '
phases:
- id: preview
  title: Pure xprompt preview fitting and header settings
  depends_on: []
  size: medium
  description: 'preview: add a pure module that reflows the highlighted xprompt, fits
    it to a width and row budget behind a quote bar, and reports hidden lines; add
    the row-budget helper and the inert ace.agent_header.collapsed_max_share setting
    (schema, default config, parser, app wiring) with unit tests.'
- id: document
  title: XPROMPT section travels with the detached identity
  depends_on: []
  size: medium
  description: 'document: extend IdentityHeader with the xprompt, attach it instead
    of rendering the body section in the agent, family, and both hint paths when xprompt
    detachment is on (off by default), give the cheap j/k path a bounded memo plus
    a pending flag, and update the inline renderable and hint-cache key, with tests.'
- id: panel
  title: Header panel preview, expansion, layout, docs, and visual verification
  depends_on:
  - preview
  - document
  size: medium
  description: 'panel: turn on xprompt detachment, render the collapsed preview and
    expanded XPROMPT section in AgentHeaderPanel with column-derived budgets, an overflow
    subtitle, a pending-height hold, and pin reapply; remove the phantom header/footer
    row; update docs and goldens; and verify with live screenshots.'
proposed_by: bbugyi200.athena.0rk
create_time: 2026-09-24 17:41:26
status: done
bead_id: sase-18g
---

- **BEAD:** [sase-18g](https://github.com/sase-org/sase--beads/blob/main/pages/sase-18g/README.md)

# Plan: AGENT XPROMPT preview in the sticky agent header

## Context

The Agents tab's right-hand detail column (`AgentDetail`, in
`src/sase/ace/tui/widgets/agent_detail.py`) stacks three widgets:

1. The sticky `AgentHeaderPanel` (`src/sase/ace/tui/widgets/agent_header_panel.py`).
2. The agent data deck (`DeckArea`, `src/sase/ace/tui/widgets/decks/`).
3. The sticky `AgentJumpPanel` footer.

The header shows the selected node's detached `IdentityHeader`
(`src/sase/ace/tui/widgets/prompt_panel/_identity_header.py`). Epic `sase-16k` split it
out of the metadata document. Collapsed, it shows two chip rows built by
`build_agent_compact_lines` (`_identity_header_compact.py`). Expanded with `d`, it shows
the full field list. The document builders attach the identity to the body they return
(an `AgentHeaderRenderable` carrier). `AgentPromptPanel.update()` publishes it to the
header sink in the same call as the body update, so header and body can never disagree.

The `AGENT XPROMPT` section (the raw prompt the user typed, humanized and
syntax-highlighted) still renders inside the deck's Context card. It appears after
`SASE CONTEXT` and before `AGENT PROMPT`, so it scrolls away with the body. It is
rendered at four sites, each reading `agent.get_raw_xprompt_content()` on the debounced
full-paint path:

- The standard agent document: `_update_display_impl` in `_agent_display_render.py`.
- The family container document: `_update_family_display` in
  `_agent_display_family_render.py`. The same function also handles family hint mode
  through `hint_state`.
- The standard hint document: `render_agent_prompt_hint_body` in
  `_agent_display_hint_body.py`, called from `_agent_display_hint_render.py`.

`update_header_only` (`_agent_display.py`) is the immediate j/k paint. It must not touch
artifact files (`sase/memory/tui_perf.md` rules 1, 7, and 8), so today it never has the
xprompt.

A live probe also showed that the collapsed header is really three rows tall, not two.
With `height: auto`, Textual 8.0.1 reserves an extra row for a horizontal scrollbar when
`scrollbar-gutter: stable` is set. The content region is 2 rows, the panel is 5 rows,
and `virtual_size.height` is 3. Adding `scrollbar-size-horizontal: 0` brings the panel
down to 4 rows, and the vertical gutter and vertical scrolling still work (checked with
a 40-row overflow). `#agent-jump-panel` has the same rule and the same wasted row.

Everything here is presentation-only Textual/Rich code, so it stays in this repo, with
no `sase-core` change.

## Design

### What the user sees

A long, hard-wrapped prose prompt (collapsed, the default):

```
╭─ AGENT SHELL ──────────────────────────────────────────────────────────╮
│  0rk · opus@xhigh ← @large · ⚡ PLAN                                     │
│  ▣ #plan · ⏳ sase-18f.1                                                 │
│  ▎ Can you help me start rendering the `AGENT XPROMPT` section in the   │
│  ▎ sticky header above the agent data deck panel? Make sure that we     │
│  ▎ provide a good preview (use as much space as is available) of the    │
│  ▎ contents in this section (i.e of the user's prompt) in this…         │
╰────────────────────────────────────────────── +4 lines · ▾ d more ─╯
```

A short, directive-heavy xprompt: seven source lines reflow into two rows, and nothing
is hidden.

```
╭─ AGENT SHELL ──────────────────────────────────────────────────────────╮
│  sase-18f.3 · opus@medium · ⚡ PLAN                                      │
│  ⌘ #gh · ▣ #bd/work_phase_bead · ⏳ sase-18f.1                           │
│  ▎ +sase %id(3, clan=sase-18f, bead=sase-18f.3) %model:@medium %auto    │
│  ▎ %w:sase-18f.1 %w(bead=sase-18f.1) #bd/work_phase_bead:sase-18f.3     │
╰──────────────────────────────────────────────────────────── ▾ d more ─╯
```

Expanded (`d`): the full identity field list, then one blank line, then the navigable
`AGENT XPROMPT` title in the body section-heading style, then the complete highlighted
xprompt in its natural line layout. Height stays `auto` with `max-height: 50%`, and the
panel scrolls with the mouse wheel as it does today.

The collapsed header is designed like this:

- **Rows 1–2 do not change.** The who/how and what/state chip rows keep their position,
  so the eye anchor never moves. This includes the `▣ #name` xprompt chips: they also
  list xprompts that do not appear literally in the raw text, such as project-tag
  expansions.
- **Quote bar.** Every preview row starts with `▎ `, the bar in the XPROMPT accent
  `#AF87FF` (the color clan summaries already use for AGENT XPROMPT entries). It reads
  as "what you asked", groups the rows visually, and costs no divider row. Fira Code has
  the `▎` glyph; do not use `❯`, which already means "cursor" and "bash step".
- **The existing highlighting is kept.** The preview is made from the same highlighted,
  humanized `Text` the body renders today, so directives, `#xprompt` references,
  mentions, and code spans keep their colors.
- **Markdown-style reflow.** Consecutive lines in a paragraph join with one space. A
  hard break becomes a dim `¶` (Fira Code has `¶`). Hard breaks are:
  - blank lines, with runs collapsed into one mark
  - a line that starts a block construct: `- `, `* `, `+ ` (the sigil must be followed
    by a space, so `+sase` and `#gh` are not block starts), `1. ` or `1) `, `#` through
    `######` followed by a space, `> `, `|`, or a ``` fence
  - the boundary after a fence or thematic-break line (`---`, `***`, `___`)
  - every line boundary inside a fenced block

  Tabs expand, and leading indentation, trailing whitespace, and leading or trailing
  blank lines are dropped. Hard-wrapped prose (typical for editor-authored prompts) then
  reads naturally, and directive stacks pack densely.

- **Full width, budgeted height.** Rows wrap at the content width minus the 2-cell bar
  gutter. Words wrap normally, and a token too long for one row is folded. The preview
  shows `min(rows needed, budget)` rows, so a short prompt never leaves padding rows. On
  overflow, the last row ends in `…` right after the last whole word that fits. The
  border subtitle then becomes `+N lines · ▾ d more`, where N counts the source lines
  not fully visible. Without overflow, the subtitle stays `▾ d more` (`▴ d less` when
  expanded), with the key taken from the live keymap as today.
- **Budget ("as much space as is available").** New config
  `ace.agent_header.collapsed_max_share` (number from 0 to 0.6, default `0.35`) caps the
  collapsed header's total height at `floor(column_rows × share)`. `column_rows` is the
  `AgentDetail` height. The preview gets that cap minus 4 rows (2 border rows and 2 chip
  rows), but at least 1 row whenever share > 0. For example, 120×40 (about 32 column
  rows) gives up to 7 preview rows, 200×55 (about 47) gives up to 12, and `0` turns the
  preview off: the header keeps its two chip rows, and the XPROMPT is only shown when
  expanded. By default, the collapsed header never takes more than about a third of the
  column.
- **No blank row.** `scrollbar-size-horizontal: 0` on `#agent-header-panel` and
  `#agent-jump-panel` removes the phantom row. A node without an xprompt shows exactly
  the two chip rows inside the border.

### What moves and what stays

- The Context card no longer contains `AGENT XPROMPT` or its trailing light divider. It
  goes from `SASE CONTEXT` (if present) straight to `AGENT PROMPT`, the fully expanded
  prompt. The words the user typed therefore stay searchable with `,/` through
  `AGENT PROMPT`, and `Ctrl+J`/`Ctrl+K` lose only the `AGENT XPROMPT` stop.
- These are unchanged:
  - the `V` metadata pager, which uses a separate builder
  - clan and tribe summary documents and their per-member XPROMPT entries
  - the run log modal
  - any `AgentPromptPanel` without a sink (tests and non-detached builders)
- `AgentPromptPanel.inline_document_renderable()` returns the identity (kind line and
  fields), then the XPROMPT section (heading, text, and light divider), then the body.
  It mirrors on-screen order, and test helpers such as `prompt_header_and_body_text`
  keep finding `AGENT XPROMPT`.
- Attempt-pinned views (`D`) never rendered AGENT XPROMPT and still don't, so they show
  no preview.
- Proc shells, monitors, gates, `bash`/`python`/`parallel` steps, top-level workflows,
  clans, and tribes have no xprompt. Their headers are unchanged apart from the removed
  blank row.

### Reliability rules

- **Same document.** The xprompt is attached to the `IdentityHeader` carried by the very
  document whose body omits it. The existing sink publishes both together on the UI
  thread, before `update()`'s unchanged-digest early return.
- **No j/k flicker for visited agents.** Each full non-hint paint records the
  highlighted xprompt, or "none", in a bounded in-memory memo on the prompt panel: an
  LRU of 128 entries keyed by `agent.identity`. `update_header_only` attaches the memo
  hit, which is pure memory and does no file I/O.
- **Stable height for unvisited agents.** On a memo miss, when the full paint could
  render an xprompt, the cheap identity carries `xprompt_pending=True`. The panel then
  holds the preview region at its last shown row count (capped by the current budget),
  with the quote bar and a dim `⋯` on the first row. The full paint about 150 ms later
  settles it, so the deck moves at most once per selection, and not at all while j is
  held.
- **Bounded UI-thread work.** Fitting only processes a source prefix large enough to
  fill the budget. It counts the remaining lines with `str.count("\n")`, never mutates
  the identity's `Text` objects, and is skipped on unchanged paints through the existing
  digest. The digest now also covers content width and budget.
- **Resize-correct.** Width changes (the panel's own resize) and column-height changes
  (`AgentDetail.on_resize`) re-fit the preview. When the header's rendered row count
  changes, `AgentDetail.reapply_main_view_pins()` runs so a bottom-pinned live reply
  stays pinned.
- **Hint mode.** Xprompt file hints keep their current document-order numbers. Because
  they set `has_hints`, the header renders expanded, so every `[N]` stays visible and
  selectable.

## Phase `preview`: pure fitting module and inert setting

These pieces are pure and inert: they touch neither the prompt-panel package nor the
header panel.

1. **New module `src/sase/ace/tui/widgets/agent_header_preview.py`:**
   - `XpromptPreviewFit` (a frozen dataclass with fields `text: Text`, `rows: int`,
     `truncated: bool`, `hidden_lines: int`).
   - `fit_xprompt_preview(source: Text, *, width: int, max_rows: int) -> XpromptPreviewFit`,
     which implements the reflow and fit rules from Design. Put reflow in a private
     helper that also records each source line's reflowed offsets, so `hidden_lines` can
     count the source lines whose content was not fully shown.
     - The result is a single `Text(no_wrap=True, overflow="ellipsis")` whose rows are
       joined by `"\n"`, each starting with the styled `▎ ` gutter and none wider than
       `width` cells.
     - Wrap with Rich `Text.wrap` against a module-level off-screen `Console`, so wide
       characters (CJK, emoji) are measured correctly.
     - `max_rows <= 0` returns an empty fit. Use a small width floor so tiny widths
       never crash.
   - `preview_row_budget(column_rows: int, share: float) -> int`, which implements the
     budget formula from Design (0 when `share <= 0` or `column_rows <= 0`).
   - Glyph and style constants (`▎`, `¶`, `#AF87FF`, dim).
2. **Setting.** Copy the pattern of `src/sase/ace/tui/agent_decks_settings.py` into a
   new `src/sase/ace/tui/agent_header_settings.py`:
   - `AgentHeaderSettings(collapsed_max_share: float = 0.35)`
   - `parse_agent_header_settings(ace_cfg)`: bools, non-numbers, negative values, and
     values above 0.6 fall back to the default, and ints coerce to float.
   - `agent_header_settings_for(widget)`
   - Parse it in `actions/_state_init_late.py` next to `_agent_decks_settings`, and
     annotate `_agent_header_settings` in `actions/startup.py`.
   - Add an `agent_header` block to `src/sase/config/sase.schema.json`
     (`collapsed_max_share`: number, minimum 0, maximum 0.6, default 0.35, with a
     description) and to `src/sase/default_config.yml` under `ace:` next to
     `agent_decks`, with a comment.
   - Leave `docs/configuration.md` for the `panel` phase, so nothing user-facing
     advertises the setting before it works.
3. **Tests.**
   - A new `tests/ace/tui/widgets/test_agent_header_preview.py` covering:
     - soft joins, and `¶` for blank lines and each block construct (and not for `+sase`
       or `#gh`)
     - fenced blocks and thematic breaks
     - indentation, tab, and trailing-whitespace normalization
     - style spans surviving reflow (the same substrings keep the same styles)
     - that every row is at most `width` cells wide and starts with the gutter
     - exact fit with no ellipsis, and the overflow ellipsis and `hidden_lines`
     - the `max_rows` values 0 and 1
     - wide characters
     - that the input `Text` is not mutated
     - a 100k-line source that fits quickly and still reports the right `hidden_lines`
     - a table-driven test for `preview_row_budget`, including 32 and 47 column rows at
       0.35, and share 0
   - Settings parser tests next to the existing decks-settings coverage.
4. **Symvision.** `fit_xprompt_preview`, `XpromptPreviewFit`, `preview_row_budget`, and
   `agent_header_settings_for` get their first non-test consumer only in `panel`. Add
   `--epic-symbol <epic bead id>(<symbol>)` entries to the `Justfile` Symvision
   invocation as needed, per `sase/memory/symvision.md`.

## Phase `document`: the XPROMPT travels with the detached identity

Work only in `src/sase/ace/tui/widgets/prompt_panel/` and its tests. With detachment
off, which stays the default in this phase, every document stays byte-identical, and all
existing tests and PNG goldens must pass unchanged.

1. **`IdentityHeader`** (`_identity_header.py`) gets new defaulted trailing fields:
   - `xprompt: Text | None = None`: the full highlighted, humanized xprompt with
     trailing newlines stripped. It may contain hint markers.
   - `xprompt_pending: bool = False`.

   It also gets these methods:
   - `with_xprompt(text, *, has_hints=False)`, which uses `dataclasses.replace`, ORs
     `has_hints`, and clears `pending`.
   - `with_xprompt_pending()`.
   - `expanded_renderable()`, which returns `expanded`, followed (when an xprompt is
     present) by a blank line, the `AGENT XPROMPT` title added with
     `append_section_heading`, and the text.

   `inline_renderable()` becomes the kind line, then `expanded_renderable()`, then the
   standard light major divider when an xprompt is present. Existing constructors in the
   tribe, clan, and workflow builders stay untouched.

2. **Gate.**
   `AgentPromptPanel.attach_identity_header_sink(sink, *, detach_xprompt=False)` stores
   the flag, and a `detaches_xprompt` property returns true only when a sink is attached
   and the flag is set. Add `detaches_xprompt` to the hint-document cache key in
   `_agent_display_hint_cache.py` next to `detaches_identity_header`, so a cached hint
   document built in the other mode is never reused.
3. **Shared helper.** Add a new module `_agent_display_xprompt.py`, because
   `_agent_display_render.py` is at 646 lines and near the `toobig` limit. It has a
   helper that takes the document carrier and the finished xprompt `Text` (plus its hint
   flag). When `detaches_xprompt` is on and the carrier has an identity, it attaches the
   xprompt through `with_identity_header(identity.with_xprompt(...))` and returns true.
   Otherwise it returns false, and the caller renders the body section exactly as today.
4. **Call sites.**
   - `_update_display_impl`: when the helper attaches, skip the heading, the text, and
     the `\n\n─…\n\n` divider that follows them.
   - `_update_family_display`, plain and hinted: skip the heading and text, and do not
     set `rendered_content_section`, so `AGENT PROMPT` gets no leading divider.
   - `render_agent_prompt_hint_body`: render the hinted xprompt into a fresh `Text` with
     the same `append_bounded_text_with_file_hints` and `apply_authored_prompt_overlays`
     calls (`region_start=0`), set the hint flag when the counter advanced, and keep the
     counter flowing, so numbering is unchanged.
5. **Memo and cheap path.**
   - After each full, non-hint, non-attempt-pinned paint of the standard or family
     document, with detachment on, record the xprompt `Text`, or `None` for "none", in
     the panel's bounded LRU keyed by `agent.identity`.
   - In `update_header_only`, with detachment on:
     - On a hit, attach the memo value (skip it when the value is `None`).
     - On a miss, mark the identity pending, but only when
       `agent_may_show_xprompt(agent)` is true. This helper sits in the new module. It
       returns false for clan containers, proc shells, monitors, gates,
       `bash`/`python`/`parallel` workflow children, top-level workflows (with the same
       predicate as `_update_display_impl`), and attempt-pinned panels, and true
       otherwise.
   - Never read files here.
6. **Tests.** Add new tests, for example
   `tests/ace/tui/widgets/test_identity_header_xprompt.py`, covering the standard,
   family, hint, and family-hint documents, with detachment on and off:
   - With detachment on, the body has no `AGENT XPROMPT` and no doubled or leading
     divider, and `identity.xprompt` has the same plain text and spans the body section
     had.
   - With detachment off, output is byte-identical.
   - Hint numbering is unchanged, and `has_hints` is set by xprompt hints.
   - The memo hit and miss produce the attached, "none", and pending states, and kinds
     with no xprompt are never pending.
   - The memo stays bounded.
   - `inline_renderable()` ordering is correct.
   - The hint cache key includes the new flag.
   - The sink receives the xprompt-bearing identity before the digest early return.

## Phase `panel`: header rendering, layout, docs, and verification

1. **Turn it on.** `AgentDetail.on_mount` attaches the sink with `detach_xprompt=True`.
2. **`AgentHeaderPanel`:**
   - Keep `column_rows` (set by `AgentDetail.on_resize` through a new
     `set_column_rows(rows)`), the settings share (`agent_header_settings_for`), and the
     last shown preview row count.
   - Collapsed content is a copy of `identity.compact`, then `"\n"`, then the
     `fit_xprompt_preview(identity.xprompt, width=<content width>, max_rows=budget)`
     rows.
     - The content width comes from the content `Static`. Before the first layout, fall
       back to a sensible width and re-fit in `on_resize`.
     - While pending, show the held placeholder rows instead.
     - Show no preview when the budget is 0 or there is no xprompt.
   - Expanded content is `identity.expanded_renderable()`.
   - The subtitle adds `+N lines · ` when the fit was truncated.
   - Extend the digest with the width, the budget, and the pending and hold state.
   - Force a repaint when a resize or budget change alters the fit.
   - Expose the rendered row count. `AgentDetail._on_identity_header` (and the resize
     path) calls `reapply_main_view_pins()` when that count changes.
3. **CSS** (`src/sase/ace/tui/styles.tcss`): add `scrollbar-size-horizontal: 0;` to
   `#agent-header-panel` and `#agent-jump-panel`, and leave the other rules alone.
4. **Docs.**
   - `docs/ace.md`:
     - Rewrite the **Header panel** bullet. Cover the XPROMPT preview (quote bar,
       reflow, `¶`, the budget, the `+N lines` subtitle, the expanded XPROMPT, hint
       mode, attempt-pinned, and share 0).
     - Say that `AGENT XPROMPT` is no longer a body section or a `Ctrl+J` stop, and that
       `,/` finds the user's words through `AGENT PROMPT`.
     - Fix the family paragraph that lists `AGENT XPROMPT` among body navigation
       anchors.
     - Fix "two concise rows" wording where it no longer holds.
   - `docs/configuration.md`: add an `agent_header` row to the `ace` table and an
     `#### ace.agent_header` section in the `ace.agent_decks` style, with the source
     path.
5. **Tests.**
   - Pilot tests in `tests/ace/tui/widgets/test_agent_header_panel.py`, using agents
     whose artifacts directory has a `raw_xprompt.md`:
     - Collapsed shows 2 chip rows plus the quote-bar preview, and the body lacks
       `AGENT XPROMPT`.
     - The preview row count equals `preview_row_budget` for the app size, and a short
       prompt uses exactly the rows it needs.
     - Overflow shows `+N lines` in the subtitle.
     - `d` shows the heading and full text, and toggles back.
     - The panel height equals the content rows + 2 (the phantom row is gone), for both
       the header and the jump panel.
     - The pending hold keeps the previous row count and then settles.
     - A visited agent paints its preview on the cheap path.
     - Share 0 means no preview, but expanded still shows the XPROMPT.
     - A column resize changes the budget.
     - A bottom-pinned body stays pinned across a row-count change.
     - The existing `len(rows) == 2` assertions still hold for an agent without an
       xprompt.
   - Fix any other mounted-`AgentDetail` tests that looked for `AGENT XPROMPT` in the
     body.
6. **Visual.**
   - In `tests/ace/tui/visual/test_ace_png_snapshots_agents_xprompt.py`, press `d`
     before capturing, so the full highlighted XPROMPT (every asserted token) is in the
     expanded header, in both the dark and light variants.
   - Add goldens (in the existing agents visual test style, controlled time and data):
     - a long hard-wrapped prose prompt with a list and a code fence, collapsed and
       truncated, with `+N lines`
     - a short directive-heavy prompt, collapsed, that fits
     - the same agent expanded
   - Run `just fix-tui-screenshots` through `/sase_monitor`, and inspect the retained
     report and every golden change before finalizing, as
     `sase/memory/tui_screenshot.md` requires. Most Agents goldens change because the
     phantom row is removed and the XPROMPT moves.
7. **Live check.** Capture the real TUI running this workspace's code with
   `sase screenshot -s 120x40 …` and `-s 200x55 … -p j -p j`, and include a `d` press.
   Confirm:
   - the quote-bar preview fills the width, highlighting is intact, and there is no
     blank row
   - the overflow subtitle appears
   - j/k does not flicker
   - the expanded view is correct
8. **Symvision.** Remove every `--epic-symbol` entry the `preview` phase added, because
   they are consumed now. Then run `just check` per `sase/memory/lint_and_test.md`.

## Out of scope

- Previewing `AGENT PROMPT` for agents without a `raw_xprompt.md`.
- Keyboard scrolling of the expanded header.
- Including header text in `,/` search.
