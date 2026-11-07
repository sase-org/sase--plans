---
tier: tale
title:
  Connect never-launched projects' sidecars so artifact_run_prune stops failing closed
goal:
  Enabled, SASE-managed projects that are registered on a machine but have never had an
  agent launch get their existing SDD sidecars connected automatically. The hourly
  artifact_run_prune job then sees every required protection source and stops reporting
  protection_unavailable, while retention stays fail-closed.
size: medium
proposed_by: bbugyi200.athena.19p
proposed_with: claude/opus
create_time: 2026-11-07 12:41:06
status: wip
---

# Plan: Connect never-launched projects' sidecars so `artifact_run_prune` stops failing closed

## Problem

Since 2026-11-07 ~07:00 EST, apollo's hourly `housekeeping/artifact_run_prune` job has
reported `check_error · reason=protection_unavailable` on every run. That blocks the
expired-trash purge, and the job shows up in every error digest:

```
artifact_run_prune protection source unavailable: body:beads
artifact_run_prune protection source unavailable: body:plans
artifact_run_prune protection source unavailable: love:beads
artifact_run_prune protection source unavailable: love:plans
artifact_run_prune: candidates=6797 selected=391 ... unavailable=4 purged_batches=0 ... reason=protection_unavailable
```

## Root cause (verified on apollo and athena)

1. `body` and `love` were registered on apollo with `sase project add` (TUI/CLI project
   onboarding, `mode: add`; journals under `~/.sase/project_onboarding/journals/`). The
   repos were already SASE-managed, so the onboarding plan ran only the `checkout` step.
   It cloned the primary repo and wrote the project spec, and nothing more.
2. A project's SDD sidecars (`beads`, `plans`, `research`, ...) get connected on a
   machine in only two ways: `sase repo init`, or **the project's first agent launch on
   that machine**. On first launch, `src/sase/axe/runner_workspace_protection.py` calls
   `sase.sdd.store.auto_connect_sdd_store`, which writes
   `<primary>/.sase/sdd-store.json` and clones the sidecars under
   `<primary>/sase/repos/<role>`. No agent has launched in `body` or `love` on apollo
   yet. Neither primary checkout has `.sase/sdd-store.json` or `sase/repos/`. The remote
   repos `bbugyi200/{body,love}--{beads,plans}` exist and are non-empty. For comparison,
   `tailnet` was onboarded the same way on 2026-10-17 and got connected 20 minutes later
   by its first launch.
3. `src/sase/core/agent_artifact_run_protection.py`
   (`_REQUIRED_SIDECAR_ROLES = ("beads", "plans")`) requires a live beads and plans
   sidecar for every enabled primary project in the repo inventory. When one is missing,
   it records `<project>:<role>` as an unavailable protection source. The trash purge
   then refuses (`agent_artifact_run_trash_quarantine.py`) and the chop returns
   `check_error`. This fail-closed behavior is deliberate (sase-ba.3: "unavailable
   protection sources blocked apply") and must stay.
4. Nothing ever closes the gap. The `sidecar_auto_sync` chop sees these roles as
   `missing` on every 30 s tick (`missing=3` in its log). By contract it only fetches
   and fast-forwards existing clones and never connects a project. So the error repeats
   hourly until someone happens to launch an agent in each project on apollo.
5. Athena shows the same failure briefly for brand-new projects (`sase-spotify` at 11:40
   EST, `body` at 07:03 EST). It clears there because the create flow connects the
   sidecars shortly afterwards. Apollo never clears.

So the defect is a lifecycle mismatch. Retention assumes every enabled, SASE-managed
project is connected on the machine, but connection only happens lazily at first launch,
and no backstop exists for a project that was registered but never launched.

## Fix

Keep retention fail-closed and close the coverage gap instead. A background backstop
connects sidecars for every enabled, SASE-managed project that is registered on the
machine but not yet connected. It uses the same connect-only path that first launch
uses. The prune's diagnostic also gets a pointer to the cause.

### 1. Connect leg in `sidecar_auto_sync` (new helper module)

Add a new module, for example `src/sase/_sidecar_auto_connect.py`, so the 562-line chop
script does not grow. Call it from `src/sase/scripts/sase_chop_sidecar_auto_sync.py`.

- Candidates are the chop's existing `_enabled_project_records()`. This is the same
  enabled-only set that `collect_repo_inventory()` gives the protection scan. Skip any
  record whose primary `workspace_dir` does not exist.
- A project is unconnected when
  `sase.sdd._store_records.is_materialized_record(read_sdd_store_record(primary))` is
  false. Connected projects cost one cached file stat and do no network work.
- For an unconnected candidate, call
  `sase.sdd.store.auto_connect_sdd_store(primary, 1)`. Workspace 1 is the primary,
  matching `auto_connect_sdd_store`'s own internal calls. It creates nothing remotely.
  It returns `False` for unmanaged or non-remote-backed projects. It raises
  `SddMaterializationError`, with a `sase repo init` remedy, when a required sidecar
  does not exist remotely, for example in the middle of `sase project create`. It
  serializes with concurrent launches and creates through `materialization_lock` inside
  `initialize_sidecars`. Add a test confirming that a second call after a concurrent
  connect is a safe no-op.
- Bound the work:
  - Run the leg **before** the early `if not targets:` return, so a project with no
    `auto_sync` roles is still connected.
  - Connect at most **one** project per tick.
  - Only connect while at least ~60 s of the chop's work budget remains
    (`_WORK_BUDGET_SECONDS` is 90 s of a 120 s SIGKILL timeout).
  - If the budget is too short, count the project as `deferred` instead.
  - Order candidates deterministically (by project key) so a failing project cannot
    starve the others. Its backoff pushes it out of the eligible set.
- Schedule and backoff:
  - Keep per-project connect state keyed separately from the role targets, for example
    `<project>:__connect__`. Reuse `_ScheduleEntry`, `_next_backoff_entry` (exponential,
    capped at 30 min) and `_next_backstop_entry` (5 min).
  - The existing stale-key prune (`live_keys` built from role targets only) must not
    delete connect entries. Either include connect keys in `live_keys` or use a separate
    state file.
  - Back off when a call raises. Use the backstop interval after a `False` result, so
    unmanaged projects are rechecked rarely and cheaply.
  - Drop entries for projects that become connected or disabled.
- Logging and summary:
  - Log one line per attempt:
    `[sidecar_auto_sync] Connected existing SDD sidecars for <project>` on success, or a
    warning carrying the exception text on failure.
  - Add `connected` and `connect_failed` counts to `_summary(...)`. Connect failures
    must **not** change the chop's status or reason on their own. The retention chop is
    the surface that reports the coverage gap, and this keeps sidecar_sync free of
    30-second noise.
  - Update any chop output-contract tests or fixtures that enumerate the summary keys.
- Capture or suppress `auto_connect_sdd_store`'s stderr warning (missing agents sidecar)
  into the chop log. It must not escape as unstructured output.
- Update the chop docstring and the `_sidecar_auto_sync.py` module docstring. Both
  currently say missing clones are never touched. They should say that unconnected
  _projects_ get a connect-only first-use connection, while individual missing role
  clones of an already-connected project are still only reported.

### 2. Point the prune diagnostic at the cause

In `_collect_project_sidecar_refs` (`src/sase/core/agent_artifact_run_protection.py`),
when `_live_sidecar_root` returns `None`, check whether the project's primary has a
materialized SDD store record. The primary is the inventory record with
`kind == "primary"` for that project, so pass it in from
`collect_ace_run_retention_protections`.

- If the project is **not connected on this machine**, record the unavailable source as
  `<project>:<role> (sidecars not connected on this machine)`.
- Otherwise keep the bare `<project>:<role>`. That is the existing case of a connected
  project whose clone is missing.
- Keep the `<project>:<role>` prefix so existing readers stay valid: CLI
  `prune`/`reclaim`/`stats`, the TUI prune modal, and preview publication.
- Blocking semantics do not change. Any unavailable source still blocks purge or apply
  exactly as before.
- Update the tests that assert exact strings only where the fixture project is
  unconnected, for example `tests/test_artifact_protection_scan.py:285`. Leave fixtures
  that only feed pre-built strings alone.

### 3. Docs and config text

Read `sase/memory/docs_writing.md` through `sase memory read docs_writing.md` before
editing these:

- `docs/contribute/sdd-storage.md`, "First Launch On A New Machine": add that the
  `sidecar_auto_sync` backstop performs the same connect-only step for an enabled,
  SASE-managed project that is registered on a machine but has not launched yet. Name
  the effect: artifact-run retention needs those sidecars as protection sources.
- `docs/reference/automation/scheduler-default-routines.md`, `sidecar_sync` table row
  and section: mention the connect leg, the one-project-per-tick bound, and the backoff.
- `src/sase/default_config.yml`, `sidecar_sync.description`: amend the text to match. No
  keymap or config values change.

## Tests

Add tests next to the existing `tests/test_axe_chop_sidecar_auto_sync*.py` (shared
fixtures live in `tests/_axe_chop_sidecar_auto_sync_support.py`) and
`tests/test_artifact_protection_scan.py`. Monkeypatch `auto_connect_sdd_store`; tests
must do no network or real clones.

1. An unconnected enabled project is connected once. The summary shows `connected=1`.
   The next tick does not call connect again because the record is now materialized.
2. A connected project is never passed to `auto_connect_sdd_store`.
3. `SddMaterializationError` leads to backoff, `connect_failed=1`, and an unchanged
   status or reason. A later tick inside the backoff window skips the project.
4. The leg runs even when no project has `auto_sync` roles (the `no_auto_sync_roles`
   path).
5. Only one connect happens per tick when several projects are unconnected. The leg
   respects the remaining work budget.
6. Connect schedule entries survive the role-target stale-key prune.
7. The protection scan labels an unconnected project's missing role
   `"<project>:beads (sidecars not connected on this machine)"`, keeps the bare label
   for a connected project with a missing clone, and both still block.

## Verification

- `just fmt`, then `sase tool run check`. Fix every UNKNOWN failure.
- Manual reasoning check against the apollo evidence above. After deploy, the next
  `sidecar_auto_sync` tick on apollo should log
  `Connected existing SDD sidecars for body` (then `love` on the next tick). The next
  `artifact_run_prune` run should report `unavailable=0`, no `check_error`, and resume
  purging expired trash.

## Rollout note (human action, not part of the agent's work)

Apollo only picks this up after a global reinstall or `sase update`, and agents must not
run those (global install is human-only). Until then, the user can clear apollo's error
right away, with no code change, by launching any agent once in `body` and once in
`love` on apollo. First launch connects the sidecars.

## Out of scope

- Changing the Rust onboarding planner so `sase project add` connects sidecars inline.
  The backstop covers every registration path (add, manual spec, re-enable) within one
  30 s tick.
- Re-cloning individual missing role clones for already-connected projects.
- Existing bead `sase-zo` (`sase repo open`/`list -p <display name>` resolves the wrong
  spec path). It showed up during diagnosis but is unrelated.
