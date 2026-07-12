---
create_time: 2026-07-08 16:04:39
status: done
prompt: 202607/prompts/vcs_list.md
tier: tale
---
# Plan: `sase vcs list` — the cross-repo constellation at a glance

## 1. Product summary

`sase vcs log` gives a chronological, cross-repository commit _timeline_. This plan adds its structural counterpart:
**`sase vcs list`**, which answers "what repos am I actually looking at, and how healthy/active is each one?"

`sase vcs list` lists **exactly the repositories whose commits `sase vcs log` would include** — the primary repo, every
configured linked repo, and the separate-repo SDD store — and for each shows useful, at-a-glance statistics plus a short
human description. It becomes the **default** action when `sase vcs` is run with no subcommand (today that defaults to
`log`).

Design goals, in priority order:

- **Intuitive** — bare `sase vcs` shows the constellation; the repo set is _guaranteed identical_ to `sase vcs log`
  because both call the same resolver. Same repo → same accent color in both commands.
- **Reliable** — read-only; one broken/missing repo degrades to a warning, never a crash; network is never on the
  critical path and always degrades gracefully.
- **Beautiful** — a clean, color-coded, aligned layout that is visually a sibling of `sase vcs log`, with `pretty` /
  `oneline` / `json` output modes.

### What the user sees (illustrative `pretty` output)

```
  Constellation · 4 repos · 2,331 commits · 6 contributors · updated 2h ago

  ● sase              primary   master   ✎ dirty
      Structured Agentic Software Engineering monorepo
      2,013 commits · 5 contributors · updated 2h ago
      d17ac0bb  feat(vcs): add log filters and full output · Bryan Bugyi

  ● sase-core         linked    master
      Shared Rust core backend for SASE domain behavior and cross-frontend APIs
      268 commits · 3 contributors · updated 5h ago
      a1b2c3d4  refactor: split aggregate crate · Bryan Bugyi

  ● sase-github       linked    master
      GitHub VCS and workspace provider plugin for repository, issue, and PR workflows
      41 commits · 2 contributors · updated 3d ago
      9f8e7d6c  fix: classify enterprise hosts · Bryan Bugyi

  ● sase              sdd        sdd-store
      —
      112 commits · 1 contributor · updated 6h ago
      4c5d6e7f  chore(sdd): sync beads · sase-bot

  ⚠ sase-nvim: checkout not found (~/code/sase-nvim)
```

The bullet color for each repo is drawn from the **same palette, in the same resolved order**, that `sase vcs log` uses
— so a repo is the same color in both commands.

## 2. Guiding principle: reuse the log constellation verbatim

The single most important design decision: **`sase vcs list` resolves its repo set through the exact same code path as
`sase vcs log`** — `resolve_log_repos(cwd=..., repo_filters=..., current_only=...)` in `src/sase/vcs_log/resolve.py`,
which returns ordered `LogRepo` objects (`name`, `path`, `kind ∈ {primary, linked, sdd}`) plus non-fatal warnings.

This makes the two commands _provably_ consistent: the promise "lists every repo whose commits `sase vcs log` would
include" is upheld by construction, not by parallel logic that could drift. No new resolution logic is written.

## 3. CLI surface & the "default subcommand" switch

### New command

```
sase vcs list [-c/--color auto|always|never]
              [-f/--format pretty|oneline|json]
              [-r/--repo NAME ...]        # repeatable; restrict to named repos (project / linked / "sdd")
              [-o/--current-only]         # primary repo only
              [-s/--sort default|name|commits|recent]
              [--no-fetch]                # skip any network description lookup (Phase 2)
```

Flags deliberately mirror the relevant subset of `sase vcs log` (`--color`, `--format`, `--repo`, `--current-only`) so
the two commands feel like one family. Commit-timeline-only flags (`--since/--until/--author/--limit/--reverse`) are
**not** carried over.

`--sort` is a small, genuinely useful addition: default keeps the resolved constellation order (primary → linked-by-name
→ sdd); `recent` sorts by last-commit time, `commits` by total commit count, `name` alphabetically.

### Making `list` the bare-`sase vcs` default

The repo already has generic machinery for this: `parser.py::_default_list_subcommands` makes **any** command group that
has an exact `list` child default to it, copies the `list` subparser's option defaults up to the group, and prints a
delegation notice (`default_list_delegation_notice`) — this is exactly how `sase plan` (bare → `plan list`) behaves.

Therefore the switch is idiomatic and small:

- Add the `list` subparser under the `vcs` group in `parser_vcs.py`.
- **Remove** the current explicit `set_defaults(vcs_subcommand="log", limit=..., ...)` block that forces bare `sase vcs`
  → `log`; replace with the `plan`-style `set_defaults(vcs_subcommand="list")`. The generic machinery then defaults the
  bare group to `list`, copies `list`'s option defaults up, and emits the standard delegation notice.
- Update the subparser `metavar="{log}"` → `{list,log}`.
- Register `"list": _handle_list` in `vcs_handler._HANDLERS` and update its usage-error string.
- `sase vcs log` (explicit) is unchanged and remains fully available.

**Behavior change:** bare `sase vcs` now runs `list` and prints
`No subcommand provided for 'sase vcs'; delegating to 'sase vcs list'.` (consistent with `sase plan`). This is the
user-requested behavior and matches an existing convention.

## 4. Statistics: what we show and where each number comes from

Per-repo statistics, chosen to be **useful, cheap to compute, and reliable**:

| Stat                                  | Source (git, run inside the provider)                                                                                 |
| ------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| Kind badge (`primary`/`linked`/`sdd`) | from the resolver — free                                                                                              |
| Total commits                         | `git rev-list --count HEAD` (fast aggregate, no per-commit parse)                                                     |
| Contributors                          | `git shortlog -sne HEAD` → distinct author identities (count; identities used for the constellation-wide union total) |
| Last activity                         | `git log -1 --format=<pinned>` → reuse existing `parse_git_log`; gives short id, subject, author, and relative age    |
| Current branch                        | `git rev-parse --abbrev-ref HEAD` (reuse `parse_git_branch_name`)                                                     |
| Working state                         | `git status --porcelain` (reuse `parse_git_local_changes`) → clean / `✎ dirty`                                        |
| Path                                  | from the resolver (rendered dim, `~`-abbreviated)                                                                     |

The header/summary line aggregates across the constellation: repo count, summed commit count, **union** of contributor
identities, and the single most-recent activity time.

Deliberately deferred (keep v1 tight; easy follow-ups): repo age/first-commit, ahead/behind vs upstream, per-repo "top
contributor", lines-changed. Noted so reviewers see the ceiling.

### New provider hook: `vcs_repo_stats`

Following the exact pattern of the `vcs_log` hook, add one **bundled** query hook that returns a provider-agnostic stats
record for a repo. One hook call per repo (the git impl issues the handful of commands above), so both the bundled
bare-git provider and the GitHub provider inherit it for free via the shared `GitQueryOpsMixin` (uniform across runtimes
— no provider-specific branching).

- `_hookspec.py`: add `vcs_repo_stats(cwd) -> VcsRepoStatsWire | None` (`firstresult=True`), grouped with the other
  query hooks (`vcs_log`, etc.).
- `_base.py` + `_plugin_manager.py`: add the matching abstract method + dispatch, mirroring how `vcs_log` is threaded
  through.
- `plugins/_git_query_ops.py`: implement `vcs_repo_stats` for all git-based providers. Must be robust to an **empty
  repo** (no commits → `total_commits=0`, `last_commit=None`, `contributors=0`) and to a detached HEAD (branch may be
  `None`).

### Core wire + facade (respecting the Rust-core boundary)

Mirror `core/vcs_log_wire.py` + `core/vcs_log_facade.py`:

- `core/vcs_repo_stats_wire.py`: a frozen `VcsRepoStatsWire` (`total_commits: int`, `contributors: tuple[str, ...]`
  (identities), `last_commit: VcsCommitWire | None`, `branch: str | None`, `dirty: bool`), a schema-version constant,
  and a `*_from_dict` rehydrator — the stable Python↔host contract.
- `core/vcs_repo_stats_facade.py`: a pure assembly helper that turns the raw git outputs into a `VcsRepoStatsWire`,
  **reusing the existing Rust-backed `parse_git_log`** for the last-commit parse.

**Boundary note (design decision, called out for review).** The genuinely non-trivial parse — turning a `git log` stream
into commit records — already lives in Rust (`parse_git_log`) and is reused here, so the boundary is honored for the
hard part. The remaining "aggregation" is trivial (`int(count)`, count distinct emails, pick the newest record), so it
is implemented as a thin local adapter in `core/`, which the boundary rule explicitly permits ("call through the Rust
binding **or a thin local adapter**"). If a future web/editor frontend needs byte-identical stats, this facade is the
single obvious place to promote to a Rust binding — exactly as `vcs_log_facade` already keeps Python "golden" references
alongside its Rust calls. No changes to `../sase-core` are required for this feature.

## 5. Descriptions

Descriptions resolve through a layered, fail-soft precedence so the common case is instant and offline:

1. **Linked-repo config description** — `linked_repos[].description` is a _required_ field in the config schema and is
   already authored for every linked repo (e.g. "Shared Rust core backend…"). This is curated, local, and instant. It is
   the primary source for linked repos. (It is currently dropped during resolution; the list service re-reads it from
   the merged config by repo name via a small helper — no change to the shared resolver / `LogRepo`.)
2. **GitHub description** _(Phase 2)_ — for repos with a GitHub remote and no config description (notably the primary
   repo and the SDD store), fetched via a provider hook (see below), cached, and skippable with `--no-fetch`.
3. **None** — rendered as a dim `—`.

Each rendered description carries a `description_source` (`config` / `github` / `null`) exposed in `json` output for
transparency.

### Phase 2 provider hook: `vcs_repo_description`

The clean, uniform way to express "some providers can describe a repo, others can't" is a provider hook (not GitHub
knowledge baked into the core service):

- `_hookspec.py` / `_base.py` / `_plugin_manager.py` (in this repo): declare `vcs_repo_description(cwd) -> str | None`;
  the bundled bare-git provider returns `None`.
- **In the `sase-github` plugin repo**: implement the hook using the `gh` CLI
  (`gh repo view <owner/repo> --json description -q .description`), with a small on-disk TTL cache keyed by `owner/repo`
  so repeated `sase vcs list` runs stay instant, a short timeout, and total graceful degradation (any failure → `None`,
  no warning noise). Parse `owner/repo` from the remote with the existing `_normalize_git_url` helper.

Phase 2 is cleanly separable: Phase 1 ships a complete, beautiful command with config-sourced descriptions and no
network dependency; Phase 2 layers in the "pulled from GitHub" enrichment for the primary/SDD repos.

## 6. New service package: `src/sase/vcs_list/`

Structured as a sibling of `src/sase/vcs_log/`, reusing its resolver and shared styling:

- `models.py` — `RepoListing` (a `LogRepo` + its `VcsRepoStatsWire` + resolved `description` + `description_source` + an
  `error: str | None` for stat-collection failures) and `VcsListResult` (`repos`, aggregate `totals`, `warnings`).
- `collect.py` — `run_vcs_list(cwd, *, repo_filters, current_only, no_fetch, sort, provider_factory=...)`: resolve via
  `resolve_log_repos`, then per-repo **failure-isolated** `vcs_repo_stats` collection (one try/except per repo,
  mirroring `collect_vcs_log`), then description resolution, then totals + sort. A repo whose stats can't be read stays
  **listed** (its name/kind/path are known) but is annotated and adds a warning — this keeps the "every repo in the
  constellation" promise honest while surfacing what `log` couldn't read.
- `render.py` — `pretty`, `oneline`, `json` renderers.

### Shared styling

Extract the small shared pieces so `list` and `log` are visually one family and impossible to drift: move
`_REPO_PALETTE`, `_GOLD`, `_make_console`, and the resolved-order color assignment into a tiny shared module (e.g.
`src/sase/vcs_log/_style.py`) and have both renderers import it. This guarantees a repo is the same accent color in
`list` and `log`.

### Output formats

- `pretty` — hand-built `rich.Text` blocks (not a `rich.Table`) to match the `vcs log` aesthetic and gracefully handle
  variable-length descriptions/subjects: a header/summary line, then a colored `●` identity line (name padded to a
  common width, kind badge, branch, dirty flag), a dim description line, and a dim stats+last-commit line. Warnings
  printed dim with the same `⚠` convention as `log`.
- `oneline` — one aligned, colorless, pipe-friendly row per repo
  (`name  kind  <commits>c  <contrib>a  <age>  branch  description`).
- `json` — a stable, `sort_keys=True` envelope: `{"repos": [...], "totals": {...}, "warnings": [...]}`, each repo
  carrying `name`, `kind`, `path`, `description`, `description_source`, `total_commits`, `contributors`, `branch`,
  `dirty`, and `last_commit` (or `null`). No Rich on the JSON path.

### Exit codes

Mirror `_handle_log`: `0` when at least one repo was resolved, `1` when nothing readable was found (e.g. not in a SASE
workspace or a VCS repo). Warnings surface in every format.

## 7. Files touched (map)

**This repo (Phase 1):**

- `src/sase/main/parser_vcs.py` — add `list` subparser + options; drop the forced-`log` default block; `metavar` update.
- `src/sase/main/vcs_handler.py` — `_handle_list` + `_HANDLERS["list"]` + usage string.
- `src/sase/main/parser.py` — reword the compact root-help entry for `vcs` (now list-first).
- `src/sase/vcs_provider/_hookspec.py`, `_base.py`, `_plugin_manager.py` — `vcs_repo_stats` hook.
- `src/sase/vcs_provider/plugins/_git_query_ops.py` — git impl of `vcs_repo_stats`.
- `src/sase/core/vcs_repo_stats_wire.py`, `src/sase/core/vcs_repo_stats_facade.py` — new.
- `src/sase/vcs_list/__init__.py`, `models.py`, `collect.py`, `render.py` — new package.
- `src/sase/vcs_log/_style.py` — extracted shared palette/console (imported by both renderers).
- `docs/vcs.md` — new `### sase vcs list` section + note that bare `sase vcs` now delegates to `list`; adjust the
  `sase vcs log` note accordingly.
- `docs/cli.md` — add the `sase vcs list` line.

**Phase 2 (optional, GitHub descriptions):**

- `src/sase/vcs_provider/{_hookspec,_base,_plugin_manager}.py` — `vcs_repo_description` hook; bare-git returns `None`.
- `sase-github` plugin repo — implement `vcs_repo_description` (gh + TTL cache); wire `--no-fetch` through the list
  service.

## 8. Testing strategy (mirror the `vcs_log` test suite)

- **Parser** (`tests/main/test_vcs_parser.py`): update `test_bare_vcs_defaults_to_log` → now asserts bare `sase vcs`
  resolves to `vcs_subcommand == "list"` with `list`'s defaults and the delegation marker set; add cases for `list` flag
  parsing and rejection of bad `--format`.
- **Handler dispatch**: `_HANDLERS["list"]` routes; unknown subcommand still exits 2.
- **Provider hook** (`tests/test_vcs_provider_vcs_repo_stats.py`, new): `vcs_repo_stats` against a real temp git repo —
  populated repo, **empty repo**, dirty vs clean, detached HEAD.
- **Core facade** (`tests/test_core_vcs_repo_stats.py`, new): assembly + `parse_git_log` reuse + wire round-trip via
  `*_from_dict`.
- **Collect** (`tests/test_vcs_list_collect.py`, new): resolution reuse, per-repo failure isolation (a bad repo becomes
  a warning, others still listed), totals + union-of-contributors, `--sort` variants, description precedence (config vs
  none), `--repo`/`--current-only` filters.
- **Render** (`tests/test_vcs_list_render.py`, new): golden-style — render to `StringIO` with `color="never"`; assert
  JSON key ordering/shape, `oneline` alignment, and `pretty` substrings (header totals, kind badges, `✎ dirty`, `⚠`
  warnings, `—` for missing description).
- **Root help** (`tests/main/test_parser_root_help.py`): reflect the reworded `vcs` entry.
- Grep the test suite for existing assumptions that bare `sase vcs` == `log` and update them.

Run `just install` then `just check` (lint + mypy + tests) before wrapping up. Prefer targeted `pytest` subsets for the
new/edited test modules plus the static gates (lint/mypy); the full suite can be flaky under sandboxed runners.

## 9. Rollout / phasing

- **Phase 1 — the command (self-contained, no linked-repo or Rust edits).** CLI + default switch, `vcs_repo_stats`
  hook + core wire/facade, the `vcs_list` service, all three render formats, config-sourced descriptions, docs, and the
  full test suite. Delivers a complete, intuitive, reliable, beautiful `sase vcs list`; linked repos get curated
  descriptions, primary/SDD show `—` until Phase 2.
- **Phase 2 — GitHub descriptions (optional enhancement).** `vcs_repo_description` hook + `sase-github` implementation
  with caching and `--no-fetch`, filling in descriptions for the primary/SDD repos from GitHub.

## 10. Open decisions (surfaced, not blocking — I've picked defaults)

1. **GitHub descriptions now or later?** Recommended as Phase 2 (needs edits to the `sase-github` plugin repo). Phase 1
   stands alone and is fully shippable.
2. **Delegation notice on bare `sase vcs`.** Kept, to match `sase plan`'s existing convention.
3. **Failed-repo rows.** A repo that resolves but whose stats can't be read stays listed with an annotation + warning
   (rather than being dropped), so the constellation view is complete and honest.
