---
tier: tale
title: Prevent local formatting setup from exhausting stitch finalization
goal: Finish stitch formatting without installing or rebuilding the application runtime,
  and preserve actionable hook diagnostics when finalization times out.
size: medium
proposed_by: bbugyi200.athena.0k3
status: done
---

# Plan: Fix stitch timeouts in the local commit hook

## Outcome and scope

Make `just fix` safe to run during host commit finalization even when the workspace's
application virtualenv or Rust extension is missing or stale. Preserve Python fixes,
generated documentation, Markdown formatting, and keep-sorted behavior. If a hook still
hangs, the failed agent must identify the hook, elapsed time, and retained output rather
than reporting only `sase stitch create stitch_timeout for main`.

This is a medium tale: the formatting prerequisites, subprocess diagnostics, and their
regressions form one bounded implementation in the `sase` repository. No GitHub plugin
change or new retry policy is justified by the observed failure. Do not increase the
30-minute finalizer limit or automatically replay a timed-out mutating stitch.

## Investigation and evidence

The user's example is the `0jv` family, specifically `0jv--code`, run `20260912052202`,
on September 12, 2026. `sase agent show 0jv` resolves this completed, failed run even
though the recent-agent listing omitted it. There is no saved coder chat; stable run
artifacts supply the evidence. Their directory is
`~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202609/12/20260912052202/`.

- `done.json`, `workflow_state.json`, and `finalizer_result.json` show that the model
  invocation completed and the host's `builtin@commit` failed with `stitch_timeout`.
  `LLMInvocationError` is the outer error wrapper, not evidence of an LLM outage.
- `finalizers/commit/attempt-1.main.stdout` contains exactly
  `Running before commit hook: just fix` (with the progress glyph). The corresponding
  stderr is empty. The workflow never printed
  `Dispatching create_commit to VCS provider...`.
- `finalizers/commit/attempt-1.main.inputs.json` identifies the primary repository
  attempt, its three changed usage-related files, and baseline HEAD
  `4c0d1c216ce72dad2380bc0526ef6389108cae52`. The message filename timestamp places
  stitch startup at approximately 09:38:06 UTC; attempt output was persisted at 10:08:06
  UTC. This matches the 1,800-second hard subprocess limit.
- `commit_results.json` contains only the earlier `sdd_commit` for the plans repo. There
  is no primary-repository completion marker. An unrelated sidecar commit must never
  rescue a timeout for the main repo.
- `tool_calls.jsonl` records `just install` starting at 09:32:40 UTC, a wheel-cache
  miss, and a release build of `sase_core_rs` still compiling near the model's final
  submission. The run's saved output also explicitly reports unfinished installation and
  verification. The still-present `/tmp/just_install.log` identifies this run's core
  build and ends with `install`/`rust-install` terminated by signal 15, with mtime near
  09:38:02 UTC. That earlier install did not complete successfully.
- Four additional retained failures stop at the same pre-hook line: `05a--code`
  (`20260907162421`), `sase-zq.1` (`20260911153440`), `sase-zn.1` (`20260911153609`),
  and `sase-zn.4` (`20260911153612`). A separate timeout for `sase-yf.2`
  (`20260908092842`) printed successful dispatch first; do not conflate that
  post-dispatch failure with this cluster or weaken existing landed-commit rescue.

The source confirms the causal design defect:

1. `Justfile`: `fix -> fmt-py / fmt-docs -> _setup` runs full application setup.
   `_setup` can refresh the linked core checkout, validate the extension, call
   `rust-install`, and install the entire editable dev environment and plugins.
   `rust-install` can invoke `maturin develop --release` and build the LSP too. The same
   dependency chain exists at the failed run's baseline HEAD.
2. `src/sase/workflows/commit/workflow.py` runs this before-hook before VCS dispatch.
3. `src/sase/workflows/commit/command_hooks.py` uses an unbounded
   `subprocess.run(..., capture_output=True)`. Its child output is printed only after a
   nonzero exit. If the outer stitch is killed, that in-memory output disappears.
4. `src/sase/finalizers/commit_repair.py` runs stitch through the 1,800-second bounded
   subprocess helper. `commit_dispatch_followup.py` emits the generic timeout text when
   no matching completion marker exists, discarding available elapsed/output context
   from the error message.

Conclusion: the example is a local pre-hook stall, not a failed GitHub commit/push.
Hidden runtime setup is the strongest supported explanation for its excessive runtime.
The exact child operation inside this historical hook cannot be proved because its
output was discarded; do not claim a demonstrated Cargo deadlock, network outage, or
specific compiler stall. Confirm the unwanted setup path with deterministic spies in the
regression harness, and fix that dependency plus the evidence loss.

The GitHub provider was inspected through `sase repo open gh:sase-org/sase-github`; its
inherited commit path is downstream of the point this attempt reached. No code in that
repository needs changing.

## Implementation

### 1. Separate formatting tools from application setup

Update `Justfile` so the complete `fix`/`fmt` recipe graph, including `fmt-py`,
`fmt-docs`, and relevant formatting checks, uses a small formatting-tool bootstrap
instead of `_setup`. A private formatting environment should isolate these tools from
the application `.venv`, including an installation already running there. Honor the
existing ability to override environment paths and avoid cross-workspace/global tool
installs. Use the repository's dependency declarations and lockfile for the required
Ruff and YAML tooling; add a narrow development tooling group if needed, and regenerate
the lockfile without unrelated dependency churn. Install only these tools, not the
`sase` distribution and its runtime dependencies.

Retain the existing pinned Prettier and keep-sorted bootstraps, while ensuring they do
not depend on application setup. Repeated formatting with tools already installed must
not reinstall them or access the network. A missing formatting tool should be
bootstrapped or fail with the actual tool/setup error; it must never silently skip a
formatter. Cold formatting can still need package downloads, which the hook logs will
make visible.

`tools/render_model_alias_docs` currently imports `sase.llm_provider`, which eagerly
imports much of the application. Remove that runtime dependency from the documentation
rendering path. Read the shipped `src/sase/llm_provider/model_alias_defaults.yml` as
presentation input using the small tooling environment. Preserve ordering, escaping,
generated-block validation, descriptions, and target/fallback text. Do not duplicate
model-selection, alias-resolution, or selector-grammar behavior: this tool renders
declared strings only, while runtime validation remains authoritative. Add a parity test
against the existing runtime accessors under the full test environment.

Leave `_setup` on application test/lint/install paths that actually require the Rust
extension. Do not add a Python backend fallback or weaken core-version verification.
Document the distinction between formatting setup and runtime setup in the existing
developer documentation or recipe comments, not in canonical memory files.

### 2. Retain hook output while the hook is running

Replace capture-until-exit in `command_hooks.py` with incremental draining to bounded,
durable logs. Record command, phase, repository identity, start time, and invocation
identity before launch; persist stdout/stderr as they arrive, with explicit truncation
information and a useful recent tail. Store run-bound logs under the agent artifacts
directory when available. For direct CLI use, provide an appropriate temporary log
location and retain failed-hook evidence. Avoid collisions between repositories,
attempts, before/after hooks, and resume invocations; completed logs are immutable.

Keep normal success output concise. Print the hook-log location at startup and useful
phase-specific status at exit, so the outer stitch's existing captured output points to
evidence even if termination prevents cleanup handlers from running. Do not forward
unbounded hook output into the outer finalizer stream: verbose formatting must not
create a new 1 MiB `stitch_output_cap` failure. Bound memory and disk use, handle both
streams without deadlock, and make partial output readable during execution.

Preserve the host's process-group cancellation: hooks must not escape into a detached
session that survives the outer timeout. Reuse existing subprocess primitives where they
fit, but do not blindly nest a helper that creates a new session. Keep the existing
overall time/attempt limits and before/after hook return semantics. No new user-facing
timeout configuration is needed for this fix.

### 3. Make timeout diagnostics explain the failure

Extend `record_stitch_artifacts` to persist outcome metadata separately from immutable
input fingerprints: return code, duration, timed-out flag, and stdout/stderr truncation
flags. Preserve existing artifact names and old-attempt readers.

Factor bounded-failure message construction so initial stitch, post-repair follow-up,
and resume paths report repository, elapsed/allowed time, available last-stage context,
and bounded output tails/log paths consistently. Prefer structured hook metadata for the
last hook state, validating that it belongs to this repository/invocation, rather than
inferring success or dispatch state from free-form output. If no hook record exists,
report the available stitch output and an unknown stage without guessing.

Keep `stitch_timeout`, `stitch_output_cap`, and existing after-commit diagnostic codes
and retry classification stable. Matching commit markers must still pass normal
postcondition verification, checkpoints must retain their existing resume behavior, and
an unrelated sidecar marker must never turn the main timeout into success. Do not add
automatic full-stitch retries, model retries, or GitHub backoff for these errors.

This work is Python subprocess/filesystem glue and developer tooling. It does not
introduce a shared backend policy and should not require a Rust wire/API change. If
implementation uncovers a necessary shared policy change, open `sase-core` through
`/sase_repo` and follow the documented core boundary instead of duplicating it here.

## Verification and acceptance

Use deterministic fixtures with short timeouts and temporary repositories. Do not
reproduce the historical failure by launching a 30-minute build or mutating `0jv`'s old
workspace. Extend these existing areas as appropriate: `tests/test_justfile_lint.py`,
`tests/test_justfile_sase_core_dir.py`, `tests/test_commit_hooks.py`,
`tests/test_commit_workflow_hooks.py`,
`tests/test_finalizers_commit_repair_fidelity.py`,
`tests/test_commit_dispatch_stitch_timeout_rescue.py`, and
`tests/test_finalizers_execution_ledger.py`.

Required behavioral coverage:

1. In an isolated recipe fixture with a missing/stale application extension and
   formatting tools available, `just fix` invokes every expected formatter and doc
   renderer, without `_setup`, core refresh, Cargo, maturin, application installation,
   or plugin installation. Use spies that fail on forbidden invocations, not just
   assertions on recipe text. Include the complete dependency graph, repeated warm
   invocation, and custom environment-path behavior.
2. The docs renderer works using only its declared tooling dependencies, produces the
   same shipped-default table as the runtime-backed source, and remains idempotent.
   Retain malformed-input and Markdown escaping/error behavior without adding a second
   runtime alias validator.
3. A real test hook writes to both streams, signals readiness, and blocks. A short outer
   timeout terminates the stitch and its child; logs contain the emitted output even
   though the hook never returns. Coordinate readiness without fragile sleeps.
4. Large hook output stays bounded and preserves useful recent lines without deadlocking
   or turning a successful verbose hook into an outer output-cap failure.
   Concurrent/multiple hook invocations get distinct evidence. Normal nonzero exits
   include command, phase, and useful output; empty hooks remain no-ops.
5. Before-hook failure prevents provider dispatch. After-hook failure retains the
   existing checkpoint/recovery semantics. Timeout messages and the generated agent
   error report expose elapsed time, relevant hook identity, and evidence location;
   outcome metadata records the timeout and caps.
6. Existing landed-commit rescue, unrelated-marker rejection, protected paths,
   fingerprint guards, and non-retryable timeout tests pass with richer messages.
   Include initial and resume/follow-up failures so the generic message is not left
   behind on another route.

Read `lint_and_test.md` through `/sase_memory_read` before finishing implementation. Run
the focused tests, then `just check`. Because `Justfile` can broaden selection, inspect
the selected lane and run `just check-full` through `/sase_monitor` if the repository's
broadening rules or selection escalation require it; full verification also remains a
landing prerequisite. If preparing the application test environment or verification
takes a long time, hand it to `/sase_monitor` and resume mechanically. Do not leave an
unmanaged background install running and then finalize an unverified change, as the
example run did.

The final implementation report should state that the demonstrated failures occurred in
the local formatting hook, distinguish the inferred setup subcause from proven evidence,
list the tests actually completed, and call out the absence of a live GitHub-outage
reproduction. This plan authoring turn changes only this scratch plan; implementation
starts after plan approval.
