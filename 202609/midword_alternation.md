---
tier: epic
title: Mid-word alternation (%{...}) everywhere
goal: '`%{a | b}` fans out, highlights, and edits the same way wherever it appears:
  at a word boundary, in the middle of a word (`foo%{bar | baz}qux`), right after
  punctuation, or nested inside another branch. One Rust-owned scanner feeds launch,
  the TUI prompt input, and the xprompt LSP, and nothing is added to the per-keystroke
  cost of typing.

  '
phases:
- id: core-grammar
  title: Core launch grammar for mid-word and nested alternation
  depends_on: []
  size: medium
  description: 'core-grammar: in sase-core, let `%{` open anywhere outside literal
    zones while `%(`/`%alt(` keep the boundary rule. Also fix the nested-alternation
    panic, keep glued `%` directives parseable, stop the model shortcut gluing onto
    an opener, and add tests.'
- id: core-scan-lsp
  title: Shared alternation scanner, Python binding, and LSP highlighting
  depends_on:
  - core-grammar
  size: medium
  description: 'core-scan-lsp: in sase-core, add one alternation scanner with its
    wire record and a code-point-offset Python binding. Add an unclosed-alternation
    editor diagnostic and xprompt LSP semantic tokens with new stable modifiers, plus
    unit and JSON-RPC tests.'
- id: sase-grammar-highlight
  title: sase grammar mirror, highlight adapter, pin, and docs
  depends_on:
  - core-scan-lsp
  size: medium
  description: 'sase-grammar-highlight: in sase, bump the core pin, relax `_ALT_DIRECTIVE_RE`
    for the brace form, and turn `alt_inspect.tokenize` into a memoized thin adapter
    over the binding. Also route project-tag alt groups through that adapter, add
    correctness and performance tests, and document the rule in `docs/xprompt.md`.'
- id: tui-alt-editing
  title: TUI prompt input editing for mid-word alternation
  depends_on:
  - sase-grammar-highlight
  size: medium
  description: 'tui-alt-editing: in sase, make brace padding and `|` separator normalization
    recognize mid-word openers. Ignore openers in literal zones, keep an unclosed
    span to its own line, and let the innermost nested span win. Stop Jinja auto-pairing
    right after `%{`, then update tests and `docs/ace.md`.'
- id: nvim-lsp-highlight
  title: sase-nvim alternation highlighting from LSP tokens
  depends_on:
  - core-scan-lsp
  size: medium
  description: 'nvim-lsp-highlight: in sase-nvim, replace the Lua copy of the alternation
    grammar with an overlay driven by LSP semantic tokens that keeps the `SaseAlt*`
    groups. Apply the new opener rule to `alt_edit.lua`, and update the tests and
    the README.'
proposed_by: bbugyi200.athena.0u1
create_time: 2026-09-29 16:21:59
status: wip
bead_id: sase-1co
---

- **PROMPT:** [prompts/202609/midword_alternation.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/midword_alternation.md)
- **BEAD:** [sase-1co](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1co/README.md)

# Plan: Mid-word alternation (`%{...}`) everywhere

## Background and verified current behavior

Today an alternation opener is recognized only at a directive-valid position: start of
line, or directly after whitespace, `(`, `[`, `{`, `"`, `'` or `:`. The same rule is
copied in four places, which must all agree:

- **Rust (authoritative).** `alt_directive_re()` in the sase-core crate file
  `crates/sase_core/src/agent_launch/directive_scan.rs`:
  `(?m)(^|[\s\(\[\{"':])(%(?:alt)?\(|%\{)`. It feeds `alt_directive_starts`, which is
  used by:
  - launch fan-out (`agent_launch/fanout.rs`);
  - launch literal-zone masking;
  - `alt_inner_ranges`;
  - `editor/alternation.rs`, which in turn feeds model-alias shortcuts and project-tag
    accept.
- **Python mirror.** `_ALT_DIRECTIVE_RE` in `src/sase/xprompt/_directive_alt.py`, used
  by:
  - `has_alt_directive` and `_plan_prompt_fanout`;
  - `_directive_collect._alt_inner_regions`;
  - `_directive_shorthand._alt_inner_ranges`;
  - `_directive_edit_core.find_alt_inner_regions`;
  - `directive_diagnostics`;
  - `jinja_inspect._alt_jinja_overlap_ranges`;
  - `alt_inspect.tokenize`, which is the highlighter for the TUI prompt input, previews,
    `highlight_spans`, and `semantic_overlay`.
- **TUI editing.** `_is_directive_valid_brace_opening` / `_DIRECTIVE_OPENING_CONTEXTS`
  in `src/sase/ace/tui/widgets/_alt_syntax_editing.py`, which handles brace padding and
  `|` normalization.
- **sase-nvim.** `VALID_PREFIX` in `lua/sase/alt_highlight.lua` (a Lua regex
  highlighter) and `lua/sase/alt_edit.lua`.

So `foo%{bar | baz}qux` currently gets no fan-out, no highlight, and no separator
editing. Exploring also turned up these defects, each reproduced against the current
binding:

1. **Nested alternation panics the Rust planner.**
   `split_prompt_for_alternatives("a %{x %{p|q} | y} b")` raises
   `PanicException: range end index 17 out of range`, because
   `split_prompt_for_alternatives_with_ids` plans both the outer and the inner
   directive, and `render_alternative_prompt` then replaces overlapping ranges.
   `%{foo%{x|y}|b}` is safe today only because the inner opener is ignored. Mid-word
   support makes that case live.
2. **Highlight and launch disagree.** `alt_inspect.tokenize("x %{don't | do} it")` shows
   no separator, because Python's `_top_level_offsets` treats `'` as a quote. Launch
   splits it into `x don't it` and `x do it`. Prose written inside a word will hit
   apostrophes often.
3. **False migration error.** `foo%{%m:opus | %m:sonnet} go` raises
   "`%m:opus ... %m:sonnet is no longer supported`" during Python directive extraction,
   because the alt isn't recognized and so its branch directives look top-level.
   `jinja_inspect` also reports "unknown tag 'm'" for the glued `{%`.
4. **Glued branch directives are silently lost.** `Review:%{%m:opus|%m:sonnet}` renders
   `Review:%m:opus`. No directive regex accepts `:` as a left boundary, so the slot gets
   no model and the literal text reaches the agent.
5. **The LSP has no alternation support.** The xprompt LSP (`crates/sase_xprompt_lsp`)
   emits no alternation semantic tokens and no unclosed-alternation diagnostic. External
   editor highlighting exists only in sase-nvim's Lua mirror.
6. **Alt tokenization repeats on every keystroke.** The TUI prompt input rebuilds its
   highlight map synchronously on every edit (Textual `TextArea.edit` calls
   `_build_highlight_map`). Alt tokenization runs up to three times per rebuild, with no
   cache:
   - `AltSyntaxHighlightMixin`;
   - `highlight_spans`, reached through `XPromptSyntaxHighlightMixin`;
   - `semantic_overlay`.

No prompt, xprompt, or config file in the repo contains a glued `%{` today, so relaxing
the brace form changes no existing content.

## Grammar decisions (normative for every phase)

1. **`%{` opens an alternation anywhere outside literal/definition zones.** That covers:
   - start of line;
   - after whitespace;
   - mid-word: `foo%{bar | baz}qux` gives `foobarqux` and `foobazqux`;
   - after punctuation: `pre-%{a | b}`;
   - immediately after another alternation: `%{a | b}%{c | d}`;
   - inside a colon value: `%m:op%{us | x}`;
   - inside another alternation's branch.

   The only way to write a literal `%{` is inside the existing literal zones: inline
   code, fenced blocks, and `%xprompts_enabled:false` regions.

2. **`%(...)` and `%alt(...)` keep the directive-valid position rule.** Relaxing them
   would turn `%(name)s` format strings, `50%(approx)`, and `--format=%(refname)` into
   fan-out.
3. **Branch semantics are unchanged:**
   - branches split on top-level `|`;
   - branches are trimmed;
   - `name=` prefixes still name branches;
   - a single branch gets an implicit empty variant, so `word%{s}` gives `words` and
     `word`;
   - named branches still correlate;
   - empty-branch whitespace collapse still applies.
4. **Nested alternation is supported.** An alternation inside a branch expands only when
   that branch is selected: `%{sase-%{core | github} | chezmoi}` gives `sase-core`,
   `sase-github` and `chezmoi`.
   - An opener that exists only because a rendered branch was concatenated with
     neighboring text is never expanded. For example, `%{a% | b}{x}` renders `a%{x}`
     literally.
   - Nested slot ids extend the parent branch id deterministically and stay unique.
   - A named outer branch keeps its name as its correlation key, and its nested variants
     multiply within that correlated slot.
   - Planning must never panic, however deep the nesting.
5. **A glued branch keeps its directives parseable.** This applies when a selected,
   non-empty branch is glued to adjacent text:
   - **Leading directive.** If the branch starts with a `%` directive marker (`%name` or
     `%(`) and the character before the alternation site is not a directive left
     boundary, the renderer inserts one space before the branch. The boundaries are
     start of text, whitespace, `(`, `[`, `{`, `"` and `'`.
   - **Trailing directive.** If the branch's last whitespace-delimited token is a
     `%name…` directive and the character after the site is alphanumeric or `_`, the
     renderer inserts one space after it.
   - **Result.** `Review:%{%m:opus | %m:sonnet}` renders `Review: %m:opus` and the slot
     model is opus. `foo %{%m:opus | %m:sonnet}bar` renders `foo %m:opus bar`.
   - **References and tags are not separated.** `#xprompt` references and `+tag`s in a
     glued branch are substituted verbatim. The docs tell users to put whitespace before
     the `%{` to keep them references.
6. **Documented Jinja ambiguity.** A literal `%` immediately followed by a Jinja `{%`
   tag (`100%{% if x %}`) is now read as an alternation. Write `100% {% if x %}` or use
   inline code instead. A test pins this behavior.

## Phase core-grammar: Core launch grammar for mid-word and nested alternation

**Repository:** the linked `sase-core` checkout. Open it with
`sase repo open sase-core`, follow its `AGENTS.md`, and verify with
`sase tool run check`.

- **`crates/sase_core/src/agent_launch/directive_scan.rs`:**
  - Rewrite `alt_directive_re()` so the brace marker needs no prefix, while the paren
    markers keep `(^|[\s\(\[\{"':])`.
  - The `regex` crate has no lookbehind. Use alternation groups, for example
    `(?m)(%\{)|(?:^|[\s\(\[\{"':])(%(?:alt)?\()`.
  - Make `alt_directive_starts` take whichever marker group matched. Today it reads
    `caps.get(2)`, and a changed group layout would silently drop every alternation.
  - The brace branch must not consume a prefix character, so adjacent openers
    (`%{a|b}%{c|d}`) are all found.
  - Update the doc comments. `alternation_body_ranges` and the literal-zone masks pick
    up the new rule automatically.
- **`crates/sase_core/src/agent_launch/fanout.rs`:**
  - **Nesting (decision 4).** `split_prompt_for_alternatives_with_ids` plans only
    outermost directives: skip any start that falls inside an already-accepted
    `[start, end)`. Suggested mechanism: while building variants, expand a branch value
    that itself contains an alternation (found with `alt_directive_starts` outside the
    value's own literal zones) into its sub-variants, so recursion only ever sees text
    that was nested in the source.
  - **Glued directives.** Implement decision 5 in `render_alternative_prompt`.
  - **Collapse helpers.** In `starts_with_directive_marker` and
    `should_preserve_directive_separator`, `%{` no longer needs a left boundary, so stop
    counting it as a marker that forces a preserved space. Keep `%(` and `%name`.
- **`crates/sase_core/src/editor/model_alias_shortcut.rs`:**
  - When an expansion ends immediately before an alternation opener, leave one
    separating space. Today `Use =la(%{x | y})` expands to `Use %m:@large%{x | y})`,
    which under the new grammar would fan out to `%m:@largex`.
  - Update `trigger_token_overlapping_a_body_keeps_body_text` to match.
- **Tests,** placed beside the code (`agent_launch/tests/fanout.rs`, the `tests` module
  in `editor/alternation.rs`, `model_alias_shortcut.rs`, and binding tests if any assert
  the old rule):
  - **Mid-word fan-out:**
    - `foo%{bar | baz}qux`;
    - `word%{s}` and `word%{|s} ok`;
    - `pre-%{a | b}`;
    - adjacent `%{a|b}%{c|d}`;
    - glued named branches with cross-directive correlation;
    - colon values: `%m:op%{us | x}`.
  - **Paren forms still need a boundary:** `x%(a,b)`, `50%(approx)` and `fmt%alt(a,b)`
    do not fan out, while `x %(a,b)` still does.
  - **Literal zones still win:** inline code, fenced blocks, disabled regions.
  - **Errors:** an unclosed `foo%{bar` returns `UnclosedDirective`.
  - **Nesting:**
    - regression for `a %{x %{p|q} | y} b` (currently panics);
    - `%{sase-%{core | github} | chezmoi}`;
    - the concatenation case from decision 4, which must stay literal.
  - **Directive separation:** both examples from decision 5, including slot models.
  - **Jinja ambiguity:** `100%{% if x %}` is an alternation.
  - **Existing tests stay green:** existing `%{+sa` project-tag trigger tests,
    `typed_plan` tab fan-out, and the colon-value fan-out tests.

## Phase core-scan-lsp: Shared alternation scanner, Python binding, and LSP highlighting

**Repository:** the linked `sase-core` checkout. Verify with `sase tool run check`.

- **Shared scanner in `crates/sase_core/src/editor/alternation.rs`.** Use a sibling file
  if that module would grow past about 1,500 lines, and keep any `mod.rs` facade-only.
  - Add one scanner with a serde `*Wire` record per alternation. Each record holds:
    - form (`brace` or `paren`);
    - marker start and opener end;
    - close offset, or none when unclosed;
    - top-level separator offsets;
    - `name=` branch-name spans;
    - nesting depth.
  - It excludes `excluded_literal_and_definition_ranges`, exactly like
    `alternation_body_ranges`. Re-express `alternation_body_ranges` on top of the
    scanner so there is one walk.
  - **One branch splitter.** Separator and branch-name offsets must come from the same
    top-level splitting that launch uses (`parse_directive_args_with_names` /
    `split_named_directive_arg`: backticks, double quotes, `[[...]]`, and bracket depth,
    not single quotes). Refactor that splitter so it can yield offsets instead of
    duplicating it. That fixes defect 2 for every frontend.
  - An unclosed opener produces one record whose marker is the error span.
- **Python binding.**
  - Add a binding such as `alternation_scan(text)` to the domain that binds editor
    content (`crates/sase_core_py/src/editor_content/`).
  - It returns a list of dicts with Python code-point offsets.
  - Convert byte to code-point offsets in one monotonic pass over the text. Do not
    recount the prefix per span the way the `project_tag` binding's
    `byte_to_char_offset` does, because the TUI calls this on keystrokes.
  - Add binding tests, including non-ASCII text before and inside an alternation.
- **Editor diagnostic.** In `crates/sase_core/src/editor/diagnostics.rs`
  (`analyze_document`), emit an error for each unclosed alternation outside literal
  zones. Put it on the marker span and reuse launch's wording
  (`unclosed %{ directive: missing closing '}'` / the `%alt` form).
- **LSP semantic tokens** (`crates/sase_xprompt_lsp/src/semantic_tokens.rs`). Add
  `raw_alternation_tokens(document)` and merge it in `document_semantic_tokens`:

  | Token                                             | Type        | Modifiers                              |
  | ------------------------------------------------- | ----------- | -------------------------------------- |
  | `%{`, `%(`, `%alt(` opener and its matching close | `operator`  | new `alternation`                      |
  | Top-level separators                              | `operator`  | `alternation` + new `separator`        |
  | `name=` branch names                              | `parameter` | `alternation`                          |
  | Unclosed opener                                   | `operator`  | `alternation` + the existing `unknown` |
  - Append the new modifiers to the legend after the `accent0`…`accent17` block, so
    existing modifier bits (including `ACCENT_MODIFIER_SHIFT = 5`) stay stable within 32
    bits.
  - Give alternation tokens a new lowest priority, so at shared bytes every existing
    token class wins. `%alt(` then keeps its MACRO name and argument tokens, and
    brace-form tokens overlap nothing.
  - Standard token types mean any LSP client colors them with no configuration.
  - Call the scanner once per semantic-tokens request, with no per-token rescans.

- **Tests:**
  - unit tests for the scanner (mid-word, nested depth, apostrophes, quotes, literal
    zones, unclosed, non-ASCII);
  - semantic-token unit tests;
  - a `crates/sase_xprompt_lsp/tests/jsonrpc_stdio*.rs` case showing that
    `foo%{bar | baz}qux` produces delimiter and separator tokens, and that `foo%{bar`
    produces the diagnostic.

## Phase sase-grammar-highlight: sase grammar mirror, highlight adapter, pin, and docs

**Repository:** this sase repo. Read the `lint_and_test` reference memory before
finishing.

- **Core pin.** Move `sase-core-revision.txt` to the sase-core commit containing the
  core-grammar and core-scan-lsp phases. See "The CI source revision pin" in
  `docs/rust_backend.md`; use `just ratchet-core-revision` when that commit is
  sase-core's remote HEAD. Local `just` recipes rebuild `sase_core_rs` from the linked
  checkout automatically.
- **`_ALT_DIRECTIVE_RE`** in `src/sase/xprompt/_directive_alt.py`:
  - The brace form matches anywhere; the paren forms keep the lookbehind.
  - One capture group covers the marker in both cases.
  - Audit every caller's use of `match.start(1)`, `match.end(1)` and `match.end() - 1`.
    The callers are listed in the Background section.
  - This fixes defect 3 (the false migration error and the `jinja_inspect` "unknown tag"
    for a glued `%{%`).
- **`src/sase/xprompt/alt_inspect.py`:**
  - Make `tokenize` a thin adapter over the new binding, obtained through
    `require_rust_binding`. It derives delimiter, separator, `branch_name`, and error
    `AltSpan`s from the scan records.
  - Delete the Python mirror scanning that becomes unused (`_find_matching_brace`,
    `_branch_spans`, `_top_level_offsets`, `_mask_protected_regions`).
  - Keep the public `AltSpan`/`tokenize` API so `highlight.py`, `semantic_overlay.py`,
    `xprompt_syntax.py` and `_alt_syntax_highlight.py` need no change.
  - **Fast path:** return `[]` without crossing FFI when the text contains neither `%{`
    nor `%(`. That check also covers `%alt(`.
  - **Memoize:** keep a small bounded LRU keyed on the text, returning immutable tuples,
    so the several callers in one highlight rebuild share one scan.
  - Expose a group-level helper (for example `alt_inspect.groups(text)`) over the same
    cached records.
- **Project tags.** In `src/sase/project_tags/tags.py`, replace `_alt_group_spans` with
  top-level brace groups from that helper, with branches split at the core-reported
  separators. The current hand-rolled `find("%{")` scan:
  - ignores literal zones;
  - closes early on `%{a {b} | c}`;
  - splits branches with `inner.split("|")`, which breaks nested alternation.
- **Tests:**
  - Update
    `tests/test_xprompt_alt_inspect.py::test_tokenize_requires_directive_valid_position`
    and
    `tests/test_directives_has_helpers.py::test_has_alt_directive_brace_shorthand_word_adjacent`
    to the new rule. Keep the `"100% of {x}"` and `"50% done"` guards.
  - Add tokenize cases: mid-word, paren forms still needing a boundary, apostrophes
    (`%{don't | do}` gets a separator span), nested, and literal zones.
  - Add `split_prompt_for_alternatives` / `split_prompt_for_models` cases: mid-word and
    glued-directive separation.
  - `extract_prompt_directives("foo%{%m:opus | %m:sonnet} go")` no longer raises.
  - `jinja_inspect.diagnose` is clean for a glued `%{%m:`.
  - Project tags inside mid-word and nested branches validate per branch.
- **Performance:**
  - Add a non-slow test, modeled on
    `tests/xprompt/test_highlight.py::test_calls_each_scanner_once`, proving that one
    highlight rebuild's alt callers (alt overlay, `highlight_spans`, semantic overlay)
    call the binding at most once per distinct text, and that alternation-free text
    never calls it.
  - Add a slow-marked benchmark in `tests/perf/`, modeled on
    `tests/perf/bench_tui_trace.py`. It asserts `alt_inspect.tokenize` p95 at or below
    today's Python implementation (and well under the 16 ms keystroke budget) on an ~80
    KB alternation-heavy prompt. Record the before and after numbers in the phase's
    final notes.
- **Docs.** In `docs/xprompt.md`, update the "Alt Directive" section (around lines
  3053–3088), plus the directive-table and syntax-block rows that describe `%{`. Cover:
  - the position rule, with mid-word, punctuation, colon-value and nested examples;
  - paren forms still needing a boundary;
  - the literal-zone escape;
  - glued-directive spacing, and verbatim `#`/`+`;
  - the Jinja ambiguity.

  Adjust the alt mentions in `docs/llms.md` if they imply the old boundary rule.

## Phase tui-alt-editing: TUI prompt input editing for mid-word alternation

**Repository:** this sase repo. Read the `tui` and `lint_and_test` reference memories
first.

- **`src/sase/ace/tui/widgets/_alt_syntax_editing.py`:**
  - Any `%` directly before `{` is an alternation opener. Replace
    `_is_directive_valid_brace_opening` / `_DIRECTIVE_OPENING_CONTEXTS` with that rule.
  - `plan_alt_brace_pair` pads `foo%{` to `foo%{  }` under the existing follow-character
    safety rule. It still does not pad before word characters.
  - **`_find_enclosing_alt_span`:**
    - recognize mid-word openers;
    - skip openers inside literal zones, using
      `sase.xprompt._literal_zones.literal_zone_ranges` and computing it only after a
      cheap `"%{"` presence check before the cursor;
    - let an unclosed span extend only to the end of the cursor's own line, so a stray
      earlier `x%{` cannot make a later `|` (for example a markdown table) rewrite text
      across lines;
    - return the innermost enclosing span when alternations nest (today the outer one
      wins).
- **`src/sase/ace/tui/widgets/_prompt_text_area_key_pairing.py`
  (`_try_jinja_auto_pair`).** Do not fire when the `{` before the cursor is itself
  preceded by `%`. Typing `%`, `#` or `{` right after `foo%{` then starts a branch
  (`%m:`, `#xprompt`) instead of a Jinja `{%  %}` or `{#  #}` pair.
- **Tests** in `tests/ace/tui/widgets/test_prompt_alt_syntax_editing.py`:
  - Update `test_is_directive_valid_brace_opening_contexts`,
    `test_find_enclosing_alt_span_ignores_non_directive_brace` and
    `test_plan_alt_brace_pair_requires_directive_valid_percent` to the new rule.
  - Keep the `word` + `{` → `word{}` and literal `foo|` guards.
  - Add cases for:
    - mid-word padding and separators;
    - innermost nested span;
    - inline-code openers being ignored;
    - an unclosed opener on another line being ignored;
    - the Jinja auto-pair guard.
  - Add an end-to-end Textual pilot test that types `foo%{bar|baz}` into the prompt
    input, following the existing prompt-input widget tests. It asserts the normalized
    text and that the alt highlight spans are present.
  - Update any visual snapshot that covers alt highlighting.
- **Docs.** In `docs/ace.md`, update the "Alt Brace Syntax" section (around lines
  8060–8100): replace the "directive-valid `%`" wording and describe mid-word typing.

## Phase nvim-lsp-highlight: sase-nvim alternation highlighting from LSP tokens

**Repository:** the linked `sase-nvim` checkout, opened with `sase repo open sase-nvim`.
Run its headless test suite as documented in its README.

- **Highlighting (`lua/sase/alt_highlight.lua`).** Replace the Lua regex grammar with an
  LSP-token overlay in the style of `lua/sase/xprompt_semantic_highlight.lua` and
  `lua/sase/project_tag_highlight.lua`. On `LspTokenUpdate`, map tokens carrying the
  `alternation` modifier onto the existing groups:

  | Tokens                         | Group               |
  | ------------------------------ | ------------------- |
  | `operator` without `separator` | `SaseAltDelimiter`  |
  | `separator`                    | `SaseAltSeparator`  |
  | `parameter`                    | `SaseAltBranchName` |
  | `unknown`                      | `SaseAltError`      |
  - Apply the groups with `vim.lsp.semantic_tokens.highlight_token`.
  - Keep the group names, their default links, and the `alt_highlight` setup keys.
  - Accept and ignore the now-unused `debounce_ms`, `max_lines` and `max_bytes` options.
  - Delete the `VALID_PREFIX` mirror, so there is one grammar.

- **Editing (`lua/sase/alt_edit.lua`).** Apply the same opener, literal-zone,
  unclosed-span, and innermost-span rules as the tui-alt-editing phase. Update
  `tests/alt_edit.lua`: its `a%{foo}` and `a%{` expectations now plan edits.
- **Tests:**
  - Rewrite `tests/alt_highlight.lua` against the token overlay, feeding synthetic
    tokens the way `tests/xprompt_semantic_highlight.lua` does.
  - Add a headless `tests/lsp_alt_highlight_smoke.lua`, modeled on
    `tests/lsp_project_tag_highlight_smoke.lua`. It asserts that `foo%{bar | baz}qux`
    gets alternation tokens from the server, and skips cleanly when the server does not
    advertise the modifier.
- **README.** Update the "Alt Brace Syntax" section:
  - highlighting now comes from the xprompt LSP;
  - the mid-word rule;
  - legacy `%(`/`%alt(` still need a boundary;
  - Neovim must expose `LspTokenUpdate`.

## Out of scope

- The `prompt_prediction` Jinja scan's missing `%{%` carve-out (a pre-existing, separate
  issue).
- Treating `|` as a directive left boundary, so that `%m` right after `|` without a
  space gets highlighted.
- Auto-separating `#xprompt` references or `+tag`s in glued branches.
- Any new escape syntax for a literal `%{` beyond the existing literal zones.
