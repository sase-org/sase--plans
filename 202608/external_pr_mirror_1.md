---
tier: tale
title: external_pr_mirror chop and the two-file Patch importer
goal:
  Every pull request on an enabled project's remote that SASE's tracked workflow did not
  create has a local Patch, adopted by a per-project AXE chop through a deterministic
  classifier, with merged and closed PRs written straight into the archive ProjectSpec
  file rather than transitioned into it.
size: medium
proposed_by: bbugyi200.athena.sase-jd.5
bead: sase-jd.5
create_time: 2026-08-11 06:16:31
status: wip
---

- **PARENT:**
  [202608/external_artifact_ingestion.md](https://github.com/sase-org/sase--plans/blob/main/202608/external_artifact_ingestion.md)
- **BEAD:**
  [sase-jd.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-jd/sase-jd.5.md)

# Plan: `external_pr_mirror` chop and the two-file Patch importer

Phase `pr_mirror` of epic `sase-jd` (bead `sase-jd.5`). Epic design:
`@plan:202608/external_artifact_ingestion.md`, section "Phase `pr_mirror`:
external_pr_mirror chop and the two-file importer".

## Goal

Every PR on an enabled project's remote that was **not** created by SASE's tracked PR
workflow gets a local Patch, created by a per-project AXE chop, with merged and closed
PRs written straight into the archive ProjectSpec file rather than transitioned into it.

## What already landed (verified in this tree)

Both dependency phases are closed and present, so this phase builds, not re-lays, the
foundation:

- `pr_seam` (`sase-jd.2`): `PullRequestWire` (`src/sase/vcs_provider/_types.py:37`),
  `vcs_list_pull_requests` (`_hookspec.py:241`, `_base.py:418`,
  `_plugin_manager.py:447`), `supports_pull_requests()` (`_registry.py:244`), the split
  issue probes, the in-memory fake, and the `sase-github` implementation over
  `gh pr list --state all` (`plugin.py:409`, fields include `id`, `mergedAt`,
  `closedAt`, `isDraft`).
- `pr_origin` (`sase-jd.3`) and `patch_pr_ui` (`sase-jd.7`): `Patch.pr_origin` +
  `normalize_pr_origin` (`src/sase/ace/patch/models/patch.py:16-33`), `PR_ORIGIN:` in
  `PATCH_SECTION_ORDER`, the `pr_origin=` parameter on `add_patch_to_project_file`
  (`patch_operations.py:252`), the structural AXE exclusion
  (`src/sase/axe/patch_filtering.py:8`, applied at `chop_runner_context.py:49`,
  `lumberjack.py:169`, and `check_cycles.py:87`, which is the path `pr_submitted_checks`
  uses), `sase patch set-origin`, and the `origin:` query property.

So this phase adds **only** the classifier, the importer, the sync engine, the chop, its
manual CLI twin, a doctor check, and docs.

## Concurrency with `sase-jd.4`

`sase-jd.4` (`issue_mirror`) is **in progress in another workspace right now** and the
epic design asks both chops to share "a library [that] handles cursor I/O, overlap
windows, URL normalization, and structured reports". Neither phase can see the other's
tree, so the shared library is written here **generically** (not PR-specific), under the
name the other phase is most likely to reach for, so the land agent resolves one honest
merge instead of reconciling two divergent private copies.

Expected overlap, all small and mergeable: one line in `pyproject.toml`, one chop entry
in `src/sase/default_config.yml`, one import line in `src/sase/doctor/runner.py`, and
`src/sase/external_mirror/`. Do **not** rename or restructure anything jd.4 owns
(`sase bead sync-external`, bead-side doctor checks). If a genuinely shared helper ends
up duplicated after both land, that is the land agent's merge, and this plan records a
`PROPOSED FOLLOW-UP` for it rather than guessing.

## Ground rules

- `just install` **first** (ephemeral workspace), then `just check`; `just check-full`
  before finishing.
- Do **not** edit `sase/memory/*.md`, `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`,
  `OPENCODE.md`, `QWEN.md`. A plan file is not user permission.
- Use `/sase_repo` to reach `sase-core`. Never clone or web-fetch it.
- Read `sase/memory/cli_rules.md` with `/sase_memory_read` before touching
  `parser_patch.py` (already read while authoring: sort listed subcommands/options
  alphabetically, give every public long option a short alias, make `-h` excellent).
- No test may require network. Every provider call in tests goes through the
  `vcs_provider/testing.py` fake.
- Record discovered follow-up work with
  `sase bead note sase-jd.5 'PROPOSED FOLLOW-UP: ...'`. Do not create beads.

---

## Step 1 — `sase-core`: the deterministic reconciliation classifier

`CLAUDE.md`'s Rust core boundary and the epic's ground rules both put the deterministic
reconciliation classifier in `sase-core`; the epic's sibling phases already did exactly
this (`730a78f feat(beads): add external ref identity field`,
`d0eeb48 feat: add Patch PR origin to core wire`,
`e40aa41 feat(query): add origin property`). Provider calls and filesystem orchestration
stay in Python.

Open the checkout with:

```bash
sase repo open sase-core -r "Add the external-PR reconciliation classifier for sase-jd.5"
```

### New module `crates/sase_core/src/external_pr/`

`url.rs` — canonical PR URL identity:

```rust
pub struct CanonicalPullRequestUrl { pub host: String, pub owner: String,
                                     pub repo: String, pub number: u64, pub key: String }
pub fn canonical_pull_request_url(raw: &str) -> Option<CanonicalPullRequestUrl>;
```

Normalization rules, each with a test: lowercase scheme and host; drop `www.`; drop
userinfo, query, and fragment; drop a trailing `/`; drop a `.git` suffix on the repo
segment; accept `/<owner>/<repo>/pull/<n>`, `/<owner>/<repo>/pulls/<n>`, and
`/<owner>/<repo>/-/merge_requests/<n>`; compare `owner`/`repo` case-insensitively (fold
to lowercase in `key`) because GitHub treats them so; tolerate a missing scheme by
assuming `https`. `key` is `"<host>/<owner>/<repo>#<number>"` and is the only value
callers compare. Return `None` for anything unparseable — never a partial match.

`wire.rs` — serde types mirroring the Python wire (`schema_version` on the request):

```rust
RemotePullRequestWire { number, url, provider_id, title, body, state, is_draft,
                        author, head_ref, base_ref, created_at, updated_at,
                        closed_at, merged_at }
LocalPatchWire        { name, pr_url, pr_origin, status, archived, reserved }
ExternalPrImportRequestWire { schema_version, remote, local_patches }
ExternalPrImportPlanWire    { action, reason, patch_name, name_base, pr_origin,
                              status, destination, description, canonical_pr_url }
```

`classify.rs` — `plan_external_pr_import(request) -> ExternalPrImportPlanWire`, pure, no
I/O, evaluated in exactly this order:

1. **Owned URL.** Any local Patch whose `pr_url` canonicalizes to the same `key` →
   `action="skip"`, `reason="already_owned"`, `patch_name=<that Patch>`,
   `pr_origin=<that Patch's existing origin, preserved verbatim>`. One exception, and
   only this one: if that Patch is `reserved` **or** has an empty `pr_url` and the body
   marker names it, fall through to rule 2 so the crash window is repaired.
2. **Valid `SASE_PATCH` marker.** Extract the trailing footer tags from `remote.body`
   with the existing `parse_commit_footer` (`crates/sase_core/src/commit_footer.rs`,
   re-exported at `lib.rs:353`) and read the canonical `PATCH` key — the footer layer
   already strips the rendered `SASE_` prefix, so both `SASE_PATCH=` and legacy `PATCH=`
   resolve. A non-blank value that is a syntactically valid Patch name:
   - names an existing local Patch (including a bare `Reserved` stub) →
     `action="repair_origin"`, `reason="marker_repair"`, `pr_origin="sase"`,
     `patch_name=<marker>`, plus the mapped `status`/`destination` so the host can
     complete a reservation.
   - names no local Patch at all (local completion never happened) → `action="adopt"`,
     `reason="marker_orphan"`, `pr_origin="sase"`, `patch_name=<marker>`. Adopting under
     the marker's own name is what makes the next pass idempotent instead of
     duplicating.
3. **No marker, no local Patch** → `action="adopt"`, `reason="unmarked"`,
   `pr_origin="external"`, `name_base` derived from the PR title (slug), falling back to
   `head_ref`, falling back to `pr-<number>`.
4. **Ambiguous evidence** → `action="adopt"`, `reason="ambiguous_marker"`,
   `pr_origin="unknown"`. This is reached when a marker key is present but its value is
   blank or not a valid Patch name. `unknown` produces a `sase patch set-origin`
   worklist; it never guesses `sase` or `external`.

Status mapping, also in `classify.rs`, is a total function of
`(state, is_draft, merged_at)`:

| Remote PR             | `status`    | `destination` |
| --------------------- | ----------- | ------------- |
| open + `is_draft`     | `Draft`     | `active`      |
| open + not draft      | `Mailed`    | `active`      |
| `merged_at` non-empty | `Submitted` | `archive`     |
| closed, not merged    | `Archived`  | `archive`     |

`merged_at` is checked **before** `closed_at`, because `gh` reports both on a merged PR.
`destination` is derived from `ARCHIVE_STATUSES` semantics, not hardcoded per row.

`description` is `title`, a blank line, then `body` with its trailing footer-tag block
stripped via the existing `commit_footer` helper — never the raw body, so the
`SASE_PATCH` marker is not copied into the local record and re-read as evidence later.

`name_base` slugging: lowercase, non-alphanumerics to `_`, collapse runs, trim `_`, cap
at 48 chars, fall back through `head_ref` then `pr-<number>` when empty. The `_<N>`
uniqueness suffix is **not** applied here — that needs the locked file view and belongs
to the Python importer (Step 3).

`tests.rs` — table-driven coverage of every rule and every URL normalization case,
including the host-form and trailing-slash variants the epic's verification list names.

### Bindings

In `crates/sase_core_py/src/lib.rs`, following the `plan_status_transition` shape:

- `plan_external_pr_import(request: dict) -> dict`
- `canonical_pull_request_url(url: str) -> dict | None`

Register both in the module init and add the module-doc bullets at the top of the file
alongside the existing entries. Add `pub mod external_pr;` and the `pub use` block to
`crates/sase_core/src/lib.rs`.

Do **not** edit versions in `Cargo.toml` — release-plz owns them
(`sase-core/AGENTS.md`). Do **not** bump the `sase-core-rs>=0.24.0,<0.25.0` floor in
this repo's `pyproject.toml`; dev installs build the extension from the linked checkout
and the release-branch reconciler ratchets the published window. Re-run `just install`
in this repo after every sase-core edit so the rebuilt extension is what the tests
import.

The finalizer's dirty-linked-repo check (`docs/commit_workflows.md:64-67`) is what
drives committing the sase-core side; do not hand-roll a separate commit flow.

---

## Step 2 — Python wire, facade, and golden reference

Mirror the `status_wire` / `status_facade` / `status_wire_conversion` trio exactly.

- `src/sase/core/external_pr_wire.py`: frozen dataclasses `RemotePullRequestWire`,
  `LocalPatchWire`, `ExternalPrImportRequestWire`, `ExternalPrImportPlanWire`;
  `EXTERNAL_PR_WIRE_SCHEMA_VERSION = 1`; `external_pr_request_to_json_dict()` and
  `external_pr_plan_from_dict()`. Action, origin, status, and destination literals live
  here as module constants so callers never spell them inline.
- `src/sase/core/external_pr_facade.py`: `plan_external_pr_import(request)` and
  `canonical_pull_request_url(url)`, both via `require_rust_binding`
  (`src/sase/core/rust.py:55`), marshalling through the wire helpers so callers never
  see dicts.
- `src/sase/core/external_pr_conversion.py`: `plan_external_pr_import_python(request)` —
  the byte-for-byte golden reference, in the role
  `status_wire_conversion._plan_status_transition_python` plays. It is the parity
  oracle, not a runtime fallback; nothing in production calls it.
- `tests/test_external_pr_classifier_parity.py`: a shared case table exercised against
  both implementations, asserting field-for-field equality.

---

## Step 3 — The two-file Patch importer

New `src/sase/ace/patch/importer.py`. It exists because neither existing path can do the
job: `VALID_TRANSITIONS` (`src/sase/status_state_machine/constants.py:44-60`) admits
only `Mailed -> Submitted`, so a merged PR must be _written_ into the archive rather
than transitioned into it; and `add_patch_to_project_file`
(`src/sase/workflows/commit/patch_operations.py:242`) resolves its destination through
`get_project_file_path` (`src/sase/workflows/utils.py:11-22`), which returns the active
spec only.

### Block formatting, shared not duplicated

Extract the Patch-block string builder out of `add_patch_to_project_file` into
`format_patch_block(...)` in `src/sase/ace/patch/storage.py`, taking name, description,
parent, pr_url, pr_origin, bug, status, and the pre-rendered commits/hooks/timestamps
blocks. `add_patch_to_project_file` then calls it and must produce a **byte-identical**
string to today's f-string at `patch_operations.py:455`; the existing commit-workflow
tests are the guard. The importer calls the same helper, so on-disk shape can never
drift between the two writers.

### API

```python
@dataclass(frozen=True)
class ProjectPatchIndex:
    by_pr_key: dict[str, LocalPatchWire]   # canonical PR url key -> patch
    names: set[str]                        # every NAME across both files
    by_name: dict[str, LocalPatchWire]

def read_project_patch_index(active_file: str, archive_file: str) -> ProjectPatchIndex
def apply_external_pr_plan(project_key, plan, *, timeout) -> ImportOutcome
```

`ImportOutcome` reports `action_taken` (`created` / `repaired` / `skipped`), the final
Patch name, the destination file, and a machine reason.

### Locking discipline

`apply_external_pr_plan` acquires `patch_lock(active_file)` **then**
`patch_lock(archive_file)` — always active-before-archive, a fixed global order so this
importer can never deadlock against another two-file holder. Inside both locks:

1. Re-read both files and rebuild `ProjectPatchIndex` **under the lock**. The pre-lock
   index that produced the plan is a hint; this rebuild is the decision.
2. Re-validate: if the canonical PR key is now owned, downgrade to `skipped`
   (`reason="raced_already_owned"`). Never list-then-create across two unlocked
   operations.
3. Resolve the final name. For `adopt` with `pr_origin="external"`, suffix `name_base`
   using the existing `get_next_suffix_number` (`sase.core.patch`) against
   `index.names`, which spans **both** files. For `adopt` with `reason="marker_orphan"`
   the marker name is used verbatim — it is already the suffixed reserved name, and
   re-suffixing it would defeat the repair.
4. Write. `destination == "active"` inserts the block at end-of-file in the active spec.
   `destination == "archive"` appends it to the archive spec, creating the archive file
   (and its parent directory) if absent. Exactly one file is written per plan.
5. For `repair_origin`, rewrite the existing block in place: set `PR:` if absent, set
   `PR_ORIGIN: sase`, replace a `Reserved` stub's `STATUS:` with the mapped status, and
   fill an empty `DESCRIPTION:`. Never touch a field the reservation already carries. If
   the repaired Patch's mapped destination is the archive, complete it in the active
   file first and then move it with the existing `move_patch_to_file`
   (`src/sase/ace/patch/archive.py:165`) — one code path for archive moves, no second
   implementation.

### What an adopted Patch gets, and does not

Gets: the resolved unique `NAME:`, `DESCRIPTION:` from the classifier, `PR:`,
`PR_ORIGIN:`, `STATUS:`.

Does **not** get: `HOOKS:`, a fabricated `PARENT:`, `STITCHES:`, `TIMESTAMPS:`, or a
workspace claim. `PARENT` is never inferred from the PR's base branch — only from
explicit provider metadata or a SASE marker, and neither exists here.

---

## Step 4 — The shared external-mirror library

New package `src/sase/external_mirror/`, generic over object type so `issue_mirror` can
adopt it:

- `state.py` —
  `MirrorCursor(last_updated_at, last_provider_id, backfill_complete, last_full_scan_at)`
  with atomic read/write (the `tempfile.mkstemp` + `os.replace` shape from
  `sase_chop_bead_store_refresh._write_backoff_state`), plus a generalized
  `BackoffEntry` / `read_backoff_state` / `write_backoff_state` / `next_backoff_entry`
  lifted from the same module. State files are keyed `<kind>__<project_key>.json` under
  the caller-supplied state dir, because `ChopScriptContext.state_dir` is the
  **lumberjack** directory shared by every chop in the lane
  (`src/sase/axe/chop_runner_context.py:39`), not a per-instance directory. Note that
  plainly in the module docstring — the epic design assumed per-instance dirs.
- `report.py` — `MirrorReport` counters (`seen`, `fetched`, `created`, `repaired`,
  `skipped`, `conflicts`, `errors`, `budget_exhausted`, `checkpoint_advanced`) with
  `summary_fields()` returning the int/str mapping `runtime.emit_summary` accepts, and a
  `reason()` that picks the single most informative no-op reason.
- `pr_sync.py` — the engine, described next.

### `sync_external_pull_requests(...)`

One function, used by both the chop and the CLI, so the manual entry point is literally
the same code path:

```python
def sync_external_pull_requests(
    *, project_key: str, workspace_dir: str, state_dir: Path,
    provider: PullRequestLister, now: datetime,
    dry_run: bool = False, full: bool = False,
    fetch_limit: int = 200, create_budget: int = 25,
    deadline: float | None = None,
) -> MirrorReport
```

Behavior:

1. **Capability gate.** `supports_pull_requests(workspace_dir)` is a structural probe
   and costs no network. Decline cleanly with `reason="pull_requests_unsupported"`
   otherwise.
2. **Fetch bound, honestly named.** The seam is
   `vcs_list_pull_requests(cwd, state, limit)` — it exposes a **limit, not a page
   cursor** (`sase-github/src/sase_github/plugin.py:409-428`). So the epic's "bound by
   page count" becomes a bounded `fetch_limit` per pass, and the report says `fetched`,
   never "pages". An incremental pass fetches `state="all"` up to `fetch_limit`; a
   `full` pass fetches `state="all"` unbounded (`limit=0`). Say this plainly in the
   docstring; do not imply pagination the provider does not offer.
3. **Overlap window.** Skip records whose `updated_at` is strictly older than
   `cursor.last_updated_at - 10 minutes`, unless `full` or
   `not cursor.backfill_complete`. Equal timestamps are always processed and deduped by
   `provider_id`, which tolerates crash-after-write.
4. **Index once, before writing anything.** Build `ProjectPatchIndex` from both
   ProjectSpec files up front; pass it into each classification.
5. **Classify and apply** each candidate through
   `sase.core.external_pr_facade.plan_external_pr_import` then `apply_external_pr_plan`.
   Stop creating (but keep counting `seen`) once `create_budget` is spent or `deadline`
   passes, following the `managed_tmp_reap` "at most N per pass" convention, so a first
   run against a large backlog converges over several passes.
6. **Cursor advance only on a clean pass.** Advance `last_updated_at` /
   `last_provider_id` **only** when every planned mutation for every processed record
   succeeded and nothing was deferred by budget or deadline. One malformed remote record
   is reported with its `provider_id` and must not advance the checkpoint past
   unprocessed data. Set `backfill_complete` only after an unbounded pass completes.
7. **Daily repair scan.** When `now - cursor.last_full_scan_at >= 24h`, run the pass as
   `full` regardless of the cursor and stamp `last_full_scan_at` on success. This is
   what makes the "every PR" promise recoverable rather than aspirational.
8. **Degraded runs never delete.** Any provider exception increments `errors`, records
   backoff, and returns; it never removes or rewrites a local Patch.
9. **`dry_run`** classifies and reports exactly what would be written, mutating nothing
   and advancing no cursor.

---

## Step 5 — The `external_pr_mirror` chop

`src/sase/scripts/sase_chop_external_pr_mirror.py`, a thin `@builtin_chop` wrapper:
resolve `target`, apply backoff, call `sync_external_pull_requests`, emit the summary.
All logic lives in Step 4 so the CLI and the chop cannot diverge.

- `pyproject.toml`:
  `sase_chop_external_pr_mirror = "sase.scripts.sase_chop_external_pr_mirror:main"`,
  inserted in the existing alphabetical run.
- `src/sase/default_config.yml`, `checks` lane (its own description already says it is
  "for checks that can tolerate delay or may touch remote PR state"), placed
  alphabetically among that lane's chops:

  ```yaml
  - name: external_pr_mirror
    script: sase_chop_external_pr_mirror
    run_every: "10m"
    timeout: "2m"
    for_each:
      source: projects
      vcs: [git, gh]
    description: |-
      Adopt remote pull requests SASE did not create as local Patches
      ...
  ```

  The description must satisfy the grammar in `docs/configuration.md:2060-2067`:
  non-blank summary line ≤ 100 chars, blank line, body, ≤ 2000 chars total.

- `target.project` is the **stable ProjectSpec directory key** and is what addresses the
  spec files; `target.workspace_dir` is the provider `cwd`; `target.name` is the display
  name and is the only one that may appear in user-facing text ("Show Project Names,
  Never ProjectSpec Keys"). Missing/blank `workspace_dir` → `check_error` with
  `reason="missing_target_workspace"`, following `sase_chop_refresh_docs._check_error`.
- `vcs: [git, gh]` filters VCS _kind_, not PR _capability_; the capability gate is the
  runtime `supports_pull_requests()` probe.
- 10m rather than 5m: this is an inventory view, not a pager.

`for_each` has no production usage in `default_config.yml` today; this is its first.
Exercise `sase axe chop list --verbose` and
`sase axe chop run 'external_pr_mirror[<project>]' --dry-run --chop-verbose` and confirm
instance identity, per-instance scheduling, and teardown-on-disable behave as
`docs/configuration.md:2110-2112` documents. If they do not, record a
`PROPOSED FOLLOW-UP` note rather than working around it silently.

---

## Step 6 — `sase patch sync-external`

The manual entry point and the same code path the chop runs.

In `src/sase/main/parser_patch.py`, registered immediately after `sync-deltas` (keeping
that alphabetical run intact; `migrate-extension` stays where it already is):

```
sase patch sync-external [-d|--dry-run] [-f|--full] [-p|--project PROJECT]
```

Every long option gets a short alias, per `cli_rules.md`. `-p/--project` accepts a
project name or alias and resolves through `resolve_project_alias_ref`; omitted means
every enabled project. `--dry-run` shows exact creations and repairs without mutating
anything and without advancing cursors. `--full` forces the unbounded repair pass.

In `src/sase/main/patch_handler.py`: `_handle_sync_external`, dispatch from
`handle_patch_command`, and add `sync-external` to the usage string at line 629 (which
is sorted). Output is a colored per-project table (adopted / repaired / skipped /
errors) via the repo's existing `sase.output` helpers, with a `--dry-run` banner.

---

## Step 7 — Doctor coverage

New `src/sase/doctor/checks_external_pr_mirror.py` exposing
`external_pr_mirror_check_specs(context)`, registered in
`src/sase/doctor/runner.py:build_doctor_registry` right after `axe_check_specs`.

Check id `axe.external_pr_mirror`, **network-free** (doctor's default mode must not make
remote calls), reporting per enabled `git`/`gh` project:

- whether `workspace_dir` resolves,
- the structural `supports_pull_requests()` probe,
- the persisted mirror state written by Step 4: consecutive failures, last recorded
  error, and how stale the cursor is.

The failure this is really guarding is the epic's risk 4 — the interactive TUI's working
`gh` proves nothing about the detached lumberjack's environment, where a silent auth
failure looks exactly like "no PRs". The persisted-state leg detects precisely that,
because it reads what the daemon actually recorded rather than what the interactive
shell can do. FAIL when a project has recorded repeated failures; WARN when its cursor
has not advanced in over a day while the capability probe passes; PASS otherwise.

Keep the module and its check id independent of whatever bead-side tracker-auth check
`sase-jd.4` adds, so the two merge additively.

---

## Step 8 — Docs

- `docs/axe.md`: a `#### Builtin external_pr_mirror` section beside the existing
  `#### Builtin refresh_docs`, covering the fan-out, the capability gate, the fetch
  bound, the cursor and daily repair scan, and the summary fields.
- `docs/project_spec.md`: document `PR_ORIGIN:` (currently undocumented there) and what
  an adopted Patch does and does not carry.
- `docs/cli.md`: `sase patch sync-external`.
- State the enforceable contract plainly, in `docs/axe.md` and in the chop description:
  the `SASE_PATCH` marker identifies **"created by SASE's tracked PR workflow"**, not
  **"created by a SASE agent"**. An agent that bypasses the tracked workflow and calls
  `gh pr create` directly is indistinguishable from a human, and is adopted as
  `external`.

---

## Step 9 — Tests

The epic's four named verifications, each as a real test:

1. **The crash window.** Remote PR exists with `SASE_PATCH=<name>`; the local file holds
   only the bare `NAME: <name>\nSTATUS: Reserved` reservation that
   `compute_suffixed_cl_name` writes at `patch_operations.py:232`. Assert the pass
   _repairs_ that entry (`PR:`, `PR_ORIGIN: sase`, real status) and creates no second
   Patch — and that a second pass over the same state is a no-op.
2. **Merged PR lands in the archive** file directly, with no illegal transition: assert
   the Patch is absent from the active spec, present in the archive spec with
   `STATUS: Submitted`, and that no `Mailed -> Submitted` transition was executed.
3. **URL-variant idempotency.** A PR whose local record differs only by scheme, `www.`,
   host case, trailing slash, or `.git` is not duplicated.
4. **End-to-end AXE exclusion.** Run the real importer to adopt a PR, then assert the
   resulting Patch is absent from `filtered_patches` — with an empty `axe_config.query`,
   with a user query that _would_ have matched it, and with a user query that matches
   nothing.

Plus:

- Rust `tests.rs` for every classification rule and URL normalization case.
- The Python↔Rust parity table (Step 2).
- Importer unit tests: both-file name uniqueness, archive-file creation, the
  active-then-archive lock order, and the under-lock re-check downgrading a raced plan
  to `skipped`.
- Engine tests against the `vcs_provider/testing.py` fake: budget exhaustion defers
  without advancing the cursor; a provider error records backoff and deletes nothing; a
  malformed record is reported by `provider_id` and does not advance the checkpoint;
  `--dry-run` writes nothing; the daily repair scan triggers on a stale
  `last_full_scan_at`.
- Chop tests: missing `workspace_dir` → `check_error`; unsupported provider → `no_op`
  with reason; summary field shape.
- CLI test for `sase patch sync-external --dry-run`.
- Doctor test for each of PASS / WARN / FAIL.

## Step 10 — Verification

```bash
just install      # required first in an ephemeral workspace; also rebuilds sase_core_rs
just check        # whole-repo lint gates + diff-scoped tests
just check-full   # every lint gate + the full suite, before finishing
```

`just check-full` rather than `just check` alone is required here: this change adds a
Rust binding, edits `default_config.yml` and `pyproject.toml`, and refactors a shared
block formatter used by the commit workflow — all squarely in the broadening set.

## Risks and how this plan handles them

1. **Merge collision with `sase-jd.4`.** Bounded to four small, obvious hunks and one
   deliberately shared package (see the concurrency section).
2. **The seam has no pagination.** Acknowledged rather than papered over: the per-pass
   bound is a fetch limit plus a creation budget, and the report and docs say `fetched`.
3. **`state_dir` is per-lane, not per-instance.** State files are project-keyed, and the
   module docstring records that the epic design's assumption did not hold.
4. **Block-formatter extraction touching the commit workflow.** Mitigated by requiring
   byte-identical output and leaning on the existing commit-workflow tests.
5. **Breadth (all PRs, not just self-authored).** Safe only because `pr_origin` already
   landed the structural AXE exclusion; test 4 asserts that end-to-end with a real
   adopted Patch rather than trusting the earlier phase.
6. **`for_each`'s first production use.** Exercised explicitly in Step 5, with a
   `PROPOSED FOLLOW-UP` note if reality diverges from the docs.

## Explicitly out of scope

- Renaming anything Commits/Stitches (`sase-j8.4` owns it).
- The Bugs-pane retirement and sub-tab reorder (`sase-jd.8`).
- Bidirectional write-back: adopting a PR never edits the remote PR.
- Webhooks. Polling is the source of truth.
- Glossary or SASE memory-file entries for "PR origin" — those need the project owner's
  explicit permission and are the epic's recorded follow-up, not a phase deliverable.
