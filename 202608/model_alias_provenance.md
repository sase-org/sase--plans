---
tier: tale
title: Show the model alias an agent was launched with in the Model field
goal:
  When an agent is launched with `%model:@<alias>`, every SASE surface that renders the
  uniform `Model:` field appends a launch-time provenance chip — `← @<alias>` — so the
  alias the user actually typed is visible next to the provider, model, and effort it
  resolved to.
size: medium
proposed_by: bbugyi200.athena.xm
create_time: 2026-08-10 15:44:48
status: wip
---

# Show the model alias an agent was launched with in the `Model:` field

## Problem

`%model:@medium_worker` and `%model:claude/opus` produce identical agent metadata. The
`@` prefix is stripped in `normalize_model_directive`
(`src/sase/xprompt/_directive_values.py:163`), the bare alias is resolved to a concrete
`(provider, model, effort)` triple in `run_agent_directives.py:272-291`, and only that
triple is written to `agent_meta.json`. The alias name is discarded at launch and is
recoverable from nowhere afterwards.

So the `Model:` field renders

```text
Model: CLAUDE(opus) @ xhigh
```

for both launches, and the user cannot tell:

- whether they asked for a specific model or delegated the choice to an alias,
- **which** alias delegated it (`@default`? `@medium_worker`? a custom `@fast`?),
- why two agents launched minutes apart from the same alias ended up on different models
  (a round-robin pool advanced, or a temporary override expired).

This matters most where the alias is chosen _for_ the user rather than by them:
`phase_model_directive_value` / `task_model_directive_value`
(`src/sase/bead/work.py:306-332`) route every phase agent, task-bead worker, and
approved-tale coder follow-up through `%m:@<size>_worker`. Those agents' `Model:` fields
currently show a bare concrete model with no hint that a size-derived alias picked it.

## Desired behavior

The uniform `Model:` value grows one optional trailing chip:

```text
Model: CLAUDE(opus) @ xhigh ← @medium_worker
```

and, in a family container's per-member lanes, one chip per lane that used an alias:

```text
Model: --plan     · CLAUDE(opus) @ xhigh ← @large_worker
       --code     · CLAUDE(sonnet) @ high ← @medium_worker
       --reviewer · CODEX(gpt-5.2) @ medium
```

Contract:

- The chip appears **only** when the launch's `%model` argument was written as
  `@<alias>` (including when the `@`-prefixed value came from an xprompt reference, as
  in `%model:@#agy_flash`). A concrete `%model:claude/opus`, a `%model("literal")`
  literal, and an agent with no `%model` at all render exactly as they do today.
- The chip is a **launch-time fact**, recorded once in `agent_meta.json` and rendered
  verbatim. It is never re-resolved at render time, so it keeps telling the truth about
  a completed run after the alias is retargeted, overridden, or deleted.
- The chip rides on the value, not the label, so every surface that already shares
  `model_label.py` gets it in one change: the ACE agent detail header (single-line and
  per-member family lanes), the workflow-step detail header, the agent run-log modal,
  and `sase agent show`.
- Agents launched before this change have no `model_alias` in their metadata and render
  unchanged. Absence is never an error.

### Why this shape

- **Chip after the value, not before it.** `@medium_worker → CLAUDE(opus)` reads
  naturally, but the family lanes deliberately align every member's value in one column
  (`tests/ace/tui/widgets/test_agent_model_section.py` asserts a single `·` column), and
  a variable-width alias prefix would destroy that. The concrete provider/model stays
  the anchor at a stable position; provenance is a suffix, exactly where the Models
  panel already puts its provenance/state column.
- **`←`, not `→`.** `append_alias_reference` (`src/sase/ace/tui/model_alias_styles.py`)
  already owns ` → @name` meaning "resolves to". The chip means the opposite direction —
  "came from" — so it gets the mirrored glyph and the same two styles. `←` (U+2190) is
  covered by the bundled snapshot fonts.
- **Same styling as an alias reference elsewhere.** A user should learn "dim connective
  plus bold light-blue `@word` = a model alias" once and have it hold in the Models
  panel, in completion rows, and here. Inventing a second visual treatment for the same
  concept is what makes a UI feel bolted together.
- **One chip, one meaning.** The chip says "the `%model` argument was this alias".
  Whatever the alias contributed — provider, model, and possibly the effort via an
  alias-borne `@<effort>` suffix — rides on that one statement. See Non-goals for why
  effort provenance is not separately marked.

## Relevant code map

Capture (alias is currently dropped here):

- `src/sase/xprompt/_directive_values.py` — `normalize_model_directive` (strips `@` and
  reports `model_had_alias_prefix`), `_validate_model_alias_prefix` (validates the
  post-expansion alias name).
- `src/sase/xprompt/_directive_extract.py:93-176` — builds `PromptDirectives`; owns both
  `model_had_alias_prefix` and the fully expanded `expanded_args["model"]`.
- `src/sase/xprompt/_directive_types.py` — `PromptDirectives`.
- `src/sase/axe/run_agent_directives.py:258-320` — resolves the launch triple and builds
  `AgentMetadataInputs`; also the re-exec branch that reuses preserved metadata.
- `src/sase/axe/run_agent_directive_metadata.py` — `AgentMetadataInputs`,
  `build_agent_meta`, `preserved_agent_metadata`.
- `src/sase/axe/run_agent_exec_plan_accept.py:119-215` — `_FollowupModel` /
  `_resolve_followup_model` / `_write_followup_model_meta`, which **rewrite** `model`
  and `llm_provider` in the current artifacts dir when an approved plan's coder
  follow-up routes through `%model:@<size>_worker`.
- `src/sase/xprompt/workflow_executor_steps_prompt.py:264-306` and
  `workflow_executor.py::_save_prompt_step_marker` — the parallel resolution/record site
  for workflow `agent:` steps.

Transport (Rust-backed; the TUI fast path reads the snapshot/index, not the JSON):

- `sase-core`: `crates/sase_core/src/agent_scan/wire.rs` (`AgentMetaWire`,
  `PromptStepMarkerWire`), `agent_scan/scanner.rs` (marker parsing),
  `agent_scan/index.rs` (`AGENT_ARTIFACT_INDEX_SCHEMA_VERSION`, `record_json` refresh
  migrations).
- `src/sase/core/agent_scan_wire_markers.py` — Python mirrors of those wire records.
  `known_field_kwargs` already drops unknown keys, so the mirrors tolerate both an older
  and a newer Rust binding.
- `src/sase/ace/tui/models/_loaders/_meta_enrichment_filesystem.py` and
  `_meta_enrichment_wire.py` — the two enrichment paths that must stay field-for-field
  identical.
- `src/sase/ace/tui/models/_loaders/_workflow_step_loaders.py` and
  `_workflow_snapshot_loaders.py` — workflow-step rows.
- `src/sase/ace/tui/models/_agent_state.py` — `Agent` dataclass fields.
  `agent_bundle.py` walks `dataclasses.fields`, so a new field bundles automatically.

Render:

- `src/sase/llm_provider/model_label.py` — `model_value_text` / `append_model_field`,
  the single shared renderer. Intentionally free of Textual imports.
- `src/sase/ace/tui/model_alias_styles.py` — the alias presentation vocabulary
  (`append_alias_reference`, `_REFERENCE_TAG_STYLE`, `_IMPLICIT_TAG_STYLE`).
- Call sites: `prompt_panel/_agent_display_header_metadata.py`,
  `prompt_panel/_agent_model_section.py`, `prompt_panel/_workflow_render.py`,
  `modals/agent_run_log_modal.py`, `src/sase/agents/cli_show.py`.

## Implementation

### 1. `sase-core`: carry `model_alias` through the scan wire and the index

Open the repo the sanctioned way and use the printed path for every read and write:

```bash
sase repo open sase-core -r "Add model_alias to the agent-scan wire and bump the artifact index schema"
```

1. `crates/sase_core/src/agent_scan/wire.rs`: add
   `#[serde(default)] pub model_alias: Option<String>,` to `AgentMetaWire` immediately
   after `reasoning_effort`, and the same field to `PromptStepMarkerWire` immediately
   after its `reasoning_effort`. Both structs already `#[serde(default)]` every optional
   field, so old `record_json` blobs deserialize unchanged.
2. `crates/sase_core/src/agent_scan/scanner.rs`: parse it in both marker builders —
   `model_alias: coerce_str(data.get("model_alias")),` beside the existing
   `reasoning_effort` line in the `agent_meta.json` builder (~line 1043) and in the
   `prompt_step_*.json` builder (~line 1247).
3. `crates/sase_core/src/agent_scan/index.rs`: bump
   `AGENT_ARTIFACT_INDEX_SCHEMA_VERSION` from `19` to `20`, add a
   `migrate_record_json_refresh_v20` with no DDL (mirroring
   `migrate_record_json_refresh_v19`, whose doc comment is the template), and dispatch
   it under `if prior_version.map_or(true, |v| v < 20)`. The doc comment must say that
   v20 adds `agent_meta.model_alias` and `prompt_steps[*].model_alias` to `record_json`
   so the ACE `Model:` field can render launch-time alias provenance. Do **not** bump
   `AGENT_SCAN_WIRE_SCHEMA_VERSION`: this is a purely additive field and
   `agent_scan_wire_conversion.py` rejects any mismatch of that constant.
4. `crates/sase_core/tests/agent_scan_parity.rs`: add `model_alias` to the
   `agent_meta.json` and prompt-step fixtures and assert it round-trips through the
   scanner and through an index rebuild.
5. Do not hand-edit `crates/sase_core/CHANGELOG.md`; it is generated by `release-plz`
   from conventional commits.

Python picks the new field up automatically once `just install` rebuilds `sase_core_rs`
from the checkout. A stale binding simply yields `None` and no chip — do not add a
version guard for that, and do not touch the `sase-core-rs` floor in `pyproject.toml`
(the release-branch reconciler ratchets it).

### 2. Capture the alias at launch

1. `src/sase/xprompt/_directive_types.py`: add `model_alias: str | None = None` to
   `PromptDirectives` with a docstring entry stating that it holds the bare alias name
   (no `@`) when, and only when, the `%model` argument was written with the `@` alias
   prefix, and `None` otherwise.
2. `src/sase/xprompt/_directive_extract.py`: in the `PromptDirectives(...)`
   construction, pass
   `model_alias=(expanded_args.get("model") or None) if model_had_alias_prefix else None`.
   Take the value **after** `expand_single_directive_args`, so an xprompt-expanded alias
   (`%model:@#agy_flash`) records the resolved alias name rather than the reference
   text. `_validate_model_alias_prefix` has already proven the name is a known alias by
   that point, so no extra validation is needed.
3. `src/sase/axe/run_agent_directive_metadata.py`:
   - add `model_alias: str | None` to `AgentMetadataInputs`;
   - in `build_agent_meta`, write `agent_meta["model_alias"] = inputs.model_alias` under
     the existing `if inputs.model:` block's sibling guard (`if inputs.model_alias:`),
     next to the `model` / `llm_provider` / `reasoning_effort` writes;
   - add `"model_alias"` to the preserved-key tuple in `preserved_agent_metadata`. This
     is required for correctness, not cosmetics: a runner re-exec reuses the preserved
     `model`/`llm_provider` verbatim, so dropping the alias would make the chip vanish
     from a resumed agent.
4. `src/sase/axe/run_agent_directives.py`: thread the alias into `AgentMetadataInputs`.
   In the `reused_selection` branch take `preserved_metadata.get("model_alias")`
   (guarded with `isinstance(..., str)` like its neighbors); otherwise take
   `directives.model_alias` — and only when `directives.model` was actually used, so the
   implicit-default branch records nothing.
5. `src/sase/axe/run_agent_exec_plan_accept.py`: keep the rewritten metadata internally
   consistent. Add a `model_alias: str | None = None` field to `_FollowupModel`;
   populate it in `_resolve_tale_size_followup` and in the explicit-`coder_model` branch
   of `_resolve_followup_model` by running the directive value through
   `normalize_model_alias_reference` (`sase.llm_provider.config`), which returns the
   bare alias or `None`. In `_write_followup_model_meta`, whenever `followup.meta`
   causes a `model` rewrite, also write `model_alias` when the follow-up routed through
   an alias and **clear** any inherited `model_alias` when it did not. Without this, a
   coder follow-up would inherit the planner's alias chip beside a different model — an
   actively wrong label, which is worse than no label.
6. `src/sase/xprompt/workflow_executor_steps_prompt.py` and
   `workflow_executor.py::_save_prompt_step_marker` (plus the abstract stub in
   `workflow_executor_steps_script.py`): add a `model_alias` parameter and record
   `effective_directives.model_alias` on the step marker, mirroring how
   `reasoning_effort` is recorded today.

Do not touch `runner_artifacts.write_agent_meta` or the axe hook runners (mentor,
fix-hook, crs, summarize-hook): they resolve concrete models from config rather than
from a `%model` directive and have no alias to report.

### 3. Plumb it to the `Agent` model

1. `src/sase/core/agent_scan_wire_markers.py`: add `model_alias: str | None = None` to
   `AgentMetaWire` (after `reasoning_effort`) and to `PromptStepMarkerWire` (after its
   `reasoning_effort`).
2. `src/sase/ace/tui/models/_agent_state.py`: add `model_alias: str | None = None` next
   to `model` / `llm_provider` / `reasoning_effort`, with a comment noting it is the
   bare launch-time alias name recorded when `%model:@<alias>` was used, rendered as a
   provenance chip and never re-resolved.
3. Mirror the read in **both** enrichment paths, keeping them field-for-field identical:
   `_meta_enrichment_filesystem.py`
   (`if data.get("model_alias"): agent.model_alias = ...`) and
   `_meta_enrichment_wire.py` (`if meta.model_alias: ...`).
4. Pass `model_alias` through to workflow-step `Agent`s in `_workflow_step_loaders.py`
   and `_workflow_snapshot_loaders.py`, beside the existing `reasoning_effort` argument.

### 4. Render the chip

1. `src/sase/llm_provider/model_label.py`:
   - Add two module-level constants that own the alias-chip vocabulary at the lowest
     layer that needs it: `MODEL_ALIAS_REFERENCE_STYLE = "bold #87D7FF"` and
     `MODEL_ALIAS_CONNECTIVE_STYLE = "dim #9E9E9E"`.
   - Add a `model_alias: str | None = None` keyword parameter to `model_value_text` and
     to `append_model_field`, both appended after `reasoning_effort` so every existing
     positional call site is unaffected.
   - In `model_value_text`, after the reasoning-effort suffix, append `" ← "` in the
     connective style and `f"@{model_alias.lstrip('@')}"` in the reference style when
     `model_alias` is truthy. Keep this a private module helper
     (`_append_model_alias_provenance`) so Symvision does not see an unused public
     symbol. The chip goes last, after the advisory glyph and the effort suffix.
   - Extend the module and function docstrings to describe the new
     `PROVIDER(model) @ <effort> ← @<alias>` shape and state the never-re-resolved rule.
2. `src/sase/ace/tui/model_alias_styles.py`: import the two new constants from
   `sase.llm_provider.model_label` and use them as the values of `_REFERENCE_TAG_STYLE`
   and a new `_PROVENANCE_CONNECTIVE_STYLE`, so the alias-reference color has exactly
   one definition. Leave `_IMPLICIT_TAG_STYLE` in place — it is used for other things
   (the `implicit` provenance tag and the pool-chip separator) and only happens to share
   a value. The dependency direction (`sase.ace.tui` → `sase.llm_provider`) is the
   existing one; do not import `sase.ace.*` from `llm_provider`.
3. Pass the alias at every call site:
   - `prompt_panel/_agent_display_header_metadata.py::_append_model_fields` →
     `append_model_field(text, agent.model, agent.llm_provider, agent.reasoning_effort, agent.model_alias)`
   - `prompt_panel/_agent_model_section.py::build_family_model_lanes` →
     `model_value_text(member.model, member.llm_provider, member.reasoning_effort, member.model_alias)`
   - `prompt_panel/_workflow_render.py` and `modals/agent_run_log_modal.py` → pass
     `agent.model_alias`
   - `src/sase/agents/cli_show.py` → pass `_optional_str(meta.get("model_alias"))`
4. Change nothing about agent list rows, `sase/agents_sync/rendering_agent_page.py`, or
   the mobile/integration agent summaries (see Non-goals). The metadata is now present
   for a later follow-up if it is ever wanted there.

### 5. Tests

Add focused tests rather than broad new suites; extend the existing modules that already
own each seam.

1. Directive extraction (`tests/test_directives_extract.py` /
   `tests/test_directives_split_models.py`): `%model:@medium_worker` sets
   `model_alias == "medium_worker"` and `model == "medium_worker"`; `%model:claude/opus`
   and `%model("literal")` set `model_alias is None`; `%model:@medium_worker@high`
   records the alias and the effort independently; an `@`-prefixed xprompt reference
   records the expanded alias name; each branch of a `%{%m:@a | %m:opus}` fan-out
   records its own value.
2. Launch metadata (`tests/test_run_agent_directive_metadata.py`,
   `tests/test_reasoning_effort_metadata_display.py` as the shape precedent):
   `agent_meta.json` gains `model_alias` for an alias launch, omits it for a concrete
   launch and for a no-`%model` launch, and `preserved_agent_metadata` carries it across
   a re-exec. Add a case where a launch-scoped `%model(medium_worker=...)` override
   retargets the alias: the recorded `model` follows the override while `model_alias`
   still reads `medium_worker`.
3. Plan-accept rewrite (`tests/test_axe_run_agent_exec_plan_followup_artifacts.py` or
   the nearest existing module): an approved tale routing through `@<size>_worker`
   records that alias, and an explicit concrete `coder_model` clears an inherited
   `model_alias`.
4. Renderer (`tests/test_reasoning_effort_metadata_display.py` and
   `tests/ace/tui/widgets/test_agent_display_name_model_metadata.py`): the chip's plain
   text and styles for one provider; no chip when `model_alias` is `None`; the chip is
   uniform across providers exactly as the effort suffix is; the chip follows both the
   advisory glyph and the effort suffix. Add one test asserting
   `MODEL_ALIAS_REFERENCE_STYLE` is the style
   `model_alias_styles.append_alias_reference` uses, so the two surfaces cannot drift
   apart.
5. Family lanes (`tests/ace/tui/widgets/test_agent_model_section.py`): a mixed family
   where some members used aliases keeps one aligned `·` column, and a long chip folds
   under the value column instead of widening the panel.
6. Loader parity: assert both `_meta_enrichment_filesystem` and `_meta_enrichment_wire`
   populate `Agent.model_alias` from equivalent inputs, in whichever existing enrichment
   parity test module covers `reasoning_effort`.

### 6. Docs

- `docs/agent_families.md` (per-member model lanes, ~line 373-393): update the example
  block to show a lane with a chip and add a sentence defining `← @<alias>` as
  launch-time provenance that is never re-resolved.
- `docs/llms.md` model-alias section: document that an agent launched through an alias
  reports it in its `Model:` field, and that the chip reflects the alias named at launch
  rather than the alias's current target.
- `docs/xprompt.md` `%model` row/section: note that the `@<alias>` spelling is retained
  and surfaced, unlike a concrete model value.

## Validation

1. `just install` first — this is an ephemeral workspace, and step 1 requires rebuilding
   `sase_core_rs` from the `sase-core` checkout before any Python check can see the new
   field.
2. In the `sase-core` checkout, run that repo's own test/lint commands (see its
   `AGENTS.md`) and make sure `agent_scan_parity.rs` passes before returning to `sase`.
3. Run the focused Python tests listed in step 5.
4. Manually confirm the index bump behaves: after `just install`, launching ACE against
   an existing `~/.sase/agent_artifact_index.sqlite` must trigger exactly one
   stale-schema rebuild (`refresh_agent_artifact_index_if_schema_stale`) and then render
   normally. A crash or an empty Agents tab here means the v20 dispatch is wrong.
5. `just test-visual`. The chip may shift pixels in agent-detail PNG goldens; inspect
   `.pytest_cache/sase-visual/` artifacts and accept with
   `--sase-update-visual-snapshots` only the diffs that are the chip (or the absence of
   one) and nothing else. If no fixture exercises an alias launch, add the alias to one
   existing agent-detail fixture so the chip is covered by a golden.
6. `just check-full` — this change touches the Rust binding, the scan wire, and shared
   render paths, so the scoped lane is not sufficient. Resolve every lint, mypy,
   Symvision, docs-sync, and snapshot failure before handing off.

## Non-goals

- **Do not mark the implicit default path.** An agent with no `%model` gets no chip.
  Rendering `← @default` there would be both noisy (it would appear on nearly every
  agent) and sometimes false: `resolve_effective_default_provider_model_with_effort`
  short-circuits on a machine-wide temporary override and never consults `@default` at
  all.
- **Do not add separate effort-source provenance.** Marking whether ` @ xhigh` came from
  `%effort`, from the alias, from a temporary override, or from config is a distinct
  feature with its own design; the chip's single claim is about the `%model` argument.
  Do not record fields for it speculatively.
- **Do not surface pool/fallback selection details.** `← @pool` beside the concrete
  member that was selected is already the honest statement; per-member availability at
  launch is not recorded and must not be re-derived at render time.
- **Do not re-resolve the alias at render time**, and do not add drift detection
  comparing the recorded model against the alias's current target. Model-alias
  resolution in a render path has frozen this TUI before (see `sase/memory/tui_perf.md`
  rule 8); render paths stay read-only over already-loaded state.
- **Do not change rerun/prompt reconstruction.** `artifact_files.py:247` rebuilds a
  `%model:<concrete>` directive from the recorded model; restoring `%model:@<alias>`
  there is a behavior change to rerun semantics and belongs in its own change.
- **Do not extend this to agent list rows, `sase/agents_sync` agent pages, or the mobile
  integration summaries.** They have their own renderers and their own width budgets.
- **Do not backfill historical agents.** Metadata written before this change has no
  alias; those agents render exactly as they do today.
