---
tier: tale
title: SDD sidecar guard for unmanaged repos
goal: 'Plan files are only ever committed to the SDD store that owns them, SASE never
  creates or reuses SDD sidecar repositories for repos that do not set is_sase_managed,
  and the rogue sase-org/sase-github--sdd state is cleaned up.

  '
phases:
- id: plan-commit-routing
  title: Commit hook routes plans to their owning SDD store
  depends_on: []
- id: creation-gate
  title: Fail-closed SDD materialization for unmanaged repos
  depends_on: []
- id: provider-guard
  title: sase-github provider honors core creation authorization
  depends_on:
  - creation-gate
- id: remediate
  title: Remove rogue sase-github--sdd state and verify end-to-end
  depends_on:
  - plan-commit-routing
  - creation-gate
  - provider-guard
create_time: 2026-07-15 14:33:49
status: wip
prompt: 202607/prompts/sdd_sidecar_unmanaged_repo_guard.md
---

# Plan: SDD sidecar guard for unmanaged repos

## Incident summary

On 2026-07-15, agent `9e` (project `sase`, xprompts `#gh sase` + `#actstat sase-github` + `#plan`) produced commit
`0b6c6228ff39 "Add SDD plan for github_ci_core_source_alignment"` on the GitHub repository `sase-org/sase-github--sdd`.
That sidecar repo should not exist at all: the `sase-github` repo is not SASE-managed (it has no `sase.yml`, so it never
set `is_sase_managed: true`), and `sase repo init` correctly refuses to initialize SDD resources for it. The plan itself
had already been correctly written and committed to the `sase` project's plans sidecar (`sase-org/sase--plans`, mounted
at `sase/repos/plans/` in host workspaces) by the plan-approval flow — the sidecar copy was a duplicate pushed to a repo
that should not exist.

Forensic timeline (all verified):

- **2026-07-10 14:38 EDT** — `sase-org/sase-github--sdd` was created on GitHub (repo `createdAt` 2026-07-10T18:38:07Z),
  and a materialized `separate_repo` store record (`probed_at` 18:38:09Z) was written to the sibling dev checkout's
  metadata at `~/projects/github/sase-org/sase-github/.sase/ sdd-store.json`, with a local clone at `.sase/sdd/`
  ("Initialize or import SDD companion store"). Runtime SDD materialization ran against a `sase-github` checkout and
  silently created the remote.
- **2026-07-11 / 2026-07-12** — two more plan duplicates were pushed there ("Add SDD plan for
  migrate_actstat_sdd_prompts", "Add SDD plan for sase_5q_record_repair_and_closeout") by the same commit-hook path
  described below. Both plans also exist, correctly, in `sase--plans`.
- **2026-07-15 13:55 EDT** — agent `9e`'s coder implemented its CI fix inside the launch-scoped linked `sase-github`
  checkout and ran `sase commit` from there (via `/sase_git_commit`). The commit workflow's `handle_sase_plan` hook
  re-copied the already-committed plan into that checkout's `.sase/sdd` clone and pushed it to
  `sase-org/sase-github--sdd` (repo `pushedAt` 2026-07-15T17:56:17Z matches).

## Root causes

Three independent defects compounded:

1. **`handle_sase_plan` resolves the SDD store from the commit cwd, not from the plan file**
   (`src/sase/workflows/commit/commit_hooks.py`). When a coder commits code in a linked-repo checkout while `SASE_PLAN`
   points at a plan that already lives (and is already committed) in the host project's plans sidecar, the hook computes
   `should_copy=True` because the plan is not inside the _cwd repo's_ store, copies the plan into that foreign store,
   and commits/pushes it ("Add SDD plan for <name>"). Every repo the coder commits in gets its own duplicate of the
   plan.

2. **Runtime SDD materialization never checks `is_sase_managed`** (`src/sase/sdd/store.py`). `materialize_sdd_store()`
   defaults `sdd_creation_authorized=None`, which preserves "provider-owned behavior":
   `_provider_materialization_result()` dispatches to the workspace provider with `create: True` and no authorization
   gate. Only the explicit `sase repo init` / `sase sdd init` command layer checks `project_management_status()`
   (`src/sase/project_management.py`). Any runtime caller (plan approval actions, accepted-plan handling, bead commands,
   plan-links handler, prompt export) that runs against a checkout of an unmanaged GitHub repo silently creates
   `<owner>/<repo>--sdd`.

3. **The GitHub provider forces creation regardless of what core asked for** (`src/sase_github/workspace_plugin.py` in
   the `sase-github` repo). `ws_materialize_sdd_store` overwrites options with `create: True`, and
   `_require_sdd_creation_authorization` treats an _absent_ `sdd_creation_authorized` key as authorized. There is no
   defense in depth on the provider side.

A contributing factor: `sase-github` has a sibling project record (`PROJECT_STATE: sibling`,
`WORKSPACE_DIR: ~/projects/github/sase-org/ sase-github`), so every ephemeral checkout of `sase-github` resolves its
"primary" to that durable dev checkout via `sase.sdd._paths.get_primary_workspace_dir()`. The rogue store record written
there on 2026-07-10 was therefore inherited by every subsequent launch-scoped linked checkout
(`ensure_workspace_sdd_clone` clones the rogue remote into each), which is why the incident kept recurring.

## Design

### Phase `plan-commit-routing` — commit hook routes plans to their owning store

Rework `handle_sase_plan` in `src/sase/workflows/commit/commit_hooks.py` so the store used for copying, status
transitions, `Complete`/`Add` commits, and the `PLAN=` tag is **the store that owns the plan file**, not whatever store
the commit cwd resolves to:

- If `SASE_PLAN` resolves to a file inside an SDD store clone (a sidecar plans/research clone or a `.sase/sdd`
  separate-repo clone), operate on that owning store: mark `status: done`, commit "Complete SDD plan for <name>" there,
  and tag the code commit with the plan reference. Never copy the plan anywhere else, regardless of cwd. Detecting
  "inside an SDD store clone" should be derivable from the plan path itself (e.g. the containing git toplevel differs
  from the code repo and matches SDD store layout), not from cwd-store equality.
- If the plan is in the code repo being committed (in-tree storage), keep the current staging behavior unchanged.
- If the plan is only in the local `~/.sase/plans/` archive (not in any store), the copy destination must be the store
  of the **host workspace that owns the launch**, resolved via the workspace marker (the same marker-based fallback
  `sase.axe.run_agent_exec_plan_sdd. commit_sdd_files_for_exec_plan` already uses), falling back to current behavior
  when the commit cwd is itself the host project checkout. A commit made from a linked or nested repo checkout must
  never target that checkout's own store.

Regression tests must reproduce the incident shape in `tests/test_commit_hooks_artifacts.py` (or a sibling module): cwd
is a repo with a materialized `separate_repo` record, `SASE_PLAN` points into a different store's clone → assert no
copy, no commit into the cwd store, and a correct `PLAN=` tag; plus an archive-only plan committed from a linked
checkout routing to the host store rather than the cwd store.

### Phase `creation-gate` — fail-closed materialization for unmanaged repos

In `src/sase/sdd/store.py`, gate remote-creating materialization on the target repo's management marker:

- Before `_provider_materialization_result()` can dispatch with `create: True`, check
  `project_management_status(<primary>/sase.yml)` for the primary checkout the record would belong to. If the repo is
  not SASE-managed, raise `SddMaterializationError` with an actionable message (mirror the `sase repo init` wording: set
  `is_sase_managed: true` in the target repository's `sase.yml`) instead of creating anything. Reusing an existing valid
  record for a managed repo stays unchanged; discovery-only paths (`preflight_sdd_sidecar`) stay read-only and
  unchanged.
- Always pass an explicit `sdd_creation_authorized` value through provider options from this path so providers can
  enforce the same contract (phase `provider-guard`): `True` only when the management gate (and, for explicit init, the
  user confirmation) passed.
- Audit the runtime callers of `materialize_sdd_store` (plan approval actions, notification modals, accepted-plan
  handling, bead CLI, plan-links handler, prompt export) to confirm they degrade gracefully when the gate raises: the
  user-facing operation should fail with the actionable message (or skip SDD persistence with a warning where SDD is
  best-effort), never crash a TUI, and never fall back to creating the remote.

Tests: materializing against an unmanaged repo raises without invoking the provider create hook; a managed repo still
materializes; explicit-init authorization semantics are preserved.

### Phase `provider-guard` — provider-side defense in depth

In the `sase-github` repo (open it with `/sase_repo`), change `ws_materialize_sdd_store` in
`src/sase_github/workspace_plugin.py` to stop forcing `create: True`: honor the `create` /`sdd_creation_authorized`
options core supplies, and require explicit authorization before `gh repo create` runs. Keep discovery ("find") behavior
for existing sidecars intact so already-materialized managed projects are unaffected. Version-skew note: with an older
core that omits the flag, the provider should refuse _creation_ (not discovery) rather than assume authorization — this
is the entire point of defense in depth. Update the plugin's tests accordingly and follow that repo's own check
workflow.

### Phase `remediate` — clean up rogue state and verify

1. Verify content parity: all three duplicated plans (`migrate_actstat_sdd_prompts`,
   `sase_5q_record_repair_and_closeout`, `github_ci_core_source_alignment`) already exist with their prompts in the
   `sase--plans` sidecar (confirmed during diagnosis; re-verify with a diff against the rogue clone before deleting
   anything local).
2. Remove the rogue local state from the sibling checkout metadata directory
   `~/projects/github/sase-org/sase-github/.sase/`: `sdd-store.json`, `sdd-materialize.lock`, and the `.sase/sdd/`
   clone. Do not touch any other contents of that checkout.
3. Confirm no other unmanaged repo has accumulated the same rogue state: check the sibling checkouts of `sase-core`,
   `sase-nvim`, `sase-telegram` (and any other `PROJECT_STATE: sibling` project) for unexpected `.sase/sdd-store.json`
   records, and check GitHub for other unexpected `*--sdd` sidecars of unmanaged repos.
4. The GitHub repo `sase-org/sase-github--sdd` itself must be deleted or archived, but that is destructive and needs the
   user's explicit decision: present the findings and the exact command (`gh repo delete sase-org/sase-github--sdd`) to
   the user via `/sase_questions` (or the final report) instead of running it autonomously.
5. End-to-end verification that the incident cannot recur: from a scratch clone of an unmanaged repo, simulate the
   incident flow (commit with `SASE_PLAN` pointing at a store-owned plan; runtime materialization attempt) and confirm
   no sidecar is created, no plan is copied, and error messages are actionable.

## Testing

- `just check` in this repo for phases `plan-commit-routing` and `creation-gate`; the plugin repo's own check workflow
  for `provider-guard`.
- New unit tests are described per phase above; the incident-shaped regression tests are the acceptance bar for "this
  never happens again".

## Risks and compatibility

- The routing change alters `handle_sase_plan` behavior for nested/linked checkouts; the normal single-repo flows
  (in-tree plans, archive-copy into the committing project's own store, sidecar-store plans committed from the host
  workspace) must keep byte-identical behavior — the existing tests in `tests/test_commit_hooks_artifacts.py` cover
  those and must keep passing.
- The creation gate could break a legitimate flow that relied on silent auto-creation for a managed repo missing its
  record; the gate only blocks repos whose `sase.yml` lacks `is_sase_managed: true`, so managed projects are unaffected.
- Provider/core version skew is addressed in `provider-guard` (older core + newer plugin refuses creation; newer core +
  older plugin is protected by the core gate alone).
- No Rust core (`sase-core`) changes are expected: SDD storage policy, commit hooks, and provider dispatch are
  Python-side today; nothing here changes wire formats or behavior another frontend consumes.

## Out of scope

- Redesigning where linked-repo work stores its plans (they correctly belong to the host project's plans sidecar today).
- Migrating the legacy `.sase/sdd` separate-repo storage of other projects to sidecar storage.
- Deleting the rogue GitHub repo without explicit user approval.
