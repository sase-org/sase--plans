---
tier: epic
title: Prompt recall tabs and bounded stash trash
goal: One reliable Prompts overlay unifies draft recall while bounded, transactional
  Trash makes deliberate stash discards recoverable.
phases:
- id: stash_lifecycle
  title: Transactional stash trash in Rust core
  size: medium
  depends_on: []
  description: 'stash_lifecycle: implement the atomic Rust stash lifecycle, binding,
    backup, and data-integrity tests.'
- id: python_contract
  title: Python contract, configuration, and upgrade boundary
  size: medium
  depends_on:
  - stash_lifecycle
  description: 'python_contract: expose strict Python wires and config, then pin a
    core revision with the new binding.'
- id: prompts_shell
  title: Reusable Prompts overlay and existing Stash and History panes
  size: medium
  depends_on: []
  description: 'prompts_shell: build a lazy tabbed modal with preserved Stash, History,
    and origin behavior.'
- id: trash_interactions
  title: Trash pane and reliable staged actions
  size: medium
  depends_on:
  - stash_lifecycle
  - python_contract
  - prompts_shell
  description: 'trash_interactions: add Trash actions, confirmations, authoritative
    repaint, styling, and tests.'
- id: rollout_polish
  title: Atomic entry-point rollout, documentation, and visual acceptance
  size: medium
  depends_on:
  - trash_interactions
  description: 'rollout_polish: route all shortcuts to the complete overlay and finish
    docs, glossary, visuals, and checks.'
proposed_by: bbugyi200.athena.0sy
create_time: 2026-09-26 14:44:18
status: wip
bead_id: sase-1au
---

- **PROMPT:** [prompts/202609/prompt_recall_tabs_and_stash_trash.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/prompt_recall_tabs_and_stash_trash.md)
- **BEAD:** [sase-1au](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1au/README.md)

# Prompt recall tabs and bounded stash trash

## Goal and source

Implement the recommendations in
`research:202609/prompt_recall_tabs_and_stash_trash/prompt_recall_tabs_and_stash_trash.md`.
A single **Prompts** overlay exposes **Stash | History | Trash**. Stash and History
retain their distinct stores and actions. Trash recovers only drafts deliberately
discarded from Stash, up to the configurable row limit. The complete entry, including
bundled pane order, frontmatter, cursor, project, source, creation time, and pin, must
survive Stash → Trash → Stash. Successful unpinned loads from Stash still consume their
rows without entering Trash; history deletion remains outside this overlay.

The work spans the linked `sase-core` repository and this `sase` repository. Shared
lifecycle and retention logic belongs in `sase_core`, exposed through `sase_core_rs`;
Python owns only wire translation, configuration, and TUI presentation. The research's
one-file design is the chosen persistence model because both collections then move under
one existing lock and one atomic replacement. A multi-pane bundle counts as one row.
`ace.prompt_stash.trash_limit` defaults to 20; zero disables recovery and permanently
discards rows marked for Trash. This is an entry-count limit, not a byte quota.

## Product contract

- Keep one stable frame, one tab strip, a consistent list/preview split, and one
  contextual footer. Use `PanelTabStrip` without number shortcuts; label the tabs
  `Stash N`, `History`, and `Trash M/N`. Stash uses its orchid accent, History its cool
  accent, and Trash amber at rest with red marks for permanent deletion. Use compact
  text at narrow widths and collapse the preview there. Keep the frame height and list
  columns stable while switching tabs.
- `[` and `]` cycle through tabs with wraparound; clicking a tab selects it. Handle
  cycling before the focused History filter consumes those keys, while leaving ordinary
  filter typing intact. `Esc` closes the overlay; `q` closes only when focus is outside
  a text input. Each tab retains its highlight, scroll, filter, loaded pages, preview
  position, and staged marks across switches. Lazy-load a tab on first activation;
  opening Stash must not scan history shards or perform disk I/O on the UI thread.
- Stash preserves `Enter`/digit restore to the prompt bar, `Tab`/`a` restore marks,
  `Space` pin, `d`/`D` discard marks, `j`/`k`, and preview scrolling. `Enter` applies
  staged marks, or restores the highlighted row when nothing is marked. A pinned restore
  leaves the row in Stash. Stash `d`/`D` commits to Trash rather than permanent
  deletion. A marked pinned row requires explicit confirmation. Before a batch expected
  to evict trash rows, show the expected permanent-loss count, including marked rows in
  a batch larger than the limit. A successful outcome names the _actual_ evictions
  because other processes may change Trash between preview and commit.
- History preserves the existing focused filter, project-aware seed and scope,
  pagination, cancelled toggle, `Enter` submit, `Tab` load, `Ctrl+G` edit, and `Ctrl+Y`
  copy. No history deletion action is added. The footer names `Enter: submit`.
- Trash is newest-deleted-first, with deletion timestamp visible. `Enter`/digits and
  marked `Tab`/`a` restore to **Stash** while the overlay stays open; they never load
  the prompt bar. `d`/`D` stage permanent deletion, with an explicit confirmation before
  applying it; `Ctrl+Y` may copy. Trash has no pin edit. The footer names
  `Enter: restore to Stash` and labels purge marks `permanently delete`.
- Empty Stash still opens the overlay, explains how to save a draft, and points to Trash
  when it has rows. Empty Trash explains where discarded drafts appear; if the limit is
  zero it says recovery is disabled. Do not jump tabs after discard. Preserve the bare
  `@` one-live-entry fast path; `,@`, `Ctrl+G p`, the stash chip, and empty `Ctrl+S`
  always open Stash. `Ctrl+K`, `,.`, and `,>` open History with their existing seed,
  home/MRU, and cancelled-state rules. Keep direct edit-newest behavior for `,Ctrl+G`.

## Reliability and upgrade contract

- The physical `prompt_stash.jsonl` file keeps legacy active entry lines and adds tagged
  trash envelopes such as `{"kind":"trash","trashed_at":"...","entry":{...}}`. A bare
  `trashed_at` field is forbidden because old serde readers ignore it and would treat
  Trash as active. Under the existing exclusive lock, trash, restore, purge, and limit
  reconciliation read once, transform stable IDs, write one temporary file, fsync it,
  and atomically replace the live file. Return an authoritative active/trash snapshot
  and changed/evicted IDs. Keep `pop` restricted to active rows; pin and all existing
  writers preserve Trash. Reject a stale `rewrite` that would resurrect an ID already in
  Trash. Unknown IDs and repeated transitions are no-ops.
- Preserve the current v1 Rust/Python result shape for **existing** stash bindings so a
  newer core can land before the Python change without breaking legacy active-only
  callers. Give the new lifecycle read and mutation endpoints separate typed, versioned
  results that include both collections and changed/evicted IDs. Never expose the new
  disk format through an older writer as a compatibility claim; preserving the old wire
  only makes upgraded Rust safe for existing Python callers.
- Set `trashed_at` once in UTC per transition. Sort and evict by deletion time with a
  stable tie break based on batch input/order and ID. Trashing a restored row gets a new
  deletion time. Enforce the limit in the same transaction, including when a lowered
  configured limit is first reconciled by a trash-aware open/write after reload. Surface
  reconciliation evictions. Never silently drop malformed rows or future record kinds on
  a rewrite: fail the mutation closed with an actionable error and leave the file
  unchanged, or preserve opaque bytes if that is demonstrably safe; tests must prove the
  chosen behavior.
- The new disk format is unsafe with old Rust writers: they skip tagged Trash rows and
  can erase them during an old `pop`/pin/rewrite. Before the first new-format write,
  create and verify a recoverable pre-upgrade backup under the stash lock; fail closed
  if backup creation fails. Require the new binding and a coordinated
  `sase-core-revision.txt` pin before exposing Trash. Document that users must restart
  old TUI processes before enabling the new behavior, and give a recovery procedure for
  pre-upgrade drafts from the backup. The backup does not protect Trash created
  afterward from an old writer, so mixed-version operation is unsupported. Include a
  regression test that demonstrates the old-reader hazard and validates the new cutover
  behavior.
- On a failed read, lock acquisition, write, confirmation cancellation, or stale
  selection, leave the visible rows and staged marks truthful. After success, repaint
  the affected panes and badge from the returned store outcome, not an optimistic local
  deletion. Keep persistence off-thread and outside Textual's serial pump; recheck the
  active tab, modal mount state, selection ID, and prompt origin after every await.

## Phase 1 — `stash_lifecycle` (medium; no dependencies)

Open `sase-core` with `/sase_repo` and follow its `AGENTS.md`. Extend
`crates/sase_core/src/prompt_stash/{wire,store}.rs` and the existing
`crates/sase_core_py/src/editor_content/` binding domain. Keep existing v1 wire results
intact, add separately versioned lifecycle snapshots/outcomes and PyO3 functions for
lifecycle read, trash, restore, purge, and reconcile. Refactor the parser and all
existing mutators to retain logical Trash rows. Add pre-upgrade backup creation before
first tagged write and explicit handling of malformed/future rows. Keep the existing
active-row disk format readable. Test field-preserving round trips; limits 20, 1, and 0;
lowered limits; batches larger than the limit; equal timestamps; pinned/bundled rows;
concurrent append/trash/restore; stale rewrites; malformed/future lines; crash-safe
replacement and backup failure; and binding registration. Run targeted tests, then the
core repository's guarded `sase tool run check` gate.

## Phase 2 — `python_contract` (medium; depends on `stash_lifecycle`)

Update `src/sase/core/prompt_stash_{wire,facade}.py` for the new Rust lifecycle results
while leaving existing v1 readers intact; keep parsing strict and avoid a Python
lifecycle fallback. Add `ace.prompt_stash.trash_limit: 20` to
`src/sase/default_config.yml` and `src/sase/config/sase.schema.json` (integer, minimum
zero), plus a defensive accessor that rejects booleans and malformed/negative values and
falls back to 20. Pass the limit to Rust rather than making Rust parse `sase.yml`.
Update binding parity checks in `tools/validate_sase_core_rs`, their tests, and
`sase-core-revision.txt` to a committed core revision containing the new binding. Verify
installs with a stale wheel fail clearly before Trash is used; test old and new wire
round trips, config fallback, and pin/schema parity. Do not route users to the new UI
yet.

## Phase 3 — `prompts_shell` (medium; no dependencies)

Build a new `Prompts` modal shell around reusable Stash and History
**widgets/controllers**, not nested `ModalScreen` instances. Extract the current
`StashedPromptsModal` and `PromptHistoryModal` content and interaction logic without
changing their behavior, including their existing tests; the shell owns tab strip,
heading, per-tab footer, focus restoration, and responsive layout. Keep child state
mounted or cached after first activation; defer History's project catalog and first page
until its first activation. Preserve the two History result contexts: a live prompt bar
loads into its captured pane with frontmatter conflict handling, while home/MRU entry
points use their own launch/load/edit callbacks. Define a typed origin/result object so
switching tabs cannot silently apply the initial tab's callback to the wrong action. Add
focused-filter tab-cycle, keyboard, mouse, narrow-width, state-preservation, and
no-history-read-on-Stash-open tests. Keep existing external entry points on the old
pickers until `rollout_polish`.

## Phase 4 — `trash_interactions` (medium; depends on `stash_lifecycle`, `python_contract`, `prompts_shell`)

Add the Trash child pane and the Stash-to-Trash action flow to the new shell. Share the
row/preview treatment between Stash and Trash while keeping their verb, pin, and mark
rules distinct. Compute previews of overflow and pinned consequences, then use explicit
confirmation; re-read the Rust outcome for actual evictions. Replace
`_apply_deletions_in_place` behavior with pending state and authoritative repaint,
including failures and concurrent changes. Coordinate local write tasks with the
existing prompt-stash async lock; use off-thread store calls and a pump-free task where
needed. Cover delete-only and combined restore/discard selections, no accidental move on
successful unpinned restore, Trash restore/purge, stale IDs, empty states, zero limit,
failed writes and lost/changed selection, and counts. Complete the overlay styles and
targeted visual snapshots before routing any entry point to it.

## Phase 5 — `rollout_polish` (medium; depends on `trash_interactions`)

Change all existing Stash and History callers to open the complete overlay on their
correct initial tab. Carry the origin context across tab switches and close/cancel
paths; preserve the bare-`@` direct restore. Update the stash badge only from
authoritative active counts, with Trash count displayed in the overlay. Update
`docs/ace.md`, `docs/configuration.md`, help/keymap text, and a compact demo showing
discard → Trash → restore-to-Stash; update default keymap config if any global key
action changes (none is planned). Explain the entry-count limit, permanent eviction,
zero setting, restart requirement, and backup recovery.

Use `/sase_memory_write` before creating the `Prompt Stash` (alias `stash`) and
`Stash Trash` glossary strands; link them to each other and republish with
`sase memory init`. Add no broad `trash` or `archive` alias. Run interaction tests for
every prior entry point and both History callback origins, the full active → Trash →
active path, failure truthfulness, focused-filter tab navigation, and first-paint
responsiveness. Capture real narrow and wide TUI screenshots and inspect their PNGs;
update and inspect targeted visual goldens with `just fix-tui-screenshots`. Read the
lint/test memory and run `just check` in `sase` (never `just check-full` unless
explicitly requested). The final cutover is complete only when discard and recovery are
available together and all checks pass.
