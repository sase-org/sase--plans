---
tier: tale
goal: Persist Agents-tab group and whole-panel fold state across sase ace restarts
  without adding work to the TUI startup critical path or degrading navigation responsiveness.
create_time: 2026-07-15 19:12:12
status: wip
prompt: 202607/prompts/agents_fold_state_persistence.md
---

# Plan: Persist Agents-tab fold state without slowing startup

## Context and outcome

The Agents tab currently has two independent, session-only collapse layers:

- grouping-tree folds owned per grouping mode and per rendered panel scope, such as the collapsed `Done` status group in
  the untagged panel; and
- whole tag-panel folds in the split layout, such as the collapsed `#chop` panel.

The grouping-tree registry already distinguishes panel keys, split versus merged layouts, and grouping modes, while
refresh reconciliation removes keys whose groups or panels disappear. Whole-panel collapse state is kept separately so
expanding a panel restores its inner group folds. Preserve those semantics across a controlled TUI quit or restart: once
startup data has settled, the same still-valid groups and panels should be folded, new groups should remain expanded,
and stale or malformed persisted entries should never break startup.

This is presentation-only ACE TUI state and remains in the Python/TUI layer. Do not broaden the change to workflow-step
folds, ChangeSpecs folds, AXE lumberjack folds, Tools/detail levels, focused panel/row selection, or the split/merged
layout preference.

## Persisted state model

Introduce one versioned, bounded ACE state file under the SASE home directory for Agents-tab folds. Keep grouping-mode
persistence backward compatible and independent. The fold snapshot should contain only collapse intent:

- collapsed whole-panel keys for the split tag-panel layout, including an unambiguous representation of the untagged
  `None` key;
- non-empty collapsed group-key tuples, scoped by `GroupingMode` and `AgentPanelFoldScope` (`panel_key` plus the
  split/merged bit); and
- a schema version for safe future evolution.

Use deterministic serialization and validate types, enum values, key depths, entry counts, and total file size on read.
Missing files, unknown versions, invalid JSON, or malformed entries should fail open to the current all-expanded
defaults and log a diagnostic without notifying or blocking the user. Ignore invalid entries without allowing partially
decoded mutable structures into app state. Omit empty registries/scopes when saving so the file remains small.

Expose explicit snapshot/import operations on the existing fold owners rather than reaching into their private
dictionaries from actions. Restoring a snapshot must preserve cache invalidation guarantees: replacing a registry or
changing its collapsed set must produce a new identity/version visible to the existing navigation and render cache
signatures. After restore, rebind `_group_fold_registry` to the active grouping mode and reconcile the active
mode/layout against the current Agents projection; inactive mode/layout state can remain lazy and be reconciled when it
becomes active, avoiding startup-time tree construction for every combination.

## Non-blocking lifecycle and race handling

Never read, parse, or write the fold-state file synchronously in `AceApp` construction, `on_mount`, a key action, a
message handler, or a render/navigation path.

Start a one-shot off-thread load from the existing post-first-paint startup orchestration as a non-gating background
operation. It must not become an awaited serial stage for first paint, the Agents loader, the AXE loader, or the
startup-stopwatch completion condition. If the bounded snapshot is ready before the first real Agents projection is
finalized, install it before that projection renders. If it is not ready, allow startup to finish normally and apply it
exactly once afterward through the existing in-memory Agents refilter/panel-refresh path, with no agent disk reload.
This gives the normal fast-file case a fold-correct first Agents render while guaranteeing that a slow or pathological
state file cannot lengthen startup.

Make late restoration safe under interaction. Until the one-shot load resolves, record fold mutations as a small ordered
intent journal (mode, panel scope/key, group key where applicable, and collapsed/expanded result). Install the loaded
baseline, replay newer in-session intent so user keystrokes always win, then clear the journal. A missing/invalid load
is still a resolved empty baseline. Queue persistence requests until this merge is complete so an early keypress cannot
overwrite unseen persisted modes or scopes with an incomplete snapshot.

After each successful Agents group collapse/expand and whole-panel collapse/expand, update the UI first, then enqueue an
immutable latest snapshot for persistence. Also persist semantic pruning performed when groups or panels disappear and
the whole-panel clear performed when switching to the merged layout. Coalesce rapid actions to one writer at a time with
generation/latest-wins behavior so an older slow write can never become the final on-disk state. Perform deterministic
serialization and an atomic temporary-file replacement entirely off-thread. Failures are best-effort and logged; they
must not roll back an already-responsive fold action.

Add a controlled-quit/restart flush point that awaits the already queued latest generation off-thread before exiting,
including the ordinary quit, quit-confirmation, stop-AXE-and-quit, and restart-TUI paths. Do not perform synchronous
disk I/O in `_do_quit`; centralize the asynchronous flush so every exit path shares it. Bound failure handling so a
persistence error cannot trap the user in the TUI.

## Integration points

- Add a focused persistence module beside the ACE/TUI state models for the versioned file format, defensive decoding,
  immutable snapshots, and atomic save.
- Extend `AgentGroupFoldRegistry`/`GroupFoldRegistry` with narrow snapshot and restore APIs that preserve panel/mode
  scope and version/cache behavior.
- Add app-state fields for load/apply status, the pre-load mutation journal, and coalesced save generations.
- Wire restoration into post-first-paint startup and the existing first Agents finalize/apply boundary without creating
  a second Agents loading path.
- Route all successful group and whole-panel mutation sites, layout clearing, and reconciliation pruning through a
  shared “fold state changed” scheduler. Keep the hot navigation/render paths disk-free.
- Extend lifecycle cleanup so pending persistence is drained consistently before controlled exit or restart.

## Verification

Add focused model/persistence tests for deterministic round trips across every grouping mode, tagged and untagged panel
keys, split and merged group scopes, empty-state omission, schema/version rejection, malformed and oversized input, and
atomic-save failure behavior.

Add action/lifecycle tests that cover:

- collapsing `Done` in the untagged `BY_STATUS` tree and collapsing `#chop`, saving, constructing a fresh app session,
  and restoring both states;
- independence of equal group keys in different tag panels, grouping modes, and split/merged scopes;
- new or vanished groups/panels defaulting expanded and stale persisted keys being pruned and eventually saved;
- an expand after a collapse persisting the expanded result rather than resurrecting stale collapse intent;
- rapid collapse/expand sequences coalescing to the latest snapshot even when an earlier write is in flight;
- a fold action racing the initial load, with the persisted baseline installed first and the newer user intent replayed
  last;
- switching to merged panels clearing/persisting whole-panel folds while retaining separately scoped inner group folds;
- corrupt/missing state leaving startup usable and all-expanded; and
- every controlled quit/restart route waiting for the latest queued generation without synchronous event-loop I/O.

Add a startup contract test with an intentionally blocked fold-state loader proving that first paint, the existing
Agents/AXE startup workers, and startup-stopwatch completion do not await it. Add the complementary fast-load test
proving that a ready snapshot is present for the first real Agents projection, plus a late-load test proving restoration
uses only an in-memory refresh and preserves current selection/focus as far as the folded rows allow.

Before implementation, capture repeated baseline `sase ace --profile` startup runs in the same environment. After
implementation, repeat warm- and cold-cache runs both with no fold file and with the maximum accepted fold file.
Acceptance requires no fold-state read/JSON/serialization span on the event-loop or awaited startup critical path, and
no regression in first-paint or startup-stopwatch median/p95 beyond measurement noise. Use `SASE_TUI_TRACE=1` to confirm
placement of the new work, and run the Agents navigation benchmark / `SASE_TUI_PERF=1` to retain the existing
p95-under-16-ms target for fold and j/k interactions.

Run targeted fold, grouping, startup, lifecycle, display, and visual tests; inspect/update a visual snapshot only if the
restored collapsed layout intentionally changes the covered first stable frame. Finally run `just install` as required
for the workspace, then `just check` for the complete repository gate.

## Acceptance criteria

- The screenshot scenario survives a `sase ace` restart: `Done` and `#chop` are folded again whenever those identities
  still exist.
- Fold state remains isolated by grouping mode, panel key, and split/merged tree scope, with new identities expanded by
  default.
- Corrupt, stale, oversized, or future-version state cannot delay or prevent startup.
- No fold persistence I/O or decoding occurs on the Textual event loop, keypress path, render path, or startup critical
  path; startup metrics do not regress.
- Rapid interaction and quit/restart preserve the latest user intent, and all tests plus `just check` pass.
