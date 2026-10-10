---
tier: tale
title: Suppress Telegram delivery with notification rules
goal:
  Athena Beads notifications remain available in SASE without generating Telegram
  messages.
size: medium
proposed_by: bbugyi200.athena.0zg
create_time: 2026-10-10 10:43:24
status: wip
---

# Suppress Telegram delivery with notification rules

Add a `telegram` boolean action to `ace.notification_rules`, then configure Athena to
suppress Telegram delivery for the same Beads notifications Bryan already silences in
the TUI. Implement this as one coordinated change across `sase-core`, `sase`,
`sase-telegram`, and `chezmoi`. The behavior is bounded enough for one coding agent; the
existing matcher, configuration layering, and transport pipeline remain the owners of
their current responsibilities.

## Findings and scope

- `sase/src/sase/notifications/delivery.py` accepts only `name`, `description`,
  `priority`, `match`, `toast`, and `sound`. Unknown keys invalidate the entire rule.
- The shared matcher and per-field resolution live in
  `sase-core/crates/sase_core/src/notifications/rules.rs`; Python delegates through
  `resolve_notification_deliveries`. Rule priority is descending, ties retain merged
  list order, and the first matching rule setting each field wins independently.
- `sase-telegram/src/sase_telegram/outbound.py:get_unsent_notifications` filters
  already-read, muted, and silent rows, with an exception for quiet decision receipts.
  It does not consult notification rules. Existing TUI suppression cannot suppress
  Telegram delivery.
- `chezmoi/home/dot_config/sase/sase.yml` already has `quiet-task-beads`, matching
  `tab: beads` with `toast: false` and `sound: none`. Its Athena overlay currently has
  no notification rules. Task triage declares `action_data.panel: beads`; core tab
  classification gives declared panels precedence over the generic Gates tab.
- Scope is outgoing notification announcements, including attachments and keyboards
  belonging to those announcements. Stored notifications, unread counts, gate requests,
  TUI access, direct Telegram command replies, and existing messages remain unchanged.
  Keep the existing matcher vocabulary and configuration location.

Before working, open each linked repo with `/sase_repo` (`sase repo open sase-core`,
`sase repo open sase-telegram`, and `sase repo open chezmoi`, each with a reason), use
the printed checkout paths, and read their `AGENTS.md` files. The paths below are
relative to the named repository, never fixed checkout locations. Read required
reference memory through `/sase_memory_read`, particularly `lint_and_test.md`.

## Delivery contract

1. A rule may set `telegram: false` to prevent Telegram notification delivery or
   `telegram: true` to allow it. Omission leaves the field for subsequent matching
   rules; the resolved default is `true`. `true` does not bypass read/mute/silent,
   first-run, cursor, or other transport eligibility checks.
2. Resolve Telegram independently of toast and sound using the existing priority, glob
   matching, AND/OR criteria, tab classification, and config layering. A rule setting
   only Telegram must be effective, and earlier rules setting both TUI fields must not
   stop evaluation before a later Telegram rule is considered.
3. Expose the resolved `telegram` boolean and `telegram_rule` provenance alongside the
   existing delivery fields. CLI diagnostics describe rule permission/suppression, not a
   guarantee that Telegram is enabled or that a message was sent.
4. Suppression is evaluated at each outbound poll. It does not mutate the notification
   or advance the delivery cursor. A later successfully sent eligible row advances the
   cursor normally. Removing a suppression rule may make older unread rows eligible if
   they are still ahead of the cursor; there is no suppression ledger or backfill.
5. This is a permanent user configuration choice, not a feature flag. Preserve the
   default behavior when no rules set Telegram. Do not add a parallel matcher in Python
   or infer Telegram suppression from `toast` or `sound`.

## Implementation

### Extend the Rust delivery contract

In `sase-core/crates/sase_core/src/notifications/rules.rs`:

- Add optional `telegram: Option<bool>` to `NotificationRuleWire`, and resolved
  `telegram` plus optional `telegram_rule` to `NotificationDeliveryWire`.
- Default resolved Telegram delivery to `true`. Keep serialization additive and legacy
  delivery payload deserialization sensible with a serde default of `true`; absent rule
  fields continue to mean unset. Do not bump the broad notification-store schema merely
  for these additive fields; verify compatibility in tests.
- Carry the field through `CompiledRule`, the no-effect-rule check, independent field
  resolution, provenance, and the early-exit condition. Preserve existing matching
  semantics and stable labels.
- Extend the inline core tests and `crates/sase_core_py/src/notifications/tests.rs`
  binding round trips. The existing binding in
  `crates/sase_core_py/src/notifications/mod.rs` already deserializes rules and
  serializes delivery results; change it only if the new contract requires it.

### Expose the action and diagnostics in SASE

- Update `src/sase/notifications/delivery.py`: allowed keys, strict boolean validation,
  sanitizer, delivery dataclass/default, conversion from the wire, and docstrings. Match
  optional-field handling to `toast`, including its runtime null-as-unset convention.
  Strings and integers are not booleans. Malformed rules still drop as a whole with a
  useful diagnostic, without breaking neighboring rules.
- Extend `src/sase/core/notification_store_wire.py`'s dataclass and rehydration. Keep
  new dataclass fields defaulted so existing construction sites still work. Old wire
  responses missing the new field can mean default `true`; actual Telegram rules require
  the updated core. Do not swallow incompatible-core errors and silently deliver
  notifications contrary to configuration.
- Update `src/sase/config/sase.schema.json` and the explanatory comment above
  `ace.notification_rules` in `src/sase/default_config.yml`. The shipped rule list
  remains empty.
- Extend `src/sase/notifications/cli_rules.py` in all surfaces: text and JSON rule
  listing, defaults, and `sase notify rules --explain`. Show the deciding rule and
  config-layer provenance for Telegram just as for the other fields.
- Update `src/sase/doctor/checks_config_notification_rules.py` so a Telegram-only rule
  is not reported as ineffective. Update malformed-value and no-effect diagnostics.
- Update `docs/notifications.md` and the `notification_rules` entry in
  `docs/configuration.md` with the boolean, defaults, independent resolution, an
  example, transport-eligibility caveat, and suppression/cursor behavior. No new CLI
  options or TUI rendering changes are needed.

### Enforce the result in Telegram

In `sase-telegram/src/sase_telegram/outbound.py`, resolve the already-filtered, sorted
candidate batch through `sase.notifications.delivery.resolve_notification_deliveries`
once and return only rows whose resolved `telegram` is true. Preserve order and the
first-run empty result. The shared adapter supplies config caching and the Rust matcher;
transport code only consumes its result.

This filtering must precede the rate limiter, formatting, PDF conversion, network sends,
and pending-action creation in `scripts/sase_tg_outbound.py`. Suppressed rows must not
reach those operations. Leave `mark_sent` and the send-failure break behavior unchanged.
Apply the rule to quiet decision receipts too, after their existing eligibility
exception. An evaluation error must not fall back to sending the batch; surface the
failure and retain its cursor for retry.

Update `sase-telegram/docs/outbound.md` and its README to document the shared rule and
its effect on the pipeline. Keep direct inbound command responses and cleanup of
already-sent keyboards outside this filtering.

### Configure Athena

Add this machine-local rule to `chezmoi/home/dot_config/sase/sase_athena.yml`, merging
into an existing `ace` mapping if one has appeared:

```yaml
ace:
  notification_rules:
    - name: quiet-task-beads-telegram
      description: Keep Beads notifications in SASE without sending them to Telegram.
      match:
        tab: beads
      telegram: false
```

Retain the global `quiet-task-beads` rule in `sase.yml` unchanged. The overlay has its
own name and sets only Telegram; normal priority suffices because the global rule does
not set that field. Matching the same tab intentionally covers everything already
silenced there, including other bead gate variants, without suppressing ordinary plan
approvals, questions, errors, or completion announcements.

## Verification

Use local checkout environments with the edited Rust extension and SASE adapter,
including the Telegram repo's source override mechanism when needed. Read its `Justfile`
before setup: `SASE_TELEGRAM_SASE_SOURCE_DIR` and `SASE_TELEGRAM_SASE_CORE_SOURCE_DIR`
select the coordinated sources. Run setup/build commands through the prescribed
workflows; do not verify against an unrelated installed wheel. Isolate notification
files, config/cache tokens, cursors, pending actions, and Telegram APIs in tests.

- Rust and binding tests: default true; false/true rules; Telegram-only rules;
  independent fields; the global TUI rule followed by an Athena Telegram rule; priority
  overrides and equal-priority order; unmatched rows; rule provenance; invalid booleans;
  old rules and additive wire deserialization.
- SASE tests: extend `tests/test_notification_delivery.py`,
  `tests/test_notification_delivery_layers.py`, `tests/main/test_notify_rules.py`,
  `tests/doctor/test_checks_config_notification_rules.py`, and existing schema/facade
  tests as appropriate. Cover text/JSON defaults, field provenance, Telegram-only
  validity, malformed rules, and cache invalidation. Explicitly verify toast-only
  suppression still allows Telegram and Telegram-only suppression preserves toast and
  sound.
- Telegram tests: extend `tests/test_outbound.py`, `tests/test_integration.py`, and
  existing snooze/resurface coverage where relevant. Include an integration case using
  the real shared resolver, the global quiet rule plus Athena overlay, a TaskTriage row
  with `panel: beads`, and a non-bead approval/error row. Assert only eligible rows
  reach mocked sends/attachments/pending-action writes and stored notification state is
  preserved.
- Cursor cases: all-suppressed batches leave the cursor unchanged; suppressed rows
  before/between/after eligible rows do not block eligible sends; a failed eligible send
  is retried even with later suppressed rows; equal timestamps retain ID order;
  first-run and snooze resurfacing behavior stay intact. Test rule changes and the
  documented re-eligibility behavior using isolated config.
- Validate the Athena YAML against the updated schema and evaluate its merged rule with
  the existing global rule. After activation, use `sase notify rules` and
  `sase doctor -C config.notification_rules` for read-only confirmation. Do not use the
  real outbound `--dry-run` for verification: it advances the live cursor.

Run formatting and `sase tool run check` in each changed code repo (`sase-core`, `sase`,
and `sase-telegram`). In the Rust repo use its `just` recipes, never bare `cargo`. Read
`lint_and_test.md` for SASE's required check workflow; do not run `check-full`. Use
`/sase_monitor` for long setup, build, and verification commands.

## Landing and activation

Declare every changed repo through `/sase_final`. As documented in
`sase/docs/rust_backend.md`, coordinated core/SASE finalization commits the linked core
first and updates `sase-core-revision.txt` to its pushed SHA before committing SASE.
`sase/sase.yml` already declares the core's `revision_pin: sase-core-revision.txt`;
preserve that coordination rather than inventing a future SHA or manually committing the
linked repo. Published versions remain release-tool owned.

The running SASE adapter, Rust extension, and Telegram plugin must all support this
action for the Athena rule to take effect. Report verified source behavior separately
from live activation. Complete any required global reinstall through the prescribed
`/sase_gate` workflow rather than bypassing the human-only install policy. Once the
chezmoi change lands, ensure its required `chezmoi update -a --force` post-commit apply
runs: `chezmoi/sase/sase.yml` already declares that command as `commit_hooks.after`. Use
the hook and host completion evidence, with a follow-up if needed; a source edit alone
does not update the home configuration. Do not claim live suppression until the updated
runtime and applied overlay are both confirmed.

Acceptance: Athena's Beads notifications remain in SASE with their existing TUI
suppression and produce no new Telegram announcement or attachment; unrelated eligible
notifications continue through the existing delivery/retry flow. The new action is
configurable, validated, documented, and explainable through the existing rule
diagnostics.
