---
tier: tale
title:
  Finish Telegram /usage — fix red sase-telegram CI, Claude model-weekly period, and
  refresh-flow gaps
goal:
  sase-telegram master CI is green again, the Claude Fable window renders as "Fable ·
  Weekly" as the original /usage design specified, and the /usage Refresh flow never
  strands a "⏳ Refreshing…" keyboard or silently drops a store-read error.
size: medium
proposed_by: bbugyi200.athena.0t8
create_time: 2026-09-27 15:39:10
status: wip
---

# Finish the Telegram `/usage` command

## Background

The original plan (`telegram_usage_command.md`, a two-repo tale) is mostly landed:

- **sase** commit `315746c2a` added `sase.integrations.usage_windows` (+
  `_usage_windows_models.py`), the `"chat"` refresh origin, tests, and the
  `docs/integrations.md` "Usage Windows" section. Its tests pass and its symvision URI
  pragmas are satisfied.
- **sase-telegram** commit `c386148` added `usage_format.py`,
  `inbound_handlers/usage_command.py`, command/callback routing, the reserved name, the
  job-tick hook in `scripts/sase_tg_inbound.py`, the not-modified fix in
  `telegram_client.edit_message_text`, tests, and docs.

A smoke render against the live usage store produces the designed layout. A review found
these remaining gaps, which this plan closes:

1. **sase-telegram master CI is red** (CI run for `c386148`, "Run tests" step; lint
   passed). Two new tests are wrong. The implementation under test is correct in both
   cases:
   - `tests/test_usage_command.py::test_tick_timeout_shows_latest_data` sets the clock
     to `2000.0` against `deadline_at: 1075.0`. That is 925 s past the deadline, so it
     hits the "more than 10 minutes past its deadline → delete" branch, not the timeout
     delivery branch.
   - `tests/test_usage_format.py::test_html_escaping_of_hostile_labels` puts the hostile
     text in `label` but keeps `period="weekly"`. The vendor label is then never
     rendered, so `&lt;script&gt;` never appears.
2. **The Claude Fable window renders as `Fable · Claude we…`, not `Fable · Weekly`.**
   The original plan said "No Rust change is needed". That assumption was wrong. The
   Claude probe (`src/sase/llm_provider/usage/_claude_support_windows.py`,
   `_usage_window_identity`) emits model-scoped weekly windows with key
   `weekly:<model-slug>`, applicability `{"kind": "models", "model_ids": [...]}`, and
   `duration_seconds: null`. In sase-core,
   `crates/sase_core/src/provider_usage/indicator.rs` `is_weekly_window` has a Claude
   key fallback,
   `(key == "weekly" || key.starts_with("weekly:")) && is_claude_product_scope(window)`,
   which only accepts `Product { product: "claude" }` applicability. So the projection
   classifies `weekly:claude-fable-5` as `period.kind = unknown`, and the renderer falls
   back to the vendor label. The existing core test fixture passes only because it gives
   the Fable window `duration_seconds: Some(WEEK_SECONDS)`, so the duration check wins
   before the key fallback.
3. **The renderer re-sorts windows by key.** `usage_format._window_sort_key` sorts each
   provider's windows alphabetically. The design contract is snapshot (vendor) order,
   which the facade already returns.
4. **The headline can pick the wrong "tightest" window.** `render_headline` uses
   `float(getattr(w, "remaining_percent", 100.0) or 100.0)`, which turns a real `0.0`
   into `100.0`.
5. **An overflowing refresh strands the busy keyboard.** In `_handle_usage_callback`, if
   the "⏳ Refreshing …" view does not fit in one message, the handler sends new chunks
   with the `⏳ Refreshing…` keyboard and returns _before_ writing the pending record.
   No completion ever arrives, and the busy button answers "Still refreshing…" forever.
6. **Store-read errors are dropped on the Refresh and tick paths.** Only `/usage` itself
   renders the designed "⚠️ Couldn't read usage data: …" state. On a Refresh tap, a
   `ProviderUsageStateError`/`OSError` from `usage_windows_report` is logged and the
   message stays unchanged. On the job tick, the record `continue`s every 5 s until it
   expires and is deleted, which leaves the `⏳ Refreshing…` keyboard stuck.
7. **Hygiene.** `_usage_facade` has a dead `if …: raise` / `raise` branch and treats
   _every_ `ImportError` as "installed sase lacks the facade". The original plan asked
   for the same guard as `update_command.start_chat_install_worker`, which only maps
   imports of the facade module itself to "unsupported". There are also unused
   `timezone` imports (`usage_format.py`, and the fallback in `_usage_timezone`), and a
   stray `from datetime import UTC` below the first-party imports in `usage_command.py`.
   The `edit_message_text` not-modified test lives in `tests/test_usage_command.py`, but
   the original plan put it in `tests/test_telegram_client.py`.

## Changes

### A. sase-core: classify Claude model-scoped weekly windows as weekly

Open the linked checkout with `sase repo open sase-core -r "<reason>"`, work only in the
printed path, and read its `AGENTS.md` first.

- In `crates/sase_core/src/provider_usage/indicator.rs`, `is_weekly_window`: widen the
  Claude key fallback so that `key.starts_with("weekly:")` also accepts
  `UsageApplicabilityWire::Models { .. }` applicability, alongside the existing Claude
  product scope. Keep `key == "weekly"` product-scope-only, and keep the existing
  duration checks first (a known non-week duration still returns `false`). Do not change
  `is_all_model_scope`, `weekly_all`, or the header-anchor logic. A models-scoped window
  is never all-models, so header policy and anchors are unaffected.
- Tests in `crates/sase_core/src/provider_usage/tests/indicator.rs`:
  - A Claude `weekly:claude-fable-5` window with `Models` applicability and
    `duration_seconds: None` projects `period.kind == Weekly`, `scope.kind == Models`,
    and `weekly_all == false`.
  - A Claude `window:<something>` window with `Models` applicability and no duration
    still projects `period.kind == Unknown`.
- Verify with sase-core's `sase tool run check` (see its `AGENTS.md`).

No sase code change is needed. The facade forwards `period` untouched, and the
`core-pin-ratchet` workflow moves `sase-core-revision.txt`. Do not add a sase test that
depends on the new classification. Visible side effects in sase are nil: the TUI compact
header name for this window stays `fable`, because the compact form drops the `wk`
token, and `sase usage list` does not render period.

### B. sase-telegram

Open with `sase repo open sase-telegram -r "<reason>"`, read its `AGENTS.md`, and work
only in the printed path. Respect the `inbound_handlers` layering rule.

1. **Fix the two failing tests**:
   - `test_tick_timeout_shows_latest_data`: set the clock inside the timeout window, for
     example `1100.0` (25 s past `deadline_at`). Assert:
     - the edited text contains `Refresh still running for codex after 100s`
     - the normal `🔄 Refresh` keyboard is attached
     - the record file is deleted
   - `test_html_escaping_of_hostile_labels`: make the vendor label actually render. Use
     `period="unknown"` on the hostile-label window, and add a second window with
     `scope="unknown"` and a hostile `scope_vendor_label`. Assert that no raw `<script>`
     appears and that `&lt;script&gt;` does.
2. **Keep vendor window order.** In `usage_format.build_usage_blocks`, iterate
   `provider.windows` as given and delete `_window_sort_key`. Add a test that windows
   with keys in non-alphabetical order (e.g. `weekly`, then `session`) render in the
   given order.
3. **Fix the tightest selection.** In `render_headline`, sort by a finite-number helper
   that defaults to `100.0` only when `remaining_percent` is missing or non-finite,
   never when it is `0.0`. Add a test with two exhausted windows at `0%` and `5%`
   (distinct labels) where the headline names the `0%` window.
4. **Never strand the busy keyboard on overflow.** In the refresh-started branch of
   `_handle_usage_callback`, when the "⏳ Refreshing …" render is more than one chunk:
   - do not send busy chunks, and leave the original message alone (the callback toast
     already acknowledged the tap)
   - still write the pending record keyed to the original `chat_id`/`message_id`

   `_finish_ready_usage_refreshes` already sends new messages when the fresh report
   overflows, and the duplicate-tap guard keeps working because it checks the same
   record path. Add a test: with a two-chunk render, no message is sent with the busy
   keyboard, the record is written, and a later settled tick delivers through
   `_send_html_chunks` with the Refresh keyboard and deletes the record.

5. **Render the store-unreadable state everywhere.** Extract one helper in
   `usage_command.py`, e.g.
   `_render_usage_view(facade, scope, *, now, status_line=None) -> tuple[list[str], bool]`,
   which returns `(chunks, unreadable)`. It wraps `_render_report_chunks` and maps
   `ProviderUsageStateError` (using the existing lazy-import check) and `OSError` to the
   single chunk `⚠️ Couldn't read usage data: {escaped error}`. Use it in:
   - `/usage`
   - the `all` view callback
   - both refresh branches (started and not started)
   - the tick

   Every one of those paths then shows that state with the `🔄 Refresh` keyboard. In the
   refresh-started branch, if the view is unreadable, still submit nothing new: edit to
   the unreadable state with the Refresh keyboard and write no record. In the tick,
   treat the unreadable state as a successful render: edit, then delete the record.
   Other exceptions keep their current log-and-skip behavior. Tests:
   - A Refresh tap whose report raises `OSError` edits the message to the unreadable
     text with the Refresh keyboard.
   - A settled tick whose report raises `OSError` does the same and deletes the record.

6. **Hygiene**:
   - Rewrite `_usage_facade` to mirror `update_command.start_chat_install_worker`. An
     `ImportError` whose `name` is `sase.integrations` or
     `sase.integrations.usage_windows`, or whose message names one of the facade
     symbols, raises a small sentinel that callers map to the "doesn't support /usage
     yet" text. Any other `ImportError` propagates and is logged by the caller as a real
     failure. Keep `test_import_error_fallback` passing, and add a case where an
     unrelated `ImportError` is _not_ reported as "doesn't support /usage yet".
   - Remove the unused `timezone` imports. Put `from datetime import UTC` in the
     standard-library import group of `usage_command.py`.
   - Move `test_edit_not_modified_returns_true_without_retry` from
     `tests/test_usage_command.py` into `tests/test_telegram_client.py`, next to the
     other `edit_message_text` tests.

No README or docs change is needed. The documented behavior is unchanged, apart from now
being honored.

## Verification

- sase-core: `sase tool run check` in the linked checkout.
- sase-telegram: from its linked checkout, `just install` (it builds against this
  workspace's sase source and the linked sase-core), then `sase tool run check`. Every
  test must pass, including the full `tests/test_usage_*.py` and
  `tests/test_telegram_client.py`.
- sase: no files change. Only if something in sase does change, follow the lint-and-test
  memory note, and rerun
  `SYMVISION_EXTERNAL_REPO_PATHS=<linked sase-telegram checkout path> just _lint-symvision`
  so the facade's URI pragmas still resolve against the edited sase-telegram tree.
- Smoke render with no sends: after rebuilding the workspace's `sase_core_rs` from the
  linked sase-core checkout (`just install` in sase), run a short Python snippet with
  the sase-telegram venv that calls `usage_windows_report()` and
  `build_usage_chunks(...)` with `sase.core.time.get_timezone()` and prints the HTML.
  Confirm:
  - the Claude section shows `Fable · Weekly · ↻ …`
  - windows appear in vendor order
  - no line contains raw tags or `None`

## Out of scope

- Renaming the odd Claude vendor label `Claude weekly week (Fable)` produced by the
  Claude probe, or any other change to `sase usage list` or TUI header output.
- Proactive low-capacity Telegram alerts (still a possible `feature` follow-up).
