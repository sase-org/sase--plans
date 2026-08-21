---
tier: tale
title: Isolate Pandoc scratch files from Git workspaces
goal:
  Interrupted Markdown PDF rendering never leaves Pandoc intermediates in an agent's Git
  checkout.
size: small
proposed_by: bbugyi200.athena.0a0
create_time: 2026-08-21 14:26:19
status: wip
---

# Isolate Pandoc scratch files from Git workspaces

## Objective

Prevent interrupted Markdown-to-PDF rendering from leaving Pandoc's
`toPdfViaTempFile*.html` and `toPdfViaTempFile*.pdf` intermediates in a SASE agent's Git
checkout, while preserving successful launch-preview and generic Markdown PDF rendering.

## Confirmed diagnosis

- The failed `sase-rr.2` commit finalizer reported exactly
  `toPdfViaTempFile3479326-0.html` and `toPdfViaTempFile3479326-1.pdf` as untracked
  paths. The HTML identifies Pandoc as its generator and has the title `launch_preview`;
  the PDF is an empty output from the same attempt.
- Both files were created at 12:10:59 while the phase agent's escalated
  `just test-scoped` run was executing the full test suite. The recorded agent tool
  calls contain no direct Pandoc or PDF-render command. Three seconds later the agent
  terminated the long-running suite.
- `tests/test_markdown_pdf.py::test_render_launch_preview_pdf_smoke_when_tools_available`
  executes the real Pandoc/installed-PDF-engine path. Successful completion is clean,
  but interrupting Pandoc reproduces the same zero-byte PDF plus HTML pair in Pandoc's
  inherited current working directory even when `TMPDIR` points elsewhere.
- `render_markdown_pdf()` currently calls `subprocess.run()` without `cwd`, so a test or
  production agent render inherits the repository root. Python removes its own reserved
  output path on handled failures, but it neither owns nor can reliably clean Pandoc's
  internal files after abrupt process termination.

Therefore the test triggered the incident, but the underlying defect is the renderer
process boundary: Pandoc scratch is allowed to land in the caller's Git working
directory. A `.gitignore` rule or finalizer deletion would only hide that defect.

## Implementation

1. In the Markdown PDF renderer, execute every Pandoc engine attempt from a dedicated
   `TemporaryDirectory` rather than inheriting the caller's current directory. Keep this
   directory under the normal system/runner temporary root so an abrupt process death
   can strand only disposable scratch, never repository files. Normal success, timeout,
   and engine-failure paths must close the context and remove the directory.
2. Make every filesystem operand passed to Pandoc independent of that working directory:
   use absolute paths for the preprocessed/source Markdown, the reserved PDF output,
   CSS, and syntax-definition files. Preserve Markdown resource lookup by explicitly
   supplying a resource path that includes the source directory and the invocation's
   original working directory; do not change the public return value for callers that
   supplied a relative destination.
3. Add focused unit coverage in `tests/test_markdown_pdf.py` that has the mocked Pandoc
   process create representative `toPdfViaTempFile*` files in its received `cwd`. Assert
   that the cwd is isolated from the repository and is removed after both a successful
   render and a handled failure/timeout. Also pin absolute command operands and
   resource-path behavior so relative caller paths do not regress.
4. Keep the existing real-tools launch-preview smoke test as end-to-end coverage. Do not
   add ignore rules for `toPdfViaTempFile*`, and do not teach commit finalizers to
   delete unknown untracked files.

## Validation

1. Run `just install` before repository checks, as required for an ephemeral SASE
   workspace.
2. Run the focused Markdown PDF suites, including `tests/test_markdown_pdf.py` and
   `tests/attachments/test_markdown_pdf_properties.py`.
3. Re-run an interruption probe from disposable scratch and verify Pandoc's
   intermediates are confined to its isolated working directory and no
   `toPdfViaTempFile*` path appears in the Git checkout.
4. Run `just check`. If its scoped selector escalates or reports unusual selection, run
   `just check-full` through `/sase_monitor` as required by the project instructions.
5. Finish with `git status --short --untracked-files=all` and confirm the tree contains
   no Pandoc intermediates or other unintended files.

## Acceptance criteria

- Interrupting the real launch-preview conversion can no longer dirty the Git worktree
  with `toPdfViaTempFile*` files.
- Successful, failed, timed-out, highlighted, fallback, and relative-path Markdown PDF
  rendering retain their existing observable behavior.
- Relative Markdown resources remain resolvable after the subprocess cwd is isolated.
- Focused tests and the required repository verification pass.
