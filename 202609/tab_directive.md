---
tier: tale
title: "%tab launch path, storage, query field, and completion"
goal: "A launch can place its presentation root with %tab. The canonical name is stored
  on agent metadata, kept across retry, revive, fork, session follow-ups, and clan
  joiners, loaded onto Agent from meta, index, and fleet rows, and queried and completed
  as tab:. The directive is stripped before the model sees the prompt.

  "
size: medium
proposed_by: bbugyi200.athena.sase-1bc.4
bead: sase-1bc.4
create_time: 2026-09-27 12:54:50
status: wip
---

- **PARENT:**
  [202609/agents_dynamic_tabs.md](https://github.com/sase-org/sase--plans/blob/main/202609/agents_dynamic_tabs.md)
- **BEAD:**
  [sase-1bc.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1bc/sase-1bc.4.md)

# Plan: %tab launch path, storage, query field, and completion

Implement epic phase `tab-directive` (bead `sase-1bc.4`) from
`plan:202609/agents_dynamic_tabs.md`. This tale is that one phase. Leave the parent epic
`sase-1bc` and every ancestor open. The directive, storage, wires, and `tab:` query are
ungated: there is no `agent_tabs` flag in this phase.

The landed sase-core commits this phase consumes are already on `origin/master` of the
linked sase-core checkout:

- `c448ed6d8da9e6874c16c4f9cfe7e1459922ac7a` — canonicalizer, `tab` directive contract
  (`COLON_PAREN`, no alias, no keywords, single-valued), typed-unit `agent_tab`, and the
  `canonicalize_agent_tab_name` Python binding.
- `0e8981a1f131d2dd040c4887ae949edf19fbeef6` — scan wire schema 11
  `AgentMetaWire.agent_tab` / `agent_tab_source`, and fleet contract 7 `agent_tab` on
  the owner and summary wires.

`canonicalize_agent_tab_name` returns `{"kind": "named", "name": "<lower>"}` or
`{"kind": "default"}` for `main`. Failures are `ValueError` whose text is the
user-facing message. Surface that text unchanged:

- empty: the core empty-name message
- `local`: `machine tabs are derived; omit %tab to land on the local machine tab`
- `all`: `there is no 'all' tab; use o in the grouping picker to see all tabs`
- other invalid input: the core grammar message

`main` is stored as absent. `%t` stays the retired-tribe migration error. There is no
`%tab` alias. Machine aliases are ordinary names: `%tab:apollo` is a named tab. Fenced
and disabled regions stay ignored by the existing literal-zone pipeline.

## Pin and Python mirrors

`sase-core-revision.txt` is `73f104486e9827fe7d83295ce5c80bf13d8b6dc3`, which is an
ancestor of the commits above. Move it to the linked sase-core `origin/master` SHA when
`0e8981a1f131d2dd040c4887ae949edf19fbeef6` is an ancestor of that SHA. At authoring time
that SHA is `0e8981a1f131d2dd040c4887ae949edf19fbeef6`. If the pin already names a
descendant of that commit, leave the pin where it is. `just check` rebuilds
`sase_core_rs` from the linked checkout, so local tests see that tree once setup runs.

`sase-1ab.10.4` (in progress) owns narrowing the shell-to-turn dual-core window. Leave
these as they are:

- `AGENT_SCAN_WIRE_SCHEMA_VERSION = 10` and
  `SUPPORTED_AGENT_SCAN_WIRE_SCHEMA_VERSIONS = {10, 11}` in
  `src/sase/core/agent_scan_wire_records.py`
- fleet assertions that accept contract 6 and 7, and fleet protocol 2 and 3
- `pyproject.toml`'s `sase-core-rs` version window (`just ratchet-core-window` is
  release-owned)

Add the new optional fields so schema 11 and contract 7 payloads round-trip:

- `AgentMetaWire.agent_tab` and `agent_tab_source` (`str | None = None`) in
  `src/sase/core/agent_scan_wire_markers.py`, beside `tribe`.
- `AgentUnitWire.agent_tab: str | None = None` in
  `src/sase/core/agent_launch_wire_records.py`.

Wire JSON that omits the keys still decodes. Add `src/sase/core/agent_tab.py`, a thin
wrapper around `require_rust_binding("canonicalize_agent_tab_name")`. Return the stored
name (`str`) or `None` for kind `default`. Raise the binding's `ValueError` with the
core message. This phase calls only the canonicalizer.

## Python `%tab` parsing

In `src/sase/xprompt/_directive_types.py`:

- Add `tab` to `_KNOWN_DIRECTIVES`. Leave it out of `_MULTI_VALUE_DIRECTIVES` and
  `_DIRECTIVE_ALIASES`.
- On `PromptDirectives`, add `agent_tab: str | None = None` (canonical stored name) and
  `agent_tab_explicit_default: bool = False` (`True` only for an explicit `%tab:main`).

In `src/sase/xprompt/_directive_collect.py`, validate parenthesized `%tab` the way
`%dispatch` is validated: no keywords, exactly one positional, a clear error for a
missing `)`. Reject `%tab+` the way `%dispatch+` is rejected. Duplicates already fail in
`_store_single_directive` once `tab` is single-valued; keep that message
(`Duplicate directive '%tab' in prompt`), including when the two values agree. Fan-out
`%{%tab:a | %tab:b}` stays legal because each branch is its own unit.

In `src/sase/xprompt/_directive_values.py`, add `resolve_agent_tab` that calls the
adapter. Absent key means no directive. Kind `named` sets `agent_tab` and leaves the
explicit-default flag false. Kind `default` sets `agent_tab` to `None` and the flag
true. A core `ValueError` becomes `DirectiveError` with the same text. Wire the result
in `src/sase/xprompt/_directive_extract.py` next to `resolve_dispatch_target`.

After extraction, reject `%tab` together with `%proc` (either `agent_tab` set or
`agent_tab_explicit_default`) with a specific `DirectiveError`:
`%tab cannot be used on a stand-alone %proc unit`. Known directives are already removed
from the model prompt via `regions_to_remove`; keep `%tab` on that path so the model
never sees it. The raw prompt keeps the directive.

Update `tests/test_xprompt_directive_contract.py` so the core contract comparison stays
exact: keywords `tab: ()`, syntax `tab: (colon, parenthesized)`, no alias, no feature
flag. The installed core after the pin exposes `tab`, so this test fails until the
runtime vocabulary includes it.

## Metadata writes

`build_agent_meta` in `src/sase/axe/run_agent_directive_metadata.py` applies the
directive after `inputs.preserved` is merged:

- named tab: set `agent_tab` to the canonical name and `agent_tab_source` to `prompt`
- explicit `%tab:main`: remove both keys
- omitted `%tab`: leave any preserved `agent_tab` / `agent_tab_source` in place

Write `agent_tab` only for a named tab. Valid preserved sources are `prompt` and
`moved`; this phase writes only `prompt`, and retry must keep a preserved `moved` value
when the prompt has no `%tab`.

Add `agent_tab` and `agent_tab_source` to the string keys `preserved_agent_metadata`
copies, so a runner re-exec keeps them. That is the retry path.

Session follow-ups: `_add_agent_session_metadata` copies the session root's stored tab
into the child meta. Resolve the root artifact (the session's first turn), not a later
member the `%id(..., session=...)` parent happens to name.
`AgentSessionAttachLaunchPlan.parent_artifacts_dir` is the attach parent; use it when
that directory is the root, and otherwise locate the root member's `agent_meta.json`
from the attach snapshot. Read `agent_tab` from that file.

In `src/sase/axe/run_agent_directives.py`, beside the tribe/session guards:

- omitted `%tab` on a follow-up copies the root tab (including copying nothing when the
  root has none) and sets `agent_tab_source` to `prompt` when a name is copied
- an explicit `%tab` whose stored value differs from the root is a `DirectiveError` that
  names the root tab (`main` when the root has none) and `sase agent tab set`
- an explicit `%tab` that canonicalizes to the same stored value is allowed
- `%tab:main` differs from a named root tab

Clan joiners, same generation: the generation id is the declarer's artifacts directory
basename (`_is_new_clan_generation` in `src/sase/axe/run_agent_directive_clans.py`). The
joiner's artifacts directory is a sibling of that directory. When the joiner omits
`%tab`, copy `agent_tab` from `<workflow_dir>/<generation>/agent_meta.json`. Apply the
same mismatch rule as sessions, as a `DirectiveError`, when the joiner's explicit `%tab`
differs from that generation tab. The declarer itself writes only its own `agent_meta`
keys; it does not invent a second store.

Durable clan-record gap: `ClanRecordUpdateWire` / `ClanGenerationRecordWire` /
`ClanLaunchDefaultsWire` in sase-core have tribe, summary, and summary script, and
unknown update keys are dropped. Phases 2 and 3 did not add `clan_tab`. A core schema
edit in this same turn cannot be named by the pin before that commit exists, and CI
builds the pin, so persisting through `record_clan_attributes` would pass locally and
fail in CI. Leave sase-core untouched. Record this on the phase bead before close:

`PROPOSED FOLLOW-UP: clan record schema has no clan_tab — add it beside tribe on ClanGenerationRecordWire, ClanRecordUpdateWire, and ClanLaunchDefaultsWire, then write it from record_clan_attributes_at_launch and inherit it in apply_clan_launch_defaults so a new generation keeps the tab after the declarer artifacts directory is gone.`

Same-generation inheritance above covers joiners while the declarer directory still
exists. `apply_clan_launch_defaults` stays tribe/summary only.

Revive: `ArtifactRestorationMixin._build_agent_meta_data` in
`src/sase/ace/tui/actions/agents/_revive_artifacts.py` writes `agent_tab` when the
`Agent` has one. `_restore_agent_meta` merges onto existing JSON, so an existing
`agent_tab_source` remains. Restoring a revived agent that has a tab keeps both keys.

Fork prefill: when the source `Agent.agent_tab` is set, the TUI fork prefix in
`_complete_agent_fork_scope` (`src/sase/ace/tui/actions/agents/_fork_actions.py`) and
the mobile fork prompt in `fork_mobile_agent`
(`src/sase/integrations/_mobile_agent_lifecycle.py`) start with `%tab:<name> ` before
the existing `#fork:` text. The user can edit it. A source with no stored tab gets no
`%tab` prefill.

Dismissed bundles: `agent_state_to_bundle_dict` persists Agent fields that are outside
`_RUNTIME_ONLY_BUNDLE_FIELDS`. Leave `agent_tab` out of that skip set so dismissal keeps
it.

## Agent model and fleet rows

Add `agent_tab: str | None = None` on `Agent` next to `tribe` in
`src/sase/ace/tui/models/_agent_state_session.py`.

Fill it from:

- filesystem meta in `_meta_enrichment_filesystem.py`, after the tribe assignment, by
  running the raw string through the canonicalizer and dropping invalid values (the
  scanner already drops them; a bad file must not fail the load)
- wire meta in `_meta_enrichment_wire.py` when `meta.agent_tab` is set
- fleet conversion `_agent_from_summary` in
  `src/sase/ace/tui/models/_fleet_agents_rows.py` via
  `optional_str(summary.get("agent_tab"))`, which is absent on contract-6 rows
- synthesized remote session containers in `_remote_agent_session_container`
  (`src/sase/ace/tui/models/_fleet_agents_nodes.py`) by copying `anchor.agent_tab`, the
  same way `agent_clan` is copied from the anchor
- provisional dispatch rows in `_agent_from_dispatch_parts`
  (`src/sase/ace/tui/models/dispatch_launch_rows.py`): parse the launch prompt with the
  directive parser and set `agent_tab` from a successful named tab

## `tab:` query

`tab:` is an exact-match field, like `tribe`, and a known-fallback field. Rows with no
stored tab match `main` and no other value.

- Add a `QueryFieldSpec(key="tab", exact_match=True, ...)` beside `tribe` in
  `src/sase/ace/query_profile/profiles/_agents_live.py`. The hint says stored tab names,
  or `main` for the default tab.
- Project it in `src/sase/ace/tui/models/agent_live_query.py`: the stored canonical
  name, or `main` when `agent.agent_tab` is empty.
- Add `"tab"` to `KNOWN_FALLBACK_FIELDS` in
  `src/sase/ace/tui/models/agent_live_query_pushdown.py`.
  `tests/test_agent_query_pushdown.py` requires every profile key to be pushable or in
  that set.
- Legacy dialect: add `tab` to `SUBSTRING_PROPERTY_KEYS` in
  `src/sase/ace/agent_query/tokenizer.py` so `tab:<value>` tokenizes. Leave bare `tab:`
  on the normal value parser (bare `tribe:` / `machine:` stay the only empty-value
  keys). In `evaluator.py`, match case-insensitively: `tab:main` matches a missing
  stored tab; any other value matches that stored name; a named row does not match
  `main`.

Update profile tests and any query golden or snapshot whose field list gains `tab`.

## Completion

Emit inventory entries of kind `tab` so the core `DirectiveValueRole::Tab` completer
(already shipped) can suggest them. Include every distinct stored `agent_tab` on the
loaded roster, plus `main` even when no row has a tab.

- `src/sase/ace/tui/_agent_completion_candidates.py`: extend the candidate `kind`
  literal with `"tab"` and build one candidate per name. The display name is the tab
  name, with no `@` prefix.
- `src/sase/integrations/_editor_helper_agents.py`: same kind `"tab"` entries for the
  editor inventory, next to `_tribe_entries`.

## Docs and memory

Docs:

- `docs/xprompt.md`: a row in the Supported Directives table (`%tab`, no alias) and a
  `### Tab Directive` section after the retired-tribe paragraph. Document the grammar,
  `%tab:main`, the `local` / `all` errors (quote the core messages), single-valued
  duplicates, fan-out, the `%proc` rejection, clan and session inheritance, and that the
  directive is stripped from the model prompt.
- `docs/ace.md` Agent Search property table: `tab` is exact, and `main` means no stored
  tab. Mention `tab:` beside the `tribe:` example in the recent-history sentence.
- `docs/query_language.md`: in the boolean-dialects bullet, state that the Agents tab
  filter's `tab:` field is exact (`main` for rows with no stored tab) and is specified
  in the ace.md Agent Search table. `tab:` is an Agents field, not a Patch property.

Memory, authorized by bead `sase-1bc.4`: add one `%tab:<name>` row to the directive
table in `sase/memory/xprompts.md`. No alias. Say it places the launch's presentation
root, `%tab:main` is the default and is stored as absent, and `%t` remains the retired
tribe migration. Follow `/sase_memory_write`: `sase skill use sase_memory_write` before
the edit, then `sase memory init`. Leave generated `AGENTS.md` and provider shims to
that command.

## Tests

Cover, in the existing directive, clan, session, revive, fork, fleet, and query
harnesses where those harnesses already exist:

- `%tab` absent, valid, uppercase (`%tab:Sase` stores `sase`), invalid, duplicated (even
  when both values match), `local`, `all`, empty, inside a fence, and inside an xprompt
- fan-out branches on different tabs, and a swarm's segments each keeping their own tab
- `%tab` plus `%dispatch` is accepted and the dispatch prompt still carries the tab;
  `%tab` plus `%proc` is rejected
- clan joiner and session follow-up inheritance, and mismatch errors that name the root
  tab and `sase agent tab set`
- retry (preserved meta), revive, and fork prefill
- scan-wire and fleet-row loading, including a contract-6 summary with no `agent_tab`
- `tab:blog` and `tab:main` in the agents-live profile and the legacy dialect

## Verification and close

Run `just check` (not `check-full`). A failure that also fails on the clean tree against
the same linked sase-core checkout is pre-existing: record `PROPOSED FOLLOW-UP:` on
`sase-1bc.4`, cite `sase-1ab.10.4` when the failure is the turn-rename pin bump, and
close this phase anyway.

Before close, run `sase bead epic-symbols sase-1bc.4`. This phase has no `--epic-symbol`
lines in the Justfile at authoring time. If any are present then, retarget each one at a
still-open bead (the parent epic or a later phase) before closing. Close only
`sase-1bc.4`:

`sase bead close sase-1bc.4 --note "<what just check and the tab tests verified>"`

The note names the pin SHA, that `%tab` is stored, loaded, queryable, and stripped from
the model prompt, and the clan-record follow-up.
