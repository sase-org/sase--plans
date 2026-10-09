---
tier: tale
size: small
title: Finish landing epic sase-1h8 (bead store performance)
goal:
  The last caller that hydrates every closed bead uses point reads instead. The
  perf-gate docs name the follow-up tasks that own each known miss. Epic sase-1h8 and
  its nested child-epic plan files are closed out.
proposed_by: bbugyi200.athena.sase-1h8.land
bead: sase-1h8
status: done
---

- **PARENT:**
  [202610/bead_store_history_independent_performance.md](https://github.com/sase-org/sase--plans/blob/main/202610/bead_store_history_independent_performance.md)
- **BEAD:**
  [sase-1h8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1h8/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-1h8.land](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.sase-1h8.land.md)
- **COMMITS:**
  - [c0b3636](https://github.com/sase-org/sase/commit/c0b36364a3c4ff10df42c3b29d884a49d7c5487d)
    — feat(beads): land sase-1h8 closeout with epic-follow point reads and perf-gate
    owners

# Finish landing epic sase-1h8

The sase-1h8 land agent verified the epic. All 14 phases are closed, as are the nested
child epics sase-1h8.13.1 and sase-1h8.13.1.9. The read model matches replay on the live
store (`sase bead doctor --verify-cache`), and every epic DISCOVERED ISSUE is resolved
or routed. Every phase PROPOSED FOLLOW-UP was triaged into task beads sase-1it,
sase-1iu, sase-1iv, sase-1iw, sase-1ix, sase-1iy and sase-1iz, or declined (see the LAND
TRIAGE note on sase-1h8). The superseded tasks sase-1h5, sase-17r and sase-x3 are
already closed.

Three small items remain. This tale must also close the epic: nothing resumes the
landing after it.

Do not create beads in this tale. Do not edit `sase/memory/**`: the stale
`sase_beads.md` wording is already filed as sase-1iz.

## Step 1: integration — epic-follow progress stops hydrating every closed bead

`_read_epic_follow_progress` in `src/sase/ace/tui/models/agent_epic_follow_progress.py`
landed mid-epic (sase-1h7, 0a80039618). After computing per-epic child progress, it
calls `bead_project.list_issues(statuses=[Status.CLOSED])`. That hydrates every closed
bead (~6,600 rows, ~480 ms on the live store, growing with closed history) only to read
the `resolution` of the few followed epics. Epic sase-1h8's goal is that hot reads stop
scaling with closed history. A point read through the read model costs ~16 ms per epic
and is history-independent.

Change it so that:

- `list_issues` is no longer called. For each epic whose progress resolved (not `None`),
  call `bead_project.show(epic_id)` inside a `try`/`except Exception`. A failure leaves
  that epic's progress without a resolution, as today.
- A resolution is attached only when the shown issue's `status is Status.CLOSED` and
  `issue.resolution is not None`. Use `issue.resolution.value`. This preserves today's
  "closed issues only" semantics.
- Everything else is unchanged: the per-epic `get_epic_children` progress, the
  `_EpicFollowProgress` shape, the outer `except Exception` returning
  `dict.fromkeys(wanted)`, and the "off the Textual event loop" contract.

Tests: add focused tests next to the existing follow-progress tests in
`tests/ace/tui/test_agent_wait_epic_follow_tui.py`, or in a new sibling module if that
file is near its size cap.

- Monkeypatch `sase.bead.store_locator.canonical_beads_dir_for_project` and
  `sase.bead.store_locator.open_bead_project_for_beads_dir`. The function imports them
  inside its body, so patch the module attributes.
- Use a fake bead project (a context manager) providing `get_epic_children` and `show`.
  Its `list_issues` should raise `AssertionError`.
- Assert:
  - a closed epic with resolution `canceled` yields `resolution="canceled"`;
  - an open epic yields no resolution;
  - an epic whose `show` raises keeps its progress with no resolution;
  - an epic whose `get_epic_children` raises maps to `None`;
  - `list_issues` is never called.

## Step 2: name the follow-up tasks that own each known perf-gate miss

The perf gate (sase-1h8.14) records misses as allowed in CI. Its comments say each miss
"cites its sase-1h8.14 follow-up". Those follow-ups are now task beads, so name them in
both places:

- In `docs/perf_runbook.md`, section "Bead history-independence gate (epic sase-1h8,
  phase sase-1h8.14)", subsection "Verdict and known misses", change the lead-in
  "(follow-ups on bead sase-1h8.14)" so it says each miss is owned by a task bead. Then
  append the owning task to each bullet:
  - `ratio:list` / `abs:active-list` → sase-1iw
  - `ratio:detail` / `abs:point-read` → sase-1iv; append one sentence: the cause is
    `neighborhood_in` in sase-core `read_model/queries.rs`, which loads every issue id
    and scans every `link_provenance` row
  - `ratio:note` / `ratio:update` → sase-1iu
  - `abs:tui-nochange` → sase-1ix

  Keep the existing measurements and wording otherwise.

- In the `Justfile`, edit the comment above `bead-perf-scale-gate`. Replace "(each cites
  its sase-1h8.14 follow-up)" with "(each is owned by a follow-up task listed in
  docs/perf_runbook.md: sase-1iu, sase-1iv, sase-1iw, sase-1ix)". Do not change the
  recipe body, the `--gate-allow` list, or the tolerance. The thresholds stay exactly as
  landed.

Run `just fmt` afterwards so Prettier rewraps the Markdown.

## Step 3: verify

- Run `sase tool run check`. It runs `just check`; do not run `just check-full`.
- `just symvision` was red on master 563f046a85 only for five unused publics in
  `src/sase/plugins/declared_commands.py`. Those came from sase-1if.6 (bd68ee4942). They
  belong to active epic sase-1if, which already has a DISCOVERED ISSUE note about them.
  This tale does not touch them. Treat any other new failure as yours, following the
  triage labels.

## Step 4: close out epic sase-1h8 (final step)

1. Run `sase bead epic-symbols sase-1h8`. At planning time it listed no entries. If
   entries now appear, resolve each one: wire it up, privatize it, add a non-test
   pragma, or delete it (read `symvision.md` with `/sase_memory_read`). Re-key a
   Justfile `--epic-symbol` line only to a still-open bead that genuinely needs the
   exemption.
2. Close the epic. Use a note file, so the text survives shell quoting:

   ```bash
   sase bead close sase-1h8 --note @<note-file>
   ```

   The note file content:

   > Land verification (sase-1h8.land plus closeout tale): all 14 phases are closed, and
   > so are child epics sase-1h8.13.1 and sase-1h8.13.1.9. Commits were reviewed in sase
   > (545caa7abd..10385fe3c3) and sase-core (6573ebb0..d2a954b0). The CI pin 5c4033f6
   > contains every binding sase calls; later core commits added no bindings, so the
   > routine ratchet covers them.
   >
   > Live store (read-only):
   >
   > - `doctor --verify-cache`: cache matches replay (7,247 issues).
   > - Cache status and seal-watch lines render: 2,175 hot files, 15 ms sweep, 193 MiB,
   >   all OK.
   > - ready, blocked, list, page, stats, closed_ids, show, detail, resolve, search and
   >   statuses_for_ids match a replay copy with no git dir exactly.
   > - issues.jsonl is ignored and untracked in the beads sidecar.
   >
   > Epic DISCOVERED ISSUEs: #1/#4 are privatized, #2/#3 are fixed, and #6 is routed to
   > sase-1iu.
   >
   > Integration: `_read_epic_follow_progress` (sase-1h7) now uses point reads instead
   > of hydrating every closed bead. The perf runbook and Justfile gate comment name the
   > owning tasks.
   >
   > A1: ready passes (1.03x from 1x to 8x). The list, detail, mutation and TUI misses
   > are tracked as sase-1iw, sase-1iv, sase-1iu and sase-1ix, per the plan's perf-gate
   > rule.
   >
   > Follow-ups filed: sase-1it, sase-1iu, sase-1iv, sase-1iw, sase-1ix, sase-1iy,
   > sase-1iz. Superseded tasks closed: sase-1h5, sase-17r, sase-x3.
   >
   > `sase tool run check` result: <fill in the run id and verdict>.
   >
   > epic-symbols: empty.

   Never pass `--force`. If the close is rejected because epic-symbol entries remain,
   finish step 4.1 and close again. If it is rejected for any other reason, stop and
   report it rather than forcing.

3. Run `just symvision`. Confirm that no sase-1h8 epic-symbol whitelist entries remain.
   The sase-1if.6 `declared_commands.py` findings above are expected and not this
   epic's.
4. Mark the plan files done. Open the plans sidecar with `/sase_repo` (repo rules).
   - Set `status: done` in the frontmatter of the epic's plan file, the PLAN path shown
     by `sase bead read sase-1h8`
     (`plan:202610/bead_store_history_independent_performance.md`).
   - Set `status: done` in the two nested child-epic plans whose beads are already
     closed but whose files still say `status: wip`:
     - `plan:202610/finish_read_model_mutations_child_epic.md` (bead sase-1h8.13.1)
     - `plan:202610/unify_bead_mutation_algorithms.md` (bead sase-1h8.13.1.9)
5. Epic sase-1h8 has no `parent_bead`, so nothing further cascades.
