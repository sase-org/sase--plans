---
tier: tale
title: Name the owning repository in every published commit table
goal:
  Every commits table SASE publishes into a sidecar repo carries a Repo column naming the repository each commit belongs
  to — on bead pages, agent pages, and family pages alike — so a reader never has to infer a commit's repo from context.
create_time: 2026-07-30 09:29:22
status: done
---

- **PROMPT:** [202607/prompts/commit_table_repo_column.md](prompts/commit_table_repo_column.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-b5.4.w1--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-b5.4.w1.md#member-code)
  - [bbugyi200.athena.sase-b5.4.w1--plan](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-b5.4.w1.md#member-plan)
- **COMMITS:**
  - [63d0ca5](https://github.com/sase-org/sase/commit/63d0ca504d48b8daed6702e38d79443d77af44cb) — feat: show repository
    names in commit tables

# Plan: Name the owning repository in every published commit table

## 1. Context

SASE publishes exactly three per-commit Markdown tables, all of them into sidecar repos:

| Table                      | Renderer                                                                  | Published to   | Header today                                   |
| :------------------------- | :------------------------------------------------------------------------ | :------------- | :--------------------------------------------- |
| Bead page → `## Commits`   | `render_commits()`, `src/sase/bead_pages/rendering_tables.py:110`         | beads sidecar  | `Commit \| Subject \| Bead \| Committed (UTC)` |
| Agent page → `## Commits`  | `render_agent_commits()`, `src/sase/agents_sync/rendering_commits.py:13`  | agents sidecar | `Commit \| Subject \| Committed (UTC)`         |
| Family page → `## Commits` | `render_family_commits()`, `src/sase/agents_sync/rendering_commits.py:34` | agents sidecar | `Role \| Commit \| Subject \| Committed (UTC)` |

None of them names a repository. Everything else that renders "commits" renders a _count_ (`render_phases()` and
`render_agents()` in the same bead-pages module, `roster.py:33`, `rendering_index_pages.py:179`) and is untouched by
this plan.

Two facts shape the work:

**Bead pages already carry per-commit repository identity.** `BeadCommitAssociation.repository` is a
`BeadCommitRepository(name, root, kind, is_primary)` (`src/sase/bead_pages/associations/models.py:22`), populated by the
primary/sidecar/linked history walk in `src/sase/bead_pages/associations/_build.py:133`. Rendering currently spends that
identity on a _conditional_ label prefix: `rendering_tables.py:127-130` renders `sase-core@9701511` for a non-primary
commit and a bare `9701511` for a primary one. That is exactly the "you have to already know" shape this plan removes —
the repo is invisible for primary commits and smuggled into the sha cell for the rest.

**Agent and family pages have no per-commit repository identity, and cannot cheaply gain one.** `CommitRecord`
(`src/sase/agents_sync/models.py:62`) is documented as "Portable primary-repository commit attribution" and holds only
`sha`, `subject`, `committed_at`. It is a wire record: `_commits_from_json()` (`src/sase/agents_sync/io.py:201`) rejects
any row whose key set is not exactly `{sha, subject, committed_at}`, under `BUNDLE_SCHEMA_VERSION`. And the data really
is single-repo — `_historical_associations()` (`src/sase/agents_sync/inventory.py:358`) walks `target.primary_checkout`
and nothing else. So on those pages the owning repo is a _per-page_ constant: the project's primary repo. This plan
renders it as such and passes it as a render argument, never as a `CommitRecord` field, so that if agent commit capture
later spans repos the column becomes per-row without a wire migration.

Boundary check: no part of this lives in the Rust core. `sase-core` knows bead _page paths_ and titles
(`crates/sase_core/src/artifact_ref/mod.rs:757`, `crates/sase_core/src/editor/completion.rs:631`) but contains no page
Markdown rendering and no commit tables. This change is Python-only.

## 2. The column contract

One rule, applied to all three tables:

- **The `Repo` column immediately precedes the `Commit` column.** Bead:
  `Repo | Commit | Subject | Bead | Committed (UTC)`. Agent: `Repo | Commit | Subject | Committed (UTC)`. Family:
  `Role | Repo | Commit | Subject | Committed (UTC)` — `Role` stays first because it is the family page's organizing
  axis, matching the member table directly above it.
- **The column is always present**, even when every row shares one repository. The point is that a page is read on
  GitHub, out of context.
- **The cell is the repository's inventory name**, escaped with `md_cell()`, plain text (not a code span) — the same
  treatment `Role` gets on family pages. Names come from one source of truth: `record.slug or record.name`, which yields
  `sase` for the primary repo, `sase--beads` / `sase--plans` / `sase--agents` for sidecars (the GitHub repo name a
  reader will recognize), and `sase-core` / `chezmoi` for linked repos.
- **Unknown repository renders `—`**, the placeholder already used for absent values across both page families
  (`bead_pages/roster.py:38`, `agents_sync/rendering_family_page.py:98`).
- **The `repo@sha` label qualification is removed** from `render_commits()`. It becomes redundant, and leaving both
  would render `sase-core` in the Repo cell next to `sase-core@9701511` in the Commit cell.

Commit ordering does not change: bead rows stay sorted by `(committed_at, repository.name, sha)` via the existing
`sort_key`, agent/family rows stay sorted by `(committed_at, sha[, role])`. Do not group rows by repository — these
tables are chronological.

## 3. Shared repository-name helper

`src/sase/bead_pages/associations/_build.py:207` already has the naming rule as a private `_repository_name()`, and
`targets.py` needs the same rule. Promote it rather than duplicating it:

- Add `repo_display_name(record: RepoRecord) -> str` to `src/sase/repo_inventory.py`, returning
  `record.slug or record.name`, and add it to that module's `__all__` (currently `repo_inventory.py:514`).
- Replace `_build.py:_repository_name()` (line 207) with the shared helper at all three of its call sites
  (`_build.py:171`, `:191`, `:198`).

## 4. Bead pages

`src/sase/bead_pages/rendering_tables.py`, `render_commits()` only:

- Header → `"| Repo | Commit | Subject | Bead | Committed (UTC) |"`, separator → `"|---|---|---|---|---|"`.
- Delete the `display_label` block (lines 127-130); the commit cell goes back to `label = f"`{md_code(row.label)}`"`
  wrapped in the existing `row.target` link.
- Repo cell: `md_cell(row.repository.name) if row.repository is not None else "—"`. `repository` is `None`-defaulted on
  the dataclass, so the `None` branch is a real path for any association built outside `_build.py`.
- Emit the repo cell first in the row f-string.

Nothing else in the module changes. `render_phases()` and `render_agents()` keep their commit _counts_, which aggregate
across repositories and stay correct as-is.

## 5. Agent and family pages

Thread the project's primary repo name from the repo inventory (same source of truth as §3) down to the two renderers.
Every new field is optional with a `None` default — `ProjectTarget` and `ProjectHoodInventory` are frozen dataclasses
constructed positionally at roughly twenty test sites, so append, never insert.

- **`src/sase/agents_sync/models.py:130`** — `ProjectTarget` gains `primary_repo_name: str | None = None` as its last
  field.
- **`src/sase/agents_sync/targets.py:169`** — pass `primary_repo_name=repo_display_name(primary)` when building the
  target; `primary` is the primary `RepoRecord` already in hand there.
- **`src/sase/agents_sync/inventory_models.py:47`** — `ProjectHoodInventory` gains
  `primary_repo_name: str | None = None`, placed after `primary_remote_url` and _before_ the `_selection_exclusions`
  `field(init=False)` so positional construction keeps working.
- **`src/sase/agents_sync/inventory.py:155`** — pass `target.primary_repo_name` through in the returned
  `ProjectHoodInventory(...)`.
- **`src/sase/agents_sync/publication.py:189`** — pass `commit_repo_name=inventory.primary_repo_name` alongside the
  existing `commit_url_base=`.
- **`src/sase/agents_sync/rendering.py:24`** — `render_browsing_payload()` gains `commit_repo_name: str | None = None`
  (keyword-only, defaulted like `commit_url_base`), forwarded through `_render_run_pages()` to both page renderers.
- **`src/sase/agents_sync/rendering_agent_page.py:27` and `rendering_family_page.py:37`** — accept
  `commit_repo_name: str | None` as a required keyword-only argument (no default, so a missed call site is a type error,
  not a silently blank column) and forward it.
- **`src/sase/agents_sync/rendering_commits.py`** — `render_agent_commits()` and `render_family_commits()` gain the same
  required keyword-only `commit_repo_name: str | None`; `_render_commit_table()` gains `repo_name: str | None`, builds
  the header per §2, and emits `md_cell(repo_name) if repo_name else "—"` immediately before the commit cell in both the
  role and no-role shapes.

Where imported foreign hoods are concerned this stays correct: publication is per project (`resolve_project_targets()`
keys by `project_key`), and every hood rendered in one payload belongs to that project, whose primary repo is the one
being named.

## 6. Tests and goldens

Bead pages — `tests/test_bead/test_bead_page_rendering.py`:

- In `_fixtures()`, give the `BeadCommitAssociation` a real
  `BeadCommitRepository("sase", Path("/repos/sase"), "primary", True)` so the goldens show a realistic `sase` cell
  instead of the unknown-value placeholder.
- Hand-edit `tests/test_bead/golden/bead_pages/root.txt:45` and `descendant.txt:27` (header, separator, and the single
  commit row). There is no refresh flag for bead-page goldens — only the agents goldens have one — so these are edited
  directly and verified by rerunning the byte-stability test.
- Replace `test_non_primary_commit_is_qualified_without_changing_primary_bytes` (line 247) with coverage of the new
  contract: a primary-repo row renders `sase` in its own cell, a linked-repo row renders `sase-core`, neither commit
  cell contains `@`, and the two pages differ only in that cell.
- Add a focused test that `repository=None` still renders the column, with `—`.

Agent and family pages:

- `tests/agents_sync/test_rendering.py` — pass `commit_repo_name` through `render_browsing_payload()` in the
  commit-bearing cases and assert the rendered header and cell on both an agent page and a family page; add one case
  asserting the `—` fallback when the name is unknown.
- `tests/agents_sync/goldens/active-no-chat.md` (agent page, line 23) and `deep-family.md` (family page, line 25) —
  regenerate with `just test-cov`-equivalent pytest run using `--sase-update-agents-goldens`, then rerun _without_ the
  flag, as `tests/agents_sync/test_publication.py:247` instructs.
- `tests/agents_sync/test_publication.py:387` — the fixture builds `ProjectHoodInventory(primary_remote_url=...)`; add
  `primary_repo_name=` and assert the published agent page carries that name, which covers the whole inventory →
  publication → rendering thread.
- `tests/agents_sync/test_inventory.py` — assert `build_project_hood_inventory()` carries the target's
  `primary_repo_name` onto the inventory.
- `tests/agents_sync/test_commit_publication_target_resolution.py` — extend `test_resolves_known_repository_kinds`
  (line 54) to assert the resolved target's `primary_repo_name`, including a project whose primary record has a slug
  (slug wins) and one with only a name.

Add a direct unit test for `repo_display_name()` beside the existing repo-inventory tests (slug present, slug absent).

## 7. Out of scope

- **Making agent-page commits multi-repo.** Adding a repo to `CommitRecord` means a `BUNDLE_SCHEMA_VERSION` bump, a
  strict-shape change in `_commits_from_json()`, matching changes across `bundles.py`, `v2_run_io.py`,
  `v2_snapshot_io.py`, and cross-machine import compatibility. Worth doing on its own; not bundled here. The signature
  chosen in §5 (`commit_repo_name` as a render argument) is what keeps that future change from having to re-plumb
  anything.
- **Linking the Repo cell to the repository's hosted page.** There is no `github_repo_url()` helper today, and the
  commit cell already links into the correct repository via `commit_url_for_repository()`. Plain text keeps the table
  scannable.
- **The ChangeSpec commits table** in `src/sase/main/search_handler.py:339` — a TUI detail view, not a published page,
  and its rows are ChangeSpec entries rather than repository commits.
- **The ACE Artifacts commits pane** (`src/sase/ace/tui/widgets/artifacts/commits_collection.py`) — not a page.
- Commit ordering, `MAX_RENDERED_COMMITS`, and the `… and N more commits` overflow line all stay exactly as they are.

## 8. Verification

`just install` first (workspaces drift), then `just check`. Then confirm against real data:

```bash
sase bead pages refresh --bead <a bead with commits>   # inspect the rendered Commits table
sase agent sync --check                                 # local, network-free reconcile
```

The bead-page check should show a `Repo` column whose primary rows read `sase` and, for a bead whose work touched a
sidecar, rows reading `sase--beads` or `sase--plans` — with no `@`-qualified sha anywhere.
