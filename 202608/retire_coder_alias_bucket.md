---
tier: tale
size: medium
title: Retire coder aliases and route tale follow-ups by size
goal:
  Remove the implicit coder alias family from every runtime and user-facing surface, and
  make accepted tale-plan follow-ups consistently use the validated tale size's
  phase-worker alias while preserving explicit approval and custom-prompt model
  overrides.
proposed_by: bbugyi200.athena.sase-il.5
bead: sase-il.5
create_time: 2026-08-10 08:55:06
status: wip
---

- **PARENT:**
  [202608/sase_sizes_memory.md](https://github.com/sase-org/sase--plans/blob/main/202608/sase_sizes_memory.md)
- **BEAD:**
  [sase-il.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-il/sase-il.5.md)

# Plan: Retire coder aliases and route tale follow-ups by size

## Context and invariants

Phase bead `sase-il.5` implements the `coder-alias` phase of the canonical size-routing
epic. The preceding phase has already made tale `size` available through launch-mode
validation, including normalization of legacy sizeless tales to `medium`. This phase
must use that validated value instead of the planner provider when selecting the default
model for a coder follow-up.

The existing precedence remains authoritative:

1. A concrete `coder_model` chosen in the approval picker wins over size-derived routing
   (apart from the legacy `worker` sentinel, which still means no explicit pick).
2. A `%model`/`%m` directive inside a custom coder prompt supersedes both the
   picker/default prefix and supplies the recorded follow-up metadata.
3. With neither override, a tale of size `<size>` launches through
   `@<size>_phase_worker`; launch-mode validation maps a legacy sizeless tale to
   `@medium_phase_worker`.

The word “coder” remains valid for the plan-chain role, UI actions, prompts, and
filenames. Only the implicit `@coder`, registered `@<provider>_coder`, their special
classification/resolution behavior, and the built-in `coders` Models-panel bucket are
retired.

## Implementation

1. Centralize tale-size follow-up routing and apply it to both launch and approval
   display.
   - Reuse the authoritative size-to-phase-worker routing rather than adding a
     provider-derived or duplicate mapping.
   - In `src/sase/axe/run_agent_exec_plan_accept.py`, select the plan path that will
     actually be handed off: the committed SDD tale when a commit succeeded, otherwise
     `plan_result.plan_file`. Validate it as a tale in launch mode, derive
     `%model:@<size>_phase_worker`, resolve its concrete metadata, and preserve
     picker/custom-prompt precedence.
   - Make invalid or unreadable approved plan data fail with an actionable validation
     error instead of silently routing through a removed alias.
   - Pass the plan under review into the custom approval modal via the existing
     `PlanApprovalModal` state. In `src/sase/ace/tui/modals/approve_options_modal.py`,
     derive the default Follow-up label from that tale's launch-validated size, with the
     same phase-worker alias resolution used by runtime routing. Keep Epic's “configured
     by frontmatter” display and explicit picker behavior unchanged.

2. Delete the implicit coder alias family from policy, configuration, resolution,
   completions, and presentation.
   - Remove `coder` from `src/sase/llm_provider/model_alias_defaults.yml` and remove
     coder/provider-coder constants and role registration from `model_alias_policy.py`.
   - Remove provider-coder construction, recognition, name expansion, description, kind,
     and fallback behavior from `model_alias_config.py`, then remove the corresponding
     compatibility re-exports from `config.py` and imports/exports from
     `llm_provider/__init__.py`.
   - Simplify `model_alias_resolution.py` so launch-scoped alias overrides apply only to
     aliases that still exist; remove the special generic-coder fallback for
     provider-specific names.
   - Collapse `AliasKind` to `default | role | user` in `alias_view.py`; remove
     hidden/generated provider-coder rows and the built-in `coders` bucket so
     `BUILTIN_MODEL_ALIAS_BUCKET_NAMES` contains only `phase_worker`. Remove the
     obsolete TUI style and widget prose.
   - Remove `@coder` and generated `@<provider>_coder` entries, ordering, descriptions,
     and generic override propagation from `xprompt/model_completion.py`. Update general
     directive-value examples to use a surviving alias.

3. Provide actionable migration diagnostics for stale user configuration.
   - Extend doctor retired-alias guidance so `coder`, configured `<provider>_coder`
     builtin keys, and alias references to them point users at `@<size>_phase_worker`
     aliases.
   - Detect stale provider-coder keys from raw configuration even though they are no
     longer registered/classified as builtins, while avoiding false positives for
     unrelated custom names ending in `_coder`.
   - Update the older `worker_models` migration message, which currently tells users to
     migrate into the alias family being removed.

4. Remove the retired configuration and user-facing documentation.
   - Update `src/sase/default_config.yml` and `src/sase/config/sase.schema.json` to
     remove `coder`, `<provider>_coder`, and `coders` examples/descriptions while
     retaining the `phase_worker` bucket and size-specific aliases.
   - Update the maintained alias-family and plan-handoff sections in
     `docs/configuration.md`, `docs/llms.md`, `docs/ace.md`, and `docs/xprompt.md` to
     describe size-derived tale routing, the legacy sizeless `medium` behavior where
     relevant, and explicit model override precedence. Adjust `docs/fakey.md` because
     its hidden provider-coder example no longer exists.
   - Review `docs/cli.md`, `docs/sdd.md`, and `docs/integrations.md`; retain
     role-language such as “run coder” where it describes the follow-up action rather
     than the retired alias mechanism. Leave historical blog posts untouched.

5. Rewrite tests around the surviving contract and remove coder-bucket fixtures.
   - Change plan-follow-up launch and metadata tests to use valid sized tale plans,
     cover each valid tale size, cover legacy sizeless normalization to
     `@medium_phase_worker`, and prove committed versus archived plan-path selection.
     Retain regression coverage for picker and custom-prompt `%model` precedence.
   - Change approval-modal tests to verify the displayed default matches the plan's
     size-derived phase-worker model and that explicit/Epic displays are unchanged.
   - Update model-policy, alias resolution/config, completion, picker, Models-panel
     navigation/rendering/bucket, config-schema, and visual snapshot fixtures to assert
     coder/provider-coder aliases and the `coders` bucket are absent while the five
     phase-worker aliases remain.
   - Add doctor tests for configured `coder`, registered `<provider>_coder`, stale
     references, and the new migration text, plus coverage ensuring ordinary user
     aliases are not misclassified.
   - Update unrelated fixtures that used `@coder` merely as a convenient valid model
     alias to use an existing phase-worker or other surviving alias. Preserve strings
     such as historical agent names (`claude_coder`) when they are identity data rather
     than model aliases.

## Verification

1. Run `just install` before repository checks because the workspace environment is
   ephemeral.
2. Run focused tests for plan follow-up routing/metadata, approval-modal model display,
   alias policy/resolution/completion/Models panel, config schema, and doctor migrations
   while iterating.
3. Run repository-wide searches for the retired constants/functions/kind/bucket and for
   documented `@coder` / registered `@<provider>_coder` behavior. Classify any remaining
   `coder` text as legitimate role/identity/history language or remove it.
4. Run `just check-full`, required because this phase changes the broadening set (alias
   registry, configuration schema, TUI surfaces, docs, and visual fixtures). If
   intentional ACE PNG output changes, inspect the generated actual/expected/diff
   artifacts before updating snapshots and rerun the dedicated visual suite.
5. Recheck `git diff --check`, inspect the final diff/status for accidental generated or
   sidecar changes, and close only `sase-il.5` with a note naming the targeted and
   full-suite verification that passed.
