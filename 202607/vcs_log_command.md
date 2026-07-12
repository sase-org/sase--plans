---
create_time: 2026-07-08 14:29:32
status: wip
prompt: 202607/prompts/vcs_log_command.md
tier: tale
---
# Plan: `sase vcs` — a linked-repo-aware, provider-agnostic `git log`

## 1. Goal & product context

Add a new top-level command, **`sase vcs`**, that shows a single **chronological, cross-repository commit timeline**.
When run from anywhere inside a SASE project it aggregates commits from:

1. **The current repo** (the primary project checkout).
2. **Every repo linked** to the current project (`linked_repos` config).
3. **The corresponding SDD store repo** (`<project>-sdd`), when the store is a separate git repo.

It behaves like a smarter `git log`, but:

- It is **VCS-provider-agnostic**. It never shells out to `git` from the command layer; it goes through the existing VCS
  provider abstraction so any current or future provider (bare git, GitHub, or a future Mercurial/jj plugin) works
  uniformly.
- Every commit line makes it **unmistakably clear which repo it came from**.
- The default rendering is **beautiful**: a colored, day-grouped, interleaved timeline in the house Rich style.

### Why this is worth building

Today a SASE project is really a _constellation_ of repos (primary + linked + SDD), but there is no single place to see
"what changed recently across all of them, in order." Agents and humans routinely lose track of activity that spans the
SDD store and linked cores. `sase vcs` gives one authoritative, time-ordered view of the whole constellation.

### Non-goals (this iteration)

- No commit _mutation_ (revert/cherry-pick/checkout) — read-only history only.
- No diff/patch display per commit — subject + metadata only (a `--patch`-style view can come later).
- No graph/branch topology rendering — a flat interleaved timeline is the target.
- No cross-repo dedup (linked repos are independent histories; SHAs do not collide). Noted as a future concern only.

## 2. Command surface (UX)

Primary command:

```
sase vcs log [options]
sase vcs          # bare form defaults to `log`
```

Using a `log` subcommand (rather than a flat `sase vcs`) keeps room for future read-only VCS views (`sase vcs status`,
`sase vcs blame`, ...) without a breaking rename. Bare `sase vcs` defaults to `log`.

### Options

| Option                           | Meaning                                                                                                                                               |
| -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| `-n, --limit N`                  | Max commits in the merged timeline (default 20). Enough commits are fetched per repo so the true top-N by time is correct even if one repo dominates. |
| `-r, --repo NAME`                | Restrict to named repo(s); repeatable. Names are the project name (primary), each linked-repo `name`, and `sdd`/`<project>-sdd`.                      |
| `--current-only`                 | Shortcut: only the current/primary repo.                                                                                                              |
| `--format {pretty,oneline,json}` | Output format (default `pretty`).                                                                                                                     |
| `--color {auto,always,never}`    | Color mode (default `auto`; honors `NO_COLOR`/TTY).                                                                                                   |

Filtering by time/author (`--since`, `--author`) is intentionally deferred so the provider-agnostic `vcs_log` contract
can stay minimal; the design leaves room to add them later as optional abstract parameters.

### Rendering

**`pretty` (default)** — a day-grouped, interleaved timeline. Each repo gets a stable color assigned deterministically
(primary first, then linked sorted by name, then SDD). The colored bullet + colored repo badge act as a "swimlane" cue
so repo membership is obvious at a glance:

```
  sase · sase-core · sase-sdd            ← legend: colored repo names + counts

  ── Today ───────────────────────────────────────────────
   ● 14:22  a1b2c3d  sase        fix(sdd): link managed workspaces to SDD store   · bryan
   ● 13:05  9f8e7d6  sase-core   feat(core): add unified git-log parser           · bryan
  ── Yesterday ───────────────────────────────────────────
   ● 18:40  4c5d6e7  sase-sdd    docs: update research notes                      · bryan
```

- Day headers: `Today` / `Yesterday` / `Mon D` / `Mon D, YYYY` (older years), computed in the user's local timezone via
  `sase.core.time`.
- Per-line: colored `●`, dim `HH:MM`, gold short-SHA (house `#D7AF5F`), bold per-repo colored badge (padded for
  alignment), subject, dim ` · author`.
- Trailing dim warnings block for any repo that could not be read (see Reliability).

**`oneline`** — one commit per line, pipe-friendly:

```
a1b2c3d sase       fix(sdd): link managed workspaces to SDD store
9f8e7d6 sase-core  feat(core): add unified git-log parser
```

**`json`** — machine-readable (`indent=2, sort_keys=True`): a `repos` list, a flat time-sorted `commits` list (each
carrying its repo label + full/short id, author, email, epoch timestamp, subject), and a `warnings` list.

The command mirrors the existing dual-output + color contract used by `sase agent list` and `sase plan search`
(`--json`-style branch returns before any Rich output; `--color` builds the console via the shared `_make_console`
idiom).

## 3. Architecture

The feature is layered to respect the **Rust core backend boundary**: shared, frontend-agnostic domain behavior (parsing
raw VCS log output into structured commits, and interleaving commits across repos into one canonical timeline) lives in
the sibling Rust core, exactly mirroring the existing `git_query` parser family (`parse_git_name_status_z`, etc., which
are pure parsers over stdout the host already collected). Repo _resolution_ and _git execution_ stay host-side, as they
do for every other provider hook.

Litmus check: a future TUI "unified history" panel or a web frontend must show the same ordering/labeling — so parse +
aggregate are core; running the command and resolving which repos exist are host concerns.

### Layer A — Rust core (`sase-core` linked repo)

> Accessed by implementers via `sase workspace open -p sase-core -r "<reason>" <N>`. Follow the existing `git_query`
> pattern end-to-end (pure parser in `sase_core`, PyO3 wrapper in `sase_core_py`, JSON-dict marshalling, pinned wire
> schema version, python-wire parity test).

1. **Neutral commit wire type** — a provider-agnostic record (not git-named, since it is also the return shape of the
   abstract `vcs_log` hook):
   `VcsCommitWire { full_id, short_id, author_name, author_email, timestamp (epoch i64), subject, body }`, serde
   `Serialize/Deserialize`, with a pinned `VCS_LOG_WIRE_SCHEMA_VERSION`. An aggregated variant `AggregatedCommitWire`
   flattens `VcsCommitWire` + `repo: String`.
2. **Pure git-log parser** in the `git_query` module: `parse_git_log(stdout) -> Vec<VcsCommitWire>`. Parses a pinned,
   separator-delimited `git log --format=...` stream (unit separator `%x1f` between fields, record separator `%x1e`
   between commits — the same robust technique already proven in `revert_agent_discovery.py`, so multi-line commit
   bodies never corrupt parsing). `%at` (epoch author time) is captured for timezone-independent sorting.
3. **Aggregator** (frontend-agnostic merge):
   `aggregate_commit_log(repos: Vec<(String /*repo label*/, Vec<VcsCommitWire>)>, limit) -> Vec<AggregatedCommitWire>` —
   sorts by `timestamp` desc, stable tie-break on `(repo, full_id)`, truncates to `limit`.
4. **PyO3 wrappers** in `sase_core_py` (`parse_git_log`, `aggregate_commit_log`) marshalling to/from `PyList[PyDict]` of
   primitives, registered in the `sase_core_rs` pymodule and documented in the module header inventory.
5. **Tests**: Rust unit tests for the parser (renames, empty, trailing record sep, multiline bodies) and aggregator
   (interleaving, tie-break, truncation), plus a python-wire parity test mirroring the
   `DeltaWire`/`GitNameStatusEntryWire` parity tests.

### Layer B — Python core facade + wire mirror (this repo)

6. `src/sase/core/vcs_log_wire.py` — mirror dataclasses `VcsCommitWire`, `AggregatedCommitWire`, `*_from_dict`
   rehydrators, and the pinned `VCS_LOG_WIRE_SCHEMA_VERSION` (companion to `git_query_wire.py`).
7. `src/sase/core/vcs_log_facade.py` — `parse_git_log(stdout)` and `aggregate_commit_log(...)` that call
   `require_rust_binding(...)` and rehydrate dicts into the mirror dataclasses (companion to `git_query_facade.py`;
   retain a `*_python` golden reference for parity tests where the house style requires it).

### Layer C — VCS provider `vcs_log` hook (this repo)

The abstraction has no history capability today; add one uniformly across all providers (no per-runtime/per-provider
special-casing):

8. `_hookspec.py` — new `@hookspec(firstresult=True) def vcs_log(self, cwd, limit)`.
9. `_base.py` — `log(self, cwd, limit) -> list[VcsCommitWire]` on `VCSProvider` (default raising `NotImplementedError`,
   matching the optional-method pattern).
10. `_plugin_manager.py` — a delegating `log(...)` using the **list-return** delegate pattern already used by
    `diff_name_status`/`diff_line_stats` (call the hook, raise `NotImplementedError` on `None`, else return the list).
11. `plugins/_git_query_ops.py` — `@hookimpl vcs_log`: runs
    `git log -n <limit> --no-merges --format=<pinned separators>` via the existing `CommandRunner._run(...)`, feeds
    stdout to the Layer-B facade `parse_git_log`, returns the list. Both `bare_git` and the GitHub provider inherit it
    for free via the shared `GitQueryOpsMixin` — no per-plugin work.

### Layer D — repo resolution + collection service (this repo)

New package `src/sase/vcs_log/`:

12. `models.py` — `LogRepo(name, path, kind)` where `kind ∈ {primary, linked, sdd}`; a
    `VcsLogResult(repos, commits, warnings)` result object.
13. `resolve.py` — resolve the repo set from cwd, reusing existing resolvers only (no reimplementation):
    - Current workspace context via `ensure_project_file_and_get_workspace_num(create_missing=False)` →
      `(project_file, workspace_num, project_name)`.
    - **Primary**: the primary workspace dir (via the project context / `get_primary_workspace_dir`), labeled with
      `project_name`.
    - **Linked**: `resolve_linked_repos_for_project(project_file=, workspace_dir=, workspace_num=, materialize=False)` →
      iterate `.repos`, using each `_ResolvedLinkedRepo.workspace_dir` and `.name`. `materialize=False` avoids side
      effects (never create checkouts just to read a log).
    - **SDD**: `resolve_sdd_store(workspace_dir, workspace_num)`; include **only** when the store is a `separate_repo`
      with an actual `.git` (skip `in_tree`, whose commits already appear in the primary history; skip `local`, which is
      unversioned). Label from the store record's `repo` field, else `sdd`.
    - Apply `--repo` / `--current-only` filtering here.
14. `collect.py` — for each resolved `LogRepo`, `get_vcs_provider(repo.path)` and call
    `.log(cwd=repo.path, limit=limit)`, each wrapped independently so one failing repo becomes a warning, not a crash
    (see Reliability). Then call the Layer-B `aggregate_commit_log` with `(label, commits)` pairs and the limit to
    produce the final ordered timeline. Returns a `VcsLogResult`.
15. `render.py` — the three renderers (`pretty`, `oneline`, `json`) plus a `_make_console(color)` helper and the
    deterministic per-repo color assignment. Reuse house utilities: `sase.core.time.get_timezone()` for local time, the
    `notifications.models.format_relative_time` style for any relative stamps, the `agent list` badge/truncation idioms,
    and the `plan_search_render` grouped layout idiom.

### Layer E — CLI wiring (this repo)

16. `src/sase/main/parser_vcs.py` — `register_vcs_parser(subparsers)`: add the `vcs` parser, a `log` sub-parser with the
    options above, and `set_defaults(vcs_subcommand="log", ...)` so bare `sase vcs` runs `log`.
17. `src/sase/main/vcs_handler.py` — `handle_vcs_command(args)` with a `_HANDLERS` dict dispatching on
    `args.vcs_subcommand` (mirrors `workspace_handler.py`), printing `Usage: sase vcs {log}` + `exit(2)` on an unknown
    subcommand. The `log` handler calls the Layer-D service and the chosen renderer.
18. `src/sase/main/parser.py` — import and call `register_vcs_parser` in the alphabetical registration block of
    `create_parser()`.
19. `src/sase/main/entry.py` — add the `if args.command == "vcs":` dispatch block (lazy import of `handle_vcs_command`),
    alongside the other command blocks. Optionally add a curated `_COMPACT_ROOT_COMMANDS` entry so `vcs` shows in the
    default `sase --help`.

## 4. Reliability & edge cases

The constellation is heterogeneous and partially materialized, so robustness is a first-class requirement:

- **Independent per-repo failure isolation.** Each repo's log is fetched in its own try/except. A missing checkout, a
  non-repo path, a provider that does not implement `vcs_log`, or a `git` error yields a **warning entry** (repo name +
  reason) and the timeline still renders from the repos that succeeded. The command exits `0` when at least one repo was
  read; warnings are surfaced (dim block in `pretty`, `warnings` array in `json`).
- **SDD storage modes.** Only `separate_repo` stores (with a real `.git`) are logged as a distinct repo. `in_tree` is
  deliberately skipped to avoid double-counting the primary history; `local` is unversioned and skipped. This gating
  uses the existing `SddStore` fields, not path guessing.
- **No unintended materialization.** Linked-repo resolution runs with `materialize=False`; reading a log must never
  create or mutate a checkout.
- **Deterministic ordering.** Sort by epoch author time desc with a stable `(repo, full_id)` tie-break, so
  identical-timestamp commits (common in scripted batches) render in a stable order run-to-run — important for snapshot
  tests.
- **Timezone correctness.** All wall-clock formatting routes through `sase.core.time.get_timezone()`; sorting uses epoch
  seconds and is TZ-immune.
- **Empty results.** No commits anywhere → a friendly dim "No commits found" line (pretty) / empty `commits` (json),
  exit `0`.
- **Non-TTY / `NO_COLOR`.** Handled by the `--color`/`_make_console` contract.
- **Not in a project.** If the current dir is not a recognized SASE workspace, fall back to logging just the current
  repo via `get_vcs_provider` (or a clear error if it is not a VCS repo at all).

## 5. Testing strategy

- **Rust** (`sase-core`): parser unit tests (multiline bodies, empty stream, trailing record separator), aggregator
  tests (interleave, tie-break, truncation), and a python-wire parity test.
- **Python core**: facade round-trip tests (`vcs_log_facade` ↔ mirror wire), golden-reference parity where applicable.
- **Provider hook**: a `vcs_log` test against a temporary git repo with a few commits (real `git`), asserting the parsed
  `VcsCommitWire` fields.
- **Resolution** (`vcs_log/resolve`): fake project layouts covering primary-only, primary+linked, and
  primary+separate-SDD, plus the `in_tree`/ `local` SDD skip and `--repo`/`--current-only` filters.
- **Collection**: fake providers (one succeeds, one raises) to prove failure-isolation and warning capture, and correct
  interleaving order.
- **Rendering**: golden-string tests for `oneline` and `json`; a focused `pretty` test with color forced off asserting
  day headers, repo labels, and ordering. (No new PNG visual snapshot needed — this is CLI Rich text, not the ACE TUI.)
- **CLI parser** (`tests/main/test_vcs_parser.py`): mirror `test_project_parser.py` — bare `sase vcs` defaults to `log`;
  option parsing; and the handler's unknown-subcommand `SystemExit(2)` path. The dynamic root help/`list`-defaulting
  cross-cutting tests pick up the new command automatically.

## 6. Build & coordination notes

- This spans **two repos**: `sase-core` (Rust parse+aggregate + PyO3 binding) and `sase` (facade, provider hook,
  service, CLI). Land the core wire/binding first.
- After Rust changes, rebuild the binding with `just rust-install` (`maturin develop --release`) and bump the
  `sase-core-rs` minimum version in `pyproject.toml`; run the version-validation tool if the wire schema changed.
- Because this workspace is ephemeral, run `just install` before `just check`.
- Run `just check` in the `sase` repo before completion (file changes here are not in the bead/research exception
  categories).

## 7. Deliverables checklist

- [ ] `sase-core`: `VcsCommitWire`/`AggregatedCommitWire` + `parse_git_log` + `aggregate_commit_log` + PyO3 wrappers +
      tests.
- [ ] `sase.core.vcs_log_wire` + `sase.core.vcs_log_facade` (mirror + parity).
- [ ] `vcs_log` hook: `_hookspec.py`, `_base.py`, `_plugin_manager.py`, `plugins/_git_query_ops.py`.
- [ ] `src/sase/vcs_log/` package (`models`, `resolve`, `collect`, `render`).
- [ ] CLI: `parser_vcs.py`, `vcs_handler.py`, `parser.py`, `entry.py`.
- [ ] Tests across Rust, core facade, provider hook, resolution, collection, rendering, and CLI parsing.
- [ ] `pyproject.toml` `sase-core-rs` version bump; `just rust-install`; docs/help surfacing.
