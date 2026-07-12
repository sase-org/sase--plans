---
create_time: 2026-07-12 12:23:40
status: wip
prompt: 202607/prompts/companion_clone_dir_prefix_strip.md
tier: tale
---
# Strip the `<project>--` Prefix from Companion Repo Clone Directory Names

## Problem & Goals

SDD companion repos are currently cloned into workspace checkouts at directories named after the full GitHub repo
basename:

```
<workspace_checkout>/sase/repos/sase--plans       # repo sase-org/sase--plans
<workspace_checkout>/sase/repos/sase--research    # repo sase-org/sase--research
```

The `<project>--` prefix is pure noise inside a workspace that already belongs to exactly one project: every path a user
or agent types (e.g. `@sase/repos/sase--plans/202607/foo.md` in prompts) repeats the project name.

Goals:

1. Clone companion repos into directories named by the repo basename with its `<project>--` prefix stripped:
   `sase/repos/plans` and `sase/repos/research`. The GitHub repo names themselves are unchanged.
2. **No backward-compatibility code.** Old clone locations are migrated once, out-of-band, on this machine for all SASE
   projects (see Phase 3).
3. Normal linked repos (`sase/repos/linked/<name>`) and the internal `.linked-cache` are untouched.

## Survey of Current Machinery (what the implementation must touch)

The recent `sase/repos/linked/` migration already forced every companion-path consumer through one public helper, so the
choke point is small:

- `src/sase/linked_repos.py`:
  - `companion_repo_clone_dir(host_checkout, name)` joins `sase/repos/<name>`; every companion call site passes the
    **repo basename** today (`sase--plans`).
  - `_sdd_companion_repo_names(primary)` returns the authoritative companion repo basenames from the SDD store record
    (`.sase/sdd-store.json` → `companions.plans.repo` / `companions.research.repo`), falling back to `{project}--plans`
    / `{project}--research` when no record exists. `is_sdd_companion_repo()` classifies a linked-repo entry name against
    that set; `_resolve_workspace_dir()` uses it to pick the companion vs. linked path.
  - `_linked_repo_clone_location()` reverse-parses a clone path into `(host_checkout, name, is_companion)`;
    `materialize_linked_repo_workspace()` recomputes the canonical dir from it.
- SDD machinery deriving the clone dir from the record's repo basename via private
  `_companion_clone_dir(workspace, repo)` helpers (each calls `companion_repo_clone_dir` with the basename):
  `src/sase/sdd/store.py` (`resolve_sdd_store`, `materialized_sdd_clone`), `src/sase/sdd/_companion_init.py`,
  `src/sase/sdd/migrate.py`. Notably, **every one of these call sites already knows which companion kind
  (`plans`/`research`) it is resolving**.
- Other companion-path consumers, also kind-aware at the call site:
  - `src/sase/main/sdd_handler.py` — `_split_companion_roots()` builds `roots[kind]` from the record repo basename.
  - `src/sase/doctor/checks_config_sdd.py` — `_regressed_split_companion_paths()` builds `{primary.name}--plans` /
    `--research` paths.
  - `src/sase/llm_provider/commit_finalizer.py` — `_separate_sdd_store_repo_may_exist()` probes both companion clones.
- Name-based (not kind-aware) resolution sites: `_resolve_workspace_dir()` in `linked_repos.py` and
  `resolve_checkout_path()` in `src/sase/main/workspace_handler_list.py` (backs `sase workspace open/path` for a
  companion entry name like `sase--plans`).
- `ensure_companion_sdd_clone()` in `src/sase/sdd/_store_link.py` validates an existing clone **by git remote URL**, not
  by directory name — a renamed clone dir with the same origin is accepted as-is, and a missing dir is re-cloned from
  the record's `remote_url`. This is what makes the one-time `mv` migration safe and sufficient.
- CI checks out both companions at the old paths: `.github/workflows/ci.yml` (`path: sase/repos/sase--plans`,
  `path: sase/repos/sase--research`) with matching assertions in `tests/test_justfile_lint.py`
  (`test_ci_lint_job_validates_split_sdd_companions`).
- Nothing else encodes the companion clone layout: the ace TUI, parser help, xprompt skill templates, and generated
  memory-init output contain no `sase/repos/<repo>--plans` literals (verified by grep); memory generation
  (`src/sase/main/init_memory/config.py`) lists companion **entry names** (`<project>--research`), which do not change.
  The SDD store record stores repo names and remote URLs only — no local paths — so no record migration is needed. Rust
  core (`sase-core`) consumes already-resolved absolute paths and has no clone-path knowledge (per the previous
  migration's survey); **this is a Python-only change**.
- The injected default linked-repo entries (`inject_default_linked_repos()` in `src/sase/_linked_repo_config.py`) keep
  their names (`<project>--plans`, `<project>--research`); on this machine they rarely resolve anyway (their `../<name>`
  primary paths do not exist) — companion clones are materialized by SDD machinery, and classification matters for
  explicitly configured entries and `sase workspace open`.

## Design

### 1. Directory name = companion kind (the stripped suffix)

The companion clone directory name becomes the companion **kind**: `plans` or `research`.

Because the provider enforces `<repo>--plans` / `<repo>--research` companion naming (creation and discovery), the kind
is exactly the repo basename with its `<project>--` prefix stripped — `sase--plans` → `plans`, `bob-cli--research` →
`research`. Deriving the name from the kind rather than by string surgery keeps the mapping total and unambiguous even
for a hypothetical companion whose repo name doesn't match the pattern, and it makes companion paths identical across
all projects:

```
sase/repos/plans                 # SDD plans companion (beads live at plans/beads/)
sase/repos/research              # SDD research companion
sase/repos/linked/<name>         # normal linked repos — unchanged
sase/repos/.linked-cache/<name>  # internal restore cache — unchanged
```

The fixed names `plans`/`research` cannot collide with `linked`/`.linked-cache`, and a normal linked repo that happens
to be named `plans` lives under `sase/repos/linked/plans` — no conflict.

### 2. `linked_repos.py` changes

- `companion_repo_clone_dir(host_checkout, dirname)`: same signature, but the second argument is now the companion
  **directory name (kind)**, not the repo basename. Update docstring/param name.
- Replace `_sdd_companion_repo_names(primary) -> frozenset[str]` with a mapping helper (repo basename → kind) built from
  the SDD store record, falling back to `{f"{project}--plans": "plans", f"{project}--research": "research"}` when no
  record exists.
- Replace `is_sdd_companion_repo(primary, name) -> bool` with `sdd_companion_clone_dirname(primary, name) -> str | None`
  — returns the clone dirname for a companion entry name, `None` for normal linked repos. (The truthiness of the result
  is the old classifier.)
- `_resolve_workspace_dir()`: use the new helper; when it returns a dirname, resolve
  `companion_repo_clone_dir(host_workspace_dir, dirname)`.
- `_linked_repo_clone_location()` / `materialize_linked_repo_workspace()`: no structural change — for the companion
  layout, `path.name` is already the new dirname and recomputing the canonical dir is a no-op.

### 3. Kind-aware call sites pass the kind directly

Delete the three private `_companion_clone_dir(workspace, repo)` basename-derivation helpers and call
`companion_repo_clone_dir(workspace, kind)` directly:

- `sdd/store.py`: `resolve_sdd_store()` (plans/research roots), `materialized_sdd_clone()` (plans).
- `sdd/_companion_init.py` and `sdd/migrate.py`: plans/research roots.
- `main/sdd_handler.py` `_split_companion_roots()`: `roots[kind] = companion_repo_clone_dir(project_root, kind)` — the
  record/repo-basename lookup for path purposes disappears entirely.
- `doctor/checks_config_sdd.py` `_regressed_split_companion_paths()`: pass `"plans"` / `"research"`.
- `llm_provider/commit_finalizer.py` `_separate_sdd_store_repo_may_exist()`: iterate kinds instead of repo basenames.

### 4. Name-based resolution site

- `main/workspace_handler_list.py` `resolve_checkout_path()`: replace the `is_sdd_companion_repo` branch with
  `sdd_companion_clone_dirname(host_primary, ctx.project_name)`; when non-None, the clone dir is
  `companion_repo_clone_dir(host_checkout, dirname)`. `sase workspace open -p sase--plans <n>` keeps its spelling and
  now resolves/creates `sase/repos/plans`.

### 5. CI workflow

- `.github/workflows/ci.yml`: change the two companion checkout `path:` values to `sase/repos/plans` and
  `sase/repos/research` (the `repository:` values are unchanged). The recorded `.sase/sdd-store.json` heredoc is
  untouched (repo names only). Ships in the same commit as the code change, so CI layout and runtime always agree.
- `tests/test_justfile_lint.py::test_ci_lint_job_validates_split_sdd_companions`: assert the new `path:` values.

## Implementation Phases

### Phase 1 — Core rename + call-site updates

- Rework `linked_repos.py` as in §2.
- Update the kind-aware call sites (§3) and the name-based resolution site (§4).
- Update `ci.yml` (§5).

### Phase 2 — Tests + docs

- Update existing tests that encode the old companion dir names — grep `tests/` for `--plans`, `--research`, and
  `sase/repos` (currently 18 files, including `tests/test_linked_repos.py`, `tests/sdd_store/*`,
  `tests/doctor/test_checks_config_sdd.py`, `tests/main/test_workspace_handler_list_path.py`,
  `tests/main/test_sdd_init_handler.py`, `tests/test_bead/*`, `tests/test_sdd*.py`, `tests/test_plan_search_facade.py`,
  `tests/ace/tui/test_startup_watchers.py`, `tests/llm_provider/test_commit_finalizer_auto_sdd_status.py`,
  `tests/test_justfile_lint.py`). Note some hits are repo-name-only (dialog text, record fixtures) and stay as-is.
- New coverage: record-driven dirname mapping (custom companion repo basename still maps to its kind dirname); fallback
  mapping without a store record; `sdd_companion_clone_dirname` returns `None` for normal linked repos;
  `resolve_sdd_store` roots land on `sase/repos/plans` / `sase/repos/research`; `resolve_checkout_path` resolves a
  companion entry name to the stripped dir; a moved clone with matching remote is accepted untouched by
  `ensure_companion_sdd_clone`.
- Docs: update path references found via grepping `docs/` for `sase/repos` and `--plans` — at minimum
  `docs/sdd_storage.md` (storage location tables), `docs/sdd.md`, `docs/beads.md`, `docs/configuration.md`. Repo-name
  references (`<owner>/<repo>--plans`) are unchanged.
- Audit the linked plugin repos (`sase-github`, `sase-telegram`, `sase-nvim`, via `sase workspace open`) and the global
  chezmoi sase config for old companion clone-path literals. Expected: none (providers traffic in repo names and remote
  URLs); update if found.
- Run `just check`.

### Phase 3 — One-time migration of existing clones on this machine

No code reads the old locations after Phase 1, and clone validation is remote-URL-based, so migration is a plain rename
per checkout. For each project from `sase project list` (`sase`, `actstat`, `bob-cli`):

1. Enumerate every checkout via `sase workspace list -j` (the primary is `workspace_num` 0; pass the full project name
   from `sase project list`, or run from within the project).
2. In each checkout that has them, `mv sase/repos/<project>--plans sase/repos/plans` and
   `mv sase/repos/<project>--research sase/repos/research`. Skip missing sources; fail loudly if a destination already
   exists. `mv` preserves unpushed commits, dirty state, and bead caches.
3. Grep `~/.sase/projects/` (`.gp` ChangeSpecs) for old companion clone-path literals and rewrite any that back active
   entries (HOOKS, artifact references). Historical/archived references may be left stale.
4. Verify: re-run the inventory (no `sase/repos/<project>--*` dirs remain), `sase doctor` storage checks are green per
   project, and `sase sdd path plans` prints the new path.

Observed inventory during planning: the sase project holds old-path companion clones in its primary plus ten numbered
checkouts; actstat in its primary only; bob-cli in its primary plus two numbered checkouts. (The two bob-cli numbered
checkouts also still hold the `bob-plugins` clones whose deletion was deferred from the previous linked-repos migration
— finish that deferred cleanup in the same sweep if their agent runs have ended.)

Safety: perform the migration right after the code change lands, ideally while no other agents hold claims (check
`sase project list` CLAIMS). Agents still running the previously installed sase version may lazily re-clone a companion
at its old path; their commits push to the same remote, and the moved clone pulls on next use (ff-only refresh), so no
data is lost — sweep any re-created old-path strays after those runs finish.

## Non-Goals / Explicitly Out of Scope

- No change to companion repo **names** on GitHub, the SDD store record schema, or remote URLs.
- No change to linked-repo entry names: the injected defaults stay `<project>--plans` / `<project>--research`, so
  `sase workspace open -p sase--plans`, env-var names, and memory-generated linked-repo listings are unchanged. (A
  `-p plans` alias is a possible follow-up, not part of this change.)
- No change to normal linked repo layout (`sase/repos/linked/`), the `.linked-cache`, or launch-time clearing.
- No Rust core (`sase-core`) changes.
- No rewriting of historical artifacts: stale absolute paths in old agent metadata, opened-workspace markers, or
  archived ChangeSpecs are tolerated (per the no-backward-compatibility requirement).

## Risks & Edge Cases

- **Concurrent old-runtime agents**: until in-flight runs finish, their installed runtime resolves old paths and can
  re-clone companions there (writes still push to the shared remote). Mitigated by migrating at a quiet moment and doing
  a post-run stray sweep.
- **Custom companion repo names**: a record companion whose basename doesn't match `<project>--<kind>` still gets the
  kind dirname (`plans`/`research`); classification remains record-driven, so resolution and SDD machinery stay
  consistent. Covered by tests.
- **Persisted absolute paths**: pre-migration artifact records and markers referencing `sase/repos/<project>--plans` go
  stale; active `.gp` references are rewritten in Phase 3, historical ones tolerated.
- **User habit change**: prompt references become `@sase/repos/plans/...`; old spellings in saved prompts/xprompts (if
  any) must be updated by hand when reused.
- **Pre-existing red gates**: `just check` has known pre-existing failures on this machine (llm_provider
  `default_effort` tests, pyvision `ChangeSpecProjectFile`, full-suite SIGTERM in sandboxed runs); verify failures
  against a clean baseline before attributing them to this change.
