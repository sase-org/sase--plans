---
status: done
create_time: 2026-07-10 16:56:47
prompt: 202607/prompts/fakey_provider.md
bead_id: sase-5o
tier: epic
---

# Plan: `fakey` — a first-class fake agent CLI provider for testing launches, failures, and retries

## Summary

Add **fakey**, a deterministic fake agent CLI that ships with SASE itself (a `fakey` console script) plus a first-class
`sase_llm` provider (`FakeyProvider`) so it launches, streams, fails, and retries exactly like `claude` / `codex` /
`qwen` / `opencode` / `agy`. Fakey's behavior is scripted through a layered **scenario** system (sane defaults → env
knobs → scenario file → prompt-embedded scenario block), giving both tests and interactive TUI sessions precise control
over replies, failures (message, exit code, channel), delays, streaming, token usage, cross-invocation attempt sequences
("fail twice, then succeed"), and file-based sync barriers for deterministic E2E timing.

On top of fakey we build the retry-testing story end to end: real-subprocess E2E tests of the retry pipeline
(`run_agent_exec_retry.py`) and two tiers of PNG snapshot coverage for retry rendering — fixture-driven goldens of every
retry visual state (none exist today), and E2E-driven goldens where a **real fakey retry chain** produces the on-disk
agent state that the Agents tab renders. This is the foundation for the upcoming refactor of SASE's failed-agent retry
handling.

## Goals

- A `fakey` CLI with first-class provider support: launchable from the TUI (`%m:fakey-large`), visible in the models
  panel, doctor, provider badges/colors — indistinguishable in plumbing from real providers.
- Sane defaults: with zero configuration, `fakey` succeeds instantly with a friendly deterministic reply; a
  retryable-failure scenario triggers SASE retries out of the box via fakey's built-in retry config.
- Highly configurable: specific failure messages, exit codes, per-attempt behavior sequences, delays/streaming, usage
  numbers, hangs, sync barriers, invocation recording.
- Top-notch E2E coverage of agent CLI failure retries: real-subprocess tests through the actual retry executor, plus PNG
  snapshot tests demonstrating retry behavior in the Agents tab.
- Intuitive, reliable, beautiful: obvious scenario grammar, bundled named scenarios (`@flaky`, `@crash`, ...), excellent
  colored `--help` (per `memory/cli_rules.md`), fully deterministic output.

## Non-goals (v1)

- **No Rust core work.** Fakey is a provider plugin + test infrastructure, the same layer as the other Python providers
  in `src/sase/llm_provider/` — it fails the core-boundary litmus test (no other frontend needs fakey's behavior).
- No `sase fakey` subcommand group. Real agent CLIs are standalone binaries; fakey is too (a console script), which is
  what makes autodetection, `SASE_FAKEY_PATH`, and subprocess plumbing behave exactly like real providers.
- No skill deployment target (`llm_skill_deploy_subpath`) and no `FAKEY.md` instruction shim — fakey never reads repo
  instructions or runs skills; it only emulates the CLI process contract.
- No fault-injection beyond the process contract (no fake MCP, no fake tool-call streams). Fakey emulates what SASE
  observes: argv, stdin, stdout/stderr, exit code, timing, and files it is told to write.

## Key design decisions

1. **Ship the CLI inside SASE** (`src/sase/fakey/`, console script `fakey` in `[project.scripts]`). Always installed,
   version-locked to SASE, zero external deps. Tests can also point `SASE_FAKEY_PATH` at a copy for hermetic runs.

2. **Behavior = scenarios, layered by specificity.** Precedence (most specific wins):
   1. A fenced ` ```fakey ` scenario block embedded in the prompt itself (fenced blocks are xprompt literal zones, so
      they survive prompt preprocessing untouched — interactive TUI testing is just "type a prompt containing the block,
      launch with `%m:fakey-large`").
   2. `FAKEY_SCENARIO` env var — a path to a scenario YAML file, or `@name` for a bundled scenario.
   3. Individual env knobs for one-liners: `FAKEY_REPLY`, `FAKEY_FAIL_MESSAGE`, `FAKEY_EXIT_CODE`, `FAKEY_DELAY`,
      `FAKEY_FAIL_TIMES` (fail N invocations, then succeed).
   4. Built-in default: succeed immediately with a canned reply.

3. **Canonical failure markers make retries work with zero config.** A retryable scenario failure prints
   `FAKEY-RETRYABLE: <message>` to the configured channel; a non-retryable one prints `FAKEY-FAIL: <message>`.
   `FakeyProvider.llm_default_retry_config()` ships `error_patterns: ["FAKEY-RETRYABLE"]` with small `max_retries` and
   `wait_times: [0]`. The prefix is unique so `find_retry_config_for_error()` can never mis-attribute real provider
   errors to fakey — while scenarios remain free to emit _other providers'_ patterns verbatim (e.g. codex's "Selected
   model is at capacity") to test cross-provider matching deliberately.

4. **Cross-invocation attempt state.** Scenarios carry an ordered `attempts:` list (last entry repeats). The attempt
   cursor persists in a small JSON state file (`FAKEY_STATE_DIR`, defaulting to alongside the scenario file, else
   `SASE_ARTIFACTS_DIR`, else a tmp fallback) so sequences survive both in-process retries and `spawn_new_agent` retry
   chains. Prompt-block scenarios key state by content hash.

5. **Sync barriers for deterministic E2E timing.** Scenario steps `wait_for: <path>` (block until a file exists, with
   timeout) and `signal: <path>` (touch a file on reaching a point) let tests freeze a live agent at any moment — the
   antidote to sleep-based flakiness in retry/TUI tests. SIGTERM during a barrier exits cleanly so kill flows are
   testable.

6. **Invocation recording.** Every run appends `invocation-<n>.json` (argv, prompt, selected env, resolved scenario,
   outcome) under the state dir. This is the "test sase agent launches" primitive: tests assert exactly what SASE passed
   (model flag, effort args, continuation nudge prepended on retry, fallback model after exhaustion).

7. **Provider mirrors real providers; no special cases in shared code.** `FakeyProvider` uses stdin prompt transport,
   `--model` flag, plain stdout streaming via the existing `stream_process_output` (agy-style), the shared interrupt
   monitor, and `effort_cli_args` with a full effort map (`--effort <level>`) so effort plumbing is testable. Optional
   token usage comes from a final `FAKEY-USAGE: {json}` stdout line the provider strips and parses. Per the
   uniform-runtimes gotcha, **zero fakey-specific branches** may be added to `_invoke.py`, the retry executor, or any
   shared orchestration — all fakey behavior lives in its own plugin module and CLI.

8. **Fakey must never win default-provider autodetection.** Since the binary is always on PATH once SASE is installed,
   fakey either omits the autodetect hooks or reports a floor priority strictly below every real provider (verify
   existing `llm_autodetect_priority` values) — explicit selection only (`provider: fakey` config, `%m:fakey-...`).
   Achieved purely through fakey's own hook return values, never registry special-casing.

## Architecture overview

### The `fakey` CLI contract (Phase 1 implements; later phases consume)

```
fakey [--model <m>] [--effort <level>] [--scenario <path|@name>] [--version] ...   # prompt on stdin
```

Scenario schema (YAML, `version: 1`):

```yaml
reply: "..." # success reply text (default: canned deterministic reply)
delay: 0.0 # seconds before responding
stream: { chunk_delay: 0.05 } # optional line-by-line streaming (exercises live_reply.md)
usage: { input_tokens: 7, output_tokens: 3 } # optional -> FAKEY-USAGE tail line
attempts: # per-invocation behavior; cursor persists; last entry repeats
  - fail: { message: "model melted", retryable: true, exit_code: 1, channel: stderr }
    steps: [{ signal: /tmp/a1.started }, { wait_for: /tmp/a1.release }, { sleep: 0.2 }]
  - succeed: { reply: "second try worked" }
```

Bundled named scenarios in `src/sase/fakey/scenarios/`: `@ok`, `@flaky` (fail retryable once, then succeed), `@flaky2`,
`@crash` (non-retryable), `@hang` (block until killed), `@slow`, `@capacity` (mimics the codex at-capacity message).
`fakey --list-scenarios` and `fakey --explain` (print the resolved scenario and exit) aid debugging. All public long
options get short aliases, sorted help, colored output (`memory/cli_rules.md`).

### Provider integration surface (Phase 3)

Template: `qwen.py`/`agy.py`. Touchpoints (from the provider-architecture survey): new
`src/sase/llm_provider/fakey.py` + `pyproject.toml` `sase_llm` entry point (required), plus the hardcoded display tables
— `src/sase/integrations/provider_badges.py` (emoji), `src/sase/ace/tui/provider_styles.py` fallback palette,
`src/sase/doctor/checks_providers.py` setup hints ("bundled with SASE — nothing to install"). Suggested identity: short
name `fky`, status color hot pink `#FF5FAF` (clearly "not a real vendor"; implementer verifies contrast/dedup against
existing provider colors), models `fakey-large` / `fakey-small` with short aliases. Auth evidence reports always-ready.
`SASE_FAKEY_PATH`, `SASE_LLM_*_ARGS`/`SASE_FAKEY_*_ARGS`, and model→provider resolution all come free from the registry.

### Retry-testing pyramid (Phases 2, 4, 5)

1. Unit: scenario engine + CLI behavior (Phase 1).
2. Provider-level: `FakeyProvider.invoke()` against the real binary (Phase 3).
3. Pipeline E2E: the real retry executor with real fakey subprocesses (Phase 4).
4. Visual: fixture-driven retry-rendering goldens (Phase 2) + E2E-driven goldens from real fakey runs (Phase 5).

## Phase breakdown

Five phases, each sized for a single agent. Dependency graph:

```
Phase 1 (fakey CLI) ──────► Phase 3 (provider) ──► Phase 4 (E2E retry tests) ──► Phase 5 (E2E-driven PNGs)
Phase 2 (fixture retry PNGs) ────────────────────────────────────────────────► Phase 5
```

Phases 1 and 2 can run in parallel; 3 needs 1; 4 needs 3; 5 needs 2 and 4.

---

### Phase 1 — The `fakey` CLI and scenario engine

**Goal:** A standalone, deterministic fake agent binary with the full scenario system. No provider wiring.

**Deliverables:**

- `src/sase/fakey/` package: `cli.py` (argparse entry point), `scenario.py` (schema, layered resolution, validation with
  actionable errors), `state.py` (attempt cursor + invocation records), bundled `scenarios/`.
- `fakey` console script in `pyproject.toml` `[project.scripts]`.
- Full CLI contract from _Architecture overview_: stdin prompt, scenario precedence, env knobs, failure markers,
  barriers with timeouts, invocation recording, `--version`, `--list-scenarios`, `--explain`, clean SIGTERM handling.
- Unit tests (`tests/fakey/`) covering the scenario engine in-process and the binary via subprocess: defaults,
  precedence layering, attempt persistence across processes, barrier timeout behavior, marker formats, malformed
  scenario diagnostics.
- CLI polish per `memory/cli_rules.md`: alphabetized options, short aliases for every public long option, beautiful
  colored `--help` with a short scenario-grammar primer.

**Out of scope:** provider class, entry-point registration, docs site pages.

**Verification:** `just check`.

---

### Phase 2 — Fixture-driven retry-rendering PNG goldens

**Goal:** Close today's zero-coverage gap on retry visuals using the existing fixture machinery (no fakey dependency).

**Deliverables:**

- New `tests/ace/tui/visual/test_ace_png_snapshots_agents_retry.py` + a retry-agent builder in
  `_ace_png_snapshot_helpers.py`, driving `patch_startup_loaders` with `Agent` fixtures that exercise every retry field
  (`retry_status`, `retry_count`/`max_retries`, `retry_next_at_epoch`, `retry_attempt`, `retry_chain_root_timestamp`,
  `using_fallback`, `retry_terminal`).
- Goldens for each visual state rendered by `_agent_list_render_agent.py`: RETRYING with live countdown
  (`retry_next_at_epoch` fixed relative to the pinned visual clock so the `(Ns)` text is stable), RUNNING retry with
  `↻N` badge and `▸<fallback-model>`, a retry chain (root + `↳`-indented attempts) with `FAILED (RETRIED)` rows and a
  final DONE, and a retries-exhausted terminal FAILED. Include a selected-row detail-pane variant showing retry
  metadata.
- Paired `assert_page_svg_contains` textual assertions so failures diagnose without opening PNGs.

**Verification:** `just test-visual` (goldens reviewed by eye before accepting) and `just check`.

---

### Phase 3 — First-class provider integration

**Goal:** `fakey` becomes a registered provider indistinguishable in plumbing from real ones.

**Deliverables:**

- `src/sase/llm_provider/fakey.py` (`FakeyProvider`) per _Key design decisions_ 3, 7, 8: stdin transport, plain
  streaming + `FAKEY-USAGE` tail parsing, shared interrupt monitor, full `--effort` map, `llm_*` metadata hooks,
  built-in `llm_default_retry_config()`, autodetect-floor behavior.
- `pyproject.toml` `sase_llm` entry point (note: requires `just install` to register in a workspace).
- Display/doctor touchpoints: provider badge emoji, `provider_styles.py` fallback palette entry, doctor setup hints.
  Explicitly skip `skills/cli_list.py` and `PROVIDER_SHIM_FILES` (non-goals) with a code-comment-free rationale in the
  PR description.
- Config: a commented `llm_provider.retry.fakey` example in `default_config.yml` (built-in hook is the real default).
- Docs: provider section in `docs/agent_providers.md`, retry-defaults note in `docs/llms.md`, and a new `docs/fakey.md`
  scenario-grammar reference wired into `mkdocs.yml`.
- Tests: provider-level invoke against the real binary (success, retryable/non-retryable failure surfaces as
  `CalledProcessError` with the marker in stderr, usage tail parsing, effort args, `SASE_FAKEY_PATH` override, interrupt
  loop), registry metadata/autodetect-floor tests (with the documented registry cache resets), retry-config merge tests
  mirroring `tests/test_llm_provider_retry_config.py`.

**Verification:** `just install` then `just check`; manual smoke: `%m:fakey-large` launch from the TUI renders the fakey
badge/colors and completes.

---

### Phase 4 — E2E retry pipeline harness and tests

**Goal:** Real-subprocess coverage of the actual retry executor — the safety net for the upcoming retry refactor.

**Deliverables:**

- Reusable harness in `tests/fakey/harness.py`: isolated SASE home/artifacts sandbox, scenario+barrier builders,
  retry-config injection (`llm_provider.retry.fakey` with `wait_times: [0]` for speed), helpers to read
  `retry_state.json` transitions, DONE markers, and fakey invocation records.
- E2E tests driving `run_execution_loop` (`src/sase/axe/run_agent_exec.py`) with `provider=fakey` and real subprocesses,
  covering at minimum:
  - retryable fail once → succeed: `RetryState` lifecycle (`retrying` → cleared), continuation nudge present in attempt
    2's recorded prompt, success DONE marker;
  - retries exhausted → terminal failure outcome (`failed_retried` path) with `retry_terminal` state;
  - fallback model: exhaustion with `fallback_model` set → recorded `--model` switch on the fallback attempt;
  - `spawn_new_agent: true` → `retry_handoff.json` handoff and retry-chain artifacts;
  - kill during the retry wait (barrier-held) → clean stop, no further attempts.
- One full detached-spawn smoke test (the real `spawn_detached_agent` path) if feasible within the sandbox, marked
  `slow`; otherwise document why exec-level is the highest practical layer.
- Keep the existing mocked retry tests (fast unit tier); the new suite is the integration tier. Barriers always carry
  timeouts and `finally`-kill cleanup so the suite cannot hang CI.

**Verification:** `just check` (new tests fast enough for the default suite; anything slower marked `slow`).

---

### Phase 5 — E2E-driven retry PNG snapshots

**Goal:** PNG tests where a **real fakey retry chain** produces the state the Agents tab renders — the requested
demonstration of retries, end to end.

**Deliverables:**

- Harness extension: run a fakey retry chain via Phase 4 helpers inside the sandbox, freeze it at interesting moments
  with barriers (using long `wait_times` while frozen), then **normalize timestamps** in the on-disk state
  (sentinel-rewrite of `retry_state.json` epochs and artifact timestamps relative to the pinned visual clock —
  precedent: `tests/agent_scan_golden/fixture_builder.py`) so goldens are byte-stable.
- Visual tests that point the _real_ agent loader (`load_agents_from_disk_with_state`) at the sandbox — not
  `patch_startup_loaders` fakes — and capture goldens for: mid-RETRYING wait (countdown visible), the completed chain
  (`FAILED (RETRIED)` root, `↻N` badges, final DONE), and a running-fallback state (`▸fakey-small`).
- Release-and-finish coverage: after the screenshot, release the barrier and assert the run completes and the TUI state
  converges (textual assertions, no additional golden).
- Robustness: barrier timeouts, `finally` process cleanup, and skip-with-diagnostics if the sandbox spawn is
  unavailable; suite stays in the `visual` marker.
- Close the loop in `docs/fakey.md`: a short "testing retries with fakey" recipe covering the exec harness and the
  visual harness.

**Verification:** `just test-visual` + `just check`; goldens eyeballed for beauty (alignment, colors, no truncation
artifacts) before accepting.

## Cross-cutting requirements (every phase)

- Run `just install` first in a fresh workspace, `just check` before finishing; visual phases also run
  `just test-visual`. Read `memory/pyvision.md` before fixing any pyvision lint failures.
- No fakey-specific branching in shared orchestration/retry/TUI code — plugin hooks and scenario config only.
- Determinism: no wall-clock- or randomness-dependent output anywhere in fakey; every blocking primitive has a timeout;
  every spawned process is cleaned up in `finally`.
- New CLI options follow `memory/cli_rules.md` (short aliases, sorted, colored help).
- Do not edit `memory/*.md`, `AGENTS.md`, or provider instruction shims.

## Risks & mitigations

- **Fakey wins autodetection on machines without real CLIs** (it is always installed). Mitigated by decision 8
  (autodetect floor / opt-out via fakey's own hooks) plus an explicit registry test pinning that behavior.
- **Attempt-cursor consumption by auxiliary invocations** (e.g. the commit finalizer re-invoking the provider would
  advance `attempts`). Mitigated: E2E tests run without `SASE_COMMIT_METHOD` so the finalizer no-ops; the scenario
  engine's "last attempt repeats" default keeps stray invocations harmless; invocation records make any surprise
  visible. Documented in `docs/fakey.md`.
- **Timestamp nondeterminism in E2E-driven goldens** (real runs write real epochs; the TUI clock is pinned). Mitigated
  by the Phase 5 normalization pass; Phase 5 must first verify which clock source the countdown renderer uses (the
  pinned `local_now` patch set) and normalize accordingly.
- **Hanging tests from barriers/`@hang`.** Every `wait_for` has a timeout; harness kills processes in `finally`;
  hang-scenarios are only used in kill-flow tests.
- **Entry-point registration staleness** (new `sase_llm` entry point needs reinstall). Called out in Phase 3; tests that
  need registry re-resolution use the documented cache-clear helpers.
- **Marker collision with real provider errors.** The `FAKEY-` prefix is namespaced and covered by a regression test
  asserting no real provider's built-in patterns match fakey markers (and vice versa, except where a scenario opts in
  deliberately).
