---
tier: epic
title: Merge-commit support for the VCS commit log
goal: 'Merge commits are first-class in every SASE commit-log surface: hidden by default
  so the timeline shows the commits a PR contained, revealable and unmistakably marked
  on demand, and browsable as a "what landed" view — with presence, counts, and diffs
  that stay truthful in every mode.

  '
phases:
- id: core
  title: Rust core — parent ids, tolerant parser, merge summary
  depends_on: []
  size: medium
  description: 'core: in the sase-core repo, add parent ids to the VCS-log wire, make
    the pinned git-log parser accept both the legacy 7-field and the new 8-field record
    layout, bump the wire schema to 3, add a strict merge-subject summary parser,
    and expose the new PyO3 bindings.

    '
- id: wire
  title: Python wire mirror and skew probes
  depends_on:
  - core
  size: small
  description: 'wire: mirror the Rust contract on the Python side — parent ids and
    schema 3 in the wire module, %P in the pinned git-log format, the tolerant golden
    parser, a merge-summary facade — and add the validator probes that make wheel/host
    skew fail loudly instead of silently returning zero commits.

    '
- id: provider
  title: Provider-level merge visibility
  depends_on:
  - wire
  size: medium
  description: 'provider: introduce the three-valued merge-visibility mode, thread
    it through the log and partition hooks so presence and ahead/behind stay consistent
    with what is displayed, prove the partition law, and make merge-commit diffs and
    author-time lookups work.

    '
- id: collect
  title: Collection models and the merges query key
  depends_on:
  - provider
  size: medium
  description: 'collect: carry merge visibility through the collection filters and
    the commit filter-query language, including the in-memory matcher, canonical tokens,
    completions, and snapshot-coverage rules that keep live preview honest.

    '
- id: render
  title: Marking merges in every renderer
  depends_on:
  - collect
  size: medium
  description: 'render: give merges a dedicated accent, a reserved marker column that
    costs nothing when no merge is visible, a legend key, parent lines in full output,
    structured JSON fields, and a condensed pull-request headline in the pretty timeline.

    '
- id: cli
  title: sase vcs log --merges and documentation
  depends_on:
  - collect
  size: small
  description: 'cli: add the -m/--merges option with its three modes, wire it through
    the vcs handler, and document the behavior and its relationship to git''s own
    merge flags.

    '
- id: tui
  title: ACE Commits pane merge affordances
  depends_on:
  - render
  size: medium
  description: 'tui: add the merge-visibility cycle key to the Commits sub-tab with
    its hint, help, availability, keymap, and config entries, and make the detail
    pane and commit modal show the merge badge, parents, and a meaningful merge diff.

    '
- id: floor
  title: Raise the sase-core-rs dependency window
  depends_on:
  - cli
  - tui
  size: small
  description: 'floor: after the core release publishes, move the sase-core-rs version
    window in pyproject.toml to the release that carries the schema-3 contract and
    confirm the exhaustive gate passes against the published wheel.

    '
- id: verify
  title: End-to-end acceptance against real merge history
  depends_on:
  - floor
  size: small
  description: 'verify: exercise every mode of every surface against a real repository
    that contains merge commits, confirm the partition law and the skew probe in a
    live environment, and record the acceptance evidence.'
proposed_by: bbugyi200.athena.wl
create_time: 2026-08-09 09:42:58
status: wip
bead_id: sase-i8
---

- **PROMPT:** [prompts/202608/merge_commit_support.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/merge_commit_support.md)
- **BEAD:** [sase-i8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-i8/README.md)

# Plan: Merge-commit support for the VCS commit log

## Why

The project owner is turning off squash-merge in their GitHub organizations. Once that
lands, every merged PR leaves behind a merge commit alongside the individual commits it
contained. SASE's commit-log stack was built for a squash world: the git provider hard
codes `--no-merges` in `src/sase/vcs_provider/plugins/_git_query_ops.py`, with no way to
turn it off and no data on the wire that would let any surface even know a commit was a
merge.

Today's default is, happily, already the behavior the owner wants: `git log --no-merges`
still traverses _through_ merges, so every commit a PR contained is shown while the
merge commit itself is elided. This was verified directly against a scratch repository
with a `--no-ff` merge: the four ordinary commits appear and the merge does not. So this
epic does **not** change what the default view shows. It makes that default a
deliberate, named, documented choice instead of a hard-coded one; it makes merges
revealable and unmistakable when revealed; and it adds a "what landed" view.

## The shape of the feature

One concept, named **merge visibility**, with exactly three modes:

| Mode   | Meaning                                               | git equivalent |
| ------ | ----------------------------------------------------- | -------------- |
| `hide` | Merge commits are omitted. **Default.** Today's view. | `--no-merges`  |
| `show` | Everything, with merges clearly marked.               | _(no flag)_    |
| `only` | Merge commits alone — the "what landed" view.         | `--merges`     |

The same word and the same three values appear everywhere: `--merges hide|show|only` on
the CLI, `merges:hide|show|only` in the filter-query language, and one cycle key in the
TUI. No surface invents its own vocabulary.

### The partition law

**For a fixed repo set, revision set, and filter set, `hide` and `only` are disjoint and
their union is exactly `show`.**

This is the design's spine. It is why `only` must _not_ use `--first-parent`: adding it
would make `only` a different traversal, counts would stop adding up, and "merges I can
see in `show`" would silently differ from "merges I get in `only`". Verified
empirically: on a scratch repo, `--no-merges` yields 4 commits, `--merges` yields 1, and
the unfiltered log yields exactly those 5. Every phase that touches selection must
preserve this, and `provider` owns the test that proves it.

The law is what makes the feature _reliable_: a user can trust that flipping modes
re-slices one history rather than running three unrelated queries.

## Deliberately out of scope

- **`sase vcs list` repo statistics.** `vcs_repo_stats` counts commits with
  `git rev-list --count HEAD` and contributors with `git shortlog -sne`; both will start
  including merge commits once squashing stops. That is repo-level aggregate data, not a
  timeline, and changing it would silently move numbers users already read. It deserves
  its own task bead; note it as a proposed follow-up rather than fixing it here.
- **`src/sase/ace/revert_agent_discovery.py`.** Its `--no-merges` is intentional: it
  hunts for SASE-authored commits to revert. Leave it alone.
- **The Agents-tab COMMITS panel.** It lists commits a specific SASE agent authored.
  Agents never author merge commits, so there is nothing to mark.
- **Linking a merge row to its SASE Patch through the PR number.** Genuinely attractive
  (a Patch's `PR:` field is exactly the number a GitHub merge subject carries), but it
  is a separate feature with its own resolution and caching questions. The `merge`
  summary data this epic puts on the wire is what a future phase would build on.

---

## Phase `core` — Rust core: parent ids, tolerant parser, merge summary

Work happens in the **sase-core** repo. Open it with the `/sase_repo` skill
(`sase repo open sase-core -r "<reason>"`) and use only the printed path. Land this
phase as its own sase-core pull request; every later phase depends on it being merged,
because subsequent phase agents get a fresh sase-core checkout at `origin/master`.

Follow `AGENTS.md` in that repo: release-plz owns versions, so do **not** hand-edit
`[workspace.package].version` or crate versions. Use Conventional Commit subjects
(`feat(vcs-log): ...`) so release-plz computes the bump.

### Parent ids on the wire

`crates/sase_core/src/vcs_log/wire.rs`:

- Add `pub parent_ids: Vec<String>` to `VcsCommitWire`, serialized with
  `#[serde(default)]` so older payloads rehydrate.
- Bump `VCS_LOG_WIRE_SCHEMA_VERSION` from `2` to `3` and update the module doc's JSON
  shape list.
- Do **not** add an `is_merge` field. It is derivable (`parent_ids.len() > 1`) and a
  stored duplicate is a chance to disagree with itself. Expose it as a method/helper
  instead.

### A parser that tolerates both layouts

`crates/sase_core/src/vcs_log/parsers.rs` currently splits each record into exactly
`FIELD_COUNT = 7` fields and drops anything else. The host is about to start emitting
`%P`, which makes it 8.

Add `%P` **before** `%s`, giving the record layout `%H %h %an %ae %at %P %s %b`. It must
go before the message fields because the parser uses `splitn`, so the final field
absorbs any extra separators — the body has to stay last.

Make the parser accept **either** count:

- 8 fields → new layout: `parent_ids` from splitting field 5 on ASCII space (an empty
  field means zero parents), subject at 6, body at 7.
- 7 fields → legacy layout: `parent_ids = vec![]`, subject at 5, body at 6.
- Any other count → dropped, exactly as today.

This tolerance is deliberate: it means a _newer_ wheel paired with an _older_ host keeps
working. The reverse direction (older wheel, newer host) is the dangerous one and is
handled by the schema probe in phase `wire`.

Root commits produce an empty `%P` field — verified against real git output — so the
zero-parent case is normal, not malformed. Cover it with a test.

### Merge summary parsing

New module (suggested `crates/sase_core/src/vcs_log/merge_summary.rs`), a **pure**
function over a commit subject and body:

```rust
pub enum MergeSummaryKindWire { PullRequest, Branch, RemoteBranch }

pub struct MergeSummaryWire {
    pub kind: MergeSummaryKindWire,
    pub reference: Option<String>,  // "123" for a PR, the branch name otherwise
    pub source: Option<String>,     // "org/feature-branch"
    pub target: Option<String>,     // "master", from "... into master"
    pub headline: Option<String>,   // first non-empty body line (the PR title)
}

pub fn parse_merge_summary(subject: &str, body: &str) -> Option<MergeSummaryWire>;
```

Recognize, strictly and case-sensitively:

- `Merge pull request #<digits> from <source>` → `PullRequest`, `reference` = the
  digits, `source` = the rest. This is what GitHub writes; it was verified against a
  real merge commit, whose body's first line is the PR title.
- `Merge branch '<name>'` optionally followed by ` into <target>` → `Branch`.
- `Merge remote-tracking branch '<name>'` optionally followed by ` into <target>` →
  `RemoteBranch`.
- Anything else → `None`.

`None` is the safe, common answer, and callers must render the raw subject when they get
it. **Never** mangle a subject you did not fully recognize — a wrong condensation is
worse than no condensation.

### Bindings

In `crates/sase_core_py/src/lib.rs`:

- `parse_git_log`, `classify_commit_presence`, and `aggregate_commit_log` already
  convert through dict payloads; extend those conversions to carry `parent_ids` in both
  directions.
- Add `vcs_log_wire_schema_version() -> u32`. This does not exist today, which is
  exactly why the host currently has no way to detect a stale wheel for this contract.
- Add `parse_merge_summary(subject: str, body: str) -> dict | None`.
- Register all new functions in the module init and update the module docstring's
  binding inventory near the top of the file.

### Tests

- Parser unit tests: 8-field record, 7-field legacy record, root commit with empty
  parents, octopus merge with three parents, and the existing malformed-record and
  multi-line-body cases still passing.
- `crates/sase_core/tests/vcs_log_parity.rs`: update the pinned assertion
  `assert_eq!(VCS_LOG_WIRE_SCHEMA_VERSION, 2)` to `3`, and extend the parity fixtures
  with parent ids.
- Merge-summary tests: each recognized form, the `into <target>` variants, an
  unrecognized subject returning `None`, a PR merge with an empty body (`headline` is
  `None`), and a subject that merely _starts_ with "Merge" but matches nothing.

---

## Phase `wire` — Python wire mirror and skew probes

Depends on `core` being merged and available. Local dev installs build `sase_core_rs`
from the workspace's sase-core checkout, so run `sase repo open sase-core` and then
`just install` before anything else; the checkout must be at or past the core phase's
merge commit.

### Mirror the contract

`src/sase/core/vcs_log_wire.py`:

- `VCS_LOG_WIRE_SCHEMA_VERSION = 3`.
- `VcsCommitWire` gains `parent_ids: tuple[str, ...] = ()`, placed to match the Rust
  field order, and an `is_merge` property returning `len(self.parent_ids) > 1`.
- `vcs_commit_from_dict` reads `parent_ids` defensively (missing key → empty tuple,
  values coerced with `str`), matching how `_presence_from_dict` already tolerates
  absent data.
- Update the module docstring's JSON-shape list, which currently enumerates the flat
  `AggregatedCommitWire` keys.

`src/sase/core/vcs_log_facade.py`:

- `VCS_LOG_GIT_FORMAT` gains `%P` between `%at` and `%s`.
- `_FIELD_COUNT` becomes `8`.
- `_parse_git_log_python`, the golden-contract reference, accepts 7 or 8 fields with the
  same rules as the Rust parser. Parity between these two implementations is a tested
  contract, so they must agree exactly, including on malformed input.
- Add a `merge_summary(subject, body)` facade calling the new binding and returning a
  small frozen dataclass (or `None`).

### Make skew loud

This is the highest-value defensive work in the epic. A host that emits 8 fields against
a wheel whose parser demands exactly 7 drops **every** record, and `sase vcs log`
reports "No commits found" with no error. That failure is silent, total, and looks like
a data problem rather than an install problem.

In `tools/validate_sase_core_rs`:

- Add `vcs_log_wire_schema_version` and `parse_merge_summary` to `REQUIRED_BINDINGS`.
- Add a schema probe asserting `vcs_log_wire_schema_version() == 3`, following the shape
  of `_validate_artifact_ref_schemas`.

The tool's own docstring already explains why version metadata alone cannot catch
unreleased-checkout skew and why behavioral probes are required — this addition is
exactly the kind it argues for.

### Tests

- `tests/test_core_vcs_log.py`: `test_vcs_log_wire_schema_version_is_two` becomes the
  schema-3 assertion; add Rust/Python parity cases for 8-field, 7-field, zero-parent,
  and octopus records.
- New coverage for the `merge_summary` facade delegating correctly and returning `None`
  for unrecognized subjects.

---

## Phase `provider` — Provider-level merge visibility

### The mode type

Define once, next to the other provider-neutral log types:

```python
MergeVisibility = Literal["hide", "show", "only"]
```

Export it so the collection layer, the query language, and the CLI all name the same
type. Default is `"hide"` at every layer, so every existing call site keeps today's
behavior without being touched.

### Thread it through _both_ queries

`src/sase/vcs_provider/plugins/_git_query_ops.py` uses `--no-merges` in two places, and
this is the subtle part:

1. `vcs_log` (line ~250) — the timeline query.
2. `vcs_partition_commits` (lines ~169 and ~178) — the `git rev-list` ahead/behind sets.

`collect.py` feeds those sets to `classify_commit_presence`, which marks any commit
found in neither set as `synced`. So if merges became visible in the log while
`rev-list` kept excluding them, **every merge commit would be mislabeled `synced`** even
when it is unpushed or incoming — and the legend's `↑n ↓n` counts would describe a
different slice of history than the rows underneath them.

The fix is to thread one mode through both. Add a keyword-only
`merges: MergeVisibility = "hide"` to:

- `VCSHookSpec.vcs_log` and `VCSHookSpec.vcs_partition_commits` in `_hookspec.py`
- the corresponding `VCSProvider.log` / `partition_commits` docstrings and signatures in
  `_base.py` (the `log` docstring currently states flatly that merge commits are
  excluded — replace that with the mode contract)
- `VCSPluginManager.log` / `partition_commits` in `_plugin_manager.py`
- `BareGitPlugin.vcs_log` / `vcs_partition_commits`, mapping `hide` → `--no-merges`,
  `show` → no flag, `only` → `--merges`, in both the `git log` and `git rev-list`
  argument builders.

`BareGitPlugin` is the only in-tree implementation; the GitHub plugin in the
`sase-github` repo implements issue and workspace hooks, not `vcs_log`, so no cross-repo
plugin change is needed. Because pluggy passes only the arguments a hookimpl declares, a
future third-party provider that ignores the new keyword still loads — it just silently
keeps hiding merges. State that contract explicitly in the hookspec docstring and add a
case to `tests/test_vcs_provider_contract.py`.

### Merge-commit diffs actually work

`vcs_show_revision` runs `git show --format= --patch <rev>`. For a merge commit git
prints **nothing** — verified: zero lines of output. Every surface that loads a commit
diff (the Commits detail pane, the commit view modal) would show "Diff unavailable for
this commit" on exactly the commits this epic is about.

Add `--first-parent` to that command. Verified against git 2.47: for a merge it yields
the complete set of changes the merge introduced relative to its first parent — which
for a PR merge is precisely "what this PR changed" — and for an ordinary commit the
output is unchanged. Note the git ≥ 2.31 requirement in the code comment.

### Author-time lookups for merge SHAs

`load_commit_created_at` in `src/sase/ace/tui/widgets/prompt_panel/_agent_commits.py`
and `_resolve_commit_created_at` in `src/sase/workflows/commit/commit_tracking.py` both
call `provider.log(cwd, 1, revs=(sha,))`. With merges hidden, asking for a merge SHA
returns its nearest non-merge ancestor; both call sites already defend against this with
an identity check that returns `None`, and both have tests named for that defense. Pass
`merges="show"` at these two call sites so a merge SHA resolves to itself. Keep the
identity guards — they are still correct — and update the two existing tests to assert
the merge SHA now resolves rather than being rejected.

### Tests

- **The partition law**, in `tests/test_vcs_provider_vcs_log.py`, against the real
  temporary git repository the fixtures already build: create a branch, merge it with
  `--no-ff`, then assert that the `hide` and `only` id sets are disjoint and their union
  equals the `show` id set.
- Rewrite `test_vcs_log_excludes_merge_commits` as a mode-parameterized test: `hide`
  excludes the merge (unchanged assertion), `show` includes it, `only` returns just it.
- `parent_ids` populated correctly for ordinary, root, and merge commits.
- `partition_commits` honoring each mode against a repo with a local merge ahead of its
  remote.
- `vcs_show_revision` returning a non-empty patch for a merge commit and identical
  output to today for an ordinary one.

---

## Phase `collect` — Collection models and the `merges` query key

### Collection filters

In `src/sase/vcs_log/models.py`, add `merges: MergeVisibility = "hide"` to **both**
`CommitFilters` and `CommitFilterSpec`, and pass it through `CommitFilterSpec.resolve`.
Merge visibility is a commit-selection filter, which is exactly what these records
model.

In `src/sase/vcs_log/collect.py`, pass `filters.merges` into both `provider.log(...)`
and `provider.partition_commits(...)` in `_collect_repo_commits`, and into the
`_local_log` fallback. `_candidate_limit`'s doubling for an active upper bound is
unaffected.

### The `merges:` query key

`src/sase/vcs_log/filter_query.py`:

- Add `"merges"` to `_FILTER_KEYS`. It is a single-valued, non-repeatable, non-negatable
  key, like `sidecar` and `limit`.
- `CommitLogFilterValues` gains `merges: MergeVisibility = "hide"`; both
  `backend_filters()` and `backend_filter_spec()` forward it.
- Parse it beside `sidecar`, rejecting anything outside the three values with a
  token-anchored error in the same style as `"sidecar: must be 'true' or 'false'"`.
- `to_query_tokens` emits `merges:<mode>` **always**, immediately after the `sidecar:`
  token. Unconditional emission is the point: the persistent filter bar becomes the
  place a user _discovers_ that merges are being hidden and that there is a key to
  change it. `sidecar:` already sets this precedent.
- `compile_commit_matcher` applies the mode in memory using `commit.is_merge` — `hide`
  drops merges, `only` keeps only merges, `show` keeps everything. The wire now carries
  parent ids, so this costs nothing and it is what makes live preview instant when
  narrowing an already-collected `show` snapshot.
- `completion_context` needs no new kind; `merges` is a static-value key.

`src/sase/ace/tui/widgets/artifacts/commit_filter_bar.py`:

- Add `("merges", "merge-commit visibility: hide, show, or only")` to `KEY_COMPLETIONS`,
  placed next to `sidecar`.
- `STATIC_VALUE_COMPLETIONS["merges"] = ("hide", "show", "only")`.
- `VALUE_HINTS["merges"] = "hide, show, or only"`.

### Snapshot coverage

`src/sase/ace/tui/widgets/artifacts/commits_collection.py` decides when a cached result
can honestly stand in for a different query.

- `snapshot_covers`: a snapshot collected with `merges="show"` covers **any** requested
  mode, because `show` is a strict superset — this follows the existing
  `if values.sidecar and not base.sidecar: return False` pattern. Otherwise the modes
  must match exactly.
- `_snapshot_breadth`: rank `show` as broader than `hide` and `only` so a `show`
  snapshot wins the preview-base slot.
- `_scope_key_for` is **not** changed. The scope key identifies the resolved repo set;
  merge visibility does not change which repos are read.

Because it is a backend facet, changing the mode schedules a recollection exactly the
way `sidecar:` does. That is correct and it is what keeps `--limit`/`limit:` honest: the
row cap must apply to _visible_ commits, so the filter has to be pushed down to git
rather than applied after a capped fetch.

### Filter summary

`filter_summary` in `_render_util.py` should mention the mode when it is not the default
(`"merges only"` / `"with merges"`), so the legend and the empty-state message tell the
truth about why rows are missing. Extend `empty_message` so `only` with no results reads
"No merge commits found" rather than the generic text.

### Tests

- `tests/test_vcs_log_filter_query.py`: parse each mode, reject a bad value with the
  right token span, reject negation and repetition, canonical-token round-trip, and
  matcher behavior for each mode over a mixed commit list.
- `tests/test_vcs_log_collect.py`: the mode reaches both provider calls; a fake provider
  asserts it received the same mode for `log` and `partition_commits`.
- Snapshot-coverage tests for the `show`-covers-everything rule.

---

## Phase `render` — Marking merges in every renderer

The requirement is that merges are _clearly_ marked. The requirement that is just as
important but unstated: when merges are hidden — the default, and therefore almost every
render — the output must be **byte-identical to today**. A feature that quietly reflows
the default view has failed.

### The accent and the glyph

`src/sase/vcs_log/_style.py` gains one constant beside `GOLD`, `INCOMING`, and
`UNPUSHED`:

```python
#: Accent for merge-commit markers. Distinct from every REPO_PALETTE entry.
MERGE = "#D787FF"
```

The glyph is **`◆`**. It is already part of this codebase's TUI glyph vocabulary (used
in about two dozen places), so it renders wherever the existing chrome does — including
under the pinned Fira Code font the PNG snapshot suite uses. A diamond node for a merge
and a round node for an ordinary commit is also a convention users meet in graph UIs
elsewhere, which is the cheapest kind of intuitiveness.

### A column that costs nothing when unused

`commit_line` in `_render_console.py` gains a `merge_column: bool` parameter, computed
once per render from the visible commit set
(`any(entry.commit.is_merge for entry in commits)`), alongside the existing `repo_width`
/ `sha_width` measurements:

- `merge_column=False` → emit nothing. Rows are exactly what they are today.
- `merge_column=True` → emit two characters between the repo column and the subject:
  `"◆ "` styled `MERGE` for merges, `"  "` for everything else, so the subject column
  stays aligned.

In `hide` mode nothing is ever a merge, so the column is never allocated and the default
view cannot regress. Apply the same rule in `render_oneline` (plain `◆`/space, no
styling) so piped output carries the mark too.

`build_timeline_commit` in `render.py` computes the flag from `result.commits`, the same
tuple it already measures for column widths, and passes it down — so the TUI timeline
and the CLI stay one implementation.

### Legend

Extend `legend()` in `_render_console.py` to append `◆ merge` to the presence key, in
the `MERGE` style, **only** when a merge is present in the rendered commits. The legend
is where the glyph is taught, which is what makes the mark discoverable rather than
cryptic.

### `full` format

In `_full_commit_lines`, for a merge:

- prefix the repo bar with `◆ ` in the `MERGE` style;
- add a dim line under the footer: `parents  <short1>  <short2>` (all parents, so
  octopus merges are shown honestly). Short ids are derived by truncating the full
  parent ids to the same length as the commit's own `short_id`.

The `full` header keeps the **raw** subject. `full` is the "show me everything" format;
condensing there would be a contradiction.

### Condensed pull-request headline

In `pretty` and the shared timeline row only, when `merge_summary` recognizes a
`PullRequest` **and** a non-empty `headline` exists, render

```
◆ #123  Add the feature title here
```

with `#123` in the `MERGE` accent, instead of
`Merge pull request #123 from org/feature-branch`. Anything else — an unrecognized
subject, a recognized merge with an empty body, a branch merge — renders the raw subject
verbatim.

This is what turns `merges:only` from a list of boilerplate into something that reads
like release notes. It is safe because the recognizer is strict and returns `None` by
default, and because the raw subject remains available in `full`, `oneline`, JSON, the
detail pane, and the modal. Do **not** condense in `oneline` or JSON: both are
machine-consumed contracts.

### JSON

`_commit_json` in `_render_plain.py` gains:

- `parent_ids`: list of full parent ids;
- `is_merge`: boolean;
- `merge`: the merge-summary object, or `null` when unrecognized or not a merge.

`render_json`'s `query` block gains `"merges": <mode>`. These are additive, so existing
consumers keep working.

### Tests

- A **no-visual-regression** test: render a merge-free result and assert the output is
  identical with the feature present — this is the guard that the default view never
  shifts.
- Pretty and oneline output with a merge present: column allocated, glyph on the right
  row, alignment preserved.
- Legend gains the merge key only when a merge is visible.
- `full` output parent line, including a three-parent octopus merge.
- Condensation applied for a recognized PR merge, and _not_ applied for an unrecognized
  subject or an empty body.
- JSON field presence and shape.

---

## Phase `cli` — `sase vcs log --merges` and documentation

Can run in parallel with `render`; both depend only on `collect`.

### The option

In `_add_log_options` in `src/sase/main/parser_vcs.py`, inserted in alphabetical
position (between `--limit` and `--no-fetch`):

```
-m, --merges {hide,show,only}
    Merge-commit visibility: hide, show, or only (default: hide)
```

`-m` is unused by `sase vcs log` today. The repo's CLI rules require a short alias for
every public long option and alphabetically ordered options; both are satisfied.

**Require an explicit value.** Do not make `-m` bare mean "show". git's own `--merges`
means _only_ merges, so a bare flag would quietly contradict the muscle memory of the
exact users this feature is for. Requiring a value makes every invocation unambiguous.

Wire it in `_handle_log` in `src/sase/main/vcs_handler.py` by adding
`merges=args.merges` to the `CommitFilterSpec` construction — the mode then flows
through the collection layer without the handler knowing anything else about it.

### Documentation

`docs/vcs.md`, the `sase vcs log` section:

- Add the option row in alphabetical position in the options table.
- Add a short subsection explaining the three modes, the partition law, the default, and
  the relationship to git: `hide` is `--no-merges`, `only` is `--merges`, and `only`
  deliberately does _not_ use `--first-parent` so the modes partition one history.
- Note that JSON output records the mode as `query.merges` and each commit carries
  `parent_ids`, `is_merge`, and `merge`.
- Add examples to the "Common forms" block: `sase vcs log --merges only --limit 20` and
  `sase vcs log -m show --format full`.
- The section currently says merge exclusion is unconditional; correct that text.

`docs/cli.md`: extend the one-line `sase vcs log` summary to mention merge visibility.

### Tests

Extend the CLI-facing tests in `tests/test_vcs_log_run.py` (and the parser tests that
cover `sase vcs log` arguments) to assert the default is `hide`, each explicit mode
reaches `CommitFilterSpec`, and an invalid value exits with argparse's usage error.

---

## Phase `tui` — ACE Commits pane merge affordances

`src/sase/ace/CLAUDE.md` governs this phase: the footer-keybinding convention, the help
popup sync requirement, and the 57-character help-modal box width are all mandatory, not
advisory.

### The cycle key

Bind **`s`** to a new `commits_cycle_merges` action: `hide → show → only → hide`.

`s` is the established key for "cycle a filter facet" in the Artifacts tab — it is
`beads_cycle_status`, `chats_cycle_provenance`, and `files_cycle_kind` in the three
sibling sub-tabs, and the Commits sub-tab does not use it. Reusing it is consistency,
not collision. (There is an app-level `s` for `change_status`; the three existing
sub-tab bindings already resolve that the same way, so follow their pattern exactly and
confirm dispatch while the Commits sub-tab is active.)

Touch points, all of which must move together:

- `src/sase/ace/tui/bindings.py` — the Commits sub-tab block.
- `src/sase/ace/tui/actions/artifacts_commits.py` — the action list and
  `action_commits_cycle_merges`, delegating to a `cycle_merges()` method on the pane.
- `src/sase/ace/tui/commands/availability.py` — `_COMMITS_ARTIFACT_COMMANDS`.
- `src/sase/ace/tui/keymaps/metadata.py` — the action's label.
- `src/sase/default_config.yml` — the `commits_cycle_merges: "s"` entry. The gotchas
  memory calls out this file specifically; it is the one most easily forgotten.
- `src/sase/ace/tui/modals/help_modal/patches_artifact_bindings.py` — a help row.
- `build_commits_hints` in `commits_rendering.py` — a `merges` hint beside `sidecars`.

On the pane, `cycle_merges()` mirrors `toggle_sdd()`: build the next
`CommitLogFilterValues` with `replace(...)` and hand it to
`_commit_filter_values(..., close_session=False)`. That routes through the existing
collection path — no new refresh code path, which is what the TUI performance rules
require. A brief toast naming the new mode makes the cycle legible.

### Detail pane

In `build_commit_detail` in `commits_rendering.py`, for a merge:

- append `  ◆ merge` in the `MERGE` style after the short id in the header;
- add a `Parents    <short> <short>` row beside `Author` / `Committed` / `Presence`;
- label the diff section so the user knows what they are looking at: "Changes introduced
  by this merge (vs first parent)". Without that label the diff is ambiguous; with it, a
  merge row becomes a legible summary of what the PR brought in.

### Commit view modal

`CommitViewSpec` (in `prompt_panel/_agent_display_state.py`) gains
`parent_ids: tuple[str, ...] = ()`; `build_commit_view_spec` populates it. The modal
shows the same badge and parents block. Because `vcs_show_revision` now passes
`--first-parent`, the modal's diff is populated for merges without any modal-side
change.

### Config

Leave `BUNDLED_COMMITS_DEFAULT_QUERY` as `"sidecar:false since:24h"`; the implicit
`merges:hide` is the default. Document in `docs/configuration.md` that
`ace.artifacts.commits.default_query` accepts `merges:` — for example
`"sidecar:false since:24h merges:show"` for someone who wants merges visible on startup.

### Tests

- Cycling advances through all three modes and back, and each step schedules a
  recollection.
- Canonical query string shows the new mode after a cycle.
- Detail pane renders badge, parents, and the merge diff label for a merge and none of
  them for an ordinary commit.
- Help modal and hint bar include the new binding (the existing help tests will catch a
  missed update).
- A PNG visual golden for the Commits pane showing a merge row alongside ordinary rows.
  Use `just test-visual` and accept it with `--sase-update-visual-snapshots`. Existing
  goldens must be untouched — if any changes, the merge column leaked into the
  merge-free path and the render phase's no-regression guarantee is broken.

---

## Phase `floor` — Raise the `sase-core-rs` dependency window

Everything before this works in a development install because `just install` builds the
extension from the workspace's sase-core checkout. Released installs resolve the
published wheel, so `pyproject.toml` must move.

Adding a public field to `VcsCommitWire` is a minor-version event for sase-core, so the
constraint currently reading `sase-core-rs>=0.21.2,<0.22.0` needs **both** bounds moved
— raising only the floor would leave the new version outside the window. Read the
version release-plz actually published for the `core` phase and set the window
accordingly (e.g. `>=0.22.0,<0.23.0`).

Follow the repo's established pattern: a dedicated
`build(deps): raise sase-core-rs floor to <version>` commit, matching the several prior
floor bumps in this repo's history.

If the core release has not published yet, do not guess a version and do not proceed —
report the block. A wrong window either fails to install or, worse, admits a wheel whose
parser drops every commit.

Then run `just check-full` — the exhaustive gate — against the published wheel, and
confirm `tools/validate_sase_core_rs` passes its new schema-3 probe.

---

## Phase `verify` — End-to-end acceptance against real merge history

Everything before this is unit- and integration-tested. This phase confirms the feature
is genuinely usable, on a repository that actually contains merge commits — the `sase`
repo's own history, a sidecar, or a scratch repository built with `git merge --no-ff`.

Exercise and record:

1. `sase vcs log` with no merge flag → output is what it was before this epic; no merge
   column, no legend key, unchanged alignment.
2. `-m show` → merges appear, marked with `◆`, aligned, legend teaches the glyph.
3. `-m only` → merge commits only; recognized GitHub PR merges render as
   `#<n>  <PR title>`.
4. The partition law in the wild: the `hide` and `only` row counts sum to the `show` row
   count on the same scope and limit.
5. All four formats under all three modes: `pretty`, `full`, `oneline`, `json`. Confirm
   the JSON fields and `query.merges`.
6. `sase ace` → Artifacts → Commits: `s` cycles the three modes, the filter bar shows
   the `merges:` token, the detail pane shows the badge, parents, and a **non-empty**
   diff for a merge commit, and the commit modal agrees.
7. The skew probe: confirm `tools/validate_sase_core_rs` reports the schema-3
   requirement, so a stale wheel fails loudly.

Record the evidence on the phase bead. Anything that does not behave as specified is a
finding for the epic, not something to paper over.

---

## Risks and how this plan addresses them

| Risk                                                                                                           | Mitigation                                                                                                                                                                           |
| -------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| A stale `sase_core_rs` wheel silently drops every commit once the host emits 8 fields.                         | Tolerant 7-or-8-field parser (`core`), a `vcs_log_wire_schema_version` binding and validator probe that fails loudly (`wire`), and a correctly widened dependency window (`floor`).  |
| Merge commits mislabeled `synced`, and ahead/behind counts describing a different slice than the visible rows. | One mode threaded through **both** `vcs_log` and `vcs_partition_commits` (`provider`).                                                                                               |
| Merge rows showing "Diff unavailable" — on exactly the commits the feature exists for.                         | `--first-parent` on `vcs_show_revision`, verified against real git output (`provider`).                                                                                              |
| The default view silently reflowing for users who never asked for this.                                        | The merge column is allocated only when a merge is visible, plus an explicit no-regression render test and untouched PNG goldens (`render`, `tui`).                                  |
| Mode filtering applied after the row cap, so `--limit 40` returns fewer than 40 visible rows.                  | Merge visibility is a backend facet pushed into the git query; the TUI recollects on change, exactly as `sidecar:` does (`collect`).                                                 |
| A condensed headline mangling a commit subject.                                                                | The recognizer is strict and returns `None` by default; `None` renders the raw subject; `full`, `oneline`, JSON, and the detail pane always show the raw subject (`core`, `render`). |
| `only` and `show` disagreeing about which commits are merges.                                                  | The partition law, with a test on real git history (`provider`).                                                                                                                     |
