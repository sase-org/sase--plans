---
tier: tale
title: Fix false stranded verdicts from a stale primary beads clone
goal:
  A bead closed from an agent workspace reaches the primary beads clone within one
  auto-sync tick, so ACE never flags its agent as stranded with close_failed.
size: medium
decisions:
  managed_auto_sync_default:
    ask:
      Should a managed project's declared builtin plans or beads sidecar that omits
      auto_sync default to auto_sync true?
    choices:
      inherit:
        Managed builtin plans/beads omitting auto_sync get true, like injected defaults;
        false still wins
      bob_cli_only:
        Keep the false default; set auto_sync true on bob-cli's plans and beads in its
        sase.yml
      neither:
        No backstop for bob-cli; bead edits from other machines stay unseen there until
        a manual pull
    default: inherit
    why: Every other path that creates these entries already enables auto_sync
    answer: inherit
proposed_by: bbugyi200.apollo.au
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-28 07:43:30
status: wip
---

# Fix False "Stranded" Verdicts From A Stale Primary Beads Clone

## Problem

ACE flagged the `bob-cli-db.land` agent as `⚐ STRANDED` with the cause "ended with
bead_action close; the bead is still open", even though bead `bob-cli-db` was closed at
2026-10-28T10:47:55Z, about four minutes before the agent ended at 10:52:11Z. The same
agent card's ARTIFACTS section showed the bead as `CLOSED`. That section reads another
source, so the two surfaces contradicted each other.

## Diagnosis (verified)

The Rust reducer (`sase_core::stranded_work::classify`) is correct. It was given a wrong
fact: the bead's status was `in_progress` when the bead was actually closed.

1. **The collector reads the primary checkout's beads clone.**
   `sase.bead.stranded_facts.collect_stranded_work_input` loads candidates from
   `canonical_beads_dir_for_project(project)`, the primary checkout's `sase/repos/beads`
   clone. It considers only `in_progress` beads, plus `open` beads that carry a
   completion failure. The newest agent member declared `close`, so the reducer returned
   `close_failed`.
2. **bob-cli's primary beads clone was 13.5 hours stale.** Its reflog shows no movement
   between 2026-10-27T22:01:38Z and 2026-10-28T11:28:59Z. At 22:01Z on Oct 27,
   `bob-cli-db` was still `in_progress` (preclaimed at 21:11:44Z), so the collector
   never saw the first close (22:54:53Z) or the second one (10:47:55Z). The pull at
   11:28:59Z came from a diagnostic `sase bead read` run in the primary checkout,
   through the post-command background refresh in `sase.main.entry`. Once that ran,
   `load_stranded_report("gh_bobs-org__bob-cli")` no longer listed `bob-cli-db`.
3. **Why nothing converged the clone:**
   - **(a) No auto_sync opt-in.** bob-cli's `sase/sase.yml` declares
     `repos.sidecar.builtin.beads: {auto_clone: true}` and does not set `auto_sync`.
     `_merged_sidecar_entries_cached` in `src/sase/_linked_repo_config_entries.py` then
     defaults it to `auto_sync: False`. The injected managed-project defaults
     (`_linked_repo_config_defaults.inject_default_linked_repos`) and `sase repo init`
     (`main/_repo_init_config.py`) both use `auto_sync: true` for plans/beads. So
     declaring the entry just to set `auto_clone` silently opted bob-cli out. The
     `sidecar_auto_sync` chop log confirms it: `targets=7`, which is the job, tailnet,
     and sase roles, and none for bob-cli.
   - **(b) No hint after a bead publication.** Nothing marks a sync hint when a bead
     mutation (`sase bead close`, a finalizer close) is published from a workspace
     clone. Hints are written only by claim and recovery paths.
     `docs/contribute/workspace-provider.md` says a hint is "recorded right after a
     workspace-sidecar publication", but the code doesn't do this for ordinary bead
     mutations.
   - **(c) The chop ignores hints for roles that haven't opted in.** A pending hint only
     matters for roles that opted into `auto_sync`. The only exception is the
     live-bead-waiter carve-out in `_targets_for_project`.
   - **(d) Launch-claim hints use the wrong key.** `_schedule_launch_claim_convergence`
     in `src/sase/axe/run_agent_runner_bead.py` writes the hint under
     `marker.project_name or marker.project_key`. For bob-cli that is the basename label
     `bob-cli`, but the chop reads hints by the directory key
     (`ProjectRecordWire.project_name`, i.e. `gh_bobs-org__bob-cli`). Stale
     `~/.sase/sidecar_sync_hints/bob-cli.json` (written 2026-10-28 10:28) and
     `sase.json` (2026-10-19) are never consumed.

So for a project without the beads opt-in, the stranded detector can report a false
negative claim ("the bead is still open") indefinitely. For an opted-in project it can
still flash a false verdict for up to about 3.5 minutes after every land: the
five-minute backstop is longer than the reducer's 120-second settle grace.

## Changes

### 1. Hint the beads role after every verified bead publication

In `src/sase/bead/cli_common_publication.py`, `ensure_bead_mutation_published` should
mark a beads-role sync hint for the store's owning project key once publication is
proven. That covers both success paths: already published, and published by the forced
synchronous push. Do not hint when it raises `BeadPublicationError` or when verification
is skipped because of an internal error.

- Add an optional keyword `project_key: str | None = None`. Pass it from callers that
  already know the key:
  - `bead/cli_common._require_published_bead_mutation`, from the routed
    `bead_context.project_key`
  - `main/bead_fast_path.py`, from its context
  - `finalizers/reconciliation_bead_store._unpublished_bead_state_error`, if a key is in
    reach
  - `sdd/_artifact_link_commit.py`, if a key is in reach
- When no key is supplied, resolve it best-effort from the workspace marker that owns
  `beads_dir` (`sase.workspace_provider.marker.find_marker_from_cwd`, using
  `marker.project_key`). Skip silently when it can't be resolved.
- Route the write through
  `sase.bead.background_store.schedule_beads_sidecar_convergence`. Respect
  `bead_refresh_mode() == "off"` the same way `mark_bead_wait_sync_hint` does. Hint
  failures must never fail the mutation.
- Use lazy imports inside the function, matching the module's style.

### 2. Write and read hints under the canonical project key

- In `src/sase/axe/run_agent_runner_bead.py`, `_schedule_launch_claim_convergence` uses
  `marker.project_key or marker.project_name`.
- Normalize in one place as defense in depth. `schedule_beads_sidecar_convergence` (or
  `mark_sidecar_sync_hint`) resolves a key, alias, or label to the canonical directory
  key with
  `sase.project_display_names.load_project_ref_display_snapshot().project_key_for_ref(ref)`.
  It falls back to the given string when the ref is unknown or ambiguous. Keep this
  best-effort and exception-safe.
- Leave the existing label-keyed hint files alone. They are inert, and later hints are
  written under the correct key.

### 3. A pending beads hint always converges the primary beads clone

In `src/sase/scripts/sase_chop_sidecar_auto_sync.py`, `_targets_for_project` widens the
existing live-waiter carve-out: when `bead_refresh_mode() != "off"` and the project has
a pending `beads` hint, the beads role becomes a target with `hinted=True` and
`require_auto_sync_opt_in=False`, even if the role has not opted into `auto_sync`.

- Read the pending hints before the early `return []`.
- Don't add a duplicate target when beads is already an opted-in role or already forced
  by a live waiter.
- The sync is still the same conservative fetch/fast-forward:
  `sync_primary_sidecar_role` refuses dirty, detached, diverged, or mismatched clones.
- Update the module docstring to describe both carve-outs: a live bead waiter, and a
  hint left by a verified bead publication or claim.

Changes 1 and 3 together mean a closed bead reaches the primary within one 30-second
chop tick. That is inside the reducer's 120-second settle grace, so `close_failed`
cannot fire while the close is still propagating.

### 4. Managed auto_sync default (decision `managed_auto_sync_default`)

> [!decision] managed_auto_sync_default = inherit In
> `src/sase/_linked_repo_config_entries.py`, when a managed project
> (`is_sase_managed: true` in the project-local config) declares the builtin `plans` or
> `beads` sidecar and omits `auto_sync`, that entry inherits `auto_sync: true`, the same
> value `inject_default_linked_repos` gives it. Custom roles (such as `research`), the
> hidden roles (`agents`, `attachments`, `attachments-private`), non-managed projects,
> and any explicit `auto_sync: false` keep their current behavior. Apply the default at
> the single normalization point every reader shares (`auto_sync_roles`,
> `sync_primary_sidecar_role`, `sase repo inventory`) so the inventory display and the
> chop agree. If the normalization cache key doesn't include the managed flag, make sure
> the cache can't serve a stale default. Update the `auto_sync` description in
> `src/sase/config/sase.schema.json` (the builtin and custom entries) to state the
> managed builtin default, and regenerate the config reference with `just fix`.

> [!decision] managed_auto_sync_default = bob_cli_only Leave the code default unchanged.
> Open the bob-cli project with `sase repo open bob-cli -r "<why>"`, and in its
> `sase/sase.yml` add `auto_sync: true` under `repos.sidecar.builtin.plans` and
> `repos.sidecar.builtin.beads`. Commit it through the final declaration's repository
> obligation.

> [!decision] managed_auto_sync_default = neither Skip this section. Changes 1–3 alone
> fix the reported false verdict for bead changes published on this host.

### 5. Docs and config text

- `docs/reference/automation/scheduler-default-routines.md`: in the bead-wait paragraph,
  say that a pending beads hint (written after a verified bead publication or a claim)
  converges the beads role even without the `auto_sync` opt-in, and that
  `sdd.bead_refresh.mode: off` disables it.
- `docs/contribute/workspace-provider.md`: the auto-sync bullet should name the
  verified-publication hint as the trigger, and say that hints are keyed by the project
  directory key.
- `docs/reference/config/repositories-behavior.md`: for the `inherit` choice, state the
  managed builtin plans/beads default.
- `src/sase/default_config.yml`: update the `sidecar_auto_sync` job description (around
  the `sidecar_sync` routine) to mention the hint carve-out. Regenerate any generated
  copy (`docs/reference/config/scheduler-behavior.md`) with `just fix`.
- Follow the docs routing in the `docs_writing` memory note. Write no new pages.

## Tests

Add regression tests next to the existing suites:

- `tests/test_axe_chop_sidecar_auto_sync_targets.py`:
  - A project with no `auto_sync` roles and a pending `beads` hint yields exactly one
    beads target, with `hinted=True` and `require_auto_sync_opt_in=False`.
  - With `bead_refresh_mode() == "off"` it yields none.
  - A project with an opted-in beads role and a hint still yields one target, not two.
- Hint keying:
  - `_schedule_launch_claim_convergence` hints the marker's `project_key`
    (`gh_<org>__<repo>`), not the basename label.
  - Normalization maps a known label or alias to its key and passes unknown refs through
    unchanged. Use `tests/test_sidecar_sync_hints.py` or
    `tests/test_bead/test_background_store.py`.
- Publication hint (new test beside the existing `ensure_bead_mutation_published`
  coverage):
  - The hint is marked for the supplied or marker-resolved project key on the
    already-published path and on the pushed-now path.
  - No hint when `BeadPublicationError` is raised.
  - No hint when the refresh mode is `off`.
- For `inherit`, in `tests/test_linked_repo_sidecar_config.py`:
  - A managed primary (project-local `is_sase_managed: true`) declaring
    `builtin.beads: {auto_clone: true}` and `builtin.plans: {auto_clone: true}`
    normalizes to `auto_sync: true`.
  - An explicit `auto_sync: false` stays false.
  - A managed custom `research` without the key stays false.
  - The existing non-managed
    `test_auto_sync_defaults_false_and_is_independent_of_auto_clone` still passes
    unchanged.

## Non-goals

- **No sase-core reducer or wire change.** The reducer classifies correctly when given
  fresh facts. A store-freshness input would mean a cross-repo wire bump, and it is
  unnecessary once publication hints converge the primary within the settle grace.
- **No ACE or presentation change**, and no change to the stranded collector's choice of
  the primary clone as its canonical read store.
- **No cleanup of the legacy label-keyed hint files** under
  `~/.sase/sidecar_sync_hints/`.

## Verification

- Run `just fix`, then `sase tool run check` (use `SASE_TOOL_BYPASS='<why>' just check`
  only if `sase tool` is unavailable). Do not run `just check-full`.
- Manual sanity check, which is optional and read-only:
  `load_stranded_report("gh_bobs-org__bob-cli")` must not list `bob-cli-db`.
