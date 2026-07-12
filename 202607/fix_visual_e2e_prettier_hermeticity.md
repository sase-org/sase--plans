---
create_time: 2026-07-11 13:57:16
status: done
prompt: 202607/prompts/fix_visual_e2e_prettier_hermeticity.md
tier: tale
---
# Fix CI PNG snapshot failures: prettier-dependent prompt rendering + empty visual-artifact uploads

## Problem

CI on `sase-org/sase` master has been red since 2026-07-10. The `visual-test` and `test (3.x)` jobs fail on exactly two
ACE PNG snapshot tests in `tests/ace/tui/visual/test_ace_png_snapshots_agents_retry_e2e.py`:

- `test_real_fakey_running_fallback_png_snapshot` — 20666/1520532 changed pixels (1.359130%)
- `test_real_fakey_completed_retry_chain_png_snapshot` — 39446/1520532 changed pixels (2.594224%)

Both exceed CI's 1% `SASE_VISUAL_PNG_MAX_DIFF_RATIO` tolerance. The failures are perfectly deterministic: byte-identical
changed-pixel counts across runs, commits, and Python versions. These tests have never passed in CI (they landed while
master was already red; commit `1c0154b` fixed their original hard error — a shell-out to a personal
`branch_or_workspace_name` helper absent in CI — leaving these pixel diffs).

A second, related defect: when these tests fail in CI, the failure-report step and the `ace-visual-artifacts` upload
come up **empty**, so CI failures cannot be debugged from artifacts alone.

## Root cause (confirmed by container repro)

The failure was reproduced byte-identically (same 20666/39446 changed-pixel counts) in an ubuntu-24.04 container
mirroring the GitHub runner layout. Diffing the pipeline outputs of that run against a local run shows the chat history
files are **identical**, but the prompt artifact (`.../artifacts/ace-run/<ts>/workflow-<name>-main_prompt.md`) — which
the agent detail view's AGENT PROMPT panel renders — differs:

- **Local (golden source)**: retry-continuation nudge hard-wrapped at 80 columns, blank runs collapsed.
- **CI**: the same nudge as one long unwrapped line.

The divergence comes from `format_with_prettier()` in `src/sase/file_references.py`, called from the late
prompt-preprocessing phase (`src/sase/llm_provider/preprocessing.py`, via `AGENT_PROMPT_WRAP_WIDTH`). It pipes prompt
markdown through the `prettier` CLI (`--prose-wrap=always`) **when `prettier` is on `PATH`**, and silently returns the
text unchanged otherwise. The machine that generated the goldens has prettier (via nvm/node); the GitHub runner does
not. The different wrap points shift every subsequent rendered line of the AGENT PROMPT panel, producing the pixel
diffs.

The fakey e2e harness (`tests/fakey/harness.py`, `FakeyRetryHarness`) isolates `SASE_HOME`, `SASE_TMPDIR`, etc., but
inherits the host `PATH`, and the retry pipeline runs in-process — so host-tool presence leaks into rendered content.
The PNG visual fixtures already pin fonts/fontconfig/colors for hermetic rendering; this is the remaining environment
leak.

### Secondary bug: empty CI failure artifacts

The `ace_png_visual` fixture (`tests/ace/tui/visual/conftest.py`) builds `artifact_root` as `Path(option).expanduser()`
from `--sase-visual-artifact-dir` (default `.pytest_cache/sase-visual`) without making it absolute. The e2e harness
`monkeypatch.chdir()`s into a pytest tmp workspace, so failure artifacts (actual/expected/diff PNGs, failure.json) are
written under the pytest tmp tree instead of the repo's `.pytest_cache/sase-visual` — which is what the CI report step
and artifact upload read.

## Design

Canonize the **no-prettier** rendering for hermetic tests (rather than installing prettier in CI): installing prettier
in CI would still leave goldens hostage to prettier version drift and would break golden regeneration for any
contributor whose prettier version differs. The deterministic fallback path (no reformatting) is reproducible on every
machine with no extra dependencies.

### 1. Add an off-switch for the prettier pass

In `src/sase/file_references.py`, make `format_with_prettier()` return the text unchanged when `SASE_DISABLE_PRETTIER`
is set to a truthy value (follow the existing `SASE_DISABLE_PLUGINS` env-var convention for naming and truthiness). This
is a production-code env knob, checked before `shutil.which("prettier")`.

### 2. Pin the knob in the hermetic test harnesses

- `tests/fakey/harness.py` (`FakeyRetryHarness.__init__`): `monkeypatch.setenv("SASE_DISABLE_PRETTIER", "1")` alongside
  the existing env pinning. This covers both the PNG e2e visual tests and `tests/fakey/test_retry_pipeline_e2e.py`, and
  works for in-process and subprocess execution alike.
- Also pin it in the shared ACE visual-snapshot fixture env setup (where colors/fontconfig are already pinned) so any
  future visual test that runs real prompt preprocessing stays hermetic.

### 3. Regenerate the two affected goldens

Regenerate `tests/ace/tui/visual/snapshots/png/agents_retry_e2e_running_fallback_120x40.png` and
`agents_retry_e2e_completed_chain_120x40.png` via
`just test-visual tests/ace/tui/visual/test_ace_png_snapshots_agents_retry_e2e.py --sase-update-visual-snapshots`. With
the knob pinned, regeneration on any machine (with or without prettier installed) produces the same unwrapped-nudge
rendering that CI produces. Only these two goldens should change; verify with `git status` that no other snapshots were
rewritten.

### 4. Anchor the visual failure-artifact dir at the repo root

In `tests/ace/tui/visual/conftest.py` (`ace_png_visual` fixture): when the `--sase-visual-artifact-dir` value is
relative, resolve it against `request.config.rootpath` instead of leaving it CWD-relative. Do NOT use
`Path.resolve()`/CWD at fixture-instantiation time — fixture ordering means the harness may have already chdir'd. After
this fix, CI failure artifacts land in the repo's `.pytest_cache/sase-visual` and the `ace-visual-artifacts` upload +
failure report become useful.

### 5. Regression tests

- Unit test for `format_with_prettier`: with `SASE_DISABLE_PRETTIER=1` set (monkeypatch), the input text is returned
  unchanged even when a fake `prettier` is on `PATH` (or `shutil.which` is monkeypatched).
- Unit test in `tests/ace/tui/visual/test_png_diff.py` (or the conftest's test coverage): a relative artifact dir
  resolves against the pytest rootpath even after `os.chdir` into a tmp dir — i.e. failure artifacts land under
  `<rootpath>/.pytest_cache/sase-visual`.

## Non-goals

- Do not install prettier in the CI workflow (version-drift fragility; see Design intro).
- No changes to the Rust core (`sase-core`) — this is test-harness hermeticity plus one env-gated early return in Python
  glue; no cross-frontend domain behavior changes.
- Do not change the retry-continuation nudge text or `_build_resume_prompt` format.

## Verification

1. `just install` (fresh workspace requirement), then
   `just test-visual tests/ace/tui/visual/test_ace_png_snapshots_agents_retry_e2e.py` on a machine that **has** prettier
   on PATH — must pass byte-exact against the regenerated goldens (proves the knob defeats the host dependency).
2. `pytest tests/fakey/test_retry_pipeline_e2e.py` — retry pipeline e2e still green with the knob pinned.
3. Full `just check` before finishing (required for any file changes in this repo).
4. Optional but recommended: rerun the ubuntu-24.04 container repro (no prettier installed) against the branch and
   confirm the two tests pass byte-exact — this is the exact CI-equivalent environment that previously reproduced the
   failure.

## Expected outcome

Master CI turns green: the two PNG e2e tests pass identically on GitHub runners and on any dev machine regardless of
installed host tools, and any future visual-test failure in CI uploads real actual/expected/diff artifacts for
debugging.
