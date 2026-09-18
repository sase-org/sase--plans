---
tier: tale
title: Make screenshot rasterization work in standard SASE installs
goal:
  A normal SASE install can run `sase screenshot` and produce a PNG without separately
  installing the visual-test extra.
size: small
proposed_by: bbugyi200.athena.0mr.f0
create_time: 2026-09-18 08:47:53
status: wip
---

# Plan: Make screenshot rasterization work in standard SASE installs

## Diagnosis

`sase screenshot` is a core CLI command, but its final SVG-to-PNG leg imports
`resvg_py`, which is declared only by the `visual` optional dependency group. The
canonical `uv tool install sase` environment therefore does not contain the renderer.
The current error compounds the packaging defect by suggesting
`uv pip install -e '.[dev,visual]'`: from a checkout, that command installs into the
checkout virtualenv, while the `sase` executable on `PATH` can continue running from the
separate uv-tool interpreter. The reproduced command follows exactly that split and
exits 2 after successfully reaching rasterization.

The full visual extra should remain optional because it carries the snapshot-only
grammar, glyph-audit, and comparison stack. Only the pinned renderer used by the shipped
screenshot command needs promotion to the runtime dependency contract.

## Implementation

1. Promote `resvg_py==0.3.3` to an unconditional project dependency in `pyproject.toml`,
   retaining the matching exact pin in the `visual` group so the documented
   golden-renderer fingerprint and visual environment remain explicit and
   version-aligned. Regenerate `uv.lock` and verify its SASE package metadata records an
   unconditional `resvg-py` requirement in addition to the visual-extra contract.

2. Update the defensive missing-renderer error in `src/sase/ace/tui/visual_render.py`. A
   missing import will now mean the SASE installation is incomplete or stale, not that
   an ordinary screenshot user forgot a test extra. Point managed installs toward
   `sase update`/reinstallation of the interpreter that actually owns the `sase` entry
   point, without adding a fallback renderer or weakening the single canonical rendering
   path.

3. Add regression coverage for both layers of the contract:
   - Assert that the screenshot rasterizer is an unconditional project dependency and
     that its runtime and visual pins cannot drift apart.
   - Adjust the missing-import test to expect the repaired installation diagnostic.
   - Extend the publish wheel smoke and its workflow contract test to render a minimal
     SVG through `sase.ace.tui.visual_render` after installing the built wheel without
     extras. This proves published metadata actually pulls in the renderer instead of
     relying on the contributor environment or a test skip.

## Validation

1. Refresh and check the lockfile, then run the focused renderer, screenshot command,
   terminal screenshot smoke, packaging, and publish-workflow contract tests. Confirm
   the terminal smoke no longer skips because `resvg_py` is absent from the default
   development install.

2. Run `just fix` and the governed `just check` suite. Treat unrelated failures as
   explicit verification caveats; do not broaden this dependency fix into unrelated TUI
   or packaging work.

3. Build the SASE wheel and install it without extras into a clean isolated environment.
   Verify the installed wheel metadata contains `resvg-py`, import the renderer, and
   rasterize a minimal SVG to a valid PNG. This is the release-shaped proof that a
   default install owns every dependency used by the core screenshot command.

4. Put that clean environment first on `PATH`, use a private tmux socket, and run the
   exact command form `sase screenshot -o <temporary-path>.png`. Require exit 0, the
   printed `png=`/`svg=` result fields, a visually loaded PNG (not merely an existing
   file), and no leftover test-owned `sase_ace_agents` session after capture.
