---
tier: tale
title: Configured tribe expansion governs every panel birth
goal:
  Every Agents-tab tribe panel takes its fold state from
  ace.tribes.<name>.initially_expanded each time the panel comes into existence, so
  @chop always starts collapsed, while manual folds last only as long as the panel does
  and agent churn inside a live panel changes nothing.
size: medium
proposed_by: bbugyi200.athena.x7
create_time: 2026-08-10 10:35:37
status: wip
---

# Plan: Configured tribe expansion governs every panel birth

## Problem

`ace.tribes.<name>.initially_expanded` is documented as "Initial state the first time
the Agents-tab panel appears", but in practice it is only a seed that any past user fold
permanently overrides:

- `effective_collapsed_panel_keys()` (`src/sase/ace/tui/models/tribe_display.py:180`)
  treats a panel as collapsed only when it is in `collapsed_intent`, or when it is
  absent from `expanded_intent` **and** config says `initially_expanded: false`.
- Those two intent sets (`_collapsed_panel_keys` / `_expanded_panel_keys`) are persisted
  to `~/.sase/ace_agents_fold_state.json` as `collapsed_panels` / `expanded_panels`
  (schema v3) and restored into memory at startup
  (`src/sase/ace/tui/actions/agents/_fold_persistence.py:258-259`).

So the one time a user expands `@chop` to watch a scheduled chop run, `chop` lands in
`expanded_panels` on disk and every later ACE session starts that panel expanded. The
bundled `chop` config (`src/sase/default_config.yml:143-147`) that exists to keep
routine background work quiet is silently dead from then on.

## Goal

Configured expansion decides a tribe panel's fold state **whenever that panel comes into
existence**, and nothing else does. `@chop` starts collapsed every time its panel is
born, including after a restart, no matter what the user did to it previously.

## Semantics to implement

These are the acceptance criteria; implement whatever code satisfies them.

1. **Birth uses config.** When a panel key becomes live and no live fold intent exists
   for it, its state is `ace.tribes.<key>.initially_expanded` (default `true`; the
   `default` config key maps to `PanelKey None`).
2. **Session start is a birth for everything.** No whole-panel fold state survives an
   ACE restart. A fresh ACE always shows `@chop` collapsed and every unconfigured tribe
   expanded.
3. **Manual folds live exactly as long as the panel does.** A user collapse/expand holds
   for the rest of that panel's life in the running session; when the panel dies its
   intent is retired, so the next birth of the same tribe uses config again.
4. **Agent churn inside a live panel changes nothing.** Agents added to, or removed
   from, an already-live tribe must never alter that panel's fold state — this is the
   invariant the current `_sync_panel_group` comment ("Whole-panel fold intent
   deliberately outlives this projection") protects, and it must survive the change.
5. **Liveness is evaluated over the unfiltered agent set**, not the query-filtered
   rendered set. Typing an Agents-tab query that hides every `@epic` agent must not
   silently retire the user's `@epic` fold and re-expand the panel when the query is
   cleared.
6. **Group folds are untouched.** In-panel group/workflow fold persistence
   (`group_folds` in the fold-state file) keeps its current cross-session behavior. Only
   whole-panel intent becomes session/lifetime scoped.
7. **Merged mode is unchanged.** In merged (`_agent_panels_grouped`) mode
   `effective_panel_collapses()` already returns an empty set and the split→merged
   transition already clears panel intent; no retirement runs while merged.

### Decision recorded for review

Two readings of "start collapsed" were considered:

- **Chosen — panel-lifetime scoping** (the semantics above): intent dies with the panel,
  so in a long-lived ACE session a chop panel that disappears and reappears is collapsed
  again. This matches "always start the `@chop` panel collapsed" for sessions that stay
  up for days, which is the normal case here.
- **Rejected — session scoping** (drop cross-session persistence only): one accidental
  expand would keep `@chop` expanded for the remaining life of a multi-day session.

Both satisfy criterion 4. If the reviewer prefers the narrower one, drop Step 1 and keep
Step 2 — the rest of the plan is unchanged.

## Step 1 — Retire whole-panel fold intent when a panel dies

**Files:** `src/sase/ace/tui/actions/agents/_panel_fold_intent.py`,
`src/sase/ace/tui/actions/agents/_display_panel_collection.py`,
`src/sase/ace/tui/models/agent_panels.py`

1. Add `retire_panel_fold_intents(owner, live_keys)` to `_panel_fold_intent.py`: discard
   from `owner._collapsed_panel_keys` and `owner._expanded_panel_keys` every key not in
   `live_keys`. It returns nothing and must be a no-op when both sets are empty (the
   overwhelmingly common path) so the added cost on a hot refresh is a length check.
2. Derive the live key set in `_sync_panel_group()` (`_display_panel_collection.py:26`)
   as the union of:
   - the panel keys of the freshly computed split-mode panel group (covers tribes
     inherited through presentation anchors), and
   - `{normalize_panel_key(a.tribe) for a in self._agents_with_children if agent_is_rendered_in_agents_panel(a)}`
     — one plain loop over the **unfiltered** loaded list with no parent-lookup dict
     construction, satisfying criterion 5.

   Using the raw `Agent.tribe` field for the unfiltered half skips presentation-anchor
   inheritance, so it can only _over_-approximate liveness (a child inheriting its
   parent's tribe is covered by the rendered half). Over-approximation is the safe
   direction: it can keep an intent alive slightly too long, never retire one that
   should have survived. `Agent.tribe` is defined at
   `src/sase/ace/tui/models/_agent_state.py:322`.

3. Ordering inside `_sync_panel_group`: the panel keys are needed to retire, and the
   effective collapse set is needed to partition the panel keys. Break the cycle by
   computing the base key order once:
   - add
     `AgentPanelGroup.from_panel_keys(panel_keys, focused_key, *, collapsed_panel_keys)`
     to `src/sase/ace/tui/models/agent_panels.py` holding the existing partition/focus
     logic, and have `from_agents()` delegate to it after calling `_panel_keys_for()`;
   - in `_sync_panel_group`, call `_panel_keys_for(self._agents)` once, retire intents
     against `set(keys) | unfiltered_keys`, then compute
     `effective_panel_collapses(self, keys)` and build the group with `from_panel_keys`.

   This also removes today's `effective_panel_collapses(self)` call that passes `None`
   and therefore materializes a candidate set from every configured tribe.

4. Skip retirement entirely when `_agent_panels_grouped` is true (criterion 7).
5. Replace the "Whole-panel fold intent deliberately outlives this projection" comment
   (`_display_panel_collection.py:50-51`) with the new contract: intent outlives
   agent-set changes within a live panel, and is retired only when the panel key stops
   being live.
6. Leave the `H` isolation record alone: `_panel_isolation_revert_record()` already
   expires a record whose target vanished, and `_marked_keys_for_panel_isolation()`
   intersects with live keys, so a retired key in `collapsed_before` stays inert.

## Step 2 — Stop persisting whole-panel fold intent

**Files:** `src/sase/ace/tui/models/agent_fold_persistence.py`,
`src/sase/ace/tui/actions/agents/_fold_persistence.py`,
`src/sase/ace/tui/actions/agents/_folding_panels.py`,
`src/sase/ace/tui/actions/agents/_panel_navigation.py`,
`src/sase/ace/tui/actions/_state_init.py`

1. `agent_fold_persistence.py`:
   - Drop `collapsed_panels` and `expanded_panels` from `AgentsFoldStateSnapshot` and
     from `_encode_agents_fold_state()`.
   - Keep both names in the decoder's `allowed_fields` for **every** accepted schema
     version and ignore their contents, so existing on-disk files (which all carry them)
     still decode to their `group_folds` instead of failing open to empty. Delete the
     now-unreachable duplicate/conflict validation for those two lists.
   - Keep `SCHEMA_VERSION = 3`: files written after this change are still valid v3
     documents that an older sase release can read without losing group folds. Record
     that reasoning in the module docstring rather than bumping the version.
   - Remove `MAX_COLLAPSED_PANELS` if nothing references it after the edit (Symvision
     flags unused module symbols); update its test references accordingly.
2. `_fold_persistence.py`:
   - `_capture_agents_fold_state()` stops reading `_collapsed_panel_keys` /
     `_expanded_panel_keys`.
   - `_install_loaded_agents_fold_state()` stops assigning them from the loaded snapshot
     (lines 258-259). In-memory intent recorded before the async load resolves must
     survive the merge untouched — that is what the deleted journal entries used to
     guarantee.
   - Delete `_AgentPanelFoldIntent`, `_AgentPanelFoldsClearIntent`,
     `_record_agents_panel_fold_change()`, `_record_agents_panel_folds_cleared()`, the
     corresponding `_replay_agents_fold_intent()` branches, and narrow the
     `AgentFoldIntent` union to the group intent.
   - Keep `_ensure_agents_fold_persistence_state()` seeding `_expanded_panel_keys`,
     since the mixin still owns that default for direct-mixin tests.
3. `_folding_panels.py:34` — `_persist_panel_fold_change()` keeps the isolation-disarm
   side effect and drops the `_record_agents_panel_fold_change` call. Rename it to
   something truthful (e.g. `_note_panel_fold_change`) and update its three call sites
   plus the test doubles that define or call it
   (`tests/ace/tui/_member_jump_navigation_helpers.py:218`,
   `tests/ace/tui/_agent_neighbor_navigation_helpers.py:186`,
   `tests/ace/tui/test_agent_panel_isolation_revert.py:98`).
4. `_panel_navigation.py:360` — drop the `_record_agents_panel_folds_cleared` lookup and
   call from the split↔merged toggle; the in-memory `clear_panel_fold_intents()` above
   it is the whole behavior now.
5. `_state_init.py:556-563` — update the comment block so it says whole-panel collapse
   state is session-scoped in-memory state whose lifetime is the panel's, while the
   one-shot load / journal / writer machinery it describes now covers group folds only.

## Step 3 — Documentation and config text

1. `docs/configuration.md:809` — restate `initially_expanded` as the state applied every
   time the panel comes into existence.
2. `docs/configuration.md:816-819` — replace "Once a user explicitly expands or
   collapses a panel, that durable choice takes precedence over `initially_expanded`,
   including after ACE restarts" with the new rule: manual folds last as long as the
   panel does and never outlive it, so a restart or a panel that disappears and returns
   re-applies the config.
3. `docs/ace.md:1319-1321` — "explicit panel folds are remembered and override the
   configured initial state" is now false; rewrite it.
4. `src/sase/config/sase.schema.json:601-604` — update the `initially_expanded`
   description ("...starts expanded until the user explicitly folds it").
5. `src/sase/default_config.yml:126-133` — replace "Explicit user panel folds always
   take precedence over the panel-only `initially_expanded` setting" in the `ace.tribes`
   comment block. The bundled `chop` entry and its description text stay as they are.
6. Grep `docs/` for any other claim that panel folds are remembered across restarts
   (`docs/agent_families.md` discusses panel folds around lines 179, 495) and fix what
   is now wrong. No CHANGELOG edit — that file is release-please generated.

## Step 4 — Tests

Existing suites to update:

- `tests/ace/tui/models/test_agent_fold_persistence.py` — the round-trip, decode, and
  `expanded_panels` parametrized cases (lines 39-40, 76, 112, 140-212) must move to the
  new contract: panel fields are never written, and a v1/v2/v3 file carrying them
  decodes to its group folds with the panel fields ignored (not a decode failure, not a
  fail-open).
- `tests/ace/tui/test_agent_fold_persistence.py` — the app-level tests that call
  `_record_agents_panel_fold_change` / `_record_agents_panel_folds_cleared` (lines 77,
  103-104, 135) and the `initially_expanded` stub at line 211.
- `tests/ace/tui/test_agent_panel_collapse_isolation.py` (incl. line 306's
  `initially_expanded` stub), `tests/ace/tui/_agent_panel_collapse_helpers.py:142`,
  `tests/ace/tui/_agent_unread_navigation_helpers.py:183` — test doubles for the deleted
  recorder hooks, and any fake app that now needs an `_agents_with_children` attribute
  for `_sync_panel_group`.
- `tests/ace/tui/models/test_agent_panels.py` — cover the new `from_panel_keys` entry
  point alongside `from_agents`.
- `tests/ace/tui/models/test_tribe_display.py` needs no change:
  `effective_collapsed_panel_keys()` keeps its exact contract; only the lifetime of the
  intents passed to it changes.

New coverage to add (put retirement tests next to the existing panel-collapse suites):

1. A panel the user collapsed keeps that state while agents are added to and removed
   from its tribe (criterion 4).
2. A panel whose tribe loses every agent retires its intent, and the next appearance of
   that tribe uses `initially_expanded` — collapsed for a `chop`-like config entry,
   expanded for an unconfigured one (criteria 1, 3).
3. A query filter that hides every agent of a live tribe does **not** retire that
   tribe's intent, because liveness comes from `_agents_with_children` (criterion 5).
4. Merged mode runs no retirement (criterion 7).
5. Startup applies config rather than any persisted panel state: a fold-state file
   containing `expanded_panels: [chop]` loads, merges, and still leaves `chop`
   effectively collapsed under `initially_expanded: false` (criteria 1, 2).
6. A panel fold recorded before the async fold-state load resolves survives
   `_install_loaded_agents_fold_state()` (the regression the deleted journal entries
   used to cover).

## Verification

```bash
just install
just check
just check-full   # persistence + shared test helpers changed; run before landing
```

`just test-visual` is not expected to be needed: with a fresh state file the rendered
default is unchanged (`@chop` collapsed, everything else expanded). Run it if a PNG
snapshot touching Agents-tab panels fails in `just check-full`.

## Out of scope

- No new config keys and no change to `effective_collapsed_panel_keys()`'s signature or
  precedence rules.
- Group/workflow fold persistence stays exactly as it is.
- No Rust core work: whole-panel fold state is presentation-only Textual state, which
  `CLAUDE.md`'s core-backend boundary explicitly leaves in this repo.
- The `H` panel-isolation revert record stays session-local and unpersisted.
