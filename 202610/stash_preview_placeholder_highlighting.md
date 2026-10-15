---
tier: tale
title: Highlight placeholders in the prompt-stash preview
goal:
  Raw <placeholder> tags in stash, trash, and pinned-stash previews render with the
  prompt editor's placeholder styling.
size: small
decisions:
  scope:
    ask:
      Should the agent panel's raw-prompt header highlight placeholders too, not just
      the stash previews?
    choices:
      prompt_text:
        All highlight_prompt_text views, including the agent raw-prompt header,
        highlight placeholders
      stash:
        Only stash, trash, and pinned-stash previews opt in via a flag; agent header
        stays unchanged
    default: prompt_text
    why:
      The raw agent prompt is the same authored text; one renderer keeps every view
      matching the editor
    answer: prompt_text
proposed_by: bbugyi200.athena.10j
decided_by: auto
create_time: 2026-10-15 11:15:29
status: wip
---

# Highlight `<placeholder>` tags in the prompt-stash preview

## Problem

The Prompts modal's Stash tab preview (`PromptStashPreviewPane` in
`src/sase/ace/tui/modals/_prompt_stash_preview.py`) renders the stashed prompt through
`highlight_prompt_text()` in `src/sase/ace/tui/util/macro_syntax.py`. That renderer
applies Monokai Markdown, semantic glossary/repo overlays, macro/alt/project-tag
overlays, and (optionally) artifact-ref overlays, but it never applies raw-placeholder
overlays. So a stashed prompt such as `Review the <research>.md file ...` shows
`<research>` as plain text, while the same prompt in the prompt-bar editor shows it
highlighted.

The prompt-bar editor's styling lives in
`src/sase/ace/tui/widgets/_placeholder_highlight.py`
(`PlaceholderHighlightMixin._register_placeholder_text_area_theme`), and uses a split
style:

- `<` and `>` delimiters: theme `accent`, `dim` (flexoki `#9B76C8`)
- inner label: theme `secondary`, `bold` (flexoki `#24837B`)

It only styles spans where the Rust `placeholder_spans()` binding reports `raw=True`.
Spans inside inline code, fenced code, or `%macros_enabled:false` regions come back
`raw=False` and stay unstyled.

The same `highlight_prompt_text(tagified_stash_text(entry.text))` call feeds three
pickers that all share `PromptStashPreviewPane`:

- `src/sase/ace/tui/modals/stash_controller.py` (`_paint_preview`): the Stash tab
- `src/sase/ace/tui/modals/trash_pane.py`: the Trash tab
- `src/sase/ace/tui/modals/update_pinned_stash_modal.py`: the update-pinned-stash modal

`highlight_prompt_text` is also used by
`src/sase/ace/tui/widgets/prompt_panel/_agent_display_prompt_render.py`
(`_render_raw_prompt`), the agent panel's raw-prompt header. The `scope` decision covers
whether that view changes too.

## Goal

A raw placeholder in any stash preview renders with exactly the same delimiter and inner
styles the prompt-bar editor uses. Placeholders in literal zones stay unstyled, and
every other existing highlight stays the same.

## Design

Placeholder detection already lives in `sase-core` (`placeholder_spans`, exposed by
`sase.macro.placeholder_completion`). This change is presentation-only, so no Rust or
binding change is needed.

### 1. One shared placeholder style definition

In `src/sase/macro/highlight_theme.py`, add a small public definition so the editor and
the preview read the same styles:

```python
@dataclass(frozen=True, slots=True)
class PlaceholderHighlightStyles:
    """Split styles for a raw ``<placeholder>``: delimiters and inner label."""

    delimiter: HighlightStyle
    inner: HighlightStyle


def placeholder_highlight_styles(
    *, accent: str | None, secondary: str | None
) -> PlaceholderHighlightStyles:
    return PlaceholderHighlightStyles(
        delimiter=HighlightStyle(accent, dim=True),
        inner=HighlightStyle(secondary, bold=True),
    )


@functools.cache
def ace_placeholder_highlight_styles() -> PlaceholderHighlightStyles:
    """Placeholder styles derived from ACE's pinned theme (``ACE_THEME_NAME``)."""
    # BUILTIN_THEMES[ACE_THEME_NAME] -> placeholder_highlight_styles(accent=..., secondary=...)
```

Add all three names to `__all__`. Leave the existing whole-span
`highlight_theme()["placeholder"]` entry and its `sase macro show` CLI consumer alone.

Refactor `_register_placeholder_text_area_theme` in
`src/sase/ace/tui/widgets/_placeholder_highlight.py` to build its two `rich.style.Style`
values from
`placeholder_highlight_styles(accent=app_theme.accent, secondary=app_theme.secondary)`.
It keeps using the live `app_theme`, so behavior does not change. Use explicit fields,
for example `Style(color=s.delimiter.color, dim=s.delimiter.dim)` and
`Style(color=s.inner.color, bold=s.inner.bold)`, so the registered `syntax_styles`
values stay identical to today's.

### 2. Placeholder overlay in the preview renderer

In `src/sase/ace/tui/util/macro_syntax.py`:

- Add a private
  `_placeholder_overlays_for_source(source: str) -> list[tuple[str, int, int]]`. It
  returns `(rich_style, start, end)` triples:
  - Return `[]` fast when `"<" not in source or ">" not in source`.
  - Call `placeholder_spans(source)` inside `try/except Exception` and return `[]` on
    failure. The overlay must fail open without dropping the rest of the highlighting.
  - Skip spans with `raw=False`.
  - Convert `span.range` and `span.inner_range` (UTF-16 LSP positions) to Python
    character offsets with
    `sase.ace.tui.util.editor_offsets.editor_range_to_offsets( source, dataclasses.asdict(span.range))`.
    `asdict` gives exactly the `{"start": {"line", "character"}, "end": {...}}` dict
    shape that helper expects, and it keeps `util/` from importing `widgets/`. Skip a
    span if either conversion returns `None`.
  - Emit three overlays per span, in the same pattern as the editor mixin:
    `(delimiter, full_start, inner_start)`, `(inner, inner_start, inner_end)`,
    `(delimiter, inner_end, full_end)`. Use
    `ace_placeholder_highlight_styles().delimiter.rich_style` and `.inner.rich_style`.
  - Compute every overlay before mutating anything, as the other overlay helpers do.
- In `_highlight_prompt_text_uncached`, apply these overlays with
  `highlighted.stylize(style, start, end)` as the **last** step, after
  `apply_artifact_ref_overlays`. Applying them last makes a placeholder win over the
  coarse `invocation_arg`/`directive_arg` and artifact-ref styles. For example, `<bar>`
  in `#foo(<bar>)` or `%wait:<x>` gets placeholder styling. This matches the shared
  `_ROLE_PRECEDENCE` in `src/sase/macro/highlight.py`, where `placeholder` (50) beats
  `artifact_ref` (60) and every argument-value role (70+). Code zones need no precedence
  handling because the `raw` filter already excludes them.
- The existing oversize cap (`_exceeds_prompt_highlight_cap`) already guards this path.
  The LRU key needs no new member because the styles derive from the pinned ACE theme.
- Do **not** add placeholders to `macro_overlay_spans` / `apply_macro_overlays`. Those
  also feed the agent-tribe prompt rows (`_agent_tribe_prompts.py`) and are out of
  scope.

> [!decision] scope = prompt_text Apply the overlay unconditionally inside
> `_highlight_prompt_text_uncached`. All four `highlight_prompt_text` callers then get
> placeholder highlighting: the stash, trash, and pinned-stash previews and the agent
> raw-prompt header. No caller changes.

> [!decision] scope = stash Add a keyword-only `placeholders: bool = False` parameter to
> `highlight_prompt_text`, thread it to `_highlight_prompt_text_uncached`, and include
> it in the LRU cache key. Apply the overlay only when it is true. Pass
> `placeholders=True` from every stash preview call site: `_prompt_stash_preview.py`
> (`_build_prompt_stash_preview` fallback), `stash_controller.py` (`_paint_preview`),
> `trash_pane.py`, and `update_pinned_stash_modal.py`. A small shared helper in
> `_prompt_stash_preview.py`, such as `highlight_stash_text(raw) -> Text`, that wraps
> `highlight_prompt_text(tagified_stash_text(raw), placeholders=True)` is preferable to
> repeating the flag at each site. Leave `_agent_display_prompt_render.py` untouched.

### 3. Docs

`docs/macros.md` ("Raw Prompt Placeholders") currently says raw placeholders "are
highlighted in prompt panes". Extend that sentence to say they are also highlighted in
the prompt-stash, trash, and pinned-stash previews. Under `scope = prompt_text`, also
mention the agent panel's raw-prompt header. Keep the existing literal-zone wording.

## Non-goals

- The History tab preview (`_prompt_history_interactions._styled_preview_for_item`)
  renders plain text with project tags only and no Markdown or macro highlighting. Do
  not change it here.
- The `sase macro show` CLI keeps its whole-span placeholder style.
- No change to placeholder detection rules in `sase-core`.
- No new PNG visual snapshot is required. Existing stash snapshot fixtures
  (`tests/ace/tui/visual/test_ace_png_snapshots_prompt_stash.py`) contain no `<...>`
  tags, so their goldens are unaffected.

## Tests

Add to `tests/ace/tui/util/test_macro_syntax.py`, reusing its `_styles_at` /
`_has_style_fragment` helpers and comparing against `ace_placeholder_highlight_styles()`
rich styles instead of hard-coded hex. Under `scope = stash`, pass `placeholders=True`.

1. `Review the <research>.md file` → `<` and `>` carry the delimiter style and
   `research` carries the inner style. `.md` and the surrounding text carry neither.
2. Literal zones stay unstyled: `` `<lit>` `` and a fenced block containing `<code>` get
   no placeholder style, while a raw `<after>` following the fence does.
3. Non-BMP offsets: `😀 <emoji> z` styles exactly `emoji` and its brackets.
4. Placeholder wins inside macro arguments: `#foo(<bar>)` → `bar` carries the inner
   style.
5. Fail-open: monkeypatch `placeholder_spans` (as imported into `macro_syntax`) to
   raise. `highlight_prompt_text` still returns the Markdown/macro-highlighted text (for
   example, `#foo` still has `MACRO_TOKEN_STYLES["invocation"]`), not a plain `Text`.
   Clear `_markdown_text_cache` or use unique text so the LRU does not mask the result.
6. Under `scope = stash` only: without the flag, `<research>` carries no placeholder
   style.

Add to `tests/ace/tui/modals/test_prompt_stash_preview.py`: building a preview for an
entry whose text contains `<research>` gives a `preview.body` whose `research` slice
carries the inner placeholder style.

Add a unit test for `placeholder_highlight_styles` / `ace_placeholder_highlight_styles`
in the existing highlight-theme tests (find them with
`rg -l highlight_theme tests/macro`). Assert the delimiter is accent + dim and the inner
is secondary + bold for the pinned flexoki theme.

Existing editor coverage in `tests/ace/tui/widgets/test_placeholder_completion.py` (the
`placeholder.delimiter` / `placeholder.inner` assertions) and
`tests/ace/tui/widgets/test_prompt_mount_dedup.py` must keep passing unchanged after the
mixin refactor.

## Verification

- Run the repo's standard agent verification (`just check`, run as the lint/test memory
  note directs) and fix any ruff, mypy, or symvision findings. The new public names in
  `highlight_theme.py` must be exported in `__all__` and used.
- Optional manual check: stash a prompt containing `<research>` and open the Prompts
  modal (Stash tab). The preview should render `<research>` with dim-purple brackets and
  a bold-teal label, matching the prompt bar.
