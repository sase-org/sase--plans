---
create_time: 2026-07-12 16:07:59
status: wip
prompt: 202607/prompts/generated_markdown_templates.md
tier: tale
---
# Plan: Single-Source Templates for Every SASE-Generated Markdown File

## Problem / Product Context

The `amd_agents_template` work gave `AGENTS.md` (and its provider shims) a single human-editable Markdown template.
Users now want the same guarantee for **all** markdown files that sase generates: for each generated file there must be
a single markdown file that contains either its contents verbatim or a template of its contents. Today the remaining
generated documents are still composed from hardcoded Python string lists, which hides agent-facing and human-facing
prose inside rendering logic where it cannot be reviewed or edited as prose.

## Complete Inventory of SASE-Generated Markdown

A full sweep of `src/sase` found every code path that composes and writes Markdown. Dispositions:

| Generated file                                                                    | Composer today                                                                                                             | Disposition                              |
| --------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------- |
| `AGENTS.md` + provider shims (`CLAUDE.md`, `GEMINI.md`, `QWEN.md`, `OPENCODE.md`) | `src/sase/amd/templates/AGENTS.template.md` + `AGENTS.minimal.template.md`                                                 | Done (prior work); shims are byte copies |
| `memory/sase.md`                                                                  | hardcoded in `root_rendering.py::_render_sase_memory` (+ `_extend_workspace_section`, `_extend_linked_repository_section`) | **Phase 2: extract template**            |
| `memory/README.md`                                                                | hardcoded in `root_rendering.py::_render_memory_readme`                                                                    | **Phase 3: extract template**            |
| `sdd/README.md`                                                                   | `SDD_README_CONTENT` constant in `src/sase/sdd/_init_files.py:22`                                                          | **Phase 4: extract packaged .md**        |
| `sdd/plans/README.md`, `sdd/research/README.md`                                   | `SDD_DIRECTORY_README_CONTENT` dict in `_init_files.py:61`                                                                 | **Phase 4: extract packaged .md**        |
| companion `plans/README.md`, `research/README.md`                                 | `SDD_COMPANION_README_CONTENT` dict in `_init_files.py:76`                                                                 | **Phase 4: extract packaged .md**        |
| Generated skill `SKILL.md` bodies                                                 | already per-skill source templates in `src/sase/xprompts/skills/*.md`                                                      | Already single-source (no change)        |
| Generated skill `SKILL.md` frame (frontmatter + audit directive)                  | hardcoded in `_init_skills_rendering.py::_build_output` / `_skill_use_audit_directive`                                     | **Phase 5: extract frame template**      |

Enumerated and deliberately **excluded** (rationale in Boundaries): files where sase only wraps verbatim user/agent
content in structural YAML frontmatter (SDD plan/prompt snapshots via `sdd/_write.py`, saved xprompts via
`xprompt/save.py`, approved memory notes via `memory/proposals/review.py`, long-note frontmatter refreshes via
`amd/_memory.py::_long_memory_description_updates`), machine-parsed archive formats (chat transcripts via
`history/chat.py::save_chat_history`), and ephemeral run artifacts (launch previews, per-run workflow logs, prompt/reply
snapshots, commit-finalizer responses, prettier scratch files).

## Current State (key mechanics to reuse)

- **Template loader prior art** — `src/sase/amd/_template.py` loads packaged templates via
  `importlib.resources.files("sase.amd").joinpath("templates", ...)`, renders with Jinja2 `StrictUndefined`
  (`keep_trailing_newline=True`), validates syntax plus missing/unknown placeholders via
  `meta.find_undeclared_variables`, and returns `(rendered, None) | (None, blocker_message)`; callers wrap failures as
  `AmdMemorySyncPlan.blockers` so nothing is written on error.
- **Override prior art** — `src/sase/amd/_config.py::resolve_amd_template_override`: project `sase.yml` key
  (root-relative, containment-validated) → user config dir for home roots (`~/.config/sase/`, or
  `CHEZMOI_HOME/dot_config/sase` for the chezmoi source root, via `_user_config_dir_for_home_amd_root`) → packaged
  default. Config keys are mirrored in `src/sase/default_config.yml` and `src/sase/config/sase.schema.json`.
- **`memory/sase.md`** — body composed in `root_rendering.py:96-127`; only dynamic inputs are `project_name` (workspace
  section emitted only when set) and the linked-repo entry list (bullet list; empty-state prose and a
  `sase workspace open` guidance block emitted only when entries exist). Frontmatter (`type: short`,
  `parent: AGENTS.md`) is applied separately by `apply_memory_frontmatter`. Downstream contract: it is a short note, so
  `validate_short_memory_structure` (exactly one H1, first heading, nothing deeper than H3, fence-aware) must hold or
  AGENTS.md inlining blocks.
- **`memory/README.md`** — composed in `root_rendering.py:251-303`; only the `## Memory Notes` blocks and
  `## Statistics` counters are data-driven, everything else is one-off prose. Nothing parses it back (note discovery
  skips `README.md`); drift is a pure byte re-render + compare in `root_planning.py::_compare_expected_memory_files`.
- **SDD READMEs** — five fully static documents stored as Python string constants, planned/written through
  `SddExpectedTextFile` in `src/sase/sdd/_init_files.py`; binary directory-map PNGs already ship as packaged assets.
- **Skill `SKILL.md`** — body rendered from per-skill sources in `src/sase/xprompts/skills/` by
  `_init_skills_rendering.py::_render_skill`; the surrounding frame (YAML frontmatter with wrapping rules for long or
  multiline descriptions, plus the `sase skill use <name>` audit directive controlled by `log_skill_use`) is hardcoded
  in `_build_output`, then Prettier-formatted.
- **Formatting** — `format_generated_memory_markdown` post-processes sase.md/README.md/AGENTS.md; generated skills use
  Prettier. Both passes stay unchanged and run after template rendering.

## Proposed Design

### Phase 1 — Shared markdown-template loader

Generalize the proven `sase/amd/_template.py` machinery into a small shared module (e.g. `src/sase/mdtemplates.py`)
providing one entry point: load a template (explicit override path or packaged `importlib.resources` anchor + filename),
validate Jinja syntax and the exact required-placeholder set (missing and unknown placeholders are both errors, with the
same actionable message shapes used today, labeled with the resolved template path or `packaged <filename>`), render
with `StrictUndefined`, and return `(rendered, None) | (None, error)`. `sase/amd/_template.py` is refactored to delegate
to it with byte-identical behavior and error messages. Conditional placeholders (`{% if var %}`) are permitted —
`meta.find_undeclared_variables` already accounts for them — but structured-list looping stays out of scope (dynamic
collections are passed as pre-rendered strings, exactly like `tier1_sections`).

Also generalize the override-resolution helper so each templated file is described by (project `sase.yml` key, user
config-dir filename, packaged anchor + filename) and resolved through the identical precedence chain as
`resolve_amd_template_override`.

### Phase 2 — `memory/sase.md` template

1. Create packaged `src/sase/main/init_memory/templates/memory-sase.template.md` holding **all** current prose,
   including the conditional sections, e.g.:

   ```markdown
   # SASE = Structured Agentic Software Engineering

   {% if project_name %}

   ## Ephemeral `{{ project_name }}_<N>` Workspace Directories

   ...current hardcoded prose... {% endif %}

   ## Linked Repositories

   {% if linked_repo_entries %} Configured linked repositories for this context:

   {{ linked_repo_entries }}

   ...current `sase workspace open` guidance prose... {% else %} No linked repositories are configured for this context.
   {% endif %}
   ```

   Context: `project_name` (string or empty) and `linked_repo_entries` (pre-rendered `- \`name\`:
   description`bullet list). The`format_generated_memory_markdown`pass and`apply_memory_frontmatter` wrapping stay in
   Python; default render must be byte-identical to today for every variant (project root, home root, chezmoi root, no
   linked repos).

2. **Validation**: template syntax/placeholder failures and — critically — a rendered body that violates
   `validate_short_memory_structure` (someone adds a second H1 or an H4) become memory-init blockers naming the resolved
   template file, and the root's init fails before any file is written. Because `memory/sase.md` feeds directly into
   AGENTS.md Tier 1 inlining, validating at the sase.md template keeps the error message pointed at the real culprit
   instead of a confusing downstream AGENTS round-trip mismatch.
3. **Overrides**: project `sase.yml` key `memory_sase_template` (root-relative), user config-dir file
   `memory-sase.template.md` (honored for home and chezmoi source roots), packaged default — plus matching
   `default_config.yml` and `sase.schema.json` entries. The directory-prefixed filename avoids the ambiguity a bare
   `sase.template.md` / `README.template.md` would create.

### Phase 3 — `memory/README.md` template

1. Create packaged `src/sase/main/init_memory/templates/memory-README.template.md` owning all one-off prose (intro,
   directory-map image reference, How Memory Files Are Used, Frontmatter Schema, Linking, Commands) with placeholders
   for the data-driven parts: `memory_notes` (pre-rendered
   `### \`path\``blocks; the "No memory notes exist yet" empty-state prose moves into an`{% if %}` branch of the template) and scalar statistics values (`total_notes`, `short_notes`, `long_notes`, `total_lines`, `total_tokens`)
   so even the statistics bullet labels live in the template. Per-note block rendering stays in Python (repeated,
   data-driven shape).
2. **Validation**: syntax/placeholder checks only — nothing parses the README back, and drift stays the existing
   byte-compare. Failures block the root's init the same way as Phase 2.
3. **Overrides**: `memory_readme_template` key / `memory-README.template.md` user file / packaged default.
4. **Docs**: update the Commands section prose (now in the packaged template) to document the two new keys alongside the
   existing `amd_agents_template` bullet. This is the only intentional content change to any default output in this
   plan; every repo with a checked-in `memory/README.md` will report drift until regenerated (see Rollout).

### Phase 4 — SDD README extraction

Move the five static documents out of `src/sase/sdd/_init_files.py` into packaged markdown files (e.g.
`src/sase/sdd/templates/README.md`, `plans-README.md`, `research-README.md`, `companion-plans-README.md`,
`companion-research-README.md`), loaded via `importlib.resources` at plan-build time. They contain no placeholders, so
they are read verbatim (no Jinja pass) and remain byte-identical; existing `SddExpectedTextFile` create/refresh
semantics are untouched. A read failure surfaces through the existing sdd-init error path. No config override keys — the
packaged file itself is the single editable source, and the shared resolver makes keys a trivial follow-up if ever
wanted.

### Phase 5 — Generated skill `SKILL.md` frame template

1. Create packaged `src/sase/xprompts/skills/SKILL.frame.template.md` owning the output frame:

   ````markdown
   {{ frontmatter }}

   {% if log_skill_use %}Before doing anything else, run this command to record that you are using this skill:

   ```bash
   sase skill use {{ skill_name }} --reason "<one-line reason for using this skill>"
   ```
   ````

   {% endif %}{{ body }}

   ```

   The YAML frontmatter block (name/description with the existing multiline and >120-column wrapping rules) remains
   Python-built and is passed in pre-rendered — that logic is serialization, not prose. The audit-directive prose moves
   into the template; `_build_output` becomes a thin context-builder over the shared loader. Prettier formatting and
   underscore unescaping stay unchanged, and default output stays byte-identical so already-deployed chezmoi skill
   files show no churn.
   ```

2. **Validation**: syntax/placeholder checks via the shared loader, plus a post-render guard that the output still
   starts with a parseable `---` frontmatter block containing `name` and `description` (providers refuse to load skills
   otherwise). Failures abort `sase skill init` with an error naming the template before any target is written. No
   override keys — skill sources (and now the frame) are edited in-repo and redeployed with the documented
   `sase skill init --force` + `chezmoi apply` flow.

## Testing

- **Golden guards (byte-identical)**: default-template renders equal the pre-change bytes for `memory/sase.md`
  (project/home/chezmoi roots, with and without linked repos), all five SDD READMEs, and representative `SKILL.md`
  outputs (short, long-wrapped, and multiline descriptions; `log_skill_use` on and off). Existing suites
  (`tests/main/test_init_memory_handler.py`, `test_init_memory_plan.py`, sdd init tests, `init_skills` tests) should
  pass unmodified — any needed assertion change outside the README Commands prose is a red flag.
- **README delta**: assertion updates confined to the documented Commands-section additions.
- **Override precedence**: for both new memory keys, project `sase.yml` beats user config-dir file beats packaged
  default; chezmoi source-root resolution mirrors the existing `amd_agents_template` tests in
  `tests/main/test_init_memory_agents_templates.py`.
- **Blockers**: missing placeholder, unknown placeholder, and Jinja syntax error for each new template; a sase.md
  template producing a second H1 or an H4 blocks with the short-note structure message and leaves every file unwritten;
  a broken skill frame template aborts `sase skill init` without writing targets.
- **Round-trip**: a customized (valid) `memory-sase.template.md` still inlines into AGENTS.md, passes
  `parse_amd_agents_document`, and shows up correctly in `sase memory list`.
- **Packaging**: the built wheel contains every new template/markdown asset (mirroring the AGENTS-template packaging
  test).
- **Loader refactor**: AGENTS template behavior and error messages stay byte-identical after delegating to the shared
  module.

## Boundaries and Non-Goals

- **Frontmatter-only frames stay in Python.** SDD plan/prompt snapshots, saved xprompts, approved memory notes, and
  long-note frontmatter refreshes wrap user/agent-authored bodies in structural YAML metadata; sase composes no prose
  there, so "a template of their contents" does not meaningfully apply.
- **Machine-parsed and ephemeral markdown is out of scope.** Chat transcripts are an archive format whose headings are
  parsed by chat tooling (templating invites breaking parsers with no editing use case); launch previews, per-run
  workflow logs, prompt/reply snapshots, and commit-finalizer responses are ephemeral run artifacts.
- **No structured-list Jinja loops.** Repeated shapes (Tier sections, README note blocks, linked-repo bullets) are
  passed as pre-rendered strings to keep round-trip parsing and drift semantics safe; richer context can be added later
  without breaking existing templates.
- **No new CLI subcommands or options** — configuration is file/`sase.yml`-key based only.
- **No Rust core changes.** All of this is Python-only CLI file generation with no TUI-parity requirement, matching the
  boundary call made for the AGENTS template work.
- **Provider shims** remain byte-for-byte copies of `AGENTS.md`; per-provider skill output paths and provider handling
  remain uniform across runtimes.

## Rollout Notes

- The README Commands prose update makes every checked-in `memory/README.md` stale; `sase memory init --check` /
  `sase validate` will report that drift until the user regenerates protected memory files (generated memory files must
  not be rewritten by agents without explicit permission).
- Skill outputs are byte-identical, so no `chezmoi apply` churn is expected; the standard `sase skill init --force` +
  `chezmoi apply` flow applies only once someone actually edits the new frame template.
