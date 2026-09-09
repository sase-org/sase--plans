---
status: done
tier: epic
title: Make the Markdown prose width a config field and default it to 88
goal: "`markdown.print_width` is a first-class SASE config field that every
  Markdown-emitting code path resolves at runtime, its shipped default is 88 instead of
  100, and the repo's own Markdown, generated artifacts, and dotfile formatter
  configuration all agree with the new default.

  "
phases:
  - id: config-field
    title: Runtime-resolved `markdown.print_width` config field
    depends_on: []
    size: medium
    description: "config-field: add the `markdown.print_width` schema/default/getter,
      turn `sase.markdown_width` into a runtime accessor instead of an import-time
      constant, migrate every consumer and test off the frozen constants, and extend the
      width guard suite to catch import-time snapshots. Effective width stays 100, so no
      Markdown reflows.

      "
  - id: default-88
    title: Flip the shipped default from 100 to 88
    depends_on:
      - config-field
    size: medium
    description: "default-88: change the default constant, `default_config.yml`, the
      schema default, and the `package.json` prettier mirror to 88, then reflow the
      repo's Markdown, regenerate every SASE-generated artifact, and correct the prose
      and tables that name 100 as the width.

      "
  - id: chezmoi-align
    title: Align chezmoi's prettier and conform configuration with the new default
    depends_on:
      - default-88
    size: small
    description:
      "chezmoi-align: move the chezmoi-managed prettier width declarations (its Justfile
      recipes and the Neovim conform.lua prettier args) from 100 to 88, apply the
      change, and verify the deployed skills and dotfiles agree with the new default."
proposed_by: bbugyi200.athena.sase-gt.land.f1
bead_id: sase-gy
create_time: 2026-09-09 19:50:11
---

- **PROMPT:**
  [prompts/202608/configurable_markdown_print_width.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/configurable_markdown_print_width.md)
- **BEAD:**
  [sase-gy](https://github.com/sase-org/sase--beads/blob/main/pages/sase-gy/README.md)

# Plan: Make the Markdown prose width a config field and default it to 88

## Background

`src/sase/markdown_width.py` is the single authority for the repo-wide Markdown prose
width, established by epic `sase-gt`. Today it is a hard-coded module constant:

```python
MARKDOWN_PRINT_WIDTH = 100

def prettier_markdown_argv(*, print_width: int = MARKDOWN_PRINT_WIDTH) -> list[str]: ...
```

The width has to become configurable, and its shipped default has to become 88. Those
are two genuinely separable changes — the first is mechanism with no behavioral change,
the second is a policy flip whose blast radius is every Markdown file SASE has ever
formatted — so they are separate phases.

### Why this is not a one-line change

The constant is consumed at **import time** in eight places, which is exactly what a
configurable value cannot be. Every one of these freezes the width into a module
attribute, a function-signature default, or an argparse default before any config is
read:

| Site                                      | Shape                                                                |
| ----------------------------------------- | -------------------------------------------------------------------- |
| `markdown_width.py:24`                    | `prettier_markdown_argv(*, print_width: int = MARKDOWN_PRINT_WIDTH)` |
| `markdown_wrap.py:10`                     | `DEFAULT_PROSE_WRAP_WIDTH = MARKDOWN_PRINT_WIDTH`                    |
| `file_references.py:23`                   | `DEFAULT_MARKDOWN_WRAP_WIDTH = MARKDOWN_PRINT_WIDTH`                 |
| `file_references.py:521,566`              | two `print_width: int = DEFAULT_MARKDOWN_WRAP_WIDTH` defaults        |
| `memory/notes.py:37`                      | `_FRONTMATTER_WRAP_WIDTH = MARKDOWN_PRINT_WIDTH`                     |
| `main/init_memory/formatting.py:12`       | `_MARKDOWN_WRAP_WIDTH = MARKDOWN_PRINT_WIDTH`                        |
| `main/parser_bead_queries.py:254,297,300` | argparse `default=` plus two help strings                            |
| `main/parser_plan.py:458,461`             | argparse `default=` plus a help string                               |

Two call sites use the constant inline and are already fine once the name changes:
`main/_init_skills_rendering.py:99,101` (`len(...) > MARKDOWN_PRINT_WIDTH` and
`width=MARKDOWN_PRINT_WIDTH - len(_YAML_BLOCK_INDENT)`).

The payoff is large: **every** programmatic prettier call in the codebase already goes
through `file_references.format_with_prettier()` /
`format_markdown_files_with_prettier()` with no explicit width — plan archive writes,
plan header/link refresh, commit hooks, prompt-archive rendering, SDD writes, and the
generated-skill renderer. Making the default resolve at call time makes all of them
config-aware with no per-call-site work.

### Why `package.json` still mirrors the default and not the effective value

`tests/test_markdown_print_width.py` pins `package.json`'s `prettier.printWidth` equal
to the Python declaration, because the prettier CLI (`just fmt-md`, CI, editors) reads
config from disk and cannot import a Python constant. That mirror must keep pointing at
the **shipped default**, not the user's effective configured value:

- A stock checkout must be self-consistent for a contributor who has no SASE config at
  all.
- Tests run with an isolated `CONFIG_DIR` (`tests/conftest.py::_isolate_sase_home`), so
  an effective value would be un-assertable without leaking developer config into the
  suite.

The consequence is a documented sharp edge, not a bug: a user who configures a
non-default width and runs `sase init` inside a repo whose prettier config still says 88
will see `just fmt-md-check` fail on the regenerated `AGENTS.md`. Phase `config-field`
documents this in `docs/configuration.md`. A `sase doctor` check that compares the
effective width against the current project's prettier config is a deliberate
**non-goal** here (see Out of scope).

## Phase `config-field`: Runtime-resolved `markdown.print_width` config field

**No behavior change.** The effective width stays 100 for the whole phase, so no
Markdown file in this repo reflows and `just fmt-md-check` keeps passing untouched. That
isolation is the point: it lets this phase's diff be reviewed as pure mechanism.

### 1. Config surface

Add a new top-level `markdown` section. A section rather than a flat
`markdown_print_width` scalar, so a future `prose_wrap` or per-surface override has an
obvious home.

`src/sase/default_config.yml` — place it next to the other small policy sections (after
`tasks:`):

```yaml
markdown:
  # Column width SASE wraps generated Markdown prose at: plan files, bead notes and pages, memory
  # shims, generated skills, prompt archives, and the default `--wrap` for `sase bead show` and
  # `sase plan show`. The prettier CLI cannot read this value, so repos whose Markdown SASE writes
  # into need their own prettier config to match.
  print_width: 100
```

`src/sase/config/sase.schema.json` — the top level is `additionalProperties: false`, so
the schema entry is mandatory or `test_default_config_matches_public_schema` fails
immediately:

```json
"markdown": {
  "type": "object",
  "description": "Markdown prose formatting policy for SASE-generated Markdown.",
  "additionalProperties": false,
  "properties": {
    "print_width": {
      "type": "integer",
      "minimum": 20,
      "default": 100,
      "description": "Column width SASE wraps generated Markdown prose at ..."
    }
  }
}
```

The `minimum` of 20 is not arbitrary — it is `markdown_wrap.MIN_PROSE_WRAP_WIDTH`, below
which `wrap_markdown()` silently returns text unwrapped. Pin that correspondence with a
test (below) so the two cannot drift.

Config Center needs no work: `build_config_inventory()` flattens the schema through the
Rust `config_inventory` binding, so the field appears in the Config tab automatically.
The Config Center PNG snapshots use a stub schema
(`tests/ace/tui/visual/_ace_config_center_config_helpers.py::_config_schema`), so they
are unaffected — confirm this rather than assuming it.

### 2. The width authority

Rewrite `src/sase/markdown_width.py`. Rename the constant so no caller can keep treating
it as the effective width, and add the runtime accessor:

```python
DEFAULT_MARKDOWN_PRINT_WIDTH = 100


def markdown_print_width() -> int:
    """Return the configured Markdown prose width."""
    from sase.config import get_markdown_print_width  # deferred: see below

    return get_markdown_print_width()


def prettier_markdown_argv(*, print_width: int | None = None) -> list[str]:
    width = markdown_print_width() if print_width is None else print_width
    ...
```

The `sase.config` import **must** stay function-local. The module docstring's promise
that `markdown_width` "imports nothing from `sase` so that any module may import it
without risking a cycle" is load-bearing in the other direction too: `sase.config.core`
will now import `DEFAULT_MARKDOWN_PRINT_WIDTH` from this module. A module-level import
here would make that a real cycle. Update the docstring to explain both directions
rather than deleting the old claim.

### 3. The config getter

`src/sase/config/core.py`, following the `get_task_history_limit` shape exactly:

```python
from sase.markdown_width import DEFAULT_MARKDOWN_PRINT_WIDTH

def get_markdown_print_width() -> int:
    """Return the validated configured Markdown prose width."""
    markdown = load_merged_config().get("markdown", {})
    value = (
        markdown.get("print_width", DEFAULT_MARKDOWN_PRINT_WIDTH)
        if isinstance(markdown, dict)
        else DEFAULT_MARKDOWN_PRINT_WIDTH
    )
    if type(value) is int and value >= MIN_PROSE_WRAP_WIDTH:
        return value
    return DEFAULT_MARKDOWN_PRINT_WIDTH
```

Re-export `DEFAULT_MARKDOWN_PRINT_WIDTH` and `get_markdown_print_width` from
`src/sase/config/__init__.py` and add both to its `__all__` (the list is alphabetized;
keep it that way). Formatting must never hard-fail: if config loading raises, fall back
to the default the way `get_runner_slot_deference_seconds_per_step` does, and do not let
a broken `~/.config/sase/sase.yml` turn `sase plan propose` into a traceback.

Import the floor from `markdown_wrap` if that stays acyclic; otherwise move
`MIN_PROSE_WRAP_WIDTH` into `markdown_width.py` and have `markdown_wrap` re-export it.
Either is fine — decide from what the import graph actually allows, and note the
decision in the phase bead.

### 4. Migrate the consumers

Delete every import-time snapshot listed in the Background table and replace it with a
call:

- `markdown_wrap.DEFAULT_PROSE_WRAP_WIDTH` and
  `file_references.DEFAULT_MARKDOWN_WRAP_WIDTH`: **remove** them, including from
  `markdown_wrap.__all__`. They are aliases whose only remaining job would be to freeze
  the value. `just _lint-symvision` will flag any straggler.
- `file_references.format_with_prettier` / `format_markdown_files_with_prettier`: change
  `print_width: int = DEFAULT_MARKDOWN_WRAP_WIDTH` to `print_width: int | None = None`
  and resolve inside. Do **not** resolve in these two and also in
  `prettier_markdown_argv` — pass `None` straight through so there is exactly one
  resolution point.
- `memory/notes.py` and `main/init_memory/formatting.py`: these hand-rolled `textwrap`
  wrappers must stay a fixpoint of prettier's width, so they call
  `markdown_print_width()` inside the wrapping function. Keep the existing comments
  explaining _why_ they must match; they are still true.
- `main/_init_skills_rendering.py:99,101`: swap the constant for a single local
  `width = markdown_print_width()` at the top of `_build_output`, so both uses in that
  branch agree.
- `main/parser_bead_queries.py` and `main/parser_plan.py`: resolve at **parser-build**
  time — `default=markdown_print_width()` and the same value interpolated into help
  text. Parser construction is per-process and lazy per subcommand (`main/parser.py`
  maps each subcommand to its registration module), so this reflects live config and
  keeps `--help` honest. Confirm that `sase --help` does not eagerly build every
  subparser; if it does, note it rather than restructuring the parser registry in this
  phase.
- `bead/cli_query.py:156`: `getattr(args, "wrap", DEFAULT_PROSE_WRAP_WIDTH)` becomes
  `getattr(args, "wrap", markdown_print_width())`.

Do not call `markdown_print_width()` inside a per-row or per-line render loop.
`load_merged_config()` is cached, but the config-token cache still stats the filesystem
every 0.75s; hoist the call to the top of the enclosing function. `bead/cli_detail.py`
and `main/plan_show_render.py` already receive an explicit width and need no change.

### 5. Guard tests

`tests/test_markdown_print_width.py` is the reason the width is one policy instead of
two habits, and it currently only defends against _literals_. A configurable width
introduces a new failure mode it cannot see: a module that snapshots the accessor at
import time is syntactically clean and silently ignores config. Extend the suite:

- Keep and rename the existing checks for `DEFAULT_MARKDOWN_PRINT_WIDTH`, including the
  two inline-literal AST guards added at `sase-gt`'s landing (bare `width=<int>` keyword
  arguments and `len(...) > <int>` thresholds, scoped to modules importing
  `sase.markdown_width`).
- **New** — no module outside the authority may bind a module-level name to
  `markdown_print_width()` or to `DEFAULT_MARKDOWN_PRINT_WIDTH`. This is the
  import-time-snapshot guard; it is what makes the deletions in step 4 stay deleted.
- **New** — no function parameter default anywhere under `src/` may be a call to
  `markdown_print_width()` or a reference to `DEFAULT_MARKDOWN_PRINT_WIDTH`. A default
  argument is evaluated once at `def` time, so this is the same bug wearing a different
  hat, and it is exactly the shape `prettier_markdown_argv` used to have.
- **New** — the three-way default contract, following
  `tests/test_artifact_capture_policy.py::test_capture_config_default_and_schema`: the
  constant, the `default_config.yml` value, and the schema `default` must all agree, and
  the schema `minimum` must equal `MIN_PROSE_WRAP_WIDTH`.
- **New** — behavioral coverage that the field actually works: patch
  `load_merged_config` and assert `markdown_print_width()`, `prettier_markdown_argv()`,
  and `format_with_prettier()` all follow the configured value; that an out-of-range,
  wrong-typed, or non-mapping `markdown` section falls back to the default; and that a
  raising `load_merged_config` still returns the default.

Keep the non-vacuity assertion in `_width_aware_modules()` — with the constant renamed,
a typo in the detection string would silently disable all four AST guards.

Update the tests that assert against the old names:
`tests/test_format_with_prettier.py`, `tests/main/test_init_skills_plan.py`,
`tests/main/test_init_memory_formatting.py`, `tests/test_bead/test_markdown_wrap.py`,
`tests/main/test_parser_plan.py`.

### 6. Docs

Add a `### markdown` section to `docs/configuration.md` in `## Configuration Sections`,
positioned to match the `default_config.yml` ordering, with the same
YAML-block-plus-field-table shape the `tasks` section uses. Add the matching
`- [markdown](#markdown)` entry to the Table of Contents. State plainly which surfaces
the field governs and record the `package.json` sharp edge described in the Background.

Do **not** touch `docs/beads.md:1195,1215` or `docs/axe.md:418` in this phase — they
correctly say 100 until phase `default-88` changes it.

### Verification

`just install` first (workspaces are ephemeral), then `just check-full`. This phase
touches the config schema and the shared width authority, both of which are in the
broadening set, so the scoped lane is not sufficient. `just fmt-md-check` passing
unchanged is the proof that the phase really was behavior-preserving.

## Phase `default-88`: Flip the shipped default from 100 to 88

### 1. The flip

Four declarations move together; splitting them across commits leaves the tree failing
`fmt-md-check`:

- `src/sase/markdown_width.py`: `DEFAULT_MARKDOWN_PRINT_WIDTH = 88`
- `src/sase/default_config.yml`: `markdown.print_width: 88`
- `src/sase/config/sase.schema.json`: the `markdown.print_width` `default`
- `package.json`: `prettier.printWidth: 88`

The three-way contract test from phase `config-field` fails loudly if any one is missed,
which is what it is for.

### 2. Reflow the repo

`just fmt-md` reflows roughly **140** Markdown files at 88 (measured with prettier 3.8.1
against the current tree; re-measure rather than trusting this number). Breakdown: ~32
under `src/sase` (including the `src/sase/xprompts/skills/*.md` skill sources), ~34
under `docs/`, 12 under `sase/memory/`, the five root instruction shims plus
`README.md`, `INSTALL.md`, `CONTRIBUTING.md`, the five under `tools/`, and assorted test
fixtures.

`.prettierignore` already excludes `CHANGELOG.md`, `sdd/` planning artifacts,
`sase_plan_*.md`, `sase/xprompts/*.md`, the generated-Markdown template sources, and
`tests/agents_sync/goldens/`, so those do not churn from the reflow itself.

**`sase/memory/*.md` requires care.** Twelve canonical memory files reflow. This is
mechanical rewrapping only — no entry may be added, edited, or removed, and no sentence
may change. The repo rule requiring explicit user permission for memory edits is
satisfied for _reflow_ by the user's request to change the width, which cannot be
honored without it; it is not authorization to change memory content. Review the memory
diff line by line and confirm it is pure rewrapping. After the reflow, run
`sase memory init` so `AGENTS.md`, the provider instruction shims, and the memory README
are regenerated from the reflowed sources rather than hand-edited.

### 3. Regenerate everything SASE generates

The reflow is only half the work — SASE's own generators now emit at 88 and their
outputs are checked in or deployed:

- `sase memory init` → `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`
  (root and `tools/`), memory README.
- `sase init skills` → deployed skills. Then `sase init skills --check` must be clean.
- `sase init --check` must report every subsystem clean when the phase is done. That
  command is the drift detector for this whole phase; a non-clean result means something
  still emits at 100.

Then re-run the full suite and update any golden that legitimately moved.
`tests/agents_sync/goldens/*.md` are the ones to watch:
`agents_sync/prompt_archive/render.py:113` calls `format_with_prettier(document)` with
no explicit width, so those exact-renderer snapshots are produced at the configured
width even though `.prettierignore` exempts them from `just fmt-md`. Regenerate rather
than hand-edit them.

### 4. Correct the prose that names 100

Grep for width claims and fix each — at minimum:

- `docs/beads.md:1195` — "wrap at 100 total columns by default"
- `docs/beads.md:1215` — the `-w, --wrap` table row, "defaults to `100`"
- `docs/axe.md:418` — "keep the file inside the 100-column prose width"
- the `docs/configuration.md` section added in phase `config-field`, plus its example
  block

Prefer wording that names the config field as the source of truth over restating a
number, so the next width change does not require another prose sweep.

### 5. Changelog

This changes the shape of every Markdown file SASE writes, which is user-visible. Make
sure the conventional-commit type and body reflect that (`feat` with a body noting the
default change), so release-please surfaces it.

### Verification

`just install`, then `just check-full` — the full suite, not the scoped lane.
`just fmt-md-check` and `sase init --check` are the two gates that actually prove the
flip is complete and consistent. Run `just test-visual` as well: the ACE PNG snapshot
suite is excluded from `just test`, and while no snapshot is expected to move,
"expected" is not "verified".

## Phase `chezmoi-align`: Align chezmoi's prettier and conform configuration

Epic `sase-gt` set the chezmoi-side prettier width to 100 in two places, which are now
stale:

- the chezmoi `Justfile` prettier recipes (three sites at `sase-gt` time)
- the Neovim `conform.lua` prettier arguments (two sites at `sase-gt` time)

Open the chezmoi repo with the `/sase_repo` skill and use only the path it prints — do
not clone, guess a path, or fetch these files over the web. Re-derive the exact line
numbers; `sase-gt`'s are a starting point, not a contract.

Move both from 100 to 88, apply the chezmoi change to `~/`, and verify:

- chezmoi's own `just fmt-md-check` passes over its tree at the new width
- the deployed skills under `~/.claude/skills/` are byte-identical to what
  `sase init skills` renders, and `sase init skills --check` is clean
- the deployed `conform.lua` in `~/.config/nvim/` carries the new value

Also check whether `~/.config/sase/sase.yml` (chezmoi-managed) should set
`markdown.print_width` explicitly. It should **not** need to — 88 is now the shipped
default, and pinning the default in user config is noise that silently detaches from
future default changes. Only add it if the user wants a width other than 88; record the
outcome either way in the phase bead.

## Out of scope

- **A `sase doctor` check for width drift** between the effective config and the current
  project's prettier config. Genuinely useful, but prettier config discovery has many
  shapes (`package.json`, `.prettierrc`, `.prettierrc.json`, `prettier.config.js`) and
  this epic is already wide. File it as a task bead at landing if it still seems worth
  it.
- **Making `prose_wrap` configurable.** Always-on prose wrapping is a policy this epic
  does not revisit; the `markdown:` section leaves room for it later.
- **Teaching external formatters to read SASE config.** `conform.lua` and repo
  `package.json` files keep their own declarations; phase `chezmoi-align` syncs them by
  hand, as `sase-gt` did.
- **Reflowing Markdown in the linked plugin repos** (`sase-github`, `sase-telegram`,
  `sase-nvim`) or the SDD sidecars. They have their own prettier configuration and their
  own landing cadence.
