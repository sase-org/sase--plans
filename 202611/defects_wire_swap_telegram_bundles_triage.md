---
tier: epic
title:
  Targeted defect fixes for the prompt_prose wire, update races, Telegram transport,
  bundle caps, and triage conflicts
goal:
  Host-produced artifact-link origins, including prompt_prose, round-trip through the
  real sase_core_rs binding, and a contract test fails on any future origin drift. Jobs
  and launches that overlap a code swap defer and retry instead of failing, an
  update-overlap reproducer guards that behavior, and a prebuilt-artifact cache miss is
  a miss rather than a failure. Telegram falls back to plain text only on entity-parse
  errors, retries per chunk without resending delivered chunks, reports credential and
  network outages once per state transition, and runs its outbound job only on the
  sending machine. Every provider root bundle fits its declared max_bytes through
  compile-time compaction, with a doctor check and a just-check test that catch growth.
  bead_task_triage recovers from request_id_conflict by advancing the generation.
parent_bead: sase-1tz.3
phases:
  - id: prose_wire
    title:
      Add the prompt_prose artifact-link origin and a host-origin binding contract test
    description:
      'prose_wire: add PromptProse to sase-core''s ArtifactLinkOriginWire with the
      automatic, commit-scoped semantics sase already relies on, and move the pin. Give
      sase one constant listing every host-produced origin, and round-trip each through
      sase_core_rs.artifact_link_validate_row in a contract test. Record what the "wire
      is stale" skew message means.'
    size: medium
    depends_on: []
  - id: swap_repro
    title: Build an update-overlap reproducer for jobs, launches, and native code
    description:
      "swap_repro: a test-only harness simulates an install swap in temp dirs under a
      real code-swap writer lock. It drives vanishing scripts, a vanishing binding,
      mixed-version lazy imports, and in-place versus atomic file replacement.
      Still-failing modes become strict xfails for the guard phases. Record which modes
      reproduce and how the real installers replace files."
    size: medium
    depends_on: []
  - id: swap_chop_guards
    title: Defer and retry scheduler jobs that overlap a code swap
    description:
      "swap_chop_guards: the code-swap writer records a durable last-swap stamp. The
      chop runner does not start scripts while a writer is live, and it treats a missing
      script within a short post-swap grace as inhibited. A run that overlapped a swap
      and died of SIGBUS or a swap-skew import error is recorded as a skipped retry, not
      an error. Flip the chop xfails."
    size: medium
    depends_on:
      - swap_repro
  - id: swap_launch_guards
    title: Defer and retry launches and finalizers that overlap a code swap
    description:
      "swap_launch_guards: agent launches, bead work, and host finalizers that hit a
      missing binding or swap-skew import error during or just after a swap wait for it
      to end and retry. Long-lived processes stop mixing pre-swap and post-swap modules.
      Fix in-place native-code replacement only if the reproducer triggered SIGBUS. Flip
      the remaining xfails."
    size: medium
    depends_on:
      - swap_chop_guards
  - id: prebuild_miss
    title: Treat a prebuilt Rust artifact cache miss as a miss, not a failure
    description:
      "prebuild_miss: prebuild consume exits 0 and records hit:false with its reason on
      a cache miss. Dev-update reconcile records the step as a miss, then builds from
      source. The prebuild producer's writer probe also recognizes install.dev and
      install.pypi swaps."
    size: small
    depends_on: []
  - id: tg_transport
    title: Fall back to plain text only on entity-parse errors and retry per chunk
    description:
      "tg_transport: in sase-telegram, plain-text fallback fires only for an
      entity-parse BadRequest. Timeouts, network errors, rate limits, and credential
      errors reach the retry wrapper unchanged. Retries wrap each chunk, so delivered
      chunks are never resent, and the outbound job resumes a partly delivered
      notification at its first undelivered chunk."
    size: medium
    depends_on: []
  - id: tg_degraded
    title:
      Collapse repeated Telegram credential and network failures into degraded states
    description:
      "tg_degraded: outbound and receiver track a persisted credential_unavailable or
      network_degraded state. Each logs one line on entering a state, rate-limited
      summaries while degraded, and one line on recovery, with no tracebacks for
      transient errors. One SASE notice fires per transition. Credential lookups are
      negatively cached for a short window."
    size: medium
    depends_on:
      - tg_transport
  - id: tg_scope
    title: Run the Telegram outbound routine only on the sending machine
    description:
      "tg_scope: move the axe.routines.telegram routine from the shared chezmoi sase.yml
      into sase_athena.yml next to the telegram_receiver proc. Confirm the machine
      overlay loads it on athena and leaves apollo and mac without it."
    size: small
    depends_on: []
  - id: bundle_trim
    title: Compact oversize provider root bundles at compile time until they fit
    description:
      "bundle_trim: when a provider declares max_bytes and the delivered bundle would
      exceed it, compile applies a deterministic compaction ladder. Step one drops
      summaries from superseded list-roster entries. Step two compacts whole list
      rosters, largest first. Bundles that already fit stay byte-identical, and no
      memory file changes."
    size: medium
    depends_on: []
  - id: bundle_gate
    title: Gate provider bundle sizes in sase doctor and just check
    description:
      "bundle_gate: a sase doctor check renders every enabled project's root bundle for
      each provider that declares max_bytes. It errors over the cap and warns when
      compaction was needed. A just-check test keeps sase's own bundles under every cap
      without compaction. Update the instruction-bundle and Codex provider docs."
    size: medium
    depends_on:
      - bundle_trim
  - id: triage_conflict
    title: Advance bead_task_triage's generation on request_id_conflict
    description:
      "triage_conflict: when gate creation fails with request_id_conflict,
      bead_task_triage cancels any pending gate under the stale id and advances and
      persists the generation. It then recreates the gate in the same pass. Other
      failures keep the deterministic retry."
    size: small
    depends_on: []
proposed_by: bbugyi200.athena.sase-1tz.3
proposed_with: claude/opus
create_time: 2026-11-20 09:19:02
status: wip
---

- **PROMPT:**
  [prompts/202611/defects_wire_swap_telegram_bundles_triage.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/defects_wire_swap_telegram_bundles_triage.md)
- **PARENT:**
  [202611/notification_log_audit_fixes.md](https://github.com/sase-org/sase--plans/blob/main/202611/notification_log_audit_fixes.md)

# Plan: Targeted defect fixes (items 6–10)

This child epic implements phase `defects` (**sase-1tz.3**) of parent epic **sase-1tz**,
`plan:202611/notification_log_audit_fixes.md`. It covers items 6–10, which are audit
findings F6 (`prompt_prose` wire mismatch), F7 (update races), F8 (Telegram transport),
F10 (Codex bundle cap), and F9 (triage request-id retry).

## Read first (every phase)

- **Audit.** The source of truth is
  `research:202611/apollo_athena_sase_notification_and_log_audit/apollo_athena_sase_notification_and_log_audit.md`
  (sections F6–F10 and ranked fixes 6–10). Read it with
  `sase artifact read <ref> "<reason>"`. If the reference resolves as missing, first
  open the research sidecar with `sase repo open sase--research -r "<why>"`. The
  `…__cdx.md`, `…__cld.md`, `…__grk.md`, `…__mus.md`, and `…__gem.md` swarm reports in
  the same directory hold the raw counts, log excerpts, and probes.
- **Final parent decisions.** The parent epic's DECISIONS are final; do not re-ask them.
  `codex_bundle = trim` selects the approach in `bundle_trim`: shrink always-inlined
  content, and keep Codex's cap at 32 KiB. Do not raise `max_bytes`. `held_heartbeat`
  and `blocked_waiters` select nothing in this epic.
- **Rust core boundary.** Shared backend and domain behavior lives in `sase_core`. Here
  that is only the artifact-link origin wire (`prose_wire`). Open sase-core with
  `sase repo open sase-core -r "<why>"` and work only in the printed path. A phase that
  adds or changes a binding sase calls must move `sase-core-revision.txt` past that
  sase-core commit; see `docs/rust_backend.md`. Instruction-bundle compilation, the chop
  runner, dev update, and triage stay where they live today, in Python.
- **Other repos.** `tg_transport` and `tg_degraded` work in the linked sase-telegram
  repo, and `tg_scope` in the linked chezmoi repo. Open each with
  `sase repo open <name> -r "<why>"`. `bundle_trim` and `bundle_gate` may render the
  bob-cli project read-only. Open it as an external repo with
  `sase repo open bob-cli -r "<why>"`. Never reach any of these repos by another path.
  sase-telegram's guarded check runs as `sase tool run check` from its checkout (see its
  `AGENTS.md`).
- **Verification.** `just check` is the agent verification recipe. Read the
  `lint_and_test.md` reference memory before finishing. Run `just check-full` only when
  explicitly instructed. Every behavior change gets focused tests, including the
  regression scenarios named in its phase.
- **Live hosts are read-only.** Global install and `chezmoi apply` are human-only, so
  live acceptance happens after the user installs. Phases may take read-only
  measurements on apollo and athena. The `tailnet.md` reference memory covers SSH
  access. Each measuring phase records two things as a note on its own phase bead: the
  exact before-numbers, and the command to repeat after install. It also adds a one-line
  summary note on **sase-1tz.3**. Never dismiss notifications, repair state, or delete
  files on either host. Never run an install, update, or swap against the live global
  install. Propose any one-time live action through `/sase_gate`.
- **Discovered work.** Phase workers are epic phase workers. Record follow-up work as
  `PROPOSED FOLLOW-UP:` notes on your own phase bead, never as new task beads.
- **Docs and memory.** User-visible behavior changes update the matching page under
  `docs/` (or the sase-telegram README/docs). Read the `docs_writing.md` reference
  memory first. A survey of the always-loaded notes and the reference listing found no
  memory note that this epic makes false, so this plan authorizes no memory edits. If a
  phase finds such a note, it records a `PROPOSED FOLLOW-UP:` and leaves memory alone.
  That includes descriptor rosters, `AGENTS.md`, and provider shims. `bundle_trim` is
  designed to leave every memory file and the committed `AGENTS.md` byte-identical. Any
  phase that touches `tools/` reads the `tools.md` reference memory first. Read
  `sase_flags.md` before changing behavior that sits behind a feature flag.
- **Already landed: build on it, don't rebuild it.**
  - `ea7916d7a7` records a missing chop script as an inhibited skip while a live
    code-swap writer holds the lock (`src/sase/axe/chop_runner_script.py:213-264`,
    `code_swap_writer_holder()` at `src/sase/dev_update/code_swap_lock.py:329`).
  - `1ccd5c0579` queues dev updates behind in-flight launches (`CodeSwapWait`,
    `wait_for_code_swap_readers`, `src/sase/dev_update/execute.py`).
  - Child epic sase-1tz.2.1's `pub_standing` collapsed the recurring "unknown variant
    `prompt_prose`" publication failure into one standing alert. `prose_wire` fixes the
    cause. After install, that alert should clear on its own.

## Baseline (from the audit and read-only code reading, 2026-11-19/20)

- **F6.** Athena's citations for `plan:202611/research_commit_gap.md` fail on every pass
  with "unknown variant `prompt_prose`". Launch-failure logs also show "sase_core_rs
  artifact-reference wire is stale: expected 3, got 4".
- **F7.** SIGBUS (`exit code -7`) appears in `sidecar_auto_sync`, `agent_auto_restart`,
  and `pending_checks_poll` within seconds of updates. Other symptoms:
  - `ModuleNotFoundError: sase_core_rs`.
  - Missing `sase_job_tg_outbound` and `sase_job_agent_auto_restart` scripts.
  - A `BeforeSpawnCallback` import failure 36 s after an update whose binding checks
    passed.

  The dev-update "Install prebuilt Rust dev artifacts" step records a failure on 18 of
  37 apollo runs, all of them `hit:false, reason:commit-mismatch`.

- **F8.** Athena logged 143 plain-text fallbacks, and all of them followed `TimedOut` or
  `NetworkError`. The Nov 14–15 gpg outage produced 408 identical
  `TelegramCredentialError` tracebacks. Apollo spawns `tg_outbound` about 12.7 times a
  minute but has sent nothing since Sep 5. Athena is the live sender.
- **F10.** bob-cli's Codex root bundle renders at 36,225 bytes, and 24,900 of those are
  its `decisions` web roster: 75 entries, 31 of them superseded, totalling 12,213 bytes.
  sase's Codex bundle is 22,099 bytes. The delivered text adds a marker line of about 90
  bytes, which leaves about 32,678 bytes of budget under the 32 KiB cap. Every Codex
  launch in bob-cli falls back to legacy delivery, about 60–75 a day across both hosts.
- **F9.** Athena logged 1,517 identical "request id already belongs to a different gate
  specification" failures in one week, and the affected bead never got its triage gate.

## Coordination with open work

- **sase-1tz.4 (parent phase `hygiene`)** depends on this phase. Its item 12.3 adds an
  `interaction_requests/` reconciliation that builds on `_reconcile` in
  `src/sase/scripts/sase_chop_bead_task_triage.py`. `triage_conflict` keeps its change
  inside the gate-creation branch so that work rebases cleanly.
- **sase-1kt, sase-1ku, sase-1kv** will retire the per-provider explicit-instruction
  flags. `bundle_trim` and `bundle_gate` must not change flag gating; the Codex gate is
  `FeatureFlag.codex_explicit_instructions` in
  `src/sase/llm_provider/_codex_instructions.py`. The doctor check skips a provider
  whose explicit channel is disabled.
- **sase-1tz.2.1 `pub_standing`** owns the standing alert for the `prompt_prose`
  failure. `prose_wire` records its alert key, so the after-install check can confirm
  the alert cleared.

## Phase `prose_wire`: the `prompt_prose` wire contract (item 6, F6)

**Today.** sase emits `prompt_prose` as the origin of prose-detected plan and research
citations (`PROMPT_PROSE_ORIGIN`,
`src/sase/agents_sync/referenced_by_planning.py:28,169`). Python treats this origin in
five ways:

- as automatic (`src/sase/sdd/_artifact_link_projection.py:23`,
  `src/sase/bead_pages/rendering_links.py:13`);
- as commit-scoped (`src/sase/sdd/_artifact_link_store_support.py:31`);
- as a usage-bearing canonical event
  (`src/sase/sdd/_artifact_link_event_canonical.py:112`);
- as drain-eligible (`src/sase/sdd/_artifact_link_outbox_drain.py:486`);
- with the label "prose citation" (`src/sase/bead/cli_detail_links.py:20-23`).

sase-core's `ArtifactLinkOriginWire`
(`crates/sase_core/src/artifact_link/wire.rs:66-100`) has only `manual`, `migrated`,
`prompt_ref`, `read`, `derived`, and `projected`. Serde therefore rejects every
`prompt_prose` row, and so does the binding `artifact_link_validate_row`
(`crates/sase_core_py/src/artifact_links/mod.rs:981`), which the store calls at
`src/sase/sdd/_artifact_link_store_support.py:317`. The host origin strings are
scattered literals, so nothing catches this drift.

**Required changes.**

1. **sase-core variant.** Add `PromptProse` (`prompt_prose`) to
   `ArtifactLinkOriginWire`, covering `as_str`, `from_name`, serde, and every `match` on
   the enum. Give it the provenance semantics sase relies on: automatic and
   commit-scoped, like `PromptRef`. For `increments_uses`, match what the Python
   canonical-event path does for `prompt_prose` today, and state the choice in a doc
   comment. Audit every Rust site that branches on origin: `events.rs`, `ownership.rs`,
   `cutover.rs`, `publication_retry.rs`, the index chips, and the bead read projection.
   Make `prompt_prose` behave like `prompt_ref` wherever the Python side does. Add an
   exhaustive `ALL` list with a unit test that round-trips every variant through serde
   and `from_name`.
2. **Do not relabel.** Never rewrite `prompt_prose` as `prompt_ref`; that would erase
   recorded provenance.
3. **One host-origin constant.** Add a single Python source of truth for every origin
   the host writes, for example `ARTIFACT_LINK_HOST_ORIGINS` beside the artifact-link
   store support. Derive the automatic, commit-scoped, and usage-bearing subsets from
   it. Point the call sites listed above at those constants instead of string literals.
4. **Binding contract test.** Add a test that builds one minimal valid row per origin in
   that constant and round-trips it through the real
   `sase_core_rs.artifact_link_validate_row`. It asserts that the origin survives
   unchanged. Because the test enumerates the constant, a future origin without Rust
   support fails it. It must run under `just check`.
5. **Pin.** Move `sase-core-revision.txt` past the sase-core commit
   (`docs/rust_backend.md`).
6. **Skew message.** "artifact-reference wire is stale: expected 3, got 4" comes from
   the existing version-skew guard `_require_artifact_ref_schema`
   (`src/sase/artifact_ref_operations.py:347-355`). That guard compares
   `ARTIFACT_REF_WIRE_SCHEMA_VERSION` (now 5) with the loaded binding, so it fires when
   Python and the binding come from different installs. That is an F7 update-window
   symptom; `swap_launch_guards` owns the deferral. Note on this bead that the guard
   exists and covers the artifact-ref wire. Also note whether the artifact-link row wire
   needs a similar version guard for enum additions; if it does, record a
   `PROPOSED FOLLOW-UP:`.
7. **Measure.** Before: on athena, read-only, count the pending or failing referenced-by
   events for the `plan:202611/research_commit_gap.md` citation family, and record the
   `pub_standing` alert for it. After install: give the command that confirms the family
   drains once and the standing alert clears.

## Phase `swap_repro`: the update-overlap reproducer (item 7.1, F7)

**Today.**

- **Lock.** The code-swap writer lock is `~/.sase/locks/code-swap-v2.lock`, with reader
  holder files in `locks/code-swap.holders/` (`src/sase/dev_update/code_swap_lock.py`).
  Three writers take it: `sase dev update` (`execute.py:137`, op `dev.update`),
  `just install` (`tools/_sase_install_run.py:295,1432`, op `install.pypi`), and
  `just install-dev` (`:1946`, op `install.dev`).
- **Chops run through the swap.** Chops launch as uv console scripts via
  `discover_chop_script` and `stream_chop_script`
  (`src/sase/axe/chop_script_runner.py:42,168`). The installer restarts the scheduler
  only after verify (`tools/_sase_install_run.py:2054-2070`), so chops keep running
  during the swap.
- **Mixed-version imports.** `BeforeSpawnCallback` lives in
  `src/sase/agent/launch_executor_types.py:119`. `launch_executor` is imported lazily
  (`multi_prompt_launch_execution.py:91`, `launch_cwd_single.py:150`), while the types
  module is imported at the top level. A long-lived process can therefore mix pre-swap
  and post-swap modules.
- **Native code.** `maturin develop` (`Justfile` `rust-dev-install*` recipes) may
  rewrite the extension `.so` in place while processes have it mapped, which is a
  plausible SIGBUS source. This is unverified. The prebuild consumer swaps files
  atomically (`src/sase/dev_update/prebuild_consumer.py:115-131`).
- **No end-to-end harness.** No existing test simulates a swap end to end. The closest
  are `tests/test_axe_chop_runner_script.py:377`, the
  `tests/dev_update/test_code_swap_lock*.py` files, and the installer harnesses in
  `tests/sase_install/`.

**Required changes.**

1. **Harness.** Add a test-only harness, for example under `tests/dev_update/`, that
   simulates an install swap in temp dirs. It needs a temp SASE home with the real
   `code_swap_writer_lock`, a temp bin dir of fake console scripts, an importable stub
   binding module, and a two-version Python package. It must never touch the live global
   install or `~/.sase`.
2. **Scenarios.** Drive the real chop runner (`run_configured_chop_once`) and
   subprocesses through these scenarios:
   - (a) a script vanishes while the writer is live;
   - (b) a script is still missing just after the writer releases;
   - (c) a running job loses its binding mid-run (`ModuleNotFoundError`);
   - (d) a long-lived process that imported types v1 lazily imports a module that needs
     a v2 symbol (the `BeforeSpawnCallback` shape);
   - (e) a memory-mapped file rewritten in place, compared with one replaced by rename,
     run in a subprocess (use a plain mmapped data file if loading a real extension is
     impractical);
   - (f) a dev-update writer waiting on readers while a chop that holds no reader lock
     is mid-run.
3. **Real install paths.** Read-only, determine whether each real install path replaces
   files in place or by rename. The paths are `tools/_sase_install_run.py` (dev and
   PyPI), the `rust-dev-install*` recipes and `maturin develop`, `uv tool install`, and
   the prebuild consumer. Read `tools.md` first.
4. **Strict xfails.** Encode every scenario that still fails as
   `pytest.mark.xfail(strict=True, reason="swap_chop_guards: …")` or
   `reason="swap_launch_guards: …"`, naming the phase that will flip it. Scenarios that
   already pass become ordinary tests that document existing coverage.
5. **Findings note.** Record on this bead which modes reproduce and whether the narrower
   guards in the next two phases cover them. Say whether any mode, most likely in-place
   native rewrites, needs atomic replacement or versioned runtime instances. Do not
   build versioned runtime instances here. If the evidence says they are required,
   record a `PROPOSED FOLLOW-UP:` that cites the scenario.
6. **Measure.** Before, read-only on both hosts: count chop runs or error-ring rows from
   the last 7 days with exit -7, `No module named 'sase_core_rs'`, a missing job script,
   or `cannot import name`. Count these within 120 s of a recorded update or install.
   Record the after-install command.

## Phase `swap_chop_guards`: scheduler jobs defer across a swap (item 7.2, F7)

**Required changes.**

1. **Last-swap stamp.** Every writer path records a durable last-swap record next to the
   lock on release: op, `started_at`, `completed_at`, and a monotonically increasing
   generation. The paths are `code_swap_writer_lock` and the installer's
   `acquire_swap_lock`; read `tools.md` before touching `tools/`. Readers tolerate a
   missing or corrupt stamp as "no recent swap".
2. **No starts during a swap.** `_run_script_chop_once` records an inhibited skip
   (`status="skipped"`, `skip_reason="inhibited"`, `advances_cadence=False`) whenever a
   live writer holds the lock, not only when the script is missing.
3. **Missing-script grace.** A script that is missing within a short grace window after
   the last swap completed counts as inhibited too, for example 120 s; make it a named
   constant. After the window it is the loud `missing_script` error, as today.
4. **Post-exit classification.** Some runs overlap a swap: they started before the last
   swap completed and finished after that swap started, or a writer was live at exit. If
   such a run exits by SIGBUS (-7) or its output matches the existing swap-skew
   patterns, record it as an inhibited skip whose reason names the swap. It retries next
   tick and appends nothing to the error ring. The patterns are in
   `src/sase/axe/runner_failure_facts.py:33-56` and `src/sase/axe/source_skew.py:134`:
   `No module named 'sase_core_rs'`, `cannot import name`, missing binding, and "wire is
   stale". Reuse those helpers rather than adding a parallel pattern list. A run that
   did not overlap a swap keeps its failure status.
5. **Flip xfails.** Turn the reproducer's chop xfails into passing tests. Add focused
   tests for each classification, for grace expiry, and for a non-overlapping failure
   that stays a failure.
6. **Docs.** Update the docs page that describes the code-swap lock or dev update.

## Phase `swap_launch_guards`: launches and finalizers defer across a swap (item 7.2, F7)

**Today.**

- `sase bead work` takes a blocking reader lock (`src/sase/bead/cli_work_entry.py:35`).
- The epic launch exec wrapper is `src/sase/bead/epic_launch.py:390`.
- Agent runners register as advisory readers
  (`src/sase/axe/run_agent_runner_launch.py:400`).
- Skew errors are only labelled, by `code_swap_explanation` in
  `run_agent_runner_errors.py:70` and `run_agent_exec_plan_accept_sdd.py:207`.
- Nothing retries an `ImportError` or a missing binding.

**Required changes.**

1. **Launch path.** An agent launch, epic launch, `sase bead work`, or runner start can
   fail with a missing binding or a swap-skew `ImportError`. When that happens while a
   writer is live, or within the swap grace from `swap_chop_guards`, it waits for the
   swap to end and retries once (bounded by `CodeSwapWait`, or a re-exec after the swap)
   instead of failing the launch.
2. **Long-lived processes.** Find which long-lived process raised the
   `BeforeSpawnCallback` failure, from the audit and logs. Then stop such processes from
   mixing module versions across a swap. Choose from the reproducer evidence between two
   approaches. One: record the swap generation at start, and when it advances, stop
   starting in-process work that lazily imports sase modules and request a clean
   restart. Two: eagerly import the lazily imported launch modules at process start.
   Either way, close the window between swap completion and the installer's scheduler
   restart.
3. **Finalizers.** Host completion finalizers that hit a missing binding during a swap
   defer and retry rather than fail.
4. **SIGBUS.** Act only if `swap_repro` triggered it. If the installing step rewrites
   the extension in place, make it write a temp file and rename it into place, or record
   a `PROPOSED FOLLOW-UP:` when that change belongs to a recipe or tool this phase
   should not own.
5. **Flip xfails.** Turn every remaining reproducer xfail into a passing test. Add
   focused tests for each launch-path deferral.
6. **Docs.** Update the matching docs page.

## Phase `prebuild_miss`: a cache miss is a miss (item 7.3, F7)

**Today.**

- `_rust_prebuild_install_step` (`src/sase/dev_update/_plan_reconcile.py:154-203`) runs
  `python -m sase.dev_update.prebuild consume`.
- That command returns `0 if outcome.hit else 1`
  (`src/sase/dev_update/prebuild.py:265`).
- `_run_rust_prebuild_step` (`src/sase/dev_update/reconcile.py:154-202`) records exit 1
  as a warned step, then builds from source.
- The producer's writer probe (`prebuild.py:167-180`) treats only `op == "dev.update"`
  as a writer.

**Required changes.**

1. **Exit codes.** `consume` exits 0 on a cache miss and emits `hit:false` with its
   reason, for example `commit-mismatch`. It exits non-zero only for real errors, such
   as an unreadable cache or a failed install.
2. **Miss status.** Reconcile records a miss as its own non-failure status, and
   dev-update history counts misses apart from failures. The fallback source build still
   runs.
3. **Writer probe.** The producer's probe treats `install.dev` and `install.pypi`
   writers as writers too.
4. **Tests.** Cover a miss, a hit, a real consume error, and the probe. Extend
   `tests/dev_update/test_prebuild.py`, whose commit-mismatch case is at :230.
5. **Measure.** Record the apollo baseline (18 of 37 step records failing with
   `hit:false`) and the read-only after-install command.

## Phase `tg_transport`: correct fallback and per-chunk retries (items 8.1–8.2, F8)

**Today.** These lines are in `src/sase_telegram/telegram_client.py` in sase-telegram.

- **Retry wrapper.** `_with_retry` (30-73) makes 4 attempts and re-raises `BadRequest`.
  It retries `TimedOut` and `NetworkError` with 2, 4, and 6 s backoff.
- **Fallback.** `_send_single_message` (107-142) catches every exception, including
  `TelegramCredentialError` from `_get_bot()` inside its `try`. When `parse_mode` is
  set, it resends as plain text.
- **Whole-message retries.** `send_message` (145-175) wraps the whole multi-chunk loop
  in `@_with_retry`, so one failing chunk resends every earlier chunk. A chunk that
  keeps failing costs up to 8 HTTP sends.
- **Edits.** `edit_message_text` (298-334) has the same catch-all fallback.
- **Library.** PTB 22.8: `BadRequest` and `TimedOut` both subclass `NetworkError`, and
  HTTP 502 is a plain `NetworkError`.

**Required changes.**

1. **Fallback trigger.** Fall back to plain text only for a `BadRequest` whose message
   is an entity-parse failure, checked before any `NetworkError` handling. Every other
   error propagates unchanged to the retry wrapper or the caller. That includes
   `TimedOut`, `NetworkError` and Bad Gateway, `RetryAfter`, `Forbidden`,
   `InvalidToken`, and `TelegramCredentialError`. Move `_get_bot()` out of the guarded
   block. Apply the same rule to `edit_message_text`.
2. **Per-chunk retry.** Retry each chunk on its own, so a chunk that was delivered is
   never resent and `reply_markup` stays on the last chunk. When a chunk exhausts its
   retries, raise an error that carries how many chunks were delivered.
3. **Resume.** The outbound job
   (`src/sase_telegram/scripts/sase_tg_outbound.py:515-535`) stores that delivered-chunk
   count in its per-notification delivery state. The next tick resumes at the first
   undelivered chunk instead of resending the whole notification.
4. **Tests** in `tests/test_telegram_client.py` and `tests/test_outbound.py`:
   - a parse `BadRequest` falls back;
   - a non-parse `BadRequest` does not;
   - `TimedOut` retries without falling back;
   - a multi-chunk failure retries only the failed chunk, with an exact HTTP send count
     of at most 4 per failing chunk;
   - a partial delivery resumes at the right chunk on the next tick.
5. **Verify.** Run `sase tool run check` from the sase-telegram checkout.

## Phase `tg_degraded`: one notice per outage, not 408 tracebacks (item 8.3, F8)

**Required changes.**

1. **States.** Classify failures into `credential_unavailable`
   (`TelegramCredentialError`) and `network_degraded` (`TimedOut`, `NetworkError`, and
   5xx Bad Gateway). Outbound runs as a fresh process every 5 s, so persist the state in
   a small file under the plugin's state dir. Record the state, `entered_at`,
   `last_seen`, and a count.
2. **Logging.** Entering a state logs one WARNING without a traceback. While degraded,
   log at most one summary line per window (15 min) with the suppressed count. Recovery
   logs one INFO line with the duration and count. Unexpected exceptions keep their
   tracebacks. Apply this to the outbound job (`sase_tg_outbound.py:479,515-535`, and
   `scripts/__init__.py:30-35`) and to the receiver
   (`sase_tg_inbound.py:466-470,507-511`), and keep the receiver's exit-75 credential
   retry contract.
3. **Notices.** Raise one SASE notification per transition into a degraded state, with a
   stable dedup key per state, through the plugin's existing notification path or sase's
   public notification API. Settle it on recovery. Outbound never tries to deliver its
   own degraded notice while degraded.
4. **Credential cache.** Negatively cache `get_bot_token` failures
   (`src/sase_telegram/credentials.py:25-98`) for a short window, so a failing
   `pass`/gpg lookup does not rerun on every tick. A token that becomes available is
   still picked up within one window.
5. **Tests.** Many consecutive credential failures yield one transition notice and a
   bounded number of log lines. Transient errors log no traceback. Recovery logs once
   and settles the notice. The negative cache expires.
6. **Measure.** Before: count athena's and apollo's `TelegramCredentialError` and
   `TimedOut` traceback lines over 7 days, read-only. Record the after-install command.

## Phase `tg_scope`: outbound only on the sender (item 8.4, F8)

**Today.** The `telegram` routine and its `tg_outbound` job are defined for every
machine in the chezmoi source `home/dot_config/sase/sase.yml:269-298`. The
`telegram_receiver` proc is scoped through the athena overlay
`home/dot_config/sase/sase_athena.yml:173-184`. Overlays are selected by machine name
(`src/sase/config/targets.py:66`, `src/sase/config/identity.py:114-123`).
`sase_apollo.yml:8-14` only notes that the receiver must not run on apollo.

**Required changes.**

1. **Move the routine.** Move `axe.routines.telegram` with its env, description, and
   comments into `sase_athena.yml`, next to the receiver, and keep the mirrored-env
   comment accurate. Update the apollo overlay's comment.
2. **Check merge semantics.** Confirm from code or tests that a routine defined only in
   a machine overlay loads on that machine. Validate the edited config with sase's
   config validation, without applying it. If overlay routines are not supported, record
   a `PROPOSED FOLLOW-UP:` and leave the config unchanged.
3. **Measure.** Before: apollo's `tg_outbound` spawn rate (about 12.7 a minute),
   read-only. After `chezmoi apply` on each host, which the human runs: the command that
   confirms apollo and mac record no `tg_outbound` runs and athena still does.

## Phase `bundle_trim`: compile-time compaction under `codex_bundle = trim` (item 9, F10)

**Today.** The bundle is built in Python, with no Rust involved:

- `compile_bundle` (`src/sase/instructions/compile.py:237`) calls `assemble_sections`
  (`sections_assemble.py:116`).
- Web rosters are re-rendered from strands at compile time by
  `render_web_body_with_roster` (`src/sase/main/init_memory/root_planning.py:290`). That
  uses `render_strand_roster` in `src/sase/memory/web/roster.py:94`, which has `inline`
  and `list` styles.

The size check measures the delivered text, which is the bundle plus the marker line
(`src/sase/llm_provider/_instruction_boundary.py:404-408`;
`instruction_delivery.py:203-209`). When the text is over the cap, the slot falls back
to `inherits_native` with reason `oversize`, and a daily deduped notification fires
(`instruction_delivery.py:107-124,308-359`). Two providers declare caps: Codex at 32 KiB
(`src/sase/llm_provider/codex.py:213`) and Grok at 120 KiB (`_grok_provider.py:199`).
Claude and Muse declare none.

**Required changes.**

1. **Compaction ladder.** Compile knows the provider. When that provider's root slot
   declares `max_bytes` and the delivered text would exceed it, apply these steps in
   order and stop at the first one that fits:
   - (a) Superseded strands in `roster: list` webs render as one compact line: number,
     title, slug, and the supersession marker, with no summary.
   - (b) Whole `roster: list` webs render compact, one line per entry with number,
     title, and slug only. Start with the largest web.
   - (c) If the bundle still does not fit, today's oversize fallback and notification
     run unchanged.

   Each compacted web ends with one short line saying that summaries were omitted to fit
   the provider budget, and pointing to `sase memory read <web>:<keyword>`.

2. **No change when it fits.** A bundle that already fits is byte-identical to today, so
   sase's committed `AGENTS.md` projection and `test_parity_committed_agents_md`
   (`tests/instructions/test_render_parity.py:354`) do not change. Compaction is a
   compile-time projection only: `render_web_descriptor_with_roster` and every
   descriptor on disk stay as they are, and no memory file changes.
3. **Reporting.** Record the applied step and the uncompacted size in the compiled
   section table and manifest, so `sase instructions render -j` and `bundle_gate` can
   report them.
4. **Cache key.** Bump `COMPILER_VERSION`, or add the cap to the cache key inputs, so a
   cached uncompacted bundle is never served to a capped provider.
5. **Flags.** Leave provider flag gating alone (sase-1kt/1ku/1kv).
6. **Tests.** Use synthetic projects:
   - a list roster that pushes a Codex bundle over 32 KiB, which step (a) brings under;
   - a larger roster that needs step (b);
   - an oversize core note, which still falls back;
   - an under-cap bundle, which stays byte-identical;
   - Claude, with no cap, which is unaffected.
7. **Measure.** Run `sase instructions render -f provider=codex -j` for sase and for
   bob-cli, from bob-cli's read-only external checkout, before and after. bob-cli starts
   at 36,225 bytes and should fit after step (a).

## Phase `bundle_gate`: catch bundle growth at authoring time (item 9, F10)

**Required changes.**

1. **Doctor check.** Add a check, for example `instructions.bundle_size`, in
   `src/sase/doctor/checks_instructions.py`. For every enabled project and every
   provider whose root slot declares `max_bytes` with its explicit channel enabled, it
   compiles the root bundle the way `sase instructions render` does. It reports ERROR
   when the delivered text exceeds `max_bytes` after compaction, because that launch
   will fall back. It reports WARN when compaction was needed, naming the step and the
   uncompacted size. Otherwise it reports OK with sizes. Enumerate projects the way
   `src/sase/doctor/checks_resources_ace_run_watches.py:86` does. Measure the runtime,
   and mark the check `deep=True` if it is slow.
2. **just-check test.** Add a test reachable from `just check` asserting that sase's own
   root bundle fits every capped provider's `max_bytes` without compaction, so sase's
   growth is caught while authoring. On failure it names the largest sections.
3. **Docs.** Update `docs/reference/memory/instruction-bundles.md`,
   `docs/contribute/instruction-bundle-internals.md` (oversize now falls back only after
   compaction), and `docs/reference/providers/codex.md:31-32`. Also update the doctor
   checks page if one lists checks. `docs/instruction_bundles.md` is a moved-page stub,
   so leave it alone.
4. **Measure.** Before, read-only on both hosts: count
   `instructions:fallback:*:codex:oversize:*` notifications and their +1s over 7 days.
   After install, give the command that confirms no new Codex oversize fallbacks.

## Phase `triage_conflict`: recover from `request_id_conflict` (item 10, F9)

**Today.** In `src/sase/scripts/sase_chop_bead_task_triage.py:507-523`, a gate-creation
failure logs and `continue`s without advancing `generation`, so the deterministic
`_request_id(project, bead, generation, kind)` repeats forever.
`_refuse_request_id_conflict`
(`src/sase/notification_gates/service_creation.py:313-338`) raises
`GateError("request_id_conflict", …)` (`model_validation.py:45`) when a stored journal
or creation result under that id has a different spec fingerprint. That happens, for
example, when a create stopped after `bundle_written` or lost its state write and the
bead body changed afterwards.

**Required changes.**

1. **Recover from the conflict.** On a `GateError` with `code == "request_id_conflict"`:
   - cancel any pending gate under the conflicting id with
     `_cancel_pending_gate(kind, request_id, reason="request_id_conflict")`, tolerating
     a gate that is not pending;
   - advance the generation and persist it in state even if the retry fails;
   - retry the create once in the same pass with the new id.

   Allow at most one advance per bead per pass. Every other failure keeps the
   deterministic retry and leaves the generation unchanged.

2. **Logging.** Log one line per conflict naming the old and new ids.
3. **Scope.** Keep the change inside the gate-creation branch (see the sase-1tz.4
   coordination note above).
4. **Tests.** Put them alongside
   `tests/test_axe_chop_bead_task_triage_gate_lifecycle.py`:
   - a bead whose body changes between a failed create and the retried create gets its
     gate on the next pass, under generation + 1;
   - the stale pending gate is canceled;
   - a non-conflict failure keeps the same request id;
   - repeated conflicts stay bounded per pass.
5. **Measure.** Before: count "request id already belongs to a different gate
   specification" lines in athena's chop logs over 7 days, read-only (the audit counted
   1,517). Record the after-install command.

## Landing (child epic land agent)

1. Confirm that every phase bead is closed and that its notes are accounted for.
2. Confirm that each measuring phase recorded its before-numbers and after-install
   command on its own bead, with a one-line summary on **sase-1tz.3**.
3. Run `just check` on the combined tree. Confirm the `sase-core-revision.txt` pin
   includes the `prose_wire` sase-core commit. Confirm that sase-telegram's
   `sase tool run check` passed for `tg_transport` and `tg_degraded`, and that the
   chezmoi change from `tg_scope` is committed in that repo.
4. Confirm that no reproducer test is still marked xfail. A scenario left xfail must
   carry a recorded `PROPOSED FOLLOW-UP:` naming the design it needs.
5. Triage the phases' `PROPOSED FOLLOW-UP:` notes.
6. Close **sase-1tz.3** with a note summarizing the verified result.

Never close the parent epic **sase-1tz** or any other ancestor.
