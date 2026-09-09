---
tier: epic
status: done
title: Repo mentions in the prompt — highlight, preview, and jump
goal: "Repo names typed into any ACE prompt input (`sase-core`, `chezmoi`,
  `gh:owner/repo`) are highlighted in their own color with the same underlined link
  affordance glossary terms already use, and `K` / `Ctrl+]` open a repo card and the
  repo itself.

  "
phases:
  - id: catalog
    title: Repo mention catalog
    depends_on: []
    size: medium
    description:
      "catalog: build the project-scoped repo mention catalog — identifier selection
      rules, the Rust-compiled matcher, the exact-identifier and path-adjacency span
      filters, and config declaration ranges."
  - id: highlight
    title: Prompt highlighting
    depends_on:
      - catalog
    size: medium
    description:
      "highlight: warm the catalog off the render path from the ACE app and overlay repo
      mentions in the prompt with a distinct bold underlined lavender style, including a
      PNG snapshot proving it reads apart from the glossary blue."
  - id: preview
    title: K opens the repo card
    depends_on:
      - highlight
    size: medium
    description:
      "preview: add the repo preview card `K` opens for the mention under the cursor,
      with kind/description/checkout/clone/remote/declaration detail and copy, edit, and
      view actions."
  - id: jump
    title: Ctrl+] opens the repo
    depends_on:
      - preview
    size: medium
    description:
      "jump: make `Ctrl+]` on a repo mention open that repo's checkout in the editor or
      a tmux pane, add a declaration choice to the jump action chooser, and handle repos
      that are not cloned yet."
proposed_by: bbugyi200.athena.059
bead_id: sase-p2
create_time: 2026-09-09 19:51:16
---

- **PROMPT:**
  [prompts/202608/prompt_repo_mentions.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/prompt_repo_mentions.md)
- **BEAD:**
  [sase-p2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-p2/README.md)

# Plan: Repo mentions in the prompt

## Goal

A repo name is one of the most load-bearing tokens a SASE prompt can contain, and today
it is dead text. After this epic, writing `sase-core` in a prompt input lights it up the
same way `Agent Hood` does — bold, underlined, and clearly a thing you can act on — but
in its own color, and with `K` and `Ctrl+]` bound to the two questions you actually have
about a repo: _what is it?_ and _take me there_.

The existing project-glossary overlay (`src/sase/ace/tui/widgets/_prompt_glossary.py`,
`src/sase/xprompt/glossary_catalog.py`,
`src/sase/ace/tui/actions/_startup_prompt_catalog.py`) is the template. This epic builds
a deliberately parallel repo-mention stack next to it rather than refactoring the
glossary stack into a generic engine: the two have different sources, different
invalidation, and different cards, and a shared abstraction would be speculative today.
Consolidation stays available later.

## Design decisions

These decisions are settled. Implement them as written; if a phase discovers one is
wrong, record a `PROPOSED FOLLOW-UP:` note on its own bead rather than redesigning
mid-epic.

### 1. What counts as a repo mention

The catalog is built from `collect_repo_inventory(project=<resolved project>)`
(`src/sase/repo_inventory.py`), which already returns `primary`, `sidecar`, `linked`,
and `external` records. Exactly one identifier is admitted per record:

| Record kind | Identifier admitted          | Example                    |
| ----------- | ---------------------------- | -------------------------- |
| `linked`    | `record.name`                | `sase-core`, `chezmoi`     |
| `sidecar`   | `record.slug` (never `name`) | `sase--beads`              |
| `external`  | `record.name`                | `gh:bbugyi200/bugyi-chops` |
| `primary`   | _nothing_                    | —                          |

The rule in one sentence: **highlight the unambiguous identifier form of every
non-primary repo.** The exclusions are the whole point of the rule:

- A sidecar's `name` is its _role_ (`agents`, `beads`, `plans`, `research`). Those are
  ordinary English words and core SASE vocabulary; highlighting them would light up half
  of every prompt. The sidecar's slug (`sase--beads`) is unambiguous, so that is what
  matches.
- The primary repo's name is the project name (`sase`), which is ambient in every prompt
  written against that project and is already project vocabulary.
- A name the project glossary already claims (as a term or an effective alias, compared
  case-folded) is dropped from the repo catalog. The glossary is authored vocabulary and
  wins; this keeps the two overlays from ever fighting over the same characters.

Other SASE projects' primary repos (`sase repo open dotdrop`) are **out of scope**: that
catalog is global rather than project-scoped, and project names already have their own
completion and token surfaces.

### 2. Matching semantics come from the Rust core, not from new Python

`crates/sase_core/src/glossary.rs` in the sibling `sase-core` repo already owns exactly
the matcher this feature needs: case-insensitive, word-boundary-anchored (`is_word_char`
counts `-` and `_`, so `sase-core` never matches inside `sase-core-extras`, and the
glossary term `sase` never matches inside `sase-core`), literal-zone aware (inline and
fenced code are skipped), and longest-match-wins on overlap.

So the repo catalog **feeds repo identifiers into that matcher as glossary input
entries** (`GlossaryInputEntry` → `compile_glossary_catalog` → `scan_glossary_spans` /
`lookup_glossary_span`, all re-exported from `src/sase/core/glossary_facade.py`).
Matching logic stays Rust-owned per the `rust_core_backend_boundary` rule, the two
overlays behave identically by construction, and this epic needs no `sase-core` release
or version-pin bump.

Two consequences must be handled in Python, because the Rust matcher is tuned for prose
terms rather than identifiers:

- **Blank definitions are a validation error.** Repos without a description get a
  synthesized one (see phase `catalog`), which the card renders anyway.
- **The matcher derives plurals.** `chezmoi` would also match `chezmois`. The scan
  wrapper therefore drops any span whose matched text, case-folded, is not _exactly_ an
  admitted identifier. This is a single equality check, it kills derived plurals without
  disabling case-insensitivity, and it makes the highlighted characters always equal to
  a real repo identifier.

A first-class `repo_mentions` API in `sase-core` is the right long-term home if the
xprompt LSP ever wants repo hovers. That is a follow-up, not this epic.

### 3. Path context suppresses a mention

`../sase-core` and `sase/repos/linked/sase-core` are paths, not references, and the
file-path hint overlay already owns them. A candidate span is rejected when the
character immediately **before** it is `/`, `\`, or `.`, or the character immediately
**after** it is `/` or `\`. A trailing `.` is deliberately still a match so
`Rebuild sase-core.` at the end of a sentence highlights.

### 4. Color and affordance

Repo mentions get the same _shape_ as glossary terms — bold plus underline, the
established "this is a definable thing you can open" affordance — in a different hue, so
the two read as siblings rather than as unrelated decorations:

- glossary (`glossary.term`): `derive_argument_color(theme.primary, …)` — muted blue,
  `#799DC3` under the pinned flexoki theme.
- repo mentions (`repo.mention`): `derive_argument_color(theme.accent, …)` — lavender,
  `#C3ABD8` under flexoki.

Accent-derived lavender was chosen over the alternatives on purpose: `theme.secondary`
derives to a teal (`#7BB3A9`) that sits too close to the glossary blue to be reliably
distinguishable in a terminal, and `theme.warning` already reads as "something is wrong"
(`jinja.unknown` is warning + underline). The lavender exactly matches
`artifact_ref.fragment`, which is an accepted trade-off: fragments only ever render
inside `@kind:payload#fragment` tokens, and structural overlays win those characters
anyway.

A double underline (`Style(underline2=True)`) was considered for extra separation and
rejected: terminals that do not implement SGR 21 drop the underline entirely, which
would silently delete the affordance.

### 5. Overlay precedence

Each highlight mixin calls `super()._build_highlight_map()` _before_ appending its own
spans, so a class listed **earlier** in `PromptTextArea`'s bases appends later and wins
on overlap. `PromptRepoMentionMixin` therefore goes immediately **after**
`PromptGlossaryMixin` in that list, which yields:

structural overlays (xprompt syntax, artifact refs, code blocks, placeholders) → beat →
glossary terms → beat → repo mentions → beat → misspelling squiggles.

Glossary beating repo mentions is belt-and-braces only; rule 1 already removes
glossary-claimed names from the repo catalog. Repo mentions beating the misspelling
squiggle is the point: `sase-core` is not a typo.

The same precedence governs `K` and `Ctrl+]` fallthrough. Both actions try, in order:
structural token → glossary → **repo mention** → word lookup → shorthand argument owner.

### 6. `K` versus `Ctrl+]`

For a glossary term both keys lead to the definition. For a repo the two questions are
genuinely different, and the split is what makes this intuitive:

- **`K` — what is it?** A repo card: kind, description, checkout path, whether it is
  cloned here, clone coverage, remote URL, env var, and where it is declared.
- **`Ctrl+]` — take me there.** The repo's checkout, opened in `$EDITOR` or a new tmux
  pane through the existing jump-action chooser, with an added `config` choice for the
  declaration site (which is where `Ctrl+]` on a glossary term goes).

### 7. Safety: never materialize a checkout from the TUI

`sase repo open` cleans the checkout it prepares and can discard untracked files.
Nothing in this epic may invoke it. When a repo is not cloned in the active workspace,
the card and the jump both say so and print the exact command for the user to run
themselves.

### 8. No feature flag

Per `sase/memory/sase_flags.md`, a flag routes user-reaching behavior that is not ready
to be unconditional. Every phase here lands a complete, coherent state: highlighting
alone is useful and correct, and each key binding is additive fallthrough that cannot
regress an existing target. The glossary feature itself landed unflagged in exactly this
order. Do not create a flag; do not add a config toggle either, since nothing here is a
permanent user choice.

## Shared constraints for every phase

- Run `just install` before anything else — workspaces are ephemeral and may have stale
  virtualenvs.
- Nothing in the prompt render path may perform I/O, project resolution, or schedule
  work. Catalog loading happens in a worker; `_build_highlight_map` only reads
  already-warm memory state, exactly as the glossary mixin does.
- Reuse the glossary mixin's overlay budget guards (`_MAX_OVERLAY_BYTES`,
  `_MAX_OVERLAY_LINES` from `src/sase/ace/tui/widgets/_jinja_highlight.py`) and its span
  cache keyed by `(compiled catalog, text)`.
- Do not edit `sase/memory/*.md`, `AGENTS.md`, or the generated provider instruction
  shims. Editing `docs/` and the in-app help modal is required and is not covered by
  that rule.
- `src/sase/ace/CLAUDE.md` requires the `?` help popup to stay in sync with any
  `sase ace` behavior change. The phases below name the exact rows to add.
- Treat all agent runtimes uniformly; no runtime-specific branching.
- Every phase ends with `just check`. The final phase (`jump`) additionally runs
  `just check-full` through `/sase_monitor` (never inline) with a `--next` action,
  because it outruns a single turn.

## Catalog: repo mention catalog

**New:** `src/sase/xprompt/repo_mention_catalog.py` **New tests:**
`tests/xprompt/test_repo_mention_catalog.py`

Model this module on `src/sase/xprompt/glossary_catalog.py`, including its
project-selection helpers (`_enabled_project_records`, `_select_project`) — reuse them
by import if they are importable as-is, otherwise mirror them.

Public surface:

- `RepoMention` (frozen dataclass): `identifier`, `kind` (`RepoKind`), `record`
  (`RepoRecord`), `index` (matcher entry index), `config_path` (`str | None`),
  `config_line` / `config_col` (1-based, `None` when unknown).
- `EditorRepoMentionCatalog` (frozen dataclass): `schema_version`, `project` (reuse the
  glossary module's resolved-project record shape), `mentions`
  (`tuple[RepoMention, ...]`), `compiled` (the `CompiledGlossaryCatalog` handle), and a
  `mention_for_index(index)` lookup.
- `EditorRepoMentionCatalogResult`: `project`, `catalog`, `diagnostics`.
- `editor_repo_mention_catalog_for_project(project_ref=None, *, launch_workspace=None, projects_root=None)`.
- `scan_repo_mentions(catalog, text) -> tuple[RepoMentionSpan, ...]`
- `lookup_repo_mention(catalog, text, *, line, character) -> RepoMentionSpan | None`

`RepoMentionSpan` wraps the underlying `GlossarySpan` and adds the resolved
`RepoMention`, so callers never touch glossary types.

Build steps inside `editor_repo_mention_catalog_for_project`:

1. Resolve the project the same way the glossary catalog does (explicit `project_ref`,
   else the launch workspace, else CWD). Return a result with a diagnostic and no
   catalog when nothing resolves.
2. `collect_repo_inventory(project=<project key or name>)`. Wrap in `try/except` and
   degrade to a diagnostic — a broken inventory must never raise into a prompt keystroke
   path.
3. Admit identifiers per design decision 1. Drop blanks, drop identifiers containing
   whitespace or a line break, and dedupe case-folded keeping the first (inventory order
   is already stable and kind-ordered).
4. Load the project's glossary catalog via
   `editor_glossary_catalog_for_project(project_ref, launch_workspace=…)` and drop any
   identifier whose case-folded form appears in any entry's `effective_aliases` or
   `term`. A failed glossary load means "no exclusions" plus a diagnostic, never a
   failed repo catalog.
5. Compile via `compile_glossary_catalog([...])`, one `GlossaryInputEntry` per
   identifier with `term=identifier`, `aliases=()`, and a synthesized `definition`:
   - description present → the description verbatim;
   - otherwise `"<kind> repository at <path>."`, e.g.
     `"External repository at /…/repos/external/gh__owner__repo."`. Compilation failure
     (a validation error the rules above did not anticipate) degrades to a diagnostic
     and no catalog.
6. Resolve declaration ranges. Round-trip parse the project's config with
   `sase.config._edit_yaml_io.make_yaml` (the mechanism
   `src/sase/xprompt/glossary_catalog.py::_glossary_source` uses) and record the 1-based
   line/col of:
   - `repos.linked[i]` for the sequence item whose `name` matches;
   - `repos.sidecar.builtin.<role>` / `repos.sidecar.custom.<role>` key for a sidecar;
   - nothing for externals — they are recorded in workspace marker files, not config, so
     `config_path` stays `None`. Any parse failure leaves the ranges `None`; this is
     metadata, never a hard error.

`scan_repo_mentions` / `lookup_repo_mention` apply, in order: the Rust scan, then the
**exact-identifier filter** (design decision 2) and the **path-adjacency filter**
(design decision 3). Both filters must live here so every future frontend inherits them.

Tests must cover: sidecar slug admitted while its bare role is not; primary excluded;
linked and external admitted; a glossary-claimed name excluded; a repo with no
description getting a synthesized one; `sase-core` not matching inside
`sase-core-extras` and `sase` not matching inside `sase-core`; the derived plural
`chezmois` rejected; `Sase-Core` matched case-insensitively; `../sase-core`,
`sase/repos/linked/sase-core`, and `sase-core/crates` all rejected while
`Rebuild sase-core.` matches; fenced and inline code skipped; declaration line/col
resolved for a linked entry and for a sidecar role; inventory failure and glossary-load
failure both degrading to diagnostics.

Build inventory fixtures directly as `RepoRecord` values and monkeypatch
`collect_repo_inventory`; do not stand up real projects on disk.

## Highlight: prompt highlighting

**New:** `src/sase/ace/tui/repo_mention_catalog.py` —
`PromptRepoMentionContext(project_ref, launch_workspace)` and
`load_prompt_repo_mention_context(context, *, generation)`, mirroring
`src/sase/ace/tui/glossary_catalog.py` exactly. The workspace _number_ is deliberately
not part of the context: it would multiply cache entries, and the clone a card or jump
wants is resolved at action time instead.

**New:** `src/sase/ace/tui/widgets/_prompt_repo_mentions.py` — `PromptRepoMentionMixin`,
modeled on `PromptGlossaryMixin`:

- `on_mount` / `_app_theme_changed` / `_register_jinja_text_area_theme` register the
  `repo.mention` style on the active theme and on `_JINJA_THEME_NAME`.
- `_on_prompt_completion_context_changed` refreshes the context (chaining `super()`
  first, as the glossary mixin does).
- `_build_highlight_map` appends `repo.mention` spans for every segment of every span
  returned by `scan_repo_mentions`, guarded by the shared overlay budgets and the
  cached-scan check.
- `_repo_mention_under_cursor(*, schedule: bool)` returns
  `(catalog, span) | _ColdRepoCatalog | None`, the same three-state shape the glossary
  mixin uses so `preview` and `jump` can both consume it.
- Style:
  `Style(color=derive_argument_color(app_theme.accent, foreground=…, background=…), bold=True, underline=True)`.

**Changed:** `src/sase/ace/tui/widgets/prompt_text_area.py` — insert
`PromptRepoMentionMixin` immediately after `PromptGlossaryMixin` in the bases list
(design decision 5), and add the per-instance caches to `__init__` if they are not
initialized inside the mixin's own `__init__`.

**Changed:** `src/sase/ace/tui/actions/_startup_prompt_catalog.py` — add
`get_prompt_repo_mention_catalog`, `is_prompt_repo_mention_catalog_warm`,
`warm_prompt_repo_mention_catalog`, `_run_prompt_repo_mention_warm`,
`_invalidate_prompt_repo_mention_catalogs`, and
`_refresh_visible_prompt_repo_mention_surfaces`, each a direct analogue of the glossary
methods, using worker group `prompt-repo-mentions`.

**Changed:** `src/sase/ace/tui/actions/_state_init_late.py` and
`src/sase/ace/tui/actions/startup.py` — initialize and type the four new app attributes
alongside the `_prompt_glossary_*` ones.

**Changed:** `src/sase/ace/tui/actions/_startup_watchers.py` and
`_request_prompt_catalog_config_refresh` — invalidate repo mention catalogs wherever
glossary catalogs are invalidated, so a watched `sase.yml` edit re-lights the prompt.

Known and accepted limitation to state in the docs: a repo opened with `sase repo open`
_during_ a live ACE session does not appear until the next config-driven invalidation.
Adding a marker-file signature to the cache key would put I/O back on the render path.

**Docs:** `docs/ace.md` — add a `#### Repo names` subsection immediately after
`#### Glossary terms`, describing what is highlighted, the color, the exclusion rules,
and the warm/invalidate behavior. Extend the glossary paragraph's sentence about the
underline affordance to name both colors so a reader can tell them apart.

**Tests:**

- `tests/ace/tui/widgets/test_prompt_repo_mention_highlighting.py` plus a
  `_prompt_repo_mention_helpers.py` mirroring
  `tests/ace/tui/widgets/_prompt_glossary_helpers.py`: spans appear for a warm catalog;
  nothing is scanned when the catalog is cold; a cold catalog schedules exactly one
  warm; overlay budgets short-circuit; the span cache is not rebuilt when neither text
  nor catalog changed; a glossary term and a repo mention in the same buffer each keep
  their own style.
- `tests/ace/tui/visual/test_ace_png_snapshots_prompt_highlighting.py` — add
  `test_prompt_repo_mention_highlight_png_snapshot` with a buffer containing a glossary
  term and a repo mention on the same line, plus a `patch_visual_repo_mention_catalog`
  helper in `tests/ace/tui/visual/_ace_prompt_png_snapshot_helpers.py`. Generate the
  golden with `just test-visual --sase-update-visual-snapshots` and **look at the PNG**:
  if the two colors are not obviously different, say so on the bead instead of accepting
  the golden.

## Preview: K opens the repo card

**Changed:** `src/sase/ace/tui/widgets/_prompt_preview.py` — after the glossary attempt
and before `_lookup_word_under_cursor`, try `_preview_repo_mention_under_cursor`. Update
the fallthrough warning text to mention repo names.

**Changed:** `src/sase/ace/tui/widgets/_prompt_repo_mentions.py` — add
`_preview_repo_mention_under_cursor() -> bool`, which pushes the card, or notifies
`"Repo catalog is still loading; try again"` (returning `True`) on a cold catalog so `K`
never falls through to an unrelated target.

**New:** `src/sase/ace/tui/modals/repo_preview_modal.py` (`RepoPreviewModal`) and
`src/sase/ace/tui/modals/repo_preview_render.py`, modeled on the glossary pair including
`CopyModeForwardingMixin` and `SourceFileActionsMixin`.

Card layout:

```
R REPO  sase-core                                                sase
Shared Rust core backend for SASE domain behavior and cross-frontend APIs.

 LINKED   AUTO-CLONE   ENV SASE_CORE
──────────────────────────────────────────────────────────────────
Project     sase
Kind        linked
Checkout    …/sase/repos/linked/sase-core
Clones      18 of 24 workspaces
Remote      git@github.com:bbugyi200/sase-core.git
Declared    …/sase/sase.yml:216:7
```

- Accent is the same accent-derived lavender as the highlight, via a
  `repo_card_accent(theme)` helper mirroring `glossary_card_accent`.
- Icon `R`, label `REPO`, right-aligned project name, matched text disclosed only when
  it differs in case from the identifier.
- Chips: the kind, then `AUTO-CLONE` / `AUTO-SYNC` when set, then `ENV <name>` when the
  record has one. Omit rows whose value is unknown rather than printing empty cells.
- `Checkout` prefers the clone registered for the active workspace
  (`RepoRecord.clone_for_workspace(app._prompt_context.workspace_num)`), else
  `record.path`; it appends ` (not cloned)` when the path does not exist, and the card
  then adds a hint line: ``Run `sase repo open <name>` to materialize this checkout.``
  Never run that command from the TUI.
- `Clones` renders `<existing> of <registered> workspaces`, omitted when there are no
  clone records.

Bindings: `esc`/`q` close, `y` copy the description, `p` copy the checkout path, `Y`
copy the declaration path, `o` open the declaration in `$EDITOR`, `Z` hand it to the
artifact viewer, plus the standard `j`/`k`, `Ctrl+D`/`Ctrl+U`, `g`/`G` scrolling.
`_source_action_path` / `_source_action_position` return the declaration path and range,
so `Y`/`o`/`Z` warn cleanly for an external repo that has none. Footer mirrors the
glossary card and ends with a `Ctrl+] opens the repo` hint.

**Changed:** `src/sase/ace/tui/styles.tcss` — add `RepoPreviewModal` rules alongside the
`GlossaryPreviewModal` block, matching its geometry.

**Changed:** `src/sase/ace/tui/modals/help_modal/binding_common.py` — add
`("K / Ctrl+] on repo name", "Preview repo / open checkout")` directly under the
existing glossary row. Keep the description within the 32-character budget documented in
`src/sase/ace/CLAUDE.md`.

**Docs:** `docs/ace.md` — extend the `#### Repo names` subsection with the card contents
and its key bindings, and add the repo case to the `K` row of the keymap table near the
end of the file.

**Tests:** `tests/ace/tui/widgets/test_prompt_repo_mention_navigation.py` (cold catalog
notifies and does not fall through; a hit pushes `RepoPreviewModal`; a miss returns
`False` so word lookup still runs) and
`tests/ace/tui/modals/test_repo_preview_render.py` (pure render assertions: not-cloned
suffix and hint, clone counts, omitted rows, chips, declaration display). Add a
`test_repo_preview_card_png_snapshot` beside the glossary preview snapshots.

## Jump: Ctrl+] opens the repo

**Changed:** `src/sase/ace/tui/widgets/_prompt_jump.py` — after the glossary attempt,
try `_jump_to_repo_mention_under_cursor`, and update the fallthrough warning text to
mention repo names.

**Changed:** `src/sase/ace/tui/widgets/_prompt_repo_mentions.py` — add
`_jump_to_repo_mention_under_cursor() -> bool` building a `JumpTarget` with
`kind_label="repo"`, `icon="R"`, `title=<identifier>`,
`source_path=<resolved checkout>`, `line=None`, `col=None`, `loadable_markdown=None`,
`is_editable=False`, and the new config fields below. Cold catalog notifies and returns
`True`, as in preview.

**Changed:** `src/sase/ace/tui/widgets/_prompt_jump_target.py` — add
`config_path: str | None = None`, `config_line: int | None = None`, and
`config_col: int | None = None` to `JumpTarget`. Defaults keep every existing
construction valid.

**Changed:** `src/sase/ace/tui/modals/jump_action_modal.py` — extend `JumpChoice` with
`"config"` and bind `c` to it ("Open declaration").

**Changed:** `src/sase/ace/tui/widgets/_prompt_jump.py`:

- `_jump_action_choices` appends `"config"` when `payload.config_path` is set.
- `_perform_jump_action` handles `"config"` by opening the declaration through the
  existing editor path — build the derived payload with
  `dataclasses.replace(payload, source_path=config_path, line=config_line, col=config_col)`.
- Fix `_open_jump_target_in_tmux_pane`: it currently passes
  `Path(payload.source_path).parent` as tmux's `-c`, which for a directory target lands
  one level _above_ the repo. Use the path itself when it is a directory, its parent
  otherwise. Cover this with a test — it is a real behavior fix, not an incidental edit.

Not-cloned handling: when the resolved checkout does not exist, notify
``"<name> is not cloned in this workspace; run `sase repo open <name>`"`` and offer only
the `config` choice; when there is no declaration either (an external repo), notify and
do not open a chooser at all. Never invoke `sase repo open`.

**Changed:** `src/sase/ace/tui/modals/help_modal/binding_common.py` — extend the
`Ctrl+]` row so it reads as covering repos, without exceeding the width budget.

**Docs:** `docs/ace.md` — document the `Ctrl+]` behavior (checkout vs. `c` for the
declaration, and the not-cloned message) in the `#### Repo names` subsection, and update
the `Ctrl+]` row of the keymap table.

**Tests:** extend `test_prompt_repo_mention_navigation.py` with: a cloned repo producing
a checkout payload with the config fields populated; a not-cloned repo notifying and
offering only `config`; a not-cloned external repo notifying with no chooser;
`_jump_action_choices` including `config` only when a declaration exists; the `config`
choice opening the declaration path and line; and the tmux `-c` directory fix.

**Close-out for the epic:** this phase runs `just check-full` through `/sase_monitor`
with a `--next` action, and confirms `just test-visual` passes with the goldens added by
`highlight` and `preview`.

## Out of scope

Recorded so a later reader knows these were considered and deliberately left out:

- Highlighting other SASE projects' primary repos (`dotdrop`) or unlinked GitHub repos
  that have never been opened in this project.
- A `repo_mentions` API in `sase-core` and repo hovers in the xprompt LSP.
- Completing repo names in the prompt, and fixing
  `src/sase/completion/candidates/catalog.py::_repo_candidates`, which offers only
  project primaries for `ValueKind.REPO` slots and therefore cannot complete
  `sase repo open sase-core` today. Worth a task bead; not this epic.
- Live invalidation when `sase repo open` materializes a checkout mid-session.
- Any config toggle or feature flag for the highlight (design decision 8).
