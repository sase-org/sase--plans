---
tier: epic
title: Resolve full bead IDs across enabled projects for every bead command
goal: Every existing-bead argument to sase bead resolves a full ID independently of
  the caller's directory and executes against the owning project's correct store and
  context.
phases:
- id: core-routing
  title: Shared Rust bead target resolution
  depends_on: []
  size: medium
  description: 'core-routing: implement tested Rust routing policy and bindings, add
    a thin Python discovery adapter, and migrate show to the shared resolver while
    preserving its presentation contract.'
- id: operation-context
  title: Route reads and writes through one explicit operation context
  depends_on:
  - core-routing
  size: medium
  description: 'operation-context: provide routed read and mutation contexts, preserve
    store ownership and publication guarantees, and integrate the Rust fast dispatch
    with the shared target resolver.'
- id: command-coverage
  title: Integrate lifecycle, relationship, and history commands
  depends_on:
  - operation-context
  size: medium
  description: 'command-coverage: route every lifecycle, parented-create, dependency,
    reference, history, and apply-status bead argument; test complete command coverage
    and document the full-ID contract.'
- id: work-and-pages
  title: Route work launches, pages, and epic symbol checks
  depends_on:
  - operation-context
  size: medium
  description: 'work-and-pages: propagate the owning project through bead work and
    parent overrides, page operations, and epic-symbol discovery; verify launch and
    projection behavior with isolated fixtures.'
proposed_by: bbugyi200.athena.0l4
create_time: 2026-09-15 09:03:28
status: wip
bead_id: sase-116
---

- **PROMPT:** [prompts/202609/global_bead_id_resolution.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/global_bead_id_resolution.md)
- **BEAD:** [sase-116](https://github.com/sase-org/sase--beads/blob/main/pages/sase-116/README.md)

# Full bead IDs work from any directory

## Problem and outcome

From outside the SASE project's directory, the user ran:

```text
sase bead close sase-xe.16.11.7.15.7 -r <reason>
Error: issue not found: sase-xe.16.11.7.15.7
```

`sase bead show` can already locate that project's beads. The outcome of this work is
that every command argument naming an existing bead can locate a full bead ID in any
enabled project's available canonical store. A valid `close` invocation against that
fixture must close the target in its owning project and publish there. Ordinary domain
validation still applies: finding a bead does not make an invalid lifecycle transition,
unsupported bead type, or forbidden relationship valid.

This is an epic because it crosses the Rust API, Python store transactions, numerous CLI
handlers, and work-launch context. Each medium phase is direct implementation work. The
last two phases can run independently after the shared operation API is complete. The
landing agent verifies the combined tree and the complete command inventory.

## Evidence and implementation anchors

- `src/sase/bead/cli_show_batch.py` and `cli_show_router.py` try the local view, then
  use `cross_project.py` for enabled-project fallback. They preserve per-project
  rendering, `show --project`, mixed read batches, and `..` descendant expansion.
- `cross_project.py` matches registry keys, display names, aliases, and the store's
  `issue_prefix`. Its current origins are explicitly read-only. Store discovery is
  centralized in `store_locator.py`, `workspace.py`, and `cli_location.py`.
- Most Python commands call `get_read_view()` or `bead_store_mutation()` without target
  context. Both select from the caller's current directory; the latter can
  auto-initialize a store before discovering that the requested bead is elsewhere.
- `src/sase/main/bead_fast_path.py` has a separate early dispatch. Its
  `_resolve_fast_path_context(argv)` currently ignores the targets in `argv`. `close`
  currently uses the Python lane, while commands such as `open`, `update`, `rm`, `dep`,
  and `ref` may execute in Rust. Reference mutation's Python fallback re-enters that
  Rust lane with `materialize=True`.
- `sase-core`'s `crates/sase_core/src/bead/cli.rs` receives resolved read paths and one
  write path. Its `read.rs::resolve_issue_id_in_issues` returns a string containing a
  dash unchanged, even if no such bead exists. Canonicalization alone cannot be an
  ownership or existence check.
- Publication in `cli_common.py` and fast-path `_apply_mutation_side_effects` can
  rediscover the caller's store. Close's `_settle_close_task_gates` similarly infers the
  project from cwd, while its symbol guard scans a working tree.
- `cli_work_context.py`, `cli_work_entry.py`, `cli_pages.py`, and `cli_epic_symbols.py`
  also derive relevant project or checkout state from cwd.
- The existing `work --wait bead=ID` input reaches `axe/run_agent_wait_deps.py` and
  `scripts/sase_chop_wait_checks.py`; both currently obtain closed IDs only from the
  waiting agent's project. Resolving the work target alone would leave a foreign bead
  wait blocked even after its bead closes.

Before accessing `sase-core`, each worker must use `/sase_repo` and
`sase repo open sase-core -r "<phase-specific reason>"`, then use only the returned
checkout. Rust paths below are relative to that checkout. Keep shared resolution policy
in `crates/sase_core`; Python owns discovery/materialization, ownership checks,
publication, presentation, and launch orchestration through thin adapters.

## Resolution and transaction contract

1. **Local first.** A target present in the current effective store keeps using that
   store. Do not consult the project registry on an ordinary successful local lookup. A
   missing local store must not prevent full-ID fallback or trigger initialization.
2. **Full IDs enable fallback.** On a local miss, consult enabled project records only,
   using the existing lifecycle facade's enabled, projects-only, non-home selection.
   Discover only each project's canonical effective store; do not merge sibling
   workspaces, legacy stores, or arbitrary filesystem directories. Closed beads are
   eligible; command-specific status/type restrictions are applied after resolution.
3. **Existence decides the result.** Project names, aliases, and configured prefixes are
   discovery hints, not proof that a bead exists. Check exact full-ID membership.
   Include custom/hyphenated prefixes and historical IDs whose prefix no longer matches
   current registry/config metadata. If fallback finds the exact ID in multiple distinct
   eligible stores, fail with the candidate project keys and paths. Deduplicate aliases
   and records that point at the same physical store. Do not pick the first prefix match
   and report a false not-found when another enabled store holds the ID.
4. **Shorthand remains local.** Resolve suffixes in the caller's effective store, or in
   the explicitly pinned store for `show --project`. A foreign full ID elsewhere in the
   argument list must not silently change the interpretation of a shorthand. Preserve
   ambiguity diagnostics, full IDs in output/relationships, and exact case semantics.
   `show --project` remains a strict pin with no fallback outside that enabled project;
   its name/alias matching remains compatible.
5. **Unavailable stores have useful errors.** Discovery is read-only and does not clone,
   initialize, or pull every enabled project. A known owner's unmaterialized or
   unreadable store receives an actionable project/store diagnostic. An unrelated
   unavailable project does not mask a match in a readable store. If no match can be
   established, distinguish unavailable relevant stores from an ordinary missing ID.
   Subsequent write preparation may use the established materialization/refresh path for
   the selected owner; it must not create a replacement empty store for a missing ID.
6. **Resolve the operation before writing.** Return a structured route carrying the
   canonical bead ID, project identity, read-store identity, and owning checkout
   context. Revalidate IDs in the authorized write store under the normal lock after any
   refresh/materialization. Never write directly through a read-only origin. Keep
   foreground CLI mutations usable from arbitrary cwd under existing user-origin rules;
   preserve explicit read-only and machine-origin ownership restrictions.
7. **Mutation batches stay within one store.** Resolve every target before the first
   event or commit. Multiple IDs belonging to the same foreign store work normally. A
   batch spanning stores fails before any mutation, naming the projects and asking the
   caller to split the command by project. This preserves `update`'s atomic single-store
   semantics without introducing distributed transactions. Preserve existing validation
   and no-op rules within the selected store. `show` retains mixed-project partial
   results; `work` retains its sequential per-target behavior.
8. **Relationships keep their domain constraints.** Resolve both sides of `dep add` and
   every operand of `dep rm`; reject cross-store edges explicitly before writing. A full
   parent ID in `create -T 'phase(ID)'` or `plan(path,ID)` selects that parent's store
   for the child. Resolve plan-work parent overrides similarly, keeping parent and child
   together; never silently reparent an existing plan in another store. Artifact
   references remain typed references, not dependency IDs.
9. **Context follows the owner; input paths follow the caller.** Lock, mutation, commit,
   push, publication verification, refresh, gate settlement, plan lookup, hosted links,
   configuration, and launch VCS context use the selected project. User-supplied
   relative files (`@reason.txt`, descriptions, design/plan paths) are interpreted
   relative to the original invocation directory before routing. Use explicit context
   parameters rather than process-wide `chdir` or environment swaps.

## Required command coverage

Inventory the registrations in `main/parser_bead*.py` and `ops/commands/bead.py` at
implementation time and account for every existing-bead input, including flags and
nested subcommands. The current inventory is:

| Surface                  | Existing-bead inputs and owner-sensitive behavior                                                         |
| ------------------------ | --------------------------------------------------------------------------------------------------------- |
| `show`                   | Each ID, `ID..`, explicit `--project`, all formats and pager links                                        |
| `close`                  | All IDs; resolve epic before `--phases`; force/reason/note/no-push; symbol guard and task-gate settlement |
| `open`, `note`, `+1`     | Target ID; note append/edit/remove; corroboration and author attribution                                  |
| `update`, `rm`, `snooze` | All IDs; update note/file-value paths; snooze and cancel; full preflight                                  |
| `create`                 | Parent inside phase/plan type syntax; child creation and destination plan-reference storage               |
| `apply-status`           | `bead_id`; preserve typed operation results and existing status validation                                |
| `dep add/rm`             | Source ID and every dependency ID; same-store relationship validation                                     |
| `dep list/tree`          | Optional scoped ID; tree/relationship reads use its store                                                 |
| `ref add/rm/list`        | Owning bead ID; `list --resolve` uses owner artifact context; preserve artifact-ref grammar               |
| `history`                | Optional ID in lost-notes mode, required ID in normal mode; restore uses the same store as preview        |
| `work`                   | Each bead target, parent overrides for plan-file targets, existing bead wait references                   |
| `pages url`              | Bead ID, existence check, owner sidecar remote and branch                                                 |
| `pages refresh`          | Optional `--bead`; dry run and `--write` use the target lineage's project                                 |
| `epic-symbols`           | Optional ID; scan its appropriate checkout's Justfile                                                     |

Commands/forms with no bead selector retain their existing project scope: bare/list,
search (its query remains text), ready, blocked, stats, init, sync, sync-external,
doctor, resolve-conflicts, onboard, task-type, and unscoped dep/ref/history/pages/symbol
operations. No new flags or global all-project listing behavior are required. Do not
treat a title, note, reason, artifact reference, or file path containing a dash as a
command target. Keep show-only `..` syntax confined to show.

## Phase core-routing

1. Add a focused Rust routing module and serializable request/result types alongside
   `bead/read.rs` and `bead/wire.rs`. Accept normalized requests and candidate store
   descriptors from host discovery. Put exact membership checks, local-first/full-ID
   policy, deduplication, ambiguity, and single-store batch validation in the core. Keep
   existing per-store bead APIs explicitly store-scoped.
2. Export the API through `crates/sase_core_py/src/lib.rs` and add a thin facade under
   `src/sase/core/`. Distinguish missing, ambiguous, unavailable, and incompatible
   stores in structured errors rather than depending on exception-message parsing.
3. Refactor `bead/cross_project.py` into the discovery/adapter layer; reuse the enabled
   project lifecycle facade and canonical locators. Preserve compatible entry points for
   existing consumers. Memoize registry and store snapshots per invocation; close views
   predictably. Do not introduce a durable global index or background scan.
4. Move show's ID/store choice to the shared route. Keep its rendering, ordering,
   deduplication, explicit pin, mixed-results exit status, page/artifact links, and
   descendant expansion behavior. Extend its current tests to use real existence
   fixtures, including duplicate-prefix projects with distinct IDs and duplicate IDs.
5. Add core and binding tests covering deep IDs such as `sase-xe.16.11.7.15.7`, renamed
   prefixes, local/shorthand precedence, unavailable stores, ambiguity, and batch
   validation. Include a local-hit test that proves no enabled-project discovery.

Acceptance: callers can obtain a verified, owner-qualified target through Rust-backed
resolution; show uses that API and keeps its public output contracts.

## Phase operation-context

1. Extend the shared CLI context API around `cli_common.py`/`cli_location.py` so a
   resolved route can open a read view or a mutation transaction without rediscovering
   the caller's project. Carry both original invocation cwd and selected owner/store.
   Keep unchanged call sites working until the later integration phases migrate them.
2. Use existing ownership APIs in `workspace_provider/ownership.py` and sanctioned
   writable-path helpers when obtaining writable foreign stores. Verify the read and
   write locations belong to the same project. Respect plain-checkout read-only contexts
   and explicit machine mutation rules without treating an ordinary foreign user CLI
   target as inherently forbidden.
3. Thread the route through auto-commit, push, publication verification, and refresh.
   Keep the transaction lock and commit ordering, no-push behavior, and unchanged/no-op
   behavior. A routed sidecar publication failure must produce nonzero exit status;
   never print success after committing or verifying a different store. Preserve
   established in-tree versus local/sidecar commit behavior.
4. Integrate `main/bead_fast_path.py` and the Rust CLI planner. Extract target operands
   using command-aware parsed arguments, sharing Rust parsing where available; never
   scan arbitrary argv strings for ID-looking text. Unsupported syntax defers without
   materializing a caller store. A resolved routing error is terminal, not a reason to
   retry against the original cwd. The `ref` materializing fallback must use the same
   route. Fast execution and its mutation summary/publication need one owner.
5. Provide tested API entry points for canonical targets, optional scopes, parent-led
   creation, and multiple operands. Publish the interface in phase notes for the two
   downstream phases; keep mutable routing state scoped to one invocation.

Acceptance: isolated transactions and fast-path tests show that events, lock, commit,
publication verification, and refresh use only the selected project, and errors leave
the caller store untouched. The early-dispatch and argparse lanes agree on routing.

## Phase command-coverage

1. Migrate lifecycle handlers in `cli_crud_*.py`, dependency handlers, references,
   `cli_history.py`, and `ops/commands/bead.py` to the phase-two API. Resolve embedded
   create parents before choosing the write store. Normalize all IDs before batch
   effects; ensure `history --restore` cannot preview one store and write another.
2. Carry owner project identity through close's gate settlement. Pass the appropriate
   checkout to the existing close symbol guard: keep the current checkout on local
   operations and the owning checkout on foreign operations. The sibling phase owns the
   standalone epic-symbols handler. Retain close resolution/conflict rules, descendant
   checks, phase selection, force restrictions, and append-only evidence.
3. Resolve reference presentation and plan/design storage using the owner's context
   while preserving original-cwd file inputs. Keep dependency edges store-local and
   distinguish a found-but-cross-store operand from a missing operand.
4. Add a parameterized command-inventory regression suite over temporary projects and
   real stores. Cover every row this phase owns through public dispatch, plus explicit
   fast/slow-path parity where supported. Do not rely only on mocks of `get_project`.
5. Update `docs/beads.md` and existing help descriptions with the resolution, shorthand,
   unavailable-store, and mixed-write-batch rules and an outside-directory close
   example. Do not add flags. Memory changes are not part of this plan.

Acceptance: every covered command works with a valid full ID in another enabled project;
invalid/mixed batches leave all stores unchanged; published mutations and close gate
settlement refer to the owner. Add fixtures for nested, closed, and task beads as
appropriate to each command's existing domain rules.

## Phase work-and-pages

1. Migrate `cli_work_entry.py`, `cli_work_context.py`, task/epic launch helpers, and
   plan-file parent handling. For each target, propagate the resolved project into
   VCS/Patch context, project configuration/xprompt lookup, SDD paths, launch locks,
   preclaims, launch requests, publication, and rollback. Target discovery alone is
   insufficient. Retain authored target order and one JSON result per processed target;
   a route must not leak into the next target or a local plan-file argument.
2. For a plan-file parent override, resolve the original file path first, then use the
   parent's owning project when creating the child. Verify existing linked plans and
   parents are compatible before writing or launching. Preserve `top-level`, dry-run,
   resume, collision, and existing launch-confirmation behavior. Full bead IDs in
   `--wait` must not select the mutation destination or become cross-store dependency
   edges. Add a shared Rust-backed status adapter using phase one's routing policy for
   the runner's `axe/run_agent_wait_deps.py` and the periodic
   `scripts/sase_chop_wait_checks.py` consumer. Both must test the requested full IDs
   against their owning enabled projects, with per-pass store caching, and direct
   bead-sync hints to those owners. Preserve waiting-marker formats and the existing
   wait grammar: unknown, unavailable, disabled, or ambiguous foreign targets stay
   unresolved, not completed. Do not add eager missing-bead rejection at launch where
   current waits can refer to future beads. Agent-name waits keep their existing scope.
3. Migrate `cli_pages.py` so scoped refresh (preview/write) and URL resolution use a
   verified target and the owner's store, project, primary root, remote, and branch. Do
   not fabricate a hosted URL merely because a full ID passes syntax validation.
4. Migrate `cli_epic_symbols.py` so a scoped foreign ID reads the owning checkout's
   Justfile. A local bead still scans the user's current working tree. Keep unscoped
   scans as they are. Coordinate with the other phase only through the shared context
   API; do not independently change its close handler.
5. Test public `work --dry-run` and mocked launch execution with real bead fixtures,
   owner-specific configurations, multi-project target order, and owner-specific
   rollback/publication. Tests must not launch real agents. Prove a foreign bead wait
   stays blocked while open and releases after closing in both the runner and periodic
   evaluator, including missing/unavailable cases and mixed-owner waits. Test page URL
   host/branch, scoped page writes, and symbol discovery with distinct caller/owner
   fixtures.

Acceptance: full-ID work, page, and symbol operations behave as if invoked in the
appropriate owning checkout; caller-relative file inputs and existing dry-run/no-push
semantics remain intact.

## Verification and landing

Use isolated temporary project registries and bead repositories. Never reproduce the
reported close against the user's live bead and never use live stores for test writes.
Cover these dimensions across the command suite, choosing meaningful combinations:

- Caller in a different enabled project, nested caller directory, and outside every
  project with no local bead store; no accidental initialization in those directories.
- Target in each supported SDD layout (in-tree, local, separate repository, combined
  plans/beads sidecar, and split beads sidecar); only canonical enabled stores enter
  fallback. Preserve existing explicit local/environment-selected store precedence.
- Custom/hyphenated/renamed prefixes; deep descendant IDs; closed beads; duplicate
  project aliases or store paths; exact-ID ambiguity; disabled and hidden projects;
  missing IDs and unavailable relevant versus unrelated stores.
- Same-store foreign batches, mixed-store rejection, mixed shorthand/full IDs,
  dependency pairs, and a missing final operand. Assert absence of events/commits in
  every store after routing/preflight rejection.
- A user-supplied reason or note containing bead-looking text and `@relative-file`
  inputs, `--option=value` forms, and existing short options do not confuse routing.
- Owner-specific commit and verified publication, no-op and no-push behavior,
  publication failure, close gate settlement, and no effects in the caller project.
- Preserve existing show tests in `test_cli_show_cross_project.py`,
  `test_cli_show_router.py`, and the multi/expansion suites; shorthand coverage in
  `test_cli_id_shorthand.py`; bulk lifecycle, dependency/reference, history, pages,
  work, and `tests/main/test_bead_fast_path*.py` suites.

Each implementation worker must read `lint_and_test.md` through `/sase_memory_read` and
run `just check` in the SASE repo after changes. Build/install the updated Rust binding
with the repository's existing development install workflow before Python integration
checks. Run `just check` or `./scripts/check.sh` at the opened sase-core root for Rust
changes, including PyO3 binding tests; `cargo test -p sase_core` alone is insufficient.
Use `/sase_monitor` for long-running verification. Do not manually change release
versions or bypass the required Rust backend with Python fallbacks.

The landing agent runs the combined command matrix, checks parser registrations for any
newly added bead-ID surfaces, and runs SASE `just check-full` through `/sase_monitor`,
plus the Rust repository's required full checks. The work is complete only when all
ID-bearing surfaces are accounted for, owner-context tests pass, and the reported
nested-ID close succeeds in an isolated outside-directory reproduction.
