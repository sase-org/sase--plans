---
create_time: 2026-07-08 12:57:48
status: wip
prompt: 202607/prompts/fix_plan_chain_sdd_ref_resolution.md
tier: tale
---
# Plan: Fix plan-chain crash when the SDD store lives under the primary repo

## Problem

A plan-chain agent (planner → auto-approve → coder) crashes with `SystemExit: 1` right after the plan is approved,
whenever the SDD store is configured as **`local`** or **`separate_repo`** and the agent runs from a **managed /
ephemeral workspace** (the `sase_<N>` clones) rather than from the primary repo checkout.

Observed failure (agent that ran with the `Tale` auto-approval, i.e. `%a:tale` = `commit_plan=True, run_coder=True`):

```
❌ ERROR: The following file(s) referenced in the prompt do not exist:
  - @.sase/sdd/tales/202607/<plan_name>.md
⚠️ File validation failed. Terminating workflow to prevent errors.

Traceback (most recent call last):
  ...
  File ".../xprompt/workflow_executor_steps_prompt.py", line 236, in _execute_prompt_step
    expanded_prompt = preprocess_prompt_late(expanded_prompt)
  File ".../llm_provider/preprocessing.py", line 162, in preprocess_prompt_late
    prompt = process_file_references(prompt, is_home_mode=is_home_mode)
  File ".../file_references.py", line 246, in process_file_references
    sys.exit(1)
SystemExit: 1
```

The planner phase completes fine and the plan is committed. The coder follow-up prompt it hands off is:

```
#gh:<changespec> @.sase/sdd/tales/<YYYYMM>/<plan_name>.md

The above plan has been reviewed and approved. Implement it now.
```

That `@` reference fails validation because the file does not exist **at that path relative to the coder's working
directory**, and `process_file_references` treats an unresolved `@`-reference as a fatal error (`sys.exit(1)`), which
kills the whole agent.

## Root Cause

The plan-chain follow-up reference builders assume the SDD store lives at `<cwd>/.sase/sdd`, but for
`local`/`separate_repo` storage it actually lives under the **primary** repo, which is a _different directory_ from the
ephemeral workspace where the coder runs.

Concretely, in `src/sase/sdd/store.py`, `_sdd_dir_for_storage(...)` resolves the store directory as:

- in-tree storage → `<workspace>/sdd` (inside the repo tree)
- `local` / `separate_repo` storage → `<primary>/.sase/sdd`

where `<primary>` is `get_primary_workspace_dir(...)`. For a managed `sase_<N>` workspace the primary is the canonical
checkout (e.g. `.../github/<org>/<repo>/`), **not** the ephemeral workspace clone. The store's docstring states this
explicitly: _"local and separate-repo storage both live under `<primary>/.sase/sdd`"_.

The two reference builders in `src/sase/axe/run_agent_exec_plan_sdd.py`:

- `build_saved_plan_ref(...)` — used for the **coder** follow-up (`@<ref>` … "Implement it now").
- `build_sdd_plan_ref(...)` — used for the **epic/legend** follow-ups (`#bd/new_epic:<ref>` / `#bd/new_legend:<ref>`);
  `build_epic_plan_ref(...)` delegates to it.

both contain the same non-version-controlled branch:

```python
try:
    sdd_relative = sdd_plan_path.relative_to(sdd_dir)
    return (Path(".sase") / "sdd" / sdd_relative).as_posix()   # <-- CWD-relative .sase/sdd/<rel>
except ValueError:
    ...
```

Because `sdd_plan_path` genuinely lives under `sdd_dir` (the primary's `.sase/sdd`), `relative_to(sdd_dir)` **succeeds**
and the builder returns a `.sase/sdd/<rel>` path that is implicitly relative to the coder's CWD. The coder's CWD is the
ephemeral workspace, whose own `.sase/sdd` is either absent or a stale/independent checkout that does **not** contain
the just-committed plan. So the reference points at the wrong place and validation fails.

This only manifests when `<primary> != <workspace>` (managed/ephemeral workspaces). When the agent runs directly in the
primary checkout (`<primary> == <cwd>`), the `.sase/sdd/<rel>` path happens to resolve, which is why the bug was latent
until managed separate-repo SDD stores were adopted (recent "support migrated separate repo SDD stores" work).

Evidence that the write itself was correct (so the fix is purely about the _reference_, not the commit): the plan was
committed to the primary store at `<primary>/.sase/sdd/tales/<YYYYMM>/<plan_name>.md` and also archived to
`~/.sase/plans/<YYYYMM>/<plan_name>.md`; only the ephemeral workspace's CWD-relative `.sase/sdd/...` copy was missing.

### Why this is a real (not cosmetic) defect

- Any plan-chain run with `run_coder=True` (the default `Tale`/approve path) under `local` or `separate_repo` storage in
  a managed workspace crashes at hand-off — the planning work is lost and the coder never starts.
- Epic and legend auto-approvals hit the identical path via `build_sdd_plan_ref`.
- The existing test
  `tests/test_axe_run_agent_exec_plan_epic_refs.py:: test_epic_prompt_uses_non_vc_sdd_ref_from_google_sibling_workspace`
  **asserts the buggy behavior**: it sets `workspace_dir` and `sdd_dir` to _sibling_ directories and then expects the
  `.sase/sdd/epics/...` CWD-relative ref, which would not resolve from that workspace. This test encoded the exact
  defect and must be corrected.

## Fix

Make the follow-up plan reference resolve from the **coder's working directory** (`workspace_dir`). Keep the clean
`.sase/sdd/<rel>` relative form **only when it actually resolves from that CWD**; otherwise fall back to the absolute
committed-plan path, which `process_file_references` accepts and (for home-rooted paths) copies into the agent workspace
— the same mechanism already used today for the not-committed fallback (`plan_result.plan_file` under
`~/.sase/plans/...`).

Extract a single shared helper (e.g. `_resolve_committed_plan_ref(...)`) in `src/sase/axe/run_agent_exec_plan_sdd.py`
and call it from the `sdd_plan_path.exists()` branch of both `build_saved_plan_ref` and `build_sdd_plan_ref`. Behavior:

1. **Version-controlled (in-tree):** unchanged — return `sdd_plan_path.relative_to(workspace_dir)` (absolute on
   `ValueError`). The in-tree store is inside the workspace tree, so this already resolves.
2. **Non-version-controlled (`local`/`separate_repo`):**
   - Build the candidate `.sase/sdd/<rel>` from `sdd_plan_path.relative_to(sdd_dir)`.
   - Return that candidate **only if** `<workspace_dir>/<candidate>` exists on disk (co-located or symlinked store — no
     behavior change for the cases that already worked).
   - Otherwise return the absolute committed path (`sdd_plan_path.as_posix()`), which resolves from any CWD and is
     validated/handled by `process_file_references`.
   - Preserve the existing `relative_to(workspace_dir)` → absolute fallback for the `ValueError` case.

This is deliberately conservative: it changes output **only** for references that previously did not resolve (the
crashing case), and leaves every already-working case (in-tree, and co-located/symlinked non-VC stores) byte-for-byte
identical.

The `sdd_plan_name`-based _synthetic_ fallback in `build_sdd_plan_ref` (used only when `sdd_plan_path` is
`None`/missing) is out of scope: current callers pass `sdd_plan_name=None` when the plan is not committed, so that
branch is inert for these flows. Leave it unchanged but note it as a latent follow-up if we ever start passing a
non-committed `sdd_plan_name`.

### Files to change

- `src/sase/axe/run_agent_exec_plan_sdd.py`
  - Add `_resolve_committed_plan_ref(...)` helper.
  - Route `build_saved_plan_ref` and `build_sdd_plan_ref` exists-branch through it.

### Tests

- `tests/test_axe_run_agent_exec_plan_epic_refs.py`
  - Fix `test_epic_prompt_uses_non_vc_sdd_ref_from_google_sibling_workspace` to assert the resolved reference (absolute
    committed path, since that filesystem layout has no reachable workspace `.sase/sdd`). Optionally add a symlinked
    variant that keeps asserting the `.sase/sdd/<rel>` form to lock in the "no regression when co-located/symlinked"
    guarantee.
- Add focused unit tests for the previously-untested `build_saved_plan_ref` covering:
  - **Managed separate-repo:** `workspace_dir` distinct from `sdd_dir`, `version_controlled=False`, plan under `sdd_dir`
    but NOT reachable via `<workspace_dir>/.sase/sdd` → expect the absolute path.
  - **Co-located store:** `sdd_dir == <workspace_dir>/.sase/sdd`, plan present there → expect `.sase/sdd/<rel>`
    (unchanged).
  - **In-tree:** `version_controlled=True` → expect `sdd/<rel>` (unchanged).
  - **Not committed:** `sdd_plan_path=None` → expect `fallback_plan_file` (unchanged).
- Add a regression assertion that the composed coder follow-up prompt's `@`-reference resolves from the workspace CWD
  for a `separate_repo` store (e.g. via `handle_plan_marker`, mirroring the epic tests), so the
  `process_file_references` `sys.exit(1)` path can never re-trigger silently.

## Validation

- `just install` then `just check` (lint + mypy + targeted tests).
- Run the new/updated tests directly: `pytest tests/test_axe_run_agent_exec_plan_epic_refs.py` plus the new coder-ref
  test module.
- Manual sanity: in a managed workspace with `sdd.storage: separate_repo`, run a plan through the `Tale` auto-approval
  and confirm the coder follow-up launches instead of crashing at hand-off.

## Risks & Mitigations

- **Over-broadening the change** could alter references that already work. Mitigated by gating the fallback on an
  on-disk existence check so only the unresolved (crashing) case changes.
- **Absolute home-rooted references get copied into `.sase/home/`** by `process_file_references`. This is acceptable and
  already the established behavior for the not-committed fallback; the coder only _reads_ the plan, so a copy is fine.
- **Ordering vs. `#gh` pre-steps:** the absolute committed path lives under the primary and always exists regardless of
  workspace checkout/stash state, so validation is stable across the follow-up workflow's pre-steps.

## Out of Scope

- Implementing the original feature the failing agent was planning (auto-commit/push of the SDD store from mutating bead
  commands + finalizer fallback). That plan was successfully authored and archived to
  `~/.sase/plans/<YYYYMM>/auto_commit_sdd_store.md`; it is a separate effort and unaffected by this fix. This plan only
  removes the infrastructure crash that prevented that (and any other) plan-chain hand-off from proceeding.
- Refreshing/syncing the ephemeral workspace's own `.sase/sdd` checkout. The primary store is authoritative; referencing
  it directly is the correct, race-free fix rather than trying to keep a per-workspace clone in sync.
