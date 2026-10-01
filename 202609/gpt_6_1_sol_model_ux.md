---
tier: tale
title: Adopt GPT-6.1 Sol across SASE model UX
goal:
  Replace GPT-6 Sol uses with GPT-6.1 Sol across SASE and linked repositories, with
  consistent model discovery, selection, completion, display, and launch behavior.
size: medium
proposed_by: bbugyi200.athena.0ul
create_time: 2026-09-30 23:19:23
status: wip
---

# Adopt GPT-6.1 Sol across SASE model UX

## Outcome and scope

Make `gpt-6.1-sol` the supported, discoverable Codex Sol model everywhere SASE currently
ships `gpt-6-sol`. Replace the old model ID in this repository and every configured
linked repository, including examples, tests, configuration comments, prompts, and
user-facing labels. Use `gpt61sol` as the new compact display/search alias, following
the existing `gpt56sol` convention.

This is one `medium` tale: the root cause is a stale model entry and its copied
references. The catalog and consumers already support dotted model IDs. One agent can
make the changes and verify the complete path without independently landed phases or new
backend architecture.

## Findings from planning

- `src/sase/llm_provider/models.yml` is the source of truth for built-in catalogs, short
  aliases, provider tier defaults, and size-alias selectors. Its Codex model list,
  short-alias mapping, `large` tier, and `@large`/`@xlarge` selectors all name the old
  model. `codex.py` reads this manifest through `model_manifest.py`.
- The registry feeds model picker rows, the Models panel, provider metadata, prompt
  completion, display badges, and launch resolution. Completion metadata is materialized
  by `src/sase/integrations/xprompt_lsp.py` for the Rust LSP; `sase-nvim` consumes LSP
  results rather than maintaining its own model list. Short aliases are display/search
  hints; completion inserts canonical model IDs.
- A tracked-text audit found old model/name references in 24 files in `sase`. It
  included hidden tracked files and spelling variants such as `gpt6_sol`. Reference
  memories were searched through audited `sase memory read` output; no old-model
  occurrences were found there. No memory edits are planned.
- The live linked inventory is `sase-core`, `chezmoi`, `sase-nvim`, `sase-github`,
  `sase-telegram`, and `sase-research-artifacts`. Each was opened with `sase repo open`
  and its tracked ordinary files searched; none currently names the old model. Some
  chezmoi defaults explicitly use `gpt-5.6-sol`; those are outside this requested
  `gpt-6-sol` replacement.
- Shared filtering, shortcut edits, routing primitives, and the LSP are already generic
  in `sase-core`. No Rust API or binding change is expected.
- The
  [official GPT-6.1 Sol model page](https://developers.openai.com/api/docs/models/gpt-6.1-sol)
  confirms the exact target ID and API effort levels `low`, `medium`, `high`, `xhigh`,
  and `max`; API `none`/`minimal` are unsupported. SASE invokes the Codex CLI and
  currently exposes its separate adapter effort contract. The migrated shipped Sol
  selectors use `xhigh`, which is compatible. This catalog update does not require an
  API endpoint change or a provider-wide effort overhaul.

## Implementation

### 1. Refresh the scope and establish the baseline

Read the current workspace instructions and use `/sase_memory_read` for
`lint_and_test.md`, `tui.md`, `tui_screenshot.md`, `xprompts.md`, and
`decisions:size-alias-effort-ladder`. Inventory repositories with
`sase repo list --json`, then use `/sase_repo` to open every current linked repo before
inspecting it. Use only the paths printed by `sase repo open` and read any applicable
`AGENTS.md`. Record existing dirty files before editing.

Repeat the tracked-text search in each repository, including hidden tracked files and
model-ID/name/alias variants. Separate ordinary source files from canonical memory,
which must be read through `sase memory read`. Cover newly added linked repos as well as
the six listed above. Scope the replacement to checked-out
source/configuration/documentation/test assets; do not rewrite Git history, archived
sidecar plans/chats/research, or live agent records.

### 2. Change the catalog and defaults at their source

In `src/sase/llm_provider/models.yml`:

- Replace `gpt-6-sol` in the Codex model list with `gpt-6.1-sol` at the same position.
  Replace its short-alias entry with `gpt-6.1-sol: gpt61sol`.
- Change the Codex `large` tier to the new ID.
- Change only the Sol members of `@large` and `@xlarge` to `codex/gpt-6.1-sol@xhigh`.
  Preserve pool/fallback operators, order, other providers, effort rungs, and the
  smaller aliases.
- Keep the manifest schema and provider hooks unchanged. Do not retain an old model
  entry, add a redirect to the new model, or add a `supersedes` link whose predecessor
  is no longer in the catalog. The user requested replacement.

Verify bare and provider-qualified new IDs resolve to Codex and survive `@high`/`@xhigh`
parsing intact. Verify tier defaults and real size-alias defaults use the new ID. Test
subprocess argv construction without making paid model calls: new default and explicit
overrides must produce `--model gpt-6.1-sol`. Preserve explicit model overrides and the
existing adapter effort contract.

### 3. Replace copied references and synchronize documentation

Update the ordinary references found by the audit, including:

- `src/sase/default_config.yml` model examples; `sase/xprompts/reads.md`; and the
  example in `src/sase/axe/run_agent_successor.py`.
- CLI help examples in `src/sase/main/parser_agent_search.py` and
  `src/sase/main/parser_bead_lifecycle_state.py`.
- Models-panel placeholders in `src/sase/ace/tui/modals/models_panel_alias_edit.py`,
  `models_panel_override.py`, and `models_panel_selector_builder.py`.
- `docs/ace.md`, `docs/beads.md`, `docs/configuration.md`, `docs/editor.md`,
  `docs/llms.md`, `docs/sdd.md`, `docs/xprompt.md`, and both affected blog posts:
  `structured-agentic-software-engineering.md` and
  `why-coding-agents-need-orchestration.md` under `docs/blog/posts/`.
- Assertions, fixture values, docstrings, and old model-specific test names in
  `tests/test_llm_provider_codex_basic.py`, `tests/test_llm_provider_core.py`,
  `tests/test_model_picker_options.py`, `tests/test_models_panel_edit_custom.py`,
  `tests/test_xprompt_swarm_local_helpers.py`, and
  `tests/ace/tui/widgets/test_model_effort_spacer.py`.

Replace old human names with `GPT-6.1 Sol` and old compact aliases with `gpt61sol`.
Preserve unrelated model IDs. Regenerate the marked catalog/default/alias tables in
`docs/llms.md` using `tools/render_model_docs` through `just fix`; edit the surrounding
prose/examples normally. Do not hand-maintain generated tables.

If the fresh linked-repository audit finds new old-model references, replace those at
their authoritative source in the same implementation. For managed skills, use the
generated-skills workflow before touching generated copies. For chezmoi source changes,
follow its required post-commit apply workflow. Report each repo's changed or no-change
outcome explicitly.

### 4. Verify every model UX path through the shared catalog

Use existing tests and narrowly extend integration coverage where it does not already
prove the new ID passes through an actual consumer. Keep generic catalog tests
data-driven; do not duplicate the entire model list in new assertions. Clear catalog
caches or use fresh processes in checks that inspect changed data.

| Surface                                                            | Required result                                                                                                                 |
| ------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------- |
| Registry/provider metadata and Codex tier resolution               | New ID belongs to Codex, appears once, and is the `large` default                                                               |
| Built-in `@large` and `@xlarge`                                    | Sol members select the new ID with the existing `xhigh` effort and routing semantics                                            |
| Model picker and Models-panel selection/edit/override flows        | New ID and `gpt61sol` are searchable and displayed with Codex styling; selecting/persisting/reloading uses the canonical new ID |
| `%model`/`%m` colon and parenthesized completion                   | Bare and `codex/`-qualified queries find the new model; acceptance inserts the canonical ID                                     |
| Explicit `==model` shortcut and `=alias` shortcut                  | New model/short-alias queries produce correct whole-token edits; size-alias previews reflect the new target                     |
| Effort suffix handling                                             | Dotted ID stays intact when attaching/removing supported `@effort`; existing space cleanup remains correct                      |
| Rust LSP / Neovim integration                                      | Freshly materialized catalog carries the new row and alias; completion filters, details, and insertion edits match ACE          |
| Agent/model labels and interactive Codex/tmux launch consumers     | New ID renders and passes through without truncation errors or fallback to an unrelated provider/model                          |
| CLI help, prompt examples, default-config comments, generated docs | All maintained examples use the new Sol ID/name/alias                                                                           |

Relevant existing coverage includes `tests/llm_provider/test_model_manifest.py`,
`test_model_alias_defaults.py`, and `test_model_policy.py`;
`tests/test_xprompt_model_completion_catalog.py` and
`test_xprompt_model_completion_payload.py`; the directive completion parity
helpers/tests; `tests/ace/tui/widgets/test_model_completion_rows.py`; and the model
picker, Models-panel, provider-core, and Codex invocation tests above. Use a real
catalog row in at least one ACE/LSP boundary check so synthetic `gpt-5.6-sol` fixtures
cannot hide failure to publish the new model. Existing unrelated fixtures may continue
testing their original supported models.

The expected implementation is manifest data plus reference/test changes. If a real
shared parser/filtering defect is discovered, fix it in `sase-core`, expose only the
necessary binding changes, and run that repo's checks. Move `sase-core-revision.txt`
past the host-landed core fix before relying on it in SASE, following
`docs/rust_backend.md`. Do not create a Python or Lua duplicate of shared domain
behavior. Do not change the core pin merely for a data update.

### 5. Validate the resulting tree and finish

Run focused model/provider/completion tests appropriate to the edited paths, including
the new real-catalog integration coverage. Inspect the real Models panel/picker
rendering for the longer ID and refreshed examples when text assertions cannot establish
layout. Start new UI/LSP processes against the updated checkout so cached catalog state
does not mask the result.

Run targeted `just fix-tui-screenshots -- <selectors>` for affected visual coverage,
starting with `tests/ace/tui/visual/test_ace_png_snapshots_models_panel_edit.py` and
`test_ace_png_snapshots_models_panel_navigation.py`; include picker/completion snapshots
if their rendered inputs change. Inspect the report and every changed golden group. A
`partial` report is not proof that skipped cases passed. Read the visual-memory
instructions before generating or reviewing images.

Run `just fix`, inspect the generated documentation diff, then run `sase tool run check`
from the SASE checkout. This is the required wrapped `just check`; do not run
`just check-full`. Run each modified linked repo's required verification from its opened
checkout, using its guarded wrapper where applicable. Use `/sase_monitor` for commands
that need a handoff; wait for the handoff command itself to exit and retain any required
visual-diff review.

Finally, repeat the tracked-text audit in SASE and every linked repo. Require no
remaining old model ID/name/alias in maintained ordinary files, and record the no-change
result for clean linked repos. The archived migration plan and Git history naturally
retain descriptions of the old ID. Verify `git diff --check` and review the complete
diff for accidental replacement of other models or user changes. Finish with
`/sase_final` for all repositories changed; the host owns commits and subsequent
completion actions.

## Acceptance criteria

1. Every former shipped Sol default and selector now targets `gpt-6.1-sol`.
2. Model selection, search hints, completion, labels, and launch argv agree on the new
   ID, with `gpt61sol` as the display/search alias.
3. All maintained old-model references in SASE and linked repositories have been
   replaced, with an explicit audit result for every repository.
4. Existing provider routing, other model defaults, explicit overrides, and effort
   precedence continue to behave as before.
5. Focused checks, required repository checks, generated-doc checks, and affected visual
   verification pass, with any material verification limitation reported.
