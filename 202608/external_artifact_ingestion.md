---
tier: epic
title: Mirror external issues into beads and external PRs into Patches
goal: 'Every issue in an enabled project''s external tracker has a corresponding bead
  and every PR not created by SASE''s tracked workflow has a corresponding Patch,
  kept current continuously by AXE on every enabled project on the machine, and the
  Artifacts tab presents those relationships on one integrated surface whose sub-tabs
  are Stitches, Patches, Beads, Files.

  '
phases:
- id: bead_ref
  title: external_ref bead identity field
  depends_on: []
  size: large
  description: 'bead_ref: add the nullable, partially-unique external_ref column to
    the sase-core bead schema and thread it through wire, jsonl, events, read, mutation,
    CLI, history, and search plus the Python mirrors; add the project-qualified external
    ref normalizer and widen bug_links to task beads through it.'
- id: pr_seam
  title: Pull-request provider seam
  depends_on: []
  size: medium
  description: 'pr_seam: add PullRequestWire, vcs_list_pull_requests, and split list/read/mutate
    capability probes to the VCS provider boundary, implement them in sase-github
    over gh pr list, extend the in-memory fake, and add provider_id to IssueWire.'
- id: pr_origin
  title: PR_ORIGIN field, SASE_PATCH stamp, and the external-Patch safety exclusion
  depends_on: []
  size: medium
  description: 'pr_origin: add the tri-state PR_ORIGIN Patch field across parser,
    storage, section order, and the four ACE styling surfaces; stamp SASE_PATCH in
    append_pr_tags; and structurally exclude external Patches from AXE work before
    any importer can create one.'
- id: issue_mirror
  title: external_issue_mirror chop
  depends_on:
  - bead_ref
  size: large
  description: 'issue_mirror: add the per-project builtin chop that diffs the tracker
    against beads on external_ref and creates unsized open task beads, with watermarks,
    a resumable backfill, per-pass budgets, backoff, a daily repair scan, a dry run,
    and a doctor check for detached tracker auth.'
- id: pr_mirror
  title: external_pr_mirror chop and the two-file Patch importer
  depends_on:
  - pr_seam
  - pr_origin
  size: large
  description: 'pr_mirror: add the per-project builtin chop that adopts unowned remote
    PRs as Patches, built on a new importer that locks the active and archive ProjectSpec
    files together and writes merged and closed PRs straight into the archive.'
- id: bead_bug_ui
  title: External-issue presentation and actions in the Beads pane
  depends_on:
  - bead_ref
  size: large
  description: 'bead_bug_ui: give Beads the external-issue chip, drift badge, detail
    section, capability-gated migrated actions, and bug filter tokens additively,
    while the Bugs sub-tab is still present, backed by one bounded per-project cache
    refresh.'
- id: patch_pr_ui
  title: PR badge and origin chip on Patch rows and detail
  depends_on:
  - pr_origin
  size: medium
  description: 'patch_pr_ui: render the PR badge and the origin chip as two independent
    signals on Patch rows and in the detail panel, add the origin query property,
    and add the mark-origin/adopt operation that clears unknown records.'
- id: tabs
  title: Retire Bugs, rename PRs to Patches, reorder the Artifacts sub-tabs
  depends_on:
  - bead_bug_ui
  - issue_mirror
  size: large
  description: 'tabs: collapse the Artifacts sub-tabs to Stitches, Patches, Beads,
    Files by deleting the Bugs pane and renaming the prs identifier to patches, keeping
    deprecated action aliases, and regenerating every affected text and PNG golden.'
proposed_by: bbugyi200.athena.xp
create_time: 2026-08-10 19:02:48
status: done
bead_id: sase-jd
---

- **PROMPT:** [prompts/202608/external_artifact_ingestion.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/external_artifact_ingestion.md)
- **BEAD:** [sase-jd](https://github.com/sase-org/sase--beads/blob/main/pages/sase-jd/README.md)

# Plan: Mirror external issues into beads and external PRs into Patches

## Goal

Two promises, held continuously, for every enabled project on the machine:

1. **Every issue in an external tracker has a bead.**
2. **Every PR not created by SASE's tracked workflow has a Patch.**

And one consequence: because beads become a faithful superset of tracker issues, the
Artifacts tab no longer needs a separate Bugs inventory. Its sub-tabs collapse to
**Stitches · Patches · Beads · Files**, and the external relationship is rendered
directly on the bead and Patch surfaces rather than in a parallel pane.

## Design principles

**Provenance is stored, never inferred.** SASE agents create PRs too, so the presence of
a `PR:` field proves nothing about who created it. A tri-state `PR_ORIGIN` field records
what we actually know, and `unknown` is an honest answer that produces a worklist rather
than a lie that cannot be detected later.

**Identity is a database invariant, not a convention.** The mirror runs every ten
minutes forever. "Do not create a duplicate" must be enforced by a partial-unique index,
not by a convention that erodes the first time a human edits a bead.

**Two state machines, never coupled.** An upstream issue closing does not close a bead.
Drift is _surfaced_, and reconciliation is an explicit user action.

**Safety is structural, not query-shaped.** External Patches have no branch, no
workspace, and no stitches. Excluding them from AXE work through the user-overridable
`axe_config.query` would silently evaporate the moment a user sets their own query. The
exclusion is applied before the user query, in code.

**One accent, two weights.** `#FF5F5F` — freed by retiring the Bugs sub-tab — becomes
the Artifacts tab's single **external** accent. Rendered as text it means "this artifact
is anchored outside SASE." Rendered as a filled badge (`bold #1a1a1a on #FF5F5F`) it
means "external _and_ needs reconciliation." Nothing else in Artifacts uses it. That one
rule is what makes the merged surface read as designed rather than accumulated.

## Decisions I am making, and why

The four questions this feature cannot avoid, resolved:

**Scope of "issue": mirror every tracker issue.** Not a configured label subset. The
merged UI's honesty depends on the bead list being a superset of the issue list; a label
filter silently breaks that the moment the Bugs pane is gone. A
`external_mirror.exclude_labels` knob exists as an escape hatch and defaults to empty;
whenever it is non-empty the Beads status line shows an explicit unmirrored count.

**Backfill: full, bounded by page count per run.** Not "only issues created after
opt-in." A watermark default quietly breaks the "every issue" promise. Bound work by
pages consumed per pass, never by truncating the inventory.

**Upstream closes: note-only.** Append an attributed `sase bead note` and render a drift
badge. A notification gate per upstream close reintroduces exactly the inbox flood we
are avoiding by creating beads `open` rather than `ready`.

**PR adoption breadth: all PRs on the repo, not just your own.** This is the one with no
obvious default, and I am choosing breadth deliberately. The request is "every PR not
created by SASE"; restricting to self-authored PRs breaks that promise the same way a
watermark would break the issue promise, and third-party contributions are precisely the
PRs most worth having a local record of. Breadth is only safe because the `pr_origin`
phase lands the structural AXE exclusion _before_ any importer exists — that ordering is
load-bearing, not incidental. A `external_mirror.pr_authors` config knob narrows it for
anyone who wants the old behavior; it defaults to unset (all).

## Coordination with the in-flight `sase-j8` epic

`sase-j8` ("Rename sase vcs to sase stitch and the ACE Commits sub-tab to Stitches") is
**in progress** and owns the `Commits` → `Stitches` user-visible rename in phase
`sase-j8.4`, plus the keymap and config key renames in `sase-j8.3`. Both edit
`artifact_tabs.py`, `_ARTIFACT_LABELS`, the numbered sub-tab bindings, and the Artifacts
PNG goldens.

Therefore:

- **The `tabs` phase must not begin until `sase-j8` has landed.** Check with
  `sase bead show sase-j8` before starting.
- **No phase in this epic renames anything Commits/Stitches.** The first sub-tab's label
  is `sase-j8.4`'s deliverable. This epic changes only the sub-tab _set_ and _order_,
  and renames `prs` → `patches`.
- If `tabs` finds the label is still "Commits" when it runs, that is a signal `sase-j8`
  has not landed — stop and report, do not do the rename here.

## Ground rules for every phase

- **Do not edit SASE memory files.** `sase/memory/*.md`, `AGENTS.md`, `CLAUDE.md`,
  `GEMINI.md`, `OPENCODE.md`, `QWEN.md` are off-limits; a plan file is not user
  permission. Glossary entries for "external issue" and "PR origin" are recorded as a
  follow-up for the project owner, not authored by a phase agent.
- **Use `/sase_repo`** to reach `sase-core`, `sase-github`, and `chezmoi`. Never clone
  or web-fetch them another way.
- **Run `just install` first** in an ephemeral workspace, then `just check`. Run
  `just check-full` before landing.
- **Read `sase/memory/symvision.md`** with `/sase_memory_read` before any phase that
  deletes symbols (`tabs`, and `bead_bug_ui` if it retires helpers).
- **PNG goldens**: `just test-visual`, inspect `.pytest_cache/sase-visual/` diffs by
  eye, then `--sase-update-visual-snapshots`. Any _new_ glyph must be added to the
  audited set and pass `tests/ace/tui/visual/test_tab_icon_glyphs.py` — prefer glyphs
  already in use (`○ ● ◇ ◆ ▤ ✦ ◌ ◐ ⊜ ► ▶ ▸ ▾ ✓`) over new codepoints.
- **New CLI subcommands or options**: read `sase/memory/cli_rules.md` first.
- **Rust core boundary**: bead identity and the deterministic reconciliation classifier
  belong in `sase-core`. Provider calls and filesystem orchestration stay in Python.

---

## Phase `bead_ref`: external_ref bead identity field

### Why a new column rather than reusing `refs`

Two designs were live and the answer is **both, with one job each**:

- **`external_ref`** (new): the mirror's idempotency key and the unambiguous "this bead
  _mirrors_ that issue" relation.
- **`bug:<project>#<n>` in the existing `refs` list**: what makes the link resolvable,
  searchable, and browser-openable by machinery that already exists
  (`BUILTIN_ARTIFACT_REF_KINDS` in `src/sase/artifact_ref_models.py:15` already includes
  `bug`; `sase artifact open` already handles it; the Beads pane already folds
  `issue.refs` into its search haystack).

`refs` alone cannot be the identity: it is a deliberate grab-bag with no uniqueness
constraint and no way to distinguish the issue a bead _mirrors_ from an issue it merely
_cites_. A human adding a second `bug:` ref to a mirrored bead would silently change
what the mirror considers that bead's identity, and the next pass would mint a
duplicate.

Reusing `changespec_bug_id` is blocked by real invariants, not by taste. In
`src/sase/bead/_db_schema.py:56-60` and mirrored in `sase-core`'s
`crates/sase_core/src/bead/schema.rs`:

```sql
CHECK(issue_type = 'plan' OR (changespec_name = '' AND changespec_bug_id = '')),
CHECK(changespec_name != '' OR changespec_bug_id = '')
```

Task beads structurally cannot carry a bug id today, and `src/sase/bead/cli_crud.py:120`
enforces the same rule at the CLI. Relaxing those to make room for an unrelated concept
weakens a genuine invariant. Do not touch them.

### Cross-project collision — the hazard that shapes the whole design

`bug_links._normalize_bug_id` (`src/sase/bug_links.py:37-69`) strips every project and
repo qualifier. Verified by execution:

```text
'bug:sase#42'                                       -> '42'
'bug:sase-github#42'                                -> '42'
'https://github.com/sase-org/sase/issues/42'        -> '42'
'https://github.com/sase-org/sase-github/issues/42' -> '42'
```

Issue #42 in `sase` and issue #42 in `sase-github` are indistinguishable after
normalization. `find_bug_links` is safe today only because its callers happen to pass a
project-scoped bead list — an accident of the call site, not a property of the helper.
The entire premise of this epic is "every enabled project on the machine," so any
cross-project caller silently mismatches.

**Do not widen `_normalize_bug_id`.** It is correct for the within-project `BUG:` tag
comparisons it was built for. Add a sibling:

```python
def normalize_external_ref(value: str | int | None, *, project: str) -> str:
    """Return the canonical project-qualified external ref, or ""."""
    # -> "bug:<stable-project-key>#<number>"
```

Resolve `project` through the **stable ProjectSpec directory key**
(`gh_sase-org__sase`), never the display name, so a project rename does not duplicate
every issue. Render the display name only in user-facing text, per the "Show Project
Names, Never ProjectSpec Keys" convention.

### Work

**In `sase-core`** (open with `/sase_repo`, path `crates/sase_core/src/bead/`):

- `schema.rs`: add `external_ref TEXT` (nullable) to all four schema literals that
  currently carry `changespec_bug_id` (around lines 28-57, 222-259, 313-354, 404-441),
  plus a partial-unique index:
  ```sql
  CREATE UNIQUE INDEX IF NOT EXISTS idx_issues_external_ref
      ON issues(external_ref) WHERE external_ref IS NOT NULL AND external_ref != '';
  ```
  Add `needs_external_ref_migration()` / `external_ref_migration_sql()` following the
  `needs_refs_migration()` / `refs_migration_sql()` precedent at `schema.rs:149-157`.
  Note the index needs its own `CREATE INDEX` in the migration path, not just the
  `ALTER TABLE`.
- `wire.rs`, `jsonl.rs`, `events.rs`, `read.rs`, `mutation.rs`, `cli.rs`, `history.rs`,
  `search.rs`: thread the field through. A create event carrying an `external_ref` that
  already exists must fail as a conflict, not silently overwrite.
- Tests alongside each, including a migration test that opens a pre-column store.

**In this repo:**

- `src/sase/bead/model.py`: add `external_ref: str = ""` to `Issue`.
- `src/sase/bead/_db_schema.py`: mirror the column and the partial-unique index.
- `src/sase/bead/_db_migrations.py`: additive `ALTER TABLE`, following the
  `changespec_bug_id` shape at `_db_migrations.py:45-47`, plus the index creation.
- `src/sase/bead/_db_codec.py`, `_project_mutations.py`, `cli_query.py`: read/write it.
- `src/sase/bead/cli_crud.py`: `sase bead create --external-ref` and
  `sase bead update --external-ref` / `--clear-external-ref`. These are **not** gated on
  `--patch` — that gate belongs to `changespec_bug_id` and must stay there.
- `src/sase/core/bead_mutation_facade.py` / `bead_read_facade.py` / `bead_wire.py`:
  thread through the binding.
- `src/sase/bug_links.py`: add `normalize_external_ref`; add
  `find_external_ref_links(external_ref, beads, patches)` that matches
  `bead.external_ref` and `bug:` entries in `bead.refs` **through the project-qualified
  normalizer**, and — unlike `find_bug_links` — does _not_ require
  `issue_type == PLAN and tier == EPIC`, so task beads participate. Keep
  `find_bug_links` and its epic-only matcher intact for existing callers.

### Verification

- Unit tests proving `bug:sase#42` and `bug:sase-github#42` do not collide.
- A test that two `create` calls with the same `external_ref` produce one bead and one
  reported conflict.
- Migration test: a store created before the column opens, migrates, and round-trips.
- `just check`, then `just check-full`.

---

## Phase `pr_seam`: Pull-request provider seam

The issue seam is complete: five hooks in `src/sase/vcs_provider/_hookspec.py:198-229`,
implemented in `sase-github`'s `plugin.py`, probed structurally by `supports_issues()`
(`_plugin_manager.py:327`) without touching the network, with a network-free fake in
`vcs_provider/testing.py`.

There is **no** PR-listing hook and no `PullRequestWire` — a repo-wide grep for
`vcs_list_pull_requests|list_pull_requests|PullRequestWire` returns nothing. This phase
is a direct transcription of the issue seam.

### Work

- `src/sase/vcs_provider/_types.py`: add a frozen `PullRequestWire` with
  `provider_id, number, url, title, body, state, is_draft, author, head_ref, base_ref, created_at, updated_at, closed_at, merged_at`.
  Tuple-valued collections only, matching `IssueWire`'s immutability contract. Also add
  `provider_id: str = ""` to `IssueWire` (the GitHub node id) for robust cursors and
  diagnostics.
- `src/sase/vcs_provider/_hookspec.py` and `_base.py`:
  `vcs_list_pull_requests(cwd, state, limit) -> list[PullRequestWire]`.
- `src/sase/vcs_provider/_plugin_manager.py` and `_registry.py`:
  `supports_pull_requests()` following the `supports_issues()` structural probe.
- **Split the capability probe.** `supports_issues()` today requires _all five_ hooks —
  its docstring says "All operations are required so a partial implementation cannot
  claim CRUD capability." That is correct for CRUD, but a synchronizer needs only
  listing, and a read-only provider should still populate badges. Add
  `supports_issue_listing()` / `supports_issue_reads()` / `supports_issue_mutations()`
  (or one structured capability record) so ACE can gate each operation independently
  instead of hiding all issue context when a provider cannot write. Keep
  `supports_issues()` as the all-five alias for existing callers.
- `src/sase/vcs_provider/testing.py`: extend the in-memory fake with PRs and with
  configurable partial capability, so downstream phases can test gating without network.
- **In `sase-github`** (open with `/sase_repo`): implement `vcs_list_pull_requests` over
  `gh pr list --json ...` following the `vcs_list_issues` shape at `plugin.py:224-256`,
  including the `_UNBOUNDED_ISSUE_LIMIT` convention and `GitHubIssueError`-style error
  wrapping. `gh pr list` needs `--state all` to see merged and closed PRs.
- Confirm and document that `gh` reports `mergedAt` distinctly from `closedAt`; the
  status mapping in `pr_mirror` depends on telling merged from closed-unmerged.

### Verification

Provider-boundary tests against the fake; a `sase-github` test over recorded `gh` JSON.
No test may require network.

---

## Phase `pr_origin`: PR_ORIGIN field, SASE_PATCH stamp, and the safety exclusion

This phase lands the field, the forward marker, and — critically — **the AXE exclusion,
before any code exists that can create an external Patch.** That ordering is the whole
point of separating this phase from `pr_mirror`.

### The field

`PR_ORIGIN: sase | external | unknown`, **absent ⇒ `unknown`**.

Absent-⇒-`sase` would be wrong in a way that is permanent: every Patch in existence
today has no such field, so the first pass would assert SASE provenance for the entire
history without evidence. Mostly true — but any PR the user hand-created and then
tracked with `sase commit` is mislabeled forever with no way to detect it. `unknown` is
honest and gives the backfill an explicit worklist. The name `PR_ORIGIN` rather than
`ORIGIN` stays precise once agents start working adopted Patches: it is the origin of
the _PR association_, not of the Patch.

Add `"PR_ORIGIN:"` to `PATCH_SECTION_ORDER` in `src/sase/ace/patch/section_order.py`
immediately after the review-URL prefixes and before `"BUG:"`. Leave
`CHANGESPEC_SECTION_ORDER` — the legacy alias — untouched.

Parse it in `src/sase/ace/patch/parser.py` alongside the `BUG:` branch at line 151; add
`pr_origin` to the `Patch` model in `src/sase/ace/patch/models/patch.py` and to the
parser state.

Per `src/sase/ace/CLAUDE.md:5-14`, a new Patch field spelling must be updated in **all
four** styling surfaces:

1. `home/dot_config/nvim/syntax/saseproject.vim` in the **chezmoi** repo (open with
   `/sase_repo`)
2. `src/sase/ace/display.py` — follow the `BUG:` block at line 119
3. `src/sase/ace/query/highlighting.py` — `QUERY_TOKEN_STYLES`
4. `src/sase/ace/tui/widgets/patch_detail.py`

### The forward marker

Stamp `SASE_PATCH=<reserved-name>` on tracked PR creation. The ordering in
`src/sase/workflows/commit/workflow.py` already works: the suffixed name is reserved at
`:166`, the payload is decorated at `:189-193`, a checkpoint carrying `reserved_name` is
saved at `:212`, and only then does `dispatch()` create the PR at `:227`. The name is
known before the body is built.

**The marker must go in `append_pr_tags`, not `build_pr_body`.** `build_pr_body`
(`src/sase/workflows/commit/pr_operations.py:111-116`) early-returns when
`SASE_ARTIFACTS_DIR` is unset, so the `**Model:**` / `**Agent:**` footer is written
_only_ for agent-driven commits — a human running `sase commit pr` interactively
produces a fully SASE-tracked PR with no footer at all. Note that `append_pr_tags`
itself early-returns `if not tags` (`pr_operations.py:97`), so `SASE_PATCH` needs an
unconditional path ahead of that guard.

This also settles a tempting shortcut: **`SASE_AGENT=` is not a usable provenance
signal.** It is absent for human-run tracked PRs and can appear in a manually created PR
via `gh pr create --fill`. It is evidence in neither direction.

Verify in `sase-github` that `payload["message"]` tags actually reach the PR **body** —
the listing API reads the body, not the commit message. If they do not, the marker needs
its own body path.

Write `PR_ORIGIN: sase` on tracked Patch creation in the same change.

### The safety exclusion

An adopted Patch has no branch, no workspace, and no stitches, but AXE chops scan
Patches and start real work against them. There is **one choke point**, not four:
`runtime.filtered_patches` is consumed by `hook_checks`, `mentor_checks`,
`workflow_checks`, `pending_checks_poll`, `comment_zombie_checks`, and
`suffix_transforms`, and is computed in a single place,
`src/sase/axe/chop_runner_context.py:49`:

```python
filtered_patches = all_patches
if axe_config.query:
    mask = evaluate_query_many_fn(axe_config.query, all_patches)
    filtered_patches = [cs for cs, keep in zip(all_patches, mask, strict=True) if keep]
```

Apply the exclusion **before** that block, structurally:

```python
all_patches = [p for p in find_all_patches_fn() if p.pr_origin != "external"]
```

Do the same in the Lumberjack tick equivalent (find it by following the other writer of
`all_changespecs.json` / `filtered_changespecs.json`).

Expressing this as `-origin:external` in the AXE query config is **not** an acceptable
sole mechanism: `axe_config.query` is user-supplied and overridable at runtime
(`sase axe start --query ...`, `docs/axe.md:96,348`). Any user who sets their own query
would silently lose the exclusion and the next tick would launch mentor agents against
third-party PRs.

`pr_submitted_checks` is a separate code path — it calls
`check_cycle_runner.run_full_check_cycle()` and does not take `filtered_patches`. Guard
it independently.

Ship the query property too (in `patch_pr_ui`), but only for UI filtering and manual
opt-in ("show me external Patches"), never as the safety mechanism.

### Verification

- A test asserting external Patches are absent from `filtered_patches` **when
  `axe_config.query` is empty, when it is a user query that would have matched them, and
  when it is a user query that matches nothing.**
- A `pr_submitted_checks` test with an external Patch present.
- Round-trip parse/serialize tests for all three `PR_ORIGIN` values and for absence.
- `sase ace` renders the field in all four styling surfaces.

---

## Phase `issue_mirror`: external_issue_mirror chop

### Placement and fan-out

**Core builtin, `for_each` fan-out.** The chop script lives in `src/sase/scripts/`
(registered in `pyproject.toml` alongside the other `sase_chop_*` entry points) because
reconciliation semantics — what a bead is, idempotency, status mapping — are core domain
logic a future GitLab plugin must not reimplement. The plugin supplies only provider
hooks.

Registration in `src/sase/default_config.yml` under the **`checks`** lane, whose own
description already says it is "for checks that can tolerate delay or may touch remote
PR state":

```yaml
- name: external_issue_mirror
  script: sase_chop_external_issue_mirror
  run_every: "10m"
  timeout: "2m"
  for_each:
    source: projects
    vcs: [git, gh]
  description: |-
    Mirror every external tracker issue into a task bead

    ...
```

`for_each: {source: projects}` (`src/sase/axe/_config_targets.py:176-190`,
`docs/configuration.md:2054,2102-2112`) expands one stable instance per enabled project
keyed `external_issue_mirror[<project>]`, each with independent scheduling, history,
checkpoints, and once-per state, and disabling a project stops its instance
automatically. A single-instance manual loop over projects would couple every project
into one pass — one project's rate limit or auth failure stalls the rest — and would
give one shared state file where per-project watermarks belong.

**Note:** no chop in `default_config.yml` uses `for_each` today; this is its first
production use. Expect to exercise `sase axe chop list --verbose` and confirm instance
identity, per-instance state directories, and teardown-on-disable actually behave as
documented, and file a task bead via `/sase_new_task` if they do not.

`vcs: [git, gh]` is a filter on VCS _kind_, not on issue _capability_. Gate at runtime
on `supports_issue_listing()` and decline cleanly with a recorded reason otherwise.

10m rather than 5m: two provider invocations per project per pass, and this is an
inventory view, not a pager.

### What it creates

For each issue with no covering bead: a **`task`** bead, status **`open`** (never
`ready`), **size unset**.

- `ready` would raise a `TaskTriage` gate per incoming issue and flood the inbox on pass
  one.
- `size` is nullable (`_db_schema.py:37-42`) and `ready` requires only
  `issue_type = 'task'`, not a size. A chop cannot honestly estimate; `large` would
  inject a fabricated number that looks like a judgment. NULL size makes "needs triage"
  mechanically visible via `size:none` and naturally gates promotion to `ready` on a
  human setting one.

Fields: title from the issue title; description from the issue body plus a provenance
line naming the issue URL and the mirroring chop; `external_ref` =
`normalize_external_ref(number, project=<stable key>)`; and the same `bug:<project>#<n>`
string appended to `refs`.

### Cursors, backfill, and idempotency

1. **First run** does an exhaustive, resumable backfill, indexing all local
   `external_ref` values and `bug:` refs **before writing anything**. Bound work per run
   by **page count**, not by truncating the inventory.
2. Persist a per-project, per-object-type high-water mark (last processed `updated_at`
   plus the stable `provider_id` as tie-breaker) in the expanded instance's state
   directory.
3. Each incremental run starts with a ~10 minute **overlap window**, pages to
   exhaustion, and dedupes by `provider_id` — tolerating equal timestamps and
   crash-after-write.
4. Advance the checkpoint **only after every planned local mutation succeeds**.
5. **Daily full repair scan** to catch missed updates, lost state, transfers, and
   renames. This is what makes the "every issue" promise recoverable rather than
   aspirational.

**The idempotency key is `(stable project key, issue number)` and must not route through
`_normalize_bug_id`.** See the `bead_ref` phase.

### Write safety

The mirror becomes a _writer_ on a store that live agents also write, and every mutation
commits to the bead sidecar. Follow `src/sase/scripts/sase_chop_bead_store_refresh.py`:
bounded per-project lock waits derived from the chop timeout, a whole-pass work budget,
and persistent exponential backoff with a state file.

**Mutation must acquire the store lock, rebuild the identity index while still holding
it, then append the create event only if the issue is still uncovered.** Never
list-then-create across two unlocked operations. The partial-unique index is the
backstop, not the plan.

Cap creations per pass following the `managed_tmp_reap` "at most N per pass" convention,
so a first run against a large backlog converges over several passes instead of one
enormous commit.

### Upstream state changes

Never auto-close a bead when its issue closes; never delete a bead for a deleted or
transferred issue. Append an attributed `sase bead note` (append-only, already
sanctioned for supplementary evidence) recording the transition, and let the UI render
the drift. A deleted or transferred issue marks the link **stale**, which the detail
panel shows.

### Degraded runs

Auth failure, rate limiting, or an outage produces a **degraded run**, never a deletion.
One malformed remote record is reported with its provider id and **must not advance the
checkpoint past unprocessed data**.

### Surfaces

- `sase bead sync-external [--project P] [--dry-run] [--full]` — the manual entry point
  and the same code path the chop runs. `--dry-run` shows exact creations without
  mutating anything and without advancing cursors. Read `sase/memory/cli_rules.md` with
  `/sase_memory_read` before adding it.
- A `src/sase/doctor/` check for tracker auth **in the detached lumberjack
  environment**. The Bugs pane proves `gh` works from the interactive TUI; the axe
  daemon runs detached with a different environment, and a silent auth failure there
  would look exactly like "no issues."
- Structured chop summary: issues seen, beads created, notes appended, conflicts, pages
  consumed, whether the checkpoint advanced.

### Known limitation to state plainly

Two machines reconciling stale copies of a hosted bead sidecar can both import the same
issue. Go through the existing publication/integration path and make integration
collapse simultaneous imports by semantic bug identity. The partial-unique index makes
the local half enforceable; the cross-machine half is bounded by integration.

---

## Phase `pr_mirror`: external_pr_mirror chop and the two-file importer

Same lane, cadence, fan-out, cursor discipline, budget, backoff, dry run, daily repair
scan, and doctor coverage as `issue_mirror`, gated on `supports_pull_requests()`. Two
chops rather than one: two stores, two failure modes, two capability gates, two backfill
costs. A shared library handles cursor I/O, overlap windows, URL normalization, and
structured reports.

### Classification

Deterministic, in this order:

1. **Canonical PR URL already owned by a local Patch** → do not create another; preserve
   its existing origin.
2. **Valid `SASE_PATCH` marker** → SASE-origin. If the named Patch is a bare reservation
   or missing (the crash window between remote PR creation and local Patch completion),
   repair it with `PR_ORIGIN: sase` rather than importing it as external.
3. **No marker, no local Patch** → create with `PR_ORIGIN: external`.
4. **Ambiguous historical evidence** → `unknown`; do not guess.

**Canonically normalize PR URLs** (host, owner/repo, number) before comparison. Raw
string equality breaks on enterprise hosts, URL variants, and renamed repos.

The bootstrap classifier for everything predating the marker is a **set difference**:
collect every `pr_url` across both ProjectSpec files via
`iter_patch_project_file_records()` (`src/sase/ace/patch/discovery.py:50`), list remote
PRs, adopt the remainder.

State the enforceable contract plainly in the docs: this identifies "created by SASE's
tracked PR workflow," **not** "created by a SASE agent." An agent that bypasses the
tracked workflow and calls `gh pr create` directly is indistinguishable from a human.

### Status mapping

| Remote PR state        | Patch status | ProjectSpec file |
| ---------------------- | ------------ | ---------------- |
| Open draft             | `Draft`      | active           |
| Open, ready for review | `Mailed`     | active           |
| Merged                 | `Submitted`  | archive          |
| Closed unmerged        | `Archived`   | archive          |

`Mailed` rather than `Ready` for an open PR: `Ready` means _locally_ ready to mail,
which a live remote PR has already passed.

### The importer — why `add_patch_to_project_file` cannot be reused

Two traps, both real:

- `VALID_TRANSITIONS` (`src/sase/status_state_machine/constants.py:44-60`) admits only
  `Mailed → Submitted`. A merged PR must be **written directly into the archive file**,
  never transitioned into it.
- `add_patch_to_project_file()` (`src/sase/workflows/commit/patch_operations.py:242`)
  resolves its destination through `get_project_file_path(project)`
  (`src/sase/workflows/utils.py:11-22`), which returns the **active** spec only. It
  accepts `status` but cannot target the archive.

So: a **new importer**, which locks the active _and_ archive ProjectSpec files, checks
**both** for the canonical PR identity and for the candidate name, and writes to the
correct destination in one operation.
`ARCHIVE_STATUSES = {Submitted, Archived, Reverted}` selects the destination.

### What an adopted Patch gets, and does not

Gets: a unique name derived from the PR title or head branch; a description from the PR
title and body with footer tags stripped; `PR:`; `PR_ORIGIN: external`.

Does **not** get: `initial_hooks`, a fabricated `PARENT`, stitches, or a workspace. Do
not infer `PARENT` from the PR's base branch — only from explicit provider metadata or a
SASE marker.

### Verification

- **The crash window**: a test where the remote PR is created but the local Patch
  completion fails, asserting the next mirror pass repairs rather than duplicates.
- A merged PR lands in the archive file without an illegal transition.
- A PR whose URL differs only by host form or trailing slash is not duplicated.
- Adopted Patches never appear in `filtered_patches` (already covered by `pr_origin`,
  but assert it end-to-end here with a real adopted Patch).

---

## Phase `bead_bug_ui`: External-issue presentation and actions in the Beads pane

**This phase is purely additive and lands while the Bugs sub-tab is still present.**
That is deliberate: the merge is only honest when the bead surface is at least as
informative as the pane it replaces, and keeping both alive briefly lets the two be
compared side-by-side on real projects.

### The visual rule

`#FF5F5F` — the Bugs accent, freed by the `tabs` phase and re-keyed as `EXTERNAL_ACCENT`
— becomes the Artifacts tab's **external** accent:

- **as text** → this artifact is anchored outside SASE
- **as a filled badge** (`bold #1a1a1a on #FF5F5F`) → external _and_ needs
  reconciliation

Nothing else in Artifacts uses it. Filled badges are already this codebase's vocabulary
for a strong categorical fact (`build_beads_scope`, `issue_meta`).

### Row chip

`_bead_text` in `src/sase/ace/tui/widgets/artifacts/beads_rendering.py` gains one chip,
placed immediately after the title and before the `[+1]` / `[reopen]` badge cluster.
State glyphs carry over verbatim from the retiring Bugs pane so recognition survives the
merge (`○` open, `●` closed — `bugs_rendering.issue_row` already renders both):

```text
◈ sase-ab12  Fix project alias resolution      ○#418   ► ready
◈ sase-cd34  Retire legacy provider hook       ●#391   ► ready
◈ sase-ef56  Refactor local cache                      ► ready
```

- `○#418` in `bold #FF5F5F` — upstream open.
- `●#391` in `bold #FF5F5F` — upstream closed.
- **Drift** (upstream closed, bead open, or the reverse) renders the chip as a filled
  badge: `●#391` in `bold #1a1a1a on #FF5F5F`.
- **Stale link** (issue deleted or transferred) renders `?#391` filled, same weight.
- **Multiple issues** get a bounded count — `○#418 +2` — and a picker. Never a silent
  first-match.

Do not use a new prefix glyph next to `▤`; the row prefix region is already carrying the
type glyph, the triage `✦`, and the plan-link `▤`, and a fourth marker there degrades
the scan. The chip in the badge region is where a per-artifact fact belongs.

**The list contains only bead rows.** Remote-only issues never become rows. If a bead is
missing for an issue, that is the mirror's problem to fix, not the list's problem to
paper over — and the status line will say so.

### Detail panel

`beads_detail.py` gains an **External issue** section:

```text
## External issue

  ○ OPEN   sase#418   ·   by bbugyi200   ·   3 comments   ·   updated 2h ago
  labels   bug · ace
  link     https://github.com/sase-org/sase/issues/418

  Relationship   mirrored by this bead
  Also linked    patch sase_fix_alias_1 · epic sase-k2
```

**Relationship** is where the `external_ref`-vs-`refs` distinction surfaces — "mirrored
by this bead" when `external_ref` matches, "referenced by this bead" when only a `bug:`
ref does. That distinction is real and worth showing once, in the one place a user goes
for detail; it does not belong on the row.

**Also linked** comes from `find_external_ref_links` (reverse links to Patches and
epics).

### Actions

Migrate the Bugs pane operations from `src/sase/ace/tui/actions/artifact_bugs.py`, each
**individually capability-gated** on the split probes from `pr_seam`: open in browser,
view remote body, edit, close/reopen, copy URL, copy `bug:` ref, attach an existing
issue to a bead, create an issue for an unlinked bead, refresh.

Both panes currently bind `j k f o y a s R e`. With Bugs gone:

- `o` (open issue in browser) and `y` (copy ref) move to Beads, **gated on "the selected
  bead has an external issue link."** That matches the footer convention in
  `src/sase/ace/CLAUDE.md`: a keymap appears in the footer iff its condition is
  sometimes true and sometimes false — which is exactly true here.
- `e` and `s` genuinely collide with bead edit and bead status cycle. Issue _mutations_
  get a `b`-prefixed pair: `be` edit issue, `bs` close/reopen issue. Do not steal the
  bead primaries; a user reaching for `e` expects to edit the bead.

Update the help modal and the command palette in the same change, per the Help Popup
Maintenance rule in `src/sase/ace/CLAUDE.md`, and keep the help modal's 57-character box
width.

### Filters

- Redefine `has:bug` — currently computed from `issue.patch_bug_id` in
  `beads_filtering.py:186` — to mean **"has an external issue link"** (either
  `external_ref` or a `bug:` ref).
- Add a `bug:` value key: `bug:none`, `bug:open`, `bug:closed`, `bug:drift`, and
  `bug:<number>`.
- Add provider label tokens so `label:regression` selects by tracker label.

### Data loading

**Never fetch per row.** One bounded batch refresh per project into a cache, with
last-refresh time and a stale indicator in the pane status line
(`external · refreshed 4m ago`, or `external · unavailable (tracker auth)`). Never a
provider call while rendering; local navigation must not block on the network. Reuse the
`_BUG_CACHE_TTL_S = 60.0` shape from `bugs.py`.

Read `sase/memory/tui_perf.md` with `/sase_memory_read` before touching the load path.

---

## Phase `patch_pr_ui`: PR badge and origin chip on Patch rows and detail

**The PR badge and the origin chip are two independent signals and must never collapse
into one boolean.** The badge answers "does this Patch have a remote review?"; the chip
answers "who created that PR?"

```text
◆ sase_refactor_cache_1     Mailed      PR#812
◆ sase_fix_windows_path_1   Mailed      PR#819   external
◆ sase_old_parser_2         Submitted   PR#601   origin?
◆ sase_local_experiment_1   WIP
```

- `PR#812` in the Patches accent `#00D7AF`.
- `external` in `bold #FF5F5F` — the same external accent as the bead chip, because it
  is the same concept.
- `origin?` in `dim #FF5F5F` — known-unknown, not an error.
- **`sase` origin renders nothing.** It is the expected case and a chip on every row is
  noise; the same "sometimes true, sometimes false" rule that governs the footer governs
  this. External Patches stand out precisely because they are the exception.

### Implementation constraint

Three functions in `src/sase/ace/tui/widgets/_patch_list_helpers.py` must stay in sync
or rows will silently fail to repaint: `format_patch_option`,
`calculate_entry_display_width`, and `row_signature`. `PatchList._patch_patch_row_impl`
(`src/sase/ace/tui/widgets/patch_list.py:311-392`) refuses to patch a row whose new
width exceeds the cached panel width, so a width function that does not account for the
new chip produces rows that render stale until a full rebuild. Update all three, and add
a test that a Patch gaining an origin chip repaints in place.

### Detail panel

`patch_detail.py` shows `PR_ORIGIN` alongside `PR:` and, for external Patches, an
explicit line: _"Adopted from an external PR — SASE automation does not act on this
Patch."_ That sentence is the user-facing half of the structural exclusion; without it,
an external Patch sitting inert in the list looks like a bug.

### Query property

Add `origin` to `VALID_PROPERTY_KEYS` in `src/sase/ace/query/tokenizer.py:31`, a
`_match_origin` branch in `match_property` (`src/sase/ace/query/matchers.py:168-196`,
which currently handles `status`, `project`, `ancestor`, `name`, `sibling`), and the
token style in `QUERY_TOKEN_STYLES`. Also update the tokenizer's "Unknown property key"
error message at line 405, which enumerates the valid keys. Fold `pr_origin` into
`get_searchable_text` in `src/sase/ace/query/searchable.py`.

`origin:external` is for UI filtering and manual opt-in. It is **not** the safety
mechanism — that landed structurally in `pr_origin`.

### The adopt operation

Add an explicit **"mark PR origin"** action, in the TUI and as
`sase patch set-origin <name> <sase|external|unknown>`, to clear `unknown` records
deliberately. This is the user-facing half of the tri-state decision: `unknown` produces
a worklist, and this is how the worklist gets drained. Surface the count of `unknown`
Patches in the Patches pane status line so the worklist is visible rather than
theoretical.

---

## Phase `tabs`: Retire Bugs, rename PRs to Patches, reorder the sub-tabs

### Hard precondition

**Do not start this phase until both are true:**

1. `sase-j8` has landed (`sase bead show sase-j8`). See the coordination section.
2. `issue_mirror` has **run clean against real projects**, and the Beads pane shows a
   chip for the issues the Bugs pane shows. The merge is honest only if the bead list is
   a superset of the issue list; any issue excluded by watermark, filter, or budget
   becomes invisible the moment the Bugs pane disappears.

If either is false, stop and report rather than proceeding.

### The result

```text
  STITCHES     PATCHES     BEADS     FILES
   #FFD700     #00D7AF   #D787FF   #FFAF5F
```

`ARTIFACTS_SUBTAB_ORDER` becomes `("stitches", "patches", "beads", "files")`.

### Retire Bugs

In `src/sase/ace/tui/artifact_tabs.py`: drop `"bugs"` from `ArtifactsSubTab`,
`ArtifactsPaneKey`, `ARTIFACTS_SUBTAB_ORDER`, and `ARTIFACTS_PANE_IDS`. Drop it from
`_ARTIFACT_LABELS` and `_DETAIL_SCROLL_IDS` in
`src/sase/ace/tui/widgets/artifacts/view.py` and from `_PLACEHOLDER_COPY` in `panes.py`.

**Re-key `#FF5F5F` out of `ARTIFACTS_ACCENTS` and into a named `EXTERNAL_ACCENT`
constant** so the visual vocabulary survives with a precise meaning rather than
lingering as a dead sub-tab entry.

Delete: `widgets/artifacts/bugs.py` (598 lines), `widgets/artifacts/bugs_rendering.py`
(79), `actions/artifact_bugs.py` (476), `ace/tui/artifacts_bugs.py`, and their tests
(`tests/ace/tui/test_artifacts_bugs.py`, `test_artifacts_bugs_backend.py`,
`visual/test_ace_png_snapshots_artifacts_bugs.py`) plus the three
`artifacts_bugs_*_120x40.png` goldens. Anything `bead_bug_ui` still needs must have been
_moved_, not left behind — check before deleting.

Read `sase/memory/symvision.md` with `/sase_memory_read` before running the lint gate;
dismantling a pane orphans symbols and symvision will fail otherwise.

Keep `show_artifacts_bugs` as a **deprecated alias routing to Beads**, so saved UI
state, command-palette entries, and muscle memory do not hard-fail.

### Rename prs → patches

Touch points: `ArtifactsSubTab`, `ArtifactsPaneKey`, `ARTIFACTS_PANE_IDS`,
`ARTIFACTS_ACCENTS`, `_ARTIFACT_LABELS` (`"PRs"` → `"Patches"`), `ArtifactsPrsPane` →
`ArtifactsPatchesPane`, the `entry_navigator` guard, `show_artifacts_prs`,
`commands/types.py`, `tab_quickstart.py`,
`modals/help_modal/patches_artifact_bindings.py`, `copy_targets.py`,
`_app_action_availability.py`, `_app_watchers.py`, `keymaps/mode_keymaps.py`,
`actions/artifacts_navigation.py`, `actions/clipboard/ _palette_registry.py`, DOM ids,
and `styles.tcss` selectors.

Keep `prs` as a **compatibility alias** for saved UI state, commands, and tests —
consistent with established practice in this repo (`ChangeSpec = Patch`,
`changespec_bug_id`, `find_all_changespecs`, `reset_changespec_pr_url`).

`ArtifactsPrsPane` already wraps the Patch view, so this is a naming and metadata
correction, not a rewrite.

### The silent renumber

Number keys are generated from `ARTIFACTS_SUBTAB_ORDER` in
`src/sase/ace/tui/keymaps/bindings.py:20-28` and again in
`src/sase/ace/tui/bindings.py:103-111`. Removing `bugs` and reordering changes every
sub-tab's number **with no error**:

| key | before   | after       |
| --- | -------- | ----------- |
| `1` | Stitches | Stitches    |
| `2` | Beads    | **Patches** |
| `3` | Bugs     | **Beads**   |
| `4` | PRs      | **Files**   |
| `5` | Files    | —           |

Muscle memory breaks silently. Call this out in the commit message, and make sure the
sub-tab strip renders its numbers (`show_numbers=True` is already set) so the new
mapping is discoverable at a glance.

### Goldens and docs

Displayed text changes, so text-comparing TUI tests and every PNG golden showing the
Artifacts tab strip need regeneration. Run `just test-visual`, inspect
`.pytest_cache/sase-visual/` diffs by eye before accepting, then rerun with
`--sase-update-visual-snapshots`. Rename snapshot ids that name the sub-tab
(`artifacts_prs_*` → `artifacts_patches_*`) along with their PNGs.

Update `docs/` wherever the Artifacts sub-tabs are enumerated.

Finish with `just check-full`.

---

## Risks carried across the epic

1. **Bead-store write contention.** The mirror writes to a store live agents also write.
   Mitigated by the `bead_store_refresh` lock/budget/backoff pattern and by
   index-under-lock mutation. (`issue_mirror`)
2. **First-run flood.** Hundreds of open issues minting hundreds of beads on pass one.
   Mitigated by durable watermarks plus a per-pass creation budget. (`issue_mirror`)
3. **Cross-machine duplicates.** Two machines reconciling stale sidecar copies. Bounded
   by going through the existing publication/integration path; the partial-unique index
   makes the local half enforceable. (`bead_ref`, `issue_mirror`)
4. **Tracker auth in the detached lumberjack.** The interactive TUI's working `gh`
   proves nothing about the axe daemon's environment. Mitigated by a `src/sase/doctor/`
   check. (`issue_mirror`)
5. **Project rename.** Persisted refs must resolve through the stable project key or
   project aliases, or a rename duplicates every issue. Compounded by the normalization
   collision. (`bead_ref`)
6. **Symvision orphans** when the Bugs pane is dismantled. (`tabs`)
7. **PNG goldens** for both the merge and the rename, on top of `sase-j8`'s own golden
   churn. (`tabs`, `bead_bug_ui`)
8. **Tracker noise.** Trackers hold feature requests and questions, not only bugs. The
   label filter is the escape hatch, but using it requires the unmirrored-count status
   line so the superset precondition stays visible. (`issue_mirror`, `bead_bug_ui`)
9. **`for_each` has no production usage yet.** Its first real exercise is this epic.
   (`issue_mirror`)

## Deliberate non-goals

- **Webhooks.** Polling is the source of truth. Webhooks need a reachable endpoint, hook
  configuration, delivery auth, and replay handling, and would still need backfill for
  downtime. A receiver can later force an immediate run of the _same_ reconciler; do not
  build a second event-processing domain model.
- **Bidirectional write-back.** Editing a bead does not edit its issue. Issue mutations
  stay explicit user actions.
- **Renaming Commits to Stitches.** `sase-j8.4` owns it.
- **Glossary and memory-file entries** for "external issue" and "PR origin". These need
  the project owner's explicit permission and are recorded here as a follow-up, not
  assigned to a phase agent.
