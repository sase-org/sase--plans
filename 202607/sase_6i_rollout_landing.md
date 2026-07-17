---
tier: tale
title: Finish and land the custom notification gates epic
goal: Restore the rollout coverage that phase 8 reported but did not land, revalidate
  the cross-repository integration, and close sase-6i with all post-close cleanup
  and plan state persisted.
create_time: 2026-07-17 02:40:49
status: done
prompt: sdd/plans/202607/prompts/sase_6i_rollout_landing.md
---

# Plan: Finish and land the custom notification gates epic

## Context

Epic `sase-6i` added first-class custom notification gates across the main SASE repository, `sase-core`,
`sase-telegram`, and the generated chezmoi skill sources. The eight child beads are closed and their source commits are
present, but the land-agent audit found one concrete handoff gap: phase `sase-6i.8` reported adding and passing an
explicit Telegram none/some/all custom-gate add-on matrix, then its finalizer committed only the five documentation
files in the main SASE repository (`3bbcfda69`). The current `sase-telegram` `origin/master` ends at `1437771`, and
`tests/test_custom_gates.py` does not contain that explicit matrix. The prior separate-repository working diff is no
longer recoverable from the prepared workspaces, so the coverage must be recreated and landed before closing the epic.

The audit otherwise confirmed the implementation in current source: the Rust wire has additive icons and generic
`CustomGate` details; the Python service validates privileged custom gates, feedback, extras, hashes, automatic-policy
rules, polling, and mobile execution; ACE uses verified modals and tracked executor tasks; plan approval preserves the
legacy presets while deriving `commit_plan` and `run_coder` from extras; Telegram uses compact server-side toggle state,
two-step feedback, executor routing, neutral HITL handling, and race dismissal; and `/sase_gate` is identical across all
five generated provider targets. The only non-epic main-repository commit after the epic began was `3b263a578`, which
preserves canonical plan identity; current plan-gate and approval code still carry its durable proposal-path behavior.

## Phase 1: Restore the missing rollout fixture

In the audited `sase-telegram` linked checkout, add explicit end-to-end coverage for a custom gate submitted through the
Telegram callback flow with each add-on subset:

- none selected;
- exactly one selected;
- all selected.

Use the existing `_custom_spec`, notification, pending-action, and callback helpers in `tests/test_custom_gates.py`.
Keep the test data-driven and assert the persisted neutral `response.json` choice, ordered `selected_extra_ids`, source,
and successful extra results. Exercise the real Telegram toggle/submit path and shared SASE executor rather than calling
the executor directly. Preserve the existing required-feedback, duplicate callback, neutral-HITL, unreadable-envelope,
and plan-toggle coverage. Do not change production code unless the restored matrix exposes a genuine behavioral defect.

Review the main SASE documentation already landed in `3bbcfda69`; it is the canonical Telegram custom-gate documentation
for this feature. Only add plugin-repository documentation if the current docs contain an actual false or missing
user-facing contract, not merely to duplicate the main documentation.

## Phase 2: Revalidate integration and landed behavior

Before landing, repeat the integration scan from the epic creation time (`2026-07-16 23:00:15 -0400`) in the main SASE,
`sase-core`, `sase-telegram`, and chezmoi repositories. Exclude the known epic commits, inspect any newer commits that
appear after this plan was authored, and update consumers if they should use the custom-gate APIs or conflict with them.
Preserve and specifically verify the canonical-plan-identity behavior from `3b263a578` through the remodeled plan extras
path.

Run focused tests that prove the restored matrix and the highest-risk cross-surface contracts: custom gate creation and
execution, CLI wait/cancellation/timeout, mobile projection/execution, plan extras and auto compatibility, ACE tracked
execution/modal behavior, neutral and legacy HITL handling, Telegram formatting/callback/race behavior, generated skill
discovery, and Rust mobile/parity fixtures. Then run each affected repository's complete supported check against the
local SASE/core builds (`just check` for main SASE and `sase-telegram`, and the complete `sase-core` workspace/parity
checks). If the two previously documented Rich terminal-rendering tests still fail, establish that they are unchanged
baseline failures rather than silently treating new failures as unrelated.

Land the recreated `sase-telegram` rollout coverage through the required SASE commit workflow before closing the epic;
do not leave the separate linked repository dirty or rely on an uncommitted handoff.

## Phase 3: Land the epic

This is the final phase and must run only after phases 1-2 are green.

1. Re-read `sase-6i` and all eight children from the canonical SASE plans sidecar and confirm every child remains
   closed.
2. Close the parent with `sase bead close sase-6i` using the canonical SASE bead store.
3. After the close, run `just symvision` in the main SASE repository. Remove every stale `sase-6i` epic-symbol whitelist
   entry and any now-unused code it reports, then rerun Symvision and the affected tests until clean.
4. Change the linked epic plan `202607/custom_notification_gates.md` frontmatter from `status: wip` to `status: done`.
5. Persist the plan-status change through the normal plans-sidecar commit flow; do not leave the canonical sidecar
   dirty. Recheck the closed bead, clean repository states, and the final plan frontmatter before reporting completion.

Preserve unrelated user changes in every repository. Use `sase repo open` for each cross-repository checkout and the
required SASE git commit workflow for all commits.
