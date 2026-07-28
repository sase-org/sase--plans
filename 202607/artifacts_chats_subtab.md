---
tier: epic
title: Artifacts → Chats sub-tab with sync provenance and agent revival
goal: 'The ACE Artifacts tab gains a fifth "Chats" sub-tab that lists every SASE chat
  transcript known to this machine — including remote transcripts imported from the
  agents sidecar repo — makes each transcript''s sync provenance (local-only / shared
  / remote) unmistakable at a glance, and lets the user jump from a chat straight
  to its agent on the Agents tab, reviving that agent first when it is dismissed.

  '
phases:
- id: discovery
  title: Chat file discovery covers imported remote transcripts
  depends_on: []
  size: small
  description: '"Phase 1 — Chat file discovery" section: add a chats-specific file
    walker that yields YYYYMM shards, legacy top-level files, and the imported `v2-*`
    shard directories, then route the existing chat storage/catalog/resume helpers
    through it so imported remote transcripts stop being invisible.

    '
- id: scaffold
  title: Register the Chats sub-tab across TUI plumbing
  depends_on: []
  size: medium
  description: '"Phase 2 — Sub-tab scaffold" section: register `chats` in the shared
    Artifacts sub-tab constants, compose an empty `ArtifactsChatsPane`, and wire bindings,
    keymaps, default config, command palette metadata, action gating, help modal,
    onboarding, and quickstart copy for the new pane.

    '
- id: catalog
  title: Headless chat catalog with sync provenance
  depends_on:
  - discovery
  size: medium
  description: '"Phase 3 — Headless provenance catalog" section: build a TUI-independent
    chat catalog that resolves each transcript''s owning agent, classifies it as local
    / shared / remote / unknown against the agents sidecar checkout, and caches the
    expensive scans keyed by mtime and sidecar HEAD.

    '
- id: cli
  title: sase chat list exposes provenance
  depends_on:
  - catalog
  size: small
  description: '"Phase 4 — CLI surface" section: add a provenance column and machine
    column to the `sase chat list` table, add `-P/--provenance` and `-m/--machine`
    filters, and extend the stable JSON schema with the new provenance fields.

    '
- id: pane
  title: Chats pane list, provenance badges, and loading
  depends_on:
  - scaffold
  - catalog
  size: medium
  description: '"Phase 5 — Pane list and provenance rendering" section: load the catalog
    off-thread with a bounded first page, render date-grouped rows carrying a three-channel
    provenance badge and gutter stripe, and implement navigation, refresh, and the
    shared Artifacts entry-navigator contract.

    '
- id: detail
  title: Chats detail panel, summary chips, and filter bar
  depends_on:
  - pane
  size: medium
  description: '"Phase 6 — Detail panel and filters" section: build the right-hand
    detail panel with an explicit PROVENANCE section, the pane-header provenance summary
    chips, the inline filter bar with `provenance:`/`machine:` tokens, the provenance
    cycle key, and the transcript preview action.

    '
- id: agentlink
  title: Open or revive the agent behind a chat
  depends_on:
  - pane
  size: medium
  description: '"Phase 7 — Agent link and revival" section: add the action that jumps
    from the selected chat to its agent on the Agents tab, reviving the agent through
    the existing revive machinery first when it is dismissed, and degrade clearly
    when no local agent backs the transcript.

    '
- id: visual
  title: Visual snapshots and documentation polish
  depends_on:
  - detail
  - agentlink
  size: small
  description: '"Phase 8 — Visual coverage and docs" section: add ACE PNG snapshot
    coverage for the populated and empty Chats pane, verify every provenance state
    renders distinctly, and finish the help/onboarding and CHANGELOG-facing documentation
    sweep.

    '
create_time: 2026-07-24 19:29:31
status: done
bead_id: sase-90
---

- **PROMPT:** [202607/prompts/artifacts_chats_subtab.md](prompts/artifacts_chats_subtab.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-90.land](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-90.land/README.md)
  - [bbugyi200.athena.sase-90.land--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-90.land.md#member-code)
- **COMMITS:**
  - [e5d953e](https://github.com/sase-org/sase/commit/e5d953eadd0b66ce4c9d8806d045048943107825) — feat(chats): expose publication quarantine provenance (sase-90)

# Plan: Artifacts → Chats sub-tab

## Why

`~/.sase/chats/` is the single richest record of what SASE agents have actually done on this machine — 8,600+
transcripts today — and there is no way to browse it from the TUI. `sase chat list` exists but is a flat, 20-row CLI
table with no notion of where a transcript came from.

At the same time, the agents sidecar repo (`~/.sase/projects/<key>/repos/agents`) now publishes agent runs (including
`agents/<global-name>/chat.md`) and imports foreign runs from other machines. That means a chat file on this disk can be
in one of three materially different situations, and the user currently has no way to tell them apart. Making that
distinction obvious is the core of this feature.

## Design overview

### The provenance taxonomy

Four states. The first three are the ones the user asked for; the fourth exists so the UI never _lies_ when it simply
could not check.

| State     | Meaning                                                                                                            | Glyph | Label              | Color     |
| --------- | ------------------------------------------------------------------------------------------------------------------ | ----- | ------------------ | --------- |
| `local`   | Written on this machine; **not** present in any agents sidecar checkout                                            | `◇`   | `local`            | `#767676` |
| `shared`  | Written on this machine **and** published to an agents sidecar (`agents/<global-name>/chat.md` exists)             | `◆`   | `shared`           | `#5FD75F` |
| `remote`  | Originated on another machine and was pulled into this machine's agent data by an agents-sidecar sync/import       | `⇣`   | `<source-machine>` | `#FFAF5F` |
| `unknown` | Provenance could not be determined (no sidecar configured, sidecar not checked out, or the index is still warming) | `◌`   | `?`                | `#585858` |

The visual encoding is deliberately **three-channel** so it survives colorblindness, dim terminals, and screenshots:
glyph _shape_ (hollow diamond → filled diamond → down-arrow), a literal _word_, and _color_. On top of that, each list
row carries a one-cell **gutter stripe** in the provenance color, so scanning the list vertically reads provenance as a
colored ribbon before you read any text. `remote` rows show the _source machine name_ in place of a generic word,
because that is the single most useful fact about a remote chat.

`unknown` must never be silently collapsed into `local`. "I checked and it is not in the sidecar" and "I could not
check" are different claims, and conflating them is exactly how a browsing UI becomes untrustworthy.

### Classification rules

For each chat file, in order:

1. **Remote** if the transcript's owning agent artifact records `imported_source_owner` in `agent_meta.json`, or
   `done.json` records `imported_transaction_key`, or (fallback, when no artifact resolves) the file lives in an
   imported shard directory and its basename starts with `imported-v2-`. Source machine/username come from
   `imported_source_owner` (`{"username": ..., "machine_name": ...}`), falling back to the importer's naming.
2. **Shared** if the owning agent's global name has a directory in some agents sidecar checkout that contains `chat.md`.
3. **Local** if a sidecar index _was successfully read_ for the owning project (or the transcript resolves to no agent
   artifact at all — mentor/CRS/hook/workflow transcripts are never published, so they are local by construction) and
   rule 2 did not match.
4. **Unknown** otherwise — specifically when the owning project's agents sidecar is configured but its checkout is
   missing or unreadable.

### Data sources (all already on disk, no new formats)

- **Chat files** — `~/.sase/chats/`. Three physical layouts coexist: `YYYYMM/` shards, legacy top-level `*.md`, and
  imported shards written by `sase/agents_sync/v2_importer.py::_chat_path` as
  `~/.sase/chats/<destination_id[:6]>/imported-v2-<safe-name>-<destination_id>.md` where `destination_id` starts with
  `v2-`.
- **Chat → agent link** — `agent_meta.json:chat_path` and `done.json:response_path` inside agent artifact directories.
  Both fields are already mirrored into `~/.sase/agent_artifact_index.sqlite` under `agent_artifacts.record_json`
  (`record["done"]["response_path"]`, `record["agent_meta"]["chat_path"]`), which is the cheap path. Note that several
  agents in a family/retry chain can share one `response_path`; the mapping is many-to-one and the catalog must keep the
  _newest_ artifact as the primary link while retaining the others.
- **Sidecar published set** — `<agents-sidecar-root>/agents/<global-name>/{meta.json,chat.md}`. `meta.json` carries
  `name`, `machine`, and timestamps. Sidecar roots come from `sase.agents_sync.targets.resolve_sync_targets()` (each
  `ProjectTarget` pairs a primary repo with its `agents` sidecar). Both the v1 layout (top-level `manifest.json` +
  `agents/`) and the v2 layout (`users/<username>/machines/<machine>/…` + `agents/`) expose the same
  `agents/<global-name>/chat.md`, so keying off that directory works for both.
- **Publication backlog (detail-panel nicety)** — `~/.sase/projects/<key>/agents-publication-outbox.json` lists agent
  hoods queued for publication with `attempts` and `last_error`. When a `local` chat's agent appears there, the detail
  panel can say "queued to publish (28 attempts, last error: …)" instead of a bare "local only", which turns a confusing
  state into a diagnosable one.

### The discovery gap this plan closes

`sase.core.paths.iter_sharded_files()` only descends into directories matching `^\d{6}$` (`_SHARD_DIR_RE`,
`src/sase/core/paths.py:174`) plus legacy top-level files. Imported chats live under `v2-xxx/` directories, so **they
are currently invisible** to `list_chat_transcripts()`, `list_chat_histories()`, `find_chat_by_timestamp()`, and
therefore to `sase chat list` and the `/sase_chats` skill. Phase 1 fixes this for chats specifically; without it the
"remote" category of this feature would always be empty.

### Backend boundary note (read this before implementing)

`CLAUDE.md` says shared backend/domain behavior belongs in `../sase-core/crates/sase_core`. This plan deliberately keeps
the new catalog in Python, in `src/sase/history/`, for these reasons:

- Neither the chat domain (`sase/history/chat_*.py`) nor `sase/agents_sync/` has any Rust counterpart today — a grep of
  `crates/sase_core/src` for "chat" finds only ChangeSpec `CHAT:` section parsing.
- Every input this feature needs (sidecar layout, importer metadata, publication outbox) is defined by Python code in
  this repo. Porting the catalog alone would mean reimplementing the sidecar and importer readers in Rust.
- The intent of the boundary rule is honored by keeping the logic **headless**: the catalog module must not import
  Textual, must be usable from `sase chat list` (phase 4), and must be tested without a TUI. A future Rust port then has
  one well-defined Python surface to replace rather than logic smeared through widget code.

Do **not** put provenance logic in `src/sase/ace/tui/widgets/`. Presentation (badges, colors, layout) lives there;
classification does not.

### Performance constraints (non-negotiable)

Per `sase/memory/tui_perf.md`:

- Cold-scanning 8,600 transcripts and reading a 64 KB head from each is far too slow to do on the event loop, and too
  slow to do synchronously on every activation. All catalog work runs off-thread (`run_worker(..., thread=True)`,
  matching `ArtifactsPlansPane._request_load`), with `loading`/`reload_pending` coalescing flags and last-request-wins
  semantics.
- The catalog needs a persistent cache keyed by `(absolute path, mtime_ns, size)` so warm loads are stat-only. The
  sidecar published-name index is cached per project keyed by the sidecar checkout `HEAD` sha; the agent-link index is
  cached against the artifact-index generation. Render paths must never stat or glob.
- First paint must not wait on O(all chats): the pane requests a **bounded newest-first first page** (500 rows), paints
  it, then extends in the background and coalesces one follow-up refresh.
- Detail-panel updates go through `DetailPanelDebouncer` (150 ms); highlight moves paint immediately.
- Programmatic `OptionList.highlighted` assignment must be wrapped in a guard flag cleared in `finally:` — see
  `_syncing_options` in the plans pane for the established pattern.

### Sub-tab ordering decision

`ARTIFACTS_SUBTAB_ORDER` is `("commits", "plans", "bugs", "prs")` and the numeric bindings `1..N` are generated from it
(`src/sase/ace/tui/bindings.py:90-98`). **Append `"chats"` last** so existing muscle memory for `1`–`4` is preserved and
Chats becomes `5`. Do not insert it in the middle.

---

## Phase 1 — Chat file discovery

**Files:** `src/sase/core/paths.py`, `src/sase/history/chat_storage.py`, `src/sase/history/chat_catalog.py`.

Add an imported-shard-aware walker. Preferred shape: a new `iter_chat_files() -> Iterator[Path]` in
`sase/history/chat_storage.py` that yields, deduplicated and stable-sorted:

1. everything `iter_sharded_files("chats", pattern="*.md")` already yields (YYYYMM shards + legacy top-level), and
2. `*.md` inside `~/.sase/chats/<dir>/` for any directory that is not a YYYYMM shard — this covers the importer's `v2-*`
   directories without hard-coding the `v2-` prefix, so a future importer revision that changes the prefix does not
   silently regress.

Guard against unbounded recursion (one directory level only, matching the importer's actual layout) and against symlink
loops. Keep the function tolerant of `OSError` on individual entries, consistent with the existing helpers.

Route these through it:

- `chat_storage.list_chat_histories()`
- `chat_storage.find_chat_by_timestamp()`
- `chat_catalog.list_chat_transcripts()`
- `chat_storage.resolve_chat_file_path()` — `find_sharded_file` has the same blind spot; add a fallback scan over the
  non-shard directories when the sharded lookup misses, so `sase chat show -b <basename>` can resolve an imported
  transcript.

**Tests:** unit tests alongside `tests/history/test_chat_catalog.py` and `tests/history/test_chat_paths.py` covering: a
YYYYMM-sharded chat, a legacy top-level chat, an `imported-v2-…` chat inside a `v2-abc/` directory, dedup when the same
basename exists in two places, and `resolve_chat_file_path` finding an imported transcript by basename.

---

## Phase 2 — Sub-tab scaffold

Register `chats` everywhere the existing four sub-tabs are registered, and mount a real-but-empty `ArtifactsChatsPane`
so the tab is navigable before the data layer lands. Use the Plans sub-tab as the reference implementation throughout —
it is the most recently added and the closest structural match.

**Shared constants** — `src/sase/ace/tui/artifact_tabs.py`:

- `ArtifactsSubTab` Literal gains `"chats"`.
- `ARTIFACTS_SUBTAB_ORDER = ("commits", "plans", "bugs", "prs", "chats")`.
- `ARTIFACTS_PANE_IDS["chats"] = "artifacts-chats-pane"`.
- `ARTIFACTS_ACCENTS["chats"] = "#5FAFFF"` (sky blue: distinct from prs `#00D7AF`, commits `#FFD700`, bugs `#FF5F5F`,
  plans `#AF87FF`, and semantically apt for agent conversation).

**View** — `src/sase/ace/tui/widgets/artifacts/view.py`: `_ARTIFACT_LABELS["chats"] = "Chats"`,
`_DETAIL_SCROLL_IDS["chats"] = "chats-detail-scroll"`, yield `ArtifactsChatsPane(id=ARTIFACTS_PANE_IDS["chats"])` inside
the `ContentSwitcher`, and extend `set_keymap_registry()` / `set_project_scope()` to forward to it.

**Pane shell** — new `src/sase/ace/tui/widgets/artifacts/chats_pane.py`: a `Vertical` subclass mixing in
`ArtifactsPaneLifecycle`, `can_focus = False`, composing a filter-bar slot, an info line
(`classes="artifacts-pane-info"`), a `Horizontal` with `#chats-list-panel` (border title "Chats") and
`#chats-detail-panel` (border title "Details") wrapping `#chats-detail-scroll`, and a `#chats-hints` line. Empty state:
"No chat transcripts found." Export from `widgets/artifacts/__init__.py` and `widgets/__init__.py`.

**Actions** — new `src/sase/ace/tui/actions/artifacts_chats.py` exposing `CHATS_ARTIFACT_ACTIONS` and
`ArtifactsChatsActionsMixin`, mirroring `actions/artifacts_plans.py`. Wire into `src/sase/ace/tui/actions/artifacts.py`:

- add `*CHATS_ARTIFACT_ACTIONS` to `NON_PRS_ARTIFACT_ACTIONS`,
- add `"chats"` to the sets in `_non_pr_artifacts_active()` and `_artifacts_entry_navigator()`,
- add `action_show_artifacts_chats()`,
- mix `ArtifactsChatsActionsMixin` into `ArtifactsMixin`,
- re-export `CHATS_ARTIFACT_ACTIONS` from `__all__`.

**App gating** — `src/sase/ace/tui/app.py::check_action`: add `"show_artifacts_chats"` to the sub-tab switch set
(~line 385) and add a `CHATS_ARTIFACT_ACTIONS` gate mirroring the `PLANS_ARTIFACT_ACTIONS` block.

**Keymaps** — the full set for this pane (defaults chosen to match sibling panes; all pane-scoped, so overlap with other
panes' keys is expected and correct):

| Action                   | Default | Label               |
| ------------------------ | ------- | ------------------- |
| `chats_next`             | `j`     | Next Chat           |
| `chats_prev`             | `k`     | Previous Chat       |
| `chats_view_selected`    | `enter` | View Chat           |
| `chats_filters`          | `f`     | Chat Filters        |
| `chats_cycle_provenance` | `s`     | Cycle Sync State    |
| `chats_open_agent`       | `a`     | Open Chat Agent     |
| `chats_open_external`    | `o`     | Open Chat in Editor |
| `chats_copy_path`        | `y`     | Copy Chat Path      |
| `chats_refresh`          | `R`     | Refresh Chats       |

Register them in `src/sase/ace/tui/keymaps/types.py` (both the action tuple list and the dataclass fields),
`src/sase/default_config.yml` (a `# Chats sub-tab` block placed after the Plans block — per repo convention this file
must be updated whenever keymaps change), `src/sase/ace/tui/bindings.py` (a `# Chats sub-tab actions.` block),
`src/sase/ace/tui/commands/_app_metadata.py`, and `src/sase/ace/tui/commands/availability.py`
(`_CHATS_ARTIFACT_COMMANDS`).

**Docs surfaces** (mandatory per `src/sase/ace/CLAUDE.md` — the `?` help popup must stay in sync):

- `src/sase/ace/tui/modals/help_modal/changespecs_bindings.py` — a "Chats Pane" section listing every key above plus the
  shared `artifact_list_navigation` rows. Respect the 57-char box width and 32-char description limits.
- `src/sase/ace/tui/widgets/changespec_onboarding.py` —
  `_ARTIFACT_DESCRIPTIONS["chats"] = "Browse agent chat transcripts and their sync state."`, and update the card
  subtitle at ~line 151 which currently reads "Browse commits, plans, bugs & PRs without leaving Artifacts".
- `src/sase/ace/tui/widgets/tab_quickstart.py` — the two hard-coded strings at ~lines 213-224: `("1", "2", "3", "4")`
  becomes `("1", "2", "3", "4", "5")` and both "Commits · Plans · Bugs · PRs" strings gain "· Chats".

**Styles** — `src/sase/ace/tui/styles.tcss`: add `#artifacts-chats-pane` and the `#chats-*` rules, modeled on the
`#plans-*` block at lines 335-405 so panel proportions match the sibling panes.

**Tests:** extend the existing sub-tab tests so `5` selects Chats and `[`/`]` cycles through five panes; assert
`ARTIFACTS_SUBTAB_ORDER[:4]` is unchanged so the renumbering guarantee is enforced by a test.

---

## Phase 3 — Headless provenance catalog

**New module package:** `src/sase/history/chat_catalog_provenance/` (or a small set of `chat_catalog_*.py` siblings —
keep individual modules well under the repo's file-size norms; the existing artifacts widgets split at ~300-400 lines is
a good target).

**Public surface:**

```python
ChatProvenance = Literal["local", "shared", "remote", "unknown"]

@dataclass(frozen=True)
class ChatCatalogEntry:
    # existing ChatTranscriptInfo fields, preserved verbatim
    path: str; absolute_path: str; basename: str; mtime: str; size_bytes: int
    workflow: str | None; agent: str | None; timestamp: str | None
    prompt_snippet: str | None; response_snippet: str | None
    # new provenance fields
    provenance: ChatProvenance
    source_machine: str | None        # remote: origin machine; shared/local: this machine
    source_username: str | None
    project_key: str | None
    agent_artifact_dir: str | None    # newest artifact that references this transcript
    agent_local_name: str | None
    agent_global_name: str | None
    sidecar_repo: str | None          # absolute path to the agents sidecar checkout
    sidecar_relpath: str | None       # e.g. "agents/<global-name>/chat.md"
    publication_pending: bool         # agent is queued in agents-publication-outbox.json
    publication_last_error: str | None

def load_chat_catalog(
    *, limit: int | None = None, query: str | None = None,
    provenance: ChatProvenance | None = None, machine: str | None = None,
    project: str | None = None, force: bool = False,
) -> ChatCatalogSnapshot: ...
```

`ChatCatalogSnapshot` carries the entries plus per-provenance counts, the set of distinct remote machines, a
`truncated: bool`, and a `diagnostics: tuple[str, ...]` list naming any project whose sidecar could not be read (these
are what make `unknown` explicable in the UI).

**Three indexes, each cached:**

1. **Transcript index** — walk `iter_chat_files()` (phase 1), `stat()` each, and read the bounded head only for files
   whose `(path, mtime_ns, size)` is not already cached. Reuse the existing parsing helpers in `chat_catalog.py`
   (`_parse_header`, `_extract_section_snippet`, `_READ_LIMIT_BYTES`) rather than duplicating them. Persist to
   `~/.sase/chats_catalog.sqlite` with atomic writes and a schema-version row; treat any corruption or schema mismatch
   as a cold rebuild, never as an error.
2. **Agent-link index** — read `~/.sase/agent_artifact_index.sqlite` (`agent_artifacts.record_json`) and map
   `expanduser(done.response_path)` and `expanduser(agent_meta.chat_path)` → artifact metadata. Fall back to scanning
   artifact directories via `sase.core.agent_artifact_paths.iter_agent_artifact_dirs` when the index is unavailable or
   stale. Keep the newest artifact per transcript as the primary link. Capture `imported_source_owner` /
   `imported_transaction_key` here — that is the authoritative remote signal.
3. **Sidecar index** — for each `ProjectTarget` from `resolve_sync_targets()`, list `<sidecar>/agents/*/` and record
   `{global_name: (machine, username, has_chat_md)}`. Cache keyed by the sidecar checkout `HEAD` sha
   (`git rev-parse HEAD` via `sase.agents_sync.git.run_git`, off-thread, bounded, never interactive). A missing or
   unreadable checkout yields no index and a diagnostic — that project's chats classify as `unknown`, not `local`.

Global-name computation for local agents must reuse the existing helpers (`sase.core.agent_identity_facade` /
`sase.core.machine_hood_facade.machine_qualify_v1_transport_agent_name` / `globalize_agent_name`) — do not re-derive the
naming scheme.

**Tests:** headless `pytest` under `tests/history/` (new `test_chat_catalog_provenance*.py` modules) with a `tmp_path`
fake `~/.sase` covering each classification rule, the many-artifacts-to-one-transcript case, a missing sidecar producing
`unknown` + a diagnostic, cache warm vs cold parity (same output, second run does no head reads — assert via a read
counter), and cache invalidation when a transcript's mtime changes.

---

## Phase 4 — CLI surface

**Files:** `src/sase/main/parser_chat.py`, `src/sase/chat/cli_list.py`.

- New `sase chat list` options, kept alphabetically sorted among the existing ones and each with a short alias (repo CLI
  convention): `-m/--machine <name>` and `-P/--provenance {local,shared,remote,unknown}`.
- Pretty table gains a `SYNC` column rendering the badge glyph + label in the provenance color, and remote rows show the
  source machine. Keep the existing columns and their order otherwise.
- `chat_info_to_json()` gains the new fields, appended after the existing keys so the documented key order stays stable
  for existing consumers.
- Update the `sase chat list` help text to describe the new filters.

**Tests:** extend `tests/main/test_chat_handler.py` — JSON schema shape, each filter narrowing correctly, and a
provenance-colored table rendering without raising for every state.

---

## Phase 5 — Pane list and provenance rendering

**Files:** `src/sase/ace/tui/widgets/artifacts/chats_pane.py` plus new `chats_data.py`, `chats_list.py`,
`chats_rendering.py`, `chats_navigation.py` siblings.

**Loading** — copy the proven shape from `ArtifactsPlansPane`: `on_first_activate` requests a load; `on_activate`
reloads only when the snapshot is missing or its project scope changed; `on_refresh` forces; `on_worker_state_changed`
applies the result, calls `_cancel_artifacts_jump_mode_for_model_change("chats")`, and re-selects the previously
selected row by stable id. Request the bounded first page (500 newest) first, paint, then schedule the full extension
through `spawn_pump_free_task` and coalesce one follow-up refresh.

**Rows** — an `OptionList` grouped by date with dim `── Today ──` / `── Yesterday ──` / `── YYYY-MM-DD ──` headers
(reuse the grouping shape from `modals/agent_run_log_modal.py::_group_agents_by_date`). Each row:

```
▌◆ shared    14:32  [sase]  sase-8v.3--code        Add lane cleanup snapshot refresh…
▌◇ local     14:07  [sase]  crs                    Review the pending diff for…
▌⇣ zeus      09:51  [bob]   bob.4j--code           Wire the ingest retry backoff…
```

- `▌` is the gutter stripe in the provenance color.
- Badge glyph + label, padded to a fixed width so the columns line up.
- `HH:MM`, project badge, presented agent name (via `present_agent_name`) or workflow when there is no agent, and the
  prompt snippet, dim, filling the remainder.
- The pane accent `#5FAFFF` is used for the selected row and panel chrome; provenance colors are reserved for the
  badge/stripe so the two encodings never fight.

**Navigation** — implement the `ArtifactEntryNavigator` protocol (`entry_targets`, `selected_entry_target`,
`select_entry_target`, `apply_entry_jump_hints`, `clear_entry_jump_hints`) so `'`-jump and the shared relative
navigation work exactly as they do for plans/commits/bugs. Use the transcript's absolute path as the stable target
component. Guard programmatic highlight assignment with a `_syncing_options` flag cleared in `finally:`.

**Actions implemented in this phase:** `chats_next`, `chats_prev`, `chats_refresh`, `chats_copy_path`,
`chats_open_external` (open in `$EDITOR` via `sase.ace.hints.build_editor_args` inside `self.app.suspend()`, the same
pattern as `agent_run_log_modal.action_open_chat`).

**Tests:** `tests/ace/tui/test_artifacts_chats_loading.py` and `…_rendering.py` following the `test_artifacts_plans_*`
structure with a monkeypatched `load_chat_catalog`, asserting: rows appear in newest-first order, each provenance state
renders its distinct glyph/label/color, the first page paints before extension completes, and j/k navigation moves the
selection without re-entrant highlight echoes.

---

## Phase 6 — Detail panel and filters

**Detail panel** (`#chats-detail-scroll`), rendered through `DetailPanelDebouncer`:

1. **PROVENANCE** — the headline section, accent-underlined. One badge line, then one plain-English sentence, then the
   supporting facts:
   - `local` → "Only on this machine. Not published to the agents sidecar." Plus, when the agent is in the publication
     outbox: "Queued to publish — 28 attempts, last error: …".
   - `shared` → "From this machine; also published to the agents sidecar." Plus sidecar repo path and
     `agents/<global-name>/chat.md`.
   - `remote` → "Pulled in from <username>@<machine> when the agents sidecar was synced." Plus sidecar repo path, global
     name, and the local imported path.
   - `unknown` → "Sync state unknown — <diagnostic>." Never phrase this as "local".
2. **CHAT** — workflow, agent (presented name), model/provider when available, timestamp, size, project.
3. **AGENT** — the resolved local agent: name, status, whether it is dismissed, artifact directory; or "No local agent
   artifact — this transcript cannot be revived."
4. **TRANSCRIPT** — first ~200 lines of the transcript, bounded, with "… (truncated, press `enter` for the full chat)".

**Summary chips** — the pane info line above the list becomes a provenance summary:
`◇ 312 local · ◆ 1,204 shared · ⇣ 58 remote (2 machines) · ◌ 4 unknown`, each chip in its provenance color and
recomputed from the snapshot (never from a render-path scan). When a provenance filter is active, the active chip is
highlighted and the others dim.

**Filter bar** — `chats_filters` (`f`, and `edit_query` for symmetry with plans) opens an inline filter bar modeled on
`plan_filter_bar.py`, supporting tokens `provenance:`, `machine:`, `project:`, `agent:`, `workflow:`, `since:`,
`until:`, and bare text ANDed across basename/agent/workflow/snippets. Wire completion sources for `provenance:` values
and the observed `machine:` values.

**Provenance cycle** — `chats_cycle_provenance` (`s`) cycles All → local → shared → remote → unknown → All, updating
both the filter state and the highlighted summary chip. This is the fast path; the filter bar is the precise one.

**View** — `chats_view_selected` (`enter`) opens the full transcript in `PreviewPanelModal`
(`src/sase/ace/tui/modals/preview_panel_modal.py`), matching `action_plans_view_selected`.

**Hints line** — `#chats-hints` renders the pane's keys via `key_display_name(...)` exactly as `plans_rendering.py`
does. Note on the footer convention in `src/sase/ace/CLAUDE.md`: non-PR Artifacts panes deliberately clear the
conditional footer (`_keybinding_modes.show_artifacts_pane`) and surface their conditional keys in this in-pane hints
line instead. Follow the pane convention — do **not** re-enable the footer for Chats — and render `a` dimmed when the
selected chat has no agent.

**Tests:** `tests/ace/tui/test_artifacts_chats_detail.py` and `…_filtering.py` — each provenance state produces its own
distinct detail copy (assert the `unknown` copy never contains the word "local"), the summary chip counts match the
snapshot, each filter token narrows correctly, `s` cycles in order, and `enter` pushes `PreviewPanelModal`.

---

## Phase 7 — Agent link and revival

**Action:** `chats_open_agent` (`a`) in `src/sase/ace/tui/actions/artifacts_chats.py`.

Behavior, which is exactly the semantics already proven by
`src/sase/ace/tui/modals/agent_run_log_modal.py::action_jump_to_agent_tab` (lines 472-504) — reuse that flow rather than
inventing a second one:

1. Resolve the selected row's agent. The catalog gives the artifact directory and agent name; resolve it to a live
   `Agent` object by matching against `self._agents` and, when the agent is dismissed, against
   `self._dismissed_agent_objects` (match on `identity` first, then `raw_suffix`).
2. If nothing resolves → `notify("No agent is associated with this chat", severity="warning")` and stop. This is the
   common case for mentor/CRS/hook transcripts and must feel intentional, not broken.
3. If the agent is dismissed → call `self._do_revive_agent(agent)` (`AgentReviveExecutionMixin`,
   `src/sase/ace/tui/actions/agents/_revive_execution.py:33`). This is the same disk mutation, dismissed-set update,
   bundle marking, artifact-index sync, and deferred refresh that the `R` revive flow ultimately performs on a selected
   agent, so a chat-initiated revive and an Agents-tab revive converge on identical state.
4. Save the current tab position (`_save_current_tab_position()`), switch `current_tab = "agents"`, and select the agent
   by scanning `self._agents` for `identity`, falling back to `raw_suffix` (identity can change across a revive). If it
   still is not found, notify rather than leaving the cursor somewhere arbitrary.
5. Revival is a slow, disk-mutating, user-initiated operation: it must not block the event loop. `_do_revive_agent`
   already schedules its own deferred artifact refresh; re-capture the selected tab/row _after_ any await before
   applying selection, per the TUI perf rules.

**Remote chats:** a `remote` transcript can resolve to an imported local artifact (the importer materializes one), in
which case the jump works normally. When it does not, notify with the specific reason — "this chat was imported from
<machine> and has no local agent artifact" — rather than the generic message.

**Tests:** `tests/ace/tui/test_artifacts_chats_agent_link.py` — a chat whose agent is active jumps and selects without
calling revive; a chat whose agent is dismissed calls `_do_revive_agent` exactly once and then lands on the Agents tab
with that agent selected; a chat with no agent notifies and changes no tab state; the remote-without- artifact case
produces the specific message.

---

## Phase 8 — Visual coverage and docs

- Add `tests/ace/tui/visual/test_ace_png_snapshots_artifacts_chats.py` and `…_artifacts_chats_empty.py` following
  `test_ace_png_snapshots_artifacts_plans*.py`: press `5`, `expect_state("artifacts_subtab", "chats")`, monkeypatch
  `load_chat_catalog` with a fixture snapshot that contains **all four provenance states** plus at least two distinct
  remote machines, and capture the golden PNGs into `tests/ace/tui/visual/snapshots/png/`. Run `just test-visual` and
  accept the new goldens with `--sase-update-visual-snapshots`.
- Visually verify from the rendered PNG that the four badges are distinguishable in both glyph and color, and that the
  gutter stripe reads as a clean ribbon down the list.
- Final sweep of the `?` help modal, the Artifacts onboarding card, and the tab quickstart to confirm every Chats key
  and the five-way sub-tab numbering are documented and accurate.
- Confirm `sase chat list --help` reads well and its options are alphabetically sorted.

---

## Cross-cutting requirements

- **Every phase that changes files in this repo must run `just install` then `just check` before reporting done.**
  Workspaces are ephemeral, so `just install` is required first even when nothing looks stale.
- Do not modify `sase/memory/*.md`, `AGENTS.md`, or the generated provider shims — nothing in this plan requires it, and
  doing so needs explicit user permission that this plan does not grant.
- Keep new modules small and focused; this repo consistently splits panes into `*_pane` / `*_data` / `*_list` /
  `*_rendering` / `*_navigation` modules, and Symvision lint enforces private/unused-symbol hygiene (see
  `sase/memory/symvision.md` if lint complains).
- The pane must degrade gracefully on a machine with no agents sidecar configured at all: every chat classifies as
  `local` (there is no sidecar to be absent from — this is _not_ `unknown`), the summary shows a single chip, and no
  error toast fires.
- Nothing in this feature may write to `~/.sase/chats/` or to any sidecar checkout. It is a read-only browser; the only
  mutation it performs is the agent revive in phase 7.
