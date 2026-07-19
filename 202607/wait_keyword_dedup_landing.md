---
tier: tale
title: Finish wait-target completion and land sase-7h
goal: 'ACE and the xprompt LSP both suppress already-selected wait targets and keywords,
  the completed group-aware completion epic is fully verified, and sase-7h is closed
  with its Symvision cleanup and durable plan status finalized.

  '
create_time: 2026-07-19 13:34:40
status: done
prompt: 202607/prompts/wait_keyword_dedup_landing.md
---

# Plan: Finish wait-target completion and land sase-7h

## Context

Epic `sase-7h` added kind-aware completion for agents, families, clans, and tribes across the Python editor-helper
bridge, the Rust xprompt LSP, and the ACE prompt input. Its child beads are closed and their landed commits are present:

- `sase-7h.1`: `390a7f1e` in `sase` and `1c28bc2e` in the linked `sase-core` repository.
- `sase-7h.2`: `4d7c9aac` in `sase`.

The landing audit found no non-epic commits after the first `sase-7h` commit in either repository, so there is no later
commit stream to reconcile. The bead-linked plan is `${SASE_SDD_PLANS_DIR}/202607/agent_group_completion.md`.

The implemented feature and its focused tests are otherwise sound: 91 relevant Python tests, both new PNG snapshots, and
the complete `sase_core` plus `sase_xprompt_lsp` Rust test suites passed during the audit. One shared-contract gap
remains. For `%wait(time=5m, )` and `%wait(runners=1, )`, the ACE completion menu still offers both `time=` and
`runners=` because selected-value extraction currently discards clauses containing `=`. The Rust/LSP path already tracks
those clauses and suppresses the selected keyword. The linked epic plan requires already-selected values in a
multi-value `%wait` clause to be excluded on both surfaces.

## Phase 1: Complete ACE selected-value exclusion

Make `%wait` selected-clause tracking represent both prompt targets and keyword clauses without changing active-clause
replacement behavior. Use the existing token-context and completion-builder flow rather than introducing another parser
or data read.

The resulting ACE behavior must:

- suppress `time=` when another clause already contains a `time=...` value;
- suppress `runners=` when another clause already contains a `runners=...` value;
- continue suppressing selected agent, family, clan, and canonical `@tribe` targets;
- skip the active clause itself, including when editing an earlier clause with selected values on the right;
- work for both colon and parenthesized `%wait` forms and preserve case-insensitive target exclusion;
- leave keyword-first ordering, prefix filtering, soft completion, and token replacement unchanged.

Keep the change localized to the ACE directive token/completion helpers and their callers. Prefer a single
selected-value representation that the keyword and target builders interpret consistently, matching the already-landed
Rust behavior.

## Phase 2: Revalidate the completed epic contract

Add focused regression coverage for duplicate keyword suppression in both parenthesized and colon forms, including a
cursor positioned in an earlier clause. Retain coverage for target exclusion and all four rendered target kinds.

Run at least:

- the directive token, directive completion, directive interaction, agent completion, xprompt agent-argument, and
  editor-helper Python tests touched by `sase-7h`;
- the two prompt-target PNG visual snapshot tests;
- `cargo test -p sase_core -p sase_xprompt_lsp` in the linked `sase-core` checkout if the implementation crosses or
  affects the Rust boundary;
- `just check` in `sase` after all source cleanup is complete, as required for repository file changes.

Do not change `%wait` parsing or launch semantics. This work only completes the menu contract already specified by the
epic.

## Phase 3: Land and finalize sase-7h

This is the final phase and must run only after the implementation and focused verification pass.

1. Run `sase bead close sase-7h`.
2. After the close, use the `/sase_memory_read` workflow to read the Symvision guidance for `sase/memory/symvision.md`,
   then run `just symvision` if the recipe is available.
3. Remove expired `sase-7h` epic-symbol whitelist entries and any newly exposed unused code reported by Symvision. Rerun
   Symvision and the required repository checks until they pass.
4. Resolve the plans sidecar through `sase repo open plans` and change only the linked epic plan's frontmatter from
   `status: wip` to `status: done` in `${SASE_SDD_PLANS_DIR}/202607/agent_group_completion.md`.
5. Re-run `sase bead show sase-7h` and inspect repository status so the handoff confirms the epic is closed, the plan is
   done, and no unintended files were changed.

Preserve the ordering above: closing the bead must precede Symvision because the epic whitelist expires on close, and
the durable plan status is the final state change.

## Risks and guardrails

- Do not close the epic before the missing ACE behavior and its regressions are complete.
- Do not edit SASE memory files; only read the Symvision memory through the audited memory workflow.
- The plans checkout is a sidecar repository, so access it through the SASE repo workflow and use the path that command
  returns.
- Preserve unrelated user changes in every checkout and do not create a commit unless explicitly requested.
