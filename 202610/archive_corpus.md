---
tier: epic
title: Archive corpus, in:archive scope, and CLI parity
goal: "sase-core answers which hidden archived runs match a query, in what order, and
  grouped how, and sase agent search in:archive returns that same answer.

  "
phases:
  - id: outcome-map
    title: One status table for archive outcome
    depends_on: []
    size: medium
    description:
      "outcome-map: move the live status-bucket match into sase-core and derive archive
      outcome from that bucket."
  - id: corpus-compile
    title: Compiled archive corpus and derived fields
    depends_on:
      - outcome-map
    size: medium
    description:
      "corpus-compile: compile a cached-ready in-memory corpus from the v3 index, with
      derived fields and index status."
  - id: corpus-query
    title: Group summaries, windows, and exact lookup
    depends_on:
      - corpus-compile
    size: medium
    description:
      "corpus-query: evaluate agents-archive queries into summaries, windowed rows,
      counts, and exact lookup."
  - id: profile-token
    title: agents-archive profile and the in token
    depends_on: []
    size: medium
    description:
      "profile-token: add the agents-archive query profile and the host-owned in: token,
      without switching views."
  - id: bindings-cache
    title: Bindings, facade, and shared corpus cache
    depends_on:
      - corpus-query
      - profile-token
    size: medium
    description:
      "bindings-cache: expose the corpus through PyO3 and one off-thread cache shared by
      the TUI and the CLI."
  - id: cli-parity
    title: sase agent search parity and the bench
    depends_on:
      - bindings-cache
    size: medium
    description:
      "cli-parity: route in:archive and in:inbox through the corpus and catalog, and
      prove CLI order matches rows()."
proposed_by: bbugyi200.athena.sase-1jm.2
parent_bead: sase-1jm.2
create_time: 2026-10-10 07:31:46
status: wip
---

- **PROMPT:**
  [prompts/202610/archive_corpus.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/archive_corpus.md)
- **PARENT:**
  [202610/agents_archive_view.md](https://github.com/sase-org/sase--plans/blob/main/202610/agents_archive_view.md)

# Plan: Archive corpus, in:archive scope, and CLI parity

This is the implementation plan for phase `sase-1jm.2` of epic `sase-1jm`.
`sase plan propose` stamps `parent_bead` from that phase. The child epic's land agent
closes this child epic. The host cascade then closes `sase-1jm.2`. Do not close
`sase-1jm` or any ancestor above `sase-1jm.2`.

The parent design is `plan:202610/agents_archive_view.md`, section "Core archive corpus,
in scope token, and CLI parity". Read it with `sase artifact read`. The decisions below
are already final. Implement only the selected branches.

- `view_name = archive`. The token is `in:archive`. User-facing CLI text says Archive.
  Do not implement `in:history` or the word History for this view.
- `toggle_filter = restore` and `jump_opens_archive = yes` belong to later parent
  phases. Do not implement them here.
- `glossary_archive_term = no` and `glossary_strand_updates = no`. Do not edit any
  memory note. Both skipped glossary edits are already `PROPOSED FOLLOW-UP:` notes on
  `sase-1jm.2`. Do not add a second copy.

Parent phases `sase-1jm.3` through `sase-1jm.8` own the Archive view, the
`agents_archive_view` flag, the record deck, restore, routing, the shelf, and pane
retirement. This epic does not start that work.

## Shared contract

The corpus is the one answer for "which archived runs match, in what order, grouped
how". Presentation stays in Python. Later phases paint banners, chips, and widgets from
the light rows this epic returns.

Leave the existing SQL helpers `query_agent_archive` and `agent_archive_facet_counts` in
place. They take caller-supplied `where_sql` and are not this corpus. Do not route the
new operations through them.

Open sase-core with `sase repo open sase-core` and read its `AGENTS.md` before editing
it. Run guarded checks with `sase tool run check` in each repo a phase changes. Do not
run `just check-full`. A failure that reproduces on the clean base tree is a
`PROPOSED FOLLOW-UP:` on that phase bead, not a reason to keep the phase open.

New Rust files stay at or under 1,500 lines. Put the corpus in new files under
`crates/sase_core/src/agent_archive/` and re-export from `mod.rs`. Do not relocate the
existing SQL functions. Do not add `macro_rules!`, do not edit `CHANGELOG.md` or crate
versions, and do not add new `core_*` aliases to `crates/sase_core_py/src/prelude.rs` or
new root re-exports in `crates/sase_core/src/lib.rs`. Import new items by module path.

### Index status

Rebuild progress lives in the Python process (`dismissed_bundle_index_progress`), not in
sqlite. The compile request takes a status the caller already probed:

- `missing` — no index file. Return an empty corpus. Do not create or rebuild the index.
- `rebuilding` plus the caller's indexed-row count. Compile whatever v3 rows are already
  readable, and carry the count on every response.
- `ok` — compile the full readable index.

Also read `schema_version` from the index meta. A present file whose version is not 3 is
`unsupported`, with an empty corpus and a typed error payload. Never panic, and never
trigger a rebuild.

`summary` carries completeness: `missing`, `rebuilding` (with the count), `unsupported`,
or `ok`.

### Derived fields

Stable row identity is the archive key
`(source_username, source_machine, source_run_id)`.

The corpus holds top-level rows whose effective visibility is `hidden`
(`COALESCE(archive_visibility_projection.visibility, archive_visibility)`).
Workflow-child rows are not list rows. They go into a name map so lookup can return the
child's owner. Rows that are `visible` or `pinned` stay out; they are inbox rows.

- `outcome` is `done`, `failed`, or `interrupted`, from the outcome-map function. Do not
  add a second status string table in the corpus.
- `last_activity_at` plus `time_basis`: `stop_time` → `ended`, else a real
  `dismissed_at` → `dismissed`, else `start_time` → `started`. Never use file mtimes.
  Parse offset-aware ISO-8601 and the naive-local forms the v3 index already stores
  (read `tests/test_dismissed_bundle_index.py` and the summary writer before choosing
  formats). The compile request carries the host timezone
  (`sase.core.time.get_timezone`) so naive values become absolute instants. Missing all
  three timestamps leaves activity empty and sorts last.
- `runtime_seconds` from the index runtime when it parses, otherwise from start and stop
  when both exist.
- Container key: `clan:{agent_clan}:{agent_clan_generation}` when both clan fields are
  non-empty, else `session:{agent_session}` when the session is non-empty, else the
  row's own archive key. A presentation root with one matching member is a plain member
  row. Two or more matching members collapse to one container whose label is the clan
  name, else the session name. The container's activity is its newest member's
  `last_activity_at`. Its outcome is the worst member outcome, ordered `failed` >
  `interrupted` > `done`. Its count is the matching members only. Member rows inside an
  expanded container sort newest activity first.
- `restorable` is `durably_revivable` and the bundle path exists. Stat only while
  compiling, never while querying.

Optional link facets, keyed by agent name, are a compile input. They use the existing
`AgentCatalogLinkFacets` shape (`relations`, `artifacts`, `count`) built by
`build_agent_catalog_link_facets` over `load_artifact_links_snapshot`. The corpus stores
them and does not compute them. Porting that helper into core is a parent-epic non-goal.

### Query surface

Python canonicalizes relative dates before the call, the same way
`sase.core.query_profile_corpus_facade` does. Rust evaluates the already literal query
with `sase_core::query::evaluate_query_many_in_corpus` against rows built from the
derived fields. Unknown fields and unknown values return the existing typed query error.
Never panic.

`group_by` is `day`, `project`, `outcome`, or `model`. Anything else is a typed error.
Day keys are `YYYY-MM-DD` in the compile timezone. Project keys come from the index
project name. Model keys use the stored model or an empty string. Outcome keys are the
three outcome values. Banner prose (`Today`, week ranges, month names) is Python and
belongs to `sase-1jm.3`.

Sort is newest `last_activity_at` first, then archive key, so the order is stable.

Operations:

- `summary(query, group_by)` → total matching presentation roots, each group (`key`,
  `count`, newest activity), and completeness.
- `rows(query, group_by, group_key, offset, limit, expanded_container_keys)` → one
  window of light rows inside that group. An unexpanded container is one row. An
  expanded container is that row plus its matching member rows, and those member rows
  consume the limit. Offset and limit page that flattened window.
- `lookup(name)` → one exact match on the short name, the canonical name, or the global
  name, including a workflow child, which resolves to its owner. Use
  `normalize_agent_archive_name` and `globalize_agent_name`. No match returns an empty
  result, not an error.
- `count(query)` → the number of matching presentation roots.

Light rows carry the archive key, container key, container flag, member count, agent
name, stored status, outcome, model, provider, project, `last_activity_at`,
`time_basis`, start time, `runtime_seconds`, and `restorable`. They do not carry prompt
or transcript text.

### agents-archive fields

Build the profile from `src/sase/ace/query_profile/profiles/_agents_shared.py`.

Shared fields: `name`, `session`, `clan`, `project`, `role`, `workflow`, `model`,
`provider`, `kind`, `status`, `since`, `until`, `after`, `before`, `min`, `max`,
`attempt`, `retry`.

Archive fields: `outcome` (`done`, `failed`, `interrupted`), `restorable` (bool), `tab`
(the bundle `agent_tab`), `tribe` (the user tribe, as on the Agents tab), `clan_tribe`,
`linked`, `relation`, `artifact`.

Inbox-only fields, which this profile does not own: `unread`, `pinned`, `needs`,
`source`, `cl`, `machine`, `text`.

Date bounds compare against `last_activity_at` for `after` and `before`, and against
start time for `since` and `until`. Runtime bounds use `runtime_seconds`.

## outcome-map

The complete status match is `sase.agent.status_buckets.status_bucket_for_values`.
Core's `DISMISSABLE_STATUSES` is a different predicate. Do not use it as the outcome
table.

Move that match, after glyph stripping, into sase-core as `status_bucket_for_status`.
`archive_outcome_for_status` maps its bucket: `Done` → `done`, `Failed` → `failed`,
every other bucket → `interrupted`. Keep glyph stripping in Python and pass the
canonical text to the binding. `status_bucket_for_values` becomes a wrapper around that
binding. Leave the module's other public constants and helpers where they are.

Do not add statuses the current function does not already bucket. The design prose lists
`TESTED`, `EPIC TIMED OUT`, and `LAUNCH REJECTED` as examples of the existing classes.
If today's function does not already treat them as Done or Failed, they stay
`interrupted`. The outcome function must not grow a second list.

`tests/test_agent_status_buckets.py` keeps its current expectations. Add core tests for
the three outcome classes, including `DONE`, `TALE DONE`, `EPIC CREATED`, `FAILED`,
`PLAN FAILED`, `RUNNING`, `WAITING`, and `QUESTION`. Add the PyO3 binding beside the
existing status bindings, with a round-trip test. Follow the sase-core binding recipe in
`AGENTS.md`.

This phase changes sase-core and the Python wrapper. Run `sase tool run check` in
sase-core and `sase tool run check` in sase. Read `sase/memory/lint_and_test.md` through
`sase memory read` before finishing the sase edit.

## corpus-compile

Compile the corpus described in the shared contract. This phase returns derived rows and
index status. It does not evaluate queries, group them, or bind them to Python.

Accept the visibility join, the workflow-child name map, link-facet input, host
timezone, and the caller's index status. `restorable` stats bundle paths only during
compile.

Tests, in sase-core, cover:

- every outcome class through `archive_outcome_for_status`
- all three time bases, plus a row with no timestamps
- naive-local and offset-aware timestamps
- clan containers, session containers, and singleton rows
- worst-outcome aggregation and matching-member counts
- workflow children absent from the row list and present in the name map
- `hidden` included; `visible` and `pinned` excluded
- `missing`, `rebuilding` with a count, `ok`, and `unsupported` schema

Run `sase tool run check` in sase-core.

## corpus-query

Add `summary`, `rows`, `lookup`, and `count` on the compiled corpus, using the shared
query surface. Errors are a `thiserror` enum mapped from the query engine's typed
errors.

Tests, in sase-core, cover:

- filter evaluation for `outcome`, `restorable`, `project`, `model`, a date bound, and a
  link facet (`linked`, `relation`, `artifact`)
- day, project, outcome, and model summaries, including empty groups omitted
- window offset and limit, and expanded containers inserting member rows into that
  window
- lookup by short name, canonical name, global name, and a workflow child that resolves
  to its owner
- an unknown field and an unknown enum value return typed errors
- an unknown `group_by` returns a typed error
- order is newest activity first and is stable when timestamps tie

Run `sase tool run check` in sase-core.

## profile-token

Add `src/sase/ace/query_profile/profiles/_agents_archive.py` and register
`agents-archive` in `src/sase/ace/query_profile/pane_registry.py`. The field set is the
shared contract. Add an `agents-archive` entry to
`tests/ace/tui/artifacts_contract/goldens/query/profile_cases.json` so
`tests/ace/tui/artifacts_contract/test_query_conformance.py` covers it. Include one
valid query per archive-only field and one unknown-field error.

Add `src/sase/ace/query/scope_token.py`, modeled on `src/sase/ace/query/limit_token.py`.

- Values are `inbox` and `archive`, case-insensitive, canonicalized to lowercase.
- At most one `in:` term. It must be top-level: never negated, never inside `OR`, `NOT`,
  or parentheses.
- Expose completion items `inbox` and `archive`.
- Scope-hint texts, built from the `agents-live` and `agents-archive` field sets:
  `restorable: is an Archive field · add in:archive or press ,a`, and the inbox mirror
  `unread: is an Inbox field · remove in:archive or press ,a`. Use the offending field
  name. Archive-only fields are the ones `agents-archive` owns and `agents-live` does
  not. Inbox-only fields are the reverse.

Wire extraction on the Agents-tab paths and the CLI path only:

- `src/sase/ace/tui/models/agent_live_query_engine.py`
- the Agents-tab filter session
  (`src/sase/ace/tui/actions/agents/_filter_bar_session.py`) and the legacy agents
  editor if it still commits a query
- `src/sase/agents/cli_search.py`, enough that a later phase can branch on the extracted
  scope. This phase may leave the catalog evaluation in place when the scope is
  `archive`; `cli-parity` replaces that branch.

Strip `in:` before dialect parse so the inbox profile does not report it as an unknown
field. Do not change which inbox rows match. Do not switch views, do not add the
`agents_archive_view` flag, and do not add `in:` to the shared filter bar's global
completions. Artifacts panes, including Artifacts → Agent, keep rejecting `in:` as an
unknown term.

When the Agents tab validates a query, an archive-only field without `in:archive` raises
the scope hint, and an inbox-only field with `in:archive` raises the mirror hint.
`in:archive` by itself stays valid and does not change the inbox result. `sase-1jm.3`
owns the view switch.

Tests live next to `tests/ace/test_limit_token.py` and the agents profile tests. Run
`sase tool run check` in sase. Read `lint_and_test.md` first.

## bindings-cache

Bind `summary`, `rows`, `lookup`, and `count` from
`crates/sase_core_py/src/agent_custody/`, next to `py_query_agent_archive`. Import the
new core functions by module path. Parse request dicts into the wire types, map the core
error to a Python exception, and return `serialize_to_py`. Register each function in
that domain's register function. Add round-trip tests in the domain's `tests.rs`.

Extend `src/sase/core/agent_archive_facade.py` with a small frozen-dataclass facade for
those four operations plus compile. Keep the existing capability helpers. The facade
does not import Textual.

Add one process-wide cache in a sibling module under `src/sase/core/`. Key it by
`(index signature, link-facet signature, agents-archive profile digest)`. Build
off-thread. Last request wins: a stale worker's result is dropped when the key changed
while it ran. One accessor serves the TUI and the CLI. Do not build at import time, on
the Textual event loop, or as a side effect of inbox startup. The accessor is lazy. The
caller passes the Python progress probe and the link facets. The cache asks
`build_agent_catalog_link_facets` for the facet input when the caller does not pass one,
off the event loop.

Do not hand-edit `sase-core-revision.txt` when this turn also declares the sase-core
checkout. The host writes that pin from the sibling SHA. Read `docs/rust_backend.md`
("The CI source revision pin"). If the opened sase-core checkout does not yet contain
the corpus operations, stop and say so on the phase bead instead of binding against a
stale tree.

Run `sase tool run check` in sase-core and in sase. Read `lint_and_test.md` before
finishing the sase edit.

## cli-parity

`sase agent search` keeps today's catalog behavior when the query has no `in:` token.
Document that in the command help and in the docs below.

- `in:inbox` uses the same catalog path, restricted to non-dismissed rows.
- `in:archive` evaluates the remainder through the shared corpus accessor and the
  `agents-archive` profile. Extract `limit:` the way the catalog path already does.
  Build link facets with `build_agent_catalog_link_facets`, the same helper the cache
  uses.
- The default table is colored and shows time, outcome, name, model, and runtime, in the
  corpus order. `-j` / `--json` emits the light-row derived fields, in that same order,
  as a stable JSON array.
- Scope-hint and token errors print to stderr and exit 2, matching today's query errors.
- A `missing`, `rebuilding`, or `unsupported` index prints a stderr notice and still
  prints the rows the corpus returned. It does not start a rebuild.

Follow `sase/memory/cli_rules.md` (read it through `sase memory read`). Update the help
and epilog in `src/sase/main/parser_agent_search.py`. Keep options alphabetically sorted
and keep every long option's short alias. Update `docs/cli.md` (`sase agent search`) and
`docs/query_language.md` (the agent-profile paragraph and a short host-owned `in:`
section next to `limit:`).

Parity oracle: one fixture, several queries (no filter, `outcome:failed`, a project
filter, a container with two members). CLI `-j` identities and order equal facade
`rows()` identities and order.

Bench: `tests/perf/bench_agent_archive_corpus.py`, beside `bench_agent_catalog.py`.
Synthetic fixture of 11,000 top-level rows plus 37,000 workflow children, not a live
home-directory archive. Report corpus build p95 and query-plus-first-window p95. Targets
are build under 300 ms and query plus first window under 30 ms. A small pytest asserts
those ceilings on the synthetic fixture. Record the measured numbers in the phase notes.
If the stat-during-compile cost blows the build ceiling, keep correctness, record the
measurement, and file `PROPOSED FOLLOW-UP:` on this phase bead rather than dropping
`restorable`.

Run `sase tool run check` in sase. Read `lint_and_test.md` first.

## Landing

The land agent closes this child epic after every phase bead is closed. It does not
close `sase-1jm`. The host cascade closes `sase-1jm.2` because this epic's `parent_bead`
is that phase. Before the close, run `sase bead epic-symbols` on this child epic and on
`sase-1jm.2`. There are no entries today. If a phase added one, re-key it to a bead that
stays open (this child epic, while it is open, or `sase-1jm`).

Triage `PROPOSED FOLLOW-UP:` notes from this child epic's phase beads with
`/sase_new_task`. Leave the two glossary notes already on `sase-1jm.2` for
`sase-1jm.land`. Do not edit memory.
