---
tier: epic
title: sase xprompt show
goal: '`sase xprompt show <name>` renders any single xprompt or workflow definition
  — its declared properties, inputs, local helpers, body, and provenance — with the
  same xprompt syntax highlighting the ACE prompt input bar uses, plus byte-faithful
  `--format raw` and stable `--format json` modes.

  '
phases:
- id: highlight
  title: Shared xprompt highlight core (roles, flattened spans, palette)
  depends_on: []
  size: medium
  description: 'highlight: add the frontend-agnostic highlight role model, an overlap-flattening
    span merger over the existing xprompt/jinja/alt/placeholder/artifact-ref scanners,
    and a flexoki-derived palette exposing Rich and ANSI styles; move the shared argument-color
    blend out of the TUI mixin.

    '
- id: resolve
  title: Definition resolution, provenance, and the JSON record
  depends_on: []
  size: medium
  description: 'resolve: add name normalization and lookup with suggestions, source
    classification and provenance (path, line, hosted URL, editability), byte-faithful
    raw-definition extraction for every source bucket, the reference scan, and the
    versioned show record with its JSON projection.

    '
- id: render
  title: Rich rendering of the show layout
  depends_on:
  - highlight
  - resolve
  size: medium
  description: 'render: build the Rich renderables for the header, properties, inputs,
    local xprompts, body with a line-number gutter and role styling, workflow steps,
    references, and hints; add the shared CLI chrome palette.

    '
- id: cli
  title: CLI wiring, help, and documentation
  depends_on:
  - render
  size: small
  description: 'cli: register the `show` subparser with its flags and examples, dispatch
    it from the xprompt handler, document the command in the three docs surfaces,
    and retire the epic symvision whitelist entries.'
proposed_by: bbugyi200.athena.s3
create_time: 2026-08-02 11:49:10
status: wip
bead_id: sase-eb
---

- **PROMPT:** [prompts/202608/xprompt_show.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/xprompt_show.md)
- **BEAD:** [sase-eb](https://github.com/sase-org/sase--beads/blob/main/pages/sase-eb/README.md)

# Plan: `sase xprompt show`

## Goal

Give SASE a first-class, human-facing way to read one xprompt definition. `sase xprompt list` already emits the whole
catalog as JSON and `sase xprompt catalog` renders a PDF, but there is no way to ask "what exactly is `#sase/reads`,
what properties does it declare, and what does its body look like?" without finding the file by hand. `show` answers
that in one command, and it renders the body with real xprompt syntax highlighting rather than generic markdown —
invocations, directives, segment separators, skill references, Jinja, placeholders, and artifact references all get the
same colors the ACE prompt input bar gives them.

Three qualities drive every decision below:

- **Intuitive** — one required positional, three flags, and a name field that accepts whatever the user has in their
  clipboard (`sase/reads`, `#sase/reads`, `#!sync`, `/sase_plan`, even `#pr:my_change`).
- **Reliable** — a definition that fails to load never crashes the command; every uncertain fact is either resolved or
  honestly reported as unknown; `--format raw` is byte-exact; `--format json` is versioned and stable.
- **Beautiful** — colors are derived from the exact same theme the TUI pins, so the CLI and the TUI agree; the layout is
  dense, aligned, and section-ruled; the body carries a line-number gutter keyed to the real file.

## Context an implementing agent needs

Everything below already exists in this repo. None of it needs to change unless the plan says so.

**Catalog and lookup**

- `sase.xprompt.loader.get_all_xprompts(project)` → `dict[str, XPrompt]`; `get_all_workflows(project)` →
  `dict[str, Workflow]`; `get_all_prompts(project)` returns both as `Workflow` objects with workflows winning on
  collision; `get_xprompt_or_workflow(name, project)` checks xprompts first, then workflows.
- `sase.xprompt.loader.detect_project()` auto-detects the project from cwd. Names may be namespaced (`sase/reads`,
  `bd/land_epic`).
- `sase.xprompt.load_issues.collect_xprompt_load_issues()` is a context manager yielding a list of per-source load
  failures; `sase xprompt list` already prints them to stderr.
- `sase.xprompt.reference_display` provides `workflow_kind_value`, `workflow_reference_prefix`,
  `workflow_reference_insertion` (`#name` vs `#!name`).
- `sase.xprompt.workflow_models.WorkflowKind` is `SIMPLE_XPROMPT | EMBEDDABLE_WORKFLOW | STANDALONE_WORKFLOW`, returned
  by `Workflow.prompt_kind()`.
- `sase.xprompt.models.XPrompt` carries
  `name, content, inputs, source_path, tags, snippet, description, skill, log_skill_use, local_xprompts`. `InputArg`
  carries `name, type, default, is_step_input, description, repeatable`; `UNSET` means required and `None` means an
  explicit null default.

**Classification and provenance**

- `sase.xprompt._catalog_sources` exposes public functions `classify(xp, project) -> CatalogEntry` (bucket is one of
  `built-in | project | config | plugin`), `source_path_display(entry)`, `definition_path(entry)`, and
  `package_xprompt_dirs()`. `sase.xprompt.catalog` already imports them under private aliases, so importing them
  directly from `_catalog_sources` is established practice.
- `sase.xprompt.xprompt_sources` resolves a `source_path` identifier to a real file (`_definition_file_for_source`) and
  finds a unique YAML definition-key line (`_resolve_definition_line`). Both are module-private; the `resolve` phase
  needs public equivalents (see below).
- `sase.xprompt_links` provides `XpromptTargetResolver` and `load_xprompt_source_records` for turning a provenance
  record into a hosted blob URL with a line anchor. It returns `None` rather than guessing.
- `sase.xprompt.config_yaml._parse_entry_blocks` / `_find_xprompts_section` already compute the exact line span of one
  entry inside a config file's `xprompts:` section.

**Scanners the highlighting is built from (all frontend-agnostic, all pure)**

| Module                                            | Entry point                                                            | Kinds                                                                              |
| ------------------------------------------------- | ---------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| `sase.xprompt.xprompt_inspect`                    | `tokenize(text, known_skills=...)`                                     | `invocation`, `invocation_arg`, `directive`, `directive_arg`, `separator`, `skill` |
| `sase.xprompt.jinja_inspect`                      | `tokenize(text)`                                                       | `delimiter`, `statement`, `variable`, `comment`, `filter`, `keyword`, `operator`   |
| `sase.xprompt.alt_inspect`                        | `tokenize(text)`                                                       | `delimiter`, `separator`, `branch_name`, `error`                                   |
| `sase.xprompt.placeholder_completion`             | `placeholder_spans(text)`                                              | placeholder ranges (Rust-backed, line/character positions)                         |
| `sase.artifact_refs`                              | `scan_artifact_refs(text)`                                             | artifact reference candidates                                                      |
| `sase.xprompt._literal_zones` / `._fenced_blocks` | `literal_zone_ranges`, `inline_literal_ranges`, `fenced_block_details` | code zones                                                                         |
| `sase.xprompt.used_xprompts`                      | `scan_xprompt_references(raw_prompt)`                                  | resolved/unresolved `#name` references with the parsed `item`                      |

**The TUI highlighting this must match** — `src/sase/ace/tui/widgets/_xprompt_syntax_highlight.py` maps span kinds to
`rich.style.Style` values pulled from the live Textual app theme, using
`derive_argument_color(base, foreground=, background=)` to produce a lighter sibling color for argument spans.
`src/sase/ace/tui/widgets/_jinja_highlight.py`, `_alt_syntax_highlight.py`, `_placeholder_highlight.py`, and
`_artifact_ref_highlight.py` do the same for their kinds. The ACE app pins its theme in
`src/sase/ace/tui/actions/_state_init.py` with `self.theme = "flexoki"`, so the concrete palette is
`textual.theme.BUILTIN_THEMES["flexoki"]`.

**CLI conventions** (from the `cli_rules` long-term memory, already verified against the code)

- `-h/--help` output must be excellent; subcommands and options sorted alphabetically (`_sort_subcommand_help` in
  `src/sase/main/parser.py` sorts the help rows centrally, so registration order does not matter for rendering).
- Every public long option needs a short alias.
- Prefer colored output where color improves readability.
- `sase xprompt` already defaults to `sase xprompt list` via `_default_list_subcommands()`; that must keep working.
- `sase bead show` is the closest precedent for a colored CLI detail view: `--color {auto,always,never}` gated through
  `sase.bead.cli_dep_render.resolve_color` (honors `NO_COLOR` and `stdout.isatty()`), additive ANSI from
  `sase.ansi_style.ansi_sgr` / `apply_ansi`, `_SECTION_COLOR = "#5F87FF"`, `_PATH_COLOR = "#87AFFF"`.

**Repo rules that bind every phase**

- Run `just install` before anything else in a fresh workspace, and `just check` before reporting done.
- Never edit `CHANGELOG.md` — it is generated by release-please from conventional commit subjects.
- Never edit `sase/memory/*.md`, `AGENTS.md`, or the provider shims. This plan does not authorize memory edits.
- `just _lint-toobig` warns at 700 lines and fails at 1000 for any file in `src/` or `tests/`. The module split below
  exists to stay well under that.
- `just _lint-symvision` fails on public symbols with no non-test consumer. Phases 1–3 create public symbols whose first
  real consumer lands in a later phase, so each of those phases adds `--epic-symbol '<bead>(<Symbol>)'` entries to the
  symvision invocation in the `Justfile` (follow the existing `sase-e6(...)` entries as the format), and the `cli` phase
  removes every entry this epic added once the consumers exist. Tests never count as consumers.

**Rust core boundary** — no `sase-core` change is required. `xprompt_inspect.py` documents the standing arrangement: the
canonical grammar lives in Rust, and these Python scanners are deliberate presentation-only lexical mirrors that both
editable and read-only frontends consume so they render exactly what launch processing recognizes. This epic adds a
second read-only frontend on top of those same mirrors and a presentation palette; neither is backend domain logic. If a
scanner turns out to disagree with launch behavior, that is a Rust-core bug to file, not something to patch around here.

## Command surface

```
sase xprompt show [-c {auto,always,never}] [-f {full,json,raw}] [-p PROJECT] NAME
```

| Flag            | Values                  | Default | Behavior                                                                     |
| --------------- | ----------------------- | ------- | ---------------------------------------------------------------------------- |
| `NAME`          | string                  | —       | XPrompt or workflow name; reference markers and argument suffixes tolerated. |
| `-c, --color`   | `auto`,`always`,`never` | `auto`  | ANSI gate. `auto` = `stdout.isatty()` and `NO_COLOR` unset.                  |
| `-f, --format`  | `full`,`json`,`raw`     | `full`  | Rendered view, machine record, or exact definition bytes.                    |
| `-p, --project` | string                  | auto    | Resolve within a specific project namespace instead of the detected one.     |

Deliberate omissions, so the surface stays small: no `--style` tier (unlike `sase bead show`, where prose highlighting
is an opt-out; here the highlighting _is_ the feature, and `--color` is the single honest gate), no pager, no `--all`,
no argument binding (`sase xprompt explain` already renders a workflow with arguments bound).

**Name normalization** — accept, in order: strip one leading `#!`, `#`, or `/` marker; strip an argument suffix
(`(...)`, `:arg`, or a trailing `+`) using the same lexical rules the parser uses, emitting a dim note on stderr that
arguments were ignored; then look up the remainder. A `/name` marker restricts nothing — it is just a marker a user may
have copied from a skill invocation.

**Lookup order** — `get_all_workflows(project)` first, then `get_all_xprompts(project)`, matching the precedence
`get_all_prompts()` already establishes (workflows win on collision). If both exist under one name, render the workflow
and add a warning row naming the shadowed xprompt and its source.

**Miss behavior** — exit 1, print `unknown xprompt: <name>` to stderr, and follow it with up to five suggestions ranked
by `difflib.get_close_matches` over the catalog keys, rendered as their insertion text (`#foo`, `#!bar`) so they can be
copied straight back into the command.

**Exit codes** — `0` rendered; `1` unknown name; `2` `--format raw` requested but the definition source could not be
read. `full` and `json` never fail on an unreadable source: they render everything they do know and report
`raw_available: false` plus a warning line.

## Phase `highlight`: shared xprompt highlight core

Two new modules, both pure (no Rich, no Textual, no ANSI in the span module; no Textual widget code anywhere).

### `src/sase/xprompt/highlight.py`

Defines the single role vocabulary every SASE frontend can style against, and a merger that turns the overlapping
per-scanner span lists into one ordered, non-overlapping partition. The TUI does not need flattening — it layers
overlapping spans onto a highlight map and the last writer wins per cell — but any linear renderer (CLI, HTML, PDF)
does, and that flattening is the genuinely new logic here.

```python
XPromptHighlightRole = Literal[
    "xprompt.invocation", "xprompt.invocation_arg",
    "xprompt.directive", "xprompt.directive_arg",
    "xprompt.separator", "xprompt.skill",
    "jinja.delimiter", "jinja.statement", "jinja.variable",
    "jinja.comment", "jinja.filter", "jinja.keyword", "jinja.operator",
    "alt.delimiter", "alt.separator", "alt.branch_name", "alt.error",
    "placeholder", "artifact_ref",
    "code.fence", "code.inline",
]


@dataclass(frozen=True, slots=True)
class HighlightSpan:
    start: int
    end: int
    role: XPromptHighlightRole


def highlight_spans(
    text: str,
    *,
    known_skills: frozenset[str] = frozenset(),
    include_artifact_refs: bool = True,
) -> list[HighlightSpan]:
    """Return a flat, ordered, non-overlapping role partition of *text*."""
```

Requirements:

- Call each scanner exactly once, map its kinds onto the role vocabulary, and concatenate.
- Flatten with an explicit, table-driven precedence. Higher precedence wins the overlapping region; the loser is split
  around it and may contribute zero, one, or two remaining fragments. Precedence, highest first: `code.fence` /
  `code.inline` → `xprompt.*` → `alt.*` → `jinja.*` → `placeholder` → `artifact_ref`. Rationale: the xprompt scanners
  already exclude protected literal zones themselves, so a code role never actually competes with an xprompt role in
  practice — but the ordering makes the invariant explicit and makes a future scanner change fail safe rather than emit
  garbage. Jinja below xprompt means `#foo({{ bar }})` keeps the invocation head colored while the Jinja expression
  inside the argument still reads as Jinja.
- Ties (identical start and end from two scanners) resolve by the same table; never emit two spans covering one offset.
- Emit nothing for zero-width spans; clamp every span to `[0, len(text)]`; assert ordering in a test, not at runtime.
- Guard rails matching the TUI's: return `[]` above 80 000 UTF-8 bytes or 1 200 lines. Reuse the numbers by defining
  `MAX_HIGHLIGHT_BYTES` / `MAX_HIGHLIGHT_LINES` here and having `_jinja_highlight.py` import them, so one constant pair
  governs both frontends. Keep the existing `_MAX_OVERLAY_BYTES` / `_MAX_OVERLAY_LINES` names as aliases in
  `_jinja_highlight.py` so every mixin that imports them from there — `_alt_syntax_highlight`,
  `_artifact_ref_highlight`, `_bullet_highlight`, `_codeblock_syntax_highlight`, `_placeholder_highlight`,
  `_todo_highlight`, and `_xprompt_syntax_highlight` — is untouched.
- Every scanner call is individually wrapped: a scanner that raises contributes no spans and does not abort the merge.
  `placeholder_spans` needs the Rust binding and its line/character ranges must be converted to character offsets — if
  the binding is unavailable, drop placeholders rather than failing.
- `include_artifact_refs=False` exists for callers that want a body-only pass without the artifact scanner.

### `src/sase/xprompt/highlight_theme.py`

Maps roles to concrete styles, derived from the exact theme ACE pins.

```python
ACE_THEME_NAME = "flexoki"


@dataclass(frozen=True, slots=True)
class HighlightStyle:
    color: str | None
    bold: bool = False
    dim: bool = False
    italic: bool = False
    underline: bool = False

    @property
    def rich_style(self) -> str: ...   # e.g. "bold #66800B"

    @property
    def ansi_sgr(self) -> str: ...     # via sase.ansi_style.ansi_sgr


def derive_argument_color(base, *, foreground, background) -> str | None: ...
@functools.cache
def highlight_theme() -> Mapping[XPromptHighlightRole, HighlightStyle]: ...
```

Requirements:

- `derive_argument_color` moves here verbatim from `_xprompt_syntax_highlight.py`; that module and
  `_artifact_ref_highlight.py` import it from the new home. Pure code move, no behavior change.
- `highlight_theme()` resolves `textual.theme.BUILTIN_THEMES[ACE_THEME_NAME]` behind a lazy, function-local import and
  builds every role's style from that theme's `success`, `warning`, `secondary`, `accent`, `foreground`, `background`,
  and `error` slots, mirroring the attribute choices the TUI mixins already make. Memoize with `functools.cache`;
  measured cost of `import textual.theme` is ~0.19 s, which is acceptable for a command that is never on a completion
  hot path, and no other CLI path pays it.
- `src/sase/ace/tui/actions/_state_init.py` sets `self.theme = ACE_THEME_NAME` instead of the string literal, so the two
  can never drift.
- Code roles (`code.fence`, `code.inline`) map to a foreground blend rather than a background band: terminal background
  manipulation across a folded line is fragile, and the CLI renders fenced bodies through a real lexer anyway.

Tests (`tests/xprompt/test_highlight.py`, `tests/xprompt/test_highlight_theme.py`):

- Every `XPromptHighlightRole` member has an entry in `highlight_theme()`, and no extras.
- Golden hex assertion for the full role→style table, so a Textual flexoki change fails CI loudly instead of silently
  shifting the palette.
- Flattening: overlapping invocation + Jinja; directive with parenthesized args containing a placeholder; an alt block
  containing an invocation; a `#name` inside a fenced block and inside backticks producing no xprompt role; identical
  spans from two scanners; adjacent spans not merged into one; output strictly ordered and non-overlapping (property
  check over the repo's own bundled xprompts is a good corpus).
- Byte/line guards return `[]`.
- A scanner that raises (monkeypatched) degrades to the remaining roles.
- The moved `derive_argument_color` still produces the values the TUI test suite expects.

## Phase `resolve`: definition resolution, provenance, and the JSON record

Two new modules plus three small public helpers added to existing modules.

### New public helpers in existing modules

- `sase.xprompt.segment_separators.xprompt_segment_count(xp) -> int` — number of top-level segments (separator count
  outside fenced blocks, plus one). `xprompt_has_segment_separators` becomes `xprompt_segment_count(xp) > 1`.
- `sase.xprompt.xprompt_sources.definition_file_for_source(source_id) -> Path | None` and
  `definition_line_for(path, name) -> int | None` — public wrappers over the existing `_definition_file_for_source` and
  `_resolve_definition_line`, with the private names kept as in-module aliases so nothing else changes.
- `sase.xprompt.config_yaml.config_entry_line_span(path, name) -> tuple[int, int] | None` — the 1-based inclusive line
  span of one entry inside a config file's `xprompts:` section, built on the existing `_find_xprompts_section` and
  `_parse_entry_blocks`.

### `src/sase/xprompt/cli_show_model.py`

The versioned record and its JSON projection. Frozen dataclasses, no I/O.

```python
SHOW_SCHEMA_VERSION = 1

@dataclass(frozen=True) class ShowInput:       name, type, required, default_display, description, repeatable, position
@dataclass(frozen=True) class ShowLocalXPrompt: name, description, input_signature, line_count
@dataclass(frozen=True) class ShowStep:         index, name, type, label, hidden, condition, output_schema
@dataclass(frozen=True) class ShowReference:    raw_ref, name, kind, resolved, source_display
@dataclass(frozen=True) class ShowProvenance:   source_id, source_bucket, source_display, definition_path,
                                                definition_line, hosted_url, editable
@dataclass(frozen=True) class XPromptShowRecord:
    name, reference, prefix, kind, is_skill, is_swarm, segment_count, description, project,
    provenance, tags, skill, snippet, log_skill_use, input_signature, inputs, local_xprompts,
    steps, body, body_first_line, raw, warnings, references

    def to_json_dict(self) -> dict[str, Any]: ...
```

`to_json_dict()` always emits every key with an explicit `null`/`[]` for unset values plus
`"schema_version": SHOW_SCHEMA_VERSION`, so consumers can rely on the shape. The pretty renderer, by contrast, omits
unset optional rows. `input_signature` reuses the existing `sase.xprompt._catalog_format.format_inputs` so it matches
the PDF catalog and the mobile projection exactly.

### `src/sase/xprompt/cli_show_resolve.py`

```python
@dataclass(frozen=True) class ShowLookupMiss:  name, suggestions      # insertion strings
def normalize_show_name(raw) -> tuple[str, bool]: ...                 # (name, arguments_were_stripped)
def resolve_show_record(raw_name, *, project=None) -> XPromptShowRecord | ShowLookupMiss: ...
```

Behavior:

1. Normalize the name as described in the command surface.
2. Load the catalog inside `collect_xprompt_load_issues()`; every collected issue becomes a `warnings` entry
   (`skipped: <source>: <error>`), so a broken neighbor is visible but never fatal.
3. Look up workflow first, then xprompt. On a miss, build `ShowLookupMiss` with
   `difflib.get_close_matches(name, catalog_keys, n=5, cutoff=0.5)` rendered through `workflow_reference_insertion`.
4. Classify with `_catalog_sources.classify(...)`, then fill `ShowProvenance`:
   - `source_display` from `source_path_display`, `definition_path` from `definition_path` (absolute, only when the file
     actually exists), `editable` = the definition path exists and is writable.
   - `definition_line` from `definition_line_for(...)` for YAML-defined entries; `None` for a whole-file `.md`.
   - `hosted_url` best-effort via `sase.xprompt_links.XpromptTargetResolver` fed a record built from the resolved
     definition path; `None` whenever the resolver declines. Never guess a URL, and never let a git or network hiccup
     fail the command — wrap the whole resolution and downgrade to `None` with a warning.
5. Extract `raw`, the exact definition source, without re-serializing anything:
   - `.md` / `.yml` / `.yaml` file source → the whole file's text, byte-for-byte.
   - config-backed source (`config`, `default_config`, `local_config`, `config_overlay:*`, `project_local_config:*`,
     `plugin_config:*`) → the exact line span from `config_entry_line_span`, unmodified.
   - anything else, or any read error → `raw = None` and a warning; `full`/`json` continue, `raw` format exits 2.
6. `body` is `XPrompt.content` (or `Workflow.get_prompt_part_content()` for prompt-bearing workflows). `body_first_line`
   is the 1-based line of the body's first line within the definition file when that is exactly determinable — for a
   `.md` file, the line after the closing frontmatter `---`; otherwise `None`, and the renderer falls back to numbering
   from 1.
7. `references` come from `scan_xprompt_references(body, extra_xprompts=xp.local_xprompts)`: `resolved` is
   `scanned.item is not None`, `kind` is the scanner's `workflow | part | swarm`, and `source_display` is the referenced
   entry's display path when resolved. Deduplicate by `raw_ref`, preserve document order.
8. `steps` is populated only for non-simple workflows, reusing the step-typing already written in
   `xprompt_browser_preview.create_workflow_preview` — extract that type/label derivation into a shared helper rather
   than writing a third copy (`sase.main.xprompt_handler._handle_list` has the second).

Tests (`tests/xprompt/test_cli_show_resolve.py`, using `tmp_path` xprompt dirs and monkeypatched loaders):

- Name normalization for `foo`, `#foo`, `#!foo`, `/foo`, `#foo(a, b)`, `#foo:bar`, `#foo+`, `#ns/foo`, and the
  arguments-stripped flag.
- Workflow-wins-over-xprompt precedence, and the shadowing warning.
- Suggestions on a near miss; empty suggestions on total nonsense.
- Provenance for each bucket: a project `.md` file, a `~/.config/sase/sase.yml` entry, a `config_overlay:` entry, a
  package built-in, and a plugin source; each asserting `source_display`, `definition_path`, and `definition_line`.
- Raw extraction is byte-identical to the file for `.md`, and to the exact entry span for a config entry (including a
  neighbor entry immediately after it, to prove the span does not over-capture).
- Unreadable source → `raw is None`, a warning, and no exception.
- Hosted-URL resolver raising → `hosted_url is None` plus a warning, command still succeeds.
- `references` marks an unresolved `#nope` as `resolved=False` and a local `#_helper` as resolved.
- `to_json_dict()` contains every documented key, has `schema_version == 1`, and round-trips through `json.dumps`.

## Phase `render`: Rich rendering of the show layout

### Shared chrome constants

New `src/sase/cli_show_palette.py` holding the two hexes that recur across SASE detail views:
`SECTION_COLOR = "#5F87FF"`, `PATH_COLOR = "#87AFFF"`. `sase/bead/cli_detail_style.py` imports them in place of its
local `_SECTION_COLOR` / `_PATH_COLOR` literals. This is a constant move only — `sase bead show` output must remain
byte-identical, and the existing bead detail tests are the proof. Do not restructure the bead palette.

### `src/sase/xprompt/cli_show_body.py`

Turns a body plus its spans into a `rich.text.Text`.

```python
def highlighted_body(text, *, known_skills=frozenset()) -> Text: ...
def body_block(text, *, first_line=None, known_skills=frozenset()) -> RenderableType: ...
```

- Build a single `Text` from the raw body, then `stylize(role_style, start, end)` for each flattened span. Never build
  Rich markup strings from user content — a body containing `[bold]` must render literally. This is the whole reason the
  renderer composes `Text` objects instead of interpolating into markup.
- `body_block` prefixes a right-aligned, dim line-number gutter (` 45 │`) using absolute file line numbers when
  `first_line` is known and 1-based body numbers otherwise, so the numbers line up with the `definition` path shown in
  the properties block.
- Long lines use `overflow="fold"` with `no_wrap=False`: fold at the character level rather than reflowing words, so
  indentation-sensitive prompt bodies survive a narrow terminal and nothing is ever truncated away.
- Fenced code blocks inside the body render through `rich.syntax.Syntax` with the info-string's lexer (falling back to
  `text`), layered under the xprompt spans — reuse the approach in `sase/bead/cli_detail_prose.py`, including its
  `Syntax.highlight()`-appends-a-newline crop fix. Do not reuse the TUI's tree-sitter injection path; that is
  Textual-only.

### `src/sase/xprompt/cli_show_render.py`

```python
def render_show(record, *, console) -> None: ...
```

Sections, in order, each preceded by a dim rule and a `SECTION_COLOR` label, each omitted entirely when it has no
content:

```
#sase/reads                                                    xprompt · swarm · 4 segments
Find new medium-to-long article recommendations with a multi-runtime research pass.
────────────────────────────────────────────────────────────────────────────────────────────
PROPERTIES
  reference    #sase/reads
  project      sase
  source       project · sase/xprompts/reads.md
  definition   /home/bryan/.../sase/xprompts/reads.md
  hosted       https://github.com/sase-org/sase/blob/<rev>/sase/xprompts/reads.md
  tags         research, reading
  skill        no
  snippet      no
────────────────────────────────────────────────────────────────────────────────────────────
INPUTS  #sase/reads(topic: text, reference_query?: text)
  topic             text  required  Reading request or topic to search for.
  reference_query   text  default   Obsidian Dataview query whose rows should be excluded.
                                    default: LIST WITHOUT ID title + " (" + url + ")" …
────────────────────────────────────────────────────────────────────────────────────────────
LOCAL XPROMPTS
  #_article_search_agent   14 lines   (no description)
────────────────────────────────────────────────────────────────────────────────────────────
DEFINITION  sase/xprompts/reads.md
  45 │ %id:reads.{@1}.agy
  46 │ %model:agy/gemini-3.6-flash-high
  47 │ %g:read
  48 │ #_article_search_agent
  49 │
  50 │ ---
────────────────────────────────────────────────────────────────────────────────────────────
REFERENCES
  #_article_search_agent   local helper   ✓
  #bob_query               skill          ✓
  #missing                 unknown        ✗
────────────────────────────────────────────────────────────────────────────────────────────
WARNINGS
  hosted URL unavailable: no hosted resolver for this repository

  sase xprompt expand '#sase/reads(...)'   preview the expansion
```

Requirements:

- Header: reference text in the `xprompt.invocation` style so the very first thing on screen already demonstrates the
  highlighting; kind chips right-aligned and dim. Chips: kind label, `skill` when applicable, `swarm · N segments` when
  `segment_count > 1`, `N steps` for workflows.
- Properties: two-column, label column dim and fixed-width, values plain except `definition`/`hosted` in `PATH_COLOR`.
  Only `reference`, `source`, and `definition` are always shown; every other row appears only when set. Missing but
  meaningful values render as a dim italic placeholder (`(unknown)`), never as a blank or a guess.
- Inputs: required/`default`/`optional` marker column; multi-line defaults collapse to a single elided continuation
  line; the signature on the section header uses the argument color.
- Local xprompts: name in the invocation-argument color, line count, description. Bodies are not expanded — they are
  reachable via `--format raw`, and expanding them would bury the primary definition.
- Workflow steps replace the `DEFINITION` body block for a workflow with no `prompt_part`; a workflow that has one shows
  both, steps first. Each step body renders through the right lexer (`python`/`bash` via `Syntax`, `prompt_part`/`agent`
  via the xprompt highlighter). Cap each step body at 20 lines with an explicit `… (N more lines)` note — never
  silently.
- References: resolved marker in the success color, unresolved in the error color. This is what makes `show` a
  navigation tool: a typo'd `#reference` in a definition is visible at a glance.
- Trailing hint line, dim: `sase xprompt explain <name>` for workflows, `sase xprompt expand '#<name>'` otherwise.
- The `Console` is constructed by the caller:
  `Console(no_color=not use_color, force_terminal=use_color, color_system="256", width=<terminal width or 100 when not a TTY>, markup=False, emoji=False, highlight=False)`.
  `markup=False` is mandatory — it is the second half of the injection defense. A fixed width off-TTY keeps piped output
  and tests deterministic.
- `--color never` must produce output with zero `\x1b` bytes; assert that directly in a test.

Tests (`tests/xprompt/test_cli_show_render.py`, `tests/xprompt/test_cli_show_body.py`):

- Plain-mode golden for a representative record covering every section.
- Color mode emits the expected SGR sequence for an invocation, a directive, and a separator, matching
  `highlight_theme()` rather than hardcoded escapes.
- A body containing `[bold]red[/bold]`, a lone `[`, and a `%` inside backticks renders literally and unstyled.
- Gutter numbers start at `body_first_line` when known and at 1 when not.
- A 500-character single-line body folds without loss; a tab survives.
- Sections with no content are absent, not empty-headed.
- Every step body over 20 lines carries the elision note.
- `sase bead show` output is unchanged after the palette constant move (existing bead tests suffice; run them).

## Phase `cli`: wiring, help, and docs

### Parser — `src/sase/main/parser_xprompt.py`

Register `show` with the three flags in alphabetical order and a `RawDescriptionHelpFormatter` epilog, matching the
shape `register_bead_show_parser` uses:

```
help:        "Show one xprompt definition with syntax highlighting"
description: Show one xprompt or workflow definition: its declared properties, typed inputs, local
             helper xprompts, highlighted body, provenance, and the references it makes. The NAME
             argument accepts a bare name or a copied reference (#name, #!name, /name); arguments
             such as #name(a, b) are ignored with a note. --format json emits a versioned record and
             --format raw emits the exact definition source bytes.
epilog:      examples:
               sase xprompt show sase/reads
               sase xprompt show '#!sync'
               sase xprompt show plan --format json | jq .inputs
               sase xprompt show coder --format raw > coder.md
               sase xprompt show t --color always | less -R
```

While in this file, also give the existing `catalog`, `expand`, `explain`, and `graph` registrations a consistent
sorted-by-name registration order. Help rendering is already sorted centrally, so this is cosmetic source hygiene, not a
behavior change — do not alter any existing flag, default, or help string while doing it.

### Handler — `src/sase/main/xprompt_handler.py`

Add a `show` branch that delegates to `sase.xprompt.cli_show.handle_show(args) -> int` and `sys.exit`s its return, and
extend the bare-invocation usage string to include `show`. Keep the handler thin; all logic lives in `cli_show.py`.

### `src/sase/xprompt/cli_show.py`

Orchestrates: normalize → resolve → dispatch on format.

- `full` → build the `Console` and call `render_show`.
- `json` → `json.dump(record.to_json_dict(), sys.stdout, indent=2)` plus a trailing newline.
- `raw` → write `record.raw` to stdout verbatim with no added trailing newline; exit 2 when it is `None`.
- Warnings always go to stderr for `json` and `raw` so those streams stay clean; in `full` mode they are the rendered
  `WARNINGS` section.

### Documentation

- `docs/xprompt.md` — new `### sase xprompt show` section under **CLI Subcommands** (placed to keep the section list
  readable), the anchor list at the top updated, and "provides five subcommands" corrected to six.
- `docs/cli.md` — a row in the xprompt table: `` `sase xprompt show` `` → "Show one xprompt definition with properties,
  provenance, and syntax highlighting."
- `docs/configuration.md` — a `### sase xprompt show` flag table in the same style as its neighbors.

### Cleanup

Remove every `--epic-symbol` entry this epic added to the `Justfile`, now that each symbol has a real non-test consumer.
Confirm with `just _lint-symvision`, and leave the pre-existing `sase-e6(...)` entries alone.

Tests:

- `tests/main/test_parser_xprompt_show.py` — `show` present in the sorted subcommand list; `-c, --color`,
  `-f, --format`, `-p, --project` all documented with both forms; examples present in the epilog; bare `sase xprompt`
  still delegates to `list`.
- `tests/main/test_xprompt_show_handler.py` — end-to-end through `create_parser()` and the handler against a `tmp_path`
  xprompt directory: exit 0 for a hit, exit 1 with suggestions for a miss, exit 2 for `--format raw` with an unreadable
  source, `--format json` parses and carries `schema_version`, `--color never` output has no escape bytes, and
  `--format raw` output equals the file bytes exactly.

## Verification

Each phase, before reporting done: `just install`, then `just check`. The `cli` phase additionally runs the command by
hand against at least one entry from each source bucket present in the workspace — a project `.md`
(`sase xprompt show sase/reads`), a config entry (`sase xprompt show actstat`), a package built-in
(`sase xprompt show t`), a skill (`sase xprompt show sase_plan`), an embeddable workflow (`sase xprompt show commit`),
and a standalone workflow (`sase xprompt show '#!sync'`) — checking each in both `--color always` and `--color never`,
and confirming `--format raw` output is byte-identical to the underlying source
(`diff <(sase xprompt show t --format raw) src/sase/xprompts/t.md`).

## Out of scope

Deliberately excluded, and worth filing as follow-up task beads if wanted:

- Migrating the ACE xprompt browser preview (`src/sase/ace/tui/modals/xprompt_browser_preview.py`) onto the shared
  highlight core. It would be a real improvement — that preview currently renders as plain markdown — but it touches the
  PNG visual snapshot suite, and this epic should not put a CLI feature behind a snapshot rebase.
- Showing a local helper xprompt directly (`sase xprompt show 'reads#_article_search_agent'`).
- A pager, `--all`, and argument binding; `sase xprompt explain` already covers the last of those.
