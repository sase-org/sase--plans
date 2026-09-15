---
tier: tale
title: Restore automatic usage refreshes after an early Codex reset
size: medium
goal:
  Restore automatic Codex usage refreshes after a limit event or early usage reset,
  prevent future refresh reminders from blocking normal polling, and recover Apollo and
  Athena.
proposed_by: bbugyi200.apollo.03
create_time: 2026-09-15 09:55:47
status: wip
---

# Restore usage refreshes after an early Codex reset

## Scope and outcome

Fix the shared Rust refresh-admission policy, cover the failure through the real
Rust/Python boundary, and recover the affected machine-local Codex schedules. One
implementation agent can complete this bounded change; use a medium tale. The
investigation and plan authoring are large work under the SASE size guidance.

The user requested diagnosis and a validated plan before implementation. This plan was
prepared with read-only machine inspection; no implementation or live usage-state
changes have been made. The scratch plan is the only authored file.

## Confirmed diagnosis

The user's suspicion is correct: the earlier usage-limit event left local scheduling
state that blocks automatic refreshes even after capacity has been reset and the
provider has been re-enabled. The OpenAI reset itself is not failing. All three machines
have already observed the post-reset weekly window, with the same next reset at
**2026-09-22 11:02:33 UTC** and `vendor_state: allowed`.

Read-only evidence collected on **2026-09-15**, around 13:46–13:50 UTC:

| Evidence                               | Apollo                | Athena                | Mac                                    |
| -------------------------------------- | --------------------- | --------------------- | -------------------------------------- |
| Cached default-window remaining        | 93%                   | 93%                   | 91%                                    |
| Last full observation, UTC             | 13:14:10              | 13:07:18              | 13:45:34                               |
| Window freshness at comparison         | unknown               | unknown               | fresh                                  |
| Persisted Codex due reason             | `disable_expiry`      | `disable_expiry`      | null                                   |
| Persisted Codex due timestamp, UTC     | Sep 19, 12:13         | Sep 19, 08:13         | null                                   |
| Installed binding's automatic decision | deferred: `scheduled` | deferred: `scheduled` | deferred: `fresh`, next normal cadence |
| Installed binding's explicit decision  | due: `explicit`       | due: `explicit`       | due: `explicit`                        |

Additional controls:

- All three report SASE 0.17.1 and usage collection enabled with a 300-second cadence.
  Apollo's installed `sase-core-rs` is 0.34.31; the opened core checkout at `c5992b1`
  still contains the faulty policy.
- The affected schedules have zero consecutive failures, no retry-after restriction, no
  active backoff, and no outstanding usage reservation. Their last collections
  succeeded. Neither affected machine currently has a Codex provider-disable record.
- Codex is background-eligible in the machines' normal shell environments. Bare SSH
  shells omit Athena's NVM bin directory and the Mac's Homebrew environment; do not
  interpret that diagnostic-shell difference as the cause. Athena's interactive zsh
  finds Codex at `~/.config/nvm/versions/node/v22.14.0/bin/codex`; the Mac's login zsh
  finds `/opt/homebrew/bin/codex`.
- The grey indicator is correct presentation of stale information: freshness becomes
  stale after two cadences and unknown after four. The default window is prominent, but
  the stored Codex Spark windows are also stale on the affected machines.

### Causal chain and code locations

1. In the SASE repo, `src/sase/llm_provider/usage_limit_disable.py` calls the refresh
   trigger after a usage-limit disable.
2. `trigger_usage_refresh_after_limit_event()` in
   `src/sase/llm_provider/usage/refresh.py` marks `limit_event` due immediately, then
   writes a future `disable_expiry` reminder and submits an explicit refresh.
3. In the sase-core repo,
   `crates/sase_core/src/provider_usage/store.rs::record_provider_usage_refresh_attempt()`
   deliberately preserves future reminders when a collection finishes. Thus even a
   successful manual refresh after the reset does not remove the old future marker.
4. `crates/sase_core/src/provider_usage/refresh.rs::evaluate_refresh_due()` returns
   `due: false, reason: scheduled` for any future `due_at` **before** evaluating a
   passed reset, a missing observation, or the normal observation cadence. It treats an
   extra reminder as a prohibition on earlier collection.
5. AXE's usage refresh and ACE's fallback both use this core admission policy. The
   indicator eventually becomes grey through
   `src/sase/ace/tui/widgets/_provider_usage_indicator.py::_entry_fragment()`.

This explains the machine difference and why merely refreshing once is insufficient. The
same policy defect applies to any provider with a future reminder; use synthetic
provider tests instead of a Codex-specific exception.

Official OpenAI documentation says to fetch `account/rateLimits/read` after consuming a
reset and to use the returned windows. SASE already collects that endpoint; no
reset-consumption call, authentication change, or inferred percentage is needed. See
[Codex App Server: rate limits and earned resets](https://learn.chatgpt.com/docs/app-server).

## Implementation

### 1. Correct shared admission semantics

Open the core repo with
`sase repo open sase-core -r "Fix future usage reminders blocking cadence"` and use only
its returned path. Follow its AGENTS.md. Keep shared policy in Rust and retain the
existing Python facade and wire schema where possible.

Update `evaluate_refresh_due()` so a future `due_at` is an additional opportunity to
refresh. It can bring collection forward; it cannot postpone an already-due ordinary
refresh.

- Retain existing precedence for provider retry-after, explicit-refresh cooldown,
  explicit requests, and automatic failure backoff. A due marker must not defeat those
  protections.
- A reached marker remains `marked_due`.
- With a future marker, a passed observed reset, a missing full observation, or an
  expired normal cadence still makes automatic collection due immediately.
- When nothing is due yet, report the earlier of the future marker and the next cadence
  timestamp. Use `scheduled` when the marker is the earlier trigger and `fresh` when the
  normal cadence is earlier; make equality deterministic.
- Preserve future markers across successful observations before their deadline, so an
  early collection does not lose the promised expiry refresh. Consume a reached marker
  through the existing finish-attempt path.
- Existing persisted schedules must recover under the corrected policy without a
  store-schema migration or deleting observations.

Keep collection scheduling separate from provider-disable enforcement. Restore the
normal polling cadence even when a usage-limit reminder exists; collection reads
capacity and does not authorize model launches. No new CLI option, UI timer, grey color
change, authentication flow, or global settings change is required.

### 2. Add meaningful regression coverage

In `crates/sase_core/src/provider_usage/tests.rs`, add focused policy/store tests with
injected clocks:

- A future marker several days away plus an observation older than 300 seconds is due by
  cadence. Use the production-shaped `disable_expiry` scenario.
- A future marker cannot suppress never-observed or reset-passed refreshes.
- With fresh data, the next deadline is the minimum of reminder and cadence; test both
  orders and the equality boundary.
- Retry-after still blocks automatic and explicit requests; failure backoff still blocks
  automatic requests; explicit cooldown still applies. Include future and reached
  markers so the fix cannot inadvertently bypass restrictions.
- Reproduce the state lifecycle: record exhausted usage, mark an expiry days ahead,
  record a successful post-reset observation with capacity available, and finish the
  attempt. Verify the future marker survives while ordinary admission succeeds at the
  next cadence. Repeat another success/cadence cycle, then verify the reminder becomes
  due at its deadline and is consumed when that attempt finishes.
- Keep provider/context/generation isolation and reservation joining intact.

Revise `usage_refresh_mark_due_is_once_per_reason_and_survives_future_due`: it currently
expects a never-observed provider to remain `scheduled`. Give it a real fresh complete
observation so it continues testing reminder preservation without encoding the faulty
suppression rule. Keep a separate never-observed regression.

Add a small integration regression in the SASE usage tests, using a temporary SASE home
and the real core binding. Exercise the limit-event trigger and the subsequent automatic
refresh admission after successful post-reset data. Stub only provider transport/proc
launching as needed; the existing tests in `tests/llm_provider/test_usage_refresh.py`
mock admission and therefore miss this bug. Test persisted state from the old policy,
not only schedules created after the fix.

Build/install the edited core into the isolated SASE test environment before running
this regression. Record the loaded binding location/version so a published old wheel
cannot produce a misleading result. Do not duplicate Rust scheduling logic in Python.

### 3. Validate the code change

- Demonstrate the new regression fails against the original rule and passes after the
  fix, using isolated fixtures and no live provider requests.
- In sase-core run `just check` (or `./scripts/check.sh all`), including workspace PyO3
  binding tests. Do not substitute `cargo test -p sase_core` alone.
- Read SASE's `lint_and_test.md` through `sase memory read`, then run `just check` for
  any tracked SASE edits. Run focused usage integration tests against the rebuilt
  binding; obey any broadening/full-check requirement from that verification.
- Use `/sase_monitor` for long checks or timed operational observation. Shared policy is
  the only runtime change, so new visual snapshots or rendering changes should not be
  necessary.
- Leave core release versions to release-plz and repository completion to the host-owned
  finalizer. Do not manually create commits or change release pins to an unpublished
  version.

### 4. Recover Apollo and Athena and verify continuing refresh

The current incident can be repaired through the existing locked store API even if
publication of the permanent core fix is still pending. Perform this only during
approved implementation, after re-reading current state to avoid relying on the
timestamps or generation recorded above.

1. Inspect `sase usage list -p codex --json`, the Codex schedule in
   `~/.sase/llm_provider_usage.json`, and current disable state on both affected hosts.
   Confirm this is still an obsolete future `disable_expiry` marker, with Codex
   available. If a new real limit has occurred, reassess before replacing its marker.
2. Through the installed SASE interpreter, use the public facade
   `mark_provider_usage_refresh_due("codex", context_id, account_generation, "manual_recovery")`
   with the current cached identity. Omitting `due_at` marks it due now and replaces the
   obsolete marker atomically. Do not hand-edit or delete JSON files, clear other
   providers, or alter credentials or disable enforcement.
3. Run `sase usage refresh -p codex --json` using the host's normal CLI environment.
   Athena needs its NVM path (normal interactive zsh provides it). This uses the
   existing bounded read-only provider probe; do not consume another OpenAI reset.
4. Verify a successful fresh observation, `vendor_state: allowed`, no remaining obsolete
   future marker, no unexpected reservation, and a next automatic due deadline
   determined by the normal cadence. Compare percentages and reset times with a
   near-contemporaneous Mac observation; allow active-use drift.
5. Observe an automatic refresh after a full cadence without another explicit request,
   proving the repair persists. Use `/sase_monitor` for the timed wait. Confirm the
   displayed default-window indicator returns to its normal freshness color, or report
   clearly if only the underlying projection could be inspected.

Treat live recovery and delivery of the permanent core fix as separate verifiable
results. Do not claim the fixed Rust code is installed on a machine without checking its
loaded binding/build provenance. Include the core change in the host-owned completion so
future usage-limit events receive the corrected policy through the normal release/update
path.

## Acceptance criteria

- The root-cause explanation connects the prior limit event, preserved future marker,
  incorrect admission order, and grey stale indicator, and explains the Mac control.
- Future reminders cannot delay normal collection, including when old persisted
  schedules are read by the fixed core. Real retry/backoff restrictions remain valid.
- Regression coverage crosses the real Rust/Python boundary and the required checks pass
  against the edited core.
- Apollo and Athena have fresh Codex data and then refresh automatically again;
  operational recovery is not just a single successful manual refresh.
- The completion report distinguishes the machines repaired, automatic-refresh evidence,
  permanent code delivery, and any remaining deployment or UI-verification limitation.
