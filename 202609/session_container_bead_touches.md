---
tier: tale
title: Credit session-container bead touches in the Context card
goal:
  A session row's Context-card Beads section shows closes, notes, and other bead touches
  recorded under the agent session's container name, so sase-1b1.8.2's close renders as
  a CLOSED row with its note, without cross-crediting individual turns.
size: small
proposed_by: bbugyi200.athena.0t9
create_time: 2026-09-27 16:33:14
status: wip
---

# Plan: Credit session-container bead touches to the agent session in the Context card

## Problem

The `sase-1b1.8.2` agent session (turns `sase-1b1.8.2--plan`, `--gate`, `--code`) closed
its phase bead and left a note (bead commits `chore(beads): note sase-1b1.8.2` and
`chore(beads): close sase-1b1.8.2`). But the session row's Context card ARTIFACTS
`Beads:` section still shows `sase-1b1.8.2 · assigned · read ×3`, with no `▐CLOSED▌`
pill, no `closed`/`noted` chips, and no note preview.

## Root cause (verified against live data)

1. **Write side: bead events record the session container name.** `sase bead close` and
   `sase bead note` stamp the event `actor` (and `closed_by` / the note `author`) from
   `$SASE_AGENT_NAME` (sase-core `bead/cli/parsing.rs::close_note_author`). In an agent
   session that variable holds the **session container name**. It is the name the root
   was launched under, before it was renamed to `<name>--plan`, and in-process
   continuation turns (the `%auto` plan→code flow) keep it. `sase/agent/identity.py`
   (`resolve_local_agent_name`) and `sase/monitor/store_lane.py` already document this.
   The live `sase-1b1.8` event stream has:
   - `note_appended` at 20:13:03Z and `issue_closed` at 20:13:28Z for `sase-1b1.8.2`,
     both with `actor: "sase-1b1.8.2"`. The coder turn `sase-1b1.8.2--code` started at
     19:14:21Z, and the plan turn was submitted at 19:12:49Z.
   - The reduced touch index row is
     `{actor: "sase-1b1.8.2", bead_id: "sase-1b1.8.2", verbs: {closed: 1, noted: 1}, close: {standing: true}}`.
   - `sase bead touched sase-1b1.8.2` lists `noted · closed · read ×3` for the bead,
     while `sase bead touched sase-1b1.8.2--code` reports nothing. The coder turn
     therefore ran with `SASE_AGENT_NAME=sase-1b1.8.2`.
2. **Read side: the Context-card loader never matches that actor.** In
   `src/sase/ace/tui/_bead_touches_loader.py`, the session path
   (`_load_bead_touches_for_agent_context_inner`) matches each touch's actor only
   against the members' concrete `agent_name`s (`sase-1b1.8.2--plan`, `--gate`,
   `--code`). The single-agent path (`_load_bead_touches_for_agent_inner`) matches only
   the row's own `agent_name`. Nothing ever matches the container name, so the row is
   silently dropped. The bead still gets a row only because `own_bead_ids_for_agent`
   marks it assigned. The `read ×3` comes from audited `bead:` artifact reads, which are
   attributed by artifacts dir, not by name. That is why the reads show up and the close
   does not. I reproduced this with the real index. With members built like this
   session's rows, `_load_bead_touches_for_agent_context_inner` returns `()`.
3. **Scope:** 164 rows in the current `gh_sase-org__sase` touch index have an actor that
   is a session container name no agent carries: 126 `noted`, 38 `+1`, and 33 `closed`,
   32 of them standing closes. Every session that mutated a bead is affected. Other
   CONTEXT lanes (artifact/memory/glossary reads, skill uses, opened workspaces) are not
   affected, because they attribute by artifacts dir first (`match_event_label`). Bead
   touches and synthesized `viewed` rows carry only an actor string.

The durable events are correct audit history and must not be rewritten. The fix belongs
on the read side: the session's container name is a legitimate identity spelling for the
session, and the TUI loader must accept it.

## Fix design

Treat the session container name as an **actor alias of the agent session**, and apply
it only where the whole session is in view:

- **Alias source.** Use `agent.agent_session_reference_name()`, but only when
  `agent.is_agent_session_root_entry` is true. That returns the explicit `agent_session`
  field recorded in `agent_meta.json`, falling back to the existing legacy base
  derivation. This is an exact name, not a prefix scan of other agents, so it keeps the
  no-prefix-matching rule in `agent_context_members.py`. Return `None` for a blank
  alias, and when the alias equals the row's own `agent_name` (exact matching already
  covers that case).
- **Session container row** (`build_context_members(agent)` returns more than one
  member):
  - Append one trailing matcher for the alias, after every member matcher, in both the
    durable-touch loop and the `viewed`-row loop.
  - Build its `(globalized, local)` pair with the existing `_member_match_params` and
    test it with the existing `touch_matches_agent`, so the globalized container
    spelling (for example `bbugyi200.athena.sase-1b1.8.2`) also matches.
  - Member matchers keep precedence (the loop already `break`s on the first match). A
    root that was never renamed therefore keeps its existing role label, and no touch is
    counted twice.
  - Give alias-attributed touches `agent_label=None`. The index cannot tell which turn
    acted, because one `(actor, bead)` row can merge several turns' events, so the
    loader must not guess a role. `merge_bead_touch_entries` already ignores `None`
    labels when it computes a row's shared label.
- **Single-member path** (`len(members) <= 1`): if the row is a session root entry whose
  alias differs from its own name (a root with no visible followups), match its own name
  **or** the alias. This applies to both durable touches and `viewed` rows. Durable rows
  stay ahead of view rows, and each touch is kept once.
- **Non-root member rows** (for example selecting the `--code` turn itself) get **no**
  alias. The container name cannot be pinned to one turn, and crediting it to one turn
  would cross-attribute other turns' work. The fix does not change these rows.
- **Out of scope:** Do not change the Rust reducer, the touch index schema, the
  `touch_matches_agent` facade semantics, `sase bead touched`, or the write-side actor.
  `sase bead touched sase-1b1.8.2` already lists the close.

## Implementation steps

1. In `src/sase/ace/tui/_bead_touches_loader.py`:
   - Add a small private helper, for example
     `_session_container_alias(agent) -> str | None`, that implements the alias rule
     above. Wrap the `Agent` property/method calls defensively: the loader must never
     raise. Its public entry points already catch everything, but keep failures local
     and return `None`.
   - In `_load_bead_touches_for_agent_inner`, build the matcher list: the agent's own
     `(globalized, local)` pair, plus the alias pair when present.
     - Filter the durable touches and the `viewed` rows with "any matcher matches".
       Iterate each sequence once, with `touch_matches_agent` for durable rows and
       `views_to_touches(view_events)` for views. Alternatively, call the existing
       `touches_for_agent` / `view_touches_for_agent` once per pair and concatenate them
       without duplicates, in order.
     - Keep `merge_view_touches` durable-first ordering, newest-first sorting,
       `MAX_KEPT_TOUCHES` capping, and the stat-keyed cache unchanged.
   - In `_load_bead_touches_for_agent_context_inner`, append
     `(None, *_member_match_params(alias, identity))` to `matchers` when an alias
     exists. Widen the matcher label type to `str | None`, and keep the first-match
     `break`.
   - Update the module and function docstrings to explain the container-name alias: why
     it exists (`SASE_AGENT_NAME` holds the session container for pre-rename roots and
     in-process continuation turns), that members win, that alias touches are unlabeled,
     and that non-root turn rows never take the alias.
2. Update the `BeadTouchDisplayEvent.agent_label` docstring in
   `src/sase/ace/tui/_bead_touches_shared.py`: `None` now also means a session-level
   touch that could not be credited to a single turn.
3. No rendering change is needed. With the touch now loaded, `merge_bead_touch_entries`
   folds in `closed`/`noted`, the standing `agent_close`, and the note preview, and
   `_agent_bead_touches.py` then renders the `▐CLOSED▌` pill and the note block.

## Tests

Add a **new** test module, for example
`tests/ace/tui/widgets/test_agent_bead_touches_session_alias.py`. Do not extend
`test_agent_bead_touches.py`: it is already 882 lines, over the 850 `toobig` warning
threshold. Reuse that module's patterns: the `_loader_env`-style fixture that stubs
`_project_name_for_agent`, `touch_index_path`, `AgentIdentitySnapshot`,
`globalize_owned_agent_name` and resets the three caches, plus a `_stub_query` helper.
If you share them, move them into a small shared helper or conftest; don't import
private test helpers across modules.

Build session members with `make_agent` using **distinct** `raw_suffix`, `start_time`,
and `artifacts_dir` values. `build_context_members` de-dupes members by cache key and
artifacts dir, so identical defaults collapse them into one member. Use
`agent_session="alpha"`, `agent_session_role` (`"root"`/`"code"`/`"gate"`),
`role_suffix` (`--plan`/`--code`/`--gate`), and `plan_chain_root=True` on the root.

Cover:

1. **Session container row credits the container actor.**
   - Setup: root `alpha--plan` plus followups `alpha--gate` and `alpha--code`. Touches:
     - `alpha--code` → `sase-2`
     - `alpha` → `sase-1`, with `verbs={"closed": 1, "noted": 1}` and a standing
       `BeadTouchClose`
     - `stranger` → `sase-3`
   - Expect: `sase-1` is returned with `agent_label is None`, `sase-2` is labeled
     `coder`, and `sase-3` is absent.
2. **Globalized container spelling matches.** An actor `owner.machine.alpha` (via the
   fake globalize map) is attributed to the session.
3. **Members win over the alias.** When the root itself is named `alpha` (never renamed)
   with followup `alpha--code`, an `alpha` touch keeps the root's `plan` label and
   appears exactly once.
4. **Root entry with no followups (single-member path)** named `alpha--plan` with
   `agent_session="alpha"` includes the `alpha` touch.
5. **Non-root member row gets no alias.** A lone `alpha--code` row
   (`agent_session_role="code"`, no followups) does **not** include the `alpha` touch.
6. **Viewed rows use the same alias.** A `bead_views` event whose `agent_name` is
   `alpha` surfaces as a `viewed` touch on the container row. Stub the views snapshot,
   for example by monkeypatching `bead_views_log_path` and `read_bead_view_events` in
   the loader module, or `_load_view_snapshot`.
7. **End-to-end merge regression for the reported bug.** Feed case 1's loader output
   into `merge_bead_touch_entries(events, (), own_bead_ids_for_agent(root))`, where the
   root has `phase_bead_id="sase-1"`.
   - The `sase-1` entry is `own`, its `agent_close.standing` is true, and its verbs
     include `closed` and `noted`.
   - Rendering it with `append_agent_bead_touch_rows` produces the `CLOSED` pill text.

All existing tests in `tests/ace/tui/widgets/test_agent_bead_touches.py` and
`tests/ace/tui/test_agent_context_members.py` must keep passing unchanged. In
particular, ordinary non-session agents keep exact-name matching.

## Verification

- Run `just check` (through `sase tool run`, as the lint/test memory requires). The
  scoped lane should select the new module and the existing bead-touch tests.
- Optional manual check: open `sase ace`, select the `sase-1b1.8.2` session row, and
  confirm that the Context card `Beads:` row now shows `▐CLOSED▌`, the `noted` chip, and
  the D10 bench note preview. The first `sase-1b1.8.1` note (actor `sase-1b1.8.1`,
  written before the root was renamed to `--plan`) should now appear on that session row
  as well. No visual golden needs to change: the existing fixtures do not use
  container-named actors. If a golden does change, regenerate it only as the TUI memory
  describes.

## Follow-up to record (not part of this tale)

This fix repairs display for all existing history. Separately, the write side could
record the concrete turn (metadata-first, like `resolve_local_agent_name`) for bead
mutations made by in-process continuation turns, which would give future touches exact
per-turn labels. That change crosses into sase-core (`close_note_author` and the bead
CLI actor) and interacts with the monitor/lane code that relies on `SASE_AGENT_NAME`
holding the container. Capture it as a proposed follow-up in the final summary rather
than changing it here.
