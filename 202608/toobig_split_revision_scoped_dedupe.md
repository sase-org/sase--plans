---
tier: tale
size: medium
title: Scope toobig_split dedupe keys to the target repository's HEAD commit
goal:
  The toobig_split chop marks a file as a duplicate only while the repository the file
  lives in is unchanged, measured by that repo's HEAD commit, so any new commit re-opens
  every still-oversized file for proposal and the chop stops being permanently inert.
proposed_by: bbugyi200.athena.0bj
---

- **AGENTS:**
  - [bbugyi200.athena.0bj](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0bj.md)
  - [bbugyi200.athena.toobig-3l.split_file.tests.ace.tui.test_config_hub_pane.0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.toobig-3l.split_file.tests.ace.tui.test_config_hub_pane.0/README.md)
  - [bbugyi200.athena.toobig-3l.split_file.tests.ace.tui.test_statistics_pane_interactions.0](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.toobig-3l.split_file.tests.ace.tui.test_statistics_pane_interactions.0.md)
  - [bbugyi200.athena.toobig-3l.split_file.tests.monitor.test_monitor_supervise.0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.toobig-3l.split_file.tests.monitor.test_monitor_supervise.0/README.md)
- **COMMITS:**
  - [e2056bd](https://github.com/sase-org/sase/commit/e2056bddebf0898dbf67f8aa8420a057dab42712)
    — docs(axe): warn that content-only once_per keys stay reserved
  - [f8ce6bb](https://github.com/sase-org/sase/commit/f8ce6bb2253eaed83700d77ea79367e9a3f12a98)
    — test(ace): split config hub pane tests
  - [062ae22](https://github.com/sase-org/sase/commit/062ae22c214e6b03683fb3933bbd03cf7788d130)
    — test(ace): split statistics pane interaction tests under 500 lines
  - [d46236d](https://github.com/sase-org/sase/commit/d46236d0f27c401320e617b9b9015ccd83826806)
    — test(monitor): split test_monitor_supervise under 500 lines

# Plan: Scope toobig_split dedupe keys to the target repository's HEAD commit

## Problem

The hourly `run_every / toobig_split[sase]` chop finds oversized files but never
launches a split agent. It has been silently inert for weeks.

### Observed evidence

Run `20260823T114517_800605` under
`~/.sase/axe/lumberjacks/run_every/chops/toobig_split[sase]/runs/`:

- `status: "skipped"`, `reason: "all 12 proposal(s) skipped by once-per dedupe"`.
- All 12 proposals carry `"validation": "duplicate"` with
  `dedupe_reason: "once-per key \`toobig_split:gh:sase-org/sase:<path>:<digest>\` was
  already seen"`.
- Every key was recorded at `2026-08-22T19:23:05Z` in that chop's `seen.json` (410
  entries total).
- Between that timestamp and the run, **12 commits landed on `sase` master** (HEAD
  `afe374f93`), yet not one file was re-proposed.

Sampling five of the twelve files confirmed the mechanism: the current
`sha256(file)[:16]` of each file still equals the digest embedded in its recorded key,
so each proposal hashes to a key the store has already seen.

This is _not_ the `wait_on` cascade bug fixed by the July epic
(`~/.sase/plans/202607/fix_toobig_split_chop_dedupe.md`). Each proposal is independently
rejected on its own key; the relink logic is working correctly.

### Root cause

Two correct-in-isolation behaviors combine into a permanent lockout:

1. **The chop derives its dedupe key from file content only.**
   `bugyi_chops/toobig_split.py::_dedupe_key` builds
   `toobig_split:{workspace}:{path}:{sha256(file_bytes)[:16]}`, falling back to the
   literal `missing` when the file cannot be read.

2. **AXE retains accepted keys for launches that did not terminally fail.**
   `src/sase/axe/chop_policy.py::apply_chop_once_per` records each accepted key in the
   chop's `seen.json`. Keys are released only when a proposal never starts or its agent
   reaches terminal failure (`src/sase/axe/chop_lifecycle.py`). A split agent that
   completes successfully — including one that examines a file and deliberately leaves
   it byte-identical — keeps its key forever.

So any file a split agent has already examined without editing is deduped permanently,
no matter how much the rest of the repository changes around it. Files hovering just
over the soft limits (`700`/`850`) are exactly the ones an agent is most likely to
decline to split, which is why the pool of permanently-suppressed files only grows.

### Intended semantics after this plan

Dedupe holds only while the codebase the file lives in is unchanged, measured by that
repository's most recent commit. Any new commit to the target repo re-opens every
still-oversized file for proposal.

This is a deliberate change of intent. The July epic explicitly wanted the opposite ("an
agent that completes successfully but leaves the file unchanged (a deliberate no-op)
keeps its key, preventing hourly churn on files an agent already examined"). That
guarantee is what produced today's inert chop, and this plan supersedes it. See
**Risks** for the churn trade-off this accepts.

## Design

### Where the fix belongs

The `dedupe_key` is authored by the chop script and is opaque to the AXE runner — it is
the runner's designated extension point for expressing "what work identity means" for a
given chop. Scoping identity to a repository revision is therefore a change to the chop
script, not to the AXE once-per engine.

Concretely: **no change to the AXE runner, the Rust chop engine, or the chop's
configuration.** In particular:

- `~/.config/sase/sase_athena.yml` (chezmoi-managed) needs no edit. The `toobig_split`
  chop declares **no `once_per` block at all**; its dedupe comes entirely from
  per-proposal `dedupe_key` values, which `apply_chop_once_per` honors on their own.
- The Rust core backend boundary is respected: we are not changing how dedupe decides,
  only what string the script hands it.

The behavior change lands in the `bugyi-chops` repository. The `sase` repository gets a
documentation fix so the trap that caused this is written down.

### Repository access

`bugyi-chops` is not a linked repo of this project. Open it as an external repo and use
only the printed path for reads and writes:

```bash
sase repo open gh:bbugyi200/bugyi-chops -r "Scope toobig_split dedupe keys to repo HEAD"
```

### Change 1 — revision-scoped dedupe key (`bugyi-chops`)

In `src/bugyi_chops/toobig_split.py`:

- Add `_repo_revision(repo_root: Path) -> str | None` that runs
  `git -C <repo_root> rev-parse HEAD` (reuse the module's existing `_run_command`
  subprocess helper) and returns the stripped full 40-character SHA. Return `None` when
  git exits non-zero, is missing, or prints nothing — for example when `repo_root` is
  not a git work tree.

- Resolve the revision **once per scan target**, before the proposal loop, not once per
  file. One `git` call per run.

- Change `_dedupe_key` to take the resolved revision instead of reading file bytes:

  ```
  toobig_split:{workspace}:{path}:{revision}
  ```

  Drop the content digest and its `missing` sentinel entirely. HEAD subsumes the content
  digest for the tree being scanned (the project's primary workspace, normally clean),
  and the user-requested rule is repo-scoped, not file-scoped.

- **Fail open when the revision is unknown.** If `_repo_revision` returns `None`, pass
  `dedupe_key=None` for every proposal in that target rather than inventing a key. This
  is safe and already supported end to end: `launch_proposal` drops `None` optional
  fields, and `apply_chop_once_per` accepts any proposal whose key resolves to `None`.
  Failing open is the correct direction here — the failure this plan fixes was a
  fail-_closed_ silence, and an un-deduped run is still bounded by the chop's
  `inhibit_if` clan guard.

`_path_digest` (used for `proposal_id`) is unrelated and must stay as it is.

### Change 2 — tests (`bugyi-chops`)

In `tests/test_toobig_split.py`:

- `_prepare_repo` (around line 124) creates a plain directory tree, **not** a git repo,
  so revision resolution will return `None` there. Give it a git-initialized variant (or
  `git init` + one commit inside `_prepare_repo`, with a deterministic author identity
  via `-c user.email=... -c user.name=...` or the `GIT_*` env vars) so tests that assert
  on dedupe keys get a real HEAD.
- Line ~344, `assert len({proposal["dedupe_key"] for proposal in proposals}) == 3`,
  still holds — distinct paths still produce distinct keys — but only once the fixture
  is a git repo. Confirm rather than assume.
- Line ~756, `assert proposals[1]["dedupe_key"].endswith(":missing")` in
  `test_absolute_scanner_paths_are_normalized_and_missing_files_still_dedupe`, asserts
  the removed sentinel and must be rewritten. A missing file now dedupes on the repo
  revision like any other path; rename the test if its name no longer describes it.
- Add coverage for the new behavior:
  - every proposal's `dedupe_key` ends with the repository's current HEAD SHA;
  - creating a new commit changes every key (same files, different keys);
  - a `repo_root` that is not a git work tree yields proposals with **no** `dedupe_key`
    field at all (fail-open), while proposals are otherwise unchanged.

### Change 3 — document the dedupe-durability trap (`sase`)

Purely documentation; no code change in this repo.

- `docs/axe.md`, the `once_per` bullet (around line 908): the text already says accepted
  keys "remain reserved for successful launches". Extend it to state the consequence — a
  key derived only from file content therefore suppresses that work permanently once an
  agent succeeds without changing the file — and recommend that chops which should retry
  as the codebase advances include the target repository's revision in the key.
- `docs/configuration.md` (around line 2695): add the same one-sentence caution next to
  the existing `once_per` / `dedupe_key` precedence note.

## Rollout

Ordering matters here, because the two repositories reach the running host by different
paths. The sase uv tool env (`~/.local/share/uv/tools/sase/`) installs `sase` as an
**editable** checkout but installs `bugyi-chops` **from its GitHub default branch** (per
`~/.local/share/uv/tools/sase/uv-receipt.toml`). Doc changes in this repo are live
immediately; the chop fix is not live until the tool env is re-synced.

1. Land the `bugyi-chops` change on its default branch (follow that repo's existing
   commit/release convention; the host install tracks the default branch and pins no
   version, so no version bump is strictly required to roll out).
2. Re-sync the sase tool environment so the new `bugyi_chops` lands (`sase update`, or a
   targeted `uv tool install --reinstall-package bugyi-chops ...` consistent with the
   receipt). Verify with the tool env's own interpreter, not the workspace venv:

   ```bash
   ~/.local/share/uv/tools/sase/bin/python -c \
     "import inspect, bugyi_chops.toobig_split as m; print(inspect.getsource(m._dedupe_key))"
   ```

3. Only after the new script is installed, clear the stale dedupe store:
   `~/.sase/axe/lumberjacks/run_every/chops/toobig_split[sase]/seen.json`. Its 410
   entries are all old-shape content-digest keys that can never match a new-shape key
   again, so they are pure ballast against the bounded `capacity: 1024` log. Removing
   the file is safe — the engine treats a missing seen document as empty, and clearing
   it can only re-allow launches. Do this while no `toobig-` clan is active, and leave
   the run history under `runs/` intact.

## Testing

- `bugyi-chops` (in the opened external checkout): `just install` first — its justfile
  drives `uv sync --group dev` — then `just check`, which runs `lint` (ruff format
  check, ruff check, mypy), `test` (pytest), and `build`.
- `sase` (this repo): run `just install`, then `just check`. The change here is
  documentation-only, but the repo rule requires the gate for any file change. Hand it
  to `/sase_monitor` with `-s TESTING -S TESTED` if it runs long.

### Live verification on the host

After the rollout steps, run the chop manually and confirm:

- `sase axe chop run` for `toobig_split[sase]` reports accepted proposals rather than
  `all N proposal(s) skipped by once-per dedupe`, and the recorded proposals'
  `dedupe_key` values end in the current `sase` HEAD SHA.
- A `toobig-N` clan launches split agents, chained by `wait_on`.
- An immediate second run is skipped by the guard, naming the active `toobig-` clan
  (`inhibit_if: agent_clan {name_prefix: toobig-}`) — proving the clan guard, not
  dedupe, is what serializes swarms.
- `sase axe chop doctor` stays OK.

Record the observed run IDs and statuses in the completion notes.

## Risks

- **Hourly churn on files agents decline to split.** This is the explicit trade-off the
  requested semantics buy. Once any commit lands, every still-oversized file is
  re-opened, including ones an agent already examined and deliberately left alone.
  Bounded by `run_every: 60m`, the `toobig-` clan `inhibit_if` guard (one swarm at a
  time), and the sequential `wait_on` chain. If churn proves excessive in practice, the
  follow-up knob is a cooldown — suppress a file whose content is unchanged and whose
  last agent succeeded within N hours or N commits — which is deliberately **out of
  scope** here.
- **Dedupe becomes nearly inert on a busy repository.** Split agents commit to `sase`
  themselves, so HEAD moves during a swarm and keys recorded minutes earlier stop
  matching. After this change dedupe only suppresses re-proposal within a single
  commit-free hour. Accepted: the clan guard is the real concurrency control, and
  dedupe's remaining job is just to avoid pointless identical rescans of a frozen tree.
- **Fail-open means no dedupe at all when git is unavailable for a target.** Strictly
  better than today's fail-closed silence, and still guarded by `inhibit_if`, but worth
  knowing: a misconfigured `repo_root` degrades to "propose every scan" rather than
  erroring.
- **Split rollout window.** Between landing the docs change here and re-syncing the tool
  env, the host chop keeps deduping exactly as it does today. Nothing breaks; the fix
  just is not live yet.
- **Clearing `seen.json` discards retry bookkeeping for anything in flight.** Perform it
  with no active `toobig-` clan so no launched proposal loses its key mid-flight.
