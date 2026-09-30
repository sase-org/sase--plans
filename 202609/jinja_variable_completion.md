---
tier: epic
title: Jinja2 variable completion in the prompt input and the xprompt LSP
goal: 'Typing `{{` in sase''s TUI prompt input, or in any editor that uses `sase-xprompt-lsp`,
  immediately shows every Jinja2 variable that is valid at that spot. The list covers
  the current xprompt''s or prompt stack''s declared `input:` properties, template
  locals, the variables sase injects into every agent prompt, and Jinja''s own globals.
  Each entry shows its type, source, default, and a description, and the entries are
  ranked the same way everywhere. Filters after `|`, tests after `is`, members after
  `wait.`/`loop.`, and statements after `{%` complete the same way. One Rust engine
  is the source of truth for the completion menu, editor hover, and the TUI''s unknown-variable
  lint.

  '
phases:
- id: catalog
  title: Rust Jinja catalog and wire types
  depends_on: []
  size: small
  description: 'catalog: add a static, documented catalog to sase-core covering sase''s
    built-in Jinja variables and their availability rules, Jinja globals, filters,
    tests, statement keywords, and namespace members, plus the request/response wire
    types the engine will return.'
- id: scan
  title: Rust Jinja tag scanner, slot classifier, and scope analysis
  depends_on: []
  size: medium
  description: 'scan: find the Jinja tag at the cursor (respecting literal zones,
    comments, raw blocks, frontmatter, and string literals), classify the completion
    slot, and extract the document scope: declared inputs, skill flag, %repeat/%wait
    directives, and position-aware template locals with the open-block stack.'
- id: assist
  title: Rust Jinja completion, ranking, documentation, hover, and scope variables
  depends_on:
  - catalog
  - scan
  size: medium
  description: 'assist: combine catalog and scope into ranked, fuzzy-matched candidates
    with availability states, shadowing, and shared markdown documentation. Expose
    jinja_completion, jinja_hover, jinja_scope_variables, and jinja_catalog as core
    functions.'
- id: bindings
  title: Python bindings for the Jinja engine
  depends_on:
  - assist
  size: small
  description: 'bindings: expose jinja_completion, jinja_scope_variables, and jinja_catalog
    through the sase_core_rs editor_completion binding domain, with round-trip tests.'
- id: lsp
  title: sase-xprompt-lsp Jinja completion and hover
  depends_on:
  - assist
  size: medium
  description: 'lsp: route in-tag positions to the engine ahead of every other completion
    surface, add `{`/`|` triggers that stay silent outside tags, derive the scope
    from the document path, render rich CompletionItems and hover, and document it
    in docs/editor.md.'
- id: python
  title: Python adapter, single source of truth, lint, and parity tests
  depends_on:
  - bindings
  size: medium
  description: 'python: add the sase Jinja adapter and move the core pin. Delete the
    Python builtin-name mirrors in favor of the Rust catalog. Switch the unknown-variable
    lint and gL/save-as-xprompt input inference to engine scope variables. Add runtime/catalog
    parity tests and update the docs/xprompt.md template-context reference.'
- id: tui-menu
  title: TUI Jinja completion menu redesign
  depends_on:
  - python
  size: medium
  description: 'tui-menu: drive the prompt input''s Jinja menu from the engine, using
    each pane''s frontmatter scope. Render aligned, theme-consistent rows with source
    badges, match highlighting, and a detail subtitle. Give Jinja precedence inside
    tags, and add dark/light PNG goldens.'
- id: tui-auto
  title: Auto-open the Jinja menu while typing
  depends_on:
  - tui-menu
  size: small
  description: 'tui-auto: open the menu the moment `{{`/`{%` auto-pair, on `|` and
    `.` inside tags, and on identifier typing, behind a new ace.prompt_completion.auto_jinja_menu
    setting. Keep alternation `|` handling out of tags and document the behavior in
    docs/ace.md.'
- id: parity
  title: TUI and LSP Jinja completion parity suite
  depends_on:
  - lsp
  - python
  size: small
  description: 'parity: prove the LSP binary and the Python adapter return identical
    ordered candidates on shared fixtures, including lifted-frontmatter versus inline-frontmatter
    documents and xprompt-path scope.'
proposed_by: bbugyi200.apollo.3g
create_time: 2026-09-30 08:47:10
status: wip
bead_id: sase-1df
---

- **PROMPT:** [prompts/202609/jinja_variable_completion.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/jinja_variable_completion.md)
- **BEAD:** [sase-1df](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1df/README.md)

# Plan: Jinja2 Variable Completion in the Prompt Input and the xprompt LSP

## Why

Today, typing `{{` in the prompt input shows a red "jinja diagnostics" error panel,
because `{{  }}` is momentarily an empty expression, and no suggestions. A basic menu
exists (`src/sase/ace/tui/widgets/jinja_completion.py`), but it opens only on Ctrl+T. It
mixes variables, every filter, and every keyword into one unranked, undocumented,
prefix-only list, and it never offers the xprompt's own declared `input:` properties. It
is also out of sync with the unknown-variable lint: the lint accepts declared inputs,
but the menu does not offer them.

External editors get nothing. `sase-xprompt-lsp` has no Jinja awareness at all, and
inside `{{ ... }}` its classifier falls back to file-history or snippet completion. It
even misreads `{{ a < b` as a `<placeholder>`.

The name lists also drift. `patch_name` is injected into every agent run
(`src/sase/axe/run_agent_exec.py` `_build_named_args`), but it is missing from
`BUILTIN_RUNTIME_NAMES` (`src/sase/xprompt/_jinja.py`), so it is never offered and the
lint flags it as unknown. No test ties the static list to the runtime.

## Design principles

1. **One engine, every surface.** Tag detection, the variable catalog, scope analysis,
   ranking, and documentation text are backend behavior per the Rust-core boundary. They
   live in sase-core as `sase_core::editor::jinja`. The TUI calls the engine through a
   thin Python adapter, and the LSP calls it directly. The TUI lint and input inference
   use the engine's scope-variable list, so "what the menu offers" and "what the lint
   accepts" cannot disagree.
2. **Offer only names that will render.** Every variable has an availability state in
   the current scope:
   - `available`: the menu offers it normally.
   - `conditional`: it renders only if a precondition holds, for example `n`/`N` need
     `%repeat`. The menu shows it dimmed, below the available entries, with a hint that
     teaches the fix. The lint still accepts it.
   - `unavailable`: it would fail in this scope. The menu never shows it; hover and the
     lint explain why.
3. **Context decides the list.** Variables appear at expression positions, filters after
   `|`, tests after `is`, members after `<namespace>.`, and statement keywords at the
   head of `{% %}`. Statement keywords put the closers for currently open blocks first.
   New-name positions (`{% set x`, `{% for x`, macro names) and string literals get no
   menu.
4. **The xprompt's own API comes first.** Ranking with an empty prefix is:
   1. block-scoped locals (innermost first)
   2. declared inputs
   3. document-level locals
   4. sase variables
   5. positional/provider variables
   6. Jinja globals

   With a typed prefix, match quality leads. An exact name match always wins.

5. **Inside a tag, Jinja owns completion.** On both surfaces, once the cursor is inside
   `{{ }}` or `{% %}`, no other completion surface (placeholder, directive, xprompt,
   `@`, file history, snippet) may claim it. The engine returns `Some` with possibly
   empty items for any in-tag position, and returns `None` only outside tags.
6. **Beautiful means consistent.** A row's name is styled exactly as the editor's Jinja
   highlighter will color that token once inserted. Rows reuse the arg-name menu's
   column grid and the shared fuzzy match highlight. The LSP uses proper
   `CompletionItemKind`, `labelDetails`, markdown documentation, and `DEPRECATED` tags
   for legacy aliases.

## Variable scope model (authoritative)

`scope` is `prompt` (a top-level agent prompt: TUI agent/snippet panes,
`sase_prompt_*.md` / `sase_ace_prompt_*.md` temp files, `sase`/`sase_prompt` language
ids, and other markdown) or `xprompt` (an xprompt definition body: TUI panes bound to an
xprompt target or a mini-xprompt pane, and LSP documents under `xprompts`/`.xprompts`/
`default_xprompts` directories or memory notes).

| Name(s)                                                                                                                | Type / shape     | Group      | Availability                                                                                                                                                                                         |
| ---------------------------------------------------------------------------------------------------------------------- | ---------------- | ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| declared inputs (`input:` / `inputs:`)                                                                                 | declared type    | input      | available in both scopes                                                                                                                                                                             |
| `root`                                                                                                                 | `str`            | sase       | available in both scopes                                                                                                                                                                             |
| `wait` (members `chats: list[str]`, `artifacts: list[dict]`)                                                           | namespace        | sase (run) | run rule                                                                                                                                                                                             |
| `patch_name`, `workspace_num`                                                                                          | `str`, `int`     | sase (run) | run rule                                                                                                                                                                                             |
| `cl_name`                                                                                                              | `str`            | sase (run) | run rule; legacy alias of `patch_name`                                                                                                                                                               |
| `n`, `N`                                                                                                               | `int`            | sase (run) | run rule, then conditional on `%repeat`/`%r` (available when the directive is in the text in `prompt` scope; always conditional in `xprompt` scope: "only when the launching prompt uses `%repeat`") |
| `agents`                                                                                                               | mapping          | sase (run) | run rule, then conditional on `%wait`/`%w` (same shape as `n`)                                                                                                                                       |
| `wait_chats`                                                                                                           | `list[str]`      | sase (run) | same as `agents`; legacy alias of `wait.chats`                                                                                                                                                       |
| `_args`, `_1` … `_k` (k = max(1, declared input count))                                                                | `list`, any      | positional | `xprompt` scope only; unavailable in `prompt` scope                                                                                                                                                  |
| `provider_name`, `provider_tool_name`, `provider_native_ask_tool`                                                      | `str`            | provider   | `xprompt` scope with truthy frontmatter `skill`; otherwise unavailable                                                                                                                               |
| `range`, `dict`, `lipsum`, `cycler`, `joiner`, `namespace`                                                             | callable         | jinja      | available in both scopes                                                                                                                                                                             |
| template locals (`set`, `for` targets, `loop`, macro names/params, `varargs`/`kwargs`/`caller`, `with`, `import … as`) | inferred/unknown | local      | available where Jinja scoping makes them visible (see the scan phase)                                                                                                                                |

**Run rule:** in `prompt` scope, when the document (inline frontmatter or the supplied
lifted frontmatter) declares at least one input, run-time names are `unavailable`. The
reason is that TUI launch renders such prompts through `render_prompt_with_inputs`
before any agent run exists, so the names are undefined (bug bead `sase-1dd`). Keep this
rule as one clearly commented catalog condition, so fixing `sase-1dd` deletes it.
Otherwise, run-time names are available (or conditional, as above).

Name collisions: the first group in rank order wins (a declared input `n` shadows the
builtin `n`), and the winning item's documentation says "Shadows the sase built-in `n`."

Inert regions (no completion, and `None` from the engine): fenced code, inline code
spans, `%xprompts_enabled:false` zones, `{# #}` comments, `{% raw %}` bodies, and the
document's leading YAML frontmatter block.

## Phase: catalog — Rust Jinja catalog and wire types

Work in sase-core (`sase repo open sase-core -r "<why>"`, read its `AGENTS.md`). Create
the module `crates/sase_core/src/editor/jinja/`: `mod.rs` is a facade with only `mod`
and `pub use` lines, and tests live beside the code. Register it in
`crates/sase_core/src/editor/mod.rs` as `pub mod jinja;`. Do not add names to the root
`pub use` list in `lib.rs`.

- `catalog.rs`: `pub fn jinja_catalog() -> &'static JinjaCatalogWire` (or a cheap clone
  builder). The catalog holds:
  - **Variables**: every sase-injected name in the scope table above, each with `name`,
    `type_label`, `group`, `summary` (one line, suitable for a menu), `documentation` (a
    short paragraph plus an example such as `{{ patch_name }}`), `availability_rule`
    (enum: `always`, `run`, `run_needs_repeat`, `run_needs_wait`, `xprompt_only`,
    `xprompt_skill_only`), `legacy_for: Option<&str>`, and `members` (namespace members
    with type and summary; `wait` → `chats`, `artifacts`). Use `docs/xprompt.md`
    "Template Context" and "Repeat Directive", and `docs/llms.md` provider skill vars,
    for wording.
  - **`loop` members**: `index`, `index0`, `revindex`, `revindex0`, `first`, `last`,
    `length`, `depth`, `depth0`, `previtem`, `nextitem`, `cycle`, `changed`.
  - **Jinja globals**: `range`, `dict`, `lipsum`, `cycler`, `joiner`, `namespace`, each
    with signature and summary.
  - **Filters**:
    - sase filters: `plan_ref_path`, `provider_disabled(mode="any")`,
      `provider_enabled(mode="any")`.
    - Jinja 3.1 built-ins: every identifier-named filter, each with a call signature
      (for example `default(default_value='', boolean=false)`) and a one-line summary.
    - Each filter has a `tier`: `sase`, `common` (`default`, `join`, `length`, `lower`,
      `upper`, `trim`, `replace`, `first`, `last`, `sort`, `unique`, `tojson`,
      `indent`), or `other`.
  - **Tests**: every identifier-named Jinja 3.1 built-in test (`defined`, `undefined`,
    `none`, `string`, `number`, …). Omit operator spellings such as `==` and `>`.
  - **Statements**: `if`, `elif`, `else`, `endif`, `for`, `endfor`, `set`, `endset`,
    `macro`, `endmacro`, `call`, `endcall`, `filter`, `endfilter`, `with`, `endwith`,
    `raw`, `endraw`, `block`, `endblock`, `extends`, `include`, `import`, `from`. Record
    each opener's closer. Exclude `do`, `break`, and `continue`: the prompt environment
    enables no extensions.
- `wire.rs`: serde snake_case types.
  - Requests:
    - `JinjaAssistRequestWire { text, position: EditorPosition, scope: JinjaScopeKind, frontmatter: Option<String> }`
    - `JinjaScopeRequestWire { text, scope, frontmatter }`
  - `JinjaCompletionWire`:
    - `slot`: one of `variable`, `member`, `filter`, `test`, `statement`, `none`
    - `namespace: Option<String>`, `prefix`, `replacement_range: EditorRange`
    - `items: Vec<JinjaCompletionItemWire>`, `shared_extension: String`
  - `JinjaCompletionItemWire`:
    - `name`, `insertion`
    - `kind`: one of `variable`, `member`, `function`, `filter`, `test`, `keyword`
    - `source`: one of `input`, `local`, `sase`, `positional`, `provider`, `jinja`
    - `type_label: Option`, `signature: Option`, `summary: Option`
    - `documentation` (markdown)
    - `required: bool`, `default_display: Option`, `choices: Vec<String>`
    - `availability: { state: available | conditional, hint: Option<String> }`
    - `legacy_for: Option<String>`, `closes: Option<String>`, `shadows: Option<String>`
    - `match_runs: Vec<(u32, u32)>` (character ranges into `name`), `rank: u32`
  - `JinjaScopeVariablesWire { known: Vec<String>, positional_pattern: bool, unavailable: Vec<{ name, reason }> }`
  - `JinjaCatalogWire`: variables, filters, tests, jinja globals, and statements, with
    their docs. Python parity tests use it.
  - Follow the neighbouring editor wires for any schema-version constant. Do not change
    `CompletionContextKind` or any existing wire; this feature is purely additive.
- Tests assert:
  - names are unique within each table;
  - every variable has a summary and documentation;
  - every legacy alias points at an existing name;
  - every opener has a closer.

## Phase: scan — Rust Jinja tag scanner, slot classifier, and scope analysis

Also in sase-core `crates/sase_core/src/editor/jinja/`. Keep each file ≤ 1,500 lines.

- `scan.rs`: tag detection.
  - Reuse the existing literal-zone helpers behind `editor/exclusion.rs` (promote
    `jinja_tag_ranges`-style scanning to `pub(crate)` rather than duplicating it). Keep
    the `%{` alternation carve-out.
  - Mask `{# #}` comments, `{% raw %}…{% endraw %}` bodies, and the leading frontmatter.
  - Support whitespace-control delimiters (`{{-`, `-}}`, `{%-`, `-%}`, `+`).
  - The tag at the cursor is the nearest `{{`/`{%` opener at or before the cursor, with
    no closer between the opener and the cursor. The closer search skips quoted string
    literals. An unclosed opener runs to the next opener or to the end of text, so
    `{{ ro` with no closer still completes, and `@{{ fi` completes as Jinja.
- `context.rs`: slot classification for the identifier token around the cursor.
  - The token is `[A-Za-z0-9_]`; `prefix` is the part before the cursor, and the
    replacement range covers the whole token.
  - `member`: the token follows `<dotted.path>.`.
  - `filter`: the token follows `|`, or is the second word of `{% filter`.
  - `test`: the token follows `is` or `is not`.
  - `statement`: the token is the first word of a `{% %}` tag.
  - `none`:
    - inside string literals;
    - after a digit-leading token;
    - at new-name positions: `{% set <targets>` before `=`, `{% for <targets>` before
      `in`, `{% macro <name>`/params, `{% block <name>`, `import … as <name>`.
  - `variable`: every other position.
  - Model this as
    `pub fn jinja_completion_slot(text, position) -> Option<JinjaSlotContext>`.
- `scope.rs`: document scope.
  - **Inputs.** Read declared inputs from the in-text leading frontmatter and from the
    optional external `frontmatter` string. External comes first; dedupe by name. Accept
    both `input:` and `inputs:`, in shortform and longform. Reuse an existing Rust
    parser: make `diagnostics.rs` `parse_local_inputs`/`frontmatter_mapping` or the
    catalog's `parse_inputs` `pub(crate)` and share it. Do not add a third parser. Keep
    type, description, required (no default), string defaults for display, and choices.
  - **Skill flag.** Read the truthy `skill` flag from either frontmatter source.
  - **Directives.** Detect `%repeat`/`%r` and `%wait`/`%w` with the existing directive
    scanner, outside inert regions and Jinja tags. Do not use a regex.
  - **Template locals.** Collect position-aware locals from statement tags before the
    cursor, tracking an open-block stack that also yields `closes` suggestions:
    - `set` targets (tuple and block forms) after declaration;
    - `for` targets plus `loop`, only inside that for body;
    - macro names after their definition;
    - macro params plus `varargs`, `kwargs`, and `caller` inside the macro body;
    - `with` assignments inside the with body;
    - `import … as` and `from … import a as b` names after the statement.
- Tests cover, in the style of `editor/completion/tests`:
  - the whole slot matrix;
  - the inert regions;
  - unclosed tags and whitespace-control delimiters;
  - `%{a | b}` and `{%if` with no space;
  - multi-line tags;
  - nested for/if stacks;
  - frontmatter input forms, including the `inputs:` alias;
  - directive detection with aliases.

## Phase: assist — Rust Jinja completion, ranking, documentation, hover, and scope variables

In sase-core `editor/jinja/`, add `assist.rs`, `docs.rs`, and `hover.rs` (split as
needed), exported from the facade.

- `pub fn jinja_completion(req: &JinjaAssistRequestWire) -> Option<JinjaCompletionWire>`
  - Returns `None` only when the cursor is outside a tag. In an inert region it also
    returns `None`. For any `none` slot it returns `Some` with empty items.
  - Builds candidates for the slot:
    - variables per the scope table and availability rules;
    - members for known namespaces (`wait`, `loop`); an unknown namespace gives empty
      items;
    - filters ordered by tier;
    - tests;
    - statements: closers for the open-block stack first (innermost first), then
      `elif`/`else` when valid, then openers in catalog order. Omit closers that do not
      match an open block.
  - Drops `unavailable` items, and applies shadowing.
  - Matches with `editor::fuzzy::fuzzy_match` against `name`, and stores `match_runs`.
  - Sorts by:
    1. exact name match first;
    2. fuzzy tier;
    3. `available` before `conditional`;
    4. group order (principle 4);
    5. fuzzy score, descending;
    6. declaration or catalog order.

    Legacy aliases sort directly after their canonical name.

  - Sets `rank` to the final index. `shared_extension` is the common case-insensitive
    extension among prefix-tier matches.

- `docs.rs`: one markdown renderer, shared by completion documentation and hover. It
  renders the bold `name`, the type in backticks, and the source label. After a blank
  line come the summary or description and an example. The last block is optional
  bullets:
  - `Default: …`
  - `Choices: …`
  - a `⚠` conditional hint, e.g. "Only defined under `%repeat` — add `%repeat:N`"
  - "Legacy alias of `patch_name` — prefer `{{ patch_name }}`"
  - "Shadows the sase built-in `n`"
  - `Closes {% for %}`
- `pub fn jinja_hover(req) -> Option<HoverPayload>`: for the identifier under the cursor
  inside a tag, resolve it by slot. It handles a variable, including an `unavailable`
  builtin (show the reason), a known member, a filter, or a test. Return `None` for
  unknown names.
- `pub fn jinja_scope_variables(req: &JinjaScopeRequestWire) -> JinjaScopeVariablesWire`
  - `known`: every variable-slot name that is `available` or `conditional`, ignoring
    locals (the Python lint's Jinja AST already handles locals), plus `loop`.
  - `positional_pattern`: true in `xprompt` scope, meaning any `_<digits>` is known.
  - `unavailable`: each name with its reason.
- Exhaustive unit tests:
  - ordering with and without a prefix;
  - availability in each scope, with and without inputs and directives;
  - shadowing;
  - exact-match promotion;
  - legacy ordering;
  - hover markdown;
  - scope-variable lists.

## Phase: bindings — Python bindings for the Jinja engine

In sase-core `crates/sase_core_py/src/editor_completion/`, following the
`placeholder_completion` binding and the `AGENTS.md` recipe:

- `jinja_completion(request: dict) -> dict | None`
- `jinja_scope_variables(request: dict) -> dict`
- `jinja_catalog() -> dict`

Register all three in `register_editor_completion` (the compiler does not check this),
and add round-trip tests in the domain's `tests/`. Run `sase tool run check` in
sase-core.

## Phase: lsp — sase-xprompt-lsp Jinja completion and hover

In sase-core `crates/sase_xprompt_lsp`:

- **Scope.** Thread the document's source path into completion and hover, the same way
  diagnostics use `DocumentSnapshot::with_source_path`. Derive `JinjaScopeKind`:
  - xprompt-directory paths and memory notes → `xprompt`;
  - prompt temp files, `sase`/`sase_prompt` language ids, and all other eligible
    markdown → `prompt`;
  - `gitcommit` → no Jinja completion.
- **Early path.** At the top of `completion_for_text_with_trigger`
  (`server/completion.rs`), before the placeholder check, call
  `sase_core::editor::jinja::jinja_completion`. Import by module path. On `Some`, return
  a Jinja response, even an empty one, and never fall through. This also fixes the
  `{{ a < b` placeholder misread and `{%if` directive misreads.
- **Triggers.** Add `{` and `|` to `trigger_characters` (`server/mod.rs`). When a
  completion request was triggered by `{` or `|` and the cursor is not inside a tag,
  return `None`, so literal braces and alternation pipes never pop a menu. This requires
  threading the trigger character, not just the trigger kind. Update the
  trigger-character tests in `server/tests/shortcuts.rs`.
- **Items.** Add `jinja_completion_response` in `lsp_convert.rs`:
  - `label` = name.
  - `kind`: variable → `VARIABLE`, member → `FIELD`, jinja-global function and filter →
    `FUNCTION`, test → `FUNCTION`, keyword → `KEYWORD`.
  - `label_details.detail` = ` <type>` or the filter signature.
  - `label_details.description` = source label plus state (`input · required`, `sase`,
    `needs %repeat`, `local`, `jinja`, `closes for`).
  - `detail` = summary; `documentation` = engine markdown.
  - `filter_text` = name; `sort_text` = `{rank:04}`.
  - `text_edit` over `replacement_range`.
  - `tags: [DEPRECATED]` when `legacy_for` is set.
  - `preselect` on rank 0.
- **Hover.** Try `jinja_hover` first in the hover path (`server/actions.rs`).
- **Tests.**
  - Server unit tests with the `support.rs` helpers:
    - frontmatter inputs listed first;
    - builtins, including the conditional `n` with and without `%repeat`;
    - run-time names hidden in an input-declaring prompt;
    - the xprompt path offers `_args`;
    - skill frontmatter offers provider vars;
    - filters after `|`, members after `wait.`;
    - statement closers inside a for loop;
    - inert regions;
    - `gitcommit` gives no Jinja completion;
    - the `{{ a < b` regression;
    - hover markdown.
  - A stdio test in `tests/jsonrpc_stdio.rs` style: open a `sase_prompt_*.md`, then
    complete after `{{ ` and after a `{` trigger outside any tag, which returns null.
- **Docs.** Open the sase repo docs from this same sase workspace and add Jinja
  completion/hover and the new trigger characters to `docs/editor.md` "LSP Features".
  The declaration commits both repos; the host pins the core revision automatically.
- Run `sase tool run check` in sase-core and `sase tool run check` in sase.

## Phase: python — Python adapter, single source of truth, lint, and parity tests

In sase:

- **Adapter.** Add `src/sase/xprompt/jinja_assist.py`, modeled on
  `src/sase/xprompt/placeholder_completion.py`. It provides:
  - frozen dataclasses:
    `JinjaScope(kind: Literal["prompt", "xprompt"], frontmatter: str | None)`,
    `JinjaCompletion`, `JinjaCompletionItem`, `JinjaScopeVariables`, `JinjaCatalog`;
  - `jinja_completion(text, cursor_offset, scope)` and
    `jinja_scope_variables(text, scope)`;
  - `jinja_catalog()`, `functools.cache`d.

  Convert character offsets ↔ `EditorPosition` the same way the placeholder completion
  widget does. Decode forward-compatibly: an unknown `source`/`kind`/`state` degrades to
  a safe default and never raises.

- **Core pin.** Move `sase-core-revision.txt` past the bindings commit
  (`just ratchet-core-revision`). Rebuild the local core so tests see the new bindings,
  per `docs/rust_backend.md`. Satisfy `tools/check_sase_core_rs_bindings`.
- **Single source of truth.**
  - Delete `BUILTIN_RUNTIME_NAMES` and `RESERVED_GLOBAL_NAMES`
    (`src/sase/xprompt/_jinja.py`).
  - Delete `RUNTIME_NAMESPACE_MEMBERS`, `builtin_runtime_member_names`,
    `completion_context`, and `JinjaCompletionContext`
    (`src/sase/xprompt/jinja_inspect.py`) once nothing uses them.
  - `known_toplevel_context()`/`builtin_runtime_names()` either go away or become thin
    wrappers over `jinja_catalog()`. `inspect_template(known=None)` defaults to the
    engine's prompt-scope known set.
  - Update the `sase.xprompt.__init__` exports, and keep symvision green.
- **Lint.** In `src/sase/ace/tui/widgets/_jinja_diagnostics.py`, replace
  `_known_jinja_names_for_prompt` with `jinja_scope_variables(text, scope)`, using the
  pane's scope (see the tui-menu phase for the scope accessor; add it here if this phase
  needs it first). Honor `positional_pattern`. Unknown-variable output names each
  `unavailable` name with the engine's reason instead of calling it unknown. Update the
  tests that monkeypatch `known_toplevel_context`.
- **Input inference.** gL local-xprompt conversion
  (`src/sase/ace/tui/widgets/_local_xprompt_conversion.py`) and save-as-xprompt input
  inference (`src/sase/ace/tui/actions/agent_workflow/_prompt_bar_save_xprompt.py`) pass
  `jinja_scope_variables(body, JinjaScope("xprompt", None))` as the known set. Builtins
  such as `wait`, `patch_name`, and `n` then never become inferred inputs. Add
  regression tests.
- **Parity tests.** These are new tests; keep each test file small.
  - `jinja_catalog()` filters equal the identifier-named keys of
    `get_jinja_env().filters`. The same holds for tests versus `.tests`, and Jinja
    globals versus `.globals`.
  - The workflow environment in `workflow_executor_utils.py` has no filter outside the
    catalog.
  - The prompt environment enables no extensions, so the statement list is valid.
  - Every name `_build_named_args` can emit is a catalog variable, and every catalog
    `run` variable is emitted by some path. Drive `%repeat` env, wait chats, and
    output-variable fixtures, reusing the existing run-agent-exec test fixtures. Skip
    `__`-internal names. Also cover `wait` from the runtime binding and `root` from
    `get_global_template_vars`.
  - `patch_name` is now known and no longer linted.
- **Docs.** In `docs/xprompt.md` "Template Context", add `patch_name`, `workspace_num`,
  `n`/`N` (linking to the repeat section), `cl_name` (legacy), the provider skill vars,
  and the Jinja globals. Add a short availability note that covers the input-declaring
  prompt limitation, and a pointer to editor completion.
- Run `just fmt`, then `sase tool run check`.

## Phase: tui-menu — TUI Jinja completion menu redesign

Read the `tui.md` and `tui_perf.md` memory notes first. In sase:

- **Scope accessor.** Add a public
  `PromptInputBar.jinja_scope_for_text_area(text_area) -> JinjaScope` in
  `_prompt_input_bar_frontmatter.py`, built on `_frontmatter_scope`:
  - a mini-xprompt pane → `xprompt` with the pane's own frontmatter;
  - a stack bound to an xprompt target (`xprompt_target()` or the read-only target) →
    `xprompt` with the stack frontmatter;
  - any other prompt-mode pane → `prompt` with the stack frontmatter;
  - feedback and approve modes → `prompt` without frontmatter.
- **Engine-backed results.** Rewrite `src/sase/ace/tui/widgets/jinja_completion.py` so
  `build_jinja_completion_result(text, cursor_offset, scope)` returns engine items.
  Carry a richer `JinjaCompletionMetadata`: kind, source, type label or signature,
  required, default, availability, hint, legacy target, closes, shadows, match runs, and
  summary.
  - Call sites that must pass the scope: the Ctrl+T handler in
    `_file_completion_tab.py`, live refresh in `_file_completion_refresh.py`, and the
    soft-completion worker (`prompt_completion.py`, `_prompt_soft_completion.py`).
  - Compute the scope on the UI thread and hand it to the worker; the worker only has
    text.
  - Keystroke paths stay pure and read-only (`tui_perf.md` rule 11).
- **Precedence.** When the cursor is inside a Jinja tag, the Jinja branch runs before
  the placeholder, VCS, directive, xprompt-arg, model-shortcut, `@`, and `#` branches in
  both the Ctrl+T dispatch and the auto path. An in-tag `slot: none` claims the cursor
  with no menu.
- **Rows.** Replace `append_jinja_completion_row` in
  `_prompt_input_bar_completion_rows_simple.py` with an aligned column grid in the style
  of the xprompt arg-name menu:
  - **Name:** styled exactly like `_jinja_highlight.py` styles that token kind
    (variable, filter, keyword), using theme tokens, not hard-coded Rich colors. Fuzzy
    runs use `_completion_match_highlight.append_highlighted`.
  - **Type or signature:** dim.
  - **Source badge:** a fixed-width chip with a distinct theme color. Chips are `input`,
    `local`, `loop`, `sase`, `arg`, `skill`, `jinja`, `%repeat`/`%wait` (conditional, in
    the warning color), `legacy`, and `closes for`.
  - **Default:** `=default` for optional inputs, using `input_default_style`.
  - **Description:** dim and ellipsis-truncated to the width.
  - Conditional and legacy rows render dim. Keep the `▸` selection marker, the 8-row
    budget, and the `↓ N more…` line.
- **Title and subtitle.**
  - The border title names the slot: `{{ variables`, `| filters`, `is tests`,
    `{% statements`, or `wait. members`. Append the scope label when it has one, e.g.
    `· #research`.
  - The subtitle shows the selected row in full: the summary, plus the
    required/default/choices details for inputs, the `⚠` hint for conditional rows, or
    `legacy → patch_name`.
  - Wire both in `_prompt_input_bar_completion_panel_labels.py` and
    `_prompt_input_bar_completion_panel.py`.
- **Accept.** Accept replaces the whole identifier with the bare name; namespaces such
  as `wait` insert only `wait`. Ctrl+T keeps its single-candidate and shared-extension
  behavior.
- **Tests.**
  - Pilot tests in a new file next to `tests/ace/tui/widgets/test_prompt_jinja.py`:
    - stack-frontmatter inputs listed first;
    - the mini pane offers its own inputs plus `_args`;
    - a conditional `n` renders dim with a hint;
    - an input-declaring prompt hides `patch_name`;
    - `|` gives filters, `wait.` gives members, `{% ` inside a for puts `endfor` first;
    - the scope label appears in the title;
    - in-tag precedence over `<` placeholders and `@`;
    - a fenced block opens no menu.
  - PNG goldens `prompt_jinja_variable_completion_{dark,light}_120x40` and
    `prompt_jinja_filter_completion_120x40`, following
    `tests/ace/tui/visual/test_ace_png_snapshots_xprompt_arg_completion.py`. Use a
    frontmatter stack with inputs so the grid shows inputs, sase, conditional, and jinja
    rows. Capture them with `just fix-tui-screenshots -- <selectors>` through
    `/sase_monitor`, and inspect the report and each golden.
- Run `sase tool run check`.

## Phase: tui-auto — Auto-open the Jinja menu while typing

In sase:

- **Setting.** Add `auto_jinja_menu: true` under `ace.prompt_completion` in
  `src/sase/default_config.yml`, and wire it through `PromptCompletionSettings`
  (`prompt_completion.py`), `src/sase/config/sase.schema.json`,
  `tests/test_config_schema_ace.py`, and `docs/configuration.md`.
- **Triggers.** Open automatically in prompt mode when the setting is on:
  - right after `_try_jinja_auto_pair` rewrites `{{`→`{{ | }}` or `{%`→`{% | %}` (never
    for `{#`);
  - when `|` is typed inside an expression;
  - when `.` is typed after a namespace that has members;
  - when an identifier character is typed at a completable slot while no menu is open.
    Mirror the `#` auto-menu semantics in `_try_auto_prompt_reference_completion`
    (`_file_completion_open.py`). The menu closes when the cursor leaves the tag, on
    accept, or when nothing matches, as the refresh already does.
- **No error flash.** Because the menu opens before the 90 ms diagnostics debounce, the
  empty `{{  }}` never flashes the red diagnostics panel; the open menu already
  suppresses it. Add a pilot test that proves it.
- **Alternation `|`.** Make sure the alternation-separator normalization in
  `_try_prompt_text_pair_edit` never rewrites a `|` inside a Jinja tag, while `%{a | b}`
  behavior is unchanged. Add tests for both.
- **Tests.** Setting off → no auto-open, but Ctrl+T still works. Typing `{{ pa` narrows
  live to `patch_name`. Existing prompt goldens stay unchanged, or their updates are
  inspected and justified.
- **Docs.** Update `docs/ace.md` "Prompt Input Widget" → "Completion" with the Jinja
  menu (triggers, slots, badges, scope, setting).
- Run `sase tool run check`, then refresh any affected goldens as in the tui-menu phase.

## Phase: parity — TUI and LSP Jinja completion parity suite

In sase, add a parity test using the existing LSP session harness
(`tests/_xprompt_directive_completion_parity_lsp_session.py` and its protocol/rows
helpers). For each shared fixture and cursor, run the `sase-xprompt-lsp` binary and
`jinja_assist.jinja_completion`, and assert identical ordered names, kinds,
source/availability labels, and documentation.

Fixtures:

- a prompt with frontmatter inputs, `%repeat`, and `set`/`for` locals, with cursors
  after `{{ `, `{{ pa`, `| `, `is `, `wait.`, and `{% ` inside a for;
- an input-declaring prompt, where run-time names are hidden;
- an xprompt-directory path with `skill: true`.

For the lifted-frontmatter case, the adapter gets `frontmatter=<yaml>` plus the body,
while the LSP gets the full document; the items must still match. Add one TUI pilot that
asserts the rendered menu rows equal the adapter's item order. Run
`sase tool run check`.

## Non-goals and follow-ups (record as PROPOSED FOLLOW-UP notes, not beads)

- Completion inside local-xprompt bodies under frontmatter `xprompts:`. That covers both
  the TUI frontmatter-panel editor, which has no completion machinery today, and LSP
  positions inside the frontmatter block.
- Workflow YAML step templates (step outputs, `for:` mapping vars).
- `agents.<name>` member completion from `%wait` targets, and `wait.artifacts[i]`
  fields.
- LSP Jinja syntax/unknown-variable diagnostics and Jinja semantic tokens.
- Telling an xprompt-bound TUI stack's Ctrl+G temp file apart from a prompt temp file,
  so the LSP can use `xprompt` scope for it.
- Fixing `sase-1dd`. When it lands, delete the catalog's input-declaring-prompt run
  rule.
- No sase-nvim change is expected, because triggers are server-advertised. Verify
  manually once and note any client gaps.

## Risks

- **`{` and `|` triggers.** These could pop menus on ordinary text. Mitigation: return
  `None` outside tags, with tests.
- **Naive tag scanning.** It can mis-scope on `}}` inside strings. Mitigation: the
  closer search is string-literal aware.
- **Python/Rust drift.** Mitigation: the Python parity suite runs in sase CI against the
  pinned core. A new Jinja release or a new injected name fails loudly.
- **Latency.** The engine runs on every keystroke. Mitigation: it is pure and linear in
  the text. The adapter may cache the parsed external frontmatter by string, but the
  Rust call itself is cheap. Verify with `SASE_TUI_TRACE=1` if typing feels slower.
