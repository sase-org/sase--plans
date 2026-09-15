---
tier: epic
title: Finish global bead routing contracts and command acceptance
goal: Full-ID bead routing remains local-first, treats routing and publication failures
  as terminal, stores owner-correct plan references, and is proven across every existing-ID
  command surface before sase-116 lands.
parent_bead: sase-116
phases:
- id: routing-contract-repairs
  title: Repair local-first routing and fail-closed owner operations
  depends_on: []
  size: medium
  description: 'routing-contract-repairs: make operation routing discover foreign
    stores only after a real local miss, preserve actionable unavailable-store errors,
    stop fast-path and apply-status retries against the caller store after routing
    failure, require routed sidecar mutations to commit and publish successfully,
    and persist plan references against the selected owner while resolving input paths
    against the caller.'
- id: command-acceptance
  title: Prove every existing-bead command through isolated owner fixtures
  depends_on:
  - routing-contract-repairs
  size: medium
  description: 'command-acceptance: build a public-dispatch matrix for every existing-ID
    operand and owner-sensitive side effect, reproduce the original deep-ID outside-cwd
    close, verify mixed-store and unavailable failures leave all stores untouched,
    and run the combined focused and repository checks.'
proposed_by: bbugyi200.athena.sase-116.land
create_time: 2026-09-15 13:51:53
status: wip
bead_id: sase-116.5
---

- **PROMPT:** [prompts/202609/global_bead_resolution_landing_repairs.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/global_bead_resolution_landing_repairs.md)
- **BEAD:** [sase-116.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-116/sase-116.5.md)

# Finish global bead routing contracts and command acceptance

## Why this child exists

The landing audit for parent epic `sase-116` reviewed all four closed phase beads, their
notes, the linked plan, the current Python and Rust sources, and the phase commits in
both repositories. The existing focused routing, show, work, page, wait, and fast-path
suites pass (157 tests), but the implementation still violates explicit parent-plan
contracts. These defects are caused by the epic and must be repaired before the parent
closes.

This child contains only the remaining implementation and acceptance work. Its
`parent_bead` relationship is the handoff back to the interrupted parent landing. Do not
add phases for closing either epic, cleaning final epic-symbol entries, running the
parent's post-close Symvision pass, or marking either linked plan done.

## Audited baseline and reproduced gaps

The SASE baseline is `53035c9671`; the parent commits are `df87d68dcb`, `dbe3cd4bce`,
`0d5afc7739`, and `1954d0f1eb`. The Rust core baseline is `fd7bc24`; its parent phase
commit is `dd64c845af`, released in `sase-core-rs` 0.34.35. Preserve the shared Rust
membership, ambiguity, unavailable-store, and single-store batch policy already
implemented in `crates/sase_core/src/bead/routing.rs`.

Four isolated probes fail the intended contract:

1. `resolve_operation_context_for_targets()` calls `enabled_project_store_snapshots()`
   for an ordinary successful local full-ID lookup. The parent requires local membership
   to win without consulting the enabled-project registry or reading every foreign
   store.
2. `_resolve_fast_path_context(..., terminal_errors=True)` catches
   `BeadOperationRoutingError` and returns the caller's local context when one exists.
   Mixed-store, ambiguous, unavailable, and missing full-ID failures can therefore be
   retried in the wrong store. `_run_apply_status()` separately retries a routed
   not-found against the local mutation path.
3. `handle_bead_create()` computes `storage_plan_path()` before resolving a foreign
   parent. A plan under a sibling owner's `sdd/plans/202609/child.md` was persisted as
   the caller-relative `sdd/plans/202609/child.md` rather than `plan:202609/child.md`;
   `normalize_workspace_path()` incorrectly treats a different sibling project as an
   ephemeral checkout of the caller.
4. A changed routed sidecar mutation whose auto-commit callback returns `False` exits
   `bead_store_mutation()` successfully. The equivalent fast-path side-effect helper
   also reports success when routed auto-commit fails. The parent requires a nonzero,
   actionable failure and forbids printing success for unpublished foreign mutations.

No child note on `sase-116.1` through `.4` contains a `PROPOSED FOLLOW-UP:` entry. The
post-start commits outside this epic are unrelated ACE, retention, usage, completion,
workspace, sudo, and disk work; the only overlap, settlement notification changes in
`epic_launch.py`, remains compatible with the routed work context. The core release
commit containing 0.34.35 and the later main dependency-floor update are already
integrated.

## Phase 1: routing-contract-repairs

Make target discovery genuinely local-first in `src/sase/bead/operation_context.py`. Use
the Rust policy for the local probe and only materialize enabled-project snapshots when
one or more syntactically full targets miss locally. Preserve one invocation snapshot
and deterministic argv order. A batch with only local targets must never call the
project registry; mixed local/foreign batches must still be resolved together and
rejected before writes when their stores differ. Keep shorthand strictly local. Ensure
an unreadable relevant local or foreign store produces the structured actionable
unavailable diagnostic rather than degrading to ordinary not-found.

Make a resolved routing error terminal in `src/sase/main/bead_fast_path.py`. Do not
retry a target-aware fast-path command against `local_operation_context`; unsupported
syntax may still defer to argparse before routing, and targetless commands retain their
current local behavior. Apply the same rule to `apply-status`: it must use its routed
context or report the route failure through the typed operation result, never open or
initialize the caller store after a miss. Cover missing, ambiguous, unavailable, and
mixed-store cases, and assert that the Rust executor and mutation context are not
entered after failure.

Strengthen publication semantics only for explicitly routed foreign stores. A changed
sidecar mutation with an intended commit that cannot be committed or verified must raise
the existing bead-publication error path, emit an actionable diagnostic, return nonzero,
and suppress success output in both Python and Rust lanes. Preserve in-tree behavior,
local best-effort compatibility, true no-ops, `--no-push`, and successful sidecar
commit/push verification. Do not invent a second publication mechanism.

Resolve user-supplied plan paths against the original invocation cwd before routing,
then derive the stored design reference from the selected owner's `BeadsLocation` /
`SddStore` and primary workspace. Reuse `plan_ref_for_store()` or an equivalently
explicit owner-root API rather than process-wide `chdir`. Never normalize a different
project merely because its checkout is a filesystem sibling; retain normalization for
actual numbered workspaces of the same owner. Test in-tree, combined and split sidecars,
separate-repo, local, external-path, and numbered-owner cases.

Run focused routing, mutation-publication, create/design-reference, fast-path, and ops
tests, then `just check`. Open `sase-core` through `/sase_repo` if shared unavailable
semantics require a Rust adjustment, and run its full repository check before closing
the phase.

## Phase 2: command-acceptance

Inventory current registrations in `src/sase/main/parser_bead*.py` and
`src/sase/ops/commands/bead.py` after rebasing on the then-current tree. Add a
table-driven isolated-project regression matrix through public dispatch for every
existing-bead input and owner-sensitive effect:

- `show` (including mixed batches, `ID..`, formats, and strict `--project`), `close`
  (including phases, gate settlement, symbol scan root, note/reason files, and no-push),
  `open`, `note`, `+1`, `update`, `rm`, and `snooze`;
- parented `create`, `apply-status`, dependency add/rm/list/tree, reference
  add/rm/list/resolve, normal and lost-note history/restore;
- sequential bead `work`, plan-file parent overrides, full-ID bead waits, page
  URL/refresh preview/write, and scoped epic-symbol scans.

Use real temporary bead stores and public command dispatch wherever practical; mocks may
isolate git/network/agent launch side effects but must not replace routing, membership,
or store selection. Cover local-hit registry avoidance, foreign valid IDs, closed and
deeply nested IDs, custom/historical prefixes, exact ambiguity, relevant and unrelated
unavailable stores, mixed shorthand/full batches, same-store foreign batches,
mixed-store rejection, and last-operand failure. Assert all rejected preflights leave
every store and caller directory unchanged.

Reproduce the original `sase-xe.16.11.7.15.7`-shaped close from a directory outside
every project without touching live stores. Verify the target closes and publishes in
its owner context, the caller does not gain an initialized store, and relative `@<path>`
inputs still resolve from the caller. Exercise both argparse and supported Rust fast
paths. Confirm no bead-ID surface landed after this plan was authored without the same
routing contract.

Run the combined focused matrix and `just check`. Record exact test counts and any
intentional exclusions in the phase note so the child and parent landing audits can
reconcile the inventory. The resumed parent landing agent remains responsible for the
original epic's final `just check-full`, epic-symbol cleanup, close, Symvision pass, and
plan-status update.

## Constraints

- Read `lint_and_test.md` and `symvision.md` through `/sase_memory_read` before
  finishing tracked SASE changes. Use `/sase_monitor` for long checks.
- Shared store-neutral routing policy belongs in `sase-core`; Python owns project/store
  discovery, operation contexts, filesystem paths, publication orchestration, and CLI
  presentation. Do not add a Python fallback for Rust policy.
- Do not mutate live bead stores in tests. Preserve exact case semantics, canonical full
  IDs, caller-relative file inputs, current validation/no-op rules, and the established
  one-store transaction boundary.
- Phase workers record genuinely unrelated discoveries only as `PROPOSED FOLLOW-UP:`
  notes on their own phase bead. Epic-caused acceptance failures remain child work.
