---
tier: tale
title: Scope the Artifacts Patches sub-tab to the current project
goal:
  The Patches sub-tab honors the shared Artifacts project scope through a visible,
  non-persisted `project:` query term, so first open seeds the current project and a
  pick made on any Artifacts pane reaches Patches too.
size: medium
proposed_by: bbugyi200.athena.07a
create_time: 2026-08-18 20:40:56
status: wip
---

# Plan: Scope the Artifacts Patches sub-tab to the current project

## Problem

The `sase-pw` epic gave SASE one **current project** and made every project-filterable
TUI surface seed from it on first open. On the Artifacts tab that seed flows through the
shared scope state `AceApp.artifacts_project_scope`. The Patches sub-tab was missed: its
filter is the Patch boolean query (`AceApp.query_string`), and nothing ever writes the
seeded project into it.

`docs/ace.md:3435` even lists the covered panes explicitly — "the shared Artifacts
project scope (Stitches, Beads, Plans, Files)" — with Patches absent, while the Patches
help modal at `src/sase/ace/tui/modals/help_modal/patches_artifact_bindings.py:76`
already promises `("+PROJECT / project:NAME", "Filter (seeds current)")`. The help text
is a promise the code does not keep.

### Confirmed reproduction

Run on a clean tree at `a317a2e35` with `just install` already done. Scratch test
(deleted after the run) stubbed `_collect_artifacts_project_choices` with two enabled
projects (`alpha`, `beta`) and `current_project="alpha"`, then opened
`AcePage(query="!!!", initial_tab="patches")` and waited for the seed:

```
SCOPE: alpha            # AceApp.artifacts_project_scope
COMMITS: Alpha          # CommitsPane.filters.project  -> seeded
QUERY_STRING: '!!!'     # AceApp.query_string          -> NOT seeded
CANONICAL: '!!!'
```

Stitches, Beads, Files, Plans, and the document provider panes are all scoped to
`alpha`; the Patch list still shows every project's patches. Pressing `p` on Patches
then highlights **All projects**, because the picker reads the Patches project scope out
of the query (`actions/artifacts.py:401-405`), which disagrees with the shared scope
every other pane is using.

### Three distinct gaps

1. **No seed.** `_ensure_artifacts_project_choices`'s worker
   (`src/sase/ace/tui/actions/artifacts.py:304-364`) resolves the seed and calls
   `_set_artifacts_project_scope(...)`, which fans out to
   `ArtifactsView.set_project_scope` (`widgets/artifacts/view.py:294-321`). That fan-out
   covers `CommitsPane`, `ArtifactPlaceholderPane`, `ArtifactsBeadsPane`,
   `ArtifactsFilesPane`, and `ArtifactsDocumentsPane`. There is no Patches leg, and
   nothing touches `query_string`.

2. **Cross-pane picks skip Patches.** In `_open_artifacts_project_picker`'s `_on_picked`
   (`actions/artifacts.py:380-399`) the query rewrite is gated on
   `self.current_artifacts_pane_key == "patches"`. Pick a project while on Beads and
   every pane re-scopes except Patches.

3. **The lazy inventory read never fires from Patches.**
   `_sync_active_artifacts_entry_state`
   (`src/sase/ace/tui/actions/artifacts_navigation.py:72-80`) returns early when the
   active pane is `patches`, so it never calls `_ensure_artifacts_project_choices()`.
   Today this is masked because `DEFAULT_ARTIFACTS_SUBTAB` is `"stitches"`
   (`_artifact_tab_model.py:24`) and `--tab patches` is only a legacy alias for
   `artifacts` (`src/sase/main/parser_ace.py:63-73`), so startup always lands on
   Stitches first. It is not masked when the user jumps straight to Patches from another
   tab — e.g. `actions/navigation/_modals.py:39` sets
   `current_artifacts_subtab = "patches"` directly.

## Hazard the implementation must not trip over

`_run_mount_state_loads` persists the startup query:
`await asyncio.to_thread(self._save_current_query)` at
`src/sase/ace/tui/actions/_startup_loads.py:164`, and `_save_current_query`
(`actions/patch/_query.py:30-34`) writes `canonical_query_string` to
`~/.sase/last_query.txt`. The CLI's default query is
`load_last_query() or load_first_saved_query("patches") or "!!!"`
(`src/sase/main/parser_ace.py:18`).

The async seed and that startup save race each other. If a naive implementation lets the
seeded query reach `last_query.txt`, the seed becomes **permanently sticky**: the next
cold start reads `!!! AND project:Alpha` back as an explicit term,
`get_sole_project_filter` returns `Alpha`, the seed branch is skipped forever, and the
current project can never re-scope Patches again. That is worse than today's bug. The
seed must never be persisted.

## Design decisions

- **Rewrite the visible query, do not add a hidden filter.** The picker already rewrites
  the Patch query, the token shows in the persistent `PatchFilterBar`, and
  `docs/ace.md:293` already documents that "at ACE startup the async Artifacts seed can
  add a visible project token" for Stitches. Patches gets the same treatment. An
  invisible filter would make the filter bar lie.

- **Write the project ref label, not the ProjectSpec key.** Patch `project:` matching
  uses `Patch.project_query_name` (`src/sase/ace/patch/models/patch.py:176-185`), which
  is the configured `PROJECT_NAME`. The picker path already converts with
  `choices.project_ref_display.label_for_ref(project_key)`; the seed must do the same.
  Never write a raw ProjectSpec key into user-visible query text.

- **All project-term editing goes through `rewrite_project_scope`**
  (`src/sase/ace/query/project_scope.py`). It is token-preserving and spelling-aware
  (`+alpha` stays a sigil, `project:alpha` stays a property), so the seed is agnostic to
  the rest of the query.

- **Seed only when the query carries no project term at all.** `get_sole_project_filter`
  ignores `NOT`/`OR` branches, so gating on it alone would let `rewrite_project_scope`
  rewrite `NOT project:beta` into `NOT project:Alpha` and silently invert a user's
  explicit filter. The gate must be "no project token anywhere, at any depth or
  polarity".

- **The seed is one-shot, session-only, and never persisted.** It does not push query
  history, does not toast, and does not reload patches from disk.

- **A user commit persists exactly what the user committed.** If the user edits the
  query while the seeded token is in the prefill and submits it, the token becomes
  theirs and is persisted. The escape hatches are deleting the token or pressing `p` →
  **All projects**, which already removes it.

- **Precedence stays in one place.** The Patches query follows whatever
  `_resolve_artifacts_scope_seed` (`actions/artifacts.py:229-247`) already returns:
  current project when enabled, then sole enabled project, then all projects, with
  `ace.current_project.seed_filters: false` skipping only the current-project step. No
  second precedence ladder.

- **No new `seeded` badge.** The Agents tab marks its seeded query
  (`actions/agents/_search_query_seed.py`) because that query is otherwise invisible.
  The Patch query is always visible in `PatchFilterBar`, and the Stitches seeded token
  carries no badge either. Out of scope; note it and move on.

## Legacy baggage to route around (do not extend it)

The Patch query surface is the oldest one in ACE. The implementation must not entangle
itself with any of this:

- `!!!` (error-suffix shorthand) is the bundled default query at
  `src/sase/main/parser_ace.py:18` and `src/sase/ace/tui/app.py:267`. It is slated for
  deprecation and removal. **Do not** key the seed off "the query is the bundled
  default", **do not** add new `!!!` literals, and **do not** add tests that only pass
  because the query happens to be `!!!`. The seed condition is purely "no project term
  present"; it must behave identically for `!!!`, `""`, `'"needle"'`, and any future
  replacement default. At least one new test must use a non-`!!!` query so the suite
  survives that removal.
- `last_query.txt` is deliberately un-namespaced and is always the Patches pane's query
  (`src/sase/ace/saved_queries.py` module docstring). Keep it that way.
- `changespecs` / `ChangeSpec` aliases (`_load_changespecs`,
  `start_agent_from_changespec`, `LEGACY_ARTIFACTS_SUBTABS`, `--tab changespecs`) are
  compatibility shims. Do not add new ones and do not rename existing ones here.
- The Stitches-only `a` (`stitches_toggle_all_projects`) compatibility action has no
  Patches equivalent and does not need one; `p` → **All projects** is the Patches escape
  hatch.

## Implementation

### 1. Query helper — `src/sase/ace/query/project_scope.py`

Add a public predicate beside `project_scope_of`:

```python
def has_project_scope(query: str) -> bool:
    """Return whether *query* carries any project term, at any depth."""
    terms, nested = _project_terms(query)
    return bool(terms) or nested
```

Add it to that module's `__all__` and re-export it from `src/sase/ace/query/__init__.py`
alongside `project_scope_of` / `rewrite_project_scope`. This is the seed gate, and it is
what distinguishes "no project term" from "a project term `get_sole_project_filter`
happens to ignore".

### 2. App state — `src/sase/ace/tui/actions/_state_init_runtime.py`

Next to the existing `artifacts_project_scope` initialization (around line 169), add:

- `self._patch_query_scope_seed_attempted = False` — the one-shot latch, mirroring
  `_agent_search_query_seed_attempted`.
- `self._patch_query_scope_seed_baseline: str | None = None` — the canonical query as it
  was immediately _before_ the seed applied, or `None` when the live query is not a
  seed. This is what the startup save persists.

Declare both on `ArtifactsMixin` (`actions/artifacts.py`) and on `PatchQueryMixin` /
`PatchFilterSessionActionsMixin` where the type checker needs them, following the
annotation style already used in those classes.

### 3. Apply helper — `src/sase/ace/tui/actions/artifacts.py`

Add one method to `ArtifactsMixin` that owns every Patches project-term write:

```python
def _apply_patches_project_scope(
    self,
    project_ref: str | None,
    *,
    seeded: bool,
    notify: bool = True,
) -> None:
```

Behavior:

- `seeded=True`:
  - Return immediately if `self._patch_query_scope_seed_attempted` is already set; set
    it before doing anything else (an empty resolve still consumes the session's one
    attempt, exactly like `_maybe_seed_agent_search_query`).
  - Return if `project_ref` is falsy — a seed never _removes_ a term.
  - Return if `has_project_scope(self.query_string)` — an explicit term of any polarity
    or depth wins.
  - Return if a live filter session is in flight (`self._live_patch_query is not None`),
    so a startup seed never clobbers text the user is already typing.
  - Compute `rewritten = rewrite_project_scope(self.query_string, project_ref)`; return
    if it is `PROJECT_SCOPE_NESTED` (defensive — the `has_project_scope` gate should
    already have caught it) and never toast on this path.
  - Record `self._patch_query_scope_seed_baseline = self.canonical_query_string`, then
    set `self.query_string = rewritten` and
    `self.parsed_query = self._parse_patch_query(rewritten)`.
  - Re-filter in place with `self._refilter_current_patch_snapshot()`. Do **not** call
    `_load_patches()` (a redundant disk read during startup), do not push query history,
    do not toast, do not call `_save_current_query()`.
- `seeded=False` (an explicit pick):
  - Clear `self._patch_query_scope_seed_baseline` (the user now owns the query).
  - Compute the rewrite; on `PROJECT_SCOPE_NESTED` emit the existing warning toast
    ("Project scope is inside a grouped expression; edit the query with `<f>`") and
    return without changing scope.
  - Delegate to `self._commit_patch_query(rewritten)` so the pick persists, pushes
    history, and reloads exactly as it does today. Pass `notify=False` from the
    cross-pane path (step 5) so picking a project on Beads does not raise a "Query
    updated" toast about a pane the user is not looking at; thread that through
    `_commit_patch_query` as a keyword-only `notify: bool = True` parameter.

Wrap the parse in a `try`/`except QueryParseError` and log-and-return on failure; a
project display name containing query metacharacters must never be able to break the
Patches pane at startup.

### 4. Seed call site — `src/sase/ace/tui/actions/artifacts.py:334-353`

In `_ensure_artifacts_project_choices`'s `_runner()`, in the branch that computes the
seed, after `self._set_artifacts_project_scope(seed, picked=False)` call
`self._apply_patches_project_scope(project_ref, seeded=True)` where
`project_ref = result.project_ref_display.label_for_ref(self.artifacts_project_scope)`
(falling back to the scope value when the snapshot does not know it, matching
`label_for_ref`'s existing contract).

Call it on the seed branch only. The `else` branch runs on every inventory reload and
must stay query-neutral, otherwise a refresh would rewrite the query the user has since
edited.

### 5. Cross-pane picks — `src/sase/ace/tui/actions/artifacts.py:380-399`

Collapse `_on_picked` onto the new helper so both panes take the same path:

- Resolve `project_ref = choices.project_ref_display.label_for_ref(result.project_key)`
  once (it is `None` for the **All projects** choice, which is exactly the removal
  signal `rewrite_project_scope` already understands).
- Call `self._set_artifacts_project_scope(result.project_key, picked=True)`.
- Call
  `self._apply_patches_project_scope(project_ref, seeded=False, notify=self.current_artifacts_pane_key == "patches")`.

Keep the existing nested-expression guard behavior for the Patches pane: when the
rewrite is refused, warn and leave both the query and the shared scope untouched, as it
does today.

Also update the picker's "what is currently selected" read at
`actions/artifacts.py:400-405`: it should keep preferring the query-derived value on the
Patches pane, which now agrees with the shared scope instead of contradicting it.

### 6. Persistence guard — `actions/patch/_query.py` and `_startup_loads.py`

- Change `_save_current_query` (`actions/patch/_query.py:30-34`) to clear
  `self._patch_query_scope_seed_baseline` after saving. Every one of its user-driven
  callers (`actions/patch/_filter_session.py:108`, `actions/patch/_query.py:118`,
  `:258`, `:294`, `actions/agents/_notification_navigation.py:279`,
  `actions/navigation/_tree.py:484`) follows a `self.query_string = ...` assignment, so
  this single change covers all six.
- Add `_save_startup_query()` beside it, which persists
  `self._patch_query_scope_seed_baseline or self.canonical_query_string` and leaves the
  baseline alone. Point `src/sase/ace/tui/actions/_startup_loads.py:164` at it.

The race is then deterministic in both orders: save-before-seed persists the unseeded
query, save-after-seed persists the recorded pre-seed baseline. Either way
`last_query.txt` never learns about the seed.

### 7. Lazy inventory from the Patches pane — `actions/artifacts_navigation.py:72-80`

In `_sync_active_artifacts_entry_state`, call `self._ensure_artifacts_project_choices()`
on the `patches` branch too, before the existing `self._refresh_display()` and early
return. The read is coalesced and off-thread, so this costs nothing when the inventory
is already loaded, and it closes the "jumped straight to Patches" hole in gap 3.

## Tests

Extend `tests/ace/tui/test_artifacts_current_project_scope.py` (it already owns the
`_ArtifactsProjectChoices` fixtures, the `run_vcs_log` stub, and the `AcePage` seed-wait
idiom) and `tests/ace/tui/test_patch_project_scope.py` (pure query-rewrite unit tests).

Unit — `tests/ace/tui/test_patch_project_scope.py`:

1. `has_project_scope` is `True` for `project:alpha`, `+alpha`,
   `(project:alpha OR "x")`, and `NOT project:alpha`; `False` for `""`, `'"needle"'`,
   and a query whose only `project`-looking text is inside a quoted string.

Integration — `tests/ace/tui/test_artifacts_current_project_scope.py`:

2. First open with two enabled projects and `current_project="alpha"` seeds the Patches
   query: `query_string` gains `project:Alpha` (the **label**, not the key `alpha`), the
   Patch list re-filters, and `artifacts_project_scope == "alpha"`. Use a non-`!!!`
   starting query (e.g. `'"needle"'`) here so the assertion does not depend on the
   deprecated shorthand.
3. Extend `test_seeded_scope_reaches_stitches_beads_files_and_plans` (or add a sibling)
   so the Patches query is asserted alongside the other panes.
4. An explicit `project:beta` in the startup query is left byte-identical — the seed
   does not rewrite it — and `artifacts_project_scope` stays `beta`.
5. `NOT project:beta` in the startup query is left byte-identical and is **not** turned
   into `NOT project:Alpha`. This is the regression test for the
   `get_sole_project_filter` blind spot.
6. `(project:beta OR "x")` is left byte-identical and produces no startup toast.
7. `ace.current_project.seed_filters: false` with two enabled projects leaves the
   Patches query untouched (mirrors the existing
   `test_seed_filters_false_keeps_today_s_unscoped_multi_project_behavior`).
8. **Persistence guard:** point `sase.ace.saved_queries._LAST_QUERY_FILE` at a tmp path,
   let the seed apply, run the startup save path, and assert the persisted text is the
   pre-seed query. Then simulate a second cold start from that persisted value and
   assert the seed still fires. This is the test that pins the stickiness hazard.
9. **User commit after a seed** persists the committed query and clears the baseline, so
   a later `_save_current_query` does not resurrect the pre-seed text.
10. A mid-session inventory reload does not re-rewrite an edited query (extend the
    existing `test_mid_session_current_project_change_does_not_re_scope` shape).
11. **Cross-pane pick:** with the Beads pane active, pick `beta` in the
    `InventoryProjectPicker`; assert the Patches query gains `project:Beta`, the shared
    scope is `beta`, and no "Query updated" toast fires while off-pane.
12. Picking **All projects** from either pane removes the term from the Patches query
    (the existing `test_picked_all_projects_is_not_reseeded_after_inventory_reload`
    covers the scope half; extend it to assert the query half).
13. A live filter session in flight suppresses the seed and still consumes the one-shot
    attempt.
14. `_sync_active_artifacts_entry_state` on the Patches pane triggers
    `_ensure_artifacts_project_choices` (assert the collector ran).

## Docs

- `docs/ace.md:3435` — add Patches to the seeded-surface list, currently "the shared
  Artifacts project scope (Stitches, Beads, Plans, Files)".
- `docs/ace.md` Patch-pane query section — document that the seed appends a visible
  `project:<name>` term, that an explicit `project:` / `+name` term of any polarity
  wins, that the seeded term is not written to `last_query.txt`, and that `p` → **All
  projects** removes it.
- `docs/configuration.md:846-848` — the `seed_filters` paragraph should name the Patches
  query as a seeded surface and state that the seed is session-only.
- Leave `patches_artifact_bindings.py:76` as-is; its "Filter (seeds current)" text
  becomes true once this lands.
- `CHANGELOG.md` is generated by release-please (`tools/validate_changelog`); do not
  edit it. Describe the change in the conventional-commit subject/body instead.

## Verification

```bash
just install
just check
```

`just check`'s scoped lane should select the touched ACE TUI and query suites; confirm
with `tools/select_tests --explain` if the selection looks thin. Because this change
touches shared query/startup plumbing, finish with a full run through a monitor:

```bash
sase monitor start --command 'just check-full' --next '<follow-up action>'
```

Also run the visual suite if any pane rendering shifts:

```bash
just test-visual
```

Symvision note: `has_project_scope` is a new public symbol and must land with its
non-test caller (step 3) in the same change, or `just check` fails the symvision gate.
