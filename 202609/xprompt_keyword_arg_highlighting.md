---
tier: epic
title: Structured highlighting for xprompt keyword arguments
goal: 'A keyword argument such as `#research_swarm(lead_model=claude-fable-5)` reads
  as structured syntax rather than one flat blob, identically in the ACE prompt widget
  and in external editors over LSP, with key, value, punctuation, literal type, and
  declaration validity each visually distinct and driven by one shared Rust grammar.

  '
phases:
- id: core-spans
  title: Argument span grammar in the Rust core
  depends_on: []
  size: medium
  description: 'core-spans: add a frontend-neutral xprompt/directive argument span
    tokenizer to sase-core that decomposes every argument form into key, assign, value,
    and delimiter roles with literal-type and validity classification, and stop treating
    unresolvable values as type mismatches.

    '
- id: lsp-tokens
  title: LSP semantic tokens for argument structure
  depends_on:
  - core-spans
  size: medium
  description: 'lsp-tokens: grow the xprompt LSP semantic token legend with standard
    LSP token types and emit argument-structure tokens from the core spans, merged
    against the existing artifact, code, and glossary tokens.

    '
- id: nvim-groups
  title: Neovim legend safety and default highlight links
  depends_on:
  - lsp-tokens
  size: small
  description: 'nvim-groups: make the glossary underline filter legend-proof, add
    default highlight links only where a standard token type reads poorly on a plain
    colorscheme, and cover the new tokens with an LSP smoke test.

    '
- id: tui-render
  title: ACE prompt widget argument rendering
  depends_on:
  - core-spans
  size: medium
  description: 'tui-render: consume the core spans in the prompt text area and the
    shared span partition, add the new highlight roles and their theme ramp, and stop
    the argument container role from overpainting nested Jinja, placeholder, and artifact
    spans.

    '
- id: visual-parity
  title: Visual snapshots and cross-surface parity
  depends_on:
  - lsp-tokens
  - tui-render
  size: medium
  description: 'visual-parity: pin the new rendering with dark and light PNG snapshots,
    prove the TUI and LSP classify the same text identically, and confirm the CLI
    and pager surfaces render every new role.'
proposed_by: bbugyi200.athena.0lq
create_time: 2026-09-15 21:08:36
status: wip
bead_id: sase-11i
---

- **PROMPT:** [prompts/202609/xprompt_keyword_arg_highlighting.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/xprompt_keyword_arg_highlighting.md)
- **BEAD:** [sase-11i](https://github.com/sase-org/sase--beads/blob/main/pages/sase-11i/README.md)

# Plan: Structured highlighting for xprompt keyword arguments

## Problem

An xprompt invocation's entire argument source is currently one highlight span.
`xprompt_inspect.tokenize` emits `invocation` for `#research_swarm` and a single
`invocation_arg` for `(lead_model=claude-fable-5)`; `_ROLE_PRECEDENCE` in
`src/sase/xprompt/highlight.py` then lets that one span win over everything nested
inside it. Directives behave the same way through `directive` / `directive_arg`.

Three concrete consequences:

1. **Nothing inside the parentheses is distinguishable.** The opening paren, the keyword
   `lead_model`, the `=`, the value `claude-fable-5`, the comma, and the closing paren
   all render in the same derived color. The value — the part a reader actually needs —
   is rendered in the _dimmest_ tone on the line, because `derive_argument_color` exists
   to push arguments back behind the invocation name.
2. **Nested syntax is swallowed.** `tests/xprompt/test_highlight.py` pins this today:
   `#foo({{ bar | upper }})` produces exactly two spans, and the Jinja expression inside
   the argument is erased by the container. The same happens to `@file:` artifact
   references, `<placeholders>`, and inline code inside argument values.
3. **External editors get no xprompt highlighting at all.** The LSP legend in
   `crates/sase_xprompt_lsp/src/semantic_tokens.rs` has four token types, spent entirely
   on artifact-reference kind/payload/fragment, directive-owned fenced code, and
   glossary terms. No invocation, directive, or argument token is ever emitted, so a
   prompt opened in Neovim is styled only by whatever Markdown grammar is attached.

The work is also duplicated: the launch-path lexical mirror lives in Python
(`xprompt_inspect.py`, `_parsing_args.py`) while the editor mirror lives in Rust
(`crates/sase_core/src/editor/xprompt_args.rs`). Adding fine-grained argument roles to
the Python mirror alone would guarantee the two surfaces drift.

## Design

### One grammar, two frontends

`sase/memory/rust_core_backend_boundary.md` gives the litmus test directly: behavior an
editor integration must match the TUI on is core backend logic. Argument tokenization is
exactly that, so the new grammar lands once in `sase-core` and is consumed by the LSP
crate natively and by Python through `sase_core_rs`.

The existing Rust parser already carries everything needed.
`parse_xprompt_calls(text) -> Vec<ParsedXpromptCall>` yields `name_span`, a `syntax`
discriminant covering every argument form (`Plus`, `Colon`, `DoubleColonText`,
`Parenthesized`, `Malformed`), an `is_open` flag for a call still being typed, and per
argument a `value_span` plus an optional `ParsedXpromptArgName` with its own `span`.
This epic turns that parse into spans and reuses it, rather than writing a third parser.

The existing Python name-level tokenizer stays. `xprompt_inspect.tokenize` keeps
emitting `invocation`, `directive`, `separator`, and `skill` spans, and keeps emitting
the container argument span; the new core spans are layered on top and win by
precedence. That keeps this epic's blast radius on argument interiors.

### Structure always, semantics only when the resolver is warm

The single most important reliability rule, because argument text is edited character by
character:

- **Structural roles** — delimiters, key, assign, value, and literal type — depend only
  on the text. They are always emitted.
- **Semantic roles** — "this keyword is not declared by this xprompt", "this value does
  not satisfy the declared input type" — are emitted only when the invoked name resolves
  to a catalog entry that declares inputs, _and_ the call is closed (`is_open` false).

That mirrors `argument_diagnostics` in `crates/sase_core/src/editor/diagnostics.rs` line
for line: it already skips unresolved entries, skips `is_open` calls, and skips entries
with no declared inputs. Deriving highlighting from the same resolution is what makes
highlight and diagnostics agree _by construction_ rather than by two parallel rulesets
that rot apart.

The consequence for the TUI is that a cold catalog is a third state, not a synonym for
"unknown". `_get_exact_warm_xprompt_arg_assist_entries()` already returns `None` to
distinguish a cold catalog from an empty one, and
`_get_warm_xprompt_skill_names_if_available` already preserves that sentinel. Cold
catalog means structural roles only — never a screen full of red keys that quietly turn
correct a second later.

### Unresolvable values are not type errors

`value_matches_input_type` currently answers a question it cannot answer. For an `int`
input, `#pr(bug_id={{ number }})` fails `value.parse::<i64>()` and is reported as
`invalid_xprompt_arg_type` today. The value is not wrong; it is not yet known.

A value whose span contains a Jinja expression, a `<placeholder>`, an artifact
reference, or a `$(...)` shell substitution is classified `unresolvable` and is never
marked invalid, on either surface. This removes an existing diagnostic false positive
and is a precondition for painting invalidity at all — without it the first thing
templated prompts would gain is a wall of false errors.

### Roles

Structural roles, emitted for both `#xprompt` and `%directive` calls:

| Role            | Covers                                                         |
| --------------- | -------------------------------------------------------------- |
| `arg_delimiter` | `(`, `)`, top-level `,`, and the introducing `:`, `::`, or `+` |
| `arg_key`       | the `name` in `name=value`                                     |
| `arg_assign`    | the `=`                                                        |
| `arg_value`     | a value with no recognized literal shape                       |

Literal refinements of `arg_value`, decided from the value text alone:

| Role               | Covers                                                |
| ------------------ | ----------------------------------------------------- |
| `arg_value_string` | `"..."`, `'...'`, `` `...` ``, and `[[ ... ]]` blocks |
| `arg_value_number` | a value parsing as `int` or `float`                   |
| `arg_value_bool`   | `true/false/1/0/yes/no/on/off`, case-insensitively    |

Validity is a separate field on each span, not a separate role: `ok`, `unknown_key`,
`type_mismatch`, `duplicate_key`, or `unresolvable`. Keeping it orthogonal is what lets
each frontend choose its own presentation without the core taking a position on whether
"wrong" means red text or a squiggle.

### Where validity is shown

Editors already have an error channel: `argument_diagnostics` publishes
`unknown_xprompt_arg`, `duplicate_xprompt_arg`, and `invalid_xprompt_arg_type`, and the
editor underlines them. Re-encoding the same fact as a loud semantic token would
double-signal. So:

- **LSP** carries structure and literal type in semantic tokens, and leaves validity to
  the diagnostics it already publishes. The one exception is the `deprecated` modifier
  on an `unknown_key`, which costs nothing and lets a theme dim an undeclared keyword
  without inventing a color.
- **ACE** has no squiggle channel in the prompt text area, so it paints validity
  directly, using the theme's `error` color as an underline rather than a foreground
  swap — an underline survives being read mid-edit, a color change reads as breakage.

Both derive from the same `validity` field on the same spans.

### Palette

The current derivation has exactly one step: `derive_argument_color` blends a family
color 40% toward the foreground. That produces the single muted tone every argument
currently shares. The new palette keeps one hue family per invocation kind — theme
`success` for `#xprompt`, theme `warning` for `%directive` — and varies _lightness and
weight_ within it, so an invocation still reads as one object:

- `arg_delimiter` and `arg_assign` blend the family color toward the **background**.
  Punctuation recedes; it is scaffolding, not content.
- `arg_key` keeps today's 40%-toward-foreground tone. Keys stay exactly the color
  arguments are now, so the change reads as "everything else moved into place" rather
  than a wholesale repaint.
- `arg_value` blends toward the **foreground** past the key. This is the central
  readability inversion: the value carries the meaning and should be the brightest thing
  inside the parentheses, not the dimmest.
- `arg_value_string`, `arg_value_number`, and `arg_value_bool` take theme `secondary`,
  `accent`, and `primary` respectively, each run through the same blend so they sit in
  the same tonal register as the rest of the group.

No new literal hex values. Every color is derived from the active Textual theme, so both
themes and the light/dark pair stay correct for free — the existing
`tests/xprompt/test_highlight_theme.py` pins the resulting Flexoki hexes, which is the
regression net for getting the derivation wrong.

For external editors the corresponding decision is to map onto **standard** LSP semantic
token types rather than inventing custom ones: `function` for the xprompt name, `macro`
for a directive name, `parameter` for a key, `operator` for `=` and delimiters, and
`string` / `number` / `keyword` for literal values. Every mainstream colorscheme already
styles those, so a user who installs nothing gets sensible colors immediately, and a
user who wants control overrides ordinary `@lsp.type.*` groups they already know.

### No feature flag

Per `sase/memory/sase_flags.md`, a beta flag is epic scaffolding for a phase that would
otherwise expose part of an unfinished feature. Each phase here lands a complete,
coherent improvement on its own: `core-spans` is invisible to users, `lsp-tokens` fully
delivers the editor half, `tui-render` fully delivers the ACE half, and the remaining
phases are polish and verification. Neither frontend depends on the other having landed.
No flag is warranted.

## Phase: core-spans — Argument span grammar in the Rust core

Work in the `sase-core` repository, opened with the `/sase_repo` skill.

Add an argument span module under `crates/sase_core/src/editor/` built on
`parse_xprompt_calls`. It must:

- Emit spans for every argument syntax in `XpromptArgSyntax`, not just `Parenthesized`:
  the `:` and `::` introducers, the `+` shorthand, and the trailing `): text` /
  `):: text` forms that `parse_parenthesized` already recognizes.
- Cover `%directive` calls as well. `parse_xprompt_calls` only scans `#` references, so
  directive argument regions must be located through the directive scan that
  `directive_diagnostics` already uses, then run through the same argument body parser.
- Exclude anything inside `prompt_literal_zone_ranges`, matching how
  `raw_artifact_ref_tokens` filters candidates today.
- Classify each value's literal shape, and classify `unresolvable` for values containing
  Jinja, `<placeholder>`, artifact-reference, or `$(...)` content.
- Offer a catalog-aware entry point taking `&[XpromptAssistEntry]` that fills the
  `validity` field, skipping unresolved entries, `is_open` calls, and entries with no
  declared inputs, exactly as `validate_call_args` does. Reuse that function's traversal
  rather than re-deriving which argument binds to which input; positional binding
  through `input_for_position` and the repeatable tail must not be duplicated.

Change `value_matches_input_type` so an unresolvable value returns true, and add a
diagnostics test proving `#pr(bug_id={{ number }})` no longer reports
`invalid_xprompt_arg_type`. This is the parity precondition described in the design; it
belongs here rather than in a later phase because both frontends read it.

Export a serializable wire type from `crates/sase_core/src/editor/wire.rs` and
`crates/sase_core/src/editor/mod.rs`, and add a `sase_core_rs` binding in
`crates/sase_core_py/src/lib.rs` returning spans as byte offsets, with the module
docstring list at the top of that file updated to match.

Unit tests belong beside the module and must cover: each argument syntax; a keyword
whose value contains a top-level `=`; quoted and `[[ ... ]]` values containing commas
and parens; a value spanning multiple lines; nested parentheses; an unterminated call;
and a call inside a fenced block producing no spans.

## Phase: lsp-tokens — LSP semantic tokens for argument structure

Work in the `sase-core` repository.

Extend `legend()` in `crates/sase_xprompt_lsp/src/semantic_tokens.rs` with the standard
token types named in the design. **Append only.** Token types are referenced by index in
the encoded stream, so reordering or inserting silently miscolors every existing
artifact-reference and glossary token in an already-running client. Add a test asserting
the first four legend entries keep their current positions, so a future edit cannot
reorder them by accident. `advertises_full_semantic_tokens_with_standard_legend` in
`crates/sase_xprompt_lsp/src/server.rs` also needs updating for the new legend.

Add a `raw_xprompt_argument_tokens` producer that converts the phase `core-spans` output
into `RawSemanticToken` values, and register it in `document_semantic_tokens` alongside
the existing artifact, code, and glossary producers. Give argument tokens a priority in
the existing `non_overlapping_tokens` scheme that lets an artifact reference inside an
argument value keep its own kind/payload/fragment tokens — an argument value is a
container in exactly the way the TUI container role is, and must lose to what it
contains.

The catalog-aware entry point needs the assist entries the server already resolves for
`argument_diagnostics`; reuse that path and its cache in `catalog_cache.rs` rather than
loading the catalog a second time per request. Where entries are unavailable, emit
structural tokens only.

Add coverage to `crates/sase_xprompt_lsp/tests/` in the style of the existing
`jsonrpc_stdio*.rs` files: a document with a keyword argument produces `parameter`,
`operator`, and typed value tokens at the right offsets, and the same document inside a
fenced block produces none.

## Phase: nvim-groups — Neovim legend safety and default highlight links

Work in the `sase-nvim` repository, opened with the `/sase_repo` skill.

`lua/sase/glossary_highlight.lua` filters `LspTokenUpdate` on `token.type == "type"` and
carries a comment stating that the xprompt LSP reserves `type` for glossary phrases and
that the filter must be revisited if the legend grows another `type` use. Phase
`lsp-tokens` grows the legend, so verify the new types do not collide with `type`, and
tighten the filter and its comment so a future legend addition cannot leak the glossary
underline onto unrelated tokens.

Audit how the new standard token types render on a plain colorscheme with no
configuration. Add `SaseXprompt*` default highlight links, in the
`nvim_set_hl(0, group, { link = ..., default = true })` style already used by
`lua/sase/alt_highlight.lua`, **only** where a standard type reads poorly — the point of
using standard types is that most of them need no help, and adding links that duplicate
the colorscheme's own choices takes control away from the user.

Add a smoke test under `tests/` following `lsp_artifact_ref_smoke.lua`, asserting the
server returns argument tokens for a buffer containing a keyword argument. Update
`README.md` where it documents the plugin's highlighting behavior.

## Phase: tui-render — ACE prompt widget argument rendering

Work in the `sase` repository.

Add the new members to `XPromptHighlightRole` in `src/sase/xprompt/highlight.py` under
the existing `xprompt.` prefix, and collect the core spans in `highlight_spans` beside
the existing producers, converting byte offsets with the `_byte_to_character_offsets`
helper already there. Guard the call with the same `MAX_HIGHLIGHT_BYTES` /
`MAX_HIGHLIGHT_LINES` budget and the same `try/except` discipline the other producers
use: a binding failure must degrade to today's flat container, never raise into a render
path.

Restructure `_ROLE_PRECEDENCE` so the containers stop overpainting. The ordering the
design requires, best first: literal zones (`code.fence`, `code.inline`); invocation and
directive names; argument key, assign, and delimiter roles; `alt.*`; `jinja.*`;
`placeholder`; `artifact_ref`; the typed `arg_value*` roles; and finally
`xprompt.invocation_arg` / `xprompt.directive_arg` as the lowest-priority fallback that
only fills what nothing else claimed.
`test_flattens_overlapping_invocation_and_jinja_by_precedence` pins the current
swallowing behavior and must be rewritten to assert the Jinja expression now survives
inside the argument.

Add every new role to `highlight_theme()` in `src/sase/xprompt/highlight_theme.py`.
`test_flexoki_role_palette_is_complete_and_stable` asserts the palette covers exactly
`get_args(XPromptHighlightRole)` and pins each hex, so a missing role is a test failure
rather than a `KeyError` in `cli_show_render._role_style` at runtime. Generalize
`derive_argument_color` into a helper taking an explicit blend target and ratio so the
delimiter, key, and value steps are three calls to one function rather than three
hand-tuned constants; keep `derive_argument_color`'s current signature and behavior for
its existing callers in `semantic_styles.py` and `pager/syntax_theme.py`.

Wire the corresponding `Style` entries into `_register_xprompt_text_area_theme` in
`src/sase/ace/tui/widgets/_xprompt_syntax_highlight.py`, and extend
`_build_highlight_map` to emit the new spans. The warm-catalog rule from the design
applies here: reuse `_get_exact_warm_xprompt_arg_assist_entries()` and its `None`
sentinel, pass entries to the catalog-aware core call only when warm, and emit
structural roles only when cold.

`sase/memory/tui_perf.md` governs this path — `_build_highlight_map` runs on the
keystroke path, so it must stay read-only, allocation-light, and free of disk access.
Confirm no regression with `SASE_TUI_PERF=1` and the p95 target in that note before
declaring the phase done.

## Phase: visual-parity — Visual snapshots and cross-surface parity

Work in the `sase` repository.

Add prompt fixtures to `tests/ace/tui/visual/_ace_prompt_png_snapshot_prompts.py` and
snapshots to `tests/ace/tui/visual/test_ace_png_snapshots_prompt_highlighting.py`
covering, in both dark and light: a valid keyword argument; mixed positional and keyword
arguments with typed values of each literal shape; an undeclared keyword; a value
holding a nested artifact reference and a Jinja expression; and a `%directive` with
keyword arguments. Follow the existing `prompt_xprompt_highlight_*` naming, and accept
new goldens with `--sase-update-visual-snapshots` per `sase/memory/lint_and_test.md`.

These snapshots are the actual acceptance test for the design's aesthetic claims. Review
them as images before accepting: the value should be the brightest element inside the
parentheses, punctuation should recede without disappearing, and the whole invocation
should still read as one object rather than a row of unrelated colors. If it does not,
the blend ratios are wrong and belong back in `tui-render`, not papered over here.

Add a parity test asserting the two frontends classify the same text identically — drive
the shared core entry point once, then assert the TUI role assignment and the LSP
token-type assignment are each a total function of the same span list, so a future
change to one surface cannot silently diverge from the other.

Confirm the non-TUI consumers of the palette still render: `sase xprompt show` through
`src/sase/xprompt/cli_show_body.py` and `cli_show_render.py`, which indexes
`highlight_theme()` by role and will raise on a missing one, and the pager theme in
`src/sase/pager/syntax_theme.py`. Add CLI coverage for an xprompt whose body contains a
keyword-argument invocation.

Run `just check-full` through the `/sase_monitor` skill before declaring the epic
landable, and `just test-visual` for the PNG suite.
