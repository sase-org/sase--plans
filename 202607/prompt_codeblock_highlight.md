---
create_time: 2026-07-14 09:33:41
status: wip
prompt: 202607/prompts/prompt_codeblock_highlight.md
tier: tale
---
# Code-Block Syntax Highlighting for the Prompt Input Widget

## Problem

The `PromptTextArea` prompt input highlights xprompt invocations, directives, alt branches, segment separators, and
known slash skills — but code blocks get no styling at all:

- The base tree-sitter markdown grammar (Textual's `markdown.scm`) captures fenced blocks as `@text.literal`, a name
  with no style in any theme we use, and resets `(code_fence_content)` to `@none`. Fenced content renders as plain text.
- Inline code spans (`` `...` ``) are invisible to the block-level markdown grammar entirely (code spans live in the
  separate `markdown_inline` grammar, which Textual does not load).

This matters beyond aesthetics: fenced code blocks are _literal zones_ — the launch path (`protect_fenced_blocks` in
`src/sase/xprompt/_fenced_blocks.py`) guarantees nothing inside them expands. Highlighting should make escaped regions
visually unmistakable, and must never paint xprompt/directive colors inside them.

There is also a real semantics gap hiding under this feature request: **inline code does not escape anything today.**
`` `run #commit here` `` expands `#commit` at launch, and `` `use %m:opus` `` gets its directive stripped. Users
reasonably expect inline code to be literal (it is in every markdown renderer), so this is a live footgun: pasting a
shell snippet or discussing xprompt syntax in backticks can silently launch/strip things. If we style inline code as
"code" while its contents still fire, the highlighting lies in the dangerous direction.

## Goals

1. **Intuitive**: what looks like code is literal; what is literal looks like code. Inline code spans become first-class
   literal zones at launch, matching fenced blocks and user intuition.
2. **Reliable**: one source of truth. All highlighting derives from the same Python lexical mirrors the launch path
   executes (`_fenced_blocks.py` and siblings), never from a parallel reimplementation that can drift. The escape rule
   you see is the escape rule you get — including live feedback: an unclosed ``` fence tints everything below it,
   exactly matching how launch would treat it, and un-tints the moment you close the fence.
3. **Beautiful**: fenced blocks get a subtle background "card" tint with dimmed fences, an accent-italic language tag,
   and real per-language token colors inside (python, bash, json, yaml, toml, …) via tree-sitter injection. Inline code
   gets the same tint language with quiet backtick delimiters. Everything is theme-adaptive (light and dark) and layers
   correctly under the existing search/yank/selection overlays.

## Non-goals

- Multi-line inline code spans. CommonMark technically allows them; we deliberately restrict inline literal zones to a
  single line (see Design §1) for predictability and cross-repo consistency.
- Changing the Rust core fanout planner or editor tokenizer (provably unaffected — see §1 and Follow-ups).
- Restyling the read-only Rich/pygments preview paths (stash modals already sub-lex fenced code via pygments' markdown
  lexer; they inherit the overlay-suppression fix automatically because they call the same `xprompt_inspect.tokenize`).

## Design

### 1. Semantics: inline code spans become literal zones (behavior change — please review)

New scanner in the xprompt lexical-mirror layer, e.g. `src/sase/xprompt/_inline_code.py`:

`inline_code_spans(text, *, masked_ranges=()) -> list[(start, end)]`

Rules (chosen for predictability and zero conflict with existing grammar):

- **Single line only.** A span opens at a backtick run and closes at the nearest same-length backtick run later on the
  _same line_ (CommonMark equal-run matching, so ` `a
  `b`` `` works). Unmatched runs produce no span. Restricting to one line means a span can never contain a`---`separator line or a line-anchored construct, so xprompt-swarm segmentation and the Rust core`plan_agent_launch_fanout`
  planner (which independently scans fenced ranges + line-start directives) need no changes to stay consistent.
- **Opener context mirrors the xprompt marker rule.** A backtick run only opens a span at line start or when preceded by
  whitespace or `([{"'` — the same leading-context class as `#` markers. This structurally protects every
  backtick-quoted colon arg (`` #name:`arg with spaces` ``, `` #git:`branch name` ``, directive colon values): those
  backticks follow `:` and can never open an inline span.
- **Recognized references and directives win.** The scanner skips backtick runs inside `masked_ranges`; callers pass the
  spans of recognized xprompt references (`iter_xprompt_references`) and directives (the `_DIRECTIVE_RE` + arg-end
  mirror), so backticks _inside_ paren args (``#research(compare `a` and `b`)``) stay part of the arg. Fenced blocks and
  `%xprompts_enabled` disabled regions are scanned first and excluded, as today — fenced always wins over inline.

Adoption — extend the two existing literal-zone primitives so every consumer inherits the semantics automatically
instead of migrating ~20 call sites one by one:

- `protect_fenced_blocks` / `unprotect_fenced_blocks` (placeholder substitution used by `processor.py` expansion,
  `_directive_extract/_directive_scan/_directive_alt*`, `_jinja`, history/metadata rewriters, launch validation,
  multi-prompt reference rewriters, …) grows a sibling step that also placeholders inline spans (mask-aware, computed
  after fenced + disabled protection). Composition lives in one helper (e.g. `protect_literal_zones`) so ordering —
  fenced → disabled → refs/directives mask → inline — is written exactly once.
- `fenced_block_ranges`-style offset filtering used by `xprompt_inspect.tokenize`, `alt_inspect.tokenize`,
  `jinja_inspect`, `used_xprompts`, `unresolved`, vcs tag parsing gets a `literal_zone_ranges(text)` counterpart adding
  inline spans to the protected set.

Net user-visible behavior change: `#refs`, `%directives`, and `%{alt|branches}` inside single-line backtick spans no
longer expand/strip/fan-out. This makes the documented mental model ("code blocks escape xprompts") true for inline
code. Flagging explicitly since prompts relying on expansion-inside-backticks (unlikely, and arguably always a bug) will
change.

`memory/xprompts.md` documents "Literal zones: fenced code + disabled regions"; updating it requires explicit user
approval at implementation time (memory-edit rule) — the implementing agent should ask rather than edit.

### 2. Structured fence details for rendering

`fenced_block_ranges` only yields whole-block ranges. Add `fenced_block_details(text)` to `_fenced_blocks.py` returning,
per block: opening-fence run, info-string span (language tag), content range, closing-fence run (None while unclosed).
It must be built from the same line scanner as `fenced_block_ranges` (shared helpers, not a re-parse) so rendering
cannot diverge from launch semantics — including the quirks: `~~~` fences, up-to-3-space indents, ≥3-char fences with
longer closers, unclosed-to-EOF.

### 3. New `CodeBlockHighlightMixin` on `PromptTextArea`

`src/sase/ace/tui/widgets/_codeblock_syntax_highlight.py`, following the established jinja/xprompt mixin pattern
(`_build_highlight_map` override + `_append_highlight_span`, theme registration chained through the existing
`_register_*_text_area_theme` hooks, `_MAX_OVERLAY_BYTES`/`_MAX_OVERLAY_LINES` guards).

Emitted spans:

- `codeblock.fence` — opening/closing fence runs (dim, recede).
- `codeblock.lang` — the info string (accent, italic).
- `codeblock.content` — whole content range (background tint only; foreground untouched so injected token colors and the
  theme's plain text show through).
- `codeblock.inline` — inline span content (same tint family), plus `codeblock.delimiter` for the backtick runs (dim).

MRO placement: rightmost of the highlight mixins (immediately left of `JinjaHighlightMixin`'s position in the super()
chain) so code-block spans append _first_ and every other overlay — jinja, xprompt, search, yank, selection, cursor —
paints above. Since Rich `stylize` merges styles, the content tint (bg-only) composes with injected token colors
(fg-only) and with the built-in markdown captures; no query surgery on Textual's `markdown.scm` is needed.

### 4. Embedded per-language highlighting (the beautiful part)

New helper module (e.g. `src/sase/ace/tui/util/code_injection.py`):

- `language_for_info_string(info)` — first word of the info string, lowered, through an alias map (`py→python`,
  `sh/shell/zsh→bash`, `js→javascript`, `yml→yaml`, `rs→rust`, …), restricted to Textual's `BUILTIN_LANGUAGES` (python,
  markdown, json, toml, yaml, html, css, javascript, rust, go, regex, sql, java, bash, xml — all shipped via
  `textual[syntax]`).
- `injected_highlights(language, code)` — parse the content substring with the matching `tree_sitter_<lang>` parser and
  run Textual's own builtin highlight query for that language (mirror how
  `SyntaxAwareDocument`/`_get_builtin_highlight_query` load them), yielding per-line byte-offset spans with **standard
  capture names** (`keyword`, `string`, `comment`, `number`, `function`, …). Because content lines start at column 0 of
  the extracted substring, rows/columns map 1:1 back into the document by offsetting the starting row.

Emitting standard capture names is the key trick: they resolve through the active `TextAreaTheme.syntax_styles` that
already styles the rest of the prompt, so embedded code automatically matches the widget's look in every theme with zero
new palette. Unknown, absent, or unparseable languages degrade gracefully to the tint-only literal look; injection
failures are fail-open (bare `except` → skip tokens, keep tint), same philosophy as `highlight_prompt_text`.

Performance (per `memory/tui_perf.md` rules — this runs on every keystroke):

- Module-level LRU memoization keyed `(language, content)` (~32 entries) so only the block being edited re-parses;
  parsers/queries themselves cached per language.
- Respect the existing overlay byte/line caps, plus a per-block injection cap (skip token injection, keep the tint, for
  pathological blocks).
- Everything is synchronous-but-tiny: scanning is O(n) like `tokenize`; a sub-parse of a typical fenced block is well
  under a millisecond. Extend the tokenizer benchmark in `tests/perf/bench_tui_trace.py` with a code-heavy prompt to
  hold the p95 < 16 ms line.

### 5. Theme-adaptive style registration

Register in the mixin's theme hook (chained like `_register_xprompt_text_area_theme`), derived from `app.current_theme`
so light/dark both work:

- `codeblock.content` / `codeblock.inline`: `bgcolor = blend(background → foreground, ~7%)` — a quiet card tint that
  reads as "escaped/literal" without shouting; slightly stronger for inline so short spans stay perceptible.
- `codeblock.fence` / `codeblock.delimiter`: `blend(foreground → background, ~45%)` — fences and backticks recede;
  content is the star.
- `codeblock.lang`: accent + italic, echoing the existing `xprompt.skill` accent family.
- No fg color on content/inline: plain text keeps theme foreground; fenced token colors come from injection. The tint is
  the shared visual language marking "xprompts cannot fire here", and — after §1 — that statement is _true_ everywhere
  it appears.

### 6. Overlay suppression inside code blocks

`xprompt_inspect.tokenize`, `alt_inspect.tokenize`, and `jinja_inspect` already exclude fenced

- disabled ranges; after §1 they exclude inline spans too. Net: no invocation/directive/ separator/skill/alt/jinja
  styling inside any code block, in the live widget _and_ in the read-only stash/history previews (same tokenizer). The
  `known_skills` slash-span path is covered by the same protected-range check.

## Testing

- **Scanner unit tests** (new `tests/test_xprompt_inline_code.py` + extensions to `tests/test_xprompt_inspect.py`):
  equal-run matching, unmatched runs, opener contexts, single-line restriction, colon-arg backticks untouched, paren-arg
  masking, fenced-beats- inline, disabled-region interaction, tokenize exclusion for every span kind.
- **Launch-semantics tests** (processor/directive suites: `test_xprompt_processor_pattern.py`, directive
  extract/scan/alt tests): `#ref`/`%directive`/`%{alt}` inside inline code survive expansion verbatim; regression
  coverage that `` #name:`arg with spaces` `` and vcs backtick refs still parse; swarm separators unaffected.
- **Widget tests** (new `tests/ace/tui/widgets/test_prompt_codeblock_highlight.py`, modeled on
  `test_prompt_xprompt_highlight.py`): span names + theme styles registered, injection token names present for a python
  block, xprompt overlay absent inside blocks, unclosed-fence live-tint behavior, cap fallbacks.
- **PNG visual snapshots** (extend `tests/ace/tui/visual/test_ace_png_snapshots_prompt_stack.py` or a sibling): a prompt
  mixing fenced python + bash, inline code, and live xprompt tokens outside the blocks — light and dark, solo and
  stacked, pinned with the existing `just test-visual` goldens.
- **Perf**: code-heavy case in `tests/perf/bench_tui_trace.py`.

`just check` before completion, per repo rules.

## Follow-ups (explicitly out of scope, candidates for beads)

- Mirror inline-code literal zones into `sase-core` (`agent_launch` fenced scanner and the `editor/` tokenizer) and the
  sase-nvim syntax files so editor frontends match; not required for correctness now (single-line spans cannot affect
  the Rust planner's line-anchored scans), but the boundary rule wants the canonical grammar in core eventually.
- `memory/xprompts.md` literal-zones wording update (needs explicit user approval).
- Optional: richer code styling in the read-only Rich preview paths (theme parity with the live widget).
