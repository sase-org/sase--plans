---
tier: tale
size: medium
title: Fix three Record deck landing defects and close sase-1jm.4.2 and sase-1jm.4
goal: "A hydrated archived agent shows its Rust light-row Outcome, Last activity, and
  Time basis on the RECORD Lifecycle card however RECORD is opened. Live parent and
  retry relations resolve to durable @agent: references. The Lifecycle heading is the
  copyable @agent: reference. Then epic sase-1jm.4.2 and its parent phase sase-1jm.4
  close.

  "
proposed_by: bbugyi200.athena.sase-1jm.4.2.land
bead: sase-1jm.4.2
create_time: 2026-10-12 11:25:30
status: wip
---

- **PARENT:**
  [202610/record_deck_refresh.md](https://github.com/sase-org/sase--plans/blob/main/202610/record_deck_refresh.md)
- **BEAD:**
  [sase-1jm.4.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1jm/sase-1jm.4.2.md)

# Fix three Record deck landing defects and close sase-1jm.4.2 and sase-1jm.4

## Context

The `sase-1jm.4.2` epic (plan `plan:202610/record_deck_refresh.md`) added the fifth ≡
RECORD agent data deck. All five of its phases are closed, and their commits are on
master (`b4dbfcd90e`, `e82db580a2`, `a6df5daa01`, `f26ac18475`, `ee673ce10d`). The land
agent verified the work against the epic plan and against the parent phase's design
(`plan:202610/agents_archive_view.md`, section _Record deck for every agent_):

- The registry, spec, picker, palette, help, docs, persistence, and layout work is done.
  New splits never auto-choose RECORD.
- The four cards exist. The off-thread loader, the cache, and stale-result rejection
  work.
- The light-row path for unreadable entries and containers works.
- The bench and idle-tick proof pass.
- No memory note, pane module, flag, or key binding changed.
- The 84 Record and layout unit tests pass.
- Integration: no commit outside the epic since it started touches the deck or Record
  seams. The concurrent commits are New Project, plan-decision cards, and the Archive
  node finder.

Three defects caused by this epic remain. This tale fixes them and then performs the
epic's closeout.

The land agent already triaged every `PROPOSED FOLLOW-UP:` and recorded the outcomes on
`sase-1jm.4.2`:

- It added +1s to `sase-1k4` and `sase-1ic`.
- It recorded `DISCOVERED ISSUE` notes on `sase-1jm` for:
  - `test_archive_help_png_snapshot`, broken by `ebd17bf5bf`
  - the revival-preview copy test, broken by `c4c7846f57`
  - the dead `bundle_path` branch in `resolve_archive_bundle_path`
- It found that the stale Justfile epic-symbol rows were already gone.

Do not re-triage any of these. Do not fix those pre-existing failures here.

Constraints carried over from the epic plan:

- No I/O in compose, render, keystroke, or message-pump paths. In-memory attribute reads
  and list scans are fine.
- Do not add a Python status→outcome table. Archived facts come only from the Archive
  light row.
- Edit no memory notes.
- Keep every module under the 700-line `toobig` gate.
- Do not change key bindings or flags. Treat `widgets/artifacts/agents_detail*.py` as
  read-only.

## Defect 1: switching to RECORD on a hydrated archived agent drops the light row

In `src/sase/ace/tui/widgets/_agent_detail_deck_refresh.py`, the `_deck_refresh_views`
RECORD arm handles an agent where `agent._archive_read_only` is true. It looks up the
light row through `archive_read_only_context(owner)` (owners `self.app`, then `self`)
and the bundle path from `bundle_path` / `_dismissed_bundle_path`. It passes both to
`panel.record_view.update_display(...)`.

`AgentDetailDeckShowMixin.show_deck` in `_agent_detail_deck_show.py` handles every
explicit switch: `p r`, `P R`, Ctrl+N/Ctrl+P, the palette, a split, parking, and
persistence restore. Its RECORD arm calls `update_display` without `archive_light_row`
or `bundle_path`. So the subject has `archive=None`, and the Lifecycle card loses
Outcome, Last activity, and Time basis.

The committed golden `agents_record_archived_lifecycle_120x40.png` shows the bug. It
reaches RECORD through `show_deck` and shows only Location, Status, Started, and Ended.
By contrast, `agents_record_unreadable_entry_120x40.png` shows `Outcome: ✓ done`,
`Last activity`, and `Time basis: ended`.

Fix:

1. Extract the RECORD arm's archive capture into one pump-safe method on
   `AgentDetailDeckRefreshMixin`. Build it to also return Defect 2's live names, for
   example a `_record_display_kwargs(agent) -> dict[str, Any]` that returns the
   `archive_light_row`, `bundle_path`, and `related_live_names` keyword arguments.
2. Use it in both the `_deck_refresh_views` RECORD arm and the `show_deck` RECORD arm,
   so the two paths build identical subjects and share the loader cache signature.
3. Keep the defensive `try/except` style of the surrounding code.

## Defect 2: live parent and retry relations never resolve in production

`RecordSubject.related_live_names` and `live_related_names(agents, suffixes)` in
`widgets/decks/record/model.py` exist so that Relations can resolve parent and retry
suffixes from the in-memory agent list first. `record/relations.py`
`_resolve_related_names` only queries the dismissed-bundle index for suffixes that are
missing from it.

No production caller ever fills it. `RecordDeckView.update_display` (`record/view.py`)
has no parameter for it, and both AgentDetail callers omit it. A parent or retry that is
still in the Inbox is absent from the dismissed index, so it renders as a dim raw suffix
instead of a resolved `@agent:` reference.

Fix:

1. Add `related_live_names: tuple[tuple[str, str], ...] = ()` to
   `RecordDeckView.update_display` and forward it to `record_subject_from_agent`.
2. In the Defect 1 helper, collect the agent's non-empty `parent_timestamp`,
   `retry_of_timestamp`, and `retried_as_timestamp`. When at least one exists, call
   `live_related_names(...)` on the app's in-memory agent list. Use
   `getattr(app, "_agents_with_children", None)`, which includes workflow children,
   falling back to `_agents`.
3. Skip the scan entirely when the agent has no such suffix. A parent's `raw_suffix`
   equals its child's `parent_timestamp`; see `_agent_list_build_remove.py`.
4. Make `live_related_names` return the durable `agent_name` first and fall back to
   `presented_agent_name` only when `agent_name` is empty.
   - Why: the Agents-tab copy action (`reference_for_agent_row` in
     `actions/clipboard/_artifact_reference_resolution.py`) copies
     `reference_for_agent_name(agent.agent_name)`. The epic requires Relations to show
     exactly the text that action delivers.
   - Update `test_live_related_names_resolve_from_memory_only` in
     `tests/ace/tui/widgets/decks/test_record_deck_shell.py` to match.

## Defect 3: the Lifecycle heading is not the copyable `@agent:` reference

`record/facts.py` sets `FactsExtras.reference` from `reference_for_agent_name(name)`,
which returns `agent:<name>` without the `@`. `lifecycle_card` renders it as the
heading. The goldens show `agent:landed` and `agent:archived-run`.

Both the epic plan and the parent phase require "the copyable `@agent:` reference" on
Lifecycle. The Relations card and the copy actions already render `@agent:…`.

Fix: render `@` + reference when a reference resolved, without doubling an existing `@`.
Keep the current fallback (`agent_name`, then "Unnamed agent") when it did not resolve.

## Tests

Follow the existing patterns in `test_record_deck_shell.py`,
`test_record_deck_facts.py`, `test_record_relations.py`, and
`test_record_deck_archive_rows.py`, all under `tests/ace/tui/widgets/decks/`.

- **Defect 1:**
  - Arrange an archive-read-only agent whose read-only context carries a light row with
    an outcome and time basis.
  - Check that `show_deck(index, DeckId.RECORD)` builds a subject whose `archive` is
    populated.
  - Check that this subject's cache signature equals the one built by the refresh path.
- **Defect 2:**
  - The helper passes `(suffix, durable name)` pairs for a parent and a retry that are
    live in the in-memory list.
  - It skips the scan when the agent has no suffixes.
  - The Relations card for that subject shows the resolved `@agent:` references.
- **Defect 3:** the Lifecycle heading starts with `@agent:`. Update the existing facts
  assertions that expect the old heading.
- **Archived golden assertion:** in
  `tests/ace/tui/visual/test_ace_png_snapshots_agents_record.py`, make
  `test_agents_record_archived_lifecycle_png_snapshot` assert that the page SVG shows
  the Outcome and Time basis lines, for example `expect="Time basis"`. Then the golden
  cannot silently lose archive facts again.

## Verification

1. Run the Record and deck unit suites with pytest. Then run `sase tool run check` (hand
   it to `/sase_monitor` if it may outrun the turn).
   - These failures are pre-existing and are KNOWN or tracked elsewhere:
     - the Symvision private `_load_archive_bundle` import (note #4 on `sase-1jm`)
     - `sase-1k4` (Project: tmp)
     - `sase-1ic` (import budget 3517 against a cap of 3513)
     - the revival-preview copy test (note on `sase-1jm`)
   - Treat any other failure as yours.
2. Rebaseline the Record and Archive PNG goldens with a targeted run, through
   `/sase_monitor`:

   ```bash
   just fix-tui-screenshots -- tests/ace/tui/visual/test_ace_png_snapshots_agents_record.py tests/ace/tui/visual/test_ace_png_snapshots_agents_archive.py
   ```

   - Earlier phases ran visual commands with `TZ=UTC`.
   - `test_archive_help_png_snapshot` fails on master for a pre-existing cause (bisected
     to `ebd17bf5bf`, recorded on `sase-1jm`). Leave it and its golden alone.

3. Read the WARNING block and inspect every updated golden. Expect:
   - `agents_record_archived_lifecycle_120x40` gains `Outcome: ✓ done`, `Last activity`,
     and `Time basis` lines.
   - The Lifecycle headings in the inbox, archived, and unreadable Record goldens read
     `@agent:…`.
   - Nothing else changes. Investigate any other difference.

## Closeout (the final step of this tale)

Closing the epic in this same turn is the normal landing. Never wait for this tale's own
commit, push, or CI.

1. Run `sase bead epic-symbols sase-1jm.4.2`.
   - For each listed `--epic-symbol` entry, either:
     - resolve the symbol (wire it up, privatize it, add a non-test pragma, or delete it
       per the Symvision epic-whitelist policy), or
     - only when a still-open later bead needs the exemption, re-key the Justfile line
       to that open bead.
   - The land agent's run listed none.
2. Close the epic:

   ```bash
   sase bead close sase-1jm.4.2 --note "<verification>"
   ```

   - The note must cover:
     - the land agent's verification: all five phases checked against the plan and
       acceptance items 1–6, and integration review of the concurrent commits with no
       conflicts
     - the follow-up triage already recorded on the bead
     - the three defects fixed here, with their tests and golden rebaselines
     - the `sase tool run check` result
   - Never use `--force` to make the close succeed.

3. Run `just symvision`.
   - The pre-existing `_load_archive_bundle` private-import failure is tracked on
     `sase-1jm` and is not an epic-whitelist problem.
   - Any stale `sase-1jm.4.2` epic-symbol entry is yours to clean up.
4. Set `status: done` in the frontmatter of the epic's plan file: the PLAN path that
   `sase bead read sase-1jm.4.2 -r "..."` shows, `202610/record_deck_refresh.md` in the
   plans repo.
5. The epic's `parent_bead` is phase `sase-1jm.4` (_Record deck for every agent_).
   1. Verify that this epic completed that phase's design. Re-read the _Record deck for
      every agent_ section of `plan:202610/agents_archive_view.md` with
      `sase artifact read`. Check:
      - the spec entry and picker key `r`
      - cycling and persistence, with and without `record`
      - the four cards with their listed facts, including the `@agent:` reference and
        the light-row outcome and time basis for archived agents
      - links loaded off-thread
      - the tests and the inbox and archived goldens
   2. Its other child epic, `sase-1jm.4.1`, is already closed as superseded.
   3. Close only that phase:

      ```bash
      sase bead close sase-1jm.4 --note "<what you verified>"
      ```

   4. Leave notes #1–#2 (glossary follow-ups) and #4 (usage-limit e2e) on `sase-1jm.4`
      for the `sase-1jm` land agent.
   5. Never close `sase-1jm`; its land agent owns it.
