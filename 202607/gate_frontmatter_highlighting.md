---
tier: tale
title: Frontmatter syntax highlighting in gate review documents
goal: 'Markdown documents reviewed in ace TUI gate modals (plan approval, custom gates,
  launch approval) and the TUI''s other raw-markdown surfaces render their YAML frontmatter
  with real syntax highlighting instead of flat plain text, using one shared composite
  lexer that falls back byte-identically to today''s rendering for documents without
  frontmatter.

  '
create_time: 2026-07-19 08:46:05
status: wip
prompt: 202607/prompts/gate_frontmatter_highlighting.md
---

# Plan: Frontmatter syntax highlighting in gate review documents

## Context

When a plan gate opens in `sase ace` (Plan Review / Epic Review), the plan document is rendered with
`Syntax(content, "markdown", theme="monokai")` (`src/sase/ace/tui/modals/plan_approval_modal.py`). Pygments'
`MarkdownLexer` has no concept of YAML frontmatter, so the block the reviewer should read first — `tier:`, `title:`,
`goal:` between `---` fences — renders as flat, unstyled paragraph text while the rest of the document gets full
markdown highlighting. The same flaw exists at every other raw-markdown render surface in the TUI: the custom gate
preview (`src/sase/ace/tui/modals/custom_gate_modal.py`), the launch approval preview
(`src/sase/ace/tui/modals/launch_approval_modal.py`), the xprompt previews
(`src/sase/ace/tui/modals/xprompt_select_modal.py`, `src/sase/ace/tui/modals/xprompt_browser_pane.py`), and everything
flowing through the central `lazy_renderable(content, "markdown")` seam (`src/sase/ace/tui/util/lazy_syntax.py`) —
notably the Artifacts→Plans document viewer and the agent prompt panel, which also display plan files and
frontmatter-led prompt documents.

## Design

### Approach: one composite Pygments lexer, zero layout changes

Render each document through a new composite lexer, `FrontmatterMarkdownLexer`, instead of the `"markdown"` string. The
lexer:

1. Detects a leading frontmatter span using the exact textual rule of the canonical parser
   `sase/sdd/frontmatter.py::parse_frontmatter` (opening `---\n` at byte 0, closing `\n---\n`).
2. Emits both fence lines as `Token.Comment.Preproc` (muted gray under monokai, so the fences recede the way a border
   would).
3. Delegates the enclosed span to Pygments' `YamlLexer` — under the existing monokai theme, keys render pink
   (`#ff4689`), plain scalars purple (`#ae81ff`), and block-scalar text cyan — and delegates the body to
   `MarkdownLexer`, preserving exact source offsets (a working prototype confirmed token-stream reconstruction is
   byte-exact).
4. Falls back to pure `MarkdownLexer` delegation when there is no opening fence or the closing fence is missing, making
   rendering byte-identical to today for every document without frontmatter.

This treatment mirrors how GitHub, Obsidian, and code editors present frontmatter: a YAML island between dim fences,
inside the same document flow. It is one `Syntax` renderable exactly as today — same monokai background, same word wrap,
same scrolling, and the `y` (copy) / `e` (edit) actions still operate on the raw file text.

Deliberate choice: the lexer highlights the YAML block whenever the fences are present, even if the YAML inside fails to
parse or is not a mapping (where `parse_frontmatter` reports no frontmatter). A gate is a review surface — a reviewer
should see broken frontmatter presented as the YAML it was meant to be, and `YamlLexer` is error-tolerant. Parity with
the canonical parser is kept at the span-detection level only.

### Shared span detection (no drift)

Add a small public helper to `src/sase/sdd/frontmatter.py`, e.g. `frontmatter_span(content: str) -> int | None`,
returning the closing-fence index (the current `end` local) or `None`. Refactor `parse_frontmatter` to use it and have
the new lexer call the same helper, so highlighting and parsing can never disagree about what counts as frontmatter.
Behavior of `parse_frontmatter` is unchanged.

### New module: `src/sase/ace/tui/util/frontmatter_syntax.py`

Follows the established idiom of the sibling `src/sase/ace/tui/util/xprompt_syntax.py` (custom Pygments lexer subclass +
module-level singleton, presentation-only, fail-open). Public API, kept minimal for Symvision:

- `FrontmatterMarkdownLexer` — the composite lexer described above (a `pygments.lexer.Lexer` subclass implementing
  `get_tokens_unprocessed` with offset-adjusted delegation).
- A module-level singleton instance (the lexer is stateless), plus a small convenience
  `markdown_document_syntax(content, *, word_wrap=True) -> Syntax` that builds the standard
  `Syntax(..., theme="monokai")` used by the direct-render call sites, deduplicating their repeated boilerplate.

Rich's `Syntax` accepts a Pygments `Lexer` instance (`lexer: Lexer | str`), so no Rich-level changes are needed.

### Wiring

1. **Central seam** — `src/sase/ace/tui/util/lazy_syntax.py`: at the two `Syntax(...)` construction points (the
   `LazySyntaxRenderCache.get` miss path and the uncached path in `lazy_renderable`), resolve `lexer == "markdown"` to
   the composite lexer singleton while continuing to key the cache by the `"markdown"` string (the effective lexer is a
   pure function of the string plus content, so cache keys stay valid). This upgrades the Artifacts→Plans viewer, prompt
   panel, workflow render, and file-panel `.md` previews in one place.
2. **Gate modals (direct `Syntax`)** — swap the `"markdown"` literal for the shared helper in `plan_approval_modal.py`,
   `custom_gate_modal.py`, and `launch_approval_modal.py`. These are the surfaces named by the request.
3. **xprompt previews (direct `Syntax`)** — same one-line swap in `xprompt_select_modal.py` and
   `xprompt_browser_pane.py`; xprompt `.md` files are frontmatter-led, so they get the identical benefit for free.

During implementation, sweep for any remaining direct `Syntax(..., "markdown", ...)` sites in `src/sase/ace/tui/` and
adopt the helper where the content is a whole document. The CLI panels in `src/sase/output.py` are deliberately out of
scope (not a TUI gate surface); they can adopt the same lexer later.

### Performance and boundaries

- Complies with the TUI performance rules: no I/O or new work in render paths. Span detection is a `startswith` (O(1)
  for non-frontmatter docs) plus one substring `find`; the existing markdown highlight caps
  (`MARKDOWN_SYNTAX_HIGHLIGHT_MAX_BYTES` / `..._MAX_LINES`) still bound all lexing; the `LazySyntaxRenderCache` behavior
  and keys are unchanged.
- Rust core boundary: this is presentation-only Rich/Pygments styling in the Python TUI. No domain behavior changes;
  frontmatter _parsing_ stays in the existing Python `sase/sdd/frontmatter.py` helper. Nothing crosses into `sase-core`.
- No keymaps, config values, or `src/sase/default_config.yml` changes.

## Testing

### Unit tests — `tests/ace/tui/util/test_frontmatter_syntax.py` (new)

- Token reconstruction is byte-exact for a realistic plan document (frontmatter with `tier`, `title`, and a folded
  `goal: >` scalar).
- Frontmatter keys produce YAML tokens (`Token.Name.Tag`), fence lines produce `Token.Comment.Preproc`, and body
  headings produce markdown tokens.
- Documents with no frontmatter, an unterminated opening fence, or empty content produce a token stream identical to
  plain `MarkdownLexer`.
- Frontmatter with invalid YAML (or a non-mapping) is still highlighted as YAML — the span rule, not YAML validity,
  decides.
- `frontmatter_span` parity: the lexer engages exactly when the helper reports a span, and the refactored
  `parse_frontmatter` behaves identically to before (existing `tests/test_sdd*.py` / prompt-frontmatter suites must stay
  green untouched).

### Lazy seam tests — extend `tests/ace/tui/util/test_lazy_syntax.py`

- The markdown path constructs `Syntax` with the composite lexer; non-markdown lexer strings pass through unchanged.
- Cache hit/miss accounting and the over-cap plain fallback are unaffected.

### PNG visual snapshots — extend `tests/ace/tui/visual/test_ace_png_snapshots_custom_gate.py`

- New snapshot: a tale plan gate whose fixture plan file carries realistic frontmatter (`tier: tale`, a long `title`,
  folded `goal: >`), pinning the YAML-highlighted frontmatter (e.g. `plan_gate_frontmatter_120x40`).
- New snapshot: a custom gate preview whose `preview_text` starts with frontmatter, pinning the shared treatment on the
  generic gate surface.
- Existing plan-gate goldens (`plan_gate_tale_five_controls_120x40`, `plan_gate_epic_action_120x40`,
  `plan_gate_tale_stacked_90x40`) use frontmatter-free fixtures and must remain byte-identical — this doubles as the
  regression proof that the no-frontmatter path is unchanged.
- Generate new goldens with `--sase-update-visual-snapshots` and visually inspect the rendered PNGs before accepting
  them.

### Verification

Run `just install`, then `just check` and `just test-visual` before completion.

## Risks and mitigations

- **Offset drift between delegated lexers** would corrupt highlighting. Mitigated by the byte-exact reconstruction unit
  test, which fails on any off-by-one in the delegation math.
- **Golden churn**: only the two new snapshots are added; the no-frontmatter fallback keeps every existing golden
  byte-identical. If an existing golden unexpectedly changes, that is a bug in the fallback path, not churn to accept.
- **Cache staleness in `lazy_syntax`**: keys already include a content digest and the lexer string; the effective lexer
  is derived purely from those, so no key change is needed and stale-entry risk is nil.
