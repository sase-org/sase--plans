---
tier: epic
status: done
title: Snippet catalog, CLI, and ACE panel
goal: "Users can inspect and safely manage SASE snippets from a shared command/domain
  layer and a polished ACE panel, including first-class navigation across #[...] calls
  and backlinks.

  "
phases:
  - id: core-relations
    title: Rust snippet relation and validation contract
    depends_on: []
    size: medium
    description:
      "core-relations: extend the shared Rust snippet composer with validated trigger,
      relation, alias, and diagnostic metadata."
  - id: catalog-mutations
    title: Project-aware snippet catalog and mutation service
    depends_on:
      - core-relations
    size: medium
    description:
      "catalog-mutations: build one provenance-aware Python service for catalog reads
      and conflict-safe config snippet writes."
  - id: snippet-cli
    title: sase snippet command group
    depends_on:
      - catalog-mutations
    size: medium
    description:
      "snippet-cli: expose add, delete, list, and show with rich and machine-readable
      output over the shared service."
  - id: panel-browser
    title: Snippets panel browsing and relation travel
    depends_on:
      - catalog-mutations
    size: medium
    description:
      "panel-browser: build the hidden asynchronous ACE panel shell, polished cards,
      filtering, and link/backlink navigation."
  - id: panel-crud-polish
    title: Panel CRUD, prompt entry, and release polish
    depends_on:
      - snippet-cli
      - panel-browser
    size: medium
    description:
      "panel-crud-polish: add tracked conflict-safe writes, expose the requested prompt
      keymaps, and complete visual, performance, documentation, and end-to-end
      verification."
proposed_by: bbugyi200.athena.08h
bead_id: sase-rd
create_time: 2026-09-09 19:51:33
---

- **PROMPT:**
  [prompts/202608/snippets_panel.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/snippets_panel.md)
- **BEAD:**
  [sase-rd](https://github.com/sase-org/sase--beads/blob/main/pages/sase-rd/README.md)

# Plan: Snippet catalog, CLI, and ACE panel

## Outcome and product contract

Deliver a single snippet feature rather than separate CLI and TUI implementations:

- `sase snippet` defaults to `sase snippet list` and exposes the requested `add`,
  `delete`, `list`, and `show` subcommands.
- ACE gains a full-screen `SnippetsPanel`, opened from the prompt input with `gT` in
  NORMAL mode and `Ctrl+G T` in INSERT mode. The prompt pane, vim mode, selection, and
  cursor are restored exactly when the panel closes.
- The command, panel, prompt completion cache, editor helper, and existing prompt
  snippet save flow share the same catalog composition and mutation primitives. No
  subprocess calls from the panel and no second interpretation of snippet syntax are
  introduced.
- The catalog is project-context aware. It composes the selected project's xprompt
  snippets with the actual ordered configuration layers that apply to that workspace;
  explicit `ace.snippets` definitions override xprompt-derived snippets exactly as
  prompt expansion does today. CLI `-p/--project` and the panel's project ring resolve
  display names, aliases, and project keys but render configured project names to users.
- The main rail contains each effective explicit trigger once. Generated initial-capital
  aliases are visible as metadata on their source entry rather than doubling the rail;
  an explicitly authored capitalized trigger remains its own entry. Shadowed definitions
  remain visible in a source/override section so deletion never appears to mysteriously
  resurrect a definition.
- Each detail card keeps the authored template and composed expansion distinct. It shows
  trigger/source badges, tabstops, runtime aliases, source path, xprompt origin,
  shadows/shadowed-by information, and a bounded composed preview. Xprompt-derived
  entries are viewable and linkable but are source-edited because converting their
  generated template back into xprompt/Jinja source would be lossy.
- Authored `#[trigger]`, `#[trigger(value)]`, and `#[trigger:value]` calls become
  first-class outbound links; backlinks are computed from the same graph. Alias calls
  resolve to the canonical explicit entry for navigation. Missing targets and cycles
  remain visible as non-followable diagnostics instead of silently disappearing.

### Panel interaction design

Follow the Glossary panel's proven two-column visual language while adapting it to
snippet semantics:

- The left rail is alphabetic and width-clamped, with compact source/read-only/link
  indicators. `/` filters triggers and source labels; `.` toggles matching the raw and
  composed bodies. Highlight movement paints immediately and detail updates are
  debounced.
- The right card renders the raw template without Markdown reinterpretation, highlights
  `$0`/`$N` tabstops and `#[...]` call sites, and presents a separate composed preview.
  `CALLS` and `CALLED BY` chip rows use `Tab`/`Shift+Tab` to select, `Enter` or `l` to
  follow, `1` through `9` for direct travel, and `Backspace` or `h` for a bounded
  breadcrumb trail. Following a hidden match clears the filter with a toast, matching
  Glossary travel behavior.
- Familiar focused bindings remain: `j`/`k`, `g`/`G`, `Ctrl+D`/`Ctrl+U`, `p`/`P` project
  cycling, `a` add, `e` edit, `d` delete, `o` open source, `Z` view source, `y` copy raw
  template, `Y` copy path, `r` refresh, `?` help, and `Esc` close. Add a dedicated
  configurable `ace.keymaps.snippets` scope and update `src/sase/default_config.yml`,
  the keymap registry/types/bindings/help surfaces, and validation tests together.
- `a` opens a vim-aware trigger/template form with live trigger and link diagnostics,
  destination cycling, raw/composed preview, and explicit collision wording. `e`
  preloads an authored config template and its source fingerprint; on an xprompt entry
  it opens the real source at the definition. `d` uses a danger confirmation naming
  backlinks, the exact file being changed, and any lower-priority definition that will
  become effective.

### CLI contract

Keep help text complete, examples practical, subcommands/options alphabetized, and every
public long option paired with a short alias:

- `sase snippet add TRIGGER TEMPLATE` validates a nonblank alphanumeric/underscore
  trigger and nonblank template, targets the resolved `ace.snippet_config_path` by
  default, supports `-t/--target` for an explicit writable YAML destination,
  `-p/--project`, `-n/--dry-run`, and `-f/--format rich|json`. It refuses an accidental
  overwrite or shadow unless `-F/--force` is given and explains which source currently
  wins.
- `sase snippet delete TRIGGER` resolves exact triggers before unambiguous prefixes and
  maps runtime aliases back to their explicit source. It removes the winning writable
  `ace.snippets` contribution, supports `-a/--all` to remove all writable config-layer
  contributions, `-n/--dry-run`, `-p/--project`, and `-f/--format rich|json`, and
  reports the restore command plus any newly revealed definition. It refuses to pretend
  that a read-only/plugin/xprompt-derived entry was deleted and points to its source
  instead.
- `sase snippet list [PATTERN]` supports `-d/--definitions` matching and
  `-f/--format table|names|json`, plus `-p/--project`. The Rich table includes trigger,
  origin, calls, backlinks, and a compact raw-template summary; JSON is deterministic
  and includes raw/composed templates, provenance, aliases, relations, shadows, and
  diagnostics.
- `sase snippet show TRIGGER` supports `-f/--format rich|markdown|json` and
  `-p/--project`. It presents one entry's raw/composed definition, source stack,
  aliases, calls, backlinks, unresolved calls, and cycle diagnostics. JSON field names
  and ordering are tested as an integration contract.

Renaming triggers and lossily editing generated xprompt templates in place are out of
scope. Users can add a config override or open the xprompt source; the UI must make that
distinction obvious.

## Phase 1: Rust snippet relation and validation contract

Work in the linked `sase-core` repository after opening it through the repository
workflow.

1. Extend the snippet-domain API around `crates/sase_core/src/xprompt_catalog.rs`
   without removing the existing `templates` or `alias_provenance` results. Add typed,
   deterministic metadata for:
   - explicit-trigger validation;
   - generated alias to explicit-source identity;
   - each syntactically valid snippet call, including authored target, canonical target
     when resolvable, positional arguments, source span, and resolution status;
   - ordered outbound and reverse-reference indexes; and
   - diagnostics for missing targets and direct/indirect cycles.

   Reuse the existing xprompt reference parser and snippet resolver so calls with
   parentheses, colon arguments, quoting, escaping, and boundary rules cannot drift from
   expansion. Graph analysis must inspect raw explicit templates before expansion
   removes call sites, and alias targets must land on the explicit identity used by the
   panel trail.

2. Preserve current expansion behavior and ordering. Existing callers that only consume
   final templates continue to work, unresolved or cyclic text remains safely literal as
   today, and explicitly authored capitalized names continue to win over generated
   aliases.

3. Export the richer contract through `sase_core` and `sase_core_py`; update the PyO3
   plain-dict shape and binding documentation/tests. Prefer adding backward-compatible
   fields to the composer response (or a companion analysis entry point if compatibility
   demands it) over a frontend-only scanner.

4. Add Rust golden cases for nested calls, positional forms, quoted delimiters,
   alias-target calls, duplicate calls, self/indirect cycles, missing targets,
   explicitly authored capitalized triggers, Unicode template text, and deterministic
   inbound ordering. Cover the binding shape as well as the core graph.

5. Run the linked core repository's formatter, clippy, and tests (`just check` or its
   equivalent) before handing off.

## Phase 2: Project-aware snippet catalog and mutation service

1. Create a focused `src/sase/snippet/` package analogous to `src/sase/glossary/`.
   Define immutable models for catalog context, effective entry, source contribution,
   relations, diagnostics, and mutation outcomes. Expand
   `src/sase/core/snippet_catalog_facade.py` to validate every new Rust field and fail
   loudly on malformed binding payloads.

2. Build one catalog loader that:
   - resolves a project ref/workspace without changing process CWD;
   - loads xprompt-derived snippets with their metadata;
   - replays the real default/plugin/user/overlay/project config-layer order with source
     paths, writable/read-only state, and shadow provenance;
   - validates config shapes and trigger/template types while retaining actionable
     diagnostics per source;
   - overlays explicit config snippets on xprompt entries, then delegates alias,
     composition, and graph semantics to Rust; and
   - returns stable alphabetic entries plus canonical exact/alias/unique-prefix lookup.

   Fix `load_snippet_config_locations(project)` so its project argument is meaningful,
   and make the editor helper and ACE prompt catalog consume this service (or a thin
   projection of it) so LSP, prompt expansion, CLI, and panel agree.

3. Consolidate writes behind shared add/update/delete operations. Extend the
   source-preserving YAML machinery in `snippet_config_yaml.py` with pure preview/apply
   support for insertion, replacement, and deletion. Preserve comments, scalar bodies,
   newline style, unrelated config, and sorted sections; write atomically with an
   expected-content digest so stale panel edits raise a typed conflict. Validate the
   entire candidate snippet set through the Rust contract before replacement.

4. Resolve destinations through the existing configured-target/chezmoi rules and return
   read path, write path, apply target, source kind, created/updated/deleted identity,
   restore command, affected backlinks, and the definition revealed after delete.
   Invalidate config/save indexes only after a successful write. Rewire the existing
   prompt-pane `write_snippet_sync` wrapper to the new mutation primitive so old and new
   authoring surfaces cannot diverge.

5. Test config precedence, named projects outside the caller CWD, invalid layers,
   xprompt/config collisions, aliases, shadow chains, custom targets, chezmoi targets,
   minimal YAML diffs, stale-write conflicts, atomic failure, delete/reveal behavior,
   lookup ambiguity, and parity among the shared catalog's prompt/editor projections.
   Run `just install` before the main repository's focused tests and `just check`.

## Phase 3: `sase snippet` command group

1. Add a lazy parser registrar and entry dispatcher (`parser_snippet.py` and a focused
   handler) using the central default-to-`list` behavior. Implement the four command
   modules under `src/sase/snippet/`; do not duplicate config loading, reference
   analysis, lookup, or mutation rules in argparse handlers.

2. Implement the CLI contract above. Rich output should use the established palette,
   clear badges/chips, compact summaries, and actionable empty/error states. Names and
   JSON output must remain color-free and pipe-safe. Error prefixes name the exact
   subcommand and nonzero exits distinguish invalid context/lookup/write failures.

3. For dry runs, execute the same validation and candidate planning as a real write but
   do not touch the destination, config caches, chezmoi, or git. Rich/JSON write results
   state whether an add creates, replaces, or shadows; delete results identify every
   removed source and what becomes effective next.

4. Add parser/help ordering tests, bare-group delegation coverage, dispatch tests, and
   unit/integration tests for every format and failure path. Include shell-quotable
   restore-command round trips for multiline templates and target paths.

5. Update the relevant command and snippet documentation with examples and the exact
   authored-versus-derived semantics. Run `just check`.

## Phase 4: Snippets panel browsing and relation travel

Keep the panel unregistered from prompt keymaps in this phase so partial UI work is not
user-reaching.

1. Add an off-event-loop panel catalog service patterned after
   `glossary_panel_catalog.py`: an ordered project ring seeded from the launch
   workspace, a small mtime/token-keyed snapshot LRU, explicit invalidation, reverse
   relations from the shared catalog, and per-project selection memory. Broken projects
   produce diagnostics in place and never remove healthy contexts from the ring.

2. Implement `SnippetsPanel` as small state, rendering, navigation, travel, load, help,
   and source-view modules rather than one oversized modal. Reuse
   `ProgrammaticSelectionGuard`, `DetailPanelDebouncer`, existing source open/view/copy
   actions, conditional footers, bounded trails, and Glossary's stale worker checks. All
   disk/config/xprompt/Rust work runs in thread workers; event handlers and render paths
   consume immutable snapshots only.

3. Build pure Rich render helpers for the adaptive rail, header/footer, empty and
   diagnostic states, raw-template syntax accents, composed preview, source stack, alias
   badges, and numbered `CALLS`/`CALLED BY` chips. Clamp the rail to preserve a
   comfortable preview width and handle narrow terminals, multiline bodies, long
   triggers, and paths without layout overflow.

4. Implement filter, project cycling, refresh, link selection/follow/back, direct
   digits, source open/view, and copy actions. Selection identity—not row number—must be
   revalidated after asynchronous loads and when filters clear.

5. Add a `SnippetPanelKeymaps` scope and binding/help builders but do not yet add the
   prompt-open continuation. Cover pure rendering, state selection, filter behavior,
   stale worker results, project switching, aliases, unresolved/cyclic calls,
   outbound/inbound ordering, filtered travel, trail bounds, programmatic-highlight
   echoes, and source actions. Run `just check`.

## Phase 5: Panel CRUD, prompt entry, and release polish

1. Add the add/edit form and delete confirmation described above. Forms use the shared
   validator and catalog planner for debounced feedback; disk reads, config parsing,
   previews, and writes never run on Textual's event loop or serial pump callbacks.
   Submit mutations through the app's tracked session-worker API with one exclusive
   scope per project/destination and register the producer in `proc_producer_sites.py`.

2. On success, atomically replace the panel snapshot, preserve/reselect the stable
   trigger or nearest neighbor, clear an excluding filter with a toast, rebuild relation
   chips, and route prompt/editor catalog invalidation through the established fast
   path. Reuse the existing commit/push and scoped chezmoi-apply offer. A delete removes
   pending session overlays as well as adding/updating them, so every mounted prompt
   observes the same effective catalog immediately. On conflict, retain the draft and
   offer reload; on failure, keep the panel open and unchanged.

3. Add `SnippetPanelRequested` and the `T` continuation to the declarative prompt
   `g`-prefix table. In NORMAL mode `gT` and in INSERT mode `Ctrl+G T` open the panel;
   lowercase `gt`/`Ctrl+G t` continues to open the existing snippet target pane. Seed
   the panel from a bare snippet trigger or `#[trigger]` under the cursor when one can
   be resolved without I/O; otherwise select the first visible entry. App-layer wiring
   captures/restores prompt focus, mode, and cursor like the Glossary panel.

4. Add dark/light PNG snapshots for populated, empty, diagnostic, relation-focused,
   add/edit, and delete-impact states. Inspect actual/expected/diff artifacts before
   accepting intentional goldens. Update `docs/ace.md`, `docs/prompt.md`,
   `docs/xprompt.md`, `docs/configuration.md`, and keymap/help references, including
   source mutability, aliases, shadow/reveal behavior, and every CLI format.

5. Add end-to-end TUI tests covering both prompt key sequences, lowercase/uppercase
   coexistence, seeded entry selection, focus restoration, tracked add/edit/delete,
   session-live expansion after every mutation, xprompt source editing, conflict
   recovery, project changes, and teardown cancellation. Run the j/k performance bench
   or trace with a large synthetic catalog and retain the established p95 under 16 ms;
   verify that no render or keystroke path stats/globs/parses config.

6. Run `just install`, `just test-visual`, and `just check`. Because the combined epic
   changes the Rust/Python boundary, CLI registration, default keymaps, prompt catalog,
   and TUI, finish with `just check-full` through `/sase_monitor`, using the required
   `TESTING`/`TESTED` statuses and a follow-up action that diagnoses any failure. Run
   the linked core repository's full check again if later phases touched its contract.

## Cross-phase invariants and acceptance criteria

- There is one parser for `#[...]` calls and one composition/alias graph. CLI, panel,
  ACE expansion, helper bridge, and LSP projections agree on fixtures byte-for-byte.
- Add/edit/delete plans are source-specific, previewable, stale-write guarded, atomic,
  and comment-preserving. No mutation silently chooses a fallback destination,
  overwrites a collision, deletes a derived/read-only source, or hides a newly revealed
  definition.
- No file/config/Rust loading occurs on the Textual event loop, render path, prompt
  keystroke path, or serial pump callback. Worker results are generation/context checked
  before UI mutation and workers/tasks are cancelled at teardown.
- The panel remains hidden until Phase 5 makes browsing and CRUD complete, so an
  intermediate phase does not expose unfinished user behavior. The CLI is registered
  only when all four subcommands are complete.
- Generated aliases are never persisted, direct explicit capitalized definitions keep
  precedence, cycles never recurse indefinitely, missing links remain diagnosable, and
  deleting one layer recomputes both expansion and backlinks against the revealed
  catalog.
- Help, docs, unit tests, integration tests, Rust tests, visual snapshots, and measured
  navigation performance cover the feature's public behavior before landing.
