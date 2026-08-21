---
tier: tale
title: Automatically persist artifact-link sidecar changes
goal:
  Every durable artifact-link index is committed and published in the smallest safe
  sidecar batch while runtime lock sentinels stay local and finalizer handoffs remain
  declaration-free.
size: medium
proposed_by: bbugyi200.athena.09s
create_time: 2026-08-21 12:51:42
status: wip
---

# Plan: Automatically persist artifact-link sidecar changes

## Outcome

Make every artifact-link mutation durable without asking an agent to reason about
machine-generated sidecar files. Explicit link commands commit their own changes, and
implicit agent `read` links that accumulate during a run are committed by the built-in
commit finalizer even when the turn ends through a plan handoff and therefore has no
final declaration. Batch all pending link-index changes into at most one commit per
affected sidecar repository; separate repositories still require separate commits.

Keep the contract narrow: the schema-v2 `links/**/*.json` indexes are durable VCS truth,
while their empty `*.lock` siblings are local `flock` sentinels rather than link data.
The locks must continue to exist locally and must never be unlinked while another
process could hold them, but newly created locks should be ignored instead of committed.
The rebuildable project aggregate and its lock under SASE state remain local and outside
all sidecar commits.

## Diagnosis

- `sase artifact read` records an agent-to-artifact `read` row through
  `ArtifactLinkStore.upsert_row`. For a plan, that atomically writes
  `links/<plan-path>.json` in the plans sidecar and opens a zero-byte sibling lock. It
  also rebuilds the project-local aggregate, but it does not commit the sidecar.
- Agent `09l` read two plans before proposing its own plan, leaving two JSON indexes and
  two locks untracked. A successful `sase plan propose` is an intentional handoff, so
  the provider correctly skipped `/sase_final`; the host later reached the commit
  finalizer with no accepted declaration. Existing reconciliation knows how to
  auto-commit bead state, prompt Q&A, and a mechanical plan-status change, but not
  artifact-link indexes, so the run failed instead of safely consuming the generated
  files.
- The JSON files are not incidental: they contain the only per-artifact schema-v2 link
  rows and are the source used to rebuild `artifact-links.json`. Documentation already
  describes them as the tracked indexes. The lock files contain zero bytes, are created
  by the generic `locked_file` helper, and never contribute to graph reconstruction.
  Some locks have already entered plans-sidecar history because broad commits swept them
  up, but that precedent does not make future locks semantic data.
- The projection refresh path already creates a local commit for a batch of JSON and
  Markdown changes, while manual `artifact link add/rm` and implicit `artifact read`
  writes do not share a complete commit boundary. The fix therefore needs one common
  path/lifecycle contract plus two commit triggers: command completion for explicit
  mutations and finalizer reconciliation for deferred agent reads.

## Implementation

1. Define the artifact-link repository-file contract next to the existing store/path
   helpers. Recognize only canonical `links/**/*.json` indexes whose payload is a
   schema-v2 object, whose `artifact_ref` maps back to that exact relative index path,
   and whose rows pass the Rust-backed artifact-link validator. Recognize a sibling
   `*.lock` only when it is a regular zero-byte file paired with such an index. Reject
   malformed, mismatched, symlinked, nonempty, or unpaired candidates instead of
   auto-committing or silently hiding them.

2. Add a narrow artifact-link lock ignore helper, modeled on the bead-store ignore
   helper, which preserves existing `.gitignore` content and installs only the rooted
   sidecar pattern for `links/**/*.lock`. Seed that rule in every document sidecar at
   sidecar initialization, and apply it lazily before the first artifact-link commit in
   an already-materialized sidecar. Include a newly created or amended `.gitignore` in
   that same first link commit. Do not delete lock files, rewrite sidecar history, or
   forcibly untrack locks that earlier commits already track; tracked empty sentinels
   can remain compatibility residue, while no new lock is added to VCS.

3. Introduce one path-scoped artifact-link commit helper over the existing SDD store
   commit machinery. Given the changed/touched link indexes, partition them by owning
   sidecar, add the lazy `.gitignore` change when needed, and create no more than one
   conventional `chore(artifact-links): ...` commit per repository. Reuse SDD store
   locking, commit markers/diff artifacts, runtime agent attribution, the existing SDD
   commit type, and the `artifact_links` file-hook cause. Never stage the whole sidecar,
   Markdown documents outside a caller's declared projection batch, bead state, an
   arbitrary JSON file, or the project-local aggregate. A no-op link upsert/removal must
   create no commit.

4. Route explicit `sase artifact link add` and `rm` through that helper after the whole
   graph mutation succeeds, so one command can update both endpoints and then make one
   commit in each repository it actually changed. Preserve current output and
   idempotency. When a bead endpoint writes `LinkAdded`/`LinkRemoved` events, fold that
   state into the existing bead mutation/auto-commit and publication boundary once per
   command rather than leaving the bead half of the edge dirty or staging it as a
   document-sidecar file. If any scoped commit or required publication fails, return a
   clear nonzero diagnostic while leaving every validated JSON/event mutation
   recoverable for finalizer or operator retry. Update artifact-link projection refresh
   to install the ignore rule and include it in the projection's existing batch commit,
   preventing a later lock-only cleanup commit.

5. Extend `prepare_commit_dirty_state` with a machine-owned artifact-link reconciliation
   pass. From the attributable post-baseline dirty state, validate and gather all
   eligible link JSON in every SDD sidecar, ignore only validated local lock sentinels,
   and commit each repository once through the shared helper. This pass may peel the
   generated link paths out of a repository that also contains unrelated work, but it
   must leave every unrelated or pre-existing path for the normal final declaration.
   Rescan after the batch so a plan-handoff run with only link data becomes clean before
   the commit finalizer tries to load a declaration. Record the auto-commit outcome in
   reconciliation/finalizer evidence, and synchronously verify publication of a
   finalizer-created sidecar commit (or fail with a recoverable publication diagnostic)
   so an ephemeral checkout cannot report success while holding the only copy.

6. Document the lifecycle explicitly: sidecar `links/**/*.json` is committed durable
   truth; `links/**/*.lock` is persistent local synchronization state and ignored;
   `artifact-links.json` is a rebuildable project-local aggregate. Explain that explicit
   commands commit once per affected sidecar, implicit agent reads batch at
   finalization, and crossing repository boundaries sets the irreducible lower bound on
   commit count. Keep this Python-side orchestration change out of the Rust core because
   the row/path validation already lives behind Rust bindings while Git finalization and
   sidecar publication are frontend/runtime concerns.

## Verification

- Add path-contract tests covering valid new and modified schema-v2 indexes, nested
  artifact paths, path/ref mismatches, schema-v1 or malformed JSON, invalid rows,
  symlinks, and paired versus nonempty/unpaired lock files.
- Add sidecar initialization and lazy-upgrade tests proving the exact lock ignore rule
  is appended without disturbing existing `.gitignore` content, reruns are byte-stable,
  locks remain usable on disk, new locks stay out of status, and historically tracked
  locks are neither deleted nor recommitted.
- Add explicit CLI integration tests for add, update/no-op, relation removal, two
  endpoints in one sidecar, a bead-to-document edge, a bead-to-bead edge, and document
  endpoints across two sidecars. Assert zero commits for a no-op, one commit for an
  arbitrary number of changed indexes in one repository, one existing bead-store commit
  when bead events change, and exactly one commit in each of two document repositories
  when both own durable rows.
- Extend projection-refresh tests to prove a first-time JSON index and its ignore rule
  land in the existing projection commit, with no dirty or lock-only residue afterward.
- Add finalizer reconciliation tests modeled on `09l`: two implicit plan reads produce
  four initial filesystem sidecars, no final declaration is published because a plan
  handoff is pending, and finalization produces one plans-sidecar commit containing the
  two JSON indexes plus the ignore rule, excludes both locks, records commit evidence,
  publishes the commit, and finishes clean. Cover mixed unrelated dirt, pre-existing
  dirty indexes, malformed candidates, multiple sidecars, publication failure, retry,
  and a second idempotent reconciliation pass.
- Run `just install`, the focused artifact-link, sidecar-init, SDD commit/publication,
  and finalizer suites, then `just check`. Inspect the resulting temporary Git histories
  and status output in the integration tests to confirm the commit-count claims rather
  than relying only on mocked call counts.

## Acceptance criteria

- A plan-only agent may read any number of plans, propose its plan, and finalize without
  a `/sase_final` declaration or a dirty-sidecar failure when link indexes are the only
  generated changes.
- Every valid JSON link mutation and bead link event is committed and published through
  its owning scoped, attributable SDD path; unrelated files are never swept into the
  automatic commit.
- Commit count is zero for no semantic change, one per changed sidecar repository for a
  batch, and greater only when mutations happen in distinct repositories or at distinct
  already-committed lifecycle boundaries.
- New per-index lock sentinels remain functional but do not appear in Git status or new
  commits; project-local aggregate files remain outside repository commits.
- Existing sidecars upgrade lazily without deleting local locks or rewriting history,
  and all focused plus repository-required verification passes.
