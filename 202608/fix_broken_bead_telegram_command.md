---
tier: tale
title: Repair the Telegram /bead command end to end
goal:
  "`/bead` in Telegram renders the active-bead picker again, built from a
  machine-readable `sase bead list` contract over enabled projects only, and no SASE
  surface ever pastes a raw Python traceback into the chat."
proposed_by: bbugyi200.athena.vm
create_time: 2026-08-07 23:27:56
status: wip
---

# Plan: Repair the Telegram /bead command end to end

## Symptom

`/bead` (no arguments) in Telegram replies with a fenced code block containing a raw
Python traceback instead of the inline-keyboard picker of active beads:

```
sase-github: Traceback (most recent call last):
  File "/home/bryan/.local/bin/sase", line 10, in <module>
    sys.exit(main())
  ...
  File ".../sase/bead/cli_query.py", line 61, in handle_bead_list
    with get_read_view() as view:
  File ".../sase/bead/cli_common.py", line 122, in get_read_view
    return get_project()
  File ".../sase/bead/cli_common.py", line 105, in get_project
    location = resolve_beads_location(cwd=cwd, materialize=True)
  File ".../sase/bead/cli_location.py", line 101, in resolve_beads_location
    store = materialize_sdd_store(context.root, context.workspace_num)
  ...
  File ".../sase/sdd/_store_materialization.py", line 314, in _resolve_sdd_creation_authorization
    raise SddMaterializationError(
sase.sdd._store_types.SddMaterializationError: SDD materialization refused:
repository is not SASE-managed; set is_sase_managed: true in the target
repository's sase/sase.yml to enable it
```

## Root cause

Three independent defects compose into this failure. All three were reproduced directly;
do not treat any of them as speculative.

### Defect 1 (primary) — the Telegram list parser pins a retired output format

`sase_telegram/bead_format.py::parse_bead_list_output` matches compact rows with:

```python
_BEAD_LIST_LINE_RE = re.compile(
    r"^(?P<icon>\S+)\s+(?P<id>\S+)\s+·\s+(?P<title>.+?)"
    r"(?:\s+←\s+(?P<parent>\S+))?$"
)
```

That is a **single** leading glyph followed by the bead id. The current
`sase bead list --format=compact` row is:

```
◆ ◐ sase-ct · Flaky ACE TUI tests under full parallel just test run [+28] [↺6]  ⧖ 7d
▸ ◐ sase-h7 · Gate input collection and repeatable non-terminal gate actions  ⧖ 6h
↳ ◐ sase-h8.5 · Fix the real-wall-clock-threshold family ← sase-h8  ⧖ 5h
```

Two leading glyphs (a type/tier glyph from `bead_type_cli_cell`, then a status glyph
from `bead_status_presentation`), optional `[+N]` / `[↺N]` badges, and a trailing
`⧖ <age>` cell from `created_cell`. The regex cannot match any of it, and the trailing
age cell also defeats the `← <parent>` anchor at `$`.

Verified against real output: 18 rows in, **0 entries out**.

The parser was written 2026-04-28 (`35cbfc7`, `5658161` in `sase-telegram`) and never
revisited. The row format changed in the `sase` repo across `01ace663f` (type
indicator), `7d4afb394`
(`feat(cli)!: render compact bead rows with glyph-only type cells`) and `e4fce05b6`
(creation-time cell, 2026-08-05). `tests/test_bead_format.py::TestParseBeadListOutput`
still asserts the April format, so the plugin's own suite stayed green while production
broke.

This alone breaks `/bead`: with zero parsed entries the picker has nothing to show for
**any** project.

### Defect 2 — a read-only bead command escalates into SDD materialization and dies with an uncaught traceback

`get_read_view()` (`src/sase/bead/cli_common.py:117`) falls through to `get_project()`,
which calls `resolve_beads_location(cwd=cwd, materialize=True)` (`:105`) whenever the
warm store is missing or unusable. That reaches `_materialize_sdd_store_with_created` →
`_resolve_sdd_creation_authorization` (`src/sase/sdd/_store_materialization.py:170`),
which raises `SddMaterializationError` at `:314` for any repository lacking
`is_sase_managed: true` in `sase/sase.yml`.

Nothing between that raise and `main()` catches it, so `sase bead list` — a pure read —
exits with a full Python traceback on stderr.

Reproduced:
`cd /home/bryan/projects/github/sase-org/sase-github && sase bead list --status open`
emits exactly the traceback above.

`sase-github` is a linked/sibling repo: it has no `sase/sase.yml`, no
`.sase/sdd-store.json` record, and no warm `.sase/sdd/` clone. It is the only enumerated
workspace in that state, which is why it alone takes the cold materialization path.
Every other enumerated workspace has a warm `.sase/sdd/beads` clone and short-circuits
before materializing.

### Defect 3 — the bot fans out over every project directory and pastes raw stderr

`_iter_known_project_workspaces()` (`src/sase_telegram/scripts/sase_tg_inbound.py:720`)
globs `~/.sase/projects/*/<name>.sase` and accepts anything with a `WORKSPACE_DIR:`.
That yields 10 workspaces, including `PROJECT_STATE: sibling` linked-repo entries
(`sase-github`, `sase-core`, `sase-nvim`, `sase-telegram`, `chezmoi`, `bob-plugins`) and
stale legacy entries (`home`). `sase project list` reports only **3** enabled projects:
`actstat`, `bob-cli`, `sase`.

Then `_send_bead_subprocess_error()` escapes the failing subprocess's stderr and sends
it verbatim, so a Python traceback lands in the chat.

### How they compose

`_show_bead_selection` → `_project_bead_entries` collects `entries` and `errors`. Defect
3 puts `sase-github` in the loop; defect 2 makes it exit non-zero with a traceback;
defect 1 makes `entries` empty for every project that _succeeded_. The guard

```python
if errors and not entries:
    _send_bead_subprocess_error(chat_id, "\n".join(errors))
    return
```

then fires and dumps the traceback. Fixing only defect 2 or 3 would replace the
traceback with `No active beads.` — still broken. Defect 1 is the one that must be fixed
for `/bead` to work at all.

## Design

Stop scraping human-readable CLI output, and stop treating every registered project
directory as a bead-bearing project.

1. **Consume the JSON contract.** `sase bead list --format=json` already emits a stable
   envelope:

   ```json
   {
     "count": 18,
     "total": 18,
     "statuses": ["open", "in_progress"],
     "implied_status_closed": false,
     "results": [
       {"id": "sase-ct", "title": "...", "status": "in_progress",
        "issue_type": "task", "tier": null, "parent_id": null, ...}
     ]
   }
   ```

   `results[]` comes from `issue_to_wire_dict` (`src/sase/bead/cli_detail_json.py:47`)
   and is the shared read-command schema — it does not churn with row cosmetics.

2. **Derive the picker glyph from the shared presentation module** rather than from
   scraped text, so the icon keeps matching the CLI:
   `sase.bead_status_presentation.bead_status_presentation(status).glyph`. The plugin
   already imports from `sase` (e.g. `sase.project_display_names`,
   `sase.integrations.changespec_tags`), so this is an established pattern; keep the
   same defensive `try/except ImportError` fallback style used by
   `display_project_name`.

3. **Enumerate enabled projects from the CLI, not the filesystem.**
   `sase project list --state=enabled --json` returns records with `display_name`,
   `project_name` (the key), `workspace_dir`, and `state`.

4. **Never render raw stderr.** Report failures as a short one-line-per-project summary;
   keep the detail in the bot log.

5. **Make the core read fail cleanly.** A read-only bead command in a repository that is
   not SASE-managed should print one actionable line and exit non-zero — not a
   traceback.

## Implementation

The `sase` repo is the current workspace. Open the plugin repo with
`sase repo open sase-telegram -r "<reason>"` and use only the path it prints.

### Step 1 — `sase` repo: clean failure for read-only bead commands

In `src/sase/main/entry.py`, the bead lane already wraps `handler(args)` in a
`try/except BeadPublicationError` with a `finally` that schedules the bead refresh.
Extend that same `try` to also catch `SddMaterializationError` (import it lazily from
`sase.sdd._store_types`, matching the lazy-import style used in
`src/sase/bead/cli_common.py:173`):

- print `f"sase bead {bead_sub}: {exc}"` to `stderr`
- `sys.exit(1)`

Keep the message on one line and keep the existing exit code. Do not swallow the error
or exit 0 — a read that cannot reach a store must not report "no beads found", and the
Telegram bot needs the non-zero code to classify the project as skipped.

Do **not** change `_resolve_sdd_creation_authorization`'s policy. Refusing remote
materialization for an unmanaged repository is correct; only the presentation of that
refusal is wrong.

Add a test under `tests/` covering: a bead read in a workspace whose config lacks
`is_sase_managed: true` exits 1, prints the refusal message once, and prints no
`Traceback` to stderr.

### Step 2 — `sase-telegram`: parse the JSON list contract

In `src/sase_telegram/bead_format.py`:

- Add `parse_bead_list_json(raw: str) -> list[BeadListEntry]` that loads the envelope,
  iterates `results`, and builds one `BeadListEntry` per record from `id`, `title`,
  `parent_id`, and a `status`-derived `icon`.
- Tolerate a missing/!dict envelope and a missing `results` key by returning `[]` rather
  than raising — a malformed payload must degrade to "no beads for this project", not
  crash the handler.
- Keep `BeadListEntry` as-is (`icon`, `bead_id`, `title`, `parent_id`) so
  `_ProjectBeadEntry` construction is unchanged.
- Retain `parse_bead_list_output` only if something else still calls it; if it becomes
  unused, delete it together with its tests. Do not leave a known-broken parser behind
  as dead code.

Icon resolution helper, in the plugin's established defensive style:

```python
def _status_icon(status: str) -> str:
    try:
        from sase.bead_status_presentation import bead_status_presentation
    except ImportError:
        return "•"
    try:
        return bead_status_presentation(status).glyph
    except Exception:
        return "•"
```

### Step 3 — `sase-telegram`: switch the list subprocess to JSON

In `src/sase_telegram/scripts/sase_tg_inbound.py`:

- Add `"--format=json"` to `_ACTIVE_BEAD_LIST_ARGS` (line ~146).
- Replace the two `parse_bead_list_output(result.stdout)` call sites —
  `_project_bead_entries` (~3589) and `_legacy_bead_entries` (~3602) — with
  `parse_bead_list_json`.

`/bead <id>` keeps using `sase bead show` + `bead_show_to_markdown`; leave that path
alone in this plan (see Out of scope).

### Step 4 — `sase-telegram`: enumerate enabled projects via the CLI

Rewrite `_iter_known_project_workspaces()` to shell out to
`sase project list --state=enabled --json` and build `_KnownProjectWorkspace` from each
record's `project_name` (key) and `workspace_dir`, skipping records with an empty
`workspace_dir` and de-duplicating on `workspace_dir` as today.

Keep `display_project_name(project)` for user-facing labels so the picker shows `sase`,
never `gh_sase-org__sase` (see the "Show Project Names, Never ProjectSpec Keys"
convention).

If the subprocess fails or emits unparseable JSON, log and return `[]` — the caller
already falls back to the single-workspace `_run_active_bead_list` path. Do not silently
reintroduce the `~/.sase/projects/*` glob as a fallback; that glob is the defect.

### Step 5 — `sase-telegram`: stop pasting tracebacks into the chat

- In `_project_bead_entries`, reduce each failure to a single line: take the **last**
  non-empty stderr line (for an uncaught exception that is the `SomeError: message`
  summary line) and truncate it to a bounded length, e.g.
  `f"{display_project_name(project.project)}: {summary}"`. Log the full stderr at
  warning level.
- In `_show_bead_selection`, when `errors and not entries`, send a plain human-readable
  message such as `No active beads. <N> project(s) could not be listed:` followed by the
  one-line summaries — not a raw code block of stderr.
- Leave the existing `skipped_error_count` note on the success path as-is; it is already
  correct behavior.

### Step 6 — tests

In `sase-telegram`:

- `tests/test_bead_format.py`: replace `TestParseBeadListOutput` with JSON-envelope
  cases — a typical multi-bead envelope (assert ids, titles, `parent_id`, and
  status-derived icons), an empty `results` list, a malformed/non-dict payload, and a
  record missing optional keys.
- `tests/test_inbound.py`: cover (a) `_iter_known_project_workspaces` builds from
  `sase project list --state=enabled --json` and excludes a `sibling` record, (b)
  `_project_bead_entries` collapses a multi-line traceback on stderr to one line, (c)
  `_show_bead_selection` with zero entries and one error sends a message that contains
  neither `Traceback` nor a fenced code block.
- Assert `_ACTIVE_BEAD_LIST_ARGS` contains `--format=json` so the contract cannot
  silently regress to row scraping.

In `sase`: the Step 1 test described above.

## Verification

1. `just check` in the `sase` workspace (`just install` first if the workspace is cold).
   Use `just check-full` before landing.
2. The plugin's own gate in the `sase-telegram` checkout (`just check` there, or its
   `Justfile` equivalent).
3. Regression check that the old failure is gone and now reports cleanly:
   ```bash
   cd /home/bryan/projects/github/sase-org/sase-github && sase bead list --status open
   ```
   Expect exit 1, one line naming the refusal, and no `Traceback`.
4. Contract check against real data — this is the check that would have caught the bug,
   so run it explicitly:
   ```bash
   cd /home/bryan/projects/github/sase-org/sase && \
     sase bead list --status=open --status=in_progress --format=json > /tmp/beads.json
   python3 -c "import json,sys; sys.path.insert(0,'<sase-telegram>/src'); \
     from sase_telegram.bead_format import parse_bead_list_json; \
     e=parse_bead_list_json(open('/tmp/beads.json').read()); print(len(e)); print(e[:3])"
   ```
   The parsed count must equal the envelope's `count` (18 at the time of writing) —
   not 0.
5. Live check: send `/bead` in Telegram and confirm the inline-keyboard picker renders
   with real bead rows and no traceback.

## Out of scope

Record these as follow-ups via `/sase_new_task`; do not fold them into this plan.

- **`bead_show_to_markdown` has drifted too.** `/bead <id>` still renders — the header
  regex matches — but `sase bead show` now emits `Type: plan · Tier: epic · Owner: ...`
  (the `_TYPE_OWNER` regex expects `Type: X · Owner: Y`, so metadata parsing bails
  early), plus `CREATED`, `CREATED BY`, and `PAGE` sections, nested `PHASES` /
  `CHILD EPICS` sub-headers under `CHILDREN`, and `· Size: <size>` / `· Tier: <tier>`
  suffixes on child rows. Degraded, not broken.
- **No contract test binds the plugin to the `sase` CLI.** Both repos' suites passed
  throughout this outage. A shared fixture or CI job that exercises the plugin's parsers
  against real `sase` output would close the whole class.
- **`sase-github` has a stale in-repo `.sase_beads/` store** (`config.json`,
  `issues.jsonl`, `beads.db`) that no current `BEADS_DIRNAME` constant (`sdd/beads`,
  `beads`, `.`) resolves to. Worth a deliberate decision: migrate or remove.
