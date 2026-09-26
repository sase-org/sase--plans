---
tier: epic
title: Complete creation-reason Context visuals
goal: The created-bead Context card has verified wide and narrow PNG goldens on the
  current Rust binding.
parent_bead: sase-1ap
phases:
- id: visual_runtime
  title: Restore the visual test runtime
  depends_on: []
  description: 'visual_runtime: Rebuild the workspace Rust binding from the linked
    sase-core revision and prove both new Context visual tests reach rendering. Repair
    a real source or binding incompatibility only if a fresh build still fails.'
  size: small
- id: visual_goldens
  title: Capture and inspect the created-bead goldens
  depends_on:
  - visual_runtime
  description: 'visual_goldens: Capture both created-bead Context PNG goldens with
    the targeted maintenance recipe, inspect the report and images, and verify the
    current card behavior after intervening TUI changes. Run just check on any tracked
    source or golden change.'
  size: medium
proposed_by: bbugyi200.athena.sase-1ap.land
create_time: 2026-09-26 14:32:44
status: wip
bead_id: sase-1ap.4
---

- **PROMPT:** [prompts/202609/1ap_visual_completion.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/1ap_visual_completion.md)
- **PARENT:** [202609/bead_creation_reasons.md](https://github.com/sase-org/sase--plans/blob/main/202609/bead_creation_reasons.md)
- **BEAD:** [sase-1ap.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ap/sase-1ap.4.md)

# Remaining work

Epic sase-1ap already landed creation-reason persistence in sase-core, the Python and
CLI/TUI creation flows, and the distinct created/assigned Context presentation. The two
phase-3 visual tests exist, but their `agents_bead_created_by_agent_120x40.png` and
`agents_bead_created_by_agent_90x32.png` goldens do not. Running the two tests on the
current tree fails before rendering: this workspace imports `sase-core-rs` 0.34.71,
whose runner-capacity wire rejects `agent_session`; the linked core checkout and
`sase-core-revision.txt` are at `e579d1d`, with the newer wire. A routine `just install`
rebuilds the development binding from the linked checkout. Avoid changing the wire
merely to accommodate a stale local installation.

## Phase 1: visual_runtime

Run `just install` with the linked `sase-core` checkout open. Confirm the imported
extension comes from that checkout, then rerun both `agents_bead_created_by_agent`
visual tests. Missing-golden comparison is the expected next failure; a startup capacity
error is not. If a fresh binding still fails, trace the actual Python/Rust request
mismatch, fix it in the owning repo, update the SASE revision pin if required, and
verify the affected contract tests. Keep this phase scoped to the runtime needed for
these captures.

## Phase 2: visual_goldens

Use `just fix-tui-screenshots -- -k agents_bead_created_by_agent` to produce both wide
and narrow goldens. Inspect the maintenance manifest and each PNG, including created,
created-plus-closed, assigned, title, and `why:` treatments. Resolve any rendering
defect exposed by the captures. Verify the targeted visual checks and run
`sase tool run check` for tracked changes; do not run `just check-full`.

The parent epic's land agent owns its bead close, Symvision check, and linked-plan
status update after this child epic lands.
