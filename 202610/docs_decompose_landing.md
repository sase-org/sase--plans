---
tier: tale
size: medium
title:
  Finish landing docs epic sase-1le.20.1 with MkDocs-accurate anchors, then close it out
goal:
  Every pre-epic sase.sh deep link into the decomposed legacy docs pages resolves again,
  the docs split and move tools record the anchor ids MkDocs actually renders, Deploy
  Docs builds and validates its handbook PDF, and epic sase-1le.20.1 plus its parent
  phase sase-1le.20 are closed with verified notes.
proposed_by: bbugyi200.athena.sase-1le.20.1.land
bead: sase-1le.20.1
status: done
---

- **PARENT:**
  [202610/docs_decompose.md](https://github.com/sase-org/sase--plans/blob/main/202610/docs_decompose.md)
- **BEAD:**
  [sase-1le.20.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1le/sase-1le.20.1.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-1le.20.1.land](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.sase-1le.20.1.land.md)
- **COMMITS:**
  - [355f483](https://github.com/sase-org/sase--plans/commit/355f48385d2236ca1cf2eef4fc05af96588b5f60)
    — docs(plans): mark docs decomposition landing done

# Finish landing docs epic sase-1le.20.1

The land agent for epic `sase-1le.20.1` (`plan:202610/docs_decompose.md`, "Decompose the
legacy root docs pages into Reference and Contribute") verified all 21 phases and
triaged every `PROPOSED FOLLOW-UP:`. The triage outcomes are on the epic bead (the
`LANDER FOLLOW-UP TRIAGE` note). The epic is not done yet. Its own split tooling
recorded legacy anchors in a spelling MkDocs never rendered, so some old deep links that
the epic promised to keep ("Old URLs keep working through redirects and anchor stubs")
now land at the top of a stub instead of on their section.

This tale finishes that work and then performs the epic's closeout. No land agent runs
after this tale, so the coder must finish the landing itself (Step 4).

Read first:

- `sase bead read sase-1le.20.1 -r "Need epic scope, triage note, and plan path"`.
- The epic plan's "Rules for every phase" and "Phase split-tool" sections.
- `docs/contribute/writing-docs.md` ("Splitting and moving pages").
- With `/sase_memory_read`: `docs_writing.md` and `lint_and_test.md`.

The epic's DECISIONS stay final: `reference_hygiene = yes`. Memory edits are authorized
only for `tui_perf.md`, `dispatch.md`, `macros.md`, `cli_rules.md`
(`docs_pointer_notes`) and `docs_writing.md` (`docs_routing_note`). This tale should
need no memory edits.

## Step 1: Apply the lander's carry-over fixes

The lander fixed and verified the following on master `021f9f2316`, but a plan handoff
does not commit its workspace. The exact diff is the patch artifact
`file:explicit:6243ebd33d7b5ba43264a0b8`. Read it with `sase artifact read` and get its
path with `sase artifact path`. Then run `git apply --check <path>` and
`git apply <path>`. Skip any hunk already present. If master moved and a hunk no longer
applies, re-create it by hand from this list:

1. **`mkdocs-pdf.yml`**: change the `exclude_docs` rule `!index.md` to `!/index.md`.
   - Comment: "Anchor the homepage rule: a bare `index.md` would also keep every
     Reference and Contribute hub".
   - The bare pattern re-included every hub `index.md`, so the strict PDF build failed
     on MkDocs' "not included in the nav" warning.
2. **`docs/overrides/main.html`**: add a `{% block content %}` override for the PDF
   build only (`{% if config.extra.pdf_handbook %}`). It renders
   `{{ super() | replace(' poster="images/', ' poster="file://' ~ config.site_dir ~ '/' ~ page.url ~ 'images/') }}`
   and plain `{{ super() }}` otherwise.
   - Why: mkdocs-exporter prints each page from a temporary file. It rewrites `src` and
     `href` but not `<video poster>`, so the homepage hero poster raised four strict
     `ERR_FILE_NOT_FOUND` errors.
   - Do not use MkDocs `hooks:`. The docs contract (`tools/check_docs` rule 6) forbids
     them.
3. **`docs/stylesheets/pdf.css`**: add `.md-path` to the hidden-chrome selector list
   beside `.md-nav`.
   - Material's `navigation.path` breadcrumbs ("Home Get started") printed above each
     chapter title, so `tools/postprocess_docs_pdf` could not find chapter start pages.
4. **`tools/validate_docs_pdf`**: replace the stale sentinels, which were removed by the
   sase-1le.19 homepage rewrite and the retired `getting_started.md`:
   - sentinel "Why Coding Agents Need Orchestration" → "One place to run agents and land
     their work";
   - numbered title check `1structuredagenticsoftwareengineering` →
     `1runateamofcodingagents`;
   - nested-heading check "Step 1 — Install SASE" → "3.1 Before you start".
5. **`tools/render_docs_map`**: skip pages whose frontmatter has `draft: true` (use
   `docs_render_utils.parse_frontmatter`), then regenerate `docs/README.md` and
   `docs/llms.txt` with `just fix`.
   - The blog plugin never publishes drafts, yet `llms.txt` listed nine of them, and
     `just docs-deploy-artifact-check` forbids that.
6. **Tests:**
   - `tests/test_docs_pdf_tools.py`:
     - `test_pdf_config_keeps_only_the_root_index_page` (uses `pathspec.GitIgnoreSpec`);
     - `test_pdf_build_resolves_relative_video_posters`, which renders `main.html` over
       a stub `base.html` with jinja2 in both PDF and web modes.
   - `tests/test_docs_moves.py`: the docs-map fixture gains a draft post that must not
     appear.
   - `tests/main/test_parser_root_help.py`: expect the compact memory help "Inspect and
     initialize SASE memory, and audit reference memory activity.". Phase `.15` changed
     that help text in `367562ba52` without updating this test, and Master Gate is red
     on it.
   - `tests/test_powerful_variables_landing.py`:
     `test_published_docs_do_not_restore_var_show` scans every `docs/**/*.md` plus the
     `sase_var` skill instead of the retired `configuration.md` and `cli.md` stubs.

Verify: `just docs-lint`, `just docs-check`, `just docs-pdf-check`,
`just docs-deploy-artifact-check`, and pytest on the four test files above. Locally, all
four Deploy Docs stages passed (PDF: 56 pages, 2.0 MiB). On master, Deploy Docs has been
red since before the epic started.

## Step 2: Make the docs move tooling compute the ids MkDocs renders

The tooling's anchor model disagrees with Python-Markdown's `toc` extension, which
MkDocs uses (see `toc` under `markdown_extensions` in `mkdocs.yml`):

- **Duplicate headings.** The tools spell them `slug-1`, `slug-2`. MkDocs renders
  `slug_1`, `slug_2` through `toc.unique()`: it appends `_N` and bumps an existing `_N`
  suffix.
- **Underscores.** `tools/docs_moves.py:slugify_heading` deletes every `_`
  (`re.sub(r"[*_~]", "", title)`). MkDocs keeps underscores that are literal text, such
  as those inside code spans: `` ## `ace.agent_decks` `` renders `aceagent_decks`, not
  `aceagentdecks`.
- **Separators.** The tools turn whitespace runs into `-` but never collapse hyphen
  runs. MkDocs collapses `[-\s]+` to one `-`: `name -r why` renders `name-r-why`, not
  `name--r-why`.

Fix the shared helpers so the tools record and pin the ids MkDocs actually renders:

- **`tools/docs_moves.py`:**
  - `slugify_heading`: derive the heading's rendered text first. Code-span content stays
    verbatim. Markdown emphasis delimiters, link URLs (keep the link text), and HTML
    tags go. Entities are unescaped.
  - Then apply `toc.slugify` semantics: NFKD to ASCII ignoring errors, drop `[^\w\s-]`,
    strip, lowercase, and collapse `[-\s]+` to `-`. Keep the explicit `{#id}`
    short-circuit.
  - `page_anchors`: use `unique()` semantics for duplicates.
- **`tools/docs_split`:**
  - `_headings` (around line 132): the same duplicate semantics.
  - `_target_used_ids` and `_unique_anchor`: the natural id on the target page must use
    `_N`.
  - The explicit-id pinning near line 552 must still add `{#id}` whenever the anchor
    recorded in `moves.yml` differs from what MkDocs would render.
  - The old-id collision suffixes (`f"{old_id}-{suffix}"` near lines 570 and 705) become
    explicit ids. Keep them only if they are always written explicitly.
- **`tools/docs_move`:** uses `page_anchors` for `--stub` anchor maps, so it inherits
  the fix. Confirm it.
- **Existing records stay.** Do not rewrite existing `-N` or underscore-stripped keys in
  `docs/_meta/moves.yml`. They are harmless extra ids on stubs, and the decision-cited
  stubs (`tool.md`, `macros.md`, `rust_backend.md`) depend on them.

Tests (`tests/test_docs_moves.py`, or a new focused module if that file nears the
`toobig` limit):

- Fixtures for each case above: duplicates, a code-span underscore, a hyphen collapse,
  and an explicit `{#id}`.
- A parity test guarded by `pytest.importorskip("markdown")`. Over every hand-written
  page under `docs/` (skip generated pages and `docs/_meta/`), assert that
  `page_anchors` heading ids equal the ids from
  `markdown.Markdown(extensions=[...the mkdocs.yml markdown_extensions that affect headings...]).toc_tokens`.
  - Python-Markdown is a MkDocs dependency, not a direct dev dependency. That is why the
    test skips when it is absent.
- A `docs_split` round-trip in which a duplicate heading lands on a target that already
  has the same title, proving the recorded anchor resolves without a hand-pinned id.

If `docs/contribute/writing-docs.md` states the old suffix convention, correct it.

## Step 3: Restore the legacy anchors the epic lost

Measure with the lander's audit script, artifact
`file:explicit:1770ea6d7e364bbca63ec7e4`. Its result on master `021f9f2316` is artifact
`file:explicit:5af1bbdc270440d8260ebb21`: 99 of 1,714 pre-epic anchors are unresolved.
Run the script from the repo root after `just docs-check` has built `site/`:

```bash
.venv/bin/python "$(sase artifact path file:explicit:1770ea6d7e364bbca63ec7e4)" b94cffe0d3 site
```

`b94cffe0d3` is the parent of the epic's first commit (`900925c994`). Repair each class:

- **Stub pages, 97 anchors.** In the matching `anchors:` map in `docs/_meta/moves.yml`,
  add the MkDocs id as an extra key. Point it at the destination of its tool-spelled
  twin: the existing key that the old tool computed for the same heading. Examples:
  - `aceagent_decks` gets the same destination as `aceagentdecks`;
  - `keybindings_1` (ace.md) goes with `keybindings-1`;
  - `sase-bead-attachment-purge-id-name-r-why-sase-bead-attachment-prune` goes with the
    `...name--r-why...` key.

  When the destination page natively renders the MkDocs id, point at that id. Otherwise
  reuse the twin's target anchor. Prefer a small one-off script that pairs keys by
  re-slugifying each pre-epic heading both ways. Commit only the resulting `moves.yml`
  data, not the script.

- **Tables of contents.** `beads.md`, `configuration.md`, and `macros.md` dropped their
  `#table-of-contents` sections as pure navigation. Map each to its move's primary
  destination page with no fragment; `check_docs_urls` accepts a value without `#`.
- **`ace.md#keybindings-agents-tab`.** It is unresolved even without a suffix. Find
  where that H2 went (the Agents tab reference pages) and map it there.
- **Redirect, 2 anchors.** `prompt.md#command-inventory` and
  `prompt.md#rerun-a-prompt-optionally-editing-first` are whole-page redirects. Their
  fragments carry to `reference/prompt-history.md`, where an edit phase removed those
  sections.
  - Add `<a id="command-inventory"></a>` and
    `<a id="rerun-a-prompt-optionally-editing-first"></a>` at the content that now
    covers them.
  - `tools/check_docs_urls` does not verify redirect fragments. Add both URLs to
    `docs/_meta/url_contract.txt` if the contract checker follows redirects for
    fragments. Otherwise rely on the audit.

Run `just fix` to regenerate stubs, `just docs-check` (strict build and URL contract),
and the audit script until it reports 0 unresolved. `check_docs_urls` already proves
every stub `anchors:` key on both the stub and its target, so the new keys stay guarded.

## Step 4: Verify and close the landing

1. Run `just fix`, then `sase tool run check`. Treat UNKNOWN or NEW failures as yours.
   Record unrelated KNOWN ones in the close note. Do not run `just check-full`.
2. Run `sase bead epic-symbols sase-1le.20.1` and `sase bead epic-symbols sase-1le.20`.
   Both were empty at landing. If entries appear, resolve them per `symvision.md` or
   re-key them only to a still-open bead.
3. Close the epic. Never use `--force`.

   ```bash
   sase bead close sase-1le.20.1 --note "<verification>"
   ```

   The note names:
   - all 21 phases closed and verified;
   - triage recorded in the epic's LANDER FOLLOW-UP TRIAGE note (new tasks `sase-1m0`,
     `sase-1m1`, `sase-1m2`; +1 on `sase-19g` and `sase-1lx`);
   - the carry-over fixes and the Deploy Docs stages passing;
   - tool anchor parity and its tests;
   - the legacy anchor audit at 0 unresolved;
   - the `sase tool run check` result.

4. Run `just symvision` and confirm the whitelist is clean.
5. Set `status: done` in the frontmatter of the epic's plan file: the PLAN path shown by
   `sase bead read sase-1le.20.1`, `202610/docs_decompose.md` in the plans sidecar. Edit
   that repo through `/sase_repo` and include it in the commit declaration.
6. The epic's `parent_bead` is phase `sase-1le.20` ("Decompose the monoliths and empty
   the legacy allowlist") of epic `sase-1le`. Re-verify that phase's exit criteria from
   `plan:202610/sase_sh_docs_redesign.md` ("Phase decompose"):
   - no root `docs/*.md` remain except `index.md`, the generated `docs/README.md` map,
     and generated stubs;
   - `docs/_meta/budgets.yml` has `ceilings: {}`;
   - `just docs-lint` shows no `reference/` page over budget;
   - `just docs-check` passes the URL contract;
   - the success metrics are recorded (epic `sase-1le` note #6).

   Then close only that phase:

   ```bash
   sase bead close sase-1le.20 --note "<what you verified>"
   ```

   Never close `sase-1le`; its own land agent owns it.
