---
tier: epic
title: Finish usage-window collection landing fixes and floor-aware header freshness
parent_bead: sase-16z
goal: "The usage-window collection work from epic sase-16z is actually complete. Its
  epic-introduced symvision failures are gone. agy and grok capability-cache entries hit
  in production. Rate-limit evidence on transport failures is classified. The stale chop
  test no longer spawns real provider CLIs. The TUI header usage indicator uses the same
  floor-aware freshness (`max(refresh_seconds, floor)`) as the CLI and Models panel, so
  a provider polled at its floor never shows as stale or unknown between probes.

  "
phases:
  - id: landing-fixes
    title: Fix sase-16z landing defects in sase
    depends_on: []
    size: medium
    description:
      "landing-fixes: resolve the three epic-introduced symvision failures (reuse
      resolve_provider_cli_command in refresh readiness; privatize the capability-cache
      dir/invalidate helpers), make executable_fingerprint resolve bare commands through
      PATH so agy/grok cache entries hit, classify rate limits on JSON-line transport
      failures, stop 429 matching decimals, remove the stale real-CLI chop test, keep
      the inline crash path from overwriting finished providers, keep a failed
      Models-panel reservation read from ending tracking early, and fix stale comments,
      a test name, and the axe.md opt-out key."
  - id: core-indicator-floors
    title: "sase-core: per-provider polling floors in the usage indicator projection"
    depends_on: []
    size: small
    description:
      "core-indicator-floors: in the linked sase-core repo, add an optional
      serde-defaulted `provider_min_intervals` map to the usage indicator projection
      request, validated like the floor-aware store read, and compute each window's
      freshness from `max(cadence_seconds, floor)` for providers that name a floor.
      Requests without the field must project exactly as today."
  - id: header-floor-freshness
    title: Floor-aware freshness for the TUI header usage indicator
    depends_on:
      - core-indicator-floors
      - landing-fixes
    size: small
    description:
      "header-floor-freshness: move the sase-core pin past core-indicator-floors, pass
      per-provider polling floors captured off the UI thread into the header indicator
      projection, and test and document that the header uses `max(refresh_seconds,
      floor)` freshness like the CLI and Models panel."
proposed_by: bbugyi200.athena.sase-16z.land
create_time: 2026-09-23 16:32:16
status: wip
---

- **PROMPT:**
  [prompts/202609/usage_collection_landing_fixes.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/usage_collection_landing_fixes.md)

# Plan: Finish usage-window collection landing fixes and floor-aware header freshness

## Context

This is the remaining work that the land agent for epic `sase-16z` ("Service-tree
usage-window collection with adaptive, provider-safe refresh") found while verifying the
epic. All eight phases (`sase-16z.1`–`sase-16z.8`) landed. Their commits are
`caca6b60f`, `ed8172fda`, `fcce8f2f3`, `5e50d27f5`, `a6e27583c`, and `1f7530272` in
sase, and `44dbc91` and `cfe1902` in sase-core. The audit found the defects below. Each
one was introduced by the epic or leaves one of its stated requirements unmet, so it
must be finished before `sase-16z` closes.

Read `sase bead read sase-16z -r "<why>"` and the parent plan
(`plan:202609/usage_window_collection_service_tree.md`) for the design. The most
relevant parts are decision 3, "display freshness uses `max(refresh_seconds, floor)`.
Faster polling must never blank the header as 'unknown'", and decision 5, which covers
cross-repo compatibility.

Rules for every phase:

- In sase, read the `lint_and_test` and `symvision` memories and verify with
  `sase tool run check`. Do not run `just check-full`.
- Work in sase-core only through `sase repo open sase-core`, read its `AGENTS.md`, and
  verify there with `sase tool run check`.
- Read the `tui` memory before touching TUI code. Keep blocking disk and store I/O off
  the UI thread and the Textual message pump.
- `mark_all_message` in `src/sase/ace/tui/modals/plugins_browser_agent_clis_actions.py`
  is also a current symvision failure, but it belongs to epic `sase-171` (recorded there
  as a DISCOVERED ISSUE). Do not fix it here unless it is still failing when your phase
  verifies. If it is, record it as a `PROPOSED FOLLOW-UP:` note rather than treating it
  as your phase's failure.

## Phase landing-fixes — Fix sase-16z landing defects in sase

All changes are in sase, pure Python. Each fix gets a regression test where noted.

1. **Symvision: `resolve_provider_cli_command`.**
   - Symvision reports this `src/sase/llm_provider/usage/_probe_meta.py` symbol as
     unused-public: its only consumers are in-file and tests.
     `src/sase/llm_provider/usage/refresh.py` `_provider_cli_ready` duplicates the same
     resolution: the Codex resolver, the `SASE_<TOKEN>_PATH` override, and
     `autodetect_cli_name`.
   - Give `resolve_provider_cli_command` an optional `metadata` parameter, the
     provider's registry metadata, looked up when omitted as today. Make
     `_provider_cli_ready` call it with the metadata it already has, so there is one
     resolver and it has a real consumer.
   - Remove the imports in `refresh.py` that become unused.
   - Existing tests monkeypatch `_probe_meta.resolve_provider_cli_command` with
     one-argument lambdas. Keep `usage_cli_fingerprint`'s one-argument call working.
2. **Symvision: capability-cache helpers.**
   - In `src/sase/llm_provider/usage/_capability_cache.py`, `capability_cache_dir` and
     `invalidate_probe_capability` are used only in-file and by tests. Privatize both to
     `_capability_cache_dir` and `_invalidate_probe_capability`, drop them from
     `__all__`, and update `tests/llm_provider/test_usage_capability_cache.py`.
   - Test imports of private names are allowed by symvision.
   - Then `just symvision` must list nothing from `_probe_meta.py` or
     `_capability_cache.py`.
3. **agy and grok capability cache never hits in production.**
   - `executable_fingerprint(executable)` only fingerprints a path that `is_file()`. The
     agy and grok probes pass the bare command name, because in production the runner's
     probe context carries no executable: `_AGY_CLI_NAME = "agy"` in `usage/agy.py`, and
     `"grok"` in `usage/grok.py`. So the fingerprint is `None` and the cache is always
     bypassed.
   - Claude is unaffected, because `resolve_claude_executable` returns a `shutil.which`
     path. The existing tests pass absolute fake paths and hide the bug.
   - Fix: when the argument is not an existing file, resolve it with `shutil.which`, as
     the subprocess spawn does, before fingerprinting.
   - Regression test: put a fake `agy`/`grok` executable on `PATH` and pass the bare
     name. Assert that the second probe skips the `--version` spawn.
4. **Rate limits on transport failures.**
   - Plan item 4 of `sase-16z.4` requires every collector to consult `detect_rate_limit`
     (`usage/_strategy.py`) on its failure paths before other classification.
   - codex (`codex_collector.py`), grok (`grok.py`), and muse (`muse.py`,
     `_transport_status`) catch `JsonLineTransportError` and classify it without looking
     at the child's stderr. `JsonLineSession` in `usage/transport.py` already collects
     bounded stderr (`stderr_text()`).
   - Make that stderr available on the transport-failure path, for example by attaching
     it to the raised `JsonLineTransportError`. Consult `detect_rate_limit(stderr=...)`
     first there, returning `error` / `rate_limited` / `retry_after_seconds`.
   - Check agy's timeout path the same way if it has partial output.
   - Tests: a fixture CLI that writes a 429 / "Too Many Requests" line with a
     Retry-After hint to stderr and exits mid-session is classified as `rate_limited`
     for each JSON-line collector.
5. **`429` false positives.**
   - `_RATE_LIMIT_TEXT_PATTERNS` contains `\b429\b`, which also matches decimals such as
     `0.429` in scanned stdout and stderr.
   - Require that the token is not part of a number: no adjacent digit, and no `.`
     followed by a digit on either side. `HTTP 429`, `429 Too Many Requests`, and
     `status 429.` must still match.
   - Add classifier cases for both directions.
6. **Stale chop test runs real provider CLIs.**
   - `tests/llm_provider/test_usage_refresh_runner.py::test_chop_emits_nothing_due_summary`
     monkeypatches `request_due_usage_refresh`. Since `a6e27583c`, the chop calls
     `submit_usage_refresh(..., execution="inline")` instead.
   - So the test now runs real inline probes against whatever CLIs the host has, and
     fails. On athena it reports
     `usage_refresh: providers=3 succeeded=0 failed=1 ... claude=unsupported codex=error grok=unsupported`.
   - `tests/test_axe_chop_usage_refresh.py::test_idle_usage_refresh_reports_nothing_due`
     already covers the nothing-due summary. Delete the stale test, or repoint it at
     `submit_usage_refresh` if it covers something the new file does not.
   - Confirm that no other test in that file reaches a real CLI.
7. **Inline crash path overwrites finished providers.**
   - When `run_admitted_refresh` raises, `refresh._run_inline_batch` calls
     `_record_inline_crash`, which records `error` / `probe_failed` (adaptive) for every
     started provider. That includes providers whose probes already recorded a success
     and released their lease, turning a success into backoff.
   - Record the crash only for providers that still hold a live reservation for this
     operation ID (`list_provider_usage_refresh_reservations`), then release those.
   - The synthesized per-provider results should say `probe_failed` only for those
     providers.
   - Test with a runner that records one success and then raises.
8. **Models panel ends tracking on a failed store read.**
   - In `src/sase/ace/tui/modals/models_panel_usage_modal.py`,
     `_live_refresh_operations` returns `{}` when the reservation read raises. Every
     pending provider then looks finished, and the "updated" toast fires early.
   - Return a distinct failure value, such as `None`, and have the poll keep its pending
     set and retry on the next tick.
   - Test.
9. **Cleanups.**
   - `refresh_runner._run_admitted_refresh`: the post-executor loop's
     `if future.cancel():` branch is dead, because every future is done once the
     executor block exits. Remove it and keep the collect-as-is behavior and its comment
     accurate.
   - `usage/muse.py`: the comments near `_ECHO_PROVIDER_ID` and
     `_POLL_TAIL_MARGIN_SECONDS`, and the `_failure_status` docstring, still say a
     missed mint "degrades to absence". It now reports `error` / `timeout` and keeps the
     last-known-good windows. Update the wording.
   - `tests/llm_provider/test_usage_refresh.py::test_limit_event_trigger_marks_due_and_submits`
     asserts that nothing is submitted. Rename it to match, for example
     `..._marks_due_without_submitting`.
   - `docs/axe.md` (usage routine section): the multi-machine opt-out reads
     `llm_provider.usage_metrics.providers.<name>: false`. The schema
     (`src/sase/config/sase.schema.json`) only allows the object form, and
     `docs/llms.md` documents `providers.<name>.enabled: false`. Use the
     `.enabled: false` form.

## Phase core-indicator-floors — sase-core: per-provider polling floors in the usage indicator projection

This phase is sase-core only.

Background: sase's header indicator (`src/sase/llm_provider/usage/peek.py`
`cached_usage_indicator_projection`) calls the `provider_usage_project_indicator`
binding. The core (`crates/sase_core/src/provider_usage/indicator.rs`,
`project_provider_entries`) recomputes each window's freshness with
`freshness_value(window.observed_at, now, cadence_seconds)` from the single request
cadence. So the header ignores the polling floors that
`load_provider_usage_store_with_floors` already honors.

Example: with `refresh_seconds: 60`, Claude has a 300 s floor and is probed roughly
every 300 s. Its header windows show `stale` after 120 s and `unknown` after 240 s,
while the CLI and Models panel show them `fresh`.

1. `UsageIndicatorProjectionRequestWire` gains
   `#[serde(default)] provider_min_intervals: Option<BTreeMap<String, f64>>`. The wire
   is `deny_unknown_fields`, so the field must be optional and defaulted. Validate it
   with the same rules as the floor-aware store read: `validate_floor_map` in
   `store.rs`. Share that helper rather than duplicate it.
2. In `project_provider_entries`, a provider named in the map uses
   `cadence_seconds.max(floor)` for `freshness_value`, and therefore for the derived
   window attention. Every other provider uses `cadence_seconds` as today.
3. Tests:
   - a golden legacy test: without the field, the projection is identical to today's;
   - a floored provider's window at an age between `2 × cadence` and `2 × floor` stays
     `fresh`, while an unfloored provider's window at the same age is `stale`;
   - invalid floors (below 60, non-finite, over 86 400, bad provider ident) are
     rejected;
   - a binding test in `crates/sase_core_py`.
4. Versioning: follow `AGENTS.md`. This is an additive, request-only optional field that
   older sase never sends, so it is not a breaking change. Use a `feat(core):` commit.

## Phase header-floor-freshness — Floor-aware freshness for the TUI header usage indicator

1. Once the `core-indicator-floors` commit is on sase-core's remote HEAD, move the pin
   with `just ratchet-core-revision`. Run `just install`, and confirm that the
   pinned-bindings check passes. Never edit the `sase-core-rs` window in
   `pyproject.toml`.
2. `src/sase/llm_provider/usage/_facade.py` `provider_usage_project_indicator` accepts
   `provider_min_intervals: Mapping[str, float] | None = None`. It adds the key to the
   request only when it is provided.
3. `src/sase/llm_provider/usage/peek.py`:
   - `refresh_usage_peek_cache` already runs off the UI thread. Have it capture
     `usage_probe_floors()` into the peek cache next to the snapshot, metrics, and
     indicator settings. Clear the floors in `_clear_usage_peek_cache`.
   - `cached_usage_indicator_projection` runs on the render path and "never reads disk".
     Pass the captured floors. Do not call `usage_probe_floors()` there, because a
     config-token miss reloads registry metadata.
4. Tests:
   - With `refresh_seconds: 60`, a Claude window observed 250 s ago (floor 300) projects
     `fresh` in the header. A Codex window (floor 120) observed 250 s ago projects
     `stale`. A provider without a floor keeps the bare-cadence behavior.
   - The render-path projection performs no floor lookup.
5. Docs: in `docs/configuration.md`, next to the existing floor/freshness text by
   `refresh_seconds`, state that the TUI header indicator uses the same
   `max(refresh_seconds, floor)` freshness as `sase usage list` and the Models panel.
