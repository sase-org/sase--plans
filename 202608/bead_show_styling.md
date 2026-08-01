---
tier: tale
title: Colorize and syntax-highlight `sase bead show`
goal:
  "`sase bead show` renders a beautiful, semantically colored detail block with markdown/code syntax highlighting in
  DESCRIPTION and NOTES, controlled by a new `-S/--style` option, while the plain-text skeleton stays byte-for-byte
  identical to today's output."
proposed_by: bbugyi200.athena.r6
create_time: 2026-08-01 09:04:08
status: wip
---

- **PROMPT:** [202608/prompts/bead_show_styling.md](prompts/bead_show_styling.md)

# Plan: Colorize and syntax-highlight `sase bead show`

## Problem

`sase bead show <id> --format full` prints a flat, monochrome block. Every line — section headers, field labels, bead
IDs, statuses, sizes, paths, and free-form markdown prose — renders in the same undifferentiated foreground color, so
scanning a large epic bead is slow and the structure has to be re-parsed by eye every time.

Two facts make this worse than it looks:

- `sase bead show` already accepts `-c/--color {auto,always,never}` (`src/sase/main/parser_bead_queries.py:246`), but
  that option is a **no-op for `--format full`**: `handle_bead_show` only threads `use_color` into the `compact` branch
  (`src/sase/bead/cli_query.py:128-165`). The advertised flag silently does nothing for the default format. That is a
  truthfulness bug independent of this feature.
- `sase bead list`, `sase bead dep tree`, and `sase bead search` already emit colored rows through
  `sase/bead/cli_dep_render.py` and the shared presentation modules, so the _default_ format of the _most-used_ read
  command is the one surface that stayed black-and-white.

## Goal

Make `sase bead show` beautiful and fast to scan, add real syntax highlighting to its prose sections, and expose one new
CLI option that controls how much styling is applied — without breaking the plain-text contract that agents, golden
tests, and the in-flight Rust port depend on.

## The central invariant

**Styling is purely additive ANSI. The plain-text skeleton never changes.**

For any bead and any styling level:

```
strip_sgr(render(style=rich)) == render(style=plain)
```

Same lines, same words, same order, same spacing, same indentation quirks. Color and highlighting only add SGR escape
sequences around characters that were already there.

This is the design decision everything else follows from, and it is what makes the feature _reliable_ rather than merely
pretty:

- Agents run `sase bead show` constantly (see `src/sase/default_config.yml:921,937,986`). They get a non-TTY, so they
  keep getting today's exact bytes. Zero prompt-behavior risk.
- `tests/test_bead/golden/cli/show.stdout` and friends stay valid unchanged. Those fixtures exist specifically to "pin
  the current public text contract before the bead backend moves into `sase-core`"
  (`tests/test_bead/test_cli_golden.py:3-6`) — this feature must not move that target.
- The invariant is directly testable as a single property over a corpus of beads, which is a far stronger safety net
  than snapshotting styled output alone.

### What the invariant rules out (deliberate non-goals)

These were considered and rejected, because each one breaks the invariant for a cosmetic gain:

- **Panels / boxes / horizontal rules** around sections. Also breaks copy-paste and reflows badly on narrow terminals.
- **Re-layout of markdown** via `rich.markdown.Markdown` (re-renders bullets, re-wraps paragraphs, boxes code fences).
  Use `rich.syntax.Syntax(...).highlight()` instead, which tints tokens _in place_ and preserves every character — the
  same technique already used at `src/sase/ace/tui/modals/_prompt_stash_preview.py:39`.
- **OSC 8 terminal hyperlinks** on URLs and paths. Non-SGR escapes would survive a simple SGR strip and complicate the
  invariant test for a marginal benefit.
- **Chips / background-colored badges** for status and size (as the TUI uses). Background fills look wrong against
  arbitrary terminal themes in a scrollback buffer, and padding a chip changes column positions. Foreground accents
  only.
- **Re-indenting multi-line descriptions.** Today `render_issue_detail` emits `f"  {issue.description}"`, so only the
  _first_ line of a multi-line description is indented and continuation lines sit flush-left
  (`src/sase/bead/cli_detail.py:267-270`). That is a real wart, but fixing it changes plain bytes. Preserve it exactly
  and file a follow-up bead (see [Follow-ups](#follow-ups)).

## CLI design

### The new option

```
-S, --style {auto,plain,color,rich}    Styling level (default: auto)
```

Registered on `sase bead show` in `register_bead_show_parser` (`src/sase/main/parser_bead_queries.py:223-260`), placed
after `--format` so the option list stays alphabetically sorted (`color`, `format`, `style`) per
`sase/memory/cli_rules.md`. The uppercase short alias `-S` satisfies the "every public long option gets a short alias"
rule while avoiding `-s`, which means `--status` on the sibling `list`/`ready`/`blocked` subcommands — reusing it for a
different meaning here would be an intuition trap.

Levels, as an escalating ladder:

| Level   | Meaning                                                                                                                           |
| ------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `auto`  | Resolve to `rich` when color is enabled, else `plain`. **Default.**                                                               |
| `plain` | No ANSI at all, regardless of `--color`. Today's exact bytes.                                                                     |
| `color` | Semantic palette on structural chrome (glyphs, IDs, section headers, labels, statuses, sizes, paths). Prose bodies left untinted. |
| `rich`  | `color` plus markdown/code syntax highlighting inside DESCRIPTION and NOTES.                                                      |

`color` exists as its own rung because markdown tinting inside dense prose is a genuine taste call: some readers want
the structural hierarchy without the inline noise. It costs one enum member.

### How `--style` and `--color` compose

One sentence in the help text, and it must be exactly this rule:

> `--color` decides **whether** ANSI may be emitted; `--style` decides **how much** styling to apply.

`--color` is the gate and always wins:

| `--color` | `--style` | stdout is a TTY | Result |
| --------- | --------- | --------------- | ------ |
| `never`   | any       | either          | plain  |
| `auto`    | `auto`    | no              | plain  |
| `auto`    | `auto`    | yes             | rich   |
| `auto`    | `rich`    | no              | plain  |
| `auto`    | `plain`   | yes             | plain  |
| `always`  | `auto`    | either          | rich   |
| `always`  | `color`   | either          | color  |
| `always`  | `plain`   | either          | plain  |

`NO_COLOR` is honored because the gate reuses the existing `resolve_color` (`src/sase/bead/cli_dep_render.py:28-36`)
unchanged. To force styling into a pipe, use `--color always`, exactly as with every other SASE command — nothing new to
learn.

### Scope of `--style` across formats

- `--format full` — the whole point; all levels apply.
- `--format compact` — `plain` forces no ANSI; `color` and `rich` are identical (there is no prose in a compact row).
  Keeps the two flags coherent instead of having `--style plain --format compact` silently emit color.
- `--format json` — **never** styled at any level. Machine output stays machine output.

### Also fixed here

`--color` starts actually working for `--format full`. That is the pre-existing no-op described above; leaving it broken
while adding a second styling flag next to it would be indefensible.

## Palette

Restraint is what makes this beautiful. Three structural hues plus the existing semantic colors, and everything else is
dim. No new color is invented where a shared presentation module already owns one.

### Reused, never duplicated

| Element                                  | Source                                                                                            |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------- |
| Status glyph + status label color        | `bead_status_presentation(...).cli_style` (`src/sase/bead_status_presentation.py`)                |
| Type value color (`plan`/`phase`/`task`) | `bead_type_presentation(...).cli_style` (`src/sase/bead_type_presentation.py`)                    |
| Bead IDs                                 | `ANSI_BOLD_BLUE` (`src/sase/bead/cli_dep_render.py:12`) — same as `bead list` rows and `dep tree` |
| Phase size value                         | new accent added to `src/sase/phase_size_presentation.py` (below)                                 |

### New chrome roles

| Role          | Style                | Applies to                                                                                                                                                       |
| ------------- | -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `TITLE`       | bold, no hue         | The bead title on the header line — the brightest thing on screen, hueless so it never fights the accents                                                        |
| `SECTION`     | bold `#5F87FF`       | `CREATED BY`, `PAGE`, `RESOLUTION`, `PARENT`, `CHILDREN`, `DEPENDS ON`, `BLOCKS`, `DESCRIPTION`, `NOTES`, `CHANGESPEC`, `PLAN`/`EPIC PLAN`/`PARENT PLAN`, `REFS` |
| `SUBSECTION`  | `#5F87FF` (not bold) | `PHASES`, `CHILD EPICS`                                                                                                                                          |
| `LABEL`       | dim                  | `Type:`, `Tier:`, `Owner:`, `Assignee:`, `Model:`, `Size:`, `Resolution:`, `Close reason:`, `Closed at:`, `Name:`, `Bug ID:`                                     |
| `SEPARATOR`   | dim                  | `·`, `→`, `←`, `↑`, and the `From parent …` connector text                                                                                                       |
| `PATH`        | `#87AFFF`            | design/plan reference lines, REFS lines                                                                                                                          |
| `URL`         | underline `#87AFFF`  | the `PAGE` URL and the `CREATED BY → <url>` line                                                                                                                 |
| `TIER`        | `#FFAF00`            | the `epic` / `plan` tier value                                                                                                                                   |
| `PLACEHOLDER` | dim italic           | `(none)`, `(unknown)`, `(unrecorded)`                                                                                                                            |
| `DANGLING`    | `#FF8787`            | `(not found)` on unresolved parent/dependency/blocker refs — a broken ref should pop, not blend in                                                               |

`#5F87FF` is chosen for section headers precisely because no status, type, or size color uses it, so structure never
reads as semantics. `#87AFFF` matches the TUI's artifact-path family (`COLOR_ARTIFACT_FILE_PATH`,
`src/sase/ace/tui/widgets/prompt_panel/_agent_context_common.py:38`).

Owner, assignee, creator, model, and child titles stay **unstyled**. Coloring them too would turn the block into
confetti.

### Color depth

Emit **xterm-256** foreground codes (`\x1b[38;5;Nm`), not truecolor. This matches the existing convention —
`bead_type_presentation` already derives 256-color codes from its hex accents via `_xterm256_foreground_style` — and
256-color is dramatically more portable across terminals and multiplexers.

## Implementation

### 1. `src/sase/ansi_style.py` (new, public)

Generic ANSI SGR helpers, so the palette is not built from hand-written escape literals scattered across modules:

- `ANSI_RESET`
- `xterm256_foreground_style(hex_color: str) -> str` — promoted from
  `bead_type_presentation._xterm256_foreground_style`. Move the implementation here and have `bead_type_presentation`
  call the public function; do not leave a duplicate copy and do not reach into the private name from a new module.
- `ansi_sgr(*, color: str | None = None, bold: bool = False, dim: bool = False, italic: bool = False, underline: bool = False) -> str`
  — compose one SGR sequence from a hex color plus attributes.
- `apply_ansi(value: str, style: str, *, enabled: bool) -> str` — the `styled()` shape already used in `cli_dep_render`,
  generalized.

Keep `cli_dep_render.ANSI_RESET` / `ANSI_BOLD_BLUE` / `styled` exported under their current names (re-exporting from
`ansi_style` is fine) so existing importers in `cli_query.py` and the dep renderers are untouched.

### 2. `src/sase/phase_size_presentation.py` (extend)

Add a CLI accent per size so `Size: medium` can be colored from the same source of truth the TUI chips use:

- `PHASE_SIZE_ACCENTS: dict[PhaseSizeValue, str]` — `xsmall` `#5FD7AF`, `small` `#87D7FF`, `medium` `#FFD75F`, `large`
  `#D75F87`, `xlarge` `#AF5FFF` (the exact hexes already embedded in `PHASE_SIZE_STYLES`).
- `phase_size_cli_style(value: object) -> str | None` — `xterm256_foreground_style` of the accent, `None` for an
  unrecognized size.

**Do not restructure `PHASE_SIZE_STYLES`.** Its string values are consumed by TUI chips that have PNG visual snapshot
goldens (`tests/ace/tui/visual/snapshots/png/`); changing them would churn binary fixtures for no benefit. Instead add a
unit test asserting each `PHASE_SIZE_ACCENTS` hex is the hex embedded in the corresponding `PHASE_SIZE_STYLES` entry,
which pins the two against drift.

### 3. `src/sase/bead/cli_detail_style.py` (new)

- `DetailStyle` — `StrEnum` of `PLAIN`, `COLOR`, `RICH` (the resolved levels; `auto` never survives resolution).
- `resolve_detail_style(*, style: str, color: str) -> DetailStyle` — implements the composition table above. Calls
  `resolve_color(color)` for the gate. Pure and directly unit-testable, with no I/O beyond the TTY check `resolve_color`
  already performs.
- `DetailPalette` — a frozen dataclass exposing one method per role above (`section(text)`, `label(text)`, `path(text)`,
  `dangling(text)`, …), each returning the input unchanged when the palette is disabled. Construct with
  `DetailPalette.for_style(style)`.

Routing every styling decision through one palette object — rather than sprinkling `if use_color:` through the renderer
— is what keeps `render_issue_detail` readable and keeps the invariant mechanically checkable.

### 4. `src/sase/bead/cli_detail_prose.py` (new)

`highlight_prose(text: str, *, style: DetailStyle) -> str` — markdown/code token tinting for DESCRIPTION and NOTES
bodies. Returns `text` unchanged unless `style is DetailStyle.RICH`.

Implementation notes that are load-bearing, not incidental:

- Use `rich.syntax.Syntax(text, "markdown", theme="ansi_dark", background_color="default").highlight(text)` to get a
  `rich.text.Text` with styles applied over the original characters.
  - `theme="ansi_dark"` maps to the terminal's own 16-color ANSI palette, so it stays readable on light _and_ dark
    backgrounds. A fixed theme like `monokai` is unreadable on a light terminal.
  - `background_color="default"` is required, otherwise `Syntax` paints a background fill across every line.
- **Fenced code blocks**: detect ` ``` ` fences and re-highlight each fence body with the lexer named on the opening
  fence, falling back to the markdown/text lexer. Wrap lexer lookup so `pygments.util.ClassNotFound` on an unknown or
  misspelled language never propagates — an unhighlightable description must still print.
- **Render `Text` back to an ANSI string** through a captured console:
  `Console(file=StringIO(), force_terminal=True, color_system="256", width=<large>, soft_wrap=True, markup=False, emoji=False, highlight=False)`.
  - `markup=False` and `emoji=False` are **mandatory correctness requirements**, not style choices. Bead descriptions
    routinely contain `[brackets]` and `:colons:`; with Rich's defaults those get interpreted as console markup and
    emoji shortcodes and the text is silently mangled. This is the single most likely way to break the invariant.
  - `soft_wrap=True` plus a large width guarantees Rich never re-wraps, pads, or crops.
  - Strip any trailing newline `Syntax.highlight()` adds; the caller owns line joining.
- **Line-by-line emission**: split the highlighted result on `\n` and re-apply the enclosing style at the start of each
  line. A description that itself contains a raw `\x1b[…m` byte cannot then bleed its color past the end of that line.
  (Raw escapes in content are passed through unchanged rather than sanitized — sanitizing would break the byte
  invariant. Document this as a known limitation.)
- Preserve the existing indentation exactly: `"  " + body`, with continuation lines flush-left.

### 5. `src/sase/bead/cli_detail.py` (modify `render_issue_detail`)

Add a keyword-only `style: DetailStyle = DetailStyle.PLAIN` parameter and build each line through the palette.
Defaulting to `PLAIN` means `_render_list_full` (`src/sase/bead/cli_query.py:265-282`) keeps its current behavior with
no change, and `sase bead list --format full` can opt in later by passing one argument.

Traps to respect while editing:

- The header uses `issue.status.value.upper()` (`OPEN`, `IN_PROGRESS`), but `bead_status_presentation(...).label`
  renders `ready` as `Ready`. Use the presentation **only for the color**; keep `.value.upper()` for the text or the
  plain bytes change.
- The plan-reference and REFS sections come back as pre-formatted line tuples from `describe_design_reference` and
  `artifact_ref_list_display_lines`. Style those **whole lines** with the `PATH` role rather than trying to parse their
  internals.
- Every currently emitted blank line, two-space indent, four-space indent, and the three-space gap before `[STATUS]`
  must survive verbatim.

### 6. `src/sase/bead/cli_query.py` (modify `handle_bead_show`)

Resolve the style once via `resolve_detail_style(style=args.style, color=args.color)` and pass it to
`render_issue_detail` for `full` and to the compact renderer's `use_color` for `compact`
(`use_color = resolved is not DetailStyle.PLAIN`). The `json` branch ignores it entirely.

### 7. No Rust change is required

`sase bead show` is explicitly excluded from the Rust fast path (`src/sase/main/bead_fast_path.py:35`), so the Python
renderer is the only implementation reached at runtime. `sase-core`'s `handle_show`
(`crates/sase_core/src/bead/cli.rs:172`) is an in-progress parity implementation that currently defers, and because the
plain skeleton is unchanged, its parity target is unaffected by this work. Per the Rust-boundary rule in `CLAUDE.md`,
terminal presentation is exactly the category that stays in this repo — consistent with `bead_status_presentation.py`
and `bead_type_presentation.py` already living here and being _mirrored_ into Rust
(`crates/sase_core/src/bead/cli.rs:2144-2151`).

Do **not** edit `sase-core` as part of this tale.

## Tests

New file `tests/test_bead/test_cli_show_style.py`, plus targeted additions to the existing
`tests/test_bead/test_cli_show.py` helpers.

Test-only ANSI strip helper: `re.compile(r"\x1b\[[0-9;]*m")` — SGR only, following the local pattern at
`tests/ace/tui/artifact_file_viewer/_helpers.py:13`.

1. **The invariant (highest value test).** Parametrized over a corpus of at least ten bead shapes: minimal task; epic
   with phases and child epics; phase with parent-epic plan; closed bead with resolution/close reason/closed at; bead
   with dependencies and blockers; bead with _dangling_ parent/dependency/blocker refs; bead with a ChangeSpec; bead
   with REFS; bead with a multi-line markdown description containing headings, bullets, and a fenced code block; bead
   whose description contains `[rich markup]`, `:emoji_shortcodes:`, and a literal `\x1b[31m`; bead with a CJK/emoji
   title. For each: assert `strip_sgr(render(RICH)) == render(PLAIN)` and `strip_sgr(render(COLOR)) == render(PLAIN)`.
2. **No stray escapes.** `render(RICH)` contains no `\x1b` byte other than SGR sequences (guards against OSC 8 links and
   background fills sneaking in).
3. **Plain is silent.** `--style plain` output contains no `\x1b` at all, even with `--color always`.
4. **Gate matrix.** Table-driven over every row of the composition table, monkeypatching `sys.stdout.isatty` and
   `NO_COLOR`, asserting the resolved `DetailStyle`.
5. **Default safety.** With a non-TTY stdout (the pytest default) `sase bead show` emits zero `\x1b` bytes — this is the
   regression guard for every agent that shells out to the command.
6. **JSON is never styled.** `--format json --style rich --color always` output parses as JSON and contains no `\x1b`.
7. **ANSI snapshots.** Golden `.ansi` fixtures under `tests/test_bead/golden/` for two representative beads (an epic
   with children and a closed phase with a markdown description), rendered at `rich`. These pin the palette so an
   accidental role/color change is caught in review.
8. **Prose robustness.** `highlight_prose` unit tests: unknown fence language, unterminated fence, empty string, text
   with trailing whitespace, and a description containing only a fence.
9. **Palette drift.** `PHASE_SIZE_ACCENTS` hexes match `PHASE_SIZE_STYLES`.
10. **Existing goldens unchanged.** `tests/test_bead/golden/cli/show*.stdout` must not be edited. If a change to those
    files seems necessary, the invariant has been broken — fix the renderer, not the fixture.

## Docs and help

- `sase bead show --help`: add `-S/--style` with the level list, the one-sentence gate-vs-level rule, and an example
  line — `sase bead show sase-64 --style rich --color always` (piping-friendly). Keep the option list alphabetically
  sorted.
- Extend the subcommand `description` to note that `--color` now applies to `--format full`.
- `docs/beads.md` (`#cli-commands`): document the option, the four levels, and the composition rule.
- `docs/cli.md:112` row for `sase bead show` needs no change (it links to `beads.md`).

## Verification

```bash
just install          # ephemeral workspace: dependencies may be stale
just check
```

Then eyeball the real thing on a real bead in a real terminal at each level:

```bash
sase bead show <id>
sase bead show <id> --style color
sase bead show <id> --style plain
sase bead show <id> --style rich --color always | less -R
sase bead show <id> --style rich --color always | sed 's/\x1b\[[0-9;]*m//g' | diff - <(sase bead show <id> --style plain)
```

That last command must print nothing. Include a screenshot or pasted terminal output in the PR description so the
palette can be reviewed visually, not just diffed.

## Follow-ups

File these as `task` beads (`sase bead create -T task …`, then `-s ready`) rather than expanding this tale:

- `sase bead list --format full` should accept the same `-S/--style` option; the renderer parameter added here makes it
  a one-line change.
- The multi-line-description indentation wart (continuation lines are flush-left) should be fixed with the golden
  fixtures updated in the same change — deliberately out of scope here because it breaks the byte invariant this tale
  relies on.
- A `bead.show.style` default in `~/.config/sase/sase.yml` / `src/sase/default_config.yml`, so users who always want
  `color` need not pass the flag.
- `resolve_color` ignores `TERM=dumb` and `FORCE_COLOR`; worth aligning with the wider ecosystem, but it is shared by
  `list`/`search`/`dep` and deserves its own change.
