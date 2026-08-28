---
tier: tale
title: Cross-project `sase bead show`
goal:
  "`sase bead show <ID>` resolves a full bead ID against the owning enabled SASE
  project's bead store from any directory, local store first, with correct per-project
  plan roots and hosted URLs and no added cost when every ID is local."
size: medium
proposed_by: bbugyi200.athena.0fh
create_time: 2026-08-28 09:54:07
status: wip
---

# Cross-Project `sase bead show`

## Problem

`sase bead show <ID>` only ever reads one bead store: the one that
`sase.bead.cli_common.get_read_view()` resolves from the process CWD. Running

```console
$ cd ~/projects/github/sase-org/sase
$ sase bead show bob-cli-1e
Error: issue not found: bob-cli-1e
```

fails even though `bob-cli` is an enabled SASE project whose bead store holds
`bob-cli-1e`. Bead IDs are already globally unique by construction — the default issue
prefix is the project's display label (`sase.bead.prefix_policy.default_issue_prefix`) —
so the ID carries everything needed to find the owning project. Today that information
is thrown away.

## Goal

`sase bead show` resolves every requested full ID against the bead store of whichever
_enabled_ SASE project owns its prefix, from any directory, while keeping today's
local-first single-store semantics and adding zero cost to the common case where every
ID is local.

## Non-Goals

- `sase bead list`, `search`, `ready`, `blocked`, `stats` stay project-local. They are
  aggregate queries with no ID to route on; making them cross-project is a separate
  product decision.
- No mutation command becomes cross-project. Everything added here is read-only.
- No change to `sase-core`. See "Rust core boundary" below.
- Bare shorthand suffixes (`sase bead show 1e`) stay local-only unless the new
  `-P/--project` option pins a store; a dash-free suffix carries no prefix, so there is
  nothing to route on.
- No change to the `--format json` envelope keys.

## Key Facts Established By Exploration

Read these before writing code; they constrain the design.

1. **`show` never enters the Rust fast path.** `try_handle_bead_fast_path` in
   `src/sase/main/bead_fast_path.py` returns `None` for `argv[0] in {"list", "show"}`,
   so `sase bead show` always reaches argparse and `handle_bead_show` in
   `src/sase/bead/cli_query.py`. The Rust `handle_show` arm in
   `sase-core/crates/sase_core/src/bead/cli.rs` is unreachable from the CLI. **Do not**
   try to route this through `bead_cli_execute`.

2. **The single-store read invariant is about sibling workspaces, not projects.**
   `tests/test_bead/test_cli_read_single_store.py` asserts that a read from `project_2/`
   must not merge in `project/`'s store, and vice versa. That test must keep passing
   unchanged. The design below satisfies it because the fallback only ever consults
   _other enabled projects'_ canonical stores, and only after the local store has
   already missed.

3. **The pieces for cross-project store resolution already exist.**
   - `sase.core.project_lifecycle_facade.list_project_records(sase_projects_dir(), "enabled", include_home=False, projects_only=True)`
     returns one `ProjectRecordWire` per enabled project with `project_name` (canonical
     key), `display_name`, `aliases`, and `workspace_dir`. This is a single Rust call.
     `sase.bead.cli_sync_external._resolve_project_records` is the precedent to follow.
   - `sase.bead.store_locator.canonical_beads_dir_for_project(project_key)` returns that
     project's canonical readable beads dir (or `None`). It is documented read-only and
     never materializes.
   - `sase.bead.store_locator.open_bead_project_for_beads_dir(beads_dir)` opens a
     `BeadProject` for any supported store layout.
   - `sase.sdd.store.resolve_sdd_store` performs no writes and no network I/O, so
     resolving a foreign project's store is safe.
   - `sase.agent.bead_display.BeadIssueLookupSession` is the existing pattern for
     opening several bead stores once each across a batch of lookups.

4. **Bead ID grammar.** IDs match `^[^\s.]+-[0-9a-z]+(?:\.\d+)*$`
   (`sase.bead.prefix_policy` docstring, `sase.agent.bead_display._BEAD_AGENT_NAME_RE`).
   Prefixes may contain dashes (`bob-cli`), so the prefix is
   `bead_id.split(".", 1)[0].rsplit("-", 1)[0]`. `docs/beads.md` ("Bead ID Arguments")
   already documents dash-bearing prefixes.

5. **Every presentation input to `show` is CWD-derived.** All five helpers in
   `src/sase/bead/cli_detail_context.py` — `design_paths_are_relative`,
   `plan_reference_roots`, `artifact_reference_context`, `resolve_bead_page_url`,
   `resolve_bead_creator_url` — call `Path.cwd()`. Rendering a `bob-cli` bead with
   `sase`'s plan roots and hosted-link resolver would print wrong `PLAN` paths and wrong
   `PAGE`/`CREATED BY` URLs. **This is the part of the change that is easy to get
   silently wrong.** The render context must be per-entry.

6. **`workspace_context_for_plan_resolution(path)`** (`src/sase/sdd/plan_refs.py`)
   accepts any path and returns `(checkout_root, workspace_num)`, falling back to
   `(path, 1)`. It is what makes a per-origin context buildable.

## Rust core boundary

Per `sase/memory/rust_core_backend_boundary.md`, shared backend behavior belongs in
`../sase-core`. This change stays in Python, deliberately:

- The actual bead-store read is already in Rust and already takes an arbitrary store
  path (`bead_show_issue_detail(beads_dir, ...)` via `sase.core.bead_read_facade`).
  Nothing about _reading_ a bead changes.
- What this plan adds is **workspace and project discovery** — choosing which
  `beads_dir` to hand Rust. `sase-core`'s own `bead/cli.rs` header states "Python still
  owns workspace discovery", and every existing piece of that discovery
  (`resolve_beads_location`, `get_project_beads_dirs_for_project`, `resolve_sdd_store`)
  is Python. Putting prefix→store routing in Rust would require moving SDD store
  resolution across the boundary too, which is a much larger change than the feature
  warrants.

If a reviewer disagrees, the seam to move later is `origin_for_bead_id` alone; keep it
free of Python-only types so that stays cheap.

## Design

### 1. New module: `src/sase/bead/cross_project.py`

```python
@dataclass(frozen=True)
class BeadStoreOrigin:
    """One enabled project's canonical, read-only bead store."""
    project_key: str        # canonical key, e.g. "gh_bobs-org__bob-cli"
    project_label: str      # effective display name, e.g. "bob-cli"
    primary_workspace: Path
    beads_dir: Path


class AmbiguousBeadProjectError(ValueError):
    """Two or more enabled projects claim one bead prefix."""
```

Public functions:

- `bead_id_prefix(bead_id: str) -> str | None` — returns the project prefix, or `None`
  for a dash-free shorthand suffix or a token that does not match the bead ID grammar.
- `origin_for_bead_id(bead_id: str) -> BeadStoreOrigin | None` — routes an ID to its
  owning project's store. Raises `AmbiguousBeadProjectError` on a tie.
- `origin_for_project_ref(project_ref: str) -> BeadStoreOrigin | None` — resolves an
  explicit `-P/--project` value by canonical key, display label, or alias (casefolded),
  reusing the matching rule in `sase.bead.cli_sync_external._resolve_project_records`.

`origin_for_bead_id` resolution order:

1. `prefix = bead_id_prefix(bead_id)`; return `None` when it is `None`.
2. **Registry stage.** For each enabled `projects_only` record, build
   `refs = {record.project_name, effective_project_name(record), *record.aliases}` and
   match `prefix` exactly (bead prefixes are derived verbatim from the label, so exact
   comparison is the correct rule, not a casefolded one). Two or more distinct projects
   matching → `AmbiguousBeadProjectError` naming the candidates.
3. **Verification.** Resolve the winner's `canonical_beads_dir_for_project` and confirm
   its `config.json` `issue_prefix` equals `prefix`. If the store is missing, return an
   origin-shaped miss the caller can diagnose (see §4); if the prefix disagrees — a
   project with a deliberately customized prefix such as `gold` — fall through to stage
   4 rather than declaring a match.
4. **Store stage** (only when stages 2–3 produced nothing). For each enabled project,
   read `canonical_beads_dir_for_project(...)/config.json` and match on the stored
   `issue_prefix`. This handles customized and legacy prefixes. It is bounded by the
   number of enabled projects, skips unmaterialized stores silently, and only ever runs
   on the cross-project path. Ties → the same ambiguity error.

Every filesystem access in this module is a read. Nothing here may call
`materialize_sdd_store`, `resolve_beads_location(materialize=True)`, or any mutation
helper.

### 2. Per-ID store routing in `sase bead show`

`resolve_show_batch` in `src/sase/bead/cli_show_batch.py` currently takes one `view` and
resolves every ID against it. Change it to route per ID:

- Add a `ShowStoreRouter` (suggested home: `src/sase/bead/cli_show_router.py`) built on
  `contextlib.ExitStack`, modelled on `BeadIssueLookupSession`. It holds the local read
  view plus a lazily populated `dict[Path, BeadProject]` of foreign stores, so each
  store is opened at most once per invocation and closed on exit.
- `_ShowEntry` gains `origin: BeadStoreOrigin | None`, where `None` means "the local
  store answered".
- For each requested ID: try the local view first. **Only on `KeyError`** consult
  `origin_for_bead_id`; if it yields a store, open it through the router and retry
  there. A `ValueError` from the local view (ambiguous shorthand) is not a cross-project
  trigger and keeps today's behavior.
- Epic expansion (`<id>..`) routes the same way: `expansion_stem` first, then resolve
  the stem's owning store, then run `expand_epic_target` against _that_ store so a
  foreign epic's children come from the foreign store.
- When `-P/--project` is given, the pinned store replaces the local view for every ID in
  the invocation, including shorthand suffixes, and prefix routing is skipped entirely.

The local-first ordering is what preserves fact #2 and makes the happy path free: if
every ID resolves locally, `list_project_records` is never called.

### 3. Per-entry render context

This is the substantive refactor. Introduce a render-context bundle and resolve it per
origin instead of once per batch.

- In `src/sase/bead/cli_detail_context.py`, give each of the five helpers an optional
  workspace parameter that defaults to today's `Path.cwd()` behavior, so existing
  callers are unchanged.
- Add a frozen `ShowRenderContext` holding `relativize_design: bool`,
  `plan_roots: tuple[Path, ...]`, `reference_context_factory`, `creator_url_for`, and
  `page_url_for`, plus a memoizing resolver
  `render_context_for(origin: BeadStoreOrigin | None) -> ShowRenderContext` keyed by
  origin (`None` → the CWD context built exactly as today). Memoize so a batch of ten
  `bob-cli` beads builds one foreign context, and keep the existing laziness in
  `_show_batch_sections` (the reference context is still only built when an entry
  actually has `refs`).
- Replace the five flat parameters on `render_show_batch`, `build_show_batch_document`,
  `_render_full_batch`, `_show_batch_sections`, and `_render_json_batch` with the
  resolver. Update both call sites: `handle_bead_show` in `src/sase/bead/cli_query.py`
  and `_bead_link_target` in `src/sase/pager/resolve.py`. The pager picks up correct
  foreign contexts for free; it should also pass the router so a `bead:` link to a
  foreign bead resolves in the pager.
- `src/sase/bead_pages/rendering.py` calls `resolve_issue_detail` directly and is
  unaffected.

### 4. Diagnostics

- Prefix matches no enabled project → today's exact message,
  `Error: issue not found: <id>` (golden
  `tests/test_bead/golden/cli/show_missing.stderr` must not change).
- Prefix matches an enabled project but that project has no readable bead store → a
  distinct, actionable line naming the project and saying its store is not materialized
  on this machine, then the usual exit 1.
- Ambiguous prefix → one error naming every candidate project and pointing at
  `-P/--project`, exit 1.
- All of these flow through the existing `_ShowFailure` channel so partial batches keep
  printing the beads that did resolve.

### 5. Foreign-bead attribution in output

`render_issue_detail` (`src/sase/bead/cli_detail_render.py`) gains an optional
`project_label: str | None = None`. When set, it renders a `Project: <label>` field on
the existing `Type: … · Owner: …` metadata line so the reader can see which store
answered. It is passed **only** for entries whose origin is foreign, so `show.stdout`,
`show_style_*.ansi`, and every other existing golden stay byte-identical. Preserve the
`DetailStyle` contract: stripping SGR escapes from a styled render must still reproduce
the `PLAIN` bytes exactly.

The `--format json` envelope is deliberately untouched.

### 6. New CLI option

Add to `register_bead_show_parser` in `src/sase/main/parser_bead_queries.py`, placed
alphabetically between `-p/--pager` and `-s/--style`:

```
-P, --project PROJECT   Resolve every ID against one enabled project's bead
                        store, by key, name, or alias (default: the current
                        project, falling back to the project each ID's prefix
                        names)
```

`-P` is capitalized because `-p` is `--pager`; this matches the existing `-N/--no-links`
and `-T/--task-type` precedent. Per `sase/memory/cli_rules.md` the option is optional,
has a short alias, and the subcommand's `description` and `epilog` must be updated:
state the cross-project default, add an example such as `sase bead show bob-cli-1e` run
from another project, and add a `--project` example. An unknown project ref prints
`Error: project '<ref>' was not found` and exits 1, matching `sase bead sync-external`.

## Implementation Steps

1. Add `src/sase/bead/cross_project.py` (§1) with unit tests that monkeypatch
   `list_project_records` and `canonical_beads_dir_for_project` — the boundary
   `tests/test_bead_statuses_for_project.py` already patches. Cover: prefix extraction
   (dashed prefixes, dotted descendants, shorthand → `None`, malformed → `None`),
   registry match, custom-prefix store match, prefix disagreement falling through,
   ambiguity, and unknown prefix.
2. Add the `ShowStoreRouter` (§2) with its own tests: one open per store, foreign store
   closed on exit, local view consulted first.
3. Refactor the render context (§3) and update both `cli_show_batch` callers. Land this
   with the existing `show` tests still green _before_ wiring routing into the handler —
   it is a pure refactor and should not change any output.
4. Wire routing + `-P/--project` into `handle_bead_show` and the parser (§2, §6).
5. Add the `project_label` render field (§5).
6. Diagnostics (§4).
7. Docs: extend the `### sase bead show <id> [<id2> ...]` section of `docs/beads.md`
   with a cross-project subsection (resolution order, local-first rule, `-P/--project`,
   the ambiguity and unmaterialized-store errors, and the explicit note that
   `list`/`search`/`ready`/`blocked`/`stats` stay project-local). Update the
   `sase bead show` row in the `docs/cli.md` command table.

## Testing

New tests, mostly under `tests/test_bead/` (a new
`tests/test_bead/test_cli_show_cross_project.py` plus the two unit modules from steps
1–2):

- **Headline:** with two projects registered, `sase bead show <foreign-id>` from the
  local project's checkout renders the foreign bead.
- **Local-first:** the same ID present in both the local and a foreign store resolves to
  the local one.
- **Zero cost on the happy path:** when every ID resolves locally,
  `list_project_records` is never called (assert via a counting monkeypatch).
- **Sibling invariant:** `tests/test_bead/test_cli_read_single_store.py` passes
  unchanged — do not edit it.
- Mixed batch (one local + one foreign) renders both, in argv order, with the divider,
  and each block uses its own project's plan roots and design relativization.
- Ambiguous prefix → error naming both candidates, exit 1.
- Unknown prefix → byte-identical `Error: issue not found: <id>`.
- Known project with no materialized store → the distinct diagnostic, exit 1.
- Foreign epic expansion `<foreign-epic-id>..` expands from the foreign store.
- `--format json` for a foreign bead emits today's envelope shape.
- `-P/--project` pins the store, makes a bare shorthand suffix resolve in that project,
  and rejects an unknown ref with exit 1.
- Foreign entries render `Project:`; local entries do not (guards the goldens).
- Pager: a `bead:` link to a foreign bead resolves through `_bead_link_target`.

Then run `just check`. If its scoped selection escalates or looks unusual — this change
touches `src/sase/main/parser*.py` and `src/sase/pager/resolve.py`, which are broadly
imported — run `just check-full` through the `/sase_monitor` skill with the
`TESTING`/`TESTED` status pair instead of inline.

## Risks

- **Wrong render context for foreign beads** (fact #5) is the most likely silent defect:
  output that looks right but carries the local project's plan paths and hosted URLs.
  The mixed-batch test exists specifically to catch it.
- **Golden churn.** Any unconditional addition to the detail render breaks `show.stdout`
  and the `show_style_*.ansi` goldens. Keep `Project:` strictly conditional on a foreign
  origin.
- **Performance regression on the hot path.** `sase bead show` is used constantly.
  Nothing in `cross_project.py` may be imported or called until the local store has
  already missed; keep the imports function-local, matching the prevailing style in
  `sase/bead/cli_*.py`.
- **Accidental writes to a foreign store.** Everything added is read-only by
  construction; a review pass should confirm no new call reaches
  `materialize_sdd_store`, `get_project()`, or `auto_commit_bead_store`.
