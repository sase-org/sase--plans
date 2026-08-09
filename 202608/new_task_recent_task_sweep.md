---
tier: tale
title:
  Sweep recently created task beads in /sase_new_task, backed by a creation-date filter
  on sase bead list
goal:
  /sase_new_task reviews every task bead created in the last week before concluding a
  report is new, and records RELATED notes for near-miss beads, using a new
  creation-date filter on `sase bead list`.
proposed_by: bbugyi200.athena.wu
create_time: 2026-08-09 17:01:38
status: wip
---

# Plan: A last-week task sweep for `/sase_new_task`

## Problem

`/sase_new_task` currently detects duplicates with one command
(`src/sase/xprompts/skills/sase_new_task.md`, step 3):

```bash
sase bead search 'symbol|filename|command|error-fragment' --regex --type task
```

Search recall depends entirely on the reporter guessing a term that literally appears in
the older bead. A task filed hours earlier by a different agent, describing the same
defect in different words, is invisible to that query — which is exactly the case the
skill exists to prevent, because near-simultaneous reports of one defect are the most
likely duplicates.

Measured against the live store on 2026-08-09:

| Population                   | Count                                                       |
| ---------------------------- | ----------------------------------------------------------- |
| Task beads, all statuses     | 186                                                         |
| Created in the last 7 days   | 100 (84 closed, 12 ready, 2 snoozed, 1 in_progress, 1 open) |
| Created in the last 3 days   | 48                                                          |
| Created in the last 24 hours | 21                                                          |

Two conclusions follow. First, a recency sweep is high-yield: more than half of every
task bead ever filed is a week old or newer. Second, it is affordable in compact form —
the full 186-row compact listing is ~2,800 words, so a one-week window is ~1,500 words
(the same population in `--format full` would be tens of thousands of words, which is
the cost the previous plan removed and must not be reintroduced).

Three gaps block that sweep today:

1. `sase bead list` cannot filter by creation date. Its options are `--color`,
   `--format`, `--limit`, `--status`, `--tier`, `--type`
   (`src/sase/main/parser_bead_queries.py:77`), and the Rust `list_issues` it calls
   filters on status, type, and tier only.
2. 84% of last week's task beads are `closed`, which the default status set excludes, so
   a sweep that does not opt into closed beads sees 16 of 100.
3. The skill never says what to do with a bead that is related but is not a duplicate,
   so that context is discovered and then thrown away.

## Design decisions

### D1 — the new bounds filter creation time and reuse the existing DATE grammar

Add `-S, --since DATE` and `-u, --until DATE` to `sase bead list` only. Parse both with
`sase.vcs_log.dates.parse_time_bound`, the grammar `sase vcs log -s/-u` and the ACE bead
filter bar's `since:`/`until:` tokens already accept: `Nh`/`Nd`/`Nw`, `today`,
`yesterday`, `YYYY-MM-DD`, `YYYY-MM-DDTHH:MM`. `--since 1w` then means one thing
everywhere in SASE.

- `-s` is already `--status`, so `--since` takes `-S`; `sase vcs log` ships capital
  short flags (`-R`, `-S`) already, so this is in-house style, and the CLI rule that
  every public long option gets a short alias is satisfied.
- The bound applies to `created_at`, **not** `updated_at`. Compact list rows are ordered
  by creation and already print the `⧖ <age>` creation cell, and the skill's question is
  literally "what was created in the last week".
- Known divergence to document, not to silently inherit: ACE's `since:`/`until:` filter
  tokens bound _last activity_ (`updated_at or created_at`,
  `src/sase/ace/tui/widgets/artifacts/beads_filtering.py:239`). The CLI help and
  `docs/beads.md` must both say "created", so the two surfaces are never confused.
- `sase bead search` is deliberately left alone: it is query-scoped by construction, and
  the sweep is a listing operation.

### D2 — `--status all`

Add `all` to `--status`'s choices on `sase bead list`; it expands to every status and
wins over any other `--status` value in the same invocation. Without it the skill would
have to spell six repeated `--status` flags into a copy-pasted command, and with the
default status set the sweep would miss the 84% of the window that is already closed.
Recently closed duplicates matter most: they tell the reporter the defect is already
fixed.

### D3 — an explicit date bound lifts the newest-20 closed default

`handle_bead_list` truncates to `DEFAULT_CLOSED_LIST_LIMIT = 20` whenever closed beads
are in scope and `--limit` was omitted, and prints no notice when it does. A silently
truncated sweep is precisely the failure mode this skill exists to prevent: the agent
sees 20 of 100 rows and concludes "no duplicate". A date bound already bounds the
output, so when `--since` or `--until` is present the 20-row default does not apply. An
explicit `--limit` still wins, so nothing that passes `--limit` changes behavior.

### D4 — the filter lands in the Python CLI layer, not in `../sase-core`

This is the one decision worth challenging, since `CLAUDE.md` routes shared backend
behavior to the Rust core. The evidence says the CLI layer is correct here:

- `sase bead list` is explicitly excluded from the Rust bead fast path
  (`src/sase/main/bead_fast_path.py:37`, pinned by
  `tests/main/test_bead_fast_path.py:309`), so `handle_bead_list` is the only
  implementation of this command. The Rust `handle_list` that exists in
  `crates/sase_core/src/bead/cli.rs` is never reached by `sase bead list`, and it defers
  on flags it does not recognize rather than erroring.
- `created_at` is already on every wire row returned by `bead_list`. The change is a
  predicate over data the store already handed back — no new store read, no new
  persistence or domain rule.
- The date grammar itself is Python (`src/sase/vcs_log/dates.py`) and is already the
  shared vocabulary for `sase vcs log` and for ACE's bead filter bar, which likewise
  filters in Python over a snapshot.
- The mobile/web frontend's bead list is served through `HelperHostBridge::list_beads`
  (a host callback used by `crates/sase_gateway/src/routes.rs:522`), not through
  `sase_core::bead::read::list_issues`, so a Rust-side filter would not be what that
  surface consumes today.
- Cost of the alternative: a new `bead_list` binding parameter cannot be relied on until
  a `sase-core-rs` release ships and the `pyproject.toml` window
  (`sase-core-rs>=0.21.3,<0.22.0`) is ratcheted. Open bead `sase-hz` records that exact
  trap on a previous binding change.

If a second surface ever needs creation-date filtering, promote the predicate into
`list_issues_in_issues` and the `bead_list` binding at that point. **The implementing
agent must not modify `../sase-core`.**

### D5 — skill shape

The sweep becomes its own numbered step between the search step and the epic step, so
the reading order stays: query for known terms → sweep what is new → check active epics
→ create. Related-but-not-duplicate beads are recorded on the newly created task with a
`RELATED:` note, matching the existing `DISCOVERED ISSUE:` (epics) and
`PROPOSED FOLLOW-UP:` (phase beads) prefix family.

The existing regression guard `assert "sase bead list --type task" not in flat`
(`tests/main/test_init_skills_sources.py:454`) will now be false by construction. It
must be **narrowed, not deleted** — the intent it protects (never dump the whole task
corpus) still holds, so the replacement must allow only a date-bounded task listing.

## Scope

In scope:

1. `src/sase/main/parser_bead_common.py` — DATE argparse type.
2. `src/sase/main/parser_bead_queries.py` — `--since`, `--until`, `--status all`, help.
3. `src/sase/bead/cli_query.py` — bound resolution, filtering, limit interaction.
4. `src/sase/bead/cli_admin.py` — one `sase bead onboard` example line.
5. `src/sase/xprompts/skills/sase_new_task.md` — the skill itself.
6. `tests/test_bead/test_cli_list.py` — parser and handler coverage.
7. `tests/main/test_init_skills_sources.py` — skill phrases and the narrowed guard.
8. `docs/beads.md`, `docs/cli.md` — flag table and command index.

Out of scope, and confirmed to need no edit:

- **Any SASE memory file** (`sase/memory/*.md`, `AGENTS.md`, `CLAUDE.md`, and the
  templates under `src/sase/main/init_memory/templates/`). No memory-edit permission was
  granted for this work. Their claims — "checks every task status for semantic
  duplicates", "gathers evidence and checks all task statuses" — stay true after this
  change.
- `../sase-core` (see D4) and `sase bead search`.
- ACE filter-bar tokens and `beads_filtering.py`.
- Chezmoi deployment. Preview with `sase skill init --diff` (read-only); do **not** run
  `sase skill init --force` or `chezmoi apply`. Deployment happens from a clean, landed
  tree per `sase/memory/generated_skills.md`.

## Step 1 — a DATE argparse type

In `src/sase/main/parser_bead_common.py`, add a module-level import of
`from sase.vcs_log.dates import VcsLogDateError, parse_time_bound` (that module imports
only stdlib at import time; `get_timezone` is resolved lazily inside its functions) and
a validator above `nonnegative_int`:

```python
def bead_date_arg(value: str) -> str:
    """Validate a ``--since``/``--until`` DATE token for bead listings."""
    try:
        parse_time_bound(value)
    except VcsLogDateError as exc:
        raise argparse.ArgumentTypeError(str(exc)) from None
    return value
```

Return the raw string and resolve it in the handler; resolution needs the operation
clock, which belongs at handling time, and this mirrors how `sase plan search` validates
its `--since`/`--until` tokens syntactically at parse time.

## Step 2 — register the options

In `src/sase/main/parser_bead_queries.py`, `register_bead_list_parser`:

- Add to the `description`: `--since`/`--until` bound bead **creation** time, and when
  either is given the newest-20 closed default no longer applies.
- Extend the `epilog` with one example and the grammar line, matching how
  `register_vcs_parser` documents dates:
  `  sase bead list --type task --since 1w --status all` plus
  `f"DATE grammar: {DATE_HELP}."` (import `DATE_HELP` from `sase.vcs_log.dates`).
- Add `all` as the first value of `--status`'s `choices`, and update its help to
  `"Filter by status; 'all' selects every status (repeatable)"`.
- Add the two new options, keeping the file's alphabetical-by-long-name ordering, so
  `--since` sits between `--limit` and `--status`, and `--until` comes after `--type`:

```python
    parser.add_argument(
        "-S",
        "--since",
        metavar="DATE",
        type=bead_date_arg,
        help="Only beads created at/after DATE",
    )
    ...
    parser.add_argument(
        "-u",
        "--until",
        metavar="DATE",
        type=bead_date_arg,
        help="Only beads created at/before DATE",
    )
```

Leave `register_bead_search_parser` untouched.

## Step 3 — filter in the list handler

In `src/sase/bead/cli_query.py`, `handle_bead_list`:

1. Expand statuses. `explicit_statuses` stays `args.status is not None`; when the
   requested values contain `all`, use every `Status` member in the existing
   open/claimed/ready/snoozed/in_progress/closed display order instead of mapping the
   literal values.
2. Resolve the window once, before reading:

```python
def _resolve_created_window(args: argparse.Namespace) -> tuple[int | None, int | None]:
    since_text = getattr(args, "since", None)
    until_text = getattr(args, "until", None)
    if since_text is None and until_text is None:
        return (None, None)
    from sase.core import time as core_time
    from sase.vcs_log.dates import (
        VcsLogDateError,
        normalize_reference_time,
        parse_time_bound,
    )

    reference = normalize_reference_time(core_time.local_now())
    try:
        since = (
            parse_time_bound(since_text).resolve(now=reference, boundary="since")
            if since_text
            else None
        )
        until = (
            parse_time_bound(until_text).resolve(now=reference, boundary="until")
            if until_text
            else None
        )
    except VcsLogDateError as exc:
        print(f"Error: {exc}", file=sys.stderr)
        sys.exit(2)
    if since is not None and until is not None and since > until:
        print("Error: --since must not be later than --until", file=sys.stderr)
        sys.exit(2)
    return (since, until)
```

Resolving "now" through `core_time.local_now()` (naive configured-tz, normalized to an
aware reference) keeps the handler on the repo's patchable test clock. The
`VcsLogDateError` branch is defensive: argparse already rejects bad tokens, but tests
and future callers build `Namespace` objects directly, exactly as
`plan_search_handler._validate_args` re-checks its own dates.

3. Apply the window to each read with a helper that parses `created_at` through
   `sase.core.time.parse_local` (it accepts the stored `2026-08-09T15:54:14Z` shape) and
   compares epoch seconds. A bead whose `created_at` is empty or unparseable is
   **excluded** whenever a bound is present — it cannot be shown to fall inside the
   window — and that is stated in the docs. Do not import the TUI's private
   `_timestamp_epoch`.

4. Apply it in both places the handler reads, so the existing "no active beads → fall
   back to closed" behavior keeps working inside the window: the initial
   `view.list_issues(...)` result and the implicit-closed re-query. `total` is computed
   after filtering, so the JSON envelope's `total` keeps meaning "matched all filters"
   and `count` keeps meaning "printed".

5. Interact with the limit default:

```python
        if limit is None and closed_in_scope and window == (None, None):
            limit = DEFAULT_CLOSED_LIST_LIMIT
```

## Step 4 — one onboard example

In `src/sase/bead/cli_admin.py`, `handle_bead_onboard`, add a line to the quick-start
block after the existing `sase bead list --tier=epic` row, keeping the description
column aligned with its neighbors:

```
  sase bead list --type=task --since=1w --status=all
                                                 Task beads created in the last week
```

Wrap it the way the block's other two-line entries are already wrapped if it does not
fit one line.

## Step 5 — rewrite the skill

In `src/sase/xprompts/skills/sase_new_task.md`:

1. Keep the frontmatter, step 1, step 2, and step 3 exactly as they are.
2. Insert a new step after step 3 (renumbering the epic step to 5 and the create step to
   6):

````markdown
4. Sweep every task bead created in the last week, then show plausible matches:

   ```bash
   sase bead list --type task --since 1w --status all
   sase bead show <plausible-task-id>
   ```

   A duplicate filed hours ago by another agent often shares no term with your queries,
   so this sweep is not redundant with step 3. `--since` bounds creation time and lifts
   the newest-20 closed default, and `--status all` matters because most recent task
   beads are already closed. Keep the sweep in the default compact format; never run it
   with `--format full`. Judge each row by the semantic-duplicate test above and
   corroborate with `sase bead +1` instead of creating a task when one matches.
````

3. In the create step, after the
   `sase bead create ... / sase bead dep add ... / sase bead update ... --status ready`
   block, add the related-bead instruction:

````markdown
When the search or the sweep surfaced beads that are related but are not duplicates — an
adjacent defect, a shared root file, a bead whose fix could collide — record one note
per bead on the new task so its worker inherits that context:

```bash
sase bead note <task-id> "RELATED: <bead-id> — <how it bears on this task>"
```
````

Authoring constraints: prose wraps at 88 columns (prettier `printWidth: 88`,
`proseWrap: always`, enforced by `just fmt-md-check`); change nothing else in steps 1,
2, 3, 5, or 6; leave step 5's
`sase bead list --type plan --tier epic --status in_progress` invocation byte-for-byte
intact.

## Step 6 — tests

**`tests/test_bead/test_cli_list.py`** — follow the file's existing style
(`create_parser().parse_args([...])` for parsing, `BeadProject(project_dir)` plus
`bead_cli.handle_bead_list(args)` and `capsys` for behavior):

- Parser: `--since`/`-S` and `--until`/`-u` populate `args.since`/`args.until`; a
  malformed DATE (`"lastweek"`) exits 2; `--status all` parses.
- `--since 1d` keeps beads created now; `--until 1d` excludes them. Both directions
  matter, and both are deterministic against real time without patching a clock.
- `--status all` includes a closed bead that the default status set omits, and the JSON
  envelope's `statuses` lists every status.
- `total` in the JSON envelope counts only beads inside the window.
- The newest-20 closed default is lifted by a date bound: monkeypatch
  `bead_cli.DEFAULT_CLOSED_LIST_LIMIT` to `1`, create two closed beads, and assert
  `--status closed` prints one row while `--status closed --since 1d` prints two. This
  keeps the test cheap instead of creating 21 beads.
- `--since` later than `--until` exits 2.

**`tests/main/test_init_skills_sources.py`**:

- In the `sase_new_task` phrase tuple (line 228), add
  `"sase bead list --type task --since 1w --status all"`, `"created in the last week"`,
  and `"RELATED:"`. Leave every existing phrase, including
  `"sase bead list --type plan --tier epic --status in_progress"`.
- In `test_sase_new_task_duplicate_detection_stays_query_scoped` (line 443), replace
  `assert "sase bead list --type task" not in flat` with a guard that still forbids an
  unbounded task dump while allowing the bounded sweep — assert that
  `re.search(r"sase bead list --type task(?! --since)", flat)` finds nothing, and that
  `"--format full"` never appears on a task listing. Keep the `sase bead search ...` and
  epic-list assertions, and update the docstring to state both invariants: duplicate
  detection stays query-scoped plus a date-bounded sweep, and the epic check stays a
  list.

## Step 7 — docs

- `docs/beads.md`, `### sase bead list`: add `-S, --since` and `-u, --until` rows to the
  flag table (values `DATE`, described as bead **creation** time), add `all` to the
  `-s, --status` values cell, and extend the paragraph under the table with the two new
  rules — a date bound replaces the newest-20 closed default, and a bead with no usable
  creation timestamp is omitted when a bound is present. Note the accepted DATE forms
  and that ACE's `since:`/`until:` filter tokens bound last activity while these CLI
  flags bound creation.
- `docs/cli.md` line 125: change the `sase bead list` description to "List bead issues
  by status, type, tier, or creation date."

## Verification

```bash
just install
sase skill init --diff            # read-only preview; never --force here
just check
```

`just check` runs the whole-repo lint gates plus the diff-scoped test lane, which
selects `tests/test_bead/test_cli_list.py` and `tests/main/test_init_skills_sources.py`.
Run `just check-full` if the scoped selection escalates or reports anything unusual.

Then sanity-check the exact command the skill now ships, against the live store:

```bash
sase bead list --type task --since 1w --status all | wc -l     # ~100 rows today, not 20
sase bead list --type task --since 1w --status all -f json | head -20
sase bead list --type task --since notadate                    # exits 2 with DATE help
sase bead list --type task --since 1w --until 2w               # exits 2, since > until
```

## Done when

- `sase bead list` accepts `-S/--since DATE` and `-u/--until DATE`, bounding bead
  creation time with the shared SASE DATE grammar, and `--status all` selects every
  status.
- A date bound suppresses the newest-20 closed default, so the sweep cannot silently
  truncate; an explicit `--limit` still wins.
- `/sase_new_task` instructs agents to sweep every task bead created in the last week
  with `sase bead list --type task --since 1w --status all` in compact format, before
  concluding a report is new.
- `/sase_new_task` instructs agents to record a `RELATED: <bead-id> — <why>` note on the
  new task for each related-but-not-duplicate bead they found.
- The narrowed regression guard still fails if an unbounded `sase bead list --type task`
  dump or a `--format full` task listing returns to the skill, and still fails if the
  epic list disappears.
- `docs/beads.md` and `docs/cli.md` describe the new flags; `sase bead onboard` shows
  the sweep.
- `just check` passes, and no memory file, no file under `../sase-core`, and no
  chezmoi-deployed skill file was modified.
