---
tier: epic
title: ViewReport notification action and the ci_watch release report
goal: 'Selecting a ci_watch release notification in ACE opens a beautiful, current
  report of recently merged and still pending release PRs instead of raising "Unsupported
  notification action: (none)", and any producer can attach a structured report to
  a notification through a generic, fail-closed ViewReport action.

  '
phases:
- id: contract
  title: ViewReport action contract and report loader
  depends_on: []
  size: medium
  description: 'contract: add the generic notification report contract, the fail-closed
    loader that resolves a live report file or an inline snapshot, ViewReport registration
    in the badge/icon/toast tables, and the fix that stops action-less notifications
    from raising an unsupported-action warning.

    '
- id: ui
  title: Report preview pane and full-screen report modal
  depends_on:
  - contract
  size: medium
  description: 'ui: render the loaded report in the notification modal''s right pane
    with a provenance header, open a scrollable full-screen ReportModal from Enter,
    wire the ViewReport dispatch branch, and cover both surfaces with unit tests and
    PNG visual snapshots.

    '
- id: chop
  title: ci_watch release ledger, published report, and notification wiring
  depends_on:
  - contract
  size: medium
  description: 'chop: give the bugyi-chops ci_watch chop a durable release ledger,
    a per-tick published release report document, pending release PR discovery for
    every release repo, and notifications that carry the ViewReport action.

    '
- id: verify
  title: Documentation and end-to-end verification
  depends_on:
  - ui
  - chop
  size: small
  description: 'verify: document the ViewReport action and the ci_watch release report,
    then verify the whole path end to end against live axe state with the full check
    and visual suites.

    '
create_time: 2026-07-29 10:54:50
status: wip
bead_id: sase-at
---

- **PROMPT:** [202607/prompts/notification_release_report.md](prompts/notification_release_report.md)
- **BEAD:** [sase-at](https://github.com/sase-org/sase--beads/blob/main/pages/sase-at/README.md)

# Plan: ViewReport notification action and the ci_watch release report

## Problem

`sase ace` shows the toast **"Unsupported notification action: (none)"** whenever a `ci_watch` notification is selected
in the notification modal.

Confirmed root cause, from live state:

```json
{
  "sender": "ci_watch",
  "notes": ["Merged release PR #46 for sase-org/sase-core"],
  "tags": ["release"],
  "action": null,
  "action_data": {}
}
```

The chop's `AgentsGate.notify()` helper posts only `notes` and `tags` through `sase notify create`, so `action` is
`None`. In `src/sase/ace/tui/actions/agents/_notification_modal_flow.py:211` the dispatch chain falls through to an
`else` branch that warns for **any** unrecognized action, including the perfectly legitimate `None`. So there are two
defects layered together:

1. A generic defect — an informational, action-less notification is treated as a producer error.
2. A missing capability — a release event carries no way to see the release picture it belongs to.

The user wants the second defect fixed properly: selecting one of these notifications should show a report of recent
releases (release-please / release-plz PRs the chop merged) **and** release PRs that are still waiting to be merged,
with a good preview in the notification modal's right pane.

## Design

### Why a generic action rather than a ci_watch special case

`ci_watch` lives in a third-party package (`bbugyi200/bugyi-chops`). Teaching `sase` to read a chop-private release
ledger would couple the TUI to one external chop's schema. Instead this plan adds one **generic** notification action,
`ViewReport`, that renders an already-standard artifact: the **chop report document**
(`{"title": ..., "blocks": [...]}`) produced by `sase.chops.ChopReport` and rendered by
`sase.axe.chop_report_render.render_chop_report`.

That reuses three things that already exist and are already tested:

- `ChopReport` — the typed report builder any chop can use (`src/sase/chops/report.py`).
- `render_chop_report` — the literal, tone-styled Rich renderer with no markup/ANSI interpretation
  (`src/sase/axe/chop_report_render.py`), already used by the AXE tab's RESULT card.
- The Rust chop-result validator, which **already validates the embedded `report` field**. Verified in this workspace:

  ```
  validate_chop_result({... "report": {"title":"X","blocks":[{"kind":"bogus"}]}})
    -> ValueError: unknown variant `bogus`, expected one of `headline`, `heading`, `text`, `kv`, `rows`, ...
  validate_chop_result({... "report": {"blocks": "nope"}})
    -> ValueError: invalid type: string "nope", expected a sequence
  ```

  So a report document can be validated fail-closed by wrapping it in a minimal chop-result envelope. **No `sase-core`
  Rust change is needed**, and the schema authority stays in Rust, which satisfies the Rust core backend boundary rule:
  validation is backend, rendering to Rich `Text` is presentation and stays in this repo alongside the existing
  renderer.

The result is that `ci_watch` gets its release report, and every other chop, hook, or agent gets the ability to attach a
structured report to a notification for free.

### The ViewReport contract

A notification participates when `action == "ViewReport"`. Its `action_data` (string→string, per
`Notification.action_data`) may carry:

| Key            | Meaning                                                                                       |
| -------------- | --------------------------------------------------------------------------------------------- |
| `report_path`  | Absolute (or `~`-prefixed) path to a JSON report document that the producer keeps up to date. |
| `report`       | A JSON-encoded report document embedded directly in the notification — an immutable snapshot. |
| `report_title` | Optional display title override; otherwise the document's own `title`, else `"Report"`.       |

Resolution order and provenance, which the UI shows to the reader:

1. `report_path` resolves, loads, and validates → provenance **live**, stamped with the file's mtime (`updated 2m ago`).
2. Otherwise `report` parses and validates → provenance **snapshot**, stamped with the notification timestamp
   (`captured 3h ago`).
3. Otherwise → a single explicit, styled failure line naming the reason (missing file, too large, invalid document).
   Never an exception, never an empty pane.

Both keys are optional and both may be present; that combination is deliberate and is the reliability story. A
`ci_watch` notification points at a stable published path so an hours-old notification still opens the **current**
release picture, while the inline snapshot guarantees the pane renders something honest even if axe state was wiped.

Fail-closed loader rules (all enforced in the loader, not the UI):

- The path must expand to an absolute path and be an existing regular file.
- The file must be at most **256 KiB**; a larger file is a failure, not a truncation.
- The content must parse as a JSON object and pass report validation via the chop-result envelope.
- Any `OSError`, `JSONDecodeError`, or `ValueError` becomes a `NotificationReport` carrying an `error` string.

Rendering safety is already guaranteed by `render_chop_report`: it builds Rich `Text` with explicit styles and never
interprets console markup or ANSI from the document.

### Where the release data comes from

Per-chop run history is capped at `MAX_CHOP_RUN_HISTORY = 10` runs (`src/sase/axe/_state_chops.py`). At `ci_watch`'s
300s cadence that is roughly 50 minutes of history — far too short for "recent releases". So `ci_watch` gains a durable
ledger of its own in the lumberjack state dir (`invocation.context.state_dir`, which resolves to
`~/.sase/axe/lumberjacks/ci_watch/`, the same directory that already holds `ci_watch_red_streaks.json`):

- `ci_watch_releases.json` — the chop-private ledger: every merge it has performed, plus per-PR notification bookkeeping
  so a pending release is announced once rather than every five minutes.
- `ci_watch_releases.report.json` — the **published** report document, rewritten atomically on every tick. This is the
  generic artifact `sase` renders; it is the only file `sase` ever reads.

Because the published report is rewritten every tick, opening any release notification shows a picture that is at most
one chop interval stale, with no network calls, no `gh` auth dependency, and no latency inside the TUI. That is the
single most important reliability decision in this plan.

### What the report looks like

Built with the existing `ChopReport` block kinds, so it inherits the same tone palette as the AXE tab:

```
━━ RELEASES ────────────────────────────────────────────────
  ✓ 2 merged today · 3 pending · 1 blocked

  RECENT
  REPOSITORY            PR     VERSION   MERGED
  ✓ sase-org/sase-core  #46    0.9.3     07-29 10:14
  ✓ sase-org/sase-core  #45    0.9.2     07-29 09:39
  ✓ sase-org/sase-tel…  #12    0.2.0     07-28 17:41

  PENDING
  REPOSITORY            PR     STATE                  AGE
  ◆ sase-org/sase       #312   checks not green       14m
  ▲ sase-org/sase-gith… #88    base branch not green  2h
  · sase-org/sase-tel…  -      no release PR          -

  ────────────────────────────────────────────────────────
  updated 2m ago   window 30d   merges recorded 11
```

Tones carry the meaning: `ok` for merged, `warn` for a pending PR that is progressing, `error` for one that is blocked,
`muted` for a repo with no release PR. `RECENT` uses **absolute** local timestamps because the document is a file that
may be read minutes after it was written; relative freshness belongs to the single provenance line, where it is honest.

## Phase 1 — `contract`: ViewReport action contract and report loader

All paths in this phase are in the `sase` repo (this workspace).

### 1.1 Report validation helper

Add to `src/sase/core/axe_chop_facade.py`:

```python
def validate_chop_report(report: Mapping[str, Any]) -> dict[str, Any]:
    """Fail-closed validate a standalone chop report document."""
    envelope = validate_chop_result(
        {
            "schema_version": CHOP_RESULT_SCHEMA_VERSION,
            "status": "ok",
            "counters": {},
            "proposed_launches": [],
            "report": dict(report),
        }
    )
    validated = envelope.get("report")
    if not isinstance(validated, dict):
        raise ValueError("report document is empty")
    return validated
```

Re-export it from `src/sase/chops/__init__.py` alongside the existing chop SDK surface so third-party chop packages can
validate a report before publishing it. Add a docstring note that the envelope is an implementation detail of reusing
the single Rust schema authority.

### 1.2 Notification report loader

New module `src/sase/notifications/report.py`:

```python
REPORT_ACTION = "ViewReport"
MAX_REPORT_BYTES = 256 * 1024

ReportSource = Literal["live", "snapshot"]

@dataclass(frozen=True)
class NotificationReport:
    document: dict[str, Any] | None   # validated report, or None on failure
    source: ReportSource | None       # None only when document is None
    title: str                        # resolved display title
    path: str | None                  # the live path, when one was used
    updated_at: str | None            # ISO-8601: file mtime (live) or notification ts (snapshot)
    error: str | None                 # short human-readable failure reason


def is_report_notification(notification: Notification) -> bool: ...
def load_notification_report(notification: Notification) -> NotificationReport | None: ...
```

- `is_report_notification` returns `True` only for `action == REPORT_ACTION`.
- `load_notification_report` returns `None` for non-report notifications and always a `NotificationReport` otherwise —
  never raises.
- Title resolution: `action_data["report_title"]` → `document["title"]` → `"Report"`. Bound the title to 64 characters.
- Failure reasons are short and specific, e.g. `"report file not found"`, `"report file is too large (312 KiB)"`,
  `"report document is invalid: unknown variant `bogus`"`. Truncate a validator message to 160 characters.
- The loader performs no network access and no subprocess calls.

Export `NotificationReport`, `is_report_notification`, `load_notification_report`, and `REPORT_ACTION` from
`src/sase/notifications/__init__.py`, following the existing export style there.

### 1.3 Register the action

- `src/sase/ace/tui/modals/notification_modal_constants.py`: add `"ViewReport": "[report]"` to `ACTION_BADGES` and
  `"ViewReport": "📊"` to `ACTION_ICONS`. A producer-supplied `icon` still wins, which is how `ci_watch` gets its own
  glyph.
- `src/sase/ace/tui/actions/agents/_toasts.py`: add a `ViewReport` branch to `_format_notification_toast` returning
  `(note or "Report available", "information")`, and add `"ViewReport": ("report", "reports")` to `_ACTION_LABELS`.

### 1.4 Stop warning about action-less notifications

In `src/sase/ace/tui/actions/agents/_notification_modal_flow.py`, the terminal `else` in `_on_dismiss` currently warns
for `result.action or '(none)'`. Split it:

- `result.action` is `None` or blank → do nothing beyond the already-performed mark-read. An informational notification
  with no action is a valid, common shape, and selecting it should feel like a no-op, not an error.
- `result.action` is a non-empty unknown string → keep the existing warning, without the `(none)` fallback text, since
  that genuinely means a producer emitted an action this build does not understand.

`tests/ace/tui/test_notification_custom_gate.py:332` asserts the warning for the unknown action `FutureGate`; that
assertion must keep passing unchanged.

### 1.5 Tests for this phase

Add `tests/test_notification_report.py`:

- Live path wins over inline snapshot when both are present; `source == "live"`, `updated_at` equals the file mtime.
- Inline snapshot is used when `report_path` is absent, missing on disk, oversized, or invalid; `source == "snapshot"`.
- Every failure mode returns `document is None` with a non-empty, bounded `error`, and raises nothing: missing file,
  directory instead of file, non-JSON bytes, JSON array instead of object, oversized file, invalid block kind, invalid
  tone.
- `~`-prefixed and relative paths: `~` expands, a relative path is rejected as invalid.
- Title resolution precedence and truncation.
- `is_report_notification` is `False` for `None` and for other actions.

Add to the existing notification-flow tests:

- Selecting a notification with `action=None` produces **no** toast and still marks the notification read.
- Selecting a notification with an unknown non-empty action still produces the warning toast.

## Phase 2 — `ui`: Report preview pane and full-screen report modal

All paths in this phase are in the `sase` repo.

### 2.1 Right-pane preview

The notification modal's right pane is built by
`src/sase/ace/tui/modals/notification_modal_attachments.py::NotificationAttachmentMixin._display_file`, which already
delegates to `self._render_question_pane(notification)` before falling back to file rendering. Follow that exact
pattern.

New module `src/sase/ace/tui/modals/notification_modal_report.py` with `NotificationReportMixin` exposing
`_render_report_pane(notification) -> tuple[str, RenderableType] | None`:

- Return `None` unless `is_report_notification(notification)`.
- Load through `load_notification_report`.
- On success, compose a `rich.console.Group`:
  1. **Provenance line** — `live · updated 2m ago` in the `ok` tone, or `snapshot · captured 3h ago` in the `muted`
     tone, reusing `format_relative_time` from `sase.notifications`.
  2. `render_chop_report(document, width=<pane width>)`.
  3. When `notification.files` is non-empty, a dim `attachments: name.json, other.txt` footer line, so attachments stay
     discoverable even though the report has taken over the pane.
- On failure, a single `error`-toned line with the reason plus a dim hint naming the path that was tried.
- Pane title: `self._detail_title(notification, report.title)`, which keeps the existing `<icon> <sender> · <title>`
  header shape.

Pane width comes from the visible `#notification-file-scroll` region, mirroring how
`NotificationAttachmentMixin._image_preview_size` sizes image previews; fall back to the module default when the region
is not yet laid out.

Wire the mixin into `NotificationModal`'s bases in `src/sase/ace/tui/modals/notification_modal.py` and add the
`_render_report_pane` call in `_display_file` immediately after the question-pane attempt, using the identical
title/content/`_reset_file_scroll` handling. Also call `self._set_image_preview_mode(False)` on that branch so a
previously viewed image attachment does not leave the layout shifted.

### 2.2 Full-screen report modal

New `src/sase/ace/tui/modals/report_modal.py` with `ReportModal(ModalScreen[None])`, modeled closely on
`preview_panel_modal.py`:

- Constructor takes the resolved `NotificationReport`.
- **Reload at open time** rather than reusing the pane's cached load, so pressing Enter always shows the freshest
  published report.
- Compose: a `#report-title` `Static` (icon, title, provenance, and the source path in dim when live), a
  `VerticalScroll#report-scroll` wrapping `Static#report-content`, and a `#report-footer` hint line.
- Content is `render_chop_report(document, width=<scroll region width>)`; re-render on resize so wide terminals get the
  full table layout rather than the narrow fallback.
- Bindings, matching `PreviewPanelModal` so the muscle memory transfers: `escape`/`q` close, `ctrl+d`/`ctrl+u` half
  page, `j`/`k` line, `g`/`G` top/bottom. Add `y` to copy the report path via
  `sase.ace.tui.actions.clipboard.copy_to_system_clipboard` (warn when the report is an inline snapshot with no path)
  and `e` to open the report file in `$EDITOR` through `sase.ace.hints.build_editor_args` under `self.app.suspend()`,
  matching `action_open_in_editor`.
- Add CSS to `src/sase/ace/tui/styles.tcss` next to the `PreviewPanelModal` rules (around line 4241), reusing the same
  `85%`/`max-width: 150`/`max-height: 42` geometry so the modal family looks consistent.

### 2.3 Dispatch

- New `handle_view_report(app, notification) -> bool` in `src/sase/ace/tui/actions/agents/_notification_handlers.py`,
  alongside `handle_view_error_report`. It loads the report, pushes `ReportModal` on success, and emits a warning toast
  naming the failure reason otherwise.
- Re-export it from `src/sase/ace/tui/actions/agents/_notification_actions.py` in the same manner as the neighboring
  handlers.
- Add the `elif result.action == "ViewReport": handle_view_report(self, result)` branch to `_on_dismiss` in
  `_notification_modal_flow.py`, before the unknown-action fallback. `ViewReport` is **not** added to the stay-unread
  list, so selecting it marks it read like any other informational action.

### 2.4 Help and hints

Per this repo's ACE guidelines, keep the help popup in sync:

- `src/sase/ace/tui/modals/help_modal/agents_bindings.py` — the notification entry near line 432 should mention that a
  report notification opens the full report.
- `notification_modal_constants.py::DEFAULT_HINT_TEXT` already documents `Enter: select`; no change is needed there, but
  the new `ReportModal` footer must document its own keys.

### 2.5 Tests for this phase

- `tests/ace/tui/test_notification_report_pane.py`: the pane returns `None` for non-report notifications; renders the
  provenance line for live and snapshot sources; renders the failure line without raising for each loader failure; the
  attachments footer appears only when files exist; the report title reaches the pane header.
- Dispatch tests: `ViewReport` pushes `ReportModal`; a failing load produces exactly one warning toast and no modal.
- `ReportModal` tests: content renders, scroll bindings move the viewport, `y` copies the path, `y` warns for a snapshot
  with no path.
- Two PNG visual snapshots, following `tests/ace/tui/visual/test_ace_png_snapshots_notification_question.py` as the
  template (it is the closest existing precedent — a rich, read-only right pane for one notification):
  1. `notification_report_pane_<W>x<H>.png` — the notification modal with a release report selected.
  2. `notification_report_modal_<W>x<H>.png` — the full-screen `ReportModal`. Use a fixed, deterministic fixture report
     (absolute timestamps, no clock-dependent strings inside the document) so the snapshot is stable; the provenance
     line must also be pinned by freezing the notification timestamp and file mtime in the fixture. Generate goldens
     with `just test-visual --sase-update-visual-snapshots` and inspect the rendered PNG before accepting it — the point
     of these snapshots is that a human confirmed the result is actually beautiful.

## Phase 3 — `chop`: ci_watch release ledger, published report, and notification wiring

This phase edits repositories other than this workspace. Open each one through the `/sase_repo` skill first and use only
the path it prints:

```bash
sase repo open gh:bbugyi200/bugyi-chops -r "Add the ci_watch release ledger and ViewReport notification wiring"
sase repo open chezmoi -r "Update the ci_watch chop description for the release report"
```

The chop source is `src/bugyi_chops/ci_watch.py`; its tests are `tests/test_ci_watch.py`. The chop is configured in
chezmoi at `home/dot_config/sase/sase_athena.yml` under the `ci_watch` lumberjack, with `release_repositories` covering
`sase-org/sase` (release-please), `sase-org/sase-core` (release-plz), `sase-org/sase-github` (release-please), and
`sase-org/sase-telegram` (release-please), and `merge_enabled: true`.

### 3.1 Pending release PRs for every release repo

Today `build_ci_watch_result` skips release handling entirely when a repo is not `GREEN`:

```python
for repo in config.merge_order:
    generator = config.release_repositories.get(repo)
    if generator is None or states[repo] is not RepoState.GREEN:
        continue
```

That is correct for _merging_ and must not change — a red branch must never auto-merge a release. But it means a pending
release PR on a red repo is invisible, which is exactly the case a human most needs to see.

Split observation from action:

- **Observation** runs for every repo in `config.release_repositories`, every tick: call `release_pr_numbers(repo)`, and
  for at most `MAX_RELEASE_PR_DETAILS = 8` PRs per tick (in `merge_order` order) call `release_pr(repo, number)` to get
  the details. Record repo, PR number, url, head oid, draft flag, mergeable, merge state, rollup summary, `createdAt`,
  and `title`.
- **Action** (the existing `plan_release_merge` / merge loop) keeps its current `GREEN`-only guard and its existing
  ordering and caps, unchanged.

Extend `ReleasePr` with `title: str` and `created_at: str`, add `title,createdAt` to the `gh pr view --json` field list
in `GitHubReader.release_pr`, and validate both in `ReleasePr.from_json` with the module's existing `_bounded` and
`CiWatchError` conventions — an unparseable `createdAt` is a `CiWatchError`, consistent with the file's fail-closed
style.

**Critical constraint:** observation must never break the chop's primary duties. Wrap report-data gathering so a
`CiWatchError` during observation degrades that repo's report row to an `error`-toned row carrying the bounded reason,
and never propagates into the fix/merge paths or the chop's exit status.

Derive a display version from the PR title with a bounded regex for a semver-like token (release-please and release-plz
both produce titles such as `chore(main): release 0.9.3` and `chore: release v0.9.3`); fall back to `-` when no token is
found. Never fail a tick over a version string.

### 3.2 Durable release ledger

New `ci_watch_releases.json` in `invocation.context.state_dir`, written with the file's existing `_atomic_write_json`
helper and loaded with the same defensive, version-checked style as `_load_streaks`:

```json
{
  "version": 1,
  "merges": [
    {
      "repo": "sase-org/sase-core",
      "number": 46,
      "url": "https://github.com/sase-org/sase-core/pull/46",
      "head_oid": "…",
      "version": "0.9.3",
      "generator": "release-plz",
      "merged_at": "2026-07-29T10:14:17-04:00"
    }
  ],
  "announced_pending": { "sase-org/sase#312": { "first_seen_tick": 2 } }
}
```

- Append one entry per successful merge, at the existing `counters["merged"] += 1` site.
- Retention: keep at most `MAX_LEDGER_MERGES = 50` entries and drop entries older than 90 days. A malformed or
  unreadable ledger is treated as empty, exactly as `_load_streaks` treats a malformed streak file — a corrupted report
  ledger must never stop a merge.
- `announced_pending` keys are `<repo>#<number>`; entries are pruned when the PR is no longer open. This is the
  bookkeeping that keeps a blocked release from producing a notification every five minutes.
- Inject the clock as a parameter defaulting to `datetime.now(...)` so `tests/test_ci_watch.py` can assert exact
  timestamps, matching the file's existing dependency-injection style for `CommandRunner`, `ActstatClient`,
  `GitHubReader`, and `AgentsGate`.

### 3.3 Published release report

Build a dedicated `ChopReport` titled `RELEASES` with the layout shown in the Design section, then write it as a
standalone JSON document to `ci_watch_releases.report.json` in the state dir via `_atomic_write_json`, on **every** tick
— including no-op ticks, so freshness does not depend on anything happening.

- Validate with `validate_chop_report` from `sase.chops` (Phase 1) before writing. A validation failure must be logged
  and skipped, leaving the previous good report in place, rather than raising.
- `RECENT` shows up to 8 merges from the last 30 days, newest first, `ok` tone, `✓` glyph, with absolute `MM-DD HH:MM`
  local timestamps.
- `PENDING` shows one row per release-configured repo: an open release PR with its blocking reason (reusing the exact
  `plan_release_merge` reason vocabulary, humanized — `release_pr_checks_not_green` → `checks not green`,
  `default_branch_not_green` → `base branch not green`, and so on), or a `muted` `no release PR` row.
- Headline: `<n> merged today · <n> pending · <n> blocked`, toned `error` when anything is blocked, `warn` when
  something is pending, `ok` otherwise.
- Footer `kv` block: `updated` (absolute local time), `window 30d`, `merges recorded <n>`.
- The chop's existing per-tick `RELEASE` section in `_build_ci_watch_report` stays as is; it answers "what did this tick
  do", which is a different question from "where do releases stand". Share the row-formatting helpers between the two
  rather than duplicating the reason vocabulary.

### 3.4 Notifications carrying the action

Extend `AgentsGate.notify` to accept `icon`, `action`, `action_data`, and `tags`, and to include them in the JSON piped
to `sase notify create`. `sase notify create` already accepts all of these (see
`src/sase/main/notify_handler.py::_handle_notify_create`) and rejects only privileged gate actions; `ViewReport` is not
privileged.

Emit:

- **On merge** — keep the existing note `Merged release PR #<n> for <repo>`, and add a second note line
  `<n> merged today · <n> pending`. Set `icon: "🚢"`, `tags: ["release"]`, `action: "ViewReport"`, and
  `action_data: {"report_path": "<state_dir>/ci_watch_releases.report.json", "report_title": "Releases"}`.
- **On a newly blocked pending release** — one notification per `<repo>#<number>`, fired only after the PR has been
  observed pending and unmergeable for **2 consecutive ticks**, so a PR that is simply about to auto-merge never
  produces one. Note: `Release PR #<n> for <repo> needs attention: <humanized reason>`. Same icon, tags, action, and
  `action_data`.
- The existing `ci_fix` proposal notification keeps its current shape; it is not a release event.

Do **not** inline the full report into `action_data["report"]` from the chop. The published file is the live source, and
duplicating a multi-kilobyte snapshot into every notification would bloat the notification store for no gain. The inline
key exists in the contract for producers that have no durable state directory.

### 3.5 chezmoi

Update the `ci_watch` chop `description` in `home/dot_config/sase/sase_athena.yml` to mention that the chop publishes a
release report and that its release notifications open it. No new config vars are required — the state dir already comes
from the chop context.

### 3.6 Tests for this phase

In `bugyi-chops`, extend `tests/test_ci_watch.py` using the existing fake-runner fixtures:

- Pending release PRs are observed and reported for a repo whose default branch is red, while **no merge is planned or
  attempted** for it.
- The merge ledger accumulates across ticks, prunes past 50 entries and past 90 days, and survives a corrupt ledger file
  by treating it as empty.
- The published report validates against `validate_chop_report`, and a report that would fail validation leaves the
  previously written file untouched.
- Version extraction handles `chore(main): release 0.9.3`, `chore: release v0.9.3`, and a title with no version.
- A blocked pending PR notifies once, not on the following tick; it notifies again only after the PR has been merged or
  closed and a new one appears.
- `AgentsGate.notify` sends `action: "ViewReport"` with the published report path, asserted against the JSON piped to
  `sase notify create`.
- A `gh` failure during report observation degrades to an error row and does not change fix or merge behavior.

Run the `bugyi-chops` repo's own `just` targets for lint and tests inside that checkout.

## Phase 4 — `verify`: Documentation and end-to-end verification

### 4.1 Documentation

- `docs/notifications.md`: add `ViewReport` to the action list at line 211; add a section documenting the contract table
  from the Design section above, the resolution order, the fail-closed loader limits, and the live-versus-snapshot
  provenance display. Note that action-less notifications are informational and select as a no-op.
- `docs/axe.md`: in the chop section, document that a chop may publish a report document and reference it from a
  notification through `ViewReport`, pointing at `ChopReport` and `validate_chop_report`.

### 4.2 End-to-end verification

1. `just install` in this workspace (required after any gap in workspace use).
2. Deploy the chezmoi change and reinstall `bugyi-chops` into the environment axe uses.
3. `sase axe chop run ci_watch -L ci_watch --chop-verbose` to force a tick, then confirm
   `~/.sase/axe/lumberjacks/ci_watch/ci_watch_releases.report.json` exists and validates.
4. Trigger or wait for a release notification, then open `sase ace`, select it, and confirm: the right pane shows the
   report with a provenance line, Enter opens the full modal, and **no** "Unsupported notification action" toast
   appears.
5. Select one of the **pre-existing** action-less `ci_watch` notifications and confirm it is now a silent no-op. Those
   older rows will never gain a report — they were created before the action existed — and that is accepted; the next
   release event produces a fully wired notification. Say so plainly in the completion report rather than implying the
   backlog was migrated.
6. `just check` in this workspace, plus `just test-visual` for the two new PNG snapshots.

## Risks and decisions

| Decision                                                  | Rationale                                                                                                              |
| --------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| Generic `ViewReport` instead of a `ci_watch` special case | Keeps chop-private schemas out of the TUI and gives every chop the same capability.                                    |
| Reuse the Rust chop-result validator via an envelope      | One schema authority, fail-closed validation, and no cross-repo Rust change. Verified working in this workspace.       |
| Live file path plus optional inline snapshot              | Freshness without network calls in the TUI, and a pane that still renders honestly if axe state is wiped.              |
| Report rewritten every tick, including no-op ticks        | Freshness must not depend on something having happened.                                                                |
| Absolute timestamps inside the document                   | A file read minutes after it was written must not lie; relative freshness lives in the single provenance line.         |
| Observation split from the merge guard                    | Pending releases on a red branch become visible without ever weakening the rule that a red branch does not auto-merge. |
| Ledger corruption treated as empty                        | A reporting concern must never be able to block a merge.                                                               |
| Two-tick debounce on pending-release notifications        | A PR that is about to auto-merge should not generate noise; a genuinely stuck one should be surfaced once.             |

The main reliability risk is added `gh` API traffic: observing release PRs for every release repo each tick adds roughly
one `gh pr list` per release repo plus up to `MAX_RELEASE_PR_DETAILS` `gh pr view` calls. At four release repos and a
300s cadence that is well within rate limits, and the per-tick detail cap bounds the worst case. Phase 3 must keep that
cap and must keep observation failures non-fatal.
