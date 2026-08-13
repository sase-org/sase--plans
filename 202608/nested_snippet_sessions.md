---
tier: epic
title: Nested snippet sessions in the prompt input widget
goal: 'Expanding a snippet while another snippet''s tabstops are still pending suspends
  the outer snippet instead of destroying it: the nested snippet''s tabstops are visited
  first, and once they are exhausted `Tab` resumes the enclosing snippet at the stop
  after the one that was nested into. Tabstop anchors survive arbitrary editing because
  they are remapped from real document deltas, `Shift+Tab` steps backwards through
  the visited stops, and the whole session state machine lives in the Rust core so
  any future frontend gets the same behavior.

  '
phases:
- id: core_expansion
  title: Rust snippet expansion planner
  depends_on: []
  size: medium
  description: 'core_expansion: add a pure Rust expansion planner that turns a template
    plus its insertion context into cleaned text and ordered tabstop offsets, and
    make it the single owner of unescaped-tabstop scanning.'
- id: core_session
  title: Rust nested snippet session state machine
  depends_on:
  - core_expansion
  size: medium
  description: 'core_session: add the pure session state machine over a flat ordered
    stop list — nest-vs-reset on expand, advance/retreat, and absolute-anchor remapping
    from document edit deltas.'
- id: core_binding
  title: PyO3 binding and wire parity for the session engine
  depends_on:
  - core_session
  size: small
  description: 'core_binding: expose the session state machine to Python as a single
    wire-shaped binding, with binding-level tests and the lib.rs binding inventory
    updated.'
- id: py_facade
  title: Python facade for the snippet session engine
  depends_on:
  - core_binding
  size: small
  description: 'py_facade: add a validated Python facade over the new binding, register
    the binding name with the core validator, and prove the round trip against a locally
    built core.'
- id: widget_engine
  title: Rewrite the prompt widget snippet mixin over the session engine
  depends_on:
  - py_facade
  size: medium
  description: 'widget_engine: replace the from-doc-end tabstop queue with the facade-backed
    session, feed every document mutation through a TextArea.edit hook, and swap the
    raw _snippet_tabstops reads for a session-active predicate.'
- id: call_sites
  title: Nest-vs-reset policy for every non-trigger expansion caller
  depends_on:
  - widget_engine
  size: small
  description: 'call_sites: make each of the five non-trigger callers of the expansion
    entry point declare whether it nests inside the active session or replaces it,
    and cover the whole-pane skeleton reset.'
- id: back_nav
  title: Shift+Tab backward tabstop navigation
  depends_on:
  - widget_engine
  size: small
  description: 'back_nav: turn the consumed Shift+Tab no-op into a retreat through
    already-visited stops, across nesting boundaries, without disturbing the bullet
    and ordered-list dedent path.'
- id: docs_pin
  title: Documentation and core version pin
  depends_on:
  - call_sites
  - back_nav
  size: small
  description: 'docs_pin: document nested sessions and backward navigation in the
    ACE and editor guides, update the keymap tables and CHANGELOG, and raise the sase-core-rs
    floor once the core release lands.'
proposed_by: bbugyi200.athena.zm
create_time: 2026-08-13 12:25:46
status: done
bead_id: sase-kz
---

- **PROMPT:** [prompts/202608/nested_snippet_sessions.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/nested_snippet_sessions.md)
- **BEAD:** [sase-kz](https://github.com/sase-org/sase--beads/blob/main/pages/sase-kz/README.md)

# Plan: Nested snippet sessions in the prompt input widget

## The defect

`SnippetExpansionMixin` keeps exactly one snippet session, in two scalars declared at
`src/sase/ace/tui/widgets/prompt_text_area.py:125`:

```python
self._snippet_tabstops: list[int] = []
self._snippet_end_from_doc_end: int = 0
```

`_expand_snippet_template_at_range` (`src/sase/ace/tui/widgets/_snippets.py:80`)
unconditionally overwrites both at `_snippets.py:147` and `_snippets.py:154`. So
expanding any snippet while another snippet's tabstops are pending discards the pending
ones. Expanding `foo $1 bar $2 baz $3 buz`, tabbing to `$2`, and expanding a second
snippet there leaves `$3` unreachable — the reported bug.

The behavior is not accidental; it is asserted. `TestSnippetPriority` in
`tests/ace/tui/widgets/test_prompt_snippet_expansion.py:224` ends with:

```python
# Old tabstops replaced by new snippet (no remaining stops)
assert not ta._snippet_tabstops
```

That assertion inverts under this plan.

## Why the current representation cannot be patched in place

Tabstops are stored as _characters from the end of the expansion_, and the expansion end
is stored as _characters from the end of the document_ (`_snippets.py:147` and
`_snippets.py:154`). `_try_advance_tabstop` (`_snippets.py:173`) resolves a stop as
`len(self.text) - _snippet_end_from_doc_end - from_end`.

This is a deliberate trick: it makes stops self-correcting under insertions that happen
_before_ them, which is the only thing a strictly forward, single-session traversal ever
sees. It has two consequences that matter here.

1. **It is forward-only.** Type at `$1`, tab to `$2`, type there, then try to go back to
   `$1`: `$1`'s stored distance-from-doc-end was computed before the `$2` insertion, so
   resolving it now lands to the right of where it belongs, by exactly the number of
   characters typed at `$2`. Backward navigation is not implementable on this
   representation, which is why `Shift+Tab` is a consumed no-op today
   (`_prompt_text_area_key_handling.py:425`).
2. **It breaks under edits after the expansion.** Any insertion past the expansion end
   shifts every resolved stop right by the same amount, because the anchor is measured
   from the document end.

Both fall out of anchoring from the wrong side. The fix is to store absolute offsets and
remap them from real edit deltas, which is possible because Textual funnels every
document mutation through one method (see "The edit funnel" below).

## Scope note on `Shift+Tab`

The prompt describes stepping through tabstops with `Tab` _and_ `Shift+Tab`. Today only
`Tab` navigates; during an active session `Shift+Tab` is consumed and ignored
(`src/sase/ace/tui/widgets/_prompt_text_area_key_handling.py:425`). Backward navigation
is therefore new work, not a regression, and it is included here because the session
model that fixes nesting is also the thing that makes backward navigation expressible.
It is isolated in the `back_nav` phase so it can be dropped without touching the nesting
fix.

## Where this belongs

`CLAUDE.md`'s Rust core backend boundary applies: template semantics and snippet session
traversal are behavior a web prompt input or another frontend would have to match, so
they are core backend logic, not presentation. The core already owns half of it —
`iter_unescaped_tabstops` at `../sase-core/crates/sase_core/src/xprompt_catalog.rs:883`
is a character-for-character twin of `_iter_template_tabstops` at
`src/sase/ace/tui/widgets/_snippets.py:200`, and `renumber_snippet_segments`
(`xprompt_catalog.rs:856`) already renumbers tabstops when `#[trigger]` references
splice one snippet into another at catalog-composition time. That static composition is
the build-time counterpart of the interactive nesting this plan adds; the two must not
diverge, which is another reason to put the runtime engine beside it.

What stays in Python: applying text through `_replace_via_keyboard`, setting
`cursor_location`, key dispatch, and the completion/hint surfaces that gate on "is a
snippet session active".

Dev installs build `sase_core_rs` from the linked checkout (`Justfile:44-106`), so the
Python phases can consume the unreleased core immediately; only the published floor in
`pyproject.toml:46` waits for the release, and that is the last phase.

## The edit funnel

Every document mutation in Textual 8.0.1 reaches `TextArea.edit()`
(`.venv/lib/python3.14/site-packages/textual/widgets/_text_area.py:1538`):

- `insert()`, `delete()`, and `replace()` all construct an `Edit` and call
  `self.edit(edit)`.
- `_replace_via_keyboard()` (`_text_area.py:2430`) — the method the whole vim tower and
  every snippet/completion path in this repo routes through — calls `self.replace(...)`.

`Edit` carries `text`, `from_location`, and `to_location`. That is exactly the delta the
anchor remap needs, and `edit()` is the single place to intercept it.

`load_text()` (`_text_area.py:1071`) deliberately does _not_ go through `edit()`; it
replaces the document wholesale. Clearing the session there is the correct semantic, and
it has a direct consequence for tests — see "Test-simulation hazard".

## Design

### Session state (owned by the Rust core)

Sessions nest positionally: a nested expansion always lands inside the enclosing
expansion, at or near the stop the user is sitting on. So the traversal order is fully
determined — outer stops already visited, then every stop of the inner snippet, then the
outer stops that follow. Rather than a stack that has to be popped and re-entered, model
the state as **one flat ordered stop list with a single cursor**, plus the session spans
needed to decide nesting:

```json
{
  "schema_version": 1,
  "stops": [
    { "offset": 42, "session": 0 },
    { "offset": 47, "session": 1 }
  ],
  "index": 0,
  "sessions": [{ "id": 0, "start": 10, "end": 60 }],
  "next_session_id": 1
}
```

- `stops` — flat, in document order, each tagged with the session that produced it.
  Offsets are absolute character offsets into the document.
- `index` — the stop the cursor currently occupies. Not a consuming queue.
- `sessions` — one span per live expansion, innermost last. Spans exist only for the
  containment test and for cancelling a nested session.

An empty `stops` list means no active session.

### Expand: nest or reset

Given a new expansion over document range `[rs, re)` and a planned stop list:

- If a session is active and `rs >= innermost.start and re <= innermost.end`, **nest**:
  splice the new stops into `stops` immediately after `index`, register the new session
  span, and set `index` to the first spliced stop.
- Otherwise **reset**: drop all sessions and stops, then install the new ones.

The containment test is what makes `_replace_active_pane_with_skeleton`
(`src/sase/ace/tui/widgets/_prompt_input_bar_local_xprompt_actions.py:120`) behave — it
replaces the entire pane body `(0,0)`→end, which cannot be contained in an existing
expansion, so it resets. It also means a trigger typed inside the outer expansion but
away from a stop still nests, which is the desired forgiving behavior.

Cap nesting depth (8 is ample). On overflow, drop the outermost session and its stops
rather than refusing to expand; a runaway stack is worse than a lost outer snippet.

### Advance and retreat

- `advance`: if `index + 1 < len(stops)`, increment and report the new offset. Otherwise
  the session is finished — clear all state and report "no target" so `Tab` falls back
  to its normal behavior. This preserves today's feel exactly: tabbing off the last stop
  ends the session.
- `retreat`: if `index > 0`, decrement and report the new offset. At `index == 0` report
  "no target"; the key handler still consumes the press, as it does today.

Crossing a nesting boundary needs no special case in either direction, because the flat
list already interleaves the sessions in traversal order. Advancing off an inner
snippet's last stop lands on the enclosing snippet's next stop; that is the fix.

### Anchor remapping

On every document edit `(edit_start, edit_end, inserted_len)`, remap every stored
offset:

- **Stops are sticky-right.** `o < edit_start` → unchanged. `o >= edit_end` →
  `o + inserted_len - (edit_end - edit_start)`. `edit_start <= o < edit_end` (the stop
  sat inside deleted text) → clamp to `edit_start + inserted_len`.

  Sticky-right is what makes typing _at_ a stop grow past it, so a later `Shift+Tab`
  back to that stop lands at the end of what was typed rather than in front of it. It
  also reproduces the current forward behavior exactly.

- **Session `start` is sticky-left** (`o <= edit_start` → unchanged), **session `end` is
  sticky-right**, so an expansion keeps containing everything typed inside it.
- If a session's span collapses (`end <= start`) or it has no surviving stops, drop that
  session; if that empties `stops`, the session state is cleared.

### Python glue

`PromptTextArea.edit()` override: compute the edit's absolute start/end against the
_pre-edit_ document, call `super().edit(edit)`, then feed `(start, end, len(edit.text))`
to the session. `Edit.from_location` / `to_location` are not guaranteed ordered — sort
them.

> **Deviation (recorded by `widget_engine`, kept as shipped).** This section originally
> called for a re-entrancy flag around the expansion's own `_replace_via_keyboard` call,
> on the theory that the expansion installs its stops from post-edit offsets and must
> not then be told about its own delta. That is wrong, and the guard was empirically
> shown to break the very bug it was meant to protect: nesting at a live outer tabstop
> and then advancing landed mid-word instead of before the next literal text. The
> expansion's substitution of a short trigger word for a longer template is an ordinary
> delta, and the _enclosing_ session's stops after the trigger have to shift by it like
> any other edit. The shipped `edit()` override therefore has no guard and feeds every
> edit, including the expansion's own, to the active session.

Replace the two scalars with one session object plus a `snippet_session_active`
predicate, and update every site that currently reads the queue's truthiness as "a
session is live":

| Site                                    | Today                                                          | After                                 |
| --------------------------------------- | -------------------------------------------------------------- | ------------------------------------- |
| `_prompt_text_area_key_handling.py:405` | `if not self._snippet_tabstops:` gates bullet/ordered shifting | `if not self.snippet_session_active:` |
| `_prompt_soft_completion.py:170`        | suppresses soft completion during a session                    | same predicate                        |
| `_xprompt_arg_hints.py:97`              | suppresses typed arg-hint detection during a session           | same predicate                        |

and every site that clears it: `_prompt_text_area_actions.py:57`, `:69`, `:87`, `:241`
(submit, submit-stack, history, and the NORMAL-mode transition), plus
`_prompt_format.py:136-137` (whole-buffer reformat).

## Test-simulation hazard

Several existing tests simulate typing with `ta.load_text(...)` —
`test_prompt_snippet_expansion.py:363` (`test_advance_after_typing`) and
`test_prompt_snippet_expansion.py:242` are the clearest cases. `load_text` bypasses
`edit()` and clears the document history, so under the new model it must _clear_ the
session, and those tests would assert against a session that no longer exists. They pass
today only because the from-doc-end trick needs no delta notification.

Every test that simulates editing inside a live session must type through the pilot or
through `insert()` / `replace()` so the edit funnel sees the delta. This is a real
behavior improvement, not just a test rewrite: `load_text` is not something the user can
do mid-session.

Tests that prime session state directly also need a real constructor instead of poking
the list: `tests/ace/tui/widgets/test_prompt_bullet_insert_editing.py:325`,
`tests/ace/tui/widgets/test_prompt_ordered_shift_editing.py:207`, and
`tests/ace/tui/widgets/test_xprompt_arg_hints.py:208`.

---

## Phase `core_expansion` — Rust snippet expansion planner

Repo: `sase-core` (open it with `/sase_repo`; never edit it through a hand-rolled path).

Add a `snippet_session` module under `crates/sase_core/src/` owning a pure planner:

- Input: the template, the leading indentation of the line the expansion starts on, and
  whether indentation should be applied to continuation lines.
- Output: the cleaned expansion text and the ordered tabstop offsets _relative to the
  expansion start_, ordered `$1, $2, …` then `$0` last.

Port the semantics from `src/sase/ace/tui/widgets/_snippets.py:90-147` exactly:

- Only the **first** occurrence of a repeated tabstop number produces a stop (`seen` at
  `_snippets.py:96`); later duplicates are removed from the text and contribute no stop.
- `\$` is a literal dollar and is unescaped in the output (`_unescape_literal_dollars`,
  `_snippets.py:219`); an escaped `$1` is not a tabstop.
- `$` not followed by a digit is literal.
- Continuation lines are indented to match the start line's indentation, and stop
  offsets shift by `newlines_before_offset * len(indent)` (`_snippets.py:112-121`).
- If the template contains no tabstop markers at all, the plan has **no stops** and no
  session is created. If it has markers but no `$0`, append an implicit `$0` at the end
  of the expansion (`_snippets.py:132`).

Make this module the single owner of unescaped-tabstop scanning: move
`iter_unescaped_tabstops` (`crates/sase_core/src/xprompt_catalog.rs:883`) into it and
have `xprompt_catalog.rs` call through, so catalog-time renumbering and runtime
expansion cannot drift.

Unit tests: escaped dollars, repeated numbers, missing `$0`, no markers, multi-digit
numbers, multi-line templates with and without indentation, and offsets that land on
continuation lines.

Do not touch `crates/sase_core_py` in this phase.

## Phase `core_session` — Rust nested snippet session state machine

Repo: `sase-core`.

In the same module, add the state type from "Session state" above and the pure
transitions:

- `expand(state, range, plan) -> state` with the containment nest-vs-reset rule and the
  depth cap.
- `advance(state) -> (state, Option<offset>)` and
  `retreat(state) -> (state, Option<offset>)`.
- `apply_edit(state, edit_start, edit_end, inserted_len) -> state` with the
  sticky-left/right remap rules.
- `clear(state) -> state`.

All transitions are total: no panics on out-of-range offsets, empty stop lists, or edits
that delete an entire expansion. Define a versioned wire representation
(`schema_version`) beside the crate's existing wire conventions and keep serialization
round-trippable.

Unit tests must cover, at minimum:

- The reported bug: expand `foo $1 bar $2 baz $3 buz`, advance to `$2`, nest a second
  snippet there, exhaust the inner stops, and assert the next advance lands on the outer
  `$3`.
- Reset when the new range is not contained (whole-document replacement).
- Two levels of nesting resuming in the right order.
- Insertions and deletions before, at, inside, and after each stop, asserting
  sticky-right stops and a session span that keeps containing text typed within it.
- A deletion that removes a whole nested expansion, dropping that session without
  corrupting the enclosing one.
- Depth-cap overflow dropping the outermost session.
- `retreat` at index 0 and `advance` off the last stop both reporting no target, with
  `advance` additionally clearing the state.

## Phase `core_binding` — PyO3 binding and wire parity for the session engine

Repo: `sase-core`.

Expose one binding from `crates/sase_core_py/src/lib.rs`, shaped like
`py_compose_snippet_catalog` (`lib.rs:1281`): a single entry point that takes the
current state dict plus an event dict and returns the new state dict plus any resolved
cursor offset. One binding rather than five keeps the wire surface and the
schema-version story small.

Add the binding to the module-level inventory comment (`lib.rs:245` area) and add
binding-level tests. Heed `sase-core`'s `AGENTS.md`: verify with `just check` from the
repo root, never `cargo test -p sase_core` alone, because that skips exactly these
binding tests and has already let stale schema-version fixtures reach master once.

Release versioning is release-plz's; do not hand-edit versions.

## Phase `py_facade` — Python facade for the snippet session engine

Repo: `sase` (this repo).

Add `src/sase/core/snippet_session_facade.py` modeled on
`src/sase/core/snippet_catalog_facade.py`: `require_rust_binding(...)`, then validate
the payload shape and raise `TypeError`/`ValueError` on anything malformed rather than
trusting the wire. Expose frozen dataclasses for the state and the transition result so
the widget layer never handles raw dicts.

Register the new binding name in `REQUIRED_BINDINGS` in
`tools/validate_sase_core_rs:33`.

`just install` builds `sase_core_rs` from the linked `sase-core` checkout
(`Justfile:44-106`), so this phase can and should be verified against the unreleased
core. Do **not** raise the `sase-core-rs` floor in `pyproject.toml:46` here; that is
`docs_pin`, after the release.

Tests: facade round trips and each malformed-payload rejection path.

## Phase `widget_engine` — Rewrite the prompt widget snippet mixin over the session engine

Repo: `sase`.

1. Replace `_snippet_tabstops` / `_snippet_end_from_doc_end`
   (`src/sase/ace/tui/widgets/prompt_text_area.py:125-126`) with a single session object
   and a `snippet_session_active` predicate. Update the `TYPE_CHECKING` stub blocks that
   declare the old attributes: `_snippets.py:23`, `_prompt_text_area_actions.py:32`,
   `_prompt_text_area_key_handling.py:69`, `_prompt_soft_completion.py:43`,
   `_xprompt_arg_hints.py:42`, `_prompt_format.py:87`.
2. Rewrite `_expand_snippet_template_at_range` (`_snippets.py:80`) to call the facade
   planner, apply the planned text through the existing `_replace_via_keyboard` call,
   place the cursor at the first stop, and hand the expansion range to the session
   `expand` transition. Keep the existing `_try_auto_placeholder_completion` calls at
   `_snippets.py:77` and `_snippets.py:189` and the current return-value contract
   (`bool`) — five callers depend on it.
3. Rewrite `_try_advance_tabstop` (`_snippets.py:173`) over the `advance` transition,
   converting the returned absolute offset with the existing `_location_from_absolute`
   helper instead of the hand-rolled row scan at `_snippets.py:184-195`.
4. Add the `PromptTextArea.edit()` override that feeds deltas to the session, with the
   re-entrancy guard described in "Python glue". Clear the session on `load_text`.
5. Delete `_iter_template_tabstops`, `_unescape_literal_dollars`, and `_is_escaped`
   (`_snippets.py:200-229`) — the core now owns that parsing.
6. Swap the three gate reads and the five clear sites listed in the design table.
7. Update the tests listed under "Test-simulation hazard", including inverting the
   trailing assertion of `TestSnippetPriority`
   (`test_prompt_snippet_expansion.py:253-254`) to assert the outer stop survives, and
   re-typing the `load_text` simulations as real edits.

Single-snippet behavior must be byte-identical; every currently passing assertion in
`tests/ace/tui/widgets/test_prompt_snippet_expansion.py` that is not about session
destruction stays as written.

## Phase `call_sites` — Nest-vs-reset policy for every non-trigger expansion caller

Repo: `sase`.

`_expand_snippet_template_at_range` has five callers besides the trigger path. Each must
be verified against the containment rule and, where the rule alone is not enough, given
an explicit reset:

| Caller                                                                    | Range                                 | Expected                              |
| ------------------------------------------------------------------------- | ------------------------------------- | ------------------------------------- |
| `src/sase/ace/tui/widgets/_file_completion_accept.py:151`                 | accepted `#xprompt` token on one line | nests when inside an active expansion |
| `src/sase/ace/tui/widgets/_prompt_soft_completion.py:290`                 | soft-completion replacement span      | nests when inside                     |
| `src/sase/ace/tui/widgets/_prompt_input_bar_actions.py:668`               | `Ctrl+T` xprompt skeleton span        | nests when inside                     |
| `src/sase/ace/tui/widgets/_xprompt_arg_hints.py:308`                      | named-argument skeleton               | nests when inside                     |
| `src/sase/ace/tui/widgets/_prompt_input_bar_local_xprompt_actions.py:140` | `(0,0)` → end of document             | always resets                         |

The last one replaces the whole pane body, so containment can never hold and the rule
resets it on its own; add a test that pins that, because a future change to the
containment test could silently start nesting a whole-document replacement inside
itself.

Note the mode interaction already documented at
`_prompt_input_bar_local_xprompt_actions.py:120-130`: that path enters INSERT before
expanding because a skeleton carrying tabstops must not drop to NORMAL, which clears the
session (`_prompt_text_area_actions.py:241`). That reasoning still holds and must keep
holding.

Tests: one per caller asserting nest or reset, plus a nested `#xprompt` accepted at an
outer snippet's tabstop resuming the outer snippet afterwards.

## Phase `back_nav` — Shift+Tab backward tabstop navigation

Repo: `sase`.

In `src/sase/ace/tui/widgets/_prompt_text_area_key_handling.py:401-430`, replace the
`if event.key == "shift+tab": return` no-op at line 425 with a `retreat` call, keeping
the existing ordering guarantees:

- The bullet / ordered-list shift at lines 405-424 still runs first _only_ when no
  session is active, so `Shift+Tab` keeps dedenting list items outside a snippet.
- `Tab` still tries expansion before advancing (lines 427-429) — the documented priority
  at `docs/ace.md:5019`.
- `Shift+Tab` at the first stop stays a consumed no-op.

Retreat must cross nesting boundaries: from an inner snippet's first stop it lands on
the enclosing snippet's stop that was nested into, and a following `Tab` returns forward
into the inner stops. Because `advance` off the last stop clears the session, backward
navigation is not available after the session ends — that is intentional and must be
asserted, not left implicit.

Tests: retreat within one snippet after typing at an earlier stop (asserting the cursor
lands at the _end_ of the typed text, per sticky-right anchoring), retreat across a
nesting boundary, retreat at index 0, and `Shift+Tab` still dedenting a list item when
no session is active.

## Phase `docs_pin` — Documentation and core version pin

Repo: `sase`.

1. `docs/ace.md` §Snippets (around line 4995-5036): document that expanding a snippet at
   an active tabstop suspends rather than replaces the enclosing snippet, that `Tab`
   resumes the enclosing snippet once the nested stops are exhausted, that `Shift+Tab`
   steps backwards, and that replacing the whole pane body ends the session. Keep the
   existing "Tab priority" paragraph at line 5019 — it is still true — and extend it
   with the resume rule.
2. Keymap tables: extend the `Tab` row at `docs/ace.md:3917` and add a `Shift+Tab` row
   for backward navigation.
3. `docs/editor.md:243-248` describes `#[trigger]` composition, the build-time
   counterpart of this feature. Add a sentence distinguishing it from interactive
   nesting so the two are not confused, and check the cross-reference at
   `docs/editor.md:268`.
4. `CHANGELOG.md` entry.
5. Once the `sase-core` release carrying the session engine lands, raise the
   `sase-core-rs` floor in `pyproject.toml:46` to that version. Do not invent a version
   — read the released one.

## Verification

Rust phases: `just check` from the `sase-core` repo root.

Python phases: `just install` first (workspaces are ephemeral and dependencies drift),
then `just check`. This change touches the prompt widget's key-handling and completion
surfaces, so run `just check-full` through `/sase_monitor` before the epic's tree lands
— never inline.

`just test-visual` is not expected to move: nothing here changes rendering.
