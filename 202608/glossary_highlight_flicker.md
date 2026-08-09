---
tier: tale
title: Stop glossary highlight flicker in the prompt input widget
goal:
  Glossary term styling stays steady on every keystroke instead of blinking off and back
  on.
proposed_by: bbugyi200.athena.wf
create_time: 2026-08-09 09:06:14
status: wip
---

# Plan: Stop glossary highlight flicker in the prompt input widget

## Problem

Glossary term styling (bold theme-accent + underline) in the ACE prompt input widget
blinks off and back on with every character typed.

## Root cause

`PromptGlossaryMixin._build_highlight_map()` drops the entire glossary overlay on the
highlight rebuild that immediately follows every edit, because its warm-catalog lookup
is gated on a text-equality check that is guaranteed to be stale at that moment.

`src/sase/ace/tui/widgets/_prompt_glossary.py:176-183`:

```python
def _warm_prompt_glossary_catalog_for_render(self) -> EditorGlossaryCatalog | None:
    """Return the already-warm catalog without scheduling work."""
    context = self._prompt_glossary_context_cache
    if context is None or self._prompt_glossary_context_text != self.text:
        return None
    return self._get_prompt_glossary_catalog(context, schedule=False)
```

The per-keystroke ordering is:

1. Textual's `TextArea.edit()` clears `_line_cache` and calls `_build_highlight_map()`
   **before** posting `Changed` (`.venv/.../textual/widgets/_text_area.py:1567-1568`).
2. At that point `_prompt_glossary_context_text` still holds the **pre-edit** text, so
   `_warm_prompt_glossary_catalog_for_render()` returns `None` and
   `_build_highlight_map()` returns early without appending any `glossary.term` span.
   The frame that renders has no glossary styling.
3. `Changed` is processed later on the message pump: `on_text_area_changed` ->
   `_on_prompt_completion_context_changed` -> `_refresh_prompt_glossary_context` resyncs
   `_prompt_glossary_context_text`, but **nothing rebuilds the highlight map**, so the
   overlay stays off.
4. The overlay only returns when some unrelated later rebuild happens to fire ---
   `_refresh_jinja_matching_delimiters`, the artifact-ref worker completion, a prompt
   catalog snapshot apply (`_refresh_visible_prompt_catalog_surfaces`, throttled to
   roughly 1/s by `_schedule_prompt_catalog_token_fallback_check`), search or yank
   refreshes. That irregular repaint is the "on" half of the flicker.

Reproduced against the real edit path (scratch test, since removed):

```
BEFORE TYPING:                                  [(0, 4, 14, 'glossary.term')]
IMMEDIATELY AFTER insert() (the rendered frame): []
AFTER message pump (Changed processed):          []
AFTER an explicit later rebuild:                [(0, 4, 14, 'glossary.term')]
```

### Why it was not caught

Every test in `tests/ace/tui/widgets/test_prompt_glossary.py` calls
`ta._refresh_prompt_glossary_context(schedule=False)` immediately before
`ta._build_highlight_map()`. That hand-synchronization is exactly what real typing never
performs, so no test drives the actual edit path.

### Why it surfaced now

The defect has existed since `bb07bd865` ("feat(ace): add prompt glossary
interactions"). `c2c8e883d` ("feat(ace): underline glossary terms in prompt") added the
underline, which made the pre-existing flicker visually obvious.

### Design intent being preserved

`bb07bd865`'s goal was to keep glossary _catalog loading and context computation_ off
the keypress and render paths. That goal is correct and must be preserved; the
text-equality guard was an over-broad way to express it, and it disabled the overlay
rather than merely deferring work. The glossary context selects a **project/workspace**,
not a text snapshot, so it does not need to be byte-identical to the current buffer to
be usable.

### Sibling overlays already do this correctly

`MisspellingHighlightMixin`, `XPromptSyntaxHighlightMixin`, and
`ArtifactRefHighlightMixin` all scan the **current** `self.text` on the render path and
consult warm state keyed by project, degrading gracefully instead of blanking:

- `_misspelling_spans_for_document` recomputes when its cached text differs, rather than
  bailing out (`src/sase/ace/tui/widgets/_misspelling_highlight.py:79-95`).
- `_get_warm_artifact_ref_known_kinds` returns `None` when cold and the overlay still
  paints `artifact_ref.neutral`
  (`src/sase/ace/tui/widgets/_artifact_ref_highlight.py:193-197`).

`PromptGlossaryMixin` is the sole outlier.

## Approach

Decouple the render path from text identity, repaint when the context actually changes,
and memoize the scan.

1. **Render path uses the last-known-good context.** Drop the
   `self._prompt_glossary_context_text != self.text` comparison from
   `_warm_prompt_glossary_catalog_for_render`. The render path keeps using the cached
   context (still never computing a new one and still never scheduling a warm), and the
   scan always runs against the current `self.text`. This alone removes the flicker.

2. **Context changes trigger a repaint.** In `_refresh_prompt_glossary_context`, compare
   the newly computed context with the cached one; when it differs, rebuild the
   highlight map and refresh. This runs on the `Changed`/`SelectionChanged` handler, off
   the render path, and covers the one case the removed guard was implicitly handling:
   the user edits a leading VCS reference so a different project's glossary applies.
   Worst case is now a single frame painted with the previous project's catalog instead
   of a per-keystroke blank.

3. **Memoize the scan.** `scan_glossary_spans` currently reruns on every highlight
   rebuild, and a rebuild can happen more than once per keystroke. Add a spans cache
   keyed by the buffer text plus the identity of the compiled catalog, mirroring
   `MisspellingHighlightMixin._misspelling_spans_for_document`. Invalidate whenever
   either key component changes.

`_prompt_glossary_context_text` is retained only if step 2 still needs it; otherwise
remove it along with its `TYPE_CHECKING` declaration so no stale-text state remains.

### Rejected alternatives

- **Compute the context on the render path.** Simplest, and `ArtifactRefHighlightMixin`
  already resolves the project from text during render, but it contradicts `bb07bd865`'s
  explicit design goal and puts `_preview_context()` (`os.getcwd()`) plus project
  resolution into every highlight rebuild.
- **Rebuild the highlight map from the `Changed` handler unconditionally.** Restores the
  overlay one frame late instead of never, so the flicker becomes a shorter flicker
  rather than disappearing. It also costs an extra full rebuild per keystroke.

## Implementation steps

1. Edit `src/sase/ace/tui/widgets/_prompt_glossary.py`:
   - Remove the text-equality guard from `_warm_prompt_glossary_catalog_for_render` and
     update its docstring to state that it returns the catalog for the last resolved
     context without scheduling work.
   - Add context-change detection plus `_build_highlight_map()` + `refresh()` to
     `_refresh_prompt_glossary_context`. Guard the repaint so it is skipped before the
     widget is mounted (`TextArea.__init__` calls `_build_highlight_map()` on an
     unmounted widget, where `self.app` raises --- see the docstring on
     `MisspellingHighlightMixin._active_app`).
   - Add the `(text, compiled-catalog)`-keyed spans cache and use it in
     `_build_highlight_map`.
   - Drop `_prompt_glossary_context_text` if step 2 leaves it unused.
   - Update the `PromptGlossaryMixin` class docstring to describe the warm-context /
     current-text split.

2. Add regression tests to `tests/ace/tui/widgets/test_prompt_glossary.py`. These must
   NOT call `_refresh_prompt_glossary_context` by hand before asserting:
   - **Typing keeps the overlay**: load text containing a glossary term, let the context
     settle, then drive a real edit (`ta.insert(...)` or `pilot.press(...)`) and assert
     `glossary.term` spans are present in `ta._highlights` on the frame immediately
     after the edit, with offsets shifted correctly. This test fails before the fix.
   - **Multi-character typing**: type several characters in a row and assert the overlay
     is present after each one.
   - **Context change repaints**: change the resolved project context and assert the
     highlight map is rebuilt with the new catalog's spans without an external trigger.
   - **Scan memoization**: count `scan` calls on the fake compiled catalog and assert
     repeated `_build_highlight_map()` calls on unchanged text scan once, and that a
     text change or a catalog swap invalidates the cache.
   - Note that `_FakeCompiledGlossary.scan` ignores its `text` argument and returns
     fixed spans; extend or add a fake that scans dynamically where a test needs offsets
     to track the edited text.

3. Keep the existing tests passing unchanged --- they assert overlay ordering
   (`glossary.term` vs `codeblock.inline`, `search.current`, `spell.misspelled`) and
   rendered style attributes, none of which this change should alter.

## Verification

- `just install`, then `just check` (`_prompt_glossary.py` is not in the broadening set;
  escalate to `just check-full` if the scoped run reports an unusual selection).
- `.venv/bin/python -m pytest tests/ace/tui/widgets/test_prompt_glossary.py -q` for the
  focused suite.
- `just test-visual` ---
  `tests/ace/tui/visual/test_ace_png_snapshots_prompt_highlighting.py` covers
  `prompt_glossary_highlight_{dark,light}_120x40.png`. Those goldens are captured
  through a settled render path, so they should be unchanged. If they do shift, treat
  that as a signal the fix altered steady-state styling and investigate rather than
  accepting the snapshot.
- Manual confirmation in `sase ace`: type a sentence containing a glossary term (for
  example `Agent Clan`) into the prompt and confirm the underline and accent styling
  stay steady on every keystroke.

## Out of scope

- Changing the glossary style itself (color, bold, underline).
- Overlay precedence between glossary and other highlight layers.
- The off-thread glossary warm pipeline in
  `src/sase/ace/tui/actions/_startup_prompt_catalog.py`; the cold-to-warm transition
  already repaints via `_refresh_visible_prompt_glossary_surfaces`.
- Auditing sibling overlays for unrelated flicker; none share this bail-out pattern.
