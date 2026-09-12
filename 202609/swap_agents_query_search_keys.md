---
tier: tale
title: Swap the Agents query and metadata-search shortcuts
goal:
  Bare slash opens the Agents structured-query editor, while leader-slash starts inline
  metadata search, without changing either feature on other tabs.
size: small
proposed_by: bbugyi200.athena.0jn.f0
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0jn.f0](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0jn.f0.md)
- **COMMITS:**
  - [5aaa244](https://github.com/sase-org/sase-research-artifacts/commit/5aaa244451eab367eb95d6ce65f03b38e8352407)
    — fix(research): use queue capacity directive

# Plan: Swap the Agents query and metadata-search shortcuts

## Outcome and scope

On the top-level Agents tab, pressing the configured app-level query key (bare `/` by
default) opens the Agents structured-query editor, and pressing the configured leader
metadata-search chord (`,/` by default) starts forward inline search in the selected
agent's metadata. The existing direct `f` filter-bar shortcut remains available. Other
Artifacts panes keep bare `/` for their query editors, and the Axe behavior is
unchanged.

This is a **small tale**: the behavior has one precise dispatch cause and one coherent
keymap-contract change, and a single coding agent can update implementation, discovery
surfaces, compatibility diagnostics, documentation, and focused tests together. It does
not need independently landed epic phases or a rollout flag; the requested swap is the
complete user-facing behavior, not a partially exposed feature.

## Findings that determine the implementation

- `src/sase/ace/tui/bindings.py` and the runtime registry currently bind both app-level
  `edit_query` and `search_forward` to `slash`. Contextual availability in
  `src/sase/ace/tui/_app_action_availability.py` disables `edit_query` on Agents and
  makes `search_forward` Agents-only, so Textual selects metadata search there.
- `actions/agent_workflow/_leader_mode.py` owns the leader `edit_query` subkey and
  bypasses app-action availability to call `action_edit_query()` only on Agents.
  `actions/base.py:action_edit_query` already routes Agents through
  `_edit_agent_search_query()`, which selects the unified auto-hiding filter bar or the
  legacy modal according to `agents_unified_query`. The swap should reuse this path.
- `actions/agents/_metadata_search.py:action_search_forward` already owns the complete
  metadata-search start guard and controller entry point. The leader chord should call
  it rather than duplicating focus, modal, or prompt-input checks.
- App and built-in-mode action IDs drive much more than dispatch: defaults, duplicate
  validation, the command palette, leader repeat, footers, Help, onboarding quickstart,
  and user overrides all derive from them. Leaving a leader action named `edit_query`
  while making it search metadata would make those surfaces and future remaps lie.
- The approved query-persistence plan establishes that explicit Agents-query commits
  persist while metadata search stays session-only. This shortcut change must enter
  through the same query/session helpers so bare `/` gains persistence automatically
  once that plan lands; it must not create another query commit or storage path.

## Behavioral contract

1. On Agents, bare `/` opens the same query-editing experience currently reached via
   `,/`: the `AgentsFilterBar` with `agents_unified_query` enabled and the legacy
   `QueryEditModal` with it disabled. Submission, preview, validation, history, saved
   slots, current-project seed handling, refresh, and remembered-query persistence are
   unchanged because the shortcut delegates to the existing action.
2. `f` remains the direct unified filter-bar alias and retains its existing Off-flag
   behavior. This change swaps `/` and `,/`; it does not remove the additional `f`
   affordance.
3. On Agents, `,/` starts forward inline metadata search. `Ctrl+R`, `n`/`N`, Enter,
   Escape, scrolling, yanking, frozen-corpus behavior, focus restoration, and teardown
   on selection/tab changes remain unchanged. Starting or completing metadata search
   never commits or persists an Agents query.
4. Bare `/` continues to edit queries on every query-capable Artifacts pane. `,/` is
   Agents-only and is inert on Artifacts and Axe, including when replayed through `,,`.
   Leader repeat remembers a successful Agents metadata-search dispatch and rechecks the
   current tab before replay.
5. Preserve semantic action names across dispatch and discovery: `edit_query` means
   structured query editing, and `search_forward` means metadata search. The canonical
   defaults become `ace.keymaps.app.edit_query: slash` and
   `ace.keymaps.modes.leader_mode.keys.search_forward: slash`; the former leader
   `edit_query` slot and former app-level `search_forward` slot are no longer active
   keymap settings.
6. Treat the setting move as the intentional keymap contract change requested here, not
   as two hidden alternate bindings. Stale overrides should be ignored with an
   actionable warning that names the replacement scope; do not keep the old behavior
   behind a feature flag or silently expose an extra direct metadata-search key.
7. Help, onboarding, footers, and the command palette must display the effective
   configured keys: Agents query help shows app `edit_query` (plus `agents_filters` when
   applicable), metadata-search help shows leader `search_forward`, and the leader
   footer/palette label says metadata search. No surface should advertise `,/` as query
   editing after the swap.

## Implementation

### 1. Make the keymap registry express the new ownership

Update `src/sase/default_config.yml` as the source of truth: keep app `edit_query` at
`slash`, remove the app-level `search_forward` default, replace the leader-mode
`edit_query` key with `search_forward: slash`, and update the adjacent comments. Mirror
that contract in `keymaps/app_keymaps.py`, `keymaps/metadata.py`,
`keymaps/mode_keymaps.py`, and the fallback list in `tui/bindings.py` so generated
bindings and typed/default registries cannot drift.

Remove the `edit_query`/`search_forward` contextual app-duplicate exemption from
`keymaps/registry.py`, because the two actions no longer share the app scope. Extend the
built-in-mode loading checks so stale `leader_mode.keys.edit_query` and
`app.search_forward` overrides cannot be merged back as runnable commands; emit a
targeted warning directing users to `app.edit_query` and
`leader_mode.keys.search_forward`, respectively. Do not translate either override into a
second live shortcut: users should make the new scope explicit if they customize the
swapped keys.

Keep `search_reverse` app-level because it is an active-search control rather than a
search-start shortcut. Do not change any keys outside the two requested entry points.

### 2. Swap contextual dispatch while reusing existing feature entry points

Remove the Agents exclusion for `edit_query` in
`src/sase/ace/tui/_app_action_availability.py`, allowing the configured app query key to
reach `actions/base.py:action_edit_query` on Agents as it already does on Artifacts.
Preserve the existing modal/prompt ownership guards and query-capability checks on all
other surfaces.

Change `actions/agent_workflow/_leader_mode.py` to recognize the leader `search_forward`
subkey only on Agents, remember it for leader repeat, and invoke the existing
`action_search_forward()` metadata controller entry point. Recheck the current tab
before both initial and repeated dispatch, and preserve the existing footer refresh and
no-op behavior elsewhere. Do not inline metadata-controller logic into leader mode or
route bare `/` through a special Agents-only commit path.

If the remembered-query implementation has landed by coding time, use its shared
explicit-commit helper unchanged and extend its entry-point coverage: a query submitted
after opening with bare `/` must persist, while opening/using `,/` metadata search must
leave the remembered query untouched. Resolve overlapping edits in the query docs/tests
by preserving that plan's storage, startup precedence, and lifecycle contract and only
replacing its former bare-`/` keybinding assumption.

### 3. Update command, footer, Help, and onboarding discovery

Move forward metadata search from app-command metadata to the built-in leader command
catalog in `commands/_app_metadata_display.py` and `commands/_mode_commands.py`, with
Agents-only scope, a metadata-search label, and searchable aliases. Keep app
`edit_query` discoverable and runnable on Agents, including from the command palette.
Update catalog coverage/availability guards for the new canonical command IDs.

Update `modals/help_modal/agents_bindings.py`, `widgets/tab_quickstart.py`, and
`widgets/_keybinding_modes.py` so normal Agents guidance uses app `edit_query` for the
query bar, the Metadata Search section uses the leader `search_forward` display, and the
leader footer describes metadata search. All displays must derive from the active
registry so custom leader prefixes and subkeys render correctly. Refresh affected Help
or onboarding visual goldens only when the asserted text/layout genuinely changes.

### 4. Document the swapped defaults and configuration migration

Update the Agent Search, Leader Mode, Query System, and custom-keymap passages in
`docs/ace.md`; the live-Agent overview in `docs/query_language.md`; and the examples,
field descriptions, contextual-scope explanation, duplicate-key table, and validation
notes in `docs/configuration.md`. State plainly that `/` (or `f`) edits the Agents query
and `,/` searches the selected metadata, while other tabs keep their current behavior.
Include the old-to-new setting names so users with overrides can migrate without
guessing.

## Verification and acceptance

Extend the existing tests instead of building a parallel harness:

- In `tests/test_keymaps_e2e.py`, prove the default context matrix end to end: bare `/`
  calls query editing on Agents and each query-capable Artifacts pane, while `,/` calls
  metadata search only on Agents. Add custom app-query and leader-search remaps to show
  that dispatch follows both configured keys and a configured leader prefix.
- In `tests/ace/tui/test_agents_filter_bar_session.py` and the legacy query-modal tests,
  open with bare `/` and prove the real unified bar/Off-flag modal behavior, including a
  successful commit. Keep `f` regression coverage.
- In `tests/ace/tui/test_agent_metadata_search.py`, switch search-start pilot coverage
  to the leader chord and retain all existing controller assertions. Verify bare `/`
  opens the query editor instead of freezing metadata and that `,/` does not modify the
  committed or remembered Agents query. Update the metadata-search PNG driver without
  changing the controller's expected rendering.
- Update leader dispatch/repeat/footer tests, keymap default/registry/binding/validation
  tests, command-catalog and availability guards, Help-display tests, and quickstart
  tests for `app.edit_query` plus `leader.search_forward`. Cover stale override warnings
  and prove removed action slots do not reappear in the registry or palette.
- Run the focused keymap, command-catalog, Help/onboarding, Agents filter-session,
  legacy-query, metadata-search, and related visual tests. If the persistence plan has
  landed, also run its Agents query entry-point and restart regression tests.

Before implementation verification, read `lint_and_test.md` through `sase memory read`.
Apply the repository's formatter and run **`just check`** on the final tree. Follow the
documented `check-full` escalation policy rather than running the exhaustive suite ad
hoc. If key handling or repaint responsiveness regresses, follow `tui_perf.md` and the
performance runbook; the accepted implementation adds no disk I/O, parsing, or blocking
work to either keystroke path.

The tale is complete when the two default Agents shortcuts have exchanged behaviors, all
configured/discovery surfaces agree with the new semantic action ownership, query
persistence remains tied only to committed structured queries, metadata search remains
session-only, and the required checks pass. This planning turn changes no implementation
files.
