---
tier: epic
title: Render every user-facing timestamp in the configured timezone
goal: 'Every timestamp SASE shows a human — TUI panes, CLI tables, generated Markdown
  pages — is rendered in the configured `timezone`, never in UTC and never in the
  host system clock, and a repo-wide guard test keeps new UTC/system-clock display
  sites from landing.

  '
phases:
- id: helpers
  title: Shared display helpers in sase.core.time
  depends_on: []
  size: small
  description: 'helpers: add `parse_local`/`format_local` to `sase.core.time` so every
    display site has one way to turn a stored timestamp (aware-UTC ISO, naive ISO,
    or epoch) into an aware configured-tz value, plus divergence-fixture unit tests.

    '
- id: artifacts
  title: Artifacts tab and artifact CLI
  depends_on:
  - helpers
  size: medium
  description: 'artifacts: fix the Files/Beads/Plans panes, the artifact-ref completion
    menu, and `sase artifact list` so artifact and bead timestamps render in the configured
    timezone instead of raw UTC.

    '
- id: tui-panels
  title: ACE modals, tools panel, and file panel
  depends_on:
  - helpers
  size: medium
  description: 'tui-panels: fix the logs, statistics, project-inventory, tasks, saved-group,
    and roster displays plus the tools and file panel fetch clocks, which currently
    show UTC or the host system clock.

    '
- id: cli-pages
  title: CLI tables, generated Markdown pages, and telemetry defaults
  depends_on:
  - helpers
  size: medium
  description: 'cli-pages: fix `sase task`, `sase repo log`, `sase memory log`, `sase
    skills log`, the agents-sync and bead-page Markdown renderers, the memory review
    TUI, notification-gate debug dumps, and the telemetry render tz defaults.

    '
- id: artifact-dates
  title: Artifact-file calendar dates in the configured timezone
  depends_on:
  - helpers
  size: medium
  description: 'artifact-dates: mint artifact `created_at` and the retention `now`
    with the configured-tz offset and make the Rust core bucket calendar dates by
    that embedded offset, so `date:`/`since:` filtering agrees with the displayed
    day.

    '
- id: guard-docs
  title: Repo-wide guard test and documentation
  depends_on:
  - helpers
  - artifacts
  - tui-panels
  - cli-pages
  - artifact-dates
  size: small
  description: 'guard-docs: add an allowlisted AST guard that fails on new system-clock
    and UTC-display patterns under `src/`, document the display convention, and run
    the full check suite.'
proposed_by: bbugyi200.athena.sn
create_time: 2026-08-03 07:44:41
status: done
bead_id: sase-em
---

- **PROMPT:** [prompts/202608/timezone_display_consistency.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/timezone_display_consistency.md)
- **BEAD:** [sase-em](https://github.com/sase-org/sase--beads/blob/main/pages/sase-em/README.md)

# Plan: Render every user-facing timestamp in the configured timezone

## Background

`timezone` in `sase.yml` is documented as governing **all** SASE wall-clock display and timestamp generation
(`docs/configuration.md`, the `timezone` entry). `sase.core.time` states the same contract in its module docstring:
never use the bare system clock for a value that is displayed alongside a configured-tz value.

A prior tale (`plans/202607/timezone_runtime_consistency.md`, commit `c318af1e7`) fixed the _arithmetic_ half of this
problem — runtimes, countdowns, `age>` filtering, BY_DATE bucketing, reply dividers — by introducing `local_now()`,
`to_local()`, and `local_timezone_name()`, plus the `tz_divergence` fixture in `tests/conftest.py` (configured
`America/New_York`, system `UTC`). It deliberately left several display sites alone, most explicitly `logs_pane` ("stays
as-is (explicitly labeled UTC)").

Since then, the _display_ half has drifted again. On the affected host (system tz `UTC`, configured `America/New_York`)
the following user-facing surfaces show UTC or the host system clock:

- **Artifacts → Files**: every row's `HH:MM` and its Today/Yesterday date group. `file_row_text` calls
  `_artifact_file_datetime(row, local_timezone=now)`, which only converts when `local_timezone.tzinfo is not None` — but
  `files_pane` passes `local_now()`, which is **naive by construction**, so the aware-UTC `created_at` is formatted
  verbatim. A file written at 21:30 EDT displays as `01:30` and groups under tomorrow.
- **Artifacts → Files detail / Beads detail**: `created_at`/`updated_at`/`closed_at` are sliced out of the stored UTC
  ISO string and printed raw.
- **Logs pane**: source mtimes and toast session headers are formatted with a literal `UTC` suffix.
- **Statistics pane and its help modal, project inventory, tasks pane, member roster, tools panel, file panel**: bare
  `datetime.now()` / argument-less `.astimezone()` / tz-less `datetime.fromtimestamp()` — the host system clock.
- **CLI**: `sase artifact list` (CREATED), `sase task show` (Created/Started/Finished), `sase repo log`,
  `sase memory log`, `sase skills log` all print the stored UTC ISO string verbatim.
- **Generated Markdown**: bead pages and agents-sync bundles render commit and instant columns as UTC.
- **Telemetry renderers**: `timezone: tzinfo = UTC` defaults in `render/axis.py`, `bars.py`, `tiles.py`, `sparkline.py`,
  so any caller that forgets to pass a tz silently labels UTC.

Two distinct bug classes are in play and both need fixing:

1. **UTC display** — a correctly stored aware-UTC instant is formatted without converting to the configured tz.
2. **System-clock display** — a value is formatted through the host clock, which only coincidentally matches the
   configured tz.

**Rust core boundary.** Timestamp _presentation_ is Python and stays here. One genuine core-side issue exists:
`crates/sase_core/src/artifact_file.rs::parse_artifact_date` derives an artifact's calendar date by converting to UTC
(`.with_timezone(&Utc).naive_utc()`), and `crates/sase_core/src/artifact_file/retention.rs::parse_now` does the same.
Once the Files pane displays local days, `date:`/`since:` filtering and retention age would disagree with the visible
day at the boundary. That is fixed in the `artifact-dates` phase. All other `chrono` usage in the core is canonical-UTC
storage, comparison, or sorting and stays untouched.

## Goals

1. Every timestamp rendered for a human resolves through the configured timezone.
2. One shared way to do it: `sase.core.time` gains `parse_local`/`format_local`, and display sites call them instead of
   hand-rolling `fromisoformat` + `astimezone` + `strftime` chains.
3. Columns and labels that previously read `UTC` keep an honest zone label (`%Z`) rather than silently changing meaning.
4. Artifact-file calendar-date filtering and retention agree with the displayed day.
5. A guard test fails the build when a new system-clock or unconverted-UTC display site is added under `src/`.

## Non-goals

- Changing any **stored** format. Agent meta, task rows, notification rows, skill/memory/repo logs, and artifact index
  rows keep their canonical shapes; only `created_at` and the retention `now` gain a configured-tz offset (still
  RFC3339, still an unambiguous instant) in the `artifact-dates` phase.
- Converting models to aware datetimes. The Agents/Tasks model convention stays **naive configured-tz wall time**; the
  new helpers return aware values for _formatting only_.
- Re-litigating the arithmetic sites already fixed by `timezone_runtime_consistency`.
- DST-transition-hour exactness for naive wall-clock deltas (pre-existing, still accepted).
- Timestamps embedded in filenames, directory names, agent name suffixes, and shard paths — already configured-tz via
  `generate_timestamp()` / `local_now()`.

---

## Phase `helpers` — Shared display helpers in `sase.core.time`

Add to `src/sase/core/time.py`, alongside the existing `local_now`/`to_local`/`get_timezone`:

```python
def parse_local(value: str | int | float | datetime | None) -> datetime | None:
    """Return *value* as an aware configured-tz datetime, or None when unparseable."""

def format_local(
    value: str | int | float | datetime | None,
    fmt: str = "%Y-%m-%d %H:%M:%S",
    *,
    default: str = "—",
) -> str:
    """Format *value* in the configured timezone, or *default* when unparseable."""
```

`parse_local` normalization rules:

- `None`, empty string, or whitespace → `None`.
- `int` / `float` → Unix epoch seconds → `datetime.fromtimestamp(value, get_timezone())`.
- `datetime` → aware: `.astimezone(get_timezone())`; naive: `.replace(tzinfo=get_timezone())` (naive is configured-tz
  wall time by repo convention).
- `str` → `datetime.fromisoformat(value.replace("Z", "+00:00"))`; a parsed naive result is treated as configured-tz wall
  time. `ValueError` → `None`.
- Guard `OSError`/`OverflowError`/`ValueError` around the epoch branch (out-of-range timestamps already crash some
  current sites).

`format_local` is `parse_local` + `strftime`, returning `default` on `None`.

Both are exported from `sase.core.time.__all__` (add one if the module has none) and documented in the module docstring
next to the existing "never use the bare system clock" paragraph, stating that `parse_local`/`format_local` are the
required entry points for **display**, while `to_local`/`local_now` remain the entry points for the naive-model
**arithmetic** convention.

Tests (`tests/test_timezone_display_consistency.py`, new file, all under the existing `tz_divergence` fixture):

- Aware-UTC ISO with `Z`, with `+00:00`, and with a non-UTC offset each convert to the configured tz.
- Naive ISO is interpreted as configured-tz wall time and comes back unchanged in wall-clock terms.
- Epoch int and float land on the configured-tz wall clock.
- `datetime` inputs, aware and naive, follow the same rules.
- Garbage strings, `None`, and empty strings return `None` / `default`.
- `format_local` honors a custom `fmt` and a custom `default`.

## Phase `artifacts` — Artifacts tab and artifact CLI

All paths below are under `src/sase/`.

1. `ace/tui/widgets/artifacts/files_rendering.py::_artifact_file_datetime` — this is the headline bug. Drop the
   `local_timezone.tzinfo is not None` guard entirely and return `parse_local(row.created_at)`. The `local_timezone`
   parameter becomes unnecessary; remove it and update `file_group_label` and `file_row_text` accordingly. `today` /
   `now` stay as the caller-supplied reference for Today/Yesterday bucketing, but compare `today.date()` against the
   converted timestamp's `.date()`.
2. `ace/tui/widgets/artifacts/files_detail.py::_minute_precision` (called for the `Created` field) — replace the
   `created_at[:16].replace("T", " ")` slice with `format_local(created_at, "%Y-%m-%d %H:%M")`.
3. `ace/tui/widgets/artifacts/beads_detail.py` — the `Created`, `Updated`, and `Closed` property rows (~lines 78–80) and
   the Markdown export line (~line 184) print the bead store's UTC ISO verbatim. Route all four through `format_local`,
   keeping `""` / `"—"` for the empty cases exactly as today.
4. `ace/tui/widgets/artifacts/plans_rendering.py::_compact_plan_date` — plan `create_time` is already configured-tz, but
   the value falls back to `owner.bead_created_at` (UTC). Parse with `parse_local` and format `%m-%d`, keeping the
   existing "return the raw prefix when unparseable" behavior.
5. `ace/tui/widgets/_artifact_ref_completion_menu.py::age_label` — `datetime.now(UTC).timestamp()` is a correct absolute
   reference and stays; the >7d fallback at the end of the function
   (`datetime.fromtimestamp(timestamp, UTC).date().isoformat()`) must use the configured tz. Use
   `parse_local(timestamp).date().isoformat()`. The unparseable `raw[:10]` fallback stays.
6. `artifact_cli/listing.py` — the CREATED column slices the stored ISO
   (`created_at[:_CREATED_COLUMN_WIDTH].replace("T", " ")`). Replace with `format_local(created_at, "%Y-%m-%d %H:%M")`
   and keep the 16-wide column.

Already correct, do not change: `chats_rendering.py` (the chat catalog writes configured-tz ISO via
`history/chat_catalog.py`), `chat/cli_list.py` (same source), `bugs_rendering.py` and `beads_rendering.py` (already on
`local_now()`/`to_local()`), and `files_filtering.py::_entry_epoch` (absolute-instant comparison, tz-independent).

Tests: extend `tests/test_timezone_display_consistency.py` under `tz_divergence` with a file row whose `created_at` is
an aware-UTC instant late in the configured-tz day, asserting the rendered `HH:MM` and the `Today` group label match the
configured tz rather than UTC; plus one assertion per site above.

## Phase `tui-panels` — ACE modals, tools panel, and file panel

All paths below are under `src/sase/`.

**Explicitly UTC-labeled displays** (convert to configured tz and swap the literal `UTC` for `%Z`, so the label stays
truthful):

- `ace/tui/modals/logs_pane_render.py::format_mtime` — `%Y-%m-%d %H:%M UTC` from a `tz=UTC` epoch conversion. Also
  update the docstring.
- `ace/tui/modals/logs_pane_toasts.py::_format_session_started_at` — same format string, applied to an aware value
  without conversion. `_toast_timestamp` (`%m-%d %H:%M:%S` / `%H:%M:%S`) has the same defect and must be converted too;
  `_timestamp_key` sorts on the absolute instant and stays UTC.

**System-clock displays** (route through `format_local` / `parse_local`):

- `ace/tui/modals/statistics_pane_rendering.py` — the `updated {HH:MM:SS}` status line.
- `ace/tui/modals/statistics_help_modal.py` — the `Last loaded` line; keep the `%Z` in the format string, which now
  renders the configured zone abbreviation.
- `ace/tui/modals/statistics_pane_projects.py::_format_timestamp` — `%b %d %H:%M`.
- `ace/tui/modals/project_inventory_rendering.py::_absolute_workspace_time` — `%Y-%m-%d %H:%M`; keep the
  `OSError/OverflowError/ValueError` fallback to `"-"`.
- `ace/tui/widgets/prompt_panel/_member_roster.py::_format_timestamp` — `value.astimezone().strftime(...)`.
- `ace/tui/modals/saved_agent_group_revival_rendering.py::_saved_group_time_label` — the relative half is correct (aware
  UTC on both sides); the absolute `%Y-%m-%d %H:%M` half must be converted.

**Tasks pane** — this pane currently mixes domains, so fix it as a unit:

- `ace/tui/modals/tasks_store_rows.py::_local_datetime` converts durable UTC rows with argument-less `.astimezone()`;
  replace with `to_local()`. Its `datetime.now()` fallback at the `_store_task_row` call site becomes `local_now()`.
- `ace/tui/modals/tasks_pane_render.py::_relative_time` and `::_elapsed` default their reference to `datetime.now()`;
  both must default to `local_now()` so they match the naive configured-tz `TaskInfo` values. Explicit `now=` parameters
  keep working unchanged.
- `ace/tui/task_queue.py` mints in-memory `TaskInfo` values with bare `datetime.now()` (log line `ts`, `started_at`,
  `finished_at`, and the `cutoff` reference). Switch to `local_now()` so in-memory rows and durable rows share one
  domain in the same pane.
- `ace/tui/task_mirror.py` writes `finished_at=_utc_timestamp(datetime.now())`, and `_utc_timestamp` does
  `value.astimezone(UTC)` — a naive system-clock reading is reinterpreted as system-local, which is the _wrong instant_
  when configured ≠ system. Pass `local_now()` (whose naive value `_utc_timestamp` already documents as configured-tz
  local); confirm `_utc_timestamp`'s naive branch attaches `get_timezone()` before converting.

**Tools panel / file panel fetch clocks** — these render `fetch_time.strftime("%H:%M:%S")`, so the stored value must be
configured-tz local:

- `ace/tui/tools/cache.py` — the two `fetch_time=datetime.now()` mints → `local_now()`; and
  `cached_tool_calls_end_reference` returns a `tz=UTC` conversion of the artifact mtime that is compared against
  tool-call timestamps and used for timeline extents — normalize it with `to_local()` so it matches the panel's domain.
- `ace/tui/widgets/tools_panel.py` — the three `latest_cached_fetch_time(agent) or datetime.now()` fallbacks →
  `local_now()`.
- `ace/tui/widgets/file_panel/_fetch.py`, `_linked_deltas.py`, `_display.py` — `fetch_time` / `fetched_at` mints →
  `local_now()`. The same-process age arithmetic in `file_panel/_panel.py` (`datetime.now() - cache_entry.fetch_time`)
  must move to `local_now()` in the same change so both sides stay in one domain.
- `ace/tui/tools/report.py::_timestamp_hhmmss` formats a parsed raw timestamp without converting; use
  `format_local(value, "%H%M%S", default="unknown")`. (`ace/tui/tools/_report_render.py::_format_local_timestamp` is
  already correct — leave it, or simplify it onto `format_local` without changing behavior.)

Leave `ace/tui/modals/quit_confirm_modal.py` and `ace/tui/actions/navigation/_fold.py` alone unless the value is
displayed; both are same-process elapsed arithmetic already covered by the prior tale's non-goals. If the phase agent
finds either value reaching a rendered string, fix it the same way and say so.

Tests: extend `tests/test_timezone_display_consistency.py` under `tz_divergence` with one assertion per fixed site,
asserting the configured-tz wall clock (and, for the previously-`UTC`-labeled sites, that the rendered zone label is no
longer the literal `UTC`).

## Phase `cli-pages` — CLI tables, generated Markdown pages, and telemetry defaults

All paths below are under `src/sase/`.

**CLI tables printing stored UTC ISO verbatim** — render with `format_local(..., "%Y-%m-%d %H:%M:%S")`, keeping each
table's existing empty-value placeholder:

- `main/task_render.py` — the `Created` / `Started` / `Finished` detail rows. `_task_duration_seconds`'s
  `datetime.now(UTC)` is correct absolute arithmetic and stays.
- `repo_open_cli_log.py` — the Timestamp column in the events table and the Timestamp detail row.
- `memory/cli_log.py` — the Timestamp columns in both tables and the Timestamp detail row.
- `skills/cli_log.py` — the Timestamp column and the Timestamp detail row.

In every case the sort keys (`key=lambda event: (event.timestamp, ...)`) keep sorting on the raw stored string; only the
rendered cell changes. `--json` output keeps the raw stored value — these commands' JSON is a machine contract.

**Generated Markdown** — these pages are committed to sidecar repos and read on GitHub, so the zone must be labeled:

- `bead_pages/rendering_tables.py` — the commit table's `Committed (UTC)` header and its
  `datetime.fromtimestamp(row.committed_at, tz=UTC)` cell.
- `agents_sync/rendering_commits.py` — the same header/cell pair in both the with-role and without-role table shapes.
- `bead_pages/rendering_identity.py::_render_instant` — `%Y-%m-%d %H:%M:%S UTC`.

For all three: convert with `parse_local` and render `%Y-%m-%d %H:%M:%S %Z`, changing the column header from
`Committed (UTC)` to `Committed`. Keep the naive-input branch in `_render_instant` (assume configured tz, matching the
repo convention) and its `md_escape` fallback for unparseable values. Update any golden/snapshot fixtures these
renderers have.

**Other display sites:**

- `memory/review_tui/_render.py::format_time_or_age` — the `>48h` branch renders `%Y-%m-%d` in UTC; convert. The
  relative branch compares two aware values and is correct.
- `notification_gates/debug.py::_iso_from_unix` and `notification_gates/debug_rendering.py::_iso_from_unix` — both do
  `datetime.fromtimestamp(value).astimezone().isoformat()` (system clock). Use `parse_local(value).isoformat()`, keeping
  the `str(value)` fallback. These two are byte-identical; if the phase agent can share one implementation without
  breaking the modules' import graph, do so.
- `telemetry/render/axis.py`, `bars.py`, `tiles.py`, `sparkline.py` — the `timezone: tzinfo = UTC` keyword defaults
  (including `format_recording_started`) silently label UTC for any caller that omits the argument. Change the default
  to `None` and resolve `timezone or get_timezone()` inside each function. Callers that pass an explicit tz are
  unaffected; `stats/views.py` already resolves `timezone or get_timezone()` and shows the intended pattern.

Tests: extend `tests/test_timezone_display_consistency.py` under `tz_divergence` with one assertion per site, and assert
that the telemetry renderers with no explicit `timezone=` produce the configured-tz label.

## Phase `artifact-dates` — Artifact-file calendar dates in the configured timezone

Once the Files pane shows local days, the day used for `date:` / `since:` filtering and for retention age must match, or
a file created at 21:30 EDT will display under one day and filter under the next.

**Rust core** (`sase-core`; the phase agent MUST open it through the `/sase_repo` skill and work only in the path that
skill prints):

- `crates/sase_core/src/artifact_file.rs` — split the single `parse_artifact_datetime` helper into two, because the two
  callers want different things:
  - the **instant** path (`parse_artifact_time`, the recency sort key in `row_recency_key`) keeps converting to UTC —
    absolute ordering must not depend on the offset;
  - the **calendar-date** path (`parse_artifact_date`, used by the `since` filter) uses the offset embedded in the
    RFC3339 string (`DateTime::parse_from_rfc3339(..).naive_local().date()`) instead of `.naive_utc()`. The existing
    naive-`%Y-%m-%dT%H:%M:%S%.f` and date-only `%Y-%m-%d` fallbacks are unchanged.
- `crates/sase_core/src/artifact_file/retention.rs::parse_now` — same change: `.naive_local().date()`.
- Add Rust unit tests covering an RFC3339 value with a negative offset whose UTC date is the following day, asserting
  the calendar date is the local one while the recency sort key is still the absolute instant.
- Bump/adjust wire schema constants only if the existing parity tests require it; no wire _shape_ change is intended.

**Python producers:**

- `core/artifact_file_helpers.py::file_created_at` — `datetime.fromtimestamp(stat.st_mtime, get_timezone()).isoformat()`
  so new index rows carry the configured-tz offset. `now_iso()` (an index-write "now") stays UTC.
- `artifact_cli/prune.py` and `artifact_cli/stats.py` — the `now=datetime.now(UTC).isoformat()` retention references
  become `datetime.now(get_timezone()).isoformat()` so the day passed to `parse_now` is the configured-tz day. Both
  remain unambiguous RFC3339 instants.

**Compatibility.** Existing index rows keep their `+00:00` offset and continue to bucket by the UTC day until the row is
re-indexed from disk mtime; this is self-healing and bounded, and every _display_ path already goes through
`parse_local`, which handles both offsets identically. Call this out in the phase's commit message. Verify against
`sase artifact doctor` whether a re-index is triggered by mtime change alone; if `created_at` is only written on first
index, note in the phase's `PROPOSED FOLLOW-UP:` whether a one-shot backfill is worth a separate task.

**Python tests:** an artifact row created near the local midnight boundary is included by a `since:` filter naming the
local day and excluded by the UTC day, under `tz_divergence`.

## Phase `guard-docs` — Repo-wide guard test and documentation

**Guard test** — add `test_no_system_clock_display_sites` to `tests/test_timezone_display_consistency.py`. Walk every
`.py` file under `src/sase/` with `ast`, and fail on:

- `datetime.now()` with no arguments;
- `<expr>.astimezone()` with no arguments;
- `datetime.fromtimestamp(<one arg>)` with no tz.

Report each violation as `path:line` with the offending source line. Maintain a module-level `_ALLOWED` frozenset of
`"relative/path.py:<lineno-independent symbol name>"` entries — key on the enclosing function/class name, not the line
number, so the allowlist does not rot on unrelated edits. Seed it with the genuine exceptions found while implementing
the earlier phases (`core/time.py::_system_timezone` is one; `scripts/sase_git_commit` is not a `.py` file and is out of
the walk). Every allowlist entry carries a one-line comment saying why it is not a display site. The test's docstring
must tell a future agent to fix the call rather than extend the allowlist by default.

If the phase agent finds the AST guard produces an unmanageably long seed allowlist, prefer narrowing the walk (for
example: skip `src/sase/**/testing.py` and `src/sase/fakey/`) over weakening the rule, and record what was skipped.

**Docs:**

- `docs/configuration.md`, the `timezone` entry: add a sentence that the setting governs generated _and_ displayed
  timestamps across the TUI, the CLI, and generated Markdown pages, and that previously UTC-labeled columns now render
  the configured zone abbreviation.
- `docs/development.md`: a short "Timestamp display convention" subsection — `parse_local`/`format_local` for display,
  `local_now`/`to_local` for the naive-model arithmetic convention, canonical UTC for storage and wire — pointing at the
  guard test and the `tz_divergence` fixture.
- `docs/ace.md`: update any pane description that documents a `UTC` suffix in the Logs pane or Files pane.

**Verification:** run `just install` then `just check`, and `just test-visual` (the Logs/Statistics/Files panes are
covered by PNG snapshots; accept intentional changes only with `--sase-update-visual-snapshots` and say in the commit
message which snapshots moved and why).

---

## Testing strategy

All new tests live in `tests/test_timezone_display_consistency.py` and use the existing `tz_divergence` fixture from
`tests/conftest.py`, which pins configured tz `America/New_York` against system tz `UTC`. That fixture is what makes
both bug classes visible: a UTC-display bug shows a +4h wall clock, and a system-clock bug shows the same +4h because
the system tz is UTC. To distinguish them where it matters, a few tests should additionally assert behavior with a third
zone, so a fix that merely swaps one hardcoded zone for another still fails.

Every phase adds assertions that **fail against pre-fix code**; the `guard-docs` phase adds the structural guard so the
class cannot silently return.

`tests/test_timezone_runtime_consistency.py` stays green untouched — it pins the arithmetic contract this plan builds
on.

## Risks and edge cases

- **Snapshot churn.** PNG visual snapshots and any Markdown golden fixtures covering the Logs pane, Statistics pane,
  Files pane, bead pages, and agents-sync bundles will move. Expected; accept deliberately and note it.
- **`%Z` on the fixed-offset fallback.** When `get_timezone()` returns the fixed-offset last-resort `tzinfo`, `%Z`
  renders an offset like `+00:00` rather than an abbreviation. Acceptable and still honest; do not special-case it.
- **Machine-readable output.** `--json` payloads and sort keys keep the raw stored values. Only rendered cells change.
  Any test asserting on JSON timestamps should be unaffected; if one is not, that is a signal the change leaked into a
  contract and must be reverted at that site.
- **Artifact index migration.** Covered above: mixed-offset rows coexist safely because display always normalizes and
  sorting always uses the absolute instant.
- **Cross-host artifact sharing.** Unchanged from the prior tale: a `~/.sase` state dir is per-host, and naive values
  are interpreted in that host's configured tz.
- **Phase overlap.** `artifacts` and `artifact-dates` both touch artifact `created_at`, from opposite ends (render vs.
  mint). They are independent because `parse_local` handles either offset; the phases share no files. If both land, the
  combined behavior is the intended one.

## Acceptance criteria

1. On a host with system tz `UTC` and `timezone: "America/New_York"`, an artifact file written at 21:30 local shows
   `21:30` in Artifacts → Files, groups under today, and appears in a `since:` filter naming today's local date.
2. The Logs pane, Statistics pane and help modal, project inventory, tasks pane, saved-group list, member roster, tools
   panel, and file panel all show the configured-tz wall clock; no user-visible surface carries a literal `UTC` suffix
   that isn't actually UTC.
3. `sase artifact list`, `sase task show`, `sase repo log`, `sase memory log`, and `sase skills log` render timestamps
   on the configured clock; their `--json` output is byte-identical to before.
4. Generated bead pages and agents-sync bundles render commit and instant columns on the configured clock with a zone
   abbreviation, and the `(UTC)` column headers are gone.
5. Telemetry renderers invoked without an explicit `timezone=` label the configured zone.
6. The guard test fails when a bare `datetime.now()`, an argument-less `.astimezone()`, or a tz-less
   `datetime.fromtimestamp()` is introduced under `src/sase/` outside the justified allowlist.
7. `just check` passes and `just test-visual` passes with only deliberately accepted snapshot updates.
