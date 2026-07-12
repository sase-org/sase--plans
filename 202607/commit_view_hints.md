---
create_time: 2026-07-09 12:56:59
status: done
prompt: 202607/prompts/commit_view_hints.md
tier: tale
---
# Plan: View a Commit's Message + Pretty Diff via the `v` Keymap (Agents tab)

## 1. Problem & Product Goal

On the **Agents** tab of the `sase ace` TUI, the agent metadata panel (right side) shows a **COMMITS:** section listing,
per repo, each commit an agent made:

```
COMMITS:
  ▣ sase
    52c99ca5d fix(tui): defer update restart for background tasks
  ▣ sase-org/sdd
    0fed546e32f4 chore(sdd): sync uncommitted SDD store changes
```

Today the `v` (view) keymap renders numeric `[N]` hints next to **artifacts, deltas, slow-tool reports, and
file-path-shaped tokens** — but **not** next to commit entries. A user cannot quickly inspect what a commit actually did
without leaving the panel.

**Goal:** Make each COMMITS entry a first-class `v` view target. When the user presses `v`, a `[N]` hint appears next to
every commit (e.g. next to `52c99ca5d` and `0fed546e32f4`). Selecting a commit's hint opens a **beautiful, in-TUI
viewer** showing:

1. The **full commit message** (subject + body), and
2. A **pretty, syntax-highlighted diff** of exactly that commit's changes.

The feature must be **intuitive** (reuses the existing `v`/hint gesture users already know), **reliable** (never blocks
the UI event loop; degrades gracefully when a diff is unavailable), and **beautiful** (matches the TUI's existing diff
aesthetic and color palette).

## 2. Design Principles & Key Findings

Research established the following, which shape the design:

- **All the data already exists in memory.** Each commit is rendered from a record in
  `agent.step_output["meta_commits"]`, which carries: the **full multi-line `message`**, the commit **`sha`**, the repo
  **`cwd`** (working dir), the **`repo_name`**, and an absolute **`diff_path`** pointing to a persisted unified-diff
  file captured at commit time. So the message needs **no** git call, and the diff is usually a cheap file read.

- **No new Rust / `sase-core` API is warranted.** Git execution is host-side by explicit design in this codebase;
  `sase-core` contains only pure parsers and has no diff/`git show` capability. The happy path (in-memory message +
  persisted `diff_path`) is pure presentation and stays in Python. The rare fallback reuses the **existing**
  VCS-provider hook `provider.show_revision(sha, cwd)` (raw `git show --format= --patch`). We deliberately add **zero**
  `sase-core` changes; this is consistent with the Rust-core boundary because the backend (commit parsing, git exec)
  already lives in core/provider — we only add presentation and reuse existing seams.

- **Extend the existing `v` flow; do not add a new key.** `v` already re-renders the metadata panel with hints and
  mounts a hint-input bar. The COMMITS section is simply the one metadata section that is not currently handed the
  shared `hint_state` accumulator. We thread it in — no keymap changes, so no `default_config.yml` binding churn.

- **Reuse the existing pretty-diff + modal machinery.** The TUI already colorizes diffs with
  `lazy_renderable(text, "diff", line_numbers=True, theme="monokai")` (size-capped, cached) and already has a
  scrollable, vim-navigable `ModalScreen` viewer (`PreviewPanelModal`). We build a focused commit viewer on the same
  foundations rather than inventing new rendering or an external pager.

- **Never block the event loop.** Per TUI perf rules, the diff read / provider fallback runs **off-thread**; the modal
  opens instantly with the (in-memory) message and a "Loading diff…" state, then fills in the diff when ready.

## 3. User Experience

### Gesture

1. User is on the **Agents** tab with an agent selected; the metadata panel shows COMMITS.
2. User presses **`v`**. The panel re-renders with `[N]` hints on every viewable target — now including a `[N]` next to
   **each commit's short SHA**:
   ```
   COMMITS:
     ▣ sase
       [3] 52c99ca5d fix(tui): defer update restart for background tasks
     ▣ sase-org/sdd
       [4] 0fed546e32f4 chore(sdd): sync uncommitted SDD store changes
   ```
   (Exact numbers depend on how many other targets precede commits; commits render before deltas/artifacts, so those
   renumber accordingly — expected.)
3. User types the commit's number and presses Enter. A **Commit Viewer modal** opens.

### The Commit Viewer modal (the "beautiful" part)

A centered `ModalScreen` overlay (mirroring `PreviewPanelModal`'s framing) with:

- **Title bar:** `⎇  <repo> · <short_sha> — <subject>` with the diff source path shown dim. `(primary)` tag when it's
  the primary repo's commit.
- **Message header block** (styled, not raw): the full commit message rendered with the subject bold and the body in
  normal weight, using the commit palette (subject `#D7D7FF`, sha `dim #D7D7AF`, section accent `bold #87D7FF`). A
  compact **change summary** line (`+7 ~14 -16 · 3 files`) computed from the diff via the existing
  `parse_unified_diff_deltas` + deltas summary helper, using the add/mod/ delete color triad (`#5FD787` / `#FFD787` /
  `#FF5F5F`).
- **A separator rule.**
- **Pretty diff:** `lazy_renderable(diff_text, "diff", line_numbers=True, theme="monokai")` — identical styling to the
  file panel's per-commit diffs, with the built-in 64 KB / 1 500-line cap + truncation notice for huge diffs.
- **Footer:** `j/k scroll · ctrl+d/u page · g/G top/bottom · y copy sha · esc close`.

### States

- **Loading:** modal appears immediately with the message header; diff area shows a dim "Loading diff…" until the
  off-thread read completes (typically instant for the persisted file).
- **Diff unavailable:** if `diff_path` is missing/unreadable and the `provider.show_revision` fallback yields nothing,
  the modal still shows the full message plus a dim "Diff unavailable for this commit." Never an error/crash.
- **Empty COMMITS:** nothing changes; commits simply contribute no hints.

### Optional suffix parity (nice-to-have, mirrors existing hint grammar)

The `v` input grammar already supports `@` (open in `$EDITOR`) and `%` (copy) suffixes. For a commit hint we extend the
same grammar naturally:

- default → open the modal,
- `%` → copy the commit SHA to the clipboard,
- `@` → open the raw diff file in `$EDITOR`.

These are polish; the required behavior is **default → modal**.

## 4. Technical Design

### 4.1 Register commits as hint targets

- Thread the shared `hint_state` accumulator into the commits renderer, mirroring how artifacts/deltas already do it:
  - `append_agent_commits_section(...)` and its helper `_append_commit_group(...)`
    (`src/sase/ace/tui/widgets/prompt_panel/_agent_commits.py`) gain a `hint_state: HeaderHintState | None = None`
    parameter.
  - The caller in `build_header_text` (`src/sase/ace/tui/widgets/prompt_panel/_agent_display_header.py`) passes the
    `hint_state` through (it already passes it to artifacts/deltas).
- When `hint_state` is present, emit a `[N]` marker before each commit's short SHA (`    [N] <sha> <subject>`) and
  register a **`CommitViewSpec`** for that hint.
- Build the `CommitViewSpec` from the **same** `meta_commits` record that produces the rendered line (the
  commit-grouping path already reads these records), so hints and lines stay strictly 1:1 and every displayed commit is
  viewable — including commits that have no persisted `diff_path` (they fall back at view time). This avoids fragile
  joining against the separate `agent_commit_diffs` accessor.

### 4.2 New data types (presentation-only)

- **`CommitViewSpec`** (frozen dataclass): `short_sha`, `sha`, `repo_name`, `cwd`, `subject`, `message` (full),
  `diff_path: str | None`, `is_primary`. Lives beside the commit renderer or in the hint-state module.
- Extend the hint accumulator/result carriers (`src/sase/ace/tui/widgets/prompt_panel/_agent_display_state.py`):
  `HeaderHintState` and `AgentHintRender` each gain `commit_views: dict[int, CommitViewSpec]` (hint number → spec),
  following the existing `tool_call_reports` side-map precedent used for deferred slow-tool reports.

### 4.3 Thread the spec map through the hint pipeline

- `AgentHintsDisplayMixin.update_display_with_hints` (`src/sase/ace/tui/widgets/prompt_panel/_agent_display_hints.py`)
  initializes and returns `commit_views` alongside `file_hints`/`tool_call_reports`.
- `_view_agent_files` (`src/sase/ace/tui/actions/hints/_files.py`) captures `hint_render.commit_views` into a new
  app-state field `self._hint_commit_views: dict[int, CommitViewSpec]`.
- Declare/init that field in the hint state types + initializer (`src/sase/ace/tui/actions/hints/_types.py`,
  `src/sase/ace/tui/actions/_state_init.py`), mirroring `_hint_tool_call_reports`.

### 4.4 Dispatch: open the modal on selection

- In `_process_view_input` (`src/sase/ace/tui/actions/hints/_processing.py`), after parsing hint numbers and **before**
  the existing file routing (pager/editor/clipboard/artifact-viewer): detect selected hint numbers present in
  `self._hint_commit_views`.
  - Default → open a `CommitViewModal` for the commit.
  - `%` suffix → copy `spec.short_sha` to clipboard (reuse existing clipboard util).
  - `@` suffix → route `spec.diff_path` (if present) through the existing open-in-editor path.
  - Any co-selected **non-commit** hints continue through the normal file routing unchanged.
  - Multi-commit selection: open the first selected commit's modal and `notify` that extras were ignored (single-select
    is the intended gesture); keeps UX predictable without stacking modals.

### 4.5 The `CommitViewModal`

- New `src/sase/ace/tui/modals/commit_view_modal.py`: `CommitViewModal(ModalScreen[None])`, structurally modeled on
  `PreviewPanelModal` (title `Static`, `VerticalScroll` body, footer `Static`; reuse its vim scroll bindings and
  `CopyModeForwardingMixin`).
- **Body renderable:** a `rich.console.Group` of `[message_header_text, separator, diff_renderable]`, where:
  - `message_header_text` is built from the in-memory `spec` (repo, sha, styled subject+body, change summary).
  - `diff_renderable` starts as a dim "Loading diff…" `Text` and is replaced once the diff is loaded, via
    `lazy_renderable(diff, "diff", line_numbers=True, theme="monokai")`.
- **Off-thread diff load** (perf rule 1): on mount, push a thread worker (`run_worker(..., thread=True)` /
  `asyncio.to_thread`) that:
  1. reads `spec.diff_path` if set (reuse the existing `_read_commit_diff_text` helper), else
  2. calls `provider.show_revision(spec.sha, spec.cwd)` via the VCS provider facade, else
  3. returns `None` → "Diff unavailable" state. Marshal the result back on the UI thread (`call_after_refresh` / worker
     completion); guard against the modal having been dismissed before the diff lands.
- **Copy-sha** action (`y`) inside the modal for quick SHA grabbing.
- Add a scoped TCSS block to `src/sase/ace/tui/styles.tcss` (modeled on the `PreviewPanelModal` block: `~85%` sizing,
  `border: thick $primary`, `background: $surface`, muted footer) using theme tokens so it tracks the app theme.

### 4.6 Reliability & performance

- Message is always in-memory → the modal is useful even with zero git access.
- Diff read/fallback is always off the event loop; the modal provides its own visible loading feedback (no need for the
  heavier tracked-task machinery, matching how the file panel already loads persisted commit diffs).
- `lazy_renderable`'s existing caps handle pathologically large diffs.
- On worker completion, re-check the modal is still mounted before updating (selection/screen may have changed during
  the await).

### 4.7 Discoverability

- If any help text or the hint-input-bar placeholder enumerates viewable target kinds, add "commits" so users learn `v`
  now covers COMMITS. Update the `v`/View help entry only if it lists specific kinds (no keybinding change otherwise).

## 5. Files to Touch (summary)

Presentation / hint wiring (all under `src/sase/ace/tui/`):

- `widgets/prompt_panel/_agent_commits.py` — thread `hint_state`, emit `[N]`, build `CommitViewSpec`; add a small
  `load_commit_diff_text(spec)` helper (persisted-file read + `show_revision` fallback).
- `widgets/prompt_panel/_agent_display_header.py` — pass `hint_state` to commits.
- `widgets/prompt_panel/_agent_display_state.py` — `CommitViewSpec` + `commit_views` on `HeaderHintState` /
  `AgentHintRender`.
- `widgets/prompt_panel/_agent_display_hints.py` — init/return `commit_views`.
- `actions/hints/_files.py` — capture `commit_views` into app state.
- `actions/hints/_processing.py` — dispatch commit hints to the modal (+ suffixes).
- `actions/hints/_types.py`, `actions/_state_init.py` — new `_hint_commit_views` app-state field.
- `modals/commit_view_modal.py` — **new** `CommitViewModal`.
- `styles.tcss` — modal styling block.

Explicitly **out of scope:** any `sase-core` (Rust) change, any new VCS-provider hook, any keymap/`default_config.yml`
binding change.

## 6. Testing

Follow existing ACE TUI test patterns (unit + the house PNG visual-snapshot suite):

- **Unit — hint registration:** `append_agent_commits_section` with a `hint_state` emits `[N]` markers on each commit
  and populates `commit_views` with correct specs; hint-counter continuity with the following deltas/artifacts sections;
  commits with and without `diff_path`.
- **Unit — dispatch:** `_process_view_input` opens `CommitViewModal` for a commit hint; routes co-selected file hints
  normally; `%` copies SHA; multi-select opens first + notifies.
- **Unit — diff loading:** `load_commit_diff_text` reads a persisted `diff_path`; falls back to
  `provider.show_revision`; returns `None` (→ "unavailable") when both fail. Assert it runs off-thread (no event-loop
  blocking).
- **Modal rendering:** header Group composition (styled message + change summary) and diff via `lazy_renderable`.
- **PNG visual snapshot** (per repo convention, goldens in `tests/ace/tui/visual/snapshots/png/`): a golden of the
  `CommitViewModal` showing message + pretty diff, plus (optionally) the COMMITS section rendered with hints. This is
  the primary guard on the "beautiful" bar.

## 7. Risks & Mitigations

- **Hint renumbering** below COMMITS: expected side effect of inserting commit hints before deltas/artifacts; covered by
  a visual/unit check.
- **Short vs full SHA** (primary repo persists short SHA): `git show`/copy accept short SHAs, and `diff_path` is the
  primary source anyway — no issue.
- **Merge commits:** persisted `diff_path` is the primary source; the `show_revision` fallback returns patch-only and is
  handled by the "unavailable" path if empty.
- **Modal dismissed mid-load:** guarded by an is-mounted check before applying the diff result.

## 8. Success Criteria

- Pressing `v` on the Agents tab renders a `[N]` hint next to every COMMITS entry.
- Selecting a commit's hint opens a modal with the full commit message and a monokai-highlighted diff of that commit,
  matching the TUI's existing diff look.
- No event-loop blocking; graceful "Diff unavailable" fallback.
- `just check` passes (lint + type + tests) and the new PNG snapshot(s) are approved goldens.
