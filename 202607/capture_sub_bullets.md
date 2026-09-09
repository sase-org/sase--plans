---
tier: epic
title: Capture sub-bullets onto existing Obsidian tasks
goal: "`bob capture` can append a sub-bullet to an existing Obsidian task via a new
  `@<route>^<block-id>` marker, `bob capture-tasks` lists a note's open tasks with their
  statuses, and the Hammerspoon capture panel resolves a bare `@<route>^` into a
  status-annotated task picker.

  "
phases:
  - id: scan
    title: Shared note-task scanner
    depends_on: []
    size: medium
    description:
      "scan: add src/native/note_tasks.rs, a pure scanner that turns one note's Markdown
      into task records (line, indentation, status, description, block ID, section,
      child span, stale-safe ref digest), plus the small markdown.rs ATX-heading
      extraction it depends on."
  - id: write
    title: Sub-bullet capture in bob capture
    depends_on:
      - scan
    size: medium
    description:
      "write: teach bob capture the @<route>^<block-id> marker plus -t/--task and hidden
      --task-ref, render and insert the child bullet at the parent task's own
      indentation, and report the new sub_bullet kind in human and JSON output."
  - id: list
    title: bob capture-tasks discovery command
    depends_on:
      - write
    size: medium
    description:
      "list: add the read-only bob capture-tasks subcommand that lists a note's open
      tasks as colored human output and stable JSON for pickers, and wire it into the
      runner, help surfaces, justfile smoke list, and README."
  - id: ui
    title: Hammerspoon task picker
    depends_on:
      - list
    size: medium
    description:
      "ui: extend the chezmoi Hammerspoon capture panel with the @<route>^<id> marker
      family, a status-annotated task chooser backed by bob capture-tasks, sub-bullet
      success notifications, and busted coverage of the new request model."
create_time: 2026-09-09 19:53:05
status: wip
---

- **PROMPT:**
  [prompts/202607/capture_sub_bullets.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/capture_sub_bullets.md)

# Plan: Capture sub-bullets onto existing Obsidian tasks

## Context

`bob capture` currently turns its `TEXT` argument into a **new** line in the Bob vault.
The route marker grammar it accepts today lives in `src/native/capture.rs`:

| Marker                            | Result                                                      |
| --------------------------------- | ----------------------------------------------------------- |
| `@<route>`                        | new `- [ ] #task …` in `<route>.md`                         |
| `@<route>#<prefix>` / `@<route>#` | new `- …` bullet in a non-`Tasks` section                   |
| `@<route>:<block-id>`             | new `- [*] #task … ^<block-id>` plus a Pomodoro ledger link |
| (none)                            | new task in `mac_inbox.md`                                  |

This epic adds a fourth marker, `@<route>^<block-id>`, which does **not** create a task.
It appends a child bullet under an **existing** task that already carries that block ID,
and fails loudly when no such task exists.

Two read-only discovery commands already back the Hammerspoon capture panel:
`bob capture-targets` (`src/native/capture_targets.rs`) and `bob capture-sections`
(`src/native/capture_sections.rs`). This epic adds a third, `bob capture-tasks`, so the
panel can offer a task picker.

### Facts established while designing this plan

These were measured against the live vault and the current sources; implementers should
not re-derive them.

- **Configured Obsidian Tasks statuses** (from
  `~/bob/.obsidian/plugins/obsidian-tasks-plugin/data.json`): `' '` Todo/`TODO`, `x`
  Done/`DONE`, `/` In Progress/`IN_PROGRESS`, `*` Next/`ON_HOLD`, `?` Blocked/`ON_HOLD`,
  `-` Canceled/`CANCELLED`. `globalFilter` is `#task`.
- **"Every type except canceled and done"** is exactly the existing
  `TaskStatusType::is_open()` predicate in `src/native/task_status_hooks.rs`
  (`Todo | InProgress | OnHold`); `NON_TASK` and `EMPTY` are also excluded, which is
  correct because neither denotes an open task.
- **Only ~32% of open vault tasks carry a block ID** (49 of 151 across 26 root notes;
  `sase.md` alone has 49 open tasks, 12 with IDs). The task picker therefore **must**
  work for tasks with no block ID. It does so without minting anything: a child bullet
  does not require its parent to have a block ID, so the picker addresses tasks by a
  stale-safe ref instead.
- **Block-ID charset** is `collect_done::is_block_id_byte`: ASCII alphanumeric or `-`.
  (Note: the existing Pomodoro usage error text wrongly claims `_` is also allowed. Do
  not copy that wording; see "Out of scope".)
- **Child-bullet indentation in the vault is mixed**: hand-written children use a tab
  (`\t- Kelly just needs 2 pay stubs…` in `cash.md`), machine-written dependency embeds
  use two spaces (`  - ![[#^unemployment]]`). `~/bob/.obsidian/app.json` sets no
  `useTab`, so Obsidian's tab default applies. Hence: match the parent's existing
  children first, and only fall back to a tab.
- **The Hammerspoon panel hotkey is `cmd+shift+ctrl+i`**, not `ctrl+shift+alt+i`
  (`hs.hotkey.bind({ "cmd", "shift", "ctrl" }, "i", …)` at the bottom of
  `home/dot_hammerspoon/init.lua` in the chezmoi repo; `ctrl+alt+shift+s` is the
  screenshot binding). The requested behavior belongs to this panel; the binding itself
  is not changed.
- `src/native/markdown.rs` already exposes `pub(crate)` fence and frontmatter helpers
  (`fence_marker`, `closes_fence`, `fenced_lines`, `strictly_closed_frontmatter_end`)
  that the scanner should reuse.

### Design principles for this epic

1. **The grammar stays symmetric.** `^` joins `#` and `:` as a suffix separator on an
   `@route` token, and the Hammerspoon panel gains the same four-way incomplete-marker
   family that `:` already has.
2. **Never create, never mint.** Sub-bullet capture only edits an existing note and only
   adds one child block. It never creates a note and never rewrites the parent task
   line.
3. **Fail loudly and usefully.** Every failure names the note, what was looked for, and
   the command that lists valid choices.
4. **Match the note, not a house style.** Indentation and line endings are copied from
   the parent task's existing children.

## Grammar

`@<route>^<block-id>` may lead or trail the body, exactly like the existing markers, and
composes with the terminal `s:<N>` and `%…` markers in either order:

```bash
bob capture '@cash^goog-exit' 'Called Morgan Stanley today.'
bob capture 'Called Morgan Stanley today. @cash^goog-exit'
bob capture 'Called Morgan Stanley today. @cash^goog-exit s:1'
```

**Marker-kind precedence.** After the leading `@`, the _first_ of `^`, `:`, `#` to
appear selects the marker kind. So `@foo^bar` is a sub-bullet marker, `@foo:bar` stays a
Pomodoro marker, `@foo#bar` stays a bullet marker, and `@foo^ba:r` is a malformed
sub-bullet marker (rejected) rather than a Pomodoro marker.

Rendered result: the body becomes one child bullet of the target task,

```markdown
- [*] #task Finish Google Exit Packet! [created::2026-07-31] ^goog-exit
  - Called Morgan Stanley today. [created::2026-07-31]
```

## Phase `scan`: Shared note-task scanner

Add `src/native/note_tasks.rs` and register it in `src/native.rs`'s module list
(`mod note_tasks;`, kept alphabetically sorted). This phase adds **no CLI surface**; it
is the library both later phases consume.

### Prerequisite refactor

Move `atx_heading` and `strip_closing_atx_hashes` out of `src/native/capture.rs` into
`src/native/markdown.rs` as
`pub(crate) fn atx_heading(line: &str) -> Option<(usize, &str)>` with identical
behavior, and have `capture.rs` call `markdown::atx_heading`. This is a pure move: no
behavior change, and `cargo test` must stay green without touching existing capture
tests.

### Public API

```rust
pub(crate) struct NoteTask {
    pub(crate) line_index: usize,        // 0-based index into the note's lines
    pub(crate) indentation: String,      // the parent task's own leading whitespace
    pub(crate) status_symbol: char,
    pub(crate) status_name: String,      // from Tasks settings; "Unknown" when unconfigured
    pub(crate) status_type: TaskStatusType,
    pub(crate) description: String,      // display text (see below)
    pub(crate) block_id: Option<String>,
    pub(crate) section: Option<String>,  // nearest preceding ATX heading title
    pub(crate) child_count: usize,       // child lines already in the block
    pub(crate) block_end: usize,         // byte offset just past the task's block
    pub(crate) digest: String,           // 8 hex chars, see "Task refs"
}

pub(crate) struct NoteTaskScan { /* tasks, plus the block-id index */ }

pub(crate) fn scan(contents: &str, settings: &NoteTaskSettings) -> NoteTaskScan;
pub(crate) fn read_settings(bob_dir: &Path) -> NoteTaskSettings;
```

`NoteTaskScan` must also answer:

- `open_tasks(&self) -> impl Iterator<Item = &NoteTask>` — `status_type.is_open()`.
- `by_block_id(&self, id: &str) -> BlockIdLookup` — `Found(&NoteTask)`,
  `NotATask { line_index, excerpt }`, `Duplicate(usize)`, or `Missing`.
- `by_ref(&self, line: usize, digest: &str) -> RefLookup` — see "Task refs".
- `suggest_block_id(&self, id: &str) -> Option<&str>` — a "did you mean" candidate.

### Settings

`read_settings` reads `.obsidian/plugins/obsidian-tasks-plugin/data.json` under the
vault root and yields the global filter plus a symbol → (name, type) map.
`src/native/task_status_hooks.rs::read_tasks_settings` already does exactly this; reuse
it rather than writing a third copy — promote the needed items (`TasksSettings`,
`TaskStatusDefinition`, `TaskStatusType`, `read_tasks_settings`,
`TaskStatusType::is_open`) to `pub(crate)` and re-export or wrap them from `note_tasks`.
Missing or malformed settings fall back to the built-in defaults that function already
carries (`' '`, `x`, `X`, `/`, `*`, `-`) and must not be an error; an unconfigured
symbol resolves to name `"Unknown"` with type `Todo`, so an unknown status is listed as
open.

### Scanning rules

A line is a task when, ignoring lines inside YAML frontmatter and fenced code blocks
(use `markdown::strictly_closed_frontmatter_end` and `markdown::fenced_lines`), it
matches `^(\s*)- \[(.)\] ` **and** its remainder contains the configured global filter
as a whitespace-delimited token (when the global filter is empty, every checkbox item
qualifies). Indented (sub-)tasks qualify too.

- `block_id`: reuse `collect_done::block_ids_in_markdown`'s trailing-`^id` rule — the ID
  must be the last whitespace-delimited token and use only
  `collect_done::is_block_id_byte` characters. Promote a single-line helper from
  `collect_done` to `pub(crate)` rather than reimplementing the rule.
- `description`: the text after `- [x] `, with the global-filter tag, every
  `[key:: value]` inline field, and the trailing `^<id>` removed, then whitespace
  collapsed and trimmed. Other tags (`#prj`, `#hide`) and links stay.
- `section`: the title of the nearest preceding ATX heading (via
  `markdown::atx_heading`), or `None` above the first heading.
- **Block extent**: starting after the task line, consume every line whose leading
  whitespace is strictly longer than the task's own `indentation`. Blank lines are
  consumed only when a deeper-indented line follows before the next non-blank line at or
  below the task's indentation. `block_end` is the byte offset just past the last
  consumed line (or just past the task line when it has no children). `child_count`
  counts the consumed non-blank lines.

### Task refs

A **ref** addresses a task that may have no block ID. Its wire form is
`<1-based-line>:<digest>`, for example `24:1f3a9c2b`. The digest is the first 8
lowercase hex characters of `sha2::Sha256` over the task line's content with trailing
whitespace and the line ending removed (`sha2` and `hex` are already direct
dependencies).

`by_ref(line, digest)` resolves in this order, which makes the ref survive edits
elsewhere in the note:

1. The task at that 1-based line whose digest matches → `Found`.
2. Otherwise, tasks anywhere in the note whose digest matches: exactly one → `Found`
   (the note shifted); zero → `Stale`; more than one → `Ambiguous`.

### `suggest_block_id`

Return at most one candidate from the note's task block IDs: a case-insensitive exact
match if one exists, otherwise the unique block ID within Levenshtein distance 2 of the
requested ID. Implement the bounded distance inline (roughly 20 lines); do not add a
dependency.

### Tests (unit tests in `src/native/note_tasks.rs`)

Cover: statuses from real settings JSON and from missing settings; `is_open` filtering
excludes `x` and `-` and includes ` `, `/`, `*`, `?`, and an unconfigured symbol; the
global filter gating checkbox lines; block IDs found, absent, and duplicated;
description cleaning (inline fields, `#task`, trailing `^id`, collapsed whitespace);
section attribution including a task above any heading; block extent across tab
children, two-space children, interior blank lines, a deeper grandchild, an indented
parent task, and end-of-file; frontmatter and fenced-code lines never scanned as tasks;
`by_ref` for exact hit, shifted-line recovery, stale, and ambiguous; and
`suggest_block_id` for case-only and one-character typos plus a no-suggestion case.

## Phase `write`: Sub-bullet capture in `bob capture`

All changes are in `src/native/capture.rs` unless noted.

### Parsing

1. Add `CaptureKind::SubBullet { target: SubBulletTarget }` where `SubBulletTarget` is
   `BlockId(String)` or `Ref { line: usize, digest: String }`.
2. Add `is_sub_bullet_marker_candidate(token)`: the token starts with `@` (not `@!`),
   its remainder contains `^`, and that `^` precedes any `:` or `#`.
3. Add `parse_sub_bullet_route_token(token)`, mirroring `parse_pomodoro_route_token`.
   The route is lowercased and validated with `is_route_token`; the block ID is
   validated with the existing `is_block_id`.
4. Amend `is_pomodoro_marker_candidate` so a `^` appearing before the `:` disqualifies
   the Pomodoro reading, preserving the precedence rule above.
5. Call the new parser from `parse_terminal_route_token`, and add it to
   `validate_special_terminal_markers` and `is_route_marker` so malformed `^` markers
   fail loudly and so `s:<N>` / `%…` extraction still works on both sides of a
   sub-bullet marker.

### New options

- `-t, --task <BLOCK-ID>`: append the text as a sub-bullet of the task with that block
  ID. Requires `--route`; conflicts with `--section`. Like `--route`, it keeps every
  `@token` in `TEXT` literal. Public, listed in help, alphabetically placed between
  `--section` and the positional `TEXT`.
- `--task-ref <REF>`: the same, addressing the task by a `bob capture-tasks` ref.
  Requires `--route`; conflicts with `--task` and `--section`. **Hidden**
  (`.hide(true)`, as `src/runner.rs` already does for its aliases) because it is a
  picker-internal argument, which is what keeps
  `public_help_surfaces_do_not_list_long_only_options` in `tests/cli.rs` green without
  inventing a short alias.

### Writing

Add `plan_sub_bullet_capture(bob_dir, target, route, sub_bullet_target, capture_block)`
returning the existing `CaptureWritePlan` so the atomic `replace_single_file` path is
reused unchanged.

1. The note must already exist. A missing note is an error, never a creation.
2. Scan with `note_tasks::scan` and resolve the target.
3. **Child indentation**, in order: (a) the leading whitespace of the parent's first
   child line, when it is strictly longer than the parent's own indentation; (b) the
   parent's indentation plus the note's dominant indent unit — a tab when more of the
   note's indented lines begin with a tab, otherwise two spaces; (c) the parent's
   indentation plus a tab.
4. Build the block: prefix the rendered bullet line and every clipboard child line with
   that indentation. Clipboard lines from `capture_clip` already carry their own
   two-space relative indent (`rendered_lines` in `src/native/capture_clip.rs`), so
   prefixing nests them correctly under the new sub-bullet.
5. Insert immediately after `block_end`, preserving the document's line ending (reuse
   `insertion_text_preserving_line_endings`). `Placement` is `Appended` when the
   insertion point is end-of-file, otherwise `Inserted`. `Placement::Created` is never
   produced in this mode.

The rendered line is `format_bullet_line`'s existing shape,
`- <body> [created::YYYY-MM-DD]`, with `[scheduled::YYYY-MM-DD]` appended when `s:<N>`
was given. `s:<N>` is accepted here for consistency with today's bullet capture, even
though Obsidian Tasks does not read a scheduled field off a non-task line; say so in the
README rather than rejecting it.

### Errors

All are `CaptureError::io` except the usage cases, and every message is a single line so
JSON mode stays a clean `{"ok":false,"error":"…"}`.

| Situation                  | Message                                                                                                                         |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `@<route>^` (no ID)        | `sub-bullet capture requires a block ID: @<route>^<block-id> (run 'bob capture-tasks -r <route>' to list task block IDs)`       |
| `@^<id>` (no route)        | `sub-bullet capture markers must use @<route>^<block-id>`                                                                       |
| bad route chars            | `sub-bullet capture route must contain only A-Z, a-z, 0-9, '_' or '-'`                                                          |
| bad ID chars               | `sub-bullet capture block ID must be non-empty and contain only A-Z, a-z, 0-9 or '-'`                                           |
| `--task` without `--route` | `--task requires --route`                                                                                                       |
| malformed `--task-ref`     | `--task-ref must use <line>:<digest>`                                                                                           |
| note missing               | `note does not exist: <abs path>`                                                                                               |
| ID not found               | `no task with block ID ^<id> in <route>.md (run 'bob capture-tasks -r <route>' to list task block IDs)`                         |
| ID not found, close match  | `no task with block ID ^<id> in <route>.md; did you mean ^<other>? (run 'bob capture-tasks -r <route>' to list task block IDs)` |
| ID on a non-task line      | `^<id> in <route>.md is not a task (line <N>: <excerpt>)`                                                                       |
| ID duplicated              | `block ID ^<id> appears <N> times in <route>.md; make it unique before capturing`                                               |
| ref stale                  | `the selected task is no longer in <route>.md; rerun the task picker`                                                           |
| ref ambiguous              | `the selected task matches more than one line in <route>.md; rerun the task picker`                                             |

A block ID that resolves to a task of any status is accepted, including a done or
canceled one — annotating why something was canceled is legitimate, and the picker never
offers those anyway.

### Output

`capture_kind_label` gains `CaptureKind::SubBullet => "sub_bullet"`. Add these `Option`
fields to `CaptureResult`, all `skip_serializing_if = "Option::is_none"` so every
existing JSON consumer is unaffected: `parent_line` (1-based), `parent_text`,
`parent_status_symbol`, `parent_status_name`. `block_id` is reused for the parent's
block ID and is `null` when the parent has none. `task_line` holds the rendered child
bullet, consistent with its documented meaning for both existing kinds.

Human output:

```
✓ captured   cash.md
  under [*] Finish Google Exit Packet!  ^goog-exit
  - Called Morgan Stanley today. [created::2026-07-31]
```

The status marker is colored through `Styler` on the same scale the next phase uses:
`[ ]` dim, `[/]` blue, `[*]` yellow, `[?]` red, anything else dim. The block ID is cyan
and omitted entirely when the parent has none. `--dry-run` keeps the `would capture`
wording already in `print_human_success`.

### Documentation

Update `build_cli`'s `long_about` with a sub-bullet paragraph, add
`bob capture '@cash^goog-exit' 'Called Morgan Stanley today.'` to `after_help`, add both
options to `README.md`'s option list, and add a README subsection documenting the
marker, the child-indentation rule, the failure modes, and the `sub_bullet` JSON shape.

### Tests

Unit tests in `capture.rs` for the marker grammar: leading and trailing position;
precedence against `:` and `#`; composition with `s:<N>` and `%` in both terminal
orders; `@foo^bar` no longer captured as literal text; malformed forms rejected.

Integration tests in `tests/cli.rs` (follow the existing capture-test helpers) for:
appending under a tab-indented parent and a two-space-indented parent; a parent with no
children in a note that uses tabs elsewhere; a parent whose last child is a deeper
grandchild (the new bullet lands at the first child's level, not the grandchild's); an
indented parent task; `--task` and `--task-ref` including the shifted-line recovery
path; a clipboard capture nesting its lines under the new sub-bullet; `--dry-run`
leaving the note byte-identical; CRLF preservation; and one integration test per error
row above asserting exit code and message in both human and JSON modes. Extend
`capture_help_lists_options_alphabetically` for `--task` and assert `--task-ref` does
not appear in `bob capture --help`.

## Phase `list`: `bob capture-tasks` discovery command

Add `src/native/capture_tasks.rs`, modeled closely on `src/native/capture_sections.rs`
(same option set, error enum, `OutputFormat`, and JSON-error convention). It is strictly
read-only.

```bash
bob capture-tasks --route NAME [-b|--bob-dir DIR] [-f|--format human|json]
```

Options are `-b, --bob-dir`, `-f, --format`, `-h, --help`, `-r, --route` — alphabetical,
each with a short alias. A missing note is **not** an error; it returns an empty list,
matching `capture-sections`.

### Wiring

- `src/native.rs`: `mod capture_tasks;`, `NativeCommand::CaptureTasks`, and the `run`
  arm.
- `src/runner.rs`: a `SUBCOMMANDS` entry named `capture-tasks`, placed after
  `capture-targets` (`capture-targets` < `capture-tasks`) so
  `subcommands_are_sorted_alphabetically` stays green, plus an `AFTER_HELP` example.
- `justfile`: add `"${root}/bin/bob" capture-tasks --help >/dev/null` to
  `install-smoke`, in sorted position.
- `tests/cli.rs`: add `capture-tasks` to
  `all_top_level_subcommand_help_is_safe_and_plain` and
  `public_help_surfaces_do_not_list_long_only_options`, and add a
  `capture_tasks_help_is_native_only` test mirroring
  `capture_sections_help_is_native_only`.

### JSON contract

```json
{
  "ok": true,
  "route": "cash",
  "relative_target": "cash.md",
  "count": 2,
  "tasks": [
    {
      "ref": "24:1f3a9c2b",
      "line": 24,
      "block_id": "goog-exit",
      "status_symbol": "*",
      "status_name": "Next",
      "status_type": "ON_HOLD",
      "text": "Finish Google Exit Packet!",
      "section": "Tasks",
      "depth": 0,
      "child_count": 1
    }
  ]
}
```

`tasks` is in document order and contains only open tasks. `block_id` and `section` are
`null` when absent. `depth` is the parent's indentation measured in levels (a tab, or
each group of up to four leading spaces, counts as one) and lets a picker indent
sub-tasks. `status_type` uses the same `SCREAMING_SNAKE_CASE` spelling as the Tasks
plugin.

### Human output

```
Capture tasks · cash.md

  Tasks
    [*] Finish Google Exit Packet!                 ^goog-exit   Next
    [?] Set up Actual Budget!                      ^budget      Blocked
    [ ] Call about unemployment to verify real ID!              Todo

  3 tasks · 1 next · 1 in progress · 1 blocked
```

Group by `section` in document order under a dimmed heading; ungrouped tasks come first
under no heading. Color via `Styler` (never raw escapes): status marker `[ ]` dim, `[/]`
blue, `[*]` yellow, `[?]` red, other dim; description default; block ID cyan; status
name dim. Pad columns with `style::pad_right` and `style::display_width` as
`capture_targets.rs` does. Indent each row by `depth` levels. Print
`No open tasks found.` when the list is empty. The summary line counts only the statuses
actually present, joined by `styler.separator()`.

### Tests

Unit tests in `capture_tasks.rs`: route normalization and rejection; a missing note
returning an empty successful result; open-task filtering against a fixture note holding
all six statuses; document ordering and section grouping; and a stable JSON shape
assertion in the style of `capture_sections.rs::json_success_shape_is_stable`. Add one
`tests/cli.rs` end-to-end test running `bob capture-tasks -r … -f json` against a temp
vault and asserting the shape, plus one asserting human output has no ANSI when stdout
is not a TTY.

### Documentation

Extend the `README.md` discovery-command section: add `capture-tasks` to the command
table near line 53, document the flag set, the "open tasks only" rule (everything except
`DONE` and `CANCELLED`, with unknown symbols treated as open), the JSON contract above,
and a fourth step in the picker sequence — run `capture-tasks` for the route and then
`bob capture --route NAME --task-ref REF -- <text>`.

## Phase `ui`: Hammerspoon task picker

This phase edits the **chezmoi** repo. Open it first with
`sase repo open chezmoi -r "<reason>"` and use only the path that command prints. Paths
below are relative to that checkout. After committing there, run
`chezmoi update -a --force` as `CLAUDE.md` in that repo requires.

### `home/dot_hammerspoon/task_capture.lua`

Add a `sub_bullet` mode completing the four-way family that `:` already has, so `^`
behaves exactly like `:` does:

| Typed marker | `mode`                                     | Behavior                      |
| ------------ | ------------------------------------------ | ----------------------------- |
| `@route^id`  | `sub_bullet`                               | capture immediately           |
| `@route^`    | `sub_bullet`, `needs_task`                 | task picker for that route    |
| `@^id`       | `sub_bullet`, `needs_target`               | note picker, then capture     |
| `@^`         | `sub_bullet`, `needs_target`, `needs_task` | note picker, then task picker |

- Add `sub_bullet_candidate(token)` mirroring `pomodoro_candidate`: the token starts
  with `@` (not `@!`) and its `^` precedes any `:` or `#`. Check it **before**
  `pomodoro_candidate`, and make `pomodoro_candidate` reject a token whose `^` precedes
  its `:`, so the two never both claim a token.
- Add `parse_sub_bullet(body, token)` returning `invalid` with the same messages the CLI
  uses for a bad route or block ID.
- Add `M.finalize_sub_bullet(request, route, block_id)` returning
  `"@" .. route .. "^" .. block_id .. " " .. request.text`, and extend
  `M.stage`/`M.reset` with a `task_ref` slot so a chosen ref survives a failed capture
  the way `block_id` already does.
- Mid-text markers stay literal, and a marker-only body still yields `text == ""` and is
  a no-op, exactly as today.

### `home/dot_hammerspoon/init.lua`

- Add `taskCaptureTasksCommand` alongside `taskCaptureSectionsCommand`, running
  `exec bob capture-tasks --format json --route "$1"`.
- Extend `taskCaptureCommand` with a sub-bullet branch. Keep the existing
  positional-parameter discipline — text is `$1` and nothing is interpolated — and pass
  the ref as its own positional parameter so
  `bob capture --format json --route "$2" --task-ref "$4" -- "$1"` runs when `$4` is
  set. Always use `--task-ref` from the picker (it is correct whether or not the task
  has a block ID, and it self-heals if the note shifted); the typed `@route^id` form
  continues to flow through the plain text path unchanged.
- Add `startTaskStage(request, route, pickedName, pickedKind)`: run `capture-tasks`; on
  `count == 0` call `notifyTargetPickerFailure` with `no open tasks in <route>.md`;
  otherwise show the chooser. Guard the async callback with the live task object exactly
  as `startSectionStage` does.
- Add `showTaskChooser(...)` with `chooser:placeholderText("Capture under task")` and
  `chooser:searchSubText(true)` so typing a status name, section, or block ID filters.
  Dismissing it refocuses the prompt with the typed text intact, matching the other
  choosers.
- Route the new modes from `submitCapturedTask`: `sub_bullet` with `needs_target` goes
  to `startTargetsStage` (extend `showTaskCaptureChooser` to continue into
  `startTaskStage` for this mode); `sub_bullet` with only `needs_task` goes straight to
  `startTaskStage`; a complete `@route^id` captures immediately.

### Chooser row design

Each row's main text is the literal Obsidian checkbox followed by the description,
`[*] Finish Google Exit Packet!`, because the checkbox is exactly what the user sees in
the vault and needs no legend. Color only the checkbox, via `hs.styledtext.new` over the
marker substring:

| Status        | Marker         | Color                |
| ------------- | -------------- | -------------------- |
| Todo          | `[ ]`          | secondary label gray |
| In Progress   | `[/]`          | `#0a84ff` blue       |
| Next          | `[*]`          | `#ff9f0a` orange     |
| Blocked       | `[?]`          | `#ff453a` red        |
| anything else | its own marker | secondary label gray |

Wrap the styled-text construction in `pcall` and fall back to the identical plain string
on failure, so a styling problem can never break the picker — the same best-effort
discipline `obsidianContentImage` and `captureTargetImage` already use. Row `subText` is
the status name, then the block ID as `^id` when present, then the section, then
`<n> notes` when `child_count > 0`, joined with the existing `captureLabelSeparator`
(`·`). Indent the row text by two spaces per `depth` so sub-tasks read as sub-tasks. No
per-row image: the colored checkbox already carries the status, and an icon would
compete with it.

### Notification

In `notifyCaptureSuccess`, add `decoded.kind == "sub_bullet"` → `✓ Sub-bullet captured`.
Set `subTitle` to the destination label already computed by `captureDestinationLabel`
followed by the parent's `parent_text`, truncated with `truncateForBanner`, and keep
`informativeText` as the captured text. The click-to-open Obsidian URL and content image
paths are unchanged.

### Tests

Extend `tests/hammerspoon/task_capture_spec.lua` (run with `just test-hammerspoon` in
the chezmoi repo, which uses busted). Mirror the Pomodoro test structure:

- all four `^` forms parse into the right mode and flags;
- `^` and `:` precedence, including `@route^bad:id` being an invalid sub-bullet rather
  than a Pomodoro request, and `@route:id` still parsing as Pomodoro;
- invalid components (`@bad.route^id`, `@route^bad.id`, `@^`) return `mode == "invalid"`
  with an error;
- mid-text `@route^id` stays literal and a marker-only body yields `text == ""`;
- every form converges on `@route^id <text>` through `finalize_sub_bullet`;
- staged `task_ref` survives a simulated capture failure and is cleared by `M.reset`;
- the existing Pomodoro, note, and section expectations still pass unchanged.

Run `just fmt-lua` (stylua) before finishing.

### Documentation

Update the README section that describes the panel's incomplete trailing markers (around
line 291 of `README.md` in **this** repo) with the four `^` forms, and keep its
`cmd+shift+ctrl+i` wording accurate.

## Verification

Each phase must leave `just all` (`cargo fmt --check`,
`cargo clippy --all-targets --all-features`, `cargo test`) green in this repo; the `ui`
phase must additionally leave `just test-hammerspoon` and `just fmt-lua` green in the
chezmoi repo. After the epic lands, this end-to-end check should pass against a scratch
vault:

```bash
bob capture-tasks -r cash -f json          # lists open tasks, none done or canceled
bob capture '@cash^goog-exit' 'Called Morgan Stanley today.'
bob capture '@cash^nope' 'x'               # exits 1 with the did-you-mean error
```

## Out of scope

- **Non-task blocks.** A `^<id>` on a plain bullet or paragraph (for example
  `^workspaces` and `^balance` in the live vault) is rejected with a clear error.
  Extending sub-bullet capture to arbitrary blocks is a separate change.
- **Minting block IDs.** Sub-bullet capture never rewrites the parent task line. A task
  with no block ID is reachable through the picker's ref, not by giving it an ID.
- **`bob capture-tasks --all`.** Listing done and canceled tasks is deliberately
  omitted; the command's contract is "open tasks".
- **The `_` in the Pomodoro block-ID error text.** `parse_pomodoro_route_token` claims
  `_` is a legal block-ID character, but `collect_done::is_block_id_byte` rejects it.
  The new sub-bullet messages state the true charset; correcting the older Pomodoro
  message is a small, separate follow-up worth filing.
- **The hotkey itself.** The panel is bound to `cmd+shift+ctrl+i`, and this epic changes
  only what that panel accepts.
