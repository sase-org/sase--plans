---
tier: tale
title: Make the beads sidecar lazy instead of auto-cloned
goal:
  Managed SASE projects no longer clone the beads sidecar during workspace preparation;
  the store materializes on demand the first time a command needs it, and every
  generated config, doc, and agent-instruction surface agrees.
size: medium
proposed_by: bbugyi200.athena.0dq
create_time: 2026-08-25 15:07:09
status: wip
---

# Plan

## Problem

Every launched agent in a numbered workspace currently pays for a full clone of the
`<project>--beads` sidecar during workspace preparation, whether or not that agent ever
touches a bead. The clone is unconditional because managed projects mark the `beads`
sidecar `auto_clone: true`.

That auto-clone predates on-demand materialization. It is no longer needed: the bead
CLI, the agent-launch bead claim, and the artifact/plan surfaces all materialize the
beads sidecar themselves when they need it.

Measured on `sase-org/sase` at the time of writing:

- A launch-time clone (`git clone --reference-if-able <primary sidecar> --dissociate`)
  takes **~2.3 s** wall clock and leaves a **164.28 MiB** pack (the plans sidecar leaves
  78.48 MiB).
- 23 of 24 numbered workspaces hold their own copy: **~3.9 GiB** of duplicated packs.
- A plain network clone of the same repo takes **~50 s** (147 s CPU at ~300 %; GitHub's
  pack generation dominates) and leaves only a 40.5 MiB server-optimized pack. This
  number matters — see finding 3.

## Verified Findings (do not re-derive)

Every claim below was read out of the tree at plan time. Line numbers are from that
revision; re-locate by symbol if they have moved.

### 1. `auto_clone` is the only thing that clones beads at launch

- `src/sase/axe/runner_workspace.py::prepare_launch_workspace_repos` evicts
  `sase/repos/` and then calls `ensure_workspace_sdd_clone`, which for sidecar storage
  materializes **only** the `plans` role
  (`src/sase/sdd/_store_workspace.py::ensure_workspace_sdd_clone`). It never touches
  beads.
- `src/sase/axe/run_agent_runner_setup.py::prepare_linked_repo_workspaces_if_needed`
  filters on `repo.auto_clone`. This is the sole launch-time beads clone.
- The other launch-path resolutions
  (`run_agent_runner_setup.refresh_linked_repos_for_workspace`, `run_agent_phases`,
  `agent/launch_spawn.py`) all pass `materialize=False`, so resolution itself does not
  clone. (Note that `sase.linked_repos._resolve_workspace_dir` _does_ materialize
  sidecars regardless of `auto_clone` when `materialize=True` — that path is used by
  `sase repo open`, `sase workspace path`, and revert, not by agent launch.)

### 2. Bead commands already materialize on demand

- `src/sase/bead/cli_location.py::resolve_beads_location(materialize=True)` calls
  `ensure_beads_sidecar_clone` for schema-3 sidecar stores.
- `src/sase/bead/cli_common.py::get_project` resolves with `require_existing=True` first
  and falls back to `materialize=True` when the location is missing or unusable;
  `get_read_view` funnels into `get_project` for non-read-only stores. Every mutating
  and most reading bead commands land here.
- `src/sase/main/bead_fast_path.py::try_handle_bead_fast_path` deliberately does _not_
  materialize (`execute_bead_cli(argv)` defaults `materialize=False`). When the store is
  cold it returns `None` and argparse's slow path materializes. Slow-path callers with
  no fallback (`artifact_cli/create.py`, `bead/cli_refs.py`,
  `bead/cli_work_from_plan_store.py`, `bead/cli_crud_create.py`) already pass
  `materialize=True`.
- `src/sase/axe/run_agent_runner_bead.py::claim_bead_for_agent_launch` materializes the
  beads sidecar itself, with a comment stating that the workflow steps that clone it run
  _after_ the claim. Bead-launched agents are already independent of `auto_clone`.

**Conclusion: the user's claim is correct.** Dropping `auto_clone` for beads does not
break bead commands or bead-driven launches.

This was confirmed end to end, not only by reading: with `sase/repos/beads` moved aside
in workspace `sase_26`, `sase bead list --status open` returned the correct four open
tasks and left a fresh, healthy `sase/repos/beads` clone behind.

### 3. Blocker: the on-demand clone path has no reference repo (fix this first)

That same experiment took **49 s**. The two clone paths are not equivalent:

- `sase/_linked_repo_workspaces.py::_materialize_remote_identified_sidecar` (the
  `auto_clone` launch path) passes `reference_repo=Path(primary_dir)`, so
  `sdd/_store_clone_ops.py::clone_sdd_store` adds
  `--reference-if-able <primary sidecar> --dissociate`. ~2.3 s.
- `sase/sdd/_store_workspace.py::ensure_beads_sidecar_clone` (the on-demand path) calls
  `ensure_sidecar_sdd_clone(clone_dir, record.beads.remote_url, strict=True, fresh=fresh)`
  with **no** `reference_repo`. Full network clone, ~50 s.

Shipping the config change alone would replace a 2.3 s launch-time clone with a 50 s
stall inside the first bead command — strictly worse for the common case. **Fix the
reference first** (change 0 below), then flip the default.

### 4. What this does and does not buy (be honest in the commit message)

In _this_ repo, `just check` and `just check-full` both run `_lint-flags`
(`Justfile:302`), which shells `tools/sase_bead list -T flag --status all` through
`tools/check_feature_flags::_list_flag_beads`. `just _lint-symvision` likewise reads
bead status (`SASE_SYMVISION_BEAD_STATUS_ONLY=1`). So any agent that changes files here
still materializes the store — just later and only once, instead of unconditionally
during workspace prep.

The win, assuming change 0 lands, is therefore:

- Real for agents that never run a bead command or `just check`: read-only reporters,
  planners, monitor follow-ups, TUI/ACE work, chops, question answering. They save ~2.3
  s and 164 MiB each.
- Real for every other managed SASE project, which has no `_lint-flags` equivalent.
- Real for disk: only workspaces that actually touch beads keep a clone.
- Roughly neutral in wall clock for a file-changing agent in the sase repo — the same
  2.3 s clone just happens during `just check` instead of during workspace prep.

Do not oversell this as "every launch gets 2.3 s faster". It is not.

### 5. The one real behavior change: generated agent instructions

`src/sase/main/init_memory/config.py` (sidecar loop, ~line 315) skips any sidecar with
`auto_clone: true`, `disabled: true`, or a role in `HIDDEN_SIDECAR_ROLES`. Everything
else **must** carry a `description` or `sase memory init` fails, and it is then rendered
into the "Repositories" list of `AGENTS.md` / `CLAUDE.md` / every provider shim, under
prose that tells agents to open it with `/sase_repo`.

Flipping `beads` to lazy therefore has two consequences that must be handled together:

1. `sase memory init` would start rejecting the managed `beads` entry for a missing
   `description`. Note that `sase doctor`'s
   `checks_config_repos._missing_description_problems` only checks the `custom` bucket
   (`checks_config_repos.py:238`), so doctor would stay silent while `sase memory init`
   failed — a real inconsistency.
2. `sase--beads` would be added to every agent's always-loaded core memory with
   `/sase_repo` guidance, which is the wrong tool: agents reach bead state through
   `sase bead`, and `sase/memory/sase_beads.md` already says so.

**Decision (already made — implement it, do not re-open):** exclude the reserved `beads`
role from the generated linked-repo memory listing regardless of `auto_clone`, the same
way `HIDDEN_SIDECAR_ROLES` excludes `agents`. Core memory then stays byte-for- byte
unchanged, and no `description` is required on the entry. Leave `plans` alone: it is
materialized unconditionally by `ensure_workspace_sdd_clone` anyway, so its listing
behavior does not change.

### 6. Surfaces confirmed unaffected

- **CI.** `tools/ci_bootstrap_sidecars` clones by role presence in
  `repos.sidecar.builtin`/`custom`, not by `auto_clone`
  (`STORE_SIDECAR_ROLES = ("plans", "research", "beads")`). Keeping the `beads:` key in
  `sase/sase.yml` is sufficient; CI is untouched.
- **`auto_sync`.** Independent of `auto_clone` by design
  (`tests/test_linked_repo_sidecar_config.py::test_auto_sync_defaults_false_and_is_independent_of_auto_clone`);
  it converges the _primary_ checkout's clone, which always exists. Keep
  `auto_sync: true` on beads.
- **Env vars.** `ResolvedLinkedRepo.to_env` skips unmaterialized repos, so
  `SASE_LINKED_REPO_*_DIR` is simply absent until the clone exists — the documented
  contract for lazy repos. `SASE_SDD_BEADS_DIR` (`sdd/env.py:59`) is a computed path and
  is set either way; its consumers (`workflows/commit/bead_hooks.py`,
  `completion/candidates/catalog_sdd.py`, `tools/check_bead_note_migration`) all guard
  on existence.
- **ACE TUI bead display.** `agent/bead_display.py::lookup_bead_issue` resolves through
  `bead/workspace.py::get_project_beads_dirs_for_project`, i.e. the _primary_ checkout's
  canonical beads dir, and `ace/tui/models/agent_bead.py` passes `local_only=True`
  precisely so the TUI never materializes. Unaffected.
- **Commit finalizer.** `workflows/commit/bead_hooks.py:83` gates bead sync on
  `isdir(cwd/"sase/repos/beads")`. A cold store means no bead mutations happened, so
  skipping is correct.

### 7. Surfaces that degrade quietly — verify, do not assume

These read the store with `require_existing=True` and return `None`/empty rather than
materializing. Confirm each still behaves acceptably when the store is cold, and fix or
file follow-ups for any that do not:

- `src/sase/feature_flags/beads.py::load_flag_bead_snapshots` → `None`, consumed by
  `doctor/checks_flags.py` and `feature_flags/cli_views.py`. Cold-store `sase flag list`
  and `sase doctor` would report no flag beads.
- `src/sase/bead/sync.py::schedule_current_bead_refresh` / `refresh_current_bead_store`
  → no-op (correct).
- `src/sase/bead/conflict_resolver.py:630`, `src/sase/bead/cli_work_commit.py:187`.

## Change List

### Code — prerequisite

0. `src/sase/sdd/_store_workspace.py::ensure_beads_sidecar_clone` — pass a
   `reference_repo` to `ensure_sidecar_sdd_clone` so the on-demand clone borrows the
   primary checkout's objects exactly like the launch path does. The function already
   holds `primary`; the reference is `sidecar_repo_clone_dir(primary, "beads")`. Skip it
   when that path resolves to `clone_dir` itself (workspace 1, where the primary clone
   _is_ the target).

   `clone_sdd_store::_matching_clone_reference` already validates that the reference is
   a Git checkout whose `origin` matches the recorded remote and never trusts its refs,
   so an absent, stale, or wrong-remote primary clone degrades to a plain clone rather
   than failing.

   Land this as its own commit and confirm the speedup before touching any config: with
   `sase/repos/beads` moved aside, `sase bead list` should drop from ~50 s to ~2 s.

### Code

1. `src/sase/_linked_repo_config.py` — in `inject_default_linked_repos`, the `defaults`
   tuple (~line 328): change the `beads` row's `auto_clone` from `True` to `False`.
   Leave `auto_sync` at `True` and leave the `plans` row untouched.
2. `src/sase/repo_inventory.py` (~line 243) — replace
   `metadata.get("auto_clone", kind in {"plans", "beads"})` with a `plans`-only default.
3. `src/sase/main/_repo_init_config.py` (~line 186) — the `beads` entry that
   `sase repo init` writes becomes `auto_sync: true` only (drop `auto_clone: true`).
4. `src/sase/main/init_memory/config.py` — in the sidecar loop, skip the reserved
   `beads` role alongside `HIDDEN_SIDECAR_ROLES` so a lazy beads sidecar is neither
   description-required nor listed in generated instructions. Name the reason in a
   comment: bead state is reached through `sase bead`, never `/sase_repo`.

### Config

5. `sase/sase.yml` — `repos.sidecar.builtin.beads` drops `auto_clone: true`, keeps
   `auto_sync: true`. Add a short comment in the style of the existing
   `sase-research-artifacts` note explaining why beads is lazy (on-demand
   materialization in `sase bead` and in the launch bead claim).

### Regeneration

6. Run `sase memory init`. With change 4 in place, `AGENTS.md`, `CLAUDE.md`, the
   provider shims, and the memory README must come back **unchanged**. A diff here means
   change 4 is wrong — fix it rather than committing the churn.

### Tests

7. `tests/test_linked_repo_sidecar_defaults.py::test_managed_project_injects_default_plans_and_beads_sidecars`
   — expected pairs become `[("sase--plans", True), ("sase--beads", False)]`.
8. `tests/main/test_repo_init_handler.py` — the exact-text assertion at ~line 42 loses
   the `auto_clone: true` line under `beads:`.
9. New regression coverage, at least:
   - `ensure_beads_sidecar_clone` passes the primary sidecar clone as `reference_repo`,
     and passes `None` when the workspace _is_ the primary (change 0);
   - the managed-project default for `beads` is lazy while `plans` stays eager
     (resolution and `collect_repo_inventory`);
   - `prepare_linked_repo_workspaces_if_needed` does not materialize a lazy beads
     sidecar (extend `tests/test_run_agent_runner_setup_linked_repos.py`);
   - `sase memory init` omits a lazy `beads` sidecar from the generated repositories
     list and does not require a description for it (peer of
     `tests/test_linked_repo_sidecar_hidden_agents.py`).
10. Re-run the existing `auto_clone` suites and fix fallout:
    `tests/test_repo_inventory.py`, `tests/test_linked_repo_sidecar_config.py`,
    `tests/test_config_schema_repositories.py`,
    `tests/doctor/test_checks_config_repos.py`,
    `tests/main/test_init_memory_handler_repositories.py`,
    `tests/test_ci_bootstrap_sidecars_tool.py`.

### Docs

11. Update the places that state beads is auto-cloned:
    - `docs/configuration.md` — the managed-project paragraph (~line 2028) and the
      `repos` example (~line 2053) plus the `auto_clone`/`auto_sync` discussion (~line
      1965), which currently implies beads is eager.
    - `docs/workspace.md:359` — "Sidecars configured with `auto_clone: true` (normally
      `plans`)" is already nearly right; make it accurate.
    - `docs/sdd.md:43`, `:595`, `:657-658`.
    - `docs/sdd_storage.md:101`, `:108`, `:133`.
    - `docs/beads.md:867`, `:872`.
    - `docs/init.md:369-371` (generated config sample).
    - Leave `docs/blog/posts/beads-and-sdd.md` alone; blog posts are dated artifacts.
12. `src/sase/sdd/assets/beads-directory-map.png.prompt.md` — update the alt text and
    the `Clone behavior` / subtitle labels from `AUTO-CLONE` / `auto_clone: true` /
    `EVERY WORKSPACE` to lazy wording. Mirror the phrasing already used in
    `research-directory-map.png.prompt.md` (`ON DEMAND`, `LAZY`, `auto_clone: false`).

    Do **not** attempt to re-render `beads-directory-map.png`: the text-free generated
    base is not stored in the repo and the pipeline needs an image-generation tool.
    Instead, file a task bead through `/sase_new_task` to re-render it, and note in the
    prompt file that the committed PNG is one revision behind its spec.

There is no manual `CHANGELOG.md` edit — the changelog is release-please generated from
the conventional commit message.

## Verification

- `just check` must pass. Because `_lint-flags` shells `tools/sase_bead`, a passing
  `just check` in this workspace is itself evidence that on-demand materialization
  works.
- Prove the lazy path end to end in a numbered workspace:

  ```bash
  # from the workspace checkout, with a clean, non-ahead beads clone
  git -C sase/repos/beads status --porcelain=v1 -b        # expect clean
  git -C sase/repos/beads rev-list --left-right --count origin/main...HEAD  # expect 0 0
  mv sase/repos/beads /tmp/beads-backup
  time sase bead list --status open                       # must succeed, ~2s not ~50s
  test -d sase/repos/beads                                # must exist again
  rm -rf /tmp/beads-backup
  ```

  If the clone is dirty or ahead of origin, do not move it — push or pick a different
  workspace. The timing here is the acceptance criterion for change 0; this exact
  sequence measured 49 s before the fix.

- `sase memory init` produces no diff (see change 6).
- `sase repo list` still shows the `beads` sidecar, now with `auto_clone` reported as
  `no`.
- Hand `just check-full` to `/sase_monitor` with the `TESTING` / `TESTED` status pair
  before landing; this change touches the linked-repo/config broadening set.

## Out Of Scope

- Do not change the `plans` sidecar. It is cloned unconditionally by
  `ensure_workspace_sdd_clone` regardless of `auto_clone`; touching its flag changes
  nothing useful and risks the SDD store home.
- Do not change `sase-core`'s `auto_clone: true` in `sase/sase.yml`; the workspace
  builds the native extension from that checkout.
- Do not try to make `_lint-flags` avoid the bead store. If the residual cost matters,
  that is separate work — file a task bead rather than widening this one.
- Do not widen change 0 to the generic `ensure_sdd_kind_clone` branch
  (`_store_workspace.py:134`), which has the same missing-reference gap for `research`
  and other document roles. File a task bead for it instead; beads is the role this plan
  is accountable for.
- Do not try to shrink the resulting pack. `--reference-if-able … --dissociate` leaves a
  164 MiB pack where a server-optimized clone leaves 40 MiB, so a post-clone repack
  would save ~124 MiB per materialized workspace at the cost of CPU on the critical
  path. That is a real, separate optimization — file a task bead, do not fold it in.
- Do not migrate existing checkouts' `sase/sase.yml`. Projects that already wrote
  `auto_clone: true` for beads keep the eager behavior until they re-run
  `sase repo init` or edit it themselves; that is the intended, non-breaking rollout.
