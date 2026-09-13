---
tier: tale
title: Fix artifact_run_prune crash on hosts with more than 10,000 ACE run dirs
goal:
  The housekeeping artifact_run_prune chop plans retention successfully at the current
  11k+ ACE-run inventory, and a future over-cap input degrades to a blocked check_error
  instead of a crash loop.
size: medium
proposed_by: bbugyi200.athena.7f
create_time: 2026-09-13 17:26:46
status: wip
---

# Fix artifact_run_prune crash: continuation retention rejects >10,000 ACE run dirs

## Problem

The `housekeeping` lumberjack's `artifact_run_prune` chop crash-loops with exit code 1
on every run. Axe error digests (2026-09-13, e.g. runs `20260913T161415_949620`,
`20260913T162642_839195`, `20260913T165750_293755`) all end with:

```
File ".../src/sase/core/continuation_facade.py", line 146, in plan_continuation_retention
    binding(continuation_wire_to_json_dict(request)),
ValueError: validation: runs has 11037 entries; maximum is 10000
```

The entry count grows across failures (11037 → 11040 → 11044) because the prune job is
the only thing that shrinks the ACE-run inventory, and it can no longer run. This is a
self-locking failure: once a host accumulates more than 10,000 ACE run directories,
retention planning refuses its input forever and the inventory only grows.

## Root cause

`plan_ace_run_retention` (`src/sase/core/agent_artifact_run_retention.py`) collects
every ACE-run artifact directory across all projects and sends them in one
`plan_continuation_run_retention` wire request
(`src/sase/core/continuation_retention.py`), which calls the Rust binding
`continuation_plan_retention`.

In the sase-core repo, `crates/sase_core/src/continuation/retention.rs` validates that
request against `MAX_NODES` (10,000, defined in
`crates/sase_core/src/continuation/schema.rs`):

- line ~76: `if request.runs.len() > MAX_NODES` → hard validation error (the crash);
- line ~128: the ancestry-closure visit guard also uses `MAX_NODES`.

`MAX_NODES` is a sane bound for a _single continuation graph_ (node-record validation in
`schema.rs`, replay planning in `replay.rs`). But the retention request's `runs` list is
_every ACE-run directory on the host_ — a number that scales with disk usage and
retention backlog, not with any one continuation chain. Reusing the graph bound for the
retention input is the defect.

Both the preview path (the chop) and the apply path hit this: `apply_ace_run_retention`
re-runs the same closure via `continuation_protected_dirs` before deleting.

## Fix design

Two layers, per the Rust-core-backend-boundary rule (retention semantics are Rust-owned;
Python is the I/O adapter):

1. **sase-core (Rust): decouple the retention input bound from `MAX_NODES`.** Add a
   retention-specific cap large enough that it is only ever hit under pathological
   conditions (proposed: 100,000 — ~9x today's 11k inventory; each wire entry is a path
   plus a few short strings, so a full request stays modest in memory).
2. **sase (Python): make the preview degrade fail-safe instead of crash-looping.** If
   the continuation planner still rejects the input (any `ValueError` from the binding),
   record it as an unavailable protection source and protect every run, the same way a
   failed artifact scan already does (`scan_unavailable`). The chop then completes with
   a visible `check_error` instead of a traceback + crash loop, and `--apply` stays
   blocked by the existing `sources_unavailable` gate.

The direct apply path must stay **fail-closed**: do not catch the error inside
`apply_ace_run_retention` / `continuation_protected_dirs`. If planning fails there, the
exception must keep aborting apply before any deletion.

## Steps

### 1. sase-core: retention-specific run cap

Work in the linked `sase-core` repo (open it with `/sase_repo`; note: known bug
`sase-zo` can make `sase repo open sase-core` fail with an unknown-repo error even
though the repo is linked and cloned — the build-target checkout the sase `Justfile`
uses is the linked one at the workspace-relative path `sase/repos/linked/sase-core`, not
the external `gh:` clone).

In `crates/sase_core/src/continuation/retention.rs`:

- Add a module-local constant next to `MAX_PATH_BYTES`:
  `const MAX_RETENTION_RUNS: usize = 100_000;` with a comment stating that the retention
  request carries every ACE-run dir on the host, a scale unrelated to the single-graph
  `MAX_NODES` bound (cite the 2026-09-13 incident: hosts lock up at
  > 10k dirs because pruning is the only way the count shrinks).
- Use it in the request-size check (`request.runs.len() > MAX_RETENTION_RUNS`), keeping
  the same error-message shape (`runs has {} entries; maximum is ...`).
- Update the closure visit guard. Each dir is enqueued at most once (`enqueue_protected`
  dedupes via the reasons map), and enqueued dirs are drawn from run dirs plus their
  starter dirs, so unique visits are bounded by `2 * runs`. Guard with
  `visited > MAX_RETENTION_RUNS.saturating_mul(2)` (defensive loop bound only) and keep
  the existing error message.
- Do NOT change `MAX_NODES` or its other uses (`schema.rs` record validation,
  `replay.rs`) — those genuinely bound single-graph inputs.
- Drop the now-unused `MAX_NODES` import from `retention.rs` if nothing else in the
  module uses it.

Tests (same file's `mod tests`):

- A request with more than 10,000 runs (e.g. 10,001 minimal runs plus one `live` run
  whose parent is a distinct old run) succeeds and still protects the live run's
  ancestry — this is the direct regression for the incident.
- A request with `MAX_RETENTION_RUNS + 1` runs fails with the cap message.

Semver note: this is a `fix(continuation)` change — sase-core versioning is
release-plz-driven from conventional commits, so describe it as a fix so a patch release
(0.34.x) is cut. The sase pyproject window `sase-core-rs>=0.34.23,<0.35.0` already
admits that patch release; no pin change is required.

### 2. sase: fail-safe preview degradation

In `src/sase/core/agent_artifact_run_retention.py`, `plan_ace_run_retention`:

- Wrap the `plan_continuation_run_retention(all_dirs, projects_root=projects_root)` call
  in `try/except ValueError`. On failure:
  - add `f"continuation retention: {exc}"` to `unavailable`,
  - use empty `continuation_reasons`,
  - set a `continuation_unavailable` flag.
- Thread that flag into `_protection_reasons` (mirroring the existing `scan_unavailable`
  parameter) and append a `"continuation_unavailable"` reason for every dir when set.
  Every run is then protected, `selected` is empty, and the chop
  (`src/sase/scripts/sase_chop_artifact_run_prune.py`) reports `check_error` with reason
  `protection_unavailable` — no code change needed there.
- Do not add any exception handling to `apply_ace_run_retention` or to
  `continuation_retention.py` / `continuation_facade.py`: the apply-time closure must
  keep raising so apply aborts before deletions (fail-closed), and `prune-runs --apply`
  additionally stays blocked by `blocked = apply and bool(plan.sources_unavailable)`
  (`src/sase/artifact_cli/prune_runs.py`).

Tests in `tests/core/test_agent_artifact_run_retention.py` (monkeypatch patterns already
exist there):

- Monkeypatch `plan_continuation_run_retention` (as imported in
  `agent_artifact_run_retention`) to raise
  `ValueError("validation: runs has 11037 entries; maximum is 10000")`; assert the plan
  has that text in `sources_unavailable`, every candidate is protected with a
  `continuation_unavailable` reason, and `counts.selected == 0`.
- Assert `apply_ace_run_retention` still propagates the same `ValueError` (no
  swallowing) when the continuation closure raises, and removes nothing.

## Verification

- sase-core: `just check` in the sase-core repo (fmt + clippy + tests).
- sase: `just check` (the dev install builds `sase_core_rs` from the linked sase-core
  checkout, so the Python tests run against the updated core).
- End-to-end sanity (optional, read-only): run the preview chop entry point
  (`sase_chop_artifact_run_prune`) on this host — with 11k+ real run dirs it should now
  complete and report reclaimable runs instead of raising.

## Acceptance criteria

- `plan_continuation_retention` accepts a request with 11,044+ runs (current host
  inventory) and only rejects above 100,000.
- The `artifact_run_prune` housekeeping chop completes on this host; the axe crash-loop
  stops.
- If the (new) cap is ever exceeded, the preview chop finishes with `check_error` and an
  explanatory `sources_unavailable` entry instead of a traceback, and every apply path
  refuses to delete anything.
- `MAX_NODES` behavior for node-record validation and replay is unchanged.

## Out of scope

- The `sase repo open` display-name/canonical-key resolution bug is tracked separately
  as task bead `sase-zo` (corroborated with a +1 during this investigation).
- Any change to retention policy itself (keep windows, protections) or to artifact
  lifecycle commands.
