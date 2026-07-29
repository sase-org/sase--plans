---
tier: tale
title: Close epic phase beads by number with `sase bead close -p`
goal:
  "`sase bead close <epic> -p 1,2,3` and `-p 1-3` close those phase beads of an epic, and using `-p` on a non-epic
  target fails with a clear error and no writes."
create_time: 2026-07-29 12:10:52
status: wip
---

- **PROMPT:** [202607/prompts/bead_close_phases.md](prompts/bead_close_phases.md)

# Add `-p|--phases` to `sase bead close`

## Goal

Let one `sase bead close` invocation close several phase beads of the same epic by number instead of by full bead ID.

- `sase bead close sase-at -p 1,2,3` behaves exactly like `sase bead close sase-at.1 sase-at.2 sase-at.3`.
- `-p` also accepts ranges, so `sase bead close sase-at -p 1-3` is the same command again.
- Using `-p` on a target that is not an epic plan bead fails with a clear, actionable error and writes nothing.

The epic bead named as the target is never itself closed by `-p`; it only supplies the ID prefix.

## Background

### Where `sase bead close` lives today

- Parser: `src/sase/main/parser_bead.py:26-52` (`ids` positional plus `-f/--force`, `-n/--note`, `-r/--reason`,
  `-R/--resolution`). Short flags `-p` and `-P` are both free on this subcommand.
- Handler: `handle_bead_close` in `src/sase/bead/cli_crud.py:204-230`, re-exported through `src/sase/bead/cli_basic.py`
  and `src/sase/bead/cli.py`, dispatched from `src/sase/main/entry.py:99`.
- The handler calls `BeadProject.close` (`src/sase/bead/project.py:359`), which delegates to the Rust core through
  `sase.core.bead_mutation_facade.close`, then commits with `require_mutation_commit_message("close", args.ids)`.

### The Rust fast path already defers unknown flags

`sase bead ...` first goes through `try_handle_bead_fast_path` (`src/sase/main/bead_fast_path.py:17`), which calls the
Rust `bead_cli_execute` binding. In sase-core, `handle_close` calls `parse_close_args`
(`crates/sase_core/src/bead/cli.rs:1450`), and that parser returns `None` for **any** unrecognized argument that starts
with `-`. `handle_close` then returns `defer()` (`handled: false`), and `try_handle_bead_fast_path` returns `None`, so
Python's argparse slow path runs the command.

Consequence: `-p`/`--phases` needs **no sase-core change**. The flag makes the whole close invocation fall back to the
Python path, which then performs the same core mutation. The only cost is the fast path's startup saving on this one
rare, interactive invocation. A golden end-to-end test (below) pins that deferral so a future sase-core change cannot
silently break it.

### How phase bead IDs are formed

Children get `<parent_id>.<N>` with `N` allocated sequentially (`BeadProject._next_child_id`,
`src/sase/bead/project.py:541`). `sase bead work` creates one phase bead per `phases:` frontmatter entry in plan order
(`src/sase/bead/epic_from_plan.py:120-133`), so an epic's phases are `<epic>.1 … <epic>.N` in plan order. Frontmatter
phase IDs are slugs, but those slugs are not part of the bead ID.

An "epic bead" is a bead with `issue_type == IssueType.PLAN` and `tier == BeadTier.EPIC`; `sase bead work` uses the same
test and rejects everything else with `sase bead work only applies to epic plan beads (got {tier} for {target})`
(`src/sase/bead/cli_work_entry.py:153,209-214`).

## Design decisions

1. **Numbers and ranges only.** `-p` accepts a comma-separated list whose items are either a phase number (`3`) or an
   inclusive range (`2-5`). Mixed forms (`1-3,5,8-9`) and surrounding whitespace (`-p "1, 3 - 5"`) are accepted. Phase
   _slugs_ from plan frontmatter are explicitly out of scope (see Non-goals).
2. **Exactly one target.** With `-p`, the command takes exactly one positional bead ID (the epic). Two or more
   positionals is a usage error, because "which epic do these phase numbers belong to" would otherwise be ambiguous.
3. **The epic is not closed.** `-p` replaces the positional target in the close set; it never adds the epic to it. This
   matches the documented equivalence and the rule that phase agents never close the parent epic.
4. **Repeatable and normalized.** `-p` may be given more than once; all values merge. The resulting phase numbers are
   deduplicated and sorted ascending, so `-p 3,1,1-2` closes `<epic>.1 <epic>.2 <epic>.3` in that order.
5. **Other flags compose unchanged.** `--force`, `--note`, `--reason`, and `--resolution` apply to the expanded phase
   IDs exactly as they would if the user had typed those IDs.
6. **Output parity.** stdout is byte-identical to the equivalent explicit-ID command (one `✓ Closed: <id> — <title>`
   line per bead). No extra "expanded to ..." chatter.
7. **All-or-nothing.** Every validation happens before the mutation, so a bad `-p` leaves the store untouched, matching
   the existing multi-ID close contract.
8. **Already-closed phases are not special-cased.** Re-closing a closed phase behaves exactly as it does when the ID is
   typed explicitly (the core reports it and records no new close event).

### Error contract

All errors print to stderr with the existing `Error: ` prefix and exit 1, before any store write.

| Situation                              | Message                                                                                                     |
| -------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| More than one positional ID with `-p`  | `Error: --phases takes exactly one epic bead ID (got 2: sase-at, sase-au)`                                  |
| Target bead does not exist             | `Error: issue not found: sase-at` (existing `KeyError` path)                                                |
| Target is not an epic plan bead        | `Error: --phases only applies to epic plan beads (got phase for sase-at.1)`                                 |
| Target is a plan-tier or untiered plan | `Error: --phases only applies to epic plan beads (got plan for sase-at)` / `(got missing tier for sase-at)` |
| Unparseable item                       | `Error: invalid --phases value: 'x' (expected phase numbers or ranges, e.g. 1,3,5-7)`                       |
| Empty item (`-p ""`, `-p 1,,2`)        | `Error: invalid --phases value: '' (expected phase numbers or ranges, e.g. 1,3,5-7)`                        |
| Zero or negative number                | `Error: invalid --phases value: '0' (phase numbers start at 1)`                                             |
| Reversed range                         | `Error: invalid --phases range: '5-3' (start must not exceed end)`                                          |
| Requested phase does not exist         | `Error: epic sase-at has no phase 4, 7 (existing phases: 1, 2, 3)`                                          |
| Epic has no phase beads at all         | `Error: epic sase-at has no phase 1 (epic has no phase beads)`                                              |
| `<epic>.<n>` exists but is not a phase | `Error: sase-at.3 is not a phase bead; close it by ID if that is intended`                                  |

The missing-phase list is truncated after 10 entries with a `(+N more)` suffix, so a fat-fingered `-p 1-100000` prints a
readable error instead of a wall of numbers.

## Implementation

### 1. New module `src/sase/bead/phase_selector.py`

Pure-Python selector parsing plus epic-scoped resolution, kept out of the CLI handler so it is unit-testable without a
store or an `argparse.Namespace`.

- `class PhaseSelectorError(ValueError)` — carries the fully formatted user-facing message (without the `Error: `
  prefix, which the handler adds, matching the other handlers in `cli_crud.py`).
- `def parse_phase_selectors(values: Sequence[str]) -> tuple[int, ...]` — split every value on `,`, strip whitespace,
  parse each item as an integer or an `A-B` inclusive range, validate `>= 1` and `start <= end`, then return the sorted
  deduplicated tuple. Raises `PhaseSelectorError` per the error contract. This function touches no store state.
- `def resolve_epic_phase_ids(project, epic_id, phase_numbers) -> list[str]` — look the target up with `project.show`
  (letting `KeyError` propagate for the existing not-found message), reject non-epic targets, index
  `project.get_epic_children(epic_id)` by the integer suffix of each direct child's ID, require every requested number
  to resolve to an `IssueType.PHASE` bead, and return the IDs in ascending phase order. Raises `PhaseSelectorError` for
  every non-epic / missing / non-phase case.
- Type the `project` parameter as `BeadProject` under `if TYPE_CHECKING:` so the module stays import-light.

### 2. Parser (`src/sase/main/parser_bead.py`)

- Add `-p/--phases` to the close subparser, keeping the flags alphabetically sorted (`-f`, `-n`, `-p`, `-r`, `-R`), with
  `action="append"`, `metavar="SPEC"`, and help text along the lines of
  `Close these phase beads of the target epic; comma-separated numbers and ranges (e.g. 1,3,5-7)`.
- Update the `ids` help to note the `--phases` form: `Issue IDs to close (exactly one epic ID when --phases is used)`.
- Give the close subparser a `description` and an `epilog` of examples, matching the `list`/`search`/`show`/`work`
  subparsers and the "make `--help` excellent" CLI rule. Examples to include:

  ```
  sase bead close sase-at.1 --note "verified with just check"
  sase bead close sase-at -p 1,2,3
  sase bead close sase-at -p 1-3
  sase bead close sase-at -p 1-3,5 --reason "phases landed together"
  ```

  Use `formatter_class=argparse.RawDescriptionHelpFormatter` as the other subparsers with epilogs do. Adding roughly 25
  lines keeps `parser_bead.py` (660 lines today) under the 700-line `toobig` info threshold.

### 3. Handler (`src/sase/bead/cli_crud.py`)

- Add a small module-level helper (e.g. `_resolve_close_ids(args, project) -> list[str]`) that returns `args.ids`
  unchanged when `--phases` was not supplied, and otherwise enforces the single-positional rule, calls
  `parse_phase_selectors`, and returns `resolve_epic_phase_ids(...)`.
- In `handle_bead_close`, resolve the IDs **inside** the existing `bead_store_mutation` block (as `handle_bead_create`
  already does for its parent lookup) so the read and the write share one store lock, and so `sys.exit(1)` on a bad
  selector leaves the mutation uncommitted.
- Catch `PhaseSelectorError` next to the existing `KeyError`/`ValueError` handling, print `Error: {exc}` to stderr, and
  `sys.exit(1)`.
- Pass the resolved IDs to **both** `mutation.project.close(...)` and
  `require_mutation_commit_message("close", resolved_ids)`, so the auto-commit message names the phase beads that were
  actually closed rather than the epic.
- Read the option with `getattr(args, "phases", None)`, matching how this handler already reads `note`, `resolution`,
  and `force`, so existing callers that build a bare `argparse.Namespace` (e.g.
  `tests/test_bead/test_cli_auto_commit.py:261`) keep working.

### 4. Tests

New file `tests/test_bead/test_cli_close_phases.py`:

- Selector parsing (no store): comma lists, ranges, mixed forms, whitespace tolerance, dedup + ascending sort, repeated
  `-p` merging, and one case per parse error in the table above.
- Parser wiring: `create_parser().parse_args(["bead", "close", "x"]).phases is None`, and that repeating `-p` appends.
- End-to-end against the `project_dir` fixture (`tests/test_bead/conftest.py`), following the style of
  `tests/test_bead/test_cli_close_resolution.py`:
  - epic with three phases, `close <epic> -p 1-2` closes exactly `<epic>.1` and `<epic>.2` and leaves the epic and
    `<epic>.3` untouched;
  - `-p` composed with `--note`, `--reason`, and `--resolution superseded` records those on each phase;
  - non-epic targets (a `plan`-tier plan bead and a phase bead) exit 1, print the expected message, and leave every bead
    open;
  - an out-of-range phase number exits 1 and lists the existing phases;
  - two positional IDs plus `-p` exits 1 with the usage message;
  - the auto-commit message names the expanded phase IDs (mirror `tests/test_bead/test_cli_auto_commit.py:280`).

Golden CLI coverage in `tests/test_bead/test_cli_golden.py` (this suite drives real `sase_main()` invocations, so it
exercises the Rust fast path, its deferral, and the argparse slow path together). The existing `current` fixture store
already has `beads-1` (epic, phases `beads-1.1` and `beads-1.2`) and `beads-3` (a `plan`-tier plan bead), so add:

- `close_phases`: `["bead", "close", "beads-1", "-p", "1-2"]` on `current`, with a new
  `tests/test_bead/golden/cli/close_phases.stdout` holding both `✓ Closed:` lines;
- `close_phases_not_epic`: `["bead", "close", "beads-3", "-p", "1"]` on `current`, exit code 1, with a new
  `tests/test_bead/golden/cli/close_phases_not_epic.stderr`.

Fast-path deferral unit coverage in `tests/main/test_bead_fast_path.py`: assert the real `bead_cli_execute` binding
reports `handled == False` for `["close", "beads-1", "-p", "1"]` and `["close", "beads-1", "--phases=1-2"]` (the Rust
parser rejects the unknown flag before touching the store, so temporary directories suffice). This is the regression
guard for the deferral the whole feature rests on.

### 5. Documentation

- `docs/beads.md`, `### sase bead close`: add a short paragraph explaining the epic-scoped `--phases` shorthand, the
  number/range syntax, that the epic itself is not closed, and the not-an-epic error; add a `-p, --phases` row to the
  flag table in alphabetical position.
- `docs/configuration.md`, `#### sase bead close` table: add the matching `-p, --phases` row and note the
  one-epic-ID-only rule in the `ids` row's description.
- `src/sase/xprompts/skills/sase_beads.md`, `### close / open`: add a `sase bead close <epic-id> -p 1-3` example and one
  or two sentences of guidance. **Before editing this file**, use the `/sase_memory_read` skill to read
  `generated_skills.md` and follow whatever regeneration/deploy step it prescribes for bundled skill sources.
- No change is expected in `src/sase/default_config.yml` (its close guidance at line 838 is about `--force` only).
  Confirm rather than assume.
- Run `just fmt` so the Markdown tables and prose stay within the repo's 120-column prose wrap.

## Verification

1. `just install` first (ephemeral workspace may have stale deps), then `just check` — must pass clean, including
   `fmt (markdown)`, `lint (symvision)`, and `lint (toobig)`.
2. `just test tests/test_bead tests/main` for the focused suites, plus the full `just test` before finishing.
3. Manual smoke against a scratch bead store (not the project store): create an epic with three phases, run
   `sase bead close <epic> -p 1,3`, confirm exactly those two phases closed and the epic stayed open, then confirm
   `sase bead close <plan-tier-bead> -p 1` fails with the not-an-epic message and changes nothing.
4. `sase bead close --help` renders the new flag and examples legibly.

## Non-goals

- **Phase slugs.** `-p auth,render` (frontmatter phase IDs) is not implemented; only numbers and ranges are. Slug
  resolution would need the plan file or the description prefix convention and belongs in a follow-up.
- **Rust fast-path parity.** `-p` deliberately runs on the Python slow path via the existing deferral. Teaching
  sase-core's `parse_close_args` about `--phases` is a possible later optimization, not part of this change.
- **Other subcommands.** `sase bead open`, `rm`, and `update` keep taking explicit IDs.

## Risks

- If a future sase-core release starts _handling_ unknown close flags instead of deferring, `-p` would break. The
  `close_phases` golden case and the fast-path deferral assertion both fail loudly in that case.
- Phase numbers are positional bead IDs, not plan slugs. If an epic's phases were created out of plan order (hand-made
  beads), `-p N` still means `<epic>.N`. The docs should say the number is the bead ID suffix, so nobody reads it as
  "the Nth phase in the plan file".
