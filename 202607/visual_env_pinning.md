---
tier: tale
goal: 'The ACE PNG visual snapshot suite renders byte-identical PNGs regardless of
  the host''s terminal color environment and timezone, the 12 visual tests that still
  fail in CI go green, and epic sase-65 (Visual snapshot determinism) is closed out:
  bead closed, expired symvision whitelist entries cleaned up, and the epic plan file
  marked done.

  '
create_time: 2026-07-15 21:29:55
status: wip
prompt: 202607/prompts/visual_env_pinning.md
---

# Plan: Pin terminal-color env and timezone for visual snapshots, then land epic sase-65

## Context

Epic sase-65 (plan `sase/repos/plans/202607/visual_snapshot_determinism.md` in the plans sidecar repo) pinned the render
stack (sase-65.1), made captures wait for expected state (sase-65.2), cut over to byte-exact comparison (sase-65.3), and
hardened CI lanes plus docs (sase-65.4). The land-agent verification confirmed all four phases are implemented as
reported. However, the epic's headline success criterion — the CI `visual-test` job green — is still unmet: the last
three completed master CI runs (through commit `92c8f2c03`) each failed the same 12 visual tests, e.g.:

- `test_ace_png_snapshots_agents_zoom.py::test_agents_file_zoom_modal_png_snapshot` (and multi_file variant)
- `test_ace_png_snapshots_agents_retry.py::test_selected_retry_metadata_png_snapshot`
- `test_ace_png_snapshots_agents_retry_e2e.py::test_real_fakey_retry_countdown_png_snapshot` (and running_fallback)
- `test_ace_png_snapshots_preview_panel.py` (xprompt, file, commit_view_modal)
- `test_ace_png_snapshots_agents_linked_repos.py` / `..._external_repos.py` (diff/commit panels)
- `test_ace_png_snapshots_config_center_workspaces.py::test_config_center_workspaces_subtab_png_snapshot`

## Diagnosis (verified with evidence)

CI itself is now fully deterministic: the 12 failing tests' `actual.png` artifacts are byte-identical across independent
CI runs (verified via sha256 over the `ace-visual-artifacts` uploads of runs 29459411192 and 29462095916). The remaining
divergence is CI-vs-golden, caused by two process-environment leaks into rendering that the epic's package/font
fingerprint cannot see:

1. **Terminal color system (9 tests).** The diff/file panels render Rich
   `Syntax(..., line_numbers=True, theme="monokai")` (`src/sase/ace/tui/util/lazy_syntax.py`, used by
   `src/sase/ace/tui/widgets/file_panel/_content.py`, preview/commit modals, etc.). Rich chooses the line-number gutter
   style from `console.color_system`: on a 256-color/truecolor terminal it emits a precomputed 30%-blend RGB (e.g.
   `#656660`), and a 90%-blend bold highlight rule; otherwise it falls back to a `dim` style flag which the SVG exporter
   renders as a different, brighter fill (`#b5b3a9`). The goldens were regenerated on athena under `TERM=tmux-256color`
   (blend path); GitHub Actions runners take the dim path. Empirically confirmed: re-running the failing tests locally
   with `env -u COLORTERM -u FORCE_COLOR TERM=dumb` reproduces CI's `actual.png` **byte-for-byte**. Note: with
   `TERM=dumb`, Rich ignores `COLORTERM` entirely (dumb-terminal early return), so both variables must be pinned.

2. **Timezone (3 tests).** Snapshot fixtures pin timestamps as absolute instants, but rendering formats them in the
   process-local timezone: the Config Center workspaces tab shows "Created/Last used ... 12:00" on athena
   (America/New_York) vs "16:00" in CI (UTC); the retry metadata line shows "Attempt 1 · 11:54:00" vs "07:54:00"; the
   retry-countdown e2e snapshot additionally shifts a wrapped traceback block by a line (knock-on layout change from the
   differing rendered strings). `tests/ace/tui/visual/conftest.py` pins `local_now` for agent-list modules but nothing
   pins `TZ` itself. Also byte-confirmed: with `TZ=UTC` (plus the color scrub) local actuals equal CI's artifacts
   exactly.

Both classes were re-confirmed at current HEAD (`a62647069`): the representative tests pass with athena's normal shell
env and fail with the CI-like env; they pass again with `COLORTERM=truecolor TERM=xterm-256color TZ=America/New_York`
even when `CI=true` is set.

Integration review of commits that landed since the epic started found no other work needed: `728595e54`, `79cce7991`,
and `a62647069` regenerated goldens on athena consistently with the current corpus; `bf17b396a` only renamed symbols in
a visual helper.

## Design

Extend the epic's principle — "the environment that defines rendering is part of the golden corpus definition" — to the
process environment, by **pinning it at fixture level** so no machine's shell or CI configuration can influence
rendering:

- In `tests/ace/tui/visual/conftest.py`, extend the existing autouse env fixture (`_force_color_for_visual_snapshots`,
  which already deletes `NO_COLOR`) to also:
  - `setenv COLORTERM=truecolor`, `setenv TERM=xterm-256color`, `delenv FORCE_COLOR` — every environment takes Rich's
    blend path, which is what the entire committed golden corpus already encodes, so **no goldens change for the color
    class**.
  - `setenv TZ=UTC` **and call `time.tzset()`**; register a teardown that calls `time.tzset()` again after monkeypatch
    restores the env (monkeypatch alone restores the variable but not libc's cached timezone). UTC is the canonical
    choice: it is CI-native and DST-free.
- Regenerate only the goldens the TZ pin invalidates (expected: the three TZ-class snapshots —
  `config_center_workspaces_tab_120x40`, `agents_retry_selected_detail_120x40`, `agents_retry_e2e_countdown_120x40` —
  discover the exact set by running the suite after the fixture change). Use the fingerprint-gated
  `just update-visual-snapshots` recipe, and review each diff to confirm it is exactly a +4h timestamp shift or its
  knock-on layout movement, nothing else.
- Add a small unit test alongside the existing visual contract tests asserting the pinned env is in effect during visual
  tests (e.g. a `visual`-marked test that reads `os.environ["TERM"]`, `["COLORTERM"]`, `["TZ"]` and
  `time.timezone == 0`), so the pin cannot silently regress.
- Documentation: update the renderer-determinism section `docs/development.md` gained in sase-65.4 to state that the
  visual fixtures pin the terminal color environment (`TERM`/`COLORTERM`/`FORCE_COLOR`/`NO_COLOR`) and `TZ=UTC` at
  runtime, so neither the contributor's shell nor CI configuration can skew renders; adjust any wording that implies the
  host terminal affects goldens.

Do NOT extend `tests/ace/tui/visual/renderer_env.json` for this: runtime pinning removes the dependency instead of
gating on it, which is strictly stronger. Keep the fingerprint package/font-scoped.

This is presentation/test-infrastructure work confined to this repo; no sase-core (Rust backend) surface is involved.

## Validation

Sandbox caveat (unchanged from the epic): the full `just test` / `just check` test phase can be SIGTERM-killed
(exit 144) here; rely on `just test-visual`, targeted pytest subsets, and the static gates locally, with CI as the
full-matrix backstop.

1. After the fixture change, `just test-visual`: expect exactly the TZ-class snapshots to fail; regenerate them via
   `just update-visual-snapshots`; then `just test-visual` green.
2. **CI simulation (the decisive check):**
   `env -u COLORTERM -u FORCE_COLOR TERM=dumb CI=true GITHUB_ACTIONS=true TZ=Asia/Tokyo just test-visual` must be fully
   green. Using a third timezone (neither EDT nor UTC) proves the TZ pin, and the color scrub proves the terminal pin.
   This simulation is byte-faithful to real CI: the same scrub reproduced CI's artifacts exactly during diagnosis.
3. Run `just test-visual` at least twice more in the normal shell to confirm the suite stays byte-stable run-to-run.
4. `just check` for the static gates (format, ruff, mypy, symvision). Known pre-existing failures unrelated to this
   change should be verified as pre-existing (git stash + rerun) rather than fixed here.
5. After the change lands on master, the CI `visual-test` job and the 3.12 matrix leg at the landed commit should go
   green (3.13/3.14 legs no longer run visual tests since sase-65.4). The CI queue can lag pushes by hours; do not block
   the landing on it — the CI simulation in step 2 is the local proof — but check `gh run list` if results are available
   before finishing.

## Landing epic sase-65 (final phase of this plan)

After the fix is validated and committed, finish the epic landing:

1. `sase bead close sase-65`.
2. Run `just symvision` — closing the epic expires its epic-symbol whitelist entries. Remove the stale whitelist entries
   and any unused code it reports.
3. Open the plans sidecar via the `/sase_repo` skill (`sase repo open plans -r ...`) and set `status: done` in the
   frontmatter of `202607/visual_snapshot_determinism.md` (the PLAN path shown by `sase bead show sase-65`).
4. Flag to the user (do NOT edit memory files — agents must never modify `memory/*.md` or provider shims without
   explicit in-conversation user permission): the Tier-1 memory text in the sase repo's agent instructions, "Local runs
   use exact pixel equality by default, while CI allows a small ratio-only renderer drift tolerance", is stale — both
   local and CI lanes now compare byte-exactly, and the visual fixtures additionally pin TERM/COLORTERM and TZ=UTC.
   Suggest the updated wording in the final report and ask the user to approve the memory edit.

## Risks

- **Another latent env dependency** (locale, LINES/COLUMNS, TEXTUAL_\* animation vars). Mitigation: the CI-simulation
  run in validation step 2 exercises a scrubbed environment; any residual divergence found there identifies the next
  variable to pin — extend the same fixture.
- **A newly-added snapshot is TZ-sensitive beyond the known three** (the corpus was fully regenerated in `a62647069`).
  Mitigation: the post-fixture `just test-visual` run discovers the exact regen set; review every regen diff for
  timestamp-only changes.
- **Pinning direction disagreement**: pinning `TZ=America/New_York` would avoid any regen, but bakes a DST-carrying zone
  into the corpus definition; UTC + a three-golden regen is the deliberately chosen tradeoff.
