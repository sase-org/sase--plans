---
tier: epic
title: Agent-closed beads stand out in the Context card
goal: 'When a SASE agent closes a bead, that agent''s Main-deck Context card lists
  the bead in SASE CONTEXT › ARTIFACTS › Beads with an unmistakable CLOSED pill. The
  close is credited to the agent that actually closed it (never the bead''s creator),
  stays visible even when the agent touched many other beads, and reads as one feature
  across the card and `sase bead touched`.

  '
phases:
- id: core
  title: Close attribution and close facts in the touch index (sase-core)
  depends_on: []
  size: medium
  description: 'core: stamp issue_closed events with the acting closer plus a durable
    closed_by payload field, credit closes in the touch-index reducer (with legacy
    same-instant note recovery), add a per-touch close record with resolution/reason/standing,
    and bump the index schema.'
- id: plumbing
  title: Python close actor, facade, merge, glyph precedence, and CLI parity
  depends_on:
  - core
  size: small
  description: 'plumbing: move the sase-core pin, make `sase bead close` always pass
    the acting agent, parse the new close record through the facade into BeadTouchEntry,
    promote closed above created in the shared glyph precedence, and expose the record
    in `sase bead touched --json`.'
- id: render
  title: CLOSED pill rendering, visibility guarantee, and goldens (TUI)
  depends_on:
  - plumbing
  size: medium
  description: 'render: paint agent-closed rows with a green check, a capped CLOSED
    pill, resolution and reopened-since states, closed-reason precedence, a lane-header
    closed count, and closure-first visible-row selection, then cover it with unit
    tests and new PNG goldens.'
proposed_by: bbugyi200.athena.0s0
create_time: 2026-09-25 14:05:24
status: wip
bead_id: sase-19p
---

- **BEAD:** [sase-19p](https://github.com/sase-org/sase--beads/blob/main/pages/sase-19p/README.md)

# Plan: Agent-closed beads stand out in the Context card

## Why this is more than a styling change

The `Beads:` sub-section of ARTIFACTS
(`src/sase/ace/tui/widgets/prompt_panel/_agent_bead_touches.py`) already renders a
`closed` verb chip and a `✓` glyph. But that chip almost never appears on the card of
the agent that did the closing. The cause is in sase-core:

- `close_issues_with_note` (`crates/sase_core/src/bead/mutation/close_remove.rs`)
  appends every `issue_closed` event with `&event.issue.created_by` as the event actor,
  which is the bead's **creator**. It is not the agent that ran `sase bead close`. The
  `--note` that `close` writes in the same mutation is correctly attributed to the
  closer (`resolve_mutation_author`).
- Live evidence, from `sase bead history` on this project: phase `sase-19f.2` was closed
  at `17:34:01Z` together with a note by `sase-19f.2`. The `issue_closed` event is
  stamped `bbugyi200.apollo.1o`, which is the planner that created the epic. The same
  pattern holds for `sase-198.3`, `sase-198`, and `sase-17m.5`. The touch index
  (`crates/sase_core/src/bead/touch_index.rs`) reduces events by actor, so the planner's
  card shows `closed` for every phase it never closed, and the phase worker's card shows
  only `noted`.
- Human closes (TUI close action, TaskTriage cancel, stale-cleanup gates) are also
  stamped with the creator. When an agent created the bead, that agent is falsely
  credited with a human's close.

A prominent CLOSED treatment on top of that data would make the misattribution _more_
visible. So this epic fixes attribution first (`core`), carries the corrected facts
through Python (`plumbing`), and only then paints the design (`render`).

## Design decisions

1. **Durable closer on new events.** Every new `issue_closed` event records the acting
   closer in two places: the envelope `actor` (so `sase bead history` is truthful) and a
   new optional payload field `closed_by`. The payload field is the explicit marker that
   says the actor is the closer. Without it, a new event where an agent closes a bead it
   created can't be told apart from a legacy creator-stamped event. Human/owner closes
   record the store owner, which is not an agent, so they correctly produce no agent
   touch.
2. **Honest legacy recovery.** Legacy close events carry no `closed_by`, and their actor
   is the creator by construction, so that actor says nothing about who closed. The
   reducer credits a legacy close only through the `close --note` fingerprint: the
   unique valid agent that appended a note anywhere in the same stream at the identical
   instant (one mutation shares one `now`). If there is no such agent, or more than one,
   the close is credited to nobody. It never falls back to the creator. This moves
   historical closes onto their real closers where the evidence exists. It also strips
   the false `closed` verbs from planners' cards.
3. **Core owns the close facts.** Whether the agent's close still stands, and its
   resolution and reason, are computed in the Rust reducer (see the Rust Core Backend
   Boundary rule). Python only displays them.
4. **One visual anchor word.** The pill always reads `CLOSED`, because the user scans
   for that word. Its color carries the resolution: green for `done`, and neutral grey
   plus a resolution chip for `canceled`/`superseded`. A close that was later undone
   loses the pill and says so, so the card never claims a close that no longer stands.
5. **Closed beads are never hidden.** Visible-row selection gives standing closes the
   first claim on the five visible slots, while rows still display newest-first.
6. **No feature flag.** This is an attribution bug fix plus an additive display. There
   is no beta behavior and no deprecated branch. No keymap or `default_config.yml`
   change is involved.

## Target look (80-cell logical text; styles annotated below)

```
▸ ARTIFACTS · 4 beads (✓ 2 closed) · 2 reads · 1 commit
  Beads:
    17:34:01  ✓ [3] sase-19f.2 ▐CLOSED▌ · own · noted ×2
                    │ 17:34 · sase-19f.2
                    │ Phase checks green; pill verified in split cards.
                    ↳ Render agent-closed beads distinctly
    17:20:45  ✓ [4] sase-1a2 ▐CLOSED▌ · canceled · created
                    ↳ canceled: duplicate of sase-19z
    16:58:02  ✎ [5] sase-19f · noted
                    ↳ Agent-closed beads in the Context card
    16:02:55  ✓ [6] sase-17m.4 · noted · closed · reopened since
                    ↳ Legacy close attribution
```

- Standing `done` close: `✓` glyph `bold #5FD75F`, and the pill is `▐` (`#5FD75F`) +
  `CLOSED` (`bold #1A1A1A on #5FD75F`) + `▌` (`#5FD75F`). The half-block caps give a
  padded pill with no inner spaces, so Rich word-wrap can never split it in narrow or
  split cards. The plain `closed` chip is dropped because the pill replaces it.
- Standing `canceled`/`superseded` close: the same pill and glyph in `#8A8A8A`, followed
  by a chip holding the raw resolution (`canceled`) styled `italic #BCBCBC`. Any
  resolution other than `done` takes this branch.
- Non-standing close (the bead was reopened after this agent's close): no pill, and the
  glyph keeps the normal bead style. The `closed` chip is styled `dim strike` and is
  followed by a `reopened since` chip styled `italic #D7AF5F`.
- `↳` line precedence for a row with a _standing_ close whose reason is non-empty:
  `closed: <reason>` for `done`, `<resolution>: <reason>` otherwise. That beats read
  reasons and the title, mirroring the existing read-reason-over-title rule. Every other
  row keeps today's precedence.
- Lane header: the `N bead(s)` detail gains ` (✓ K closed)` when K standing closes
  exist. The parentheses are dim and `✓ K closed` is `bold #5FD75F`, so a folded lane
  still announces closes. Build the details as `Text`, which
  `append_context_lane_header` already accepts.
- Rows without an agent close render exactly as today.

## Phase `core` — close attribution and close facts (sase-core)

Work in the linked `sase-core` checkout (`sase repo open sase-core`) and read its
`AGENTS.md` first.

1. **Payload.** Add `closed_by: Option<String>` to `BeadEventPayloadWire::IssueClosed`
   (`crates/sase_core/src/bead/events/wire.rs`) with
   `#[serde(default, skip_serializing_if = "Option::is_none")]`. Legacy events must
   round-trip byte-identically. Update every constructor and pattern match (grep
   `IssueClosed {` across `events/`, `mutation/`, `history.rs`, `read.rs`, and tests).
   When `closed_by` is present, `validate_for` rejects a blank value.
2. **Mutation actor.** In `close_issues_with_note`, rename `note_author` to
   `actor: Option<String>`. Resolve it once: trimmed non-blank value, else
   `store.config.owner`. Use it for the note author (unchanged behavior) and as the
   envelope actor plus `closed_by` on **every** `issue_closed` event in the batch,
   covering requested ids, `--force`-swept descendants, and auto-closed delegated parent
   phases. That replaces `&event.issue.created_by`. `close_issues` passes `None`. Keep
   the pyo3 `bead_close` arity: its existing `author` argument now means the close
   actor. Update the binding docs in `crates/sase_core_py/src/beads/`.
3. **Rust CLI parity.** `handle_close`
   (`crates/sase_core/src/bead/cli/mutate_commands.rs`) passes `close_note_author()`
   unconditionally, not only when `--note` is given, and `close_note_author()` must
   treat the bare launcher flag `SASE_AGENT=1` as no identity. That mirrors Python's
   `discover_agent_identity`. The Python fast path skips `close` today, but the Rust CLI
   must not disagree.
4. **Reducer crediting** (`crates/sase_core/src/bead/touch_index.rs`):
   - Parse a close detail (`closed_by`, `resolution`, `close_reason`) for `issue_closed`
     in `ParsedEvent`.
   - For `issue_closed` only, the credited actor is `agent_actor(closed_by)` when
     `closed_by` is present. Humans and the owner get no credit, and legacy recovery is
     never applied to such an event. Otherwise, apply the legacy rule from Design
     decision 2: collect the valid agent authors of `note_appended` events per exact
     instant (`at` equality) across the whole stream, and credit only a unique author.
     All other operations keep using the envelope actor exactly as today.
   - Uncredited closes contribute no verb and no timestamp to any touch. Bead metadata
     (`status`) still follows every event.
5. **Close record.** Add
   `BeadTouchCloseWire { closed_at: String, resolution: String, reason: Option<String>, standing: bool }`
   and `BeadTouchWire.close: Option<BeadTouchCloseWire>`, the latter with
   `#[serde(default, skip_serializing_if = "Option::is_none")]`.
   - Record the actor's latest credited close for that bead, in stream order.
   - `resolution` is the payload resolution in snake_case, `done` when absent.
   - `reason` is the trimmed `close_reason`, `None` when blank.
   - `standing` is true iff the bead's final reduced status is `closed` **and** this
     event is the bead's last `issue_closed` event in stream order, credited or not.
     Re-closing an already-closed bead writes no event, so any later close implies a
     reopen in between.
   - Known limit, to document in the doc comment: a `task_plus_one_recorded` reopen is
     not replayed, so such a bead reads as standing until its next status event.
6. **Schema.** Bump `BEAD_TOUCH_INDEX_WIRE_SCHEMA_VERSION` to `3`. Every existing index
   becomes a miss and fully rebuilds, so legacy streams are re-reduced under the new
   rule. Update the module docs: the "nothing new is tracked at write time" claim now
   has the `closed_by` exception. Also update the fixture
   `tests/fixtures/bead/touch_index/expected_query.json` and the shape-contract test.
7. **Tests** (Rust, beside the existing ones):
   - Close mutation: the supplied actor lands on the envelope and on `closed_by`, with
     owner fallback. Forced descendants and the delegated parent share the actor. A
     close without a note still records the actor.
   - Legacy events without `closed_by` still parse and re-serialize unchanged.
   - Reducer, new format: a closer that is not the creator gets `closed` plus
     `close.standing`, and the creator gets no `closed`. A human `closed_by` produces no
     touch.
   - Reducer, legacy: unique same-instant note author is credited. Two different
     same-instant authors are ambiguous, and no note means no credit (the creator is no
     longer credited).
   - Close record: a later reopen gives `standing: false`; reopen then close by someone
     else gives `standing: false`; resolution and reason pass through, with a missing
     resolution reading as `done`.
   - A schema-2 index is a cache miss.
   - The pyo3 binding test shows that `bead_close(..., author)` stamps the close actor.
8. Verify with `sase tool run check` from inside the sase-core checkout. The
   reopen-attribution bug (`open_issue` and `reopen_closed_ancestors` also stamp
   `created_by`) is **out of scope**. Record it as a `PROPOSED FOLLOW-UP:` note on this
   phase bead.

## Phase `plumbing` — Python close actor, facade, merge, glyphs, CLI (sase)

1. **Pin and local core.** Once `core` is pushed, move `sase-core-revision.txt` with
   `just ratchet-core-revision`. Confirm the new pin contains the `core` commit
   (`git merge-base --is-ancestor` in the opened sase-core checkout). Then run
   `just rust-install`, which may need `/sase_monitor`, so local tests use the new
   binding.
2. **Close actor.** In `handle_bead_close` (`src/sase/bead/cli_crud_lifecycle.py`),
   always resolve `author = resolve_mutation_author(mutation.project)` and pass it, with
   or without `--note`. Update the `author` docs on `BeadProject.close`
   (`src/sase/bead/_project_mutations_lifecycle.py`) and
   `sase.core.bead_mutation_facade.close` to say it is the actor recorded on the note
   and every close event. Leave the other callers passing nothing, because each is a
   human decision that should record the store owner: the TUI close action, the task,
   snooze, flag, and stale-cleanup gate actions, and the external mirror.
3. **Facade.** In `src/sase/core/bead_touch_index_facade.py`, add a frozen
   `BeadTouchClose(closed_at: str, resolution: str = "done", reason: str = "", standing: bool = False)`
   and `BeadTouch.close: BeadTouchClose | None = None`. Parse with a tolerant
   `_close_from_dict` that mirrors `_note_preview_from_dict`:
   - A non-mapping value or a blank `closed_at` gives `None`.
   - A non-string or blank resolution gives `done`, and the reason is stripped.
   - `standing` is true only for a real `True`. Export the new name.
4. **Merge.** In `src/sase/ace/tui/bead_touches.py`, add
   `BeadTouchEntry.agent_close: BeadTouchClose | None = None`. Fold it in `_BeadBucket`
   from every contributing touch: prefer a standing close, then the newest `closed_at`
   by parsed moment. View rows and audited reads carry none. Session rows need no
   special handling: every contributing touch is already this session's.
5. **Glyph precedence.** In `src/sase/bead/touch_glyphs.py`, change the precedence to
   closed > created > reopened > edited group > read > viewed > removed, because the
   terminal outcome outranks the origin. Update the docstring and the tests (for
   example, `tests/test_bead/test_cli_touched.py` asserts `✚` for created+closed; that
   becomes `✓`).
6. **CLI parity.** `sase bead touched --json` rows gain
   `"close": {closed_at, resolution, reason, standing}` or `null` (see
   `src/sase/bead/cli_touched.py` near the row dict). Text output changes only through
   glyph precedence. Before touching parser or help text, read the CLI rules memory
   (`cli_rules.md`); this adds no new flags.
7. **Tests:**
   - Facade parsing: valid, missing, and malformed close records.
   - The merge fold.
   - Glyph precedence.
   - `touched --json`.
   - An end-to-end regression on a temporary bead store:
     - agent A creates a bead, and agent B closes it without `--note`;
     - refresh the touch index;
     - B's touches include `closed` with `close.standing`, A's do not, and
       `sase bead history` shows B on the close.
   - Check `tests/test_bead/test_cli_history.py` and
     `tests/test_bead/test_snooze_close_regression.py` for close-actor expectations, and
     update any that encoded the creator.
8. Verify with `sase tool run check`.

## Phase `render` — CLOSED pill, visibility, goldens (sase TUI)

Read the `tui`, `tui_perf`, and `tui_screenshot` memories first. The change is pure
rendering over already-loaded entries. Add no I/O on the render or navigation path.

1. **Constants.** In `_agent_context_common.py`, add the named colors from "Target
   look": closed green `#5FD75F`, muted `#8A8A8A`, pill text `bold #1A1A1A`, the
   resolution chip `italic #BCBCBC`, and reopened-since `italic #D7AF5F`. Do not inline
   hex strings.
2. **Rows.** In `_agent_bead_touches.py`, change `append_agent_bead_touch_rows` to
   produce the look above:
   - glyph style chosen from `entry.agent_close`;
   - the pill after the bead id;
   - the chip rewrite: drop `closed` under a pill, add the resolution chip, or add the
     struck `closed` plus `reopened since`;
   - the standing-close `↳` precedence. Keep `assert cell_len(glyph) == 1`.
3. **Visibility.** Add one `visible_bead_entries(entries)` helper. Standing closes claim
   visible slots first (newest first, up to `MAX_VISIBLE_BEADS`), the remaining slots
   fill by rank, and the chosen rows render in the original newest-first order. Use it
   in both `_bead_touch_hint_labels` and `append_agent_bead_touch_rows`, so hint numbers
   and rows can never diverge. The overflow footer's "earliest" time comes from the
   hidden rows. Only in the rare case where more standing closes exist than slots, the
   footer appends ` (✓ K closed)`.
4. **Header.** In `_agent_artifacts_lane.py`, build the details as `Text` with the
   ` (✓ K closed)` suffix on the beads count.
5. **Unit tests** in `tests/ace/tui/widgets/test_agent_bead_touch_rows.py`:
   - exact plain text and span styles for standing `done`, standing `canceled`,
     non-standing, and no-close rows (the last unchanged);
   - `↳` precedence;
   - header count;
   - visible selection with more than five entries, where an older standing close stays
     visible and hints stay aligned;
   - a narrow `Console` render (for example width 40) in which `▐CLOSED▌` stays
     contiguous.
6. **Goldens.** Add `agents_bead_closed_by_agent_120x40`, plus a split/narrow variant at
   `90x32`, to `tests/ace/tui/visual/test_ace_png_snapshots_agents_sase_context.py`.
   Model them on `test_agents_bead_note_preview_png_snapshot` by monkeypatching
   `load_bead_touches_for_agent_context` with touches that cover all four row states.
   Run `just fix-tui-screenshots -- <selectors>` through `/sase_monitor` when it is
   long, then inspect the report. Look at every creation, plus any existing golden whose
   fixture carries a `closed` verb and therefore shifts. If the half-block caps render
   misaligned in the snapshot renderer, fall back to a flat unpadded
   `bold #1A1A1A on <color>` `CLOSED` badge and note the reason in the test docstring.
7. Optionally confirm in a live `sase screenshot` of a real agent that closed a bead
   (the Main deck, Context card). Verify with `sase tool run check`.

## Compatibility and rollout notes

- Older sase-core builds ignore `closed_by` when reading. A stream merge run by an older
  core may drop the field, which only degrades that event to the legacy rule. It is
  never an error.
- The schema bump means the Context card shows no bead rows until the first post-upgrade
  refresh (post-mutation, post-sync, or lumberjack tick) rebuilds the index. A
  long-running TUI keeps its imported core until restart; the stale-code detector
  already surfaces that.
- `sase bead touched` output changes intentionally for history. Creators lose
  misattributed `closed` verbs. Closers who used `--note` gain them. Historical closes
  without a note show no closer.

## Out of scope

- Reopen attribution (`issue_opened` stamps `created_by`). This is the `core` follow-up.
- Clan/tribe Context aggregation styling.
- A `closed_by` field on issues, or "Closed by" in `sase bead read`.
- Replaying `+1` reopens in the reducer.
