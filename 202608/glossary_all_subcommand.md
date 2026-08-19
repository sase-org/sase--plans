---
tier: tale
title: Add `sase glossary all`, a dictionary-style full-catalog view
goal:
  A new `sase glossary all` subcommand prints every configured glossary term in full —
  alphabetically ordered, letter-sectioned, with aliases, definitions whose cross-term
  mentions are underlined, and inbound backlinks — in `rich`, `markdown`, and `json`.
size: medium
proposed_by: bbugyi200.athena.07s
create_time: 2026-08-19 09:59:02
status: wip
---

# Add `sase glossary all`, a dictionary-style full-catalog view

## Problem

The `sase glossary` group has no way to read the whole glossary.

- `sase glossary list` (also the bare `sase glossary` default) prints a table of every
  term with aliases, an outbound-reference count, and a **truncated first sentence**
  (`src/sase/glossary/cli_list.py:117-124`, `_SUMMARY_WIDTH = 72`). It is a table of
  contents, not the text.
- `sase glossary show TERM [TERM ...]` prints full definitions but requires naming every
  term. Reading the whole `sase` glossary today means typing all 30 terms, quoting the
  multi-word ones.

So the one thing a glossary is for — sitting down and reading it — is the one thing the
CLI cannot do. `sase glossary all` fills exactly that gap.

## Design

I own the design decisions below; implement them as specified rather than re-deciding
them. Each records the alternative that was rejected, so a reasonable-looking "fix"
during implementation does not silently undo a deliberate choice.

### D1. A subcommand, not `show --all`

`sase glossary all` is a peer of `list`/`show`, not a flag on `show`. `show`'s `TERM`
positional is `nargs="+"` (`src/sase/main/parser_glossary.py:284-288`), so a
`show --all` variant would have to make a required positional conditionally optional —
an argparse shape that produces bad `--help` output and an ugly "either TERM or --all"
runtime check. A separate subcommand keeps every existing signature intact, and it is
what was asked for.

### D2. `all` is `show` over every term — route it through the closure resolver

Do **not** write a new loop over `catalog.entries`. Build the view by handing every
entry to the existing resolver:

```python
resolve_glossary_closure(resolved.catalog, resolved.compiled, entries)
```

`resolve_glossary_closure` accepts `Sequence[str | GlossaryEntry]`
(`src/sase/glossary/resolution.py:186-190`, `_resolve_roots` at line 262), so entries
can be passed directly with no lookup round-trip. Three properties fall out of this for
free, and they are the reason for the choice:

1. **Every node is `origin="requested"` at `depth=0`.** Every catalog entry is already a
   root, so the `builders_by_index` check at line 236 always hits and no `related` node
   is ever created. `truncated` is therefore always `False` regardless of `depth`; pass
   no `depth` argument.
2. **Every cross-reference in the whole glossary gets underlined.**
   `_highlighted_definition` (`src/sase/glossary/render.py:201-221`) underlines a span
   only when its target is in `printed_indices`. In a full-catalog closure every term is
   printed, so _every_ mention of _every_ term is highlighted. This is the visual payoff
   of the feature and it costs zero new code.
3. **`node.also_referenced_by` becomes the complete inbound backlink set.** Roots carry
   `referrer=None`, so `_record_additional_referrer` (line 336) records every referring
   term into `also_referenced_by`, deduped, in root order. Outbound references are
   visible inline as underlines; inbound backlinks are the information the definition
   text cannot carry, so those get their own line.

### D3. Alphabetical order and dictionary letter sections

Sort entries by `(entry.normalized_term, entry.index)` before resolving, and render a
left-aligned `rich.rule.Rule` whenever the first letter changes:

```
A ──────────────────────────────────────────────────────────────────────
```

`sase glossary add` keeps `sase.yml` sorted, but a hand-edited config need not be, and
letter sections built on an unsorted catalog would repeat (`A … S … A …`). Sorting makes
the sections correct by construction and makes the output independent of file order.
This is a deliberate divergence from `list`, which prints catalog order; in practice the
two agree because the config is maintained sorted. Do not "fix" `list` to match — that
is out of scope.

The group key is `term[:1].upper()` when that character is alphabetic, else `#`.

### D4. `all` is not an audited read

`all` is unaudited, exactly like `show`. It gains no `-r/--reason`, imports nothing from
`sase.glossary.read_log`, and writes no event. The rich view ends with one dim line:

```
Not an audited read — agents must use: sase glossary read <term> -r "<why>"
```

Rationale: the audit log exists so `sase glossary log` can attribute agent reads
(`docs/memory.md:194-204`), and a convenient unaudited full dump is exactly what would
rot it. `show` already occupies the unaudited-human-view slot, so `all` introduces no
new kind of hole — but it does make the hole convenient, and the footer is the cheap,
honest guard against that. An audited `all -r "<why>"` mode was considered and
**deliberately deferred** (see out of scope).

### D5. Same three formats as `show`, with a `list`-shaped JSON payload

`-f/--format` is `json | markdown | rich`, default `rich` — identical choices, ordering,
and default to `show`/`read`, so the group has one format vocabulary.

The JSON payload is _not_ the closure payload. In a full-catalog closure every node has
`"depth": 0`, `"origin": "requested"`, `"referrer": null`, and top-level `depth_limit`,
`truncated`, and `requested_references` are all constants — noise in a machine format.
Instead emit a catalog payload that is a strict superset of `list -f json`'s per-term
record (`src/sase/glossary/cli_list.py:127-145`), reusing its `reference_terms` field
name so the two are interchangeable, plus the new `referenced_by`:

```json
{
  "project": "sase",
  "count": 30,
  "terms": [
    {
      "term": "Agent Hood",
      "aliases": ["hood", "agent neighborhood"],
      "definition": "An agent hood is …",
      "reference_terms": ["Sase Agent"],
      "referenced_by": ["Agent Neighbor"],
      "source": { "config_path": "sase/sase.yml" }
    }
  ]
}
```

The rule the three formats follow: **`rich` and `markdown` carry only what the
definition text cannot; `json` carries the whole graph.** Outbound references are
already legible in the prose (and underlined in `rich`), so only `json` lists
`reference_terms`; inbound backlinks appear in all three.

### D6. No `PATTERN`, no `-d/--depth`

`all` takes no positional arguments — a filtered "all" is a contradiction, and `list`
already owns filtering (`PATTERN` plus `-d/--definitions`). `-d/--depth` is meaningless
when every term is already included. Adding either would create two overlapping filter
surfaces in one command group.

## Command surface

```
sase glossary all [-f {json,markdown,rich}] [-p REF]
```

Both options already have short aliases, and neither is required, so the
`sase/memory/cli_rules.md` rules ("every public long option gets a short alias",
"options must not be required", "keep subcommands sorted alphabetically") all hold.
Register `all` **immediately after `add`** in `parser_glossary.py` — argparse lists
subcommands in registration order, and
`test_parser_glossary_help_lists_subcommands_alphabetically`
(`tests/main/test_glossary_parser_handler.py:148-154`) asserts that order.

## Output specification

### `rich` (default)

Header, then letter-sectioned entries, then the footer.
`_print_rich_without_trailing_whitespace` already strips wrap padding from every line;
the catalog view goes through it too.

```
GLOSSARY  sase                                                        30 terms

A ──────────────────────────────────────────────────────────────────────────

● Agent Clan
An agent clan is a named, rootless container for agents that run in parallel …
referenced by Agent Family, Agent Node

● Agent Hood
aka hood · agent neighborhood
An agent hood is a group of agents that are all named with the same … prefix …
referenced by Agent Neighbor

C ──────────────────────────────────────────────────────────────────────────

● Current Project
…

Not an audited read — agents must use: sase glossary read <term> -r "<why>"
```

Per entry, in this order: `● Term` (accent bullet, bold term); `aka a · b` (dim, only
when `display_aliases` is non-empty); the stripped definition with cross-term mentions
underlined in the accent color; `referenced by A, B` (dim, only when
`also_referenced_by` is non-empty). One blank line between entries and after each rule.

Note the deliberate difference from `show`'s node block
(`src/sase/glossary/render.py:150-199`): backlinks move **below** the definition and the
label changes from `also mentioned by` to `referenced by`. In `show`, those lines
explain _why a term appeared_ and must precede it; in a catalog every term appears
unconditionally, so backlinks read as a dictionary's "see also" and belong at the end.
There is also no right-hand tag column (every entry would say `REQUESTED`) and no
indentation (every node is depth 0).

### `markdown`

Same assembly style as `_glossary_closure_markdown` (`render.py:249-263`) — pieces
joined by `\n`, trailing `\n`:

```markdown
GLOSSARY: sase

# Agent Hood

aka hood, agent neighborhood

An agent hood is a group of agents …

referenced by Agent Neighbor
```

Flat `#` headings (every term is depth 0), no letter rules — headings are already
navigable — and no `*Requested*` provenance line, which would be noise on every entry.

### `json`

As specified in D5. `json.dump(..., indent=2, sort_keys=True)` followed by a newline,
matching every other glossary JSON emitter.

### Empty catalog

A project can resolve to a catalog with zero entries. `resolve_glossary_closure` returns
an empty closure for empty roots (`resolution.py:198-205`), so the handler needs no
special case — handle it in the renderer:

- `rich`: print `no glossary terms configured for <project>` and nothing else (no
  header, no footer). Exit 0, matching `list`'s "no terms matched" behavior.
- `markdown`: the `GLOSSARY: <project>` line alone.
- `json`: `{"project": "<project>", "count": 0, "terms": []}`.

A project with **no** glossary configured at all is a different case and is already
handled by `resolve_glossary_cli_project`, which raises `GlossaryCliError`.

## Implementation steps

### 1. `src/sase/glossary/render.py` — add the catalog renderer

Add a `# --- catalog ---` section. Everything below is new except the small
`_build_header` refactor.

**Refactor first (no behavior change):** extract the header grid that `_build_header`
(line 131) builds into `_header_grid(project_name: str, right: str) -> RenderableType`
holding the existing `Table.grid` / two-column / `GLOSSARY` + project-name body
verbatim, and make `_build_header` a thin caller that computes the existing
`"{total} {term_word} · {requested} requested · {related} related"` string and passes it
as `right`. `show`'s output must be byte-identical after this.

**New public entry point**, mirroring `render_glossary_closure`'s signature and format
dispatch exactly:

```python
def render_glossary_catalog(
    closure: GlossaryClosure,
    *,
    project_name: str,
    output_format: GlossaryShowFormat,
    console: Console | None = None,
) -> None:
```

`json` → `_glossary_catalog_json_payload` via
`json.dump(..., indent=2, sort_keys=True)`; `markdown` →
`sys.stdout.write(_glossary_catalog_markdown(...))`; otherwise
`_print_rich_without_trailing_whitespace(target, _glossary_catalog_renderable(...))`.

**New private helpers:**

- `_glossary_catalog_renderable(closure, *, project_name) -> Group` — returns the empty
  message when `not closure.nodes`; otherwise
  `_header_grid(project_name, f"{n} term{'s' if n != 1 else ''}")`, a blank `Text("")`,
  then per node: a `Rule` when `_catalog_group_key` changes, the entry blocks, a blank
  line; then the dim audit-hint footer. `printed_indices` is
  `{node.entry.index for node in closure.nodes}`, as in `_glossary_closure_renderable`.
- `_catalog_group_key(term: str) -> str` — `term[:1].upper()` if alphabetic else `"#"`.
- `_build_catalog_entry_blocks(node, *, printed_indices) -> list[RenderableType]` — the
  four lines from the output spec. Reuse `_highlighted_definition` unchanged. No
  `Padding`, no `Table.grid` — plain `Text` renderables in a list, which is why this
  view needs no expand/padding workaround.
- `_glossary_catalog_markdown(closure, *, project_name) -> str`.
- `_glossary_catalog_json_payload(closure, *, project_name) -> dict[str, object]` and
  `_catalog_term_json(node, terms_by_index)`.
- `_outbound_reference_terms(node, terms_by_index: Mapping[int, str]) -> tuple[str, ...]`
  — walk `node.spans` in order, skip `span.entry_index == node.entry.index` and repeats,
  map index to term. Build
  `terms_by_index = {n.entry.index: n.entry.term for n in closure.nodes}` once in the
  payload builder.

Use `Rule` from `rich.rule` and style it
`Rule(Text(key, style=f"bold {_ACCENT}"), align="left", style="dim")` — verified to
render `A ────…` filling the console width with no trailing whitespace. Add module
constants next to `_ACCENT` for the audit-hint and empty-catalog strings.

Add `render_glossary_catalog` to `__all__`. The file is 315 lines today and the `toobig`
gate warns at 700, so it stays comfortably in one module; keeping it there is what lets
the catalog view reuse the module-private `_highlighted_definition`,
`_print_rich_without_trailing_whitespace`, and `_ACCENT` without tripping the symvision
private-misuse rule.

### 2. `src/sase/glossary/cli_all.py` — new handler

Model it on `cli_list.py`/`cli_show.py`:

```python
def handle_glossary_all_command(
    args: argparse.Namespace, *, console: Console | None = None
) -> None:
    """Print every glossary term configured for a project."""
    try:
        resolved = resolve_glossary_cli_project(getattr(args, "project", None))
    except GlossaryCliError as exc:
        print(f"sase glossary all: {exc}", file=sys.stderr)
        sys.exit(1)

    closure = resolve_glossary_closure(
        resolved.catalog,
        resolved.compiled,
        _catalog_order(resolved.catalog.entries),
    )
    render_glossary_catalog(
        closure,
        project_name=resolved.project_name,
        output_format=cast(GlossaryShowFormat, getattr(args, "format", "rich")),
        console=console,
    )
```

with `_catalog_order(entries)` returning
`tuple(sorted(entries, key=lambda entry: (entry.normalized_term, entry.index)))`. Export
`handle_glossary_all_command` in `__all__`. No `GlossaryLookupError` handling is needed
— entries are passed as objects, so no lookup can fail.

### 3. `src/sase/main/parser_glossary.py` — register the subcommand

Extract the shared format option so `show`, `read`, and `all` cannot drift:

```python
def _add_closure_format_option(parser: argparse.ArgumentParser) -> None:
    parser.add_argument(
        "-f", "--format",
        choices=("json", "markdown", "rich"),
        default="rich",
        help="Output format (default: rich)",
    )
```

Call it from `_add_closure_arguments` in place of the inline block (lines 296-303) and
from the new `all` parser.

Register `all_parser` between the `add` and `del` blocks:

- `help="Print every glossary term in full"`
- `formatter_class=argparse.RawDescriptionHelpFormatter`
- `description`: what it prints (every term, alphabetical, aliases, full definition,
  underlined cross-references, backlinks), how it differs from `list` (a table with
  truncated summaries) and `show` (named terms plus their closure), and that it is not
  an audited read, so agents must use `sase glossary read TERM -r "<why>"`.
- `epilog` examples: `sase glossary all`, `sase glossary all -f markdown`,
  `sase glossary all -f json`, `sase glossary all -p sase`.
- Then `_add_closure_format_option(all_parser)` and `_add_project_option(all_parser)`.

Also add `  sase glossary all` to the **group** parser's `epilog`, directly after the
two `sase glossary list` lines and before the `show` lines, so the examples read list →
all → show → read.

### 4. `src/sase/main/glossary_handler.py` — dispatch

Add the `all` branch immediately after the `add` branch, in the established shape (local
import of `sase.glossary.cli_all`, call, `sys.exit(0)`), and update the fallback usage
string to `Usage: sase glossary {add,all,del,list,log,read,show}`.

### 5. Regenerate the completion snapshot

The structural completion spec is checked in and validated:

```bash
just sync-completion-spec
```

Commit the resulting `tests/completion/snapshots/cli_spec.json` diff. Do **not** add an
entry to `src/sase/completion/kinds.py` — that map keys
`(group, subcommand) → ValueKind` for **positional term** completion (lines 117-120),
and `all` has no positional.

### 6. Docs

`docs/cli.md`: add one row to the command table, immediately after the `sase glossary` /
`sase glossary list` row:

> `sase glossary all` — Print every glossary term in full, alphabetically, with aliases,
> definitions, and inbound references. → `[Glossary](memory.md#glossary)`

`docs/memory.md`, the `## Glossary` section:

- Add `sase glossary all` and `sase glossary all -f markdown` to the example block
  (lines 149-162), right after the two `list` lines.
- Add a paragraph documenting the command between the `list` paragraph (line 176) and
  the `show` paragraph (line 183): the alphabetical letter-sectioned full catalog, the
  three formats and the JSON shape, the highlighted cross-references and `referenced by`
  lines, and — explicitly — that it is not audited and agents must use `read`.

Run `just fmt-md` afterwards; both files are prose-wrapped and the `cli.md` table is
pipe-aligned by the formatter.

`CHANGELOG.md` is generated by release-please from conventional commits — do not
hand-edit it.

### 7. Tests

**New `tests/main/test_glossary_cli_all.py`.** Follow `test_glossary_cli_show.py`'s
shape: build fixtures with `glossary_entry`/`glossary_span`/`resolved_glossary_project`
from `tests/main/glossary_cli_helpers.py`, monkeypatch
`cli_all.resolve_glossary_cli_project`, and drive
`Console(file=StringIO(), force_terminal=False, color_system=None, width=160)`.
`diamond_resolved_glossary_project()` (Alpha → Beta/Gamma → Delta) is a ready-made
fixture with backlinks; add a small out-of-order and a `#`-prefixed fixture where
needed.

Cover:

- **rich**: header reads `GLOSSARY  sase` and `4 terms`; every term appears; `REQUESTED`
  and `RELATED` appear nowhere; `referenced by Beta, Gamma` appears under Delta; `aka B`
  under Beta; the audit-hint footer line is present; every line equals its `rstrip()`
  (the same trailing-whitespace guard `show`'s tests use).
- **letter sections**: a fixture spanning two letters emits one rule line per letter, in
  order, and none repeat; a term starting with a non-letter groups under `#`.
- **ordering**: entries supplied in non-alphabetical config order print alphabetically.
- **markdown**: `# Alpha` headings for every term, `referenced by …`, and no
  `*Requested*` provenance line.
- **json**: exact payload for the diamond fixture — `project`, `count`, and per-term
  `term`/`aliases`/`definition`/`reference_terms`/`referenced_by`/`source`; assert
  `reference_terms` for Alpha is `["Beta", "Gamma"]` (outbound, span order) and
  `referenced_by` for Delta is `["Beta", "Gamma"]` (inbound).
- **empty catalog**: `resolved_glossary_project(entries=())` prints
  `no glossary terms configured for sase` in rich, `{"count": 0, "terms": []}` in json,
  and exits 0.
- **error path**: `resolve_glossary_cli_project` raising `GlossaryCliError` exits 1 with
  `sase glossary all: …` on stderr.

**`tests/main/test_glossary_parser_handler.py`:**

- `test_parser_glossary_help_lists_subcommands_alphabetically`: add `"all"` to
  `expected` and change the literal to `"{add,all,del,list,log,read,show}"`.
- Add `all` parse coverage (`glossary_subcommand == "all"`, `format`, `project`) and a
  `-p` before/after case alongside the existing ones.
- Add `test_glossary_all_dispatches_to_all_handler`, mirroring the existing dispatch
  tests (monkeypatch `sase.glossary.cli_all.handle_glossary_all_command`, assert
  `SystemExit` code 0 and the forwarded namespace).

**`tests/main/test_glossary_cli_show.py`:** no assertion should change. If any does, the
`_build_header` extraction in step 1 was not behavior-preserving — fix the refactor, not
the test.

## Explicitly out of scope

- **An audited `all -r/--reason` mode.** It is a real idea — an agent that needs the
  whole vocabulary currently has to name every term in `read` — but it is a separate
  feature with its own surface (a whole-catalog `GlossaryReadEvent`,
  `require_agent_identity()` failing for humans who pass `-r` outside an agent context,
  and `sase glossary log` rendering a 30-term event). Ship the view first. Do not add
  `-r` here.
- **Changing `list`.** Its catalog-order listing, `_SUMMARY_WIDTH` truncation, `Refs`
  column, and `-f names`/`table`/`json` formats stay exactly as they are.
- **Changing `show`/`read` output.** The only edits to their code paths are the two
  behavior-preserving extractions (`_header_grid`, `_add_closure_format_option`).
- **The ACE Glossary panel** (`src/sase/ace/tui/modals/glossary_panel*.py`) and the `K`
  preview (`src/sase/ace/tui/widgets/_prompt_glossary.py`). The TUI already browses the
  full catalog interactively; it needs no "all" view and must not be touched.
- **`sase/memory/glossary.md`, `AGENTS.md`, and the provider shims.** The Tier 1
  glossary note tells agents to use `sase glossary read`, which stays correct. Memory
  files must not be edited without explicit user permission in the implementing
  conversation, and nothing here requires it. `docs/` is not memory and is in scope.
- **A pager.** The repo has no CLI pager convention and this change does not introduce
  one; long output is the shell's problem.

## Rust core backend boundary

This stays in Python. The Rust core already owns everything domain-level that `all`
consumes — catalog validation, effective aliases, and phrase matching via
`sase.core.glossary_facade` (`build_glossary_catalog`, `compile_glossary_catalog`,
`scan_glossary_spans`). Reference-closure walking already lives in Python
(`src/sase/glossary/resolution.py`) and is reused unchanged.

What this change adds is entirely CLI presentation: alphabetical ordering, first-letter
sectioning, Rich renderables, a Markdown serialization, and a JSON envelope. No wire
format, binding, or `sase-core` test changes are needed, and nothing here is behavior
another frontend would have to match — a future TUI "read the whole glossary" view would
share the _catalog and spans_ it already gets from the core, not this module's Rich
layout. If such a view is ever built and needs identical grouping, promote
`_catalog_group_key` at that point; do not pre-emptively move it to `sase-core` now.

## Verification

```bash
just install    # ephemeral workspace: dependencies may be stale
just sync-completion-spec
just check
```

The diff-scoped lane should select the glossary CLI tests; if it does not, run them
explicitly:

```bash
just test -- tests/main/test_glossary_cli_all.py \
              tests/main/test_glossary_cli_show.py \
              tests/main/test_glossary_parser_handler.py \
              tests/completion
```

Manual confirmation, which is the real check on "beautiful" — run each and look at it:

```bash
sase glossary all
sase glossary all -f markdown | head -40
sase glossary all -f json | python3 -m json.tool | head -40
sase glossary all -p sase | tail -5      # footer present
sase glossary show Stitch                # unchanged from before the change
```

In the first command, confirm: one letter rule per initial letter in alphabetical order;
mentions of other terms underlined inside definitions (this is the feature — if nothing
is underlined, the closure was not built over all entries); `referenced by` lines under
the terms other definitions cite; and no `REQUESTED` tags or indentation anywhere.

No ACE PNG visual snapshots are affected; do not pass `--sase-update-visual-snapshots`.

## Definition of done

- `sase glossary all` prints every configured term in full, alphabetically, in letter
  sections, with aliases, underlined cross-references, and `referenced by` backlinks.
- `-f markdown` and `-f json` emit the shapes specified above; `-p/--project` works
  before and after the subcommand; the empty catalog and unresolvable-project paths
  behave as specified.
- `all` records no audit event and the rich footer points agents at
  `sase glossary read`.
- `sase glossary --help` lists `{add,all,del,list,log,read,show}` and `all`'s own
  `--help` explains it against `list` and `show`.
- `sase glossary list`, `show`, and `read` output is unchanged.
- `tests/completion/snapshots/cli_spec.json`, `docs/cli.md`, and `docs/memory.md` are
  updated; `CHANGELOG.md` is not.
- `just check` passes.
