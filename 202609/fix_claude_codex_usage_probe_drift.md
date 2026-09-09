---
tier: tale
title: Fix Claude and Codex subscription-usage probes broken by vendor CLI drift
goal: sase usage collects Claude and Codex subscription windows reliably again, and
  future probe failures carry bounded diagnostics naming the actual cause.
size: medium
proposed_by: bbugyi200.athena.0hd
status: done
---

# Fix Claude and Codex subscription-usage probes broken by vendor CLI drift

## Problem

`sase usage` reports `error: probe failed` for both `claude` and `codex` (grok is
healthy). Both failures are deterministic — every probe fails, 100% of the time — and
were diagnosed live on 2026-09-09 against the installed CLIs. The feature only _looks_
intermittently reliable because passive Claude stream-event observations (recorded from
running Claude Code sessions) sometimes mask the always-failing Claude probe with fresh
windows, while Codex has no passive source and always shows the error.

### Root cause 1 — Codex: `account/rateLimits/read` params rejected

`collect_codex_usage()` in `src/sase/llm_provider/usage/codex_collector.py` sends:

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "account/rateLimits/read",
  "params": { "excludeResetCreditDetails": true }
}
```

codex-cli **0.153.4** (`~/.config/nvm/versions/node/v22.14.0/bin/codex`, resolved by
`resolve_codex_executable()`) declares this method's params as unit and rejects the
request:

```json
{
  "error": {
    "code": -32600,
    "message": "Invalid request: invalid type: map, expected unit"
  },
  "id": 3
}
```

`_observation_from_error()` does not recognize `-32600` (only `-32601` and
unauthenticated markers), so the observation degrades to the generic
`outcome=error, reason_code=probe_failed`.

Verified live against the same app-server session: sending the request with **no
`params` key**, with `"params": null`, or with `"params": {}` all succeed and return a
full result with `rateLimitsByLimitId` containing bucket `codex` (weekly window only:
`primary.usedPercent` 48, `windowDurationMins` 10080, `secondary` null) and bucket
`codex_bengalfox` (`limitName` "GPT-5.3-Codex-Spark", primary + secondary). Only the
non-empty params map fails. `secondary: null` is already handled (`_window_from_slot`
returns `None` for non-dict slots). The response includes a `rateLimitResetCredits`
block that the parser already ignores; its size is a few KB, far under the transport
line/total bounds, so losing `excludeResetCreditDetails` costs nothing.

The `initialize` / `initialized` handshake and best-effort `account/read` both work
(`account/read` returns `{"account":{"type":"chatgpt","planType":"pro",...}}`, and the
existing `_extract_auth_mode_hint` maps `chatgpt` correctly).

### Root cause 2 — Claude: `--max-budget-usd 0` now rejected

`usage_probe_argv()` in `src/sase/llm_provider/usage/_claude_support_command.py` builds
the guarded probe with `--max-budget-usd 0`. Claude Code **2.1.266** (installed at
`~/.local/bin/claude`) tightened flag validation:

```
error: option '--max-budget-usd <amount>' argument '0' is invalid. --max-budget-usd must be a positive number greater than 0
```

The command exits rc=1 with empty stdout; `collect_claude_usage()` maps the nonzero
return code to `outcome=error, reason_code=probe_failed`. All preflights still pass
(2.1.266 ≥ min 2.1.263; `-p --help` contains every `print_help_supports_zero_cost_probe`
marker; `auth status --json` reports `loggedIn: true` subscription auth in ~0.2s).

Verified live: the identical argv with `--max-budget-usd 0.01` succeeds in ~2.2s (rc=0)
and the JSON payload proves the probe was still free — `num_turns: 0`,
`total_cost_usd: 0` — so `has_zero_cost_markers()` accepts it, and the `result` text
parses into session/weekly/model windows via `parse_usage_windows()`.

### Root cause 3 — failures are indistinguishable, so drift goes unnoticed

Every failure path collapses into `diagnostic: "usage probe failed"`:

- Codex discards the JSON-RPC error code/message in `_observation_from_error()`.
- Claude discards the CLI return code and stderr in the `collect_claude_usage()` failure
  branches.
- The isolated worker's stderr is logged only at DEBUG and only as a byte count
  (`_log_worker_stderr` in `src/sase/llm_provider/usage/probe.py`); when worker stdout
  is empty the observation says nothing about why.

This is why two independent vendor regressions read as one vague "probe failed" and why
the tests never caught them: the codex test fixture
(`tests/llm_provider/fixtures/usage_probe/codex_app_server_cli.py`) accepts any params
on `account/rateLimits/read`, and the claude fake runner accepts a `0` budget.

## Fix

### Step 1 — Codex: send `account/rateLimits/read` with no params

In `src/sase/llm_provider/usage/codex_collector.py`:

- Send the `account/rateLimits/read` request **without a `params` key** (matches unit
  params on codex-cli 0.153.4; older/newer servers that accept a map also accept absent
  params, per JSON-RPC).
- Forward-compat guard: if the paramless call returns an invalid-request/invalid-params
  error (`-32600` or `-32602`), retry **once** with the legacy
  `{"excludeResetCreditDetails": true}` params on a new request id before degrading to
  an error observation. Keep the retry inside the existing session and deadline.
- Update the stale code comment about `excludeResetCreditDetails` to record the verified
  0.153.4 behavior.

### Step 2 — Claude: use the minimum positive probe budget

In `src/sase/llm_provider/usage/_claude_support_command.py`:

- Change the `usage_probe_argv()` budget from `"0"` to `"0.01"`, named as a module
  constant (e.g. `CLAUDE_USAGE_PROBE_BUDGET_USD = "0.01"`).
- Update the docstrings that call it a "zero-budget" argv: the guard is now a one-cent
  cap; the zero-cost _proof_ remains `has_zero_cost_markers()` (unchanged), which
  rejects any observation whose payload shows nonzero turns or cost.
- Do **not** bump `CLAUDE_USAGE_MIN_VERSION` (2.1.263): every CLI in the supported range
  accepts a positive budget; only `0` became invalid.

### Step 3 — Carry bounded diagnostics on probe failures

Make future drift self-describing in `sase usage list -v` instead of "usage probe
failed". All diagnostics must be single-line, length-bounded (~200 chars), and built
from data the probe already holds — never from environment or auth material:

- Codex `_observation_from_error()`: include the RPC method, error code, and truncated
  message, e.g. `app-server account/rateLimits/read error -32600: Invalid request: ...`.
  Apply the same to the `initialize`-error branch in `_collect_with_session()`.
- Claude `collect_claude_usage()` nonzero-returncode and `"failed"` branches: include
  the exit code and a truncated first stderr (falling back to stdout) line, e.g.
  `claude /usage exited 1: error: option '--max-budget-usd <amount>' ...`.
- Worker path in `src/sase/llm_provider/usage/probe.py`: when worker stdout is empty
  (`_observation_from_worker_stdout` probe_failed branch), attach a truncated stderr
  snippet to the diagnostic, and raise `_log_worker_stderr` to WARNING with a bounded
  snippet when the probe produced no usable stdout. Reuse one shared truncation helper.
- Check `validate_observation()` / `validated_status_observation()` in
  `src/sase/llm_provider/usage/types.py` for any diagnostic length limit and stay within
  it.

### Step 4 — Tests that would have caught this

- `tests/llm_provider/fixtures/usage_probe/codex_app_server_cli.py`: mirror the live CLI
  in the default modes — respond
  `{"error":{"code":-32600,"message":"Invalid request: invalid type: map, expected unit"}}`
  to `account/rateLimits/read` when the request carries a non-empty `params` map, and
  succeed when params are absent/null/empty. Add an env-driven mode for the opposite
  server (paramless fails with `-32600`, legacy params succeed) to cover the Step 1
  retry.
- `tests/llm_provider/test_codex_usage_probe.py`: assert the collector's request omits
  params and succeeds against the strict fixture; add a retry-path test using the new
  mode; add a test asserting the diagnostic on an RPC error carries the code/message.
- `tests/llm_provider/test_claude_usage.py`: update the argv assertion (currently
  `argv[argv.index("--max-budget-usd") + 1] == "0"`) to the new constant, and make the
  fake runner reject a literal `0` budget with the live CLI's rc=1 option error so a
  regression to `0` fails the suite. Add a diagnostic assertion for the
  nonzero-returncode branch.
- Keep every fixture/runner change consistent with the existing env-driven-mode
  conventions in those files.

## Out of scope

- No new feature flags, CLI subcommands/options, or wire-schema changes; observation
  schema and `sase_core_rs` store bindings are untouched (windows/observations keep the
  same shape).
- No change to refresh scheduling, passive stream-event capture, or the grok collector.
- No `CLAUDE_USAGE_MIN_VERSION` bump.

## Verification

1. Targeted tests:
   `.venv/bin/python -m pytest tests/llm_provider/test_codex_usage_probe.py tests/llm_provider/test_claude_usage.py tests/llm_provider/test_usage_probe.py -q`
2. Live end-to-end on this host (both CLIs installed and authenticated):

   ```bash
   .venv/bin/python -c "
   from sase.llm_provider.usage.probe import run_usage_probe, default_probe_context
   for p in ('claude', 'codex'):
       obs = run_usage_probe(default_probe_context(p)).observation
       print(p, obs['outcome'], obs.get('reason_code'), len(obs['windows']))
   "
   ```

   Expected: both providers report `ok` with nonzero window counts (before the fix:
   `error probe_failed 0` for both).

3. `sase usage refresh -p claude -p codex` followed by `sase usage list -v`: both rows
   show window data with `collection_status: ok`; the codex row shows the weekly `codex`
   bucket (~48% used at diagnosis time) and the `codex_bengalfox` buckets.
4. Read the `lint_and_test` sase memory note and run the verification it mandates for
   tracked-file changes (at minimum `just check`).
