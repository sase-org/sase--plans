---
tier: epic
title: Antigravity (agy) subscription usage windows and default header indicator
goal: "SASE collects Antigravity's four subscription usage windows on the normal
  background cadence at zero model cost, and the top-right TUI header shows agy's Gemini
  weekly window by default (for example `🪐 96% 6d1h`) whenever agy is installed and in
  use, proven by a live post-landing screenshot saved as a SASE artifact.

  "
phases:
  - id: core-normalizer
    title: Rust normalizer and Gemini weekly anchor rule
    depends_on: []
    size: medium
    description:
      "core-normalizer: in sase-core, add agy.rs to normalize the `agy -p /usage
      --output-format json` envelope into honest model_family-scoped windows with the
      hardening guards (omitted-zero TSV cross-check, turn-ran guard, logged-out
      envelope), add the narrow agy arm to is_weekly_all_window, export and bind
      provider_usage_normalize_agy_usage, and cover it with fixtures and tests."
  - id: agy-collector
    title: Hardened agy usage collector and provider hooks
    depends_on:
      - core-normalizer
    size: medium
    description:
      "agy-collector: add collect_agy_usage (version floor, own process group, stderr
      auth-prompt kill, deadline margins, no auto-update, private log file), wire
      AgyProvider usage hooks, raise the sase-core-rs floor and ratchet the core
      revision, add a scripted fake-agy test fixture, and document the collector."
  - id: header-default
    title: Header naming polish, default config, and snapshots
    depends_on:
      - agy-collector
    size: small
    description:
      "header-default: render model_family scopes as the bare family in compact header
      names only, add commented agy indicator examples to default_config.yml, pin the
      shipped-default agy header outcome in tests, and add agy visual snapshot coverage."
proposed_by: bbugyi200.athena.0os
create_time: 2026-09-21 15:31:49
status: wip
---

- **PROMPT:**
  [prompts/202609/agy_usage_windows.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/agy_usage_windows.md)

# Plan: Antigravity (agy) subscription usage windows and default header indicator

## Goal

Antigravity (`agy`) is a registered provider with no subscription-usage hooks. This epic
adds a zero-cost collector for its usage windows and makes its Gemini weekly window show
by default in the TUI's top-right usage indicator. The target default rendering is
`🪐 96% 6d1h`. Under 5-hour pressure it becomes `🪐 96% 6d1h · 5h/gemini 12% 4h50m`.

## Background

The design follows the consolidated research report
`research:202609/agy_usage_windows/agy_usage_windows.md`. Read it with
`sase artifact read research:202609/agy_usage_windows/agy_usage_windows.md "<why>"`
before starting any phase. Its §3 (hazards H1–H7) and §4 (applicability) matter most.

The vendor side is already solved. `agy >= 1.1.11` answers `/usage` in print mode
without starting a model turn. It was re-verified while authoring this plan against
`agy 1.2.7`:

- Command:
  `AGY_CLI_DISABLE_AUTO_UPDATE=1 agy -p /usage --output-format json --mode plan --sandbox --print-timeout 15s --log-file <tmp>/agy-usage.log`
  with stdin `/dev/null`.
- Result: exit 0 in 3.5 s, `num_turns: 0`, `conversation_id: ""`, and every `usage`
  token count 0.

The captured payload is the canonical success fixture. Store it verbatim as a fixture in
phase `core-normalizer`:

```json
{
  "conversation_id": "",
  "status": "SUCCESS",
  "response": "Gemini Models\tWeekly Limit Remaining\t96%\t2026-09-27T20:14:34Z\nGemini Models\tFive Hour Limit Remaining\t87%\t2026-09-21T23:31:12Z\nClaude and GPT models\tWeekly Limit Remaining\t100%\t2026-09-28T19:16:26Z\nClaude and GPT models\tFive Hour Limit Remaining\t100%\t2026-09-22T00:16:26Z\n",
  "duration_seconds": 0,
  "num_turns": 0,
  "usage": {
    "input_tokens": 0,
    "output_tokens": 0,
    "thinking_tokens": 0,
    "cache_read_tokens": 0,
    "total_tokens": 0
  },
  "command": {
    "name": "usage",
    "data": {
      "description": "Within each group, models share a weekly limit and a 5-hour limit. ...",
      "groups": [
        {
          "name": "Gemini Models",
          "description": "Models within this group: Gemini Flash, Gemini Pro",
          "buckets": [
            {
              "id": "gemini-weekly",
              "name": "Weekly Limit Remaining",
              "description": "You have used some of your weekly limit, it will fully refresh in 6 days.",
              "window": "weekly",
              "remaining_fraction": 0.9612414240837097,
              "reset_time": "2026-09-27T20:14:34Z"
            },
            {
              "id": "gemini-5h",
              "name": "Five Hour Limit Remaining",
              "description": "You have used some of your 5-hour limit, it will fully refresh in 4 hours, 14 minutes.",
              "window": "5h",
              "remaining_fraction": 0.8664969801902771,
              "reset_time": "2026-09-21T23:31:12Z"
            }
          ]
        },
        {
          "name": "Claude and GPT models",
          "description": "Models within this group: Claude Opus, Claude Sonnet, GPT-OSS",
          "buckets": [
            {
              "id": "3p-weekly",
              "name": "Weekly Limit Remaining",
              "window": "weekly",
              "remaining_fraction": 1,
              "reset_time": "2026-09-28T19:16:26Z"
            },
            {
              "id": "3p-5h",
              "name": "Five Hour Limit Remaining",
              "window": "5h",
              "remaining_fraction": 1,
              "reset_time": "2026-09-22T00:16:26Z"
            }
          ]
        }
      ]
    }
  }
}
```

(Keep the full `command.data.description` string in the stored fixture; it is elided
here only for width.)

Everything downstream of a collector is already provider-generic: the store, refresh
scheduler, eligibility, `sase usage`, the Providers / Usage modal, the header indicator,
and model hints. The `🪐` badge (`src/sase/integrations/provider_badges.py`) and the agy
palette also exist. agy becomes background-eligible as soon as it declares
`probe: True`. It is referenced by the shipped `@xsmall` pool
(`src/sase/llm_provider/model_alias_defaults.yml`), and `_provider_cli_ready` in
`src/sase/llm_provider/usage/refresh.py` honours `SASE_AGY_PATH` or an `agy` on `PATH`.
`UsageHeader` (`src/sase/ace/tui/widgets/usage_header.py`) already docks the usage
cluster at the top right. **No layout change is needed for "top-right". Confirm that
before touching layout code, and change none if it holds.**

This epic follows the structure of the Muse usage-window epic (`sase-14c`; plan
`plan:202609/muse_usage_windows.md`; commits sase-core `3cae8ef` and sase `608640272`).
Use those as the working template for file layout, bindings, fixtures, and tests.

## Decisions

These are judgement calls. A reviewer who disagrees should say so at the plan gate.

1. **Honest `model_family` scoping, not `account`.** The vendor states that each group
   has its own weekly and 5-hour limit. Tagging Gemini buckets `account` would make
   model-picker and alias hints report `agy/claude-opus-4-6-thinking` as exhausted
   whenever Gemini is exhausted. It would also size a Claude-via-agy usage-limit disable
   from a Gemini reset. So `gemini-*` buckets are `model_family` `gemini` and `3p-*`
   buckets are `model_family` `3p`.
2. **One narrow core anchor arm makes the header work.** With honest scoping and no core
   change, the header shows nothing. `is_weekly_all_window` is false for every agy
   window, so none is the unlabeled anchor or matches `weekly_all: always`. The fix is
   one arm in `is_weekly_all_window` only. It matches `provider == "agy"`,
   `key == "gemini-weekly"`, and `model_family` `gemini`. `is_all_model_scope` and
   `classify_scope` stay untouched, so tooltips and the Models panel still say
   `family: gemini`. This is the fourth `provider == "…"` allowlist arm. Replacing that
   allowlist is task bead `sase-14i` and stays out of scope.
3. **The normalizer lives in Rust.** This follows the Grok/Muse precedent, the
   `rust_core_backend_boundary` core memory, and decision `rust-core-required`. Grok's
   omitted-zero bug (`db535fabd`) is the reason the H3 trap belongs to core. The
   subprocess, version gate, and stderr watch stay in Python as process I/O glue, like
   `usage/muse.py`.
4. **Gemini 5-hour and `3p-*` windows stay on the generic default:** shown only below
   20% remaining. This matches the vendor's framing of 5 h as a global-demand smoother.
   If agy is never routed to Claude/GPT, the `3p-*` windows sit at 100% and stay hidden.
   Unlike Muse, no `never` entry ships. Commented examples show how to pin or hide them.
5. **Compact header names drop the `family:` prefix** (`5h/gemini`, not
   `5h/family:gemini`). This saves 7 cells per badge in a budget-constrained header.
   Tooltips, the Models panel, and `sase usage list` keep `family:gemini`. agy is the
   first and only `model_family` emitter, so nothing else changes.
6. **Header group order stays provider-name order.** `agy` sorts first, becomes the
   provider a click on the cluster opens, and can push one more provider into `+N`
   overflow at narrow widths (Codex dropped out at 40 cells in research). Accept this;
   ordering by `llm_autodetect_priority` is a possible later change, not part of this
   epic.
7. **No feature flag.** The user-facing switch is the existing permanent config
   `llm_provider.usage_metrics.providers.agy`. Each phase lands a complete, coherent
   surface. Phase `core-normalizer` exposes nothing on its own.

## Phase 1: Rust normalizer and Gemini weekly anchor rule (`core-normalizer`)

All of this lands in the `sase-core` repo. Open it with the `/sase_repo` skill
(`sase repo open sase-core -r "<why>"`) and edit only through the printed path.

### Normalizer

Add `crates/sase_core/src/provider_usage/agy.rs`, modelled on the sibling `muse.rs`. Use
the same versioned request wire, the same `schema_version` check against
`PROVIDER_USAGE_OBSERVATION_SCHEMA_VERSION`, the same `validate_usage_observation` exit,
and the same fallback when vendor-derived values fail validation.

```rust
#[serde(deny_unknown_fields)]
pub struct ProviderUsageNormalizeAgyUsageRequestWire {
    pub schema_version: u32,
    pub payload: Value,          // the whole `--output-format json` stdout object
    pub model_ids: Vec<String>,  // AgyProvider's known model catalog (bare ids)
    pub provider: String,
    pub context_id: String,
    pub account_generation: u64,
    pub request_started_at: f64,
    pub now: f64,
}

pub fn normalize_agy_usage(
    request: ProviderUsageNormalizeAgyUsageRequestWire,
) -> Result<ProviderUsageObservationWire>
```

Decision order for the envelope:

1. **Not an object** → structured error observation, `malformed_payload`.
2. **`status != "SUCCESS"`**:
   - If the `error` text mentions authentication (the logged-out shape is
     `{"status":"ERROR","error":"authentication failed or timed out",…}`), return
     outcome `unauthenticated` with reason `logged_out`.
   - Otherwise return `error` / `probe_failed` with a bounded diagnostic.
3. **Turn-ran guard (H1).** `num_turns` present and non-zero, a non-empty
   `conversation_id`, or `command.name != "usage"` → `error` / `vendor_drift` with a
   named diagnostic. This is the post-hoc backstop behind the Python version gate. It
   means `/usage` was treated as a prompt.
4. **Buckets → windows.** One window per `command.data.groups[].buckets[]`:

   | Observation field  | Derivation                                                                                      |
   | ------------------ | ----------------------------------------------------------------------------------------------- |
   | `key`              | bucket `id` verbatim (`gemini-weekly`, `gemini-5h`, `3p-weekly`, `3p-5h`)                       |
   | `label`            | `"<group name> <bucket name>"`, e.g. `Gemini Models Weekly Limit Remaining`                     |
   | `used_percent`     | `clamp((1 − remaining_fraction) × 100, 0, 100)`                                                 |
   | `resets_at`        | `reset_time` parsed as RFC 3339 with `chrono` (as `grok.rs` does); `None` if absent             |
   | `duration_seconds` | from `window`: `weekly` → 604800, `5h` → 18000, generic `<N>h` / `<N>d`, anything else → `None` |
   | `period_start`     | `resets_at − duration_seconds` when both are known, else `None`                                 |
   | `applicability`    | see below                                                                                       |
   | `vendor_state`     | `Allowed`                                                                                       |
   | `source`           | `Probe`                                                                                         |

   `remaining_fraction` must accept JSON integers (`1`) as well as floats. It must be
   finite and in `[0, 1]`; anything else is malformed. Pass unstarted buckets' rolling
   resets (request time + window) through unchanged. Never derive burn rates across
   snapshots.

5. **Applicability.**
   - `gemini-*` ids → `ModelFamily { family: "gemini", model_ids }`, where `model_ids`
     are the request catalog ids that start with `gemini-`.
   - `3p-*` ids → `ModelFamily { family: "3p", model_ids }`, where `model_ids` are the
     remaining catalog ids.
   - Any other id → `Unknown { vendor_label: <group name>, vendor_id: <bucket id> }`.
     Enterprise and business accounts likely use other group names; they must degrade,
     not fail.
6. **Omitted fraction (H3).** The vendor's struct tags are `omitempty`, so an
   **exhausted** bucket may arrive with no `remaining_fraction`. Never skip such a
   bucket. Cross-check the payload's `response` TSV row with the same group name and
   bucket name (`<group>\t<bucket>\t<status>\t<reset>`):
   - `0%` → `used_percent: 100`.
   - A non-percent status (the 1.0.8-era `Disabled`) → omit that window and add a
     diagnostic.
   - Neither matches → return the observation with `completeness: Partial` and a
     `malformed_payload`-class diagnostic.
7. **Empty `groups`** → an authoritative-empty `Ok` observation with a named diagnostic.
   It is never `0%` and never an error.
8. **Envelope for success:** `outcome: Ok`, `completeness: Complete`,
   `account_mode: Some("subscription")`, `plan: None`. Never persist the tier string.

A malformed binding _request_ (wrong schema version, unknown fields) still returns
`ProviderUsageError::Validation`. Malformed _vendor data_ becomes a structured error
observation.

### Anchor rule

In `crates/sase_core/src/provider_usage/indicator.rs`, extend `is_weekly_all_window`
only:

```rust
fn is_weekly_all_window(provider: &str, window: &UsagePublicWindowWire) -> bool {
    (is_all_model_scope(provider, window) && is_weekly_window(provider, window))
        // agy's Gemini weekly window is honestly model_family-scoped, but it is the
        // provider's headline window: make it the header anchor without claiming
        // an all-model scope anywhere else.
        || (provider == "agy"
            && window.key == "gemini-weekly"
            && matches!(&window.applicability,
                UsageApplicabilityWire::ModelFamily { family, .. } if family == "gemini")
            && is_weekly_window(provider, window))
}
```

Do not touch `is_all_model_scope` or `classify_scope`.

### Exports and binding

- Re-export `normalize_agy_usage` and its request wire from `provider_usage/mod.rs` and
  `crates/sase_core/src/lib.rs`, following the Muse lines.
- Add `py_provider_usage_normalize_agy_usage` in `crates/sase_core_py/src/lib.rs`,
  exposed as `provider_usage_normalize_agy_usage`. Add it to that file's module-level
  binding-inventory doc comment next to the Muse entry, and add a Python round-trip
  binding test like the Muse one.

### Fixtures and tests

Put fixtures wherever `crates/sase_core/src/provider_usage/fixtures` already keeps Grok
and Muse ones. Cover:

- the verified live payload above, asserting all four windows, keys, `model_family`
  families, filtered `model_ids`, `duration_seconds`, `period_start`, and `plan: None`;
- an integer `1` fraction;
- both omitted-zero variants (TSV `0%` → 100% used; TSV `Disabled` → window omitted with
  diagnostic) and the no-TSV-match partial case;
- the logged-out `ERROR` envelope → `unauthenticated` / `logged_out`;
- a non-auth `ERROR` envelope → `error` / `probe_failed`;
- a turn-ran envelope (`num_turns: 1`, non-empty `conversation_id`) → `vendor_drift`;
- an unknown group/bucket id → `Unknown` applicability, still `Ok`;
- an unstarted bucket (no `description`, rolling reset) passing through;
- empty `groups` → authoritative empty;
- an out-of-range fraction → malformed;
- wrong `schema_version` and unknown request fields → validation error;
- **projection tests**:
  - `gemini-weekly` is the only agy `weekly_all` window and becomes the unlabeled
    anchor;
  - `3p-weekly` is weekly but not `weekly_all`;
  - `classify_scope` still reports `ModelFamily` for `gemini-weekly`;
  - the arm does not fire for a non-agy provider or a non-`gemini` family.

### Verification and landing

Run `just check` from the `sase-core` repo root; it wraps `./scripts/check.sh all`.
Never verify with `cargo test -p sase_core` alone, because it excludes the
`sase_core_py` binding tests. Do not hand-edit crate versions. release-plz owns them and
publishes a new `sase-core-rs` once this lands on master.

Record `PROPOSED FOLLOW-UP:` notes on this phase bead for:

- the fourth allowlist arm, so the lander can corroborate `sase-14i`;
- a real exhausted payload capture, once one occurs.

## Phase 2: Hardened agy usage collector and provider hooks (`agy-collector`)

This lands in the `sase` repo and consumes the phase 1 binding.

### Collector

Add `src/sase/llm_provider/usage/agy.py` exporting
`collect_agy_usage(context: UsageProbeContext, executable: str | None = None)`.

1. **Executable.** Use `executable or context.executable or _agy_bin()`, which honours
   `SASE_AGY_PATH` (`src/sase/llm_provider/agy.py`). `FileNotFoundError` → `error` /
   `not_installed`.
2. **Version floor (H1).** Run `<agy> --version`, which prints a bare `1.2.7` in about
   0.1 s, with a short timeout. Below `1.1.11`, or unparseable, return `unsupported` /
   `unsupported_cli_version` and **never spawn `/usage`**. Older builds would run
   `/usage` as a real paid agent turn every tick.
3. **Spawn.**
   - argv:
     `<agy> -p /usage --output-format json --mode plan --sandbox --print-timeout 15s --log-file <cwd>/agy-usage.log`
   - `cwd=context.working_directory` (the probe's managed temp dir, which also keeps the
     log file out of `~/.gemini/…`, H5).
   - `stdin=DEVNULL`, `start_new_session=True` (its own process group).
   - env: the worker's already-filtered `os.environ` plus
     `AGY_CLI_DISABLE_AUTO_UPDATE=1` (H4). Do not hand-roll an env (H6).
   - Use a plain `subprocess.Popen`; this is one request and one response, so
     `JsonLineSession` is unnecessary.
4. **Auth-prompt kill (H2).** Logged-out print mode prints `Authentication required…` to
   stderr and then blocks on an OAuth paste prompt. `--print-timeout` does **not** bound
   that wait. Read stderr incrementally on a reader thread. On the marker, kill the
   whole process group and return `unauthenticated` / `logged_out` promptly, well under
   the deadline.
5. **Deadline.** Bound everything by the probe context's deadline with the same margins
   as Muse's `_subprocess_deadline` (`usage/muse.py`). On expiry, kill the process group
   (`os.killpg`) and return `error` / `timeout`. Catch `subprocess.TimeoutExpired`, not
   `TimeoutError`: they are unrelated classes, and the research found that exact bug in
   a draft. If you share the deadline helper rather than mirroring it, promote it to a
   public helper; do not import a private name across modules (Symvision).
6. **Normalize.** `json.loads(stdout)`; failure → `error` / `parse_error`. Otherwise
   call `require_rust_binding("provider_usage_normalize_agy_usage")` exactly as
   `usage/muse.py` and `usage/grok.py` do. Take `model_ids` from
   `AgyProvider().llm_known_model_names()`. Return the observation. A non-zero exit
   whose stdout is a JSON envelope still goes to the normalizer, which owns the `ERROR`
   mapping.
7. Use `validated_status_observation` / `bounded_probe_diagnostic` (`usage/types.py`)
   for every non-normalizer status, like Muse.

### Provider hooks

In `src/sase/llm_provider/agy.py`:

```python
@hookimpl
def llm_usage_capabilities(self) -> dict[str, object]:
    return {"probe": True, "passive_events": False}

@hookimpl
def llm_usage_probe(self, context: UsageProbeContext) -> dict[str, object] | None:
    from .usage.agy import collect_agy_usage

    return collect_agy_usage(context, executable=context.executable or _agy_bin())
```

### Core version floor and pin

`tools/check_sase_core_rs_bindings` checks that the **minimum published** `sase-core-rs`
accepted by `pyproject.toml` exposes every binding named by `require_rust_binding`.
Local `just install` builds core from the sibling checkout and will not notice skew; CI
will. Master Gate builds core at `sase-core-revision.txt`. Therefore:

- Confirm phase 1 is on `sase-core` master and that release-plz has published a
  `sase-core-rs` containing `provider_usage_normalize_agy_usage`. If it has not
  published yet, wait with `/sase_monitor`. Never pin an unpublished floor.
- Raise the `sase-core-rs>=…` floor in `pyproject.toml` to that version, keeping the
  upper bound, and refresh `uv.lock` the way `608640272` did.
- Move `sase-core-revision.txt` forward with `just ratchet-core-revision`. Run `--check`
  or `--report-only` first; exit 0 means already current, 2 means a ratchet is pending,
  reported, or applied, and 3 means it could not be determined safely.

### Tests

Add a scripted fake `agy`, `tests/llm_provider/fixtures/usage_probe/agy_usage_cli.py`,
modelled on `muse_msp_cli.py`. It should answer `--version`, switch behaviour on a
scenario environment variable, and append every argv it receives to a log the test can
read. Add `tests/llm_provider/test_agy_usage_probe.py` covering:

- success (four windows through the real binding);
- old version (`1.0.10`) → `unsupported_cli_version`, **asserting from the argv log that
  `/usage` was never spawned**;
- auth prompt: the fake writes `Authentication required` to stderr and then sleeps.
  Assert `logged_out` and that the call returns well under the deadline, and that the
  child process group is gone;
- timeout (fake sleeps silently) → `timeout`, child killed;
- malformed stdout → `parse_error`;
- omitted-zero payload with a `0%` TSV row → 100% used;
- turn-ran envelope → `vendor_drift`;
- missing executable → `not_installed`;
- `AgyProvider.llm_usage_capabilities()` returns
  `{"probe": True, "passive_events": False}`, and agy appears in
  `eligible_usage_providers()` when its CLI is resolvable. Follow
  `tests/llm_provider/test_usage_eligibility.py`.

### Docs

- `docs/agent_providers.md`, Antigravity CLI section: add a "Subscription usage"
  subsection like Muse's. It should cover the zero-token `/usage` probe, ~3–5 s per
  refresh, `agy >= 1.1.11`, the four windows and their `model_family` scoping, and the
  `llm_provider.usage_metrics.providers.agy` switch.
- Add agy to the collector lists in `docs/llms.md` and `docs/configuration.md`.
- Do not hand-edit `CHANGELOG.md`; release-please generates it.

### Live check

agy is installed on this host. After the tests pass, run one real
`sase usage refresh -p agy` from the workspace environment. Confirm with
`sase usage list -p agy -v` that four windows land with populated resets and zero token
spend.

## Phase 3: Header naming polish, default config, and snapshots (`header-default`)

Read `sase/memory/tui.md` and its children with the `/sase_memory_read` skill before
touching TUI code.

### Compact-name polish

In `src/sase/ace/tui/widgets/_usage_indicator_format.py`, `_usage_specifier` calls
`_scope_suffix(scope)` for both compact and full names. Make the compact form render
`model_family` as the bare family (`gemini`, `3p`), for example by passing `compact`
through or post-processing the suffix in the `if compact:` block as is already done for
`all`. The full form keeps `family:gemini`. Leave `_scope_tooltip_text` and
`usage/presentation.py` unchanged. Expected compact names:

- `gemini-5h` → `5h/gemini`
- `3p-5h` → `5h/3p`
- `3p-weekly` → `3p` (weekly periods are already omitted in compact names)

The anchor `gemini-weekly` renders unlabeled. Unit-test the compact and full forms.

### Default config

In `src/sase/default_config.yml`, under
`llm_provider.usage_metrics.indicator.providers`, ship **no** active agy entry (decision
4). Add a comment explaining that agy's Gemini weekly window is the header anchor
through `weekly_all: always`. Add a commented example to the existing commented
`providers:` block that pins `gemini-5h: always` and hides `3p-weekly` / `3p-5h` with
`never`. Keep the neighbouring commented examples accurate. Because no active default
changes, `src/sase/config/sase.schema.json` needs no `default` update. Confirm this and
leave the schema alone if it holds.

### Shipped-default tests

Add `tests/llm_provider/test_agy_usage_indicator_default.py`, modelled on
`test_muse_usage_indicator_default.py`, driving the shipped `default_config.yml` through
the real projection and `usage_indicator_groups` / `build_usage_indicator_segment`.
Assert:

- a live-shaped snapshot (96% / 87% / 100% / 100% remaining) renders the agy group as
  just the unlabeled Gemini weekly anchor (`🪐 96% …`);
- Gemini 5h at 12% remaining adds `5h/gemini 12% …`;
- `3p-*` at 100% stay hidden;
- `3p-5h` below 20% remaining appears as `5h/3p`;
- no rendered segment contains `family:`;
- installed-only: with no agy CLI resolvable, agy is not eligible and no agy group
  renders.

### Visual snapshots

Add agy coverage to
`tests/ace/tui/visual/test_ace_png_snapshots_provider_usage_indicator.py` using the
existing `real_projection` / `usage_provider` fixture helpers. Include an agy group next
to at least one other provider at 60, 80, and 140 columns, with a Gemini 5h pressure
case. Run `just fix-tui-screenshots -- <selectors>` explicitly, because `just check`
does not run PNG snapshots. Inspect the report and every golden creation or update
before accepting; generation is not approval. Use `/sase_monitor` with `TESTING` /
`TESTED` if the run is long.

## Verification (all phases)

- Every phase that changes git-tracked files in the `sase` repo runs `just check`
  (prefer `sase tool run check`) before finishing, per `sase/memory/lint_and_test.md`.
  Run `just install` first if the workspace virtualenv is stale, and `just fix` before
  handing a long run to a monitor. No phase in this plan names `just check-full`; do not
  run it.
- Phase `core-normalizer` runs `just check` in the `sase-core` repo instead, which is a
  different recipe.

## Landing requirements (epic land agent)

These are in addition to the standard land-agent steps. **The epic bead must not be
closed, and this plan's `status` must not be set to `done`, until a screenshot proving
success has been saved as a SASE artifact.**

1. **Update the installed sase.** After verifying that all three phases landed on `sase`
   and `sase-core` master, run `sase update`. This host uses the editable dev install,
   so it fast-forwards the sase and sase-core checkouts and rebuilds the Rust binding.
   Do not pass `-t`/`--to`; never switch install modes.
   - The Rust rebuild can take minutes. Give the command a generous timeout, or run it
     through `/sase_monitor` if it would outrun the turn.
   - Confirm the installed sase now contains this epic's work:
     `sase usage refresh -p agy` must succeed, and `sase usage list -p agy -v` must show
     the four agy windows.
2. **Take a live screenshot.** Read `tui_screenshot.md` with `/sase_memory_read` first.
   Then capture the real TUI, for example:
   `sase screenshot -o <tmp>/agy_usage_header.png -s 160x45 -w '🪐' -d 2000`. Look at
   the PNG itself, not just its existence. Success means the top-right usage cluster
   shows the `🪐` group with the unlabeled Gemini weekly anchor (`🪐 NN% XdYh`), no
   `family:` text, and no layout breakage. Other provider groups may also appear.
3. **Save every capture as an artifact, whether or not it shows problems:**
   `sase artifact create -p <png> -k image -l "agy usage header indicator (<pass|fail>: <one-line note>)" --bead <this epic's bead id>`.
   Keep each printed `file:explicit:…` reference.
4. **Fix anything wrong.** Any problem the screenshot or the refresh reveals is this
   epic's work. Examples: no `🪐` group, a `family:` prefix, a wrong percentage or
   countdown, truncation, or a stale install. Fix it, following the land prompt's "plan
   the remaining work" rule if it is not small. Then re-run steps 1–3 until a capture
   demonstrates success.
5. **Close with evidence.** The close note must cite the successful screenshot's
   artifact reference, and any earlier failing ones. Set `status: done` on this plan
   only after that close succeeds. If a success screenshot cannot be produced, for
   example because agy is logged out on this host, do not close. Record a note on the
   epic bead naming the blocker and the captured artifact refs, and report it.

## Risks

1. **The omitted-zero shape is unobserved.** The TSV cross-check covers both plausible
   shapes. A real exhausted payload should become a fixture when one occurs.
2. **Logged-out behaviour on macOS is unverified** (H7). The early stderr abort likely
   covers it; verify on the Mac before relying on default-on there.
3. **Catalog lag.** A Gemini model that agy ships before SASE's catalog knows it matches
   `does_not_apply` until the catalog is bumped.
4. **Enterprise, API-key, and ADC accounts** may report other group names; these degrade
   to `Unknown` applicability by design.
5. **Allowlist accumulation.** This is the fourth `provider == "…"` arm in
   `indicator.rs`. It is tracked by `sase-14i` and must not expand this epic.

## Out of scope

- Replacing the `indicator.rs` provider allowlists with declared headline-window
  classification (`sase-14i`).
- Marking agy usage refresh due after `AgyProvider.invoke` runs
  (`_mark_usage_refresh_due`).
- Surfacing `/credits` (a pay-as-you-go balance, not a window) in the Models panel.
- Re-ordering header provider groups by `llm_autodetect_priority`.
- Research §8 side findings: the stale `_SUPPORTED_AGY_TRAJECTORY_VERSIONS`
  (`{"1.0.10"}`, so trajectory extraction is silently off on 1.2.7), and the stale
  `agy.py` docstring and `invocation_option_args`, which still claim
  `--output-format`/`--effort` do not exist. The lander should route these through
  `/sase_new_task` if they are not already tracked.
