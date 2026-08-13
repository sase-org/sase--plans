---
tier: tale
title: external_pr_mirror chop and the two-file Patch importer
goal:
  Every pull request on an enabled project's remote that SASE's tracked workflow did not
  create has a local Patch, minted by a per-project AXE chop on top of a new importer
  that locks the active and archive ProjectSpec files together and writes merged and
  closed PRs straight into the archive.
size: medium
proposed_by: bbugyi200.athena.sase-jd.5
bead: sase-jd.5
create_time: 2026-08-10 20:21:02
status: wip
---

- **PARENT:**
  [202608/external_artifact_ingestion.md](https://github.com/sase-org/sase--plans/blob/main/202608/external_artifact_ingestion.md)
- **BEAD:**
  [sase-jd.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-jd/sase-jd.5.md)

# Plan: external_pr_mirror chop and the two-file Patch importer

Phase `pr_mirror` of epic `sase-jd`
([@plan:202608/external_artifact_ingestion.md](https://github.com/sase-org/sase--plans/blob/main/202608/external_artifact_ingestion.md)).
Both dependencies have landed: `pr_seam` (`498ef310f` here, `2b0d6a8` in sase-github)
and `pr_origin` (`295140319` here, `d0eeb48` in sase-core).

## What already exists (verified)

- `PullRequestWire` with
  `number, title, state, provider_id, url, body, is_draft, author, head_ref, base_ref, created_at, updated_at, closed_at, merged_at`
  (`src/sase/vcs_provider/_types.py:37`),
  `provider.list_pull_requests(cwd, state, limit)`
  (`src/sase/vcs_provider/_base.py:418`), `supports_pull_requests(cwd)`
  (`src/sase/vcs_provider/_registry.py`), and a network-free fake in
  `src/sase/vcs_provider/testing.py`.
- sase-github implements `vcs_list_pull_requests` over `gh pr list --state all`,
  collapsing GitHub's `MERGED` into `state="closed"` with `merged_at` populated
  (`src/sase_github/plugin.py:409`). `limit <= 0` means unbounded
  (`_UNBOUNDED_ISSUE_LIMIT = 1_000_000`).
- `PR_ORIGIN` parses, serializes, and renders; `normalize_pr_origin` defaults absent and
  invalid values to `unknown` (`src/sase/ace/patch/models/patch.py:28`). The Rust parser
  mirrors it (`crates/sase_core/src/parser.rs:297`).
- The AXE safety exclusion is structural and already in place:
  `filter_axe_candidate_patches` (`src/sase/axe/patch_filtering.py`) is applied in
  `chop_runner_context.py`, `lumberjack.py`, and `check_cycles.py` before any user
  query.
- `SASE_PATCH=<reserved-name>` is stamped unconditionally by `append_pr_tags`
  (`src/sase/workflows/commit/pr_operations.py:92`) into `payload["message"]`, and
  sase-github passes `payload.get("_pr_body", message)` as the `gh pr create --body`
  value — so **the marker does reach the PR body**, which is what
  `vcs_list_pull_requests` returns. No separate body path is needed.
- `parse_commit_footer` (Rust, exposed via `src/sase/core/commit_footer_facade.py`)
  parses the trailing tag block and canonicalizes `SASE_PATCH=` to the key `PATCH`.
- `for_each: {source: projects}` expands one stable instance per enabled project and
  puts `{name, project, vcs, workspace, workspace_dir, enabled, launchable}` into
  `ChopScriptContext.target` (`src/sase/axe/_config_targets.py:118-190`). The project
  record's `workspace_dir` is the primary repo checkout, so it is a valid provider
  `cwd`. No chop in `default_config.yml` uses `for_each` yet — this is its first
  production use.

## Design decisions

### Listing strategy: the daily full scan is the correctness backstop

The provider seam has **no cursor** — `list_pull_requests(cwd, state, limit)` and
`gh pr list` order by creation date descending. A limit-bounded incremental slice is
therefore a _creation_ window, not an _update_ window: an old PR updated today falls
outside a `limit=100` slice on a repo with more than 100 PRs. So:

- **Incremental pass** (every 10m): `state="all"`, `limit=100`. Cheap freshness pass
  over the newest PRs. The watermark is used only to skip records whose `updated_at` is
  older than `watermark - 10m` overlap, and dedupe is by `provider_id`.
- **Full repair scan** (first run, and at least daily): `state="all"`, `limit=0`
  (unbounded). The inventory is **never truncated**; only the _mutations_ are budgeted
  (`_MAX_IMPORTS_PER_PASS = 25`). A full scan that defers work does not stamp
  `last_full_scan_at`, so it resumes next pass and converges over several passes.

This is the honest reading of "bound work by pages, never by truncating the inventory"
against a seam that has no pages. Document the contract plainly: the "every PR has a
Patch" promise is held by the daily full scan, and incremental passes only shorten the
latency for recently created PRs.

### The classifier lives in Rust

Per the epic's ground rules and `CLAUDE.md`'s Rust core boundary, PR URL
canonicalization and the origin/status decision are deterministic, I/O-free domain logic
a future web or CLI frontend must agree with. They go in `sase-core`. Provider calls,
cursor files, and ProjectSpec writes stay in Python.

## Work

### 1. `sase-core`: deterministic PR reconciliation classifier

Open with `/sase_repo` (`sase repo open sase-core`). New module
`crates/sase_core/src/pr_mirror/` (`mod.rs` + `wire.rs`), registered in `lib.rs`.

**`normalize_pull_request_url(url: &str) -> PullRequestUrlWire`** returning
`{canonical, host, owner, repo, number, parsed}`:

- drop scheme, userinfo, default port, query, fragment, trailing slashes
- lowercase host, strip a leading `www.`
- lowercase `owner`/`repo`, strip a trailing `.git` from `repo`
- accept `/<owner>/<repo>/pull/<n>`, `/pulls/<n>`, `/pull-requests/<n>`, and GitLab's
  `/-/merge_requests/<n>`
- canonical form `"<host>/<owner>/<repo>#<n>"`; when the shape does not parse, set
  `parsed=false` and fall back to the scheme-stripped, lowercased, slash-trimmed string
  so equality still beats raw comparison

**`classify_pull_request(request) -> PullRequestImportDecisionWire`**. Input carries the
PR record (`number, url, state, is_draft, merged_at, body`), the optional `url_owner`
(the local Patch already owning the canonical URL), and the optional `marker_owner` (the
local Patch named by the body's `SASE_PATCH` marker); both are
`{name, pr_origin, status, pr_url, is_reservation}`. The marker itself is extracted
inside Rust by reusing `commit_footer::parse_commit_footer` on the body and reading the
canonical `PATCH` key. Output:
`{action, pr_origin, status, destination, patch_name, marker_name, canonical_url, reason}`.

Decision order (exactly the epic's order):

| #   | Condition                                      | action   | pr_origin          | reason                  |
| --- | ---------------------------------------------- | -------- | ------------------ | ----------------------- |
| 1   | canonical URL already owned locally            | `skip`   | owner's, preserved | `already_owned`         |
| 2a  | marker names a bare reservation                | `repair` | `sase`             | `reservation_repair`    |
| 2b  | marker names a Patch with no `PR:`             | `repair` | `sase`             | `missing_pr_url_repair` |
| 2c  | marker names a Patch bound to a _different_ PR | `import` | `unknown`          | `marker_conflict`       |
| 2d  | marker names no local Patch                    | `import` | `sase`             | `marker_patch_missing`  |
| 3   | no marker                                      | `import` | `external`         | `no_marker`             |
| 4   | marker present but blank/unusable              | `import` | `unknown`          | `ambiguous_marker`      |

Status and destination mapping:

| Remote state           | status      | destination |
| ---------------------- | ----------- | ----------- |
| open + draft           | `Draft`     | active      |
| open, ready            | `Mailed`    | active      |
| closed + `merged_at`   | `Submitted` | archive     |
| closed, no `merged_at` | `Archived`  | archive     |

`Mailed` rather than `Ready`: a live remote PR has already passed "locally ready to
mail". Destination is selected by `status::constants::ARCHIVE_STATUSES`, so a merged PR
is **written into the archive**, never transitioned into it (`VALID_TRANSITIONS` admits
only `Mailed → Submitted`).

Rust unit tests for URL normalization (host case, `www.`, `.git`, trailing slash,
query/fragment, enterprise host, GitLab path, unparseable input) and for every row of
both tables.

**PyO3 bindings** in `crates/sase_core_py/src/lib.rs`, following the
`py_parse_commit_footer` shape: `pr_mirror_wire_schema_version`,
`normalize_pull_request_url`, `classify_pull_request`.

sase-core must be committed and pushed to its master before this repo's PR: sase CI
builds `sase_core_rs` from a fresh `sase-org/sase-core` checkout. The published-floor
smoke job only runs on the release-please branch, and
`tools/check_sase_core_rs_bindings` discovers new bindings statically, so no manual
floor bump is needed.

### 2. This repo: Rust facade

`src/sase/core/pr_mirror_facade.py`, following `commit_footer_facade.py`: frozen
dataclasses `PullRequestUrl` and `PullRequestImportDecision`, `require_rust_binding`
lookups, schema-version assertion, and no Python fallback.

### 3. This repo: the two-file Patch importer

New `src/sase/ace/patch/importer.py`. `add_patch_to_project_file` cannot be reused: it
resolves its destination through `get_project_file_path(project)`
(`src/sase/workflows/utils.py:11`), which returns the **active** spec only.

```python
def import_patch(
    *, project_file, name, description, pr_url, pr_origin, status,
    lock_timeout=30.0,
) -> ImportedPatch | None
def repair_patch_pr_association(
    *, project_file, patch_name, pr_url, pr_origin, status, lock_timeout=30.0,
) -> ImportedPatch | None
```

Both take the **active** spec path, derive the archive path with
`get_archive_file_path`, and hold `patch_lock` on active _then_ archive — always in that
order, so no deadlock with `add_patch_to_project_file`, which holds active and reads
archive unlocked.

`import_patch`, under both locks:

1. read both files; build the name set and the canonical-URL → name map across both
   (canonicalizing every `PR:`/`CL:` value through the Rust normalizer)
2. if the canonical URL is already present, return `None` — the importer is idempotent
   on its own, independent of the classifier
3. suffix the name with `get_next_suffix_number` over the union of both name sets
4. destination = archive when `status in ARCHIVE_STATUSES`, else active; create the
   archive file if it does not exist
5. append a block in `PATCH_SECTION_ORDER` order — `NAME`, `DESCRIPTION`, `PR`,
   `PR_ORIGIN`, `STATUS` and nothing else. **No `PARENT`** (never inferred from the base
   branch), no `HOOKS`, no `STITCHES`, no workspace.
6. `write_patch_atomic`

`repair_patch_pr_association`, under both locks:

- a **bare `Reserved` stub** is replaced by a full block written to the mapped
  destination, removing the stub from wherever it lived — this closes the crash window
  between remote PR creation and local Patch completion without minting a duplicate
- a **real Patch** only gains `PR:` and `PR_ORIGIN:` inserted in section order; its
  `STATUS` and its file are left alone, because its own lifecycle owns them

Name derivation for imports: sanitize the PR title into a Patch name, falling back to
the head branch, then to `pr_<number>`; reuse `src/sase/core/patch.py` helpers where
they fit. Description = PR title, blank line, PR body with the trailing SASE tag block
stripped via `parse_commit_footer(...).body`.

### 4. This repo: shared mirror-state library

New `src/sase/external_mirror/` — deliberately object-type-agnostic so the in-flight
`issue_mirror` phase (`sase-jd.4`) can adopt it:

- `state.py`:
  `MirrorCursor(updated_at, provider_id, last_full_scan_at, failures, next_attempt_at)`,
  atomic read/write to `<state_dir>/external_mirror/<kind>/<project-key>.json`,
  `overlap_window()`, and `next_backoff()` following the `bead_store_refresh`
  exponential shape
- `pull_requests.py`: `run_pull_request_mirror_pass(...) -> MirrorPassResult`, the one
  code path both the chop and the CLI call

`run_pull_request_mirror_pass` per pass:

1. gate on `supports_pull_requests(workspace_dir)`; decline with a recorded reason
2. honor the backoff record; record failure backoff _before_ the provider call, so a
   SIGKILL at the chop timeout cannot erase the entry it needs
3. pick full vs incremental (no cursor, `--full`, or `last_full_scan_at` older than 24h
   → full)
4. list PRs; apply the `external_mirror.pr_authors` filter when non-empty
5. parse the project's active + archive specs once into the URL and name indexes
6. process candidates ordered by `(updated_at, provider_id)`, classify, then act: `skip`
   / `repair` / `import`, re-checking the index under the importer's lock
7. stop at `_MAX_IMPORTS_PER_PASS = 25` or the work deadline and count the remainder as
   `deferred`
8. advance the watermark **only when every planned mutation succeeded and nothing was
   deferred**; stamp `last_full_scan_at` only for a full pass that completed cleanly
9. a malformed remote record is reported with its `provider_id` and never advances the
   checkpoint past unprocessed data
10. `dry_run` classifies and counts but performs no mutation and no cursor write

### 5. This repo: the chop

`src/sase/scripts/sase_chop_external_pr_mirror.py` — a thin `@builtin_chop`
("external_pr_mirror") adapter that reads `runtime.context.target`, calls
`run_pull_request_mirror_pass`, and emits
`external_pr_mirror: pull_requests=… imported=… repaired=… skipped=… conflicts=… deferred=… failed=… full_scan=… reason=…`.
Missing `target.project` or `target.workspace_dir` is a `check_error`, matching
`refresh_docs`.

Register the entry point in `pyproject.toml` and the chop in `default_config.yml` under
the **`checks`** lane (its description already covers "checks that can tolerate delay or
may touch remote PR state"):

```yaml
- name: external_pr_mirror
  script: sase_chop_external_pr_mirror
  run_every: "10m"
  timeout: "2m"
  for_each:
    source: projects
    vcs: [git, gh]
  description: |-
    Adopt remote pull requests SASE's tracked workflow did not create as Patches
    ...
```

`vcs: [git, gh]` filters VCS _kind_, not PR _capability_; the capability gate is the
runtime `supports_pull_requests()` probe.

Exercise `sase axe chop list --verbose` against the expanded instances and confirm
per-instance identity, scheduling, and teardown-on-disable behave as
`docs/configuration.md` documents. Record a `PROPOSED FOLLOW-UP:` note on `sase-jd.5` if
they do not.

### 6. This repo: config knob

Add to `src/sase/default_config.yml` and `src/sase/config/sase.schema.json`:

```yaml
external_mirror:
  # Restrict PR adoption to these author logins. Empty means every author.
  pr_authors: []
```

Read through a small `src/sase/external_mirror/config.py` accessor modeled on
`get_vcs_provider_config`. Per-pass budgets stay module constants, as in
`bead_store_refresh`.

### 7. This repo: CLI

`sase patch sync-external` (alias `sase changespec sync-external`), registered in
`src/sase/main/parser_patch.py` and handled in `src/sase/main/patch_handler.py`. It
calls the same `run_pull_request_mirror_pass`. Options, alphabetical, each with a short
alias per `sase/memory/cli_rules.md`:

- `-n, --dry-run` — show exact creations without mutating or advancing cursors
- `-f, --full` — force the full repair scan
- `-p, --project PROJECT` — one project (default: every enabled project)

Rich, colored summary output.

### 8. This repo: doctor check

`src/sase/doctor/checks_deep_vcs_pull_requests.py`, registered as a **deep** check
`vcs.pull_requests` in `src/sase/doctor/checks_deep.py`. For each enabled project with a
workspace dir, it probes `list_pull_requests(state="open", limit=1)` in an environment
scrubbed to what the detached lumberjack inherits. This is the check that distinguishes
"no PRs" from a silent auth failure in the daemon environment, which the interactive
TUI's working `gh` proves nothing about. Deep, because it touches the network.

### 9. Docs

- `docs/axe.md`: add `external_pr_mirror` to the `checks` lane table plus a paragraph
  covering the classification order, the status/destination mapping, the incremental vs
  daily-full-scan contract, and the structural AXE exclusion of adopted Patches.
- `docs/configuration.md`: document `external_mirror.pr_authors` and note the first
  production `for_each` chop.
- State the enforceable contract plainly: this identifies **"created by SASE's tracked
  PR workflow"**, not "created by a SASE agent". An agent that bypasses the tracked
  workflow and calls `gh pr create` directly is indistinguishable from a human.

## Verification

New tests, all network-free against `vcs_provider/testing.py`:

- **The crash window**: the remote PR exists and carries the marker while the local
  Patch is still a bare `Reserved` stub — assert the next pass repairs the stub in place
  with `PR_ORIGIN: sase` and creates no second Patch.
- A merged PR lands in the **archive** file with `STATUS: Submitted` and no illegal
  transition; a closed-unmerged PR lands in the archive as `Archived`; open draft and
  open ready land in the active file as `Draft` and `Mailed`.
- URL variance: `http` vs `https`, `www.`, trailing slash, `.git`, query and fragment,
  and host case all resolve to one canonical identity and do not duplicate.
- End-to-end: an adopted `PR_ORIGIN: external` Patch is absent from `filtered_patches`
  with a real adopted Patch on disk (the exclusion itself is `pr_origin`'s test; this
  asserts the two compose).
- Importer: concurrent-safe name suffixing across active _and_ archive; idempotent
  re-import; no `PARENT`, `HOOKS`, or `STITCHES` on an adopted Patch.
- Classifier: every row of both decision tables, including `marker_conflict` and
  `ambiguous_marker` producing `unknown` rather than a guess.
- Chop: capability decline, backoff, per-pass import budget with `deferred` counted, dry
  run mutating nothing, and the checkpoint not advancing past a malformed record.
- CLI: `--dry-run` output and exit status.

Then `just install` (ephemeral workspace), `just check`, and `just check-full` before
landing. `cargo test` and `cargo clippy` in sase-core.

## Out of scope

- Anything Commits/Stitches — `sase-j8.4` owns that rename.
- SASE memory files (`sase/memory/*.md`, `AGENTS.md`, `CLAUDE.md`, and the provider
  shims). Glossary entries for "PR origin" are the project owner's call.
- The `origin` query property, the Patch row badge, and `sase patch set-origin` — those
  are the `patch_pr_ui` phase.
- Webhooks and bidirectional write-back, per the epic's non-goals.
