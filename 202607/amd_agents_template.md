---
create_time: 2026-07-12 15:41:34
status: done
prompt: 202607/prompts/amd_agents_template.md
tier: tale
---
# Plan: Extract the `sase memory init` AGENTS.md Skeleton into a Human-Editable Markdown Template

## Problem / Product Context

Users want a single markdown file they can edit to change the base ("skeleton") contents that `sase memory init` writes
into agent instruction files (`AGENTS.md` plus its byte-for-byte provider copies `CLAUDE.md`, `GEMINI.md`, `QWEN.md`,
`OPENCODE.md`). Today no such template exists — the skeleton is hardcoded as Python string lists, so changing prose like
the "IMPORTANT: You should not modify..." preamble or the Tier 1/Tier 2 intro text requires editing Python code. That is
a poor editing experience for humans and hides agent-facing prose (which deserves the same review attention as any other
prompt text) inside rendering logic.

## Current State

- **Managed skeleton** — `src/sase/amd/_memory.py::_render_managed_agents` hardcodes the full document frame:
  `# {title}` H1, the IMPORTANT preamble, `## Tier 1 (short-term) Memory` heading + intro line, and
  `## Tier 2 (long-term) Memory` heading + intro lines. The dynamic parts (inlined short-note sections, long-note
  reference entries) are rendered by `sase.amd.inline_memory.inline_memory_section` and
  `sase.memory.notes.render_memory_note_references` and spliced between the hardcoded lines.
- **Minimal fallback** — `src/sase/main/init_memory/root_rendering.py::_minimal_agents_content` hardcodes a second,
  smaller shape (`# Agent Instructions` + one inlined `memory/sase.md` section), used only when no AMD H1 title resolves
  (created only if `AGENTS.md` is missing).
- **Provider shims** — byte-for-byte copies of the root `AGENTS.md` (`src/sase/amd/constants.py::PROVIDER_SHIM_FILES`),
  so a single AGENTS.md template automatically covers all agent files.
- **All roots share one path** — project roots, `~`, and the chezmoi source root all render via `plan_amd_memory_sync`
  (`src/sase/main/init_memory/root_planning.py`), so one template serves every root kind.
- **Round-trip parsing constraint** — `src/sase/amd/_agents_doc.py::parse_amd_agents_document` locates memory sections
  by the _exact_ H2 headings `## Tier 1 (short-term) Memory` and `## Tier 2 (long-term) Memory`. These headings are
  structural anchors that template edits must not break (they feed `sase memory list`, description harvesting, and drift
  checks).
- **Existing knob** — only the H1 title is configurable today, via `amd_h1_title` in project `sase.yml` or the user
  config dir (including the chezmoi source), resolved by `src/sase/amd/_config.py::resolve_amd_h1_title`.
- **Prior art** — Jinja2 is already a dependency with shared helpers (`src/sase/xprompt/_jinja.py`, StrictUndefined
  conventions), and packaged non-Python assets already ship via `importlib.resources` (e.g.
  `sase.memory/assets/memory-directory-map.png`).

## Proposed Design

### Phase 1 — Packaged default template

1. Create `src/sase/amd/templates/AGENTS.template.md` containing the managed skeleton verbatim, with Jinja2 placeholders
   for the dynamic parts:

   ```markdown
   # {{ title }}

   IMPORTANT: You should not modify any of these memory files without approval from the user.

   ## Tier 1 (short-term) Memory

   The following memories contains core (always loaded) context:

   {{ tier1_sections }}

   ## Tier 2 (long-term) Memory

   The below files contain detailed reference material. When working in their domain, you MUST use your
   `/sase_memory_read` skill to review their contents. Do not read canonical memory files directly.

   {{ tier2_entries }}
   ```

   - `title`: the resolved AMD H1 title (unchanged resolution rules).
   - `tier1_sections`: the pre-rendered, numbered inlined short-note sections (existing `inline_memory_section` output,
     joined). Section _bodies_ stay Python-rendered — the template owns only the frame around them.
   - `tier2_entries`: the pre-rendered long-note reference entries (existing `render_memory_note_references` output,
     joined).

2. Create a sibling `src/sase/amd/templates/AGENTS.minimal.template.md` for the minimal fallback shape
   (`# {{ title }}` + `{{ tier1_sections }}`), replacing the hardcoded `_minimal_agents_content` frame. This keeps _all_
   agent-file base prose in the templates directory rather than leaving one variant stranded in Python.

3. Add a small loader/renderer module (e.g. `src/sase/amd/_template.py`) that:
   - loads templates via `importlib.resources.files("sase.amd").joinpath("templates", ...)` (mirroring the existing
     PNG-asset pattern; hatchling already packages non-Python files under the package dir, so no packaging changes are
     expected);
   - renders with Jinja2 `StrictUndefined` following the `sase.xprompt._jinja` conventions;
   - keeps the existing `format_generated_memory_markdown` post-processing pass unchanged.

4. Rewire `_render_managed_agents` and `_minimal_agents_content` to build the context values and delegate the document
   frame to the templates. The default template output must be **byte-identical** to today's generated content so
   `sase memory init --check` reports no drift anywhere after this change.

### Phase 2 — Validation (protect the structural anchors)

Because humans will edit the template, `sase memory init` must fail loudly (as `AmdMemorySyncPlan.blockers`, consistent
with existing short-note structure blockers) rather than silently generating an unparseable document:

- Template render errors (missing/unknown placeholder under StrictUndefined, Jinja syntax error) become blockers naming
  the template file and the error.
- After rendering, run `parse_amd_agents_document` on the result and verify: both tier sections are found, the parsed
  short-memory paths match the inlined notes, and the parsed long-memory entries match the expected note paths. A
  mismatch (e.g. someone renamed a tier heading) becomes a blocker explaining which structural anchor is missing.
- The `{{ title }}`, `{{ tier1_sections }}`, and `{{ tier2_entries }}` placeholders are required in the managed
  template; validate their presence before rendering so error messages are actionable ("template must contain
  {{ tier1_sections }}").

### Phase 3 — User/project overrides (edit without forking the repo)

Follow the established `amd_h1_title` precedence so global edits live where humans already manage SASE config:

1. **Project override**: optional `amd_agents_template: <root-relative path>` key in the project `sase.yml`, pointing at
   a template checked into the repo.
2. **User override**: `AGENTS.template.md` in the user config dir (`~/.config/sase/`), with the chezmoi source-root
   equivalent (`dot_config/sase/AGENTS.template.md`) honored when initializing the chezmoi home root — same
   dual-location handling as `_user_config_dir_for_home_amd_root`.
3. **Fallback**: the packaged default template.

The same resolution applies to the minimal template (project key `amd_agents_minimal_template`, user file
`AGENTS.minimal.template.md`), though overriding it is expected to be rare.

Drift semantics come for free: expected content is computed from the resolved template at runtime, so editing a template
makes previously generated files stale and the next `sase memory init` rewrites them (and `--check` reports the drift) —
exactly the desired behavior.

## Testing

- **Golden guard**: default-template render is byte-identical to the pre-change output for a representative root
  (managed with short+long notes, managed with no long notes, minimal fallback). Existing suites in
  `tests/main/test_init_memory_managed_agents.py` and `test_init_memory_agent_docs.py` largely cover this already and
  should pass unmodified — treat any needed assertion change as a red flag.
- **Validation blockers**: missing placeholder, Jinja syntax error, renamed tier heading, and removed tier section each
  produce a clear blocker and leave files unwritten.
- **Override precedence**: project `sase.yml` key beats user config-dir file beats packaged default; chezmoi source-root
  resolution mirrors the `amd_h1_title` tests (`test_init_memory_chezmoi.py`).
- **Round-trip**: a customized (but valid) template still yields a document that `parse_amd_agents_document` and
  `sase memory list` interpret correctly.

## Boundaries and Non-Goals

- **Rust core boundary**: the entire `sase memory init` / AMD rendering stack is Python-only CLI behavior in this repo
  today with no TUI-parity requirement, so no `sase-core` changes are planned. If memory rendering ever migrates to the
  Rust core, the templates migrate with it.
- **Out of scope**: templating the generated `memory/sase.md` body and `memory/README.md` (memory files, not agent files
  — natural follow-up using the same loader if wanted); changing the tier heading contract in `_agents_doc.py`; exposing
  structured note lists to the template for Jinja-loop restructuring (the pre-rendered-string context keeps round-trip
  parsing safe; richer context can be added later without breaking existing templates).
- **No new CLI subcommands or options** — configuration is file/`sase.yml`-key based only.
- **Docs**: update the generated `memory/README.md` "Commands" prose (in `root_rendering.py::_render_memory_readme`) to
  mention that `sase memory init` renders agent files from `AGENTS.template.md` and where overrides live.
