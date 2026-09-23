---
tier: epic
title: Rock-solid artifact link jumps
goal: 'Following any artifact link (the `$` link rail, the `$0` Links panel, relation
  jumps) always lands on the target row: the destination Artifacts sub-tab switches
  to a verified, readable context query that shows the target among its natural family
  (for example `id:sase-16n.* limit:100` for a closed epic phase), selects it, explains
  the rewrite in one clear toast, and restores the user''s query with `^`. Jumps fail
  only for truly dangling refs.

  '
phases:
- id: core-glob
  title: Wildcard matching in the sase-core query evaluator
  depends_on: []
  size: small
  description: 'core-glob: in the linked sase-core repo, make `*` a wildcard inside
    string-field property values (anchored for exact-match fields and `sha`, unanchored
    for substring fields, literal for enums), with Rust tests proving `id:alpha-1.*`
    semantics and byte-identical behavior for values without `*`.'
- id: host-glob
  title: Wildcard parity, pin bump, pushdown guard, and docs in sase
  depends_on:
  - core-glob
  size: small
  description: 'host-glob: move the sase-core CI pin past the wildcard commit, mirror
    the semantics in the Python reference evaluator with parity tests, keep glob values
    out of Agents-tab index pushdown, and document wildcards in the query language
    reference and field hints.'
- id: seam
  title: Never lose a pane report, never treat loading as absence
  depends_on: []
  size: medium
  description: 'seam: fix the confirmed root cause (Beads/Plans/Agents/Files resolve
    a request synchronously, the report is dropped during dispatch, and the pane returns
    PENDING so the transaction hangs), capture synchronous reports at the host seam,
    make fold-hidden pending targets truthful, wait on loading panes, re-resolve refs
    after load, fix project scope handling, re-index hydrated rows, and add real-pane
    regression tests.'
- id: planner
  title: Plan-then-commit reveal engine with Beads context queries
  depends_on:
  - host-glob
  - seam
  size: medium
  description: 'planner: replace the trial-and-error limit-drop, widening, and neutral
    rungs with one verified context rewrite (fold, acquire, context, identity, neutral,
    honest failure), add the `host_reveal_context` pane hook, the dialect-aware query
    renderer, the limit policy, and hidden-reason analysis, and ship the Beads family
    queries (`id:<epic>.*`) end to end.'
- id: flat-panes
  title: Agent and File context queries plus Agents-tab reveal
  depends_on:
  - planner
  size: medium
  description: 'flat-panes: add hood/family context queries for the Artifacts Agent
    pane and creating-agent context for Files, and make `agent:` jumps reveal folded
    rows on the Agents tab, falling back to Artifacts ▸ Agent when the Agents-tab
    filter hides them.'
- id: bounded-panes
  title: Stitch, Plan/provider, and Patch context queries
  depends_on:
  - planner
  size: medium
  description: 'bounded-panes: acquire-then-reveal for panes whose inventory is bounded.
    Stitch jumps rewrite to a repo-and-day window, Plan and provider panes use a `path:`
    context, and Patch jumps show the patch''s stack, with Stitches'' reveal row read
    from the unfiltered collection.'
- id: toast
  title: The reveal toast, lens chip, and user docs
  depends_on:
  - planner
  size: small
  description: 'toast: add a pure, escaped toast formatter showing the new query,
    what hid the target, the scope change, and the real restore/back keys, plus unified
    failure copy, a context label on the lens chip, and rewritten reveal-ladder docs.'
- id: entry-points
  title: One engine for every jump, plus the end-to-end matrix
  depends_on:
  - flat-panes
  - bounded-panes
  - toast
  size: medium
  description: 'entry-points: route relation-panel misses and cross-pane relation
    jumps through the engine, fix `job:` jumps hidden by a collapsed scheduler fold
    and unconfigured provider kinds, and add a Links-panel-driven end-to-end test
    matrix covering every artifact kind.'
proposed_by: bbugyi200.athena.0pq
create_time: 2026-09-23 08:23:28
status: wip
bead_id: sase-16t
---

- **PROMPT:** [prompts/202609/artifact_link_jumps.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/artifact_link_jumps.md)
- **BEAD:** [sase-16t](https://github.com/sase-org/sase--beads/blob/main/pages/sase-16t/README.md)

# Plan: Rock-solid artifact link jumps

## Why jumps fail today

The machinery from epic `sase-w3` (a host-owned "reveal ladder": fold, drop `limit:`,
identity query, minimal widening, `limit:all`, targeted hydration) exists. It almost
never runs in the real app, and when it does it writes the wrong queries.

### Root cause 1 (confirmed with a real-pane repro): dropped synchronous reports

Reproduced with a real `ArtifactsBeadsPane` under `sase.ace.testing.AcePage`, using the
`tests/ace/tui/_artifacts_beads_helpers.snapshot` fixture with a project-aware loader.
The Artifacts scope is already the bead's project and the query is
`-status:closed limit:100`. Following `bead:alpha-1.1` (a closed phase) switches to the
Beads sub-tab and then:

- the query stays `-status:closed limit:100` and nothing is selected;
- `app._link_follow_transaction` stays open forever at the first ladder rung (`rung=3`);
- no toast appears.

This is exactly what the user sees. The mechanism:

1. `_request_artifacts_target` (`src/sase/ace/tui/actions/_link_follow_targets.py`) sets
   `_link_follow_dispatching = True` and calls the pane's `request_entry_target`.
2. Beads (`widgets/artifacts/beads_navigation.py`), Plans (`plans_navigation.py`),
   Agents (`agents_navigation.py`), and Files (`files_navigation.py`) all set a pending
   target and call `_refresh_options()` synchronously. When the current query's result
   is already cached, that refresh resolves the request immediately through
   `_complete_entry_request`: `MISSING`, `FAILED`, or even `SELECTED` after a fold
   auto-expands.
3. `_complete_link_follow_request` ignores the report because dispatch is in progress,
   on the assumption that the return value will carry it. The pane then returns
   `LinkRequestState.PENDING` anyway. It has already cleared its pending target, so it
   will never report again.

Stitches (`commits_detail.py`) and Patches (`panes.py`) return the real state and are
not affected.

### Root cause 2: the ladder writes the wrong queries

Even when a `MISSING` does arrive, for example after a scope switch forced a reload, the
ladder:

- drops the user's `limit:` first (`-status:closed limit:100` becomes `-status:closed`),
  which cannot help a filtered-out row;
- then isolates the row (`id:alpha-1.1`) or subtracts terms (minimal widening), which
  gives either a context-free single row or an arbitrary widened list;
- commits every rung as a visible query change and re-evaluates each time.

The user's expectation for the example is the target's family, with their limit kept:
`id:sase-16n.* limit:100`, with `sase-16n.7` selected.

### Other defects found

These are from code reading plus two Explore surveys, and each is assigned to a phase
below.

- `_handle_missing_link_follow` skips every rung and toasts "not in its inventory"
  whenever the pane is loading. Loading is treated as absence.
- `target_project_scope` treats `target.parts[0]` as a project for every project-scoped
  pane. For Agents, Files, and Stitches, that is an agent name, a logical file id, or a
  repo label.
- It also narrows an "All projects" scope down to one project when nothing requires it.
- Plans' `_pending_option_id` finds rows hidden under collapsed banners, so it reports a
  false `SELECTED` while the highlight lands elsewhere.
- Agents never expands a collapsed banner for a pending target. Files does.
- Stitches' `host_query_row_for_target` reads the filtered `self.result`, so no reveal
  can ever use it for a filtered-out commit.
- Hydrated rows are merged without re-indexing in Beads and Agents. Plans' merge
  disables filtering entirely, so the rewritten query cannot match the new row.
- A chip target built before the pane loaded (Beads chips always say `task`) is never
  re-resolved, so a phase is looked up as a task.
- `agent:` jumps check only the visible Agents-tab rows. A folded agent falls through to
  a different surface.
- `job:` jumps fail when the scheduler service row is collapsed.
- Cross-pane relation jumps (`_navigate_to_relation_target`) bypass the transaction and
  the ladder entirely.
- Tests use fake panes (`tests/ace/tui/_link_follow_helpers.py`), which is why none of
  this was caught. Their shared `_link_follow_outcomes` counter also leaks between tests
  (see note #2 on bead `sase-w3`).

## Design

### Principles

1. **One engine.** Every jump into an Artifacts pane (link rail, Links panel, relation
   jumps) goes through `_follow_artifacts_target` and its transaction.
2. **Plan, then commit once.** Each jump makes at most one query rewrite, verified
   against the target's own row before it is committed. The query bar always truthfully
   describes what is shown. Query history records exactly one `^` entry, which restores
   the original query and selection. This reuses the existing collapsed-transition and
   `LinkReveal` lens machinery.
3. **Land in context.** The rewrite shows the target inside its natural family, not
   isolated. The user's `limit:` is preserved and raised only if the family would not
   fit.
4. **Never lose a report, never treat loading as absence.** A jump fails only when the
   ref is genuinely dangling or the pane cannot load, and the toast says which.
5. **Explain every rewrite.** One toast shows the new query, what hid the target, any
   scope change, and the real keys that restore (`^`) and walk back (`ctrl+o`).

### The jump sequence

| Step          | When                                                                                                         | What happens                                                                                                                            | Toast?             |
| ------------- | ------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------- | ------------------ |
| Resolve       | always                                                                                                       | `entry_target_for_ref` on the destination pane. Re-resolved once the pane has loaded.                                                   | —                  |
| Scope         | the current scope is a specific project that excludes the target                                             | switch to the target's project, or to All projects if its project is unknown. Never narrow from All.                                    | yes (scope line)   |
| Select        | the target is visible                                                                                        | select it                                                                                                                               | no                 |
| Fold          | a collapsed fold hides it                                                                                    | expand the minimum fold, then select. No query change.                                                                                  | no                 |
| Acquire       | the row is not in the pane's _unfiltered_ inventory (capped snapshot, deep archive, outside Stitches window) | targeted hydration (existing `hydrate_ref`, moved _before_ any rewrite), then continue                                                  | yes (fetched line) |
| Context       | the query or limit hides it                                                                                  | commit `<context query> limit:<N>` from `host_reveal_context`, after `HostQueryProbe` verifies the context terms match the target's row | yes                |
| Identity      | no context, verification failed, or still missing after Context                                              | `<identity field>:<value>` with the preserved limit                                                                                     | yes                |
| Neutral       | still missing and the pane allows it (never Stitches)                                                        | `limit:all`                                                                                                                             | yes                |
| Honest report | everything missed                                                                                            | "No such artifact" for dangling refs, a load error, or "not in <Pane>"                                                                  | warning/error      |

### Context queries per artifact kind

The table is the answer to "what should the query become". Each row is built from facts
in the pane's unfiltered row, never by string-guessing when a real field exists. For
example, a bead's parent comes from `parent_id`.

| Pane · target                        | Context query (flat dialects join repeated keys with OR; boolean dialects use `OR`)                                                                                                                        | Label                  |
| ------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------- |
| Bead · phase `sase-16n.7`            | `id:sase-16n.*`. The epic renders as an expanded context header above all its phases (verified with a real pane).                                                                                          | `epic sase-16n`        |
| Bead · epic `sase-16n`               | `id:sase-16n id:sase-16n.*`, then expand the epic's fold after selecting it                                                                                                                                | `epic sase-16n`        |
| Bead · task/flag `sase-abc`          | `id:sase-abc`                                                                                                                                                                                              | `bead sase-abc`        |
| Agent · `sase-16n.7` (hood member)   | `name:sase-16n.*`, plus `OR name:sase-16n` only when an agent named `sase-16n` exists. The hood is the immediate dotted namespace, as in `models/agent_hoods.agent_hood`.                                  | `sase-16n hood`        |
| Agent · family shell `x--y`          | `family:x`                                                                                                                                                                                                 | `family x`             |
| Agent · hood root `foo` with members | `name:foo OR name:foo.*`                                                                                                                                                                                   | `foo hood`             |
| Agent · lone `foo`                   | `name:foo`                                                                                                                                                                                                 | `agent foo`            |
| File                                 | `agent:<creating agent>` when known, else `id:<logical id>`                                                                                                                                                | `files from <agent>`   |
| Plan / provider doc                  | `path:<ref payload>`, e.g. `path:202609/project_tags.md` (substring, so it also shows the doc's lifecycle rows)                                                                                            | `plan project_tags.md` |
| Stitch `repo@sha`                    | `repo:<repo> since:<day> until:<day>`, plus `sidecar:true` for sidecar repos, `merges:show` for merges, and the `project:` token only when the repo belongs to it. Identity escalation adds `sha:<sha12>`. | `<repo> · <day>`       |
| Patch (non-terminal)                 | `ancestor:<root of its parent chain>` (the whole stack)                                                                                                                                                    | `stack <root>`         |
| Patch (Submitted/Reverted/Archived)  | `name:<patch>`. A `name:` term lifts the hide toggles through `query/introspection.py`.                                                                                                                    | `patch <name>`         |

**Limit policy.** Let C be the current `limit:` cap (`extract_limit`). The context query
keeps `limit:C`. The provider may cheaply report `member_count`, for example from Beads'
`phases_by_epic` or one pass over an in-memory snapshot. If `member_count > C`, use the
smallest multiple of `ace.page_size` that is at least `member_count`, matching the
Ctrl+J paging convention. The context step never writes `limit:all`.

**Why not the old rungs.** Dropping the limit discards the user's paging for no benefit.
Minimal widening produces arbitrary lists, such as every open phase in the project.
Identity alone lands on a lone row with no context. These become fallbacks (identity,
neutral) or are deleted (limit drop, widening).

### Wildcard `*` filter syntax

`*` matches any run of characters, including an empty run, inside a string-field
property value. Matching stays case-insensitive.

- **Exact-match string fields** (`id`, `name`, `family`, `project`, `agent`, ...) and
  `sha` use anchored whole-value globs:
  - `id:sase-16n.*` matches `sase-16n.1` and `sase-16n.10`;
  - it does not match `sase-16n`, `sase-16nx`, or `sase-16n5`.
- **Substring string fields** (`path`, `assignee`, `model`, ...) use unanchored globs:
  `path:202609/*tags` matches anywhere in the value.
- **Enum, bool, int, and date fields** are unchanged. `*` is literal there, so it
  matches nothing.
- **Composition:** negation (`-id:sase-x7.*`), comma lists (`id:sase-16n,sase-16n.*`),
  and boolean dialects all work.
- **Scope:** matching runs in the shared Rust evaluator, so every Artifacts pane, the
  Patches pane, the live Agents tab, and `sase agent search` get it at once.
- **Escaping:** there is none, and `?` is not special. Neither is needed for ids and
  names.

The tokenizer, flat and boolean parsers, canonicalizer, and `sase.bead.filter_query`
already pass `id:alpha-1.*` through unchanged (verified). Only matching needs to change.

### The toast

The toast is information-severity with a timeout of about 8 seconds. It uses Textual
markup with every dynamic value escaped, the destination pane's accent color, and is
recorded in toast history through `AceApp.notify`. Target look for the user's example:

```
↪ Bead sase-16n.7
query  id:sase-16n.* limit:100
was    -status:closed limit:100 · hidden by -status:closed
epic sase-16n · ^ restore · ctrl+o back
```

- The new query line is bold in the accent color.
- The "was" line is dim, and the hiding terms are highlighted.
- The key names come from the live keymap: `prev_query`, and the binding that dispatches
  `_walk_link_trail_back`. They are never hard-coded.

Optional lines appear only when relevant:

- `scope  sase → All projects`
- `fetched  outside the loaded Stitch window`
- `Agents tab filter hides it — showing Artifacts ▸ Agent`

The "hidden by" reason comes from the `HostQueryProbe`:

- flat dialects list the tokens that exclude the row;
- boolean dialects say "hidden by your query";
- a filter that matches but a cap that cuts says `past limit:100`.

No toast appears when the jump only selected or expanded a fold. The pane's lens chip
keeps its existing `^`-to-return behavior and shows the context label:
`↩ sase-16n.7 · epic sase-16n`.

### Decisions and non-goals

- **No feature flag.** Every landed phase leaves jumps strictly more reliable than
  today. A pane without a context provider uses the identity fallback, so no phase
  exposes a half-built feature.
- **The Agents-tab main filter is never rewritten.** It is the primary dashboard. Hidden
  agents are shown in Artifacts ▸ Agent instead.
- **Context-query policy stays host-side in Python** (`src/sase/ace/link_reveal*.py`),
  beside the existing identity reveal:
  - It composes TUI pane-dialect query text from Python-owned pane profiles and snapshot
    facts.
  - Wildcard _matching_ semantics are core logic and go into `sase-core`.
  - Reopen this if a second frontend needs link-jump reveal: move the builder into
    `sase-core` then.
- **No config changes.** No new keybindings or config values, so
  `src/sase/default_config.yml` is untouched.
- **This epic completes `sase-w3`'s goal.** Update the `sase-w3` docs sections it wrote
  rather than adding parallel ones.

### Shared API additions (defined by the phases below)

- `ArtifactEntryNavigator.host_reveal_context(target) -> RevealContext | None`. The
  default is `None`, which means use the identity fallback.
- `ArtifactEntryNavigator.entry_target_project(target) -> str | None`. The default is
  `parts[0]` for panes whose identity leads with the project; `None` means unknown.
- `RevealContext` (frozen dataclass) has these fields:
  - `alternatives: tuple[tuple[str, str], ...]`: `(field, value)` pairs OR'd together;
  - `constraints: tuple[str, ...]`: extra ANDed tokens, for example the Stitches window;
  - `label: str`;
  - `member_count: int | None`;
  - `expand_target_fold: bool`.
- A pure renderer `render_reveal_query(context, profile, *, current_query, page_size)`
  produces the text:
  - flat dialects repeat the key token;
  - boolean dialects produce `(a OR b)`;
  - values go through `quote_value(..., keyed=True)`, which must keep `*` bare.
- `explain_hidden(probe, current_query, profile) -> HiddenReason`, with the kinds
  `filtered(terms)`, `filtered_boolean`, `limited(cap)`, `unloaded(hint)`, and `scoped`.
- `RevealOutcome` is the data the toast formatter consumes:
  - pane label, ref, new and old canonical queries;
  - hidden reason, scope change, context label;
  - hydrated flag, Agents-tab fallback flag.

## Phase core-glob: Wildcard matching in the sase-core query evaluator

Open the linked repo with `sase repo open sase-core -r "<why>"` and read its
`AGENTS.md`.

1. In `crates/sase_core/src/query/evaluator.rs`, change the String arm of
   `match_property` (`FieldValueKind::String | FieldValueKind::Enum`):
   - When the lowercased query value contains `*` and the field kind is `String`, match
     with a small allocation-free glob. Use two-pointer matching with backtracking on
     `*`; consecutive `*` collapse.
   - Anchor the glob to the whole value for `exact_match` fields and for `sha`. Leave it
     unanchored for substring fields.
   - Enum fields keep literal equality.
   - Values without `*` must take exactly today's code path.
2. Grep the query module for any other consumer of `exact_match` / `values_lower`: facet
   indexes, hash lookups, or fast paths that answer equality without `match_property`.
   Route glob values to the scanning path there, so no index answers a glob as equality.
3. Add tests in `crates/sase_core/src/query/tests.rs` for both flat and boolean
   profiles:
   - `id:alpha-1.*` matches `alpha-1.1` and `alpha-1.10` but not `alpha-1` or
     `alpha-10`;
   - `id:alpha-1,alpha-1.*`;
   - `-id:alpha-1.*`;
   - an unanchored glob on a substring field;
   - `sha:ab*12`;
   - an enum value containing `*` matches nothing;
   - case-insensitivity;
   - regression cases proving values without `*` behave identically.
4. Run `sase tool run check` in sase-core. It must pass; `sase_core_py` binding tests
   are included.

## Phase host-glob: Wildcard parity, pin bump, pushdown guard, and docs in sase

1. Pin and build:
   - Move `sase-core-revision.txt` past the core-glob commit with
     `just ratchet-core-revision`.
   - Make sure the linked sase-core checkout (`sase repo open sase-core`) includes that
     commit, so the local `sase_core_rs` rebuild has it.
2. Mirror the exact semantics in the Python reference evaluator
   (`src/sase/ace/query/profile_evaluator_matching.py::_match_text_field`). Extend the
   existing Rust-vs-reference parity tests with the core-glob cases.
3. Check whether `src/sase/ace/query/matchers.py` (the Patch matcher) still serves any
   production path. If it does, give its `name`/`project` equality the same glob
   semantics.
4. `src/sase/ace/tui/models/agent_live_query_pushdown.py` must treat any `PropertyMatch`
   value containing `*` as not pushable (fallback). Add a test proving `project:sa*` /
   `provider:*` never becomes an index equality filter.
5. Make sure the query-bar highlighters (`profile_highlighting.py` and the flat bars)
   style a `*` inside a value as part of the value, not as an error.
6. Update field hints:
   - Beads `id` hint: mention `id:<epic>.*`
     (`src/sase/ace/query_profile/profiles/_beads.py`).
   - Agents `name` hint: mention `name:<hood>.*`.
7. Docs:
   - Add a "Wildcards" subsection under Property Filters in `docs/query_language.md`,
     with a semantics table (exact vs substring vs enum) and examples.
   - Add a one-line pointer in the Beads filter docs (`docs/beads.md`).

## Phase seam: Never lose a pane report, never treat loading as absence

1. **Host seam.** While `_link_follow_dispatching` is set,
   `_complete_link_follow_request` records `(generation, state)` in a dispatch slot
   instead of discarding it. `_request_artifacts_target` then upgrades a returned
   `PENDING` to the recorded `SELECTED`/`MISSING`/`FAILED` for that generation. Update
   the docstrings in `_link_follow_transaction.py` / `_link_follow_targets.py`
   accordingly. This is the defense-in-depth guarantee that no pane can lose a report
   again.
2. **Pane contract.** Beads, Plans, Agents, and Files `request_entry_target` must return
   the state their synchronous refresh actually reached. Implement this once on
   `ArtifactEntryNavigator`:
   - `_complete_entry_request` remembers the last completed state for the pending
     request;
   - a helper runs the refresh and returns that state, or `PENDING` when nothing
     completed. Leave Stitches and Patches as they are.
3. **Truthful fold handling:**
   - Plans' `_pending_option_id` must only match rows present in the rendered option
     list. When the pending target sits under a collapsed banner, expand that banner
     first, mirroring `files_options.py`'s pending-banner expansion.
   - Agents gets the same pending-banner expansion.
4. **Loading is not absence.** In `_handle_missing_link_follow`, if the destination pane
   is loading (`pane_is_loading`), keep the transaction open and re-request the target,
   so the pane holds a pending target and reports after the load. Allow at most one
   re-request per load, so a stuck loader cannot loop.
5. **Re-resolve after load.** When a transaction resumes from `PENDING`, or receives
   `MISSING` from a pane that was not loaded at dispatch, re-run
   `_resolve_link_follow_target(ref, target)`. If the resolved identity differs (for
   example a bead chip's `task` becoming the real `phase`), update the transaction
   target and re-request once.
6. **Project scope:**
   - Add `entry_target_project(target)`:
     - Beads, Plans/providers, Patches: `parts[0]`;
     - Agents/Files: the row's `project` from the unfiltered snapshot;
     - Stitches: the repo's owning project if known, else `None`.
   - Replace `target_project_scope` in `_follow_artifacts_target` with this rule:
     - an All-projects scope is never narrowed;
     - a specific scope that excludes the target switches to the target's project;
     - if the project is unknown and the pane cannot resolve the target under the
       current scope, switch to All projects.
   - Record the scope change on the transaction so the planner and toast can report it.
   - Update `tests/ace/tui/test_link_follow.py`, which asserted narrowing from All.
7. **Hydrated rows become queryable.** After `install_hydrated_row`:
   - Beads and Agents must rebuild or refresh the filter and query index (bump the key
     `_ensure_filter_index` uses).
   - Plans must not disable filtering (`plans_filter_session.py`); re-index instead.
8. **Test hygiene.** Add an autouse fixture that snapshots and restores
   `_link_follow_outcomes` for the link-follow test modules. This fixes the leak
   recorded on `sase-w3`.
9. **Real-pane regression tests** (no fake panes):
   - Use the `AcePage` pattern from
     `tests/ace/tui/test_artifacts_beads_filtering.py::test_hide_closed_default_is_visible_and_clearable`.
   - Load with a project-aware loader:
     `monkeypatch.setattr(beads_pane, "load_beads_snapshot", lambda project, **_: snapshot(tmp_path, project=project))`.
   - The user's scenario:
     1. Scope to `alpha` and wait for the load.
     2. With the default `-status:closed limit:100`, call `app._follow_artifacts_target`
        for `bead:alpha-1.1`.
     3. Assert the transaction does not stay open and the phase ends selected. The
        engine is still the old ladder at this point.
   - Add sync-report tests for Files, Agents, and Plans.
   - Add a unit test for the host seam with a pane that reports synchronously and then
     returns `PENDING`.

## Phase planner: Plan-then-commit reveal engine with Beads context queries

1. Add `RevealContext`, `render_reveal_query`, the limit policy, and `explain_hidden` as
   pure functions (for example `src/sase/ace/link_reveal_context.py`). Keep every file
   under the repo's 1000-line limit.
2. Add `host_reveal_context` to `ArtifactEntryNavigator`, with a default of `None`.
3. Rework `_link_follow_ladder.py` / `_link_follow_transaction.py` into the jump
   sequence above:
   - Steps: Fold → Acquire → Context → Identity → Neutral → honest report.
   - Acquire moves hydration before any rewrite. It is triggered when
     `host_query_row_for_target(target)` is `None`.
   - Context verifies with
     `host_query_probe(target).matches(<context terms without limit>)` before committing
     through the existing `_commit_reveal_query` / collapsed-transition session, so
     there is exactly one `^` entry.
   - After `SELECTED`, honor `expand_target_fold`.
   - Delete the limit-drop and widening rungs, `minimal_widening_query`, and any helpers
     left unused (Symvision will flag leftovers).
   - Update outcome labels to `select`, `fold`, `context`, `identity`, and `neutral`.
4. Store the context label on `LinkReveal` and return a `RevealOutcome` from
   finalization. Keep a plain interim toast string; the toast phase makes it final.
5. Implement Beads `host_reveal_context` next to `host_query_row_for_target` in
   `beads_options.py`, following the table:
   - phase → `id:<parent_id>.*`, with `member_count` from `phases_by_epic`;
   - epic → `id:<id> id:<id>.*` with `expand_target_fold=True`;
   - task/flag → `id:<id>`.
6. Tests:
   - Pure tests for the renderer (flat and boolean), the limit policy, and
     `explain_hidden`.
   - Real-pane AcePage Beads tests:
     - (a) the user's scenario: the query becomes `id:alpha-1.* limit:100`, `alpha-1.1`
       is selected, a single `^` restores `-status:closed limit:100` and the prior
       selection, and the outcome reports hidden-by `-status:closed`;
     - (b) a closed epic target: the epic is selected and expanded;
     - (c) a task cut by a small `limit:`: context with a preserved or raised limit;
     - (d) a second jump while the lens is live: `^` still returns to the user's own
       query;
     - (e) a bead missing from the snapshot: hydrated, then revealed.
   - Update the fake-pane ladder tests whose expectations encoded the deleted rungs.

## Phase flat-panes: Agent and File context queries plus Agents-tab reveal

1. Artifacts ▸ Agent `host_reveal_context` (`agents_options.py`), per the table:
   - Hood detection is the immediate dotted namespace (share the logic of
     `models/agent_hoods.agent_hood`).
   - Family shell detection uses the `<family>--<suffix>` rule in
     `sase/agents/catalog/_family.py`.
   - The hood root is included only when that agent exists.
   - `member_count` comes from one pass over the unfiltered snapshot.
   - The dialect is boolean, so alternatives render with `OR`.
2. Files `host_reveal_context` (`files_options.py`): `agent:<creating agent>` with
   `member_count`, else `id:<logical id>`.
3. Agents tab: `_follow_loaded_agent` (`_link_follow_targets.py`):
   - Search the full loaded agent set, not just visible rows.
   - Reveal with `prepare_agent_navigation_target` / `reveal_agent_navigation_target`
     (`actions/navigation/_agent_reveal.py`), opening folds, groups, and panels.
   - On `TARGET_FILTERED`, fall through to the Artifacts ▸ Agent jump, with the
     outcome's Agents-tab-fallback flag set.
4. Real-pane tests:
   - agent hood context from a filtered state;
   - an agent cut by `limit:`, with the limit raised by `member_count`;
   - Files `agent:` context;
   - an Agents-tab folded agent revealed in place;
   - an Agents-tab-filtered agent landing in Artifacts ▸ Agent.

## Phase bounded-panes: Stitch, Plan/provider, and Patch context queries

1. **Stitches:**
   - `host_query_row_for_target` (`commits_collection.py`) reads the last _unfiltered_
     collection, plus any acquired facts.
   - `hydrate_ref` resolves the checkout for sidecar and linked repos from the project's
     repo inventory, not only from repos in the displayed result.
   - Stitches' `install_hydrated_row` stores the commit's facts (repo, day in the
     configured timezone, merge, sidecar) without injecting the row into the displayed
     result.
   - `host_reveal_context` builds the repo-and-day window. The re-collection it triggers
     fetches the commit, and the pane reports `SELECTED` when collection completes.
   - The Neutral step is disallowed for Stitches, because `limit:all` means a
     full-history collection.
2. **Plans and provider panes** (`plans_options.py`): `path:<ref payload>` context, with
   a fallback to the row's full identity. A non-empty context query triggers the
   deep-archive scan: the pane is loading, the transaction waits, and the pane reports
   after the scan. Confirm end to end that a provider pane (for example `ref:research`)
   accepts the rendered `path:` query. The provider parser and profile may disagree on
   keys and on repeatability; fix that if it is real.
3. **Patches** (`panes.py`):
   - Walk the `parent` chain in `_all_patches` to the root. Use `ancestor:<root>` for
     non-terminal targets and `name:<patch>` for terminal ones.
   - Check whether `select_entry_target` reveals a Patch inside a collapsed group. If it
     does not, implement `expand_fold_for_entry_target` for Patches.
4. Real-pane tests:
   - a stitch older than the default `since:24h` window (repo-and-day context,
     selected);
   - a sidecar-repo stitch;
   - an archived plan beyond the preview window;
   - a filtered-out Patch shown with its stack;
   - a terminal Patch hidden by the hide toggles.

## Phase toast: The reveal toast, lens chip, and user docs

1. Add a pure formatter, for example `src/sase/ace/tui/actions/_link_follow_toast.py`.
   - It turns `(RevealOutcome, key names, accent)` into `(title, markup message)`
     exactly as designed above.
   - Every dynamic value is escaped with `textual.markup.escape`.
   - Key names come from the live keymap through the existing `key_display_name`.
   - Wire it into `_finalize_selected_link_follow` with a timeout of about 8 seconds.
   - Snapshot-test the message text for each hidden reason and each optional line.
2. Unify failure copy: a dangling ref, a pane load failure, an unconfigured pane, and
   "not in <Pane>". Each gets a title naming the artifact and one reason line, with
   warning or error severity.
3. Lens chip: show the context label (`↩ <id> · <label>`) through `build_reveal_chip` in
   every pane's scope text.
4. Docs:
   - Replace "The Reveal Ladder" in `docs/ace.md` with "Link Jumps", covering the
     sequence table, the context-query table, the toast, and `^` / `ctrl+o`.
   - Update the reveal-lens and identity-field sections of
     `docs/artifacts_pane_contract.md`, and document the `host_reveal_context` and
     `entry_target_project` contract.
   - Cross-link the wildcard docs.
   - Update the `?` help modal text if it describes the old ladder.
5. Visual check: read the `tui_screenshot` reference memory, then capture the toast for
   the user's scenario with the live screenshot tooling. Iterate until it reads cleanly.

## Phase entry-points: One engine for every jump, plus the end-to-end matrix

1. In `_navigate_to_relation_target` (`actions/navigation/_tree.py`), route cross-pane
   relation targets and same-pane misses for non-Patches panes through
   `_follow_artifacts_target`. Use the target's canonical ref (`ref_for_target`) and a
   trail hop. Patches keeps its existing `reveal_entry_target` relation lens.
2. `_follow_chop_link`:
   - Also expand `service:scheduler`, mirroring the pending-selection path in
     `axe_display/_loader_items.py`.
   - Undo the expansions when the follow fails.
3. An unconfigured document kind (for example `research:` with no provider pane) must
   not land on Stitches. Report it with the unified failure copy instead.
4. Build the end-to-end matrix with `AcePage`, driving the real Links panel key path
   (`$0`, then the chip letter) for:
   - a closed bead phase, a closed epic, and a task beyond the limit;
   - an agent hood, an Agents-tab folded agent, and an Agents-tab filtered agent;
   - a file;
   - a deep-archive plan;
   - a stitch outside the window;
   - a filtered Patch;
   - a `job:` chop under a collapsed scheduler;
   - a dangling ref. Each case asserts:
   - the final pane, query text, and selected row;
   - exactly one `^` entry restoring the original query and selection;
   - the toast title and body;
   - that `ctrl+o` returns to the origin.
5. Perf: confirm the jump path adds no UI-thread I/O. Hydration stays pump-free, and the
   provider `member_count` passes are in-memory only. Also confirm the `$`-rail j/k
   bench (`tests/ace/tui/bench_tui_jk_link_rail.py`) does not regress.

## Verification for every phase

- Read the `lint_and_test` reference memory before finishing. Run `just check` through
  `sase tool run`, as that memory instructs.
- Any phase touching sase-core also runs `sase tool run check` there.
- Real-pane `AcePage` tests are required for behavior claims. Fake-pane tests alone are
  what let this bug ship.
