---
tier: tale
title: XPrompt syntax highlighting in the agent metadata panel
goal: 'The AGENT XPROMPT section of the agent metadata panel on the Agents tab renders
  with the same xprompt token highlighting the prompt input widget supports (invocations,
  args, directives, separators, known skills, alt branches), in both the normal and
  file-hint render paths, without measurable impact on TUI responsiveness.

  '
create_time: 2026-07-16 16:38:37
status: done
prompt: 202607/prompts/agent_xprompt_panel_highlighting.md
---

# Plan: XPrompt syntax highlighting in the agent metadata panel

## Problem

The prompt input widget (`PromptTextArea`) highlights xprompt syntax — `#invocation`, `:args`/`(args)`, `%directives`,
`---` separators, known `/skill` references, and `%{a | b}` alt branches — via `sase.xprompt.xprompt_inspect.tokenize()`
overlays. The AGENT XPROMPT section of the agent metadata panel (`AgentPromptPanel`, right side of the Agents tab)
renders the same content as **plain humanized text** in both of its render paths:

- Standard path: `src/sase/ace/tui/widgets/prompt_panel/_agent_display_render.py` (`_update_display_impl`, the "AGENT
  XPROMPT section" block appends `self._display_raw_xprompt(...)` as a plain string into `header_text`).
- Hint path: `src/sase/ace/tui/widgets/prompt_panel/_agent_display_hints.py` (`update_display_with_hints`, appends the
  humanized xprompt through `append_text_with_file_hints` with no token styling).

A read-only xprompt highlighter already exists and is the established convention for read-only surfaces:
`src/sase/ace/tui/util/xprompt_syntax.py::highlight_prompt_text(text) -> rich.text.Text`. It applies a monokai markdown
`Syntax` base plus `xprompt_inspect` and `alt_inspect` token overlays using the hardcoded `XPROMPT_TOKEN_STYLES`
palette, fails open to plain `Text` on oversized (> `MARKDOWN_SYNTAX_HIGHLIGHT_MAX_BYTES` = 24 KB) or malformed input,
and is already used by the stashed-prompts / pinned-stash / stash-preview modals. The metadata panel should reuse it.

## Design decisions

1. **Reuse `highlight_prompt_text`, not the TextArea theme machinery.** The prompt input's theme-adaptive styles live in
   a `TextAreaTheme` overlay (`_xprompt_syntax_highlight.py`) that only works inside a `TextArea`. Read-only Rich panels
   already standardize on `highlight_prompt_text` with the hardcoded `XPROMPT_TOKEN_STYLES` palette (which visually
   matches the dark-theme prompt-input colors), and the panel's other sections (AGENT PROMPT markdown) are already
   fixed-theme monokai. Stay with that convention.

2. **Close the `/skill` parity gap in the shared helper.** `highlight_prompt_text` currently never emits `skill` spans
   (it calls `tokenize()` without `known_skills`, and `XPROMPT_TOKEN_STYLES` has no `"skill"` entry). Extend it:
   - Add an optional keyword param `known_skills: frozenset[str] = frozenset()` that is forwarded to
     `xprompt_inspect.tokenize()`.
   - Add a `"skill"` style to `XPROMPT_TOKEN_STYLES` (bold, in the accent/blue family used elsewhere in the palette,
     e.g. `bold #87AFD7`-ish — implementer picks a hex that reads well next to the existing entries and differs from
     `separator`).
   - Existing callers are unaffected (default keeps current behavior).

3. **Skill names come from the app-owned warm prompt catalog, fail-open.** Mirror the prompt input's approach
   (`_xprompt_arg_hints.py::_get_warm_xprompt_arg_assist_entries`): call
   `app.get_prompt_catalog_assist_entries(project, schedule=...)` through `getattr` guards, filter `entry.is_skill`, and
   derive the project context from the raw xprompt's leading VCS tag (reuse the existing derivation logic if it can be
   shared cheaply; otherwise fall back to the agent's project or `None`). When the app or catalog is unavailable/cold
   (e.g. unit tests use `FakePromptPanel` with no app), degrade to `frozenset()` — everything else still highlights.
   Using `schedule=True` (like the prompt input's non-warm getter) is acceptable since warming runs in an app worker,
   but the lookup itself must stay disk-free on the render path.

4. **Standard path: highlighted `Text` appended into the header, with a small keyed cache.** In `_update_display_impl`,
   replace the plain `header_text.append(f"{raw_xprompt}\n")` with `header_text.append_text(...)` of the highlighted
   Text built from the _humanized_ xprompt (so token offsets match what is displayed; humanized refs like `#gh:widgets`
   still tokenize as invocations). Add a small LRU cache on the mixin mirroring the existing `_humanized_text_cache`
   pattern (keyed by humanized-content hash/length plus the skill-name set; limit ~24; cleared in
   `_reset_markdown_render_cache_for_agent` alongside the other caches) so auto-refresh ticks and repeated selections
   don't re-run pygments. Preserve the current trailing-newline/divider structure exactly (`highlight_prompt_text`
   right-crops the synthetic trailing newline already).

5. **Hint path: overlay token styles on the hint-marked text; keep hints authoritative.** File-hint markers (`[N]`) are
   inserted inline, so pre-computed offsets on the raw text would misalign. Instead: record the panel Text's plain
   length before `append_text_with_file_hints(...)` appends the xprompt section, then tokenize the appended slice
   (`header_text.plain[start:]` at that moment, i.e. humanized + hint-marked) and `stylize` the spans with the `start`
   offset, wrapped in a fail-open `try/except`. Do **not** apply the markdown `Syntax` base in hint mode — hint mode
   deliberately renders plain text (see the `update_display_with_hints` docstring); only the xprompt token overlays are
   added. A marker landing inside a token simply breaks that one match and the text stays plain — acceptable. Hint
   numbering, mappings, and marker styling must be unchanged.

6. **Alt-branch spans come along for free** via `highlight_prompt_text`'s existing `alt_inspect.tokenize` overlay,
   matching the prompt input's `AltSyntaxHighlightMixin`. For the hint path, apply the same alt overlays as the xprompt
   ones (share a small "apply overlays to a Text region" helper in `util/xprompt_syntax.py` so the span-kind → style
   mapping is defined once).

## Performance notes (constraints the implementation must respect)

- The AGENT XPROMPT section is only built inside the full detail update, which is already debounced by
  `DetailPanelDebouncer` (150 ms) — the j/k highlight fast path (`update_header_only`) is untouched. Do not add any work
  to the highlight/keystroke path.
- `highlight_prompt_text` is already capped (24 KB fail-open to plain text) and tokenization is regex-only; typical
  xprompts are well under a few KB. The content-keyed cache (decision 4) makes repeat renders O(1); this mirrors how
  AGENT PROMPT markdown goes through `LazySyntaxRenderCache`.
- Skill-name lookup must be a warm in-memory read (no disk I/O, subprocess, stat, or glob on the render path) and must
  not introduce a new refresh path.
- Sanity-check responsiveness after implementation: run the visual suite and, if any doubt remains, a quick
  `SASE_TUI_PERF=1` j/k spot check on the Agents tab (target p95 < 16 ms per `sase/memory/tui_perf.md`).

## Implementation steps

1. **Extend the shared helper** (`src/sase/ace/tui/util/xprompt_syntax.py`): `known_skills` param, `"skill"` style, and
   a reusable overlay-application helper usable both on whole strings (existing behavior) and on a
   `(text_obj, region_start, region_source)` triple for the hint path. Keep the module fail-open guarantees.
2. **Add a skill-name provider for the panel** (small getattr-guarded helper in the prompt_panel package or on the
   mixin): agent/raw-xprompt → `frozenset[str]` of warm skill names, defaulting to empty.
3. **Wire the standard path** in `_agent_display_render.py`: highlighted, cached xprompt Text (built from humanized
   content) appended in place of the plain string; cache cleared with the existing per-agent cache reset.
4. **Wire the hint path** in `_agent_display_hints.py`: post-append token overlay on the appended region as per
   decision 5.
5. **Tests** (see below), then `just install && just check`.

## Testing

- `tests/ace/tui/util/test_xprompt_syntax.py`: `known_skills` emits styled `skill` spans; default call emits none;
  existing behavior unchanged.
- `tests/ace/tui/widgets/test_agent_display_xprompt.py` (conventions: `FakePromptPanel`, `make_artifact_agent`,
  `plain_of`):
  - Standard path: rendered AGENT XPROMPT section carries token styles (e.g. an invocation span styled with the
    `invocation` style) on the _humanized_ text, and plain-text assertions from existing tests still pass (highlighting
    must not alter the plain rendering).
  - Hint path: `[N]` markers, `file_hints` mappings, and section compactness unchanged; token styles present on the
    xprompt region.
  - Fail-open: an oversized (> 24 KB) xprompt still renders as plain text.
  - Cache behavior: consecutive `update_display` calls for the same agent/content highlight once (e.g. count calls via
    monkeypatch); cache resets when the displayed agent identity changes.
- PNG visual snapshot: add one Agents-tab detail-panel snapshot whose fake agent has a raw xprompt exercising the token
  kinds (e.g. `#gh:sase %auto #pr:my_change %m:opus` + a `---` separator), following the existing
  `tests/ace/tui/visual/test_ace_png_snapshots_agents*.py` helpers; generate the golden with
  `--sase-update-visual-snapshots` and verify via `just test-visual`.

## Risks / edge cases

- **Humanization interplay**: tokenization must run on the humanized (displayed) text, never the raw text, or spans
  misalign. Covered by the logical-project-name tests.
- **Hint markers inside tokens**: rare (`[[ ... ]]` multi-line args, paths inside invocation args); the overlay simply
  fails to match there and text stays plain — no crash, hints always win.
- **Style collisions with hint markers**: `stylize` overlays could recolor a `[N]` marker if a token span covers it;
  verify the marker remains visually distinct in the snapshot, and if it doesn't, skip token spans that overlap marker
  ranges.
- **`FakePromptPanel` has no app**: every app/catalog access must be getattr-guarded (skill spans simply absent in tests
  that don't stub it).

## Non-goals / follow-ups

- The agent run-log modal (`src/sase/ace/tui/modals/agent_run_log_modal.py`) also renders an AGENT XPROMPT preview as
  plain text; once the shared helper supports skills it could adopt `highlight_prompt_text` in one line, but that
  surface is out of scope here.
- No theme-adaptive restyling of read-only surfaces, and no changes to the editable prompt input's highlighting or to
  the Rust core (highlighting is presentation-only Python by design; the canonical grammar stays in the Rust core
  mirrors already consumed by `xprompt_inspect`).
