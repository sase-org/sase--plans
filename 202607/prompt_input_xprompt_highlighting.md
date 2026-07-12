---
create_time: 2026-07-12 16:24:34
status: wip
prompt: 202607/prompts/prompt_input_xprompt_highlighting.md
tier: tale
---
# Plan: Live xprompt Syntax Highlighting in the Prompt Input Widget

## Problem & Motivation

The `sase ace` TUI has excellent xprompt syntax highlighting in the prompt stash preview pane (`highlight_prompt_text`
in `src/sase/ace/tui/util/xprompt_syntax.py`): `#xprompt` invocations glow green, `%directives` gold, `%{a | b}` alt
fan-outs purple, and `---` swarm separators blue — with fenced code blocks and `%xprompts_enabled:false` regions
correctly left alone.

The prompt input widget (`PromptTextArea`), where users actually _compose_ these prompts, has none of this. It
highlights Jinja (`JinjaHighlightMixin`), the `%{...}` alt shorthand (`AltSyntaxHighlightMixin`), and search matches
(`SearchHighlightMixin`) — but `#gh:sase`, `#pr:my_change`, `%auto`, `%m:opus`, and `---` all render as plain markdown
text while typing. This is exactly where highlighting has the most value: instant feedback that a marker was recognized
(and will expand at launch) is the difference between confidence and silently shipping a typo'd directive as literal
prompt text.

## Current State (two highlighting systems)

1. **Read-only preview** (`src/sase/ace/tui/util/xprompt_syntax.py`): Rich `Syntax(text, "markdown")` + `Text.stylize()`
   overlays computed by `_apply_xprompt_overlays`, using the _same lexical parsers the launcher uses_:
   - invocations: `iter_xprompt_references` (`src/sase/xprompt/_parsing_references.py`)
   - directives: `_DIRECTIVE_PATTERN` + `_KNOWN_DIRECTIVES`/`_DIRECTIVE_ALIASES`
     (`src/sase/xprompt/_directive_types.py`)
   - alt expressions: `_ALT_DIRECTIVE_RE` (`%{...}`, `%(...)`, `%alt(...)` forms)
   - separators: `_SEGMENT_SEPARATOR_RE` (`src/sase/xprompt/segment_separators.py`)
   - protection: `fenced_block_ranges` + `disabled_region_ranges` Styles are the fixed-hex `XPROMPT_TOKEN_STYLES`
     palette. Used by the stash preview pane, `stashed_prompts_modal.py`, and `update_pinned_stash_modal.py`.

2. **Editable input** (`src/sase/ace/tui/widgets/prompt_text_area.py`): Textual `TextArea` (`language="markdown"`,
   tree-sitter base highlighting) plus overlay mixins that override `_build_highlight_map()`, call `super()`, then
   append `(start_byte, end_byte, style_name)` spans into `self._highlights[row]` via the shared
   `_append_highlight_span` / `_line_byte_spans` helpers (`_jinja_highlight.py`). Overlay styles are registered as named
   `syntax_styles` on one shared `TextAreaTheme` (`sase-jinja-prompt`), derived from the _app theme_
   (`accent`/`success`/`warning`/`error`), and re-registered on theme change via the `_app_theme_changed` /
   `_register_jinja_text_area_theme` chain. Size guards: `_MAX_OVERLAY_BYTES = 80_000`, `_MAX_OVERLAY_LINES = 1_200`.
   The input's per-token analyzers live in frontend-agnostic inspect modules: `sase/xprompt/alt_inspect.py` and
   `sase/xprompt/jinja_inspect.py`.

The asymmetry: the token rules exist only as Rich-`Text` stylize calls (preview), and the input widget's overlay
plumbing exists but has no xprompt tokenizer to feed it.

## Design Goals

- **Intuitive**: the same tokens that light up in the stash preview light up live as you type, with the same hue
  semantics (green = invocation, gold = directive, purple = alt, blue = swarm separator). Recognized markers visibly
  "activate"; unrecognized ones stay plain — implicit typo feedback with zero false alarm risk.
- **Reliable**: one tokenizer feeds both the preview and the input widget, and it is built from the exact parsing
  primitives the launch path uses (`iter_xprompt_references`, the directive pattern/known-set, the segment separator
  regex, fence/disabled-region protection). Highlighting can never drift from launch behavior, and preview can never
  drift from input. Fail-open everywhere: guards and exceptions degrade to unstyled text, never to hidden text.
- **Beautiful**: input styles are app-theme-derived (like the Jinja/alt overlays already are), so they harmonize with
  every theme instead of hardcoding hex values into an editable widget; hue mapping mirrors the preview palette so the
  two surfaces read as one system.
- **Fast**: all work rides the existing per-edit `_build_highlight_map()` pass — no new refresh paths, timers, I/O, or
  event-loop work. Same byte/line guards as the sibling overlays, plus cheap early-outs. Compiled-regex linear scans
  over an ≤80KB buffer are microseconds-scale.

## Architecture

### 1. Shared frontend-agnostic tokenizer: `src/sase/xprompt/xprompt_inspect.py`

New module mirroring the `alt_inspect` / `jinja_inspect` convention:

- `XPromptSpanKind = Literal["invocation", "invocation_arg", "directive", "directive_arg", "separator"]`
- `@dataclass(frozen=True, slots=True) class XPromptSpan: start, end, kind` (character offsets)
- `tokenize(text: str) -> list[XPromptSpan]`:
  - Early-out families: skip invocation scanning when `"#" not in text`, directive scanning when `"%" not in text`,
    separator scanning when `"---" not in text`.
  - Compute protected ranges once (`fenced_block_ranges` + `disabled_region_ranges`) and drop any overlapping span —
    same `_overlaps_protected` semantics as the preview today.
  - Invocations via `iter_xprompt_references`: marker+name (+HITL suffix) → `invocation`, argument source →
    `invocation_arg`. This automatically covers `#name`, `#!name`, `#ns/name`, `(...)`, `:arg`, `` :`quoted` ``, `+`,
    `: text` / `:: text` shorthands, and VCS workspace refs (`#gh:sase`, `#git:home`) since they match the same lexical
    grammar.
  - Directives via the existing pattern + alias/known-set filtering and the paren-arg end-extension
    (`find_matching_paren_for_args`): name → `directive`, args → `directive_arg`. Unknown directives stay unstyled
    (parity with preview; see Rejected Alternatives).
  - Separators via `_SEGMENT_SEPARATOR_RE` → `separator`.
  - Alt expressions are deliberately **not** in this module — `alt_inspect` owns them (see §3).

Refactor `xprompt_syntax._apply_xprompt_overlays` to consume `xprompt_inspect.tokenize` for
invocation/directive/separator spans (mapping kind → `XPROMPT_TOKEN_STYLES`). Pure refactor; the preview's rendered
output must not change. The existing `tests/ace/tui/util/test_xprompt_syntax.py` suite is the behavioral spec that
proves that.

Rust-core boundary note: like `alt_inspect`, this is presentation-layer span computation that mirrors grammar whose
canonical home is the Rust core. No behavior/domain change → no `sase-core` change; the new module's docstring should
state this mirroring contract explicitly.

### 2. New overlay mixin: `src/sase/ace/tui/widgets/_xprompt_syntax_highlight.py`

`XPromptSyntaxHighlightMixin`, modeled closely on `AltSyntaxHighlightMixin`:

- `_build_highlight_map()`: call `super()`, apply the shared `_MAX_OVERLAY_BYTES` / `_MAX_OVERLAY_LINES` guards and a
  cheap `#`/`%`/`---` presence check, then append `xprompt.<kind>` spans via the existing `_append_highlight_span`
  (which handles char→per-row byte-span translation and multi-line spans).
- Theme registration hooks into the same chain the alt mixin uses (`on_mount`, `_app_theme_changed`,
  `_register_jinja_text_area_theme`) so all overlay families keep sharing the single `sase-jinja-prompt` `TextAreaTheme`
  and stay live across app-theme switches.
- Style mapping (app-theme-derived, hue-consistent with the preview palette):
  - `xprompt.invocation` → success, bold (preview: `bold #87D787`)
  - `xprompt.invocation_arg` → success, non-bold (preview: `#5FAF87`)
  - `xprompt.directive` → warning, bold (preview: `bold #FFD75F`)
  - `xprompt.directive_arg` → warning, non-bold (preview: `#D7AF5F`)
  - `xprompt.separator` → secondary/accent-family, dim bold (preview: `dim bold #87AFFF`) Exact bold/dim tuning is
    finalized against the PNG snapshots — the requirement is that name vs. arg are visibly distinct within the same hue
    family, and every style stays readable on the built-in themes (check at least the default dark theme plus one light
    theme).
- MRO placement in `PromptTextArea`: insert **between** `SearchHighlightMixin` and `JinjaHighlightMixin` so the per-row
  append order (reverse MRO) becomes: tree-sitter markdown → jinja → **xprompt** → search → alt. Search match/current
  spans must stay above xprompt token colors so find-in-prompt never loses its target; alt stays the top overlay exactly
  as today.
- Interplay: directives _inside_ alt branches (`%{%m:opus | %m:sonnet}`) get directive styling while the braces/pipes
  keep their alt styling — same layered result the preview produces. The mixin never styles alt delimiters itself (no
  double-painting).
- Applies uniformly in all prompt-bar modes (prompt, feedback, approve-prompt) and to every pane of a multi-prompt
  stack, matching how the Jinja/alt overlays behave. Per-pane text is what the user sees in that pane, so per-pane
  tokenization is correct by construction.

### 3. Alt paren-form parity (adjacent gap, kept as its own milestone)

Today the preview highlights all three alt spellings (`%{...}`, `%(...)`, `%alt(...)`) but the input's
`alt_inspect.tokenize` only recognizes `%{...}` — so `%alt(a | b)` gets nothing while typing. Extend
`alt_inspect.tokenize` to the paren forms (mirroring `_ALT_DIRECTIVE_RE` and `find_matching_paren_for_args`, which it
already documents mirroring from the Rust core), then migrate the preview's alt handling in `xprompt_syntax.py` onto
`alt_inspect.tokenize`. Net effects: input highlights all alt spellings; preview _gains_ branch-name and
unmatched-opener error styling it currently lacks; one alt grammar mirror remains instead of two.

`XPROMPT_TOKEN_STYLES` grows entries for the new alt kinds surfaced in the preview (`branch_name`, `error`) alongside
the existing `alt_delimiter`.

## Performance Analysis & Verification

Per the TUI perf rules: no synchronous I/O, subprocesses, or new event-loop work — `tokenize` is pure compiled-regex +
linear scans, invoked only inside `_build_highlight_map()`, which Textual already runs synchronously on document edits
(not on cursor movement). The added cost is additive with small constants on top of the existing tree-sitter + jinja +
alt passes, bounded by the same 80KB/1200-line guards and skipped entirely for token-free text. No caching layer is
warranted at these sizes; matching the sibling overlays' shape is the point.

Verification steps (before/after):

- `sase ace --profile` typing session in the prompt bar; confirm `_build_highlight_map` remains negligible relative to
  frame budget.
- Add tokenizer timing to the existing slow-marked bench surface (print-only p50/p95 table, same style as
  `tests/perf/bench_tui_trace.py`) at guard-limit input size (80KB), so regressions are measurable without flaky
  wall-clock asserts in CI.
- `SASE_TUI_PERF=1` sanity check that prompt-bar typing stays within the p95 < 16 ms paint target.

## Testing

- **Tokenizer unit tests** (`tests/xprompt/`): span kinds/offsets for every invocation argument form, directive aliases
  and unknown-directive skips, separators, fence/disabled-region protection, `# heading` / mid-word negatives,
  empty/huge inputs. Reuse the scenario matrix from `tests/ace/tui/util/test_xprompt_syntax.py`.
- **Preview refactor safety**: existing `test_xprompt_syntax.py` must pass unchanged (Milestone 1 is
  behavior-preserving); extend it for the new alt branch-name/error styling in Milestone 3.
- **Widget tests** (`tests/ace/tui/widgets/test_prompt_xprompt_highlight.py`, mirroring
  `test_prompt_alt_syntax_highlight.py`): `xprompt.*` spans land in `ta._highlights`; styles registered on the shared
  theme; coexistence with jinja + alt + search overlays; fenced-block suppression; oversized-buffer skip; theme-switch
  re-registration.
- **PNG visual snapshots** (`tests/ace/tui/visual/test_ace_png_snapshots_prompt_stack.py`): new snapshot(s) of the
  prompt bar containing a representative composite prompt — e.g. `#gh:sase %auto #pr:my_change %m:opus fix the bug` plus
  a fenced block and a `---` — in both solo and multi-pane stack layouts. This is also the beauty gate: judge the
  rendered palette here and tune bold/dim before accepting goldens.

## Milestones

1. **Shared tokenizer + preview refactor** — add `xprompt_inspect.py`; rewire `xprompt_syntax._apply_xprompt_overlays`
   invocation/directive/separator handling onto it; tokenizer unit tests; existing preview tests green unchanged.
2. **Input widget overlay** — `XPromptSyntaxHighlightMixin` + MRO insertion + theme styles; widget tests; PNG snapshots;
   perf verification.
3. **Alt paren-form parity** — extend `alt_inspect.tokenize` to `%(...)`/`%alt(...)`; migrate preview alt handling onto
   it; style-table additions; tests for both surfaces.

Each milestone is independently shippable; 3 can be dropped without weakening 1–2.

## Non-Goals / Future Work

- **Other read-only surfaces**: the xprompt browser preview, xprompt select modal preview, agent detail prompt panel,
  and prompt history preview render plain markdown today. After Milestone 1 they can adopt the shared
  tokenizer/`highlight_prompt_text` cheaply, but each has its own rendering/caching context (e.g.
  `LazySyntaxRenderCache`) — deferred to a follow-up CL.
- **Unknown-directive / unknown-xprompt warning styles**: styling `%typo` or `#nonexistent` with a warning underline
  needs the (async-warmed) xprompt catalog to avoid false positives on plain prose (`50 %off`) and cold-start flicker.
  Deferred; the known-set gate gives implicit feedback already (recognized tokens light up, typos don't).
- **`@{{ file }}` include highlighting**: no parser support exists anywhere yet (Jinja overlay styles the `{{ }}` part);
  out of scope.
- **Rust core changes**: none — presentation-layer only, mirroring existing canonical grammar.

## Rejected Alternatives

- **Tree-sitter `highlight_query` extension**: xprompt grammar is not a tree-sitter language, and invocation/directive
  ends depend on paren-matching and shorthand rules that regex+scan already implement; the overlay-mixin pattern is the
  established, theme-integrated path.
- **Reusing `highlight_prompt_text` (Rich `Text`) inside the TextArea**: TextArea renders from its document +
  `_highlights` map, not a Rich `Text`; converting per keystroke would fight the widget instead of extending it.
- **Fixed-hex colors in the input widget**: would match the preview byte-for-byte but break on light/custom app themes
  and diverge from the Jinja/alt overlay convention; hue-consistent theme-derived styles keep both promises.
- **Highlighting unknown directives as errors in v1**: false-positive risk on ordinary `%`-prose; revisit with
  catalog-aware validation (see Future Work).
