---
tier: tale
title: Migrate plan approval to gate shells
goal:
  Plan review no longer blocks its planner, while preserving exact tale and epic
  continuation behavior.
size: medium
proposed_by: bbugyi200.athena.sase-ud.11
bead: sase-ud.11
---

- **PARENT:** [202608/gate_shells.md](gate_shells.md)
- **BEAD:**
  [sase-ud.11](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ud/sase-ud.11.md)

# Migrate `/sase_plan` approval to gate shells

## Goal

Put both tale and epic plan approval behind the existing `gate_shell_handoff` beta flag.
With the flag disabled, preserve the current blocking approval loop. With the flag
enabled, the runner must create a durable plan gate shell and finish the planner as
`DONE`; the gate shell owns review status and settlement, persists feedback across
runner deaths, and launches exactly the same coder or replanner that the blocking flow
launches today.

## Implementation

1. Add a plan-shell adapter around the existing tiered plan gate specification. Preserve
   all current validation, editable-plan resources, command results, archive protocol
   fields, host-owned epic launch behavior, and automatic approval handling, while
   adding the shell contract for both tiers. Configure tale branches so `approve+commit`
   launches a clean-context coder, `feedback` launches a family-forked replanner, and
   `reject` launches nothing; configure epic approval with no successor. Pin every
   currently hard-coded plan status accent in the shell and branch metadata, including
   failure and terminal outcomes.

2. Persist the state that the blocking runner currently keeps only in `LoopState`.
   Record each plan gate shell's plan path, tier, original/base prompt inputs, previous
   feedback-shell link, feedback text, archive response fields, model-routing inputs,
   workspace/VCS context, and any prompt/archive metadata required by accepted-plan
   processing. Rebuild feedback bullets in order by walking settled plan gate shells, so
   repeated feedback survives process exit and restart without a new store.

3. Move plan-specific settlement behavior onto the gate-shell branch. Validate and
   consume `host_v1`/`host_v2`, `saved_plan_path`, and `plan_archive_ref` exactly as the
   current accepted-plan path does; record `plan_committed`; publish the planner prompt
   archive; and retain the existing degraded epic-launch behavior. Build the settled
   branch's next action from durable state. For approved tales, launch the coder with
   the existing `--code` suffix, byte-identical composed prompt, resolved model and
   model metadata, `fork: none`, and inherited workspace claim. For feedback, launch the
   next plan-family member with the accumulated requirements and `fork: family`.
   Rejection and epic approval must not launch an LLM successor.

4. Branch `handle_plan_marker` on `gate_shell_handoff`. Keep the disabled branch as the
   existing wait-loop implementation. On the enabled branch, adopt the marker, create
   the plan gate shell, record relationships, and delegate terminalization to the common
   gate-marker handler without calling `wait_for_gate`. Preserve the `%auto`
   short-circuit by settling synchronously and continuing in-process with no detached
   agent; use the same accepted-plan and feedback behavior as the disabled branch.
   Remove the wait loop from `handle_plan_approval` on the enabled architecture and keep
   only reusable gate creation/response translation needed by the compatibility branch.

5. Update the generated `/sase_plan` skill source to describe both flag states and the
   gate-shell handoff contract: proposal output precedes termination, enabled review no
   longer keeps the planner alive, the shell owns feedback/approval continuation, and
   agents must not poll or wait. Preview generated deployment with
   `sase skill init --diff`; do not deploy from the dirty workspace.

## Verification

Add focused tests for both flag states and for tale approve, feedback, reject, epic
approve, and `%auto`. Assert the enabled non-auto path never calls `wait_for_gate`, the
auto path creates no second agent, and the approved tale family has exactly the planner,
gate, and `--code` rows. Add golden coverage that the coder prompt and model routing are
byte-identical to the pre-migration flow, plus two- and three-round feedback rebuilds
across simulated runner deaths. Cover valid and invalid `host_v1`/`host_v2` archive
responses, saved-plan containment, `plan_committed`, branch statuses/accents, host-owned
epic launch, and claim inheritance. Run the focused plan-shell and existing
plan-approval suites, preview the generated skill diff, then run `just check`.

Before closing `sase-ud.11`, run `sase bead epic-symbols sase-ud.11` and resolve or
re-key every remaining symbol to an open bead. Close only `sase-ud.11` with a note that
records the tests and end-to-end scenarios actually verified; never close the parent
epic or any ancestor.
