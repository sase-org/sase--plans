---
tier: tale
title: Wrap `sase bead show` prose at a configurable width
goal:
  "`sase bead show` wraps DESCRIPTION, NOTES, and +1 evidence prose at a configurable column budget (default 120)
  without ever splitting a URL or an inline code span, and `-S/--style {auto,plain,color,rich}` becomes `-s/--style
  {auto,plain,rich}`."
proposed_by: bbugyi200.athena.sl
create_time: 2026-08-03 07:13:04
status: done
---

- **PROMPT:**
  [prompts/202608/bead_show_prose_wrap.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/bead_show_prose_wrap.md)

# Plan: Wrap `sase bead show` prose at a configurable width

## Problem

`sase bead show --format full` emits DESCRIPTION and NOTES bodies verbatim, so a bead whose description is one long
paragraph prints as one enormous line. Measured on a real bead in this repo:

```
$ sase bead show sase-bv --style plain | awk '{ print length }' | sort -rn | head -3
1014
313
121
```

A 1014-column line is soft-wrapped by the terminal at whatever the window happens to be, with no indent continuation and
no control over where breaks land — the exact opposite of the scannable block the styling work
(`sase/repos/plans/202608/bead_show_styling.md`) set out to build. Every other Markdown artifact SASE writes is already
wrapped at 120 columns by Prettier (`Justfile:304`, `--prose-wrap=always --print-width=120`); the most-used bead read
command is the surface that never learned to.

Two smaller defects come along with it:

- **`--style color` is a rung nobody can see.** `DetailPalette.for_style` enables the same palette for `COLOR` and
  `RICH` (`src/sase/bead/cli_detail_style.py:66-69`); the only difference is whether `highlight_prose` tints the prose.
  Diffing the two levels on `sase-bv` shows they differ on **2 of 32 lines**, and the difference is a bare `\x1b[49m`
  background-reset with no actual token coloring, because that bead's description is ordinary prose with no Markdown
  syntax to highlight. Typical beads are ordinary prose. The rung costs an enum member, a docs row, and a column in the
  gate matrix to deliver a distinction users cannot perceive.
- **`-S` is the odd one out.** Every other short alias in the bead CLI is lowercase. `-S` was chosen defensively
  (`bead_show_styling.md:89-93`) to avoid colliding with `-s/--status` on the sibling `list`/`ready`/`blocked`/`search`
  subcommands — but `sase bead show` has no `--status` option, so `-s` is unambiguous there.

And one wart this feature cannot leave alone: `render_issue_detail` emits `f"  {description}"` as a single list element
(`src/sase/bead/cli_detail.py:231-246`), so **only the first line of a multi-line description is indented** and every
continuation line sits flush-left. The prior tale documented this deliberately and deferred it
(`bead_show_styling.md:76-79`) because fixing it changes plain bytes. Wrapping without fixing it would produce a block
whose first line is indented two columns and whose freshly created wrapped lines are not — so it is in scope here.

## Goal

Make long bead prose readable at a predictable, user-chosen column budget; never damage the content while doing it; and
tighten the two CLI rough edges next to it. Beautiful means: paragraphs break where a human would break them, list items
keep a hanging indent under their own text, code and tables come through untouched, and a URL is always one
copy-pasteable token.

## The central invariant, extended

The styling tale's invariant was `strip_sgr(render(rich)) == render(plain)`. Wrapping is **layout, not styling**, so the
invariant generalizes rather than weakens — it now holds per width:

```
for every width W (including "no wrapping"):
    strip_sgr(render(style=rich, wrap=W)) == render(style=plain, wrap=W)
```

Two further properties make wrapping itself safe, and both are directly testable:

1. **Content preservation.** `"".join(wrap(t, W).split()) == "".join(t.split())` — every non-whitespace character
   survives, in order. Nothing is hyphenated, inserted, or dropped; the only edits are "replace one whitespace run with
   a newline plus indent."
2. **Break-only, never reflow.** Short lines are never joined into longer ones. A line already within budget is emitted
   **byte-for-byte unchanged**.

Property 2 is the reliability keystone and it is worth being explicit about why:

- **Wrapping is idempotent.** `wrap(wrap(t, W), W) == wrap(t, W)`, so re-showing a bead never re-breaks prose.
- **Author line structure survives.** Bead descriptions are written by agents and humans who use semantic line breaks;
  reflowing would silently rewrite their paragraphs. Prettier's `--prose-wrap=always` reflows because it is formatting a
  file it owns. We are rendering someone else's text and must not.
- **The blast radius on the existing text contract is tiny and verified.** No line in any `tests/test_bead/golden/cli/`
  fixture exceeds 118 columns today, so break-only wrapping leaves every golden `.stdout` byte-identical. Only content
  that was already unreadable changes.

### Deliberate non-goals

- **No reflowing/joining of short lines** (above).
- **No wrapping outside prose.** Child rows, `DEPENDS ON`/`BLOCKS` rows, `PLAN`/`REFS` paths, and the title line stay
  unwrapped. Those are structured, column-scanned lines; a wrapped 121-column plan path stops being copy-pasteable, and
  a wrapped child row loses the `[STATUS]` column that makes the section skimmable.
- **No hyphenation and no mid-token breaking.** A token wider than the budget overflows rather than being split.
- **No terminal-width magic on the default.** `--wrap 120` means 120 on an 80-column terminal too. `--wrap auto` is the
  option for people who want the terminal to decide (see [CLI design](#cli-design)).
- **No re-rendering of Markdown.** Same constraint as the styling tale: tint and break, never re-emit.

## CLI design

### `-w/--wrap`

```
-w, --wrap WIDTH   Wrap DESCRIPTION and NOTES prose at WIDTH columns: an integer
                   >= 20, 'auto' to fit the terminal, or 'none' to disable
                   (default: 120)
```

Registered on `sase bead show` in `register_bead_show_parser` (`src/sase/main/parser_bead_queries.py:223-274`), placed
after `--style` so the option list stays alphabetically sorted — `color`, `format`, `style`, `wrap` — per
`sase/memory/cli_rules.md`. `-w` is free on this subcommand and reads as "wrap".

| Value        | Meaning                                                                                        |
| ------------ | ---------------------------------------------------------------------------------------------- |
| `<int >= 20` | Wrap so no rendered line exceeds that many columns. **Default: `120`.**                        |
| `auto`       | Use the terminal width (`shutil.get_terminal_size(fallback=(80, 24)).columns`), floored at 20. |
| `none`       | No wrapping. Today's exact bytes.                                                              |
| `0`          | Accepted as a synonym for `none`; `none` is the canonical spelling shown in help and docs.     |

`1`–`19` is a parse error (`argparse.ArgumentTypeError`), not a silent clamp: a budget that narrow cannot hold a hanging
indent plus a word, and quietly ignoring what the user typed is worse than telling them.

**The number is a total column budget, not a text width.** `--wrap 120` means no rendered output line exceeds 120
columns, so the two-space `DESCRIPTION` indent is subtracted before wrapping (content width 118), and the four-space
`+1` evidence-note indent likewise (116). "Wrap at 120" should mean what a user checking with `awk '{print length}'`
expects it to mean.

Default `120` rather than `auto` because it is what the user asked for, because it matches the width every other SASE
Markdown artifact is already wrapped to, and because a fixed default is reproducible: piping to a file gives the same
bytes as the terminal. `auto` exists for people who want the other trade, and a config default is a
[follow-up](#follow-ups).

### Scope of `--wrap` across formats

- `--format full` — the point of the option.
- `--format compact` — no effect. One structured row, no prose.
- `--format json` — **never** wrapped. Machine output stays machine output, exactly as with `--style`.

`--wrap` is independent of `--color` and `--style`: layout is not styling, so `--style plain --wrap 120` wraps and
`--wrap none --style rich` highlights without wrapping. Agents that pipe the command get wrapped-at-120 plain text by
default, which is consistent with every Markdown file in the repo; `--wrap none` restores the pre-change bytes exactly.

### `-s/--style {auto,plain,rich}`

Two changes, both from the user:

- `-S` becomes `-s`. No hidden `-S` alias is kept: argparse would print it in the help output, and the option shipped on
  2026-08-01 (`6e8029b7b`), two days before this plan, so there is no installed base to protect. A clean break beats a
  permanent second spelling.
- The `color` value is removed, along with `DetailStyle.COLOR`. `resolve_detail_style` becomes: gate closed → `PLAIN`;
  `plain` → `PLAIN`; `auto` or `rich` → `RICH`.

Worth stating plainly since it contradicts the prior tale: `-s` is safe **on this subcommand** because `sase bead show`
has no `--status`, but it does mean `-s` carries one meaning on `show` and another on `list`/`ready`/`blocked`/`search`.
That is a real (small) intuition cost, accepted deliberately in exchange for a consistently lowercase bead CLI.

`DetailStyle` keeps its `StrEnum` shape with two members rather than collapsing to a bool — `DetailPalette.for_style`,
`highlight_prose`, and the `--style` values all read better against a named level, and a future rung stays cheap.

## Wrapping algorithm

New module `src/sase/markdown_wrap.py`. Generic, Markdown-structure-aware, bead-agnostic — the same split that puts
`ansi_style.py` at the top level under `cli_detail_style.py`.

```python
DEFAULT_PROSE_WRAP_WIDTH = 120
MIN_PROSE_WRAP_WIDTH = 20

def wrap_markdown(text: str, *, width: int) -> str: ...
```

### Module constraints

**`markdown_wrap.py` must import only the standard library** (`re`, `unicodedata`).
`src/sase/main/parser_bead_common.py` needs `DEFAULT_PROSE_WRAP_WIDTH` and `MIN_PROSE_WRAP_WIDTH` at parser-registration
time, and parser modules are imported on every `sase` invocation. Verified: importing `sase.main.parser_bead_queries`
today loads **0** `rich` modules, while importing `sase.file_references` loads **51**. Do not reach for
`rich.cells.cell_len` (nicer emoji handling, wrong trade here) and do not import `file_references` for its
`DEFAULT_MARKDOWN_WRAP_WIDTH`; instead pin the two constants together with a unit test (see [Tests](#tests)), the same
drift-guard pattern the styling tale used for `PHASE_SIZE_ACCENTS`.

Column width is measured with a local `_cell_width(text)` helper: `unicodedata.combining(ch)` truthy → 0 columns,
`unicodedata.east_asian_width(ch) in {"W", "F"}` → 2 columns, else 1. `len()` would mis-measure CJK titles and emoji.

### Line classification

Process `text.split("\n")` in order, tracking fenced-code state. A line is emitted **verbatim** when any of these hold —
each rule fails safe, leaving a long line long rather than damaging content:

| Rule                                                                                                                                         | Why                                                                                                                           |
| -------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| Inside (or delimiting) a fenced code block — a ` ``` ` or `~~~` run opens, a run of the same character of at least the opening length closes | Code is line-significant                                                                                                      |
| `_cell_width(line) <= width`                                                                                                                 | The break-only guarantee; the overwhelmingly common case                                                                      |
| Table row: `^\s*\|`                                                                                                                          | Wrapping destroys column alignment                                                                                            |
| Contains a tab                                                                                                                               | Tab stops make column math a lie; matches the existing refusal in `src/sase/memory/notes.py:276-277`                          |
| Unbalanced inline-code delimiters (odd number of backtick runs)                                                                              | A code span opened on this line and closes on a later one; breaking inside it is exactly what this feature promises not to do |
| Indented code: leading indent >= 4 columns **and** not a list-item or blockquote line                                                        | Protects indented code blocks while still wrapping nested bullets                                                             |

Otherwise the line is wrapped with a computed continuation prefix:

| Line shape                              | Continuation prefix                                                                  |
| --------------------------------------- | ------------------------------------------------------------------------------------ |
| List item `^(\s*)([-*+]\|\d+[.)])(\s+)` | Spaces equal to the full match's column width — a hanging indent under the item text |
| Blockquote `^(\s*>+\s?)+`               | The same quote prefix repeated, so the quote stays a quote                           |
| ATX heading `^\s*#{1,6}\s+`             | The leading whitespace only — a continuation line must not start with `#`            |
| Anything else                           | The line's own leading whitespace, preserving nesting                                |

If the continuation prefix would leave fewer than `MIN_PROSE_WRAP_WIDTH` content columns, fall back to the line's plain
leading whitespace; if that is still too narrow, emit the line verbatim.

### Atoms — the never-split units

Tokenize the wrappable content into **atoms** (never broken) separated by whitespace **gaps**, then fill greedily. The
atom patterns, matched in this order:

1. **Inline code spans** — ``(`+)(?:(?!\1).)+?\1`` : an opening run of _n_ backticks closed by a run of exactly _n_.
   Covers `` `foo bar baz` `` and ``` ``a ` b`` ```.
2. **Markdown links and images** — `!?\[[^\]]*\]\([^)\s]*(?:\s+"[^"]*")?\)`, taken whole. Breaking between `]` and `(`
   would produce invalid Markdown; treating the construct as one atom is simpler and always correct. Link texts are
   short in practice, and an over-wide one simply overflows.
3. **Autolinks** — `<[^>\s]+>`.
4. **Bare URLs** — `(?:https?|ftp|file)://\S+` and `www\.\S+`.
5. Everything else splits on whitespace runs, so ordinary long words, file paths, and bead IDs are naturally atomic —
   they contain no spaces.

Fill rule: append `gap + atom` while the result fits `width`; otherwise break, emit the continuation prefix, and
continue. Gaps inside a kept segment are preserved verbatim (double spaces, aligned columns), and the gap consumed by a
break disappears. Trailing whitespace is stripped from every emitted segment **except** the final segment of a source
line, so a Markdown hard break (a line ending in two spaces) survives.

The guarantee, stated so a test can assert it: **no output line exceeds `width` unless it consists of a single atom that
alone exceeds `width`** (plus its continuation prefix).

## Rendering integration

### `src/sase/bead/cli_detail.py`

`render_issue_detail` gains a keyword-only `wrap: int | None = None` (`None` = no wrapping), so `_render_list_full` and
`_render_search_full` (`src/sase/bead/cli_query.py:302-318,453-476`) keep today's behavior with no edit — the same
opt-in shape `style` already uses.

All three prose sites route through one helper:

```python
def _prose_lines(text: str, *, style: DetailStyle, wrap: int | None, indent: str) -> list[str]:
    body = text
    if wrap is not None:
        content_width = wrap - len(indent)
        if content_width >= MIN_PROSE_WRAP_WIDTH:
            body = wrap_markdown(text, width=content_width)
    plain_lines = body.split("\n")
    styled_lines = highlight_prose(body, style=style).split("\n")
    if len(styled_lines) != len(plain_lines):   # defensive; must not happen
        styled_lines = plain_lines
    return [f"{indent}{styled}" if plain else "" for plain, styled in zip(plain_lines, styled_lines)]
```

Call sites: `DESCRIPTION` and `NOTES` with `indent="  "`, and `_render_plus_one_evidence_lines`
(`src/sase/bead/cli_detail.py:334-335`, which already splits and indents by four) with `indent="    "`.

Four load-bearing details in those nine lines:

- **`.split("\n")`, not `.splitlines()`.** A description stored with a trailing newline renders a genuinely empty line
  today; `splitlines()` would swallow it and change bytes for a case that has nothing to do with wrapping.
- **The emptiness test reads the _plain_ line, and empty lines are emitted as `""`.** In `RICH` mode `highlight_prose`
  appends `ANSI_RESET` to every line (`src/sase/bead/cli_detail_prose.py:125-126`), so a blank line arrives as
  `"\x1b[0m"` — truthy. Testing the styled line would indent it to `"  \x1b[0m"`, which strips to `"  "` against a plain
  `""` and **breaks the central invariant**. There is nothing on a blank line to reset, so dropping the escape is free.
- **Wrap first, then highlight.** Wrapping stays ANSI-unaware (no SGR-state tracking across breaks), the invariant holds
  by construction at every width, and `highlight_prose` needs no change at all.
- **Indent every line.** This is the flush-left continuation fix, and it applies identically to authored line breaks and
  to wrap-created ones.

### `src/sase/bead/cli_query.py`

In `handle_bead_show`, resolve once next to the existing style resolution and pass it into the `full` branch only:

```python
wrap = resolve_wrap_width(getattr(args, "wrap", DEFAULT_PROSE_WRAP_WIDTH))
```

`compact` and `json` ignore it.

### `src/sase/main/parser_bead_common.py`

```python
def wrap_width(value: str) -> int | None:
    """Parse ``--wrap``: 'none'/'0' -> None, 'auto' -> -1 sentinel, else an int >= 20."""
```

Validation belongs in the argparse `type=` callable so a bad value produces a usage error and exit 2, not a traceback
mid-render. Return a normalized token — `None` for disabled, an `int` for an explicit width, and a named sentinel for
`auto` (`WRAP_AUTO`), keeping the terminal lookup out of parse time and therefore out of unit tests.
`resolve_wrap_width` (next to it, or in `cli_detail_style.py` beside `resolve_detail_style` — implementer's call, but
keep both in one place) turns the token into `int | None`, resolving `auto` through
`shutil.get_terminal_size(fallback=(80, 24))` and flooring at `MIN_PROSE_WRAP_WIDTH`.

### `src/sase/bead/cli_detail_style.py`

Delete `DetailStyle.COLOR` and its branch in `resolve_detail_style`. `DetailPalette` is unchanged — it already keys off
`style is not DetailStyle.PLAIN`.

### No Rust change

`sase bead show` is excluded from the Rust fast path (`src/sase/main/bead_fast_path.py:35`), so the Python renderer is
the only implementation reached at runtime, and terminal presentation is the category the boundary rule in `CLAUDE.md`
keeps in this repo. Note for the record: `sase-core`'s parity renderer emits the same unwrapped
`"\nDESCRIPTION\n  {}\n"` (`crates/sase_core/src/bead/cli.rs:311`) and will drift further behind — it has no golden
tests of its own, and a [follow-up](#follow-ups) tracks it. **Do not edit `sase-core` in this tale.**

## Tests

New `tests/test_bead/test_markdown_wrap.py` for the algorithm, plus additions to
`tests/test_bead/test_cli_show_style.py` (which already owns `strip_sgr` and the fixture beads).

**Algorithm properties** — parametrized over widths `{20, 40, 80, 120}` and a corpus of prose shapes:

1. **Content preservation**: `"".join(wrap(t, W).split()) == "".join(t.split())` for every corpus entry.
2. **Idempotency**: `wrap(wrap(t, W), W) == wrap(t, W)`.
3. **Width honored**: every output line is within `W`, _or_ is a single atom plus its continuation prefix.
4. **Break-only**: text whose lines are all within `W` comes back byte-identical (assert on a multi-line fixture).
5. **URLs never split**: a paragraph with a 150-character `https://…` URL yields that URL intact on one line.
6. **Inline code never split**: `` `foo bar baz` `` never straddles a break, including a double-backtick span containing
   a backtick, and a line with an unbalanced backtick run is passed through verbatim.
7. **Markdown links never split**: `[some text](https://example.com/very/long)` survives whole.
8. **Fenced code verbatim**: fence bodies (including long lines, ` ``` ` and `~~~` fences, and an unterminated fence)
   are unchanged.
9. **Tables and tab-bearing lines verbatim.**
10. **Hanging indents**: `-`, `*`, `+`, `1.`, and `1)` items align continuations under the item text; nested items keep
    their nesting; blockquote continuations keep `> `; a heading continuation does not start with `#`.
11. **CJK/emoji width**: a line of wide characters wraps by columns, not by `len`.
12. **Degenerate inputs**: empty string, whitespace-only, a single over-wide atom, `width` below `MIN_PROSE_WRAP_WIDTH`.

**Renderer and CLI**:

13. **The extended invariant**: over the existing bead corpus × wrap widths `{None, 40, 120}`,
    `strip_sgr(render(RICH, wrap=W)) == render(PLAIN, wrap=W)`. Includes the bead whose description contains a literal
    `\x1b[31m`, `[rich markup]`, and `:emoji_shortcodes:`.
14. **Blank lines stay blank**: a description with an interior blank line renders `""`, not `"  "` and not
    `"  \x1b[0m"`, at every style — the regression guard for the trap above.
15. **Continuation indent**: a multi-line description renders **every** line at two columns. This intentionally replaces
    the old flush-left behavior; any golden asserting the old shape is updated in this change, with the reason stated in
    the commit message.
16. **Trailing newline**: a description ending in `"\n"` renders the same bytes as today.
17. **`+1` evidence notes** wrap at four-space indent and stay within budget.
18. **Total-budget semantics**: with `--wrap 60`, no line of the DESCRIPTION block exceeds 60 columns (indent included).
19. **`--wrap none` / `--wrap 0`** reproduce the pre-change bytes for a long-line bead.
20. **`--wrap auto`** uses the terminal width, monkeypatching `shutil.get_terminal_size`.
21. **Parse errors**: `--wrap 19`, `--wrap -5`, `--wrap wide` exit 2 with a usage message.
22. **JSON is never wrapped**: `--format json --wrap 40` parses as JSON and the description round-trips unchanged.
23. **`--style` surface**: `-s rich` works, `-S rich` is now an error, `--style color` is now an error, and the gate
    matrix test drops its `color` rows.
24. **Constant drift**: `DEFAULT_PROSE_WRAP_WIDTH == file_references.DEFAULT_MARKDOWN_WRAP_WIDTH`.
25. **Existing goldens**: `tests/test_bead/golden/cli/show*.stdout` must still pass unedited (verified: no fixture line
    exceeds 118 columns). If one needs editing for any reason other than test 15, the break-only guarantee has been
    violated — fix the wrapper, not the fixture.

## Docs and help

- `sase bead show --help`: add `-w/--wrap` after `--style` with the value list and `(default: 120)`; update `--style` to
  the three remaining values; add an example — `sase bead show sase-64 --wrap auto`. Keep options alphabetical.
- Extend the subcommand `description` to say prose is wrapped at 120 columns by default and that URLs and inline code
  spans are never broken.
- `docs/beads.md:847-871`: drop the `color` row from the `--style` table, retitle `-S` to `-s`, and add a `--wrap`
  section covering the value list, the total-column-budget semantics, the break-only guarantee, and what is never
  wrapped (code fences, tables, URLs, inline code, structured rows).
- `docs/cli.md` needs no change (it links to `beads.md`).

## Verification

```bash
just install     # ephemeral workspace: dependencies may be stale
just check
```

Then on a real, long-prose bead:

Capture a baseline **before** touching any source file, so the `--wrap none` escape hatch can be proven byte-exact:

```bash
sase bead show sase-bv --style plain > /tmp/show_before.txt
```

Then, after the change:

```bash
sase bead show sase-bv | awk '{ print length }' | sort -rn | head -3   # nothing above 120
sase bead show sase-bv --wrap 80 | awk 'length > 80'                   # only lone long URLs, if any
sase bead show sase-bv --style plain --wrap none | diff - /tmp/show_before.txt
sase bead show sase-bv --wrap auto
sase bead show sase-bv --style rich --color always | sed 's/\x1b\[[0-9;]*m//g' \
  | diff - <(sase bead show sase-bv --style plain)
```

The two `diff`s must print nothing — the first modulo the multi-line indent fix, the second at every `--wrap` value.
Paste a terminal screenshot of a long-description bead at `--wrap 120` and at `--wrap 80` into the PR so the hanging
indents and the untouched URLs can be reviewed by eye.

## Follow-ups

File as `task` beads via `/sase_new_task` rather than expanding this tale:

- `sase bead list --format full` and `sase bead search --format full` should accept `-s/--style` and `-w/--wrap`; the
  renderer parameters added here make each a one-line change.
- A `bead.show.wrap` (and `bead.show.style`) default in `~/.config/sase/sase.yml` / `src/sase/default_config.yml`, so
  someone who always wants `auto` need not pass the flag.
- `sase-core`'s `handle_show` parity renderer (`crates/sase_core/src/bead/cli.rs:311`) now trails the Python renderer by
  both the indent fix and prose wrapping; decide whether to port or to delete the deferred parity path.
- `src/sase/xprompt/cli_show_render.py` renders prose bodies with the same shared palette and has the same unbounded
  line problem; `wrap_markdown` is reusable there.
