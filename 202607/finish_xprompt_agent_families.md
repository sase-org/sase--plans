---
tier: tale
title: Finish and land xprompt agent families
goal: 'Close the remaining editor and documentation parity gaps for the %family directive,
  lock down its execution-neutral launch contract, validate the integrated SASE and
  sase-core implementations, and finish the sase-6g landing sequence without leaving
  stale symvision exemptions or plan state.

  '
create_time: 2026-07-16 21:11:20
status: done
prompt: 202607/prompts/finish_xprompt_agent_families.md
---

# Plan: Finish and land xprompt agent families

## Context

Epic `sase-6g` has eight closed phase beads. The implementation is present in the current SASE, sase-core, and chezmoi
checkouts: directive parsing, runner-slot accounting, launch-time membership metadata, generation-pinned wait/fork
resolution, cleanup cascades, TUI aggregation and member details, epic bead-work adoption, and the research swarm
migration. The current feature commits are `702ab603a`, `e24fd654f`, `8c73c22c5`, `c3040b945`, `a0a81e445`, `d39577633`,
and `560177340` in SASE; `c8ea7de` and `a90acdc` in sase-core; and `b802fe45` in chezmoi. Focused validation currently
passes 152 Python tests, including the ACE PNG snapshot. Rust formatting and clippy pass; the complete Rust suite passed
every feature-related test, and its sole host-bridge harness failure passed on an isolated rerun.

The land audit found two concrete parity gaps. First, the Rust editor catalog in sase-core does not advertise
`%family`/`%f`, so xprompt LSP completion and hover lag the Python parser and ACE completion. This intersects the later
`f6e09fe` editor-catalog change that landed while the epic was in progress. Second, the later general directive
documentation in `docs/xprompt.md` omits `%family` from the supported-directive table and syntax examples even though
`docs/agent_families.md` documents the feature. The epic plan also calls for an explicit execution-neutrality regression
test; the behavior is covered in pieces today, but the promised comparison is not encoded as one durable gate.

## Phase 1: Restore editor and documentation parity

Open the linked sase-core checkout with the repository skill before editing it. Add `family` to
`crates/sase_core/src/editor/directive.rs` with alias `f`, a single required argument, and wording consistent with the
Python/ACE catalog: membership joins a parallel family rooted at another launch segment. Extend the catalog tests to
cover canonical alias resolution, completion metadata, and the fact that `%family` is single-valued. Add or extend an
xprompt LSP completion test so the public editor surface, not only the catalog helper, is protected. Do not add fixed
argument candidates: root names and roles are open-ended.

Update the Supported Directives table and syntax examples in `docs/xprompt.md` with `%family`/`%f`, both colon and
`(root, role=token)` forms, its execution-neutral semantics, and a link to `docs/agent_families.md`. Preserve the
gate-owned `%auto` wording from the post-epic-start integration commits.

## Phase 2: Lock down launch neutrality and validate the integrated feature

Extend `tests/test_parallel_agent_family_launch.py` with a direct comparison of the same multi-prompt launch with and
without `%family`. Assert that planned names, rewritten waits, models, VCS/project contexts, workspace/deferred-work
properties, and spawn ordering are unchanged; only the family payload and the directive text/metadata may differ.
Include a member fan-out case if needed to make the plan's documented rule—member fan-out is legal and every variant
joins the same generation—explicitly testable without weakening the root fan-out rejection test.

Run the focused Python tests for directives, launch planning/metadata, runner-slot admission, cleanup, status
aggregation, member details, launch preview, bead-work rendering, and the ACE family PNG snapshot. Run `just rust-check`
for sase-core parity, allowing a single isolated rerun only for the already-observed host-bridge availability flake.
Then run `just check` from SASE, as required for repository changes. Treat any failure related to the feature or the
edited surfaces as remaining epic work and fix it before landing.

## Phase 3: Land the epic (final phase)

Only after both repositories validate, close the epic with `sase bead close sase-6g` and confirm all eight children and
the parent are closed. After closing, run `just symvision`; the close expires any `sase-6g` symbol whitelist entries. If
symvision reports stale whitelist entries or newly unused code, read the required symvision memory through the audited
memory skill, remove only the reported stale entries/code, and rerun symvision plus `just check` after those edits.

Open the plans sidecar through the repository skill and change only the epic plan `202607/xprompt_agent_families.md`
frontmatter from `status: wip` to `status: done`. Re-run plan validation if the sidecar workflow provides it, then
verify clean/expected status in SASE, sase-core, chezmoi, and the plans sidecar. Do not edit SASE memory files or
generated provider instruction files.
