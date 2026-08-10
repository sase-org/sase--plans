---
tier: epic
title: Finish core-owned tale size semantics and land sase-il
goal: "sase-core owns the full tale-size contract the sase-il design specified — tales
  accept only xsmall/small/medium, launch mode normalizes legacy tales instead of
  failing, and the size field descriptions point at sase/memory/sase_sizes.md instead of
  restating it — so the generated size note is genuinely the single source of size
  truth, and epic sase-il can then be closed out.

  "
phases:
  - id: core-tale-size-contract
    title: Complete the tale size contract in sase-core
    depends_on: []
    size: medium
    description:
      "core-tale-size-contract: restrict tale `size` to xsmall/small/medium, add the
      launch-mode normalization the design specified, reduce the size/model field
      descriptions to pointers at the canonical size memory note, and cut a sase-core
      release."
  - id: sase-adopt-contract
    title: Adopt the completed contract in sase
    depends_on:
      - core-tale-size-contract
    size: medium
    description:
      "sase-adopt-contract: raise the sase-core-rs floor to the new release, delete the
      Python launch-compatibility shim now that core owns it, migrate the 21 committed
      tale plans the sase-il backfill over-sized to `large`, and cover the completed
      contract with tests."
  - id: land-sase-il
    title: Land and close epic sase-il
    depends_on:
      - sase-adopt-contract
    size: medium
    description:
      "land-sase-il: run the full gate, close epic sase-il with a verification note,
      clear any expired sase-il symvision whitelist entries, and mark the
      sase_sizes_memory plan file done."
proposed_by: bbugyi200.athena.sase-il.land
parent_bead: sase-il
create_time: 2026-08-10 10:54:04
status: wip
---

- **PROMPT:**
  [prompts/202608/finish_tale_size_semantics.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/finish_tale_size_semantics.md)
- **PARENT:**
  [202608/sase_sizes_memory.md](https://github.com/sase-org/sase--plans/blob/main/202608/sase_sizes_memory.md)

# Plan: Finish core-owned tale size semantics and land sase-il

Epic `sase-il` made one generated memory note (`sase/memory/sase_sizes.md`) the
canonical source of sase-size truth, gave tale plans a required `size`, and retired the
coder alias bucket. Landing verification confirmed five of six phases are complete and
correct. Phase `sase-il.3` (`core-tale-size`), however, shipped a partial contract in
`sase-core`, and `sase-il.4` recorded the gap as a `PROPOSED FOLLOW-UP` rather than
closing it. Because the epic's own memory note and `/sase_plan` skill already publish
the stricter rule, the repo currently documents one contract and enforces another.

Three concrete gaps remain, all in `../sase-core/crates/sase_core/src/plan/validate.rs`
plus its sase-side adapter. Open the core checkout with `/sase_repo`
(`sase repo open sase-core`) and use only the path it prints.

**Gap 1 — tales accept `large` and `xlarge`.** `validate_tale_size` calls the shared
`is_plan_size` helper, which matches all five sizes. Verified directly against the
installed 0.23.0 binding: a tale declaring `size: large` validates `ok=True` with no
diagnostics. `sase/memory/sase_sizes.md` and `src/sase/xprompts/skills/sase_plan.md`
both state that a tale must be `xsmall | small | medium` and that larger work belongs in
an epic. The design said core must reject `large`/`xlarge` with a message explaining
that a tale is single-agent work.

**Gap 2 — the size field descriptions still restate the taxonomy.** The design's central
goal was that one note owns every size instruction. `PHASE_SIZE_DESCRIPTION` still
carries the complete per-size explanation plus the plan-first rule plus the five alias
names; `PHASE_MODEL_DESCRIPTION` restates size-override behaviour; and
`TALE_SIZE_DESCRIPTION` tells the agent to "use the same five-step size taxonomy", which
becomes actively wrong once Gap 1 is fixed. Nothing in the crate mentions
`sase/memory/sase_sizes.md`. These descriptions are agent-facing:
`plan_frontmatter_schema` flows through `src/sase/main/plan_validate_handler.py`,
`src/sase/main/plan_propose_handler.py`, `src/sase/_plan_approval_protocol.py`, and
`src/sase/bead/cli_work_from_plan_render.py`, and
`src/sase/main/plan_validate_render.py` renders `field.description` verbatim whenever a
plan fails validation.

**Gap 3 — launch-mode tale normalization lives in Python.** The design said `Launch`
mode must report a missing tale `size` as a warning and normalize it to `medium`.
`validate_tale_size` has no `PlanValidationMode::Launch` branch and errors in every mode
(verified: `plan_validate(sizeless_tale, "tale", "launch")` returns `ok=False` with
`tale-size-missing` at error severity). `sase-il.4` compensated with
`_launch_mode_compatibility_content` in `src/sase/sdd/plan_validate.py`, which rewrites
the frontmatter to inject `size: medium` before calling the binding and synthesizes the
warning diagnostic in Python. Its own docstring calls this a stopgap "until older core
wheels catch up". This is shared backend behaviour that any frontend must match, so it
belongs in core per the Rust core backend boundary rule.

## Complete the tale size contract in sase-core

Work in the `sase-core` checkout from `/sase_repo`. All of this lives in
`crates/sase_core/src/plan/validate.rs` unless noted.

1. **Narrow the accepted tale sizes.** Introduce a tale-specific accepted-size predicate
   (`xsmall`, `small`, `medium`) rather than reusing `is_plan_size`, which must keep
   accepting all five for phases. Keep the existing `tale-size-invalid` code. Give the
   out-of-range case its own message: name the offending value, state that a tale is
   work one follow-up agent implements directly, and say that `large` and `xlarge` work
   belongs in an epic plan. Keep the wrong-YAML-type branch reporting the accepted set
   too. Update `SIZE_FIELD_TYPE` usage so the tale schema entry advertises
   `xsmall | small | medium` while the phase entry keeps all five — introduce a separate
   constant instead of reusing the shared one.

2. **Add the launch-mode branch.** Mirror the existing phase-size handling around
   `PlanValidationMode::Launch`:
   - missing tale `size` → `warning` `tale-size-missing`, normalized to `medium`;
   - `large`/`xlarge` tale `size` → `warning` `tale-size-invalid`, normalized to
     `medium`;
   - `Authoring` keeps reporting both as errors. `medium` is the right normalization
     target both ways: it is the largest size a tale may declare, so a legacy or
     over-sized tale keeps at least today's follow-up capability instead of silently
     downgrading. Use warning text that says the tale is being treated as `medium` for
     launch, matching the wording the Python shim used so existing callers read the same
     way.

3. **Reduce the three descriptions to pointers.** Cut `PHASE_SIZE_DESCRIPTION`,
   `PHASE_MODEL_DESCRIPTION`, and `TALE_SIZE_DESCRIPTION` to one line each: the accepted
   values (or, for `model`, that an explicit model overrides size-derived routing) plus
   an instruction to read `sase/memory/sase_sizes.md` with the `/sase_memory_read`
   skill. Do not restate per-size meanings, the plan-first rule, or the alias names —
   that is exactly the duplication the note replaced. Update the in-module tests that
   assert on these constants, including the one asserting `@medium_phase_worker` appears
   in `PHASE_SIZE_DESCRIPTION`, and `crates/sase_core/tests/plan_validate_parity.rs`.

4. **Update fixtures and tests.** `tale_size_is_required_strict_and_normalized`
   currently loops `["xsmall", "small", "medium", "large", "xlarge"]` as valid tale
   sizes; split it into an accepted-set loop and a rejected-set loop. Add launch-mode
   coverage for both the missing and the over-sized case, asserting warning severity and
   the normalized `medium` on the returned wire plan. Sweep `crates/sase_core` and
   `crates/sase_gateway` for tale fixtures declaring `large`/`xlarge` and fix them.

5. **Decide the wire version deliberately.** `PLAN_WIRE_SCHEMA_VERSION` is 3 and the
   wire shape does not change here — `size` is already `Option<String>`. Only validation
   rules and descriptions change. Do not bump it unless something in the payload shape
   actually changes; if you do bump it, update every consumer and parity test in both
   repos and say so in the phase notes so the next phase can match.

Run that repo's own check target, then land the change and cut a release. The narrowing
is breaking for callers that accepted `large` tales, so pick the version accordingly and
record the released version in this phase's completion notes — the next phase needs it.

## Adopt the completed contract in sase

1. **Raise the floor.** Move the `sase-core-rs` window in `pyproject.toml` to the
   release from the previous phase, following the precedent of the prior bumps
   (`8ed11bb80`), and refresh `uv.lock`.

2. **Delete the Python shim.** Remove `_launch_mode_compatibility_content` and
   `_LEGACY_TALE_LAUNCH_SIZE` from `src/sase/sdd/plan_validate.py`, and simplify
   `validate_plan` back to a single binding call with no diagnostic splicing now that
   the binding returns the warning itself. Check whether
   `_content_for_core_plan_validator` is still the only remaining content rewrite and
   leave it alone if so.

3. **Migrate the over-sized committed tales.** Twenty-one archived tale plans under
   `202608/` in the plans sidecar declare `size: large` and would fail authoring-mode
   committed-plan validation once core narrows. Every one of them was synthesized by
   `sase-il.3`'s own backfill commit `97845e6e` ("chore(plans): backfill tale sizes for
   committed plans"), not authored by the agent that wrote the plan — verified by
   diffing the backfill's file list against the current `large` set, which found zero
   independently authored `large` tales. Correct them to `medium`, which is what the
   backfill should have assigned to a single-agent tale, and commit the correction to
   the plans sidecar with a message that says the backfill over-assigned. The affected
   files are `ace_app_boot_amortization`, `ace_patch_terminology`, `agents_reload_cost`,
   `clear_ace_tui_test_surface`, `complete_python_patch_storage`, `core_ref_contract`,
   `event_driven_tui_waits`, `gate_inputs_ace`, `gate_inputs_ace_1`, `gate_inputs_core`,
   `gate_inputs_remote`, `gate_inputs_telegram`, `patch_stitch_compatibility_audit`,
   `patch_tui_config_surface`, `patch_workflow_contracts`, `python_patch_storage`,
   `python_ref_registry`, `python_ref_registry_1`, `python_ref_registry_2`,
   `rust_prebuild_cache`, and `workflows_cli_terminology`. Re-derive the list rather
   than trusting it verbatim; more plans may have been archived by the time this runs.

4. **Cover the completed contract.** Extend `tests/test_plan_validate.py` and the plan
   gate/approval tests so they assert against the real binding: authoring mode rejects a
   `large` tale with `tale-size-invalid`; launch mode accepts both a sizeless tale and a
   `large` tale with a warning and a normalized `medium` on the validated plan; and
   `require_plan_approval_validation` still admits both, so
   `validated_tale_followup_model_directive` in `src/sase/tale_followup_routing.py`
   routes them to `@medium_phase_worker`. Keep the existing strict-authoring cases from
   `b9008c535` passing.

5. **Verify the guidance now matches enforcement.** `sase/memory/sase_sizes.md`,
   `src/sase/xprompts/skills/sase_plan.md`, and `src/sase/main/plan_explain.py` already
   state the three-size tale rule, so no wording change should be needed. Confirm that,
   and confirm `sase plan validate --explain` output for a tale now shows the shortened
   descriptions and the `xsmall | small | medium` field type. If any of those three
   files does need a change, note that `sase/memory/sase_sizes.md` is generated: edit
   `src/sase/main/init_memory/templates/memory-sase-sizes.template.md` and run
   `sase memory init`, never the canonical note.

Run `just install` first, then `just check`. Run `just validate-committed-plans`
explicitly after the sidecar migration.

## Land and close epic sase-il

This phase finishes the landing that this plan interrupted. Everything below was already
verified against master at `344a0b8ff` and needs only re-confirmation on the combined
tree.

1. Run `just install`, then `just check-full`.

2. Close the epic:

   ```bash
   sase bead close sase-il --note "<what was verified>"
   ```

   The note should record that all six phases were verified against their commits
   (`2f71b6bc4`, `f21c8d850`, `46fbdc07a`, `f42a68c07`, `b9008c535`, `344a0b8ff` plus
   the core commit `3c10a0c` and release `119495d`), that the three `tale-size-missing`
   committed-plan `DISCOVERED ISSUE` notes on the epic are resolved (validation now
   passes for every committed plan), and that this plan completed the core-owned tale
   size semantics that `sase-il.4` had recorded as a `PROPOSED FOLLOW-UP`.

3. After the close succeeds, run `just symvision`. Epic-symbol whitelist entries for
   `sase-il` expire at close, so remove any stale entries and the unused code it
   reports. Read `sase/memory/symvision.md` with `/sase_memory_read` before touching
   whitelists.

4. Set `status: done` in the frontmatter of `plans:202608/sase_sizes_memory.md`, and
   also mark this plan file done.

## Verification

Every phase runs `just install` first, because workspace virtualenvs are ephemeral. The
sase-core phase runs that repo's own check target inside the checkout `/sase_repo`
prepared. The `sase-adopt-contract` phase runs `just check` plus an explicit
`just validate-committed-plans`. The `land-sase-il` phase runs `just check-full`.

Broad ACE/Textual/contract full-suite failures reported by several `sase-il` phases are
already tracked on task beads `sase-iv`, `sase-iq`, `sase-ct`, and `sase-ii`; treat
those specific failures as known rather than as regressions from this plan, but confirm
the failure signature matches before dismissing anything.
