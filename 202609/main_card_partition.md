---
tier: tale
title: Card-partitioned Main documents
goal:
  Every Agents metadata-panel builder emits Context/Reply/Output/Summary card parts that
  tree walkers understand. Failed-agent tracebacks move to the top of the Reply or
  Output card under a TRACEBACK heading, and the rest of the panel renders unchanged.
size: medium
proposed_by: bbugyi200.athena.sase-17d.2
bead: sase-17d.2
status: done
---

- **PARENT:**
  [202609/agents_tab_decks_and_cards.md](https://github.com/sase-org/sase--plans/blob/main/202609/agents_tab_decks_and_cards.md)
- **BEAD:**
  [sase-17d.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-17d/sase-17d.2.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-17d.2](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-17d.2.md)
- **COMMITS:**
  - [9abf08b](https://github.com/sase-org/sase/commit/9abf08b5df74ebc17f5e293e4909702867105880)
    — feat(agents-tab): card-partitioned Main documents

# Plan: Card-partitioned Main documents (phase `main-card-partition`, bead sase-17d.2)

This is phase `main-card-partition` of epic sase-17d ("Agents tab agent data decks and
cards", plan `plan:202609/agents_tab_decks_and_cards.md`, §3.1, §4.1 and §6). The phase
bead is **sase-17d.2**. When the epic plan and this plan disagree about this phase, this
plan wins. It refines §6 using the current code.

All paths are relative to `src/sase/ace/tui/` unless they start with `src/`, `tests/` or
`docs/`.

## 1. Goal

Every Agents metadata-panel builder (`AgentPromptPanel`, under `widgets/prompt_panel/`)
emits its document as ordered **card parts**: `context`, `reply` (titled "Reply" or
"Output") or `summary`. Tree walkers learn the card wrapper. A failed agent's error
traceback moves from its current spot (today it often sits between the `AGENT PROMPT`
heading and the prompt body) to the **top of the Reply/Output card**, under a new
`TRACEBACK` section heading.

**Invariant:** the visible panel stays byte-identical to today except for the traceback
move. No feature flag is involved yet: `agent_decks` is created in `deck-panel-core`,
which is the first consumer of the card structure. There is no `sase-core` change,
because this is presentation-only Textual/Rich state.

## 2. Design decisions

### D1. Module: `widgets/decks/card_part.py`

- Create the `widgets/decks/` package with an `__init__.py` that holds only a docstring.
  Do not re-export anything, so later phases can add modules that import `prompt_panel`
  without creating import cycles.
- `card_part.py` depends only on `rich`.
- Do not touch the `widgets/__init__.py` / `.pyi` lazy exports.

### D2. `CardPart` API

`card_part.py` contains the following:

- **Constants:**
  - `CONTEXT_CARD_ID = "context"`, `REPLY_CARD_ID = "reply"`,
    `SUMMARY_CARD_ID = "summary"`
  - `CONTEXT_CARD_TITLE = "Context"`, `REPLY_CARD_TITLE = "Reply"`,
    `OUTPUT_CARD_TITLE = "Output"`, `SUMMARY_CARD_TITLE = "Summary"`
  - The Output card reuses `REPLY_CARD_ID`, so a sticky Reply follows onto process nodes
    (epic §3.1).
- **`class CardPart`:** a slotted class with `card_id: str`, `title: str` and
  `renderables: tuple[RenderableType, ...]`.
  - `__rich_console__` yields `Group(*self.renderables)`.
  - `__rich_measure__` delegates to that same `Group`.
  - Rendering `Group(CardPart(a, b), CardPart(c))` must be byte-identical to rendering
    `Group(a, b, c)`. Add a unit test that compares the two segment streams exactly.
- **Small constructors:** `context_card(*r)`, `reply_card(*r)`, `output_card(*r)`,
  `summary_card(*r)`. Builders use these so ids and titles never drift.
- **`card_document(*parts: CardPart | None) -> Group`:**
  - Drops `None` parts and parts with no renderables.
  - Returns `Group(*parts)`.
  - Every builder calls `self.update(card_document(...))`.
- **`flatten_card_document(content: object) -> object`:** the legacy-shape projection.
  - If `content` is a `CardPart`, or a `Group` with any top-level `CardPart`, return the
    in-order concatenation of the card children plus any loose top-level renderables.
    When that list has exactly one element, return the element itself (never a one-child
    `Group`); otherwise return `Group(*children)`.
  - Return any other `content` unchanged.

Symvision: every public symbol above needs a non-test consumer in this phase (builders,
`update()` and the walkers). Do **not** add `split_card_parts` yet. The epic plan (§6)
lands it in `deck-panel-core` together with its first real consumer. Tests reach the
card structure by reading `Group.renderables` directly, through a small test-local
helper. No `--epic-symbol` entry should be needed. If one turns out to be necessary, key
it to a later still-open phase bead of sase-17d, never to sase-17d.2.

### D3. Display flattening in `AgentPromptPanel.update()` (the key decision)

`widgets/prompt_panel/__init__.py` `update()` keeps running the identity/jump sinks and
the digest on the **card-structured** content. It then passes
`flatten_card_document(content)` to `super().update(...)`.

Why:

- **Visual type.** Textual's `visualize()` turns a bare top-level `rich.text.Text` into
  a `Content` visual, but wraps everything else in a `RichVisual`. Several single-card
  documents are a bare `Text` today: clan summaries, header-only paints without a
  traceback, the no-prompt case, "attempt not found", and plain-`Text` headers in tests.
  Wrapping them in `Group(CardPart(...))` would silently change their visual type and
  the section-anchor path (`_anchors_for_strips` versus `_anchors_for_rich_visual` in
  `_section_navigation.py`). Unwrapping one-element documents keeps the legacy type.
- **Legacy shape.** `panel.content` keeps today's shape for every existing reader:
  - metadata search (`actions/agents/_metadata_search.py`)
  - the zoom modal (`modals/zoom_panel_content.py`, `zoom_panel_search.py`)
  - `src/sase/ace/testing/prompt_document.py`
  - Textual-level tests

Keep `self._identity_last_content = content` as the card-structured document. It is the
future Main-source payload. `inline_document_renderable()` then learns the wrapper (D4).

Performance: flattening is one shallow pass over the top-level children, with no I/O and
no rendering, so it is safe on the j/k path (`tui_perf.md` rules 1 and 7). The digest
skip still returns before any flatten or `super().update()` work.

### D4. Tree walkers

- **`util/renderable_digest.py` `_update_digest`:** add a `CardPart` branch (before the
  generic fallbacks) that hashes a tag byte, `card_id`, a separator, `title`, and then
  each child. Card id and title are therefore part of the digest. Unchanged documents
  still produce equal digests and still skip.
  - Avoid an import cycle: `util/` must not import `widgets/`. Either duck-type on a
    marker attribute (for example `CardPart` defines `__sase_card_part__ = True`, and
    the digest checks `getattr(node, "__sase_card_part__", False)` and reads `card_id`,
    `title` and `renderables`), or place the digest hook where no cycle arises. Check
    the real import graph first.
- **`widgets/prompt_panel/_identity_header.py` `_find_carrier`:** descend into
  `CardPart.renderables`. This covers `find_identity_header`, `find_member_jump_map` and
  `find_member_roster`. The existing `.renderable` fallback already descends through
  `CachedRenderable`.
- **`AgentPromptPanel.inline_document_renderable()`:** run
  `flatten_card_document(content)` first, so its `isinstance(content, Text)` roster
  branch keeps working. This also covers the zoom seed
  (`actions/agents/_panel_detail.py`), which reads through this method.
- **`widgets/renderable_text.py` `renderable_to_text`:** no code change, because Rich
  renders `CardPart` natively. Add a test that pins this.
- **`_agent_display_hints.py` `_plain_renderable_content`:** add a `CardPart` branch
  (join the plain text of its children). Only needed if a card document still reaches it
  after D5.

### D5. Hint mode: split the single `Text` and cache per card

Today, hint-mode documents are one growing `Text` (`header_text`) wrapped in one
`CachedRenderable` by `_prepare_cached_hint_renderable`. The family hint path is already
a `Group`.

- **Split into two Texts.** Builders write the context sections into `header_text`, then
  write the reply/output sections into a separate `reply_text = Text()`. The single hint
  counter is threaded through both, so `[N]` numbering stays continuous and follows
  reading order.
- **Avoid a blank line at the split.** Rich renders a `Text` that ends in `"\n"` inside
  a `Group` with one extra blank line (checked: `Group(Text("a\n"), Text("b\n"))`
  renders `a\n\nb\n\n`, while `Text("a\nb\n")` renders `a\nb\n\n`). Whenever one Text
  stream is split into consecutive parts, set the leading part's `end = ""`.
  `AgentHeaderRenderable` has an `end` setter. Do this **before** wrapping in
  `CachedRenderable`.
- **Cache per card.** `_prepare_cached_hint_renderable` accepts a card document and
  returns a new card document in which each part's children are wrapped in **one
  `CachedRenderable` per card**, built from that card's plain text. This keeps card
  boundaries visible to walkers and to the future deck views, and keeps segment caching
  per card.
  - Store that card document in `_agent_hint_renderable`.
  - Widen `AgentHintRenderCacheEntry.renderable` in `_agent_display_hint_cache.py` and
    the `isinstance(renderable, CachedRenderable)` cache-insert guard in
    `_agent_display_hints.py` to accept the card document, so hint-cache hits keep
    working.
- **Affected modules:**
  - `_agent_display_hint_render.py`
  - `_agent_display_hint_body.py` (`render_agent_prompt_hint_body`: write `AGENT REPLY`
    / `AGENT CHAT` / "Waiting…" into the reply target, not into `header_text`)
  - `_agent_display_hint_sections.py` (proc shell, monitor and gate: preview and section
    go to Context, `build_*_output` goes to Output)
  - the family hint path (the same code as D6's family builder)
- **Hint-mode traceback.** Today it is appended with hints right after the header,
  before the proc/monitor/gate dispatch. Move it to the top of `reply_text` under the
  `TRACEBACK` heading, and annotate it with `append_bounded_text_with_file_hints` using
  the running counter.
  - For proc shells, monitors and gates, sequence the work: annotate context first, read
    the annotator's count, append the annotated traceback, then build a fresh annotator
    from the updated counter for the output.

### D6. The `TRACEBACK` block

- Add one shared helper in `widgets/prompt_panel/`, for example `_traceback_section.py`
  (or `_helpers.py` if that stays small). It returns the Rich-mode block:
  - a divider `Text` (`"\n" + "─" * 50 + "\n\n"`, dim, matching the existing reply
    header dividers)
  - `append_section_heading(..., "TRACEBACK", section_id="traceback")`
  - the existing `error_tb_syntax`
- Add a Text-mode variant for hint mode (heading plus annotated traceback text).
- `TRACEBACK` becomes a new Ctrl+J/K section stop and fold anchor. That is the intended
  visible change.
- Place the block at the **top of the Reply/Output card**, before `reply_header`,
  `OUTPUT`, `LOG TAIL` or `STEP OUTPUT`.

### D7. `show_empty` stays cardless

Epic §3.1 says "No selection: none, so the Main deck shows its empty state". An empty
card document cannot render the legacy `No agent selected` text, and `deck-panel-core`
already delegates `show_empty` in deck mode (epic §7). So `show_empty` keeps publishing
its loose `Text`. Document this with a one-line comment that names `deck-panel-core`.
"Attempt not found" is a single Context card.

## 3. Card mapping per builder

| Builder (module)                                                                                                       | Context card                                                                                                                             | Reply / Output card                                                                                                                                                                                                            |
| ---------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Regular / queued agent, with prompt (`_agent_display_render.py`, all three status branches)                            | `header_text` (with `AGENT XPROMPT` and the `AGENT PROMPT` heading) + `prompt_syntax`                                                    | **Reply**: TRACEBACK block (if any), then `reply_header` and every existing reply renderable (phases, merged attempt history, timestamp dividers, markdown, "No response file found." / "Waiting for agent response...")       |
| Regular agent, no prompt file (same module)                                                                            | `header_text` ending in "No prompt file found."                                                                                          | **Reply**: TRACEBACK block only (card omitted when there is no traceback)                                                                                                                                                      |
| Family container (`_agent_display_family_render.py`, normal and hint)                                                  | `header_text` + xprompt + `AGENT PROMPT` and the prompt renderable                                                                       | **Reply**: TRACEBACK block (still gated by the `error` family fold level, as today), then `AGENT REPLY · N` and the phases                                                                                                     |
| Attempt-pinned (`_agent_display_attempts.py`)                                                                          | banner (`ATTEMPT ERROR`) + `error_full` / snippet + divider + `AGENT PROMPT` + prompt. The attempt's own error stays here, per epic §3.1 | **Reply**: `ATTEMPT N REPLY` header + chunks / "(no partial reply captured)"                                                                                                                                                   |
| Proc shell / monitor / gate (`_agent_display_step_render.py`, and hint variants in D5)                                 | header + `build_proc_shell_preview` (proc) + `build_*_section`                                                                           | **Output**: TRACEBACK block + `build_*_output` (`LOG TAIL` / `OUTPUT`)                                                                                                                                                         |
| Bash / Python step                                                                                                     | header + `BASH COMMAND` / `PYTHON CODE` heading + source                                                                                 | **Output**: TRACEBACK block + `STEP OUTPUT` header + output                                                                                                                                                                    |
| Parallel step                                                                                                          | header only                                                                                                                              | **Output**: TRACEBACK block, then a **new** `STEP OUTPUT` heading `Text` (moved out of `header_text`), then output / "No output available." Set `header_text.end = ""` if needed so the no-traceback rendering stays identical |
| Top-level workflow (`_workflow_render.py` `_build_workflow_detail_renderable`, `_workflow_display.py`)                 | the whole document. Its traceback stays in place, because a workflow row is Context-only (epic §3.1) and it has no prompt-body quirk     | none                                                                                                                                                                                                                           |
| Clan container (`_update_display_impl` clan branch, `update_header_only` for clans, `_update_clan_display_with_hints`) | none                                                                                                                                     | **Summary** card = the whole document                                                                                                                                                                                          |
| Tribe summary (`show_tribe_summary` → `build_tribe_detail_text`)                                                       | none                                                                                                                                     | **Summary** card = the whole document                                                                                                                                                                                          |
| `update_header_only` (non-clan)                                                                                        | the cheap header                                                                                                                         | **Reply** holding only the TRACEBACK block when a traceback exists (see below)                                                                                                                                                 |

Why `update_header_only` gets a traceback-only Reply card: epic §6 says "Context only".
But the partial paint has shown the traceback since Phase 3, and moving it into Reply
matches the card set and order of the full paint that replaces it about 150 ms later, so
j/k never makes the traceback jump. It also keeps the flag-off change limited to the
traceback move. Record this as a note on the bead.

Wrap at the `self.update(...)` call sites. Do not push card logic into
`build_header_text` or `build_tribe_detail_text`, which also feed the identity or other
callers. Build the card parts at the end of each branch, after all in-place appends, so
late `append` calls still land in the right card.

Module size: `_agent_display_render.py` is 636 lines, and the `toobig` gate warns
at 700. If the change pushes it near that limit, move the prompt/reply branch assembly
into a focused helper module, for example `_agent_display_reply_render.py`. Keep every
new module under about 500 lines.

## 4. Implementation steps

1. **Characterization tests first, before touching any builder.** Add a test module, for
   example `tests/ace/tui/widgets/test_prompt_panel_card_partition.py`, plus helpers if
   it grows past about 500 lines. It drives the builders through a mixin-only fake panel
   that records `update()` calls, following the existing
   `tests/ace/tui/widgets/test_agent_display_header_only.py` pattern.
   - Record the current `renderable_to_text(...)` output for a fixture of every node
     kind in §3, with and without a traceback, in both normal and hint mode. Use a fixed
     width and pin every volatile input, such as timestamps and paths.
   - Keep the no-traceback expectations unchanged through the whole phase. Rewrite only
     the traceback expectations, in step 5.
   - Also pin the styled output for a representative subset (regular agent, hint-mode
     agent, parallel step): use `Console(record=True).export_text(styles=True)`, or
     compare `console.render` segments. This catches `end=""` blank-line drift and style
     drift that plain text would miss.
2. **Add the card module (D1, D2)** with unit tests: segment-stream equivalence,
   `card_document` dropping empty parts, and `flatten_card_document` (single-element
   unwrapping, loose renderables, non-card passthrough).
3. **Walkers and the `update()` choke point (D3, D4).** Tests:
   - digest includes id and title (same children in different cards → different digests;
     rebuilt equal documents → equal digests)
   - `_find_carrier` finds a carrier nested in a `CardPart`
   - `inline_document_renderable` with a card-wrapped `Text` plus roster matches the
     legacy output
   - `renderable_to_text` on a card document
4. **Wrap every builder** per §3, and do hint mode per D5. Run the step-1 tests after
   each builder. The no-traceback expectations must stay green without being edited.
5. **Traceback move (D6).** Update the traceback expectations. "Equivalent modulo the
   traceback" means two things:
   - the new text with the `TRACEBACK` block removed equals the legacy text with the
     traceback removed
   - the `TRACEBACK` heading is the first section of the Reply/Output card
6. **Fix the existing tests the new shape breaks.** Fake panels that record raw
   `update()` arguments and flatten only `Text`/`Syntax` children will now see
   `CardPart`s. Examples:
   - `tests/ace/tui/widgets/_agent_display_helpers.py`
   - `tests/ace/tui/widgets/test_agent_display_header_only.py`
   - `test_agent_display_attempt_pinned.py`
   - `test_agent_display_clan.py`
   - `test_identity_header.py`
   - `_agent_display_header_enrichment_helpers.py`
   - `_agent_deltas_helpers.py`
   - `test_agent_display_workflow_async.py`
   - `test_agent_display_bead_async.py`
   - `test_agent_clan_aggregation_async.py`

   Route the recorded renderable through `flatten_card_document` in those helpers,
   instead of weakening assertions. Update any assertion about the traceback's position
   to the new position. Tests that read `panel.content` through a real app should need
   no change (D3).

## 5. New tests (required by epic §6)

- **Partition per node kind.** For every row of §3, check card ids, titles and order,
  and that the traceback is the first section of Reply/Output. Also check:
  - Output cards use id `reply` with title `Output`.
  - Clan and tribe produce only `summary`; workflow produces only `context`.
  - `show_empty` stays cardless.
- **Plain-text equivalence** (§4 steps 1 and 5).
- **Hint numbering continuity.**
  - Hint-mode regular, family and proc-shell documents that have hints in both cards
    number `[1]..[n]` in reading order with no gaps or repeats across the card boundary.
  - Hint mappings still resolve to the right paths.
  - A traceback containing a file path gets a hint number after the prompt's hints.
- **Digest stability.** Two builds of the same agent produce equal digests, and a second
  `AgentPromptPanel.update()` with a rebuilt equal document skips (the section
  generation does not bump). Also check that the hint-cache hit path re-publishes the
  cached card document.

## 6. PNG goldens

- Only views that show a failed agent's traceback should change. Find the fixtures, for
  example with `grep -rln "error_traceback" tests/ace/tui/visual`, which currently lists
  the clan fixtures and the tribe panel. Check whether those views actually render an
  `error_tb_syntax` traceback or only clan/tribe error snippets. Clan and tribe
  summaries should **not** change.
- Run a targeted `just fix-tui-screenshots -- <selectors>` over the Agents-tab visual
  modules (`tests/ace/tui/visual/test_ace_png_snapshots_agents*.py`). This can outrun a
  turn, so run it through `/sase_monitor` with `TESTING`/`TESTED`.
- Inspect `latest-report.json` and every update group.
- Acceptance: updates are limited to traceback-bearing views, where the traceback now
  sits under `TRACEBACK` at the top of the reply. Any other drift is a regression to
  fix, not a golden to accept.
- If no existing golden shows the traceback quirk, say so in the bead note. Do not add a
  new golden in this phase; the flag-on goldens start in `deck-navigation-keys`.

## 7. Verification and closing

1. `just install` if needed, then `just fix` (or `just fmt`), then `sase tool run check`
   (through `/sase_monitor` if it risks outrunning the turn). Do not run
   `just check-full`.
2. Before landing, run `pytest -s -m slow tests/ace/tui/bench_tui_jk.py` with
   `SASE_TUI_PERF=1` and confirm j/k p95 did not regress (epic §4.8). This phase only
   adds a shallow flatten walk on `update()`.
3. Run `sase bead epic-symbols sase-17d.2`. Resolve any leftover entry, or re-key it to
   a later still-open phase bead of sase-17d.
4. `sase bead close sase-17d.2 --note "<what was verified>"`. Close **only** sase-17d.2,
   never the epic sase-17d or any ancestor. Record discovered follow-ups as
   `sase bead note sase-17d.2 'PROPOSED FOLLOW-UP: ...'`; do not create beads.

## 8. Out of scope

- The `agent_decks` flag, the `widgets/decks/` model, views, panels and
  `split_card_parts` (all `deck-panel-core`).
- Stripping leading dividers from cards for paged rendering (`deck-panel-core` /
  `deck-spread-mode`).
- Any key, keymap or `default_config.yml` change.
- Any `sase-core` change.
