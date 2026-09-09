---
tier: epic
status: done
title: Notification +1 corroboration and ci_watch incident-combination dedup
goal: "A repeated CI failure adds a quiet, visible +1 note to its existing notification
  instead of a new notification; a new notification arrives only when a repo that no
  existing CI-failure notification covers starts failing.

  "
phases:
  - id: core
    title: Rust notification store +1 model and upsert
    depends_on: []
    size: medium
    description:
      "core: add plus-one entries and a dedup key to the Rust notification row, an
      append-plus-one operation, and an atomic create-or-plus-one/supersede upsert with
      parity tests."
  - id: cli
    title: sase notify +1 and create upsert
    depends_on:
      - core
    size: medium
    description:
      "cli: mirror the new wire in Python, add the notify +1 subcommand and create
      dedup-key/plus-one/supersede flags, and render +1 evidence in notify list/show and
      JSON."
  - id: panel
    title: Notification panel +1 badges and iteration
    depends_on:
      - cli
    size: medium
    description:
      "panel: render +1 badges and an evidence section in the ACE notification modal,
      add a + key that cycles the detail pane through +1 notes, and extend the PNG
      snapshot suites."
  - id: chop
    title: ci_watch incident-combination notifications
    depends_on:
      - cli
    size: medium
    description:
      "chop: collapse per-repo ci_watch failure notifications into one
      incident-combination notification that +1s material deltas, supersedes itself when
      a new repo starts failing, and falls back to legacy behavior on older sase CLIs."
  - id: verify
    title: Integrated verification and config alignment
    depends_on:
      - panel
      - chop
    size: small
    description:
      "verify: run both repos' verification lanes plus the monitored check-full landing
      gate, smoke the upsert end to end, and align the chezmoi-managed chop description
      with the new subprocess surface."
proposed_by: bbugyi200.athena.05k
bead_id: sase-y6
create_time: 2026-09-09 19:52:18
---

- **PROMPT:**
  [prompts/202609/ci_watch_notification_plus_one.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/ci_watch_notification_plus_one.md)
- **BEAD:**
  [sase-y6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-y6/README.md)

# Plan: Notification +1 corroboration and ci_watch incident-combination dedup

## Problem and verified evidence

The `ci_watch` chop (in `gh:bbugyi200/bugyi-chops`, `src/bugyi_chops/ci_watch.py`) sends
one SASE notification per repository incident: `_send_required_notifications` loops
`state["failures"]` per repo, and the incident key is a per-repo, sha-independent
fingerprint over the failing `(workflow, job, conclusion, steps)` tuples. When several
tracked repos go red — or one repo's failing-job set churns — the user receives a
sequence of `🚨 CI failure:` notifications that all point at the same live `CI WATCH`
report.

The SASE notification store (`~/.sase/notifications/notifications.jsonl`, all access
through the Rust core in `crates/sase_core/src/notifications/` under an exclusive lock
with atomic merge-rewrites) has no update, dedup, or corroboration mechanism: rows are
immutable after `append_notification`, and the only mutations are the
`NotificationStateUpdateWire` verbs (read/dismiss/mute/snooze variants). The chop never
reads the store back; its dedup lives only in its private `ci_watch_state.json`.

Two facts make the design below safe and cheap:

1. Every delivery consumer — the Telegram `tg_outbound` chop, the mobile gateway
   snapshot (`src/sase/integrations/_mobile_notification_snapshot.py`), and the ACE
   toast poller — dedupes on the `(activity_at, id)` cursor, where `activity_at` is
   `resurfaced_at ?? timestamp`. A row mutation that leaves those fields alone is
   invisible to every delivery cursor: no re-send, no re-toast. A new row alerts
   normally. So "+1 quietly, notify on new" needs no consumer changes at all.
2. Task beads already have a mature +1 concept (`TaskPlusOneEvidenceWire`,
   `sase bead +1`, the shared badge vocabulary in
   `src/sase/bead/plus_one_presentation.py`: accent `#FF87D7`, `plus_one_badge()` `[+N]`
   chips, `+1 EVIDENCE` sections). The notification +1 borrows that visual and naming
   vocabulary wholesale so the two features read as one system.

## Design contract

### Notification +1 entries

A notification gains an append-only list of _plus-one_ entries plus an optional
sender-scoped dedup key:

- `plus_ones: [{timestamp, sender, note}]` — `timestamp` RFC-3339 with timezone, minted
  by the CLI at call time; `sender` non-blank; `note` non-blank, trimmed,
  whitespace-collapsed, capped at 2000 chars by the core (senders may bound tighter).
- `dedup_key: str | None` — an opaque, exact-match string scoped to `sender`.
- The displayed count is always derived (`len(plus_ones)` plus a `plus_ones_dropped`
  overflow counter); there is no stored counter. The core caps stored entries at 500 per
  row: appending beyond the cap drops the oldest entry and increments
  `plus_ones_dropped`, so the total count stays truthful while the store stays bounded
  against a runaway sender.

Deliberate divergences from the bead +1 (state each in the docs):

- **No one-per-reporter rule.** A bead +1 is one reporter's corroboration; a
  notification +1 is a new _occurrence in time_, and the same sender is expected to +1
  repeatedly. The original creator counts like anyone else.
- **No status interaction.** A +1 never changes `read`, `dismissed`, `silent`, `muted`,
  `snooze_until`, `resurfaced_at`, or `timestamp`. It therefore never reorders the
  panel, never wakes a snoozed row, never resurrects a dismissed row, and never
  re-triggers Telegram/mobile/toast delivery. +1s on a dismissed or snoozed row are
  recorded silently — dismissing an incident means "stop showing me", and the evidence
  still accrues for later reading.
- **No artifact refs / observed-since provenance.** Notes may carry URLs; there is no
  reopen decision to feed.

### The create-or-plus-one upsert

`sase notify create` grows three optional inputs (JSON stdin keys, with matching flags
that override): `dedup_key` (`-k/--dedup-key`), `plus_one_note` (`-p/--plus-one-note`),
and `supersedes` (`-S/--supersedes`, an old dedup key). Semantics, executed atomically
inside the Rust store lock:

| Condition                                                           | Effect                                                                                                                                                                                                                                            |
| ------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| No `dedup_key`                                                      | Today's create, byte-for-byte unchanged (prints the new id).                                                                                                                                                                                      |
| `dedup_key` set, no row matches `(sender, dedup_key)`               | Create the row with `dedup_key` stored on it.                                                                                                                                                                                                     |
| `dedup_key` set, a row matches                                      | Append a +1 with `plus_one_note` (required in this branch; reject the call up front if `dedup_key` is set without `plus_one_note`) to the newest matching row instead of creating. The creation payload (notes, icon, tags, action) is discarded. |
| Created **and** `supersedes` set, rows match `(sender, supersedes)` | On each match: append a final +1 `superseded by: <new notes[0], bounded>` and set `dismissed=true`. No match is a silent no-op.                                                                                                                   |

Matching ignores read/dismissed/muted/snoozed state on purpose: a dismissed incident
notification keeps absorbing +1s quietly, which is exactly the "stop pinging me"
contract. When `--dedup-key` is provided the command prints a one-line JSON outcome
`{"action": "created"|"plus_oned", "id": "<uuid>"}` so scripted callers can record what
happened; the flagless output stays the bare id.

### The ci_watch incident-combination contract

The chop's notification unit changes from _repo incident_ to _incident combination_: the
set of repos it has announced as failing.

- One active incident at a time, persisted in chop state:
  `{dedup_key, announced: [repos...], notification_sent, notified_at, described: {repo: fingerprint}}`.
  `dedup_key` is `ci-failure/<sorted announced repos joined by ",">` — human-readable
  and stable.
- Let `F` be the repos RED this tick (existing per-repo classification unchanged).
  - `F = ∅`, incident active → +1 the notification by key with a resolution note
    (`all green: <repos>`), then clear the incident. Recurrence later starts a fresh
    incident and a fresh notification, preserving today's "recurrence is announced
    again" contract.
  - `F ⊆ announced` → compute material deltas against `described`: a repo whose
    fingerprint changed, a repo that recovered, a repo that re-failed. Any delta → one
    `sase notify +1` by key with one concise, bounded note naming only the changed parts
    (e.g.
    `sase-org/sase-core recovered; sase-org/sase: evidence changed (Master Gate › test (3))`);
    update `described` only on send success so failures retry next tick. No delta →
    silence (the live report already shows currency).
  - `F ⊄ announced` (a repo no existing notification covers went red) → roll the
    incident: `announced := F`, new `dedup_key`, and send
    `sase notify create -k <new-key> -p "<still-failing note>" -S <old-key>` with the
    full aggregate payload. The store creates the new (unread → alerting) notification
    and retires the old one with a superseding +1 + dismissal. This is the **only** path
    that produces a new notification while any CI-failure notification is live — exactly
    the "notify me only when a repo that wasn't failing before starts failing" rule,
    which supersedes a literal one-per-combination reading (a shrinking combination must
    +1, not re-notify).
- Passing `-p` on every create makes the store the dedup backstop the user asked for: if
  chop state is lost while the same combination is still red, the create call finds the
  existing `(ci_watch, key)` row and +1s it instead of duplicating.

## Phase: core

In the `sase-core` linked repo (open it with `/sase_repo`; never via a sibling path),
extend `crates/sase_core/src/notifications/`:

1. `wire.rs`: add `NotificationPlusOneWire { timestamp, sender, note }` and, on
   `NotificationWire`, `plus_ones: Vec<NotificationPlusOneWire>`,
   `plus_ones_dropped: u32`, and `dedup_key: Option<String>` — all with
   `#[serde(default)]` and `skip_serializing_if` on the empty/zero/None values so rows
   without +1s serialize byte-identically to today and old readers keep parsing new rows
   (verify `NotificationWire` does not deny unknown fields). Keep schema version 1: the
   change is purely additive.
2. `store.rs`: two new operations, both under the existing exclusive lock +
   `merge_and_rewrite_notifications_unlocked` machinery:
   - _append plus-one_: target by exact id **or** by `(sender, dedup_key)` (newest
     matching row wins); validate/trim/cap the note; enforce the 500-entry cap with
     `plus_ones_dropped`; return a typed outcome distinguishing applied / no-match /
     invalid.
   - _create-or-plus-one_: implement the upsert table above, including the `supersedes`
     retirement (final +1 note + `dismissed=true` on old rows). The caller supplies the
     fully-minted creation row (id/timestamp minted by Python, preserving the current
     create contract); the core decides create vs +1 atomically and returns which
     happened plus the affected id.
3. `crates/sase_core_py/src/lib.rs`: PyO3 bindings for both operations following the
   existing `apply_notification_state_update` / `append_notification` wrapper patterns.
4. Tests: extend `crates/sase_core/tests/notification_store_parity.rs` (and fixtures) —
   round-trip of rows with/without +1s, legacy rows default cleanly, upsert create vs +1
   vs supersede branches, dismissed/snoozed rows still match, cap-and-dropped-counter
   behavior, concurrent append-vs-upsert merge safety, and that no +1 path ever touches
   `timestamp`/`resurfaced_at`/state flags. Run this repo's own fmt/lint/test lanes.

## Phase: cli

In the sase repo:

1. Mirror the wire: `plus_ones`, `plus_ones_dropped`, `dedup_key` on the `Notification`
   dataclass (`src/sase/notifications/models.py`) with defaults, on the Python wire
   mirrors (`src/sase/core/notification_store_wire.py`), and thread the two new bindings
   through `src/sase/core/notification_store_facade.py` and
   `src/sase/notifications/store.py` wrappers. Hydration must tolerate unknown keys both
   directions.
2. Extract the sender-agnostic +1 vocabulary (accent colors, `plus_one_badge`, labels)
   from `src/sase/bead/plus_one_presentation.py` into a shared module both bead and
   notification code import; re-export from the bead module so existing callers and
   tests stay untouched (mind symvision when moving symbols).
3. `sase notify +1 <id-or-prefix> <note>` (parser in `src/sase/main/parser_commands.py`
   / handler alongside `src/sase/main/notify_handler.py`): positional id accepts a
   unique prefix (reuse the prefix-resolution precedent from
   `src/sase/notifications/pending_actions.py`); `-k/--dedup-key` addresses by
   `(sender, key)` instead of id; sender defaults to the resolved current agent/user
   identity with the group `-s/--sender` override. By-id no-match is an error; by-key
   no-match prints `{"action": "no_match"}` and exits 0 (scripted callers treat it as a
   quiet miss). Follow `sase/memory/cli_rules.md`: required values positional, every
   long option short-aliased, help sorted and excellent.
4. `sase notify create`: accept `dedup_key`/`plus_one_note`/`supersedes` from the stdin
   JSON and the `-k`/`-p`/`-S` flags; route through the upsert binding when a dedup key
   is present; print the JSON outcome line in that mode and the bare id otherwise.
   Reject `-k` without a plus-one note.
5. Rendering: `sase notify show` gains a `+1 EVIDENCE` markdown section (entry header
   `+1 <sender> · <timestamp>`, wrapped note) plus `dedup_key`; JSON output gains
   `plus_ones`, `plus_one_count` (derived, includes dropped), and `dedup_key`.
   `sase notify list` shows the `[+N]` badge in the pretty table and folds +1 notes into
   the `-q` query haystack.
6. Tests modeled on `tests/test_bead/test_cli_plus_one.py` and
   `test_plus_one_presentation.py`: parser shape, prefix resolution, by-key addressing
   and no-match outcome, upsert branches through a real temp store, supersede
   retirement, output contracts (bare id vs JSON outcome), badge/section agreement
   across list/show/JSON, and cursor-invariance (a +1'd row is not re-delivered by
   `read_mobile_notification_snapshot` and does not change
   `notification_activity_cursor`).

## Phase: panel

In the sase repo's ACE TUI (respect `sase/memory/tui_perf.md`: everything below renders
from already-loaded rows; no new I/O or FFI on key paths):

1. Row badge: in `_create_styled_label`
   (`src/sase/ace/tui/modals/notification_modal_options.py`), append the shared accent
   `+N` chip right after the truncated title when `plus_ones` is non-empty, visually
   consistent with bead `[+N]` badges and the existing dim tag-overflow chip.
2. Detail pane, static view: the summary/gate pane (`notification_modal_gate.py`) gains
   a `+1 EVIDENCE` group under the notes group listing every entry
   (`+1 <sender> · <relative time> — <note>`); the report-pane provenance line
   (`notification_modal_report.py`) gains `· +N (latest <age>)` when +1s exist, so
   ci_watch rows surface the count without leaving the live report.
3. Iteration: a new `+` binding on the notification modal cycles the detail pane through
   the selected row's +1 entries, newest first — header
   `+1 i/N · <sender> · <absolute time>` in the accent style, the wrapped note below, a
   dim hint line, and wrap-around back to the default pane after the oldest entry. No
   +1s → a brief status hint. Selection or tab change resets to the default pane. Slot
   the mode into the `_display_file` dispatch (`notification_modal_attachments.py`)
   ahead of the normal panes; update the footer hints
   (`notification_modal_constants.py`) and the modal `BINDINGS` (they are hardcoded in
   the modal today — check whether `src/sase/default_config.yml` needs a keymap entry
   per the gotchas memory and follow whichever convention the modal actually uses).
4. Tests: modal unit tests for badge, evidence group, and the full iteration cycle/reset
   behavior; PNG snapshots extending
   `tests/ace/tui/visual/test_ace_png_snapshots_notification_report.py` and the list-row
   fixtures (a row with `[+3]`, the +1 pane view); run `just test-visual` and update
   goldens intentionally.

## Phase: chop

In `gh:bbugyi200/bugyi-chops` (open with `/sase_repo`; do not locate, clone, or fetch it
another way), rework `src/bugyi_chops/ci_watch.py` failure notifications per the
incident-combination contract:

1. State v2: add the `incident` sibling to `failures`/`releases` in the state schema
   with its sanitizer in `_load_state`; keep per-repo `failures` fingerprints as the
   delta source. Migration: when loading state that has `failures` rows with
   `notification_sent=true` and no `incident`, synthesize an incident from those repos
   with `notification_sent=true` so the upgrade never re-announces a live incident.
2. Implement the tick logic from the design contract at the existing seam (after the
   per-repo loop, inside `_update_failure_state` / `_send_required_notifications`):
   resolution +1, delta +1 (one bounded note per tick naming only changes; `described`
   updated only on success so failed sends retry), and the roll path
   (`create -k … -p … -S …`) as the only new-notification path. Aggregate payload:
   `notes[0]` compact — `CI failure: <repos, common owner stripped when shared>` with a
   `(<n> repos)` suffix past two — then per-repo job evidence reusing
   `_failure_notification_notes` bounding, plus the existing tags/icon/ViewReport
   action. Release notifications are untouched.
3. Capability fallback: the new CLI surface may not exist on the installed sase. On an
   unrecognized-argument failure from `notify create -k` or `notify +1` (argparse exit +
   stderr signature), fall back to the legacy per-repo notification flow for that tick
   and record `plus_one_supported: false` in the decision ledger; never let the
   capability probe count as a send failure. Parse the JSON outcome line when `-k` is
   used and record created/plus_oned/superseded/resolved decisions in the ledger.
4. Update the chop's in-repo description (the subprocess surface now includes
   `sase notify +1`) and the subprocess-boundary test accordingly.
5. Tests, following the existing multi-tick pattern
   (`test_failure_incident_dedupes_changes_resets_and_retries`): two repos failing → one
   notification; fingerprint churn → +1 not create; recovery → +1; re-failure within
   announced → +1; new repo → create with supersede; all green → resolution +1 + cleared
   incident; recurrence → fresh notification; state-loss → plus_oned outcome adopted;
   send failure → retry; v1-state migration; the capability fallback; and the exact
   argv/stdin wire contract via `QueueRunner`. Run this repo's `just check`.

## Phase: verify

1. In sase: `just install`, then `just check`; run `just check-full` only through
   `/sase_monitor` with the `TESTING`/`TESTED` status pair and inspect the result (this
   is an epic's combined tree). Re-run the focused notification store, CLI, modal,
   mobile-bridge, and PNG suites explicitly.
2. End-to-end smoke against a temporary SASE home: `sase notify create -k … -p …` twice
   (one row, one +1), a superseding create, `sase notify +1` by prefix and by key, and
   confirm `list`/`show`/JSON agree on counts and that a +1'd row's activity cursor is
   unchanged.
3. In bugyi-chops: re-run `just check` plus the focused ci_watch multi-tick tests.
4. Open the `chezmoi` linked repo with `/sase_repo` and update the ci_watch chop entry
   in the SASE overlay config: the description's subprocess-surface sentence must now
   include `sase notify +1`, and the incident-combination behavior prose should replace
   the per-repository sentence. If the overlay cannot be opened or resolved, record the
   exact needed edit in this phase's bead note for the user instead of guessing.
5. Version-skew note for the land summary: the sase release must land before an upgraded
   bugyi-chops starts using the new flags on this host; until then the chop's capability
   fallback keeps legacy behavior. No pin or release is performed by this epic.

## Non-goals and boundaries

- No feature flag: `create` without `--dedup-key` is byte-identical to today, +1
  surfaces render only when entries exist, and the chop change ships complete with its
  own fallback — nothing user-reaching lands half-finished.
- No re-alerting +1 variant (`resurfaced_at` bumping), no +1 wake targets for snoozed
  notifications, no TUI action to file a +1 by hand, no notification deletion, and no
  artifact refs on +1 entries.
- No changes to `tg_outbound`, the mobile gateway surface, or toast polling — their
  cursor semantics are what make +1s quiet; tests only pin that invariant. Mobile
  rendering of +1 entries is future work.
- Release notifications and the merge/release halves of ci_watch are untouched; the
  live-report mechanism is untouched.
- Do not modify SASE memory files; the chezmoi overlay edit in `verify` is config, not
  memory.

## Acceptance criteria

- Two repos failing concurrently produce exactly one unread CI-failure notification;
  evidence churn, recoveries, and re-failures within the announced set produce quiet +1
  notes visible in the panel, `notify show`, and JSON.
- A repo outside every existing CI-failure notification going red produces exactly one
  new unread notification, and the prior one is dismissed with a superseding +1
  breadcrumb.
- A +1 never changes a notification's read/dismissed/muted/snoozed state, order, or
  delivery cursor; Telegram/mobile/toast deliver nothing for it.
- The `+` key cycles through +1 notes in the notification panel with the accent styling
  shared with bead badges; PNG snapshots cover the badge and the pane.
- `sase notify -h` surfaces read excellently per `cli_rules.md`; all sase lanes
  (`just check`, monitored `just check-full`, `just test-visual`) and the bugyi-chops
  `just check` are green.
