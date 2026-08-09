---
tier: epic
title: Underline glossary terms in ACE prompts and in LSP-backed editors
goal: 'A matched project glossary term reads as a definable link everywhere SASE renders
  prompt text: bold, theme-accent, and underlined in the ACE prompt input, and underlined
  on top of the colorscheme''s semantic-token color in Neovim, without weakening the
  red misspelling underline it sits next to.

  '
phases:
- id: ace
  title: Underline glossary matches in the ACE prompt input
  depends_on: []
  size: medium
  description: 'ace: add the additive underline to the `glossary.term` text-area style,
    neutralize leaked underlines inside inline-code chips, make the visual glossary
    matcher fake skip code literals like the Rust matcher, refresh widget assertions,
    and regenerate dark plus new light PNG goldens with the ACE docs.'
- id: editor
  title: Underline glossary semantic tokens in LSP-backed editors
  depends_on: []
  size: medium
  description: 'editor: give sase-nvim an overridable `SaseGlossaryTerm` underline
    applied to the xprompt LSP''s glossary semantic tokens through `LspTokenUpdate`,
    with a headless test, README coverage, and corrected editor/LSP documentation
    in the sase repo.'
proposed_by: bbugyi200.athena.w9
create_time: 2026-08-09 07:49:39
status: wip
bead_id: sase-i2
---

- **PROMPT:** [prompts/202608/glossary_term_underline.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/glossary_term_underline.md)
- **BEAD:** [sase-i2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-i2/README.md)

# Plan: Underline glossary terms in ACE prompts and in LSP-backed editors

## Why

A glossary match is the one span in a prompt that has a definition behind it: `K`
previews it and `Ctrl+]` jumps to its `sase/sase.yml` entry. Today it renders as bold
theme-accent text, which reads as emphasis rather than as something you can follow.
Underlining it completes the link idiom, and it binds multiword phrases (`Agent Clan`,
`agent instruction file`) into one visual unit that bold alone cannot express.

Both surfaces that highlight glossary matches are in scope, so the affordance does not
change when a prompt moves from the ACE prompt bar into `$EDITOR`:

- ACE prompt input: `src/sase/ace/tui/widgets/_prompt_glossary.py`, style
  `glossary.term` registered at `_register_glossary_text_area_theme` (~line 304).
- Editors: the Rust xprompt LSP emits each glossary phrase as a standard `type` semantic
  token (`crates/sase_xprompt_lsp/src/semantic_tokens.rs`, `raw_glossary_tokens`), which
  a client theme colors. The protocol cannot ask a client to underline anything, so the
  editor half of this work belongs in the client SASE ships: `sase-nvim`.

## Design decisions (already made — do not relitigate)

### 1. Keep the color and the bold; add a plain single underline

`glossary.term` becomes
`Style(color=derive_argument_color(primary, ...), bold=True, underline=True)`.

Rendered variants were compared through the committed snapshot renderer (Rich SVG →
resvg, Fira Code) against `textual-dark` and `textual-light` backgrounds, on a line that
also contains a remembered misspelling:

- **A. bold only (today)** — reads as emphasis, not as a followable term.
- **B. bold + underline (chosen)** — unmistakably a link; the underline runs through the
  space inside `Agent Clan` and binds the phrase; stays clearly distinct from the red
  misspelling underline by hue and weight in both themes.
- **C. underline only (rejected)** — loses the scannability the current bold provides;
  glossary matches stop standing out in a dense prompt.
- **D. undimmed raw `primary` (rejected)** — `derive_argument_color` exists to keep
  contrast against the active background; do not bypass it.

### 2. Plain `underline`, not a fancier underline

Rich exposes `underline2` (SGR 21) but no curly/dotted/colored underline, `underline2`
is unreliable across terminals (many treat SGR 21 as bold-off), and Rich's SVG export
only emits `text-decoration: underline` and `line-through` (`rich/console.py` ~line
2384), so anything else would be invisible in the PNG goldens that guard this look.
Plain `underline=True` is the only underline that is both broadly supported and
verifiable.

### 3. The underline is additive; existing paint order stays exactly as it is

`PromptTextArea`'s mixin order (`src/sase/ace/tui/widgets/prompt_text_area.py` lines
74-101) already decides overlaps, and no mixin order changes here:

- Glossary spans are appended after misspelling spans, so glossary color keeps winning
  over `spell.misspelled` for a word that is both. Keep
  `test_glossary_overlay_wins_over_misspelling`.
- Structural overlays (code blocks, xprompt/artifact/placeholder spans, search, yank)
  are appended after glossary spans and keep winning on color. Rich style addition only
  overrides attributes the later style sets, so an underline can persist under a
  structural color — the same way the misspelling underline already does.

### 4. Two underline meanings, deliberately distinguished by hue and weight

After this change an underline no longer uniquely means "misspelled": glossary is
`bold` + theme accent + underline, misspellings stay regular weight + `#FF8787` +
underline (`src/sase/core/word_lookup.py:22`). That is accepted. When one word is both,
glossary wins outright and the misspelling signal is not separately visible — a
project-authored glossary term outranks a dictionary miss, and that is already the
current precedence.

### 5. Code literals must never render underlined

Both matchers deliberately claim "no semantics inside code": the Rust glossary scanner
skips `prompt_literal_zone_ranges` (`crates/sase_core/src/glossary.rs` ~line 293, test
`scan_skips_fenced_and_inline_code_literals`) and `_scan_misspelling_spans` skips
`code_literal_ranges`. Verified against the real matcher through the Python binding:

```
'a `Patch` literal'                     -> []
'Ask the Agent Clan to review the Patch' -> [('Agent Clan', 8, 18), ('Patch', 33, 38)]
```

So no real prompt can underline code. Two things still have to be made true, because the
underline is the first overlay attribute that would visibly leak into a code chip:

- `codeblock.inline` and `codeblock.delimiter` already pin `bold=False, italic=False`
  precisely to neutralize leaked attributes
  (`src/sase/ace/tui/widgets/_codeblock_syntax_highlight.py` ~lines 321-335); they must
  pin `underline=False` for the same reason, so the guarantee is structural rather than
  dependent on every present and future scanner remembering to skip literals.
- The visual snapshot fake matcher does _not_ skip literals today
  (`_VisualCompiledGlossary.scan` in
  `tests/ace/tui/visual/_ace_prompt_png_snapshot_helpers.py` ~line 291), and the
  glossary golden prompt contains `` `Agent Clan` ``. Left alone it would bake an
  underlined code chip into a committed golden that real ACE never renders.

### 6. Out of scope

- No new config key. `prompt_spellcheck.highlight` stays misspelling-only; glossary
  highlighting has no toggle today and does not gain one.
- No change to glossary matching, precedence, or suppression semantics. A project whose
  glossary defines an alias that collides with a directive or reference token (verified:
  `%clan:reviewers` matches an alias `clan`; `#memory/patch` matches `patch`) will paint
  the underline beneath the structural color. sase's own glossary has no such alias, the
  span is a genuine term occurrence, and adding a suppression rule would belong in the
  Rust core so ACE and the LSP agree. Revisit only if it looks wrong in practice.
- No Rust core or LSP legend change. Glossary is the only `type` token the server emits,
  so a client can already target it precisely.
- No underline for glossary terms outside prompt text (preview panel titles, completion
  rows, agent panels).
- `MISSPELLING_COLOR` is a fixed `#FF8787` and its underline is noticeably faint on
  `textual-light`; that predates this work and is not fixed here.

## Phase `ace`: Underline glossary matches in the ACE prompt input

### Source changes

1. `src/sase/ace/tui/widgets/_prompt_glossary.py`, `_register_glossary_text_area_theme`
   (~line 315): add `underline=True` to the `glossary.term` `Style`, keeping the
   existing `derive_argument_color(...)` color and `bold=True`. Record in the mixin
   docstring (which today only says "Memory-only glossary overlay and actions") that the
   style is the definable-term link affordance and that its underline is additive — the
   same note `MisspellingHighlightMixin`'s docstring carries about overlays that win on
   overlap.
2. `src/sase/ace/tui/widgets/_codeblock_syntax_highlight.py` (~lines 321-335): add
   `underline=False` to `codeblock.inline` and `codeblock.delimiter`, next to the
   existing `bold=False, italic=False`.

### Test changes

3. `tests/ace/tui/visual/_ace_prompt_png_snapshot_helpers.py`: make
   `_VisualCompiledGlossary.scan` drop matches that fall inside inline or fenced code,
   so the fake mirrors the Rust matcher. Prefer the helper production code already uses
   (`code_literal_ranges` from `sase.xprompt._literal_zones`, as in
   `_misspelling_highlight.py:17`); if a lint gate objects to that import from `tests/`,
   keep an equivalent minimal backtick scan local to the helper instead of loosening the
   fake's fidelity.
4. `tests/ace/tui/widgets/test_prompt_glossary.py`:
   - Line ~102: `assert style.underline is not True` becomes an assertion that the style
     _is_ underlined, still bold, and still not the warning/error color.
   - Lines ~227-236 (`test_glossary_style_reregisters_after_theme_switch`): the sentinel
     is `Style(color="red", underline=True)` and the assertion is
     `after.underline is not True`; re-registration must now be detected by something
     the new style does not carry (for example the sentinel color being gone), not by
     absence of underline.
   - Keep `test_glossary_overlay_wins_over_misspelling` and the order assertions in
     `test_glossary_overlay_order_keeps_search_and_code_in_front` unchanged; they encode
     decision 3.
   - Add: for a word that is both a glossary match and a remembered misspelling, the
     effective rendered style is the glossary color and is underlined. Resolve the
     effective style the way the code-block tests do —
     `text_area.render_line(y).crop(cell, cell + 1)` and read the segment style
     (`tests/ace/tui/widgets/test_prompt_codeblock_highlight.py` ~line 43).
   - Add: with a fake catalog that emits a glossary span inside an inline-code chip (the
     existing `_catalog_for_text(... occurrence_count=2)` pattern already produces one),
     the rendered style inside the chip is not underlined. This is the regression test
     for decision 5's `underline=False` pins.
5. `tests/ace/tui/visual/test_ace_png_snapshots_prompt_highlighting.py`
   (`test_prompt_glossary_highlight_png_snapshot`, ~line 300): parametrize
   `textual-dark` / `textual-light` exactly like the neighbouring misspelling and
   code-block snapshot tests (set `page.app.theme = theme` before `wait_for_startup`),
   renaming the golden to `prompt_glossary_highlight_dark_120x40` and adding
   `prompt_glossary_highlight_light_120x40`. The light golden is the point: the
   underline has to read well on a light background too, and nothing else in the corpus
   covers glossary rendering in light.
6. Regenerate goldens under `tests/ace/tui/visual/snapshots/png/`: delete the old
   `prompt_glossary_highlight_120x40.png`, produce both new goldens with
   `just test-visual --sase-update-visual-snapshots`, then rerun `just test-visual`
   clean. Inspect both PNGs before committing them and confirm: `Agent Clan` and `Patch`
   are underlined through the whole phrase, the `` `Agent Clan` `` code chip is _not_
   underlined, and the underline is legible in light theme. The renderer is pinned —
   goldens must come from a `just install-visual` environment matching
   `tests/ace/tui/visual/renderer_env.json`.

### Documentation

7. `docs/ace.md`, `#### Glossary terms` (~line 4329): state that warm glossary matches
   render bold, in the theme accent, and underlined, and that the underline marks a term
   you can `K` or `Ctrl+]`. In the following `#### Word definitions & spellcheck`
   section (~line 4359, "gets a red underline"), add the one-line contrast so the two
   underlines are documented as distinct signals.
8. `docs/configuration.md`, `### glossary` (~line 530, "ACE highlights warm glossary
   matches in prompt text"): same wording update, one sentence.

### Verification

- `just install` first (ephemeral workspace), then `just check`.
- `just test-visual` for the goldens, and `just fmt` for the Markdown edits.
- `just check-full` before landing.

## Phase `editor`: Underline glossary semantic tokens in LSP-backed editors

Open the plugin repo with `/sase_repo` (`sase repo open sase-nvim -r "..."`) and use the
printed path for every read and write. This phase lands changes in two repos: the
plugin, and the sase repo's editor documentation.

### sase-nvim

1. New module `lua/sase/glossary_highlight.lua`, following the shape of
   `lua/sase/alt_highlight.lua`:
   - Declare a stable, overridable group:
     `vim.api.nvim_set_hl(0, "SaseGlossaryTerm", { underline = true, default = true })`,
     defined in a `define_highlights()` function re-run on `ColorScheme` like the
     `SaseAlt*` groups. Setting only `underline` is deliberate: the client's colorscheme
     keeps owning the token color, and SASE only adds the affordance.
   - Register an `LspTokenUpdate` autocmd in its own augroup whose callback keeps only
     tokens with `type == "type"` from a client named `sase-xprompt-lsp` (the name used
     in `lua/sase/lsp.lua`), then calls
     `vim.lsp.semantic_tokens.highlight_token(token, ev.buf, ev.data.client_id, "SaseGlossaryTerm")`.
     `type` is the glossary token type in the server legend (`GLOSSARY_TOKEN_TYPE = 3` →
     `SemanticTokenType::TYPE`); artifact references use `namespace`/`string`/`number`.
     Note that assumption in a module comment so a future legend addition is caught
     here.
   - Guard the feature: no-op unless both `vim.lsp.semantic_tokens.highlight_token` and
     the `LspTokenUpdate` event exist (README advertises Neovim >= 0.8; the API landed
     later), and expose `setup({ enabled = true })` with the same enabled-by-default
     convention as the other features. Buffer eligibility needs no separate filter — the
     event only fires for buffers the sase LSP is attached to.
   - `highlight_token` stacks at `vim.hl.priorities.semantic_tokens + 3` above the base
     `@lsp.type.type` mark. Verify empirically (`vim.inspect_pos` on a matched term in a
     live buffer) that the colorscheme color survives and only the underline is added;
     if an attribute-merge surprise appears, keep the group underline-only and adjust
     priority rather than reintroducing a color.
2. Wire it into `lua/sase/init.lua`'s `setup()` next to `alt_highlight`, with a
   `glossary_highlight` opts table, and extend the `--- @param opts?` annotation.
3. Test `tests/glossary_highlight.lua`, run headless the way the sibling tests are
   (`nvim --headless -u NONE -c "set rtp+=." -l tests/glossary_highlight.lua`): assert
   the default `SaseGlossaryTerm` definition is underline-only and `default = true`, and
   drive the exported callback with synthetic `LspTokenUpdate` payloads (stubbing
   `vim.lsp.semantic_tokens.highlight_token`) to prove a `type` token from
   `sase-xprompt-lsp` is highlighted while a foreign client, a foreign token type, and a
   disabled config are not. Do not require a live LSP session.
4. `README.md`: a short glossary-terms subsection near the artifact-reference semantic
   highlighting coverage, with a highlight-group table row for `SaseGlossaryTerm`
   (default: underline, color left to the colorscheme), the override snippet convention
   already used for `SaseAlt*`, the `setup()` opts, and a manual smoke check plus its
   headless equivalent — matching how the other features are documented.

### sase repo documentation

5. `docs/editor.md`: two statements are now wrong or incomplete — "The provider
   currently emits artifact-reference tokens only." and "so the `sase-nvim` plugin needs
   no matching syntax or capability changes" (~lines 162-169). Correct them, note that
   glossary phrases are emitted as standard `type` tokens, and document that sase-nvim
   adds an overridable `SaseGlossaryTerm` underline on top of the client theme while
   other editors style `type` tokens with their own semantic-token theme. Add the
   glossary phrase mention to the `Semantic highlighting` row of the LSP feature table
   (~line 76).
6. `docs/xprompt.md` (~line 235, "Glossary phrases are emitted as standard `type`
   semantic tokens"): one sentence that sase-nvim underlines them by default and that
   the underline is overridable.
7. Keep to these two doc files. Phase `ace` owns `docs/ace.md` and
   `docs/configuration.md`, so the phases can run in parallel without touching the same
   file.

### Verification

- Plugin: the new headless test, plus the existing `tests/alt_highlight.lua` to prove
  the shared `setup()` wiring still passes.
- Manual: open an eligible prompt buffer in a project with a glossary, confirm a matched
  term is underlined and keeps its colorscheme color, and confirm a colorscheme override
  of `SaseGlossaryTerm` wins.
- sase repo: `just install`, then `just check` and `just fmt` for the doc edits.
