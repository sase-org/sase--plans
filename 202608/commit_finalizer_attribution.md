---
status: done
tier: epic
title: Commit finalizer stops failing agents whose work actually landed
goal: "An agent whose work is committed and pushed is never marked FAILED by the commit
  finalizer's discarded-work guard. Commit provenance survives conflict resolution, the
  finalizer decides attribution from evidence the run itself owns rather than from
  commit-message footers alone, and a concurrent agent's activity in a shared clone can
  no longer be misread as this agent discarding its work.

  "
phases:
  - id: restamp
    title: Make the SASE commit footer survive conflict resolution
    depends_on: []
    size: medium
    description:
      "restamp: verify and re-stamp the run's SASE_* provenance footer onto HEAD during
      `sase stitch create --resume`, before the push, so a hand-resolved rebase conflict
      can no longer land the run's own commit unattributed."
  - id: ledger
    title: Record a run-owned commit ledger
    depends_on:
      - restamp
    size: medium
    description:
      "ledger: persist every commit SHA this run created or finalized into the agent
      artifacts directory, including the resume path that currently records a null
      result, so later stages have positive proof of what the run committed."
  - id: guard
    title: Decide attribution from run-owned evidence
    depends_on:
      - ledger
    size: medium
    description:
      "guard: teach the discarded-work guard to consult the run-owned commit ledger
      before declaring a discard, so a commit the run provably created counts as
      attributable even when its footer is missing."
  - id: shared
    title: Stop blaming an agent for concurrent activity in shared clones
    depends_on:
      - guard
    size: medium
    description:
      "shared: classify foreign-agent and already-published transitions in machine-wide
      clones as races rather than discards, extending the exemption beyond sdd-kind
      repos and putting the relaxation behind a feature flag."
  - id: hardening
    title: Actionable diagnostics and regression coverage
    depends_on:
      - shared
    size: small
    description:
      "hardening: replace the guard's dead-end failure text with an operator-actionable
      diagnostic, add end-to-end regressions reproducing both failure clusters, and
      refresh the affected docs."
proposed_by: bbugyi200.athena.05d
bead_id: sase-p5
create_time: 2026-09-09 19:50:08
---

- **PROMPT:**
  [prompts/202608/commit_finalizer_attribution.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/commit_finalizer_attribution.md)
- **BEAD:**
  [sase-p5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-p5/README.md)

# Plan: Commit finalizer stops failing agents whose work actually landed

## Why

Agents are being marked `FAILED` by the commit finalizer _after_ their work has been
committed and pushed. The failure is not a symptom of lost work — it is the guard
misreading a successful landing.

The most recent instance failed a phase worker whose commit is on `master` and matches
`origin/master`. The finalizer wrote:

```
Commit finalizer failed: dirty work vanished without an attributable commit. The
finalizer will not treat discarded, reset, or foreign-agent work as successful
finalization.
- main: <agent workspace checkout> (no newly reachable commit was attributed to this agent)
  - Justfile
  - src/sase/glossary/mutation.py
  - src/sase/glossary/relations.py
  - tests/test_glossary_mutation.py
  - tests/test_glossary_relations.py
```

Every one of those files is present in the landed commit. The agent did its job, closed
its bead, pushed, and was failed anyway. Because it was a phase worker, the false
failure also stalled its whole epic: the parent showed one failure with eight sibling
phases left `WAITING` behind it.

### Scope of the problem

Across all recorded finalizer runs in this project's artifacts, outcomes break down as:

| outcome                             | count |
| ----------------------------------- | ----- |
| `finalized` / `clean` (healthy)     | 4616  |
| `failed` / `dirty_after_max_passes` | 51    |
| `failed` / `dirty_work_discarded`   | 9     |

Eight of the nine `dirty_work_discarded` failures carry the `missing_agent_provenance`
reason (the ninth is `head_not_advanced` and is out of scope). All eight are recent and
clustered, which matches the report that agents "keep failing" rather than this being a
long-standing rare event.

### Root cause

The guard lives in `src/sase/llm_provider/commit_finalizer_git_progress.py`. For each
repo that was dirty before a finalizer pass and clean after it,
`discarded_dirty_work_evidence` concludes the work was discarded unless some commit
newly reachable in `before_head..after_head` carries a trailing `SASE_AGENT=` footer
matching the current agent.

That inference — _"my footer is absent, therefore I discarded my work"_ — is unsound.
The footer is a best-effort message decoration, not a durable record of what the run
committed, and there are two proven ways to reach the failure state with the work safely
landed.

**Cluster A — the footer is lost during conflict resolution.** Three of the eight
failures are this shape, and every one of them ran the commit skill with
`method=resume`.

`src/sase/workflows/commit/workflow.py:210` stamps the footer onto the payload message
via `apply_tracked_commit_tags` _before_ dispatch, so the message the workflow intends
to commit is correct. In the most recent failure the run's own `commit_results.json`
records exactly that intended message, footer included:

```
SASE_BEAD=[sase-p1.1][1]
SASE_TYPE=stitch
SASE_AGENT=[bbugyi200.athena.sase-p1.1][2]
```

Dispatch then hit a rebase conflict on the `Justfile` epic-symbol whitelist, so the
workflow returned `CONFLICT` and told the agent to resolve it and re-run
`sase stitch create --resume`. The agent resolved the conflict and, because upstream had
already dropped one of the whitelist paragraphs, re-authored the commit message to match
reality. Re-authoring dropped the entire `SASE_*` footer block along with the stale
paragraph.

Nothing downstream noticed. `resume_commit_workflow`
(`src/sase/workflows/commit/workflow_resume.py:57-72`) validates the resumed commit by
comparing **only the subject line** against the checkpoint. The subject was unchanged,
so resume proceeded, pushed, and reported success — landing the run's own commit with no
provenance footer at all. Confirming this at the repo level: 197 of the most recent 200
commits carry a `SASE_AGENT=` footer, and the handful that do not include this commit.

The same code path also destroys the only positive evidence that would have exonerated
the agent. Because dispatch failed with `CONFLICT`, `cp.dispatch_result` is `None`, so
the resumed run records `"result": null` and `"stitch_id": null` in `commit_result.json`
— the run finalized and pushed a commit whose SHA it never wrote down.

**Cluster B — a shared clone moves under a concurrent agent.** The remaining five
failures name the agents sidecar clone at `~/.sase/projects/<key>/repos/agents`, and one
also names the beads store. That clone is machine-wide, not per-workspace: every
concurrent agent publishes transcripts into the same checkout.

`collect_dirty_state` picks it up two different ways — as `kind="external"` via
`_dirty_opened_external_repos` (any repo this run opened through `/sase_repo`) and as
`kind="sdd"` via `_dirty_agents_prompt_archive_repo`. Either way, agent X's finalizer
can snapshot the clone dirty from agent Y's in-flight publication, watch agent Y commit
and clean it mid-pass, and then fail X because the new commit carries Y's `SASE_AGENT=`
footer.

The two escape hatches that exist do not cover this. `_new_commits_are_attributable`
exempts commits tagged with the agents-sync auto-commit `TYPE`, but only the sync paths
that actually stamp that type. `_published_store_state_is_exempt` is narrower still: it
requires `repo.kind == "sdd"` _and_ `git_upstream_ahead_count == 0`, so a
`kind="external"` opened-repo record gets no exemption at all, and an `sdd` clone with a
queued or deferred push is still treated as a discard.

**Why the blast radius is so large.** The guard raises `CommitFinalizerError` from
`_fail_on_discarded_dirty_work`, which is terminal. It fires after the commit is pushed,
so nothing is recoverable by retrying; it fails a phase worker whose bead is already
closed; and it stalls every sibling phase waiting on that worker. There is no advisory
tier, no re-check against evidence the run owns, and the message it prints tells the
operator what the finalizer refuses to do but not what to do about it.

## What this plan changes

Three independent things go wrong, so the fix has three independent parts, sequenced so
each one narrows the failure surface before the next relaxes anything:

1. **Stop producing unattributed commits** (`restamp`). The footer becomes durable
   across conflict resolution, which removes Cluster A at its source.
2. **Stop relying on the footer as the only evidence** (`ledger`, `guard`). The run
   records the SHAs it commits and the guard consults that ledger, so a footer lost to
   any future rewrite is no longer fatal.
3. **Stop attributing concurrent work to the wrong agent** (`shared`). Foreign-agent and
   already-published transitions in machine-wide clones are classified as races, which
   removes Cluster B.

Phase `hardening` then makes the remaining genuine failures legible.

The guard's real purpose — refusing to call a run finalized when an agent reset,
stashed, or `git checkout --`'d its own work away — must survive all of this. Every
phase below is a narrowing of the guard's false-positive surface, never a removal of its
true-positive behavior, and each phase is expected to prove that with a test that still
fails on a genuine discard.

## Make the SASE commit footer survive conflict resolution

Close Cluster A at the source: a resumed commit must carry the run's provenance footer
before it is pushed.

In `resume_commit_workflow` (`src/sase/workflows/commit/workflow_resume.py`), after the
existing conflict-state and subject checks pass and **before**
`provider.finalize_commit(...)` runs, compare the footer on `HEAD` against the tags the
checkpoint's payload message carries. When the run-owned tags are missing or no longer
match, re-stamp them onto `HEAD` and continue.

Design notes for the implementing agent:

- Re-stamping before `finalize_commit` is deliberate and safe. That hook already amends
  `HEAD` — `vcs_finalize_commit` in
  `src/sase/vcs_provider/plugins/_git_commit_dispatch.py` calls `_amend_bead_changes`,
  which runs `git commit --amend --no-edit`, and only then pushes. Amending a
  not-yet-pushed commit at this point is established behavior on this path, not a new
  risk.
- Reuse the existing footer helpers in `src/sase/workflows/commit/runtime_tags.py`
  (`parse_trailing_commit_tags`, `update_trailing_commit_tags`, and the
  `PRODUCED_RUNTIME_COMMIT_TAG_KEYS` / `STALE_RUNTIME_COMMIT_TAG_KEYS` sets) rather than
  hand-rolling footer parsing. Note that readers canonicalize both the legacy `AGENT=`
  and current `SASE_AGENT=` spellings, so a re-stamp must not duplicate a tag that is
  already present under either spelling.
- Preserve the agent's edits. The agent rewrote the body for a legitimate reason — the
  landed message was _more_ accurate than the checkpointed one. Re-stamp the tag block
  only; never restore the checkpointed body over the author's.
- Keep the run-owned tag set narrow: `SASE_AGENT` is what the guard reads, with
  `SASE_TYPE` and `SASE_BEAD` restored alongside it because the origin classifier and
  bead tooling depend on them.
- If the re-stamp cannot be performed, fail the resume loudly there — with the commit
  still unpushed and recoverable — instead of pushing an unattributed commit and letting
  the finalizer fail the agent afterwards.

Also tighten the resume identity check itself. Subject-only matching is what let a
substantially rewritten commit pass as "the expected commit"; it should additionally
confirm that `HEAD` is a commit this run is entitled to finalize (for example, that
`HEAD` is not already pushed to the upstream branch and that its tree still contains the
checkpointed diff's paths). Keep the check advisory enough not to break legitimate
conflict resolutions that change content — the goal is to stop silently finalizing an
unrelated commit, not to force the body to be byte-identical.

Regression coverage belongs in `tests/test_commit_workflow_resume.py`, which already
fakes the provider and exercises the conflict, subject-mismatch, and
`NotImplementedError` paths. Add a case where `HEAD`'s message has lost its footer and
assert the amend happens before the `finalize_commit` call.

## Record a run-owned commit ledger

Give the finalizer a source of truth that no message rewrite can erase.

Persist a ledger of every commit SHA this run created or finalized into the agent
artifacts directory, alongside the existing `commit_results.json`. It needs, per entry,
at least the repo path, the commit SHA, and the dispatch method that produced it.

The important gap to close is the resume path. Today
`_run_tracking_steps(cp, cp.dispatch_result)` is called with `cp.dispatch_result`, which
is `None` whenever the original dispatch ended in `CONFLICT` — which is precisely when
the footer is most likely to have been rewritten. The resumed run must resolve the SHA
it actually finalized (it has the repo and `HEAD` in hand) and record it, so the entry
that currently reads `"result": null` carries a real SHA.

Record the SHA _after_ any amend performed by `restamp` or `_amend_bead_changes`, since
amending rewrites it, and re-resolve it after the push if the push path can rebase. A
ledger entry pointing at a SHA that no longer exists is worse than no entry, so prefer
recording late and, where a rebase may have moved the commit, record enough to re-find
it (for example the patch-id or the tree of the committed paths) rather than the SHA
alone.

Keep this additive: existing `commit_result.json` and `commit_results.json` consumers
must not change shape. Prefer extending those records with the resolved SHA over
introducing a parallel artifact that can drift, and if a new artifact is warranted give
it a `schema_version` like its neighbors.

## Decide attribution from run-owned evidence

Rework `discarded_dirty_work_evidence` in
`src/sase/llm_provider/commit_finalizer_git_progress.py` so the footer becomes one
attribution signal among several rather than the only one.

For a repo that went from dirty to clean with `HEAD` advanced, treat the transition as
attributable when **any** of the following holds:

1. A commit newly reachable in `before_head..after_head` carries a matching
   `SASE_AGENT=` footer. This is today's check and stays first.
2. A commit newly reachable in that range appears in the run-owned ledger from `ledger`.
   This is the new signal, and it is what would have exonerated the recent failure.
3. A commit newly reachable in that range carries the agents-sync auto-commit `TYPE`.
   This exists today and stays.

Only when none of those hold should the run record discard evidence.

Ledger lookups must be resilient: the finalizer runs after a push that may have rebased
the commit, so match on the recorded SHA first and fall back to a content-based match
before giving up. A ledger that cannot be read must degrade to today's footer-only
behavior rather than raising — the guard's failure mode should never itself be a new way
to fail an agent.

Extend `tests/llm_provider/test_commit_finalizer_no_progress.py` with a case proving a
ledger-recorded but footer-less commit no longer produces evidence, and — critically — a
case proving that a genuine discard, where the ledger has no entry and `HEAD` did not
advance, still produces evidence and still fails.

## Stop blaming an agent for concurrent activity in shared clones

Close Cluster B, the majority of observed failures.

The core mistake is applying a single-owner assumption to a machine-wide checkout.
`~/.sase/projects/<key>/repos/agents` is shared by every concurrent agent, so "these
files went clean under someone else's commit" is the _expected_ steady state there, not
evidence of a discard.

Widen the exemption in `_published_store_state_is_exempt`, which today requires both
`repo.kind == "sdd"` and `git_upstream_ahead_count == 0`:

- Cover `kind="external"` records too. A repo the run merely opened through `/sase_repo`
  is not work this agent is responsible for committing, and today it gets no exemption
  at all — yet it is the single most common repo in the observed failures.
- Treat "another SASE agent committed this" as a race rather than a discard. When the
  newly reachable commits carry a well-formed `SASE_AGENT=` footer naming a _different_
  agent, the work was committed by someone; the correct response is a warning naming
  that agent, not failing this one.
- Reconsider the strict `ahead == 0` requirement. The beads-store failure shows an `sdd`
  clone being treated as a discard while its push was merely queued or deferred. Being
  ahead of upstream means publication is pending, which is a different condition from
  work being destroyed, and the existing `_fail_on_unpublished_bead_state` path already
  exists to report unpublished state honestly. Route pending publication there rather
  than through the discard guard.

Note the interaction with `_dirty_agents_prompt_archive_repo`, which deliberately
reports the agents sidecar only for canonical prompt-file edits. Whatever exemption
lands must not weaken that narrower, intentional case.

Put the relaxation behind a feature flag created with `sase flag new` — this changes
agent-facing failure behavior in a safety guard, so it needs a kill-switch back to
strict classification if it ever masks a real discard. Read `sase/memory/sase_flags.md`
with the `/sase_memory_read` skill before adding it, and note that `sase flag new` also
files the flag's removal bead. The flag should default to the new, relaxed behavior: the
current behavior is actively failing agents whose work landed, so strict classification
is the fallback, not the default.

## Actionable diagnostics and regression coverage

Make whatever failures remain useful.

`discarded_dirty_work_message` currently states what the finalizer refuses to do and
lists the changed files, which leaves the operator to reverse-engineer the cause from
artifacts. Give it the facts a reader needs to act: which repo, whether `HEAD` advanced,
what the newly reachable commits actually were and who they were attributed to, whether
a run-owned ledger entry was found, and the concrete next step. Distinguish the
`head_not_advanced` and `missing_agent_provenance` reasons in that output — they have
genuinely different causes and different remedies.

Add end-to-end regressions reproducing both clusters against the real guard:

- A resumed commit whose message was rewritten during conflict resolution finalizes
  successfully and lands with its footer intact.
- A shared clone that goes clean under a concurrent agent's commit does not fail the
  observing agent.
- A genuine discard — dirty files reverted with no commit anywhere — still fails, with
  the improved message.

Refresh the affected documentation to match the new behavior, including any commit
finalizer and `sase stitch create --resume` docs that describe the current subject-only
resume check or the current discard semantics.

## Verification

Run `just check` while iterating. Because this touches the commit workflow, the VCS
dispatch path, and the finalizer, run `just check-full` before landing — through
`/sase_monitor` with a `--next` action, never inline.

Beyond the test suite, the landed epic should be able to answer this concretely: replay
the eight recorded `missing_agent_provenance` failures and confirm each would now be
classified as attributable or as a race, while the recorded genuine-discard behavior is
unchanged. The finalizer result artifacts for those runs are available under the
project's `artifacts/ace-run` tree and carry the full error text, changed-file lists,
and the accompanying `commit_results.json` and `commit_skill_invoked.json` for each run.

## Out of scope

- The 51 `dirty_after_max_passes` failures. That is a different guard with a different
  cause — the agent genuinely left the tree dirty — and nothing here should change it.
- The single `head_not_advanced` discard. It may well be a true positive; it should be
  re-examined only after the `missing_agent_provenance` noise is gone.
- Changing whether the finalizer runs at all, its pass budget, or the
  `SASE_DISABLE_COMMIT_STOP_HOOK` escape hatch.
