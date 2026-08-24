---
tier: tale
size: small
title: Prevent release metadata checks from timing out in the runner queue
goal:
  Keep self-healing sase-core release runs alive long enough for the post-build metadata
  check and PyPI publish to execute under ordinary runner contention.
proposed_by: bbugyi200.athena.0cl
---

# Prevent release metadata checks from timing out in the runner queue

## Diagnosis

`actstat --repo sase-org/sase-core --limit 50 --only-failures` separates the recent red
Actions history into three categories:

- Release-plz run 748 built every wheel and the sdist successfully, but its
  `twine check` job executed no steps and was cancelled exactly at the job's 15-minute
  timeout. The dependent PyPI publish was therefore skipped. This is the still-present
  defect: the lightweight post-build job has a shorter queue budget than the costly
  build jobs it gates, even though the workflow is explicitly designed to recover from
  runner starvation.
- Release-plz run 749 published successfully; only a pending `Release-plz PR` job was
  superseded by the next member of its deliberate concurrency group. Preserve that
  serialization rather than weakening release safety to hide an expected cancellation.
- Older CI runs failed on stale test expectations or Clippy diagnostics in intermediate
  commits. Later commits fixed those source issues, and the newest settled CI runs on
  master pass, so do not reopen or duplicate those fixes.

## Implementation

1. Update `.github/workflows/release-plz.yml` so the `metadata-check` / `twine check`
   job has a runner-contention budget consistent with the downstream release-critical
   path (prefer 60 minutes, matching the longest wheel job).
2. Add a concise workflow comment explaining that this timeout must cover queue
   starvation as well as the fast `twine` commands, so a future cleanup does not restore
   the misleading 15-minute limit.
3. Leave release-PR and merge concurrency, publish guards, build references, job
   dependencies, and release-plz-managed Cargo versions unchanged.

## Verification

1. Run `actionlint .github/workflows/release-plz.yml` to validate GitHub Actions syntax
   and expressions.
2. Run `just check` from the `sase-core` repository root, as required by its agent
   instructions, to execute formatting, Clippy, and all workspace tests, including the
   PyO3 binding tests.
3. Review the final diff and confirm it is limited to the metadata-check timeout and its
   rationale; re-run both validation commands after any correction.

## Acceptance criteria

- `twine check` no longer expires at 15 minutes before any step can run during ordinary
  runner contention.
- The self-healing wheel-to-metadata-to-PyPI dependency chain is unchanged.
- Deliberate release-plz concurrency behavior is unchanged.
- `actionlint` and `just check` pass on the final tree.
