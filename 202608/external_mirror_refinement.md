---
tier: epic
title: Configurable external mirror filters, its own lumberjack, and two-way bug/PR sync
goal: "A user controls exactly which external bugs and pull requests become beads and
  Patches, release-please and release-plz PRs are excluded by default, the mirror chops
  run in their own generously paced lumberjack instead of overrunning the checks lane, a
  bead linked to a bug tracks that bug's open/closed state, an adopted external Patch
  tracks its PR's merge state, and the ProjectSpec corruption the mirror has been
  silently accumulating is fixed and repaired.

  "
phases:
  - id: spec_repair
    title: ProjectSpec description truncation and duplicate-block repair
    depends_on: []
    size: large
    description: "spec_repair: fix the two-blank-line record terminator that silently
      drops any Patch whose DESCRIPTION contains a blank run, in both the Python and
      Rust parsers plus the block writer, and add the raw-text de-duplication repair
      that reclaims the archives the external PR mirror has already corrupted.

      "
  - id: filters
    title: Configurable bug and pull-request filters
    depends_on: []
    size: large
    description: 'filters: add external_mirror.issues.filters and
      external_mirror.pull_requests.filters as glob criterion lists with "!"-prefixed
      exclusions, default the PR head-ref criterion to drop release-please and
      release-plz branches, and migrate exclude_labels and pr_authors onto the new
      surface as deprecated aliases.

      '
  - id: lane
    title: Dedicated external_mirror lumberjack and lane-independent state
    depends_on: []
    size: medium
    description: "lane: move both mirror chops out of the 300-second checks lane into a
      new external_mirror lumberjack with a 15-minute interval and 5-minute chop
      timeout, and relocate the PR mirror's cursor and backoff state out of the lane
      directory so no consumer hardcodes a lane name again.

      "
  - id: bug_status
    title: Bug state drives mirrored bead status
    depends_on:
      - filters
    size: large
    description: "bug_status: reverse the epic's original note-only decision and close
      or reopen a mirrored bead when its tracker issue closes or reopens, guarded so an
      in-progress, claimed, or parented bead only gets the note it gets today.

      "
  - id: patch_status
    title: Adopted external Patches track their pull request
    depends_on:
      - spec_repair
      - filters
    size: large
    description: "patch_status: add the refresh action the external PR classifier is
      missing so an already-adopted Patch follows its PR from open to merged or closed
      and moves from the active ProjectSpec into the archive, across the sase-core wire
      and the Python importer.

      "
  - id: perf
    title: Bounded per-pass cost for the PR mirror
    depends_on:
      - spec_repair
      - patch_status
    size: medium
    description:
      "perf: stop re-reading and re-parsing the whole active-plus-archive ProjectSpec
      index once per mutation in both the sync loop and the importer, replacing it with
      one locked batch apply over an incrementally maintained index."
proposed_by: bbugyi200.athena.yn
create_time: 2026-08-12 11:27:53
status: wip
---

- **PROMPT:**
  [prompts/202608/external_mirror_refinement.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/external_mirror_refinement.md)

# Plan: Refine the merged external bug and PR mirror

## What is actually happening today

Everything below was measured on this machine on 2026-08-12 against the real
`gh_sase-org__sase` project, not inferred from the code.

**The PR mirror times out on every run and drags its lane past its interval.** Every
`external_pr_mirror[sase]` run since 10:00 has ended
`"status": "timeout", "error": "timed out after 120s"` with zero output bytes
(`~/.sase/axe/lumberjacks/checks/chops/external_pr_mirror[sase]/runs/`). The `checks`
lane declares `interval: 300`, but its own `status.json` reports `uptime_seconds: 1226`
across `cycles_run: 4` — an average cycle of ~306s, with the 11:11:36 cycle taking 621s
before the next one started. The overrun is real and it delays `pr_submitted_checks` and
`stale_running_cleanup`, which share that lane.

**The root cause is silent ProjectSpec data loss, not slow I/O.**
`gh_sase-org__sase-archive.sase` is **33.8 MB** and contains **3375 `NAME:` lines** —
but only **289 unique names**, and `parse_project_file` returns only **264 Patches**.
The 25 missing names are exactly the release-please Patches
(`chore_master_release_0_1_1_1` and siblings), each appended **129 times**.

The mechanism is precise. `format_patch_block` (`src/sase/ace/patch/storage.py:56`)
prefixes every description line with two spaces, so a blank line inside a description is
written as `"  "`. Both parsers then treat any line that strips to empty as a blank and
end the record after two of them:

- Python: `parse_patch_from_lines` (`src/sase/ace/patch/parser.py:285-291`) tests
  `line.strip() == ""`.
- Rust: `crates/sase_core/src/parser.rs:437-441` tests `line.trim().is_empty()` — the
  identical rule, so the two are bug-compatible.

A release-please body contains `---` followed by two blank lines. The record therefore
ends before its `PR:` and `STATUS:` lines are read, `build_patch()` returns `None`
because `status` is unset (`parser.py:93`), and the Patch vanishes from
`read_project_patch_index`. With the record invisible, both the `by_pr_key` dedup and
the `index.names` suffix dedup miss, so `apply_external_pr_plan` adopts the same PR
again and `_append_patch_block` writes another 10 KB copy — **every pass, forever**. 129
passes produced 3111 phantom blocks and 31 MB of garbage, and the growing file is what
pushed the pass from 2.5s (08:02) to a 120s timeout (10:00).

This is a general data-loss bug, not a mirror bug: _any_ Patch whose DESCRIPTION
contains two consecutive blank lines is invisible to every consumer of
`parse_project_file`.

**Author filtering would not catch release-please here.** All 32 release-please PRs on
`sase-org/sase` report `author.login = bbugyi200` and `is_bot = false`, because
release-please runs under a personal token. The stable signals are the head branch
(`release-please--branches--master`, 32/296 PRs matched by `release-please--*`) and the
`autorelease: pending` / `autorelease: tagged` labels. `PullRequestWire` carries
`head_ref` but not labels, so the default filter must key on the branch.
`external_mirror.pr_authors` cannot express this at all.

**Measured pass costs**, from the run records:

| chop instance                                       | duration        |
| --------------------------------------------------- | --------------- | ------------- | ------------- |
| `external_issue_mirror[actstat                      | bob-cli         | sase]`        | 0.93 – 1.13 s |
| `external_pr_mirror[actstat                         | bob-cli]`       | 0.83 – 1.03 s |
| `external_pr_mirror[sase]`, healthy (08:02 – 09:48) | 2.5 – 3.3 s     |
| `external_pr_mirror[sase]`, since 10:00             | 120 s (timeout) |

Component costs on the 33.8 MB archive: 0.13 s per full read, 0.21 s per full parse.
`sync_external_pull_requests` re-reads the index after **every** mutation
(`pr_sync.py:158`) and `apply_external_pr_plan` re-reads it again under the lock
(`importer.py:96`), and `_append_patch_block` reads the file, concatenates, and
`write_patch_atomic` reads it a third time to compare before writing — roughly 0.5 s of
pure I/O per adopted PR, 25 adoptions per pass.

**Two related gaps found while digging:**

- An adopted external Patch never tracks its PR again. `plan_external_pr_import` returns
  `ACTION_SKIP` / `already_owned` for any PR already owned by a local Patch
  (`external_pr_conversion.py:104-129`), and `filter_axe_candidate_patches` structurally
  excludes `pr_origin: external` Patches from all AXE work. So a PR adopted while open
  keeps its adoption-time `Mailed`/`Draft` status and stays in the active ProjectSpec
  even after it merges. This is the exact PR-side twin of the bug/bead gap this plan is
  asked to close.
- `sase patch sync-external` and `doctor/checks_external_pr_mirror.py` both hardcode
  `lumberjack_state_dir("checks")`, so moving the chop to another lane silently orphans
  every cursor. The issue mirror already avoids this by using the lane-independent
  `sase_subdir("external_mirror")`.

---

## Phase `spec_repair`: ProjectSpec description truncation and duplicate-block repair

Order matters inside this phase: fixing the parser first would make 3111 duplicate
blocks abruptly _visible_ as real Patches with colliding names. Land the repair path
before the parser change takes effect.

**1. Raw-text de-duplication repair.** Add a repair that does not depend on the parser,
because the records it must find are the ones the parser cannot see. Scan a
ProjectSpec's bytes for top-of-line `NAME: ` boundaries, group the raw blocks by name,
and keep exactly one block per name — the **last**, which is the most recently written
and therefore carries the freshest STATUS. Write through `write_patch_atomic` under
`patch_lock` on the active file before the archive, matching `apply_external_pr_plan`'s
ordering.

Expose it as a `sase doctor` check with a confirmed fix rather than a new `sase patch`
subcommand: this is machine-state health, it must run over every project rather than
one, and `sase doctor` already owns confirmed repairs. Register it in
`src/sase/doctor/runner.py` next to the two external-mirror checks. The check reports
per project the duplicate-name count and reclaimable bytes; the fix rewrites the files.
Read `sase/memory/cli_rules.md` with `/sase_memory_read` before naming the flag.

**2. Writer hardening.** `format_patch_block` must never emit a sequence that can
terminate its own record. Collapse any run of two or more blank lines in the supplied
description to a single blank line before indenting. This keeps old parsers, external
tooling, and any un-migrated ProjectSpec safe regardless of the parser change, and it is
the reason the parser change is a repair rather than a load-bearing dependency.

**3. Parser fix, in both implementations.** Count a line toward the two-blank-line
record terminator only when it is _truly_ empty — no leading whitespace — so an indented
in-description blank stays description content. Two genuinely blank lines still end a
record, which is what hand-written specs and the existing golden corpus rely on.

- Python: `src/sase/ace/patch/parser.py` `parse_patch_from_lines`.
- Rust: `crates/sase_core/src/parser.rs` `parse_one` and the module doc comment at lines
  5-18 that states the `end-on-two-blank-lines` contract.

Per the Rust core backend boundary rule in `CLAUDE.md`, parsing is core backend logic:
change the Rust parser in `../sase-core`, keep the Python parser in lockstep, and extend
the shared golden corpus with a Patch whose DESCRIPTION contains a blank run so the two
implementations can never drift back apart. Do not bump the wire schema — the record
shape is unchanged, only which records are produced.

**4. Regression coverage.** A round-trip test that formats a release-please-shaped body,
writes it, parses it back, and asserts the Patch survives with its `PR:`, `PR_ORIGIN:`,
and `STATUS:` intact. A test that two adoption passes over the same remote PR produce
exactly one block. A repair test over a fixture archive with known duplicates.

---

## Phase `filters`: Configurable bug and pull-request filters

**Config shape.** Follow the established `fileHookFilters` convention in
`src/sase/config/sase.schema.json` — a `filters` object whose keys are criteria, each a
list of globs where a `!`-prefixed entry is an exclusion. Do not invent a second filter
vocabulary for this feature.

```yaml
external_mirror:
  issues:
    filters:
      author_globs: []
      label_globs: []
      title_globs: []
      states: [] # open | closed; empty means both
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

Semantics, stated once and shared by both kinds: within a criterion, a record matches if
it matches any positive glob (or if there are no positive globs) and matches no
`!`-prefixed glob. A record is mirrored only if it satisfies every criterion. Matching
is case-folded, consistent with the accessors this replaces. Empty everywhere means
"mirror everything", which preserves the epic's superset promise for issues.

**Defaults.** Issues keep matching everything. Pull requests ship the four head-ref
exclusions above and nothing else, so the change in observable behavior is exactly
"release-please and release-plz PRs stop becoming Patches". Verified against the live
repo: those globs match 32 of 296 PRs, all of them `release-please--branches--master`,
and match zero human PRs. Choose head-ref over author because release-please here
authors as `bbugyi200`, and over title because titles are user-configurable.

**Implementation.** One pure, dependency-free matcher module under
`src/sase/external_mirror/` holding the criterion model and the glob evaluator,
unit-tested on its own. `issues.py` replaces `_is_excluded`/`_excluded_count` with it
and `pr_sync.py` replaces `_authored_remotes` with it. Both already count dropped
records into `unmirrored` and both already keep dropped records out of the checkpoint,
so clearing a filter later re-examines them — preserve that.

**Back-compat.** `external_mirror.exclude_labels` and `external_mirror.pr_authors` stay
readable. `exclude_labels: [x]` folds into `issues.filters.label_globs: ["!x"]`;
`pr_authors: [a]` folds into `pull_requests.filters.author_globs: ["a"]`. When a project
sets both the legacy key and its modern equivalent, the modern one wins and validation
emits a deprecation diagnostic. Keep `src/sase/external_mirror/config.py`'s two public
accessors as thin shims over the new model so existing tests keep their meaning.

**Surfaces.** Update `src/sase/config/sase.schema.json`, `src/sase/default_config.yml`
(with the four default globs and comments matching the file's existing density),
`docs/configuration.md` (the table at 2911-2918), `docs/axe.md` (316, 852),
`docs/beads.md` (580), and `docs/change_spec.md` (8). The Beads pane already renders
`N remote-only` from `external_unmirrored_counts` (`beads_rendering.py:287-304`); give
the Patches pane the same honesty signal via `_patch_list_banner.py`, since a non-empty
PR filter is now the default and a silently shorter Patch list would be
indistinguishable from a broken mirror.

---

## Phase `lane`: Dedicated external_mirror lumberjack and lane-independent state

**One new lane, not two.** Both chops are the same class of work — one bounded remote
poll per project, fanned out by `for_each: {source: projects}` and run concurrently
within a tick. The issue mirror costs ~1 s and gains nothing from isolation; splitting
it out would double the lane process, the state directory, and the doctor surface for no
measured benefit. What actually needs isolating is `pr_submitted_checks` and
`stale_running_cleanup`, which today wait behind a 120-second PR mirror.

```yaml
external_mirror:
  interval: 900
  chop_timeout: "5m"
```

Justification from the measured table: a healthy full pass is 1–3.5 s, so 900 s leaves
~250x headroom; the worst observed pass is timeout-capped at 120 s, so even an
unrepaired project leaves ~7x. A lane waits for its tick to drain, so the worst-case
cycle is `interval + chop_timeout` = 20 minutes, which is an acceptable freshness bound
for tracker mirroring and still four times fresher than the effective cadence the
overrunning `checks` lane delivers today. Drop the now-redundant per-chop
`run_every: "10m"` — the lane interval is the cadence — and keep an explicit per-chop
`timeout` on the PR mirror.

Write the lane `description` to the schema's contract (line 1 a summary of at most 100
characters, line 2 blank if a body follows), matching the voice of the five existing
lanes.

**Lane-independent state.** Move the PR mirror's `external_pr__<project>.json` cursors
and `external_pr__backoff.json` from `lumberjack_state_dir("checks")` to
`sase_subdir("external_mirror")`, the lane-independent root the issue mirror already
documents and uses (`state.py:1-11`). Migrate existing files on first read so no project
silently restarts its backfill. Then delete the hardcoded lane name from
`doctor/checks_external_pr_mirror.py:39` and `main/patch_handler.py`'s
`ensure_lumberjack_dirs_fn("checks")`, and stop threading `runtime.context.state_dir`
into `sync_external_pull_requests`.

**Budget constants.** `sase_chop_external_pr_mirror._CHOP_TIMEOUT_SECONDS = 120` and
`issues._MirrorBudget.work_seconds = 90` are both hand-copied from the old
`timeout: "2m"`, and the issue mirror's docstring says so. Derive both from one shared
constant next to the lane's configured timeout so the next timeout change cannot leave a
stale budget behind.

**Loose ends in the same sweep.** `_check_error` in `sase_chop_external_pr_mirror.py`
emits a summary dict missing the `unmirrored` key that `MirrorReport.summary_fields()`
includes, so a failed run reports a different counter set than a successful one; make
both go through `MirrorReport`. Also update `docs/axe.md`'s lane inventory and any test
that asserts the `checks` lane's chop membership.

---

## Phase `bug_status`: Bug state drives mirrored bead status

**This deliberately reverses an earlier decision.** The `sase-jd` plan chose "upstream
closes: note-only" to avoid "a notification gate per upstream close". That concern was
about gates, and auto-closing a bead raises no gate — a closed bead leaves the
`TaskTriage` queue rather than entering it. The user has asked for state to track, so
state tracks. The note stays as the audit record.

**Scope: mirrored beads only.** Sync a bead only when its `external_ref` matches the
issue — the "mirrored by this bead" relationship the detail panel already distinguishes
from a plain `bug:` ref (`beads_detail.py`). A bead that merely _references_ an issue
keeps today's note-only behavior; its status is the user's, not the tracker's.

**Transitions.**

- Upstream `open -> closed`: `project.close([bead_id], resolution=done, note=...)` with
  `author="external_issue_mirror"`, where the note records the observed transition and
  its timestamp exactly as the current note does.
- Upstream `closed -> open`: `project.open(bead_id)`, plus an attributed note. `open`
  reopens closed ancestors too, which is correct here — a mirrored task bead is
  top-level.
- Upstream absent (deleted or transferred): unchanged. Note only, link marked stale.
  There is no upstream state left to trust.

**Guards — each of these appends the note and skips the transition.**

- Bead status is `in_progress` or `claimed`: a runner owns it and closing it out from
  under an agent is destructive.
- Bead has any unclosed descendant: `close` rejects it anyway
  (`sase/memory/sase_beads.md`), and `--force` semantics are not the mirror's to invoke.
- Bead is already in the target state: no event, no commit.

Take these guards from the current tree, not from this description — re-read
`sase/memory/sase_beads.md` with `/sase_memory_read` before writing the close call.

**Mechanics.** Extend `_plan_notes` into a transition planner returning notes _and_
status changes, apply both inside the existing single `bead_store_write_lock` /
`commit_external_issue_mirror` / `publish_bead_claim` block in `_apply`, and charge
status changes against the same `max_notes` work budget and `work_deadline` so a large
backlog still converges over passes instead of one enormous commit. Advance
`upstream_states[ref]` only for transitions actually applied — the existing
`applied_note_refs` mechanism already has this shape, so a deferred transition is
retried next pass.

Add `beads_closed` and `beads_reopened` to `MirrorReport` and `summary_counters`,
surface them in the chop summary and in `sase bead sync-external`'s output table, and
make `--dry-run` list planned transitions the way it already lists planned creations.
Verify the `bug:drift` filter token and the drift badge still mean something after this
lands: drift becomes the _unreconciled_ set — guard-skipped beads and reference-only
links — which is more useful than it is today.

---

## Phase `patch_status`: Adopted external Patches track their pull request

The symmetric gap. Add a fourth action to the external-PR classifier, beside `adopt`,
`repair_origin`, and `skip`, that refreshes an owned Patch whose recorded STATUS no
longer matches the state the remote PR maps to under `_mapped_status_and_destination`.

**Guard it hard.** Emit the refresh action only when the owned Patch has
`pr_origin: external`. A Patch SASE's own tracked workflow created has a lifecycle AXE
owns; the mirror must never write its STATUS. This mirrors the `bug_status` phase's
"mirrored beads only" rule and keeps `filter_axe_candidate_patches`'s exclusion the
single source of truth about which Patches the mirror may touch.

**Where the work lands.** The classifier is core backend logic: `crates/sase_core/src/`
behind `plan_external_pr_import`, with `src/sase/core/external_pr_conversion.py` kept in
lockstep as the Python mirror and `EXTERNAL_PR_WIRE_SCHEMA_VERSION` bumped, since
`ExternalPrImportPlanWire` gains a new action value. The importer already has the
machinery to apply it: `_rewrite_patch_fields` plus `_move_patch_to_archive_locked` are
exactly what `_repair_existing_patch` uses. Report the outcome as a new `refreshed`
counter on `MirrorReport` rather than overloading `repaired`, which means "origin marker
repaired" and should keep meaning that.

**Cursor interaction.** A refresh is a mutation, so it must be charged against
`create_budget` and the deadline like any other, and it must not advance the cursor on a
partial pass. Adding this makes the daily `DAILY_REPAIR_SCAN` full pass load-bearing
rather than merely defensive, because a PR that merges long after adoption may fall
outside the incremental `OVERLAP_WINDOW`.

---

## Phase `perf`: Bounded per-pass cost for the PR mirror

With the archive repaired and release PRs filtered, the sase project's pass should
return to its healthy 2.5–3.5 s. The quadratic shape underneath it stays, though, and
`patch_status` adds a second class of mutation that pays it.

Three costs, all measured above:

1. `pr_sync.py:158` re-reads and re-parses the entire active-plus-archive index after
   every mutation, purely to see the record it just wrote.
2. `apply_external_pr_plan` re-reads the same index under the lock (`importer.py:96`)
   for its raced-ownership check.
3. `_append_patch_block` reads the destination, concatenates, and `write_patch_atomic`
   reads it again to compare before writing.

Replace the loop with a single locked batch apply: acquire the active and archive locks
once for the pass, read the index once inside them, apply every planned mutation against
an index updated **incrementally** in memory (the raced-ownership and name-suffix checks
both need only the in-memory view once the lock is held for the whole batch), and write
each destination file once at the end. Keep the per-pass creation budget and the
deadline; they are what makes a large backlog converge over passes.

Hold the lock for the batch, not for the pass's provider call — fetch first, plan
second, then take the lock. Locking across the network call would block the ACE TUI and
every other ProjectSpec writer for the duration of a `gh` round trip.

**Do not** widen the incremental fetch in this phase. `fetch_limit=200` against
`gh pr list`'s created-descending order can miss an old PR that was just updated; that
is a real gap, but it is a provider-ordering question that deserves its own task bead
rather than a change smuggled into a performance phase.

---

## Verification

Beyond each phase's unit coverage, the epic is done when, on this machine:

- `~/.sase/projects/gh_sase-org__sase/gh_sase-org__sase-archive.sase` parses to the same
  number of Patches as it has unique `NAME:` lines, and is on the order of megabytes
  rather than tens of megabytes.
- Successive `sase patch sync-external --project sase --dry-run` runs plan zero
  creations for PRs already adopted, including the release-please ones — which are now
  dropped by the default filter and counted as `unmirrored`.
- No `external_pr_mirror` run record carries `"status": "timeout"`, and the
  `external_mirror` lane's `status.json` shows `uptime_seconds / cycles_run` close
  to 900.
- Closing a tracker issue that has a mirrored bead closes that bead on the next pass,
  with an attributed note recording the transition; reopening it reopens the bead.
- Merging a PR adopted while open moves its Patch to `Submitted` in the archive on the
  next pass.

Run `just check-full` before landing the combined tree: this epic touches the parser,
the config schema, the AXE lane inventory, and `../sase-core`, all of which are in the
broadening set.
