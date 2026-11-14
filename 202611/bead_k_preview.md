---
tier: epic
title: Bead previews for K in the prompt input
goal: "Pressing K in the prompt input's NORMAL mode on a bead ID (bob-cli-kx,
  sase-1ss.1) or an @bead:<id> ref opens a fast, theme-aware bead card that never blocks
  typing. The card shows the bead's identity, status, description, phases, lineage,
  dependencies, and plan, and lets you drill into related beads or open the full detail.
  Ctrl+] opens the full bead detail too. Bead IDs that resolve get a status-tinted
  underline, so the K affordance is visible before you press it. Hyphenated words that
  are not beads keep today's word lookup.

  "
decisions:
  mention_tint:
    ask: How should prompt bead IDs be colored once they are underlined?
    choices:
      status:
        Each ID takes its bead's status color (in-progress gold, closed green, open
        blue)
      uniform:
        One theme-derived bead color, matching the glossary and repo-name siblings
    default: status
    why: Status color shows at a glance whether a bead you reference is still live
    answer: status
  jump_target:
    ask: What should Ctrl+] open when the cursor is on a bead ID?
    choices:
      detail: The full bead detail document in the pager, with followable bead links
      plan:
        The bead's plan file in $EDITOR, falling back to the pager when it has no plan
    default: detail
    why: Every bead has a detail document; many beads (tasks, flags) have no plan file
    answer: detail
phases:
  - id: peek
    title: K on a bead opens a bead card
    depends_on: []
    size: medium
    description:
      "peek: recognize bare and @bead: IDs at the cursor, route them through a warm
      app-level bead store directory, splice the bead step into the K chain with a
      byte-for-byte resume of the word lookup for non-beads, and ship the
      BeadPreviewModal core (title, chips, description, property grid, copy/plan
      actions, loading state)."
  - id: relations
    title: Related beads, navigation, and full detail
    depends_on:
      - peek
    size: medium
    description:
      "relations: add lineage, a numbered phase/children list with a progress meter, and
      dependencies to the card, in-card drill-down (1-9, p, backspace, r), Enter and
      Ctrl+] opening the full bead detail in the pager, help/docs updates, and PNG
      goldens."
  - id: mentions
    title: Status-tinted bead mentions in the prompt
    depends_on:
      - peek
      - relations
    size: medium
    description:
      "mentions: underline resolved bead IDs in prompt text using a memory-only overlay
      fed by an app-level, fingerprint-revalidated mention index whose resolution runs
      only on discrete events (mount, NORMAL-mode entry, K), never per keystroke."
proposed_by: bbugyi200.apollo.dy
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-14 09:03:55
status: wip
---

- **PROMPT:**
  [prompts/202611/bead_k_preview.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/bead_k_preview.md)

# Plan: Bead previews for K in the prompt input

## Why

Today `K` in the prompt's NORMAL mode runs this chain
(`src/sase/ace/tui/widgets/_prompt_preview.py` `_preview_token_under_cursor`):
macro/skill/workflow/file → glossary term → repo mention → plain-word lookup
(spellcheck, then `dict`) → shorthand-argument owner → a "move the cursor onto…"
warning. Beads are the most common thing people reference in prompts ("See the
bob-cli-kx epic bead for context"), but:

- `K` on `bob-cli-kx` asks `dict` and toasts **"No definition found for 'bob-cli-kx'"**.
- `K` on `sase-1ss` (it has digits) falls through to the generic warning.
- `K` on `@bead:sase-1ss` looks up the English word **"bead"**.

This epic makes beads a first-class `K` target. It follows the same design language as
the glossary and repo-name previews: an underlined affordance in the prompt, a compact
card on `K`, and a jump on `Ctrl+]`.

## Design at a glance

### What counts as a bead reference

| Form                    | Example                                     | Notes                                                 |
| ----------------------- | ------------------------------------------- | ----------------------------------------------------- |
| Bare full ID            | `bob-cli-kx`, `sase-1ss.1`                  | `<prefix>-<base36 counter 1–4>(.<digits>)*`           |
| Artifact ref            | `@bead:sase-1ss`, `bead:sase-1ss`           | Cursor anywhere on the ref, including `@` and `bead:` |
| Quoted or in a sentence | `` `sase-1ss` ``, `(sase-1ss)`, `sase-1ss.` | Trailing sentence punctuation is not part of the ID   |

The following are never bead references:

- Path-adjacent or URL tokens: `src/sase-1ss/x`, `…/bob-cli-lh/README.md`,
  `sase-1ss.md`.
- Project tags (`+bob-cli`), macros (`#…`), directives (`%…`), and `~`-paths.
- Tokens containing `--` or ending in `-`.

Bare shorthand IDs (`1ss.1`) are out of scope because they are ambiguous in free text.
The directive form `%wait:sase-1ss` is allowed, because the ID follows a `:`.

### What K does

| Token under the cursor                                                            | Outcome                                                                                                                                                                       |
| --------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Bead ref or bare ID that resolves                                                 | The **bead card** opens.                                                                                                                                                      |
| Bare ID whose prefix no enabled project owns (`well-known`, `bob-cli`)            | Unchanged: today's chain runs. When the store directory is warm this happens synchronously, so non-beads see zero added latency.                                              |
| Bare ID whose prefix is owned but no such bead exists (`cash-flow`, `bob-cli-kz`) | The word path runs exactly as today, so a real definition or the spellcheck panel can still win. Only its final dead-end toast changes, to `No bead 'bob-cli-kz' in bob-cli`. |
| Ref form that does not resolve                                                    | Info toast `No bead 'sase-zzz' in sase`, or `No enabled project owns bead prefix 'foo'`. No word lookup runs.                                                                 |
| Prefix owned by several projects                                                  | The prompt's `+project` tag wins if it is one of them. Otherwise a warning names the projects and suggests adding a `+project` tag.                                           |
| Owning store unavailable                                                          | Warning `bob-cli beads are unavailable: <reason>`.                                                                                                                            |

### The card (target look, epic example)

```
▸ EPIC  bob-cli-kx                                                    bob-cli
Gmail archive on task close
◐ in progress   ●●●●◐ 4/5 phases closed
┌─────────────────────────────────────────────────────────────────────────┐
│ bob gmail sync archives a bob-labelled thread's saved messages exactly  │
│ once when every linking Obsidian task is closed, visibly, undoably, …   │
│                                                                         │
│ PHASES                                                                  │
│ 1 ● .1  Mutation-path spike                                     small   │
│ 2 ● .2  Archive-on-close machinery in bob-cli                   medium  │
│ 5 ◐ .5  Gated live enable and verification                      medium  │
│ ─────────────────────────────────────────────────────────────────────── │
│ Project   bob-cli                                                       │
│ Plan      ~/…/plans/202611/gmail_archive_on_task_close.md               │
│ Created   Nov 3 (11d ago)                 Updated   2h ago              │
│ Notes     3 · latest 2h ago                                             │
│           Phase 5 is waiting on the gated live run…                     │
└─────────────────────────────────────────────────────────────────────────┘
      j/k | y id | ⏎ detail | 1-5 phase | o plan | Z view | Y path | esc
```

Visual rules:

- The identity glyph and label use the bead **type** color (plan ▸ gold, phase ↳ blue,
  task ◆ purple; flag ⚑).
- The status chip uses the canonical **status** glyph and color from
  `bead_status_presentation`.
- Every fixed palette color passes through
  `sase.macro.highlight_theme.derive_argument_color` against the active theme's
  foreground and background. Light themes stay legible.
- The container, scroll border, and footer reuse the sibling cards' CSS shape
  (`RepoPreviewModal` / `GlossaryPreviewModal` in `src/sase/ace/tui/styles.tcss`), so
  the three cards read as one family.

### Performance and reliability contract (from the TUI perf rules)

- **Keypress path: memory only.** The only work on the keypress path is detecting the
  mention at the cursor (regex) and a prefix-ownership check against the warm directory.
  There is no disk I/O, subprocess, or lock there.
- **Off-thread work.** Routing, detail reads, plan resolution, and pager document builds
  all run in `asyncio.to_thread` inside `run_worker`.
- **Stale results are dropped.** Every async result is checked against
  `_prompt_preview_request_id` (K) or `_prompt_jump_request_id` (Ctrl+]) and against
  `is_mounted` before it touches UI. The modal's own loads use a modal-local request id.
- **Read-only, audit-free reads.** Use
  `sase.core.bead_read_facade.show_issue_detail(..., include_links=False)` and
  probe-based routing. Never call `record_bead_reads`, the `sase bead show` or
  `sase bead read` CLI paths, attachment fetches (network), or `resolve_bead_page_url`
  (runs git). A human peek is not an agent read.
- **Measured costs (this host).**
  - Building the cross-project store snapshot set (`enabled_project_store_snapshots()`)
    takes 0.3–0.6 s. That is why it is cached app-wide and pre-warmed.
  - Probe routing takes about 25 ms.
  - A warm `show_issue_detail` takes 10–50 ms, but the first read of a store in a
    process can take 1–4 s. That is why the card opens immediately in a loading state
    and fills itself.

## Phase `peek` — K on a bead opens a bead card

### Mention detection (Textual-free, no I/O)

New `src/sase/bead/mentions.py`:

- `BeadMention` (frozen): `bead_id`, `start`, `end`, `form: Literal["bare", "ref"]`, and
  `prefix`.
  - `start` and `end` are absolute text offsets that cover the whole clickable token
    (for refs, the full `@bead:<id>`).
  - `prefix` is the part before the final `-<counter>` of the top-level ID; reuse
    `sase.bead.cross_project.bead_id_prefix`.
- `bead_mention_at(text: str, offset: int) -> BeadMention | None`, implementing the
  "What counts" table above:
  - **Ref form.** Find the `@bead:`/`bead:` token around the cursor and parse it with
    `sase.artifact_ref_operations.parse_artifact_ref`. Accept only `kind_type == "bead"`
    with a non-empty `payload.id`.
  - **Bare form.** Grow the maximal `[A-Za-z0-9_.-]` run around the cursor and strip
    trailing sentence punctuation, then full-match
    `^[A-Za-z0-9][A-Za-z0-9_-]*-[0-9a-z]{1,4}(?:\.\d+)*$`.
  - **Bare-form rejections.**
    - Reject tokens containing `--` or ending in `-`.
    - Reject when the character before the token is one of `/ \ . + # % @ ~`, a word
      character, or `-`.
    - Reject when the character after it is `/`, `\`, a word character, or `-`.
    - Reject when it is followed by `.` plus a word character that is not a numeric
      phase suffix (file names such as `sase-1ss.md`).
  - **Shape rule.** The counter rule mirrors the pager's bare-ID rule in
    `src/sase/pager/link_scan.py`. Do not refactor the pager: its suffix rule is
    per-origin and intentionally looser.

### Store directory, routing, and preview loading (Textual-free, does I/O; call off-thread only)

New `src/sase/ace/tui/models/bead_preview.py`:

- `BeadStoreRef` (frozen): `project_key`, `project_label`, `beads_dir: Path`,
  `primary_workspace: Path`.
- `BeadStoreDirectory` (frozen) wraps the `tuple[BeadStoreSnapshot, ...]` from
  `sase.bead.cross_project.enabled_project_store_snapshots()`. It offers a memory-only
  `owners_for_prefix(prefix)`, which matches `issue_prefix` or `project_refs`
  casefolded, the same way `cross_project` does. It also exposes a
  `load_bead_store_directory()` factory.
- `BeadRoute` is a small union of frozen results:
  - `BeadRouteFound(bead_id, store)`
  - `BeadRouteMissing(bead_id, project_labels)`
  - `BeadRouteNotBead(bead_id)`
  - `BeadRouteAmbiguous(bead_id, project_labels)`
  - `BeadRouteUnavailable(bead_id, project_label, reason)`
- `route_bead_mention(bead_id, directory, *, project_ref: str | None) -> BeadRoute`:
  - When no store owns the prefix, return `NotBead`.
  - Otherwise call `route_full_id_with_snapshots(bead_id, directory.snapshots)` and map
    the error kinds: `not_found` → `Missing`, `ambiguous` → `Ambiguous`, `unavailable` →
    `Unavailable`.
  - When the result is ambiguous and `project_ref` names one candidate, probe that store
    with `probe_bead_target_owner` and return `Found`.
- `BeadPreview` (frozen) carries:
  - `issue` and `ancestors` (nearest first)
  - `children`, `depends_on`, and `blocks`
  - `store`
  - `plan_reference`, plus `plan_source: Literal["self", "parent"] | None`
  - `plan_path: str | None` (absolute and existing, or `None`)
- `load_bead_preview(bead_id, store) -> BeadPreview`:
  - Read the bead with
    `show_issue_detail(store.beads_dir, bead_id, include_links=False)`. A missing bead
    re-raises `KeyError`.
  - The plan reference is the bead's own `design`. For a phase it is the parent plan's
    `design`, matching `sase/bead/cli_detail_resolution.py` `_resolve_plan_link`.
  - Resolve the plan path with `canonical_plans_dir_for_project(store.project_key)` plus
    the same `resolve_plan_path` call that
    `widgets/artifacts/beads_data_sources.resolve_plan_link` uses.

### App-level warm directory cache

Add a small app mixin, `src/sase/ace/tui/actions/_startup_prompt_bead_directory.py`,
mixed in wherever `StartupPromptCatalogSemanticsMixin` is. Initialize its state next to
the existing glossary and repo-mention state. It mirrors the repo-mention catalog API
and provides:

- `get_bead_store_directory(*, schedule: bool = True) -> BeadStoreDirectory | None`,
  memory only.
- `warm_bead_store_directory()`: one in-flight worker with a generation guard.
- `publish_bead_store_directory(directory)`, for K workers that had to load the
  directory inline while it was cold.
- `_invalidate_bead_store_directory(reason=...)`, called from
  `_request_prompt_catalog_config_refresh` in `_startup_prompt_catalog_core.py` next to
  the glossary and repo invalidations.

### Prompt wiring

- **New mixin.** Add `src/sase/ace/tui/widgets/_prompt_bead_mentions.py` with a
  `PromptBeadMentionMixin` in `PromptTextArea`'s bases.
  - Its `_prompt_mount_hook` chains `super()` and warms the directory.
  - Add `_bead_store_directory_for_action()`, using the same `_active_app()` / `getattr`
    style as `_prompt_repo_mentions.py`.
- **New preview step.** Add `src/sase/ace/tui/widgets/_prompt_bead_preview.py` with a
  `PromptBeadPreviewMixin` that defines `_preview_bead_under_cursor(fallback) -> bool`.
  - **Fast path.** When the directory is warm and the mention is bare with no owning
    store, return `False`; the chain continues synchronously.
  - **Async path.** Otherwise bump `_prompt_preview_request_id` and run a worker. The
    worker loads and publishes the directory inline if it is cold, routes the ID, and
    applies the outcome table.
- **Resumable fallback in `_prompt_preview.py`.**
  - Extract the tail of `_preview_token_under_cursor` after the repo-mention step (word
    lookup → shorthand-argument owner → warning) into one helper.
  - The helper takes a snapshot captured at keypress: the text, the offset, the
    `WordSpan` from `extract_lookup_word`, the shorthand token, and the preview context.
  - Insert the bead step between repo mention and that tail. Non-beads then get today's
    behavior byte-for-byte, even when resumed from the worker after a cold directory
    load.
- **Word-lookup changes in `_prompt_word_lookup.py`.**
  - Let `_lookup_word_under_cursor` accept a precomputed span.
  - Thread an optional `bead_miss` hint through `_resolve_word_lookup_async` and
    `_handle_definition_result`. For a missing bead, only the `no_match` toast and the
    "nothing to look up" dead end change wording.
- **Warning text.** Update the generic warning to mention bead IDs.

### Card core (this phase)

New `src/sase/ace/tui/modals/bead_preview_modal.py`:

- `BeadPreviewModal(CopyModeForwardingMixin, SourceFileActionsMixin, ModalScreen[None])`,
  with constructor `(store, bead_id, *, preview: BeadPreview | None = None)`. A
  preloaded `preview` serves tests and snapshots.
- Without a preview it mounts at once in a loading state: the title shows `◌ <id>` plus
  the project, and the body shows a dim `Loading…`. A thread worker then fills it.
- Load failures render inside the card (a red line with the reason, and `r` to retry)
  instead of toasting and vanishing.

New `src/sase/ace/tui/modals/bead_preview_render.py` holds pure Rich helpers in the
style of `repo_preview_render.py`:

- `bead_card_palette(theme)`: contrast-adapted type, status, and size colors.
- `build_bead_title`: the identity row (type glyph,
  `EPIC`/`TALE`/`PLAN`/`PHASE`/`TASK · <task type>`/`FLAG`, and the ID) with the project
  label right-aligned; the bold title beneath; then the chip row.
- The chip row: status chip, size chip (phase/task), `task_type_chip`,
  `flag_key_chip`/`flag_due_chip`, `snooze_wake_chip`, the `plus_one_badge`, and the
  closed resolution.
- `build_bead_meta`: a dim-label grid.
  - Rows: Project; Plan (`~`-abbreviated path; labeled `Plan (epic)` when inherited;
    reference plus red `cannot resolve` when unresolved); Assignee, Owner, and Model
    when set.
  - Rows: Created (`bead_created_label`); Updated; Closed with close reason when closed;
    and Notes (`N · latest <relative>`, plus the latest note's first three lines, dim).
- The description renders in a `Markdown` widget; when empty, a dim `No description`.

Bindings, mirroring the siblings:

- `esc`/`q` close.
- `y` copies the bead ID; `Y`/`shift+y` copies the plan path.
- `o` edits the plan and `Z` views it (both through `SourceFileActionsMixin`, absent
  when there is no plan).
- `j`/`k`, `ctrl+d`/`ctrl+u`, and `g`/`G` scroll.
- The footer is built dynamically, so it lists only the keys that apply.

Add CSS for `BeadPreviewModal` next to `RepoPreviewModal` in `styles.tcss`, using the
same ids pattern.

### Docs and help

- In `src/sase/ace/tui/modals/help_modal/binding_common.py`:
  - Change the `K` row to `Preview macro/skill/file/bead`.
  - Add a `K on bead ID` row, `Preview bead card`. Descriptions stay ≤ 32 chars.
- In `docs/reference/tui/prompt-completion-panels.md`, add a `## Bead IDs` section
  between "Repo names" and "Word definitions & spellcheck". It covers the recognized
  forms, the outcome table, and the card.
- In `docs/reference/tui/prompt-modes.md`, update the `K` row to mention bead IDs.

### Tests

- `tests/test_bead_mentions.py` covers every row of the recognition table:
  - Cursor at the start, middle, and end of an ID; on the `@` and on `bead:`.
  - Phase IDs, trailing `.`/`,`/`)`, backticks, multi-line offsets, and whitespace
    (`None`).
  - Every rejection case: paths, URLs, `+tag`, `#macro`, `%dir` before an ID, `.md`, and
    `--`.
- `tests/ace/tui/models/test_bead_preview.py`:
  - Build stores with `BeadProject.init(tmp_path, beads_dirname="beads")`, as
    `tests/test_bead_statuses_for_project.py` does.
  - Cover each route outcome: found, missing with an owned prefix, not a bead, ambiguous
    with and without `project_ref`, and unavailable.
  - Cover `load_bead_preview` for an epic with phases and dependencies, phase plan
    inheritance, and a missing bead (`KeyError`).
  - Assert that no artifact-read rows are written: monkeypatch `record_bead_reads` to
    raise.
- `tests/ace/tui/widgets/test_prompt_bead_preview.py` (`PromptPage`, modeled on
  `test_prompt_repo_mention_navigation.py`):
  - K on a bare ID and on `@bead:` pushes `BeadPreviewModal`.
  - With a warm directory, an unknown prefix takes the synchronous word lookup, and no
    worker is started.
  - A missing owned ID shows the bead-aware toast; ambiguous and unavailable show
    warnings.
  - With a cold directory, the worker loads the directory inline, publishes it, and
    still resumes the word lookup for non-beads.
  - A second K or a cursor move drops the stale result.
- `tests/ace/tui/modals/test_bead_preview_render.py` and `test_bead_preview_modal.py`:
  - Title and chips for epic, phase, task, flag, and closed beads.
  - Light-theme colors differ from dark (contrast adaptation is applied).
  - Loading → loaded, an in-card load error with `r` retry, and the `y`/`Y`/`o` actions
    (monkeypatched).

### Verify

- Run the repo checks per the `lint_and_test` memory note.
- Optionally run a live `sase screenshot` pass: type a prompt containing a real bead ID,
  press `Esc` then `K`, and inspect the PNG.

## Phase `relations` — Related beads, navigation, and full detail

### Relations block (render helper `build_bead_related`, shown above the meta grid)

- **Lineage.** For beads with ancestors, show a row such as
  `Epic  ◐ bob-cli-kx · Gmail archive on task close`. The label is `Epic`, `Plan`, or
  `Parent` depending on the ancestor. An unresolved ancestor shows `?`.
- **Children.**
  - The heading is `PHASES` for plans and `CHILDREN` otherwise.
  - Each row has a drill-down number (1–9 only; later rows are unnumbered), the status
    glyph in status color, and the ID suffix relative to the parent (`.1` when the ID
    starts with `<parent>.`). The title follows, with the size chip right-aligned.
  - Closed rows are dimmed.
- **Dependencies.** `Blocked by` and `Blocks` rows show the status glyph, ID, and title.
- **Progress meter.** For a plan with children, add a meter to the chip row: one status
  glyph per child in its status color, capped at 12 glyphs then `…`, plus
  `N/M phases closed`.

### In-card navigation

- `1`–`9` open the Nth numbered child in the same card. `p` opens the parent. Both are
  same-store loads through `load_bead_preview`.
- `backspace` returns to the previous bead. When there is history, the title's top line
  shows a dim `⌫ <previous id>` breadcrumb.
- `r` reloads the current bead.
- The footer lists only the applicable keys, for example `1-5 phase`, `p parent`, and
  `⌫ back`.

### Full detail

- Add a public
  `bead_detail_link_resolution(bead_id, *, project_ref=None) -> LinkResolution` to
  `src/sase/pager/beads.py`. It is a thin wrapper over `_bead_id_link_resolution`.
- In the card, `enter` runs that wrapper off-thread and pushes
  `PagerScreen(document, links_enabled=True)` on top of the card, the way
  `actions/agents/_hijack_recovery_panel_impl.py` does. Bead IDs in the detail stay
  followable, and closing the pager returns to the card.
- An unresolved result toasts its `unresolved_message`.

### Ctrl+]

- Add `_jump_to_bead_under_cursor()` to `PromptBeadPreviewMixin` and insert it into
  `_prompt_jump.py`'s chain after repo mention.
- Ctrl+] uses the same detection and routing as K, with the same miss, ambiguous, and
  unavailable messages. It has no word fallback: a non-bead continues to the
  shorthand-argument step.
- Update the jump warning text to mention bead IDs.

> [!decision] jump_target = detail Ctrl+] opens the full bead detail in the pager
> directly, exactly like `enter` in the card.

> [!decision] jump_target = plan When the bead (or its parent epic) has a resolved plan
> path, Ctrl+] opens it through the existing `JumpTarget` / `_present_jump_actions` flow
> (editor or tmux-pane chooser). Otherwise it falls back to the pager detail.

### Docs, help, and goldens

- Help modal: set the `Ctrl+]` row to `Jump to macro/skill/file/bead`, and make the bead
  row `K / Ctrl+] on bead ID` → `Preview bead / full detail` (or `/ open plan`, per
  `jump_target`).
- Docs: extend the `## Bead IDs` section with card navigation and full detail, and
  update the `Ctrl+]` row in `prompt-modes.md`.
- Add `tests/ace/tui/visual/test_ace_png_snapshots_bead_preview.py`, modeled on
  `test_ace_png_snapshots_glossary_preview.py`. It covers:
  - the epic card with phases, dark and light, at 120×40;
  - a phase card with lineage and dependencies, dark;
  - the loading state, dark.
- Generate the goldens with `just fix-tui-screenshots -- <selectors>` (through
  `/sase_monitor` when long), then inspect every PNG.

### Tests

- Modal tests:
  - Drill-down by number, `p`, the `backspace` history and breadcrumb, and `r` reload.
  - `enter` pushes `PagerScreen` with the wrapper monkeypatched, and an unresolved
    result toasts.
  - Numbers beyond the child count are no-ops.
- Render tests: the meter cap and the closed-row dimming.
- Widget tests: Ctrl+] on a bead opens the pager (or the plan, per decision), and
  non-beads keep today's Ctrl+] behavior.

## Phase `mentions` — Status-tinted bead mentions in the prompt

### Index and resolution (no per-keystroke work)

- **Memory-only scan.** In `src/sase/bead/mentions.py`, add
  `scan_bead_mentions(text, prefixes) -> tuple[BeadMention, ...]`. It covers the bare
  form restricted to owned prefixes, plus refs, and reuses phase `peek`'s shape and
  rejection rules. Also add `BeadStoreDirectory.owned_prefixes()`.
- **App-level index.** Extend the app mixin from `peek` with a bounded (about 2048
  entries, LRU) `BeadMentionIndex`. It maps a bead ID to its status, or to _missing_,
  together with the owning store key and that store's `store_fingerprint` token at
  resolution time.
- **Resolution requests.**
  - `request_bead_mention_resolution(ids)` coalesces into one in-flight worker: a
    pending set, last-request-wins, with flags released on failure.
  - The worker groups IDs by owning store, recomputes `store_fingerprint`, and
    re-queries `bead_read_facade.statuses_for_ids` only for unknown IDs or IDs whose
    store fingerprint changed. IDs absent from the result are _missing_.
  - It publishes the results. Only if something changed does it call
    `_refresh_visible_prompt_semantic_surfaces(global_change=True)`.
- **Triggers.** Resolution runs only on discrete events, never from
  `_build_highlight_map` and never per keystroke:
  - prompt mount, and when the directory warm publishes;
  - `_enter_normal_mode` in `_prompt_text_area_actions.py`, which covers "finished
    typing";
  - a K or Ctrl+] bead resolution, which seeds the index with the status it just loaded.
  - Each trigger does a memory-only scan of the current text and requests only
    unresolved or stale IDs.

### Overlay

- In `PromptBeadMentionMixin`, override `_build_highlight_map` the same way
  `_prompt_repo_mentions.py` does.
  - Apply the same size guards (`_MAX_OVERLAY_BYTES` / `_MAX_OVERLAY_LINES`) and a scan
    cache keyed by (text, directory identity).
  - Append a style span only for IDs whose index entry is a known status. Unknown and
    missing IDs stay plain, so `cash-flow` never sprouts an underline.
- Do not restyle `@bead:` refs; the artifact-ref overlay already owns them. Place the
  mixin before `PromptGlossaryMixin` / `PromptRepoMentionMixin` in `PromptTextArea`'s
  bases so they win on overlap.
- **Styles.** Extend `SemanticHighlightStyles` in
  `src/sase/ace/tui/util/semantic_styles.py` and include the new colors in
  `signature()`. Register the styles in the TextArea theme, including the Jinja theme,
  the same way as `_register_repo_mention_text_area_theme`.

> [!decision] mention_tint = status The overlay registers one `bead.mention.<status>`
> style per bead status. Each is the canonical
> `bead_status_presentation(status).rich_color` adapted with `derive_argument_color`
> against the theme, bold and underlined.

> [!decision] mention_tint = uniform The overlay registers a single `bead.mention` style
> derived from the theme's `secondary` color, bold and underlined, as a third sibling of
> the glossary and repo styles.

### Docs

Add a paragraph at the top of `## Bead IDs`, written before the K card paragraphs to
match the "Repo names" section's shape. It covers:

- which IDs are underlined and in what color;
- that unresolved or missing IDs stay plain;
- that resolution happens off the typing path on `Esc`, on mount, and on `K`, so a
  just-typed ID lights up when you leave INSERT mode.

Also update the highlight-color comparison sentence in the "Word definitions &
spellcheck" section, which lists the underline colors.

### Tests and verification

- Unit tests:
  - scan restricted to owned prefixes;
  - index coalescing (two requests make one worker);
  - fingerprint-driven revalidation (an unchanged token means no re-query);
  - the LRU bound.
- Widget tests:
  - typing in INSERT mode schedules no resolution (assert no worker or facade call per
    keystroke);
  - `Esc` schedules one;
  - a published index styles the ID with the expected style name, and a missing ID stays
    unstyled;
  - `_build_highlight_map` performs no I/O (monkeypatch the facade to raise).
- PNG golden: a prompt with an in-progress ID, a closed ID, and a missing ID, in dark
  and light.
- Perf check: run `pytest -s -m slow tests/ace/tui/bench_tui_jk.py` before and after the
  change, and confirm p95 holds.

## Out of scope

- Bare shorthand IDs (`1ss.1`) and agent names such as `bob-cli-lh.land` in free text.
- Writing to beads from the card (claim, close, note). The Beads pane owns mutations.
- Bead completion while typing (already covered by `%wait` completion and `@bead:` ref
  completion).
- Moving mention recognition into `sase-core`. The routing and reads it builds on are
  already Rust-backed, and the recognizer is a Textual-free module
  (`sase/bead/mentions.py`) that matches the precedent of the repo-mention and pager
  bead-ID recognizers. A future move would be mechanical.
