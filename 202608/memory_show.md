---
tier: tale
title: Add `sase memory show` and refactor `sase memory read` onto shared view logic
goal:
  "`sase memory show <note>.md` prints a long-term memory note without writing an audit
  event or requiring agent identity, and `sase memory read` produces byte-identical
  stdout by routing through the same shared resolve/render code, mirroring the `sase
  glossary show`/`sase glossary read` pair."
size: medium
proposed_by: bbugyi200.athena.05s
create_time: 2026-08-18 07:26:28
status: wip
---

# Plan: `sase memory show`

## Goal

Add a `sase memory show` subcommand that resolves and prints a long-term memory note the
same way `sase memory read` does today, minus the audit event and the agent-identity
requirement. Refactor `sase memory read` so both subcommands share one resolution
function and one renderer, exactly like `sase glossary show`/`sase glossary read` share
`resolve_glossary_view()`/`emit_glossary_view()`. Add a `-f/--format` option
(`json | markdown | rich`) to both, with `markdown` as the default so today's
`sase memory read` stdout stays byte-identical.

## Current State

The glossary pair (the model to copy):

- `src/sase/glossary/cli_show.py` owns `resolve_glossary_view()` (project + closure
  resolution), `emit_glossary_view()` (format dispatch), and
  `handle_glossary_show_command()`.
- `src/sase/glossary/cli_read.py` imports both helpers, normalizes the reason, requires
  agent identity, appends the audit event, and only then calls `emit_glossary_view()`.
  Nothing is printed unless the read was recorded.
- `src/sase/glossary/render.py` owns `GlossaryShowFormat` and
  `render_glossary_closure()` with `json`/`markdown`/`rich` branches. Its module
  docstring states the invariant: `show` and `read` must print identical output for
  identical arguments.
- `src/sase/main/parser_glossary.py` has
  `_add_closure_arguments(parser, *, require_reason=False)`, called once for `show` and
  once with `require_reason=True` for `read`.

The memory side today:

- `src/sase/memory/cli_read.py` (59 lines) is the only entry point. It calls
  `normalize_read_reason()`, `require_agent_identity()`,
  `validate_memory_read_path(args.memory_path, home_root=Path.home())`,
  `read_memory_content()`, then `_render_memory_read_output()` (body + `## Children`
  section), builds and appends the event, and writes the string to stdout.
- `src/sase/memory/read_log.py` owns validation (`ValidatedMemoryPath`,
  `MemoryReadContent`, frontmatter stripping) and the audit log. No changes needed
  there.
- `src/sase/memory/notes.py` owns `discover_memory_notes()` and
  `render_children_section()`.
- `src/sase/main/parser_memory.py` registers the `read` subparser with a single
  `memory_path` positional plus required `-r/--reason`.
- `src/sase/main/memory_handler.py` dispatches on `args.memory_subcommand` and prints
  `Usage: sase memory {agent-docs,init,list,log,read,review,write}` for unknown
  subcommands.

## Design Decisions

1. **Module layout mirrors glossary.** New `src/sase/memory/cli_show.py` (resolve +
   emit + handler) and new `src/sase/memory/render.py` (format dispatch). `cli_read.py`
   imports the two shared helpers and keeps only the audit wrapper.
2. **`markdown` is the shared default format for both subcommands.** `sase memory read`
   stdout is consumed by agents and by the `/sase_memory_read` skill as raw markdown;
   its current output must not change. Both subcommands keep the same default so the
   glossary invariant ("identical output for identical arguments") holds. `rich` is
   available for humans via `-f rich`.
3. **`show` never requires agent identity and never writes to the audit log.** That is
   the capability gap it fills: `sase memory read` fails outside an agent context, so a
   human shell currently has no supported way to view a note.
4. **One positional path, as today.** `sase glossary show` takes `TERM [TERM ...]`, but
   `memory_path` stays single-valued: the audit event schema is one event per note, and
   multi-note markdown output would need a separator that changes single-note output.
   See Non-Goals.
5. **No `-p/--project`.** Memory roots resolve from cwd then `~`, unchanged. Adding
   project selection would change `read`'s resolution contract. See Non-Goals.
6. **No new backend/domain logic.** This is CLI wiring and presentation over existing
   Python helpers in `sase.memory.read_log` and `sase.memory.notes`; nothing crosses the
   Rust core boundary.

## Implementation

### 1. `src/sase/memory/render.py` (new)

Module docstring must state the show/read identical-output invariant, like
`src/sase/glossary/render.py` does.

```python
MemoryShowFormat = Literal["json", "markdown", "rich"]


@dataclass(frozen=True, slots=True)
class ResolvedMemoryNote:
    """A validated long-term memory note plus the context show/read print."""

    content: MemoryReadContent
    children: tuple[MemoryNote, ...]
    origin: Literal["home", "project"]
    project_name: str


def render_memory_note(
    view: ResolvedMemoryNote,
    *,
    output_format: MemoryShowFormat,
    console: Console | None = None,
) -> None:
    """Print *view* to stdout in the requested format."""
```

- `markdown` (default): produce exactly what `_render_memory_read_output()` produces
  today — `content.body`, and when
  `render_children_section(view.children, content.path.note)` is non-empty, the body
  normalized to end with a blank line followed by the children section. Move that helper
  here verbatim as `_memory_note_markdown(view) -> str` and write it with
  `sys.stdout.write()`. Byte-for-byte parity with today is a hard requirement.
- `json`: `json.dump(payload, sys.stdout, indent=2, sort_keys=True)` plus a trailing
  newline, mirroring `_glossary_closure_json_payload`. Payload:

  ```json
  {
    "project": "<project_memory_name(cwd)>",
    "origin": "project",
    "note": {
      "path": "sase/memory/hub.md",
      "canonical_path": "hub.md",
      "resolved_path": "<absolute path>",
      "type": "long",
      "parent": "AGENTS.md",
      "description": "Hub memory.",
      "body": "<frontmatter-stripped body, no children section>",
      "byte_count": 123,
      "frontmatter_stripped": true
    },
    "children": [{ "path": "sase/memory/child_a.md", "description": "Alpha child." }]
  }
  ```

- `rich`: a `Group` printed through `console or Console()`, styled with
  `SECTION_COLOR`/`PATH_COLOR` from `src/sase/cli_show_palette.py` the way
  `_build_header()` in the glossary renderer does:
  - header grid row: left `MEMORY` (bold accent) + the note's canonical relative path
    (`view.content.path.note.relative_path`); right, dim:
    `<origin> · <type> · N children` (use `child`/`children` correctly for N == 1, and
    omit the children segment when N == 0).
  - the note description on its own dim line when present.
  - the body rendered with `rich.markdown.Markdown` (already used in
    `src/sase/memory/review_tui/app.py` and `src/sase/notification_gates/cli_act.py`).
  - a trailing `Children` block listing `relative_path — description` per child, dim,
    only when children exist. Do not render the markdown `## Children` prose block in
    this format; the structured list replaces it.

### 2. `src/sase/memory/cli_show.py` (new)

```python
def resolve_memory_view(args: argparse.Namespace) -> ResolvedMemoryNote:
    """Resolve the note, children, and origin shared by ``show`` and ``read``."""


def emit_memory_view(
    view: ResolvedMemoryNote,
    args: argparse.Namespace,
    *,
    console: Console | None = None,
) -> None:
    """Print a resolved note using the same renderer as ``show``."""


def handle_memory_show_command(
    args: argparse.Namespace, *, console: Console | None = None
) -> None:
    """Print one long-term memory note without recording an audited read."""
```

- `resolve_memory_view()` computes `home_root = Path.home()`, calls
  `validate_memory_read_path(args.memory_path, home_root=home_root)`, then
  `read_memory_content()`, then `discover_memory_notes(content.path.content_root)`.
  `origin` is `"home"` when `content.path.content_root` equals the resolved
  `home_root.expanduser().resolve(strict=False)`, else `"project"`. `project_name` is
  `project_memory_name(Path.cwd())`. Resolve `Path.home()` lazily inside the function so
  the existing `monkeypatch.setattr(Path, "home", ...)` tests keep working.
- `emit_memory_view()` reads the format with
  `cast(MemoryShowFormat, getattr(args, "format", "markdown"))` — `getattr` with a
  default is required because existing tests build bare
  `argparse.Namespace(memory_path=..., reason=...)` objects, exactly as
  `emit_glossary_view()` does.
- `handle_memory_show_command()` wraps `resolve_memory_view()` in
  `except (MemoryReadError, OSError, UnicodeError)`, prints `f"sase memory show: {exc}"`
  to stderr and `sys.exit(1)`, then calls `emit_memory_view()` outside the `try`. It
  must not call `require_agent_identity()`.

### 3. `src/sase/memory/cli_read.py` (refactor)

Keep `handle_memory_read_command(args)` as the only public symbol, add an optional
`console: Console | None = None` keyword to match the glossary read handler, and reduce
the body to:

1. `reason = normalize_read_reason(args.reason)`
2. `agent = require_agent_identity()`
3. `view = resolve_memory_view(args)`
4. `event = build_memory_read_event(view.content, reason=reason, agent=agent, cwd=Path.cwd())`
5. `append_memory_read_event(event)`
6. outside the `try`: `emit_memory_view(view, args, console=console)`

Keep the existing `except (AgentIdentityError, MemoryReadError, OSError, UnicodeError)`
tuple and the `sase memory read: {exc}` stderr prefix. Delete
`_render_memory_read_output()` (its logic now lives in `render.py`). The "nothing is
printed unless the read was recorded" ordering must be preserved.

### 4. `src/sase/main/parser_memory.py`

- Add a module-private helper mirroring `_add_closure_arguments`:

  ```python
  def _add_memory_view_arguments(
      parser: argparse.ArgumentParser, *, require_reason: bool = False
  ) -> None:
  ```

  It adds the `memory_path` positional (same `metavar` and help text as today — the dest
  must stay `memory_path` so `NAME_TABLE` in `src/sase/completion/kinds.py` keeps
  offering memory completions), then `-f/--format` with
  `choices=("json", "markdown", "rich")`, `default="markdown"`, help
  `"Output format (default: markdown)"`, and, when `require_reason`, the existing
  required `-r/--reason`.

- Rewrite the `read` subparser to call
  `_add_memory_view_arguments(read_parser, require_reason=True)`; its description keeps
  the current wording and gains a sentence noting it is identical to `show` except that
  it records an audited read first.

- Register a `show` subparser directly after `read` (help output is sorted centrally by
  `_sort_subcommand_help()` in `src/sase/main/parser.py`, so registration order does not
  affect the sorted `{...}` metavar):
  - `help`: `"Print a long-term memory note without recording a read"`
  - `formatter_class=argparse.RawDescriptionHelpFormatter`
  - `description`: explains it resolves the note exactly like `sase memory read` —
    project `sase/memory/` first, then `~/sase/memory/` — strips leading frontmatter,
    appends the `## Children` section, and writes no audit event; and that agents
    consulting memory to do work must use `sase memory read` instead.
  - `epilog` with examples: `sase memory show generated_skills.md`,
    `sase memory show sase_beads.md -f rich`, `sase memory show cli_rules.md -f json`
  - `_add_memory_view_arguments(show_parser)`

- Add `  sase memory show generated_skills.md` to the `memory` group epilog examples,
  placed to keep that example list readable next to the existing `read` example.

### 5. `src/sase/main/memory_handler.py`

Add, following the existing pattern (lazy import inside the branch):

```python
if sub == "show":
    from sase.memory.cli_show import handle_memory_show_command

    handle_memory_show_command(args)
    sys.exit(0)
```

and update the usage string to
`Usage: sase memory {agent-docs,init,list,log,read,review,show,write}`.

### 6. `src/sase/xprompts/skills/sase_memory_read.md`

Add one bullet to `## Rules`: `sase memory show` prints the same content but records no
audit event, so it is for humans and tooling — never use it in place of
`sase memory read` when consulting memory to do work. Source-template edit only; the
chezmoi deploy (`sase skill init --force`) happens later from a clean, landed tree per
`sase/memory/generated_skills.md` and is not part of this change.

## Tests

New file `tests/main/test_memory_cli_show.py`, following the shape of
`tests/main/test_glossary_cli_show.py` and reusing `write()` from
`tests/main/memory_handler_helpers.py`:

1. `show` prints body plus the `## Children` section and writes no audit log — assert
   `not memory_read_log_path(cwd=tmp_path).exists()`.
2. `show` succeeds with no `SASE_AGENT_NAME`/`SASE_AGENT`/`SASE_ARTIFACTS_DIR` in the
   environment (`monkeypatch.delenv(..., raising=False)`), proving it needs no agent
   identity.
3. `show` stdout is byte-identical to `read` stdout for the same note (the glossary
   `test_read_records_event_then_matches_show_stdout` analogue), for both the default
   format and `-f json`.
4. `-f json` payload: assert `project`, `origin`, `note.canonical_path`, `note.type`,
   `note.description`, `note.body`, `note.frontmatter_stripped`, and the sorted
   `children` list.
5. `-f rich` rendering through a fixed-width
   `Console(file=StringIO(), force_terminal=False, color_system=None, width=160)`:
   assert the path, `long`, the description, and both child paths appear.
6. Home-root fallback sets `origin == "home"` in the `json` payload.
7. Error paths exit 1 with `sase memory show:` on stderr, empty stdout, and no log file:
   a missing note, a nested (non-flat) path, and a `type: short` note.

Updates to existing tests:

- `tests/main/test_memory_read_list.py`: existing `read` assertions must pass unchanged
  (that is the regression guard for byte-identical output). Add one test that
  `read -f json` still appends exactly one event.
- `tests/main/test_memory_parser_handler.py`: parse `["memory", "show", "foo.md"]` and
  assert `memory_subcommand == "show"`, `memory_path == "foo.md"`,
  `format == "markdown"`; assert `-f rich` parses and an invalid `-f` exits; assert
  `read` still defaults `format` to `"markdown"`; extend the handler-dispatch test so
  `show` routes to `sase.memory.cli_show.handle_memory_show_command`.
- `tests/main/test_parser_command_help.py`: update the metavar assertion to
  `{agent-docs,init,list,log,read,review,show,write}`, and assert the `show` subparser
  help documents `-f, --format` and the "records no audit event" wording.

## Docs

- `docs/cli.md`: add a `sase memory show` row immediately before the `sase memory read`
  row, matching the column formatting of the surrounding table and linking to
  `memory.md#show-a-note`.
- `docs/memory.md`: add a `## Show a Note` section before `## Audited Reads` covering
  path resolution (project then home), `type: long` only, frontmatter stripping, the
  `## Children` section, the three `-f/--format` values with `markdown` as the default,
  and the fact that no audit event is written and no agent identity is needed. In
  `## Audited Reads`, state that agents must use `read`, not `show` — the same wording
  the glossary section uses ("Agents should always use `read`, not `show`") — and note
  that `show` is the supported way for a human shell to view a note.

## Non-Goals

- **Multiple note paths per invocation.** `sase glossary show` accepts
  `TERM [TERM ...]`; this keeps one `memory_path`. Rationale in Design Decisions #4.
- **`-p/--project` selection.** Root resolution stays cwd-then-home.
- **Near-miss "did you mean" candidates** for an unknown note. The glossary resolver
  offers them; adding them here would change `sase memory read`'s error output, which is
  shared and asserted by existing tests.
- **Allowing `type: short` notes through `show`.** Resolution stays identical to `read`.
- **Any edit to `sase/memory/*.md`, `AGENTS.md`, or the provider shims** (`CLAUDE.md`,
  `GEMINI.md`, `OPENCODE.md`, `QWEN.md`), and no `sase memory init` run. Memory-file
  edits require explicit user permission that this plan does not carry. The generated
  `sase/memory/README.md` template
  (`src/sase/main/init_memory/templates/memory-README.template.md`) and
  `src/sase/memory/assets/memory-directory-map.prompt.md` mention `sase memory read`;
  leave both alone and propose any update separately.
- **Deploying the edited skill template** to chezmoi.
- **`CHANGELOG.md`**, which is release-generated and validated by
  `just _lint-changelog`.

## Verification

```bash
just install
just check
```

`just check` runs ruff, mypy, symvision, toobig, keep-sorted, markdown/python format
checks, and the diff-scoped test lane. Notes for the implementer:

- `fmt-md-check` covers `docs/*.md` and the plan-adjacent markdown edits; run `just fmt`
  before `just check` if prettier reformats them.
- If symvision flags a new public symbol as unused, read `sase/memory/symvision.md` with
  the `/sase_memory_read` skill before changing anything. Export exactly what is used:
  `render.py` exports `MemoryShowFormat`, `ResolvedMemoryNote`, `render_memory_note`;
  `cli_show.py` exports `emit_memory_view`, `handle_memory_show_command`,
  `resolve_memory_view`.
- Sanity-check the two commands by hand from the repo checkout:

  ```bash
  sase memory show cli_rules.md | head -20
  sase memory show cli_rules.md -f rich
  sase memory show cli_rules.md -f json | head -20
  sase memory show does_not_exist.md; echo "exit=$?"
  ```

Hand `just check-full` to `/sase_monitor` before landing.
