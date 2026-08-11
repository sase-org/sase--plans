---
tier: tale
title: external_issue_mirror chop
goal:
  Every issue in an enabled project's external tracker has a corresponding open task
  bead, created and kept current by a per-project AXE chop with watermarks, a resumable
  backfill, per-pass budgets, backoff, a daily repair scan, a dry run, a manual CLI
  entry point, and a doctor check for detached tracker auth.
size: medium
proposed_by: bbugyi200.athena.sase-jd.4
bead: sase-jd.4
create_time: 2026-08-10 21:30:28
status: wip
---

- **PARENT:**
  [202608/external_artifact_ingestion.md](https://github.com/sase-org/sase--plans/blob/main/202608/external_artifact_ingestion.md)
- **BEAD:**
  [sase-jd.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-jd/sase-jd.4.md)

# Plan: `external_issue_mirror` chop

This is phase `issue_mirror` of epic `sase-jd`, whose plan is
`plans:202608/external_artifact_ingestion.md`. Read that plan's `issue_mirror` section
alongside this one; everything below either implements it or explains a deliberate
departure from it.

## Goal

Every issue in an enabled project's external tracker gets a corresponding task bead,
kept current by AXE, on every enabled project on the machine. This phase lands the
per-project builtin chop that diffs the tracker against beads on `external_ref` and
creates unsized `open` task beads, plus its watermarks, resumable backfill, per-pass
budgets, backoff, daily repair scan, dry run, manual CLI entry point, and a doctor check
for detached tracker auth.

Phase `bead_ref` (`sase-jd.1`) has landed: `Issue.external_ref`, the partial-unique
index, `normalize_external_ref`, and `find_external_ref_links` all exist and are
verified present in this tree (`src/sase/bug_links.py:97,176`,
`src/sase/bead/_db_schema.py:45`).

## Three decisions that differ from the epic plan, and why

The epic plan was written before the issue seam's actual shape was pinned down. Three of
its mechanisms cannot be built as written. Implement the alternatives below and record
the reasoning in code comments so the next reader does not "restore" the original.

### 1. There is no page cursor, so the pass lists the full inventory

`vcs_list_issues(cwd, state, limit)` (`src/sase/vcs_provider/_hookspec.py:205`) is the
whole seam: no cursor, no offset, no `since`, and **no ordering guarantee**. The GitHub
implementation is one `gh issue list --state <s> --limit <n>` call
(`sase-github/src/sase_github/plugin.py:286-317`) whose default ordering is by creation,
while the in-memory fake orders by `updated_at`. Any windowed listing would therefore
silently skip issues on at least one provider — which breaks the exact promise ("the
bead list is a superset of the issue list") that the `tabs` phase depends on.

So:

- Each pass performs **one full listing** (`state="all"`, `limit=0` → the provider's
  unbounded path). The inventory is never truncated.
- The per-pass bound moves to **local writes**: a bead-creation cap and a wall-clock
  work budget, both of which is where the expensive, contended work (store lock + git
  commit + publish) actually lives.
- The summary counter is `provider_calls`, not `pages_consumed`.
- **The ~10-minute overlap window is not implemented.** An overlap window exists to stop
  a windowed cursor from dropping records between passes; with a full listing every pass
  there is no window to overlap, and dedupe-by-`provider_id` is automatic. If a real
  cursor is added later, the overlap window comes with it.

Record this as a `PROPOSED FOLLOW-UP:` on `sase-jd.4` (see Follow-ups).

### 2. Mirror state lives in `~/.sase/external_mirror/`, not the chop state dir

`ChopScriptContext.state_dir` is `ensure_lumberjack_dirs(<lane>)`
(`src/sase/axe/chop_runner_context.py:39`) — the **lumberjack's** directory, shared by
every chop in the lane, not a per-instance directory. `sase bead sync-external` must
read and write the same cursors as the chop, and it does not know which lane the chop is
configured in.

State therefore lives at a stable, lane-independent path:

```text
~/.sase/external_mirror/issues/<project-key>.json     # per-project cursor + backoff
~/.sase/external_mirror/tracker_auth.json             # doctor evidence (all projects)
```

resolved through `sase_subdir("external_mirror")` (`src/sase/core/paths.py:111`), which
tests already redirect via `SASE_HOME`.

### 3. `size:none` is not a filter token today

The epic plan justifies NULL size by saying it is "mechanically visible via
`size:none`". It is not: `size:` accepts only `PHASE_SIZE_VALUES`
(`src/sase/bead/filter_query.py:350`). Mirrored beads still create with size unset (that
part of the decision stands and is right — a chop cannot honestly estimate), and are
discoverable with `sase bead list -T task -s open`. Adding the `size:none` token is
filter-grammar work owned by no phase in this epic; record it as a follow-up rather than
smuggling it in here.

## Ground rules

- **Do not edit SASE memory files.** `sase/memory/*.md`, `AGENTS.md`, `CLAUDE.md`,
  `GEMINI.md`, `OPENCODE.md`, `QWEN.md` are off-limits.
- Run `just install` first (ephemeral workspace), then `just check`; finish with
  `just check-full`.
- No test may touch the network. Every provider interaction in tests goes through
  `sase.vcs_provider.testing.FakeIssueProvider`.
- This phase touches no ACE/TUI surface and regenerates no PNG goldens.
- `sase-jd.5` (`pr_mirror`) is **in flight in parallel** and its plan says it reuses "a
  shared library". This phase creates `src/sase/external_mirror/` with the
  object-type-agnostic pieces (`state.py`, `report.py`, `auth.py`); `sase-jd.5` adds
  `pull_requests.py` beside `issues.py`. Expect the land agent to resolve a small
  overlap in `default_config.yml` and the `external_mirror` config block. Do **not**
  build speculative PR-shaped abstractions here — symvision would flag them as unused
  public symbols (see `sase/memory/symvision.md`), and unused-but-whitelisted API is
  worse than a five-line duplication the sibling phase can hoist.

## Work

### 1. `src/sase/external_mirror/` — the shared reconciler package

`__init__.py` holds only a module docstring; consumers import submodules directly so
every public symbol has a real non-test consumer (chop script, CLI handler, or doctor
check).

**`state.py`** — per-project cursor and backoff, one JSON document per project:

```json
{
  "schema_version": 1,
  "project": "gh_sase-org__sase",
  "watermark_updated_at": "2026-08-10T18:00:00Z",
  "watermark_provider_ids": ["I_kwDOabc123"],
  "backfill_complete": true,
  "last_full_scan_at": "2026-08-10T06:00:00Z",
  "last_success_at": "2026-08-10T18:02:11Z",
  "upstream_states": { "bug:gh_sase-org__sase#42": "open" },
  "failures": 0,
  "next_attempt_at": null
}
```

- `read_mirror_state(path)` / `write_mirror_state(path, state)` — atomic
  write-temp-then-rename, and **tolerant reads**: a corrupt, truncated, or
  wrong-schema-version file returns a fresh empty state rather than raising, following
  `_read_backoff_state` in `src/sase/scripts/sase_chop_bead_store_refresh.py:70`.
- `next_backoff(previous_failures, *, now, run_every_seconds, max_seconds)` — the
  exponential shape from `_next_backoff_entry` (`sase_chop_bead_store_refresh.py:157`),
  capped at one hour.
- `upstream_states` maps `external_ref` → last observed remote state
  (`"open" | "closed" | "absent"`). It is the record that makes transition notes
  idempotent, and it is pruned to refs still covered by a local bead, so it stays
  bounded by the mirrored-bead count.
- Guard writes with
  `best_effort_test_state_write_allowed(path, category="external-mirror-state")`
  (`src/sase/core/state_write_guard.py`), matching `_atomic_json_write` in
  `src/sase/axe/chop_script_context.py`.

**`report.py`** — the structured pass result shared by the chop, the CLI, and tests:

```python
@dataclass(frozen=True)
class MirrorReport:
    project: str            # stable ProjectSpec key
    display_name: str       # user-facing name — render this, never the key
    issues_seen: int = 0
    beads_created: int = 0
    notes_appended: int = 0
    conflicts: int = 0      # already covered under the lock; a real duplicate was avoided
    unmirrored: int = 0     # skipped by external_mirror.exclude_labels
    deferred: int = 0       # planned creations the pass budget could not apply
    provider_calls: int = 0
    checkpoint_advanced: bool = False
    created_refs: tuple[str, ...] = ()   # dry-run detail; also the CLI's output
    degraded: str = ""      # non-empty reason when the pass was degraded
```

plus `summary_counters(report) -> dict[str, int]` for `runtime.emit_summary`
(`checkpoint_advanced` marshals to `0`/`1`; `parse_summary` keeps only int-valued
tokens, `src/sase/chops/sdk.py:159`).

**`auth.py`** — the detached-environment evidence file:

- `record_tracker_probe(project, *, outcome, source, detail="")` where `outcome` is
  `"ok" | "auth_error" | "rate_limited" | "unavailable" | "unsupported"` and `source` is
  `"chop" | "cli"`. Called after **every** provider listing attempt, success or failure.
- `read_tracker_probes() -> dict[str, TrackerProbe]`, tolerant of a missing or corrupt
  file.
- Classify from the raised exception's rendered text using the same markers the GitHub
  plugin already uses (`GitHubAuthenticationError` /`GitHubRateLimitError` surface as
  `authentication required` / `rate limit`); the plugin's exception classes are not
  importable from this repo, so match on the message, and default to `unavailable`.

**`config.py`** — `excluded_issue_labels() -> frozenset[str]`, reading
`external_mirror.exclude_labels` through `load_merged_config()` with the defensive
`try/except → default` shape of `bead_refresh_mode`
(`src/sase/sdd/_integration_marker.py:16`). Case-folded comparison; empty by default.

**`issues.py`** — the reconciler, and the single code path the chop and the CLI both
run:

```python
def run_issue_mirror_for_project(
    *,
    project_key: str,
    display_name: str,
    workspace_dir: str,
    dry_run: bool = False,
    full: bool = False,
    source: Literal["chop", "cli"] = "chop",
    budget: MirrorBudget = DEFAULT_BUDGET,
    now: datetime | None = None,
    provider: object | None = None,   # injected in tests; default get_vcs_provider(cwd)
    log: Callable[[str], None] | None = None,
) -> MirrorReport: ...
```

Ordered stages:

1. **Backoff gate.** If `next_attempt_at` is in the future and not `full`, return a
   degraded-free report with `degraded="backoff"`; no provider call.
2. **Capability gate.** `supports_issue_listing(workspace_dir)`
   (`src/sase/vcs_provider/_registry.py:215`) — the split probe from `pr_seam`, not
   `supports_issues()`, because listing is all a synchronizer needs. On a miss, record
   `record_tracker_probe(..., outcome="unsupported")` and return
   `degraded="unsupported_provider"`.
3. **List.** One `provider.list_issues(workspace_dir, state="all", limit=0)`.
   - Success → `record_tracker_probe(outcome="ok")`.
   - Exception → classify, `record_tracker_probe`, persist the incremented backoff
     entry, return `degraded=<classification>`. **The watermark does not move.**
4. **Filter.** Drop issues carrying any `exclude_labels` label; count them in
   `unmirrored`.
5. **Short-circuit.** If `backfill_complete`, not `full`, the remote's max `updated_at`
   is `<=` the stored watermark, and the remote `provider_id` set is unchanged, return
   early with `checkpoint_advanced=True` and no store lock. This is the steady state and
   must cost zero bead-store I/O.
6. **Plan.** Read local beads for the project (`get_project_beads_dirs_for_project` →
   `sase.core.bead_read_facade.list_issues`, read-only, no lock) and build the identity
   index: for each bead, `normalize_external_ref(bead.external_ref, project=key)` plus
   every `bug:`-prefixed entry in `bead.refs` through the same normalizer. Compare
   against `normalize_external_ref(issue.number, project=key)`. Produce the create list,
   the transition-note list, and the disappearance-note list.
7. **Apply**, only when the plan is non-empty and `dry_run` is false:

   ```python
   with bead_store_write_lock(beads_dir) as already_locked:
       with open_bead_project_for_beads_dir(beads_dir) as project:
           index = build_identity_index(project.list_issues())   # REBUILT under the lock
           for candidate in planned_creates[: budget.max_creations]:
               if candidate.ref in index:      # someone else got there first
                   report.conflicts += 1
                   continue
               project.create(...)
               index.add(candidate.ref)
           for note in planned_notes[: budget.max_notes]:
               project.append_note(...)        # BeadProject.append_note, under the lock
       if changed:
           committed = commit_external_issue_mirror(
               beads_dir, project_key, already_locked=already_locked
           )
   if committed:
       publish_bead_claim(beads_dir, "external_issue_mirror", project_key)
   ```

   The list-then-create-under-one-lock ordering is mandatory: the partial-unique index
   is the backstop, not the plan. Follow `_reconcile_project_claims`
   (`src/sase/scripts/sase_chop_bead_claim_checks.py:67`) for the lock/commit/publish
   shape.

8. **Advance.** Write the watermark, `upstream_states`, and (for a full scan)
   `last_full_scan_at` **only** when the pass listed successfully and applied every
   planned mutation (`deferred == 0` and no per-item errors). Clear the backoff entry on
   success. A dry run never writes state.

**What a created bead gets:**

- `issue_type=TASK`, status `open` (the `create` default — never `ready`, which would
  raise one `TaskTriage` gate per incoming issue and flood the inbox on pass one),
  `size=None`. The `--size` requirement is a CLI-level gate in
  `src/sase/bead/cli_crud.py:123`, not a core invariant; `_db_schema.py:37-42` allows
  NULL. Add a test that pins this.
- `title` = the issue title, verbatim; fall back to `Issue #<n>` when blank.
- `description` = the issue body, then a provenance block:

  ```text
  ---
  Mirrored from <issue url> by SASE's `external_issue_mirror` chop.
  Upstream state when mirrored: open.
  ```

- `external_ref` = `normalize_external_ref(issue.number, project=<stable key>)` →
  `bug:gh_sase-org__sase#42`. Identity and storage use the key.
- `refs` gains `bug:<display-name>#<n>` — the user-facing spelling documented in
  `docs/getting_started.md:191`, per "Show Project Names, Never ProjectSpec Keys". Both
  spellings normalize to the same identity because `_normalize_external_project`
  resolves display names and aliases through `resolve_project_alias_ref`
  (`src/sase/bug_links.py:145`). **Add a test that pins exactly this**, because it is
  the one place the two spellings have to agree.
- `created_by=None` — core attributes the bead to the store owner. A synthetic
  `"external_issue_mirror"` creator would render as a dead agent-page link through
  `resolve_bead_creator_url`. The provenance line carries the truth.

**Upstream state changes** never auto-close, never delete. Append one attributed note
through
`sase.core.bead_mutation_facade.append_note(..., author="external_issue_mirror")` (or
`BeadProject.append_note`, `src/sase/bead/_project_mutations.py:157`, when already
inside the store lock) and update `upstream_states` so it happens exactly once per
transition:

```text
Upstream issue sase#391 changed state: open -> closed (observed 2026-08-10T18:02Z by
external_issue_mirror). This bead's status is unchanged; reconcile deliberately.
```

**Disappearance (deleted or transferred)** marks the link stale with one note and
`upstream_states[ref] = "absent"`:

```text
Upstream issue sase#391 is no longer present in the tracker listing (deleted or
transferred), observed 2026-08-10T18:02Z by external_issue_mirror. The external link is
stale.
```

**Critically: run the disappearance check only after a successful full listing.** A
degraded run must never be read as "every issue vanished." Assert this with a test.

**Budgets** (`MirrorBudget`, one default shared by chop and CLI so both converge the
same way):

```python
DEFAULT_BUDGET = MirrorBudget(
    work_seconds=90.0,        # 0.75 × the chop's 2m timeout
    lock_wait_seconds=30.0,
    max_creations=25,         # "at most N per pass", per managed_tmp_reap
    max_notes=50,
)
```

Keep a comment tying `work_seconds` to `timeout: "2m"` in `default_config.yml`, the way
`sase_chop_bead_store_refresh.py:29-35` does.

### 2. `src/sase/bead/_sync_git.py` + `sync.py` — the mirror commit

Add `commit_external_issue_mirror(beads_dir, project, *, already_locked=False)`
delegating to `_commit_bead_state` with
`message=f"chore(beads): mirror external issues for {project}"` and
`op_prefix="bead.external_issue_mirror"`, following `commit_bead_claim_reconciliation`
(`_sync_git.py:142`). Re-export it from `sase/bead/sync.py` alongside its siblings.

### 3. `src/sase/scripts/sase_chop_external_issue_mirror.py` — the chop

Thin: resolve `runtime.context.target` (`project`, `name`, `workspace_dir`) the way
`sase_chop_refresh_docs.py:44` does, honor `runtime.context.dry_run`, call
`run_issue_mirror_for_project(source="chop")`, and emit the summary:

```text
external_issue_mirror: issues_seen=41 beads_created=3 notes_appended=1 conflicts=0
  unmirrored=0 deferred=0 provider_calls=1 checkpoint_advanced=1
```

- Missing/blank `target.workspace_dir` → `check_error` with reason
  `missing_target_workspace`, mirroring `refresh_docs`'s `_check_error`.
- A degraded report sets `reason=<degraded>`; auth and rate-limit degradations set
  `result.status = "check_error"` so `sase axe chop doctor` surfaces them. `backoff`,
  `dry_run`, `no_changes`, and `unsupported_provider` stay `no_op`.
- Register with `@builtin_chop("external_issue_mirror")` — the base name, not the
  instance id; `--exclude-decorator builtin_chop` in the `Justfile` keeps symvision off
  the handler.
- Add
  `sase_chop_external_issue_mirror = "sase.scripts.sase_chop_external_issue_mirror:main"`
  to `pyproject.toml` (keep the block sorted; it goes between
  `sase_chop_epic_launch_flush` and `sase_chop_error_digest`).

### 4. `src/sase/default_config.yml` — registration and config

In the **`checks`** lane (`interval: 300`), whose description already says it is "for
checks that can tolerate delay or may touch remote PR state", keeping the entries
sorted:

```yaml
- name: external_issue_mirror
  script: sase_chop_external_issue_mirror
  run_every: "10m"
  timeout: "2m"
  for_each:
    source: projects
    vcs: [git, gh]
  description: |-
    Mirror every external tracker issue into an open task bead

    Expands to one instance per enabled project and diffs that project's tracker
    against local beads on external_ref, creating unsized open task beads for
    uncovered issues. Upstream state changes append an attributed note and never
    close a bead. A per-pass creation cap, bounded store-lock waits, and
    persistent exponential backoff keep one large backlog or one unreachable
    tracker from stalling the pass.
```

Add the config block, sorted next to its neighbors:

```yaml
external_mirror:
  # Tracker labels whose issues are never mirrored into beads. Empty (the
  # default) keeps the bead list a strict superset of the issue list, which is
  # what lets the Artifacts tab drop a separate issue inventory. Every entry
  # here removes issues from that superset.
  exclude_labels: []
```

and the matching `external_mirror` object in `src/sase/config/sase.schema.json`
(`additionalProperties: false`, `exclude_labels` as an array of non-empty strings,
default `[]`) — placed in the alphabetically sorted `properties` map. `sase-jd.5` adds
`pr_authors` to the same object.

`vcs: [git, gh]` filters on VCS **kind**, not issue capability; the runtime
`supports_issue_listing()` gate in stage 2 is what handles capability, and it declines
with a recorded reason.

**`for_each` has no production usage today** — verified: no `for_each` appears anywhere
in `default_config.yml`, only in `docs/configuration.md:2104`'s example. Exercise
`sase axe chop list -a` after landing and confirm instance identity
(`external_issue_mirror[<project>]`), independent scheduling, and teardown when a
project is disabled. If any of it misbehaves, record a `PROPOSED FOLLOW-UP:` note — do
not try to fix the target-expansion engine inside this phase.

### 5. `sase bead sync-external` — the manual entry point

Read `sase/memory/cli_rules.md` with `/sase_memory_read` before writing it (already done
for this plan; re-read if you deviate). Requirements it imposes: alphabetically sorted
subcommands and options, a short alias for every long option, excellent `-h`, and
colored output.

- `src/sase/main/parser_bead_store.py`: `register_bead_sync_external_parser`, with
  `-n/--dry-run`, `-f/--full`, `-p/--project <name>`. A rich `description` and an
  `epilog` with two examples.
- `src/sase/main/parser_bead.py`: register it after `register_bead_sync_parser` (`sync`
  < `sync-external`).
- `src/sase/bead/cli_sync_external.py`: `handle_bead_sync_external`. With no
  `--project`, iterate every enabled non-home project from
  `list_project_records(sase_projects_dir(), "enabled", include_home=False, projects_only=True)`;
  with `--project`, resolve the value against `project_name`, `effective_project_name`,
  and aliases, the way `_resolve_bug_scope` does
  (`src/sase/ace/tui/artifacts_bugs.py:89`). Call
  `run_issue_mirror_for_project(source="cli", ...)` per project.
- Output: one Rich table row per project — display name, issues seen, beads created,
  notes, deferred, and a degraded reason when present — then the created refs under
  `--dry-run`. Exit non-zero if every attempted project was degraded; a partial failure
  warns and still reports the successes. Render `PROJECT_NAME:`, never the key.
- Export the handler from `src/sase/bead/cli.py` and wire `"sync-external"` into
  `_BEAD_HANDLERS` plus the usage string in `src/sase/main/entry.py:105-137`.

### 6. `src/sase/doctor/checks_external_mirror.py` — detached tracker auth

Register `external_mirror_check_specs(context)` in
`src/sase/doctor/runner.py::build_doctor_registry`, immediately after
`axe_check_specs(context)`. One non-deep check, `id="axe.external_mirror"`,
`group="axe"`, title "External mirror tracker auth".

It reads `auth.read_tracker_probes()`, because **doctor cannot reproduce the
lumberjack's detached environment** — the daemon records what it actually saw, and
doctor reports that evidence. State this in the check's `details`; it is the whole point
of the check. Only `source == "chop"` probes answer the detached question. A
`source == "cli"` probe is reported as supporting evidence and explicitly labelled as
not answering it, because a working interactive `gh` proves nothing about the daemon
(this is exactly the failure the Bugs pane hid).

| Condition                                                    | Status  |
| ------------------------------------------------------------ | ------- |
| No enabled `external_issue_mirror` instance configured       | `SKIP`  |
| Configured, but no chop-sourced probe newer than 30m (3×10m) | `WARN`  |
| Any chop probe `rate_limited` or `unavailable`               | `WARN`  |
| Any chop probe `auth_error`                                  | `ERROR` |
| Every configured project has a fresh `ok` chop probe         | `OK`    |

`next_steps` name real commands: `gh auth login`, `sase bead sync-external --dry-run`,
`sase axe chop list -a`, `sase doctor -C axe.chops`.

### 7. Docs

- `docs/axe.md`: add `external_issue_mirror` to the **checks (5-minute interval)** table
  and a subsection covering the 10m cadence, per-project `for_each` fan-out, the full
  listing (and why there is no cursor), the creation cap, backoff, the daily repair
  scan, degraded runs, and `sase doctor -C axe.external_mirror`.
- `docs/beads.md`: an "External issue mirroring" concept subsection near "Discovered
  Follow-Up Capture and Triage", and a `### sase bead sync-external` entry in the
  alphabetically ordered `## CLI Commands` section (between `sase bead stats` and
  `sase bead work` — check the actual local ordering and match it).
- `docs/configuration.md`: document `external_mirror.exclude_labels` in the settings
  table, and note that a non-empty value stops the bead list from being a superset of
  the issue list.

## Verification

New tests (network-free; every provider call goes through `FakeIssueProvider`, and bead
stores are real ones built with `BeadProject.init(tmp_path)` with
`canonical_beads_dir_for_project` monkeypatched, following
`tests/test_axe_chop_wait_checks_beads.py:27-44`):

**`tests/test_external_mirror_state.py`**

- Round-trip; corrupt/truncated/wrong-version file yields fresh state, never raises.
- Backoff grows exponentially and caps at one hour; a success clears the entry.
- `upstream_states` prunes refs with no surviving local bead.

**`tests/test_external_mirror_issues.py`** (the reconciler, called directly)

- An uncovered issue produces one bead with status `open`, `size is None`, the expected
  `external_ref`, and a `bug:<display-name>#<n>` ref; a second pass creates nothing.
- **Cross-project non-collision**: issue `#42` in `sase` and issue `#42` in
  `sase-github` produce two distinct beads and neither is ever treated as covering the
  other.
- **Ref/`external_ref` spelling agreement**: a bead carrying only `refs=["bug:sase#42"]`
  (display spelling, no `external_ref`) is recognized as covering
  `bug:gh_sase-org__sase#42` and is not duplicated.
- A pre-existing bead with a matching `external_ref` created between plan and apply is
  detected by the under-lock rebuild and counted as a `conflict`, not overwritten.
- Creation budget: 40 uncovered issues create exactly 25, report `deferred=15`, and **do
  not advance the watermark**; the next pass creates the rest and then advances.
- Upstream close appends exactly one note across three passes and never changes bead
  status.
- A disappeared issue appends one stale-link note, but **a provider error appends none**
  and leaves `upstream_states` untouched.
- `exclude_labels` skips the labelled issue and counts it in `unmirrored`.
- `dry_run=True` writes no bead, no commit, and no state file, and still reports the
  exact `created_refs`.
- A provider raising an auth-shaped error records an `auth_error` probe, sets backoff,
  and leaves the watermark where it was.

**`tests/test_axe_chop_external_issue_mirror.py`**

- End-to-end through `run_builtin_chop("external_issue_mirror", ["--context", ...])`
  with a written `ChopScriptContext` carrying a `target`, asserting the result
  document's `status`, `reason`, and counters (pattern:
  `tests/test_axe_chop_refresh_docs.py:28-55`).
- Missing `target.workspace_dir` → `check_error` / `missing_target_workspace`.
- `context.dry_run` → `no_op` / `dry_run`, and nothing on disk changed.
- A provider without `vcs_list_issues` (`FakeIssueProvider(capabilities=[])`) → `no_op`
  / `unsupported_provider`.

**`tests/test_bead_sync_external_cli.py`**

- `--dry-run` prints the planned creations and mutates nothing.
- `--project` narrows to one project and accepts an alias.
- An unknown `--project` value exits non-zero with an actionable message.
- Output renders `PROJECT_NAME:`, never the ProjectSpec key.

**`tests/doctor/test_checks_external_mirror.py`**

- `SKIP` when unconfigured; `WARN` on a stale probe; `WARN` on rate limit; `ERROR` on
  auth failure; `OK` when fresh; and a CLI-sourced probe alone does **not** produce
  `OK`.
- Add `"axe.external_mirror"` to the id set in
  `tests/main/test_doctor_command.py:200-212`.

Then:

1. `just install`
2. `just check`
3. `just check-full`
4. Manual, against real projects — this is the part that decides whether `tabs`
   (`sase-jd.8`) may ever proceed:
   - `sase axe chop list -a` → confirm one `external_issue_mirror[<project>]` instance
     per enabled project.
   - `sase bead sync-external --dry-run` → confirm the planned creations look like the
     real tracker inventory and that a disabled project produces no instance.
   - `sase doctor -C axe.external_mirror`.

## Risks

1. **A large first backlog.** Bounded by the 25-per-pass creation cap plus the
   non-advancing watermark, so a 500-issue repo converges over ~20 passes (~3.5h at 10m)
   instead of one enormous commit. `sase bead sync-external` is the manual accelerator.
2. **Bead-store write contention.** The mirror writes to a store live agents also write.
   Bounded lock waits, a whole-pass work budget, backoff written _before_ the attempt (a
   timeout SIGKILL must not erase it), and index-rebuild-under-lock.
3. **An unbounded listing against a huge repo could exceed the 2m timeout.** The pass is
   then SIGKILLed with the backoff already persisted, so it retreats instead of
   re-attacking. It converges to a degraded run, never to data loss — but it is the
   strongest argument for the cursor follow-up.
4. **Cross-machine duplicates.** Two machines reconciling stale sidecar copies can both
   import one issue. The partial-unique index makes the local half enforceable; the
   cross-machine half is bounded by the existing publication/integration path. State
   this limitation plainly in `docs/axe.md` rather than implying it is solved.
5. **`for_each`'s first production use.** Verify, do not assume; report rather than
   repair.

## Follow-ups to record on `sase-jd.4`

After the work lands, append these with
`sase bead note sase-jd.4 'PROPOSED FOLLOW-UP: ...'` — do not create beads:

- Issue-listing seam has no cursor or ordering guarantee — every mirror pass must list
  the full inventory. Add `since`/cursor/ordering to `vcs_list_issues` (and its GitHub
  implementation) so the mirror can page and bound work by pages consumed.
- `size:none` is not a valid `sase bead list` filter value, so sizeless mirrored beads
  are not mechanically visible as "needs triage" the way the epic plan assumes. Add the
  token to `filter_query.py` and the Beads pane filter.
- Anything `sase axe chop list -a` reveals about `for_each` instance identity,
  per-instance scheduling, or teardown-on-disable that contradicts
  `docs/configuration.md:2112`.
