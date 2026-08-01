---
tier: tale
title: Hide Fakey from agent-CLI update inventories
goal:
  Agent-CLI management surfaces omit the internal Fakey provider while Fakey's routing, diagnostics, tests, and demos
  continue to work.
proposed_by: bbugyi200.athena.rg
create_time: 2026-08-01 10:36:25
status: done
---

- **PROMPT:** [prompts/202608/hide_fakey_from_updates.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/hide_fakey_from_updates.md)
- **AGENTS:**
  - [bbugyi200.athena.rg](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.rg.md)
- **COMMITS:**
  - [1c29afa](https://github.com/sase-org/sase/commit/1c29afae277b55a4bbe584c9f73193d90ef3f1a4) — fix(agent-clis): hide internal providers from CLI management

# Hide Fakey From Agent-CLI Update Inventories

## Objective

Remove the internal `fakey` testing provider from the SASE Admin Center's **Updates → Agent CLIs** inventory, including
its summary counts and update planning inputs, without hardcoding the provider name in TUI code or weakening Fakey's
test, demo, routing, autodetection, retry, or doctor behavior.

## Why This Is a Tale

This is one cohesive provider-metadata and shared-inventory change. One follow-up coding agent can add the opt-in
visibility contract, consume it in the existing agent-CLI service, update focused tests and documentation, and run the
repository gate without needing independently deliverable phases or cross-agent dependencies.

## Current Behavior and Root Cause

The screenshot at `.sase/home/tmp/screenshots/20260801_102805.png` shows `Fakey` as one of six installed agent CLIs in
the Admin Center. Its detail panel says the executable is not installed while the install method is `bundled`. Both
observations follow from the current provider contract:

- `FakeyProvider.llm_autodetect_cli_name()` returns `"fakey"`, so the agent-CLI detector treats it as a provider CLI.
- `FakeyProvider.llm_install_metadata()` declares `manager: bundled`, and `AgentCliStatus.installed` intentionally
  considers every bundled provider installed even when its executable cannot be resolved. That is why the row has an
  installed glyph and contributes to the installed count in the screenshot.
- `load_plugins_catalog_for_pane()` obtains `agent_cli_statuses` through `collect_agent_cli_statuses()` and passes the
  same tuple to the TUI, update-status snapshot construction, and pane-local update planning. The renderer correctly
  displays every status it receives; hiding only a row inside `plugins_browser_agent_clis.py` would leave counts,
  background candidates, and update actions inconsistent.
- `collect_agent_cli_statuses()`, `detect_agent_cli_statuses_for_names()`, `sase agent-cli list/update`, the Admin
  Center, and update-candidate revalidation all derive provider inputs through
  `sase.agent_clis.operations._provider_metadata()`. Its own contract says CLI and TUI callers should share this
  inventory rather than build separate ones.
- Fakey already implements `llm_hidden_from_model_pickers()`. That flag must not be repurposed: its hookspec and the
  approved `hide_fakey_from_model_pickers` plan explicitly restrict it to model-selection display and promise that
  provider inventory remains unaffected. A third-party provider could legitimately hide its models while still exposing
  an independently managed CLI.

The correct boundary is therefore a new, independently opt-in provider metadata flag consumed once by the shared
agent-CLI management service. Fakey is bundled with SASE and cannot be independently installed or updated, so it opts
out; real providers and third-party providers that omit the flag remain unchanged.

## Behavioral Contract

After this change:

- The Admin Center's **Updates → Agent CLIs** list, summary, detail selection, marks, and `A` update plan receive only
  independently manageable provider CLIs. Fakey is absent and the total/installed counts decrease accordingly.
- `sase agent-cli list`, JSON inventory, named/all update planning, automatic update snapshots, and bounded candidate
  revalidation use the same filtered inventory. Keeping these surfaces aligned preserves the shared-service contract and
  prevents a hidden provider from reappearing in a confirmation or top-bar update count.
- Fakey remains a fully registered LLM provider. Explicit `fakey-large`, `fakey-small`, and `fakey/fakey-*` resolution,
  `SASE_LLM_EXEC_PROVIDER=fakey`, the `fakey` console script, model aliases, provider styles/badges, autodetect
  ordering, retry defaults, test harnesses, demos, and `sase doctor` continue to work.
- `llm_hidden_from_model_pickers()` retains its current model-selection-only meaning. The new flag is separate so each
  visibility decision can evolve independently.
- Providers that omit or return a value other than literal `True` from the new hook remain visible, preserving
  compatibility with existing third-party provider plugins and older metadata payloads. This additive optional field
  does not require a metadata schema-version bump.

## Implementation

1. Add an opt-in agent-CLI-management visibility hook.
   - In `src/sase/llm_provider/_hookspec.py`, add a `firstresult=True` hookspec named
     `llm_hidden_from_agent_cli_management(self) -> bool | None` alongside the other provider metadata hooks.
   - Document that it hides providers which are internal or not independently manageable from `sase agent-cli`
     inventory/update operations and the Admin Center's Agent CLIs update surface, but does not affect routing,
     resolution, autodetection, direct invocation, or doctor diagnostics. Omission means visible.
   - In `src/sase/llm_provider/_registry_metadata.py`, normalize the hook into a top-level provider field such as
     `"hidden_from_agent_cli_management"` using the strict `is True` convention already used by
     `hidden_from_model_pickers`. Keep all aggregate routing and autodetect metadata untouched.
   - In `src/sase/llm_provider/fakey.py`, implement the new hook with `@hookimpl` returning `True`. Do not remove or
     alter Fakey's install metadata, console entry point, model metadata, or existing picker-hiding hook.

2. Filter at the shared agent-CLI service boundary.
   - Update `_provider_metadata()` in `src/sase/agent_clis/operations.py` to omit provider records whose normalized
     `hidden_from_agent_cli_management` field is literal `True`.
   - Keep the filter metadata-driven and generic; do not compare the registry name with the string `"fakey"` and do not
     add a second filter to the Admin Center renderer or loader.
   - Preserve the empty-name fast path in `detect_agent_cli_statuses_for_names()`. Because both full collection and
     named detection call `_provider_metadata()`, the same opt-out must cover CLI/TUI inventory, `--all` planning,
     explicit hidden-provider queries, and cached candidate revalidation before any executable/npm/latest-version
     probing occurs.
   - Do not change `AgentCliStatus.installed` or generic `InstallMethod.BUNDLED` behavior; other bundled provider CLIs
     may still be legitimate management entries.

3. Add provider-metadata and shared-inventory regression coverage.
   - In `tests/test_llm_provider_registry.py`, mirror the existing model-picker visibility tests for the new hook:
     literal `True` is normalized as hidden, `False` is visible, and an omitted hook defaults to visible. Also include
     the new accessor/metadata call in the memoization coverage if the implementation exposes a public accessor.
   - In `tests/fakey/test_provider.py`, assert the live registry marks Fakey hidden from agent-CLI management while its
     existing routing, resolution, aliases, autodetect-last ordering, setup/doctor metadata, and model-picker hiding
     assertions remain intact.
   - In `tests/agent_clis/test_operations.py`, pass a synthetic metadata payload containing visible, hidden, and
     partial/legacy providers and capture the provider map handed to `detect_agent_cli_statuses()`. Prove that hidden
     entries are removed before probing while visible and hook-absent entries remain. Cover both full collection and
     named bounded detection so a hidden provider cannot be rediscovered by revalidation or an explicit update query.
   - Retain the existing synthetic bundled-status planning test: bundled install handling is still supported even though
     Fakey no longer enters the management inventory.

4. Cover the Admin Center-facing outcome without duplicating policy in presentation code.
   - Extend the nearest Updates-pane loading/browser test only as needed to demonstrate that the filtered shared
     inventory drives the rendered option IDs and summary counts, and that normal real-provider rows/update planning
     remain available. Prefer exercising the production collection boundary over teaching the renderer about hidden
     metadata.
   - Assert there is no `agent-cli__fakey` option and no `Fakey` detail selection when using the real filtered
     inventory. Do not make fixture-only renderer tests silently filter arbitrary supplied statuses; those fixtures
     should remain able to render any `AgentCliStatus` for isolated UI coverage.
   - The existing config-center visual snapshots use deterministic real-provider fixtures and should not require golden
     changes. Run the focused Agent CLIs visual snapshot in comparison mode to verify that assumption; update a golden
     only if a production-registry-backed snapshot legitimately changes, and inspect its generated diff first.

5. Update user-facing documentation.
   - In `docs/fakey.md`, state that Fakey is intentionally absent from agent-CLI management inventories and the Admin
     Center Updates list, while explicit execution and doctor diagnostics remain available.
   - In `docs/agent_providers.md`, describe `sase agent-cli` as listing independently manageable provider CLIs, remove
     or generalize the Fakey-specific bundled row in the update matrix, and keep the documented CLI/Admin Center
     inventory equivalence accurate.
   - In `docs/ace.md`, clarify that **Updates → Agent CLIs** omits providers that opt out of independent CLI management
     (including the bundled internal Fakey provider).
   - Do not remove Fakey from provider, model, retry, test, or demo documentation; those capabilities remain supported.

## Validation

1. Run `just install` before any checks because SASE workspaces are ephemeral and provider entry points must reflect the
   edited checkout.
2. Run focused tests for the affected contracts:
   - `just test -- tests/test_llm_provider_registry.py tests/agent_clis/test_operations.py tests/agent_clis/test_cli.py`
   - `just test -- tests/fakey/test_provider.py tests/test_update_status_compute.py tests/test_update_status_revalidation.py`
   - `just test -- tests/ace/tui/test_plugins_browser_pane_agent_clis.py tests/ace/tui/test_plugins_browser_loading_freshness.py`
3. Run the focused comparison-only visual suite:
   `just test-visual -- tests/ace/tui/visual/test_ace_png_snapshots_config_center_plugins.py`. If a snapshot changes,
   inspect `.pytest_cache/sase-visual/` before accepting it, regenerate only an intentional affected golden, and rerun
   without the update flag.
4. Exercise the installed command surface after `just install`: verify `sase agent-cli list -j` contains the expected
   real providers but no `fakey`, and verify the real registry still resolves Fakey through the existing focused tests
   rather than launching an external agent.
5. Run the mandatory full `just check` gate and resolve all formatting, lint, mypy, Symvision, test, and PNG snapshot
   failures before handoff.

## Acceptance Criteria

- The SASE Admin Center's **Updates → Agent CLIs** sub-tab contains no Fakey row, its summary does not count Fakey, and
  its update/mark/selection paths cannot target Fakey.
- The shared `sase agent-cli` list/update inventory and update-status candidate pipeline also omit Fakey, so CLI, TUI,
  confirmation, cache, and revalidation surfaces remain consistent.
- Hiding is driven by an optional provider metadata hook and tested with a synthetic hidden provider; there is no
  Fakey-name special case in agent-CLI or TUI code, and hook-absent third-party providers remain visible.
- Generic bundled-provider behavior remains supported and covered.
- Fakey remains fully registered and explicitly routable, its console script/test harness/demos/retry behavior remain
  intact, and `sase doctor` can still diagnose it.
- Documentation accurately distinguishes agent-CLI management visibility from model-picker visibility and runtime
  support.
- All focused tests, the relevant visual comparison, command-surface sanity check, and `just check` pass.
