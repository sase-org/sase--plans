---
tier: tale
title: "Projects statistics: lines by language and by repo"
goal:
  sase stats project and the TUI Projects sub-tab break code lines down by language and
  show which repos hold them.
size: medium
decisions:
  card_block:
    ask: How should the summary card's CODE section show the language breakdown?
    choices:
      rows:
        Up to 4 bar rows under CODE, each naming the repos that hold it; card grows by
        at most 5 lines
      strip:
        One stacked language bar plus chips under CODE; repos appear only under g →
        language
    default: rows
    why:
      Rows answer which language and which repo in the default view, matching the
      by-repo block
    answer: rows
  repo_language_column:
    ask:
      Should the repo grouping (-g repo / TUI By Repo) also gain a LANGUAGE column with
      each repo's top language?
    choices:
      add:
        Wide layouts (≥100 cols) add '● Python 99%' after TESTS; it is the last detail
        column dropped
      skip:
        Repo table keeps its columns; language-per-repo lives on the card and under g →
        language
    default: add
    why: Gives the repo view the inverse repo → language lookup for one extra column
    answer: add
proposed_by: bbugyi200.athena.147
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-25 13:38:00
status: wip
---

# Projects statistics: lines by language and by repo

## Goal

`sase stats project` (and the identical **SASE Admin Center → Statistics → 03 Projects**
sub-tab) tells you how big a project's code is ("3.1M lines · 46% tests") but not
**what** that code is or **where** it lives. Add a language breakdown of the code-line
total that also shows which repo each language's lines belong to, in three places that
share one data path:

1. **The summary card** (every grouping, CLI + TUI): a compact `by lang` block under
   `CODE`.
2. **A new grouping `language`** (`-g language` / TUI `g` → **By Language**): the full
   Language → Repo tree with exact lines, files, and shares.
3. **The repo grouping** (`-g repo`): code lines that reconcile with the new view, plus
   each repo's top language.

It must be intuitive (reads like the existing `by repo` block and `BY MODEL` tree),
reliable (every number reconciles to the Rust-computed total, and every gap is labeled
rather than silently dropped), and beautiful (aligned columns, a validated language
palette, graceful narrow and ASCII layouts).

## What exists today

- The Rust wire already carries everything needed, but nothing renders it:
  `grep -rn languages src/sase/stats` hits only `src/sase/stats/project/models.py`.
  - `project.code.languages` and `portfolio.code.languages` hold `{name, files, lines}`
    rows that sase-core aggregates over the **source + tests** role lines, deduplicated
    by repo identity. On real data, `sum(languages.lines) == code.code_lines` exactly
    (sase: 3,126,199).
  - Every measured code repo's `snapshot.languages` holds the same rows for that repo.
    `snapshot.role_counts` holds per-role `{files, lines}`.
- Language names are sase-core's canonical source-language ids (lowercase): `python`,
  `rust`, `css`, `lua`, `json`, `bash`, `sh`, `zsh`, `jinja`, `javascript`, `html`,
  `diff`, … . Some languages count 0 SLOC (`diff`: 3 files, 0 lines).
- **Pre-existing inconsistency the new view would expose:** the repo table's
  `CODE LINES` column shows `snapshot.tracked_lines`, which counts every role, including
  docs, config, and other. The card's `CODE` total and the `Code lines` legend use
  source + tests only. For real sase data the repo rows add up to 3.28M while the card
  says 3.1M, and `TESTS` is computed against the wrong denominator. This plan fixes it
  (see §3), because the language view's per-repo lines must equal the repo table's.
- The TUI renders the same Rich renderables through
  `StatisticsProjectsRenderingMixin._projects_renderable` →
  `render_project_stats(view, grouping, ctx)`. The `g` cycle, footer chip, and `?` help
  all read `sase.stats.registry`. A new grouping registered there therefore appears in
  the TUI with no Textual code changes. Code arrives in a second, merged load
  (`statistics_pane_project_merge.py` already copies `code` and repo `snapshot`,
  including `languages`). Until it lands, `view.code_pending` is true and the card says
  `measuring code…`.

## Design

### 1. Card: `by lang` block (decision `card_block`)

> [!decision] card_block = rows Insert a `by lang` row into the card body after `CODE`
> and its lifetime line, before `RUNS`, for both one-project and portfolio cards. It
> uses the **same grid geometry** as the OUTPUT `by repo` block, so the count, bar, and
> share columns line up vertically across the two blocks. Factor the shared column setup
> and width tiers out of `render_repo_breakdown` (name cap 28/22/16/12 and bar
> 24/14/10/0 at widths ≥100/≥80/≥60/<60) into one helper that both blocks call.

Target look (real sase numbers, 120 columns):

```text
│ CODE     3.1M lines · 46% tests · 89k doc lines · 7 of 7 code repos                                         │
│                     Feb 14, 2026 · 8 months · commits on 248 of 252 days · 18,680 commits all time          │
│ by lang  Python                            2.5M ████████████████████████  82%  sase, sase-telegram, sase-listen +3 more │
│          Rust                              559k █████▎                    18%  sase-core                     │
│          CSS                                10k ▏                         <1%  sase, sase-telegram           │
│          +7 more                           7.9k ▏                         <1%  Lua, JSON, Bash, Shell, Jinja, … │
│ RUNS     15,715 runs on athena · ✓ 1,906 × 56 · …                                                           │
```

Rules:

- Rows are languages with `lines > 0`, sorted by lines descending, then files
  descending, then label. The block shows at most **4 rows at width ≥ 60 and 3 below**.
  When there are more languages, the last row becomes `+N more` (dim). Its count and
  share are the folded total, and its note column lists the hidden language labels
  (ellipsis on overflow).
- Count = `format_compact` lines (`2.5M`, `559k`), right-aligned. Bar = `fine_bar` in
  the language's color, scaled to the largest visible row. Share = share of the
  subject's `code.code_lines` (see share labels below).
- Name = the language's display label in normal ink. Color goes on the bar, not the
  text, following the dataviz rule that text wears text tokens.
- The note column (dim) names where the lines live, ordered by that language's lines
  descending, three names maximum, then `+N more`:
  - one project with ≥2 measured code repos: repo names
    (`sase, sase-telegram, sase-listen +3 more`);
  - one project with a single code repo: no note, because it would repeat the same repo
    on every row;
  - portfolio card: **project** labels (`sase, bob-cli`), since the portfolio's unit is
    the project. Repo detail lives under `g` → language.
- When the subject has code lines with no detected language, a `?  no language detected`
  row (existing `UNKNOWN_STYLE`) is always shown last. It does not count against the
  cap, so a coverage gap is never folded away.
- Hide the whole row when code is pending (the CODE line already says
  `measuring code…`), when code is unavailable, when `languages` is missing, or when
  there are 0 code lines.

> [!decision] card_block = strip Instead of the grid, add one `by lang` line: a
> 24/16/10-cell `stacked_bar` (language colors, glyph-cycled for no-color/ASCII), then
> chips like `Python 82%  Rust 18%  CSS <1%  +7`, with labels in ink and the `●` swatch
> in the language color. No repo notes appear on the card. Everything else in this plan
> is unchanged.

### 2. New grouping: `language` (CLI `-g language`, TUI **By Language**)

Register `language` in `PROJECT_REPORT_GROUPINGS` as
`("project", "repo", "language", "model", "patch")` with the label `By Language`. It
sits next to `repo` because both describe code, so the TUI `g` cycle becomes project →
repo → language → model → patch, and one-project scopes still skip `project`. Add it to
the `ProjectsGroupBy` and `ProjectStatsGrouping` Literals and to the allowed set in
`src/sase/stats/project/request.py:54`. Argparse choices stay sorted automatically
(`{language,model,patch,project,repo}`), as the CLI rules require.

Target look (one project, 120 columns):

```text
BY LANGUAGE · source + test lines at each code repo's default-branch tip
LANGUAGE                                                        LINES    FILES   SHARE
──────────────────────────────────────────────────────────────────────────────────────────────────────────────
● Python                                                    2,549,105   12,958     82%  ████████████████████████
      sase                                                  2,478,941   12,699     79%  ███████████████████████▍
      sase-telegram                                            32,345       92      1%  ▎
      sase-listen                                              21,060       98      1%  ▏
      +3 more repos                                            16,759       69      1%  ▏
● Rust  sase-core                                             559,051    1,168     18%  █████▎
● CSS                                                          10,167        7     <1%  ▏
      sase                                                     10,131        6     <1%  ▏
      sase-telegram                                                36        1     <1%  ▏
● Lua  sase-nvim                                                4,389       29     <1%  ▏
  …
?  no language detected                                         1,234       11     <1%  ▏          (only when > 0)

total                                                       3,126,199   14,229    100%
No countable lines: Diff (3 files)
not measured: missing (not cloned)
```

Rules:

- **Title:** `BY LANGUAGE · source + test lines at each code repo's default-branch tip`.
  Code is a lifetime snapshot, so unlike other tables the title never shows the range.
  The title wraps at narrow widths like the other tables' titles.
- **Parent rows:** use the subject's Rust aggregates (`project.code.languages` for one
  project, `portfolio.code.languages` for several), so totals stay correct under
  sase-core's repo-identity dedup. Use the same ordering as the card. A `●` swatch in
  the language color is followed by the **bold** label in ink. Zero-line languages are
  not rows. They go in a dim `No countable lines: Diff (3 files)` footnote so the FILES
  total still reconciles.
- **Child rows (where the lines live)** appear only when the scope has more than one
  measured code repo:
  - If a language lives in exactly one repo, collapse it onto the parent row:
    `● Rust  sase-core`, with the repo in dim ink, mirroring `🦋 muse  auto` in BY
    MODEL.
  - Otherwise list up to **3** child rows indented under the parent, sorted by lines
    descending, then a dim `+N more repos` row carrying the folded lines and files, so
    children sum to the parent.
  - Child labels: one project → the repo name. Several projects → the project accent
    square `■` plus `project` (single-code-repo project, with ` · repo` appended when
    the names differ, as in `_collapsed_tree_row`) or `project · repo` (multi-repo
    project).
- **`?  no language detected` sentinel:** total = `code.code_lines − Σ language lines`,
  and files likewise. Per-repo children = `repo_code_lines − Σ repo language lines` (> 0
  only). Shown last, only when > 0, styled `UNKNOWN_STYLE`.
- **Columns:** LANGUAGE (ratio 1, ellipsis), LINES (full `format_count`), FILES, SHARE,
  then an unheaded bar column. FILES appears at width ≥ 80. The bar column (24 cells at
  ≥120, 16 at ≥100) appears only at width ≥ 100. Parent bars use the language color and
  child bars the same color with `dim`. All bars share one scale, set by the largest
  parent.
- **Total row** after a section break: `total`, `code.code_lines`, `code.code_files`,
  `100%`.
- **Dim notes** below the table, each only when it applies:
  - zero-line footnote;
  - `not measured: <repo> (<reason>)` for code repos with no snapshot, using the by-repo
    wording and issue messages;
  - overlap:
    `N lines are in repos counted under more than one project; totals count each once`,
    when Σ child lines exceeds the Rust totals. This mirrors the existing
    `shared_commits` note.
- **States:**
  - `view.code_pending` → the table area shows dim italic `measuring code…` (the TUI
    first paint).
  - Code is `None` and not pending → dim `Code could not be measured; see Issues below.`
  - 0 code lines or no code repos → dim `No code lines measured.`
  - `code.languages is None` → dim `No language data in this report.`

### 3. Repo grouping: consistent code lines (+ decision `repo_language_column`)

- **Fix:** the repo table's `CODE LINES` shows `repo_code_lines(snapshot)`, which is
  source + tests (§4). `TESTS` becomes `test_lines / repo_code_lines`. Multi-repo
  project rows already use `code.code_lines`, so after this fix they equal the sum of
  their repo rows, and every repo row equals that repo's language rows plus `?`. Apply
  the same fix to `_repo_md_row` in Markdown.

> [!decision] repo_language_column = add Add a `LANGUAGE` detail column right after
> `TESTS`: a `●` swatch in the top language's color, then `<Label> <share>` in ink,
> where share = top language lines / repo code lines (for example `● Python >99%`,
> `● Rust 99%`). It shows `—` with no data. Multi-repo project rows in the portfolio
> tree use the project's `code.languages`. Like the other detail columns it appears only
> when `ctx.show_detail_columns` (≥100). Under width pressure it is dropped **after**
> DOC LINES, FIRST COMMIT, and LAST COMMIT, so it outlives them. In ASCII mode, omit the
> swatch. Markdown's repo table gains a matching `Language` column.

### 4. Shared data path: no sase-core change

Add a Textual- and Rich-free module `src/sase/stats/project/languages.py` that projects
the existing wire into one breakdown, used by the card, the language table, the repo
table, and Markdown:

- `repo_code_lines(snapshot) -> int | None` is a thin adapter mirroring sase-core
  `project_stats::code::code_totals`: source + tests lines from `role_counts`, falling
  back to `tracked_lines` when `role_counts` is absent. Name sase-core's `code_totals`
  in its docstring so the two stay in lockstep.
- Frozen dataclasses (names indicative):
  - `LanguageSlice`: project key/label, repo id/name, lines, files.
  - `LanguageRow`: `language: str | None` (where `None` is the `?` sentinel), lines,
    files, share, slices.
  - `LanguageBreakdown`: total lines and files, rows, zero-line languages, unmeasured
    repos, `overlap_lines`, `many_projects`, measured and configured repo counts.
- `language_breakdown(view) -> LanguageBreakdown | None`. It returns None when code is
  pending or unavailable, and only considers `tier == "code"` repos.

Why no Rust change: the wire already carries `languages` at the project, portfolio, and
repo-snapshot levels plus `role_counts`. Everything new is a projection of those fields:
ordering, folding, labels, colors, and one subtraction for `?`. Totals always come from
Rust's identity-deduplicated aggregates, and per-repo detail comes from the snapshots,
with any overlap labeled. Promoting per-repo `code_lines` into the wire would add a
sase-core release plus a pin move for a two-field sum. That is out of scope, and the
docstring points at the Rust definition. Also do **not** merge `sh`, `bash`, and `zsh`
rows: they are distinct core ids, so they get distinct labels in one shared color
family.

### 5. Language identity: labels and palette

Put this in `src/sase/stats/render/languages.py` together with the Rich renderers for
§1–§2 and the shared share-label helper:

- **Labels** for every id sase-core can emit (`source_language.rs` suffix and basename
  maps plus `project_stats/classify.rs` shebangs): python→Python, rust→Rust,
  typescript→TypeScript, tsx→TSX, javascript→JavaScript, jsx→JSX, css→CSS, html→HTML,
  lua→Lua, sh→Shell, bash→Bash, zsh→Zsh, fish→Fish, json→JSON, yaml→YAML, toml→TOML,
  xml→XML, sql→SQL, markdown→Markdown, rst→reStructuredText, jinja→Jinja, diff→Diff,
  c→C, cpp→C++, go→Go, java→Java, ruby→Ruby, perl→Perl, php→PHP, awk→Awk, r→R,
  powershell→PowerShell, docker→Dockerfile, make→Makefile. Unknown ids fall back to the
  id with its first letter upper-cased.
- **Colors** follow the language identity, never its rank, so Python is the same blue on
  the card, in the language table, in the repo table, CLI and TUI alike. The 8 slots
  come from the dataviz reference dark categorical palette. They were validated against
  the TUI's Statistics surface `#1c1b1a` with `validate_palette.js --mode dark`: all
  checks pass (lightness band, chroma ≥ 0.10, worst adjacent CVD ΔE 8.4, normal-vision
  ΔE 19.3, all ≥ 3:1 contrast). Hues echo GitHub-linguist families:

  | ids                 | color                                                                                  |
  | ------------------- | -------------------------------------------------------------------------------------- |
  | python              | `#3987e5` blue                                                                         |
  | rust                | `#d95926` orange                                                                       |
  | typescript, tsx     | `#199e70` aqua                                                                         |
  | javascript, jsx     | `#c98500` yellow                                                                       |
  | lua                 | `#d55181` magenta                                                                      |
  | sh, bash, zsh, fish | `#008300` green                                                                        |
  | css                 | `#9085e9` violet (kept far from HTML's red; red/magenta is the palette's weakest pair) |
  | html                | `#e66767` red                                                                          |
  | every other id      | `#8a8984` neutral (4.9:1 on the surface); no generated hues                            |
  | `?` sentinel        | existing `UNKNOWN_STYLE`                                                               |

  Add a comment saying that any new slot must be re-validated with the dataviz
  validator.

- **Share labels** (one helper for the card, language table, and repo column): an exact
  0 → `0%`, a share that rounds to 0 but is > 0 → `<1%`, a share that rounds to 100 but
  is < 1 → `>99%`, otherwise a whole percent. Never print `100%` for a language that is
  not alone.
- **ASCII / no color:** in ASCII mode the swatch is omitted and `fine_bar` uses its
  ASCII fallback. In both cases identity rests on the always-present text label.

### 6. Markdown, JSON, legend, and CLI help

- **Markdown** (`render_project_stats_markdown`):
  - card: a `**BY LANGUAGE** Python 2,549,105 (82%) · Rust 559,051 (18%) · …` line (all
    languages with lines > 0, then `?` if any) after `**CODE**`;
  - `-g language`: a `## By language` table (`Language | Lines | Files | Share`) with
    the same tree as the Rich table, using the `_model_markdown` conventions: indented
    child names, collapsed `Rust · sase-core`, `+N more repos`, a total row, and the
    same notes as plain lines.
- **JSON:** the wire is unchanged (languages are already in it). The envelope's
  `query.group_by` reports `language` when selected.
- **Legend** (`PROJECT_REPORT_LEGENDS`): add
  `MetricLegend("By language", "code lines per language, detected from file extension, then shebang, at each code repo's default-branch tip; repo rows show where they live; ? = code lines with no detected language")`
  after `Doc lines`. It flows into `-h` definitions and the TUI `?` help.
  `render_legend`'s `More:` hint order becomes
  `repo · language · patch · model · project`, minus the active grouping.
- **CLI epilog** (`_project_epilog`): add the examples
  `sase stats project -p sase -g language` and
  `sase stats project -p sase -j | jq '.data.projects[0].code.languages | sort_by(-.lines)'`.

## Files to touch

- `src/sase/stats/project/languages.py` (new): the breakdown projection and
  `repo_code_lines`.
- `src/sase/stats/render/languages.py` (new): labels, palette, share labels, the card
  block renderer, and the `render_language_table` grouping renderer.
- `src/sase/stats/render/repos.py`: extract the shared breakdown-grid helper, use the
  code-lines fix, and add the LANGUAGE column.
- `src/sase/stats/render/card.py`: insert the `by lang` row.
- `src/sase/stats/render/compose.py`: add `"language": render_language_table`.
- `src/sase/stats/render/markdown.py`: add the card line, `_language_markdown`, and the
  repo code-lines and Language column changes.
- `src/sase/stats/render/legend.py`: update the `More:` hint.
- `src/sase/stats/render/__init__.py`: export the new renderer if the package API lists
  it.
- `src/sase/stats/registry.py`: add the grouping, label, Literal, and legend.
- `src/sase/stats/project/query.py`: add `language` to the Literal.
- `src/sase/stats/project/request.py`: add `language` to the allowed groupings.
- `src/sase/main/parser_stats.py`: add the epilog examples.
- Tests and docs, listed below. No `src/sase/ace/**` changes should be needed. If any
  turn out to be, read the `ace.md` memory first.

Keep new modules focused; if `repos.py` or `markdown.py` would grow past a comfortable
size, put the language pieces in `languages.py` rather than growing them.

## Tests

- **`tests/stats/_project_render_fixtures.py`:** make the fixture data realistic and
  internally consistent.
  - Lowercase ids, multiple languages per repo, `role_counts` whose source + tests
    equals each repo's code lines, and project `code.languages` equal to the
    per-language sum over measured repos.
  - Cover: a zero-line `diff`; one repo with unrecognized lines (exercises `?`); the
    unmeasured `missing` repo; a single-repo language (collapsed row); a language spread
    over ≥4 repos (exercises `+N more repos`); and in the portfolio fixture a second
    project with rust/javascript plus a repo shared by two projects (exercises the
    overlap note).
  - Replace the current `{"name": "Python", ...}` placeholder in `_code()`.
- **New `tests/stats/test_project_languages.py`** covering:
  - the `repo_code_lines` role_counts path and its fallback;
  - parent totals coming from Rust aggregates;
  - ordering and tie-breaks;
  - slices per repo;
  - the `?` row (project level and per repo);
  - zero-line languages;
  - unmeasured repos;
  - `overlap_lines`;
  - portfolio owner labels;
  - `None` for pending or unavailable code.
- **Render tests** (new `tests/stats/test_render_languages.py` and/or
  `test_render_project.py`):
  - every id sase-core can emit has a label;
  - unknown id → neutral color plus capitalized label;
  - share labels `0%`/`<1%`/`>99%`;
  - card cap and `+N more` listing the hidden labels;
  - `?` pinned outside the cap;
  - no repo note for a single-repo project;
  - the card's count/bar/share columns align between `by repo` and `by lang` (same
    column offsets in exported text).
- **Goldens:**
  - New: `one_project_language_{120,100,80,60,50}.txt`, `all_projects_language_120.txt`,
    `code_pending_language_120.txt`, `one_project_language.md`,
    `all_projects_language.md`.
  - Regenerate every existing render golden with `SASE_UPDATE_PROJECT_RENDER_GOLDENS=1`
    (the card block and repo-table changes touch most of them). Read every diff: card
    rows aligned, repo `CODE LINES` now source + tests, LANGUAGE column only at ≥100,
    nothing truncated that wasn't before.
- **Registry, CLI, and request:**
  - `test_registry.py`: the grouping order and the `By Language` label.
  - `test_cli.py`: `-g language` parses and renders, and the JSON envelope's `group_by`
    is `language`.
  - `test_query.py` and the request tests accept `language`.
  - Hand-update both `tests/stats/golden/cli/project_help*.txt` goldens for the new
    choice, legend, and examples.
- **TUI** (`tests/ace/tui/test_statistics_pane_projects.py` and the related tests):
  - update the cycle expectations to include `language`: one project →
    `repo, language, model, patch`; several → `repo, language, model, patch, project`;
  - add a test that the pane renders `BY LANGUAGE` with fixture repo names under `g` →
    language;
  - add a test that while code is pending, the language grouping shows `measuring code…`
    and switching to it does not trigger a reload (`_selection_changed(reload=False)`);
  - add a test that the merged code report (`merge_project_code_stats`) produces the
    language rows.

## TUI verification ("make sure it works and looks good")

1. **PNG goldens**
   (`tests/ace/tui/visual/test_ace_png_snapshots_config_center_statistics.py`): add
   `config_center_statistics_projects_language_120x40` (one project, `g` → language),
   modeled on the existing `..._projects_repo_120x40` test. Then run
   `just fix-tui-screenshots -- -k statistics_projects` through `/sase_monitor` (it can
   be long). Inspect the report and **every** updated golden, at minimum `projects`,
   `projects_single`, `projects_model`, `projects_repo`, `projects_patch`,
   `projects_narrow_90x30`, and `projects_measuring_code`. Check alignment, that colors
   read on the dark surface, that the 90x30 narrow layout wraps cleanly, and that the
   measuring-code snapshot shows no `by lang` block.
2. **Live capture of the real TUI with real data**, following the `tui_screenshot`
   memory (run this checkout's code, not a stale global install): open Admin Center →
   Statistics → 03 Projects with the project filter on `sase` and capture with
   `sase screenshot`. Capture the default model grouping (card `by lang` block), `g` →
   By Language, and `g` → By Repo, at 120 columns and at a narrow width (about 90). Look
   at each PNG. The `by lang` columns line up with `by repo`, the Python/Rust bars are
   clearly colored, and nothing important falls below the first screen except the table
   body, which scrolls with Ctrl+D/U.
3. Confirm the CLI matches: `sase stats project -p sase -r 90d`, `-g language`,
   `-g repo`, `-f markdown -g language`, and a portfolio run
   (`sase stats project -g language`).

## Docs

- `just fix` regenerates `docs/reference/cli/stats.md` from argparse.
- `docs/reference/tui/machines-and-tools.md` (Projects sub-tab):
  - add language to the `g` list;
  - add one or two sentences on the card's `by lang` rows and the Language → Repo tree;
  - note that repo `CODE LINES` count source + test lines.
- `docs/reference/tui/admin-center.md`: add language to the `g` list.
- `docs/contribute/telemetry.md`: add language to the Projects row's cycle list and to
  the one-project cycle sentence.

## Verification

- Run `just fix` (formatting, keep-sorted, generated CLI docs), then
  `sase tool run check` until green. Treat UNKNOWN triage as yours.
- Complete the TUI verification steps above, including inspecting the visual report.

## Out of scope

- Adding per-repo `code_lines` or per-language test splits to the sase-core wire.
- Changing language detection, or merging shell dialects in core.
- A LANGUAGE column on the multi-project `-g project` table.
