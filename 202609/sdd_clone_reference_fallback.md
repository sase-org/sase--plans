---
tier: tale
title: Fall back to a referenceless SDD sidecar clone on any reference-assisted failure
goal: A transient failure in a reference-assisted SDD sidecar clone retries without
  the local reference instead of failing the agent launch.
size: small
proposed_by: bbugyi200.athena.0iq
status: done
---

# Fall Back To A Referenceless SDD Sidecar Clone On Any Reference-Assisted Failure

## Background: the `0io` agent launch failure

Agent `0io` (project `gh_bobs-org__bob-cli`, workflow `ace(run)-260910_114657`,
2026-09-10 11:46:57 -0400) FAILED during launch, before the provider ever started.
`prepare_linked_repo_workspaces_if_needed` was materializing the `beads` sidecar for a
numbered workspace and the clone died at checkout:

```
failed to clone SDD store git@github.com:bobs-org/bob-cli--beads.git into <workspace>/sase/repos/beads:
fatal: unable to parse commit 8c09ae950bbf8201016911d1c9c90726b426e11b
warning: Clone succeeded, but checkout failed.
```

Diagnosis established:

- For numbered workspaces, `_materialize_remote_identified_sidecar`
  (`src/sase/_linked_repo_workspaces.py`) passes the primary sidecar clone as
  `reference_repo`, so `clone_sdd_store` (`src/sase/sdd/_store_clone_ops.py`) runs
  `git clone --reference-if-able <primary> --dissociate <remote> <target>`.
- The "unparseable" commit `8c09ae95` is the primary beads clone's _current HEAD_ and
  parses fine there now (`git cat-file -t` → `commit`). The primary was not durably
  corrupt; the failure was transient.
- The primary beads store is written to concurrently by every bob-cli agent (the
  `bob-cli-1y` epic agents were committing beads at that exact time; the primary had
  been fast-forwarded and auto-gc'd at 11:34, and has cruft packs). `git-clone(1)`
  explicitly warns that borrowing objects from a reference repository races with
  gc/prune activity in that repository; `--dissociate` copies borrowed objects at the
  end of the clone and is not atomic against the reference mutating mid-clone.
- `clone_sdd_store` already knows how to recover by retrying _without_ the reference,
  but only on `SddGitCommandTimeout`. A non-zero clone exit whose stderr does not match
  `_TRANSIENT_REMOTE_CLONE_ERRORS` (this checkout failure does not) hard-fails via
  `handle_failed_sdd_clone(strict=True)`, which raises `SddMaterializationError` and
  kills the agent launch — even though a plain referenceless clone from the healthy
  GitHub remote would have succeeded.

Root cause, in one sentence: a reference-assisted sidecar clone treats every non-timeout
failure as fatal, so any transient inconsistency in the local reference repo (a pure
transfer optimization) fails the whole agent launch.

## Change: generalize the referenceless-retry fallback in `clone_sdd_store`

Edit `src/sase/sdd/_store_clone_ops.py` only inside `clone_sdd_store` and its module
constants:

1. Rename `_MAX_TIMEOUT_RETRIES_WITHOUT_REFERENCE` to `_MAX_RETRIES_WITHOUT_REFERENCE`
   (value stays `1`) and rename the local counter `timeout_retries_without_reference` to
   `retries_without_reference`. The existing timeout branch keeps using them exactly as
   today.
2. In the failed-attempt branch (`result.returncode != 0`), _before_ the existing
   transient/hard-fail decision, add the same fallback the timeout branch has: when
   `reference is not None`,
   `retries_without_reference < _MAX_RETRIES_WITHOUT_REFERENCE`, and
   `attempt < len(_REMOTE_CLONE_RETRY_DELAYS)`, then:
   - `_remove_partial_sdd_clone(workspace_sdd)`,
   - increment `retries_without_reference`,
   - log a warning naming the remote, target, reference, and the failure `detail`
     (mirror the existing timeout warning's shape: "Failed cloning SDD store %s into %s
     with local object reference %s; retrying without the reference: %s"),
   - set `reference = None`, rebuild `clone_args` via `_remote_clone_args`, and
     `continue` immediately (no sleep, exactly like the timeout fallback).
3. Only when that fallback is not available does the existing logic run unchanged:
   non-transient detail or exhausted attempts → `handle_failed_sdd_clone`; transient
   detail → sleep per `_REMOTE_CLONE_RETRY_DELAYS` and retry.

Intended policy after the change, stated plainly: the first failed attempt of any kind
(timeout or non-zero exit, transient-looking or not) that used a local reference drops
the reference and retries immediately; at most one such referenceless fallback happens
per `clone_sdd_store` call; all subsequent behavior (transient retry budget, hard
failure, strict semantics, partial-clone cleanup) is unchanged. Referenceless clones
(`reference is None` from the start) behave exactly as today.

Do not change `_matching_clone_reference`, `_remote_clone_args`,
`handle_failed_sdd_clone`, `clone_sdd_store_from_primary`, or any caller.

## Tests

Add to `tests/sdd_store/test_sidecar_clone.py`, following the style of
`test_sidecar_clone_timeout_retries_without_reference_and_cleans_partial`:

1. `test_sidecar_clone_checkout_failure_retries_without_reference`: first `run_sdd_git`
   call returns `returncode=128` with stderr containing
   `fatal: unable to parse commit 8c09ae950bbf8201016911d1c9c90726b426e11b` and
   `warning: Clone succeeded, but checkout failed.` while leaving a partial file in the
   target; second call asserts the partial file is gone and succeeds. Assert the first
   recorded git argv contains `--reference-if-able` and the second does not, the
   function returns `True` under `strict=True`, and no sleeps were recorded (monkeypatch
   `sase.sdd._store_clone_ops.time.sleep` like the transient test does).
2. `test_sidecar_clone_reference_fallback_is_capped`: every call fails with the same
   non-transient `returncode=128` stderr; assert exactly two git invocations happen (one
   with the reference, one without), `SddMaterializationError` surfaces the diagnostic
   under `strict=True`, and the target directory is removed.

Confirm the existing timeout tests still pass unmodified apart from the constant rename
they reference only through behavior (they monkeypatch functions, not the constant, so
no test edits are expected beyond the two additions).

## Verification

Run the repo's standard verification (`just check`) and make sure
`tests/sdd_store/test_sidecar_clone.py` and `tests/test_linked_repo_workspaces.py` pass.

## Out of scope

- No operational repair is needed: the primary bob-cli beads clone is healthy, the
  failed run's partial clone was already cleaned up by `handle_failed_sdd_clone`, and
  the held workspace is released by dismissing the failed `0io` agent in `sase ace`
  (user action, not code).
- Hardening `pull_sdd_clone` / existing-clone refresh paths is untouched; only the fresh
  reference-assisted clone path failed here.
