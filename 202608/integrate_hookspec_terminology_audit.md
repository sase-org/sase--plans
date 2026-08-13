---
tier: tale
title: Integrate the frozen hookspec compatibility contract with the terminology audit
goal:
  The late hookspec compatibility fix passes the strict Patch/stitch audit, the full
  tree is verified, and reopened epic sase-hn.8.6 is landed cleanly.
size: medium
proposed_by: bbugyi200.athena.sase-hn.8.6.land
bead: sase-hn.8.6
create_time: 2026-08-09 07:33:14
status: done
---

- **PARENT:**
  [202608/patch_audit_gate_repair.md](https://github.com/sase-org/sase--plans/blob/main/202608/patch_audit_gate_repair.md)
- **BEAD:**
  [sase-hn.8.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-hn/sase-hn.8.6.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-hn.8.6.land](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-hn.8.6.land.md)
- **COMMITS:**
  - [11cd863](https://github.com/sase-org/sase/commit/11cd8634d6be9acd2c0e1b6fa5ff8fe5779a08ed)
    — test: annotate legacy workspace hook arguments

# Integrate the frozen hookspec compatibility contract with the terminology audit

## Goal

Finish landing epic `sase-hn.8.6` after two commits reached `origin/master` during its
first close sequence. Commit `466a24c38` only raises the published `sase-core-rs` floor
and needs verification. Commit `1659154a7` correctly restores the legacy
`changespec_file` and `changespec_parent` argument names at the Pluggy hookspec
boundary, but its new contract test introduces eight occurrences that the
now-content-aware Patch/stitch terminology audit classifies as defects.

The hookspec names must remain unchanged: Pluggy binds hook implementations by argument
name, and the shipped out-of-tree plugins still use these legacy spellings. Integrate
the new contract with the audit by making each intentional retained-boundary occurrence
locally self-declaring. Do not weaken an audit rule, add a path exemption, or rename the
compatibility API.

Epic `sase-hn.8.6` has been reopened and carries a `DISCOVERED ISSUE:` note for this
integration defect. It is caused by work in the terminology epic, so it remains epic
work rather than becoming a standalone follow-up task. The original epic plan is
`@plan:202608/patch_audit_gate_repair.md`; reflect the reopened state while work is in
progress and restore `status: done` only in the final landing phase.

## Phase 1: annotate the intentional compatibility boundary

1. In `tests/test_workspace_provider_hookspec.py`, keep the frozen hook signatures and
   their behavioral assertions intact.
2. Make the two explanatory prose occurrences around the historical breakage explicitly
   say that `changespec_file` and `changespec_parent` are legacy compatibility names.
3. Add concise local comments such as `legacy hook argument name` beside the two frozen
   tuple entries and beside the nested test plugin's parameter/return occurrences. The
   audit examines a narrow line context, so every one of the eight reported occurrences
   must have an adjacent compatibility marker.
4. Run the terminology audit after the edit and inspect its output. It must scan `main`,
   `sase-core`, `sase-github`, `sase-telegram`, `sase-nvim`, and `chezmoi` with zero
   defects, zero stale required rules, and no missing repositories.

## Phase 2: verify the integrated tree

1. Run the focused contract tests covering the integration and the audit:
   `tests/test_workspace_provider_hookspec.py`,
   `tests/test_patch_stitch_terminology_audit.py`,
   `tests/test_sase_core_rs_telemetry_smoke_tool.py`, and
   `tests/ace/tui/actions/test_prompt_save_xprompt.py`.
2. Run `git diff --check` and `just check-full`. The full check must pass after the
   fast-forwarded dependency-floor and hookspec commits plus the annotation patch.
3. Reconfirm that there are no new non-epic commits beyond `origin/master`, and record
   both late commit IDs and their dispositions for the close note: `466a24c38` required
   no adaptation, while `1659154a7` required the audit annotations above.

## Phase 3: land the reopened epic

1. Close `sase-hn.8.6` without force. The close note must summarize the original four
   child commits, all child-note and `PROPOSED FOLLOW-UP:` review outcomes, the two late
   integration commits, the eight repaired audit findings, and all final verification.
2. After the close succeeds, run `just symvision`. If it reports stale epic-symbol
   entries or unused code, read the Symvision memory through the required audited memory
   workflow, remove only the reported stale entries/unused code, reverify, and append
   the result to the bead. Do not force the close merely to bypass lifecycle validation.
3. Finally, set `status: done` in the frontmatter of
   `@plan:202608/patch_audit_gate_repair.md` and run committed-plan validation. Leave
   the primary checkout and plan sidecar clean, and append any evidence that arrived
   after the close with `sase bead note` rather than attempting a conflicting re-close.
