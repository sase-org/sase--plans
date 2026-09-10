---
tier: tale
title: Catch writes made to an opened repository's primary checkout
goal:
  The commit finalizer catches attributable writes made through stale paths to opened
  repositories' primary checkouts without claiming pre-existing shared work.
size: medium
proposed_by: bbugyi200.athena.research.0s.image.f0
create_time: 2026-09-09 19:59:57
status: wip
---

# Catch writes made to an opened repository's primary checkout

## Objective

Fix the commit-finalizer blind spot demonstrated by agent `research.0s.image`: when an
agent opens an isolated linked or sidecar checkout but writes through a stale absolute
path into that repository's primary checkout, SASE must not finish with
`reason: no_changes`. Detect attributable misdirected writes, give the agent the normal
bounded commit follow-up against the checkout that actually changed, and retain the
existing protection for pre-existing or unrelated shared-primary work.

## Root cause and evidence

The saved run artifacts establish both halves of the failure:

- `sase repo open research` printed and marker-recorded the run's isolated `research`
  checkout.
- The agent later copied the generated PNG to an absolute primary-checkout path carried
  forward from the prior conversation instead of using the printed path.
- `commit_finalizer_result.json` contains `status: clean`, `reason: no_changes`, and no
  changed files because `collect_dirty_state()` correctly inspected the recorded
  isolated checkout, which remained clean; it never inspected the corresponding primary
  checkout.

The path-selection mistake is an agent-guidance failure, but scanning every configured
primary as an ordinary commit target would be unsafe: primary checkouts are shared and
may contain concurrent human or agent work. The backstop therefore needs both an
opened-repository scope and a trustworthy pre-turn baseline.

## Implementation

1. Extend the finalizer's resolved linked-repository target data to retain both the
   workspace checkout and its configured primary checkout. Populate the pair uniformly
   from launch metadata and the configuration fallback, normalize both paths, and keep
   legacy static/shared records on their existing compatibility path.
2. Expand runner-start baseline capture so every distinct configured primary counterpart
   receives an explicit entry, including `{}` for a clean checkout. Keep the existing
   dirty fingerprints for the main, linked, external, and SDD repositories. The explicit
   empty entry is important evidence: absence means baseline capture could not establish
   ownership and must not authorize a shared-primary commit prompt.
3. During dirty-state collection, consider a primary counterpart only when this run's
   opened-repository marker names that linked repo, the primary and workspace paths are
   different, and the baseline contains that primary path. Compare its current dirty
   fingerprints with the baseline and add only new or subsequently modified paths to the
   blocking set; leave unchanged pre-existing paths excluded and reported through the
   existing pre-existing-details mechanism. Deduplicate aliases by normalized path.
4. Give attributable primary-checkout dirt an explicit dirty-repository kind and prompt
   label explaining that the agent wrote outside its isolated checkout. The finalizer
   should instruct the agent to preserve the detected work by invoking the existing
   `/sase_git_commit` workflow from the checkout that actually changed, then verify
   cleanliness exactly as it does for other non-primary repositories. Do not silently
   copy, reset, stash, or discard the shared checkout, and retain current bounded-pass,
   attributable-HEAD, publication, and lost-work checks.
5. Strengthen the canonical `sase_repo` generated-skill source so agents bind the
   command's stdout to a repository directory and use that value for every subsequent
   read, write, and command. State explicitly that absolute repository paths appearing
   in user text or prior transcript output are stale/informational after
   `sase repo open` resolves the current run's checkout. Add a focused source-content
   assertion and preview the generated output with `sase skill init --diff`; do not edit
   generated provider files directly.

## Regression coverage

- An opened linked/sidecar repo whose isolated checkout stays clean but whose clean-at-
  baseline primary gains a file must trigger one finalizer pass naming that file and
  actual checkout; an attributable commit there must finalize successfully.
- A primary path dirty before baseline and unchanged afterward must not trigger a
  commit; if the agent edits that same path or adds another path, only the attributable
  work must block.
- A dirty primary for a repository this run never opened must remain out of scope.
- Missing, corrupt, or path-incomplete baseline data must never convert shared-primary
  dirt into a commit requirement.
- Primary/workspace path aliases and duplicate discovery sources must produce one
  obligation.
- Existing isolated linked-repo, external-repo, SDD, baseline inheritance, no-progress,
  discarded-work, and successful-commit tests must remain green.
- The shipped `sase_repo` skill source must retain the stdout-path rule and the new
  stale-path warning.

## Verification

Run focused baseline, linked-repository finalizer, prompting, and generated-skill source
tests while iterating. Run `sase skill init --diff` to inspect provider-neutral rendered
guidance. Because the SASE repository changes, run `just install` and then `just check`;
if scoped selection escalates or reports unusual coverage, use `/sase_monitor` for
`just check-full` with the required `TESTING`/`TESTED` statuses. Re-run focused tests
after formatting or lint fixes, inspect the final diff for unrelated changes, and
confirm no implementation path broadens primary-checkout prompting beyond opened,
baseline-attributable repositories.
