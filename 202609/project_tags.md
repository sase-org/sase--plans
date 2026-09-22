---
tier: epic
title: Xprompt project tags (+sase)
goal: 'Prompts name their project with a `+<project>` project tag by default. SASE
  resolves the tag to the project''s VCS type and runs the same `#gh:`/`#git:` workflow
  as before. Every completion surface (TUI, LSP/Neovim, shell) inserts tags. Every
  surface that shows raw prompts renders tags in the project''s accent color. Project
  names are enforced unique, case-insensitively, across all VCS types.

  '
phases:
- id: core-tags
  title: sase-core project tag lexer, resolver, expander, and bindings
  depends_on: []
  size: medium
  description: 'core-tags: add the sase_core project_tag module (tag scanning with
    literal-zone skipping, anchored-position detection, case-insensitive resolution
    with suggestions, in-place expansion to VCS refs, completion trigger and selection-apply),
    the v5 catalog wire types, and Python bindings.

    '
- id: unique-names
  title: Case-insensitive project name uniqueness across VCS types
  depends_on: []
  size: small
  description: 'unique-names: make every write-time project ref check case-insensitive
    and reserve home. Widen doctor project.name_collisions to every key/name/alias
    conflict, and mirror the rule in sase-core collision warnings and sase-github
    name allocation. Then verify this machine has no collisions.

    '
- id: backend
  title: Python project tag backend and launch integration
  depends_on:
  - core-tags
  - unique-names
  size: medium
  description: 'backend: shared accent module, cached project tag catalog, expansion
    inside prompt canonicalization and launch_query, launch-unit validation, tag-aware
    helpers for raw-text consumers, generators that default to tags, and the v5 LSP
    catalog payload. Also moves the core pin.

    '
- id: lsp
  title: sase-xprompt-lsp project tag support
  depends_on:
  - core-tags
  size: medium
  description: 'lsp: tag completion with in-place edits, semantic tokens with accent
    modifiers plus a palette capability, hover, diagnostics, quick fixes and a rewrite-to-tag
    code action, and leading-project detection that understands tags.

    '
- id: tui-editor
  title: TUI prompt editor completion and tag defaults
  depends_on:
  - backend
  size: medium
  description: 'tui-editor: the + trigger through the core binding, in-place accept,
    accent-styled completion rows ordered current-project-first, tag prefills and
    MRU cycling, editor project context from tags, pre-submit tag validation, and
    shell completion for +.

    '
- id: tag-display
  title: Tag rendering in the agent panel and prompt editor
  depends_on:
  - tui-editor
  size: medium
  description: 'tag-display: the humanizer renders project refs as tags; the shared
    tokenizer emits project_tag spans; accent styles in both highlight systems; the
    prompt editor, the AGENT XPROMPT sections (main, family, hint, and clan), and
    their visual snapshots.

    '
- id: tag-surfaces
  title: Accent-colored tags on every remaining raw-prompt surface
  depends_on:
  - tag-display
  size: medium
  description: 'tag-surfaces: prompt history, stash, launch approval, runners/revive/run-log
    previews, the metadata pager, the ACE query +project shorthand, CLI prompt output,
    and the sase project list/show TAG column and JSON fields.

    '
- id: nvim
  title: sase-nvim project tag highlighting and picker
  depends_on:
  - lsp
  - backend
  size: small
  description: 'nvim: accent highlight groups built from the server palette, a semantic-token
    handler for saseProjectTag, + support in the Ctrl+T picker, README, and smoke
    tests.

    '
- id: plugins
  title: sase-telegram and sase-github tag adoption
  depends_on:
  - tag-display
  size: small
  description: 'plugins: Telegram project-context capture and copy-text buttons use
    tags, its inbound docs and tests are updated, and sase-github docs present +<project>
    as the default.

    '
- id: docs-memory
  title: Docs, skills, memory, config, and machine verification
  depends_on:
  - unique-names
  - tui-editor
  - tag-surfaces
  - nvim
  - plugins
  size: medium
  description: 'docs-memory: docs and CLI help move to +<project>; fix the sase_run
    skill example; add the Project Tag glossary strand and update the sase-project
    and xprompts notes; drop redundant chezmoi gh_* xprompt shortcuts; run the final
    doctor verification on this machine.'
proposed_by: bbugyi200.athena.0pl
create_time: 2026-09-22 18:48:42
status: wip
bead_id: sase-16n
---

- **PROMPT:** [prompts/202609/project_tags.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/project_tags.md)
- **BEAD:** [sase-16n](https://github.com/sase-org/sase--beads/blob/main/pages/sase-16n/README.md)

# Plan: Xprompt project tags (`+sase`)

## Why

Prompts pick a project with a VCS workflow call such as `#gh:sase` or `#git:notes`. That
forces the user to remember each project's VCS type. It also looks different from how
SASE already shows a project everywhere else: the top-right `+sase` chip, the Agents-tab
`+project` filter shorthand, and the `+query` launch picker.

A project tag makes the prompt match the chip. You type `+sase` and SASE works out that
`sase` is a GitHub project. The launch then runs exactly the same `#gh:sase` workflow it
runs today.

```text
+sase %auto #pr:tag_parsing  Teach the lexer about project tags.
%m:opus +bob-cli  Why does `bob query` hang on empty vaults?
%{+sase | +bob-cli} Audit the README for stale commands.     # one agent per project
```

## Design contract (every phase implements against this)

### D1. Grammar

- **Shape.** A project tag is `+` immediately followed by a _tag name_. The name matches
  `[A-Za-z](?:[A-Za-z0-9_.-]*[A-Za-z0-9_])?`: it starts with a letter and never ends in
  `.` or `-`.
- **Left boundary.** The `+` must be at the start of the text, or directly after
  whitespace, `{`, or `|`. The `{` and `|` cases let tags work inside `%{a | b}` alt
  groups.
- **Right boundary.** The name must be followed by the end of the text, whitespace, `|`,
  or `}`.
- **Result: a tag is a standalone word.** None of these are tags: `chmod +x file` (the
  name `x` never resolves, see D3), `C++`, `a+b`, `+1`, `+ item`, `+sase,`, and `#name+`
  (the existing `:true` shorthand). The same goes for `%dir+`.
- **Literal zones.** Tags inside fenced code, inline code,
  `%xprompts_enabled:false … :true` regions, or a prompt segment's leading YAML
  frontmatter block are inert. Reuse the Rust literal-zone scanner
  (`prompt_literal_zone_ranges`).
- **Not in the grammar.** Names that can't be written as tags (say, one starting with a
  digit) keep their `#gh:`/`#git:` spelling everywhere.

### D2. Resolution

A tag name resolves against the _tag targets_. These are every non-sibling project
record (enabled or disabled) plus the system `home` project. Each target is reachable by
its directory key, its `PROJECT_NAME:`, or any alias.

1. Try an exact match first, then a case-insensitive (casefold) match. This lets a phone
   keyboard's `+Sase` work.
2. If more than one target matches, the tag is **ambiguous**. SASE never guesses.
   Uniqueness (D8) makes this unreachable on a healthy machine.
3. A resolved target carries these fields:
   - `key`
   - `name`: the display name
   - `tag`: `+<name>`, or null when the name isn't in the grammar
   - `workflow_type`: `gh`, `git`, …, or null
   - `vcs_ref`: `#<workflow_type>:<name>`
   - `provider_display`
   - `state`: `enabled`, `disabled`, or `system`
   - `workspace_dir`
   - `accent` and `accent_index`: null for disabled projects and `home`
4. **VCS type comes from the plugin.** Use `detect_workflow_type(project_file)`, the
   pluggy hook. Do not use Rust's heuristic `vcs_kind`. Detection may spawn processes,
   so the catalog is cached by project-spec signature and built off the UI thread in the
   TUI (tui_perf rule 11).

### D3. Anchored tags are strict; others are forgiving

A tag is **anchored** when it is the first word on its line. Leading whitespace and
`%directive` tokens before it on that line don't count, which matches
`extract_vcs_workflow_tag` and Telegram's `%i:a +sase …`. Alt fan-out puts each branch
into its unit's prompt, so an anchored check after fan-out also covers
`%{+ssae | +sase}`.

| Case                                              | Launch behavior                                               | Editor behavior          |
| ------------------------------------------------- | ------------------------------------------------------------- | ------------------------ |
| Resolved, enabled, has a VCS provider             | expands (D4)                                                  | accent-colored           |
| Resolved but disabled                             | **error**: "`+x` is disabled — `sase project enable x`"       | warning diagnostic       |
| Resolved but no VCS provider detected             | **error**, naming the missing workspace                       | warning                  |
| Ambiguous                                         | **error** listing the claimants and pointing at `sase doctor` | error                    |
| Unknown and anchored                              | **error**, with did-you-mean suggestions and the known tags   | warning with quick fixes |
| Unknown and not anchored                          | plain text; no expansion, no diagnostic                       | unstyled                 |
| More than one workspace target in one launch unit | **error** naming both spans                                   | —                        |

- "Workspace target" counts project tags and VCS workflow refs together. Today's "at
  most one VCS workflow" guard already rejects two refs in one unit.
- Example error:
  `Unknown project tag +ssae (line 1). Did you mean +sase? Known: +actstat +bob-cli +sase`.
- Every launch error is raised before _any_ unit of the launch spawns.

### D4. Execution: the backend is `#gh:`/`#git:` unchanged

Expansion rewrites each resolved tag in place into `#<workflow_type>:<key>`, for example
`+sase` → `#gh:gh_sase-org__sase`. That is byte-for-byte the canonical form the existing
alias canonicalizer already produces from `#gh:sase`.

After expansion, nothing downstream changes. That includes VCS tag extraction,
`#git:home` default normalization, swarm/alt inheritance, the Rust typed planner, the
`gh.yml`/`git.yml` workflows (`setup`, `inject` with `prompt_part: ""`, `release`,
`diff`), the MRU store, and `raw_xprompt.md`.

Expansion runs in two places:

- **Inside `canonicalize_project_aliases_in_prompt`, before its `#` guard.** Its roughly
  12 callers (launch, typed planning, the child runner, `resolve_xprompt_aliases` in the
  processor, and so on) then understand tags for free.
- **Explicitly at the `launch_query` ingress**, once `maybe_dispatch_launch` has decided
  the launch is local. Remote-dispatched prompts are forwarded verbatim, so the remote
  machine resolves tags against _its_ projects. Names are portable; keys are not.

### D5. Canonical storage, tag display

- **Storage.** Durable stores keep the canonical `#<wf>:<key>` form exactly as today:
  `raw_xprompt.md`, prompt history, the VCS MRU, and agent meta.
- **Display.** Every display path already goes through `humanize_vcs_refs_in_text` /
  `humanize_project_refs_in_prompt`. These now also **tagify**: a colon-form ref
  `#<wf>:<ref>` becomes `+<name>` when all of these hold:
  - `<ref>` is a project key, name, or alias
  - the project's detected workflow type equals `<wf>`
  - the name is in the tag grammar
  - the ref has no HITL marker and is not the paren form
- **Not tagified:** Patch refs (`#gh:sase_fix_1`), `owner/repo`, `@agent`, paren forms,
  and mismatched-provider refs.
- **`home`.** `#git:home` renders as `+home`.
- **Tagify needs the catalog.** If the catalog isn't warm on a TUI render path, the
  humanizer falls back to today's `#gh:sase`.
- **One consequence (intended).** Relaunch, fork, copy, and MRU-cycling text that goes
  back into an editor is tag-form. It launches identically. Any code that re-parses
  humanized text must use the tag-aware helpers from the backend phase.

### D6. Colors

A tag renders exactly like the top-right chip in `current_project_indicator.py`:

- the `+` in `dim <accent>`, and the name in `bold <accent>`
- the accent is the one `project_accent(key, among=…)` gives that project
- one canonical `among` set, shared by the chip, the Projects pane,
  `sase project current`, and every tag surface: enabled, non-system project keys. This
  also fixes today's TUI/CLI drift.
- `home` and disabled projects use a neutral dim style
- unknown anchored tags in editors get the theme warning color with an underline

Neovim gets the same hex values: the LSP publishes the Python-owned 18-color palette,
and every semantic token carries its `accentN` index.

### D7. Completion

- **Trigger.** `+` opens the "projects & PRs" menu at any tag left-boundary (D1). That
  now includes the start of a line and after a tab, `{`, or `|`.
- **Accept, in place.** Accepting a row **replaces the typed `+query` token in place**:
  - a project row inserts `+sase ` (or `#<wf>:<name> ` when the name isn't in the tag
    grammar)
  - a PR row inserts `#gh:<patch> `
- **Switching projects.** Accepting also removes every _other_ workspace target (VCS
  refs and resolved tags) from the same `---` segment, outside literal zones, collapsing
  whitespace the way `_strip_trigger_token` does. So picking a project always leaves
  exactly one.
- **Row order.** The current project first, then MRU recency, then alphabetical. PR rows
  follow. `home` resolves as a tag but is never offered as a row; it stays hidden, as it
  is in project lists.
- **Row rendering.** `+name` in its accent, then a dim `GitHub · #gh:sase`, then a
  `current` badge where it applies.
- **One implementation.** The Rust core owns trigger detection and the accept algorithm.
  The TUI calls the new bindings and the Python mirrors (`find_vcs_project_trigger`,
  `apply_vcs_project_selection` plus their golden vectors) are deleted. This follows the
  Rust-core boundary rule.

### D8. Uniqueness

- **The rule.** Across all non-sibling projects of every VCS type, plus the reserved
  `home`, each directory key, `PROJECT_NAME:`, and alias identifies at most one project,
  compared case-insensitively.
- **Where it's enforced:**
  - at write time: project creation, `#git:` init, GitHub name allocation, alias and
    name mutations
  - in `sase doctor`
  - in Rust's advisory `parse_warnings`
- **This machine, checked at planning time.** Enabled projects are
  `actstat (gh_bbugyi200__actstat)`, `bob-cli (gh_bobs-org__bob-cli)`,
  `sase (gh_sase-org__sase)`, and system `home`. The sibling specs are `bob-plugins`,
  `chezmoi`, `sase-core`, `sase-github`, `sase-nvim`, and `sase-telegram`. There are no
  collisions, so no project files need editing today. The docs-memory phase re-verifies
  with the widened doctor check and fixes anything that appears, using
  `sase project alias …` or a `PROJECT_NAME` change.

### D9. What intentionally stays `#gh:` / `#git:`

- `#gh:`/`#git:` stay **permanently supported**, not deprecated. They remain the only
  spelling for:
  - Patches
  - `owner/repo`
  - `@agent`
  - PR numbers
  - paren forms with arguments, such as `#gh(sase, n=3)`
  - creating a _new_ project (`#git:<new-name>`, `#gh:<org>/<repo>`)
- Chop `for_each: source: projects` workspace refs (`gh:owner/repo` from
  `axe/_config_targets.py`) are a documented job-result protocol and stay as they are.
- `DEFAULT_VCS_WORKFLOW_PREFIX = "#git:home"` stays internal. It just displays as
  `+home`.
- Dated blog posts and the demo tapes/seed scripts are historical media. Leave them
  alone.

### D10. No feature flag

Nothing is deprecated, so no `sunset` flag. The phase order keeps every landed state
coherent, so no `beta` flag either:

1. After the backend lands, `+sase` launches work.
2. tui-editor then inserts tags and makes MRU cycling, prefills, and editor context
   tag-aware.
3. Only after that does tag-display switch display strings (including MRU display pairs)
   to tag form and color them.

For the same reason, phases that edit the same repo are chained, except where they own
disjoint files.

## Phase core-tags (sase-core)

Open with `sase repo open sase-core` and follow its `AGENTS.md`: free functions over
`*Wire` structs, no `macro_rules!`, facade `mod.rs`, files at most 1,500 lines.

1. **New module `crates/sase_core/src/project_tag/`.** Its facade exports:
   - `scan_project_tags(text) -> Vec<ProjectTagSpanWire>`
     - `start`/`end` cover the `+`; `name_start`, `name`, and `anchored: bool` are also
       returned
     - implements D1 and D3's anchoring rule
     - skips literal zones via the existing scanners (`prompt_literal_zone_ranges`, and
       `directive_scan` for leading directives)
   - `resolve_project_tag(name, &[ProjectTagTargetWire]) -> ProjectTagResolutionWire`
     - `Resolved { target_index }`, `Ambiguous { candidates }`, or
       `Unknown { suggestions }`
     - suggestions rank known tags with an existing core fuzzy/edit-distance helper;
       take the top 3
   - `expand_project_tags(text, targets) -> ProjectTagExpansionWire`
     - returns the rewritten `text`, plus a per-tag report: span, name, anchored,
       resolution, and replacement
     - only resolved tags that have a `workflow_type` are rewritten, to
       `#<workflow_type>:<key>`
     - never errors; policy (D3) belongs to the callers
   - `project_tag_trigger(text, cursor) -> Option<{start, end, query}>`
     - replaces the `vcs_project_trigger_token` position rule in `editor/token.rs` with
       the D1 left boundary
     - keep `vcs_project_trigger_token` as a thin wrapper so existing callers compile
   - `apply_project_tag_selection(text, trigger_span, insertion, workflow_names, targets) -> {text, cursor}`
     - implements the D7 accept algorithm, including removing other targets per segment
     - replaces `apply_vcs_project_selection`, `vcs_prepend_offset`, and
       `vcs_replace_regex` in `editor/completion/vcs_candidates.rs`; update its LSP call
       sites there
2. **Catalog wire v5 (`editor/wire.rs`).**
   - `VcsProjectEntry` gains optional serde-default fields `key`, `tag`, `accent_index`,
     and `current`.
   - The catalog gains `accent_palette: Vec<String>` and
     `project_tags: Vec<ProjectTagTargetWire>`.
   - v1–v4 catalogs must still load.
   - Follow the "Change a wire schema version" recipe. The Python constant
     `VCS_PROJECT_CATALOG_SCHEMA_VERSION` moves to 5 in the backend phase.
3. **Python bindings.** Add a binding domain modeled on `editor_completion`:
   `project_tag_scan`, `project_tag_resolve`, `project_tag_expand`,
   `project_tag_trigger`, and `project_tag_apply_selection`.
   - Return **Python code-point offsets**, following the `xprompt_argument_spans`
     convention.
   - Register every function and add round-trip tests.
4. **Tests.** Table-driven Rust tests cover:
   - every D1 example
   - literal zones and multi-byte text
   - anchored detection (after `%m:opus`, on a later line, after frontmatter, inside
     `%{…|…}`)
   - resolution (exact, casefold, ambiguous, unknown with suggestions, key/alias)
   - expansion
   - the D7 accept vectors: port the existing golden vectors and add in-place and
     switch-project cases
5. Run `just check` in sase-core before finishing.

## Phase unique-names (sase, sase-core, sase-github)

1. **sase, `src/sase/project_alias_records.py`.** Casefold every comparison in
   `_occupied_project_refs`, `project_alias_map_from_records(strict=…)`,
   `project_ref_conflicts_from_records`, `validate_project_name`,
   `validate_project_aliases`, and `allocate_project_name`.
   - Treat `home` as a reserved ref that no other project may claim.
   - Add the same casefold to `find_project_ref_owner` in `project_aliases.py`. That
     automatically covers the `bare_git_init.py` and `bare_git_ref.py` guards and
     `project_file_utils.create_project_file`.
   - A `#git:Sase` init that collides by case with `sase` is refused with a message that
     suggests `+sase`.
2. **Doctor.** `doctor/checks_project.py::_check_project_name_collisions` stops
   filtering to directory-key conflicts. It reports every conflict: key, name, or alias,
   casefolded, across VCS types, disabled projects included, siblings excluded, `home`
   reserved.
   - Keep the existing detail/next-step/data helpers.
   - Next steps name the concrete `sase project alias remove …` / rename commands.
3. **sase-core.** `add_project_ref_collision_warnings` in `project_spec.rs` casefolds
   and flags claims on `home`. Add tests, then run `just check`.
4. **sase-github.** `workspace_plugin.py::_project_refs` and its callers compare
   casefolded, so `_allocate_canonical_project_name` and `_ensure_useful_repo_name`
   (through core `allocate_project_name`) never allocate a case-variant duplicate. Add
   tests next to the `foo`/`foo_1` cases.
5. **Tests.** Case-variant collisions are rejected at every write path, doctor reports
   them, and existing exact-match behavior still passes.

## Phase backend (sase)

1. **Pin.** Once core-tags has landed on sase-core master, run
   `just ratchet-core-revision` (see `docs/rust_backend.md`) so CI builds the new
   bindings. `just install` rebuilds the local extension from the linked checkout.
2. **`src/sase/project_accents.py`.**
   - Move `PROJECT_ACCENTS`, `project_accent`, and the map helpers out of
     `ace/tui/project_styles.py`, which keeps re-exports.
   - Add `project_accent_index(key)` and `accent_among_keys(records)`, the D6 canonical
     set.
   - Switch `ace/tui/widgets/launch_context_source.py`,
     `ace/tui/modals/projects_pane.py`, and `main/project_handler_current.py` to the
     shared set.
3. **`src/sase/project_tags/` package** (the facade over the core bindings):
   - **Catalog.**
     - `ProjectTagCatalog` builds D2 targets from `list_project_records`, using
       `detect_workflow_type` and accents.
     - It is cached by project-spec signatures, like
       `build_vcs_project_completion_entries`.
     - `load()` blocks; `peek()` never does and returns the last snapshot or `None`.
     - `vcs_project_completion._build_entries` derives its project rows from the same
       catalog, so there is one source.
   - **Functions.**
     - `find_project_tags(text)`
     - `expand_project_tags(text)`
     - `validate_project_tags_for_launch(text)`: raises `ProjectTagError` with the D3
       messages; fits the existing launch-failure reporting
     - `project_tag_for(key_or_ref)`
     - `effective_vcs_workflow_tag(prompt)`: `extract_vcs_workflow_tag` after expansion
     - `effective_find_vcs_workflow_tag(prompt)`
   - Every function short-circuits when `"+"` isn't in the text.
4. **Expansion wiring (D4).**
   - Call `expand_project_tags` first inside `canonicalize_project_aliases_in_prompt`,
     before its `"#" not in prompt` guard.
   - In `main/query_handler/_launch.py::launch_query`, validate and expand once the
     launch is known to be local, before force-reuse, typed dispatch, MRU recording, and
     `launch_agents_from_cwd`.
   - In `agent/launch_cwd_agents.py`, add a unit guard next to
     `_guard_hard_disabled_launch_units`. After swarm/alt fan-out and before any spawn,
     it enforces D3 for each unit, including the rule of at most one workspace target.
   - `_prompt_segment_has_vcs_workflow_ref` / `normalize_default_vcs_workflow_segment`
     treat a resolvable tag as an existing workspace ref, so `#git:home` is never
     injected next to a tag.
   - Verify that every Python caller of the Rust typed planner passes canonicalized
     text: `launch_request_planning.py`, `direct_typed_launch.py`,
     `launch_admission_runtime.py`, and `axe/chop_typed_admission.py`.
   - If the approval preview path surfaces errors cheaply, have LaunchApproval preview
     (`build_preview_plan`) show tag validation errors.
5. **Raw-text consumers.** Switch these from literal `#gh:` parsing to the tag-aware
   helpers:
   - `history/prompt_metadata.py`
   - `history/prompt_history_project_filter.py`
   - `integrations/_mobile_agent_context.py`
   - `integrations/_mobile_agent_launch.py`
   - `monitor/followup_continuation.py`: treat a tag naming the recorded project as
     "already prefixed"
   - `shells/followup.py`
   - `agent/_restart_preview.py`
   - `ace/tui/actions/agent_workflow/_launch_submit_helpers.py`
   - `artifact_ref_prompt_context.py::_leading_or_embedded_tag`

   Then grep for
   `extract_vcs_workflow_tag|find_vcs_workflow_tag|get_ref_patterns|get_vcs_tag_pattern`
   and classify each hit. Code downstream of canonicalization needs no change. Code that
   reads authored text switches helpers.

6. **Generators default to tags.** Each falls back to `#wf:` when the name isn't in the
   tag grammar:
   - `bead/work.py`: the task prompt prefix, and the first-phase/non-Patch epic segment
     prefix. Later Patch phases keep `#gh:<patch>`.
   - the `bare_git_ref.py` provider-mismatch hint, e.g. "Use +sase instead"
   - `sase project show|enable|disable|set-current|alias …` accept a `+name` spelling
     (strip one leading `+`)
7. **LSP catalog v5.**
   - `vcs_project_catalog_payload()` emits schema 5: `accent_palette`, `project_tags`,
     and per-entry `key`/`tag`/`accent_index`/`current`.
   - Entries are ordered as D7: current project, then MRU, then name. PR rows come
     after.
   - `integrations/xprompt_lsp.py` materializes it as today.
   - Update `tools/validate_sase_core_rs` and the schema tests.
8. **Tests.** Cover:
   - expansion equivalence: a `+sase` prompt and a `#gh:sase` prompt canonicalize
     identically, and produce identical launch requests and `raw_xprompt.md`
   - casefold, disabled, and unknown-anchored vs. unanchored cases
   - alt fan-out `%{+sase | +bob-cli}` and swarms
   - two targets in one unit
   - remote dispatch passes tags verbatim
   - MRU still records `#gh:<key>`
   - bead-work prompts
   - catalog caching; `peek()` never builds
   - the launch-query ordering

## Phase lsp (sase-core `crates/sase_xprompt_lsp`)

1. **Completion** (`server/completion.rs`, `lsp_convert.rs`).
   - The VcsProject context uses `project_tag_trigger`.
   - Project items have label `+sase`, `filterText` `+sase`, and detail
     `GitHub · #gh:sase`. Documentation shows the workspace dir, aliases, state, and a
     `current` note.
   - The `textEdit` replaces the trigger token. `additionalTextEdits` come from
     `apply_project_tag_selection`.
   - `sortText` preserves catalog order. PR items keep their behavior but become
     in-place edits.
2. **Semantic tokens** (`semantic_tokens.rs`).
   - Add token type `saseProjectTag`, with modifiers `sigil`, `unknown`, `disabled`, and
     `accent0`…`accent17`. Stay within the 32-bit modifier budget.
   - Emit one sigil token (`+`) and one name token per tag.
   - Advertise `capabilities.experimental.sase.projectTagPalette` (the catalog's
     `accent_palette`) in `initialize`.
   - Update the legend test in `server/tests/shortcuts.rs`.
3. **Hover.** A tag hover shows the name, provider and VCS ref, workspace dir, state,
   and aliases.
4. **Diagnostics** (D3 editor column). Anchored unknown tags get a warning with
   suggestions. Ambiguous tags get an error. Disabled or provider-less projects get a
   warning.
5. **Code actions.**
   - A quickfix replaces an unknown tag with each suggestion.
   - `refactor.rewrite` "Use project tag +sase" rewrites a colon-form project ref, when
     it is resolvable and its provider matches, into its tag.
6. **Leading project.** `server/catalogs.rs::leading_vcs_project` also recognizes a
   leading resolved tag. `@` completion, artifact diagnostics, and glossary context then
   work for tag prompts.
7. **Tests.** Cover each feature with v5 fixture catalogs, plus a v4 catalog that still
   works without accents. Run `just check`.

## Phase tui-editor (sase)

1. **Trigger and accept.**
   - `_file_completion_context.py::_get_vcs_project_trigger` and the `+` auto-open in
     `_prompt_text_area_key_handling.py` use the `project_tag_trigger` binding. The
     auto-open fires at every D1 boundary.
   - `_file_completion_accept_kinds.py::_accept_vcs_project_completion` applies
     `project_tag_apply_selection` and moves the cursor to the returned position.
   - Delete the Python mirror and move its golden-vector test to Rust (core-tags).
2. **Rows.**
   - `_prompt_input_bar_completion_rows_vcs.py::append_vcs_project_completion_row`
     renders D7: accent-colored `+name` (`dim` sigil, `bold` name), a dim
     `provider · #wf:name`, and a `current` badge.
   - `widgets/vcs_project_completion.py::_candidate` inserts the tag.
   - Candidate order follows the catalog.
3. **Tag prefills.** These produce tags instead of `#gh:`:
   - `actions/agent_workflow/_entry_points.py::_vcs_prompt_prefix` (Patch prefixes stay
     `#gh:`)
   - `_entry_custom.py`: the `+` launch picker and the Space-key MRU head
   - `_entry_quick_launch.py`
   - `_prompt_bar_requests.py`
   - `actions/clipboard/_artifacts.py`
   - `_prompt_input_bar_stack_navigation.py`: new-pane seeding
   - the `current_project_indicator.py` tooltip "(+sase)"
   - `project_management_rendering.py`'s origin ref
4. **MRU cycling.** `widgets/_vcs_mru_cycling.py` (Ctrl+N/P) finds the current leading
   workspace target, whether it is a tag or a ref, and replaces it with the next MRU
   entry's display form (a tag for projects).
5. **Editor project context.** Derive it from tags via `effective_vcs_workflow_tag` in:
   - `_xprompt_arg_hints.py::_xprompt_arg_assist_project_from_text`
   - `prompt_completion_root.py::resolve_prompt_completion_base_dir`
   - `ace/tui/_agent_completion_prompt.py`
   - `_xprompt_arg_assist_detection.py`
6. **Pre-submit validation.** When the catalog is warm (`peek()`), reject D3 errors in
   `_launch_submission.py` with an error toast and keep the prompt in the editor. When
   it is cold, `launch_query` still validates.
7. **Catalog warm-up.** `_warm_vcs_project_completion_catalog` also warms
   `ProjectTagCatalog`, off-thread (tui_perf rules 1 and 11).
8. **Shell completion.** Add a `+` fragment marker for `sase run` prompts in
   `completion/emit_bash.py`, `emit_zsh_preamble.py`, and `emit_fish.py`, backed by
   project tag candidates.
9. **Tests and snapshots.**
   - Update `tests/ace/tui/test_entry_points_vcs_prefix_*`, the MRU cycling tests, and
     the completion tests.
   - Refresh `test_ace_png_snapshots_vcs_project_completion.py` goldens through
     `/sase_monitor` with `just fix-tui-screenshots -- <selectors>`. Inspect the report
     before finalizing (see `sase/memory/lint_and_test.md`).

## Phase tag-display (sase)

1. **Humanizer tagify (D5).**
   - Extend `project_alias_prompts.humanize_project_refs_in_prompt` and
     `project_display_names.humanize_vcs_refs_in_text` with `project_tags: bool = True`.
     They use the catalog snapshot for workflow-type checks.
   - **Audit every caller** (about 30 files, listed by
     `grep -rn humanize_vcs_refs_in_text src/sase`). Pure display keeps the default. Any
     site that re-parses the result must use the backend helpers, e.g. `_fork_scope.py`,
     `_entry_relaunch.py`, `vcs_xprompt_mru.load_*_pairs`, and `prompt/cli_run.py`. Pass
     `project_tags=False` only where re-parsing can't be made tag-aware, and add a
     comment.
2. **Tokenizer.** `xprompt/xprompt_inspect.py::tokenize` emits `project_tag` spans
   (resolved; carries accent, state, and sigil/name sub-spans). It emits
   `project_tag_unknown` only for anchored unresolved tags. It uses the core scan and
   the catalog snapshot.
3. **Role system.**
   - `xprompt/highlight.py` gains roles `xprompt.project_tag.sigil`,
     `xprompt.project_tag.name`, and `xprompt.project_tag.unknown`.
   - `HighlightSpan` carries an optional `accent`.
   - `highlight_theme.py` resolves accent roles to `dim <accent>` / `bold <accent>`,
     neutral for no accent.
   - Add the spans to `_ROLE_PRECEDENCE`.
4. **Prompt editor.** `_xprompt_syntax_highlight.py::_register_xprompt_text_area_theme`
   registers the named styles `project_tag.sigil.<0-17>`, `project_tag.name.<0-17>`,
   `project_tag.*.neutral`, and `project_tag.unknown` (warning color and underline).
   `_build_highlight_map` maps spans to them. Re-highlight when the catalog becomes
   warm.
5. **Read-only highlighter.**
   - `ace/tui/util/xprompt_syntax.py::apply_xprompt_overlays` / `highlight_prompt_text`
     style tags. Add the catalog signature to the LRU key.
   - `semantic_overlay._protected_ranges` protects tag spans.
   - Expose one helper, e.g. `stylize_project_tags(text: rich.Text, source: str)`, that
     the tag-surfaces and CLI work reuses.
6. **AGENT XPROMPT.**
   - `prompt_panel/_agent_display_render.py`, `_agent_display_family_render.py` (the
     numbered-hint path too), and `_agent_display_hint_body.py` render tagified,
     accent-colored prompts.
   - The clan member bodies in `_agent_display_clan_sections.py` and
     `_agent_clan_member_content.py` gain the same highlighting; today they are plain.
   - Add the catalog signature to `AgentPromptHighlightContext.fingerprint`.
7. **Tests and snapshots.**
   - Unit tests: `tests/xprompt/test_highlight*.py`, `tests/test_xprompt_inspect.py`,
     `tests/ace/tui/util/test_xprompt_syntax.py`, `test_agent_display_xprompt.py`, and
     humanizer tests (tagify vs. Patch/owner-repo/mismatch/paren).
   - Add a tag case to the `_ace_prompt_png_snapshot_prompts.py` fixtures.
   - Refresh the `agents_xprompt_panel_highlighting*` and prompt-highlighting goldens
     via `/sase_monitor`, and inspect the report.

## Phase tag-surfaces (sase)

Apply the tag-display helper (tags only; don't restyle whole prompts that are plain
today) to:

1. **Prompt history.**
   - The preview in `_prompt_history_interactions.py` changes from
     `Static(markup=False)` to a styled `Text`.
   - The project column in `_prompt_history_rows.py` uses accents.
   - Stash: `_prompt_stash_preview.py`, `stashed_prompts_modal.py`,
     `update_pinned_stash_modal.py`, and the `prompt_stash_row.py` project column.
2. **Modals.**
   - The launch approval body (`launch_approval_modal.py`, via
     `markdown_document_syntax`: add the overlay)
   - `runners_modal.py`
   - `revive_agent_rendering.py`
   - `saved_agent_group_revival_rendering.py`
   - `agent_run_log_modal.py`
   - the metadata pager AGENT XPROMPT (`actions/agents/_metadata_pager_conversation.py`,
     with a project-tag role in `pager/syntax.py`)
   - `notification_modal_options.py`, where it renders Rich text
3. **ACE query language.** The `+project` shorthand highlighting
   (`ace/query/highlighting.py`) uses the project's accent instead of fixed `#AF87D7`.
   It is the same word and should be the same color.
4. **CLI** (colored only when color output is enabled):
   - `sase agent show` (`agents/cli_show.py`)
   - `sase prompt show|list|search` (`prompt/cli_*.py`)
   - `sase xprompt show` (through the role system)
   - `sase project list`: a `TAG` column in accent color; `--json` adds `tag`,
     `workflow_type`, and `accent`
   - `sase project show` prints its tag
5. **Tests and snapshots.** Unit tests per surface. Refresh the prompt-history, stash,
   launch-context, and Projects-pane goldens through `/sase_monitor` with inspection.

## Phase nvim (sase-nvim)

Open with `sase repo open sase-nvim`.

1. **Highlighting.** Add `lua/sase/project_tag_highlight.lua`, wired from
   `lua/sase/init.lua` like `xprompt_semantic_highlight.lua`.
   - On `LspAttach`, read
     `client.server_capabilities.experimental.sase.projectTagPalette`.
   - Define `SaseProjectTagSigil<N>` (fg), `SaseProjectTag<N>` (fg, bold),
     `SaseProjectTagNeutral` (links `Comment`), and `SaseProjectTagUnknown` (links
     `DiagnosticUnderlineWarn`), all `default = true`. Re-apply on `ColorScheme`.
   - An `LspTokenUpdate` handler maps `saseProjectTag` modifiers to those groups via
     `vim.lsp.semantic_tokens.highlight_token`.
2. **Ctrl+T picker.** `lua/sase/complete/_token.lua` stops treating `+` as a delimiter
   at a tag boundary and classifies `+query` as a `project` token. `complete.lua` offers
   projects from `sase project list --json` (`tag` field) and inserts `+name`. The LSP
   remains the primary completion path.
3. **README.** Tags are the default; the `+` trigger now inserts `+sase` in place.
4. **Tests.** Update `tests/lsp_vcs_project_smoke.lua` (`+s` → `+sase` in place). Add a
   highlight test modeled on `tests/xprompt_semantic_highlight.lua`.
5. The user's Neovim config (chezmoi `home/dot_config/nvim/lua/plugins/sase_nvim.lua`
   and `cmp.lua`) sets `native_completion = false` and uses nvim-cmp with cmp-nvim-lsp.
   It gets `+` from the server's trigger characters. Confirm that no config change is
   needed.

## Phase plugins (sase-telegram, sase-github)

1. **sase-telegram** (`sase repo open sase-telegram`).
   - `scripts/sase_tg_inbound.py::_record_project_context` and every other raw-prompt
     VCS read use `effective_vcs_workflow_tag`.
   - Fork/Wait/Retry copy text and prompt displays become tag-form automatically through
     `display_vcs_refs_in_text` → sase's humanizer. Update the
     `tests/test_formatting.py` and `tests/test_inbound.py` expectations.
   - Add a `+Sase` (mobile auto-capitalization) inbound test.
   - `docs/inbound.md` documents `+sase` as the default and keeps `#gh@sase` as the
     shorthand for Patch refs.
2. **sase-github** (`sase repo open sase-github`). `README.md` and `docs/xprompts.md`
   say that `+<project>` is the default way to target a GitHub project. `#gh:` remains
   for `owner/repo`, Patches, PR refs, and `@agent`.

## Phase docs-memory (sase, chezmoi)

1. **Docs.**
   - Add a "Project Tags" section to `docs/xprompt.md`: grammar, resolution, anchored
     strictness, expansion, colors, and how tags relate to `#gh:`/`#git:`.
   - Update project-ref examples (keep Patch/`owner/repo`/`@agent` examples as they are)
     in:
     - `README.md`
     - `docs/getting_started.md`
     - `docs/workspace.md`
     - `docs/ace.md`: the `+` picker now inserts `+sase`
     - `docs/editor.md`: LSP tag features and semantic-token legend
     - `docs/project_spec.md`: uniqueness rule
     - `docs/configuration.md`
     - `docs/mobile_gateway.md`
     - `docs/architecture.md`
     - `docs/beads.md`
     - `docs/prompt.md`
     - `docs/cli.md`
     - `docs/sdd.md`
     - `docs/artifact_references.md`
     - `docs/workflow_spec.md`
     - `smoke/pypi/README.md`
     - `src/sase/config/sase.schema.json` descriptions
   - Blog posts and demos stay untouched (D9).
2. **CLI help.** Update the `sase run` examples in `main/parser_commands.py`,
   `main/parser_root_help.py`, and `main/parser_prompt.py`. Update the matching greps in
   `.github/workflows/publish.yml` (lines ~231/330) in the same change.
3. **Skill source.** In `src/sase/xprompts/skills/sase_run.md`:
   - fix the wrong-provider `#git:sase` example to `+sase`
   - teach `+<project>`, keeping `#gh:<patch>`/`#gh:@agent`

   Don't hand-edit or deploy the chezmoi `SKILL.md` copies. They are regenerated with
   `sase skill init --force` from the landed revision after the epic merges
   (`sase/memory/generated_skills.md`).

4. **Memory.** The user asked for the glossary entry and reference updates. Follow the
   `/sase_memory_write` edit-and-republish flow, then run `sase memory init`:
   - **New strand `sase/memory/glossary/project-tag.md`.** Keyword `Project Tag`,
     aliases `xprompt project tag` and `project tags`. About 6 lines:
     - the `+<project>` word names the project an agent runs in
     - it resolves by key, name, or alias, case-insensitively, against unique names
     - it expands to the project's VCS xprompt ref and shares its backend
     - it is the default spelling; `#gh:`/`#git:` remain for Patches, `owner/repo`,
       `@agent`, and new projects
     - anchored tags must resolve
     - it renders in the project's accent, like the chip
   - **`sase/memory/glossary/sase-project.md`.** Add that prompts name an existing
     project with a project tag, and that keys, names, and aliases are unique across
     projects and VCS types, case-insensitively.
   - **`sase/memory/xprompts.md`, "Project-Task Launches".** Start project work with a
     project tag (`+sase`) or a workspace ref. Add a `+<project>` table row. Change the
     typical prompt to `+sase %auto #pr:my_change <task text>`.
5. **chezmoi** (`sase repo open chezmoi`). Remove the redundant `gh_dotfiles` and
   `gh_sase` entries from `home/dot_config/sase/sase.yml` `xprompts:`.
   - `#gh_sase` still works through built-in underscore normalization, and `+sase`
     replaces the shortcut.
   - No `dotfiles` project spec exists any more.
6. **Machine verification.**
   - Run the widened `sase doctor` project checks. Confirm `project.name_collisions` is
     clean. Fix any collision with `sase project alias`/rename, and report what changed.
   - Run `sase project list` and check the TAG column.
   - Do a dry run of a `+sase` prompt through LaunchApproval preview or
     `sase xprompt expand`. Confirm it expands to the same canonical ref as `#gh:sase`.

## Verification

- **sase-core phases.** `just check` in sase-core.
- **sase phases.** `just install` (it picks up the linked sase-core), then `just check`
  (prefer `sase tool run check`). Run PNG goldens only through the targeted
  `just fix-tui-screenshots -- <selectors>` under `/sase_monitor`. Don't run
  `just check-full` unless explicitly asked.
- **sase-nvim, sase-telegram, sase-github.** Each repo's own test recipe.
- **End-to-end sanity after backend.**
  - `+sase` and `#gh:sase` launches produce identical `raw_xprompt.md` and agent meta.
  - `+Sase` works.
  - An anchored `+ssae` fails before any spawn, with a suggestion.
  - `chmod +x` in prose stays untouched.

## Risks and mitigations

- **Hidden re-parsers of humanized text.** Tagify changes what display strings contain.
  Mitigation: the mandatory caller audit in tag-display plus the tag-aware backend
  helpers. Tagify degrades to today's output when the catalog is cold.
- **UI-thread cost.** Catalog builds spawn processes. Mitigation: `peek()` only on
  render and keystroke paths, off-thread warm-up, and signature-keyed caches.
- **False positives.** Tags must be standalone words, start with a letter, and resolve
  to a known project. Only anchored unknowns are errors.
- **Parity drift.** Trigger and accept logic move to one Rust implementation. The Python
  mirror is deleted.
