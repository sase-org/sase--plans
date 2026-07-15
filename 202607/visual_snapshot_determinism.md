---
tier: epic
title: Visual snapshot determinism
goal: 'Master CI is green again and the ACE PNG visual snapshot suite is deterministic
  across machines: when the suite passes on any properly-installed machine, it passes
  in CI, because every environment renders byte-identical PNGs from a pinned render
  stack and every capture waits for the expected UI state instead of racing it.

  '
phases:
- id: pin-render-stack
  title: Pin the render stack and gate on a renderer-environment fingerprint
  depends_on: []
- id: deterministic-capture
  title: Replace timing-based captures with expected-state waits
  depends_on:
  - pin-render-stack
- id: exact-goldens
  title: Cut over to byte-exact comparison and regenerate all goldens
  depends_on:
  - deterministic-capture
- id: guardrails-and-docs
  title: CI lane hardening, regen tooling polish, and contributor docs
  depends_on:
  - exact-goldens
bead_id: sase-65
---

# Plan: Visual snapshot determinism

## Problem

Master CI for `sase-org/sase` has been red for 100+ consecutive runs. At HEAD the only failing jobs are `visual-test`
and the `test (3.14)` matrix leg (3.12/3.13 legs are cancelled by fail-fast), and every failure is an "ACE PNG snapshot
mismatch" from the visual suite in `tests/ace/tui/visual/`. Repeated band-aid attempts already landed
(`03df1f88d test: tighten visual snapshot drift tolerance` regenerated 123 goldens and added a material-diff gate)
without fixing the redness. The user's requirement: preserve the usefulness of these tests while making them
deterministic — if they pass on one machine, they should almost certainly pass in CI.

## Diagnosis (verified, with evidence)

The rendering pipeline is: Textual app under `Pilot` → `export_svg()` (SVG text produced by Textual/Rich in pure Python)
→ `resvg_py` rasterization with bundled Fira Code fonts (`tests/ace/tui/visual/png_diff.py::render_svg_to_png`,
`skip_system_fonts=True`) → pixel comparison against committed PNG goldens in `tests/ace/tui/visual/snapshots/png/`.

Verified facts from comparing CI failure artifacts (the `ace-visual-artifacts` upload) against local renders:

1. **Rasterization is already byte-deterministic.** Re-rendering a CI-produced `actual.svg` locally with the pinned
   `resvg_py==0.3.3` and bundled fonts reproduces CI's `actual.png` byte-for-byte. All divergence originates at the
   SVG-generation layer, not the rasterizer.

2. **SVG generation depends on unpinned package versions.** CI installs with
   `uv pip install --no-sources -e ".[dev,visual]"` (Justfile `install-visual` / `_setup-visual`), which ignores
   `uv.lock` and resolves the latest allowed versions. At the time of diagnosis CI resolved `textual==8.2.8`,
   `rich==15.0.0`, `pygments==2.20.0`, `tree-sitter==0.26.0`, `markdown-it-py==4.2.0`, `mdit-py-plugins==0.6.1`,
   `pillow==12.3.0`, while `uv.lock` (and typical dev venvs) hold `textual==8.0.1`, `rich==14.3.3`, `pygments==2.19.2`,
   `tree-sitter==0.25.2`, `markdown-it-py==4.0.0`, `mdit-py-plugins==0.5.0`, `pillow==12.2.0`. The machine that
   regenerated the current goldens had a third version set. These version differences change composited colors in the
   exported SVG (example: the prompt footer hint-row background renders `#191818` in the goldens but `#100f0f` in both
   CI and a lock-consistent venv — an alpha-aware color distance of 9, one above the material threshold of 8). Small
   blend shifts over large areas also blow through the 1% ratio cap (`SASE_VISUAL_PNG_MAX_DIFF_RATIO=0.01` exported by
   the Justfile).

3. **With identical package versions, renders are byte-identical across machines and Python versions.** A local Python
   3.14 venv and CI's Python 3.12 produced byte-identical SVG and PNG for the same tests once package versions matched.
   Eight of the currently failing tests produce byte-identical "actual" images locally and in CI — for those the
   committed golden itself is what disagrees (stale golden).

4. **A second, independent failure class: capture-timing races.** Some captures snapshot a stable-but-wrong frame.
   Observed locally: `stashed_prompts_narrow_modal` captured a blank terminal frame (modal never mounted);
   `prompt_stack_g_prefix_hints` captured the normal footer hints instead of the post-`g` keypress hint bar. Observed in
   CI: `agents_file_zoom_modal` differs from golden only in the diff-gutter line-number color (focus/animation-timing
   dependent). The existing convergence helper `wait_for_visual_idle`
   (`tests/ace/tui/visual/_ace_png_snapshot_helpers.py`) requires 3 identical consecutive frames, but a stable wrong
   state satisfies it. ~30 tests fail on one machine but not another due to this class.

5. **Known bounded platform drift.** `png_diff.py` documents that macOS vs Linux `resvg` rasterization can differ by up
   to 8 visible channel levels. The material threshold exists for that reason. This only matters when goldens are
   produced on a different OS than CI.

The current tolerance regime (1% changed-pixel ratio, material threshold 8, 0 material pixels) is a band-aid over class
2: it sometimes absorbs version skew and sometimes does not, which is exactly the observed "constant flakiness".

## Design

Three principles, in priority order:

1. **One pinned render stack everywhere.** The packages whose versions define SVG output are part of the golden corpus
   definition, exactly like `resvg_py` already is (see the existing comment on the `visual` extra in `pyproject.toml`).
   Pin them exactly; assert the pins at test time with a fast, actionable failure.
2. **Captures wait for expected state, not elapsed stability.** Every snapshot must be preceded by a semantic-state wait
   (widget mounted, sentinel text visible, focus settled), with the frame-convergence probe kept as a second line of
   defense.
3. **Byte-exact comparison as the default gate.** Once 1 and 2 hold, tolerances no longer serve a purpose on the
   canonical platform; exact equality gives a binary, trustworthy signal. The tolerance env overrides remain available
   for renderer investigations and non-canonical platforms (macOS iteration), but the default — locally and in CI — is
   exact.

Canonical render platform: Linux x86_64 (CI runners and the home-server dev environment). Golden regeneration is refused
outside it.

## Phase: pin-render-stack

Pin the SVG-relevant render stack and add a committed renderer-environment fingerprint that the visual suite asserts
before running.

- In `pyproject.toml`, extend the `visual` optional-dependency group (which already exact-pins `resvg_py==0.3.3` with a
  rationale comment) with exact pins matching the current `uv.lock` versions: `textual`, `rich`, `pygments`,
  `markdown-it-py`, `mdit-py-plugins`, `linkify-it-py`, `tree-sitter`, every `tree-sitter-*` grammar wheel that
  `textual[syntax]` pulls in, and `pillow` (comparison-side only, pinned for reproducible diffing). Using the lock's
  versions (textual 8.0.1 et al.) means dev venvs already comply and only CI's resolution changes. Run `uv lock` so the
  lockfile and pyproject agree. All test lanes (`just test`, `test-visual`, `test-cov`) already install `.[dev,visual]`
  via `_setup-visual`, so every lane converges on the pins.
- Add a committed manifest, e.g. `tests/ace/tui/visual/renderer_env.json`, recording: the pinned package versions, a
  SHA-256 of each bundled font in `tests/ace/tui/visual/fonts/`, and (diagnostic-only, not gated for comparison runs)
  Python version and platform.
- Add a session-scoped autouse fixture in `tests/ace/tui/visual/conftest.py` that compares `importlib.metadata` versions
  and font hashes against the manifest and fails the suite immediately on mismatch with a remediation message ("run
  `just install-visual`; if this is an intentional renderer upgrade, follow the regen workflow"). A version mismatch
  must produce this one clear error, never a wall of pixel diffs.
- Gate golden regeneration: `--sase-update-visual-snapshots` refuses to write goldens when the fingerprint mismatches or
  the platform is not Linux. An intentional renderer upgrade updates pins, lockfile, manifest, and goldens together
  (workflow documented in guardrails-and-docs).
- Unit-test the fingerprint logic (mismatch message, regen refusal) alongside the existing
  `tests/ace/tui/visual/test_png_diff.py` patterns.

Success criteria: with a lock-consistent venv the visual suite runs (still red against current stale goldens —
expected); with any deliberately skewed package version the suite fails fast with the fingerprint error; CI's
`install-visual` resolves exactly the pinned set.

## Phase: deterministic-capture

Eliminate the capture-timing race class so identical environments always capture identical frames.

- Add semantic wait helpers next to `wait_for_visual_idle` in `_ace_png_snapshot_helpers.py`: at minimum a
  `wait_for_svg_contains(page, text)` (poll the exported SVG for sentinel text with a deadline, mirroring the existing
  `assert_page_svg_contains` helper) and a predicate-based `wait_for_state(page, fn)` convenience. Failure messages must
  say what sentinel was awaited and dump the last frame like `wait_for_visual_idle` does.
- Audit and fix the race-prone tests. Known offenders from the diagnosis (fix the class, not just this list):
  `test_ace_png_snapshots_prompt_stash.py` (blank modal captures — all stashed-prompts modals),
  `test_ace_png_snapshots_prompt_stack.py:: test_prompt_stack_g_prefix_hints_png_snapshot` (footer hint bar not yet
  updated after the `g` keypress; also the vim-cursor and search-highlight variants),
  `test_ace_png_snapshots_agents_zoom.py` (diff-gutter color depends on focus timing),
  `test_ace_png_snapshots_agents_retry*.py`, `test_ace_png_snapshots_preview_panel.py`,
  `test_ace_png_snapshots_config_center_edit.py` / `..._config_center_workspaces.py`,
  `test_ace_png_snapshots_frontmatter_panel.py`, `test_ace_png_snapshots_vcs_*_completion.py`,
  `test_ace_png_snapshots_inputs.py`, `test_ace_png_snapshots_agents_interactions.py`. For each: after the triggering
  interaction, wait for the expected state (modal screen mounted, sentinel text in the frame, expected focus) before
  `assert_page_png`.
- Where a wrong-state capture can still satisfy frame convergence (e.g. blank modal), prefer sentinel text that only
  exists in the expected state.
- Validation: run `just test-visual` several times back-to-back (5+) in one venv; every run must produce the identical
  pass/fail set, and the set of failures must be explainable purely as stale goldens (identical actual bytes
  run-to-run). A simple loop with a hash over the failure artifacts directory is sufficient evidence. Note: this repo's
  agent sandbox can SIGTERM the full `just test` run; the dedicated `just test-visual` lane runs in under a minute and
  is the validation vehicle here.

Success criteria: repeated suite runs are byte-stable on one machine; no test captures a frame that depends on scheduler
timing.

## Phase: exact-goldens

Flip the default comparison to byte-exact, regenerate every golden once in the pinned environment, and turn CI green.

- Change the Justfile default exports (`SASE_VISUAL_PNG_MAX_DIFF_RATIO`, `SASE_VISUAL_PNG_MATERIAL_DIFF_THRESHOLD`,
  `SASE_VISUAL_PNG_MAX_MATERIAL_DIFF_PIXELS`, Justfile lines ~28–30) so the default is exact equality (`png_diff.py`
  already defaults to exact when no env/kwargs are set — the simplest change is to stop exporting non-zero defaults).
  Keep the env vars themselves working as documented escape hatches for renderer investigations and macOS-local
  iteration; update `test_png_diff.py` expectations accordingly.
- Add a `just update-visual-snapshots` recipe that wraps the regen honestly: ensures `_setup-visual`, then runs the
  visual lane with `--sase-update-visual-snapshots` (which after pin-render-stack refuses to run on a mismatched
  fingerprint or non-Linux platform).
- Regenerate **all** goldens (not only the currently failing ~26): going byte-exact will surface every latent sub-1%
  skew currently hidden by the ratio allowance, so a full clean baseline from the pinned env is required. Review the
  regen diff for unexpected content changes (this is also the moment any real rendering regression hidden by the
  band-aid would surface — e.g. confirm the footer hint-row color change is legitimate current rendering).
- Validate: `just test-visual` green repeatedly; then land and confirm the CI `visual-test` job and the
  `test (3.12/3.13/3.14)` matrix legs go green (the matrix legs run the visual tests too via the `fast`/`cov` marker
  expressions in `tools/run_pytest`). Diagnosis evidence says Python 3.12 vs 3.14 renders identically with pinned
  packages, so all legs should agree; if a leg disagrees, that is new signal to investigate, not to tolerate.

Success criteria: master CI fully green; local `just test-visual` green and byte-stable; comparison failures from here
on mean "the UI actually changed".

## Phase: guardrails-and-docs

Keep it fixed.

- CI hygiene: in `.github/workflows/ci.yml`, decide and implement whether the visual suite keeps running in all three
  matrix legs. Recommendation: keep it in the 3.12 leg (preserves the coverage contribution to the `--cov-fail-under=50`
  gate in `tools/run_pytest cov`) and the dedicated `visual-test` job; exclude it from the 3.13/3.14 legs (marker
  expression change in `tools/run_pytest`, e.g. `"not slow and not visual"` for non-canonical legs) so a hypothetical
  future Python-version-sensitive render cannot redden two extra jobs. Verify the coverage gate still passes for the
  legs that drop visual coverage (they don't gate coverage today — only 3.12 uploads).
- Renderer upgrade workflow documentation (contributor-facing, e.g. `tests/ace/tui/visual/README.md` or CONTRIBUTING):
  how to bump the render stack intentionally — update pins in the `visual` extra, `uv lock`, refresh
  `renderer_env.json`, `just update-visual-snapshots`, review the golden diff, one commit. Also document the macOS
  story: bounded-drift env overrides for local iteration, regen only on Linux (or by accepting the
  `ace-visual-artifacts` actuals from a CI run of the branch).
- Optional (include if cheap): a small tool that downloads a CI run's `ace-visual-artifacts` and copies each failure's
  `actual.png` over its golden, as a canonical-platform regen path for non-Linux contributors. The failure-report
  tooling (`tools/render_visual_snapshot_failure_report`) already maps artifacts to repo paths via `failure.json`'s
  `expected_repo_path`.
- Note for the maintainer (do NOT edit without explicit user approval in-conversation): the Tier-1 memory text
  describing PNG snapshot behavior ("Local runs use exact pixel equality by default, while CI allows a small ratio-only
  renderer drift tolerance") becomes stale after exact-goldens. Memory files must not be modified by agents; flag the
  needed wording change to the user instead.

Success criteria: a contributor can upgrade textual deliberately in one documented motion; CI legs cannot disagree with
each other about visual snapshots; the regen path is impossible to run from a skewed environment by accident.

## Testing strategy (cross-phase)

- Unit level: fingerprint assertion behavior, regen refusal, tolerance-resolution changes
  (`tests/ace/tui/visual/test_png_diff.py` already covers the comparison contract — extend it).
- Suite level: repeated `just test-visual` runs must be byte-stable per phase (deterministic-capture onward).
- System level: master CI (all jobs) green after exact-goldens; verify via the CI run on the landed commit, and confirm
  the previously failing jobs (`visual-test`, `test (3.14)`) specifically.
- Sandbox caveat for implementing agents: the full `just test` / `just check` test phase can be SIGTERM-killed
  (exit 144) in this environment; rely on `just test-visual`, targeted pytest subsets, and the static gates locally, and
  on CI for the full matrix.

## Risks

- **Missed render-relevant package.** The pin set is an allowlist; a package outside it could still influence SVG
  output. Mitigation: the fingerprint manifest asserts the same set, and any future local-vs-CI divergence with matching
  fingerprints identifies the missing package — extend the set then. (Complete closure via `uv sync --locked` installs
  everywhere was considered and rejected for now: it reshapes the whole install flow, including the local `sase_core_rs`
  wheel override in the Justfile `rust-install` path, for marginal benefit over the allowlist + fingerprint.)
- **Race-fix completeness.** The audit list comes from observed failures; other tests may harbor latent races.
  Mitigation: repeated-run byte-stability validation in deterministic-capture and exact-goldens, plus the semantic-wait
  helpers making the right pattern the easy pattern.
- **Byte-exact brittleness on intentional UI changes.** Every real UI change now requires a golden update — that is the
  intended contract; `just update-visual-snapshots` plus the failure artifacts/report keep the loop cheap.
- **macOS contributors.** Byte-exact defaults will fail for them at the rasterization layer (documented ≤8-level drift).
  The documented overrides and the CI-artifact regen path are the mitigation; the canonical gate stays Linux.
