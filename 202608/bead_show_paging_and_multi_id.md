---
tier: tale
title: Give `sase bead show` color-preserving paging and multi-bead arguments
goal:
  "`sase bead show` accepts one or more bead IDs and renders them in argv order in every
  format, and long output on a terminal is paged through `less -R` with the detail
  block's ANSI styling fully intact, while a single-ID invocation stays byte-identical
  to today in all three formats."
size: medium
proposed_by: bbugyi200.athena.0dv
create_time: 2026-08-25 18:01:17
status: wip
---

# Plan

## Why this shape

Two features, one command surface, one design invariant that keeps them safe:

> **`sase bead show <one-id>` must keep producing today's exact bytes and exit code in
> every format.** Additional IDs extend the output; the pager only changes _where_ the
> same bytes are written.

Everything below follows from that invariant. It is what lets this land without touching
a single existing golden, without reshaping the JSON contract agents already parse
(`src/sase/workflows/commit/bead_hooks.py:261` reads `detail["issue"]` from
`sase bead show <id> --format json`), and without a compatibility flag.

## What the code does today

- `register_bead_show_parser()` (`src/sase/main/parser_bead_queries.py:340`) declares
  one required positional `id` plus `-c/--color`, `-f/--format`, `-N/--no-links`,
  `-s/--style`, `-w/--wrap`.
- `handle_bead_show()` (`src/sase/bead/cli_query.py:260`) opens one read view, resolves
  one bead (`view.show()` for `compact`, `resolve_issue_detail()` otherwise), then
  `print(..., end="")`s one of three renderings. A `KeyError`/`ValueError` prints
  `Error: issue not found: <id>` (or the raw message) to stderr and exits 1.
- `src/sase/main/entry.py:148` dispatches `show` and `sys.exit(0)`s after the handler,
  so a non-zero exit must come from inside the handler.
- `sase bead show` is deliberately excluded from the Rust bead fast path
  (`src/sase/main/bead_fast_path.py:49` returns `None` for `list`/`show`), so all of
  this is Python-only. Nothing here belongs in `../sase-core`: paging is terminal
  presentation, and multi-ID resolution is a loop over the existing single-bead store
  read.
- Styling is already correct and reusable: `DetailPalette`/`DetailStyle`
  (`src/sase/bead/cli_detail_style.py`) emit purely additive ANSI, and
  `resolve_detail_style()` gates on `resolve_color()`, which is `sys.stdout.isatty()`
  under `auto`. **This is exactly why paging must not use a shell pipe**:
  `sase bead show x | less -R` makes stdout a pipe, `resolve_color()` returns False, and
  the color is gone before `less` ever sees it. Rendering to a string first and handing
  it to a child pager keeps `sys.stdout` a TTY at decision time, so `rich` styling
  survives.
- A local, un-shared pager already exists at `src/sase/artifact_cli/read.py:338`
  (`_print_or_page`). It has two defects this plan must not copy: it re-prints the whole
  body when the pager exits non-zero (so quitting `less` early dumps the text again),
  and it offers `bat` as a pager for text that is already ANSI-colored.

## Design

### 1. `-p/--pager {auto,always,never}`, default `auto`

Tri-state, mirroring `-c/--color`, so the two knobs read the same way. Placed between
`--no-links` and `--style` to keep the option list alphabetical, with a short alias, and
not required — per the project's CLI rules.

Resolution order for the pager command: `SASE_PAGER`, then `PAGER`, then a `less` found
on `PATH`. An env value that is empty or whitespace disables paging (the Git-style
"`PAGER=` means no pager" convention). The command string is split with `shlex.split`,
so `SASE_PAGER="less -S"` works.

**`less` is the only auto-detected fallback.** Do not add `bat` or `more`. The entire
point of this feature is ANSI fidelity, and `less -R` is the only widely available pager
that guarantees it; `bat` re-highlights its input and `more` varies by platform. A user
who wants either can still say so through `SASE_PAGER`.

When the resolved program's basename is `less`, append options we require rather than
relying on env (less treats `$LESS` as if prepended to argv, so trailing argv options
win over a hostile `LESS`):

- always append `-R` (pass ANSI through) unless argv already carries
  `-R`/`-r`/`--RAW-CONTROL-CHARS`.
- in `auto` mode only, also append `-F` (quit if it fits) unless already present — a
  cheap second opinion on our own height estimate. Never append `-F` in `always` mode;
  it would defeat the explicit request.
- set `LESS` to `FRX` (auto) / `RX` (always) **only when `LESS` is unset**, matching
  Git's default so short output stays on screen after quitting instead of vanishing with
  an alternate-screen restore.

`auto` pages only when **all** of these hold:

1. `sys.stdout.isatty()`.
2. `TERM` is set and is not `dumb`.
3. `SASE_AGENT` is unset. A SASE agent that gets a TTY would otherwise block forever on
   an interactive pager; this guard is the difference between a nice feature and a hung
   agent turn.
4. A pager command resolves.
5. The rendered text is taller than the terminal: estimated display rows >
   `shutil.get_terminal_size(fallback=(80, 24)).lines - 1`. Estimate rows by stripping
   SGR escapes, measuring with `rich.cells.cell_len` (already a dependency, already used
   by the compact renderer), and counting `max(1, ceil(cells / columns))` per line so a
   long wrapped line is not undercounted.

`always` drops conditions 3 and 5 — an explicit `--pager always` outranks the agent
guard and the height check — but still requires a terminal, a usable `TERM`, and a
resolvable pager. It is not "always" in the sense of piping through a pager into a file:
spawning `less` on a redirect is useless, so a non-TTY still writes directly. When
`always` is asked for and no pager resolves, print one `warning:` line to stderr, then
write directly; `auto` stays silent in that case.

`never` always writes directly.

### 2. Multiple IDs

`id` becomes `ids` with `nargs="+"` and `metavar="ID"`. Zero IDs remains an argparse
usage error (exit 2), exactly as today.

- **Order** is argv order. Never sorted — the user chose the order.
- **Duplicates** are collapsed after resolution, keyed on the canonical `issue.id` from
  the resolved bead, keeping the first position. This is what makes
  `sase bead show sase-64 64` render one bead rather than two, since shorthand and full
  IDs resolve to the same canonical ID.
- **Misses are per-ID and non-fatal to the batch.** Resolve every requested ID; render
  everything that resolved; then, **after the pager exits**, print one
  `Error: issue not found: <id>` line per failure to stderr and `sys.exit(1)`. Printing
  the diagnostics last is the whole reason this ordering is specified: stderr is not
  paged, so a diagnostic printed _before_ the pager scrolls out of sight behind it. With
  exactly one ID, nothing resolves, nothing prints to stdout, and the stderr line plus
  exit 1 are byte-identical to today — the invariant holds with no arity special-case in
  the code.
- **Infrastructure failures still fail the whole command.** The artifact-link
  neighborhood error path (`_with_artifact_link_neighborhood`, `cli_query.py:328`) keeps
  its current behavior: print the error and the `--no-links` hint, exit 1 immediately. A
  malformed link index is not a per-bead miss.

Per format:

- **`full`** — one detail block per bead. With two or more beads each block is preceded
  by a divider carrying its ordinal:

  ```text
  ── 1/3 ─────────────────────────────────────────────────────────────────────────────
  ● sase-64 · Give bead show colored paging   [OPEN]
  ...

  ── 2/3 ─────────────────────────────────────────────────────────────────────────────
  ◐ sase-65 · ...
  ```

  The counter is **left-anchored**, not centered, so the ordinals line up in a single
  column down the left edge — that is what makes a long paged run scannable while
  scrolling. The rule glyphs use `DetailPalette.separator` (dim) and the counter uses
  `DetailPalette.section`, so the divider inherits the existing palette and disappears
  cleanly under `--style plain`. Divider width is the resolved `--wrap` width, falling
  back to `markdown_print_width()` when wrapping is disabled — deterministic, so goldens
  do not depend on the running terminal. Exactly one bead means no divider at all.

- **`compact`** — pass all resolved issues to the existing
  `_render_list_compact(issues, use_color=...)`. It already measures column widths
  across the whole list, so multiple beads come out as one aligned table for free. No
  dividers; a table does not want rules between rows.

- **`json`** — shape follows arity. One ID emits today's single-bead envelope object,
  unchanged, because agents already parse it. Two or more emit a JSON array of those
  same envelope objects, in the same order. An array is the right multi shape (it is
  what `jq` users expect), and the arity rule is the only way to add it without a
  breaking change. Misses are not represented in the array; exit 1 is the signal that
  the array is incomplete.

### 3. Where the code goes

Two new modules, so `cli_query.py` stays a thin handler and the pager is reusable by
future commands:

- **`src/sase/cli_pager.py`** — generic, knows nothing about beads. Public surface:
  `PagerMode` (a `StrEnum` of `auto`/`always`/`never`), `resolve_pager_mode(str)`,
  `resolve_pager_argv() -> list[str] | None`, and `page_or_print(text, *, mode)`.
- **`src/sase/bead/cli_show_batch.py`** — bead-aware batch resolution and assembly:
  resolve an ordered, deduped list of `IssueDetail`s plus an ordered list of failures,
  and assemble the `full`/`compact`/`json` body for one or many beads, including the
  divider.

## Work to do

### 1. `src/sase/cli_pager.py` (new)

`page_or_print(text, *, mode)`:

1. Decide per the rules above; when not paging, write `text` to `sys.stdout` unchanged
   (do not add or strip a trailing newline — callers already pass a body ending in
   `\n`).
2. Launch with
   `subprocess.Popen(argv, stdin=subprocess.PIPE, text=True, encoding="utf-8", errors="replace", env=env)`
   and let stdout/stderr be inherited.
3. **Distinguish "could not start" from "user quit".** Only an `OSError` from `Popen`
   falls back to a direct write. A `BrokenPipeError` while writing, or any non-zero exit
   status, means the user quit the pager early — swallow it and write **nothing** more.
   This is the bug in `artifact_cli/read.py` that must not be reproduced: re-dumping the
   body after someone presses `q` is worse than not paging at all.
4. Close `stdin` (tolerating `BrokenPipeError` again) and `wait()` before returning, so
   the pager owns the terminal until the user leaves it.
5. Set `SIGINT` to `SIG_IGN` for the pager's lifetime and restore the previous handler
   in a `finally`, so Ctrl-C inside `less` is handled by `less` instead of raising
   `KeyboardInterrupt` through the CLI. Wrap the `signal.signal` calls so a
   non-main-thread caller degrades (it raises `ValueError` off the main thread) rather
   than crashing.

### 2. `src/sase/bead/cli_show_batch.py` (new)

- A small frozen result type carrying the ordered resolved details and the ordered
  failures (each failure being the requested ID plus the message to print).
- A resolver that, for one open read view, walks the requested IDs in order; uses
  `view.show()` for `compact` and `resolve_issue_detail(...)` otherwise; catches
  `KeyError` → `issue not found: <id>` and `ValueError` → the exception's own message,
  exactly as the current handler formats them; and drops a bead whose canonical ID was
  already emitted.
- `show_divider(index, total, *, palette, width)` producing `"── {i}/{n} " + "─" * fill`
  at exactly `width` columns.
- Body assembly per format. For `full` with n > 1:
  `"\n\n".join(f"{divider}\n{block.rstrip(chr(10))}")` + a trailing newline; for n == 1
  return the block untouched. For `compact`, delegate to `_render_list_compact`. For
  `json`, one envelope object for n == 1 and `json.dumps([...], indent=2) + "\n"` for
  n > 1.
- Resolve `artifact_reference_context()` at most once per batch, lazily, the first time
  a bead in the batch has `refs` — today's handler computes it per call, and
  re-resolving it per bead would be a real regression on a long batch.

### 3. `src/sase/bead/cli_detail_json.py`

Extract the envelope construction from `render_issue_detail_json()` into a
dict-returning
`issue_detail_wire_dict(detail, *, created_by_url, page_url, include_links)`, and have
`render_issue_detail_json()` become
`json.dumps(issue_detail_wire_dict(...), indent=2) + "\n"`. The batch module builds its
array from the dict builder. Do **not** duplicate the envelope shape in a second place —
the two renderings drifting apart is precisely the failure mode this module's existing
docstrings warn about.

### 4. `src/sase/bead/cli_query.py`

Rewrite `handle_bead_show()` as thin orchestration: read `args.ids`, resolve style,
wrap, and pager mode; open one read view; call the batch resolver and body assembler;
close the view; `page_or_print(body, mode=...)` when the body is non-empty; print each
failure to stderr; `sys.exit(1)` when there were failures. Keep
`_with_artifact_link_neighborhood` and its immediate-exit error path as-is, applied per
resolved bead.

### 5. `src/sase/main/parser_bead_queries.py`

- Positional `id` → `ids`, `nargs="+"`, `metavar="ID"`, help
  `"Full or shorthand issue IDs to show (rendered in the order given)"`.
- Add `-p/--pager` with `choices=["auto", "always", "never"]`, `default="auto"`, placed
  between `--no-links` and `--style`.
- Update `help=` to `"Show one or more issues"` (matching `sase bead close`'s "Close one
  or more issues" phrasing).
- Update `description=` to say the command renders every listed bead in the order given,
  that duplicate IDs collapse, that a missing ID reports on stderr and exits 1 without
  suppressing the beads that did resolve, that `--format json` emits one envelope for
  one ID and an array for two or more, and that long output on a terminal is paged with
  color intact.
- Extend the epilog examples with `sase bead show sase-64 sase-65 sase-at.1` and
  `sase bead show sase-64 --pager always`, keeping the example list sorted the way it is
  now.

### 6. `src/sase/completion/kinds.py`

`_BEAD_ID_SLOTS`: `(("bead", "show"), "id")` → `(("bead", "show"), "ids")`. The entry
stays in its current sorted position. Repeated-value completion then works the same way
it already does for `bead close`/`rm`/`snooze`/`update`, which are the existing
`nargs="+"` bead-ID slots.

### 7. Tests

**New `tests/test_cli_pager.py`** — the generic module, with no bead involvement:

- `SASE_PAGER` wins over `PAGER`; `PAGER` wins over a `PATH` `less`; an empty or
  whitespace value disables paging; a value with arguments is `shlex`-split.
- `-R` is appended for `less` and not duplicated when already present; `-F` is appended
  in `auto` and absent in `always`; `LESS` is defaulted only when unset.
- `auto` writes directly when: stdout is not a TTY; `TERM` is unset/`dumb`; `SASE_AGENT`
  is set; the text fits the terminal; no pager resolves.
- `auto` pages when the text is taller than the terminal on a TTY.
- `always` pages text that fits, still writes directly without a TTY, and emits exactly
  one `warning:` line when no pager resolves (while `auto` emits none).
- The row estimate ignores SGR escapes and counts a line wider than the terminal as more
  than one row.
- A fake pager that exits before reading (raising `BrokenPipeError` into the writer) and
  a fake pager that exits non-zero both produce **no** direct write — the regression
  guard for the `artifact_cli/read.py` defect.
- `OSError` from `Popen` falls back to a direct write.
- The prior `SIGINT` handler is restored after a paged run, including when the pager
  fails.

**New `tests/test_bead/test_cli_show_multi.py`** — batching, reusing the existing
`cli_show_style_test_helpers` multi-issue view and `strip_sgr`:

- The parser yields `args.ids == [...]` for one and for several IDs.
- `full` with one bead is byte-identical to the pre-change rendering (no divider).
- `full` with three beads emits `── 1/3 `, `── 2/3 `, `── 3/3 ` dividers in argv order,
  each exactly the resolved wrap width, and `strip_sgr(rich) == plain` still holds
  across the whole batch.
- Argv order is preserved and not sorted; a full ID and its shorthand for the same bead
  collapse to one block.
- `compact` with three beads emits three aligned rows in argv order.
- `json` with one ID parses to a `dict` with `issue`; with two IDs parses to a `list` of
  two envelopes whose `issue.id`s match argv order; single-ID bytes are unchanged.
- One missing ID among several: the resolved beads are on stdout, stderr has exactly one
  `Error: issue not found:` line, and the exit code is 1.
- A single missing ID: empty stdout, one stderr line, exit 1 (the unchanged path).
- All IDs missing: empty stdout, one stderr line each, exit 1.
- `--no-links` still suppresses link sections for every bead in the batch.

**New `tests/test_bead/test_cli_show_pager.py`** — the wiring only, not the pager
internals: `--pager never` writes straight to stdout for one and for many beads (so the
rest of the suite's `capsys` assertions stay valid), `--pager always` on a stubbed TTY
hands the fully assembled body — dividers included — to the pager exactly once, and the
missing-ID stderr lines are emitted **after** the pager returns.

**Updates to existing tests** (mechanical, `args.id` → `args.ids`):

- `tests/main/test_parser_narrowing.py:60` — `assert args.id == "sase-1"` becomes the
  list form.
- `tests/completion/test_kinds.py` — `test_path_override_wins_over_name_table` and
  `test_bead_show_id_path_override_present` move to the `ids` dest.
- `tests/completion/test_build.py::test_kind_resolution_precedence_on_real_tree` —
  `p.dest == "id"` becomes `"ids"`.

Everything else under `tests/test_bead/test_cli_show*.py` goes through
`create_parser().parse_args([...])` and must keep passing untouched. If any of them
needs editing, the invariant has been broken — fix the code, not the test.

### 8. Docs

- `docs/beads.md`: rename the `### sase bead show <id>` heading to
  `### sase bead show <id> [<id2> ...]`, matching the `sase bead snooze` heading style.
  **This changes the anchor**, so update the two in-page links that point at
  `#sase-bead-show-id` (`docs/beads.md:1196` and `docs/beads.md:1512`) to
  `#sase-bead-show-id-id2`. Document multi-bead ordering, dedupe, the divider, the
  per-format multi behavior, the arity-shaped JSON, the stderr-after-output miss
  reporting and exit 1, and the full `--pager` semantics including `SASE_PAGER`/`PAGER`,
  the `SASE_AGENT` guard, and why paging preserves color where a shell pipe does not.
  Add `-p, --pager` to the flag table between `--no-links` and `--style`.
- `docs/cli.md:198`: the `sase bead show` row summary becomes something like "Show one
  or more issues; long terminal output pages with color intact."

## Verification

- `just install` first — a numbered workspace may be stale.
- `.venv/bin/python -m pytest tests/test_cli_pager.py tests/test_bead/ tests/completion/ tests/main/test_parser_narrowing.py tests/test_commit_bead_hooks.py -q`
- Manual smoke on a real terminal, since the pager is the one thing tests can only
  simulate:
  - `sase bead show <a-long-bead>` pages, and the detail block is still colored inside
    `less`.
  - `q` exits without re-dumping the body.
  - Ctrl-C inside the pager does not produce a traceback.
  - `sase bead show <a> <b> <c>` shows left-aligned `1/3`, `2/3`, `3/3` dividers.
  - `sase bead show <a> | cat` is uncolored, unpaged, and byte-identical to before.
  - `sase bead show <a> <bogus>` prints `<a>`, then the error line last, and `echo $?`
    is 1.
- `just check` inline.
- `just check-full` through `/sase_monitor` (never inline), using the `TESTING`/`TESTED`
  status pair.

## Non-goals

- Do not add a `pager:` block to `src/sase/default_config.yml`. `SASE_PAGER`/`PAGER` is
  the established, portable way to express a durable pager preference, and a config knob
  would need schema, merge, and docs work for no behavior a user cannot already get.
- Do not add paging or multi-ID support to `sase bead list`, `sase bead search`,
  `sase plan show`, or any other command here. `src/sase/cli_pager.py` is written to be
  reusable, but adopting it elsewhere is separate work with its own goldens.
- Do not migrate `src/sase/artifact_cli/read.py` onto the new module in this change; see
  the follow-up below.
- Do not change the single-bead JSON envelope, the compact row, or any existing golden.
- Do not touch `../sase-core` or the Rust bead fast path; `show` is excluded from it by
  design.
- Do not hand-edit `CHANGELOG.md` — it is generated by release-please. Describe the
  change in a `feat:` commit subject and body.
- Do not modify `sase/memory/*.md`, `AGENTS.md`, or the generated provider shims.

## Follow-up filed separately

`src/sase/artifact_cli/read.py:338` `_print_or_page` duplicates pager logic and carries
two defects the new module fixes: it re-prints the entire body when the pager exits
non-zero (so quitting `less` early dumps the text a second time), and it offers `bat` as
a pager for already-ANSI-colored text. Converging it onto `src/sase/cli_pager.py`
changes `sase artifact read`'s observable behavior and belongs to that command's own
change. File it with `/sase_new_task` rather than smuggling it into this one.
