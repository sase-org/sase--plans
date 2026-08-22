---
tier: epic
title: Restore green GitHub Actions across source, visual, performance, and release
  lanes
goal: 'GitHub Actions uses complete source-built Rust artifacts, portable test assertions,
  deterministic visual cursor normalization, evidence-based performance floors, and
  safe release-lock source validation so every currently failing SASE lane passes
  without weakening product contracts or accepting unintended generated changes.

  '
phases:
- id: ci-runtime-artifacts
  title: Ship the source-built xprompt LSP to every CI consumer
  depends_on: []
  size: medium
  description: 'ci-runtime-artifacts: build, publish, install, and contract-test the
    xprompt LSP beside the Rust wheel from the same sase-core revision.'
- id: portable-cli-contracts
  title: Make CLI and skills rendering assertions environment-independent
  depends_on: []
  size: small
  description: 'portable-cli-contracts: reuse portable metavar assertions and tolerate
    Rich line wrapping while retaining the tested user-facing contracts.'
- id: visual-cursor-convergence
  title: Eliminate stale cursor paint from visual snapshots
  depends_on: []
  size: medium
  description: 'visual-cursor-convergence: normalize focused and blurred input cursor
    caches before accepting a converged visual frame.'
- id: query-perf-floor
  title: Recalibrate the persistent-query absolute performance floor
  depends_on: []
  size: small
  description: 'query-perf-floor: add an evidence-backed per-anchor ceiling while
    preserving the hardware-independent Rust-versus-Python gate.'
- id: release-lock-normalization
  title: Accept equivalent canonical PyPI registry spellings in the release ratchet
  depends_on: []
  size: small
  description: 'release-lock-normalization: normalize only the trailing-slash-equivalent
    PyPI simple-index source and keep all non-PyPI lock rewrites fail-closed.'
- id: integration-verification
  title: Reproduce every failed lane and run exhaustive verification
  depends_on:
  - ci-runtime-artifacts
  - portable-cli-contracts
  - visual-cursor-convergence
  - query-perf-floor
  - release-lock-normalization
  size: medium
  description: 'integration-verification: combine the repairs, run focused reproductions
    plus full repository and visual gates, and inspect the resulting Actions signal.'
proposed_by: bbugyi200.athena.0al
create_time: 2026-08-22 12:30:18
status: wip
bead_id: sase-s1
---

- **BEAD:** [sase-s1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-s1/README.md)

# Plan: Restore green GitHub Actions

## Diagnosis and scope

`actstat` identified two families of failures on current master history:

- Publish run `32569838194` failed in `sync-release-metadata` because
  `tools/ratchet_core_window --allow-transitive-lock-refresh` rejected the refreshed
  `asttokens` stanza as a non-PyPI source. The release branch's input stanza is the
  canonical PyPI simple index, and the guard compares its registry string byte-for-byte.
  `uv` has emitted both trailing-slash spellings of that same index. The repair must
  accept only that semantic no-op and continue rejecting path, Git, alternate-registry,
  and malformed sources.
- CI run `32558460537`, plus the newer partially settled run `32568874089`, exposed
  independent source-test, visual, and performance failures. Later master commits
  already repair the `%final` completion contract, contract-manifest budget, and prompt
  editor visibility regression seen in the older logs; do not reimplement or revert
  those fixes.

The remaining root causes are:

1. `build-core` uploads only `sase_core_rs`, while the newly split directive/finalizer
   parity suites require `.venv/bin/sase-xprompt-lsp`. All source lanes consume the
   prebuilt wheel but never receive the matching LSP binary, producing dozens of
   identical missing-binary failures. The LSP in sase-core master already contains the
   current `%final` behavior, so no sase-core source change is needed.
2. Several help tests assume one `argparse` spelling for short and long options with a
   metavar. Supported Python patch releases render either `-m MODEL, --model MODEL` or
   `-m, --model MODEL`; `tests/main/parser_help_helpers.py` already provides the helper
   that accepts both. The retired-skill rendering test likewise assumes a full temporary
   path remains contiguous even when Rich wraps a long CI path at column 160.
3. The visual artifact contains 347 broadly distributed mismatches with the same
   signature: a blurred background `TextArea` retains a painted cursor while a modal or
   other focused control also has the correct cursor. The convergence helper clears
   `TextArea._line_cache` and queues a repaint only for the focused widget, so a line
   cached while the now-blurred widget was focused can remain visible indefinitely when
   cursor blinking is disabled.
4. The performance report fails only
   `evaluate_query_many.synthetic_1000_specs.persistent_query_keystroke`'s absolute
   93.38us ceiling. Eight consecutive hosted reports measure 178.28-184.36us, and a
   clean local run reproduces 147.07us, while the same reports still show the shipped
   Rust route beating Python by roughly 29-30x. The historical 66.70us capture is not a
   portable absolute baseline for this microsecond-scale anchor; this is consistent
   hardware calibration drift rather than a new core regression.

The implementation stays in the SASE repository. The linked sase-core checkout is an
input used to build matching artifacts; do not modify it unless new evidence proves a
core defect, in which case stop and revise the plan before crossing the repository
boundary.

## Phase 1: Ship the source-built xprompt LSP to every CI consumer

Update `.github/workflows/ci.yml` so `build-core` compiles the `sase_xprompt_lsp`
release binary from the same checked-out sase-core SHA used by `maturin`, stages it in
the existing core artifact directory, and uploads it with the wheel and provenance file.
Reuse the existing Rust cache and hardened Cargo network settings where appropriate; do
not add a second sase-core checkout to consumer jobs.

Update `.github/actions/setup-sase/action.yml` to require exactly one wheel, one
provenance file, and one `sase-xprompt-lsp` binary. After the selected install recipe
creates `.venv`, atomically install the binary at `.venv/bin/sase-xprompt-lsp`, mark it
executable, and run its lightweight probe. Keep exporting `SASE_CORE_WHEEL` for later
`just` recipes. A malformed or partial artifact must fail with a targeted diagnostic
instead of silently falling back to a stale PATH binary.

Extend `tests/test_github_actions_ci.py` to pin the producer/consumer contract: one core
checkout and revision, both Rust deliverables built from it, both carried by the shared
artifact, and the LSP installed and probed by the composite action. Exercise
`tests/test_xprompt_directive_completion_parity.py` and
`tests/test_xprompt_finalizer_completion_parity.py` against a freshly built local LSP to
prove that the artifact closes the exact failure rather than skipping parity coverage.

Acceptance criteria:

- Every job using `.github/actions/setup-sase` receives the wheel and LSP from one
  recorded sase-core SHA.
- A missing or duplicate LSP artifact fails setup explicitly.
- Directive and finalizer parity suites pass with the freshly built LSP.
- No source lane clones or compiles sase-core independently.

## Phase 2: Make CLI and skills rendering assertions environment-independent

Replace hard-coded metavar substrings in the failing pipe, memory, completion, and proc
help tests with `assert_metavar_option_documented`. Supply each option's actual metavar,
including choices and `FILE`/`NAME` values, and retain the surrounding semantic and
parser-result assertions. Apply the helper consistently to duplicate pipe coverage so
the suite does not keep one patch-version-sensitive copy.

In `tests/main/test_skills_handler.py`, keep proving that both retired source and live
paths are rendered, but compare against a whitespace-normalized rendering (or an
equivalently precise wrapped-path assertion) so Rich line breaks in long pytest scratch
directories do not change the result. Do not widen the production table or drop either
path assertion.

Acceptance criteria:

- The focused parser and handler tests pass under supported Python 3.12 and 3.14 patch
  releases.
- The tests still fail if an option, metavar, retired status, source path, or live path
  is genuinely absent.
- No production CLI help or skills rendering behavior changes in this phase.

## Phase 3: Eliminate stale cursor paint from visual snapshots

Repair `tests/ace/tui/visual/_ace_png_snapshot_waits.py` so every visual convergence
cycle derives cursor visibility from current focus for both `Input` and `TextArea`, and
invalidates/repaints a `TextArea` whose cached line may contain the opposite cursor
state. Ensure the helper tells `wait_for_visual_idle` when a queued repaint must be
drained before SVG sampling. Keep this normalization confined to the visual test harness
and use Textual's current cache behavior deliberately; do not alter ACE product focus or
cursor behavior.

Add a focused regression in `tests/ace/tui/visual/test_visual_idle.py` that constructs
or models a formerly focused, now-blurred text area with a stale cursor-bearing cache
and proves the convergence barrier clears/repaints it before accepting the frame. Cover
the focused-cursor case as well so the existing caret visibility contract remains
intact.

Validate one modal snapshot that previously showed two cursors, then run the complete
PNG suite. Inspect actual/expected/diff artifacts for any residual class of mismatch. Do
not update golden PNGs: the current expected frames are correct, and mass rebaselining
would conceal the race.

Acceptance criteria:

- The regression test fails against the old focused-only invalidation logic and passes
  with the repair.
- Formerly focused background editors render without a caret; the focused editor keeps
  its caret.
- `just test-visual` passes with no accepted snapshot updates.

## Phase 4: Recalibrate the persistent-query absolute performance floor

Add a documented per-anchor slowdown-factor override in
`tests/perf/baselines/phase7_regression_floor.json` for
`evaluate_query_many.synthetic_1000_specs.persistent_query_keystroke`. Derive the
smallest defensible factor from the eight consecutive hosted medians (178.28-184.36us)
and the local 147.07us reproduction, leaving narrow headroom above the observed hosted
maximum. Record the run IDs, observed range, resulting ceiling, retained detection
margin, and a condition for revisiting the override.

Do not change `must_beat_python`: the same-machine relative comparison is the portable
product-performance contract and currently has a very large margin. Do not add retries
for a consistently reproduced measurement or alter the benchmark workload. Extend the
baseline/phase-7 tests if needed to pin the override and its explanation.

Acceptance criteria:

- `just phase7-perf-check` passes on the unchanged product implementation.
- The persistent route must still beat the live Python comparison in the same process.
- The new absolute ceiling remains tight enough to fail a material regression beyond the
  observed hosted distribution.

## Phase 5: Accept equivalent canonical PyPI registry spellings in the release ratchet

In `tools/ratchet_core_window`, replace the exact registry-string comparison with a
small, explicit canonical-PyPI predicate that treats only `https://pypi.org/simple` and
`https://pypi.org/simple/` as equivalent. Apply it to both the before and after
transitive package stanza checks, retain artifact URL/version/hash validation, and
include the rejected source value in the failure diagnostic without exposing
credentials.

Extend `tests/test_ratchet_core_window_tool.py` with the live `asttokens` shape and an
after-refresh no-trailing-slash registry case. Preserve negative tests for local paths,
Git sources, alternate hosts, credentials, query/fragment variants, malformed source
tables, direct-dependency changes, and unexpected package metadata. Reproduce the
release-branch ratchet in a disposable checkout or scratch project so validation cannot
mutate the live release branch.

Acceptance criteria:

- The transitive `asttokens` refresh succeeds when the sole source difference is the
  canonical PyPI trailing slash.
- All non-PyPI or ambiguous source rewrites still fail closed and restore input files.
- The release-branch reconciliation completes through the final `uv lock` idempotence
  check in scratch state.

## Phase 6: Reproduce every failed lane and run exhaustive verification

Review the combined diff for accidental overlap and verify that no generated visual
goldens, release-branch files, linked-repository files, or unrelated user changes were
included. Run `just install` before repository checks, as required for an ephemeral SASE
workspace.

Re-run the exact focused failures first:

- workflow contract plus directive/finalizer parity suites with the fresh LSP;
- parser help and skills retired-drift tests under Python 3.12 and 3.14;
- visual convergence regression and one representative formerly double-cursor modal;
- `just phase7-perf-check`;
- release ratchet contract tests and a scratch release-branch reconciliation.

Then run `just check-full` through `/sase_monitor` with `TESTING`/`TESTED` statuses and
a concrete continuation action, because this epic changes CI-wide test infrastructure.
Run `just test-visual` as the separate exact-pixel gate. If either exposes a genuine new
root cause, fix it in the owning phase, repeat that phase's focused reproduction, and
rerun both integration gates until clean. Do not paper over unrelated confirmed CI
failures; use `/sase_new_task` only for genuinely out-of-scope discoveries after
checking duplicates and causal links.

After the repository changes enter the normal commit/push workflow, use `actstat` to
inspect the fresh Publish and CI runs. Distinguish superseded/cancelled runs from
settled failures, download any new failure artifacts, and repeat local reproduction plus
the relevant gate until the new head is green or an external service condition is
proven.

Acceptance criteria:

- All focused reproductions pass.
- Monitored `just check-full` and full `just test-visual` pass on the combined tree.
- A fresh Actions run no longer reports the diagnosed Publish, source-test, visual, or
  performance failures.
