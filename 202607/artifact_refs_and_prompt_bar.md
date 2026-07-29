---
tier: epic
title: Kind-tagged artifact references and prompt-bar integration
goal: 'One reference grammar names every artifact SASE knows: `plans:` generalizes
  into `<kind>:<payload>` references covering commits, chats, bugs, artifact files,
  and every configured document-sidecar role, owned by the Rust core and spent in
  the ACE copy menus, the prompt bar (highlighted, completed, and expanded at launch),
  and external editors through LSP completion, diagnostics, and semantic tokens.

  '
phases:
- id: ref-core
  title: Kind-tagged artifact reference grammar in the Rust core
  depends_on: []
  size: large
  description: 'ref-core: add the `artifact_ref` module to the Rust core — parse,
    render, canonicalize, and resolve kind-tagged references with per-kind payloads,
    fragment anchors, caller-supplied document roles, and a prompt-text scanner —
    reusing the `plans:` machinery without changing any `plan/refs.rs` behavior, and
    expose it through new PyO3 bindings.

    '
- id: ref-facade
  title: Python artifact-reference facade and context resolution
  depends_on:
  - ref-core
  size: medium
  description: 'ref-facade: add the Python facade that builds per-project resolution
    context (document roles and roots, chats root, artifact index, repos, projects),
    wraps the new bindings behind schema gates, renders canonical references from
    ACE entry targets and rows, and bumps the sase-core-rs floor.

    '
- id: copy-ref
  title: Copy and hand off references from the Artifacts sub-tabs
  depends_on:
  - ref-facade
  size: medium
  description: 'copy-ref: give Commits, Plans, Chats, and Bugs copy-mode targets that
    copy the selected entry''s `@`-reference and seed a new agent prompt with it (including
    marked sets), and show the logical reference beside its resolved path in the Plans
    and Chats detail surfaces.

    '
- id: prompt-grammar
  title: Recognize and expand artifact references at launch
  depends_on:
  - ref-facade
  size: medium
  description: 'prompt-grammar: recognize `@<kind>:<payload>` references in launched
    prompts through the core scanner and expand them per kind — documents, chats,
    and artifact files to real paths, commits and bugs to their locators — before
    file-reference processing, failing the launch clearly when a known-kind reference
    does not resolve.

    '
- id: prompt-highlight
  title: Artifact-reference highlighting in the prompt input widget
  depends_on:
  - ref-facade
  size: medium
  description: 'prompt-highlight: syntax-highlight artifact references in the prompt
    editor with per-part styles (sigil, kind, separator, payload, fragment) driven
    by the core scanner and a cached known-kind set, following the xprompt highlighter
    pattern, with PNG snapshot coverage.

    '
- id: prompt-complete
  title: Artifact-reference completion in the prompt bar
  depends_on:
  - ref-facade
  - prompt-highlight
  size: large
  description: 'prompt-complete: complete artifact references in the prompt bar —
    kinds after `@`, payloads per kind from warm cached providers for documents, artifact
    files, chats, commits, and bugs — through a pre-tokenizer context detector, and
    record used references into file-reference history.

    '
- id: lsp-complete
  title: Artifact-reference completion and diagnostics in the xprompt LSP
  depends_on:
  - ref-core
  - ref-facade
  size: large
  description: 'lsp-complete: complete and diagnose artifact references in the Rust
    xprompt LSP server, fed by a launcher-materialized artifact catalog, mirroring
    the prompt bar''s recognition and known-kind rules so editors and the TUI agree
    on what a reference means.

    '
- id: lsp-tokens
  title: Semantic-token highlighting for artifact references in editors
  depends_on:
  - lsp-complete
  size: medium
  description: 'lsp-tokens: add a semantic-tokens provider to the xprompt LSP that
    colors artifact references in external editors using standard LSP token types,
    so default editor themes highlight them with no client configuration.

    '
create_time: 2026-07-29 12:44:53
status: wip
---

# Plan: Kind-tagged artifact references and prompt-bar integration

## Context

The consolidated research report `202607/artifact_refs_and_inspector/artifact_refs_and_inspector.md` in the `research`
sidecar (open it with the `/sase_repo` skill) ranked nine recommendations. Epic `sase-as` shipped item 1 (tranche-zero
defects) and replaced its research-specific item 4 with generic document-sidecar roles. This epic implements items 2 and
8:

- **Item 2:** extend `plans:` into a kind-tagged artifact reference, following the `sase-9z` playbook.
- **Item 8:** artifact refs in the prompt bar — `@`-completion, launch-time expansion, and copied-ref hand-off — plus,
  beyond the report, excellent syntax highlighting and completion in the prompt input widget **and** in external editors
  through SASE's LSP support.

One instruction from the research report is deliberately **not** taken: its proposed `research:<path>` kind. Epic
`sase-as` removed the name `research` from the behavioral role registry — every configured non-reserved sidecar role is
a document sidecar (`src/sase/sdd/_store_types.py:32`, `document_sidecar_roles`), and only shipped presentation presets
stay name-keyed (`src/sase/sdd/_paths.py:18`). Hardcoding a `research:` kind would reintroduce exactly that mistake.
Instead, **document-role kinds are dynamic**: the grammar accepts any configured document-sidecar role as a reference
kind, `plans` is merely the reserved instance, and no SASE code names `research`. A `research:202607/x.md` reference
works on this project only because its config declares that role.

### The reference grammar

Canonical (storage) form, one line, no whitespace:

```
<kind>:<payload>[#<fragment>]
```

with `<kind>` matching `[a-z][a-z0-9_-]*` and these payload grammars:

| Kind                         | Payload                                                         | Example                                    |
| :--------------------------- | :-------------------------------------------------------------- | :----------------------------------------- |
| `plans` or any document role | POSIX relpath under the role's roots                            | `plans:202607/durable_plan_refs.md`        |
| `commit`                     | `<repo>@<sha>` (7–40 lowercase hex; canonical render = full 40) | `commit:sase@a1ed6146f…`                   |
| `chat`                       | relpath under the chats root (`~/.sase/chats/`)                 | `chat:202607/main-commit-260707_115135.md` |
| `bug`                        | `<project>#<n>`                                                 | `bug:sase#123`                             |
| `file`                       | an artifact-file id (`explicit:<hex24>` or `default:<hex24>`)   | `file:default:52895d68931185056fd0e49f`    |

Prompt form: the canonical form prefixed with `@`, e.g. `@plans:202607/foo.md#L12-L18`.

Design decisions, made here so phases do not re-litigate them:

- **Builtin kinds are exactly `commit`, `chat`, `bug`, `file`.** Every other syntactically valid kind is a candidate
  document role. Kind classification (builtin / configured document role / unknown) requires project context and is
  supplied by the caller — the same seam the document-corpora API already uses, where Python owns role discovery and
  Rust takes bare `(root, kind)` pairs (`src/sase/plan_search/facade.py:102-107`,
  `crates/sase_core/src/plan/read.rs:56-72`).
- **Unknown kinds are prose, not errors.** A token like `@user:handle` whose kind is neither builtin nor a configured
  role is left untouched everywhere: launch expansion skips it, the highlighter leaves it unstyled, and no diagnostic
  fires by default. This keeps existing prompts byte-stable (today `@plans:x` is silently ignored because `:` is
  excluded from the file-ref token class, `src/sase/file_references.py:37`) and makes the known-kind set the single
  definition of refhood. Completion is what prevents typos.
- **Fragment anchors resolve the `#` collision by kind.** The kind is parsed first and selects the payload grammar:
  `bug` consumes `#<n>` as payload and takes no fragment; every path-shaped kind (document roles, `chat`, `file`) treats
  the first `#` as the start of an optional fragment — `L<n>`, `L<n>-L<m>`, `page=<n>`, or `t=<seconds>`. Parsing is
  unambiguous, and xprompt tokenization cannot see the `#` because it never follows whitespace
  (`src/sase/xprompt/_parsing_references.py`). Resolution ignores fragments; they are carried, rendered, and shown.
- **Left context escapes literals.** A ref candidate's `@` must sit at start-of-line, after whitespace, or after `"`/`'`
  — the same rule as `_FILE_REF_PATTERN` — so writing a ref in backticks (`` `@plans:x.md` ``) keeps it literal in
  prompts and in the highlighter (which additionally honors `literal_zone_ranges`).
- **Bug and project labels use the user-facing project name** (`bug:sase#123`, never `gh_sase-org__sase`), per the
  project-display-name convention; resolution accepts the key as an alias.
- **`plan/refs.rs` behavior is frozen.** Bead `design` fields, their wire records, and every existing `plan_reference_*`
  binding keep byte-identical behavior — including "any non-`plans:` value is a legacy path"
  (`crates/sase_core/src/plan/refs.rs:31-49`). The new module reuses the internals (payload validation, ordered-root
  probing, month-drift recovery) by refactoring, not by forking; no stored data migrates.

### Verified current state

Everything below was checked against the working trees on 2026-07-29; cite it rather than re-deriving it.

**The Rust substrate is one kind wide.** `plan/refs.rs` hardcodes `plans` (`refs.rs:12-13`), rejects other kinds in
`render_plan_reference` (`refs.rs:56-60`), and resolves with ordered roots plus month-drift recovery (`refs.rs:81-145`),
returning `PlanReferenceResolutionWire` statuses `exact|drifted|ambiguous|missing` (`plan/wire.rs:86-92`). The sase-as
document-corpora work (`13cb8b7`, v0.12.10) did not touch it. Python reaches it only through the
`src/sase/sdd/plan_refs.py` chokepoint, and role discovery already lives in `document_sidecar_roles` +
`SddStore.kind_root` (`src/sase/sdd/_store_types.py:32,129`).

**Identity tuples for every sub-tab row already exist.** `ArtifactEntryTarget` variants: `("commit", repo, full_sha)`
(`widgets/artifacts/commits_timeline.py:25`), `("chat", absolute_path)` (`chats_list.py:35`),
`("bug", project_scope, number)` (`bugs.py:387`), and `("plan", project, row_kind, identity)` (`plans_list.py:49`).
Archive rows additionally carry the corpus role (`plans_data_models.py:38`, `ProjectArchive.role`) and search results
carry a corpus-relative `relpath` (`src/sase/plan_search/model.py`), so a document reference renders directly as
`<role>:<relpath>`.

**Chats mirror the plans layout.** Transcripts live under `sase_subdir("chats")` in `YYYYMM/` shards with legacy
top-level files and imported non-shard directories, and `resolve_chat_file_path` (`src/sase/history/chat_storage.py:49`)
already implements basename-drift lookup across shards.

**Artifact-file ids are already a `prefix:hash` grammar.** `artifact_file_id`
(`src/sase/core/artifact_file_helpers.py:64-80`) returns `explicit:<hex24>` or `default:<hex24>`; the index is read by
`read_artifact_file_index` (`src/sase/core/artifact_file_explicit.py:146`) with no query API — full-scan under a shared
lock.

**The prompt bar has no `@` grammar of record.** Four independent `@` grammars disagree: launch processing
(`src/sase/file_references.py:37`), history recording (`src/sase/history/file_references.py:17`), display hints
(`widgets/prompt_panel/_file_path_hints.py:18`), and the completion widget's strip heuristic
(`widgets/file_completion.py:32,92`). None highlight: no mixin emits spans for `@` tokens today. `#` refs, by contrast,
have one canonical lexical source consumed by both the highlighter and launch processing
(`src/sase/xprompt/_parsing_references.py:36`, `xprompt_inspect.tokenize`).

**`:` is a hard token delimiter in every tokenizer** — Rust (`crates/sase_core/src/editor/token.rs:336`), Python
(`widgets/file_completion.py:24`), Lua (`sase-nvim/lua/sase/complete/_token.lua:8`) — so `@plans:202607/foo.md`
truncates to `@plans` and classifies as nothing. The working precedent is `#gh:` refs, which bypass generic tokenization
through a dedicated pre-tokenizer detector (`crates/sase_core/src/editor/completion.rs:110`,
`detect_vcs_ref_context_at_position`).

**Editor support is LSP, not tree-sitter.** No tree-sitter grammar exists in sase or sase-nvim. The Rust
`sase_xprompt_lsp` server (`crates/sase_xprompt_lsp/src/server.rs`) provides completion (`@` is already a trigger
character, `server.rs:938`), hover, code actions, and push diagnostics — but no semantic-tokens provider
(`server.rs:975`). The Python launcher (`src/sase/integrations/xprompt_lsp.py`) already materializes JSON catalogs the
server re-reads per request (`_prepare_xprompt_lsp_environment`, `:218-300`). sase-nvim consumes the LSP natively
(`lua/sase/lsp.lua:266`), so it needs no changes for either completion or semantic-token highlighting.

**Copy mode is sub-tab aware post-sase-as.** Key groups `artifacts_{commits,plans,chats,bugs}` exist in
`keymaps/mode_keymaps.py:66-91` and `default_config.yml:361-403` (the two must stay in sync), with dispatch and
per-target handlers in `actions/clipboard/_artifacts.py`, palette labels in `commands/_mode_commands.py:49`, footer
rendering in `widgets/_keybinding_modes.py:393`, and help sections in `modals/help_modal/changespecs_bindings.py`. Marks
are keyed on `ArtifactEntryTarget` per sub-tab (`actions/artifacts.py:192`). The Bugs pane already demonstrates prompt
seeding: `action_start_agent_from_bug` builds a prompt and calls `_show_prompt_input_bar_for_home`
(`actions/artifact_bugs.py:418`, `actions/agent_workflow/_prompt_bar_mount.py:351`).

### Ground rules for every phase

- Run `just install` first (workspace virtualenvs are ephemeral), then `just check` before finishing, per the
  build-and-run memory.
- Per the Rust-core-boundary memory: reference grammar, resolution, scanning, and LSP behavior are core backend logic
  and belong in the `sase-core` linked repo — open it with the `/sase_repo` skill; never clone or web-fetch it. Python
  keeps role/root discovery, keymaps, widgets, and rendering. Wire and binding changes land in `sase-core` first, with
  the `sase-core-rs` floor bump in this repo.
- Phases touching the prompt widget's keystroke or refresh paths (`prompt-highlight`, `prompt-complete`) MUST read the
  TUI-performance memory with the `/sase_memory_read` skill first; `prompt-grammar` must read the xprompts memory.
  Keystroke paths are read-only, subprocess-free, and warm-cache-only.
- Do not edit `sase/memory/*.md`, `AGENTS.md`, or generated provider shims; this plan does not authorize it.
- User-facing project labels render `PROJECT_NAME`, never ProjectSpec keys. New config keys need matching
  `src/sase/config/sase.schema.json` updates; new keymap defaults need matching `src/sase/default_config.yml` updates,
  kept in sync with `mode_keymaps.py`.
- No new CLI subcommands are added by this epic (`sase artifact` is a later research item).

---

## Kind-tagged artifact reference grammar in the Rust core

Work in the `sase-core` linked repo. Add a new `crates/sase_core/src/artifact_ref/` module (re-exported from the crate
root) owning four operations plus a scanner, with no PyO3 types:

- **Parse** a canonical string into a typed reference: kind (builtin variant or document-role string), kind-specific
  payload struct, optional fragment. Payload validation reuses the containment rules `validate_reference_path` enforces
  today (`plan/refs.rs:147-174`) for path-shaped kinds; `commit` validates repo-name shape and 7–40 lowercase hex; `bug`
  validates a non-empty project label and decimal issue number; `file` validates the `(explicit|default):<hex24>` id
  shape. Fragments parse into a typed enum (`Lines`, `Page`, `Time`); `bug` and `commit` reject fragments. A malformed
  payload for a well-formed kind is a parse error, not a silent fallback.
- **Render** a typed reference back to its canonical string (full 40-hex sha for `commit`, `/`-separated relpaths,
  fragment re-attached).
- **Canonicalize** an absolute path into a document, chat, or file reference given per-kind roots, returning `None` when
  the path lies under no supplied root — the same contract as `canonicalize_plan_reference` (`plan/refs.rs:66-78`).
- **Resolve** a typed reference against a caller-supplied context — ordered roots per document role, the chats root, the
  artifact-index path, known repo names, known project names/keys — returning a structured outcome that extends the
  established status vocabulary: `exact` / `drifted` / `ambiguous` / `missing` for path-shaped kinds (reusing the
  ordered-root probe and month/shard-drift recovery from `plan/refs.rs:104-142`, generalized so chats' basename drift
  matches plans' month drift), plus `unknown_kind`, `unknown_repo`, and `unknown_project`. `file` resolution scans the
  artifact index (`index.jsonl` line format, envelope `{"schema_version": 1, "artifact": {...}}`) for the id and returns
  the stored path. `commit` and `bug` resolution validate their namespace (repo/project known) and echo the locator;
  they never invoke git or the network.
- **Scan** prompt text for `@`-prefixed reference candidates, returning byte spans per part (sigil, kind, separator,
  payload, fragment) plus a well-formedness flag, applying the left-context rule from the Context section and stripping
  trailing `.,;:!?)` the way file-ref parsing does (`src/sase/file_references.py:88`). The scanner is context-free (it
  does not know which kinds exist) and does not skip fenced zones — callers filter, exactly as the xprompt highlighter
  does with `literal_zone_ranges`.

Refactor `plan/refs.rs` to delegate its payload validation and ordered-root/drift machinery to shared internals. Its
public API, error strings, wire record, and schema version are frozen; the existing Rust tests must pass unmodified.

Add wire records for the parse and resolution outcomes with their own schema-version constants, following the
`PLAN_*_WIRE_SCHEMA_VERSION` convention in `plan/wire.rs`. Expose PyO3 bindings in `crates/sase_core_py/src/lib.rs`
following the `plan_reference_*` pattern: `artifact_ref_parse`, `artifact_ref_render`, `artifact_ref_canonicalize`,
`artifact_ref_resolve`, `artifact_ref_scan_prompt`, and `artifact_ref_wire_schema_version`, registered in the module
init.

Rust tests: round-trip render/parse per kind; every rejection case; fragment parsing including the `bug`/`commit`
rejection; first-root-wins ordering; drift resolving to one hit; drift with two hits reporting `ambiguous`; unknown
kind/repo/project outcomes; scanner spans for refs at line start, mid-sentence, inside backticks (still emitted —
filtering is the caller's job), with trailing punctuation, and with a second `@` or `:` inside the payload.

Coordinate a `sase-core-rs` release so the next phase can pin it (release-plz owns versioning; sase-as shipped the same
dance for v0.12.10).

---

## Python artifact-reference facade and context resolution

Add `src/sase/artifact_refs.py` as the single Python entry point, mirroring the shape of `src/sase/sdd/plan_refs.py`
(schema-version gates, `require_rust_binding`, typed dataclasses):

- **Context building.** `artifact_ref_context(workspace_dir, workspace_num, project=None)` assembles: ordered roots per
  document role — `store.kind_root(role)` for `document_sidecar_roles(store.split_sidecar_roles(), include_plans=True)`
  with `sase_subdir("plans")` appended for the `plans` role only (matching `resolve_plan_roots`, `plan_refs.py:51-67`);
  the chats root `sase_subdir("chats")`; the artifact-index path `default_artifact_files_index_path()`; repo names from
  the project inventory; and project display-names and keys (both accepted on input, names rendered on output). Roles
  resolve per project; a missing sidecar clone contributes its path anyway (resolution reports `missing`, it does not
  crash) — follow the per-role error isolation the Plans pane established (`plans_data_sources.py:94-95`).
- **Binding wrappers** for parse/render/canonicalize/resolve/scan with the same strict schema-version checks
  `plan_refs.py:137-163` applies.
- **Reference rendering from ACE identities.** `reference_for_entry_target(subtab, target, ...)` maps the four tuple
  shapes to canonical refs: commits → `commit:<repo>@<sha>`; chats → `chat:<relpath>` by relativizing against the chats
  root (an absolute path outside it yields no reference); bugs → `bug:<display_name>#<n>`; plan archive rows →
  `<role>:<relpath>` from `ProjectArchive.role` and the match's `relpath`; epic/phase rows → the bead's canonical
  `plans:` design ref when it parses as one; proposal rows → `canonicalize` against the plans roots (the local archive
  root makes pre-approval plans referenceable). Return `None` rather than inventing a reference.
- **Version plumbing.** Bump the `sase-core-rs` floor in `pyproject.toml` and `uv.lock`; keep the declared-minimum
  assertion in `tests/test_sase_core_rs_telemetry_smoke_tool.py` in step (sase-as's land note records it drifting once
  already); add the six new bindings to `REQUIRED_BINDINGS` in `tools/validate_sase_core_rs`.

Tests: context assembly for a project with a non-`research`-named document sidecar (fixture role, e.g. `designs` — never
assert on the literal `research`); reference rendering for each entry-target shape including the no-reference cases;
resolution round-trips through real fixture roots for one document role, a chat, and an indexed artifact file;
display-name vs key acceptance for `bug` refs; schema-gate failure behavior.

---

## Copy and hand off references from the Artifacts sub-tabs

Spend the reference in copy mode, uniformly across the four non-PR sub-tabs:

- **`%@` — copy reference.** Add an `at` target to all four `artifacts_*` key groups that copies the selected entry's
  prompt-form reference (`@plans:202607/foo.md`) via `reference_for_entry_target`; when the sub-tab has marks, copy the
  marked set one per line through `format_multi_copy_content`, and let the toast name the count. An entry with no
  reference warns and names why (e.g. an imported chat outside the chats root).
- **`%!` — reference to a new agent prompt.** Seed the prompt bar with the workspace ref prefix and the reference —
  `#gh:<project> @<ref> ` — through `_show_prompt_input_bar_for_home`, exactly as `action_start_agent_from_bug` does
  (`actions/artifact_bugs.py:432-452`); a marked set seeds one prompt containing all refs. Derive the project from the
  entry (bug/plan rows carry it; commits and chats use the pane's project scope), rendering the display name.

Both keys are uniform across the four groups. Update the four synchronized surfaces together: `mode_keymaps.py` and
`default_config.yml` key blocks, `_COPY_LABELS` in `commands/_mode_commands.py`, the copy footer
(`widgets/_keybinding_modes.py`), and the help modal (`help_modal/changespecs_bindings.py`). If `!` or `@` collides with
anything in a group, pick another key pair but keep it identical across all four groups and say so in the completion
message.

**Show the logical reference.** In the Plans detail pane, archive rows gain a "Reference" property row beside the
existing path row (`widgets/artifacts/plans_detail.py` already renders "Plan reference"/"Resolved plan" for bead-linked
rows — match that presentation). The Chats detail surface gains the same row. Follow the reference-then-resolved-path
display convention `plan_ref_display.py` established.

Docs and tests: update `docs/ace.md` copy-mode tables (`:97-106`, `:473`) and the `?` help popup per
`src/sase/ace/CLAUDE.md`. Test per sub-tab: `%@` copies the expected canonical form; `%!` opens the prompt bar seeded
with prefix + ref; marked-set variants; the no-reference warning; and that PR-sub-tab behavior is untouched. The PNG
suite covers detail panes — run `just test-visual` and accept goldens only for intentional changes.

---

## Recognize and expand artifact references at launch

Make `@<kind>:<payload>` mean the same thing to a launched agent that it means in the UI. Read the xprompts memory first
(launch pipeline, literal zones).

Add an expansion step to `src/sase/llm_provider/preprocessing.py` immediately before `process_file_references`
(currently step 3, `:160-163`), inside the existing fenced-block protection so code blocks stay literal. The step scans
with `artifact_ref_scan_prompt` via the facade, classifies each candidate against the known-kind set for the launch
context, and:

- **Document, chat, and file refs** rewrite to `@<resolved-path>`, handing the result to the existing file-reference
  machinery (validation, home-dir copying) unchanged. A fragment is detached before the path lands in the `@` token —
  `@plans:x.md#L12-L18` becomes `@<abs-path> (lines 12-18)` — because the file-ref grammar would otherwise treat `#…` as
  part of the filename and fail the exists-check.
- **Commit refs** rewrite to the `<repo>@<full-sha>` locator plus the repo's resolved checkout path when the repo is
  known in the launch workspace; **bug refs** rewrite to `#<n> <url>` via the same URL resolution the Bugs copy target
  uses (`resolved_bug_url`).
- **Unknown kinds pass through untouched** (the prose rule from the Context section).
- **A known-kind reference that fails to parse or resolve fails the launch** with the same explicit
  error-then-`sys.exit(1)` contract missing file refs use (`file_references.py:244-247`), naming each bad reference and
  its resolution status (`missing`, `ambiguous`, …).

Mirror the non-mutating validation path (`validate_file_references`) for the xprompt handler
(`src/sase/main/xprompt_handler.py:55`), and keep `is_home_mode` semantics parallel to file refs. Note the resolution
context is built from the _launched agent's_ workspace, so document refs resolve against that workspace's sidecar
clones.

**History safety.** `extract_recordable_file_refs` (`src/sase/history/file_references.py:45`) must not mangle refs:
teach it to record a well-formed artifact reference verbatim (`@`-form) and never a colon-truncated fragment of one.
(The history _menu_ work lands with `prompt-complete`.)

Tests: expansion per kind against fixture roots; fragment detachment; unknown-kind passthrough (regression:
`@user:handle` and `@plans` bare survive byte-identically); fenced-block immunity; backtick-literal immunity; the
failure contract for a missing document, an ambiguous drift, and an unknown repo; home-mode behavior; and `docs/llms.md`
pipeline docs updated (`:1420` region).

---

## Artifact-reference highlighting in the prompt input widget

Read the TUI-performance memory first. Add an `ArtifactRefHighlightMixin` to the prompt editor following the xprompt
highlighter pattern exactly (`widgets/_xprompt_syntax_highlight.py:50` — `_build_highlight_map` chaining,
`_append_highlight_span`, theme registration on app theme change, the 80 KB/1,200-line overlay guards):

- Early-out unless the text contains `@`. Get spans from `artifact_ref_scan_prompt` through the facade (an in-process
  binding call, no I/O), filter with `literal_zone_ranges`, and classify kinds against a cached known-kind set — the
  four builtins plus the document roles for the prompt's target project. The role set is tiny; warm it off-thread on bar
  mount alongside the existing xprompt warmers (`prompt_input_bar.py:318-322`) and treat a cold cache as indeterminate
  (style the ref neutrally; never flash an error style while data loads, and never resolve anything on the keystroke
  path).
- Styles per part: `artifact_ref.sigil`, `artifact_ref.kind`, `artifact_ref.separator`, `artifact_ref.payload`,
  `artifact_ref.fragment`, with a distinct subdued treatment for well-formed refs whose kind is unknown (they are prose
  — visibly _not_ live references, but not errors either). Derive sibling colors with the established
  `_derive_argument_color` approach and register per-theme styles so both light and dark themes read well. Kinds should
  read as one family with the xprompt invocation color space without being confusable with `#` invocations.
- Register the mixin in the `PromptTextArea` stack (`widgets/prompt_text_area.py:70-92`).

Tests: unit tests for span emission (valid ref, unknown kind, fenced/inline-code immunity, fragment spans, two refs on
one line, ref at the 80 KB boundary), and PNG visual snapshots for the prompt bar showing a highlighted document ref, a
`commit` ref, and an unknown-kind token side by side — run `just test-visual` and accept new goldens deliberately.

---

## Artifact-reference completion in the prompt bar

Read the TUI-performance memory first; every rule below about keystroke paths is load-bearing. Follow the xprompt
completion engine as the template (`widgets/xprompt_completion.py`, warm-cache pattern
`_file_completion_context.py:356`).

**Context detection.** `:` is a token delimiter, so add a dedicated artifact-ref context detector that runs before
generic tokenization — the `#gh:` precedent — recognizing `@`, `@<partial-kind>`, `@<kind>:`, and
`@<kind>:<partial-payload>` around the cursor. Register a new completion kind (e.g. `"artifact_ref"`) and wire it
through all five completion surfaces: open (`_file_completion_open.py`), Ctrl+T (`_file_completion_tab.py`), retype
refresh (`_file_completion_refresh.py`), accept (`_file_completion_accept.py`), and panel title/rows
(`_prompt_input_bar_completion_panel.py`). Bare-`@` tokens that are not ref-shaped keep their existing file-path
behavior byte-for-byte.

**Two-stage candidates.**

1. After `@`: the kinds — four builtins plus the target project's document roles (from the same cached role set the
   highlighter uses). Accepting a kind inserts `@<kind>:` and re-opens completion, the directory-drill-down pattern
   (`_file_completion_accept.py:362`).
2. After `@<kind>:`: payloads from per-kind providers, all warm-cache-only on the keystroke path (return nothing and
   schedule a warm when cold, exactly like `_build_warm_xprompt_completion_candidates`):
   - **documents** — role-labeled corpus entries via the plan-search facade, warmed off-thread, candidates showing
     relpath with title metadata, newest first;
   - **file** — the artifact index, read off-thread and cached keyed on the index file's mtime/size; display label +
     kind + age, insert the full id;
   - **chat** — the chats root scan (bounded, recency-sorted), reusing `iter_chat_files`;
   - **commit** and **bug** — served only from the Artifacts panes' already-warm snapshots when available; never a git
     subprocess or network call from the completion path. Prefix-filter and rank case-insensitively like xprompt
     candidates; support the shared-prefix extension behavior.

**Settings and polish.** Add an `auto_artifact_menu` completion setting (default on) beside `auto_xprompt_menu` in
`PromptCompletionSettings`, `default_config.yml`, and the config schema; auto-open only once the token is unambiguously
ref-shaped (`@` plus two kind characters, or any `@<kind>:`) so plain `@`-path typing never flickers. Panel title names
the stage ("artifact kinds" / "`plans:` documents"). Respect `NavigationGate` and the debounce settings.

**History menu.** Extend the `file_history` completion source so recorded artifact refs (from `prompt-grammar`'s
recorder) appear alongside recent files, `@`-form, with the existing Ctrl+D delete affordance.

Docs and tests: add the kind to the completion-kind list in `docs/ace.md` (`:2979`) and the help popup. Test: context
detection at every cursor position including mid-payload; both stages' candidates against fixture data (fixture role
named `designs`); warm-cache miss returns empty and schedules; accept inserts and re-opens across the kind boundary;
bare-`@` path completion regression; history round-trip. Include a perf guard that the keystroke path performs no file
I/O when caches are warm (assert via the existing tracing/bench patterns rather than wall-clock).

---

## Artifact-reference completion and diagnostics in the xprompt LSP

Work spans the `sase-core` linked repo (server) and this repo (launcher). The goal is editors and the TUI agreeing on
what a reference is, powered by the same `artifact_ref` module.

**Launcher catalog (this repo).** Extend `_prepare_xprompt_lsp_environment` (`src/sase/integrations/xprompt_lsp.py:218`)
to materialize an `artifact_ref_catalog.json` beside the existing vcs-project and model catalogs: per project — document
roles with their corpus roots — plus the chats root, the artifact-index path, repo names, and project
display-names/keys. Reuse the facade's context builder so the catalog and the TUI cannot drift. Export its path in the
server environment; refresh rides the existing `sase.xpromptLsp.refreshCatalog` command.

**Server recognition (sase-core).** Add an artifact-ref context detector ahead of generic tokenization in
`classify_completion_context_with_workflows` (`crates/sase_core/src/editor/completion.rs:83`), built on
`artifact_ref::scan`, mirroring the `detect_vcs_ref_context_at_position` shape. The active project comes from the
document's leading workspace ref, the same resolution the vcs detectors already perform; fall back to the default
project when absent.

**Completion.** Two stages matching the prompt bar: kinds after `@` (builtins + the active project's roles from the
catalog), payloads after `@<kind>:` — documents by scanning catalog corpus roots (bounded, names/titles), `file` from
the artifact index, `chat` from the chats root. `commit` and `bug` payloads are not completed in the LSP (no git, no
network from a request handler); their kinds still complete. `@` is already a trigger character (`server.rs:938`).

**Diagnostics.** Push-based, joining the existing family (`editor/diagnostics.rs`): `malformed_artifact_ref` (known
kind, invalid payload or illegal fragment) and `unresolved_artifact_ref` (document/chat/file refs whose target does not
exist — cheap existence probes only), both respecting the fenced/literal zones the existing diagnostics honor. Unknown
kinds produce no diagnostic (the prose rule). Never resolve `commit`/`bug` beyond shape.

Tests follow the server's existing patterns (`server.rs` integration tests, `completion.rs` unit tests): context
classification at each cursor position; kind and payload candidates against a temp catalog + fixture roots (role named
`designs`, never `research`); diagnostics for a malformed fragment, a missing document, and silence for unknown kinds
and fenced examples; and a regression that plain `@path` file completion still classifies as `FilePath`. Verify locally
against a real editor with `just rust-lsp-install`, and note in the completion message that the LSP binary is
distributed by local build, not by the `sase-core-rs` wheel.

---

## Semantic-token highlighting for artifact references in editors

In the `sase-core` linked repo, add a `semantic_tokens_provider` (full-document) to `sase_xprompt_lsp`
(`server.rs:921-977` currently defaults it off), emitting tokens for artifact references from the shared scanner and the
same catalog-backed known-kind classification `lsp-complete` introduced:

- Use standard LSP token types and modifiers so default editor themes color refs with zero client configuration — one
  type per part (kind, payload, fragment), with a modifier distinguishing document-role kinds from builtins, and no
  token at all for unknown-kind prose. Choose types that read well in stock themes (e.g. `namespace`/`string`/ `number`)
  and record the mapping in the legend and the docs.
- Tokens skip fenced/literal zones, matching the diagnostics.
- Keep the legend extensible: this provider is the foundation for future xprompt/directive token emission, but this
  phase emits artifact-ref tokens only — say so in a comment rather than emitting extra token types speculatively.

Confirm nvim needs no plugin change (native `vim.lsp.semantic_tokens` activates from the server capability;
`lua/sase/lsp.lua` passes default capabilities). Add server tests asserting the encoded token stream for a document
containing a document ref, a `commit` ref, an unknown-kind token, and a fenced example; and a capabilities test that the
provider and legend are declared. Update the LSP docs in this repo (`docs/` wherever `lsp-complete` documented the
feature) with the token legend, and verify end-to-end in a real editor via `just rust-lsp-install`.

---

## Out of scope

Named so no phase drifts into them; most are later items from the same research report:

- `sase artifact` as a read CLI, and the `sha256`/`size_bytes`/`mime_type` index fields (research item 3). The
  prompt-bar and LSP completion read the index in-process; no CLI is added or required.
- Making `PreviewPanelModal` a real reader (item 4), the Files sub-tab (item 5), the unified "Copy as…" palette and OSC
  52 transport (item 6), bulk-mark actions beyond the existing copy path (item 7), and Jump All / key-vocabulary cleanup
  (item 9).
- A `research:` kind, or any code-known document-role name. Document kinds come from project config, full stop.
- A `pr:`/ChangeSpec reference kind, persisting non-`plans` references in structured bead fields, and extending
  `sase bead doctor` to new kinds — nothing structured stores them yet; the bead design field remains `plans:`-only and
  its doctor coverage is complete (`sase-9z`).
- A tree-sitter grammar. None exists; editor highlighting is delivered through LSP semantic tokens.
- sase-nvim plugin changes (native LSP consumption covers completion and tokens) beyond what its README already
  documents.
- Re-minting artifact-file ids, new artifact stores, or index schema changes.
