---
tier: epic
title: Client-side notification delivery rules
goal: "A person receiving SASE notifications can match them by tab, sender, action, tag,
  title, or note text, and decide per match whether a TUI toast is shown and whether the
  announcement is the terminal bell, a custom sound file, or silence. Task-bead
  notifications announce nothing on every machine, and kellys_mbp announces with a sound
  file instead of the bell.

  "
phases:
  - id: core-rules
    title: Rule matcher in the Rust core
    depends_on: []
    size: medium
    description:
      "core-rules: add notification delivery rule wire types, the first-match-per-field
      resolver, a case-insensitive glob matcher, and the batched
      resolve_notification_deliveries PyO3 binding to sase-core, reusing tab_key_for for
      the tab criterion."
  - id: config-rules
    title: Config surface and Python facade
    depends_on:
      - core-rules
    size: medium
    description:
      "config-rules: ratchet the pinned core revision, add ace.notification_rules to
      default_config.yml and the JSON schema, and add the token-cached Python facade
      that reads the rules and resolves deliveries through the new binding."
  - id: sound-backend
    title: Sound file playback
    depends_on: []
    size: small
    description:
      "sound-backend: add a presentation-side sound module that resolves a platform
      audio player (afplay on macOS; paplay, aplay, then ffplay on Linux) and plays a
      file without ever raising into the event loop."
  - id: tui-delivery
    title: Apply rules in the notification poll
    depends_on:
      - core-rules
      - config-rules
      - sound-backend
    size: medium
    description:
      "tui-delivery: resolve each arriving notification's delivery on the existing
      worker hop, filter toast-suppressed rows out before batching, and play at most one
      resolved sound per poll tick in place of the unconditional tmux bell."
  - id: observability
    title: sase notify rules, doctor check, and docs
    depends_on:
      - config-rules
      - sound-backend
      - tui-delivery
    size: medium
    description:
      "observability: add the sase notify rules subcommand with per-notification
      explanation, a config.notification_rules doctor check, and the notifications docs
      section describing matching, resolution, and playback."
  - id: chezmoi-config
    title: The two requested configurations
    depends_on:
      - config-rules
      - observability
    size: small
    description:
      "chezmoi-config: add the global task-bead suppression rule to sase.yml and the
      sound-file rule to sase_kellys_mbp.yml in the chezmoi repo, after confirming the
      shipping build understands the key."
proposed_by: bbugyi200.athena.0o7
create_time: 2026-09-20 13:11:26
status: wip
---

- **PROMPT:**
  [prompts/202609/notification_delivery_rules.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/notification_delivery_rules.md)

# Client-Side Notification Delivery Rules

## Goal

Let the person receiving notifications decide, per notification, whether a TUI toast is
shown and what sound announces it (terminal bell, a custom sound file, or silence).
Today both behaviors are hardcoded: every newly-unread notification toasts, and any
non-empty batch rings the tmux bell three times.

Two configurations are the first consumers, and both land in the user's chezmoi repo:

1. On `kellys_mbp` (the only machine where the TUI runs locally), a custom sound file
   replaces the terminal bell for all notifications.
2. On every machine, task-bead notifications announce nothing — no toast, no sound.

## Background

### Where the current behavior lives

`AgentNotificationPollingMixin._poll_agent_completions_once`
(`src/sase/ace/tui/actions/agents/_notification_polling.py`) is the single place new
notifications are announced. It computes `new_notifications` (newly-unread rows that
have not yet been delivered against the durable activity cursor), then:

- toasts every one of them via `format_batch_toasts` + `self.notify(...)`, and
- sets `should_ring_bell = bool(new_notifications)` and calls `_ring_tmux_bell_async()`,
  which shells out to the vendored `tmux_ring_bell` script with `3` bells at `0.1s`.

Nothing else in the repo rings the bell or toasts from the notification store, so this
is the only application point the epic needs.

`format_batch_toasts` collapses batches of 4+ into one grouped toast per severity
bucket, so suppression must happen _before_ batching or a suppressed row would still be
counted in a grouped toast.

### What a notification actually carries

Field survey of the live store on athena (200 most recent rows), which is what the
matching vocabulary should be built from:

| sender                 | action                | tags                                     | panel       |
| ---------------------- | --------------------- | ---------------------------------------- | ----------- |
| `wait_checks`          | —                     | `wait`, `blocked`, `terminal-dependency` | —           |
| `axe`                  | `ViewErrorReport`     | —                                        | —           |
| `bead`                 | `TaskTriage`          | `bead`, `task`, `<task_type>`            | `beads`     |
| `bead`                 | `BeadStaleCleanup`    | `bead`, `task`, `stale`                  | `beads`     |
| `poseidon-cache-watch` | —                     | `poseidon`, `storage`                    | —           |
| `epic-launch`          | —                     | `epic`, `launch`                         | —           |
| `remote-attention`     | `RemoteAttention`     | `attention`, `apollo`, `gate`            | `attention` |
| `gate`                 | `GateExecutionFailed` | `gate`, `execution`, `error`             | —           |
| `user-agent`           | `JumpToAgent`         | `done`                                   | —           |

`Notification.notes[0]` is the headline the notification panel renders as a row's title
(`notification_modal_options.py:108`) and the text most toast branches fall back to.

### Tab ownership already exists in the Rust core

`sase_core::notifications::tabs::tab_key_for` assigns every notification to exactly one
panel tab, with a declared precedence: muted/snoozed synthetic tabs, then a
gate-declared `panel`, then the synthetic `hitl` tab for gate actions, then `errors`,
then the first display tag, then `general`. `ace.notification_tabs` in
`default_config.yml` already keys per-tab styling off exactly these keys. Reusing
`tab_key_for` as a match criterion means "the tab I see this row in" and "the tab I
match on" cannot drift.

### Why this is core backend logic

Per the `rust_core_backend_boundary` core memory: a rule engine that answers "how should
this notification be announced" is behavior a web app, the Telegram outbound job, or
mobile would need to match the TUI on. It also needs `tab_key_for`, which is already in
the core. So the matcher lives in `../sase-core/crates/sase_core`, behind a
`sase_core_rs` binding; only playback and toast emission stay in Python.

Playing a sound file and painting a Textual toast are presentation, and stay in this
repo.

### Config layering makes rule order work out

`src/sase/config/layers.py` gives the `user` layer (`~/.config/sase/sase.yml`)
`list_strategy="replace"` and every machine `overlay` layer
(`~/.config/sase/sase_<machine>.yml`) `list_strategy="concatenate"`. So a list at
`ace.notification_rules` merges as: `sase.yml`'s rules first, then the machine file's
rules appended after. With first-match-wins evaluation that is exactly the precedence
the two target use cases want — the global bead-suppression rule is consulted before the
machine's catch-all sound rule, with no priority tuning.

### No feature flag

`sase_flags.md`: "Do not flag anything users are meant to choose forever; that is a
config field." This is a permanent config field whose absence reproduces today's
behavior byte for byte, so no flag bead is created.

## Design

### Config shape: `ace.notification_rules`

An ordered list, sibling to the existing `ace.notification_tabs` and
`ace.notification_indicator_max_counts`:

```yaml
ace:
  notification_rules:
    - name: quiet-task-beads
      description: Task-bead triage is too noisy to announce right now.
      match:
        tab: beads
      toast: false
      sound: none
    - name: mac-chime
      sound: /System/Library/Sounds/Glass.aiff
```

Each entry:

| Key           | Type              | Meaning                                                       |
| ------------- | ----------------- | ------------------------------------------------------------- |
| `name`        | string, optional  | Label for `sase notify rules`, doctor output, and diagnostics |
| `description` | string, optional  | Free prose explaining why the rule exists                     |
| `priority`    | integer, optional | Evaluation weight; higher is consulted earlier. Default `0`   |
| `match`       | object, optional  | Criteria. Omitted or `{}` matches every notification          |
| `toast`       | boolean, optional | Whether a TUI toast is shown                                  |
| `sound`       | string, optional  | `bell`, `none`, or a path to a sound file                     |

### Matching

Six criteria, each drawn from something the person can see on the notification row:

| Criterion | Matched against                                                                                                                        |
| --------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `tab`     | The owning panel tab key from `tab_key_for` (`beads`, `hitl`, `errors`, `general`, `attention`, a tag tab, `__muted__`, `__snoozed__`) |
| `sender`  | `Notification.sender`                                                                                                                  |
| `action`  | `Notification.action` (empty string when the row has no action)                                                                        |
| `tags`    | Any one of `Notification.tags`                                                                                                         |
| `title`   | `Notification.notes[0]`, the row headline                                                                                              |
| `note`    | Any line in `Notification.notes`                                                                                                       |

Semantics, chosen for predictability over cleverness:

- Every value is a **case-insensitive glob** (`*`, `?`, `[...]`), the same idiom
  `artifact_refs.file.roots[].path_globs` already uses in this config file. A pattern
  with no wildcard is therefore an exact, case-insensitive match. There is no hidden
  substring or fuzzy matching: `title: "Plan ready"` does not match
  `"Plan ready for review"`; `title: "Plan ready*"` does.
- Each criterion accepts a single string or a list of strings. A list is **any-of**.
- Multiple criteria in one `match` are **all-of**.
- `tab` additionally accepts `gates` as an alias for the core's `hitl` key, because the
  notification panel labels that tab "Gates". Normalization is one-way (`gates` →
  `hitl`) and case-insensitive.

### Resolution: first rule that sets a field wins that field

Rules are evaluated in descending `priority`, ties broken by position in the merged
list. For each of `toast` and `sound` independently, the **first matching rule that sets
that field** decides it. A field no matching rule sets falls back to the built-in
default: `toast: true`, `sound: bell` — which is today's behavior exactly.

This is `ssh_config`'s "first obtained value wins", and it has two properties worth
naming:

- **Negation is unnecessary.** "Silence everything except gates" is a
  `match: {tab: hitl}` rule with the defaults spelled out, followed by a catch-all
  suppression rule. No `not:` key, no set algebra.
- **A broad rule can sit under narrow ones.** A machine-wide `sound:` catch-all does not
  have to re-state, or defend itself against, the specific rules above it.

`priority` exists so a machine overlay can deliberately override a global rule despite
landing later in the concatenated list. Neither of the two target configurations needs
it.

### Behaviors

`toast: false` suppresses the Textual toast only. The notification is still stored,
still unread, still counted in the top-bar indicator, and still present in the
notification panel. This epic changes **announcement**, never notification state or the
inbox.

`sound` is a single scalar with two reserved words:

- `bell` — the existing tmux terminal bell (default).
- `none` — silence.
- anything else — a path to a sound file, `~` and `$VAR` expanded. A literal file named
  `bell` or `none` is addressed as `./bell`.

Playback resolves a player by platform, first available wins, and every failure is
swallowed the way the current bell swallows `FileNotFoundError`:

- macOS: `afplay <file>`
- Linux: `paplay`, then `aplay`, then `ffplay -nodisp -autoexit -loglevel quiet`

A missing file, a missing player, or a non-zero exit never interrupts the poll tick and
never raises into the event loop. Playback runs on a worker thread, exactly like
`_ring_tmux_bell_async` does today, so the tmux subprocess can never block Textual.

### One announcement per poll tick

The poll loop delivers a batch. Per tick:

- Each new notification resolves its own delivery. Rows with `toast: false` are removed
  from the list handed to `format_batch_toasts`, so they are not counted in a grouped
  toast either. If every row is suppressed, no toast is emitted.
- **At most one sound plays**, chosen as the resolved sound of the first notification in
  the batch (activity order) whose sound is not `none`. If every arriving row resolves
  to `none`, the tick is silent. This preserves the current "one bell per tick, not one
  per row" behavior and prevents a five-notification burst from becoming five chimes.

The two axes are independent: a row can be silent but toast, or sound without toasting.

### Observability

Rules that silently fail to match are the main way a feature like this becomes
frustrating, so the epic ships a way to see the answer:

- `sase notify rules` — prints the merged rules in evaluation order with their source
  config layer, their criteria, and the behaviors they set.
- `sase notify rules -e/--explain <notification-id>` — for one stored notification,
  prints its extracted match fields (tab, sender, action, tags, title) and, for each of
  `toast` and `sound`, which rule decided it or that it fell back to the default.
- A `sase doctor` check (`config.notification_rules`) that flags unknown criterion keys,
  malformed globs, `sound` paths that do not exist, and a configured sound file with no
  available player on this platform.

## Non-Goals

- Changing whether a notification is **created**, stored, read, muted, or snoozed.
  Senders already own `silent`; the person already owns mute and snooze. This is a
  third, purely client-side axis.
- Routing rules for Telegram, mobile, or any non-TUI surface. The matcher lands in the
  Rust core precisely so those surfaces _can_ adopt it later, but no surface other than
  the TUI is wired in this epic.
- Volume, per-rule repeat counts, speech synthesis, or desktop (libnotify /
  `terminal-notifier`) notifications.
- Matching on attached `files`, `action_data`, `dedup_key`, or the derived
  priority/error class. `tab: errors` already covers the error class. These are
  deliberately left out to keep every criterion something visible on the row; add one
  later if a real need appears.

## Rollout Hazard (read before `chezmoi-config`)

`ace` is `"additionalProperties": false` in `src/sase/config/sase.schema.json`. A
machine still running a sase build without `ace.notification_rules` will report the key
as unknown in `sase config validate` and `sase doctor`. `chezmoi-config` writes the
chezmoi configs and MUST NOT be applied to the user's machines until the sase release
carrying `core-rules` through `observability` is installed there. That phase states this
in its own handoff.

---

## Rule matcher in the Rust core

**Phase id:** `core-rules`

**repo:** `sase-core` (open with `/sase_repo`; do not guess the path)

Add notification delivery rules to `crates/sase_core/src/notifications/`.

1. New module `rules.rs`:
   - Wire types, serde-derived, in `wire.rs` or `rules.rs` alongside the existing
     notification wire types and carrying `NOTIFICATION_STORE_WIRE_SCHEMA_VERSION`:
     - `NotificationRuleWire { name, description, priority (default 0), match, toast, sound }`
       where `toast: Option<bool>` and `sound: Option<String>`.
     - `NotificationRuleMatchWire { tab, sender, action, tags, title, note }`, each
       `Option<Vec<String>>`, each deserializing from either a bare string or a list.
     - `NotificationSoundWire` — an enum serialized as `{"kind": "bell"}`,
       `{"kind": "none"}`, or `{"kind": "file", "path": "..."}`.
     - `NotificationDeliveryWire { toast: bool, sound: NotificationSoundWire, toast_rule: Option<String>, sound_rule: Option<String> }`,
       where the two `*_rule` fields name the deciding rule (its `name`, or a positional
       `rule[<index>]` fallback) so the Python `--explain` surface needs no second
       evaluation pass.
   - `resolve_notification_delivery(rules: &[NotificationRuleWire], row: &NotificationWire) -> NotificationDeliveryWire`
     implementing: stable sort by descending `priority` with original index as
     tie-break, then first-match-per-field.
   - `resolve_notification_deliveries(rules, rows) -> Vec<NotificationDeliveryWire>` so
     a whole poll batch costs one FFI hop, matching how `classify_notification_tabs`
     already batches.
   - Criterion matching: case-insensitive glob. Implement the glob matcher in the crate
     (`*`, `?`, `[...]`, `[!...]`) rather than taking a dependency, and unit-test its
     edge cases directly (empty pattern, bare `*`, unclosed `[`, literal `*` via `[*]`,
     non-ASCII).
   - `tab` values normalize `gates` → `hitl` (case-insensitively) before comparison; the
     row's tab comes from `tab_key_for`.
   - `action` matches against `""` when the row's action is `None`, so `action: ""`
     selects action-less rows.

2. Export from `notifications/mod.rs` next to the existing `classify_notification_tabs`
   export.

3. PyO3 binding in `crates/sase_core_py/src/lib.rs`: `resolve_notification_deliveries`,
   following the `py_classify_notification_tabs` pattern exactly — `py.allow_threads`
   around the core call, registered in the module init, and documented in the
   module-level binding list at the top of that file.

4. Tests in `crates/sase_core/src/notifications/rules.rs` and a binding test in the
   `sase_core_py` suite. Cover at minimum:
   - No rules → `toast: true`, `sound: bell` for every row shape in the field survey.
   - First-match-per-field: a narrow rule setting only `toast`, a later catch-all
     setting only `sound`, resolving to one field from each, with both `*_rule` names
     reported.
   - `priority` reordering a later rule ahead of an earlier one.
   - The `tab: beads` rule suppressing a `TaskTriage` row and not touching an `axe`
     `ViewErrorReport` row.
   - `gates` / `GATES` / `hitl` all selecting the same rows.
   - All-of across criteria, any-of within `tags`.
   - Unknown criterion keys rejected by serde rather than silently ignored.

**Verification:** `just check` from the sase-core repo root (never
`cargo test -p sase_core` alone — it skips the binding tests; see that repo's
`AGENTS.md`). Commit on a branch in sase-core and note the resulting SHA in the phase
bead; `config-rules` needs it.

---

## Config surface and Python facade

**Phase id:** `config-rules`

1. Ratchet `sase-core-revision.txt` to the sase-core SHA from `core-rules` (see
   `tools/ratchet_core_revision`, and `tools/check_sase_core_rs_bindings` for the
   published-floor gate — add `resolve_notification_deliveries` to `REQUIRED_BINDINGS`
   there if the static scan cannot see the new call site yet). Run `just install` so the
   workspace venv rebuilds the core from the new pin.

2. `src/sase/default_config.yml`: add `ace.notification_rules: []` directly after
   `notification_indicator_max_counts`, with a comment block in the same voice as the
   `notification_tabs` block above it — what the list is, that evaluation is first-match
   -per-field in priority order, and what `sound: bell | none | <path>` means.

3. `src/sase/config/sase.schema.json`: add `ace.notification_rules` as an array of
   objects with `"additionalProperties": false`, mirroring the `notification_tabs` block
   for description style and constraint tightness. `match` is an object with
   `"additionalProperties": false` over the six criteria, each
   `oneOf [string, array-of-string]`. `sound` is a string. `priority` is an integer with
   the same `-1000..1000` bounds `notification_tabs.priority` uses.

4. New `src/sase/notifications/delivery.py`:
   - `NotificationDelivery` / `NotificationSound` dataclasses mirroring the wire types.
   - `notification_delivery_rules()` — reads `ace.notification_rules` from
     `load_merged_config()`, tolerating stored junk the way
     `notification_tab_style._configured_tab_styles_for_token` does (skip malformed
     entries rather than raising), cached on `current_config_token()` with
     `lru_cache(maxsize=1)` so a render or poll pays one config read.
   - `resolve_notification_deliveries(notifications)` — one call through
     `require_rust_binding("resolve_notification_deliveries")` in
     `src/sase/core/notification_store_facade.py` (add the facade function beside
     `classify_notification_tabs`), returning `list[NotificationDelivery]`.
   - Wire conversion in `src/sase/core/notification_store_wire.py` beside the existing
     notification wire converters.

5. Python tests: config parsing (including malformed entries, a bare-string criterion, a
   missing `match`), the default-empty case, token-cache invalidation, and a parity test
   that the Python-side default (`toast=True`, `sound=bell`) matches what the core
   returns for an empty rule list.

**Verification:** `just check`.

---

## Sound file playback

**Phase id:** `sound-backend`

New `src/sase/notifications/sound.py`, presentation-side and deliberately independent of
the rule model so it can be built and tested before the matcher exists:

- `resolve_sound_player(path)` → the argv to run, or `None` when no player is available.
  Platform order: `darwin` → `afplay`; `linux` → `paplay`, `aplay`,
  `ffplay -nodisp -autoexit -loglevel quiet`. Discovery via `shutil.which`, cached.
- `play_sound_file(path)` → expands `~` and `$VAR`, returns `False` without raising when
  the file is missing, no player is available, the player is not on `PATH`, or the
  subprocess exits non-zero or times out.
  `subprocess.run(..., check=False, capture_output=True)` with a short timeout, matching
  `_ring_tmux_bell`'s posture.
- Tests with a fake `which` and a fake `subprocess.run` covering: each platform's
  preference order, no player available, missing file, non-zero exit, timeout, and a
  path containing spaces. No test may actually play audio or shell out.

**Verification:** `just check`.

---

## Apply rules in the notification poll

**Phase id:** `tui-delivery`

In `src/sase/ace/tui/actions/agents/_notification_polling.py`, inside
`_poll_agent_completions_once`:

1. After `new_notifications` is finalized (i.e. after the auto-dismiss filtering that
   already rewrites it), resolve deliveries for the batch in **one** call, on the
   existing `asyncio.to_thread` worker hop rather than the event loop — the config read
   and the FFI call must not land on the main thread. Fold it into
   `_prepare_notification_reconciliation` or add a sibling worker call; do not add a
   second blocking step to the tick.

2. Toasts: pass only the rows whose delivery has `toast=True` to `format_batch_toasts`.
   Emit nothing when the filtered list is empty. The 4+ grouping threshold applies to
   the filtered list, so suppressed rows never appear in a grouped count.

3. Sound: replace `should_ring_bell = bool(new_notifications)` with a resolved
   `NotificationSound` for the tick — the sound of the first row (activity order) whose
   sound is not `none`, or nothing. Keep this the last step in the tick, after the
   indicator and toast updates, for the reason the existing comment gives.

4. Rename/retarget `_ring_tmux_bell_async` into an `_announce_notification_sound_async`
   that dispatches `bell` to the existing `_ring_tmux_bell` sync leaf and `file` to
   `play_sound_file`. **Keep `_ring_tmux_bell` as a sync leaf method on the mixin**:
   `tests/_notification_toasts_helpers.py` and
   `tests/test_notification_toast_polling_concurrency.py` patch it by name, and the
   concurrency test asserts it runs off the event loop.

5. Tests, extending the existing `tests/test_notification_toast_polling*.py` and
   `tests/_notification_toasts_helpers.py` fakes:
   - No rules configured → current behavior unchanged (toast per row, bell once). This
     is the regression guard for every existing notification poll test.
   - `tab: beads` suppression: a `TaskTriage` row arriving alone produces no toast and
     no sound; an `axe` row in the same tick still toasts and still rings.
   - A file sound replaces the bell, `_ring_tmux_bell` is not called, and
     `play_sound_file` receives the configured path.
   - A 6-row batch with 3 suppressed groups only the surviving 3 (and stays under the
     grouping threshold, producing 3 individual toasts).
   - Mixed sounds in one tick play exactly one sound.
   - Toast-suppressed rows still update the unread indicator and the snapshot cache.

**Verification:** `just check`. This phase changes no rendered TUI layout, so PNG
goldens should be untouched; if `just check` reports otherwise, do not blanket-update
goldens — investigate.

---

## `sase notify rules`, doctor check, docs

**Phase id:** `observability`

1. `sase notify rules` subcommand. Read `cli_rules.md` first. Requirements from it that
   apply here: subcommands and options sorted alphabetically in help, every public long
   option gets a short alias, no required options, and colored output where color helps.
   Note that `notify` has no `list` child conflict to worry about — it already defaults
   to `list`, so `rules` must be an explicit subcommand and must not disturb that
   default (`src/sase/main/parser.py`, `_default_list_subcommands()`).
   - Bare `sase notify rules`: the merged rules in evaluation order, numbered, each with
     its name, originating config layer, criteria, and the behaviors it sets. An empty
     list prints the built-in defaults and says no rules are configured.
   - `-e/--explain <notification-id>`: the row's extracted match fields, then one line
     per behavior naming the deciding rule or the default. Uses the `*_rule` fields the
     core already returns.
   - `-j/--json` for machine-readable output, matching `sase notify list -j`.

2. `sase doctor` check `config.notification_rules`, added beside
   `check_config_notification_tabs` in `src/sase/doctor/checks_config.py` with its
   implementation in a new `checks_config_notification_rules.py` sibling. It flags:
   unknown criterion keys, a `match` that is not an object, an unparseable glob, a
   `sound` path that does not exist, a configured sound file with no available player on
   this platform (via `sound-backend`'s `resolve_sound_player`), and a rule that sets
   neither `toast` nor `sound` (a no-op rule). Remedy text names `sase notify rules`.

3. `docs/notifications.md`: a "Delivery Rules" section covering the config shape, the
   six criteria and the glob semantics, first-match-per-field resolution, `priority`,
   the one-sound-per-tick rule, the `bell | none | <path>` scalar and its reserved
   words, the platform player table, and an explicit statement that `toast: false`
   suppresses the announcement but not the notification, the indicator, or the panel.
   Include the "silence everything except gates" worked example, because it is the
   clearest demonstration of why first-match-wins removes the need for negation.

4. `src/sase/notifications/sound.py` and the new facade functions need real non-test
   consumers by the end of this phase or symvision will fail; see `symvision.md` if it
   does. Do not reach for a pragma before checking that the intended caller is actually
   wired.

**Verification:** `just check`. If the doctor check's output is snapshotted anywhere,
`just fix-tui-screenshots` may be needed — check before assuming.

---

## The two requested configurations

**Phase id:** `chezmoi-config`

**repo:** `chezmoi` (open with `/sase_repo`; do not guess the path)

**Precondition, stated in the handoff:** do not apply these files to the user's machines
until a sase build carrying `core-rules` through `observability` is installed there.
`ace` is `"additionalProperties": false`, so an older sase reports `notification_rules`
as an unknown key.

1. `home/dot_config/sase/sase.yml` — under the existing `ace:` block, after `tribes:`,
   add:

   ```yaml
   notification_rules:
     - name: quiet-task-beads
       description:
         Task-bead triage is too noisy to announce right now; the rows still land in the
         Beads tab and the unread indicator. Remove this rule once the triage-volume fix
         is in.
       match:
         tab: beads
       toast: false
       sound: none
   ```

   `tab: beads` is the right criterion rather than `sender: bead`: `beads` is the
   gate-declared panel every task-bead row routes to (`TaskTriage`, `BeadSnooze`,
   `BeadStaleCleanup` all carry it), it is the tab name visible in the TUI, and it will
   keep matching if a future task-bead notification arrives from a different sender.

2. `home/dot_config/sase/sase_kellys_mbp.yml` — add an `ace:` block with:

   ```yaml
   ace:
     notification_rules:
       - name: mbp-chime
         description:
           kellys_mbp is the only machine running the TUI locally, so it is the only one
           where a sound file beats the terminal bell.
         sound: /System/Library/Sounds/Glass.aiff
   ```

   Verified present on that machine over the tailnet: macOS 26.5, `/usr/bin/afplay`, and
   `/System/Library/Sounds/Glass.aiff` among the 14 stock sounds. Any of `Blow`,
   `Bottle`, `Hero`, `Ping`, `Pop`, `Purr`, `Sosumi`, `Submarine`, or `Tink` is a
   one-word swap if the user prefers a different chime.

   This rule intentionally sets only `sound` and has no `match`, so it applies to
   everything, while the global `quiet-task-beads` rule — earlier in the concatenated
   list, because the `user` layer precedes the machine `overlay` layer — still silences
   task beads on this machine too. Confirm that with `sase notify rules` after applying.

3. Verify from the mbp (or by pointing a local run at those layers): `sase notify rules`
   lists both rules with `quiet-task-beads` first, `sase doctor` reports
   `config.notification_rules` green, and
   `sase notify rules --explain <a TaskTriage id>` attributes both `toast` and `sound`
   to `quiet-task-beads`.

**Verification:** `sase config validate` and `sase doctor` clean on both layers. No
`just check` in the chezmoi repo; follow that repo's own conventions for committing.
