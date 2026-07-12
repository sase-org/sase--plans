---
create_time: 2026-07-11 09:44:44
status: wip
prompt: 202607/prompts/rename_research_c_to_research_lead.md
tier: tale
---
# Plan: Rename the Research-C Model Alias to Research-Lead

## Goal

Replace the personal custom model alias `research_c` with the role-oriented alias `research_lead`, and make the active
research swarm use that alias for its final lead/consolidator agent. Preserve the current model selection and bucket
behavior: `research_lead` will still resolve to `codex/gpt-5.6-sol` and remain a member of the `research` model-alias
bucket.

## Current state and design decision

The live alias is defined only in the chezmoi-managed SASE configuration at `home/dot_config/sase/sase.yml`. It is
currently described as an unused extra/manual lane. The active `home/dot_xprompts/research_swarm.md` workflow instead
uses `research_a` for both the first independent researcher and the final consolidator.

A key-only rename would therefore produce an alias named `research_lead` that is neither described nor used as the lead.
To keep configuration names, user-facing descriptions, and runtime behavior coherent, this change will assign the
renamed alias to the final consolidator segment. Because `research_c` and `research_a` both currently resolve to
`codex/gpt-5.6-sol`, the swarm's concrete provider/model selection will not change; only the semantic alias attached to
the final role will become more precise.

No SASE application code, schema, or model-alias resolution logic needs to change. Alias names are globally flat, and
bucket membership is independent of alias resolution.

## Managed configuration changes

Work in the linked `chezmoi` repository opened with the required `sase workspace open -p chezmoi ...` workflow.

### `home/dot_config/sase/sase.yml`

- Rename the `llm_provider.model_aliases.custom.research_c` key to `research_lead`.
- Preserve its `model: codex/gpt-5.6-sol` target and `bucket: research` membership.
- Rewrite its description to identify it as the research swarm's final lead/consolidator rather than an unused extra
  lane.
- Narrow the `research_a` description to the first/primary independent researcher role, since it will no longer also
  represent the consolidator.
- Keep `research_b` as the assisting/second-opinion researcher.
- Update `llm_provider.model_aliases.buckets.research.description` so the summarized roles are primary researcher,
  second-opinion researcher, and lead/consolidator, removing the stale “extra” wording.
- Preserve alphabetical ordering of the custom aliases and all existing YAML structure.

### `home/dot_xprompts/research_swarm.md`

- Keep the first `research.@.cdx` segment on `@research_a`.
- Keep the second `research.@.cld` segment on `@research_b`.
- Change the final `research.@.final` segment from `@research_a` to `@research_lead`.
- Leave the image segment's explicit `codex/gpt-5.6-sol` model unchanged.

This keeps the workflow topology, waits, group names, prompt body, and concrete models unchanged while giving the final
role its own descriptive alias.

## Reference handling and non-goals

- Do not edit the completed `model_alias_buckets` prompt or plan. Their `research_c` references record the original,
  already-completed migration and are historical evidence rather than live consumers.
- Do not rename synthetic `research_c` fixtures in the SASE unit, interaction, or visual tests. Those fixtures validate
  generic bucket handling and do not depend on Bryan's personal aliases.
- Do not add compatibility aliasing from `research_c` to `research_lead`. The old alias is unused by the current swarm,
  and retaining it would make the rename incomplete and leave four entries in the three-role bucket.
- Do not modify SASE source, schema, default configuration, generated provider shims, or memory files.
- Do not change any provider/model target or add another research agent to the swarm.

## Rollout and verification

1. Re-scan the linked chezmoi sources for live `research_c` consumers before editing. Treat hits under historical SDD
   records as intentionally retained, and investigate any newly discovered operational reference before proceeding.
2. Make the two managed-source edits and run formatting/structural checks appropriate to the chezmoi repository,
   including `git diff --check` and any configured keep-sorted validation.
3. Preview the chezmoi diff for only the managed SASE config and research swarm targets. Confirm the preview removes
   `research_c`, introduces `research_lead`, updates the three role descriptions, and changes only the final swarm model
   directive.
4. Apply only those approved chezmoi targets so the live SASE configuration and xprompt are synchronized with their
   managed sources. Confirm a subsequent targeted chezmoi diff is empty.
5. Validate the effective merged SASE configuration with the current bucket-aware SASE checkout/runtime:
   - `research_a` resolves to `codex/gpt-5.6-sol` and remains in `research`;
   - `research_b` resolves to `claude/opus` and remains in `research`;
   - `research_lead` resolves to `codex/gpt-5.6-sol` and remains in `research`;
   - `research_c` is absent;
   - `sase doctor -C config.model_aliases` reports no alias or dangling-bucket problem.
6. Expand/inspect the effective `research_swarm` xprompt and confirm its three role aliases are `research_a`,
   `research_b`, and `research_lead`, with no operational `research_c` reference.
7. If practical, inspect the Models panel's `research` bucket to confirm it still contains exactly three aliases and the
   renamed row presents the new lead/consolidator description. No SASE `just check` run is required because this plan
   changes only personal chezmoi YAML/Markdown; repository-specific checks and end-to-end configuration validation are
   the relevant gates.

The currently installed global `sase` executable may still be backed by an older canonical checkout that predates
bucket-aware doctor handling and can falsely classify `model_aliases.buckets` as a legacy alias. Use the current
bucket-aware SASE checkout/runtime for the diagnostic above, and report that pre-existing installation skew separately
rather than treating it as a regression from this rename.
