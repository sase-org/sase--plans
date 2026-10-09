---
tier: tale
title: Dictionary definition card for prompt K word lookups
goal:
  The K dictionary panel shows the term and its first definition at a glance, cleanly
  and on every theme.
size: medium
decisions:
  lead_source:
    ask:
      Should the highlighted first definition come from WordNet whenever dict returns a
      WordNet entry?
    choices:
      wordnet:
        WordNet's modern one-line sense first, then Webster's 1913, then other sources
      server_order:
        First definition dict returns (Webster's 1913 on dict.org; older, wordier)
    default: wordnet
    why:
      WordNet glosses are modern one-liners; Webster's 1913 often leads with archaic
      senses
    answer: wordnet
proposed_by: bbugyi200.apollo.60
decided_by: auto
create_time: 2026-10-09 05:56:50
status: wip
---

# Dictionary Definition Card: Make the Term and First Definition Unmissable

## Problem

`K` on a correctly spelled prompt word opens `WordDefinitionModal`
(`src/sase/ace/tui/modals/word_definition_modal.py`). Today it dumps raw `dict` output
into one scroll box:

- The term is a small `≡ WORD refuting` line. The subtitle lists every source's full
  name after a hardcoded `dict.org —`, which is wrong when a local `dictd` answers.
- The first definition is buried. For `refuting` the reader has to get past
  `refute \re*fute"\ (r[-e]*f[=u]t"), v. t. [imp. & p. p. {Refuted}; …]` and a 3-line
  etymology before reaching "To disprove and overthrow by argument…".
- GCIDE markup (`{Confute}`, `['e]`, `[=u]`), `[1913 Webster]` citation lines, and
  hard-wrapped 70-column text are shown verbatim.
- The modal is always 85%×85% with a box inside a box, so short entries float in empty
  space. The footer spends three rows on a rule plus hints.

## Design (the target)

A **dictionary card** that matches the sibling glossary card's language (pills, dim
labels, theme-aware accent; see `glossary_preview_render.py`). It has two clearly
separated zones:

1. **Hero (pinned, never scrolls).** It answers "what word, what does it mean" in about
   one second:
   - **Headline row.** The headword as a **pill** (`refute`, bold dark text on the
     accent background, the same pill style as the glossary chips). After it come the
     part of speech in italic (`transitive verb`) and the syllabification, dim
     (`· re·fute′`). On the right, dim, `looked up “refuting”` appears only when the
     displayed headword differs from the word under the cursor (case-insensitive). This
     explains lemma redirects such as refuting → refute.
   - **Lead definition callout.** The first definition as clean prose in normal
     foreground text. It sits on a subtle `$boost` background with a `tall` accent bar
     on its left edge. Below it, up to two usage examples appear in italic dim curly
     quotes joined by `·`.
   - **Attribution row.** Right-aligned and dim: `WordNet · sense 1 of 2` (the
     ` · sense 1 of N` part appears only when N > 1).
2. **Full entries (scrolls below the hero).** Every source, cleaned and structured, in
   `dict`'s order. Each source gets a left-aligned rule heading such as
   `WEBSTER'S 1913 ─────────`. Consecutive sections from the same database (GCIDE often
   returns one section per part of speech) share one heading. The content has numbered
   senses with hanging indents, italic dim examples and quotations (`— Addison`
   attribution), labeled `synonyms` / `antonyms` rows, and the thesaurus as one flowing
   `·`-separated word list. Cross-references `{Confute}` render as accent-colored
   `Confute` without braces. GCIDE citation-tag lines such as `[1913 Webster]` are
   removed.

Chrome:

- A `round` border in the accent color, with the border title `≡ DICTIONARY`.
- The key hints move into the **border subtitle** (bottom-right):
  `j/k scroll · ctrl+d/u page · g/G top/end · y copy · esc close`. Keys are bold accent
  and labels dim. This saves the three footer rows.
- The inner bordered box is removed.

Target (dark theme, 120×40, `refuting`, server returning GCIDE + Moby like the user's
screenshot; the pill and bar render as colored cells):

```
╭─ ≡ DICTIONARY ───────────────────────────────────────────────────────────────────────╮
│                                                                                      │
│  [ refute ]  transitive verb · re·fute′                         looked up “refuting” │
│                                                                                      │
│ ▌ To disprove and overthrow by argument, evidence, or countervailing proof; to       │
│ ▌ prove to be false or erroneous; to confute.                                        │
│ ▌ “to refute arguments” · “to refute testimony”                                      │
│                                                                      Webster's 1913  │
│                                                                                      │
│  WEBSTER'S 1913 ───────────────────────────────────────────────────────────────────  │
│  refute  transitive verb  re·fute′                                                   │
│    (r[-e]*fūt") [imp. & p. p. Refuted; p. pr. & vb. n. Refuting.] [F. réfuter, L.    │
│    refuteare to repel, refute. Cf. Confute, Refuse to deny.]                         │
│    To disprove and overthrow by argument, evidence, or countervailing proof; to      │
│    prove to be false or erroneous; to confute.                                       │
│      “to refute arguments” · “to refute testimony” · “to refute opinions or          │
│      theories” · “to refute a disputant”                                             │
│      “There were so many witnesses in these two miracles that it is impossible to    │
│      refute such multitudes.” — Addison                                              │
│    synonyms  To confute; disprove. See Confute.                                      │
│                                                                                      │
│  MOBY THESAURUS · 18 words ────────────────────────────────────────────────────────  │
│  apologetic · confounding · confutative · confuting · contradictory · contrary ·     │
│  excusatory · excusing · extenuating · extenuative · justificatory · justifying ·    │
│  palliative · refutative · refutatory · rehabilitative · vindicative · vindicatory   │
│                                                                                      │
╰─────────────────────────── j/k scroll · ctrl+d/u page · g/G top/end · y copy · esc ─╯
```

Design principles the implementation must hold:

- **Glanceable.** Only the headword is a pill, and only the lead definition sits on the
  barred background. Nothing else in the card competes with those two.
- **Honest and lossless.** Parsing only restyles. Every word in a source body still
  appears in the card, except for an explicit list of removed structural tokens (see
  Tests). Structure that cannot be parsed falls back to a cleaned verbatim rendering;
  content is never dropped.
- **Theme-safe.** No hardcoded `white`/`black` foregrounds for body text: use the
  theme's foreground (Rich `dim`/`italic`/`bold` modifiers only) and the accent from a
  small palette resolved from `app.current_theme.dark`:
  - dark: accent `#5FD7AF`, pill foreground `#1a1a1a` (the existing dictionary mint)
  - light: accent `#00875F`, pill foreground `#FFFFFF`
- **Compact when short, bounded when long.** The card shrinks to fit short entries. Long
  entries (`run`, `set`, about 45–50 KB from `dict`) cap at 90% height with the hero,
  the scroll area, and the bottom-border hints all visible.

## Lead Definition Selection

> [!decision] lead_source = wordnet
>
> Priority: (1) WordNet (`wn`). If a GCIDE entry is also present, take the first WordNet
> sense whose part-of-speech family (noun/verb/adjective/adverb) matches the **first
> GCIDE entry's** part of speech; otherwise take WordNet's first sense. This fixes
> WordNet's noun-first ordering, e.g. `run` → "move fast by using one's feet…" instead
> of the baseball noun. (2) The first sense of the first GCIDE entry. (3) The first
> prose paragraph (≥ 6 words, after a lone headword line) of the first other
> non-thesaurus source. (4) No lead: the hero shows only the headline pill with the
> looked-up word.

> [!decision] lead_source = server_order
>
> Priority: the first section in `dict` order that yields a sense (GCIDE, WordNet, or
> generic, using the same per-source extraction), skipping `moby-thesaurus`.

Common rules:

- The Moby Thesaurus is never a lead; it lists related words, not a definition.
- **Headword shown**
  - WordNet: verbatim (it keeps meaningful case, e.g. `UNIX`).
  - GCIDE: lowercased when the GCIDE headword is Title-case and the looked-up word is
    all lowercase (GCIDE capitalizes every headword).
  - Generic: the looked-up word.
- **Syllabification** comes from the first GCIDE entry whose headword matches the shown
  headword case-insensitively, even when the lead is from WordNet. Convert `*` → `·`,
  `"` → `′` and `` ` `` → `″` (the 1913 Webster primary/secondary stress marks), and
  apply the same case rule. Hide it when it adds nothing (no `·`, `′`, or `″` after
  conversion, e.g. `\Run\`).
- **Gloss** is plain text: cross-reference braces removed, markup normalized, collapsed
  whitespace, `--` → `—`. If the gloss lacks terminal punctuation, add a period. Cap it
  at 300 characters at a word boundary with `…`; the full text stays in the details.
- **Sense count** in the attribution: WordNet counts all senses in that section; GCIDE
  counts numbered top-level senses in that entry (1 when unnumbered).

## Per-Source Parsing Spec

All parsing is pure (no Textual/Rich), runs on `DefinitionSection` bodies, and is total.
Each per-database parser is wrapped in `try/except Exception` that falls back to the
generic parser for that section. A parser that yields no blocks for a non-blank body
also falls back. `build_definition_card(word, sections)` never raises.

Database identity: add `database: str = ""` to `DefinitionSection` in
`src/sase/core/word_lookup.py`, populated from the header regex's existing `database`
group. Rename `_parse_definition_sections` to public `parse_definition_sections(stdout)`
(add it to `__all__`) so tests can build sections from raw fixtures. When `database` is
empty (older callers and tests), infer it from the source prefix:
`The Collaborative International Dictionary` → `gcide`, `WordNet` → `wn`,
`Moby Thesaurus` → `moby-thesaurus`.

Source display labels (fallback: the full `source` string verbatim):

| database         | label                     |
| ---------------- | ------------------------- |
| `gcide`          | Webster's 1913            |
| `wn`             | WordNet                   |
| `moby-thesaurus` | Moby Thesaurus            |
| `foldoc`         | FOLDOC                    |
| `jargon`         | Jargon File               |
| `devil`          | Devil's Dictionary        |
| `easton`         | Easton's Bible Dictionary |
| `hitchcock`      | Hitchcock's Bible Names   |
| `bouvier`        | Bouvier's Law Dictionary  |
| `vera`           | V.E.R.A.                  |
| `elements`       | The Elements              |
| `world02`        | CIA World Factbook 2002   |

### GCIDE (`gcide`)

1. **Markup normalization.** Do this first, so nested markup no longer breaks bracket
   scanning. Use a conservative table:
   - Acute `['x]` → á é í ó ú ý (and uppercase forms); grave ``[`x]`` → à è ì ò ù.
   - Diaeresis `["x]` → ä ë ï ö ü ÿ; circumflex `[^x]` → â ê î ô û.
   - Tilde `[~n]` / `[~a]` / `[~o]` → ñ ã õ; cedilla `[,c]` → ç.
   - Ligatures `[ae]` `[AE]` `[oe]` `[OE]` → æ Æ œ Œ; `[root]` → √.
   - Macron `[=x]` → x + U+0304 and breve `[x^]` → x + U+0306, then NFC.
   - Any other bracket token is left **verbatim**; never delete unknown markup.
2. **Dedent and classify lines.** Dedent the body with `textwrap.dedent`. Split
   paragraphs at blank or whitespace-only lines. Drop **citation-tag lines** (a line
   whose stripped content is a single `[…]` group with no nested brackets, e.g.
   `[1913 Webster]`, `[PJC]`, `[Webster 1913 Suppl.]`); a dropped line also ends the
   current block. A new block starts at a line beginning with any of:
   - a sense marker `\d+\.` (depth 0)
   - a sub-sense `\([a-z]\)` (depth 1)
   - `Syn:` (label `synonyms`)
   - `Note:` (label `Note`)
   - a `{Phrase}` sub-entry at block indent

   Indentation rule, relative to the current block's **start indent**:
   - ≤ +4 is a continuation line.
   - ≥ +5 is a quotation line, grouped into a `quote` block.

   This holds for the header at 0 / body at 3, for `1.` at 3 with continuation at 6 and
   quotes at 12, and for `(a)` at 6 with continuation at 10 and quotes at 16. Verify it
   against every fixture.

3. **Reflow.** Join a block's lines with single spaces and collapse runs of spaces.
4. **Header block.** Scan the first block in order:
   - Headword up to ` \`; if there is no `\…\`, the headword runs up to the first `,`.
   - Syllables `\…\`.
   - Optional balanced `( … )` pronunciation.
   - `,`, then the part-of-speech run of 1–7-letter abbreviation tokens ending in `.`
     (with `&`). Map it:
     - `v. t.` → transitive verb; `v. i.` → intransitive verb; `v.` → verb
     - `n.` → noun; `n. pl.` → plural noun
     - `a.` / `adj.` → adjective; `adv.` → adverb; `p. a.` → participial adjective
     - `prep.` → preposition; `conj.` → conjunction; `interj.` → interjection; `pron.` →
       pronoun
     - `p. p.` → past participle; `p. pr.` → present participle
     - unknown → the raw abbreviation
   - Zero or more **balanced** `[…]` groups (inflections and etymology). Together with
     the pronunciation they form a `meta` block.

   Any remaining text is the entry's unnumbered first sense.

5. **Senses.** In sense text, split the GCIDE usage convention: the first `; as, ` (or a
   leading `as, `) starts examples, split on `; `. The gloss is everything before it.
6. **Quotes.** Split the attribution at the last `--`; render it as `— Author`. A
   wrapped attribution such as `--Sir J.` / `Stephen.` joins into `— Sir J. Stephen.`.

### WordNet (`wn`)

- The first non-blank dedented line is the headword.
- Sense lines match `^(?:(adj|adv|n|v)\s+)?(\d+)?\s?:\s` (`adj 1: …`, `2: …`, `n 1: …`).
  A POS prefix opens a new POS group, rendered as an italic accent heading (`noun`,
  `verb`, `adjective`, `adverb`). Deeper-indented lines continue the sense.
- In the reflowed sense, extract trailing bracket lists `[syn: {a}, {b}]`, `[ant: …]`,
  `[also: …]`, and `[see also: …]` into labeled item rows (`synonyms`, `antonyms`,
  `also`, `see also`). Drop items equal to the headword (case-insensitive) and omit a
  row that ends up empty.
- Quoted strings `"…"` become examples. The gloss is the text before the first quote,
  minus any trailing `;`. Any leftover non-quote text is appended to the gloss so
  nothing is lost.

### Moby Thesaurus (`moby-thesaurus`)

- The header `^(\d+) Moby Thesaurus words? for "(.+)":` gives the count (if the header
  is missing, count the items).
- The rest, reflowed and split on `,`, gives the items.
- The rule heading becomes `MOBY THESAURUS · N words`.

### Generic (every other database, and the fallback)

- Dedent and keep lines **verbatim** (no reflow; FOLDOC/Jargon contain preformatted
  text).
- Only `{X}` cross-references are restyled.
- Paragraphs are separated by one blank line.

## Rendering Spec

- **`HangingIndent` renderable.** Hanging indents use a small custom Rich renderable
  (`__rich_console__` wraps the body with `Text.wrap(console, max_width - indent)`,
  prints the marker cell before the first line and spaces before the rest, and provides
  `__rich_measure__`). Do not create one `Table.grid` per sense: GCIDE `run` and `set`
  have hundreds of blocks, and per-block tables are measurably slower.
- **Block styles:**
  - Sense marker: bold accent, right-aligned in a 3-column cell; sub-senses are indented
    one level.
  - Gloss: default foreground.
  - Examples: italic dim in curly quotes joined by `·`.
  - Quotes: italic dim, with ` — Author` (dim, not italic) appended.
  - Entry line: bold headword, two spaces, italic POS, two spaces, dim syllables.
  - Meta: dim, indented 2.
  - Labeled rows: dim label (`synonyms`), two spaces, items in accent joined by dim `·`.
  - Cross-refs: accent, no underline (they are not clickable).
- **Source heading:**
  `rich.rule.Rule(Text(LABEL.upper()…, style="bold <accent>"), align="left", style="<accent> dim")`,
  with one blank line before every heading except the first.
- **Hero:** three widgets.
  - `#word-definition-headline`: a two-column `Table.grid(expand=True)`, mirroring
    `build_glossary_title`. Left column: the pill, POS, and syllables. Right column: the
    `looked up “…”` disclosure.
  - `#word-definition-lead`: gloss text plus a newline and the examples line.
  - `#word-definition-attribution`.

  Omit the lead and attribution widgets when there is no lead.

## Modal, CSS, Keys

Rewrite `WordDefinitionModal` to take a prebuilt card:
`WordDefinitionModal(card: DefinitionCard)`.

- `compose()` resolves the palette from `self.app.current_theme` and yields:

  ```
  Container#word-definition-container
    Vertical#word-definition-hero
      #word-definition-headline
      #word-definition-lead
      #word-definition-attribution
    VerticalScroll#word-definition-scroll
      Static#word-definition-content
  ```

- Set `border_title` / `border_subtitle` as Rich `Text`.
- Apply the palette accent to the container border and the lead's left border through
  inline `styles` (`("round", accent)` / `("tall", accent)`) so light themes get the
  darker accent.

Replace the `WordDefinitionModal` block in `src/sase/ace/tui/styles.tcss` with:

```
WordDefinitionModal { align: center middle; }
WordDefinitionModal > Container {
    width: 88; max-width: 94%;
    height: auto; max-height: 90%;
    border: round #5FD7AF;
    border-title-align: left; border-subtitle-align: right;
    background: $surface; padding: 1 2;
}
WordDefinitionModal #word-definition-hero { dock: top; height: auto; }
WordDefinitionModal #word-definition-headline { height: auto; }
WordDefinitionModal #word-definition-lead {
    height: auto; margin-top: 1; padding: 0 1;
    background: $boost; border-left: tall #5FD7AF;
}
WordDefinitionModal #word-definition-attribution { height: auto; text-align: right; color: $text-muted; }
WordDefinitionModal #word-definition-scroll { height: auto; max-height: 100%; margin-top: 1; scrollbar-gutter: stable; }
WordDefinitionModal #word-definition-content { width: 100%; height: auto; }
```

**`dock: top` on the hero is load-bearing.** I measured Textual 8.0.1 with a headless
probe:

- **Non-docked hero + scroll `max-height: 100%`:** the scroll overflows the card by the
  hero's height, so its last lines are clipped and unreachable.
- **Scroll `height: 1fr`:** fits long content but inflates short cards to full height.
- **Docked hero + auto/`max-height: 100%` scroll:** fits short content compactly (e.g. a
  15-row card), and at 120×40 the scroll gets exactly the remaining 24 rows. It also
  holds at 70×24 with a 13-row hero.

Keys: keep every existing binding (`escape`/`q` close, `ctrl+d`/`ctrl+u` half-page,
`j`/`k`, `g`/`G`/`shift+g`). Add `y` to copy the lead gloss with
`schedule_copy_delivery(self, card.lead.gloss, copied_label="definition", task_name="sase-word-definition-copy")`,
matching the glossary card's `y`. With no lead,
`notify("No definition summary to copy", severity="warning")`.

## Off-Thread Card Build

In `PromptWordLookupMixin._resolve_word_lookup_async`
(`src/sase/ace/tui/widgets/_prompt_word_lookup.py`):

1. When `definitions.status == "ok"`, build the card with
   `await asyncio.to_thread(build_definition_card, span.word, definitions.sections)`
   (lazy import, following the existing modal-import pattern). Parsing 50 KB of GCIDE
   must not run on the event loop.
2. Re-check `_word_lookup_request_is_current(request_id)` after that await.
3. Pass the card through `_handle_definition_result` to push
   `WordDefinitionModal(card)`.

Other statuses are unchanged.

## Files

- **`src/sase/core/word_lookup.py`:** add `DefinitionSection.database`; make
  `parse_definition_sections` public.
- **`src/sase/ace/tui/modals/word_definition_card.py` (new, pure):** dataclasses
  (`DefinitionCard`, `LeadDefinition`, `DictEntry`, `DictBlock`; suggested shape),
  database inference and labels, WordNet/Moby/generic parsers, lead selection, and
  `build_definition_card`.
- **`src/sase/ace/tui/modals/_word_definition_gcide.py` (new, pure):** the GCIDE markup
  table and the GCIDE parser, split out so every file stays well under the `toobig`
  700-line threshold.
- **`src/sase/ace/tui/modals/word_definition_render.py` (new):** the palette,
  `HangingIndent`, hero/headline/lead/attribution builders, the details `Group` builder,
  and the border title and hint `Text`.
- **`src/sase/ace/tui/modals/word_definition_modal.py`:** the rewrite described above.
- **`src/sase/ace/tui/styles.tcss`:** the CSS block above.
- **`src/sase/ace/tui/widgets/_prompt_word_lookup.py`:** the off-thread card build.
- **`docs/ace.md`** (`#### Word definitions & spellcheck`): describe the card:
  - the headword pill and the `looked up` disclosure
  - the highlighted first definition and its source attribution
  - the cleaned per-source entries below
  - `y` to copy the definition

Rust boundary: this stays in Python. The `dict` client it extends already lives in
Python, and turning `dict` text into a card is presentation shaping for one TUI widget
that no other frontend consumes. Moving the DICT adapter into `sase-core` would be a
separate decision.

## Tests

Fixtures: capture raw `dict` stdout once into `tests/fixtures/dict/<word>.txt` for
`refuting`, `ephemeral`, `loquacious`, `serendipity`, `run`, and `unix` (with
`dict -- <word> > …`). If the DICT servers are unreachable, use the appendix texts for
`refuting` and `ephemeral`, and build smaller hand-written WordNet/Jargon samples for
the others from the formats shown in this plan. Commit the fixtures verbatim, keeping
trailing whitespace on the separator lines. Load them through
`parse_definition_sections`.

- **`tests/ace/tui/modals/test_word_definition_card.py`:**
  - Per-fixture lead assertions:
    - `refuting` → headword `refute`, `transitive verb`, syllables `re·fute′`, gloss
      starting "To disprove and overthrow", examples starting `to refute arguments`,
      source `Webster's 1913`, `looked up` shown.
    - `ephemeral` → WordNet "lasting a very short time", `adjective`, syllables
      `e·phem′er·al`, sense count 2.
    - `run` → a WordNet **verb** sense via POS alignment.
    - `serendipity` → WordNet with no syllables.
    - `unix` → WordNet `UNIX` verbatim.
  - The markup table, including that unknown tokens survive verbatim.
  - Citation lines are dropped.
  - GCIDE quote attribution, including the wrapped `Sir J. Stephen`.
  - WordNet syn/ant extraction with headword filtering.
  - Moby count and items.
  - Database inference from source names.
  - A parser that raises (monkeypatched) falls back to generic without losing text.
  - `build_definition_card` is total on garbage bodies: empty, whitespace,
    `"  n 1: a greeting"` without a headword line, and unbalanced `[`/`{`/`\`.
  - The `server_order` lead priority is covered by a parametrized test of the lead
    selector only if that decision branch is chosen.
- **`tests/ace/tui/modals/test_word_definition_render.py`:**
  - Render hero and details to `Console(width=80, record=True, color_system=None)` and
    assert on the plain text: rule headings, hanging-indent continuation lines start
    with spaces, braces removed, no `[1913 Webster]`.
  - **Lossless invariant** over every fixture: the casefolded alphabetic tokens of
    length ≥ 2 (`[^\W\d_]{2,}`) of each raw body must all appear in the rendered card
    text. Normalize markup on the raw side and remove exactly these structural tokens
    there: GCIDE citation-tag lines, the Moby count header line, the GCIDE `Syn:` label
    word, and the WordNet bracket labels (`syn`, `ant`, `also`, `see also`). Remove
    nothing else.
  - Palette: dark vs light accent and pill foreground.
- **`tests/ace/tui/modals/test_word_definition_modal.py`:**
  - The hero widgets exist with the expected text.
  - The no-lead card omits the lead and attribution widgets.
  - `y` calls the copy helper with the gloss (monkeypatch `schedule_copy_delivery` in
    the modal module), and warns when there is no lead.
  - `j`/`G` scroll the details while the hero stays put.
- **Update `tests/test_word_lookup.py`:** assert `database` is captured, and cover the
  public `parse_definition_sections`.
- **Update `tests/ace/tui/widgets/test_prompt_normal_mode_word_lookup.py`:** the K flow
  still pushes `WordDefinitionModal`; add an assertion that the pushed modal's card
  headword/lead came from the faked sections.
- **PNG snapshots** in `tests/ace/tui/visual/test_ace_png_snapshots_word_lookup.py`.
  Build the card from the fixtures via `build_definition_card`:
  - `word_definition_modal_120x40`: replace the fake fixture with `refuting`, dark
    theme.
  - `word_definition_modal_wordnet_120x40`: `ephemeral`, dark; WordNet lead and multiple
    sources.
  - `word_definition_modal_wordnet_light_120x40`: `ephemeral` with
    `page.app.theme = "textual-light"`.
  - `word_definition_modal_long_70x24`: `run` at 70×24; this proves the bounded layout
    with hero, scroll, and bottom-border hints all visible.

## Verification

1. `just fmt`, then `sase tool run check`. Run `just install-venv` first if the venv is
   stale.
2. Run
   `just fix-tui-screenshots -- tests/ace/tui/visual/test_ace_png_snapshots_word_lookup.py`
   through `/sase_monitor` (it can take more than 10 minutes). Then **open every
   created/updated PNG** and check these acceptance criteria:
   - The pill headword and barred lead definition are the most prominent elements.
   - No raw `{`/`}` braces, `[1913 Webster]` lines, or `['e]`-style accents remain.
   - Short cards are compact; the long card shows the hero, a scroll area, and the
     bottom-border hints.
   - The light theme is legible: accent text and borders are visible, the pill text has
     contrast, and there is no white-on-white.
3. Optional live check with dict available: start `sase screenshot --keep`, type a
   prompt containing `refuting`, put the cursor on it in NORMAL mode, press `K`, and
   recapture. Real dict output can differ from the fixtures; check it still reads well.

## Out of Scope

- A loading indicator while `aspell`/`dict` run.
- Following cross-references with `1`–`9`, as the glossary card's `SEE ALSO` does.
- Changing the spellcheck panel.
- `GlossaryPreviewModal` has the same non-docked title/footer + `max-height: 100%`
  scroll pattern, which clips long definitions. It is tracked separately as task bead
  `sase-1is`; leave it alone here.

## Appendix: Raw Fixture Texts

`refuting`: the shape the user's server returned (GCIDE v0.48 + Moby). Blank separator
lines in real output contain two spaces.

```
2 definitions found

From The Collaborative International Dictionary of English v.0.48 [gcide]:

  refute \re*fute"\ (r[-e]*f[=u]t"), v. t. [imp. & p. p.
     {Refuted}; p. pr. & vb. n. {Refuting}.] [F. r['e]futer, L.
     refuteare to repel, refute. Cf. {Confute}, {Refuse} to deny.]
     To disprove and overthrow by argument, evidence, or
     countervailing proof; to prove to be false or erroneous; to
     confute; as, to refute arguments; to refute testimony; to
     refute opinions or theories; to refute a disputant.
     [1913 Webster]

           There were so many witnesses in these two miracles that
           it is impossible to refute such multitudes. --Addison.
     [1913 Webster]

     Syn: To confute; disprove. See {Confute}.
          [1913 Webster]

From Moby Thesaurus II by Grady Ward, 1.0 [moby-thesaurus]:

  18 Moby Thesaurus words for "refuting":
     apologetic, confounding, confutative, confuting, contradictory,
     contrary, excusatory, excusing, extenuating, extenuative,
     justificatory, justifying, palliative, refutative, refutatory,
     rehabilitative, vindicative, vindicatory


```

`ephemeral` (dict.org: GCIDE ×2, WordNet, Moby):

```
4 definitions found

From The Collaborative International Dictionary of English v.0.54 [gcide]:

  Ephemeral \E*phem"er*al\, a.
     1. Beginning and ending in a day; existing only, or no longer
        than, a day; diurnal; as, an ephemeral flower.
        [1913 Webster]

     2. Short-lived; existing or continuing for a short time only.
        "Ephemeral popularity." --V. Knox.
        [1913 Webster]

              Sentences not of ephemeral, but of eternal,
              efficacy.                             --Sir J.
                                                    Stephen.
        [1913 Webster]

     {Ephemeral fly} (Zo["o]l.), one of a group of neuropterous
        insects, belonging to the genus {Ephemera} and many allied
        genera, which live in the adult or winged state only for a
        short time. The larv[ae] are aquatic; -- called also {day
        fly} and {May fly}.
        [1913 Webster]

From The Collaborative International Dictionary of English v.0.54 [gcide]:

  Ephemeral \E*phem"er*al\, n.
     Anything lasting but a day, or a brief time; an ephemeral
     plant, insect, etc.
     [1913 Webster]

From WordNet (r) 3.0 (2006) [wn]:

  ephemeral
      adj 1: lasting a very short time; "the ephemeral joys of
             childhood"; "a passing fancy"; "youth's transient
             beauty"; "love is transitory but it is eternal";
             "fugacious blossoms" [syn: {ephemeral}, {passing},
             {short-lived}, {transient}, {transitory}, {fugacious}]
      n 1: anything short-lived, as an insect that lives only for a
           day in its winged form [syn: {ephemeron}, {ephemeral}]

From Moby Thesaurus II by Grady Ward, 1.0 [moby-thesaurus]:

  47 Moby Thesaurus words for "ephemeral":
     brief, brittle, capricious, changeable, corruptible, deciduous,
     dying, episodic, evanescent, evergreen, fading, fickle, fleeting,
     flitting, fly-by-night, flying, fragile, frail, fugacious,
     fugitive, half-hardy, hardy, impermanent, impetuous, impulsive,
     inconstant, insubstantial, momentary, mortal, mutable, nondurable,
     nonpermanent, passing, perennial, perishable, short, short-lived,
     subject to death, temporal, temporary, transient, transitive,
     transitory, undurable, unenduring, unstable, volatile


```
