---
create_time: 2026-07-09 14:02:05
status: done
prompt: 202607/prompts/beautiful_vcs_log_tags.md
tier: tale
---
# Plan: Make `sase vcs log` SASE Tags Beautiful

## Goal

`sase vcs log -t/--tags` already surfaces the trailing `SASE_*` commit-footer metadata, but it renders that metadata as
flat, uniformly-`dim` machine syntax — `subject  · TYPE=sdd · PLAN=sdd/foo.md  · bryan`. The `KEY=VALUE` form leaks the
raw footer grammar into an otherwise carefully typeset log.

This plan redesigns how those tags **look** in the two human-facing colored formats (`pretty` and `full`), turning each
tag into a small, semantic, colored "chip" whose glyph and color carry meaning at a glance. The redesign is purely
presentational: no new CLI options, no wire-schema change, no `sase-core` change, and default (no `-t`) output stays
byte-for-byte identical.

## Design Lead Summary (what "beautiful" means here)

Three ideas drive the look:

1. **Glyphs and color replace key noise.** SASE writes a small, known set of footer keys. For the high-frequency
   identity/category keys we drop the spelled-out key entirely and let a single-cell glyph + a semantic color say what
   it is. `TYPE=sdd` becomes a colored `◆ sdd`; `AGENT=worker-1` becomes a gold `@worker-1`; `BUG=123` becomes `#123`.
2. **Descriptive keys keep a quiet word-label.** Path/host/arbitrary keys (`PLAN`, `MACHINE`, and any user PR tag) keep
   a dim lowercase key word plus a styled value, so the display degrades gracefully for tags we don't recognize.
3. **Everything is drawn from the existing SASE palette and glyph vocabulary**, so the result reads as native to
   `sase vcs log` rather than bolted on. No new symbols are invented; no glyph collides with the presence glyphs
   (`● ↑ ↓ ·`) already used on every row.

### Visual target — `pretty`

```
  ── Today ────────────────────────────────────────────
   ↑ 14:22  a1b2c3d  sase       fix(sdd): link store   ◆ sdd · @worker-1 · plan sdd/tales/foo.md   · bryan
   ● 13:05  4c5d6e7  sase       docs: notes            ◆ config · #412   · bryan
```

- `◆ sdd` — diamond swatch + value, both in the per-type accent color.
- `@worker-1` — gold, matching the house "agent name is gold" convention.
- `plan sdd/tales/foo.md` — dim `plan` word; the path directory dims and the basename takes the accent.
- `#412` — soft-red bug/issue chip.
- The colored chips visually separate themselves from the trailing dim ` · bryan` author suffix.

### Visual target — `full` (aligned metadata block, the "hero" view)

```
   ▌ sase  fix(sdd): link store
     body line one
     body line two
     ◆ type     sdd
     @ agent    worker-1
       plan     sdd/tales/foo.md
       machine  athena
     # bug      412
     a1b2c3d · Bryan <b@x> · 14:22 · 38m ago · synced
```

`full` has room to breathe, so it renders an aligned two-column block: `(glyph-or-space) key   value`. The glyph and
value colors are the _same spec_ as `pretty`; only the layout differs.

## Semantic Style Spec

Known keys (canonical, `SASE_` stripped) map to a marker glyph + colors. All hex values already appear in the SASE
codebase palette.

| Key       | Marker | Marker/value color            | `pretty` form           | Notes                                     |
| --------- | ------ | ----------------------------- | ----------------------- | ----------------------------------------- |
| `TYPE`    | `◆`    | per-value (table below)       | `◆ <value>`             | Category swatch.                          |
| `AGENT`   | `@`    | gold `#FFD700`                | `@<value>`              | Matches `_AGENT_NAME_ANNOTATION_STYLE`.   |
| `BUG`     | `#`    | soft red `#FF8787`            | `#<value>`              | Reads as an issue id.                     |
| `PLAN`    | —      | dir `dim`, basename `#5FAFFF` | `plan <dir/><basename>` | Path basename accented, directory dimmed. |
| `MACHINE` | —      | key + value `dim` (`#8A8A8A`) | `machine <value>`       | Provenance; intentionally recedes.        |
| _unknown_ | —      | key `dim`; value default fg   | `<key> <value>`         | Any other `SASE_*` PR tag; stays calm.    |

`◆`, `@`, `#`, and `·` are all single-cell and already used (or ASCII), so they render correctly under the pinned Fira
Code used by the visual snapshot suite. `◆` is unused elsewhere in this renderer; `●`/`↑`/`↓` are deliberately avoided
because they are the presence glyphs on the same rows.

### `TYPE` value → color map

Derived from the real auto-commit `TYPE` vocabulary in the tree, using existing house hues:

| `TYPE` value | Color     | Family                             |
| ------------ | --------- | ---------------------------------- |
| `sdd`        | `#87D7FF` | sky blue                           |
| `init`       | `#5FD75F` | green (genesis)                    |
| `beads`      | `#5FD7AF` | teal (bead accent)                 |
| `bead_work`  | `#00D7AF` | bright teal                        |
| `memory`     | `#AF87FF` | purple                             |
| `skills`     | `#D787AF` | pink                               |
| `xprompt`    | `#FFAF5F` | amber                              |
| `config`     | `#D7AF5F` | gold (vcs `GOLD`)                  |
| _(unknown)_  | `#AF87D7` | neutral mauve (documented default) |

Unknown `TYPE` values fall back to the neutral mauve so an unrecognized type never implies a false category color.

### Tag ordering

For visual formats, present tags in a stable semantic priority — `TYPE`, `AGENT`, `MACHINE`, `PLAN`, `BUG`, then any
unknown keys in their original footer order — so `TYPE` is always the leftmost chip and columns line up across rows.
`commit_tag_view` already de-dupes and preserves footer order; the ordering step is a display-only reshuffle layered on
top of it. JSON output (`sase_tags`) keeps its current footer order and is _not_ reordered.

## Scope

**In scope (redesigned):** `pretty` and `full` colored output — the human-facing formats.

**Deliberately unchanged:**

- `oneline` — the plain, pipe/grep-friendly format. Its `[KEY=VALUE ...]` suffix stays machine-readable; adding color or
  glyphs there would hurt its scripting contract. (Keeps its existing golden test green.)
- `json` — the `sase_tags` object is data, not visuals; unchanged (including the `{}` opt-in shape and footer order).
- The CLI surface — no new flags; `-t/--tags` still gates all tag rendering, and default output is untouched.
- The wire schema and `sase-core` — this is presentation-only Textual/Rich glue, which the repo boundary rules
  explicitly allow to live in Python. `commit_tag_view` in `src/sase/vcs_log/tags.py` keeps parsing exactly as today.

## Technical Approach

1. **New tag-style module: `src/sase/vcs_log/_tag_style.py`.**
   - Holds the presentation spec: the `TYPE` color map, `AGENT`/`BUG`/`PLAN`/`MACHINE` colors, marker glyphs, the
     key-label dim style, and the semantic key-ordering priority.
   - Exposes a small builder that turns the parsed `(key, value)` tags from `commit_tag_view` into an ordered tuple of
     lightweight chip descriptors. A chip descriptor carries: canonical `key`, `marker`, `marker_style`, and the value
     rendered as one or more `(text, style)` segments (so `PLAN` can dim its directory and accent its basename).
   - This keeps all color/glyph decisions in one testable place, out of `render.py`, and shared by both layouts.
   - Keep module-internal symbols private (leading underscore) or genuinely consumed by `render.py` so pyvision does not
     flag test-only public symbols (a known gotcha from the prior pass).

2. **`pretty` — inline chips (`_commit_line` in `src/sase/vcs_log/render.py`).**
   - After the subject, append the ordered chips joined by a dim `·`, each chip built from the shared spec: glyph-marker
     tags render `marker + value`; descriptive tags render `dim key + value`.
   - Preserve the existing trailing ` · author` suffix and all alignment/soft-wrap behavior. No tags ⇒ no change.

3. **`full` — aligned block (refactor `_print_full_commit`).**
   - Extract the body/tag/footer construction into a helper that **returns the list of `Text` lines** it would print;
     the caller prints them. This makes full-mode styles span-testable (the research showed `_print_full_commit`
     currently prints directly and is not span-inspectable) and cleanly hosts the aligned block.
   - Render present tags as aligned rows `(glyph|space) key<pad> value`, key column padded to the widest present key,
     using the same chip spec as `pretty`. Continue to strip the raw `SASE_*` footer from the printed body (existing
     `commit_tag_view.body` behavior) so the footer is never duplicated.

4. **`oneline` / `json` — untouched** beyond leaving the existing `_oneline_tag_suffix` / `_commit_json` paths as-is.

5. **No changes** to `parser_vcs.py`, `vcs_handler.py`, `collect.py`, `commit_tag_view`, or any `sase-core` code.

## Tests

Follow the house patterns the research surfaced: plain-substring goldens with `color="never"`, plus **span-based
`Text.spans` assertions with exact hex** for the new colors.

- `tests/test_vcs_log_tags.py` (or a new `tests/test_vcs_log_tag_style.py`)
  - Chip builder: `TYPE` maps each known value to its color; unknown `TYPE` value → neutral mauve; `AGENT` → gold
    marker+value; `BUG` → `#` + soft red; `PLAN` splits directory (dim) vs basename (accent); `MACHINE` dim; unknown key
    → dim key label + default value.
  - Ordering: `TYPE, AGENT, MACHINE, PLAN, BUG` priority with unknown keys trailing in footer order.
  - `commit_tag_view` semantics unchanged (existing tag-parse tests stay green).
- `tests/test_vcs_log_render.py`
  - Update `test_pretty_tags_suffix_before_author` and `test_full_tags_line_and_footer_cleanup` to the new plain-text
    forms (glyphs/labels render even with color off; styles strip).
  - Add span tests (`color` on / building the `Text` directly) asserting exact styles: the `TYPE` swatch hex, the gold
    `AGENT`, the soft-red `BUG`, the `PLAN` basename accent — for both `pretty` (`_commit_line` returns `Text`) and
    `full` (via the refactored line-returning helper).
  - Assert the raw `SASE_*` footer never appears in `full` output (regression guard, already covered — keep it).
  - Confirm default (no `-t`) `pretty`/`full`/`oneline`/`json` goldens are unchanged, and `oneline --tags` /
    `json --tags` outputs are unchanged.

**Optional visual guard (recommended, scoped):** add a single PNG snapshot of a tagged `pretty` render by feeding a
`Console(record=True).export_svg()` capture through the existing `render_svg_to_png` + `assert_png` helpers from the ACE
visual suite (pinning width + Fira Code). This locks the actual pixels of the redesign. It is new glue, so it is
optional — the span tests are the primary color guarantee; include the PNG only if it stays small and deterministic.

## Verification

```bash
just install
pytest tests/test_vcs_log_render.py tests/test_vcs_log_tags.py tests/main/test_vcs_parser.py
just check
```

If the optional PNG snapshot is added, also run `just test-visual` and commit its golden. Manually sanity-check the real
look with `sase vcs log --tags` and `sase vcs log --tags --format full` on a repo whose recent commits carry `SASE_*`
footers.

## Risks & Follow-ups

- **Glyph/font portability.** The chosen glyphs are single-cell and already used in-tree (`◆`) or ASCII (`@`, `#`, `·`);
  no double-width emoji are introduced, so column alignment stays stable in `full`.
- **Palette on 8-color terminals.** `make_console("always")` downsamples truecolor hex to ANSI; that is why the color
  tests assert on the returned `Text` spans (exact hex) rather than ANSI escapes. Default `auto` behavior on real
  truecolor terminals is unaffected.
- **Scope discipline.** Keeping `oneline`/`json` and the wire schema unchanged confines the churn to two renderer
  functions plus one new style module, and keeps default output byte-for-byte stable.
- **Future:** if another frontend needs styled tags, promote the chip spec (not the raw parsing) into a shared
  presentation layer; parsed tag _data_ would still belong in the Rust-backed wire model per the boundary rules.
