---
tier: tale
title: Simplify every artifact-reference prompt expansion
goal:
  Every known artifact ref expands to portable semantic prose without generated
  @-prefixed filesystem paths while resolution and artifact metadata remain intact.
size: medium
proposed_by: bbugyi200.athena.09r
create_time: 2026-08-21 12:08:47
status: wip
---

# Simplify every artifact-reference prompt expansion

## Problem

Artifact references have one authoring grammar but several unrelated prompt-expansion
implementations. Document providers render an `expansion_format`; `stitch` and `patch`
render inside their resolvers; and `chat`, `file`, `bead`, and `agent` fall through a
legacy renderer that emits `@<absolute-path>`. The result is inconsistent and leaks
workspace-local checkout paths into prompts. It also causes the following ordinary
file-reference pass to reinterpret generated `@path` tokens, requiring staging
suppression state between the two preprocessors.

`@research` is the desired model. It expands to portable prose such as
`the 202608/report.md file in the research sidecar repo`, while the authored citation,
canonical identity, resolved path, capture, staging, consumption, and publication data
remain separate metadata. Apply that model consistently to every known artifact-ref
kind. In particular, `@plan:202608/foobar.md` must expand to
`the 202608/foobar.md file in the plans sidecar repo`, and no built-in artifact
expansion may inject an `@` sigil in front of a filesystem path.

This changes expansion output only. Authored prompt citations remain
`@<kind>:<argument>`, stored/CLI identities remain bare `<kind>:<argument>`, and the
ordinary authored `@path` file-reference grammar is not part of this migration.

## Scope and decisions

- Keep parsing, canonicalization, completion, resolution, capture, staging, consumption,
  ref-use recording, publication links, and fragment grammar intact. Prompt prose must
  be derived after resolution without replacing the canonical ref or resolved-path
  metadata used by those systems.
- Make the built-in `plan` provider and the unconfigured document-provider default use
  the same pointer format already used by `research`:
  `the {repo_relative_path} file in the {sidecar_role} sidecar repo`. Thus plan and
  default document refs no longer materialize a sidecar merely to render a prompt and an
  unresolved pointer remains non-fatal under the existing pointer semantics.
- Continue honoring an explicitly configured provider `expansion_format`; the provider
  extension point is not being removed. Update examples and defaults so every known
  shipped provider is portable prose. Explicit custom path-bound providers remain an
  opt-in compatibility capability and must use prose such as `the {checkout_path} file`,
  not the old `@{checkout_path}` examples.
- Centralize built-in wording in the prompt renderer. Resolvers should return facts
  (`ArtifactEntry`, locator, canonical identity, resolved path), not final prompt text.
  Remove the `BuiltinEntryOutcome.prompt_text` shortcut and the stitch/patch renderer
  constants so aliases and full/short resolution paths cannot diverge.
- Render all filesystem targets as prose without an `@` path token. Keep the actual
  resolved or captured path on the staging/consumption records so access and publication
  behavior do not depend on the prose.
- Preserve fragment annotations after the simple pointer, for example `(lines 10-20)`,
  `(page 2)`, and `(time 30s)`.
- Do not change `sase-core`: its grammar, resolution wire, kind catalog, and closed
  expansion formatter already provide the needed facts and formatting primitive. Do not
  change `sase-research-artifacts`: its provider already declares the target document
  format. Add integration coverage in this repository so that format remains the golden
  document behavior.
- Land the behavior atomically with its tests and docs. It is an intentional completed
  behavior replacement, not an early or disabled path, so no feature flag is needed.

## Expansion contract

Use one parameterized contract test to keep these cases exhaustive. Exact articles and
capitalization should follow this table so future kinds have an obvious precedent.

| Authored kind                                    | Expanded prompt pointer                                     | Notes                                                                                                               |
| ------------------------------------------------ | ----------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| `@plan:<path>` and deprecated `@plans:<path>`    | `the <path> file in the plans sidecar repo`                 | Same pointer behavior as `research`; alias warning/canonicalization remains unchanged.                              |
| `@research:<path>` and any default document kind | `the <path> file in the <role> sidecar repo`                | Provider overrides still render through their declared format.                                                      |
| `@file:<path>` and `@file:<source>:<digest>`     | `the <captured-or-materialized-path> file`                  | Never emit `@<path>`; keep strict capture/materialization and telemetry.                                            |
| `@bead:<id>`                                     | `the <canonical-id> bead in the <project> project`          | Use the resolved locator/entry facts, not the generated page path.                                                  |
| `@agent:<name>`                                  | `the <canonical-name> agent in the <project> project`       | Remove the absolute README path and the verbose sibling-file suffix; staging still retains the resolved agent page. |
| `@stitch:<sha>` and `@commit:<repo>@<sha>`       | `the <full-sha> stitch in the <repository> repo`            | Preserve full-SHA resolution and alias equality; omit the checkout path.                                            |
| `@patch:<name>`                                  | `the <name> Patch in the <project> project`                 | Remove the current invalid `sase patch show` inspection hint.                                                       |
| historical `@chat:<path>`                        | `the <resolved-path> file`                                  | Preserve its historical-reader warning and fragment support, but remove the `@` path sigil.                         |
| historical `@bug:<project>#<number>`             | `issue #<number> in the <project> project (<resolved-url>)` | Preserve URL resolution and the historical-reader warning.                                                          |

## Implementation

1. In `src/sase/artifact_ref_prompt_rendering.py`, replace the document-vs-legacy split
   with one renderer that accepts the resolved entry facts and dispatches every known
   `kind_type` through the contract above. Keep document-provider format
   validation/rendering for dynamic document kinds, reuse the closed expansion formatter
   where its placeholders fit, and keep one shared fragment-appending step. Add small
   helpers for splitting the project/object locators returned for beads and agents, and
   fail loudly on malformed successful outcomes rather than silently falling back to a
   local path.
2. In `src/sase/artifact_ref_prompt.py` and
   `src/sase/artifact_providers/builtin_entries.py`, pass `outcome.entry` and canonical
   resolution facts into the centralized renderer. Delete the resolver-owned
   `prompt_text` branch. Preserve `_ExpandedArtifactRef.resolved_path`, `canonical_ref`,
   `stable_id`, captured records, Jinja protection, and ref-use `prompt_text` so only
   the replacement wording changes.
3. In `src/sase/artifact_providers/builtin_entry_stitch.py` and
   `src/sase/artifact_providers/builtin_entry_patch.py`, remove prompt rendering and
   expansion-format validation. Keep resolution, ambiguity diagnostics, full-SHA lookup,
   project display names, locators, canonical references, and structured entry
   properties unchanged. Remove the agent transcript-sibling wording helper from the
   legacy rendering path; agent resolution/enrichment continues to discover and stage
   the published page.
4. In `src/sase/artifact_providers/_builtin.py` and `src/sase/sidecar_ref_config.py`,
   change both the built-in plan format and `DEFAULT_DOCUMENT_REF_EXPANSION_FORMAT` to
   the shared sidecar pointer format. Keep provider override validation and
   pointer/path-bound classification. Verify plan, deprecated plans, research, and an
   unconfigured custom document kind all select the intended role and format.
5. Simplify the late-preprocessing seam in `src/sase/llm_provider/preprocessing.py`,
   `src/sase/artifact_ref_prompt.py`, and `src/sase/file_references.py` only as far as
   the new invariant safely allows. Built-in expansions must no longer need protection
   from the ordinary `@path` pass. Retain compatibility for explicitly configured
   provider formats that can still produce path-shaped text; do not accidentally
   re-stage or reject their output. Update comments and names that still describe all
   artifact refs as generated `@path` tokens.
6. Update public documentation in `docs/artifact_references.md`,
   `docs/configuration.md`, `docs/plugins.md`, `docs/getting_started.md`,
   `docs/llms.md`, and `docs/agent_images.md`. Distinguish authored `@kind:` citations
   from expanded prose, replace every `@{checkout_path}` default/example, describe plan
   as a pointer like research, and retain the documented custom path-bound escape hatch
   without promising an `@`-prefixed expanded path. Do not edit generated memory files.

## Tests

- Replace path-oriented assertions in `tests/artifact_refs/test_rendering.py`,
  `test_preprocessing_expansion.py`, `test_preprocessing_effects.py`,
  `test_prompt_materialization.py`, and `test_aliases.py` with the exact expansion
  matrix above. Include quoted arguments, Unicode boundaries, and every fragment type.
- Update the builtin provider tests so stitch/patch/bead/agent resolvers assert only
  resolution facts, while end-to-end prompt tests assert the centralized wording. Cover
  short and full bead IDs, local and global agent names, cross-project ambiguity,
  commit/plans aliases, and historical chat/bug warnings.
- Add a regression proving `@plan:202608/foobar.md` expands to the requested plans
  sidecar pointer without cloning a missing plans sidecar, and that a missing plan path
  follows the same non-fatal pointer behavior as research. Retain coverage for an
  explicit custom `{checkout_path}` provider so compatibility materialization does not
  regress.
- For both path-backed and indexed `@file` forms, assert the expanded prompt contains
  the captured/materialized path but never `@<path>`, while the staged manifest,
  consumption event, ref-use row, digest, logical locator, and publication inputs keep
  their pre-migration values.
- Add a full-pipeline regression showing a built-in artifact expansion is not parsed a
  second time by `process_file_references`, and a separate regression proving an
  ordinary authored `@path` reference still follows its existing grammar.
- Search the shipped code, tests, and docs for stale built-in `@{checkout_path}` and
  `@<absolute-path>` expansion claims; allow only compatibility fixtures that
  deliberately exercise an explicit custom provider.

## Verification

1. Run `just install` first because the workspace is ephemeral.
2. Run the focused artifact suites during implementation:
   `pytest tests/artifact_refs tests/artifact_providers tests/test_sidecar_ref_config.py`.
3. Run `just check` after all repository changes.
4. Because this changes shared launch-prompt preprocessing, run `just check-full`
   through `/sase_monitor` before landing, with the required `TESTING`/`TESTED` statuses
   and a follow-up action that inspects and repairs any failure.
5. Manually preview one prompt containing plan, research, file, bead, agent, stitch,
   patch, and fragments. Confirm the expanded prompt contains portable semantic prose,
   contains no generated `@` filesystem token, records the original canonical refs, and
   does not clone plan/research sidecars merely to phrase their pointers.

## Out of scope

- Changing the authored `@<kind>:<argument>` syntax, bare canonical identities, ACE or
  LSP completion, artifact links, or publication-link formatting.
- Retiring explicitly configured path-bound document providers or the launch-time
  materializer they still require.
- Changing artifact resolution strictness for non-document kinds; missing bead, agent,
  stitch, patch, file, chat, and bug refs keep their current diagnostics.
- Editing SASE memory, changing sidecar inventory globs, or releasing provider packages.
