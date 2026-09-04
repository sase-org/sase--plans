---
tier: tale
size: medium
title: The generic host-owned reveal ladder
goal: "Replace the single ad-hoc limit-drop fallback in the ``$`` link-follow path with
  an ordered, host-owned reveal ladder (fold expansion, limit drop, minimal widening via
  the pane's own query matcher, neutral ``limit:all``, then an honest taxonomy-aware
  toast); wrap every query rewrite in a self-retiring ``LinkReveal`` lens committed once
  through each pane's existing query-history seam, so exactly one ``^`` restores the
  user's own query; and delete the four pane-local filter-clear mutations that bypass
  that seam today.

  "
proposed_by: bbugyi200.apollo.sase-w3.4
bead: sase-w3.4
create_time: 2026-09-04 09:27:36
status: wip
---

- **PARENT:** [202609/link_follow_reliability.md](link_follow_reliability.md)
- **BEAD:**
  [sase-w3.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-w3/sase-w3.4.md)

# Phase sase-w3.4 — The Generic Host-Owned Reveal Ladder

## What This Is

This is the implementation plan for **phase bead `sase-w3.4`** (`reveal-ladder`) of epic
`sase-w3` (_Artifact link-follow reliability_). Read the epic plan first:

```bash
cat "$SASE_SDD_PLANS_DIR/202609/link_follow_reliability.md"   # or:
sase bead show sase-w3.4          # prints the resolved epic plan path
```

The epic's **Decisions**, **Cross-repo ground rules**, and **Verification ground rules**
sections are binding and must not be relitigated. In particular: **no feature flag** —
this phase lands as an unconditional improvement, and the built-in escape hatch for any
query rewrite is `^` (previous query).

Close `sase-w3.4` when done. Do **not** close epic `sase-w3` or any ancestor. Record any
discovered follow-up work with
`sase bead note sase-w3.4 'PROPOSED FOLLOW-UP: <one-line summary — detail>'`; do not
create beads.

## Ground Already Covered (do not redo)

Phases `sase-w3.1`-`sase-w3.3` have landed. The tree you start from already has:

- `ArtifactEntryNavigator.entry_target_for_ref(kind, payload)`
  (`src/sase/ace/tui/widgets/artifacts/entry_navigation.py`) — resolves a link ref to
  the destination pane's **own** row identity from its unfiltered snapshot. The default
  implementation delegates to the sase-core matching facade via `known_target_for_ref`.
- `LinkFollowMixin._resolve_link_follow_target`
  (`src/sase/ace/tui/actions/link_follow.py`) — addresses follows by canonical ref, with
  `chip.neighbor_target` demoted to a routing/fast-path hint.
- The tri-state `LinkRequestState` (`SELECTED` / `PENDING` / `MISSING` / `FAILED`), the
  host-owned `_LinkFollowTransaction` with generation tags, the shared
  `ArtifactEntryNavigator._complete_entry_request` completion seam, and
  `_handle_link_follow_outcome` / `_handle_missing_link_follow` /
  `_finalize_selected_link_follow` / `_notify_link_follow_failed`.
- A uniform `host_limit_query()` / `apply_host_limit_query(query, *, grow=False)`
  adapter on **every** pane — beads, plans, files, agents, stitches
  (`*_filter_session.py` / `agents_query.py` / `commits_filtering.py`) **and** patches
  (`src/sase/ace/tui/widgets/artifacts/panes.py`). Each implementation commits through
  its pane's `_commit_*` funnel, which records a query-history transition through
  `app._record_artifacts_query_transition`. **That adapter is the query-history seam
  this phase must use for every rewrite.**
- Link-follow no longer calls `reveal_entry_target` at all (the FAMILY rung is gone);
  `reveal_entry_target` survives only for relation navigation
  (`src/sase/ace/tui/actions/navigation/_tree.py`). Leave that path alone.

Today's _only_ reveal rung is `_drop_head_slice_limit` (`link_follow.py`), guarded by
`_LinkFollowTransaction.limit_drop_attempted`. Everything else in this plan is new.

## Design

### The ladder

`LinkFollowMixin` gains an ordered ladder attempted on an authoritative `MISSING`, each
rung tried at most once per follow, in this order:

| #   | Rung                                                 | Query change?                       |
| --- | ---------------------------------------------------- | ----------------------------------- |
| 1   | Select in place                                      | no (already implemented, unchanged) |
| 2   | Project-scope switch                                 | no (already implemented, unchanged) |
| 3   | **Expand the minimum fold hiding the row**           | no                                  |
| 4   | **Drop the `limit:` head slice**                     | yes                                 |
| 5   | _(reserved: identity reveal — phase `sase-w3.5`)_    | —                                   |
| 6   | **Minimal widening** (drop only the excluding terms) | yes                                 |
| 7   | **Neutral query** (`limit:all`)                      | yes                                 |
| 8   | Honest, taxonomy-aware toast                         | —                                   |

**Deliberate ordering note.** The epic plan's numbered list puts the `limit:` drop
before fold expansion, but its own test invariant for this phase reads "fold expansion
preferred over any query change". Fold expansion is strictly cheaper — it mutates no
query and pushes no history entry — and running it first never prevents a later `limit:`
drop, so this plan resolves the two statements in favour of the test invariant: **fold
expansion first**. State this in the module docstring so the next reader does not "fix"
it back.

Rungs run across the async boundary: a rung fires, re-requests the target, and may get
`PENDING`; the later authoritative `MISSING` advances to the next rung. The rung cursor
lives on the (frozen) `_LinkFollowTransaction` and only ever increases, so the ladder
terminates.

### Exactly one history entry per follow

The epic requires "exactly one previous-query record per reveal", "one `^` restores the
exact prior source and selection", and "no intermediate `limit:all` or empty query
pollutes history". A follow may fire two or three rewriting rungs, each of which commits
through the seam and would push its own record.

Solution: the host **pins** the query-transition recorder for the life of one follow.
While pinned for a pane, the _first_ transition records normally (pushing the user's own
pre-follow query) and every later transition for that pane is skipped. `^` from the
final revealed query therefore lands on the user's query, and `_` returns to the reveal.

A second follow arriving while a `LinkReveal` is still live on the same pane must not
stack a second record either: the revealed query was never the user's choice. Detect
that with `is_link_reveal_active`, reuse the live lens's origin `QueryRecord`, and
suppress the new push entirely.

### The `LinkReveal` lens

A new module `src/sase/ace/link_reveal.py`, modelled directly on the existing
`src/sase/ace/relation_reveal.py` (read it first — same self-retiring semantics, same
`QueryRecord` origin stamping, same "no clear step" rule):

```python
@dataclass(frozen=True, slots=True)
class LinkReveal:
    pane_id: str
    ref: str                       # the followed link ref, e.g. "bead:sase-123"
    origin: QueryRecord            # the user's own query, before any rung fired
    origin_target: ArtifactEntryTarget | None   # selection to restore with ^
    revealed_canonical: str        # the canonical query the ladder committed


def make_link_reveal(*, pane_id, ref, origin_source, origin_canonical,
                     origin_target, revealed_canonical) -> LinkReveal: ...


def is_link_reveal_active(reveal, *, pane_id, current_canonical) -> bool: ...
```

`is_link_reveal_active` returns `False` once the pane's live canonical query has moved
away from `revealed_canonical` (a user edit, `^`/`_`, a saved-query load, or a fresh
reveal) or once `origin.is_stale(pane_id)` reports a dialect change — no explicit clear
step, exactly like `RelationReveal`.

Store live lenses in `self._link_reveals: dict[str, LinkReveal]` keyed by pane id,
initialised next to `_relation_reveals` in
`src/sase/ace/tui/actions/_state_init_navigation.py` and declared on the same typed
state protocols that declare `_relation_reveals`
(`src/sase/ace/tui/actions/navigation/_types.py`,
`src/sase/ace/tui/actions/patch/_display.py` — grep `_relation_reveals` for the complete
set). Phase `sase-w3.6` renders the chip; this phase only records and queries the lens.

### Answering "would this query show the row?"

Minimal widening needs to test candidate queries against the target's **unfiltered** row
without committing anything. Matching is Rust-owned
(`src/sase/core/query_profile_corpus_facade.py` is explicit that the Python reference
evaluator is parity-test-only, never a production matcher), so build a **one-row**
`ArtifactQueryIndex` for the target and evaluate candidates against it.

Two additions to the `ArtifactEntryNavigator` contract
(`src/sase/ace/tui/widgets/artifacts/entry_navigation.py`):

```python
def host_query_row_for_target(self, target) -> ArtifactQueryRowInput | None:
    """Return the profile-query row entry backing *target* in this pane's
    unfiltered snapshot, or ``None`` when the pane cannot answer."""
    return None


def host_query_probe(self, target) -> HostQueryProbe | None:
    """Build a one-row matcher for *target*, or ``None`` when unavailable."""
```

`host_query_probe`'s default implementation is generic: take
`host_query_row_for_target(target)` plus the pane's compiled profile
(`getattr(self, "_query_profile", None)`, the same lookup
`ArtifactsQueryHistoryActionsMixin._query_profile_digest` already uses) and return

```python
@dataclass(frozen=True, slots=True)
class HostQueryProbe:
    index: ArtifactQueryIndex

    def matches(self, query: str) -> bool:
        """Whether *query* would include this probe's single row."""
```

built with
`compile_artifact_query_index(pane_id=profile.pane_id, generation=0, profile=profile, entries=(row,))`
and answered with `evaluate_artifact_query_many(query, index).matched_row_ids`. A
one-row corpus makes both calls trivial, so this stays safe on the keystroke path; wrap
them so a `ProfileQueryError` / any binding failure degrades to "no answer" rather than
raising into an action handler. Put `HostQueryProbe` and its builder in `link_reveal.py`
(the navigator imports it lazily inside the method to keep module import cost off pane
import).

Only panes override `host_query_row_for_target`. Each override is a short snapshot
lookup that reuses the row-entry builders already in
`src/sase/ace/tui/widgets/artifacts/query_rows.py` — promote the private
`_bead_query_entry` / `_plan_query_entry` / `_files_query_entry` (check its real name) /
`_agent_query_entry` / `_commit_query_entry` to public names and update their in-module
callers:

| Pane              | target parts                | lookup                                                                                                                         |
| ----------------- | --------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| beads             | `(project, kind, bead_id)`  | `row_option_id(snapshot, kind, project, bead_id)` into `self._filter_index.by_option_id`                                       |
| plans / documents | `(project, kind, identity)` | the plans filter index's `by_option_id` (same `_row_option_id` shape)                                                          |
| files             | `(logical_id,)`             | the loaded `FilesSnapshot` rows by `entry.logical_id`                                                                          |
| agents            | `(name,)`                   | `AgentsSnapshot.rows` by `row.name`                                                                                            |
| stitches          | `(repo, full_sha)`          | the collected `VcsLogResult` commits by `(repo, commit.full_id)`                                                               |
| patches           | `(project, name)`           | the `Patch` object out of `app._all_patches` (`coerce_artifact_query_row` already special-cases patch rows via `is_patch_row`) |

A pane that has no snapshot loaded returns `None`; the ladder then simply skips the
widening rung.

### Minimal widening

Host-side, in `link_reveal.py`, pure and unit-testable:

```python
def minimal_widening_query(query: str, probe: HostQueryProbe) -> str | None:
```

1. `remainder, cap = extract_limit(query)` (`sase.ace.query.limit_token`); a
   `LimitTokenError` → `None`.
2. If `probe.matches(remainder)` → `None`. The filter is _not_ what is hiding the row
   (it is the head slice or a fold), so widening has nothing to do.
3. `tokens = tokenize(remainder, error_type=...)` from `sase.filter_tokens`
   (quote-aware; `FilterToken.raw` preserves the original source slice). A parse failure
   → `None`.
4. Bail out to `None` if any token looks like boolean-grammar syntax (`AND`, `OR`,
   `NOT`, or a token containing `(`/`)`): the patches and agents dialects are
   `boolean=True` and token subtraction is not sound there. Those panes fall through to
   the neutral rung, which is correct and honest.
5. Keep the tokens the row satisfies (`probe.matches(token.raw)`), drop the rest. If
   nothing was dropped → `None`.
6. Re-join the kept `raw` slices, verify the result with `probe.matches(...)`, and
   return it with `limit:all` re-appended (reuse the existing `_limit_all_query`
   helper). Verification failing → `None`.

This is exactly "drop only the excluding terms": `since:24h` for an old stitch,
`-status:closed` for a closed bead, while everything else the user typed survives.

### Fold expansion

Add to the contract:

```python
def expand_fold_for_entry_target(self, target) -> bool:
    """Expand the minimum fold hiding *target*; no query change."""
    return False
```

Implement it **once** on `ArtifactGroupFoldMixin`
(`src/sase/ace/tui/widgets/artifacts/group_fold_navigation.py`) using the same shape as
`_focused_group_key` / Files' existing `_expand_group_for_pending_target`: walk
`self._group_build_result(fold_registry=registry).rows`, and expand the collapsed
banners whose `banner.member_targets` contain _target_. That covers Files, Plans /
Documents, Stitches, and Agents in one place. Beads overrides it with an epic-fold
version built on the existing `_expand_parent_for_pending_target` (`beads_options.py`),
refactored so the target is a parameter instead of `self._pending_entry_target`.

Leave the existing pane-local calls to those helpers in place — they are idempotent
(expanding an already-expanded fold returns `False`) and they still serve
non-link-follow callers.

### Closing an open inline filter session

Add:

```python
def close_host_filter_session(self) -> None:
    """Close any open inline filter editor before a host query rewrite."""
```

Default no-op; implement on the five filter-session panes by delegating to each pane's
existing `_close_filter_session` / `_close_agent_filter_session` when its
`_filter_session_open` flag is set. The ladder calls it once, before its first rewriting
rung.

### The honest toast

Replace the single `_notify_missing_link_target` message with three distinct outcomes:

- **Dangling** (`parse_link_ref` fails, or the chip carries no target and
  `target_for_ref_kind` cannot route the ref): `No such artifact: bead:sase-999`. This
  is the `_follow_single_link_chip` early-return path that today wrongly reports "not
  visible in the destination pane".
- **Not in inventory** (ladder exhausted, pane authoritative `MISSING`):
  `Bead has no bead:sase-999 in its inventory` — use `_pane_label(target)` for the pane
  name.
- **Load failure**: the existing `_notify_link_follow_failed` copy, unchanged.

When a rewriting rung fires, notify with the restore hint instead of today's "Expanded …
limit to show linked …":

```
Revealed bead:sase-123 — press ^ to restore your query
```

One notice per follow, emitted when the follow finalizes as `SELECTED` with a live lens
— not once per rung.

### Deleting the four seam-bypassing mutations

`_clear_filter_for_entry_jump` and its `_notify_filter_cleared_for_entry_jump` sibling
exist on four panes and mutate pane query state directly
(`self.filters = BeadFilterValues()`, `self.query_source = ""`, …) from inside
`_refresh_options`, bypassing the history seam entirely:

- `src/sase/ace/tui/widgets/artifacts/beads_options.py`
- `src/sase/ace/tui/widgets/artifacts/agents_options.py`
- `src/sase/ace/tui/widgets/artifacts/plans_options.py`
- `src/sase/ace/tui/widgets/artifacts/files_options.py`

**Delete all eight methods and their call sites.** The host ladder's widening and
neutral rungs replace them, and removing them is a strict improvement for the other
callers of `request_entry_target`: `Ctrl+O` trail restore
(`src/sase/ace/tui/actions/link_trail.py`) restores the hop's query explicitly and must
not have it cleared underneath; `_restore_artifacts_query_selection`
(`src/sase/ace/tui/actions/artifacts_query_history.py`) has just applied the query it is
restoring a selection for; and relation navigation has its own `RelationReveal` path.
After deletion, the `elif` chains fall straight through to the existing authoritative
`MISSING` / `FAILED` reporting.

## Implementation Steps

1. **Read first.** `src/sase/ace/relation_reveal.py`,
   `src/sase/ace/tui/actions/link_follow.py`,
   `src/sase/ace/tui/actions/artifacts_query_history.py`,
   `src/sase/ace/tui/widgets/artifacts/entry_navigation.py`, and
   `tests/ace/tui/test_link_follow.py` (the duck-typed `_Pane` / `_DeferredPane` /
   `_App` harness is the model for every new ladder test).

2. **`src/sase/ace/link_reveal.py`** — `LinkReveal`, `make_link_reveal`,
   `is_link_reveal_active`, `HostQueryProbe`, `build_host_query_probe`,
   `minimal_widening_query`. Mirror `relation_reveal.py`'s docstring style; explain the
   self-retiring rule and the boolean-dialect bail-out.

3. **Contract additions** in `entry_navigation.py`: `expand_fold_for_entry_target`,
   `close_host_filter_session`, `host_query_row_for_target`, `host_query_probe` — all
   with working defaults, all documented, none abstract (so no pane construction
   breaks).

4. **Pane implementations**: `host_query_row_for_target` on beads / plans / files /
   agents / stitches / patches; `expand_fold_for_entry_target` on
   `ArtifactGroupFoldMixin` plus the beads override; `close_host_filter_session` on the
   five filter-session panes. Promote the needed private row-entry builders in
   `query_rows.py` to public names.

5. **History pinning** in `ArtifactsQueryHistoryActionsMixin`: a
   `_collapsed_query_transitions: str | None` pane-id field plus
   `_begin_collapsed_query_transitions(pane_id)` / `_end_collapsed_query_transitions()`,
   honoured at the top of `_record_artifacts_query_transition` (record the first
   transition for the pinned pane, skip the rest). Initialise the field in
   `_state_init_navigation.py`. Make sure `_end_*` runs on every exit path of a follow,
   including cancellation via `_cancel_link_follow_transaction` (`link_trail.py` calls
   it when the user navigates away) and supersession by a second follow.

6. **The ladder** in `link_follow.py`: replace `limit_drop_attempted: bool` on
   `_LinkFollowTransaction` with a monotonic `rung: int` cursor plus the captured origin
   `QueryRecord` and origin selection; rewrite `_handle_missing_link_follow` to walk the
   rung table; add `_reveal_*` helpers for each rung; record/refresh the `LinkReveal`
   after every successful commit; emit the reveal notice on finalize; implement the
   three-way toast taxonomy. Keep `_pane_is_loading` guarding the whole ladder, as the
   limit rung is guarded today.

7. **Patches through the ladder**: change `apply_host_limit_query` in
   `src/sase/ace/tui/widgets/artifacts/panes.py` to call
   `app._commit_patch_query(query, notify=False)`, so a reveal does not also toast
   "Query updated" (`_commit_patches_limit_query` in `actions/artifacts_limit.py`
   already passes `notify=False`; this makes the two consistent).

8. **Delete** the four `_clear_filter_for_entry_jump` /
   `_notify_filter_cleared_for_entry_jump` pairs and their call sites.

## Tests

Extend `tests/ace/tui/test_link_follow.py` (duck-typed fake panes) plus a new
`tests/ace/test_link_reveal.py` for the pure helpers. Cover, at minimum:

- **Rung order**: a pane that can expand a fold is preferred over any query change; a
  pane whose row is excluded by one term gets a widening rewrite, not `limit:all`;
  `limit:all` fires only after widening returns nothing.
- **Exactly one record**: a follow that fires two rewriting rungs leaves exactly one
  query-history record, holding the user's _pre-follow_ source and canonical; one `^`
  restores it together with the remembered selection; `_` reapplies the revealed query;
  no intermediate `limit:all` or empty query appears anywhere in the stack.
- **No reveal stacks on a reveal**: a second follow while the lens is live pushes no
  additional record and keeps the original origin.
- **Lens lifecycle**: `is_link_reveal_active` is `True` immediately after a reveal,
  `False` after any user query edit, after `^`, and after a profile-digest change.
- **Minimal widening unit tests**: `-status:closed` dropped for a closed bead while
  `project:demo` survives; `since:24h` dropped for an old stitch; a query the row
  already matches returns `None`; a boolean-grammar query returns `None`; quoted values
  containing spaces survive the round trip.
- **Toast taxonomy**: a dangling ref says "No such artifact", an exhausted ladder says
  the pane has no such entry in its inventory, and a `FAILED` outcome keeps its distinct
  load-failure copy.
- **Filter session**: a follow with an open inline filter session closes it cleanly and
  still commits exactly one rewrite.
- **Former direct-mutation paths**: the four panes whose `_clear_filter_for_entry_jump`
  was deleted now leave a `^` record when the host reveals their row.
- Update `test_follow_link_drops_head_slice_limit_before_missing_warning`, which asserts
  the old `"Expanded File limit to show linked file:hidden.txt"` copy, to the new reveal
  notice.

## Verification

Read `sase/memory/lint_and_test.md` and `sase/memory/tui_perf.md` with
`/sase_memory_read` before finishing.

```bash
just install        # ephemeral workspace clones may have drifted deps
just check          # agent default: all lint gates + diff-scoped tests
```

Plus the targeted suites while iterating:

```bash
.venv/bin/pytest tests/ace/tui/test_link_follow.py tests/ace/test_link_reveal.py \
  tests/ace/tui/test_link_trail.py tests/ace/tui/test_artifacts_query_history.py -q
```

Before closing the bead run `sase bead epic-symbols sase-w3.4` and resolve any
`--epic-symbol` leftovers (there are none at plan time; if this phase must leave a
symbol for `sase-w3.5`/`sase-w3.6`, key its Justfile entry to a still-open bead — the
parent epic or that later phase — never to `sase-w3.4`). Then:

```bash
sase bead close sase-w3.4 --note "<what you verified>"
```

Do not run `just check-full` inline; if it is needed, run it through `/sase_monitor`. Do
not close `sase-w3` or any ancestor.

## Constraints And Risks

- **TUI perf** (`sase/memory/tui_perf.md`): the ladder runs on a keystroke path. The
  one-row `ArtifactQueryIndex` build and each `evaluate_artifact_query_many` call are
  bounded and synchronous by design (one row, a handful of candidate queries) — do not
  let the probe grow into a whole-corpus evaluation, and do not add disk I/O,
  subprocesses, or unbounded awaits. Re-capture UI state after any await; the
  generation-tagged transaction from `sase-w3.3` is the guardrail.
- **Rust boundary** (`rust_core_backend_boundary` core memory): matching must stay in
  Rust via the corpus facade. This phase adds no Rust surface and needs no `sase-core`
  change; the lens, the ladder, and the rung ordering are presentation-layer host policy
  and stay in this repo.
- **Symvision** (`sase/memory/symvision.md`): every new public symbol needs a real
  non-test consumer. `LinkReveal` / `make_link_reveal` / `minimal_widening_query` /
  `HostQueryProbe` are consumed by `link_follow.py`; `is_link_reveal_active` is consumed
  by the "no reveal stacks on a reveal" check in `link_follow.py` — keep that consumer,
  or the symbol needs an epic whitelist keyed to a still-open bead.
- **Do not** widen scope into `sase-w3.5` (identity query fields), `sase-w3.6` (lens
  chip, Links-panel pre-flagging, help modal, PNG snapshots), or `sase-w3.7` (targeted
  hydration). Rung 5 is a documented gap the ladder is _shaped_ for, not implemented
  here.
