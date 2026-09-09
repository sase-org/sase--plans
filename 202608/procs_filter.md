---
tier: epic
status: done
title: Procs tab query filtering
goal: "The Admin Center Procs tab has a slash-revealed query bar backed by a real,
  shared query dialect: free text matches a proc's command and output, closed
  `key:value` filters cover monitor/running/status/runtime/completion-time, every term
  negates with `-`, boolean keys have a bare shorthand, and `m` cycles the monitor
  filter on, inverted, and off.

  "
phases:
  - id: grammar
    title: Bare boolean flags and host bound keys in the shared flat grammar
    depends_on: []
    size: medium
    description:
      "grammar: teach the shared profile-driven flat query grammar two closed,
      digest-stable extensions -- a bare `key` shorthand for boolean fields and a host
      registry of directional date/duration bound keys -- across the flat parser,
      canonicalizer, value normalizer, evaluator, and highlighter."
  - id: dialect
    title: Procs query profile and row adapter
    depends_on:
      - grammar
    size: medium
    description:
      "dialect: author the `procs` query schema, the `ObservedProc` -> query-row adapter
      with its bounded, version-keyed searchable-text cache, and a small pure facade the
      pane (and any future CLI) calls to parse and evaluate a proc query."
  - id: bar
    title: Procs filter bar widget and Admin Center key integration
    depends_on:
      - dialect
    size: small
    description:
      "bar: add the `ProcsFilterBar` FilterBar subclass, a shared show-while-active
      resting mode, bare-flag completion candidates, an Admin Center priority-Tab
      hand-off hook, and the pane's filter-bar styling."
  - id: pane
    title: Procs pane filter session
    depends_on:
      - bar
    size: medium
    description:
      "pane: wire `/` into the Procs pane -- a filter-session mixin that filters the
      rendered rows, keeps selection and jump hints stable, refreshes on output churn,
      honors `limit:`, and reports match counts, errors, and the no-match empty state."
  - id: monitor
    title: The `m` monitor-filter cycle
    depends_on:
      - pane
    size: small
    description:
      "monitor: add a shared quote-aware flag-token toggle helper and bind `m` to cycle
      the monitor filter through on, inverted, and off, opening and dismissing the query
      bar as the query gains and loses its last term."
  - id: rust
    title: Mirror the shared grammar extensions in sase-core
    depends_on:
      - grammar
    size: medium
    description:
      "rust: port the bare-boolean shorthand and the host bound-key registry into the
      `sase-core` Rust flat parser, canonicalizer, and evaluator so both implementations
      of the shared grammar stay byte-identical."
  - id: polish
    title: Documentation, visual snapshot, and copy review
    depends_on:
      - monitor
    size: small
    description:
      "polish: document the proc query dialect in the ACE Procs Tab reference, add a PNG
      snapshot of the filtered pane, and do a final pass over hint text, placeholder
      copy, and error wording."
proposed_by: bbugyi200.athena.0bh
bead_id: sase-s9
create_time: 2026-09-09 19:51:10
---

- **PROMPT:**
  [prompts/202608/procs_filter.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/procs_filter.md)
- **BEAD:**
  [sase-s9](https://github.com/sase-org/sase--beads/blob/main/pages/sase-s9/README.md)

# Plan: Procs tab query filtering

## Goal

Filtering the **Procs** tab of the SASE Admin Center should feel like every other query
surface in SASE: press `/`, a query bar appears at the top of the pane, matches narrow
live as you type, `Enter` commits, `Esc` restores. The dialect it speaks is proc-shaped
— free text searches the command string and its output, `monitor`, `running`, `min:`,
`max:`, `before:`, and `after:` cover the questions you actually ask about background
work, `-` negates anything, and a boolean key can be written bare.

The example from the request must work exactly as written:

```
"just check" -monitor -min:300
```

— non-monitor procs that ran for less than 300 seconds and mention `just check` in their
command or their output.

## Design

### Reuse, do not reinvent

This repo already has a profile-driven query stack, and the Procs tab should join it
rather than grow a fourth grammar:

- `sase.filter_tokens` — the lexer. It already implements exactly the token shape the
  request describes: whitespace-separated tokens, double quotes that preserve spaces,
  and a leading `-` that negates a keyed token _or_ a wholly quoted free-text token.
- `sase.ace.query_profile` — `ArtifactQuerySchema` / `CompiledQueryProfile`: a pane
  declares its fields (kind, filterable, searchable, repeatable, negatable, exact,
  static values, hint) and the host compiles them into an immutable, digest-stable
  profile.
- `sase.ace.query.profile_reference_flat` — the flat-dialect parser and canonicalizer
  that turns tokens into the shared `QueryExpr` AST.
- `sase.ace.query.profile_evaluator` — coerces a mapping row into a typed
  `ArtifactQueryRow` and evaluates a `QueryExpr` against it.
- `sase.ace.query.profile_highlighting` — highlights a flat query by mirroring the
  parser's branch order, so the colors can never disagree with the parse.
- `sase.ace.tui.widgets.filter_bar.FilterBar` — the slash-style inline editor with
  completion, sigil, live status lane, error rendering, and a highlighted closed
  display. It is already profile-configurable (`profile=` in `__init__`), so the Procs
  bar is a subclass plus a schema.

The two other query languages in the tree are deliberately _not_ reused:
`sase.ace.query` (the boolean Patch dialect, with `AND`/`OR`/`NOT`/parens and sigils)
and `sase.ace.agent_query` (the Agents tab dialect, with `!`-negation and `age>2h`
comparisons). Both negate with `!`/`NOT`, not `-`, and neither has a bare-boolean
shorthand — the request's grammar is the _flat_ grammar, so the flat grammar is what
this epic extends.

### Two closed extensions to the shared flat grammar

Everything the request asks for is expressible today except two things. Both are added
once, in the host, for every flat pane — not bolted onto Procs.

**1. Bare boolean flags.** A token with no unquoted colon whose case-folded text is a
declared, filterable `bool` field expands to `key:true`. The leading `-` keeps its
ordinary meaning, so `-monitor` is exactly `-monitor:true`. Quoting opts out:
`"monitor"` stays free text. Because the rule is derived entirely from fields the
profile already declares, it needs **no schema field and no wire change**, so no
compiled profile's digest moves and no saved query is invalidated.

- Canonical form is always the long spelling (`monitor:true` / `-monitor:true`), so the
  shorthand is purely lexical and cache keys stay stable.
- The existing "a non-repeatable key may only appear once" rule then makes
  `monitor -monitor` a clean parse error rather than a silently empty result.

_Known, intended side effect:_ the Stitches pane declares a `sidecar` boolean, so a bare
`sidecar` token there stops being free text and starts meaning `sidecar:true`. That is
the correct behavior for a declared boolean flag; it gets a regression test and a line
in the docs.

**2. Host bound keys.** Direction for typed comparisons is currently decided by
hard-coded key names (`since` ⇒ `>=`, `until` ⇒ `<=`) in both the Python evaluator and
the Rust one. Replace that with a closed, host-owned registry in
`sase.ace.query_profile.registry`, alongside the existing `HOST_SIGIL_CHARS` /
`HOST_PREDICATES` vocabularies:

```python
HOST_DATE_BOUND_KEYS: dict[str, str] = {
    "since": ">=", "after": ">=", "until": "<=", "before": "<=",
}
HOST_DURATION_BOUND_KEYS: dict[str, str] = {"min": ">=", "max": "<="}
```

- A `date`-kind field named in `HOST_DATE_BOUND_KEYS` compares in that direction and
  resolves bare day-granularity values against the matching boundary — so `before:`
  behaves precisely like `until:` ("through the full named day") and `after:` like
  `since:`. `since`/`until` keep byte-identical behavior; the table just replaces the
  `if key == "until"` special cases.
- An `int`-kind field named in `HOST_DURATION_BOUND_KEYS` accepts either a bare integer
  (seconds) or a whole-unit duration literal (`30s`, `5m`, `2h`, `1d`), normalizes to
  seconds, and compares in that direction. Every other `int` and `date` field keeps
  today's equality semantics.

This is what makes `min:300` and `min:5m` mean "ran at least five minutes" and
`before:2026-08-20` mean "finished on or before that day".

### The Procs dialect

`procs_query_schema()` joins the built-in schemas in `sase.ace.query_profile.profiles`
and is registered in `pane_registry` under pane id `procs`. It is flat
(`boolean=False`), declares no sigils, macros, or host predicates, and marks every field
negatable.

Free text — a bare word or a quoted string, negated with a leading `-` — matches the
row's searchable text: **the command string, the row label, and the retained command
output**. That is the request's "command string and/or the command output", plus the
label so a monitor row is findable by its agent name.

| Key        | Kind     | Meaning                                                         |
| ---------- | -------- | --------------------------------------------------------------- |
| `text:`    | string   | Same corpus as bare free text (explicit spelling).              |
| `cmd:`     | string   | Command string only (`shlex.join` of `ObservedProc.command`).   |
| `out:`     | string   | Retained command output only.                                   |
| `name:`    | string   | Row label / display name.                                       |
| `agent:`   | string   | A monitor row's member agent name.                              |
| `project:` | string   | Project display name.                                           |
| `status:`  | enum     | `pending`, `running`, `settling`, `success`, `error`, `killed`. |
| `kind:`    | enum     | `command`, `tui`, `detached`.                                   |
| `monitor`  | bool     | The row is a `sase monitor start` proc shell.                   |
| `running`  | bool     | The row is active and owned by a live session.                  |
| `failed`   | bool     | Terminal status is `error` or `killed`.                         |
| `exit:`    | int      | Exit code (exact).                                              |
| `min:`     | duration | Runtime **at least** N (`300`, `5m`, `2h`).                     |
| `max:`     | duration | Runtime **at most** N.                                          |
| `after:`   | date     | Completed at or after the bound.                                |
| `before:`  | date     | Completed at or before the bound.                               |
| `since:`   | date     | **Started** at or after the bound.                              |
| `until:`   | date     | **Started** at or before the bound.                             |
| `limit:`   | host     | Row cap; `all` removes it.                                      |

Notes that the hints and docs must state plainly:

- Runtime is `(finished_at or now) - started_at`, so `min:`/`max:` also read sensibly
  for a still-running proc.
- A running proc has no completion time, so `before:`/`after:` never match it — and
  therefore `-before:X` _does_ include running procs. That is ordinary `NOT` over a
  missing value, and it is the useful reading.
- `before:`/`after:` bound the **completion** time; `since:`/`until:` bound the
  **start** time. Each field's `hint` says so, and the hints surface in the completion
  menu.
- Booleans take the bare shorthand: `monitor` ≡ `monitor:true`, `-monitor` ≡
  `-monitor:true`.

Deliberately out of scope: a `session:` key (the existing `a` scope toggle owns that
axis and the projection pre-filters on it), and a `sase proc list --query` CLI flag. The
dialect is authored as pure, non-Textual domain code precisely so that CLI flag is a
small follow-up, but this epic does not add it.

### Evaluating in Python, on purpose

Flat Artifacts panes route production matching through the Rust corpus facade
(`sase.core.query_profile_corpus_facade`) because they hold large, snapshot-
generation-keyed corpora. The Procs pane does not: it renders a bounded, in-memory
observer projection that re-ticks every 0.25 s, and its most useful haystack — live
command output — changes continuously. Building a per-generation Rust corpus over
churning output would cost more than the match it enables.

So the Procs pane evaluates synchronously through the shared Python evaluator
(`evaluate_query_with_profile_context`), with these guards:

- The searchable text and the `out:` field are the **tail** of the retained output,
  capped at `PROC_QUERY_OUTPUT_TAIL_CHARS = 32_768` per row. The pane already renders
  only a tail, so this is consistent with what the user sees, and the `out:` hint says
  "last 32 KB".
- Output text is materialized **only when the parsed query actually needs it** — a
  single walk of the AST decides whether any free-text, `text:`, or `out:` term is
  present. A query of `monitor -min:300` touches no output at all.
- Rows cache their coerced query row keyed by
  `(proc_id, log.version, status, finished_at)`, so a keystroke re-evaluates
  already-built rows instead of rebuilding them.
- A test asserts the whole-corpus evaluation budget so this stays true.

The grammar itself still gets mirrored into Rust (phase `rust`) — one shared grammar
must have one meaning, and a future `sase proc list --query` or another pane may well
want the Rust path.

### The query bar

`ProcsFilterBar` subclasses the shared `FilterBar`, configured from the compiled `procs`
profile, accented with the Procs teal `#48CAE4` that the gear chips and the runners
modal already use for procs.

Resting behavior differs from Artifacts, matching the request: the bar is **hidden when
there is no query**, and **visible (read-only, syntax-highlighted) whenever a query is
active**, so an active filter is never invisible. That is a new shared
`FilterBar.SHOW_WHEN_ACTIVE` resting mode, not a Procs-only hack.

Two Admin Center integration details:

- `ConfigCenterModal` binds `tab` / `shift+tab` as **priority** bindings, so they win
  over the focused editor and the completion menu would never see `Tab`. The tab actions
  gain a small `consume_priority_tab()` hand-off hook: if the active pane reports that
  it consumed the key (a completion candidate is highlighted), the tab does not switch.
  Panes that do not implement the hook are unaffected. `Enter` accepting a highlighted
  candidate before submitting already works and stays the primary path.
- `FilterBar`'s inner editor forwards `ctrl+j` / `ctrl+k` to the app's Artifacts paging
  actions. That is wrong inside the Admin Center, so it moves behind a
  `FORWARD_ARTIFACTS_PAGING` class flag that `ProcsFilterBar` turns off.

### Pane behavior

- `/` reveals the bar prefilled with the current query, cursor at the end.
- Typing filters live; the status lane shows `N matches` or the parse error with its
  exact span message.
- `Enter` commits and returns focus to the row list; `Esc` restores the query that was
  active when the session opened.
- The rendered `_tasks` list _is_ the filtered list, so selection restoration, jump
  hints, output preview, kill/edit/copy, and the header counts all keep working against
  what is on screen. The header gains a `· N/M shown` segment while a filter is active.
- When rows exist but none match, the output panel says so and names the key that clears
  it, rather than showing the generic "No procs yet."
- The 0.25 s refresh currently rebuilds only when the status snapshot changes. With an
  output-sensitive filter the visible set can change while statuses do not, so the
  change-detection snapshot also folds in the filtered identity list.
- The query lives on `ProcsSessionState` next to `all_sessions`, so closing and
  reopening the Admin Center in the same process keeps the filter.
- `limit:` is honored (rows are capped after matching), so the completion the host
  injects into every filter bar is not a lie.

### `m` — the monitor cycle

`m` cycles the monitor filter through three states:

```
(no monitor term)  →  monitor  →  -monitor  →  (no monitor term)
```

That is "toggle the filter on and off" as requested, with the inverted stop in between
so the request's _"Only show monitor / non-monitor procs"_ is one key away in both
directions. Each press rewrites the query text through a shared, quote-aware
`toggle_flag_token` helper in `sase.filter_tokens`, then:

- if the query is now non-empty, apply it and show the bar in its resting, highlighted
  state **without stealing focus** from the row list;
- if the monitor term was the last one, clear the filter and remove the bar entirely —
  exactly the "remove the query bar if this is the last remaining filter" behavior in
  the request.

The helper normalizes an explicit `monitor:true` / `monitor:false` already in the query
rather than appending a duplicate (which the grammar would reject).

## Phases

### Bare boolean flags and host bound keys in the shared flat grammar

Extend the shared flat grammar in `src/sase/ace/query/` and
`src/sase/ace/query_profile/`. Non-Textual code only.

1. Add `HOST_DATE_BOUND_KEYS` and `HOST_DURATION_BOUND_KEYS` to
   `query_profile/registry.py` and export them from the package, documented in the same
   "closed host vocabulary" voice as `HOST_SIGIL_CHARS` and `HOST_PREDICATES`.
2. `profile_reference_flat._flat_clauses`: before falling through to `_text_clause`,
   treat a token as a bare flag when it is not `wholly_quoted`, carries **no quoted
   characters at all** in its body (`not any( token.body_quoted)` — so `-"monitor"`
   stays negated free text, exactly as it does today), has no unquoted colon, and whose
   case-folded body is a filterable `bool` field. Emit
   `_FlatFieldClause(key, ("true",), negated=token.negated)`. Keep the existing negation
   and single-occurrence guards on that path (a bare flag on a non-negatable bool field
   must still raise "may not be negated"). `canonical_flat_query` therefore emits
   `key:true` / `-key:true` with no extra work.
3. `profile_reference_support.normalize_query_value`: for an `int` field named in
   `HOST_DURATION_BOUND_KEYS`, accept `\d+` (seconds) or `\d+[smhd]` and normalize to
   seconds; reject composite literals like `1h30m` with a message that suggests the
   two-term form. For a `date` field, pick the boundary from `HOST_DATE_BOUND_KEYS`
   instead of the `key == "until"` special case.
4. `profile_evaluator`: replace the `since`/`until` branch in `_match_date_field` with a
   `HOST_DATE_BOUND_KEYS` lookup, and give the `int` branch of `_match_field` the same
   `HOST_DURATION_BOUND_KEYS` direction lookup (falling back to equality). Behavior for
   every currently declared field is unchanged.
5. `profile_highlighting._classify_token`: add the bare-flag branch in the same position
   the parser uses, styled `property_key`, so a bare `monitor` colors like a key rather
   than a free-text term.
6. Tests: bare flag parses and canonicalizes; `-flag` canonicalizes to `-key:true`; a
   quoted `"flag"` and a negated quoted `-"flag"` both stay free text; `flag -flag`
   raises the single-occurrence error; a bare flag on a non-negatable field raises;
   every date/duration bound key resolves in the right direction; duration literals
   round-trip to seconds; composite duration literals raise; the highlighter agrees with
   the parser. Include the intended Stitches `sidecar` behavior change as an explicit
   regression test.

Verify with `just check`.

### Procs query profile and row adapter

1. `procs_query_schema()` in `src/sase/ace/query_profile/profiles.py`, exactly the field
   table in the Design section, with a `free_text_hint` of
   `"command, label, output (implicit AND)"` and per-field hints that state the time
   axis (`before`/`after` = completed, `since`/`until` = started) and the output tail
   cap. Source the `status:` and `kind:` enum values from the same constants
   `sase.main.parser_proc` mirrors, without importing the proc store into schema
   construction. Register `procs` in `query_profile/pane_registry.py`.
2. A new module — `src/sase/ace/tui/_proc_query.py` — holding:
   - `PROC_QUERY_OUTPUT_TAIL_CHARS = 32_768`;
   - `proc_query_row(proc, *, now, with_output)` building the mapping row (`stable_id`,
     `fields`, `searchable_text`) the shared coercer consumes;
   - `query_needs_output(expr)`, a one-pass AST walk that reports whether any free-text,
     `text:`, or `out:` term is present;
   - `ProcQueryFilter`, a small stateful helper owning the compiled profile, the
     parsed-AST cache keyed on the raw query string, the per-row coerced-row cache keyed
     on `(proc_id, log.version, status, finished_at)`, and a `matching(rows, *, now)`
     entry point returning the filtered rows. Parse errors surface as the shared
     `ProfileQueryError` so the bar's existing error rendering works unchanged.
3. Tests covering each field against hand-built `ObservedProc` rows: monitor and running
   booleans and their bare/negated spellings; `min`/`max` over both finished and running
   procs against a pinned clock; `before`/`after` excluding running procs and `-before:`
   including them; `since`/`until` over start time; free text hitting command, label,
   and output; `cmd:` and `out:` scoping to one side; the request's exact
   `"just check" -monitor -min:300` query; the output tail cap; `query_needs_output`
   gating; and a corpus-level evaluation budget assertion.

Verify with `just check`.

### Procs filter bar widget and Admin Center key integration

1. `FilterBar` gains `SHOW_WHEN_ACTIVE: ClassVar[bool] = False`. When set, the resting
   display in `__init__`, `close()`, and `set_query()` follows "visible iff the query is
   non-empty", and `_apply_accent` runs for these bars too so the teal border and sigil
   render.
2. `FilterBar` gains `FORWARD_ARTIFACTS_PAGING: ClassVar[bool] = True`; the inner
   editor's `ctrl+j` / `ctrl+k` forwarding is gated on it.
3. Bare-flag completion: in `_filter_bar_completion`, a filterable `bool` field offers a
   bare `key` candidate (hinted as a flag) alongside `key:`, so the shorthand is
   discoverable rather than folklore.
4. `src/sase/ace/tui/modals/procs_filter_bar.py`: `ProcsFilterBar(FilterBar)` with
   `ACCENT = "#48CAE4"`, procs-specific DOM ids, `SHOW_WHEN_ACTIVE = True`,
   `PERSISTENT = False`, `FORWARD_ARTIFACTS_PAGING = False`, and its own `QueryChanged`
   / `Submitted` / `Dismissed` messages. Completion metadata comes from the compiled
   profile, so no key table is duplicated here.
5. `ConfigCenterModal.action_next_center_tab` / `action_prev_center_tab` consult an
   optional `consume_priority_tab()` on the active pane before switching.
6. TCSS in `styles.tcss` for `ProcsFilterBar` (completion highlight, border, sigil)
   following the existing per-bar blocks.
7. Tests: resting visibility flips with the query; the accent applies; `ctrl+j` does not
   reach the Artifacts action; a bool field offers a bare completion candidate; `Tab`
   accepts a highlighted candidate instead of switching tabs and still switches tabs
   when no candidate is highlighted.

Verify with `just check`.

### Procs pane filter session

1. `src/sase/ace/tui/modals/procs_pane_filter.py` — `ProcsPaneFilterMixin` modeled on
   `FilesFilterSessionMixin`: `show_filters()`, the three message handlers, committed vs
   live query, restore-on-dismiss, and a `_filtered_tasks(rows)` helper driving
   `ProcQueryFilter`.
2. `ProcsPane`: mount `ProcsFilterBar` above `#procs-panels`, add the mixin, and add
   `("slash", "focus_filter", "Filter")` to `BINDINGS`. `_merged_tasks()` applies the
   active query and then the `limit:` cap.
3. `_refresh_running_output`'s change detection folds the filtered identity list into
   `_status_snapshot()` so an output-sensitive filter re-renders when the visible set
   changes without a status change.
4. `_title_text()` appends `· N/M shown` while a filter is active;
   `_display_output(None)` distinguishes "no procs yet" from "no procs match the active
   filter"; `_hints()` gains `/: filter` as a protected token.
5. `ProcsSessionState` gains `query: str = ""`; the pane seeds from it on mount and
   writes back on commit.
6. Tests in `tests/ace/tui/`, reusing `_procs_pane_helpers`: `/` opens the bar
   prefilled; live narrowing; commit and dismiss; selection survives a filter change and
   falls back sanely when the selected row is filtered out; jump hints renumber over
   visible rows only; the no-match empty state; `limit:` capping; a parse error renders
   in the status lane and leaves the previous rows untouched; the query survives an
   Admin Center close/reopen; a running proc's new output moves it into a matching
   filter on the next tick.

Verify with `just check`.

### The `m` monitor-filter cycle

1. `toggle_flag_token(text, key)` in `src/sase/filter_tokens.py`: quote-aware
   three-state cycle over a flag token, normalizing an existing `key:true` / `key:false`
   / `-key` spelling in place and preserving the rest of the query verbatim (including
   surrounding whitespace collapse rules the canonicalizer already implies).
2. `ProcsPane.action_toggle_monitor_filter`, bound to `m`: rewrite the query, apply it,
   show or remove the bar per the Design section, and never steal focus from the row
   list. Add `m: monitor` to the hint line.
3. Tests: the full cycle from empty; the cycle when other terms are present (only the
   monitor term changes); normalizing a hand-typed `monitor:false`; the bar disappearing
   when the monitor term was the last one; focus staying on the row list; `m` inside the
   open editor still typing the letter `m`.

Verify with `just check`.

### Mirror the shared grammar extensions in sase-core

Open the `sase-core` linked repo with the `/sase_repo` skill and work in the printed
path. Do not bump the `sase-core-rs` floor in this repo's `pyproject.toml` — per
`docs/rust_backend.md`, the release lane owns that window.

1. `crates/sase_core/src/query/flat.rs`: the bare-boolean-flag branch, in the same
   position as the Python parser, with the same negation and single-occurrence guards
   and the same `key:true` canonical output.
2. `crates/sase_core/src/query/evaluator.rs`: replace the hard-coded `since`/`until`
   date arms with the host bound-key table, and add the duration-bound direction for
   `int` fields, mirroring `HOST_DATE_BOUND_KEYS` / `HOST_DURATION_BOUND_KEYS`
   byte-for-byte.
3. Rust tests in `crates/sase_core/src/query/tests.rs` covering the same cases as the
   Python grammar tests.
4. Back in this repo, extend the parity parameters in
   `tests/test_query_profile_corpus_facade.py` so bare flags and bound keys are asserted
   identical through the Rust parse/canonicalize path.

Both halves must be verified: `cargo test` in the core checkout, and `just check` here.
Because the Procs pane evaluates in Python, this phase is not a prerequisite for the
user-facing feature — it is what keeps one grammar from having two meanings, and it can
land in parallel with `dialect` through `polish`.

### Documentation, visual snapshot, and copy review

1. `docs/ace.md`, "Procs Tab": a "Filtering procs" section with the full key table, the
   bare-flag and `-` negation rules, the two time axes, the runtime definition, the
   output tail cap, the `m` cycle, and the worked `"just check" -monitor -min:300`
   example.
2. Document the bare-boolean shorthand where the shared flat dialect is described,
   including the intended Stitches `sidecar` change.
3. A PNG snapshot in `tests/ace/tui/visual/` of the Procs tab with an active filter, so
   the bar's teal accent, highlighted closed display, and `N/M shown` header are pinned.
   Follow the visual-suite conventions in this repo's memory: run `just test-visual`,
   inspect `.pytest_cache/sase-visual/` on failure, and accept intentional changes with
   `--sase-update-visual-snapshots`.
4. Final copy pass over hint text, the completion hints, the placeholder, and the
   parse-error wording.

Land with `just check-full` through `/sase_monitor`, since this is the epic's combined
tree.

## Risks and mitigations

- **Shared-grammar blast radius.** The bare-flag rule changes what a bare `sidecar`
  means on Stitches. Mitigated by: no wire or digest change (no saved query is
  invalidated), an explicit regression test, a documented note, and the fact that
  production Stitches matching canonicalizes in Python before Rust evaluates, so Python
  and Rust agree from the first commit even before the `rust` phase lands.
- **Python/Rust grammar drift.** Real, and the reason the `rust` phase exists. The
  parity tests in `tests/test_query_profile_corpus_facade.py` are extended in that phase
  so drift fails a test rather than surprising a user later.
- **Filtering cost on the 0.25 s tick.** Mitigated by the output tail cap, the
  needs-output AST gate, the version-keyed row cache, and an explicit evaluation budget
  test.
- **Priority-`Tab` hand-off.** Touching Admin Center tab switching is delicate; the hook
  is opt-in per pane, defaults to today's behavior, and is covered by a test on both
  branches.
