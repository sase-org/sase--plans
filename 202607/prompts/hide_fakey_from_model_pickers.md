- **PLAN:** [../202607/hide_fakey_from_model_pickers.md](../hide_fakey_from_model_pickers.md)

 
%xprompts_enabled:false
# Previous Conversations

You are forking from 1 prior source. Source sections are independent parents, and section order carries no priority. Members inside an agent family section are sequential: each member continued the previous member's work. Carry forward relevant goals, constraints, decisions, and unfinished work with attribution when it matters. The New Query is the active request and takes precedence over conflicting source instructions.

## Source 1 of 1 — agent family `pw`

- **Members shown:** 2 of 2 (sequential chain, oldest first)

Family members ran as one sequential chain: each member continued the previous member's work, and the last member reflects the family's final state. These are transcripts of prior agents' conversations, not your own — attribute decisions to the named member when it matters.

### Member 1 of 2 — agent `pw--plan`

- **Outcome:** `completed` · **Model:** `codex/gpt-5.6-sol` · **Launch:** `20260731065538`
- **Transcript:** `~/.sase/chats/202607/gh_sase_org__sase-ace_run-pw__plan-260731_065538.md`

**User:**

Can you help me remove the `claude-opus-5` and `claude-sonnet-5` models from builtin pickers (`opus` and `sonnet` default to using these)? Also, let's add support for `codex/gpt-5.3-codex-spark` (i.e. show it in pickers). Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.

**Assistant:**

*Plan submitted for review.*

**Plan file:** `/home/bryan/.sase/plans/202607/refresh_builtin_model_catalog.md`

> # Refresh the Builtin Model Catalog
> ## Objective
> Remove the redundant explicit `claude-opus-5` and `claude-sonnet-5` entries from SASE's builtin model pickers while
> keeping the floating Claude `opus` and `sonnet` defaults intact, and register `codex/gpt-5.3-codex-spark` so Spark is
> available in both the modal model picker and `%model` completion.
> ## Current Behavior and Contract
> - Provider `llm_known_model_names()` metadata is the shared source for the modal picker, the `%model` completion catalog
>   (including the JSON snapshot consumed by the Rust xprompt LSP), and automatic provider inference for bare model names.
>   `llm_model_short_aliases()` supplies display/filter hints and same-provider fan-out suffixes.
> - Claude's large/small tier defaults already resolve to the provider-owned floating aliases `opus` and `sonnet`; SASE

*See full plan file for details.*

### Member 2 of 2 — agent `pw--code`

- **Outcome:** `completed` · **Model:** `codex/gpt-5.3-codex-spark` · **Launch:** `20260731065944`
- **Transcript:** `~/.sase/chats/202607/gh_sase_org__sase-ace_run-pw__code-260731_065538.md`

**User:**

@sase/repos/plans/202607/refresh_builtin_model_catalog.md

The above plan has been reviewed and approved. Implement it now.

**Assistant:**

Understood. I’ll open the approved plan at `sase/repos/plans/202607/refresh_builtin_model_catalog.md`, extract the required implementation steps, then apply a single focused patch set for those files only. After that I’ll summarize exactly what changed.

I have the plan scope now: three provider/docs files plus three targeted tests. Next I’ll read those files once, then apply all edits in one patch set.

I’ve got the implementation plan mapped to code and tests. I’m now checking the docs and any shared provider fixtures so the model list stays consistent across docs and completion metadata assumptions.

I’ve identified where docs and tests need to change. I’m checking for any additional tests/fixtures that would fail from removing those two Claude IDs or adding `gpt-53.3` Spark entries before I patch.

I found one likely collateral test in `tests/ace/tui/widgets/test_agent_list_helpers.py` that references `claude-opus-5` as a display shorthand. I’m checking that helper to avoid an avoidable regression by changing it only if necessary.

I confirmed `short_model_name` is keyword-driven and not bound to registry alias metadata, so no collateral change is required there. I’ll now inspect completion plumbing to add a real-registry regression safely without altering the existing fixture-based test coverage.

I’ve applied provider and core-test updates and will now patch picker tests: remove assertions for removed point-version rows, add Spark row checks in Codex, and keep floating `opus`/`sonnet` coverage.

I’ve completed the implementation edits across provider metadata, docs, and targeted tests. I’ll do one sanity pass over the diff to confirm all requested coverage points are covered, then summarize exactly what’s changed.

I’m doing one final static sweep for remaining direct assertions on the removed Claude point-version bare IDs in active tests to ensure no requested behavior is left unintentionally unguarded.

Implemented the plan in the workspace.

### What changed

- Updated built-in Claude metadata to remove explicit point-version IDs from picker/routing sources:
  - Removed `claude-opus-5` and `claude-sonnet-5` from Claude known models.
  - Removed `opus5` and `sonnet5` short-alias entries.
  - Kept `opus`, `sonnet`, `haiku`, `claude-haiku-4-5`, `claude-fable-5`, and tier defaults unchanged.
  - Files:
    - [src/sase/llm_provider/claude.py](/home/bryan/.local/state/sase/workspaces/sase-org/sase/sase_12/src/sase/llm_provider/claude.py)

- Registered Spark in Codex built-in catalog with alias:
  - Added `gpt-5.3-codex-spark` to known models.
  - Added short alias `gpt53spark`.
  - File:
    - [src/sase/llm_provider/codex.py](/home/bryan/.local/state/sase/workspaces/sase-org/sase/sase_12/src/sase/llm_provider/codex.py)

- Tightened/added provider and picker tests:
  - `tests/test_llm_provider_core.py`
    - Explicit syntax still resolves `claude/claude-opus-5` and `claude/claude-sonnet-5`.
    - Bare `claude-opus-5`/`claude-sonnet-5` now resolve as `(None, model)`.
    - Added Spark presence in short-alias map.
    - Added negative checks for removed Claude short aliases in `model_short_alias_map`.
  - `tests/test_model_picker_options.py`
    - Removed expected picker assertions for removed Claude point-version entries.
    - Added assertions for Spark in Codex picker with alias `gpt53spark`.
    - Added explicit absence assertions for removed `claude-opus-5`/`claude-sonnet-5`.
  - `tests/test_xprompt_model_completion.py`
    - Added a real-registry regression test that does not rely on the fixture metadata:
      - Confirms removed point-version IDs are absent.
      - Confirms `gpt-5.3-codex-spark` appears with provider `codex` and alias `gpt53spark`.

- Updated user-facing docs:
  - Clarified Claude floating alias behavior (`opus`/`sonnet`) in `llms.md`.
  - Updated automatic provider table to remove point-version Claude names and add Spark under Codex.
  - Updated short-alias table to remove `opus5`/`sonnet5` and add `gpt53spark`.
  - File:
    - [docs/llms.md](/home/bryan/.local/state/sase/workspaces/sase-org/sase/sase_12/docs/llms.md)

### Validation

I did not run tests/checks in this pass. If you want, I can run the focused targeted test modules and the relevant model-completion snapshot tests next.

I have the exact change set in the six files listed by the finalizer. I’ll commit just those files using the SASE commit flow and then report a concise summary.

Implemented and committed.

Committed as `create_commit` via SASE commit flow with these files:
- `docs/llms.md`
- `src/sase/llm_provider/claude.py`
- `src/sase/llm_provider/codex.py`
- `tests/test_llm_provider_core.py`
- `tests/test_model_picker_options.py`
- `tests/test_xprompt_model_completion.py`

Summary of changes:
- Removed `claude-opus-5` and `claude-sonnet-5` from Claude built-in models and short-alias map.
- Added Spark model in Codex metadata:
  - `gpt-5.3-codex-spark`
  - short alias `gpt53spark`
- Updated docs table + alias map in `docs/llms.md` to match catalog behavior.
- Updated targeted tests for:
  - removed Claude point-version IDs behavior,
  - explicit presence checks for Spark,
  - picker option expectations aligned with new catalog.

Commit completed successfully; I did not run tests/checks in this pass.

---

%xprompts_enabled:true
# New Query

 Also, can you help me remove `fakey` from all model pickers (this is an internal, fake LLM provider used for testing)? Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 