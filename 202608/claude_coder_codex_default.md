---
tier: tale
title: Default the Claude coder alias to Codex GPT-5.5
goal:
  Unconfigured Claude-authored plan follow-ups resolve through @claude_coder to codex/gpt-5.5 while preserving
  alias-specific override behavior.
size: medium
proposed_by: bbugyi200.athena.s6
create_time: 2026-08-02 11:55:44
status: done
---

- **PROMPT:**
  [prompts/202608/claude_coder_codex_default.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/claude_coder_codex_default.md)

# Plan: Default `@claude_coder` to `codex/gpt-5.5`

## Goal

Change the shipped, unconfigured `@claude_coder` model alias from `claude/sonnet` to `codex/gpt-5.5`, matching the
existing `@codex_coder` default. A plan authored by Claude must still select the provider-specific
`%model:@claude_coder` directive; only that alias's final implicit target changes.

## Current design and constraints

- `src/sase/llm_provider/model_alias_defaults.yml` is the declared single source of truth for shipped model-alias
  defaults. Its `coder.provider_targets` map currently pins `claude` to `claude/sonnet` and `codex` to `codex/gpt-5.5`.
- The existing resolution pipeline already projects that data into concrete launches, alias views, the Models panel, and
  `%model` completion metadata. No routing algorithm or Rust-core API change is needed for a data-only retarget.
- Existing precedence must remain intact: launch-scoped specific/generic coder choices, provider-specific
  temporary/configured values, and explicit generic `@coder` values continue to override the shipped target. Other
  registered `<provider>_coder` aliases must continue to fall back through `@coder`.
- Configuration examples that deliberately demonstrate cross-provider overrides are examples, not shipped defaults; do
  not mechanically rewrite them merely because they mention `claude_coder` or `claude/sonnet`.
- The repository's deterministic visual fixtures encode an implicit `@claude_coder` target. If that fixture is updated
  to represent the new shipped policy, regenerate only the affected PNG goldens and inspect their rendered model badges,
  coder-bucket summaries, and completion rows before accepting them.

## Implementation

1. Update the canonical alias policy.
   - Change `aliases.coder.provider_targets.claude` in `src/sase/llm_provider/model_alias_defaults.yml` to
     `codex/gpt-5.5`, leaving the `codex` entry unchanged.
   - Replace nearby claims that these targets are “provider-local” with wording such as “direct shipped targets” in
     `src/sase/llm_provider/model_alias_policy.py`, `src/sase/llm_provider/model_alias_resolution.py`,
     `src/sase/default_config.yml`, and `src/sase/config/sase.schema.json`. Preserve comments about genuinely
     provider-local model paths, which describe target syntax rather than coder policy.

2. Update behavioral and presentation coverage.
   - In `tests/llm_provider/test_model_alias_defaults.py`, pin both shipped provider-coder entries to `codex/gpt-5.5` so
     the bundled policy mapping cannot drift.
   - In `tests/llm_provider/test_config_role_aliases.py`, update the implicit-value, resolved-target, provider/model,
     and explanatory expectations for `@claude_coder`, while retaining coverage that the active default provider does
     not affect either direct target and that configured/temporary generic and specific overrides still win.
   - In `tests/llm_provider/test_alias_view.py`, assert that the unconfigured `claude_coder` row exposes the same Codex
     target and direct (non-`@coder`-fallback) provenance as `codex_coder`.
   - Strengthen `tests/llm_provider/test_model_alias_defaults_docs_sync.py` so documentation validation associates a
     provider-coder alias with its documented target. A mere occurrence of `codex/gpt-5.5` elsewhere must not allow a
     stale `@claude_coder` claim to pass now that the two aliases share a value.
   - Update only synthetic visual fixtures that claim to show the implicit `@claude_coder` target in
     `tests/ace/tui/visual/_ace_models_panel_png_snapshot_fixtures.py` and
     `tests/ace/tui/visual/test_ace_png_snapshots_model_completion.py`. Regenerate the directly affected goldens under
     `tests/ace/tui/visual/snapshots/png/`, review the actual/expected/diff artifacts, and leave unrelated snapshots
     untouched.

3. Align user-facing documentation with the new out-of-box behavior.
   - Update the implicit-alias table and delegated-work explanation in `docs/llms.md`.
   - Update the shipped-default prose in `docs/configuration.md`.
   - Update the accepted-plan coder handoff description in `docs/xprompt.md`.
   - Update both the Models-panel row description and approval-time “Follow-up default” description in `docs/ace.md`.
   - State clearly that Claude-authored and Codex-authored plan follow-ups still use distinct aliases but both aliases
     now resolve to `codex/gpt-5.5` when no stronger override is present.

## Validation

1. Run `just install` first, as required for an ephemeral SASE workspace.
2. Run the focused nonvisual tests:

   ```bash
   .venv/bin/pytest \
     tests/llm_provider/test_model_alias_defaults.py \
     tests/llm_provider/test_config_role_aliases.py \
     tests/llm_provider/test_alias_view.py \
     tests/llm_provider/test_model_alias_defaults_docs_sync.py
   ```

3. Run the focused Models-panel and model-completion visual tests normally to enumerate intentional failures. Refresh
   only those files with `just test-visual -- --sase-update-visual-snapshots <test-paths>`, inspect every changed PNG
   and diff artifact, then rerun the same focused visual tests without update mode to prove exact convergence.
4. Search source, tests, and current reference docs for stale statements that `@claude_coder` implicitly resolves to
   `claude/sonnet` or that shipped coder targets are provider-local. Distinguish legitimate explicit-override examples
   and unrelated Claude targets from shipped-default claims.
5. Run the mandatory full repository gate:

   ```bash
   just check
   ```

## Acceptance criteria

- With no relevant overrides, resolving `@claude_coder` returns provider `codex`, model `gpt-5.5`, and no alias-borne
  effort; `@codex_coder` continues to return the same target.
- Claude-authored plan handoffs still emit `%model:@claude_coder`, so user configuration and temporary overrides remain
  independently addressable by planner-provider alias.
- Generic and provider-specific override precedence is unchanged, and unpinned provider-coder aliases still inherit
  `@coder`.
- Alias views, completion metadata, Models-panel output, current docs, and relevant PNG goldens consistently show the
  new default.
- Focused tests, visual snapshots, stale-wording checks, and `just check` all pass.
