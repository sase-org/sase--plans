---
tier: tale
title: Stop SDD store resolution from aborting on an unresolvable agents sidecar
goal:
  Resolving an SDD store never fails because the hidden agents sidecar root cannot be resolved, unrelated commands keep
  working when the owning project key is missing, and the four `just test` failures pass from any checkout — including
  one that has a materialized `.sase/sdd-store.json`.
proposed_by: bbugyi200.athena.u8
create_time: 2026-08-06 13:08:12
status: wip
---

- **PROMPT:**
  [prompts/202608/sdd_store_agents_sidecar_degradation.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/sdd_store_agents_sidecar_degradation.md)

# Plan: Stop SDD store resolution from aborting on an unresolvable agents sidecar

## Problem

`just test` fails with four failures that reproduce only in a checkout that has a materialized sidecar store record on
disk:

```
FAILED tests/test_artifact_file_e2e.py::test_reference_expansion_updates_unused_and_show_consumption - AssertionError: assert 1 == 0
FAILED tests/test_plan_search_cli.py::test_handler_rejects_negative_limit - sase.sdd._store_types.SddMaterializationError: could not resolve the owning SASE project key for the agents sidecar from <checkout>
FAILED tests/test_plan_search_cli.py::test_handler_rejects_invalid_date - sase.sdd._store_types.SddMaterializationError: could not resolve the owning SASE project key for the agents sidecar from <checkout>
FAILED tests/test_bead/test_claimed_status.py::test_show_explains_claim_owner - sase.sdd._store_types.SddMaterializationError: could not resolve the owning SASE project key for the agents sidecar from <checkout>
```

The artifact failure is the same defect one layer up: `handle_show` catches the error and turns it into
`Error: malformed artifact reference: could not resolve the owning SASE project key for the agents sidecar from <checkout>`,
returning `1` instead of `0`.

All four are one bug, and it is a production robustness bug, not only a test bug.

## Root cause

1. `resolve_sdd_store()` (`src/sase/sdd/_store_resolution.py:79-88`) eagerly resolves a filesystem clone root for
   **every** role recorded in `.sase/sdd-store.json`, using the raising helper `sidecar_clone_root()`:

   ```python
   sidecar_dirs = {
       role: sidecar_clone_root(workspace_dir, role)
       for role in record.sidecars
       if role not in {"plans", "beads"}
   }
   ```

2. For the hidden `agents` role, that root is machine-level rather than per-workspace: `resolve_sidecar_clone_root()`
   (`src/sase/sdd/_sidecar_init.py:72-96`) routes it through `infer_project_name_from_cwd()` and
   `hidden_sidecar_clone_dir()`, which resolve to `~/.sase/projects/<key>/repos/agents`. Every other role resolves to
   `<workspace>/sase/repos/<role>` and can never fail.

3. `infer_project_name_from_cwd()` (`src/sase/bead/project_name.py:66-125`) only returns a key when
   `~/.sase/projects/<key>/<key>.sase` exists — the checkout marker, the provider hook, and the registry scan all
   require that spec file. When no key is resolvable, `resolve_sidecar_clone_root()` returns `None` and
   `sidecar_clone_root()` (`src/sase/sdd/_sidecar_init.py:99-105`) raises `SddMaterializationError`.

4. Because that raise happens inside generic store resolution, it aborts **any** command that merely resolves the store
   — `sase plan search`, `sase bead show`, `sase artifact show` — even though none of them ever asks for the agents
   root. `agents` is explicitly never a document role (`document_sidecar_roles()`,
   `src/sase/sdd/_store_types.py:32-49`), and no caller in `src/` requests `kind_root("agents")`.

   The two callers that genuinely want the agents root already degrade instead of exploding:
   `src/sase/axe/runner_workspace.py:165-186` returns `None`, and `src/sase/main/_repo_init_sidecars.py:264-280` calls
   `resolve_sidecar_clone_root()` and emits a warning. `resolve_sdd_store()` is the lone outlier that uses the raising
   variant for a role it was not asked for.

5. Why this shows up only in some checkouts. `.sase/` is gitignored (`.gitignore:61`), so `.sase/sdd-store.json` is
   machine-local state that exists only in the checkout where `sase repo init` ran — the project's primary checkout,
   whose record lists an `agents` sidecar. The autouse `_isolate_sase_home` fixture (`tests/conftest.py:289`) redirects
   `HOME`/`SASE_HOME` to a per-test tmpdir, so no project registry is visible and the project key is never resolvable.
   Tests that resolve the store from the ambient process CWD therefore hit step 3 whenever the checkout they run in has
   that record. In a numbered agent workspace there is no record, and the `.sase/checkout.json` route to the primary is
   itself defeated by the isolated home, so the store resolves `in_tree` and the same four tests pass. That is why the
   failures look machine-specific.

6. Coverage gap that let this ship: no test constructs a store record containing an `agents` sidecar.
   `tests/sdd_store/test_store_roles.py` and `tests/sdd_store/test_resolution.py` only exercise
   `plans`/`beads`/`research`/`designs`, which cannot fail to resolve.

## Reproduction (before the fix)

Run from a directory that carries a sidecar store record, with `$REPO` set to this checkout's root:

```bash
mkdir -p /tmp/sase_ambient_store/.sase
cat > /tmp/sase_ambient_store/.sase/sdd-store.json <<'JSON'
{
  "discovery": "found",
  "host": "github.com",
  "provider": "github",
  "schema_version": 3,
  "sidecars": {
    "agents": {"remote_url": "git@github.com:sase-org/sase--agents.git", "repo": "sase-org/sase--agents"},
    "beads": {"remote_url": "git@github.com:sase-org/sase--beads.git", "repo": "sase-org/sase--beads"},
    "plans": {"remote_url": "git@github.com:sase-org/sase--plans.git", "repo": "sase-org/sase--plans"},
    "research": {"remote_url": "git@github.com:sase-org/sase--research.git", "repo": "sase-org/sase--research"}
  },
  "storage": "sidecar_repos"
}
JSON

cd /tmp/sase_ambient_store && "$REPO"/.venv/bin/python -m pytest \
  -c "$REPO"/pyproject.toml --rootdir="$REPO" \
  "$REPO"/tests/test_plan_search_cli.py::test_handler_rejects_negative_limit \
  "$REPO"/tests/test_plan_search_cli.py::test_handler_rejects_invalid_date \
  "$REPO"/tests/test_bead/test_claimed_status.py::test_show_explains_claim_owner \
  "$REPO"/tests/test_artifact_file_e2e.py::test_reference_expansion_updates_unused_and_show_consumption \
  -q
```

This reproduces all four reported failures verbatim, including the `assert 1 == 0` shape of the artifact failure. It
must pass after the fix. Delete `/tmp/sase_ambient_store` when finished.

## Fix

### 1. Degrade unresolvable sidecar roots in `resolve_sdd_store()`

In `src/sase/sdd/_store_resolution.py`, build `sidecar_dirs` with the degrading `resolve_sidecar_clone_root()` instead
of the raising `sidecar_clone_root()`. Roles whose clone root resolves normally keep their entry; roles that resolve to
`None` are recorded separately with the reason string from `unresolved_project_key_message()` and are omitted from
`sidecar_dirs`.

Keep the `plans` and `beads` lookups on the raising `sidecar_clone_root()`: those roles are required for a sidecar store
and can never degrade to `None`.

Do not swallow anything else. `resolve_sidecar_clone_root()` already documents that only the unresolvable-project-key
case returns `None`; every other failure still raises.

### 2. Carry the unresolved roles on `SddStore` and keep the actionable error at point of use

In `src/sase/sdd/_store_types.py`, add one new field to the `SddStore` dataclass
(`src/sase/sdd/_store_types.py:106-120`), defaulted so every existing keyword construction in `src/` and `tests/` keeps
working:

```python
unresolved_sidecars: Mapping[str, str] = field(default_factory=dict)
```

Keys are roles recorded in the store record whose clone root could not be resolved; values are the human-readable
reason.

Then:

- `kind_root()` and `repo_root_for_kind()` (`src/sase/sdd/_store_types.py:129-155`) raise
  `SddMaterializationError(self.unresolved_sidecars[kind])` when a caller asks for one of those roles, so a caller that
  genuinely needs the agents clone still gets today's actionable message instead of a vague `ValueError`. Roles that are
  simply absent from the record keep raising the existing `ValueError`.
- `split_sidecar_roles()` (`src/sase/sdd/_store_types.py:168-174`) is unchanged: it is derived from `sidecar_dirs`, so
  unresolved roles drop out automatically. That is the correct meaning — the role has no resolved repository path.
- Widen the `sase.sdd.store` re-exports only as far as the new tests and the doctor check need
  (`sase/memory/symvision.md` covers the unused-symbol lint if it fires).

### 3. Keep `sase doctor` honest about the dropped role

`src/sase/doctor/checks_config_sdd.py:257-275` iterates `store.split_sidecar_roles()` and reports a missing clone per
role. Once unresolved roles are omitted from that tuple, the agents sidecar would silently disappear from the health
report. Add an explicit issue so the degradation is visible rather than silent: for each entry in
`store.unresolved_sidecars`, append an `_StorageIssue("error", f"unresolved-{role}-sidecar-root", <reason>)`.

This is strictly better than today: before the fix, `sase doctor` in this state crashes with `SddMaterializationError`
instead of reporting anything.

### 4. Make the four failing tests hermetic

Even with the fix, these tests resolve the SDD store from the ambient process CWD, so their behavior depends on which
checkout runs them. Pin each to an isolated working directory so the result is the same everywhere:

- `tests/test_plan_search_cli.py`: `test_handler_rejects_negative_limit` (line 219) and
  `test_handler_rejects_invalid_date` (line 229) take `tmp_path`/`monkeypatch` and `monkeypatch.chdir(tmp_path)`. The
  file's existing `corpus` fixture already establishes the same convention for the tests that need a real corpus.
- `tests/test_bead/test_claimed_status.py`: add the chdir to the `claimed_view` fixture (line 58) so every test in the
  file is pinned, not just `test_show_explains_claim_owner` (line 97).
- `tests/test_artifact_file_e2e.py`: `test_reference_expansion_updates_unused_and_show_consumption` (line 281) already
  builds its own tmp `ArtifactRefContext`; pin its CWD too, since `handle_show` builds a second context from the process
  CWD.

Verify each still passes after the chdir — the assertions must hold because the resolution is isolated, not because the
ambient checkout happens to be quiet.

## Tests

1. `tests/sdd_store/test_store_roles.py` — new test: write a store record containing `plans` plus `agents`, force the
   project key to be unresolvable (monkeypatch `sase.bead.project_name.infer_project_name_from_cwd` to return `None`, or
   point `sase_projects_dir()` at an empty tmpdir), then assert:
   - `resolve_sdd_store(...)` returns a store instead of raising;
   - `"agents"` is absent from `store.sidecar_dirs` and from `store.split_sidecar_roles()`;
   - `store.unresolved_sidecars["agents"]` contains the
     `could not resolve the owning SASE project key for the agents sidecar` text;
   - `store.kind_root("agents")` raises `SddMaterializationError` with that message;
   - `store.kind_root("plans")` still resolves normally.
2. `tests/sdd_store/test_store_roles.py` — new positive test: with a resolvable project key (a tmp
   `~/.sase/projects/<key>/<key>.sase` spec plus a checkout marker, or a monkeypatched `infer_project_name_from_cwd`),
   `store.kind_root("agents")` equals `~/.sase/projects/<key>/repos/agents` and `"agents"` is in
   `split_sidecar_roles()`. This locks in that the fix does not quietly disable the agents sidecar when it is healthy.
3. Doctor coverage for the new `unresolved-agents-sidecar-root` issue, alongside the existing sidecar-clone checks (find
   the current doctor SDD-storage tests and extend the nearest one rather than adding a new file).
4. The four originally failing tests, verified both in-tree and through the reproduction recipe above.

## Verification

- `just install` first (ephemeral workspaces may have stale deps).
- The reproduction command above: 4 passed.
- `.venv/bin/python -m pytest tests/sdd_store tests/test_plan_search_cli.py tests/test_bead/test_claimed_status.py tests/test_artifact_file_e2e.py tests/test_plan_search_facade.py -q`
- Doctor and store consumers touched by the `SddStore` change:
  `.venv/bin/python -m pytest tests/doctor tests/test_sdd_commit_store.py tests/artifact_refs -q` (adjust paths to the
  real test locations).
- `just check` before replying. Run `just check-full` before landing, since this changes a type used across the store,
  doctor, bead, artifact, and commit paths.

## Notes and non-goals

- This stays in Python. `sase/memory/rust_core_backend_boundary.md` puts shared backend behavior in `../sase-core`, but
  SDD store resolution has no Rust binding today (`src/sase/sdd/` calls `require_rust_binding` only for plan
  header/validate/refs), and this change introduces no new domain logic — it makes an existing Python resolver degrade
  instead of abort. Porting store resolution to the Rust core is separate work.
- Do not "fix" this by relaxing `infer_project_name_from_cwd()` to accept a checkout marker without a registry spec
  file. That would change project-registry semantics well beyond this bug.
- Do not chdir the whole suite away from the repo root in `tests/conftest.py`. Many tests legitimately read repo files
  from the CWD; the targeted pins in step 4 are the safe scope. If a broader ambient-state guard for the test suite
  still looks worthwhile afterwards, file it as a task bead through `/sase_new_task` instead of widening this plan.
