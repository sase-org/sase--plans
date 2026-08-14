---
tier: tale
title: Skip the pypdf-dependent docs-PDF unit test when pypdf is not installed
goal:
  "Post-landing GitHub Actions for the sase-org/sase master branch is green again. The
  `test` matrix (3.12/3.13/3.14) and `coverage-contexts` jobs no longer fail with
  `ModuleNotFoundError: No module named 'pypdf'` in tests/test_docs_pdf_tools.py, while
  the real PDF-export pipeline (postprocess_docs_pdf, pypdf) keeps being exercised for
  real by the docs-build/Deploy Docs jobs that already install it."
size: xsmall
proposed_by: bbugyi200.athena.sase-m4.6--2
bead: sase-m4.6
create_time: 2026-08-14 16:31:54
status: done
---

- **PROMPT:**
  [prompts/202608/docs_pdf_test_pypdf_import_skip.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/docs_pdf_test_pypdf_import_skip.md)
- **BEAD:**
  [sase-m4.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-m4/sase-m4.6.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-m4.6--2--code](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-m4.6--2--code/README.md)
  - [bbugyi200.athena.sase-m4.6--2--plan](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-m4.6--2--plan/README.md)
- **COMMITS:**
  - [357c45c](https://github.com/sase-org/sase/commit/357c45c7235f4d8f23539787dc16f4df41955470)
    — test(docs): skip pypdf-dependent docs-PDF test when pypdf is absent

# Plan: Skip the pypdf-dependent docs-PDF unit test when pypdf is not installed

## Context

Commit e4baf07717f5a9cb836316b8db5416d1af3f8096 (bead sase-m4.2, part of epic sase-m4)
added `tests/test_docs_pdf_tools.py` and a new `pypdf>=5,<7` entry, but only inside the
`docs-pdf` optional-dependencies group in `pyproject.toml` — not `dev`. That `docs-pdf`
extras group is not actually installed via `.[docs-pdf]` anywhere; `just docs-pdf-check`
instead does its own direct `uv pip install ... pypdf>=5,<7` (see the comment above that
recipe: "Keep versions in sync with the `docs-pdf` optional dependency group in
pyproject.toml"). `just install` / `just install-visual` (used by every `test` and
`coverage-contexts` job in `.github/workflows/ci.yml`, via
`.github/actions/setup-sase/action.yml`) install only `.[dev]` / `.[dev,visual]`, so
`pypdf` is never present in those jobs.

`tools/postprocess_docs_pdf` does `from pypdf import PdfReader, PdfWriter` at module
level (line 15). The new test
`test_pdf_image_optimization_reencodes_each_shared_rgb_png_once` calls a
`_load_postprocess()` helper that loads that whole script as a module via
`importlib`/`SourceFileLoader`, which triggers the `pypdf` import and fails with
`ModuleNotFoundError` when `pypdf` is absent. The test itself only exercises
`tool._optimize_images()` against `SimpleNamespace` mocks — it does not need real
`pypdf` PDF-reading/writing behavior, only the module to be importable.

Confirmed via `gh run view --repo sase-org/sase --job <id> --log-failed` for CI run
31832121634 (exact commit e4baf07717f5a9cb836316b8db5416d1af3f8096): `test (3.12)`,
`test (3.13)`, `test (3.14)`, and `coverage-contexts` all fail on exactly this one test
with `ModuleNotFoundError: No module named 'pypdf'`. This workspace's local `.venv`
happens to already have `pypdf` 6.15.0 installed (left over from an earlier
`just docs-pdf-check` run in this same ephemeral workspace), which is why
`just check-full` did not catch this locally — it is an environment-masking gap, not a
flake.

The other two tests in `tests/test_docs_pdf_tools.py`
(`test_pdf_config_disables_remote_material_fonts`,
`test_pdf_check_recipe_keeps_strict_mkdocs_export`) only read `mkdocs-pdf.yml` /
`Justfile` text and do not touch `pypdf`; they must keep running unconditionally.

The real, functional PDF-export pipeline (mkdocs build + `postprocess_docs_pdf` +
`validate_docs_pdf` against a real `pypdf`) is already exercised end to end elsewhere in
CI: the `docs-build` job (PRs) and the `Deploy Docs` workflow (master pushes,
`.github/workflows/docs-deploy.yml`) both run `just docs-pdf-check`, which installs
`pypdf` itself. `Deploy Docs` already succeeded for
e4baf07717f5a9cb836316b8db5416d1af3f8096. So skipping the narrow unit test when `pypdf`
is not importable does not remove real coverage of the PDF pipeline — it only avoids
failing the unrelated `dev`-only `test`/`coverage-contexts` jobs that were never meant
to install docs tooling.

This codebase already uses `pytest.importorskip` elsewhere for optional-dependency-gated
tests (e.g. under `tests/ace/tui/visual/`), so the fix below follows an established
local convention rather than inventing a new one.

## Implementation

In `tests/test_docs_pdf_tools.py`, scope an import-skip guard to only the one test that
needs `pypdf` (`test_pdf_image_optimization_reencodes_each_shared_rgb_png_once`), not
the whole module — the other two tests must keep running in every environment. The
cleanest place is inside `_load_postprocess()` (or at the top of
`test_pdf_image_optimization_reencodes_each_shared_rgb_png_once`, before calling
`_load_postprocess()`): add `pytest.importorskip("pypdf")` and add the `pytest` import
the file is currently missing.

Do not touch `mkdocs-exporter`, `pillow`, or any other dependency; do not change the
`docs-pdf` optional-dependencies group in `pyproject.toml`; do not change
`just docs-pdf-check`, `just install`, `just install-visual`, or any CI workflow file.
This is a narrow test-collection fix only.

## Test coverage and validation

- Run `just install` first (ephemeral workspace).
- Run the focused test: `.venv/bin/pytest tests/test_docs_pdf_tools.py -v` — all three
  tests must pass locally (pypdf is already installed in this workspace's venv).
- Simulate the CI-without-docs-pdf-extra condition and confirm the fix actually works:
  temporarily uninstall pypdf in a scratch venv (or run with `PYTHONPATH`/import hook
  tricks) — at minimum, verify by code inspection plus a quick
  `python -c "import sys; sys.modules['pypdf'] = None"`-style forced-ImportError check,
  or by running
  `uv run --no-project --python <workspace venv python> -m pytest tests/test_docs_pdf_tools.py -p no:cacheprovider`
  with pypdf's dist-info temporarily renamed, that
  `test_pdf_image_optimization_reencodes_each_shared_rgb_png_once` reports `SKIPPED`
  (not failed, not errored) and the other two tests in the file still `PASSED`.
- Run the mandatory repository-wide `just check`.
- Do not re-run the full `just check-full` / exact-commit GitHub Actions verification
  loop yourself after this fix — that remains phase bead sase-m4.6's job once this
  tale's change lands; leave that continuation to the bead's next action.
