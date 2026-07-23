---
tier: tale
title: Integrate capitalized snippet aliases into SASE catalogs and live saves
goal: ACE, live saves, and the editor helper consume the shared Rust composer while
  preserving authored-source state, metadata, precedence, and event-loop responsiveness.
bead: sase-8u.2
parent: sase/repos/plans/202607/capitalized_snippet_aliases.md
create_time: 2026-07-23 08:33:38
status: done
---

- **PROMPT:** [202607/prompts/capitalized_snippet_host_integration.md](prompts/capitalized_snippet_host_integration.md)

# Plan: Integrate capitalized snippet aliases into SASE catalogs and live saves

## Context

Bead `sase-8u.2` is the host-integration phase of the capitalized-snippet-alias epic. Its prerequisite added the
canonical composer and `compose_snippet_catalog` PyO3 binding in `sase-core`. The Python host still resolves snippet
references itself in ACE and the editor helper, and its live save path currently recomposes from the effective cache.
Once aliases exist, that cache is no longer a safe source catalog: a previously generated `Foo` would look explicit and
prevent a later save of `foo` from refreshing its alias.

The implementation must preserve the existing source precedence—xprompt snippets first, authored `ace.snippets`
overrides second, and session-local pending saves last—then call the Rust contract exactly once on the resulting
explicit catalog. Generated aliases remain runtime-only, inherit helper metadata from their provenance source, count as
usable entries, and never pollute authored-user collision checks. All catalog building and live-save composition must
stay in the existing worker/`asyncio.to_thread` paths rather than moving work onto Textual's event loop.

## Add a validated Rust composition facade

Create a small facade under `src/sase/core/` that accepts a string-to-string mapping and calls
`require_rust_binding("compose_snippet_catalog")` with that literal binding name. Normalize the returned plain-dict wire
into an immutable result carrying two ordinary dictionaries: final templates and alias provenance.

Validate the boundary instead of coercing arbitrary values: the top-level payload, `templates`, and `alias_provenance`
must be mappings; every key and value in both nested mappings must be a string; every provenance alias must exist in
final templates; and every provenance source must exist in the explicit input. Raise clear `TypeError`/`ValueError`
diagnostics for malformed wire data. Do not implement capitalization, reference resolution, alias collision handling, or
provenance inference in Python.

Add focused facade tests with a fake binding for argument forwarding, normalized output, and each malformed wire class.
Ensure the static binding inventory discovers `compose_snippet_catalog`.

## Preserve raw explicit state in ACE snapshots

In `src/sase/ace/tui/prompt_catalog.py`, keep the current xprompt, user-config, and pending-save merge order. Preserve
that merged uncomposed mapping as a new field on `PromptCatalogSnapshot`, then compose it through the facade for the
published `snippets` mapping. Keep `user_snippets` limited to authored `ace.snippets`; generated aliases and pending
saves must not appear persisted.

Change `compose_pending_snippet_saves` to accept raw explicit sources, apply the complete pending-save overlay, and
delegate to the same facade. Remove direct use of the Python snippet-reference resolver from these catalog paths.

Update snapshot construction sites and tests to cover:

- aliases generated for xprompt, user-config, and pending-save entries;
- `#[Foo]` references and aliases derived from already reference-composed templates;
- authored `Foo` collisions at both xprompt/user precedence boundaries;
- preservation of raw explicit templates separately from final composed templates and authored user entries.

## Recompose live saves from explicit sources

In `src/sase/ace/tui/actions/agent_workflow/_prompt_bar_save_xprompt_snippets.py`, base immediate save recomposition on
the current snapshot's raw explicit catalog plus a complete copy of `_pending_snippet_saves`. When no snapshot has been
published yet, fall back to authored `_user_snippets`, not the effective `_snippets_cache`. Preserve the existing
generation and pending-snapshot retry guards, catalog rebuild scheduling, and `asyncio.to_thread` boundary.

Extend the prompt-save regressions so a lowercase save immediately exposes and expands its capitalized alias in the same
mounted prompt, a second save updates both spellings, and an explicitly authored `Foo` remains untouched. Assert that
composition runs off the event-loop thread and that pending state retains only the authored trigger/template.

## Compose editor-helper entries with provenance

In `src/sase/integrations/_editor_helper_snippets.py`, retain the existing xprompt/user merge and invalid-trigger
filtering, then send the raw template map through the new facade. Build the final entry sequence in the composer's
stable template order. For explicit triggers, retain the merged entry's metadata; for generated triggers, clone metadata
from the provenance source while replacing only `trigger` and final `template`. Set helper statistics and the success
message from the final entry count.

Add helper-bridge regressions for generated entry ordering, source metadata, source display path, descriptions,
templates, total count, user-over-xprompt precedence, explicit capitalized collisions, `#[Foo]` references, and invalid
trigger filtering before composition.

## Published-core compatibility and verification

Do not guess or predeclare an unpublished core version. Identify the first release-plz-published `sase-core-rs` release
that contains `compose_snippet_catalog`, verify the binding directly from that exact wheel, then advance the lower and
upper compatibility bounds in `pyproject.toml` to that release line. Refresh dependency metadata only where the
repository's normal workflow requires it. Do not point released SASE at an older or unpublished core wheel, and do not
close the bead while the required published minimum is unavailable.

Run `just install` before Python validation so the workspace builds the linked core contract. During iteration, run the
facade, prompt-catalog, prompt-save, prompt-expansion, editor-helper, and binding-inventory tests. Verify that the new
binding appears in `tools/check_sase_core_rs_bindings --list`. In a clean temporary environment with no linked-source
override, install the exact declared minimum `sase-core-rs` wheel and run `tools/check_sase_core_rs_bindings` to prove
every statically required binding exists.

Finish with the repository-mandated `just check`, fixing and rerunning until it passes. Reinspect the final diff for
raw/effective catalog separation, authored-name collision behavior, deterministic ordering, and event-loop boundaries.
Close only `sase-8u.2`; leave parent epic `sase-8u` open and create no additional beads.
