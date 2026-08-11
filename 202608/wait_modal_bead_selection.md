---
tier: tale
title: Select bead wait targets from the Agents-tab Wait panel
goal:
  Pressing `w` on the Agents tab opens a Wait panel whose Beads field is editable and
  completes against the same canonical bead store that resolves the wait, so a bead gate
  can be added, changed, or removed without hand-editing a launch prompt, and a bead ID
  that would park the agent forever is caught before it is applied.
size: medium
proposed_by: bbugyi200.athena.xs
create_time: 2026-08-11 06:28:53
status: wip
---

# Plan: Select bead wait targets from the Agents-tab Wait panel

## Problem

Bead gating already works end to end everywhere except the one surface where waits are
edited.

`%wait(bead=<id>)` parses into `PromptDirectives.wait_beads`
(`src/sase/xprompt/_directive_values.py:73`), lands in `agent_meta.json` /
`waiting.json` as `wait_for_beads` (`src/sase/axe/run_agent_wait.py:151`), and is
resolved by the `wait_checks` chop plus the runner fallback against the waiting agent's
project bead store (`src/sase/scripts/sase_chop_wait_checks.py:140`,
`src/sase/axe/run_agent_wait_deps.py:58`). The TUI already renders those waits with live
status badges in the prompt panel (`_agent_wait_section.py:228`), agent rows, clan
sections, and tribe summaries.

But `WaitModal` — the `w` panel on the Agents tab — is write-only for agents and
read-only for beads:

- `src/sase/ace/tui/modals/wait_modal.py:400-405` renders the current bead waits as a
  `Static` labelled **"Beads (read-only)"**, and `_apply()` (line 637) hands the
  untouched prefill straight back as `WaitModalResult.beads`.
- `docs/ace.md:1184` states the contract plainly: "Beads the agent is waiting on appear
  above the fields as a read-only summary; they are preserved but cannot be edited
  here."

So the only way to gate an agent on a bead is to type `%wait(bead=...)` into the launch
prompt before launching. Once an agent is parked there is no way to add a bead gate, no
way to swap the bead it is waiting on, and no way to drop a bead gate while keeping the
rest of the wait — `Ctrl+R` (run now) clears _every_ wait condition, which is the only
existing escape.

The write path is also the one place where a typo is unrecoverable. Missing beads,
unavailable stores, and read failures all deliberately fail closed and leave the agent
parked (`docs/axe.md:218-224`); a prior audit recorded this exactly
(`sase/repos/plans/202607/audit_24h_fixes.md:462`: "a typo currently parks the agent
forever"). Opening bead editing without verification would turn a rare launch-prompt
mistake into a routine one.

## Goal

The Wait panel treats beads as a first-class wait target, exactly as symmetric with
agents as the underlying directive already is, and verifies what it is about to write.

1. **Editable.** A `Beads` field, prefilled from the agent's current bead waits, sits
   directly under `Agents` and accepts the same comma-separated grammar.
2. **Completable.** The field completes against the agent's project bead store, with
   rows that read like every other bead surface in SASE.
3. **Pickable implies resolvable.** Candidates come from the same canonical store the
   wait resolver consults, so a bead offered by the picker is a bead the resolver can
   see.
4. **Verified.** A live preview classifies every typed ID, and applying a wait that can
   never resolve takes a deliberate second `Enter`.
5. **Responsive.** No bead-store I/O on the Textual event loop, ever.

## Semantics to implement

These are the acceptance criteria; implement whatever code satisfies them.

### Field and result

1. `Beads` is an editable `_WaitInput` between the `Agents` field's completion list and
   `Time`, prefilled with the agent's current `waiting_for_beads` joined by `", "`. The
   read-only `#wait-beads-summary` `Static` is removed.
2. `WaitModalResult.beads` is parsed from that field with the existing
   `_parse_agents_value` comma grammar, order-preserving and de-duplicated.
3. Clearing every field (agents, beads, time, runners, priority) is still "run now":
   `_apply()`'s `run_now` test reads the parsed beads instead of the constructor
   prefill.
4. `Ctrl+R` still returns `run_now=True` with `beads=[]`, unchanged.

### Candidates

5. Candidates are every **non-closed** bead in the agent's project, read from the
   canonical store resolved by `canonical_beads_dir_for_project` — the same store
   `closed_bead_ids_for_project` reads for resolution. A closed bead cannot gate
   anything, so it is never offered.
6. The agent's own `epic_bead_id` and `phase_bead_id` are excluded: an agent that waits
   on the bead it exists to close can never start.
7. Ordering is deterministic: status rank first (`in_progress`, `claimed`, `ready`,
   `open`, `snoozed`), then `updated_at` descending, then ID. The bead someone is
   actively working is the one most likely to be gated on.
8. The filter fragment matches bead ID or title, case-insensitively, as a substring —
   mirroring `WaitAgentCandidate.search_text`.
9. At most 100 rows render per keystroke; when the filtered set is larger, a trailing
   dim, disabled row reads `…N more — keep typing`. Nothing is silently dropped.

### Rendering

10. Each row uses the canonical cross-surface bead language, not a new one:
    `bead_status_presentation(...).tui_glyph` / `.rich_style` for the status glyph
    (`src/sase/bead_status_presentation.py`), a bold truncated ID, the title, and a dim
    `status · <type chip> · ⌂ <age>` detail built from `bead_type_presentation` and
    `bead_age_label` / `BEAD_CREATED_GLYPH` — the same detail grammar
    `_artifact_ref_entity_catalogs.py:107` already uses for bead completion.
11. A bead already present in the field renders with a dim `· selected` suffix and keeps
    its position, so accepting it twice is visibly a no-op rather than a silent one.

### One completion list at a time

12. `#agent-completion` and `#bead-completion` each sit directly under their own input,
    and exactly one is displayed at a time: the list belonging to whichever of the two
    completion inputs was focused most recently, defaulting to `Agents` on mount.
13. Focusing `Time`, `Runners`, or `Priority` does not change which list is displayed,
    so tabbing down the panel never makes it jump or resize. Panel height is therefore
    exactly three lines taller than today (label, input, preview), not nine.
14. The hidden list is not focusable, which keeps `shift+tab` traversal linear.

### Preview and verification

15. A `#beads-preview` `Static` under the field reuses the existing `wait-time-neutral`
    / `wait-time-valid` / `wait-time-error` class trio and reports:

    | State                               | Class   | Message                                                                                                                               |
    | ----------------------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------- |
    | empty                               | neutral | `starts when every listed bead is closed`                                                                                             |
    | catalog loading                     | neutral | `loading <project> beads…`                                                                                                            |
    | no project / store unavailable      | neutral | `bead store unavailable — IDs not verified`                                                                                           |
    | all known, none closed              | valid   | up to three `<id> <glyph> <status>` entries, then a `N beads · N open · N in progress` aggregate                                      |
    | some known ID already closed        | valid   | valid message plus `<id> is already closed — it will not hold the launch`                                                             |
    | unknown ID, or the agent's own bead | error   | `<id> is not in <project>'s bead store — the agent would park forever`, or `<id> is this agent's own bead — it can never close first` |

16. **Risky-wait guard.** `Enter` while the beads preview is in the error state does not
    dismiss: it focuses the beads field and the footer becomes
    `enter again to wait on unverified beads | esc cancel`. A second `Enter` applies as
    typed. Any edit to the beads field, or leaving the error state, disarms the guard.
    This is deliberately a soft block, not a hard one: bead stores sync through git, so
    an ID that is genuinely valid upstream can be legitimately absent locally, and a
    hard block would trap that user. The unavailable-store state is neutral, never
    error, so a missing store never arms the guard.
17. Existing hard validation is unchanged: an invalid time, runners, or priority value
    still blocks apply and focuses its own field, and those checks run before the bead
    guard so the most objective failure is reported first.

### Loading

18. The bead catalog loads in a thread worker started from `on_mount`. The field accepts
    typing immediately; the list shows a dim `loading beads…` placeholder row and the
    preview shows its loading message until results land. A failed or empty load
    degrades to the unavailable-store state, never to an exception.
19. Reads are cached and revalidated by the store index's mtime/size, so reopening the
    panel or filtering does not re-read the store.
20. No bead-store call happens on the event loop, in a render path, or in a keystroke
    handler (`sase/memory/tui_perf.md` rules 1, 8, 11).

### Navigation

21. `tab` accepts the highlighted completion when there is one, exactly as today, and
    otherwise falls through to `focus_next`. Today `tab` is a dead key when no candidate
    is highlighted, so this is strictly additive and makes the new field reachable
    forward as well as backward.
22. `enter` on the focused bead completion list accepts the highlighted row, matching
    the existing agent-list behavior in `on_key`.

## Design decisions

- **The picker reads the resolver's store, not a broader catalog.** ACE already has a
  bead completion catalog for artifact references
  (`_artifact_ref_entity_catalogs.load_bead_candidate_catalog`), but it spans every
  store in `ArtifactRefContext.bead_stores`. Offering a bead the resolver cannot see
  would produce exactly the permanent park this plan exists to prevent, so the new
  loader goes through `canonical_beads_dir_for_project` — the resolver's own path.
- **Warn-then-confirm beats hard rejection.** Bead stores are git-synced, so "not in the
  local store" is strong evidence of a typo but not proof. The two-`Enter` guard makes
  the common mistake loud and the rare legitimate case one keystroke more expensive.
- **Swap the list rather than stack two.** Two permanently visible 6-row lists would
  push the panel past a 32-row terminal. Swapping keeps the list under the field being
  edited, holds total height constant, and reuses the existing list styling.
- **Reuse the canonical bead presentation modules.** `bead_status_presentation` and
  `bead_type_presentation` exist precisely so that a bead looks the same in the CLI, the
  Artifacts tab, and here. No new glyph or color is introduced by this work.
- **Split the modal file.** `wait_modal.py` is already 640 lines against the
  `just toobig` 700/850/1000 thresholds. Bead catalog loading, row rendering, and
  validation live in their own modules so the modal file grows by roughly the size of
  the field wiring alone.
- **Rust core boundary.** The store read goes through the existing Python adapters over
  the Rust-backed bead store; the candidate ranking and row shaping are picker
  presentation and stay here. Note for the future: if a web or CLI frontend ever needs
  the same "beads this agent may wait on" list, the filter and rank rules are the part
  that should move to `../sase-core`, and they are deliberately isolated in one pure
  function to make that move cheap.

## Non-goals

- Cross-project bead waits. Resolution is single-project by construction
  (`closed_bead_ids_for_project(project_name)`); offering another project's beads would
  be a backend change, not a panel change.
- Space-toggle multi-select inside the completion list. The comma-separated field plus
  completion already covers both typing and picking, and diverging from the Agents field
  would cost the shared mental model.
- A separate bead browser modal, bead search syntax, or status/type filter chips.
- Any change to `%wait(bead=...)` parsing, wait resolution, the chops, the launch-prompt
  path, or the bead store itself.
- Bead waits from any surface other than the Agents-tab Wait panel.
- Changing how bead waits render outside the panel; those surfaces are already correct.

## Relevant code map

Read before editing:

- `src/sase/ace/tui/modals/wait_modal.py` — the panel. `WaitModalResult` (49),
  `_parse_agents_value` / `_active_fragment` / `_replace_active_fragment` (101-116),
  `_candidate_option` (272), `compose` (391), `on_key` (449), `_refresh_completion`
  (517), `_accept_candidate_index` (591), `_apply` (606).
- `src/sase/ace/tui/actions/agents/_wait_actions.py` — `_wait_agent` (52) builds the
  modal; `_apply_wait` (124) and `_apply_live_runner_wait` (307) persist the result.
- `src/sase/ace/tui/actions/agents/_directive_persistence.py` —
  `wait_meta_patch_for_token` (129) already lists `wait_for_beads` in `remove_keys`, and
  `_write_waiting_marker` (268) already pops it, so an emptied field clears correctly;
  this plan adds the regression tests that pin that.
- `src/sase/ace/tui/actions/agents/_wait_helpers.py` — `wait_spec_label` (53),
  `result_has_wait_spec` (83), `prompt_wait_spec` (94) already carry beads.
- `src/sase/bead/store_locator.py` — `canonical_beads_dir_for_project`,
  `open_bead_project_for_beads_dir`, `closed_bead_ids_for_project`,
  `bead_statuses_for_project`.
- `src/sase/ace/tui/models/agent_wait_beads.py` — the TTL-bounded status cache whose
  shape the new catalog cache follows.
- `src/sase/bead_status_presentation.py`, `src/sase/bead_type_presentation.py`,
  `src/sase/bead_time_presentation.py` — canonical glyphs, colors, chips, ages.
- `src/sase/ace/tui/widgets/_artifact_ref_entity_catalogs.py` — the mtime-keyed store
  cache and bead detail grammar to mirror.
- `src/sase/ace/tui/styles.tcss:5152-5232` — the `WaitModal` block.

## Step 1 — Candidate source in the bead store locator

In `src/sase/bead/store_locator.py`, add a peer of `closed_bead_ids_for_project`:

```python
def open_bead_candidates_for_project(project: str) -> tuple[Issue, ...] | None:
    """Return non-closed beads from the canonical store, or ``None``."""
```

- Resolve through `canonical_beads_dir_for_project` and
  `open_bead_project_for_beads_dir`, exactly as its peers do, so the picker and the
  resolver can never disagree about which store is canonical.
- Request every status except `Status.CLOSED` at the store boundary rather than
  filtering in Python.
- Return `None` when the store is unavailable and swallow exceptions the same way the
  neighbouring helpers do — an unavailable store is a neutral UI state, not a crash.

## Step 2 — Wait bead catalog model

New module `src/sase/ace/tui/models/wait_bead_catalog.py`:

- `WaitBeadCandidate` — frozen dataclass with `bead_id`, `title`, `status`,
  `type_label`, `created_at`, `updated_at`, and a `search_text` property over ID and
  title.
- `WaitBeadCatalog` — frozen dataclass with `candidates: tuple[WaitBeadCandidate, ...]`,
  `available: bool` (False when the store could not be read or no project is known), and
  a `by_id` mapping for validation lookups.
- `load_wait_bead_catalog(project_key, *, own_bead_ids)` — worker-thread entry point.
  Calls Step 1, drops `own_bead_ids`, converts to candidates, and applies the ordering
  from semantics 7. Results are cached and revalidated by the store index's mtime and
  size, following `_artifact_ref_entity_catalogs._read_cached_bead_store`'s pattern and
  bounded like `agent_wait_beads`'s cache.
- `filter_wait_bead_candidates(catalog, fragment, *, limit)` — pure, returning the
  bounded rows plus the omitted count for the `…N more` row.
- `classify_wait_bead_selection(catalog, bead_ids, *, own_bead_ids, project_label)` —
  pure, returning the preview state (`neutral` / `valid` / `error`), the message, and
  whether the risky-wait guard should arm. This is the function that would move to
  `../sase-core` if another frontend ever needs it.

Keep it presentation-free (no Rich, no Textual) so it can be unit-tested directly.

## Step 3 — Bead rows and panel wiring

New module `src/sase/ace/tui/modals/wait_modal_beads.py` holding the Rich row builder
(`bead_candidate_option`), the loading/empty/overflow placeholder rows, and the
`_BeadsValidation` dataclass that mirrors the existing `_TimeValidation` shape.

In `wait_modal.py`:

- Constructor gains `bead_project_key: str | None`, `own_bead_ids: frozenset[str]`, and
  a `bead_catalog_loader` seam defaulting to `load_wait_bead_catalog`, so unit tests and
  the PNG snapshot inject a deterministic catalog with no store on disk.
- `compose` replaces the read-only summary with the `Beads` label, `_WaitInput`,
  `_BeadCompletionList`, and `#beads-preview`, and updates the summary line to "Wait for
  agents, beads, a time floor, and/or a runner threshold."
- `on_mount` starts the catalog worker (`run_worker(..., thread=True)`) and applies the
  result on the UI thread, re-running the filter and preview once it lands.
- `_refresh_completion` generalizes over the active completion field; keep the
  `_programmatic_highlight` guard around every programmatic `highlighted` assignment for
  both lists (`sase/memory/tui_perf.md` rule 12).
- `_apply` validates time, runners, and priority first, then the beads guard, then
  dismisses with the parsed beads.

`_wait_actions.py:_wait_agent` passes the project key — derived the same way
`agent_wait_beads._wait_bead_project_key` derives it, from `project_file`'s parent
directory name — and the agent's own `epic_bead_id` / `phase_bead_id`.

## Step 4 — Styling

In `src/sase/ace/tui/styles.tcss`, replace the `#wait-beads-summary` rule with
`#bead-completion` styling that reuses the `#agent-completion` list, highlight, and
padding rules verbatim (extend the existing selectors rather than duplicating blocks),
and add `#beads-preview` to the three `wait-time-*` preview selector lists. Add
`overflow-y: auto` to `WaitModal > Container` so the taller panel degrades to a scroll
on short terminals instead of clipping its footer.

## Step 5 — Documentation

- `docs/ace.md:1178-1205` (§Wait Modal): five editable fields, not four; delete the
  read-only sentence; document candidate scope and ordering, the preview states, the
  two-`Enter` guard, the single visible completion list, and `tab`'s fall-through.
- `src/sase/ace/tui/modals/help_modal/agents_bindings.py:90` — the `w` description must
  mention beads; keep it inside the help modal's documented column budget
  (`src/sase/ace/CLAUDE.md`, "Help Modal Box Formatting") and let that suite verify it.
- The footer keybinding convention in `src/sase/ace/CLAUDE.md` needs no change: this
  adds no Agents-tab keymap, only fields inside an existing panel.
- Add a CHANGELOG entry consistent with the repo's existing `feat(ace)` phrasing.

## Step 6 — Tests

Update in `tests/ace/tui/test_wait_modal.py`:

- `test_modal_displays_and_preserves_read_only_bead_waits` becomes a prefill/round-trip
  test against the editable field.
- `test_modal_run_now_cancels_bead_waits` still passes unchanged; keep it as the
  `Ctrl+R` pin.

Add:

1. Typing bead IDs returns them in `WaitModalResult.beads`, order-preserved and
   de-duplicated.
2. Emptying a prefilled beads field while agents remain returns `beads=[]`.
3. Emptying every field returns `run_now=True` (regression against reading the prefill).
4. Completion filters on ID and on title, and `tab` inserts the highlighted bead with a
   trailing comma.
5. Focusing `Beads` displays `#bead-completion` and hides `#agent-completion`; focusing
   `Time` afterwards leaves that unchanged.
6. `tab` with no highlighted candidate moves focus instead of doing nothing.
7. The risky-wait guard: first `Enter` with an unknown ID does not dismiss and
   re-focuses beads; second `Enter` dismisses with the typed IDs; editing the field
   disarms it.
8. An unavailable catalog never arms the guard and reports the neutral message.
9. The agent's own phase bead is absent from candidates and is an error when typed.

New `tests/ace/tui/models/test_wait_bead_catalog.py` (pure, no Textual): ordering across
all five non-closed statuses, own-bead exclusion, filter matching and the overflow
count, and every branch of `classify_wait_bead_selection`.

New coverage in `tests/test_bead_statuses_for_project.py`'s style for
`open_bead_candidates_for_project`: closed beads excluded, unknown project returns
`None`, using the `BeadProject.init` + `monkeypatch` pattern already established there.

Action-level, alongside the existing wait/fork suites (`test_agent_marking_wait_fork.py`
and its neighbours): applying a bead-only edit to a `WAITING` agent writes
`wait_for_beads` to both `agent_meta.json` and `waiting.json`, and clearing beads while
keeping an agent dependency removes the key from both — the behavior `remove_keys` and
`_write_waiting_marker` already implement but nothing currently pins.

Visual: regenerate `wait_modal_100x32.png` (the panel gained a field) and add one
snapshot with the beads field focused over an injected catalog, so bead row rendering is
covered. Use `just test-visual` with `--sase-update-visual-snapshots` and inspect the
artifacts under `.pytest_cache/sase-visual/` before accepting.

## Verification

```bash
just install
just check
just test-visual   # PNG goldens change; regenerate and inspect the diff artifacts
just check-full    # shared bead-store helper + modal + snapshots changed; run before landing
```

Manual pass in a real ACE session, since the point of the feature is the interaction:

1. `w` on a `WAITING` agent, type a fragment in `Beads`, confirm rows render with status
   glyphs, pick one with `tab`, apply, and confirm the agent row and prompt panel show
   the new bead wait with its status badge.
2. Close that bead elsewhere and confirm the agent launches on the next `wait_checks`
   cycle — the whole point of picking from the resolver's own store.
3. Reopen `w`, clear the beads field, apply, and confirm `wait_for_beads` is gone from
   both markers and the agent is no longer bead-gated.
4. Type a nonsense ID and confirm the error preview plus the two-`Enter` guard.
5. Confirm the panel stays responsive while the catalog loads on a project with a large
   bead store, and that `~/.sase/logs/tui_stalls.jsonl` records no new stall.

## Risks and mitigations

- **A large bead store makes the panel slow to open.** Mitigated by the worker load, the
  mtime-keyed cache, the 100-row render cap, and by the field being usable while
  loading. Verified by manual step 5 against a large store.
- **The guard becomes a nuisance if a store is routinely stale.** Mitigated by making
  store-unavailable a neutral state, so the guard only arms when the store was read
  successfully and the ID was genuinely absent.
- **The list swap surprises someone mid-edit.** Mitigated by constant panel height, by
  the list always sitting under the field it completes, and by non-completion fields
  never changing which list shows.
- **Bead editing on a `RUNNING` agent silently kills it.** Unchanged existing behavior —
  `_apply_wait_running` routes through `ConfirmKillModal`, and `wait_spec_label` already
  names the beads in that confirmation.
