---
tier: epic
title: '#{project} snippet variable'
goal: 'Snippet templates can write #{project}, and it expands to the display name
  of the project the prompt targets, consistently in sase''s TUI and in LSP editors.
  The user''s chezmoi `epic` and `bd` snippets then emit the correct bead-ID prefix
  in every project.

  '
phases:
- id: core-snippet-vars
  title: Core substitution helper and LSP support
  depends_on: []
  size: medium
  description: 'core-snippet-vars: in sase-core, add the snippet_variables substitution
    helper, an additive `variables` field on the snippet-session Plan event, and LSP
    snippet-completion substitution resolved from the document''s leading project
    tag or VCS ref, then the catalog''s current project, with tests.'
- id: tui-snippet-vars
  title: TUI resolution, CI pin, and docs
  depends_on:
  - core-snippet-vars
  size: medium
  description: 'tui-snippet-vars: in sase, ratchet the core pin, thread `variables`
    through the snippet-session facade, and resolve #{project} on TUI Tab expansion
    (the prompt target, then the cached current project, else verbatim with a warning)
    using no keystroke-path I/O. Add tests and document it in ace.md and editor.md.'
- id: chezmoi-epic-snippet
  title: Switch the chezmoi epic and bd snippets
  depends_on:
  - tui-snippet-vars
  size: xsmall
  description: 'chezmoi-epic-snippet: in the chezmoi repo''s sase.yml, change the
    `epic` and `bd` snippets from the hard-coded sase- prefix to #{project}-, verify
    them with sase snippet show, and apply chezmoi after the commit lands.'
proposed_by: bbugyi200.apollo.2a
create_time: 2026-09-27 08:18:49
status: done
bead_id: sase-1b6
---

- **PROMPT:** [prompts/202609/snippet_project_variable.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/snippet_project_variable.md)
- **BEAD:** [sase-1b6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1b6/README.md)

# `#{project}` Snippet Variable

## Goal

Let sase snippet templates reference the name of the project a prompt targets with a new
`#{project}` token. The first use is the user's `epic` snippet, defined in the chezmoi
repo's `home/dot_config/sase/sase.yml`. Its template changes from
`the sase-$1 epic bead` to `the #{project}-$1 epic bead`, so bead IDs come out right in
every project (`sase-…`, `bob-cli-…`, …).

## Strategy Review: Adjustments To The Request

The request is sound, and `#{project}` is kept as the syntax. It fits the existing `#`
sigil family: `#name` is an xprompt and `#[trigger]` is a snippet call. The adjustments
below matter:

1. **Resolve at expansion time, not at catalog-load time.** The TUI loads one snippet
   catalog per process with no project (`load_snippet_catalog()` in
   `src/sase/ace/tui/prompt_catalog.py`). The LSP caches catalogs by editor root dir.
   One prompt bar can target any project. So `#{project}` stays literal through catalog
   composition (the Rust composer only scans `#[`), and each surface substitutes it at
   the moment a snippet is inserted. `#[epic]`-style composition (for example the user's
   `repic` snippet) therefore inherits the variable for free.
2. **"Current project" means the prompt's target project first.** In the SASE glossary,
   the _current project_ (MRU-derived, `sase project current`, the TUI top-bar
   `+<project>` chip) supplies defaults and never overrides an explicit choice. So
   `#{project}` resolves in this order:
   1. the prompt's own explicit target (a leading `+<project>` tag or `#gh:`/`#git:` VCS
      ref, or, in the TUI, a non-home prompt context),
   2. then the SASE current project,
   3. otherwise it stays unresolved.

   The working directory is never consulted. This matches the glossary rule that the
   working directory never makes a project current. It also avoids the LSP's
   root-basename guess (for example `bryan` when nvim is started from `~`). Note:
   untagged prompts _launch_ in `home`. Falling back to the current project rather than
   `home` is deliberate. It matches the literal request ("the current project's name")
   and the `Space` prefill habit, where the current project's tag is what a prompt
   usually starts with.

3. **Value = the project's display name** (`PROJECT_NAME`, for example `sase` or
   `bob-cli`), not its directory key (`gh_sase-org__sase`). This is also the default
   bead issue prefix (`src/sase/bead/prefix_policy.py`), and both existing bead stores
   match it. Caveat: a bead store _can_ carry a customized `issue_prefix`. If one ever
   diverges, add a separate `#{bead_prefix}` variable on the same machinery rather than
   overloading `#{project}` (out of scope).
4. **Unresolved or unknown variables pass through verbatim.** This mirrors how an
   unresolved `#[missing]` call stays literal. Only a known variable with a resolved
   value is replaced; any other `#{name}` (for example Ruby's `"#{x}"`) is untouched.
   There is no escape syntax (not needed until a literal `#{project}` must coexist with
   a resolvable one). The TUI shows a warning toast when a snippet it just expanded
   still contains an unresolved `#{project}`.
5. **The substitution grammar lives in sase-core.** The TUI and the LSP (nvim and other
   editors) must agree. Per the Rust core boundary rule, the `#{name}` scanner and
   substitution are a `sase_core` function. Each frontend only picks the project value
   from state it already has, reusing its existing "which project does this prompt
   target" resolver.
6. **Also convert the `bd` snippet.** `bd: the sase-$1 bead` has the identical
   hard-coded prefix, so it becomes `the #{project}-$1 bead`. `rep`
   (`the ../sase-$1 repo`) is **not** changed, because it names sibling repo
   directories, not the project.

Rejected alternatives:

- Launch-time prompt variable expansion: the user cannot see the result while editing,
  untagged prompts would yield `home`, and it widens the collision surface to all prompt
  text.
- Per-project snippet overrides in each project's `sase/sase.yml`: the TUI catalog does
  not follow the prompt's target project, and the snippet would be duplicated per
  project.
- `${project}`-style syntax: it collides with shell `${VAR}` in prompts and with the
  `$N` tabstop namespace.

## Phase Overview

1. `core-snippet-vars` (sase-core): the Rust substitution helper, the `Plan`-event
   `variables` field, and LSP completion-time substitution.
2. `tui-snippet-vars` (sase): move the CI core pin, pass variables through the Python
   facade, resolve the project on TUI snippet expansion, add tests, and update docs.
   Depends on phase 1.
3. `chezmoi-epic-snippet` (chezmoi): switch the `epic` and `bd` snippets to
   `#{project}-`. Depends on phase 2, so the user's snippet never expands to a literal
   token on a sase build that lacks support.

## Phases

### Phase 1: Core substitution helper and LSP support

- **id:** `core-snippet-vars`
- **size:** medium
- **depends_on:** []
- **repo:** sase-core (open with `sase repo open sase-core`; read its `AGENTS.md`)

Work:

1. Add a flat top-level module `crates/sase_core/src/snippet_variables.rs` (with a
   module doc comment so `just modules` lists it) and register it with
   `pub mod snippet_variables;` in `crates/sase_core/src/lib.rs`. Do **not** add names
   to the root `pub use` list. Contents:
   - `pub const PROJECT_SNIPPET_VARIABLE: &str = "project";`
   - `pub fn substitute_snippet_variables(template: &str, variables: &BTreeMap<String, String>) -> String`.
     It scans for `#{<name>}` where `<name>` matches `[A-Za-z_][A-Za-z0-9_]*`, with no
     whitespace and no left-boundary requirement, so `foo/#{project}/` works. It
     replaces only names present in `variables`, and escapes `$` in the inserted value
     as `\$` (mirroring `escape_snippet_arg` in `snippet_catalog.rs`) so the tabstop
     scanner treats values literally. Everything else, including unknown names,
     malformed tokens (`#{`, `#{}`, `#{a b}`), and `#[...]` calls, is left untouched. It
     returns the input unchanged when the map is empty or the template has no `#{`.
   - Unit tests in the same file (a `#[cfg(test)] mod tests`): a known variable, an
     unknown variable left verbatim, an empty map, multiple occurrences, adjacency to
     `-$1` (for example `#{project}-$1` → `sase-$1`, tabstop intact), `$` escaping in
     the value, malformed tokens, UTF-8 text around tokens, and coexistence with
     `#[call]`.
2. `crates/sase_core/src/snippet_session.rs`: add
   `#[serde(default)] variables: BTreeMap<String, String>` to
   `SnippetSessionEvent::Plan`. `apply_session_event` substitutes the template via
   `substitute_snippet_variables` before `plan_snippet_expansion`. This change is
   additive: older payloads without the field still deserialize, and older cores ignore
   it. Update the existing `Plan { .. }` constructions in that file's tests. Add a test
   showing that a `Plan` event with `variables = {project: sase}` yields `the sase-`
   text and the `$1` offset right after the dash. Also add a binding round-trip test
   beside `apply_snippet_session_event_binding_drives_nesting_through_dicts` in
   `crates/sase_core_py/src/editor_completion/tests/snippets.rs` that passes a
   `variables` dict.
3. LSP (`crates/sase_xprompt_lsp`): in the `CompletionContextKind::SnippetTrigger`
   branch of `server/completion.rs`, the `vcs_catalog` is already loaded above it.
   Resolve the document's snippet project with a new helper in `server/catalogs.rs` (for
   example `active_snippet_project(document, vcs_catalog) -> Option<String>`):
   - first
     `leading_vcs_project(document.text(), &vcs_catalog.entries, &vcs_catalog.project_tags)`,
   - then the project-row entry whose `current == Some(true)` (use its `name`),
   - and **no** `config.project` root-basename fallback (see "Strategy Review" item 2).

   Build `{"project": name}` when resolved, and apply `substitute_snippet_variables` to
   each snippet candidate's insertion template before `sase_snippet_items` converts it
   with `sase_template_to_lsp_snippet`. Add tests in `server/tests/snippets.rs` for:
   - a `+tag`-led document,
   - a `#gh:`-led document,
   - no tag with a current entry in the VCS catalog fixture,
   - no tag and no current entry (the item text keeps a literal `#{project}`,
     LSP-escaped as today).

4. Commit subject: Conventional Commits `feat(snippets): …`. This is not a breaking
   change: the wire change is additive and adds no binding rename.
5. Verify with `sase tool run check` (the guarded `just check`; allow 10+ minutes). A
   targeted `just test -p …` run does not replace it.

### Phase 2: TUI resolution, CI pin, and docs

- **id:** `tui-snippet-vars`
- **size:** medium
- **depends_on:** [`core-snippet-vars`]
- **repo:** sase

Read the `tui` and `tui_perf` reference memories first, because this change runs on the
prompt keystroke path.

Work:

1. Move sase's CI core pin past phase 1's commit with `just ratchet-core-revision`
   (`sase-core-revision.txt`). Rebuild the local `sase_core_rs` from the updated linked
   sase-core checkout (`just rust-install`) so local tests exercise the new `Plan`
   field.
2. `src/sase/core/snippet_session_facade.py`: add a keyword-only
   `variables: Mapping[str, str] | None = None` parameter to `plan_snippet_expansion`.
   Include `"variables": dict(variables)` in the `plan` event only when non-empty. Add
   cases to `tests/test_core_snippet_session_facade.py`.
3. `src/sase/ace/tui/widgets/_snippets.py`:
   - `_expand_snippet_template_at_range` takes an optional `variables` mapping and
     forwards it to `plan_snippet_expansion`.
   - `_try_expand_snippet` (the snippet-registry Tab path) builds variables only when
     `"#{" in template`. This is a cheap guard, so ordinary snippets pay nothing.
   - Add a method such as `_snippet_project_name() -> str | None`:
     1. `self._xprompt_arg_assist_project_from_text()`. This existing resolver
        (`_xprompt_arg_hints.py`) handles the leading `+tag` or `#` VCS ref and then a
        non-home prompt context, and already returns the canonical display name.
     2. Otherwise, the cached current project from the app's `LaunchContextSource`
        (`#launch-context-source`, `state.project_snapshot.project.display_name`).
     3. Otherwise `None`.

     There must be no disk I/O, subprocess, or catalog build on this path. It only reads
     already-warm in-memory state, and all lookups degrade to `None` on exception.
     Declare the borrowed method stub under `TYPE_CHECKING` like the existing ones.

   - When the expanded text still contains an unresolved `#{project}`, show a one-line
     warning toast that suggests adding a `+<project>` tag.
   - Audit any other path that inserts a _snippet-registry_ template (grep
     `_get_snippets` / `get_snippets`) and route it through the same variables logic.
     Xprompt skeleton and directive-template callers of
     `_expand_snippet_template_at_range` stay unchanged.

4. Tests in `tests/ace/tui/widgets/test_prompt_snippet_expansion.py` (follow its
   existing harness):
   - a `+sase`-led prompt expands `epic` to `the sase-` with the cursor at `$1`,
   - a non-home prompt context is used when there is no tag,
   - no tag falls back to a stubbed current project,
   - nothing resolved leaves `#{project}` verbatim and emits the warning,
   - a composed `#[epic]` caller is substituted too,
   - an unknown `#{foo}` stays verbatim,
   - a template without `#{` never calls the resolver (spy).
5. Docs:
   - `docs/ace.md` "Snippets": add a "Snippet variables" subsection covering the syntax,
     the `project` variable, the resolution order, the display-name value, pass-through,
     no escape syntax, capitalized aliases not uppercasing substituted values, and
     composition inheritance.
   - `docs/editor.md` "Authoring Snippets": a short paragraph on `#{project}` and LSP
     resolution (leading tag or VCS ref, then the current project).
6. Run `just check` (via `sase tool run check` per the lint_and_test memory). Do not run
   `just check-full`.

### Phase 3: Switch the chezmoi `epic` and `bd` snippets

- **id:** `chezmoi-epic-snippet`
- **size:** xsmall
- **depends_on:** [`tui-snippet-vars`]
- **repo:** chezmoi (open with `sase repo open chezmoi`; read its `AGENTS.md`)

Work:

1. In `home/dot_config/sase/sase.yml`, under `ace.snippets` (inside the keep-sorted
   block):
   - change `epic` from `the sase-$1 epic bead` to `the #{project}-$1 epic bead`,
   - change `bd` from `the sase-$1 bead` to `the #{project}-$1 bead`.

   Leave `rep` and every other snippet unchanged. `repic` picks up the change through
   `#[epic]`.

2. Confirm with `sase snippet show epic` and `sase snippet show repic` that the
   templates load without diagnostics.
3. Follow the chezmoi repo's "Chezmoi Apply After Commit" gotcha
   (`chezmoi update -a --force` once the commit lands).

## Acceptance

- Typing `epic<Tab>` in a TUI prompt starting with `+sase` inserts `the sase-` with the
  cursor before ` epic bead`. With `+bob-cli`, it inserts `the bob-cli-`. With no tag,
  it uses the TUI's current-project chip value.
- The same snippet completed through `sase lsp` in an editor yields the same text for a
  `+tag`-led or `#gh:`-led document.
- Snippets without `#{` behave exactly as before, with no extra work on the Tab path.
- `sase tool run check` passes in sase-core, and `just check` passes in sase.
