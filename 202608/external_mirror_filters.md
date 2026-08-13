---
tier: tale
title: Configurable bug and pull-request filters
goal:
  One glob-criterion filter surface governs which external issues become beads and which
  external pull requests become Patches, release-please and release-plz PRs are excluded
  by default, the two legacy knobs keep working as deprecated aliases, and the records a
  filter drops are visible in the CLI and the Patches pane instead of silently missing.
size: medium
proposed_by: bbugyi200.athena.sase-k2.2
bead: sase-k2.2
create_time: 2026-08-12 11:50:43
status: wip
---

- **PARENT:**
  [202608/external_mirror_refinement.md](https://github.com/sase-org/sase--plans/blob/main/202608/external_mirror_refinement.md)
- **BEAD:**
  [sase-k2.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-k2/sase-k2.2.md)

# Plan: Configurable bug and pull-request filters (epic phase `filters`, bead sase-k2.2)

Implements the `filters` phase of `@plan:202608/external_mirror_refinement.md`. Nothing
here depends on the sibling `spec_repair` or `lane` phases, and nothing here may assume
their changes have landed.

## Goal

Replace the external mirror's two ad-hoc knobs (`external_mirror.exclude_labels`,
`external_mirror.pr_authors`) with one filter surface shared by both mirror kinds:
`external_mirror.issues.filters` and `external_mirror.pull_requests.filters`, each a
`fileHookFilters`-shaped object of criteria whose `*_globs` lists accept `!`-prefixed
exclusions. Ship default PR head-ref exclusions so release-please and release-plz PRs
stop becoming Patches, keep the two legacy keys working as deprecated aliases, and make
the filtered-out records visible instead of silently missing.

## Background measured against the current tree

- `src/sase/external_mirror/config.py` has exactly two accessors, both thin readers over
  `load_merged_config()["external_mirror"]`.
- `issues.py` applies them via `_is_excluded` / `_excluded_count`; `pr_sync.py` applies
  them via `_authored_remotes`. Both already count dropped records into a report's
  `unmirrored` field.
- `src/sase/config/file_hooks.py:428` is the established glob evaluator:
  `wcmatch.glob.globmatch` with `DOTGLOB | GLOBSTAR | NEGATE | NEGATEALL`, guarded by an
  `if filters.<criterion>:` emptiness check because `globmatch(value, [])` is `False`.
  `wcmatch` is already a runtime dependency.
- `IssueWire` carries `labels`, `author`, `title`, `state`; `PullRequestWire` carries
  `author`, `head_ref`, `base_ref`, `title`, `state`. Every criterion this plan adds is
  already on the wire.
- `external_mirror` in `src/sase/config/sase.schema.json:2254` is
  `additionalProperties: false`, and
  `tests/test_config_schema.py::test_default_config_matches_public_schema` validates
  `default_config.yml` against it, so schema and defaults must land together.
- The user config layer merges lists with `list_strategy: "replace"`
  (`src/sase/config/layers.py:200`), so a user who sets `head_ref_globs` replaces the
  shipped defaults wholesale rather than appending to them.

## Design decisions

### 1. Criterion vocabulary

Follow `fileHookFilters` exactly, including its two shapes:

- `*_globs` — a list of globs; a `!`-prefixed entry is an exclusion.
- `states` — a plain enum list (`open` / `closed`), the analogue of
  `fileHookFilters.ops`.

```yaml
external_mirror:
  issues:
    filters:
      author_globs: []
      label_globs: []
      title_globs: []
      states: []
  pull_requests:
    filters:
      author_globs: []
      base_ref_globs: []
      head_ref_globs:
        - "!release-please--*"
        - "!release-please/*"
        - "!release-plz-*"
        - "!release-plz/*"
      title_globs: []
      states: []
```

Semantics, shared by both kinds: within a criterion, a record matches if it matches any
positive glob (or the criterion has no positive globs) and matches no `!`-prefixed glob.
A record is mirrored only if every criterion accepts it. An empty criterion accepts
everything. Matching is case-folded.

### 2. Evaluate positives and negatives explicitly, not via `NEGATE`

`label_globs` runs against a multi-valued field. `wcmatch`'s `NEGATE` semantics are
defined for one value at a time, and OR-ing them across labels is wrong: an issue
labelled `("bug", "question")` under `label_globs: ["!question"]` would be accepted
because `bug` clears the negative. So the evaluator splits the criterion once into
positives and negatives and states one rule that is correct for both single- and
multi-valued fields:

- accepted by positives when there are no positives, or some value matches some
  positive;
- rejected when some value matches some negative.

Each individual pattern is still matched with `wcmatch.glob.globmatch` under
`DOTGLOB | GLOBSTAR | NEGATE | NEGATEALL | IGNORECASE`, so the pattern language stays
identical to file hooks. On single-valued fields this rule is equivalent to passing the
whole list to `globmatch` with `NEGATE | NEGATEALL`; verified against `wcmatch` in this
workspace. Do not keep two code paths.

Deliberate consequences to encode in tests: an issue with no labels is rejected by a
positive-only `label_globs` and accepted by a negative-only one; an empty `head_ref`
(providers that do not populate it) clears a negative-only criterion but not a positive
one, matching how `hook_matches_event` treats an unattributed agent name.

### 3. Default filters

Issues ship every criterion empty, preserving the epic's "beads are a superset of
issues" promise. Pull requests ship only the four head-ref exclusions. Head-ref is the
right key here because release-please on this repo authors as a human account, and
titles are user-configurable. The observable behavior change is exactly "release-please
and release-plz PRs stop becoming Patches".

### 4. Legacy aliases

`exclude_labels` and `pr_authors` keep working:

- `exclude_labels: [x, y]` folds into `issues.filters.label_globs: ["!x", "!y"]`.
- `pr_authors: [a, b]` folds into `pull_requests.filters.author_globs: ["a", "b"]`.

The fold applies **only when the modern criterion is empty**. A criterion that is empty
after merge is indistinguishable from unset, and both legacy keys default to `[]`, so
"modern wins" has to mean "modern wins when it says something". Document that rule where
the accessors live.

Deviation from the epic plan, recorded deliberately: the epic says to keep
`excluded_issue_labels()` and `mirrored_pr_authors()` as public shims. Once `issues.py`
and `pr_sync.py` call the new accessors, those two public functions have no non-test
consumer, and `sase/memory/symvision.md` is explicit that a test-only reference does not
keep a public symbol alive — `just lint` would fail. Demote them to private helpers
(`_legacy_excluded_labels()`, `_legacy_pr_authors()`) consumed by the fold, which keeps
their exact reading behavior and their meaning while satisfying the linter. The three
tests that monkeypatch them move to the new accessors.

### 5. A filter change must actually re-examine dropped records

The epic phase says dropped records stay out of the checkpoint "so clearing a filter
later re-examines them". That is only half true in the current tree and must be finished
here, because a filter surface whose edits do nothing until upstream changes is not user
control:

- `issues.py` early-returns when the upstream watermark and the full provider-id set are
  unchanged. Both are computed over _all_ issues, including excluded ones, so clearing
  `exclude_labels` today changes nothing until some issue is touched upstream.
- `pr_sync.py` drops filtered records before `latest_seen` is computed, so the cursor
  never advances past them — but a previously excluded PR older than the cursor stays
  outside `OVERLAP_WINDOW` until the 24-hour `DAILY_REPAIR_SCAN`.

Fix both with one mechanism: each filter model exposes a stable `fingerprint()` (a
digest over its canonicalized criteria). Persist it beside the existing state and, when
the stored fingerprint differs from the current one, treat the pass as a full pass —
`issues.py` skips its unchanged-upstream early return, `pr_sync.py` sets `full=True`.

- Issues: a new `filters_fingerprint` field on `_MirrorState`, written in the same
  `deferred == 0` block that already advances the watermark.
- PRs: a new `filters_fingerprint` field on `MirrorCursor`, written by
  `_advanced_cursor` on a clean pass.

Do **not** bump `ISSUE_SCHEMA_VERSION`. Both readers are already tolerant, so an
upgraded machine reads `""`, disagrees with the computed fingerprint once, and performs
exactly one re-examination pass. That is the migration behavior we want, and a version
bump would instead discard every project's watermark.

Filters gate creation only. A record that a new filter now excludes keeps whatever bead
or Patch it already has; the mirror never deletes. Say so in the docs.

### 6. Make the dropped records visible

A non-empty PR filter is now the default, so a silently shorter Patch list must not be
indistinguishable from a broken mirror. Three surfaces, cheapest first:

**CLI.** `sase patch sync-external`'s Rich table gains a `Filtered` column between
`Fetched` and `Adopted`, fed by `report.unmirrored`. `sase bead sync-external`'s
per-project line gains `filtered=<n>` when non-zero.

**Durable count.** `pr_sync` records the last pass's filter outcome in a single
lane-independent document at `_mirror_state_dir("external_pr") / "unmirrored.json"`,
shaped `{"<project_key>": {"fetched": N, "unmirrored": M, "observed_at": "<iso>"}}`.
Deliberate choices:

- Lane-independent from birth (`sase_subdir("external_mirror")`), so the sibling `lane`
  phase's relocation of the cursor and backoff files does not touch it.
- One shared document, read-modify-write, matching the existing `<kind>__backoff.json`
  precedent. A lost update between two concurrent per-project chop instances costs one
  stale cosmetic counter until the next pass; note that in the module docstring rather
  than adding a lock.
- Written on every non-dry-run pass that got a provider listing, including passes that
  end dirty, because the count reflects the fetch and not the mutations.
- Written through `best_effort_test_state_write_allowed(...)` like `write_mirror_state`
  does. This path resolves under the real `~/.sase`, unlike the cursor writes that tests
  already redirect via `state_dir`, so an unguarded write would have tests mutating the
  user's state directory.

**TUI.** The Patches pane's level-0 banner chip becomes `N PRs  ·  M remote-only` when a
count is known. Threading, respecting `sase/memory/tui_perf.md` rule 8 ("render paths
never stat/glob") and rule 1 (no synchronous I/O on the event loop):

- `state.py` exposes a pure `read_pr_unmirrored_counts() -> dict[str, int]`.
- `src/sase/ace/tui/actions/patch/_loading.py` wraps it in a module-level cache keyed by
  the document's `st_mtime_ns` and size, calls it from `_prepare_patch_load_from_disk`
  (already off-thread via `asyncio.to_thread`) and from the synchronous `_load_patches`
  path, and stores the display-name-keyed mapping on the app. Map project key to label
  with `project_display_name_for`, because the banner's level-0 key is the display name
  (`models/patch_groups/_keys.py:59`).
- `_display.py` passes that mapping to `PatchList.update_list` only when
  `grouping_mode is PatchGroupingMode.BY_PROJECT`; `render_grouped` forwards it to
  `format_patch_banner_option`, which applies it only at `group.level == 0`.
- Every new parameter is optional and defaults to `None`, so existing callers and the
  `_changespec_list_banner` compatibility alias keep working untouched.

### 7. Deprecation diagnostic

Add `src/sase/doctor/checks_config_external_mirror.py` with a `config.external_mirror`
check, registered in `config_check_specs` (`src/sase/doctor/checks_config.py`). This
follows `checks_config_repos.py`'s nested-key deprecation precedent;
`DEPRECATED_TOP_LEVEL_KEYS` is not the right home because both keys are nested.

- WARN naming the replacement when a legacy key is non-empty.
- WARN more loudly when a legacy key and its modern criterion are both non-empty,
  stating that the legacy value is ignored.
- OK otherwise.

Also mark both legacy keys `"deprecated": true` in `sase.schema.json`, matching the
`amd_agents_template` precedent at line 637.

### 8. Placement relative to the Rust core boundary

The matcher stays in Python under `src/sase/external_mirror/`. `CLAUDE.md`'s boundary
rule asks whether another frontend would need the behavior to match the TUI; this is
config-driven selection applied at the VCS provider seam, over `IssueWire` /
`PullRequestWire`, which `src/sase/vcs_provider/_types.py:17-20` documents as
deliberately host-side ("tracker commands and their JSON normalization are host/plugin
concerns"). It also depends on `wcmatch`, the same Python glob engine file hooks use. No
`sase-core` change, no wire schema bump.

## Implementation steps

1. **`src/sase/external_mirror/filters.py`** (new). Pure, no SASE imports beyond typing:
   - `IssueFilters` and `PullRequestFilters` frozen dataclasses holding tuple criteria.
   - `matches(issue)` / `matches(remote)` methods.
   - `fingerprint()` on each.
   - A shared private glob-criterion evaluator implementing decision 2, and a private
     state-criterion evaluator (case-folded exact membership, empty accepts everything).
2. **`src/sase/external_mirror/config.py`**. Add `issue_filters()` and
   `pull_request_filters()` reading the new config paths with the legacy fold from
   decision 4; demote the two old accessors to private helpers; keep every read wrapped
   in the existing fail-soft `try/except`.
3. **`src/sase/config/sase.schema.json`**. Add `issues` / `pull_requests` objects, each
   with a `filters` object; `additionalProperties: false` at every level; `*_globs`
   arrays of nonempty strings; `states` an enum array of `open` / `closed`. Mark
   `exclude_labels` and `pr_authors` deprecated with descriptions naming the
   replacement.
4. **`src/sase/default_config.yml`**. Write the full block from decision 1 with comments
   at the surrounding density, including a note that setting a criterion replaces the
   shipped defaults rather than appending to them.
5. **`src/sase/external_mirror/issues.py`**. Replace `_is_excluded` / `_excluded_count`
   with one `IssueFilters` resolved once per pass and threaded into both the
   early-return count and the mirrorable partition. Add the fingerprint gate from
   decision 5. Update the `unmirrored` field comment.
6. **`src/sase/external_mirror/pr_sync.py`**. Replace `_authored_remotes` with the
   `PullRequestFilters` partition, add the fingerprint gate, and record the unmirrored
   count. Update the module docstring.
7. **`src/sase/external_mirror/state.py`**. Add `filters_fingerprint` to `MirrorCursor`
   (tolerant read, written by `write_mirror_cursor`) and to `_MirrorState`; add
   `read_pr_unmirrored_counts()` / `write_pr_unmirrored_count()` per decision 6.
8. **`src/sase/external_mirror/report.py`**. Retarget the `unmirrored` comment from
   `pr_authors` to the filter surface.
9. **CLI surfaces.** `src/sase/main/patch_sync.py` (`Filtered` column, including the two
   early `add_row` error paths so the column count stays consistent) and
   `src/sase/bead/cli_sync_external.py` (`filtered=<n>`).
10. **TUI surfaces.** `_loading.py`, `_display.py`, `_patch_list_render.py`,
    `_patch_list_banner.py` per decision 6.
11. **Doctor.** New check module plus registration per decision 7.
12. **Docs.** `docs/configuration.md` (the `external_mirror` YAML block and its table at
    2905-2918), `docs/axe.md` (316 and 852), `docs/beads.md` (580),
    `docs/change_spec.md` (8). Each currently names a legacy knob by name; each must
    describe the filter surface, the shipped PR default and why head-ref was chosen, and
    the fact that filters gate creation only.

## Tests

- `tests/test_external_mirror_filters.py` (new): the evaluator on its own — empty
  criterion accepts everything; positive-only; negative-only; mixed; case folding;
  multi-valued `label_globs` including the "one label clears the negative" trap from
  decision 2; no-labels and empty-string-value edges; `states`; the four shipped globs
  against `release-please--branches--master`, `release-plz-1.2.3`, and a human branch;
  `fingerprint()` stability across equal models and difference across unequal ones.
- `tests/test_external_mirror_config.py` (new): legacy fold in both directions, modern
  criterion winning when non-empty, legacy applying when the modern criterion is empty,
  malformed config degrading to empty filters.
- `tests/test_external_mirror_issues.py`: rewrite
  `test_exclude_labels_skips_and_counts_unmirrored` against `issue_filters`; add a
  fingerprint-change test asserting the unchanged-upstream early return is skipped and a
  newly includable issue gets a bead.
- `tests/test_external_pr_sync.py`: rewrite the two `pr_authors` tests against
  `pull_request_filters`; add a default-filter test proving a
  `release-please--branches--master` PR is counted `unmirrored` and never adopted while
  a human PR is; add a fingerprint-change test proving the pass goes full.
- `tests/test_config_schema.py` already validates `default_config.yml` against the
  schema; confirm it passes rather than adding a duplicate.
- `tests/doctor/test_checks_config_external_mirror.py` (new): OK, single-deprecation
  WARN, and both-set WARN.
- Patch-banner rendering test covering the `M remote-only` chip and its absence when no
  count is known; a `_loading.py` cache test proving a second call with an unchanged
  document does not re-read.
- `tests/main/test_patch_sync_external_cli.py` and
  `tests/test_bead_sync_external_cli.py`: assert the new column / field.

## Verification

`just install`, then `just check`. Run `just check-full` as well: this change touches
`src/sase/config/sase.schema.json` and `src/sase/default_config.yml`, which are in the
broadening set.

Then confirm the phase's own slice of the epic's verification list on this machine:

```bash
sase patch sync-external --project sase --dry-run
```

must report a non-zero `Filtered` count for `sase` and must plan zero adoptions for any
`release-please--*` PR. Do not attempt the epic-level checks that belong to
`spec_repair`, `lane`, `bug_status`, `patch_status`, or `perf`.

## Out of scope

Explicitly left to their own phases, even if noticed while working here: the ProjectSpec
two-blank-line parser bug and archive de-duplication (`spec_repair`), the dedicated
`external_mirror` lumberjack and the relocation of the PR cursor and backoff files
(`lane`), mirrored-bead status transitions (`bug_status`), refreshing adopted external
Patches (`patch_status`), and the per-pass index re-read (`perf`). Record anything else
discovered as `PROPOSED FOLLOW-UP:` notes on bead sase-k2.2 rather than filing beads.
