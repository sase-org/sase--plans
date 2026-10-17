---
tier: tale
size: medium
title: Finish the sase.sh docs-redesign landing and close epic sase-1le
goal:
  Every sase.sh page shows working glossary tooltips, just check really runs the docs
  build, the docs hygiene lint ignores link destinations, generated CLI pages are
  identical on every supported Python, and epic sase-1le is closed with its plan marked
  done.
proposed_by: bbugyi200.athena.sase-1le.land
bead: sase-1le
status: done
---

- **PARENT:**
  [202610/sase_sh_docs_redesign.md](https://github.com/sase-org/sase--plans/blob/main/202610/sase_sh_docs_redesign.md)
- **BEAD:**
  [sase-1le](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1le/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-1le.land](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.sase-1le.land.md)
- **COMMITS:**
  - [04490ca](https://github.com/sase-org/sase/commit/04490cabb19b0e4c2eb27e3b29864af58df12936)
    — fix(docs): land sase-1le check, glossary, hygiene, and CLI usage defects

# Finish the sase.sh docs-redesign landing and close epic sase-1le

Epic `sase-1le` (plan `plan:202610/sase_sh_docs_redesign.md`) is the sase.sh docs
redesign. All 20 phases are closed, and the decompose sub-epic `sase-1le.20.1` has
landed. The land agent verified the epic and triaged every proposed follow-up. Its
results are in the `LANDER FOLLOW-UP TRIAGE (sase-1le.land, ...)` note on `sase-1le`;
read it with `sase bead read sase-1le -r "..."`. Four epic-caused defects remain. This
tale fixes them and then closes the epic.

No other agent resumes this landing. After the coding work, this tale's final section
**is** the epic closeout. Do it in the same turn and do not wait for this work's own
commit.

Read `lint_and_test.md` and `docs_writing.md` with `/sase_memory_read` before editing.
Never run `just check-full`.

## Out of scope

Separate tasks track these; leave them alone:

- the `dev-shell` recipe's `$$PATH` (`sase-1m5`);
- bare install commands in blog posts (`sase-1m4`);
- config schema descriptions (`sase-1m6`);
- the `sase/memory/macros.md` discovery bullet (`sase-1la`).

Do not edit any memory note.

## 1. `just check` never runs the docs build (epic note #5)

`Justfile` recipe `check`, the docs stage:

```just
    @docs_scope="$$(tools/docs_changed)"; \
    if [ "$$docs_scope" = run ]; then \
```

Just is not Make: it passes `$$` to `sh` unchanged, and `sh` expands it to the shell
PID. As a result, `tools/docs_changed` never runs, `docs_scope` is never `run`, and
every `just check` prints `Skipping docs build; no docs inputs changed.` Also, the links
phase exit criterion ("a deliberately broken anchor fails `just check` locally") is
unmet.

- Change both occurrences to ordinary shell syntax: `docs_scope="$(tools/docs_changed)"`
  and `"$docs_scope"`. Touch no other recipe.
- Add a regression test next to the existing Justfile tests
  (`tests/test_justfile_lint*.py` and `tests/_justfile_lint_helpers.py` show the
  conventions; keep any new file under the 500-line limit). The test must exercise
  **both** branches:
  1. Extract the docs-stage snippet from the `check` recipe: the lines from
     `docs_scope=` through `fi`, with the leading `@` and the line-continuation
     backslashes removed.
  2. Run the snippet with `sh` in a temporary directory that contains:
     - a stub `tools/docs_changed` that prints `run` in one case and `skip` in the
       other;
     - a stub `tools/run_silent` that records its arguments.
  3. Assert that the `run` case invokes
     `run_silent "docs (build + links)" just docs-check` and that the `skip` case prints
     the skip message.

  The test must fail against the old `$$` text.

## 2. Glossary tooltips are broken site-wide (phase sase-1le.19 note #1)

Commit `8531a39a0a` cleared the renderer/Prettier drift by piping the output of
`tools/render_glossary` through `format_with_prettier`, including
`docs/_includes/abbreviations.md`. Prettier rewrites each `*[term]: definition` line
into `_[term]: …` emphasis and joins the lines into one paragraph. `mkdocs.yml`
auto-appends that file to every page through `pymdownx.snippets`. At master
`3e950ebdbd`, the built site therefore has zero `<abbr>` elements, and every page ends
in a visible paragraph listing the glossary terms.

- In `tools/render_glossary` `generated_outputs`, return the raw
  `render_abbreviations_text(webs)` for the abbreviations file. Keep Prettier formatting
  for `docs/reference/glossary.md` only.
- Make sure every definition is a single line. Collapse internal whitespace and newlines
  in the definition text if `render_abbreviations_text` does not already do so.
- Add `docs/_includes/abbreviations.md` to `.prettierignore` with a one-line comment,
  following the style of the neighboring entries. The comment should say that `abbr`
  definitions are line-oriented `*[term]: text` syntax that Prettier rewrites into
  emphasis.
- Regenerate with `just fix` (or `just fmt-docs`). The committed file is then the
  generated-marker comment followed by one `*[term]: …` line per `ABBR_TERMS` entry.
- Tests in `tests/test_render_glossary.py`:
  - The rendered abbreviations output has exactly `len(ABBR_TERMS)` lines matching
    `^\*\[[^\]]+\]: \S`.
  - Using `pytest.importorskip("markdown")`, render a short paragraph that mentions
    `Patch`, with the generated abbreviations appended, through Python-Markdown's `abbr`
    extension. Assert that the output contains `<abbr title=` and does not contain
    `[Patch]:`.
- Verify on the real site: run `just docs-check`, then confirm that
  `site/concepts/index.html` and one `site/reference/...` page contain `<abbr` and that
  neither contains `[project]:` text.

## 3. Hygiene lint flags filenames in link destinations (phase sase-1le.12 note #2)

`tools/check_docs` `check_hygiene` (Guide tiers) and `check_reference_hygiene` scan each
line after removing fenced code, HTML comments, and inline code. Link and image
destinations are still scanned. As a result,
`![diagram](../images/sase-concepts-model.svg)` is reported as the bead ID
`sase-concepts`. The phase worked around this by renaming the image.

- Before scanning, blank out:
  - inline link and image destinations (`](…)`, including an optional title);
  - reference-style link definitions (`[label]: url`);
  - HTML `src`, `href`, and `poster` attribute values.

  Link text stays scanned. Share one helper between the two checks.

- Tests in `tests/test_check_docs_tool.py`:
  - A Guide fixture whose only `sase-…` text is an image or link destination passes.
  - A fixture with a bead ID in visible text or link text still fails with the bead-id
    finding.
  - The same pair for a hand-written `reference/` fixture.

## 4. Generated CLI pages depend on the Python version (task sase-1lm, caused by sase-1le.16)

`tools/render_cli` `_render_node` writes `parser.format_usage()` into each usage block.
Python 3.13+ argparse formats usage differently from 3.12:

- it keeps a subparser `...` on the choices line;
- it renders mutually exclusive groups together (`[-R | -r]`, `(-a DIR | -p | name)`);
- it wraps lines at different points.

The `docs/reference/cli/*.md` pages therefore flip with the regenerating interpreter.
`just docs-gen-check` fails on the "other" Python (this checkout's `.venv` is 3.14; CI
uses 3.12). `git show e69c455bd6 -- docs/reference/cli/` shows every divergent
construct, and all the divergence is inside the usage blocks. Read the task first with
`sase bead read sase-1lm -r "..."`.

- Replace the `format_usage()` call with a usage builder in `render_cli` that does not
  depend on the interpreter version. It must not call argparse's `format_usage`,
  `format_help`, or `_format_usage`; argparse's per-action `_format_args` and metavar
  logic is acceptable only if the cross-version check below proves it stable.
- Pick one style and always use it:
  - `usage: <prog>` followed by the actions in parser order;
  - optional actions as `[-x METAVAR]`, or without brackets when required;
  - every mutually exclusive group rendered together as `[a | b]`, or `(a | b)` when
    required, whether or not its members are adjacent;
  - positionals by metavar or choices;
  - subparsers as `{a,b,c} ...` on one token line.

  Wrap long usage deterministically on whole tokens at a fixed width, never splitting a
  bracketed group, with continuation lines indented to align after `usage: <prog> `.

- Tests in `tests/test_render_cli.py`: build synthetic parsers and assert exact expected
  usage strings. Cover:
  - a non-adjacent mutually exclusive group;
  - a required mutually exclusive group that contains a positional;
  - subparsers with a long choice list;
  - required options with metavars;
  - `nargs` `?`, `*`, and `+`;
  - a usage long enough to wrap.

  CI runs 3.12 and local `.venv` runs 3.14, so any version dependence fails on one side.

- Cross-version check: `render_cli` imports `sase` lazily, so rendering a synthetic
  parser needs only the stdlib. Write a throwaway script that loads `tools/render_cli`
  with `SourceFileLoader`, builds the synthetic parser, and prints the rendered node.
  Run it with Python 3.12, 3.13, and the `.venv` 3.14, using for example
  `uv run --no-project --python 3.12 python <script>` or interpreters on PATH, and
  confirm the outputs are byte-identical. Do not commit the script.
- Regenerate with `just docs-gen` (or `just fix`), then confirm that
  `tools/render_cli --check` and `just docs-gen-check` pass. Expect a one-time diff
  across `docs/reference/cli/`.
- Close the task:

  ```bash
  sase bead close sase-1lm --note "<what changed and the 3.12/3.13/3.14 byte-identical evidence>"
  ```

## 5. Verify

1. Run `just fix`.
2. Run `sase tool run check`. Its docs stage must now run (`docs (build + links)`), not
   print the skip line. The run may still report the known project-label audit failure
   tracked by `sase-1lx`. Treat every UNKNOWN failure as yours.
3. Run `just docs-check` (strict build and URL contract) and redo the `<abbr>` checks
   from section 2.
4. Run `just docs-pdf-check`.
5. Run `just docs-lint`.
6. Prove the docs gate end to end. Temporarily break one anchor in a hand-written
   `docs/reference/` link and confirm that `sase tool run check` fails at
   `docs (build + links)`. Then revert the break and rerun until green.

## 6. Epic closeout (final step; do it in this turn)

1. Run `sase bead epic-symbols sase-1le`. For each listed `--epic-symbol` entry, follow
   the Symvision epic-whitelist policy: wire the symbol up, privatize it, add a non-test
   pragma, or delete it. Re-key an entry only to a still-open later bead that needs it.
   The list was empty at triage time.
2. Close the epic with a note that records the verification. Fill in the bracketed
   results from your own runs:

   ```bash
   sase bead close sase-1le --note "Landing verified. All 20 phases are closed. The decompose phase landed through sub-epic sase-1le.20.1, which leaves only index.md, the generated README.md, and generated stubs at the docs root, and an empty budgets.yml allowlist. Every PROPOSED FOLLOW-UP on .1-.19 and epic notes #1-#7 was triaged in the LANDER FOLLOW-UP TRIAGE note: new tasks sase-1m4, sase-1m5, sase-1m6, sase-1m7 (snoozed to 2026-11-06), and sase-1m8; a +1 on sase-1em; sase-1ld closed as fixed; sase-1la partially fixed. Integration: the 36 non-epic commits since 1a7a80a306 wrote only typed or generated docs, and every src/ and README sase.sh URL is canonical. The lander tale fixed the four epic-caused defects. (1) The just check docs stage now runs: the Justfile no longer misuses \$\$, a regression test covers both branches, and a deliberately broken anchor failed sase tool run check [ToolRun id]. (2) Glossary tooltips render: the abbreviations are no longer Prettier-mangled, site pages contain <abbr>, and no page shows the glossary paragraph. (3) The check_docs hygiene lint ignores link destinations. (4) render_cli usage is interpreter-independent: identical on 3.12, 3.13, and 3.14, and sase-1lm is closed. Results: sase tool run check [verdict]; just docs-check [N] URLs; docs-pdf-check [pages and size]; docs-lint ok. Human follow-ups: set extra.analytics.token or enable Cloudflare automatic setup, and run the reader-testing round."
   ```

   If the close is rejected over leftover epic-symbol entries, resolve them and close
   again. Never use `--force` merely to make the close succeed.

3. Run `just symvision` and confirm the whitelist is clean.
4. Set `status: done` in the frontmatter of the epic's plan file. That is the PLAN path
   shown by `sase bead read sase-1le -r "Need the plan path for closeout"` (the
   `202610/sase_sh_docs_redesign.md` plan, currently `status: wip`).
5. `sase-1le` has no `parent_bead`, so the landing ends here.
