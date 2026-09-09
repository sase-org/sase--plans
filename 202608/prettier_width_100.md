---
tier: epic
title: Unify the Markdown prose width behind one constant and move it from 120 to 100
goal: "Every place that wraps Markdown prose derives its width from a single declared
  source of truth guarded by a test, and that source declares 100 columns instead of 120
  across the sase repo and the chezmoi dotfiles repo.

  "
phases:
  - id: unify
    title: Collapse every prose-width declaration onto one source of truth
    depends_on: []
    size: medium
    description: "unify: introduce sase.markdown_width as the single width authority
      plus a package.json prettier config block, rewire all five Python declaration
      sites and both Justfile recipes to derive from them, and add a guard test that
      fails if any site re-forks the number. No width change and no Markdown reflow in
      this phase.

      "
  - id: flip
    title: Move the declared width from 120 to 100 and reflow the repo
    depends_on:
      - unify
    size: medium
    description: "flip: change the single width authority and the package.json mirror
      from 120 to 100, reflow every prettier-owned Markdown file, regenerate the derived
      memory shims, agent instruction files, and generated skill sources, and update the
      doc prose that names 120 explicitly.

      "
  - id: chezmoi
    title: Move the chezmoi repo and the editor formatter to 100
    depends_on:
      - flip
    size: small
    description:
      "chezmoi: move the chezmoi repo's own prettier recipes and the nvim conform
      prettier args from 120 to 100, reflow chezmoi's Markdown, and redeploy the
      generated skills so the skills sase writes at 100 pass chezmoi's own formatting
      gate."
proposed_by: bbugyi200.athena.uj
status: done
bead_id: sase-gt
create_time: 2026-09-09 19:51:09
---

- **PROMPT:**
  [prompts/202608/prettier_width_100.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/prettier_width_100.md)
- **BEAD:**
  [sase-gt](https://github.com/sase-org/sase--beads/blob/main/pages/sase-gt/README.md)

# Plan: Unify the Markdown prose width behind one constant and move it from 120 to 100

## Problem

The repo-wide Markdown prose width is nominally unified at 120 columns, but that
agreement is a coincidence of seven independent literals rather than a single
declaration. Nothing structurally prevents them from drifting apart again, and two of
them live in a different repository. Changing the width today means finding and editing
all of them by hand, which is exactly the failure mode that produced the earlier
80-vs-120 split retired in commit `5da193482`.

### Every place the width is declared today

Inside this repo (all currently `120`):

| #   | Site                                            | Form                                                            | What it governs                                                                                                                                                                                      |
| --- | ----------------------------------------------- | --------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `Justfile:296` (`fmt-md`)                       | `--prose-wrap=always --print-width=120` literal                 | `just fmt` over `**/*.md`                                                                                                                                                                            |
| 2   | `Justfile:324` (`fmt-md-check`)                 | same literal                                                    | `just check` / `just check-full` / CI (`.github/workflows/ci.yml:118`)                                                                                                                               |
| 3   | `src/sase/file_references.py:20`                | `DEFAULT_MARKDOWN_WRAP_WIDTH = 120`                             | default of `format_with_prettier()` and `format_markdown_files_with_prettier()` — plans, SDD writes, plan archive/header/link refresh, commit hooks, prompt archive, agent prompts, generated skills |
| 4   | `src/sase/main/_init_skills_rendering.py:18-23` | `_PRETTIER_FORMAT_ARGS` with a `--print-width=120` literal      | the second, independent prettier argv used by generated-skill rendering                                                                                                                              |
| 5   | `src/sase/main/init_memory/formatting.py:8`     | `_MARKDOWN_WRAP_WIDTH = 120` (a `textwrap` width, not prettier) | generated memory shims — must stay a prettier fixpoint or `fmt-md-check` fails on `AGENTS.md` etc.                                                                                                   |
| 6   | `src/sase/memory/notes.py:34`                   | `_FRONTMATTER_WRAP_WIDTH = 120` (a `textwrap` width)            | `description:` frontmatter wrapping, shaped deliberately to match what prettier would keep                                                                                                           |
| 7   | `src/sase/markdown_wrap.py:8`                   | `DEFAULT_PROSE_WRAP_WIDTH = 120`                                | `sase bead show` / `sase plan show` display wrapping and the `--wrap` CLI default                                                                                                                    |

Sites 3 and 7 are already _intended_ to be the same number —
`tests/test_bead/test_markdown_wrap.py:204` asserts
`DEFAULT_PROSE_WRAP_WIDTH == DEFAULT_MARKDOWN_WRAP_WIDTH`. That assertion is the only
existing structural link between any two sites; sites 1, 2, 4, 5, and 6 are unguarded.

Outside this repo, in the `chezmoi` linked repo (also `120`):

| #   | Site                                                          | What it governs                                                                                              |
| --- | ------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| 8   | `Justfile:65`, `Justfile:83`, `Justfile:109`                  | chezmoi's own `just fmt-md` / `fmt-md-check`, run by its CI                                                  |
| 9   | `home/dot_config/nvim/lua/plugins/conform.lua:187` and `:211` | `prepend_args = { "--prose-wrap=always", "--print-width=120" }` — the editor formatter for Markdown and YAML |

Site 9 matters because it is a _writing_ code point: formatting a sase Markdown file
from the editor at 120 while the repo checks at 100 would silently break `just check`.
Site 8 matters because sase renders generated skills and deploys them into chezmoi
(`home/dot_claude/skills/*/SKILL.md` and the other provider trees, 18 skills × several
providers); if sase renders them at 100 and chezmoi checks at 120, chezmoi's CI fails on
files it does not own.

The other three linked repos (`sase-github`, `sase-nvim`, `sase-telegram`) and the
`sase-core` Rust repo do not invoke prettier at all and declare no prose width. They are
out of scope.

### Explicitly out of scope

These are unrelated 120s and must not be touched:

- `pyproject.toml:199` `line-length = 88` — Python source formatting, a different
  policy.
- `Console(width=120)` in TUI code and tests (e.g.
  `src/sase/ace/tui/widgets/renderable_text.py:14`) — terminal render widths, not prose
  widths.
- Every `120` that is a timeout, a byte limit, a preview truncation, or a histogram
  bucket.
- `src/sase/markdown_wrap.py:9` `MIN_PROSE_WRAP_WIDTH = 20` — a floor, not the policy
  width.

## Design: what "truly unified" means here

The width cannot collapse to literally one declaration, because two consumers cannot
read the same one:

- The **prettier CLI** (`just fmt-md`, CI, editors) resolves configuration from files on
  disk. It cannot import a Python constant.
- **sase's own programmatic prettier calls** run from an installed package and format
  files that live outside any repo (plans under `~/.sase/plans/`, prompt archives, agent
  prompts). They must pass an explicit `--print-width`, because prettier's config
  discovery would resolve to whatever repo the process happens to be sitting in — or to
  nothing.

So the target shape is **two declarations, pinned equal by a test**:

1. `package.json` gains a `"prettier"` config block — prettier 3.8.1 reads it natively
   (verified: `prettier --find-config-path` resolves to `package.json`, and the width
   takes effect). It holds `{"proseWrap": "always", "printWidth": <N>}`. This is the
   declaration the prettier CLI, CI, and any editor with config discovery see. Putting
   it in the existing `package.json` rather than a new `.prettierrc` avoids adding a
   root dotfile and keeps the prettier pin and its config in one file.
2. `src/sase/markdown_width.py` — a new leaf module (no sase imports, so anything may
   import it) declaring `MARKDOWN_PRINT_WIDTH: int` and a `prettier_markdown_argv()`
   helper returning the shared
   `["prettier", "--prose-wrap=always", f"--print-width={...}", "--parser=markdown"]`
   argv. This is the declaration every Python code point derives from.
3. A guard test reads `package.json` and asserts its `prettier.printWidth` equals
   `MARKDOWN_PRINT_WIDTH` and its `prettier.proseWrap` is `"always"`. That test is what
   makes the two declarations one policy rather than two habits.

Everything else becomes a derivation, and the `Justfile` stops declaring the width at
all.

## Phase 1 — Collapse every prose-width declaration onto one source of truth

**This phase changes no widths and must produce a zero-byte diff in every Markdown
file.** That property is what makes it independently reviewable: if `just fmt-md`
dirties anything, the refactor is wrong.

### Work

1. Add `src/sase/markdown_width.py`:
   - `MARKDOWN_PRINT_WIDTH = 120` (still 120 in this phase), with a docstring stating it
     is the single authority for Markdown prose width and naming `package.json`'s
     `prettier` block as its mirror.
   - `prettier_markdown_argv(*, print_width: int = MARKDOWN_PRINT_WIDTH) -> list[str]`
     returning the shared argv.
   - Keep the module free of sase imports so `markdown_wrap.py` can import it without a
     cycle.
2. Add the `"prettier": {"proseWrap": "always", "printWidth": 120}` block to
   `package.json`.
3. Rewire the Python sites:
   - `src/sase/file_references.py` — define
     `DEFAULT_MARKDOWN_WRAP_WIDTH = MARKDOWN_PRINT_WIDTH` (keep the existing name as a
     re-export; it is imported by tests and reads naturally at its call sites) and build
     both `subprocess.run` argvs from `prettier_markdown_argv(print_width=print_width)`
     instead of assembling them inline.
   - `src/sase/main/_init_skills_rendering.py` — replace the `_PRETTIER_FORMAT_ARGS`
     literal list with `prettier_markdown_argv()`.
   - `src/sase/main/init_memory/formatting.py` —
     `_MARKDOWN_WRAP_WIDTH = MARKDOWN_PRINT_WIDTH`.
   - `src/sase/memory/notes.py` — `_FRONTMATTER_WRAP_WIDTH = MARKDOWN_PRINT_WIDTH`.
   - `src/sase/markdown_wrap.py` — `DEFAULT_PROSE_WRAP_WIDTH = MARKDOWN_PRINT_WIDTH`.
4. Drop the width from the `Justfile`: `fmt-md` becomes
   `{{ prettier_bin }} --write "**/*.md"` and `fmt-md-check` becomes
   `{{ prettier_bin }} --check "**/*.md"`, both now governed by `package.json`. Verify
   by hand that `just fmt-md-check` still passes _before_ touching anything else, which
   proves the config block is being discovered rather than silently ignored.
5. Add `tests/test_markdown_print_width.py` (or extend
   `tests/test_format_with_prettier.py`) with the guard tests:
   - `package.json`'s `prettier.printWidth` equals `MARKDOWN_PRINT_WIDTH` and
     `prettier.proseWrap` is `"always"`.
   - The `Justfile` contains no `--print-width` or `--prose-wrap` flag at all (a regex
     over the file), so a future edit cannot reintroduce a second CLI declaration.
   - A source scan asserting that `MARKDOWN_PRINT_WIDTH`'s definition in
     `src/sase/markdown_width.py` is the only `print-width`/prose-width literal under
     `src/` — i.e. no other `--print-width=<digits>` string and no other module-level
     width constant bound to a bare integer literal. Keep this scan narrow and explicit
     about what it matches so it does not become a flaky whole-repo grep.
6. Update the tests that hardcode `120` so they derive from the constant instead — they
   are the reason a naive width flip would look green while lying:
   - `tests/test_format_with_prettier.py:35,48,49,65,66`
   - `tests/main/test_init_skills_formatting.py:40`
   - `tests/main/test_init_memory_formatting.py:61,107`
   - `tests/main/test_init_memory_managed_agents.py:167,219`
   - `tests/test_bead/test_markdown_wrap.py:204` — this assertion becomes trivially true
     once both sides derive from the same constant; replace it with an assertion that
     both names resolve to `MARKDOWN_PRINT_WIDTH`, or delete it in favor of the new
     guard test rather than leaving a tautology behind.

### Verification

- `just install` first (ephemeral workspace), then `just check-full`.
- `git diff --stat -- '*.md'` must be empty.
- `just fmt-md-check` must pass with the flags removed from the `Justfile`.
- Read `sase/memory/symvision.md` via `/sase_memory_read` before resolving any Symvision
  complaint about the new module or the re-exported constant.

## Phase 2 — Move the declared width from 120 to 100 and reflow the repo

### Work

1. Set `MARKDOWN_PRINT_WIDTH = 100` in `src/sase/markdown_width.py` and
   `printWidth: 100` in `package.json`. Those two edits are the entire policy change;
   everything below is derived output.
2. Run `just fmt-md`. This reflows about 144 tracked Markdown files: ~74 under `docs/`,
   31 under `src/` (including the generated-skill sources in
   `src/sase/xprompts/skills/`, the `src/sase/ace/` and `tools/` directory-level agent
   instruction files, and the asset prompt files), 12 under `sase/memory/`, 6 under
   `demos/`, 7 README files under `tests/`, 5 under `tools/`, `smoke/pypi/README.md`,
   and the root `README.md`, `CONTRIBUTING.md`, `INSTALL.md`, `AGENTS.md`, `CLAUDE.md`,
   `GEMINI.md`, `OPENCODE.md`, `QWEN.md`. Files under `.prettierignore` (`CHANGELOG.md`,
   `sdd/`, `sase/xprompts/*.md`, the memory and SDD Jinja templates,
   `tests/agents_sync/goldens/`) are untouched by design.
3. Regenerate derived artifacts, in this order, and commit the results with the flip:
   - `sase memory init` — regenerates `AGENTS.md` and the `CLAUDE.md` / `GEMINI.md` /
     `OPENCODE.md` / `QWEN.md` shims from the reflowed `sase/memory/*.md` notes at the
     new width. This is mandatory, not optional, per the repo's memory workflow rule.
   - `sase init --check` (and `sase init` where drift is reported) for the skills and
     directory-level agent instruction files, so the generated skills render at 100.
   - Re-run `just fmt-md` afterwards and confirm it is a fixpoint. The `textwrap`-based
     generators (`init_memory/formatting.py`, `memory/notes.py`) hand-wrap to _match_
     prettier; if they and prettier disagree at 100 anywhere, `fmt-md-check` will
     oscillate. Fix the generator, not the output, if that happens.
4. Update the doc prose that states the number in words — these do not follow the
   constant:
   - `docs/axe.md:396` — "the 120-column prose width".
   - `docs/beads.md:1087` — "wrap at 120 total columns by default".
   - `docs/beads.md:1106` — the `-w, --wrap` table row, "defaults to `120`".
   - The argparse help strings in `src/sase/main/parser_bead_queries.py:299` and
     `src/sase/main/parser_plan.py:461` interpolate `DEFAULT_PROSE_WRAP_WIDTH` already
     and need no edit — but confirm no committed doc snapshot pins the old rendered
     text.
5. Sweep for any other committed test expectation that encodes a wrapped-at-120 string.
   `tests/agents_sync/goldens/` is prettier-ignored and is renderer output rather than
   prose, but `src/sase/agents_sync/prompt_archive/render.py:110` does run the shared
   formatter, so run the full suite rather than the scoped lane and fix whatever the run
   surfaces.

### Note on memory-file permission

This phase edits `sase/memory/*.md`, `AGENTS.md`, and the provider instruction shims.
Those files normally require explicit user permission to touch. Approving this epic
**is** that permission, scoped to the mechanical reflow and the `sase memory init`
regeneration described here — no wording, ordering, or content changes to any memory
note.

### Verification

- `just check-full` (not `just check`) — the change touches the broadening set.
- `just fmt-md` twice in a row leaves the tree clean the second time.
- `sase memory init --check` and `sase init --check` both report no drift.
- Spot-check that no reflowed file gained a line longer than 100 columns except where a
  single unbreakable token (URL, inline code span, table row) legitimately exceeds it.

## Phase 3 — Move the chezmoi repo and the editor formatter to 100

This phase must land promptly after phase 2. Between the two, sase renders generated
skills at 100 while chezmoi's CI still checks them at 120, so any skill deploy in that
window fails chezmoi's gate.

### Work

Open the repo with `sase repo open chezmoi -r "<reason>"` and use only the path it
prints.

1. `Justfile:65`, `Justfile:83`, `Justfile:109` — change `--print-width=120` to
   `--print-width=100`. Chezmoi installs prettier globally rather than pinning it in a
   `package.json`, so keeping explicit flags there is correct; adding a config file to
   that repo is a separate decision and not part of this plan.
2. `home/dot_config/nvim/lua/plugins/conform.lua:187` and `:211` — change `prepend_args`
   to `{ "--prose-wrap=always", "--print-width=100" }`.

   Decision recorded: pin the editor to 100 rather than deleting `--print-width` in
   favor of per-repo config discovery. Dropping the flag would be the more principled
   unification — each repo's own `package.json`/`.prettierrc` would then govern — but
   conform's prettier formatter is global, so repos with no prettier config would
   silently fall back to prettier's defaults (80 columns, `proseWrap: preserve`). That
   blast radius is larger than this change should carry. Worth revisiting separately.

3. Run chezmoi's `just fmt-md`. This reflows about 124 of its 128 tracked Markdown
   files.
4. Re-run `sase init` (or `sase skill init`) from the sase workspace so the generated
   skills are redeployed into chezmoi at 100 and the `home/.sase-skills-manifest.json`
   hashes match, then confirm chezmoi's `just fmt-md-check` passes over the deployed
   `SKILL.md` trees.
5. Commit in the chezmoi repo through the normal sase commit workflow and apply.

### Verification

- chezmoi's `just check` (or at minimum `just fmt-md-check`) passes.
- From the sase workspace, `sase init --check` reports no skill drift against the
  deployed chezmoi tree.
- Format a sase Markdown file from the editor and confirm the result still satisfies
  `just fmt-md-check` in the sase repo — that is the end-to-end proof that the writing
  code point and the checking code point finally agree.

## Risks

- **Generator/prettier disagreement at 100.** `init_memory/formatting.py` and
  `memory/notes.py` reimplement prettier's wrapping with `textwrap` plus hand-rolled
  protections for inline code spans. Those approximations were tuned at 120 and may not
  be fixpoints at 100. Phase 2's "run `just fmt-md` twice" check is the detector; the
  fix belongs in the generator.
- **Review size.** Phase 2 is a ~144-file diff. Keeping phase 1 width-neutral is what
  makes phase 2 reviewable as "one-line policy change plus mechanical output".
- **Cross-repo window.** Phases 2 and 3 are coupled through the skill deploy path; see
  the phase 3 preamble.
