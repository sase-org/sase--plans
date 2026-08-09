---
tier: epic
title: Opt-in regex mode for sase bead search
goal: '`sase bead search -e/--regex <pattern>` matches beads with a regular expression
  on both the Rust fast path and the Python fallback, while the default literal search
  keeps its current substring semantics and compiles no regex at all.

  '
phases:
- id: core
  title: Rust core regex matcher and fast-path flag
  depends_on: []
  size: medium
  description: 'core: add a shared bead query matcher to sase-core, thread an opt-in
    `regex` argument through `search_issues`/`search_issues_in_issues` and the `bead_search`
    PyO3 binding, teach the Rust bead CLI to parse `-e`/`--regex` and to highlight
    and snippet regex matches, and cover all of it with Rust tests.

    '
- id: floor
  title: Adopt the released core in the sase dependency floor
  depends_on:
  - core
  size: small
  description: 'floor: after sase-core publishes the release that carries the regex
    binding, raise the `sase-core-rs` minimum in pyproject.toml, refresh uv.lock,
    and confirm the published-core minimum smoke tooling accepts the new floor.

    '
- id: cli
  title: Python CLI flag, rendering, tests, and docs
  depends_on:
  - floor
  size: medium
  description: 'cli: add the `-e`/`--regex` option to the `sase bead search` argparse
    parser, plumb it through the bead read facade and BeadProject, handle invalid
    patterns and regex snippet selection in the Python renderer, and update the bead
    search tests and documentation.'
proposed_by: bbugyi200.athena.w8
create_time: 2026-08-09 07:40:29
status: done
bead_id: sase-i1
---

- **PROMPT:** [prompts/202608/bead_search_regex.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/bead_search_regex.md)
- **BEAD:** [sase-i1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-i1/README.md)

# Plan: Opt-in regex mode for `sase bead search`

## Problem

`sase bead search <query>` only does case-insensitive literal substring matching. There
is no way to search beads with a regular expression, which makes it hard to answer
questions like "which beads reference a `sase-g<number>` ID" or "which beads mention
either `Patch` or `ChangeSpec` at the start of a line".

Regex matching must be **opt-in behind a CLI flag** so the default literal path stays as
fast as it is today: no pattern compilation, no per-field regex engine dispatch, and no
change to the literal matching code that runs for every plain search.

## Where the behavior lives (read this first)

`sase bead search` has two lanes and both must gain regex support, because either one
can serve a given invocation:

1. **Rust fast path (the common lane).** `src/sase/main/entry.py` calls
   `try_handle_bead_fast_path()` in `src/sase/main/bead_fast_path.py`, which hands raw
   `argv` to the `bead_cli_execute` binding. `crates/sase_core/src/bead/cli.rs` parses
   the arguments itself, runs the search, and renders the output. This lane handles
   `--format compact` and `--format json`.
2. **Python fallback lane.** `--format full`, `--help`, and any argument the Rust parser
   does not understand `Defer` back to argparse, which reaches `handle_bead_search()` in
   `src/sase/bead/cli_query.py`. That handler still gets its matches from Rust:
   `BeadProject.search()` (`src/sase/bead/_project_queries.py`) calls
   `bead_read_facade.search()` (`src/sase/core/bead_read_facade.py`), which calls the
   `bead_search` PyO3 binding.

Both lanes therefore bottom out in
`crates/sase_core/src/bead/search.rs::search_issues_in_issues`, which is why the
matching change belongs in sase-core and is phase `core`. The Python side only adds the
flag, plumbs it through, and renders.

Open the Rust repo with `sase repo open sase-core -r "<reason>"` and work only in the
path that command prints. Do not clone or web-fetch it any other way.

## Design decisions (binding for all phases)

- **Flag:** `-e` / `--regex`, a boolean store-true option on `sase bead search` only.
  `-e` is free on this subcommand (`-c`, `-f`, `-n`, `-s`, `-t` and `-r` for `--tier`
  are taken). Per the project CLI rules, every public long option gets a short alias and
  the option list stays alphabetically sorted, so `--regex` is registered between
  `--limit` and `--status`.
- **Literal path is untouched.** In literal mode the matcher keeps doing exactly what it
  does today: `field.value.to_lowercase().contains(&query.to_lowercase())`, with the
  query lowercased once per search. No `Regex` is constructed and no regex crate code
  runs. This is the performance requirement from the request, and it is protected by a
  test asserting that a literal query containing regex metacharacters matches literally.
- **Unanchored semantics.** Regex mode uses "find anywhere in the field" (`is_match`),
  mirroring substring behavior, so `-e auth` behaves like plain `auth`.
- **Case-insensitive by default.** The pattern is compiled with
  `RegexBuilder::new(pattern).case_insensitive(true)`, matching the literal mode's
  case-insensitivity. Users opt back into case sensitivity with an inline `(?-i)`.
- **No implicit multiline flag.** `^`/`$` anchor the whole field value and `.` does not
  cross newlines unless the user writes `(?m)`/`(?s)`. Several searched fields (`refs`,
  `plus_one_evidence`, `close_history`) are newline-joined blobs; documenting the
  default is enough, no special casing.
- **Empty-query guard is unchanged.** `query.trim().is_empty()` stays a validation error
  in both modes. A whitespace-only pattern is degenerate, and keeping one guard avoids
  forking behavior between modes.
- **Invalid patterns are usage errors.** A pattern the regex engine rejects produces
  `Error: invalid search regex: <engine message>` on stderr with exit code 2, from both
  lanes.
- **Rust owns pattern validation.** The Python lane does not re-validate the pattern; it
  surfaces the Rust validation error. Python's own regex dialect is only used
  best-effort for snippet line selection (see phase `cli`), because Rust and Python
  regex syntax are not identical (for example a leading global `(?-i)` is valid in Rust
  and a syntax error in Python `re`).
- **JSON envelope gains one additive key.** `render_search_json` in Rust and
  `_render_search_json` in Python both emit `"regex": <bool>` immediately after
  `"query"`, so a machine consumer knows how to interpret the echoed query. Nothing
  existing is renamed or removed.

## Explicitly out of scope

- `sase plan search`, the ACE TUI query language (`src/sase/ace/query/`), and every
  other search surface. Only `sase bead search` changes.
- A pre-existing snippet-parity gap between the two lanes: Rust's `single_line_snippet`
  always takes the **first** line of a multi-line field, while Python's
  `_single_line_snippet` takes the **matching** line (which is what `docs/beads.md`
  documents). Do not fix that here. Keep each lane's existing line-selection behavior
  and only make it regex-aware. A separate task bead should track the parity fix.

## Rust core regex matcher and fast-path flag

Work in the sase-core checkout. `crates/sase_core/Cargo.toml` already depends on the
workspace `regex` crate, so no dependency change is needed.

### `crates/sase_core/src/bead/search.rs`

1. Add a shared matcher type, e.g.:

   ```rust
   pub(crate) enum BeadQueryMatcher {
       Literal { pattern: String, needle: String },
       Regex { pattern: String, regex: Box<Regex> },
   }
   ```

   with:
   - `pub(crate) fn new(query: &str, regex: bool) -> Result<Self, BeadError>` —
     validates the empty/whitespace query first (same message,
     `"search query cannot be empty"`), then either lowercases the needle once or
     compiles the pattern with `RegexBuilder::new(query).case_insensitive(true)` and a
     bounded `size_limit` so a pathological pattern cannot allocate without bound. A
     compile failure returns
     `BeadError::validation(format!("invalid search regex: {err}"))`.
   - `pub(crate) fn matches(&self, value: &str) -> bool` —
     `value.to_lowercase().contains(needle)` for `Literal`, `regex.is_match(value)` for
     `Regex`.
   - `pub(crate) fn byte_ranges(&self, text: &str) -> Vec<(usize, usize)>` — the
     highlight/snippet ranges. `Literal` reuses the existing case-folded offset walk
     that currently lives in `cli.rs` as `case_insensitive_byte_ranges`; move that
     helper (and `folded_with_offsets`) here so both callers share one implementation.
     `Regex` uses `find_iter` and **skips zero-length matches** so patterns such as `a*`
     or `\b` cannot emit empty highlight spans.
   - `pub(crate) fn pattern(&self) -> &str` — the original query text, for the
     `No beads match "<query>".` message and the JSON envelope.

2. Add `regex: bool` as the **last** parameter of `search_issues` and
   `search_issues_in_issues`. Both build a `BeadQueryMatcher` once, before the loop, and
   `matched_field_names` takes `&BeadQueryMatcher` instead of `&str`. Also add a
   `pub(crate) fn search_issues_with_matcher(issues, matcher, statuses, issue_types, tiers, limit)`
   entry point so `cli.rs` can compile the pattern once and reuse the same matcher for
   rendering instead of compiling it twice.

3. Preserve the existing ordering and limit semantics exactly: candidates still come
   from `list_issues_in_issues`, are still iterated in reverse so newest matches come
   first, and `limit == 0` still means unlimited.

### `crates/sase_core/src/bead/cli.rs`

1. `SearchArgs` gains a `regex: bool` field. `parse_search_args` accepts `-e` and
   `--regex` as a bare boolean flag. Every other `-`-prefixed token still returns
   `Defer`, so `--regex=true` and a pattern that begins with `-` keep deferring to
   argparse (which reports them the way it does today); mention the `--` separator in
   the Python-side docs rather than special-casing it here.
2. `handle_search` builds the matcher once via
   `BeadQueryMatcher::new(&search_args.query, search_args.regex)`, maps a validation
   error to the existing `usage_error(format!("Error: {}\n", err.message))` path, and
   passes the matcher to `search_issues_with_matcher` and to the renderers.
3. `render_search_compact`, `compact_snippet`, `single_line_snippet`, and the highlight
   helper take `&BeadQueryMatcher` in place of `query: &str`, using
   `matcher.byte_ranges(...)` for highlight spans and match centering, and
   `matcher.pattern()` for the no-match message. Delete the now-duplicated
   `case_insensitive_byte_ranges`/`folded_with_offsets` bodies from `cli.rs` in favor of
   the shared ones.
4. `render_search_json` emits the new `regex` field right after `query`.

### `crates/sase_core_py/src/lib.rs`

Extend `py_bead_search`'s signature to
`(beads_dir, query, statuses=None, issue_types=None, tiers=None, limit=None, regex=false)`
and forward `regex` to `core_bead_search_issues`. Keeping `regex` last with a default
means existing positional callers are unaffected and the Python facade can pass it as a
keyword.

### Rust tests

Add to the `search.rs` and `cli.rs` test modules:

- Literal mode treats metacharacters literally: query `a.c` matches a title containing
  `a.c` and does **not** match `abc`. This is the regression guard for "regex is
  opt-in".
- Regex mode matches an anchored pattern (`^Needle`), an alternation, and a character
  class across at least a title and a newline-joined field.
- Regex mode is case-insensitive by default and case-sensitive with a leading `(?-i)`.
- An invalid pattern returns a `validation` `BeadError` whose message starts with
  `invalid search regex:`, and the CLI renders it as a usage error.
- Empty and whitespace-only queries are still rejected in both modes.
- Newest-first ordering and `--limit` behave identically in regex mode.
- `parse_search_args` accepts `-e` and `--regex` alongside the other flags and still
  defers on unknown flags.
- Compact rendering highlights every regex match in a row, and a zero-width-capable
  pattern neither panics nor emits empty highlight spans.
- The JSON envelope carries `regex` for both `true` and `false`.

### Verification

From the sase-core checkout: `cargo fmt --all -- --check`,
`cargo clippy --workspace --all-targets -- -D warnings`, and `cargo test --workspace` —
the three gates its CI runs.

Commit as a `feat(core):` change so release-plz picks it up. Do **not** hand-edit any
crate version: release-plz owns versions in that repo.

## Adopt the released core in the sase dependency floor

This phase is a small, mechanical dependency bump in the sase repo, and it exists as its
own phase because it cannot start until sase-core has actually published the release
that carries the new binding argument.

1. Confirm the `core` phase's commit is on sase-core `master`, that release-plz's
   release PR has merged, and that the resulting version is on PyPI. sase-core's
   release-plz config sets `features_always_increment_minor = false`, so a non-breaking
   `feat` on a `0.x` line bumps the patch component: expect `0.21.2`. Verify the real
   published version rather than assuming it.
2. Raise the floor in `pyproject.toml` from `sase-core-rs>=0.21.1,<0.22.0` to
   `sase-core-rs>=<published version>,<0.22.0`. The `<0.22.0` ceiling already admits a
   patch release, so only the floor moves.
3. Refresh `uv.lock` and reinstall (`just install`). `just install` builds the extension
   from the local sase-core checkout and **fails when that checkout is behind the
   pyproject floor**, so make sure the checkout is updated to the released commit first.
4. Sanity-check the published wheel really exposes the new argument, in a throwaway
   venv: install `sase-core-rs==<published version>` and call
   `sase_core_rs.bead_search(<beads dir>, "needle", regex=True)` against a temporary
   bead store. This is the check that the repo's existing
   `tools/check_sase_core_rs_bindings` gate cannot make, because it verifies binding
   _names_ only, not signatures.
5. Also run `python3 tools/smoke_sase_core_rs_telemetry --print-minimum pyproject.toml`
   and confirm it reports the new minimum, since CI's `published-core-minimum-smoke` job
   installs exactly that version.
6. Run `just check`.

Landing the floor bump before the CLI flag is deliberate: it removes any window in which
sase `master` could ship a `--regex` flag that a published `sase-core-rs` install cannot
serve, which is the exact class of skew that `tools/check_sase_core_rs_bindings` was
written for.

## Python CLI flag, rendering, tests, and docs

All of this is in the sase repo, and it must run against a core build that already has
the `regex` argument (guaranteed by the `floor` phase). Run `just install` first: sase
workspace directories are ephemeral and may hold a stale virtualenv.

### `src/sase/main/parser_bead_queries.py`

In `register_bead_search_parser`:

- Add `parser.add_argument("-e", "--regex", action="store_true", help=...)`, placed
  between the `--limit` and `--status` registrations so the option list stays
  alphabetical.
- Update `description`: it currently says the query is a "literal query string". It
  should say that the query is a literal, case-insensitive substring by default, and
  that `--regex` interprets it as a regular expression that is case-insensitive unless
  the pattern starts with `(?-i)`.
- Add an epilog example, e.g. `sase bead search '^sase-g' --regex`.

### Plumbing

- `src/sase/core/bead_read_facade.py`: `search()` takes `regex: bool = False` and passes
  it to the binding **as a keyword** (`binding(..., regex=regex)`), keeping the existing
  positional arguments untouched.
- `src/sase/bead/_project_queries.py`: `BeadProject.search()` takes
  `regex: bool = False` and forwards it.
- `src/sase/bead/cli_query.py`: `handle_bead_search` passes `regex=args.regex`.

### Error handling and rendering in `src/sase/bead/cli_query.py`

- Wrap the `view.search(...)` call so a `ValueError` from the binding whose message
  names an invalid regex is printed to stderr as `Error: <message>` and exits with code
  2 — the same text and exit code the Rust lane produces. The existing empty-query guard
  at the top of the handler stays as it is.
- Snippet selection is the only place Python inspects the query itself.
  `_compact_snippet` and `_single_line_snippet` currently do
  `lowered_query in line.lower()`. Introduce a tiny module-local helper that returns a
  callable predicate:
  - literal mode: today's `lowered_query in line.lower()`;
  - regex mode: `re.compile(query, re.IGNORECASE).search`, and if `re.error` is raised
    (Rust-only syntax that Python's `re` rejects), fall back to "always false" so the
    renderer degrades to its existing first-line behavior instead of crashing. Rendering
    must never be the thing that fails a search whose matching already succeeded.
- `_render_search_json` emits `"regex": regex` immediately after `"query"`, matching the
  Rust envelope.
- Python's compact renderer does not ANSI-highlight matches today; do not add
  highlighting here. Only the Rust lane highlights, and phase `core` covers it.

`src/sase/main/bead_fast_path.py` needs no change: it only inspects `--format` to decide
whether to defer, and `--regex` does not affect that.

### Tests

- `tests/test_bead/test_cli_search.py`: `args.regex` defaults to `False` and is `True`
  for both `-e` and `--regex`; a regex search finds beads an equivalent literal search
  cannot; a literal search with metacharacters (`a.c`) does not match `abc`; an invalid
  pattern exits 2 with `Error: invalid search regex: ...`; the JSON envelope carries
  `regex`; a regex-mode compact snippet picks the matching line of a multi-line
  description.
- `tests/main/test_bead_fast_path.py`:
  `try_handle_bead_fast_path(["search", "x", "--regex"])` still routes through the Rust
  executor (it is not deferred), and `--format full` with `--regex` still defers.
- `tests/test_bead/test_project_rust_delegation.py` and
  `tests/test_core_facade/test_bead_read.py`: the `regex` argument is forwarded through
  `BeadProject.search` and the facade.

### Documentation

- `docs/beads.md`, section `### sase bead search <query>`: the sentence "This is
  substring search, not regex or glob matching" is now wrong. Replace it with the
  literal default plus the `--regex` opt-in, the case-insensitivity rule and the `(?-i)`
  override, the fact that `^`/`$` anchor the whole field unless `(?m)` is used, and a
  note that a pattern starting with `-` needs `--` (e.g.
  `sase bead search --regex -- '-x'`). Add
  `| -e, --regex | flag | Interpret the query as a regular expression |` to the flag
  table, keeping the table's existing ordering convention, and add a
  `sase bead search '^sase-g' --regex` example to the fenced block.
- `docs/cli.md`, the `sase bead search` row: extend the one-line description to mention
  the opt-in regex mode.

### Verification

`just install` then `just check`. Because this change touches the bead CLI, the parser,
and the core facade, finish with `just check-full` before landing.
