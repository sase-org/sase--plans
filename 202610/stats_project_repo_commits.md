---
tier: tale
title: Per-repo commit counts in the Statistics Projects report
goal:
  sase stats project and the Admin Center Statistics → Projects tab show how many
  commits landed in each code repo of a project (sidecars excluded), honestly and
  beautifully, through one shared renderer.
size: medium
decisions:
  portfolio_repo_rows:
    ask:
      Should the default all-projects table also nest each multi-repo project's code
      repos under its row?
    choices:
      compact:
        One row per project; per-repo counts live in the one-project card and the -g
        repo tree
      nested:
        Dim repo sub-rows under each multi-repo project; the all-projects table roughly
        triples in height
    default: compact
    why:
      That table compares projects; the -g repo tree already answers per-repo at that
      scope
    answer: compact
proposed_by: bbugyi200.athena.13b
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-23 22:32:12
status: wip
---

# Per-repo commit counts in the Statistics → Projects report

## Goal

Make "how many commits landed in each repo of this project" a first-class,
always-visible part of the Projects report, in both `sase stats project` and the Admin
Center **Statistics → 03 Projects** sub-tab. Count only code repos; sidecar
(records-tier) repos are never counted. The user wrote "`bob stats project`", which is
the Pomodoro tool and has no project view. They meant `sase stats project`. Both
surfaces call one shared renderer (`sase.stats.render.render_project_stats`), so one
change covers the CLI rich output, the TUI tab, and the Markdown export.

## What exists today (verified)

- The Rust core already measures every repo: each `ProjectStatsRepo` in the wire carries
  `tier` (`code` | `records`), `kind` (`primary` | `linked` | `sidecar`), `status` (`ok`
  | `partial` | `unavailable`), and `output.commits` for the selected range. `output` is
  `null` when the repo's history could not be read, and `partial` means a shallow clone
  or a truncated history walk, so the count is a lower bound. The project total is
  computed over **unique SHAs** across code repos, while per-repo counts are not
  deduplicated. **No Rust, wire, binding, or JSON change is needed.** Everything below
  is presentation, which belongs in this repo under the Rust core boundary.
- `-g repo` renders a flat repo table plus a `RECORDS · sidecars, never totaled` section
  (`render_repo_table` in `src/sase/stats/render/tables.py`). However:
  1. **The TUI By Repo view is broken across several projects.** The REPO column is a
     Rich `ratio=1` column. Rich gives a ratio column only the width left after every
     fixed column and ignores `min_width`, so at body widths of about 100–110 (the TUI's
     120×40 body is about 104) repo names collapse to `s…`. Reproduced with the render
     fixtures at width 104. The committed golden
     `tests/ace/tui/visual/snapshots/png/config_center_statistics_projects_repo_120x40.png`
     shows only `…` where the names should be.
  2. The flat table interleaves projects (`bob-cli`, `sase`, `bob-cli`, ...), so reading
     "the repos of project X" means scanning the whole table.
  3. Zero-commit rows print `0%` agent share, which should be `—`.
  4. The Markdown repo section has no Project column when several projects are shown.
     Its Records table also takes the in-range `output.commits`, while Rich shows
     lifetime commits.
- The one-project card shows the total ("216 commits") and a provider split, but no repo
  split. The TUI seeds the Projects filter from the current project, so the one-project
  card is what the user sees most often.

## Design principles

1. **Honest numbers.** An unmeasured repo is `—`, never `0`. A partial history is a
   lower bound and prints `≥N`. A nonzero share never rounds to `0%` (`<1%`). A
   zero-commit agent share is `—`. If per-repo counts add up to more than the project
   total because some commits are in more than one repo, say so.
2. **Sidecars are never counted.** Every per-repo commit view filters on
   `repo.tier == "code"`. The existing RECORDS section stays separate and untotaled.
3. **One renderer, every surface.** The CLI, TUI, and Markdown read one breakdown
   function, so they cannot disagree.
4. **Stable layout.** The card is identical in every `g` grouping, so cycling `g` in the
   TUI never makes the card jump. The repo block depends only on the Output facet, so it
   paints with the first frame while Code is still measuring.
5. **Beautiful through restraint.** A sorted horizontal bar chart in a single hue (the
   project's accent, which also colors the card border), with direct labels and no
   legend. Rows are separated by position, not by extra colors.

## Target look

One-project card at 120 columns (real `sase` numbers from the last 7 days):

```text
╭─ ■ sase ─────────────────────────────────────────────────────────────────────────────────────────────────────────────╮
│ OUTPUT   216 commits · 97% by agents · 0 plans committed                                                             │
│ by repo  sase                    160 █████████████████▊       74%                                                    │
│          sase-core                44 ████▉                    20%                                                    │
│          sase-research-artifacts  10 █▏                        5%                                                    │
│          sase-github               1 ▏                        <1%                                                    │
│          sase-telegram             1 ▏                        <1%                                                    │
│          no commits in range: sase-listen, sase-nvim                                                                 │
│ CODE     3.1M lines · 46% tests · 89k doc lines · 7 of 7 code repos                                                  │
│                     Feb 14, 2026 · 8 months · commits on 246 of 250 days · 18,638 commits all time                   │
│ RUNS     641 runs on athena · ✓ 161 × 2 · 25% success · 319 h agent time · 48 Patches                                │
│          Commits by provider  ...                                                                                    │
╰──────────────────────────────────────────────────────────────────────────────── Last 7 days · Oct 16 – Oct 23, 2026 ─╯
```

Edge cases at 80 columns (partial history, unmeasured repo, quiet repo, shared commits):

```text
│ OUTPUT   412 commits · 99% by agents · 61 plans committed (14 epic · 47 tale) │
│ by repo  widgets      380 ████████████▉  92%                                 │
│          widgets-core ≥32 █▏              8% partial history                 │
│          missing        —                    not measured: not cloned        │
│          no commits in range: widgets-nvim                                   │
│          3 commits are in more than one repo; the total counts each once     │
```

At narrow widths, names truncate with `…`, bars shrink (and are dropped below 60
columns), and active repos beyond the row cap fold into one `+N more repos` row. The
example below is illustrative and uses a lowered cap:

```text
│ by repo  sase             160 ███████▍   74%             │
│          sase-core         44 ██         20%             │
│          sase-research-a…  10 ▌           5%             │
│          sase-github        1 ▏          <1%             │
│          +2 more repos      2            <1%             │
```

`-g repo` with several projects becomes a Project → Repo tree, mirroring the existing
Project → Patch tree:

```text
REPO · Last 7 days · ...
PROJECT → REPO                  KIND      COMMITS   AGENT   CODE LINES   TESTS   DOC LINES   FIRST COMMIT    LAST COMMIT
■ bob-cli                                     241     98%    1,068,841     18%      39,153   Jun 01, 2026   Oct 23, 2026
    bob-cli                     primary       178     97%      532,632     30%      36,113   Jun 01, 2026   Oct 23, 2026
    bob-plugins                 linked         63    100%      483,896      0%       1,813   Jun 20, 2026   Oct 23, 2026
    bob-mac-capture             linked          0       —       52,313      8%       1,227   Aug 13, 2026   Sep 30, 2026
■ sase                                        216     97%    3,262,610     46%      89,135   Feb 14, 2026   Oct 23, 2026
    sase                        primary       160     98%    ...
    ...
■ tailnet                       primary         4      0%        1,343      0%           0   Oct 17, 2026   Oct 17, 2026
```

## Implementation

### 1. Breakdown model (one source of truth)

Create `src/sase/stats/render/repos.py`. Add a pure, Rich-free function
`repo_commit_breakdown(project: ProjectStatsProject, *, max_rows: int) -> RepoCommitBreakdown | None`
that returns frozen dataclasses:

- Return `None`, so no block renders, when the project has fewer than **two**
  `tier == "code"` repos (a single repo just repeats OUTPUT), or when
  `project.output is None` or `project.output.commits is None`.
- **Active rows**: code repos with `status in {"ok", "partial"}`, a non-null `output`,
  and `output.commits > 0`. Each row has `name`, `kind`, `commits`, `partial: bool`
  (`status == "partial"`), and `share`. Sort by
  `(-commits, kind != "primary", name.casefold(), id)`.
- **Share denominator** = the sum of active-row commits, so the shares add up to about
  100% even when commits are shared between repos.
- **Overflow**: when more than `max_rows` rows are active, keep `max_rows - 1` rows and
  fold the rest into one `hidden_count` / `hidden_commits` row. That row always stands
  for at least two repos, so a lone repo is never hidden behind a `+1 more` row.
- **Unmeasured**: a code repo whose `output` is null, or whose `status` is anything
  other than `ok` or `partial` (future statuses fail safe). It carries `name` and a
  `reason` (`repo.issue.message`, else the status). Keep config order. Unmeasured repos
  are never folded.
- **Quiet**: names of measured code repos with `commits == 0`, in config order.
- **Shared commits** =
  `max(0, sum(active commits incl. hidden) - project.output.commits)`.

### 2. Bar primitive

Add `fine_bar(value, maximum, *, width, style, ascii_fallback=False) -> Text` to
`src/sase/stats/render/primitives.py`. Unicode mode uses eighth-block precision
(`▏▎▍▌▋▊▉█`). A nonzero value always shows at least `▏`. ASCII mode uses `#` cells, and
any nonzero sub-cell value shows `|`. Pad with spaces to exactly `width` cells so
columns align, and draw no track. Values ≤ 0 give blank cells; values ≥ maximum fill the
bar. Export it from `__all__` next to `share_bar` and `stacked_bar`.

### 3. Card block (`src/sase/stats/render/card.py`)

In `render_project_card`, only when exactly one project is shown, insert the block as
the row right after `OUTPUT`. Its label is `by repo`, dimmed on the label column's
accent. Build the block in `repos.py` (for example
`render_repo_breakdown(breakdown, ctx, accent)`) as a `Table.grid` with columns name,
count, bar, share, and note:

- **Width tiers** (`ctx.width`):

  | Width | Name cap | Bar | Row cap |
  | ----- | -------- | --- | ------- |
  | ≥ 100 | 28       | 24  | 6       |
  | ≥ 80  | 22       | 14  | 6       |
  | ≥ 60  | 16       | 10  | 5       |
  | < 60  | 12       | —   | 4       |

  Names use `no_wrap=True, overflow="ellipsis"`. No row may wrap at any tier.

- **Styles.** The bar uses the project accent (`project_border_style(view)`, the same
  color as the card border). The primary repo's name is bold. The share is dim, and
  `<1%` is used for shares in (0, 0.5%). A partial count prints `≥N`, with a dim
  `partial history` note at ≥ 80 columns. The overflow row is dim: `+N more repos`, plus
  the hidden commits and their share.
- **Unmeasured rows** are dim and come after the active rows:
  `name  —  ·  not measured: <reason>`.
- **Quiet line** is dim: `no commits in range: a, b, c`. List at most 6 names, then
  `+N more`.
- **Shared-commit line** is dim and appears only when shared > 0:
  `N commits are in more than one repo; the total counts each once`.
- The portfolio card (several projects) is unchanged.

### 4. Shared cell semantics (`src/sase/stats/render/_text.py`)

Add two helpers and use them everywhere a commit count or agent share is printed:

- `agent_share_cell(output)` returns `—` when `output` is None or `output.commits` is 0
  or None. Otherwise it keeps the current `agent_share` and then `share_of` fallback.
- `repo_commits_cell(repo)` returns `—` for unmeasured, `≥N` for partial, and `N`
  otherwise.

Use `agent_share_cell` in the project table row and its total row, the repo rows and
tree, the Markdown project and repo rows, and the card OUTPUT line. With 0 commits the
OUTPUT line drops the `· N% by agents` phrase instead of printing `0% by agents`.

### 5. `-g repo`: the Project → Repo tree, with columns that fit

Move `render_repo_table` and its private helpers from `tables.py` into `repos.py`. Keep
exporting it from `sase.stats.render`, and update the imports in `compose.py` and
`__init__.py`.

- **Several projects**: first column `PROJECT → REPO`.
  - Each project with one or more code repos gets a bold header row: the accent square,
    the label, and the `(disabled)` / `(unregistered)` suffix as in `_project_cell`. Its
    cells are the project's commits, agent share, code lines, test share, and doc lines,
    plus first and last commit from `project.lifetime`. KIND is blank.
  - Repo rows follow, indented four spaces: name with a dim status reason when not ok,
    kind, `repo_commits_cell`, `agent_share_cell`, snapshot lines, tests and doc lines,
    and first and last commit. Zero-commit and unmeasured repo rows are dim.
  - A project with exactly one code repo collapses into its header row and takes that
    repo's KIND and counts. Append the repo name dimmed when it differs from the project
    label (the provider/model collapse already works this way).
  - Order projects by `(-commits, label.casefold(), key)`, putting None last. Order
    repos with the breakdown's sort key. Projects with no code repos do not appear in
    this table.
- **One project**: keep the flat table without a PROJECT column, built with the same row
  helpers so `≥`, `—`, and dimming match.
- **Column fit (fixes the TUI bug).** The name column gets a floor of
  `min(longest name cell, 24)` cells. Choose the optional detail columns (DOC LINES →
  FIRST COMMIT → LAST COMMIT, dropped in that order) by estimating each fixed column's
  width as `max(len(header), max cell len) + padding` and dropping columns until the
  floor fits. Still show them only when `ctx.show_detail_columns`. Apply the same fit to
  the RECORDS table's REPO column. Leave RECORDS otherwise as it is: flat, sidecars,
  never totaled.
- Do **not** put the breakdown bars in the repo table. The table compares many metrics,
  and the bars live in the card.

> [!decision] portfolio_repo_rows = nested Also emit indented, dim repo sub-rows under
> each project that has two or more code repos in `render_project_table` and
> `_project_markdown`. Fill only the COMMITS and AGENT cells (`repo_commits_cell` /
> `agent_share_cell`); leave every other cell blank. Use the breakdown's sort and its
> unmeasured `—` rows. Skip quiet repos in this nested form; list them in a dim trailing
> sub-row `no commits: a, b`. Keep the `total` row project-level. Under `compact`
> (default), `render_project_table` is unchanged except for `agent_share_cell`.

### 6. Markdown (`src/sase/stats/render/markdown.py`)

- **Card.** After the OUTPUT line, when the breakdown is not None, add
  `**BY REPO** sase 160 (74%) · sase-core 44 (20%) · …`. Partial repos print `≥32 (8%)`.
  Then add `no commits: a, b` and `not measured: missing (not cloned)` when present, and
  the shared-commit sentence when shared > 0. Markdown is an export, so it has no row
  cap.
- **`## By repo` with several projects.** Use a `Project → Repo` header with project
  rows and indented repo rows, matching `_patch_markdown`, and the same ordering,
  collapse, and cell semantics as the Rich tree. With one project, keep the current
  columns and apply the new cell semantics.
- **Records.** Add a Project column when several projects are shown. Use the same commit
  source as Rich: lifetime commits, falling back to `snapshot.lifetime_commits`.

### 7. Definitions, help, and TUI help

- In `src/sase/stats/registry.py`, add `PROJECT_REPORT_LEGENDS` entry
  `MetricLegend("By repo", "each code repo's commits in range, sidecars excluded; ≥ marks a lower bound from shallow or truncated history; a commit in several repos counts once in the total")`
  immediately after `By model`. It reaches `sase stats project -h` and the TUI `?` help
  modal automatically. The compact legend stays unchanged.
- In `_project_epilog` in `src/sase/main/parser_stats.py`, add the example
  `sase stats project -p sase -j | jq '.data.projects[0].repos[] | select(.tier == "code") | {name, commits: .output.commits}'`.
  Per-repo counts are already in the JSON; this example makes them discoverable.

### 8. Docs

- `docs/reference/tui/machines-and-tools.md`, Statistics Tab paragraph: say that a
  one-project report breaks its commits down by code repo directly under OUTPUT, with
  sidecar repos never counted, `—` for unmeasured repos, and `≥` for lower bounds. Also
  say that `g` → repo shows a Project → Repo tree when several projects are in scope.
  Keep the edit to about two sentences, which is within the Reference page's budget.
- Run `just fix` to regenerate generated CLI docs. Do not edit `CHANGELOG.md`.

## Tests

- **`tests/stats/test_render_primitives.py`** (`fine_bar`): exact width, nonzero gives
  at least a sliver, zero is blank, maximum is full, and the ASCII glyphs.
- **New `tests/stats/test_render_repos.py`** (breakdown unit tests):
  - code tier only: a sidecar with many commits never appears;
  - fewer than two code repos gives None, and output None gives None;
  - the sort order, including the primary tie-break;
  - the cap, the never-fold-one rule, and hidden commits;
  - unmeasured rows (output None, and an unknown future status);
  - `≥` for partial;
  - quiet repos;
  - shares add up to about 100%, and `<1%`;
  - the shared-commit count when per-repo counts add up to more than the total;
  - empty range (all quiet, no bars).
- **Fixtures (`tests/stats/_project_render_fixtures.py`).**
  - `_repo()` must emit `output: None, lifetime: None` when `status == "unavailable"`,
    as the Rust wire does.
  - Add a quiet linked repo (0 commits) to `_sase_project`, keeping the repo commits
    adding up to the project total.
  - `empty_range_view` must zero the per-repo commits so they stay consistent with its
    0-commit OUTPUT.
  - Add `repo_edge_view()` covering partial history, shared commits, more than 6 active
    repos, and an unknown status.
- **Goldens (`tests/stats/test_render_project.py`).**
  - Regenerate with `SASE_UPDATE_PROJECT_RENDER_GOLDENS=1` and review every diff by
    hand. Expected diffs: the new block in the one-project goldens, `—` for the missing
    repo, `—` agent share at 0 commits, and the repo tree.
  - Add goldens: `one_project_repo_breakdown_{120,80,60,50}.txt`,
    `repo_edge_{120,80}.txt`, `all_projects_repo_{120,104,80}.txt`,
    `all_projects_repo.md`, and `repo_edge.md`.
  - Add a regression assertion that at width 104 every repo name in the several-project
    repo table is printed in full. This is the TUI bug.
- **CLI help goldens.** Update mode rewrites only the running interpreter's
  `tests/stats/golden/cli/project_help*.txt`. Hand-apply the identical epilog and
  definitions diff to the other file (`project_help.txt` for Python < 3.13).
- **TUI.**
  - Make the visual fixture `_populated_statistics_view` / `_populated_project_stats` in
    `tests/ace/tui/visual/_ace_config_center_statistics_views.py` honor `project_filter`
    by returning only the matching project.
  - Add `test_config_center_statistics_projects_single_png_snapshot` (golden
    `config_center_statistics_projects_single_120x40`) with the filter set to the
    multi-repo `widgets` project, so the one-project card with its `by repo` block is
    covered.
  - Extend `tests/ace/tui/test_statistics_pane_projects.py` with a text assertion that
    the rendered single-project report contains the repo rows.

## Verification

1. `.venv/bin/pytest tests/stats tests/main/test_parser_stats.py tests/ace/tui/test_statistics_pane_projects.py`
2. Live CLI checks:
   - `COLUMNS=120 sase stats project -p sase`, `COLUMNS=80 sase stats project -p sase`,
     and `COLUMNS=104 sase stats project -g repo`. Names must be readable, with no
     wrapped rows.
   - `sase stats project -p sase -f markdown`
   - `sase stats project -p bob-cli` (three code repos)
   - `sase stats project -p tailnet` (one code repo: no block)
   - `sase stats project -p sase -r all`
3. TUI PNG goldens: run
   `just fix-tui-screenshots -- tests/ace/tui/visual/test_ace_png_snapshots_config_center_statistics.py`
   through `/sase_monitor`, since visual runs can be long. Then inspect every creation
   and update in the report. Expected changes: the new single-project snapshot, the repo
   snapshot (now a tree with visible names), and possibly the help and narrow snapshots
   (the new legend). Prove the goldens are host-independent with a targeted `--check`
   run twice, once with `TZ=UTC`.
4. Live TUI confirmation: `sase screenshot` on Admin Center → Statistics → 03 Projects,
   seeded to the current project. Inspect the PNG.
5. `sase tool run check`.

## Non-goals

- No Rust, wire, binding, `sase-core-revision.txt`, or JSON shape change.
- No change to the portfolio card, to sidecar/RECORDS semantics (beyond the Markdown
  parity fix), or to CLI flags.
- No per-repo bars inside the repo table.
