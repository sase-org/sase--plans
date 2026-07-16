---
tier: tale
title: Fix all failing CI jobs on sase master
goal: 'Every GitHub Actions CI job on sase master passes again: the phase7-perf-floor
  import error is fixed, the ACE PNG visual goldens render identically on every supported
  Python version, the Python 3.12 test leg no longer fails on argparse help-format
  assertions, and the visual wait helpers no longer flake under the coverage-instrumented
  matrix leg.

  '
create_time: 2026-07-16 13:20:59
status: done
prompt: 202607/prompts/fix_master_ci_failures.md
---

# Plan: Fix all failing CI jobs on sase master

## Context

CI on `sase-org/sase` master has been red since 2026-07-11 (last green run: commit `5df88d7ca`). The most recent
completed run (`29512859968`, commit `0d33d2a8c`) plus near-HEAD in-progress runs show exactly three failing job groups;
all other jobs (lint, build, docs, install-smoke, launch-perf-floor, bead-backend) are green. Root causes were confirmed
by reproducing locally and by downloading the CI visual-failure artifacts.

The unifying theme of two of the four failure classes: the local dev venv runs Python 3.14 while CI runs the visual
suite and part of the test matrix on Python 3.12, and some test expectations were authored against 3.13+-only behavior.
The renderer-environment fingerprint (`tests/ace/tui/visual/renderer_env.json`) pins packages, fonts, and platform, but
records the Python version as _diagnostic only_ — so version-coupled golden content slipped through.

### Failure classes at HEAD

1. **`phase7-perf-floor`** — `just phase7-perf-check` fails with
   `ImportError: cannot import name '_done_from_snapshot' from 'sase.agent.running'` raised from
   `tests/perf/bench_agent_scan.py` (`_run_scenarios`). The helpers `_done_from_snapshot` and `_running_from_snapshot`
   moved to `src/sase/agent/running_listing.py` in the running-agent listing split (`470c4b023`), and the follow-up
   visibility fix (`e217bf31a`) stopped re-exporting private names from `sase.agent.running`. The perf bench was never
   updated. Reproduced locally at HEAD.

2. **`visual-test`** — deterministic PNG mismatch (15702/1520532 pixels, ~1.03%, identical on every run) in
   `tests/ace/tui/visual/test_ace_png_snapshots_agents_retry_e2e.py::test_real_fakey_retry_countdown_png_snapshot`
   against golden `tests/ace/tui/visual/snapshots/png/agents_retry_e2e_countdown_120x40.png`. The agent detail pane
   renders a _real_ runtime traceback captured via `traceback.format_exc()` into `workflow_state.json` step `traceback`
   fields. Python 3.13+ formats tracebacks with `...<N lines>...` source elision, while 3.12 uses the older `^^^^`
   caret-anchor style. The golden was regenerated under Python 3.14 (in `26682ba37`, sase-65), but the CI `visual-test`
   job and the 3.12 matrix leg render under Python 3.12 — so the golden can never match on CI. Confirmed by diffing the
   CI `ace-visual-artifacts` expected/actual PNGs: the only differing region is the traceback text block.

3. **`test (3.12)` matrix leg, deterministic part** — three tests in `tests/main/test_parser_command_help.py` assert the
   argparse option-help format introduced in Python 3.13 (`-p, --project PROJECT`). Python 3.12 renders
   `-p PROJECT, --project PROJECT`, so `test_repo_open_help_documents_inference_and_required_reason`,
   `test_repo_list_help_documents_scope_workspace_and_examples`, and
   `test_repo_log_help_documents_filters_lookup_and_examples` fail only on the 3.12 leg (3.13/3.14 legs pass them).

4. **`test (3.12)` matrix leg, flaky part** — the 3.12 leg owns visual coverage in the matrix
   (`SASE_PYTEST_EXCLUDE_VISUAL` is set for the other legs) and runs `just test-cov` with coverage instrumentation.
   Under that overhead, assorted visual tests intermittently fail with "Timed out waiting for ACE visual render
   convergence" (2.0s default in `wait_for_visual_idle`) or "Timed out ... waiting for SVG sentinel" (5.0s default in
   `wait_for_state`/`wait_for_svg_contains`), with a different test set each run. Helpers live in
   `tests/ace/tui/visual/_ace_png_snapshot_waits.py`.

5. **Already fixed, no action** — the `test (3.14)` failure in run `29512859968` (`_FakeApp` missing `call_later` in
   `tests/ace/tui/test_artifact_index_maintenance_scheduler.py`) was fixed at HEAD by `b8b7d65e1`, which routed
   `_resume_artifact_index_maintenance_after_schema_rebuild` through `_spawn_artifact_index_maintenance_task()`.
   Verified locally: the module's tests pass at HEAD.

## Design

### Fix 1: phase7 perf bench import (`tests/perf/bench_agent_scan.py`)

Update the `_run_scenarios` local import block: import `_done_from_snapshot` and `_running_from_snapshot` from
`sase.agent.running_listing` (their new home), keeping the public `list_all_agents` / `list_running_agents` imports from
`sase.agent.running`. Per the symvision rules, test files may import private symbols across modules, so no re-export or
visibility change in `src/` is needed (and none should be added — `e217bf31a` deliberately removed the private
re-exports). Sweep `tests/perf/` for any other imports still targeting moved `running.py` helpers.

### Fix 2: make e2e visual goldens interpreter-independent (robust PNG fix)

Do **not** loosen PNG diff tolerances and do **not** simply re-pin the golden under 3.12 — the golden would then break
whenever a dev regenerates the corpus under a different local Python, which is exactly what happened. Instead remove
interpreter-version-dependent content from the rendered surface:

- In `tests/fakey/harness.py`, `FakeyRetryHarness.normalize_visual_timestamps` already rewrites `workflow_state.json`
  step `error`/`traceback` fields to replace the repo root with `/workspace/sase`. Extend that normalization to replace
  each step's `traceback` value **entirely** with a fixed, canonical multi-line traceback literal (using
  `/workspace/sase/...` paths, styled like a real traceback but hand-written, so its text is identical under every
  Python version). Keep the `error` field as-is apart from the existing path rewrite — its message contains no
  interpreter-formatted content.
- Audit the same file for any other persisted fields that could embed `traceback.format_exc()` output (e.g. attempt
  metadata, retry state) and canonicalize those the same way if they are rendered.
- Regenerate affected goldens with `just update-visual-snapshots` (the renderer-environment fingerprint enforces the
  pinned package/font/Linux environment). Commit only the goldens that actually changed — expected:
  `agents_retry_e2e_countdown_120x40.png`, and possibly the other `agents_retry_e2e_*` goldens if their panes render
  traceback content.
- Leave the renderer fingerprint's "python_version is diagnostic only" policy unchanged: the invariant this plan
  establishes is that no rendered snapshot content may depend on the interpreter version, which is stronger and keeps
  golden regeneration possible from any modern local Python.

### Fix 3: version-agnostic argparse help assertions

In `tests/main/test_parser_command_help.py`, add a small helper that asserts an option with a metavar is documented,
accepting both argparse renderings: `-p, --project PROJECT` (Python >= 3.13) and `-p PROJECT, --project PROJECT` (Python
<= 3.12). Convert every metavar-bearing option assertion in the three failing tests (`--project`, `--reason`,
`--workspace`, `--agent`, `--id`, `--repo`) and sweep the rest of the file for any other `-x, --long METAVAR` literals
that would fail on 3.12. Flag-only assertions (`-a, --all`, `-j, --json`, `-c, --check`, ...) render identically on both
versions and stay as-is.

### Fix 4: harden visual wait helpers against coverage-leg slowness

In `tests/ace/tui/visual/_ace_png_snapshot_waits.py`, raise the default deadlines so slow, coverage-instrumented CI legs
stop flapping; the helpers poll and return as soon as their condition holds, so healthy runs are not slowed — only the
failure latency grows:

- `wait_for_state` / `wait_for_svg_contains`: default `timeout` 5.0 → 15.0.
- `wait_for_visual_idle`: default `timeout` 2.0 → 8.0.

Keep `_VISUAL_STABLE_FRAME_COUNT` and the pending-work detection unchanged — convergence is achievable in these tests
(they pass in the dedicated single-purpose visual job); only the deadline is too tight under coverage.

## Validation

All of these must pass locally before mailing:

1. `just install`, then `just phase7-perf-check` — proves Fix 1 (currently fails at HEAD with the ImportError).
2. `just test-visual` under the default (3.14) venv — proves goldens match after Fix 2 regeneration.
3. Cross-version proof (the core "robust and reliable" requirement): create a scratch Python 3.12 venv (e.g.
   `uv venv --python 3.12` outside the repo venv, `uv pip install -e ".[dev,visual]"`), then run:
   - the full visual suite (`python tools/run_pytest visual`) — must pass with **zero** mismatches against the same
     committed goldens produced under 3.14, proving interpreter independence;
   - `python -m pytest tests/main/test_parser_command_help.py` — proves Fix 3 on the version that previously failed. If
     the 3.12 visual sweep surfaces additional version-dependent goldens beyond the retry e2e ones, fix them with the
     same canonicalization approach (normalize the rendered content, never widen tolerances).
4. `tests/ace/tui/test_artifact_index_maintenance_scheduler.py` — confirm still green (guards the already-fixed 3.14
   failure).
5. `just check` — mandatory repo gate for any file change (lint, symvision, tests).

After mailing, watch the CI run on the fix commit and confirm every job is green, including all three matrix legs,
`visual-test`, and `phase7-perf-floor`.

## Risks

- The 3.12 visual sweep may reveal more interpreter-coupled goldens (one suspicious one-off:
  `test_xprompt_save_snippet_mode_png_snapshot` mismatched in a single 3.12 matrix run). The canonicalization approach
  generalizes; if a mismatch turns out to be timing flake instead, the Fix 4 timeout headroom addresses it.
- Regenerated goldens are environment-sensitive by design; regeneration must happen inside the pinned renderer
  environment (the fingerprint fixture refuses otherwise), and only intentionally-changed goldens may be committed.
- Raising wait timeouts increases worst-case wall clock for genuinely broken tests in the visual lanes; bounded and
  acceptable (single-digit seconds per failing test).
