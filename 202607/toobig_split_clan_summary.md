---
tier: tale
title: Enrich the toobig_split clan summary with per-file names and line counts
goal: The toobig_split clan summary lists each oversized file's path and current line
  count, severity-ranked and beautifully rendered within the existing clan-summary
  frame.
create_time: 2026-07-23 11:24:47
status: done
---

- **PROMPT:** [202607/prompts/toobig_split_clan_summary.md](prompts/toobig_split_clan_summary.md)

# Enrich the `toobig_split` clan summary with per-file names and line counts

## Summary

The `toobig_split` chop launches a clan of wait-chained "split-file" agents, one per oversized Python file. The clan
summary shown in the `sase ace` Agents tab currently states only a _count_ of files and a static mission blurb — it
never says _which_ files are being split or _how big_ they are. This plan enriches that summary with a compact,
color-coded `TARGETS` list: every oversized file's repo-relative path plus its current line count, severity-ranked,
rendered beautifully within the existing 76-column clan-summary frame.

The change is **entirely self-contained in the `toobig_split` chop** and needs no changes to the sase repo. See "Design
decision" for why an enriched _static_ summary — not a dynamic summary script — is the right tool here.

## Target repository (read carefully)

All code changes live in the **`bugyi-chops`** package, an external repo installed editable into the sase venv. It is
**not** the sase repo.

- Open it first with the `/sase_repo` skill: `sase repo open gh:bbugyi200/bugyi-chops -r "<reason>"`. Use the printed
  checkout path as the only path for reads and writes.
- Files to change (repo-relative to that checkout):
  - `src/bugyi_chops/toobig_split.py` — the chop implementation (rendering + line counting).
  - `tests/test_toobig_split.py` — its unit/integration tests.
- Validate with **that repo's own** tooling, not the sase repo's: run `just check` (which runs `ruff format --check`,
  `ruff check`, `mypy`, `pytest`, and `build`) from inside the bugyi-chops checkout. The sase repo's `just check` is
  irrelevant to this change; do not run it for validation.
- Constraints that repo already enforces (honor them): `mypy --strict`, ruff (line-length 100, rules `B,E,F,I,SIM,UP`),
  and a pytest coverage gate of `fail_under = 90`.

## Background: how it works today

`src/bugyi_chops/toobig_split.py`:

- `_scan_files(...)` runs `toobig --files-only <tree> <limits>` for each tree in `vars.trees` (default
  `("src", "tests")`), collecting a deduped, order-preserving list of repo-relative posix paths (`files`).
  `--files-only` yields paths but no line counts.
- `_render_clan_summary(file_count, tree_count, limits) -> str` builds a 5-line Rich-markup string: a header
  (`◆ TOOBIG SPLIT · N FILES`), a two-line `MISSION` block, and a facts line
  (`{tree_count} scan roots · limits a / b / c lines · sequential queue`). Each line is a single-styled `Text`; the
  function returns `"\n".join(line.markup for line in lines)`.
- `build_result(...)` proposes one launch per file (scan order), each `clan=CLAN_TEMPLATE` (`"toobig-@"`), wait-chained
  (`wait_on=prior_id`), and all carrying the **same** `clan_summary=` string.

On the sase side, that `clan_summary` string is emitted verbatim into a
`%clan(<clan>, tribe=chop, summary=[[<summary>]])` directive (`chop_proposals._clan_declaration_directive`) and later
rendered as Rich markup in the clan panel (`src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan.py`,
`Text.from_markup(...)` with a plain-text fallback). Multi-line summaries already round-trip through this path today
(the current summary is 5 lines).

## Design decision: enriched static summary (chosen) vs. dynamic summary script (rejected)

The user flagged that we might need to switch from a static clan summary to a dynamic clan-summary **script**. We
evaluated both and chose the **enriched static summary**.

Why static wins here:

1. **The data already exists at chop time.** The chop scans the repo and holds the exact file list; line counts are one
   cheap read per file away (count `b"\n"`, the same `wc -l` semantics `toobig` uses). The summary should describe
   exactly the batch this clan was launched to fix — a static snapshot is the _correct_ semantic, not a limitation.
2. **Live progress is already covered.** As agents work the sequential wait-chain, the clan panel's per-member roster
   shows each `split_file.*` agent's status (running/completed/…). A dynamic re-scan of the summary would largely
   duplicate that, while the static summary supplies the stable "what/how big" context the roster lacks.
3. **A chop cannot drive a dynamic summary script today.** The chop→launch proposal protocol only carries a static
   `clan_summary` string; `_clan_declaration_directive` has no `summary_script=` branch, and the proposal schema
   (`sase.core.axe_chop_facade` / `validate_chop_proposal`) does not model one. Supporting dynamic summaries from a chop
   would be a cross-cutting sase feature (SDK `propose`, prepared-proposal model, directive emission, proposal-result
   schema — possibly the Rust core chop-result schema) — far out of proportion to a decorative panel.
4. **Reliability.** A dynamic script runs at launch time in each member's _separate_ fresh workspace, knowing only the
   clan name/generation/tribe — not the original file list — so it would have to re-scan (requiring `toobig` on the
   launch PATH, a subprocess per member over potentially hours, a 20s timeout), and any failure silently downgrades the
   summary to _nothing_. For a decorative element, a summary that is always correct beats one that is occasionally live
   but sometimes blank.

Net: the enriched static summary delivers precisely what was asked (file names + current line counts), is self-contained
in one file, and adds no launch-time fragility. If the user specifically wants live-refreshing counts, that is a larger,
separate feature (extend the chop proposal protocol to carry a `clan_summary_script`) and should be its own epic; it is
called out in "Out of scope / future".

## Detailed design

### New/changed rendering in `src/bugyi_chops/toobig_split.py`

Introduce a small typed entry and re-shape `_render_clan_summary` to accept per-file data.

```python
@dataclass(frozen=True)
class FileEntry:
    path: str            # repo-relative posix path (as produced by _scan_files)
    line_count: int | None   # newline count; None if the file could not be read
```

New helpers:

- `_line_count(path: Path) -> int | None` — `return path.read_bytes().count(b"\n")`, returning `None` on `OSError` (a
  scanned file may be missing — see the existing
  `test_absolute_scanner_paths_are_normalized_and_missing_files_still_dedupe`).
- `_classify(line_count: int | None, limits: tuple[int, int, int]) -> str` — returns a severity key: `"violation"` if
  `line_count > limits[0]`, `"warning"` if `> limits[1]`, `"fyi"` if `> limits[2]`, `"neutral"` otherwise, and
  `"unknown"` when `line_count is None`.
- `_elide_path(path: str, max_cells: int) -> str` — if the path fits, return it unchanged; otherwise drop leading
  characters and prefix `…/` so the result's `cell_len` is `<= max_cells`. Elide from the **left** (keep the filename
  and its nearest parents, which are the most identifying).

Re-signature the renderer:

```python
def _render_clan_summary(
    entries: Sequence[FileEntry],
    tree_count: int,
    limits: tuple[int, int, int],
    *,
    max_rows: int = CLAN_SUMMARY_MAX_ROWS,
) -> str: ...
```

`file_count` derives from `len(entries)` (drop the old first positional arg). `build_result` becomes:

```python
files = _scan_files(...)                       # unchanged
if not files:                                  # unchanged no-op branch
    ...
entries = [FileEntry(path=p, line_count=_line_count(target.repo_root / p)) for p in files]
clan_summary = _render_clan_summary(entries, len(trees), limits)
# proposal loop unchanged — still iterates `files` in scan order
```

Line counting reads each oversized file directly (not via `toobig`), so **no new `toobig` invocations** are added — the
existing `calls == ["--files-only src ...", "--files-only tests ..."]` assertions and the `toobig`-on-PATH test remain
valid.

### Rendered layout (target)

Rows are sorted by `line_count` descending (ties by path ascending; `None` counts sort last), then capped at `max_rows`.
Line counts are right-justified with thousands separators to a common column width so paths align. Each rendered line
remains a **single** styled `Text` (preserving the "one span per line" invariant), colored by that file's severity so
color encodes urgency and a glyph gives a colorblind-safe redundant cue.

```
◆ TOOBIG SPLIT · 3 FILES
MISSION
Decompose oversized Python modules into focused, reviewable units
without changing behavior.

TARGETS
▲ 1,214  sase/ace/tui/app.py
◆   902  sase/axe/run_agent_runner.py
•   731  …/tests/deep/path/test_foo.py

2 scan roots · limits 1,000 / 850 / 700 lines · sequential queue
```

When `len(entries) > max_rows`, append a final dim row `…and {N - max_rows} more` (the header count still reflects the
true total). A blank line separates the `MISSION` block from `TARGETS`, and `TARGETS` from the facts line.

Row format: `f"{glyph} {count_str:>{width}}  {path}"`, where `count_str` is `f"{c:,}"` (or `"?"` when
`line_count is None`), `width` is the max `count_str` length across displayed rows, and `path` is
`_elide_path(entry.path, max_cells)` with `max_cells = CLAN_SUMMARY_WIDTH - (2 + width + 2)`.

### New module constants

- `CLAN_SUMMARY_MAX_ROWS = 10`.
- Severity → style:
  - `CLAN_SUMMARY_VIOLATION_STYLE = "bold #FF5F87"`
  - `CLAN_SUMMARY_WARNING_STYLE = "bold #FFAF5F"`
  - `CLAN_SUMMARY_FYI_STYLE = "#87D7FF"`
  - `CLAN_SUMMARY_NEUTRAL_STYLE = "dim #A8A8A8"` (used for `"neutral"` and `"unknown"`).
- Severity → glyph (all single-cell): `▲` violation, `◆` warning, `•` fyi, `·` neutral/unknown.
- Reuse `CLAN_SUMMARY_SECTION_STYLE` for the `TARGETS` header and `CLAN_SUMMARY_FACTS_STYLE` for the overflow row.
  `CLAN_SUMMARY_WIDTH` (76) is unchanged.

Glyphs/hex colors are tunable aesthetic choices; keep them as named constants so they are easy to adjust.

### Invariants to preserve

- Every rendered line's `cell_len` `<= CLAN_SUMMARY_WIDTH` (enforced by count column + `_elide_path`).
- Summary never contains the `]]` sequence (would break the `summary=[[...]]` directive). Paths/counts/ glyphs cannot
  produce `]]`; keep it that way (no user-controlled bracket runs).
- Total size stays far under the 32 KiB clan-summary cap (≤ `max_rows` rows ⇒ ~1 KB).
- The summary remains valid Rich markup that `Text.from_markup` can parse.

## Testing (in `tests/test_toobig_split.py`)

Update and extend the chop's own suite; keep coverage ≥ 90%.

- **Rework the `_render_clan_summary` unit tests** (`test_clan_summary_has_canonical_text_styles_and_width`,
  `test_clan_summary_handles_one_file_and_formats_custom_limits`) for the new `entries`-based signature. Assert:
  header/mission/facts lines unchanged in wording; the `TARGETS` block lists each entry's path and formatted count in
  severity order; per-row style matches `_classify`; each line has exactly one span;
  `max(line.cell_len) <= CLAN_SUMMARY_WIDTH`.
- **New unit tests** for: mixed severities → correct glyph+style per row; a long path is left-elided with `…/` and still
  within width; a `None` line count renders as `?` with neutral style and sorts last; more than `CLAN_SUMMARY_MAX_ROWS`
  entries ⇒ exactly `max_rows` rows plus a `…and K more` line and a header count equal to the true total; the count
  column is right-aligned to a common width.
- **`_line_count`** unit test: correct newline count for a normal file; `None` for a missing path.
- **Integration** (`test_scan_deduplicates_files_and_emits_stable_wait_chain`): the byte-for-byte
  `CANONICAL_SUMMARY_PLAIN` equality no longer holds (real files are 1 line each, and the summary now lists them).
  Replace the exact-match assertion with structural checks: all proposals share one `clan_summary`; the parsed summary
  contains each scanned path (`src/pkg/large.py`, `src/pkg/shared.py`, `tests/large.py`) and still contains the header
  and facts lines. Keep the existing `calls` assertion (still exactly the two `--files-only` invocations) and all
  proposal/agent-name/wait-chain assertions.
- **`test_sase_planning_emits_one_summary_and_promotes_a_surviving_tail`** uses the summary abstractly via an
  `authored_summary` variable and asserts the `%clan(..., summary=[[...]])` round-trip; it should keep passing
  unchanged. Explicitly verify it still passes (confirms the richer multi-line/multi-markup summary survives directive
  extraction).
- Leave the fail-closed / no-op / project-resolution / scanner-failure tests unchanged; confirm the missing-file test
  still passes (it now also exercises the `None` line-count render path).

## Validation steps (run inside the bugyi-chops checkout)

1. `sase repo open gh:bbugyi200/bugyi-chops -r "Enrich toobig_split clan summary with file names and line counts"`.
2. Make the code + test changes at the printed checkout path.
3. `just check` (ruff format check, ruff lint, mypy strict, pytest w/ coverage, build). Iterate until green.
4. Sanity-render the new summary from a Python REPL in that venv (construct a handful of `FileEntry` values across
   severities, print `Text.from_markup(_render_clan_summary(...))`) to eyeball the layout, alignment, elision, and
   colors before finishing.

## Out of scope / future

- **Dynamic (live-refreshing) clan summary from a chop.** If live counts are wanted, extend the chop proposal protocol
  to carry a `clan_summary_script` (SDK `propose`, `_PreparedChopProposal`, `_clan_declaration_directive` → emit
  `summary_script=`, proposal-result schema, and the accompanying script) — a separate epic spanning the sase repo (and
  likely sase-core), not this tale.
- **Reordering the wait-chain by size** (split biggest files first, unifying launch order with the severity-sorted
  display). A reasonable follow-up, but it changes launch behavior and proposal/agent ordering and is deliberately
  excluded here.
- **A micro magnitude bar** (e.g. an inline `▏▎▍` bar of line_count vs. limit) per row — a possible aesthetic
  enhancement, left out to keep the first version simple and robust.
- Any changes to the chezmoi axe config or the sase repo.
