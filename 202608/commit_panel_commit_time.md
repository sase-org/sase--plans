---
tier: tale
title: Show when a commit was created in the ACE commit panel
goal:
  The shared commit panel (Agents tab hints and Artifacts → Commits) shows each commit's creation time as a
  right-aligned "Today 07:05:54 · 2h ago" chip, sourced from the commit's real author time, persisted for new commits
  and lazily recovered for historical ones.
proposed_by: bbugyi200.athena.tv
create_time: 2026-08-06 08:16:37
status: wip
---

- **PROMPT:**
  [prompts/202608/commit_panel_commit_time.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/commit_panel_commit_time.md)

# Plan: Show when a commit was created in the ACE commit panel

## Problem

`CommitViewModal` (`src/sase/ace/tui/modals/commit_view_modal.py`) is the commit panel opened from both the Agents tab
(numbered `[N]` commit hints in the prompt-panel header) and the **Commits** sub-tab of the **Artifacts** tab (`Enter`
on a timeline row). Its title renders repo, SHA, subject, and diff path — but never _when_ the commit happened:

```
COMMIT 1/4  beads  3ffa3be7b499 - chore(beads): close sase-fq.8.2
/home/bryan/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202608/06/20260806070554/commit_diffs/001.diff
```

Time is first-class metadata for a commit — it is how you tell "the commit this agent just made" from "the commit it
made two hours ago at the start of the run", and it is the one field the panel drops. The Artifacts _side detail_ pane
already shows `Committed  <date>`, so opening the modal from that pane currently _loses_ information the user could
already see.

`CommitViewSpec` (`src/sase/ace/tui/widgets/prompt_panel/_agent_display_state.py`) carries no time field, so neither
producer can supply one today.

## Design

### 1. What the panel shows

A compact **time chip**, right-aligned on the title's second (metadata) line, opposite the diff path / plan reference:

```
COMMIT 1/4  beads  3ffa3be7b499 - chore(beads): close sase-fq.8.2
/home/bryan/.sase/…/commit_diffs/001.diff                              Today 07:05:54 · 2h ago
```

Chip format — absolute wall clock, then relative age:

| When the commit landed | Chip                           |
| ---------------------- | ------------------------------ |
| Today                  | `Today 07:05:54 · 2h ago`      |
| Yesterday              | `Yesterday 21:04 · 14h ago`    |
| Earlier this year      | `Aug 6 09:31 · 12d ago`        |
| A previous year        | `Aug 6, 2025 09:31 · 366d ago` |

Rationale for each decision — implement these as specified, they are the point of the change:

- **Absolute + relative together** is already sase's established commit-time vocabulary: the CLI's `sase vcs log --full`
  footer renders `<short_id> · <author> · <HH:MM> · <2h ago>` (`src/sase/vcs_log/render.py:407`). The panel should speak
  the same language, not invent one.
- **The day tier reuses `_day_label`** (`src/sase/vcs_log/render.py:590`), the exact vocabulary of the Artifacts
  timeline's day banners (`Today` / `Yesterday` / `Aug 6` / `Aug 6, 2026`). A user who presses `Enter` on a row under
  the `Today` banner sees `Today` in the modal. The two surfaces cannot disagree because they share the labeller.
- **Seconds only for today/yesterday.** Agent commits inside one run land seconds apart, so seconds are what
  disambiguate them; for a commit from August they are noise. Older tiers use `%H:%M`, matching the timeline rows.
- **Second line, right-aligned, not the first line.** Line 1 is identity (repo · SHA · subject) and the subject is the
  thing you scan for; the modal is `width: 85%; max-width: 150`, so on an 80-col terminal a right-hand chip on line 1
  would eat the subject. Line 2 is already the dim provenance line, and a long absolute artifact path is exactly the
  right thing to ellipsize instead. Placing the chip in the _pinned_ title (not the scrollable content) keeps it visible
  while paging a diff.
- **Styled, not dim-uniform.** The absolute part gets a warm accent so it reads as data rather than chrome; the
  ` · <relative>` tail stays dim. The diff path keeps its existing `_COLOR_MUTED`.
- **No glyph.** Clock emoji are double-width and render inconsistently under the visual suite's pinned Fira Code.
  `Today 07:05:54` is self-evidently a time.
- **Absent means absent.** When no time can be resolved, render no chip at all — never `unknown`, never a guess.
  Historical commits degrade to today's exact layout.

### 2. Where the time comes from

`CommitViewSpec` gains `created_at: int | None` (epoch seconds, `None` = unknown). Producers fill it from the best
source they have, in this order:

1. **Artifacts → Commits** — already free. `build_commit_view_spec`
   (`src/sase/ace/tui/widgets/artifacts/commits_rendering.py:164`) has `entry.commit.timestamp`, the epoch author time
   from the vcs-log wire. This alone fixes the Artifacts surface completely.
2. **Agents tab, commits recorded from now on** — `write_result_marker` and `record_sdd_commit_result_marker`
   (`src/sase/workflows/commit/commit_tracking.py`) start recording `committed_at` in `commit_result.json` /
   `commit_results.json`, resolved from the VCS at commit time. That value flows through `read_commit_results_metadata`
   into `step_output["meta_commits"]` and out to the panel with no further I/O on the render path.
3. **Agents tab, imported agents** — free today: `agents_sync`'s `CommitRecord` (`src/sase/agents_sync/models.py:63`)
   _already_ carries `committed_at`, and `v2_import_rendering.py:215` already writes those dicts straight into
   `meta_commits`. Reading the key is all that is needed.
4. **Agents tab, commits already on disk (all existing agent history)** — lazily recovered from the VCS inside the
   modal's existing background diff worker, only when `created_at is None`. This is what makes the feature real on day
   one instead of only for future commits, and it costs nothing once (2) is landing times: the lookup is skipped
   whenever a time is already known.
5. Nothing resolves → no chip.

**Author time (`%at`) is the semantic**, matching the vcs-log wire, so a commit that appears on both the Agents tab and
the Artifacts timeline shows the identical instant on both. Imported records' `committed_at` (`%ct`) is used as-is when
that is all that exists; for agent-created commits the two are the same instant.

Rejected alternative: using the `commit_diffs/NNN.diff` artifact's mtime as a cheap local proxy. It is written seconds
before the commit and needs only a `stat`, but it is an approximation, and a panel that presents an approximation as a
timestamp is worse than one that shows nothing.

### 3. Scope boundary

No `sase-core` / `sase_core_rs` changes. The chip is presentation, and the time source reuses the existing
provider-agnostic `VCSProvider.log(...)` API (`src/sase/vcs_provider/_base.py:248`), which is already Rust-backed
underneath. Do not open the sibling core repo for this change.

## Implementation

Run `just install` first — workspaces are ephemeral and dependencies may be stale.

### Step 1 — Time chip renderer (`src/sase/vcs_log/render.py`)

Add a public builder next to the existing `build_timeline_day` / `build_commit_presence` family:

```python
def build_commit_time_chip(
    timestamp: int,
    *,
    now_local: datetime | None = None,
) -> Text:
    """Build the compact ``<day> <clock> · <age>`` chip for one commit time."""
```

- Resolve `dt_local = _to_local(timestamp)` and `reference = now_local or _local_now()`.
- Absolute segment: `_day_label(dt_local, reference)` + `" "` + `%H:%M:%S` when the label is `Today` or `Yesterday`,
  else `%H:%M`.
- Relative segment: `" · "` + a now-injectable relative age. Add a private
  `_relative_age_between(dt_local, now_local) -> str` implementing the same `Ns/Nm/Nh/Nd ago` ladder the module already
  uses (`_format_fetch_age`, `src/sase/vcs_log/render.py:497`), returning `"just now"` for a non-positive delta so clock
  skew cannot print a negative age. Have the existing `_relative_age` delegate to it so there is one ladder, not two.
- Style: absolute segment `#D7AF5F`, the ` · <age>` tail `dim #D7AF5F`. Set `no_wrap = True`.
- **`now_local` must be honoured** — the visual snapshot suite pins it for determinism.
- Export `build_commit_time_chip` from `__all__`.

### Step 2 — Carry the time on the view model

`src/sase/ace/tui/widgets/prompt_panel/_agent_display_state.py`: add to `CommitViewSpec`

```python
    created_at: int | None = None
```

Default `None` keeps every existing construction site (including the five in `tests/`) valid. Document it as "epoch
seconds when the commit was created; author time where available".

### Step 3 — Render the chip (`src/sase/ace/tui/modals/commit_view_modal.py`)

- Split `_build_title()` into the identity line (today's first line, unchanged `Text`) and a metadata row, and return a
  `Group`. The metadata row is a `rich.table.Table.grid(expand=True)` with two columns: left = diff path / plan
  reference (`no_wrap=True`, `overflow="ellipsis"`, existing `_COLOR_MUTED`), right = `justify="right"`, `no_wrap=True`,
  the chip.
- Emit the metadata row whenever _either_ side has content, so a commit with a time but no diff path still gets its
  chip, and a commit with neither renders exactly as today.
- The chip appears in both commit mode and plan mode — it is the same commit either way.
- `_build_title()`'s return type becomes `RenderableType`. Update the four existing assertions in
  `tests/ace/tui/modals/test_commit_view_modal.py` (lines 92, 369, 370, 379) to render the renderable through a
  `Console`/`StringIO` and assert on the output; that file already imports both.
- Add a module-level import of `build_commit_time_chip` (not a local import) so the visual test can monkeypatch it in
  this namespace.

### Step 4 — Lazy recovery for historical commits

In `src/sase/ace/tui/widgets/prompt_panel/_agent_commits.py`, beside `load_commit_diff_text`:

```python
def load_commit_created_at(spec: CommitViewSpec) -> int | None:
    """Resolve a commit's author time from the VCS when it was never recorded."""
```

- Return `None` immediately unless `spec.created_at is None` and both `spec.sha` and `spec.cwd` are set.
- `get_vcs_provider(spec.cwd).log(spec.cwd, 1, revs=(spec.sha,))`.
- **Verify identity before trusting the result.** The git hook appends `--no-merges`
  (`src/sase/vcs_provider/plugins/_git_query_ops.py:250`), so asking for a _merge_ commit silently returns its nearest
  non-merge ancestor. Accept the row only when `full_id` or `short_id` matches `spec.sha` case-insensitively; otherwise
  return `None`. A missing time is fine; a wrong time is not.
- Wrap everything in `except Exception: return None` — a recycled or deleted workspace must degrade silently, exactly
  like `load_commit_diff_text` already does.

In the modal, fold this into the **existing** diff worker rather than adding a second one:

- Introduce a small frozen `_CommitLoad(diff_text: str | None, created_at: int | None)` result and change
  `_diff_worker: Worker[str | None]` to `Worker[_CommitLoad]`; the worker body calls `load_commit_diff_text(spec)` and
  then `load_commit_created_at(spec)`.
- Cache resolved times per index alongside `_diff_text_by_index` and apply them when reading the cache back in
  `_sync_current_diff_state_from_cache`, so `ctrl+n`/`ctrl+p` round-trips do not re-shell.
- On worker success, refresh **`#commit-view-title` as well as `#commit-view-content`** — today only the content is
  updated, so a backfilled chip would not appear until the next navigation.
- Track the effective time for the current commit (spec value, else backfilled value) and have `_build_title` read that,
  not `spec.created_at` alone.

### Step 5 — Read the time on the Agents-tab path

`src/sase/ace/tui/widgets/prompt_panel/_agent_commits.py`:

- Add `_commit_created_at_from_record(record) -> int | None` accepting a `committed_at` that is either an `int` or a
  numeric `str` (the marker readers normalize to `str`), rejecting negatives, booleans, and unparseable values.
- Pass `created_at=...` in the `CommitViewSpec(...)` built by `_commit_line_from_record`.
- In `_persisted_commit_lines` (the legacy single-commit fallback), read `step_output["meta_commit_committed_at"]`
  through the same parser.
- Optional, cheap, and worth doing: `CommitDiffInfo` and `_CommitInfo` do not need the field — leave them alone.

### Step 6 — Read the time on the Artifacts path

`src/sase/ace/tui/widgets/artifacts/commits_rendering.py`, in `build_commit_view_spec`:

```python
        created_at=entry.commit.timestamp,
```

### Step 7 — Persist the time at commit time

`src/sase/workflows/commit/commit_tracking.py`:

- Add a best-effort private resolver:

```python
def _resolve_commit_created_at(cwd: str, sha: str | None) -> int | None:
```

using the same `provider.log(cwd, 1, revs=(sha,))` + identity-verification rule as Step 4, wrapped in
`except Exception: return None`. A failure here must never fail a commit.

- In `write_result_marker`, resolve from `(commit_cwd, result)` and add `"committed_at": <int>` to the marker dict when
  it resolves. Resolving _inside_ the function keeps the two call sites in `src/sase/workflows/commit/workflow.py`
  (lines 368 and 396 — the second is the idempotent re-write) consistent without changing their signatures.
- Do the same in `record_sdd_commit_result_marker` from its `cwd` / `result` arguments.
- `_persist_primary_commit_metadata` needs no change.

`src/sase/axe/run_agent_helpers_state.py`:

- `_commit_result_list_record`: pass `committed_at` through as `str(int(value))` when it is a non-negative int or
  numeric string. Widen the annotation if needed and keep the dict `str`-valued so the surrounding readers stay uniform.
- `read_commit_result_metadata`: emit `meta_commit_committed_at` from `commit_result.json`'s `committed_at`, same
  normalization.

`src/sase/ace/tui/models/_loaders/_done_loaders.py`:

- `_single_commit_record_from_metadata`: carry `meta_commit_committed_at` into the record as `committed_at`.
- `_merge_commit_records` already merges with `dict.update`, so nothing else is needed — verify this with a test rather
  than by inspection.

`COMMIT_META_KEYS` in `src/sase/ace/tui/widgets/prompt_panel/_helpers.py:89` and
`src/sase/ace/tui/models/_agent_clan_sections.py:53` detect _presence_ of commit metadata and do not need the new key;
leave them as they are.

## Tests

Add or extend, following each file's existing style:

- **`tests/vcs_log/`** (match the existing layout for `render.py` tests) — `build_commit_time_chip` across all four day
  tiers and the relative ladder (`just now`, `s`, `m`, `h`, `d`), driven by an explicit `now_local` so the assertions
  are deterministic; seconds present for today/yesterday and absent for older tiers; a future timestamp renders
  `just now`, never a negative age.
- **`tests/ace/tui/modals/test_commit_view_modal.py`** — chip present and correctly formatted when `created_at` is set;
  no chip and no layout change when it is `None`; the chip appears after the worker backfills a time (assert the title
  widget was refreshed, not just the content); the chip tracks the selected commit across `ctrl+n`/`ctrl+p`; the chip
  renders in plan mode; a long subject and a long diff path do not push the chip off the row. Update the four existing
  `_build_title()` assertions for the new return type.
- **`_agent_commits`** — `committed_at` as `int` and as numeric `str` both parse; missing, negative, boolean, and
  garbage values yield `None` and never raise; `load_commit_created_at` returns `None` for a missing `cwd`/`sha`, for a
  provider that raises, and — importantly — when the provider returns a commit whose id does not match the requested SHA
  (the `--no-merges` trap).
- **`commits_rendering`** — `build_commit_view_spec` carries `entry.commit.timestamp`.
- **`tests/test_commit_artifacts.py`** (or the closest existing home) — `write_result_marker` records `committed_at`; a
  resolver failure still writes a complete marker; `record_sdd_commit_result_marker` behaves the same;
  `_commit_result_list_record` and `read_commit_result_metadata` pass the value through; `_merge_commit_records`
  preserves it.
- **Visual** — update the existing `commit_view_modal_120x40` and `commit_plan_view_modal_120x40` fixtures in
  `tests/ace/tui/visual/test_ace_png_snapshots_preview_panel.py` to give the specs a `created_at`, monkeypatching
  `sase.ace.tui.modals.commit_view_modal.build_commit_time_chip` with a pinned `now_local` (the file already
  monkeypatches `load_commit_diff_text` and `load_commit_plan_document` in that namespace). Regenerate both goldens with
  `just test-visual -- --sase-update-visual-snapshots` and eyeball the PNGs before accepting: the chip must be
  right-aligned, unclipped, and not colliding with the path.

## Verification

```bash
just install
just check
just test-visual
just check-full          # before landing
```

Then confirm by eye in the real TUI: open a commit from an Agents-tab `[N]` hint (a historical agent, to exercise the
lazy path) and from Artifacts → Commits, and check the chip agrees with `git show -s --format=%at <sha>` and with the
timeline's day banner.

## Out of scope

- The compact commit rows in the Agents-tab prompt-panel header (`[1] 3ffa3be7b499 <subject>`). That header is already
  dense; the modal is where the detail belongs.
- The Artifacts side detail pane, which already renders `Committed  <full date>`. Do not restyle it in this change.
- Backfilling `committed_at` into already-written `commit_results.json` markers. Step 4 recovers those at view time;
  rewriting historical artifacts is not worth the risk.

## Observation for the implementer

While reading `meta_commits`, note that imported (agents-sync) records carry `subject`, not `message` — so
`_commit_info_from_record` / `_commit_line_from_record`
(`src/sase/ace/tui/widgets/prompt_panel/_agent_commits.py:186,197`) appear to fall back to `(message unavailable)` for
imported agents. This was noticed while designing, **not verified**, and is not part of this change. If you confirm it
against a real imported agent, file it separately with `/sase_new_task` rather than folding a second fix into this one.
