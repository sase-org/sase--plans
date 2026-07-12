---
create_time: 2026-07-08 21:11:20
status: done
prompt: 202607/prompts/automate_sdd_companion_repo.md
tier: tale
---
# Plan: Automate creation of the `<project>--sdd` GitHub companion repository

## Product context / goal

We recently added "separate repo" SDD storage, where a project's SDD files (research, tales, beads, etc.) live in a
dedicated GitHub companion repository instead of in-tree. Today the companion repo is only **created** behind the
explicit `sase sdd migrate --create` command; plain `sase sdd init` (and the `sase init` onboarding that wraps it) will
_discover and clone_ an existing companion repo but will **not create one** if it is missing.

This plan makes companion-repo creation a first-class, automatic part of initialization:

- **`sase sdd init`** (and therefore **`sase init`**, which runs the SDD init spec) will, on a GitHub project,
  automatically **create the companion repo if it does not exist**, then materialize it locally.
- The companion repo lives in the **same GitHub org as the current repo** (derived from `remote.origin.url`) and is
  named **`<project>--sdd`** (double dash), e.g. a checkout of `foo-org/foo` gets `foo-org/foo--sdd`.
- Default **discovery order changes** from `<owner>/sdd` → `<owner>/<repo>-sdd` (single dash) to **`<owner>/<repo>--sdd`
  (double dash) → `<owner>/sdd`**. The project-specific repo is now the preferred/primary name and the one created when
  nothing is found; the bare `<owner>/sdd` remains a discovery-only fallback for teams that share one org-wide SDD repo.
- Output while creating/materializing is **clear, structured, and beautiful**, and every failure mode (no `gh`, not
  authenticated, no network, no org permission, repo already exists) is handled gracefully with an actionable message —
  and never leaves the project in a half- configured "config says separate_repo but no repo exists" state.

Everything is Python + the `sase-github` plugin; **no Rust core changes** are required (SDD storage policy already lives
in Python `sase/sdd/store.py`, and the Rust core only receives fully-resolved paths).

---

## How it works today (grounding)

- **CLI entry**: `sase sdd init` → `sase/main/sdd_handler.py::run_sdd_init`. `sase init` (bare) → `run_init_onboarding`
  runs specs **memory → sdd → skills**; the `sdd` spec reuses `plan_sdd_init` / `run_sdd_init`
  (`sase/main/init_registry.py`). `sase init sdd` is a direct alias that also calls `run_sdd_init`. In all three, the
  onboarding/alias paths call `run_sdd_init` with **`storage=None`** (no `--storage`).
- **Storage resolution**: `sase/sdd/store.py`.
  - `materialize_sdd_store(workspace_dir, workspace_num)` consults a provider hook only when config or provider policy
    is `separate_repo`, writes `.sase/sdd-store.json`, clones, and bootstraps guides/beads. It dispatches
    `ws_materialize_sdd_store` — **which has no `create` flag** (discover-only).
  - `create_sdd_remote(...)` dispatches `ws_create_sdd_remote`, which **does** honor `options["create"]`. Only
    `sase sdd migrate --create` reaches it today.
  - For a fresh GitHub project (`sdd.storage: auto`, no record), `_resolve_sdd_storage` maps the provider's
    `separate_repo` **policy** to `local` until a companion record exists.
- **Provider hooks** (`sase/workspace_provider/_hookspec.py`, `firstresult=True`): `ws_materialize_sdd_store`
  (discover+clone) and `ws_create_sdd_remote` (verify-or-create). Both are implemented **only in the `sase-github`
  linked plugin**; bare-git has no companion concept.
- **`sase-github` plugin** (`src/sase_github/workspace_plugin.py`):
  - `_companion_sdd_candidates(owner, repo)` → `[(owner, "sdd"), (owner, f"{repo}-sdd")]` (single dash, org-wide `sdd`
    **first**).
  - `_discover_companion_sdd_repo` probes candidates in order via `gh repo view`; returns the first `found`, else
    `candidates[0]` marked `not_found` (this is the name that gets created).
  - `_create_github_sdd_repo(host, repo_full_name)` → `gh repo create <owner>/<repo> --private`.
  - `_probe_github_repo` → `found | not_found | unavailable`. On `unavailable`, `_discover_companion_sdd_repo` returns
    `None`, so `ws_create_sdd_remote` returns `None` and the caller surfaces a generic "no provider could verify…" error
    (hides real cause).
  - Owner/repo/host come from `remote.origin.url` via `parse_github_remote_url` (config.py), honoring `sdd.repo.name`
    override.
- **Docs/config** referencing the old naming: `src/sase/default_config.yml:378`, `docs/sdd_storage.md:21,41`,
  `docs/configuration.md:1055`, `docs/sdd.md:234`.

> Note: The `sase-github` plugin is a linked repo. To edit/read it, an implementer must run
> `sase workspace open -p sase-github -r "<reason>" <workspace_num>` (where `<workspace_num>` is the number of the
> primary `sase` workspace they were started in) and use the printed path.

---

## Design decisions (please confirm at approval)

1. **Auto-upgrade to separate-repo on GitHub.** Plain `sase sdd init` / `sase init` on a GitHub project (provider policy
   `separate_repo`) will create + materialize the companion repo and write `sdd.storage: separate_repo`. This changes
   the previous default for GitHub projects (which wrote the legacy `sdd.version_controlled: true` in-tree alias).
   Non-GitHub projects (bare-git, hg — no `separate_repo` policy) are **unchanged**. Users can still opt out explicitly
   with `sase sdd init --storage in_tree|local`.
2. **Naming: `<project>--sdd` (double dash), org = the current repo's origin owner.** Discovery order
   `<owner>/<repo>--sdd` → `<owner>/sdd`. We drop the single-dash `<owner>/<repo>-sdd` candidate entirely (per the
   request's "search … instead"). The `sdd.repo.name` override still wins when set.
3. **Creation only from explicit user commands.** The create capability rides on `ws_create_sdd_remote` (gated by
   `options["create"]`). The broadly-called, setup-time `ws_materialize_sdd_store` stays create-free, so background
   agent setup never silently creates GitHub repos.
4. **Reliability: create/materialize first, write config last.** We only persist `sdd.storage: separate_repo` after the
   repo exists and is materialized, so a failure never strands the project in a "separate_repo but nothing there" state
   that `sase doctor` would flag.
5. **Repo stays `--private`** (existing convention) and gains a `--description`.

---

## Implementation

### A. `sase-github` plugin — naming, creation polish, error clarity

File: `src/sase_github/workspace_plugin.py` (+ `config.py` comment, plugin tests). Open via
`sase workspace open -p sase-github …`.

1. **Rename + reorder candidates** — `_companion_sdd_candidates`:
   ```python
   return [(owner, f"{repo}--sdd"), (owner, "sdd")]
   ```
   Now the primary/created name is `<owner>/<repo>--sdd`, with org-wide `sdd` as a fallback. `_companion_sdd_repo`
   (returns `candidates[0]`) automatically follows; confirm it has no other callers that assumed `sdd` (grep showed it
   is effectively unused outside the module).
2. **Distinguish "created" vs "reused"** — in `ws_create_sdd_remote`, add a **transient** `"created": True` to the
   mapping returned on the create branch (and `False`/omit on the already-exists branch). `normalize_sdd_store_record`
   ignores unknown keys, so it is never persisted to `sdd-store.json`; it exists only so the CLI can say "Created …" vs
   "Using …".
3. **Better creation UX** — extend `_create_github_sdd_repo` to pass
   `--description "SDD companion repository for <owner>/<repo>"`. Treat an "already exists" / name-taken
   `gh repo create` failure as success (re-probe; if now `found`, return a `found` record) so a race or a pre-existing
   repo is adopted rather than erroring.
4. **Classify `unavailable` on the create path** — so real causes surface instead of a generic message:
   - Give `_probe_github_repo` (or a small wrapper) the ability to report _why_ it was unavailable, reusing existing
     `_looks_like_auth_error` / `_looks_like_network_error` plus a tool-missing signal.
   - In `ws_create_sdd_remote` only (not `ws_materialize_sdd_store`, which must stay best-effort and return `None`),
     when discovery is `unavailable`, **raise** a `RuntimeError` with an actionable message, e.g. "GitHub CLI is not
     authenticated — run `gh auth login`", "gh not found — install the GitHub CLI", or "could not reach <host>". The
     main-repo layer already converts provider exceptions into a clean CLI error.

### B. Main repo — a create-aware materialization entry point

File: `src/sase/sdd/store.py`.

1. Refactor the tail of `materialize_sdd_store` (write record → clone → bootstrap guides/beads) into a small private
   helper, `_finalize_materialized_store(primary, workspace, workspace_num, result_mapping) -> SddStore`, so it can be
   shared.
2. Add:

   ```python
   @dataclass(frozen=True)
   class SddInitOutcome:
       store: SddStore
       repo: str | None        # "owner/repo--sdd"
       remote_url: str | None
       created: bool           # True iff the companion repo was just created

   def create_and_materialize_sdd_store(workspace_dir, workspace_num) -> SddInitOutcome: ...
   ```

   Behavior:
   - If a materialized record already exists → finalize/clone and return `created=False`.
   - Else dispatch
     `workspace_provider.create_sdd_remote(primary, workspace, {"workspace_num": …, "create": True, "vcs_name": …})`.
     - `None` → raise
       `SddMaterializationError("This project has no provider that can create a companion SDD repository (only GitHub is currently supported). Use `sase
       sdd init --storage local` for local storage.")`.
     - Record with `discovery == "not_found"` → raise `SddMaterializationError(...)`.
     - Otherwise read the transient `created` flag, `_finalize_materialized_store(...)`, and return the outcome.
   - Provider `RuntimeError`s (auth/network/tool-missing) propagate as `SddMaterializationError` with their message.

3. Leave the existing `materialize_sdd_store(...)` signature and all its current callers untouched (agent setup, plan
   approval, etc.), so blast radius stays minimal.

### C. Main repo — `run_sdd_init` orchestration + beautiful output

File: `src/sase/main/sdd_handler.py`.

1. **Effective-storage helper** `_effective_init_storage(project_root, storage_arg) -> str`: `storage_arg` if given,
   else project `sdd.storage` from `sase.yml` (`_configured_project_sdd_storage`), else the provider policy
   (`sase.workspace_provider.get_sdd_storage_policy_by_vcs` via the workspace's detected VCS), else `None`. This is the
   single source of truth used by **both** `run_sdd_init` and `plan_sdd_init` so preview and execution agree.
2. **New separate-repo branch** in `run_sdd_init` (covers explicit `--storage separate_repo`, existing `separate_repo`
   config, **and** the auto/GitHub-policy default): call `create_and_materialize_sdd_store(project_root, 1)`, print
   structured progress (below), then on success `write_sdd_init_config(path, storage="separate_repo")`,
   `ensure_sdd_initialized`, print the README path, return 0. On `SddMaterializationError`, print a clear `error:`
   message and return 1 **without** writing separate-repo config.
   - Keep `_delete_negative_sdd_record` so a prior "not found" probe cannot block re-creation.
3. **Non-separate-repo paths unchanged**: `in_tree` / `local` write config as today; a project with no policy and no
   config keeps writing the legacy `version_controlled: true`.
4. **Beautiful output** — centralize in the CLI layer using a Rich `Console` (mirroring `init_onboarding` styling). Read
   the persisted record via `read_sdd_store_record` after materialization for the exact repo/remote. Example success:
   ```
   Setting up separate-repo SDD storage for foo-org/foo
     → ensuring companion repository on github.com …
     ✓ created foo-org/foo--sdd                      (or: ✓ using existing foo-org/foo--sdd)
     → materializing SDD store at .sase/sdd …
     ✓ initialized guides + beads and pushed initial commit
   SDD ready: separate repository foo-org/foo--sdd
   ```
   Example failure (no config written, exit 1):
   ```
   Setting up separate-repo SDD storage for foo-org/foo
     → ensuring companion repository on github.com …
   error: GitHub CLI is not authenticated. Run `gh auth login`, then re-run `sase sdd init`.
   ```
   Keep the final machine-readable README path line (`print(readme_path)`) so existing consumers/tests that read
   stdout's last line still work; route the decorative lines to `Console` (stderr or a distinct block) as appropriate.
5. **`plan_sdd_init`** — when `_effective_init_storage` is `separate_repo` and no materialized record exists yet, add
   one `InitAction` summarizing the companion-repo step, e.g. `operation="create"`, `path=<project>/.sase/sdd`,
   `detail="create or connect GitHub companion repository <owner>/<repo>--sdd"`. Derive the name locally where cheap
   (from a record if one exists, otherwise a generic label "the GitHub companion SDD repository" to avoid a network
   probe during planning). Update `_summarize_sdd_actions` accordingly so the onboarding summary reads well.

### D. Onboarding confirmation copy

File: `src/sase/main/init_onboarding.py`. In `_prompt_for_plan`, when `plan.command == "sdd"` and the plan includes the
companion-repo action, append a heads-up to the prompt (parallel to the existing memory note):
`" This may create and push to a GitHub companion repository."` So `sase init` (interactive, no `--yes`) tells the user
before it creates anything.

### E. Docs & config comments (non-memory, safe to edit)

Update the naming/order everywhere it is documented: `src/sase/default_config.yml:378`, `docs/sdd_storage.md` (lines ~21
and ~41), `docs/configuration.md:1055`, `docs/sdd.md:234`. New wording: "GitHub checks `<owner>/<repo>--sdd`, then
`<owner>/sdd`", and note that `sase sdd init` / `sase init` now create the companion repo automatically. (These are not
memory files or provider shims, so no special permission is needed.)

---

## Error-handling matrix

| Condition                                                | Where detected                               | User sees                                                                        | State left behind                                                   |
| -------------------------------------------------------- | -------------------------------------------- | -------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| `gh` not installed                                       | plugin probe/create → RuntimeError           | "gh not found — install the GitHub CLI"                                          | none (no config)                                                    |
| `gh` not authenticated                                   | plugin `unavailable`(auth) → RuntimeError    | "run `gh auth login`"                                                            | none                                                                |
| No network / host unreachable                            | plugin `unavailable`(network) → RuntimeError | "could not reach <host>; check your connection"                                  | none                                                                |
| No permission to create in org                           | `gh repo create` non-zero                    | gh's stderr + "you may lack repo-create rights in <owner>"                       | none                                                                |
| Repo already exists (race/preexisting)                   | `gh repo create` name-taken                  | adopts it: "✓ using existing …"                                                  | normal success                                                      |
| Clone/push fails after create                            | `create_and_materialize` → error             | "companion repo created but clone/push failed; re-run `sase sdd init` to finish" | repo exists remotely; record may be present → recoverable on re-run |
| Not a GitHub project, explicit `--storage separate_repo` | `create_sdd_remote` returns None             | "only GitHub supports companion repos; use `--storage local`"                    | none                                                                |

---

## Testing

**`sase-github` plugin** (`tests/test_workspace_plugin.py`, `tests/test_config.py`):

- Update existing SDD tests to the new order/name: `test_create_sdd_remote_creates_missing_ private_repo` (created repo
  is now `<owner>/<repo>--sdd`), `test_legacy_companion_repo_is_ fallback_when_org_sdd_is_missing` and
  `test_create_sdd_remote_uses_legacy_fallback_when_it_ exists` (fallback is now `<owner>/sdd`, primary is
  `<repo>--sdd`), and any probe-order asserts.
- New: `_companion_sdd_candidates` returns `[(owner, f"{repo}--sdd"), (owner, "sdd")]`.
- New: `ws_create_sdd_remote` sets transient `created=True` on create, not on reuse; adopts a pre-existing repo on "name
  already exists".
- New: create path raises actionable errors on `unavailable` (auth / network / gh-missing), while
  `ws_materialize_sdd_store` still returns `None` (best-effort) for the same conditions.

**Main repo**:

- `tests/main/test_init_sdd_plan.py`: `run_sdd_init` with `--storage separate_repo` now routes through
  `create_and_materialize_sdd_store` (update `test_sdd_run_explicit_separate_repo_ invokes_materialization`); add a
  fresh-GitHub-project test where a fake provider policy is `separate_repo` and `storage=None` → creation happens and
  `sdd.storage: separate_repo` is written; add a bare-git (no policy) test showing unchanged legacy behavior; add a
  failure test asserting exit 1 + no separate-repo config on `SddMaterializationError`; extend `plan_sdd_init` tests for
  the companion-repo action.
- `tests/sdd_store/test_materialize.py` (+ `tests/sdd_store/_helpers.py` fake-plugin pattern): cover
  `create_and_materialize_sdd_store` — creates when missing (`created=True`), reuses an existing record
  (`created=False`), and raises `SddMaterializationError` on `not_found` / `None` provider results.
- `tests/main/test_init_onboarding_entry.py`: assert the sdd prompt copy mentions creating a companion repository when
  the plan includes that action.
- Capture-based assertions on the new structured stdout/stderr lines.

**Validation** (from an ephemeral `sase` workspace, after `just install`): `just fmt`, `just check` (ruff + mypy + fast
pytest), and the plugin's own test suite from its `sase workspace open` path. Manual smoke: in a throwaway GitHub
checkout, `sase sdd init` creates `<owner>/<repo>--sdd`, materializes `.sase/sdd`, and writes
`sdd.storage: separate_repo`; re-run is idempotent ("using existing"); `sase doctor` reports the SDD storage clean.

---

## Out of scope / notes

- No Rust core changes (SDD policy is Python-side; core only receives resolved paths).
- No new persisted config/record schema fields (`created` is transient only).
- `sase sdd migrate` keeps working; it and `sase sdd init` now share `create_and_materialize_sdd_store`'s creation
  semantics via the same `ws_create_sdd_remote` hook, so behavior stays consistent.
- Existing single-dash `<owner>/<repo>-sdd` companion repos (if any were created during the feature's early days) will
  no longer be auto-discovered; they can be adopted by setting `sdd.repo.name` or renaming to `<repo>--sdd`. Called out
  here as the one backward-incompatible edge of the rename.
- A read-only `ws_peek_sdd_remote` hook (to show the exact target name in the `sase init` plan preview without a network
  probe) is a possible follow-up polish; this plan keeps the plan preview name-generic and shows the exact name at run
  time to limit surface area.
