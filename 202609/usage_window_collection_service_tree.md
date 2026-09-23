---
tier: epic
title: Service-tree usage-window collection with adaptive, provider-safe refresh
goal: 'Periodic usage-window collection runs inside the scheduler service tree (a
  dedicated `usage` routine whose job probes inline) and no longer creates periodic proc
  rows. Usage windows refresh sooner where the numbers actually move (a 60 s routine
  tick plus a "hot" cadence for providers in use). Per-provider polling floors, jitter,
  honored Retry-After, and reason-aware backoff keep every provider from being
  overwhelmed. Each provider''s collection errors are classified, surfaced with a retry
  time, and recovered from without blanking last-known-good windows.

  '
phases:
  - id: core-attempt-policy
    title: "sase-core: reason-aware attempt recording and rate-limit policy"
    depends_on: []
    size: medium
    description:
      "core-attempt-policy: in the linked sase-core repo, add the `rate_limited` reason
      code, an optional observation `retry_after_seconds`, reason-aware backoff classes
      behind an opt-in `adaptive` attempt flag (Retry-After clamping, rate-limit
      escalation, parking, 1 h drift park, 60 s explicit cooldown), collector-health
      `retry_at`/`last_failure_reason`, and pruning of superseded-generation schedule
      rows. Legacy requests must behave exactly as they do today."
  - id: core-admission-policy
    title:
      "sase-core: floors, jitter, parking, hot cadence, and reservation reads in
      admission"
    depends_on:
      - core-attempt-policy
    size: medium
    description:
      "core-admission-policy: extend due/admit evaluation (opt-in via `adaptive` and new
      optional request fields) with per-provider polling floors, deterministic ±10%
      jitter, CLI-fingerprint unparking, and hot cadence (hot hints, warn-level windows,
      passive-coverage suppression). Add a `mark_provider_usage_hot` store operation, a
      read-only live-reservation listing, and per-provider floor-aware freshness on
      reads, all with bindings and golden legacy-compat tests."
  - id: probe-robustness
    title: Probe and runner robustness fixes
    depends_on: []
    size: medium
    description:
      "probe-robustness: fix the runner's batch-deadline double-record, the probe
      TypeError re-run, the process-tree kill gap, the worker env allowlist, the 2 s
      agy/grok version timeouts, Muse's missed-mint blanking, and Codex's best-effort
      `account/read` poisoning the session. Each fix gets a regression test. This phase
      is pure Python and needs no core change."
  - id: rate-limit-plumbing
    title: Rate-limit classification and reason-aware attempt plumbing
    depends_on:
      - core-attempt-policy
      - probe-robustness
    size: medium
    description:
      "rate-limit-plumbing: move the core pin, add a shared rate-limit/Retry-After
      classifier that every collector consults, normalize `not_installed`, and pass the
      reason code, Retry-After, and `adaptive=True` into every recorded attempt. Render
      the new collector-health retry information in `sase usage list -v` and the Models
      panel, and update the usage docs."
  - id: adaptive-admission
    title: Plugin polling floors, CLI fingerprints, and limit events that only mark due
    depends_on:
      - core-admission-policy
      - rate-limit-plumbing
    size: medium
    description:
      "adaptive-admission: move the core pin, let plugins declare
      `min_probe_interval_seconds` (claude 300, muse 180, agy/grok/codex 120), compute
      CLI fingerprints, and pass floors, fingerprints, and `adaptive=True` through
      admission, attempts, and floor-aware reads. Limit events only mark providers due
      and no longer force explicit probes."
  - id: usage-routine
    title: Dedicated `usage` scheduler routine that probes inline
    depends_on:
      - adaptive-admission
    size: medium
    description:
      "usage-routine: move `usage_refresh` out of `checks` into a new 60 s `usage`
      routine whose job runs the admitted batch in-process under non-proc operation IDs.
      Store-based waiting replaces proc waits for joiners (CLI and Models panel). The
      TUI fallback runs only when the scheduler does not own collection, `u` toasts
      render real receipt reasons, and the tests and docs are updated to match."
  - id: hot-cadence
    title: Hot cadence for providers in active use
    depends_on:
      - usage-routine
    size: medium
    description:
      "hot-cadence: add `llm_provider.usage_metrics.active_refresh_seconds` (default
      120), pass the active cadence and warn percent into admission, and write
      best-effort 15-minute hot hints from the agent launch path and limit events.
      Update the config schema, defaults, and docs."
  - id: capability-cache
    title: CLI capability cache for usage probes
    depends_on:
      - adaptive-admission
    size: medium
    description:
      "capability-cache: add a fingerprint-keyed, TTL-bounded on-disk cache of CLI
      version and help capability results so warm Claude probes spawn 2 processes
      instead of 5, and agy/grok skip `--version` spawns. Invalidate an entry on
      fingerprint change, TTL expiry, or a drift/unsupported-version result."
proposed_by: bbugyi200.athena.0q3
create_time: 2026-09-23 11:06:08
status: wip
---

- **PROMPT:**
  [prompts/202609/usage_window_collection_service_tree.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/usage_window_collection_service_tree.md)

# Plan: Service-tree usage-window collection with adaptive, provider-safe refresh

## Context and decisions

The user asked to migrate the periodic usage-window collector off background procs, "to
a service proc". They also want more frequent refreshes, no provider overload, and
graceful per-provider error handling. They asked for the research report
`research:202609/usage_window_collector_hosting/usage_window_collector_hosting.md` to
win wherever it disagrees with their original plan. Every phase worker should read it
first:

```bash
sase artifact read research:202609/usage_window_collector_hosting/usage_window_collector_hosting.md "Design context for the usage-window collector epic"
```

The decisions this epic implements, all taken from that research and verified against
the code on 2026-09-23:

1. **Service-tree ownership, not a new daemon.** The sase scheduler is already a builtin
   service proc. Periodic collection moves into a dedicated `usage` scheduler routine
   (60 s interval). Its `usage_refresh` job runs the admitted probe batch **in-process**
   instead of submitting a proc.
   - This removes every periodic `usage-refresh` proc row. On athena those are 74% of
     non-service proc rows, and they evict user proc history from the 100-row budget.
   - It gives Services-tab job nodes, bounded run history, overrun metrics, and the `r`
     manual-run key for free.
   - It is restarted by `sase update`.
   - A dedicated `usage` daemon is explicitly deferred. Reopen that decision only if one
     of these becomes true: long-lived vendor push connections are wanted;
     user-triggered refreshes must also stop producing proc rows; or the `usage` lane
     chronically overruns.
2. **User-triggered refreshes stay procs.** This covers Refresh-panel `u`, Models-panel
   `u`, and `sase usage refresh`. It matches the user's own rule that user-initiated
   work may be a visible proc, and those paths already pass through the same Rust
   admission. The TUI fallback loop also stays on the proc path, but runs **only** when
   the scheduler does not own collection.
3. **Faster where it matters, never uniformly faster.**
   - The routine ticks every 60 s instead of 300 s, so due work, passed resets, and
     marked-due providers are picked up within a minute.
   - Providers in active use get a "hot" cadence (default 120 s).
   - Plugin-declared floors bound everything. Claude's floor is 300 s because its usage
     endpoint is publicly reported to throttle 30–60 s pollers with sticky 429s.
   - `refresh_seconds` stays the idle cadence, and display freshness uses
     `max(refresh_seconds, floor)`. Faster polling must never blank the header as
     "unknown".
4. **Provider protection and error policy live in `sase_core`** (the Rust boundary
   rule). Collectors only classify evidence: reason code and Retry-After. This covers
   floors, single-flight leases (existing), deterministic ±10% jitter, a 60 s explicit
   cooldown, clamped Retry-After, and reason-aware backoff classes.
5. **Cross-repo compatibility.** Dev installs build `sase_core_rs` from the linked
   `sase-core` checkout at HEAD. Released sase also accepts every core in its
   `sase-core-rs` minor window. So every sase-core change here is **opt-in**:
   - Requests without the new fields (`adaptive`, `min_interval_seconds`, …) must
     produce exactly today's decisions and persisted state. Each core phase adds a
     golden legacy-compat test.
   - New persisted fields use `#[serde(default)]` and are skipped when empty, so rows
     that do not use them stay readable by older cores.
   - Do not add fields to the refresh _outcome_ wires, which Python parses with
     exact-field checks. New free-form reason strings (`floor`, `parked`, `cli_changed`)
     are fine.
   - The sase phases that consume new core behavior move `sase-core-revision.txt` with
     `just ratchet-core-revision`, after the core phase they depend on has landed, then
     rerun `just install`.
   - Never edit the `sase-core-rs` window in `pyproject.toml`; the release job owns it
     (see `docs/rust_backend.md`).
6. **No feature flag.** Each phase lands complete, permanent behavior. Nothing is a beta
   and no old branch must stay reachable for users. (The legacy branch in core exists
   only for wire compatibility; see Follow-ups.)

Error classes the core applies when `adaptive` is set:

| Class        | Reason codes                                                                                                                              | Retry policy                                                                                                                                                                                                                                  |
| ------------ | ----------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Success      | outcome `ok`, `not_applicable`, or `unsupported` with no reason                                                                           | reset failure and rate-limit streaks, parking, and last failure reason; next due at the provider's cadence                                                                                                                                    |
| Transient    | `timeout`, `deadline_exceeded`, `probe_failed`, `parse_error`, `malformed_payload`, `account_context_changed`, or an error with no reason | backoff `max(cadence, floor) × 2^(n-1)`, capped at 1800 s, jittered                                                                                                                                                                           |
| Rate-limited | `rate_limited`                                                                                                                            | Retry-After > 0: `retry_after_until = now + clamp(retry_after, 900, 21600)`; Retry-After 0 or missing: `now + min(900 × 2^(k-1), 7200)`, where k is consecutive rate-limited attempts. Explicit requests already respect `retry_after_until`. |
| Auth         | outcome `unauthenticated`, `logged_out`, `api_mode`                                                                                       | today's generic backoff (30 min cap). Explicit retries are allowed after the cooldown.                                                                                                                                                        |
| Parked       | `not_installed`, `unsupported_cli_version` (any outcome)                                                                                  | backoff 6 h, with `parked_fingerprint` = the attempt's `cli_fingerprint`. Unparked early when a due/admit request carries a different fingerprint. Explicit requests bypass it, as with any backoff.                                          |
| Vendor drift | `vendor_drift`                                                                                                                            | fixed 3600 s backoff (parked for 1 h)                                                                                                                                                                                                         |

Explicit cooldown under `adaptive` is 60 s (legacy: 5 s).

Non-goals:

- No direct-HTTP vendor usage calls.
- No adding, averaging, or merging of overlapping usage windows.
- No routing or disable decisions driven by these numbers.
- No cross-machine federation. Document the per-machine opt-out,
  `llm_provider.usage_metrics.providers.<name>: false` in a machine overlay, for hosts
  that share an account.
- No persisted hourly probe budget, and no change to the 3-probe local concurrency cap.

Rules for every phase:

- Work in the linked sase-core repo only through `sase repo open sase-core`, read its
  `AGENTS.md`, and verify there with `sase tool run check`.
- In sase, read the `lint_and_test` memory and verify with `sase tool run check`. Do not
  run `just check-full`.
- Read the `tui` and `tui_perf` memories before touching TUI code.
- Keep blocking disk and store I/O off the UI thread and the Textual message pump.

## Phase core-attempt-policy — sase-core: reason-aware attempt recording and rate-limit policy

All changes are in `crates/sase_core/src/provider_usage/` (`mod.rs`, `refresh.rs`,
`store.rs`, tests) and the provider-usage binding domain in `crates/sase_core_py`.

1. `UsageReasonCode::RateLimited` (`rate_limited`).
2. `ProviderUsageObservationWire.retry_after_seconds: Option<f64>`:
   - `#[serde(default, skip_serializing_if = "Option::is_none")]`;
   - validated in `validate_usage_observation` as finite and ≥ 0, clamped to ≤ 86 400;
   - persisted with `last_attempt` as usual.
3. `ProviderUsageRefreshAttemptWire` gains optional fields, all serde-defaulted:
   - `reason_code: Option<UsageReasonCode>`;
   - `min_interval_seconds: Option<f64>` (finite, 60..=86 400);
   - `cli_fingerprint: Option<String>` (bounded length, no control characters);
   - `adaptive: bool` (default `false`).

   The attempt binding accepts them as keyword arguments with defaults, so existing
   Python callers are unaffected.

4. `ProviderUsageRefreshScheduleWire` gains persisted fields, each `#[serde(default)]`
   and skipped when empty or zero:
   - `last_failure_reason: Option<String>`;
   - `consecutive_rate_limits: u32`;
   - `parked_fingerprint: Option<String>`.
5. Add a pure, table-tested policy function (for example `refresh_failure_policy`) that
   implements the class table in the Context section.
   `record_provider_usage_refresh_attempt` uses it only when `adaptive` is true:
   - the floor enters the transient backoff base as `max(cadence, floor)`;
   - the rate-limit streak escalates;
   - parked attempts store `parked_fingerprint`;
   - any success clears the new fields;
   - `cooldown_until = now + 60`.

   With `adaptive` false, the code path, the 5 s cooldown, and the unclamped
   `retry_after_seconds` handling are exactly today's. Jitter is not applied here;
   `core-admission-policy` owns it.

6. `UsageCollectorHealthWire`, the public snapshot, gains:
   - `last_failure_reason: Option<String>`;
   - `retry_at: Option<f64>`, the later of `retry_after_until`/`backoff_until` when it
     is in the future.

   Do not add health-state enum variants. Python consumes the snapshot as a dict, so
   these additive fields are safe for older sase.

7. Superseded-generation cleanup: when the store writes state, drop schedule rows whose
   `(provider, context_id)` has a provider record at a newer `account_generation`.
   Athena currently holds a stale generation-1 `claude` row.
8. Tests:
   - the class table, clamps, and streak escalation;
   - a golden test proving legacy (non-adaptive) attempts produce byte-identical
     schedules to pre-change behavior;
   - round-trip of old store files that lack the new fields;
   - generation pruning;
   - binding tests in `sase_core_py`.

Versioning: follow sase-core `AGENTS.md` (release-plz owns versions; Conventional
Commits). An older core reading a store written with `rate_limited` or the new fields
drops those rows with a diagnostic; this is a downgrade-only scenario. Apply AGENTS.md's
breaking-change rule to that fact and mark the commit accordingly.

## Phase core-admission-policy — sase-core: floors, jitter, parking, hot cadence, and reservation reads in admission

This phase is also sase-core only, and builds on `core-attempt-policy`.

1. `ProviderUsageRefreshDueRequestWire` and `ProviderUsageRefreshAdmitRequestWire` gain
   optional, serde-defaulted fields (Python bindings take them as defaulted kwargs):
   - `adaptive: bool`;
   - `min_interval_seconds: Option<f64>`;
   - `cli_fingerprint: Option<String>`;
   - `active_cadence_seconds: Option<f64>` (≥ 60; treated as ≤ the idle cadence);
   - `warn_percent: Option<f64>`.
2. `hot_until: Option<f64>` persisted on the schedule (serde default, skipped when
   `None`).
3. Adaptive due evaluation. Keep the legacy function or path intact. Under `adaptive`,
   the order is:
   1. `retry_after` defers everything, explicit requests included.
   2. The explicit cooldown defers explicit requests (`cooldown`).
   3. Explicit requests are due (`explicit`).
   4. Parking: `backoff_until > now` with `parked_fingerprint` set. If the request's
      `cli_fingerprint` is present and differs, the provider is due (`cli_changed`);
      otherwise it is deferred (`parked`).
   5. `backoff`.
   6. **Floor:** when `last_started_at + min_interval_seconds > now`, defer with reason
      `floor` and `next_at = last_started_at + floor`. This applies to every remaining
      automatic reason.
   7. `marked_due`, then `reset_passed`, then `never_observed`, then cadence. The
      cadence interval is `(hot ? max(active, floor) : max(idle, floor)) × jitter`,
      never below the floor.
4. **Jitter:** a deterministic factor in [0.9, 1.1] from a stable hash of
   `(provider, context_id, account_generation, observed_at)`. It applies to cadence
   intervals and adaptive backoff durations, never to Retry-After or floors. It must be
   reproducible in tests.
5. **Hot**, when `active_cadence_seconds` is provided, means `hot_until > now`, or any
   stored window that has not reset has `used_percent ≥ warn_percent`.
   - **Passive-coverage suppression:** when every stored window's `received_at` is
     within `active_cadence_seconds` of now, the provider is not treated as hot. Live
     stream events, such as Claude's `rate_limit_event`, already keep it fresh.
   - This rule is provider-neutral. Do not branch on provider name.
6. New store operation
   `mark_provider_usage_hot(sase_home, {provider, context_id, account_generation, until}, now)`,
   with a binding:
   - exclusive lock;
   - `until` capped at `now + 3600`;
   - no write when the existing `hot_until ≥ until − 60` (coalescing).
7. New read-only `list_provider_usage_refresh_reservations(sase_home, now)` (shared
   lock), with a binding. It returns live, unexpired reservations as the existing
   reservation wire list.
8. Read path: the store-read binding accepts an optional `provider_min_intervals` map.
   Per-provider freshness uses `max(cadence, floor)`. When the map is absent, reads
   behave as today.
9. Tests:
   - evaluation-order table tests;
   - jitter bounds and determinism;
   - a floor that blocks `marked_due` and `reset_passed`;
   - park and unpark on a fingerprint change;
   - each hot source and passive suppression;
   - `mark_provider_usage_hot` coalescing and cap;
   - reservation listing (expired rows excluded);
   - floor-aware freshness;
   - golden legacy-compat tests for due/admit/read without the new fields;
   - binding tests.

## Phase probe-robustness — Probe and runner robustness fixes

All changes are in `src/sase/llm_provider/usage/`, pure Python, each with a regression
test in `tests/llm_provider/`.

1. **Batch-deadline double-record** (`refresh_runner._run_admitted_refresh`).
   - When the wait loop breaks at the work deadline, leaving the `ThreadPoolExecutor`
     block waits for running probes. Those probes record their real observation and
     attempt and release their lease.
   - The code afterwards records `deadline_exceeded` for every entry still in
     `in_flight`, a second record that can turn a success into backoff.
   - Fix: after the executor exits, collect results from futures that finished (they
     already recorded themselves). Record `deadline_exceeded` only for jobs that never
     started or were cancelled.
2. **Probe `TypeError` re-run** (`probe.probe_in_process`). Any `TypeError` raised
   _inside_ a collector re-runs the probe positionally, which means a second credential
   mint for Muse. Decide the calling convention once, by inspecting the signature for a
   `context` parameter, instead of catching `TypeError`.
3. **Process-kill gap** (`transport._terminate_process_tree`).
   - The code sends SIGKILL only if the root survives the SIGTERM grace period. A
     grandchild that ignores SIGTERM outlives a root that exits and is no longer found
     under the root's pid.
   - Fix: snapshot the descendant pids and process groups before SIGTERM, then SIGKILL
     the snapshot's surviving groups and pids unconditionally after the grace period.
4. **Worker env allowlist** (`probe._ALLOWED_ENV_NAMES`): allow the following. None of
   them match the denied secret markers.
   - `CLAUDE_CONFIG_DIR`;
   - `HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY`, `ALL_PROXY` (the check upper-cases names,
     so lowercase variants follow);
   - `NODE_EXTRA_CA_CERTS`.
5. **Version-probe timeouts:** raise `_VERSION_PROBE_TIMEOUT_SECONDS` in `agy.py` and
   `grok.py` from 2.0 to 4.0, still bounded by the remaining deadline.
6. **Muse missed mint** (`muse._poll_usage`). A budget that runs out with no `usage`
   member currently becomes an authoritative-empty observation, which blanks the stored
   Muse windows.
   - Return `error` / `timeout` instead, so last-known-good windows are kept and age
     through freshness.
   - A genuinely logged-out host then shows aging windows plus collector-health failures
     instead of a blank.
   - Update the docstring that documents the old trade-off, and the tests.
7. **Codex best-effort `account/read`** (`codex_collector._probe_auth_mode` with
   `transport.JsonLineSession.read_response`).
   - A transport error on the best-effort read closes the session, so the following
     `account/rateLimits/read` fails.
   - Keep the best-effort call from poisoning the session. Give it a short sub-deadline,
     and either do not close on its failure or start a fresh session for the rate-limit
     read within the remaining deadline.
   - Verify the current behavior with a test first.

## Phase rate-limit-plumbing — Rate-limit classification and reason-aware attempt plumbing

1. Move the core pin (`just ratchet-core-revision`) to a sase-core commit containing
   `core-attempt-policy`, run `just install`, and confirm the pinned-bindings check
   passes.
2. `types.py`: add `"rate_limited"` to `UsageReasonCode`, and a `_DIAGNOSTIC_BY_REASON`
   entry such as "provider is rate-limiting usage requests".
3. `_strategy.py`: add a shared classifier, for example `detect_rate_limit(...)`.
   - Inputs: JSON-RPC and ACP error mappings, exit code, stdout, and stderr.
   - It recognizes HTTP 429, "rate limit"/"rate-limited", and "too many requests" in
     text and in error codes, messages, or data.
   - It extracts Retry-After from `retry-after: N`, "retry after/in N seconds|minutes",
     or `retry_after`/`retryAfter` fields.
   - It returns evidence, including an optional `retry_after_seconds`, or `None`.
4. Every collector (claude, codex, agy, grok, muse) consults the classifier on its
   failure paths **before** other classification. On a match it returns outcome `error`,
   reason `rate_limited`, and `retry_after_seconds`. agy must check stderr and non-JSON
   output before falling through to `parse_error`.
5. Normalize "CLI not installed": muse, agy, and grok report outcome `unsupported` with
   reason `not_installed`, as claude and codex already do.
6. Runner (`refresh_runner._finish_job` and its callers):
   - pass `reason_code` and `retry_after_seconds` from the persisted observation, plus
     `adaptive=True`, into `record_provider_usage_refresh_attempt`;
   - deadline and crash paths pass `deadline_exceeded` / `probe_failed`;
   - extend the Python facade (`_facade.py` / `store.py`) for the new attempt fields.
7. Presentation:
   - `sase usage list -v` (plain and rich) and the Models-panel detail
     (`models_panel_usage_rendering.py`) render `collector_health.last_failure_reason`
     and `retry_at`, for example "rate-limited · retry ~14:05" or "CLI response changed
     · retry in 52m";
   - JSON output passes the snapshot through unchanged.
8. Docs:
   - `docs/llms.md`, "Subscription usage extension": `rate_limited`, the observation
     `retry_after_seconds`, and the classifier expectation for plugin authors;
   - a short error-class table where the usage refresh policy is documented
     (`docs/configuration.md` usage section and/or `docs/agent_providers.md`).
9. Tests:
   - classifier fixtures, including agy's non-JSON rate-limit text and Retry-After
     parsing;
   - each collector's rate-limit path;
   - the runner passing reason and Retry-After;
   - presentation of retry information.

## Phase adaptive-admission — Plugin polling floors, CLI fingerprints, and limit events that only mark due

1. Move the core pin again to include `core-admission-policy`, then run `just install`.
2. Plugin floors:
   - widen `_registry_metadata._usage_capabilities` to carry
     `min_probe_interval_seconds` (finite, 60..=86 400, otherwise dropped); callers that
     test `probe is True` keep working;
   - declare it in each provider's `llm_usage_capabilities()`: claude 300, muse 180, agy
     120, grok 120, codex 120;
   - document it in `_hookspec.py`.
3. Helpers in `refresh.py` (or a small sibling module):
   - `usage_probe_floor(provider)` and `usage_probe_floors()`, memoized per config
     token;
   - `usage_cli_fingerprint(provider)`, which returns `"<path>:<mtime_ns>:<size>"` for
     the executable resolved exactly as `_provider_cli_ready` resolves it (including
     codex's resolver), or `None`.
4. `_admit_one` passes `adaptive=True`, `min_interval_seconds`, and `cli_fingerprint` to
   due and admit. The runner payload carries each provider's floor and fingerprint, and
   `_finish_job` passes them to the attempt.
5. Limit events: `trigger_usage_refresh_after_limit_event` only marks due
   (`limit_event`, plus `disable_expiry` at expiry) and no longer submits an explicit
   refresh.
   - The next routine tick picks the provider up, subject to its floor.
   - Adjust the `usage_limit_disable` caller and the `USAGE_REFRESH_ORIGINS` handling.
     Keep origin normalization tolerant.
6. Reads: the Python `load_provider_usage(...)` wrapper passes `provider_min_intervals`
   from `usage_probe_floors()`, so the CLI, the header indicator, and the Models panel
   all use floor-aware freshness. Confirm the TUI callers still load off the UI thread.
7. Tests:
   - Claude is never auto-probed within 300 s, even when marked due or when a reset
     passed;
   - explicit requests bypass the floor but respect the 60 s cooldown and `retry_after`;
   - a parked provider unparks on a fingerprint change;
   - limit events spawn no probe;
   - floor-aware freshness.

   Update `tests/llm_provider/test_usage_refresh.py` expectations that encoded the old
   explicit-submit limit-event behavior or exact non-jittered timings.

8. Docs: `min_probe_interval_seconds` in `docs/llms.md`, and the floor semantics next to
   `refresh_seconds` in `docs/configuration.md`.

## Phase usage-routine — Dedicated `usage` scheduler routine that probes inline

1. `src/sase/default_config.yml`: add `axe.routines.usage`.
   - `interval: 60` and `job_timeout: "90s"`, well above the 45 s batch deadline. A
     killed job would orphan probe workers, which run in their own sessions.
   - A description following the Description Grammar.
   - Its single job `usage_refresh` (`script: sase_job_usage_refresh`), moved out of
     `checks` with an updated description.
   - `tests/test_axe_lumberjack_config.py` must assert the job now lives in `usage`.
2. Inline execution in `refresh.py`:
   - Add an inline mode to the shared submit path, for example `execution="inline"`, so
     admission, receipts, and lease-release-on-failure stay shared.
   - Inline operation IDs must be recognizably non-proc, for example
     `usage-job:<new_proc_id()>`, with a helper such as
     `is_inline_usage_operation(...)`.
   - After admission, call the existing `_run_admitted_refresh` in-process with the same
     payload the proc runner gets: isolated workers, deadlines, and per-job lease
     release.
   - On an exception, release any started leases and mark those providers `error`.
   - Return the per-provider runner results alongside the receipt.
3. `src/sase/scripts/sase_chop_usage_refresh.py`:
   - scheduled runs perform due-only inline refreshes;
   - manual runs (`runtime.context.source != "scheduled"`, meaning the Services-tab `r`
     or `sase axe job run`) perform explicit inline refreshes, still subject to cooldown
     and Retry-After;
   - emit a summary with per-provider status (for example
     `claude=ok codex=ok grok=backoff`) plus counts, and reason `nothing_due` when
     nothing ran;
   - fix the module docstring.
4. Store-based joins:
   - Add a Python facade for `list_provider_usage_refresh_reservations` and a helper,
     for example `wait_for_usage_refresh_operations(ids, timeout)`, that polls until
     none of the IDs hold a live reservation.
   - `usage_handler._wait_for_operation_ids` keeps `wait_for_proc` for proc IDs. Inline
     IDs use the store wait and synthesize per-provider results from the refreshed
     snapshot, so exit codes and `--json` `operation_results` stay meaningful.
5. Models panel (`models_panel_usage_modal.py`):
   - Replace the proc-scope-conflict detection in `_attach_in_flight_refreshes` and
     `_poll_pending` with store reservations, which cover both proc- and inline-owned
     refreshes.
   - Run the store reads in a coalesced thread worker, never in the timer callback.
     Follow the `tui_perf` rules.
6. TUI fallback (`_usage_refresh_fallback.py`):
   - Each tick, inside the existing `to_thread` work, skip submission when the scheduler
     owns collection: `is_axe_running()` and an enabled job whose script is
     `sase_job_usage_refresh` exists in the loaded axe config, in any routine.
   - Otherwise keep today's due-only proc submission.
   - Tick every 60 s, since due evaluation gates actual probing.
7. Receipt reasons:
   - `_UsageRefreshProviderResult` carries `due_at` from the due/admit outcome, and
     `to_json` includes it.
   - Add one shared compact renderer for toasts, for example "Refreshing usage: claude,
     codex · grok rate-limited, retry ~14:05 · agy cooldown, retry in 45s".
   - Use it in `refresh_panel._notify_usage_refresh_receipt` and in the Models panel's
     no-provider-started path.
   - Remove the blanket "Usage refresh already running" message.
8. Tests:
   - the inline job submits no proc, releases its leases, and writes the summary;
   - a manual run is explicit;
   - the CLI join-wait on an inline operation;
   - Models-panel tracking of an inline-owned refresh;
   - the fallback gate in both states;
   - toast text for each deferral reason.
9. Docs:
   - `docs/axe.md`: a new `usage` routine section, removal of the job from the `checks`
     table and prose, and the `r` manual-run semantics;
   - the cadence and "background proc" wording in `docs/llms.md`,
     `docs/agent_providers.md`, and `docs/configuration.md`;
   - the multi-machine opt-out.

## Phase hot-cadence — Hot cadence for providers in active use

1. Config: `llm_provider.usage_metrics.active_refresh_seconds`, default 120, minimum 60,
   clamped to ≤ `refresh_seconds`. Add it to:
   - `UsageMetricsSettings` and its parsing/validation in `usage/config.py`;
   - `src/sase/default_config.yml` (usage_metrics block, with a comment);
   - `src/sase/config/sase.schema.json`;
   - the `docs/configuration.md` table.
2. `_admit_one` passes `active_cadence_seconds` and `warn_percent` (from settings) into
   due and admit.
3. Hot hints:
   - a Python facade for `mark_provider_usage_hot(provider, until)` in the usage store
     module;
   - it is best-effort, never raises, and logs at debug level.
4. Hint writers (15 minutes):
   - the agent launch path in `llm_provider/_invoke.invoke_agent`, after the provider is
     resolved, only when collection is enabled for that provider and it has probe
     capability. It must not add latency or failure modes to launches;
   - `trigger_usage_refresh_after_limit_event`.
5. Tests:
   - settings validation and clamping;
   - admission passes the active cadence;
   - launch and limit-event hints are written, and failures are swallowed;
   - an end-to-end due decision where a hot codex is due at ~120 s and a hot claude
     still waits for its 300 s floor.
6. Docs: the hot rules (launch hint, warn-level window, limit event, passive-coverage
   suppression) in `docs/configuration.md` / `docs/llms.md`.

## Phase capability-cache — CLI capability cache for usage probes

1. New module, for example `usage/_capability_cache.py`:
   - per-provider JSON entries under the SASE home cache directory, for example
     `~/.sase/cache/usage_probe_capabilities/<provider>.json`;
   - keyed by `usage_cli_fingerprint`;
   - 24 h TTL;
   - atomic temp-file-and-rename writes;
   - corruption-tolerant: a bad entry is a miss.

   Workers read and write it directly, so nothing is passed through the payload.

2. Claude (`usage/claude.py` `_preflight_usage_probe` and helpers): cache the
   `--version`, `-p --help` (zero-cost probe support), and `auth status --help` results.
   `auth status` and the `/usage` run stay uncached, so a warm probe spawns 2 processes
   instead of 5.
3. agy and grok: cache the `--version` result.
4. Invalidation: a fingerprint change or TTL expiry. A probe that ends in
   `unsupported_cli_version` or `vendor_drift` deletes its provider's entry.
5. Tests:
   - warm versus cold spawn counts using fake executables;
   - a fingerprint change invalidates the entry;
   - a corrupt entry is ignored;
   - drift invalidation.

## Follow-ups (not in this epic; phase workers record `PROPOSED FOLLOW-UP:` notes rather than beads)

- Delete core's legacy, non-adaptive refresh branch once sase's published `sase-core-rs`
  window requires a core that has the adaptive policy.
- After about a week of classified attempts (429 counts, durations), revisit the floors
  and the hot cadence. Do not lower any floor before measuring.
- Promote to a dedicated `usage` daemon only under the reopen conditions in decision 1.
