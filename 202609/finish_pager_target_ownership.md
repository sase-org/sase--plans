---
tier: epic
status: done
title: Finish pager target ownership at every action and entry point
goal:
  Pager source targets remain inside proved owner repositories, terminal owner-scoped
  outcomes cannot be bypassed by copy or generic filesystem search, every production
  pager entry point freezes configured artifact kinds before paint, and clean installs
  require the first released Rust binding containing those guarantees.
parent_bead: sase-xy.5.5
phases:
  - id: prove-owner-provenance
    title: Bound source-directory resolution to proved owner provenance
    size: medium
    depends_on: []
    description:
      "prove-owner-provenance: make source-directory and project identity constrain
      repository selection in Rust, rejecting out-of-inventory or mismatched provenance
      without regressing stale-checkout fallback, revision checks, filters, directories,
      ambiguity, or bounded search."
  - id: close-pager-actions
    title: Make pager copy and scanning honor the complete target contract
    size: medium
    depends_on: []
    description:
      "close-pager-actions: treat every returned owner-scoped outcome as terminal for
      copy as well as follow/edit, preserve semantic target identity through action
      dispatch, and freeze configured kinds in generic and ref-less bead pager sections."
  - id: publish-complete-contract
    title: Publish and ratchet the completed clean-install contract
    size: medium
    depends_on:
      - prove-owner-provenance
      - close-pager-actions
    description:
      "publish-complete-contract: release the corrected Rust resolver, raise sase's
      minimum and locked sase-core-rs revision to that release, and prove the combined
      pager contract from a clean installation with focused and full landing checks."
proposed_by: bbugyi200.athena.sase-xy.5.5.land
bead_id: sase-xy.5.5.4
create_time: 2026-09-09 19:52:24
---

- **PROMPT:**
  [prompts/202609/finish_pager_target_ownership.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/finish_pager_target_ownership.md)
- **PARENT:** [202609/pager_target_landing_repairs.md](pager_target_landing_repairs.md)
- **BEAD:**
  [sase-xy.5.5.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xy/sase-xy.5.5.4.md)

# Finish pager target ownership

## Why this remaining-work child exists

The land audit for `sase-xy.5.5` reviewed its three closed phase beads and all notes,
the linked plan, the current `sase` and `sase-core` source, and every commit landed
after the epic was created. Most of the repair plan is present, but the combined tree
still violates three acceptance boundaries that this epic introduced or promised to
close. These are epic work, not unrelated follow-ups, so `sase-xy.5.5` must stay open
until this child lands.

The audited baseline is `sase` commit `b67c74ce` and `sase-core` release commit
`a47171dd` / tag `v0.32.41`. The only unrelated commits after the parent child epic
began were the remote-dispatch landing (`8e00e742` in `sase`, `65203fc3` in
`sase-core`). Preserve them. In particular, `65203fc3` removes the obsolete
`remote_dispatch` directive gate; the phase-1 `PROPOSED FOLLOW-UP:` is already resolved,
and the land audit observed all 25 tests in
`tests/test_xprompt_directive_completion_parity.py` passing. Do not create a duplicate
task or reintroduce that flag.

## Reproduced remaining gaps

1. The released `sase-core-rs==0.32.41` trusts `owner.source_directory` after assigning
   it the requested repository name even when that directory is outside every checkout
   for that repository. With repository `owner` inventoried at an empty checkout and a
   different directory containing `secret.py`, the public
   `artifact_ref_resolve_document_source_target()` binding returns `status: exact`,
   `repository: owner`, and the unrelated path. This contradicts the parent plan's
   requirement that source-directory lookup use valid provenance and enforce repository
   containment. `ArtifactRefDocumentOwnerWire.project_key` also remains unused by the
   Rust resolver and by Python context selection, so stale owner paths can fall through
   to an unrelated viewer-project context without validating project agreement.
2. `copy_text_for_target()` calls `_owned_file_path_resolution()`, but only returns
   early for a successful target. After any owner-scoped failure it calls
   `_search_existing_path()` and can copy a decoy from the generic anchors. The land
   probe supplied a terminal `denied_filtered` result and observed copy return the
   decoy's absolute path. Follow/edit correctly stop on the Rust result. Copy must share
   that terminal behavior for ambiguous, filtered, revision-unavailable, missing-
   checkout, temporary-error, and proven-missing outcomes; only the absence of an owner
   lookup may enter generic search.
3. Configured-kind propagation is incomplete. `cli_pager._run_sase_pager()` creates its
   fallback stdin section with `default_link_context()` but leaves
   `PagerSection.known_kinds` empty. Bead-show sections derive their kinds from a
   reference context that `context_for()` refuses to construct when `issue.refs` is
   empty, so a custom typed reference appearing only in a title, description, or note is
   mis-scanned as a file path. Audit the remaining production `PagerSection`
   constructors with available link/artifact context and close equivalent gaps in
   generated landing/card or ACE sections where their body can contain typed refs.

## Behavioral contract

1. A source-relative hit is eligible only when the source directory is proved to belong
   to the owner repository/project under the supplied context. Explicit repository or
   project evidence cannot be satisfied by relabeling an unrelated path. Preserve the
   supported producer-artifact case where correlated owner metadata provides sufficient
   identity even if a stale checkout must fall through to a live same-repository
   checkout; document exactly which evidence is trusted. If `project_key` cannot safely
   constrain repository selection with the current wire, either extend the wire with a
   versioned contract or reject mismatched/unknown contexts—do not leave a decorative
   identity field that permits viewer-cwd fallback.
2. Owner-scoped resolution is one decision for follow, edit, and copy. Once Rust returns
   a resolution, Python must use its selected path or its original logical token and
   diagnostics; it must not stat generic anchors or owner candidates again. A `None`
   result meaning no owner context could be assembled is the only case that retains the
   ordinary generic path resolver.
3. Keep the complete immutable scanner target on `PagerTargetSpan` through cache and
   action dispatch. Equal visible strings with different Markdown, hosted, artifact, or
   reference-label destinations must neither share failure state nor collapse to a
   different action target. Preserve supported Markdown line/heading behavior and the
   separately actionable hosted URL occurrence.
4. Every production pager section that can contain typed references and has an artifact
   or link context freezes that context's configured kinds before rendering. This
   includes generic already-rendered CLI text and bead text with no stored `refs` rows.
   Context/config discovery remains outside paint and keypress handling, is cached per
   document/project where appropriate, and does not change plain non-pager rendering.
5. The minimum dependency and lock file select the first published `sase-core-rs`
   release containing the final provenance behavior. The tracked `sase-core-revision` is
   that release commit, the required binding/schema probes remain green, and no local
   unpublished wheel is accepted as clean-install evidence.

## Phase 1: Bound source-directory resolution to proved owner provenance

Work in `sase-core`'s artifact-reference repository resolver, wire types only if needed,
and focused unit/PyO3 tests. Reproduce the out-of-inventory source-directory acceptance
with real distinct repository roots, then make repository and project agreement explicit
before a source-relative hit can win. Cover an owner repository mismatch, an owner
project mismatch or unavailable project context, a source directory nested inside the
valid checkout, a valid correlated producer workspace, and stale-source fallback to a
live same-repository inventory checkout. Retain and rerun the existing directory,
filter, revision, ambiguity, traversal, and suffix-budget cases. Avoid interactive or
unbounded VCS work. Run the complete `sase-core` check surface required by that repo.

## Phase 2: Make pager copy and scanning honor the complete target contract

In `sase`, make `_owned_file_path_resolution()` authoritative for copy whenever it
returns a result. A successful owned file or directory copies its selected path;
terminal and retryable failures copy the original logical token without invoking
`_search_existing_path()`. Add regression coverage that plants an existing generic
anchor decoy behind each representative terminal/retryable owner outcome and proves no
second search occurs. Keep follow/edit diagnostics and retry caching unchanged.

Pass the scanned target's semantic identity through copy/follow/edit helpers wherever
the action needs more than its normalized string, with tests for equal visible labels
and distinct Markdown/hosted/artifact destinations. Then audit production `PagerSection`
construction. At minimum, freeze configured kinds for the generic `cli_pager` fallback
and for bead-show text even when the bead has no `refs`; cover a custom configured kind
appearing only in a bead note. Add focused coverage for any other context-bearing
constructors fixed by the audit. Do not perform config or filesystem discovery during
painting or label key handling.

Run the focused pager document/link-scan/action/resolve suites, CLI pager and bead-show
pager suites, relevant ACE view-files/link suites, formatting, and ordinary `just check`
after installing the workspace as required.

## Phase 3: Publish and ratchet the completed clean-install contract

After phase 1 is on `sase-core`'s default branch, publish the first normal release that
contains it. Update `sase`'s declared minimum, `uv.lock`, and `sase-core-revision.txt`
to that exact released commit. Keep the document scanner and target resolver names and
schema-version getters required by `tools/validate_sase_core_rs`; add a behavioral
capability probe only if version and schema identity cannot prevent the bad provenance
behavior from recurring within the supported window.

From a clean install, rerun the Rust repository checks, the target-resolution and
configured-kind entry-point regressions, copy/follow/edit pager behavior, rendered-link
contract/navigation/failure suites, artifact-read/ACE/bead-show paths, and relevant
visual snapshots. Run `just check` and the monitored `just check-full` landing gate per
`lint_and_test.md`. Preserve the remote-dispatch parity integration and verify its
focused suite remains green.
