---
tier: tale
size: medium
title: ProjectSpec description truncation and duplicate-block repair
goal:
  A Patch whose DESCRIPTION contains a blank run survives every ProjectSpec read and
  write in both the Python and Rust parsers, the block writer can no longer emit a
  description that terminates its own record, and `sase doctor -F` reclaims the archives
  the external PR mirror has already filled with duplicate blocks.
proposed_by: bbugyi200.athena.sase-k2.1
bead: sase-k2.1
create_time: 2026-08-12 11:46:44
status: wip
---

- **PARENT:**
  [202608/external_mirror_refinement.md](https://github.com/sase-org/sase--plans/blob/main/202608/external_mirror_refinement.md)
- **BEAD:**
  [sase-k2.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-k2/sase-k2.1.md)

# Plan: stop losing Patches to indented blank runs, and reclaim the archives already corrupted

This is the `spec_repair` phase of epic `sase-k2`
(`@plan:202608/external_mirror_refinement.md`). It is a prerequisite for the epic's
`patch_status` and `perf` phases.

## What is broken

`format_patch_block` (`src/sase/ace/patch/storage.py:56-57`) prefixes every DESCRIPTION
line with two spaces, so a blank line inside a description is written to disk as the
two-character line `"  "`. Every record-terminator implementation then treats any line
that _strips_ to empty as blank, and ends the Patch record after two of them:

| implementation                                       | test used                |
| ---------------------------------------------------- | ------------------------ |
| `src/sase/ace/patch/parser.py:285`                   | `line.strip() == ""`     |
| `src/sase/ace/patch/raw_text.py:61-64`               | `line.strip() == ""`     |
| `src/sase/ace/timestamps/recording.py:97-100`        | `line.strip() == ""`     |
| `crates/sase_core/src/parser.rs:437-441` (sase-core) | `line.trim().is_empty()` |

A release-please PR body contains `---` followed by two blank lines. The record
therefore ends before its `PR:` and `STATUS:` lines are read, `build_patch()` returns
`None` because `status` is unset (`parser.py:93`), and the Patch vanishes from
`read_project_patch_index`. With the record invisible, both the `by_pr_key` dedup and
the `index.names` suffix dedup in `src/sase/ace/patch/importer.py` miss, so
`apply_external_pr_plan` adopts the same PR again and `_append_patch_block` writes
another ~10 KB copy on every pass, forever.

This is a general ProjectSpec data-loss bug, not a mirror bug: _any_ Patch whose
DESCRIPTION contains two consecutive blank lines is invisible to every consumer of
`parse_project_file`.

Measured on the host machine on 2026-08-12, on the `gh_sase-org__sase` project:

- `gh_sase-org__sase-archive.sase` is **34 028 597 bytes** with **3392 `^NAME: ` lines**
  but only **289 unique names**; `parse_project_file` returns **264 Patches**.
- The 25 missing names are exactly the release-please Patches
  (`chore_master_release_0_1_1_1` … `chore_master_release_0_13_0_1`), each appended
  105–131 times.
- The active `gh_sase-org__sase.sase` currently holds zero Patches, so all corruption is
  in the archive. Do not assume that: the repair must handle both files.

Confirm the numbers before and after with:

```bash
ARCHIVE=~/.sase/projects/gh_sase-org__sase/gh_sase-org__sase-archive.sase
grep -c '^NAME: ' "$ARCHIVE"; grep '^NAME: ' "$ARCHIVE" | sort -u | wc -l
```

## Ordering constraint

Landing the parser fix _before_ the repair path exists would make ~3100 duplicate blocks
abruptly visible as real Patches with colliding names. Build and land the repair path
first (steps 1–3), then the writer and parser changes (steps 4–6), and run the live
repair (step 10) immediately after installing the new code. Between install and repair
the mirror cannot make things worse: with the parser fixed, `by_pr_key` finally hits and
`apply_external_pr_plan` stops appending.

## Step 1 — pure raw-text block splitter and de-duplicator

New module `src/sase/ace/patch/duplicate_blocks.py`. Pure text in, pure text out, **no
IO and no dependency on the parser** — the records it must find are exactly the ones the
parser cannot see.

```python
@dataclass(frozen=True)
class DuplicateBlockScan:
    total_blocks: int
    unique_names: int
    duplicate_names: tuple[str, ...]      # sorted, names with >1 block
    dropped_blocks: int
    reclaimable_bytes: int
```

Public functions:

- `split_patch_blocks(text) -> tuple[str, tuple[tuple[str, str], ...]]` — return the
  preamble and an ordered `(name, block_text)` sequence.
- `scan_duplicate_patch_blocks(text) -> DuplicateBlockScan`.
- `dedupe_patch_blocks(text) -> tuple[str, DuplicateBlockScan]`.

Splitting rules:

- A block anchor is a line that starts with `NAME: ` **at column 0**. Descriptions are
  always two-space indented by the writer, so an anchor can never be description text.
- A block starts at its anchor, then walks _backwards_ over blank lines and, if the line
  immediately before those blanks is a `## Patch` / `## ChangeSpec` heading
  (`storage.is_patch_heading`), includes that heading too. Mirror the backward walk in
  `archive.extract_patch_block:108-128`. Attaching the leading separator blanks to the
  _following_ block is what keeps blank separators from accumulating when a block is
  dropped.
- A block ends where the next block starts; the last block runs to end of text.
- The preamble is everything before the first block start (project metadata such as
  `WORKSPACE_DIR:`, `PROJECT_NAME:`, `RUNNING:`).

De-duplication rules:

- The key is `line[6:].strip()` from the anchor. A block whose name is empty has **no**
  key and is never dropped.
- Keep exactly one block per name: the **last**, because it was written most recently
  and therefore carries the freshest STATUS.
- Emit `preamble + "".join(kept blocks in their surviving order)`.
- `reclaimable_bytes` is the summed length of the dropped blocks.

Required property: for a file with no duplicate names, `dedupe_patch_blocks` returns the
input **byte for byte**. Assert this directly in the tests — it is what makes the repair
safe to run over every project.

## Step 2 — locked repair driver

New module `src/sase/ace/patch/duplicate_repair.py`.

```python
@dataclass(frozen=True)
class DuplicateBlockPreview:
    project_key: str
    project_label: str
    active_file: str
    archive_file: str
    active_scan: DuplicateBlockScan
    archive_scan: DuplicateBlockScan
```

- `plan_duplicate_block_repairs(*, projects_root: Path) -> tuple[DuplicateBlockPreview, ...]`
  — enumerate every direct subdirectory of `projects_root` whose name passes
  `sase.core.paths.is_valid_sase_project_name` (include `home`; corruption does not care
  about lifecycle state), resolve both spec paths with
  `project_spec_path.preferred_project_spec_path(dir, key)` and `(…, archive=True)`,
  read and scan each that exists. Read-only; takes no lock. Return only previews with at
  least one duplicate. Resolve `project_label` through
  `sase.project_display_names.load_project_display_snapshot().label_for(key)` — per
  `CLAUDE.md`, user-facing text shows `sase`, never `gh_sase-org__sase`.
- `apply_duplicate_block_repairs(previews) -> tuple[DuplicateBlockRepairResult, ...]` —
  per project take `patch_lock(active_file)` then `patch_lock(archive_file)` (active
  before archive, matching `importer.apply_external_pr_plan:94-95`), **re-read and
  re-dedupe under the lock** rather than writing the preview's text, and persist with
  `write_patch_atomic`. The operation is content-derived and idempotent, so re-deriving
  under the lock is both the concurrency guard and the correctness guard. Report
  per-file `dropped_blocks` and `reclaimable_bytes` actually applied.
- Let `LockTimeoutError` and `OSError` propagate as a per-project failure recorded in
  the result rather than aborting the whole sweep.

Missing files are skipped, not created.

## Step 3 — `sase doctor` check plus confirmed fix

**Check.** New module `src/sase/doctor/checks_project_spec_duplicates.py` exporting
`project_spec_duplicate_check_specs(context) -> tuple[CheckSpec, ...]` with a single
spec: `id="project.duplicate_patch_blocks"`, `group="project"`,
`title="Duplicate Patch blocks"`. Register it in `src/sase/doctor/runner.py`'s
`build_doctor_registry` immediately after `external_pr_mirror_check_specs(context)`;
because `diagnostics/render.py:_group_checks` merges by group name, it renders as the
first row of the existing Project group.

Statuses:

- `SKIP` when `~/.sase/projects` does not exist.
- `ERROR` when the projects root cannot be scanned (`OSError`).
- `OK` when no project has a duplicate name.
- `WARN` otherwise. Not `ERROR`: the condition is fully repairable by a documented
  command and does not indicate a broken install, which matches how
  `project.junk_directories` treats recoverable project-root drift.

`summary` names the affected project count and total reclaimable bytes. `details` list
one bounded line per project (cap at 10, consistent with `_MAX_DETAIL_ROWS` elsewhere in
the doctor package) as `WARN: <label>: <n> duplicate name(s), <bytes> reclaimable`.
`next_steps` is `("Run `sase doctor
-F` to preview and apply the ProjectSpec de-duplication repair.",)`. `data` carries
`project_count`, `duplicate_name_count`, `dropped_block_count`, `reclaimable_bytes`, and
a `projects` row list holding the project key, label, per-file counts and byte totals.

Cost: the scan streams `~34 MB` in the worst case observed, about 0.1 s. That is inside
the default doctor budget; do not mark the check deep.

**CLI.** In `src/sase/main/parser_doctor.py` add two options, following the established
`sase bead doctor` repair convention (`src/sase/main/parser_bead_store.py:140-170`) and
`sase/memory/cli_rules.md`'s "every public long option gets a short alias":

- `-F, --fix-duplicate-blocks` — "Preview and, after confirmation, keep one block per
  Patch name in every ProjectSpec"
- `-y, --yes` — "Apply requested repairs without an interactive confirmation"

Update the parser epilog with a `sase doctor -F` example.

**Handler.** In `src/sase/main/doctor_handler.py`, after the report is rendered or
printed as JSON, and only when `-F` was passed: plan the repairs, render a preview
table, and return early with a "nothing to repair" line when the preview is empty.
Otherwise confirm with an `input()` prompt unless `--yes` was given, returning `False`
without prompting when `sys.stdin.isatty()` is false — copy the shape of
`sase.bead.cli_admin._confirm_design_ref_repairs`. On decline, print "ProjectSpec
de-duplication cancelled; no changes applied." and change nothing. On accept, apply and
print one summary line per repaired file plus a total. Keep the process exit code the
report's own `report.exit_code()`; the printed summary tells the user to rerun
`sase doctor` to see the warning clear. Do not run the repair in `-L/--list-checks`
mode.

## Step 4 — writer hardening

In `format_patch_block` (`src/sase/ace/patch/storage.py`), collapse any run of two or
more blank lines in the supplied description to a single blank line **before**
indenting. "Blank" here means `line.strip() == ""`, so an already-indented blank
collapses too. This keeps old parsers, external tooling, and un-migrated ProjectSpecs
safe regardless of the parser change, and it is what makes step 5 a repair rather than a
load-bearing dependency. It must not otherwise alter the description: single blank
lines, leading and trailing whitespace handling, and the existing `.strip()` stay as
they are.

## Step 5 — Python record-terminator fix

Count a line toward the two-blank-line record terminator only when it is _truly_ empty —
no leading whitespace — so an indented in-description blank stays description content.
Two genuinely blank lines still end a record, which is what hand-written specs, the
golden corpus, and `format_patch_block`'s `"\n\n"` block separator rely on.

Add one shared helper next to the other spelling helpers in
`src/sase/ace/patch/storage.py`:

```python
def is_record_separator_line(line: str) -> bool:
    """Return True when *line* counts toward the two-blank-line record terminator."""
    return line.rstrip("\r\n") == ""
```

Use it in all three Python implementations:

- `src/sase/ace/patch/parser.py:285` in `parse_patch_from_lines`.
- `src/sase/ace/patch/raw_text.py:61` in `get_raw_patch_text` — otherwise the raw text
  every viewer shows is truncated at the description's blank run.
- `src/sase/ace/timestamps/recording.py:97` in `_find_timestamps_insert_point` —
  otherwise `patch_end_idx` lands inside a description and a `TIMESTAMPS:` header is
  inserted into the middle of the body. This one is a latent write corruption, which is
  why the fix covers all four terminator implementations and not only the two the phase
  description names.

Do **not** touch the `stripped == ""` branch in `_parse_section_content`: that one is
about preserving blank lines _inside_ a description and is already correct.
`archive.extract_patch_block` needs no change — it already ends a block on the next
`NAME:`/heading rather than on blank lines.

## Step 6 — Rust record-terminator fix (sase-core)

Parsing is core backend logic per `CLAUDE.md`'s Rust boundary rule, so the Rust parser
must move in lockstep. Open the linked repo with the `/sase_repo` skill
(`sase repo open sase-core -r "..."`) and use only the path it prints.

**Warning:** `sase repo open` cleans the checkout and resets it to `origin/master`. Open
it exactly once, at the start, and never re-open it after editing — a second open
discards the work.

Changes in `crates/sase_core/src/parser.rs`:

- In `parse_one_patch` (around line 437), the terminator currently tests
  `stripped.is_empty()` where `stripped = line.trim()`. Test the raw `line.is_empty()`
  instead. `str::lines()` already strips line terminators, so that is the exact
  equivalent of the Python helper. Leave the `stripped` binding in place for the
  `last_content_idx` tracking, which should keep using the trimmed test.
- Update the module doc comment's `end-on-two-blank-lines` bullet (lines 5-18) to say
  the terminator counts only truly empty lines, and note that two-space-indented blanks
  inside a DESCRIPTION are content.
- Add a `#[cfg(test)]` unit test in the existing `mod tests` asserting that a Patch
  whose DESCRIPTION contains an indented blank run still yields its `PR`, `PR_ORIGIN`,
  and `STATUS`, and a second asserting two genuinely blank lines still separate records.

Do **not** bump the wire schema: the record shape is unchanged, only which records get
produced. Do **not** touch `[workspace.package].version` or crate versions — release-plz
owns them (`sase-core/AGENTS.md`).

Do **not** bump `sase-core-rs` in this repo's `pyproject.toml`. The pin is
`>=0.24.0,<0.25.0` while sase-core master is already past `v0.26`, and no production
Python code path routes ProjectSpec parsing through Rust: `parser_facade`'s
`parse_patch_project_file` is Python-only, and `parse_project_bytes` /
`parse_patch_project_bytes` are referenced only by `tools/validate_sase_core_rs`, the
perf benches, and facade tests. The Rust change here is parity, not the fix; pulling the
floor forward is out of scope for this phase.

Verify with `just rust-check` from this repo (it resolves the linked checkout and runs
fmt-check, clippy, and tests). Neither `just check` nor `just check-full` runs it.

## Step 7 — shared golden corpus

Extend the corpus with a Patch whose DESCRIPTION contains a blank run so the two
implementations can never drift apart again. Append it to the **archive** corpus, which
is the natural home for an adopted external release PR (`Submitted`) and has the
smallest blast radius: `myproj.sase` is additionally consumed by
`tests/_query_golden_corpus.py`, `tests/perf/bench_core_parse.py`, and the
`parse_project_bytes.golden_myproj` perf floor in
`tests/perf/baselines/phase7_regression_floor.json`, while the archive corpus is read
only by `tests/test_core_golden.py` and sase-core's `golden_corpus_parity.rs`.

Append to `tests/core_golden/myproj-archive.sase` after its current line 15
(`STATUS: Reverted`), keeping existing `start_line` values untouched. Lines 16 and 17
are truly blank; lines marked `··` below are the two-space indented blanks that are the
whole point of the fixture:

```text
NAME: release_blank_run_1
DESCRIPTION:
  chore(master): release 1.2.3
··
  :robot: Body text after an indented blank run.
··
··
  ## [1.2.3](https://example.test/repo/compare/v1.2.2...v1.2.3)
PR: https://example.test/repo/pull/123
PR_ORIGIN: external
STATUS: Submitted
```

Copy the file byte for byte to sase-core's
`crates/sase_core/tests/fixtures/myproj-archive.sase` — `golden_corpus_parity.rs`
documents the fixtures as byte-for-byte copies.

Then update the expectations:

- `tests/test_core_golden.py::test_corpus_parses_to_expected_names` — archive names
  become `["archived_one", "reverted_two", "release_blank_run_1"]`.
- `tests/test_core_golden.py::test_archive_corpus_wire_json_snapshot` — a third record
  with `start_line`/`end_line` `18`, `status: "Submitted"`, `pr_url`
  `"https://example.test/repo/pull/123"`, `pr_origin: "external"`, all section lists
  empty, and the description
  `"chore(master): release 1.2.3\n\n:robot: Body text after an indented blank run.\n\n\n## [1.2.3](https://example.test/repo/compare/v1.2.2...v1.2.3)"`.
  Regenerate with `inline-snapshot` (`pytest --inline-snapshot=fix`) rather than hand-
  editing, then read the diff to confirm it matches the description above.
- sase-core's
  `golden_corpus_parity.rs::archive_corpus_matches_python_golden_after_end_line_normalization`
  — add the matching third `json!` object by hand.

## Step 8 — documentation

- `docs/project_spec.md` — the "Blank lines between Patches" note at line 351 and the
  format note at line 19 must say the separator is two _truly empty_ lines, and that
  two-space-indented blank lines inside a DESCRIPTION are body content, not a separator.
- `docs/cli.md` — the "Doctor Support Reports" section at lines 406-438 says
  `sase doctor` "is read-only by default: it does not launch agents, call LLM APIs,
  repair state…". Keep that sentence (it is still true of the default) and add the
  `-F`/`--fix-duplicate-blocks` opt-in, the `project.duplicate_patch_blocks` check, and
  a `sase doctor -F` example beside the existing ones. Also add the flag to the command
  table entry at line 319 if that table lists options.

## Step 9 — regression coverage

Python, new file `tests/test_project_spec_duplicate_blocks.py` unless an existing module
is a clearly better home:

1. **Round trip.** Format a release-please-shaped body through `format_patch_block`
   (title, blank, `---`, blank, blank, body), write it, parse it back, and assert the
   Patch survives with `pr_url`, `pr_origin`, and `status` intact.
2. **Writer hardening.** `format_patch_block` given a description containing three
   consecutive blank lines emits at most one blank line between the surrounding body
   lines, and the two-blank-line separator can never appear inside the emitted block.
3. **Terminator.** A ProjectSpec where a Patch's description holds two indented blanks
   parses to one complete Patch; a ProjectSpec where two Patches are separated by two
   truly blank lines still parses to two Patches.
4. **`get_raw_patch_text`** returns the whole block, including the indented blank run,
   for the same fixture.
5. **Timestamps.** `add_timestamp_entry_atomic` on a Patch whose description holds an
   indented blank run appends the `TIMESTAMPS:` section at the end of the record, not
   inside the description.
6. **Idempotent dedup.** `dedupe_patch_blocks` over a clean multi-Patch fixture returns
   the input byte for byte; over a fixture with a name repeated three times it keeps the
   last block only, preserves the preamble and the other Patches' order, and reports the
   right `dropped_blocks`/`reclaimable_bytes`. Cover a `## Patch`-headed fixture and an
   anchor with an empty `NAME:` value (never dropped).
7. **Repair driver.** Over a temp projects root with an active and an archive file,
   `plan_duplicate_block_repairs` finds the duplicates and
   `apply_duplicate_block_repairs` rewrites both files, is a no-op on a second run, and
   leaves clean projects untouched.
8. **Two adoption passes.** Extend `tests/test_external_pr_importer.py`: run
   `apply_external_pr_plan` twice for the same remote PR whose description contains a
   blank run and assert the destination file holds exactly one block for that PR — the
   direct regression test for the bug that produced 3111 phantom blocks.

Doctor, new file `tests/doctor/test_checks_project_spec_duplicates.py`, following
`tests/doctor/test_checks_external_pr_mirror.py`'s fixture style: `OK` on a clean
projects root, `WARN` with populated `data` on a corrupted one, `SKIP` when the projects
root is absent, and project rows labelled with the display name rather than the
ProjectSpec key.

Rust: the two `parser.rs` unit tests from step 6 plus the extended parity test.

## Step 10 — verification

```bash
just install       # ephemeral workspaces need this before anything else
just check-full    # this change touches the parser and the doctor surface
just rust-check    # not covered by check/check-full
```

Then repair the host machine's real archive, which is the epic's own acceptance
criterion. **This mutates `~/.sase` state, so back up first**, and note that other SASE
agents may be running against this project — `apply_duplicate_block_repairs` takes
`patch_lock`, so the write is safe, but the backup is what makes it reversible:

```bash
ARCHIVE=~/.sase/projects/gh_sase-org__sase/gh_sase-org__sase-archive.sase
cp "$ARCHIVE" "$ARCHIVE.bak-spec-repair"
sase doctor -C project.duplicate_patch_blocks -v     # preview the warning
sase doctor -C project.duplicate_patch_blocks -F     # confirm and apply
```

Expected afterwards:

- `grep -c '^NAME: ' "$ARCHIVE"` equals `grep '^NAME: ' "$ARCHIVE" | sort -u | wc -l`
  equals **289**.
- `python -c "from sase.ace.patch.parser import parse_project_file as p; import sys; print(len(p(sys.argv[1])))" "$ARCHIVE"`
  prints **289** — same count as unique names, which is the property the bug broke.
- The file is on the order of megabytes, not tens of megabytes.
- `sase doctor -C project.duplicate_patch_blocks` reports `OK`.
- `sase patch sync-external --project sase --dry-run` run twice plans zero creations for
  PRs already adopted, including the release-please ones. (They are still adopted at
  this point; the epic's `filters` phase is what stops mirroring them.)

Remove the `.bak-spec-repair` copy only after the counts above check out, and report the
before/after numbers.

## Out of scope

- The default PR head-ref filter that drops release-please and release-plz branches —
  epic phase `filters`.
- Moving the mirror chops into their own lumberjack — epic phase `lane`.
- Refreshing an adopted Patch when its PR merges — epic phase `patch_status`.
- Removing the per-mutation full re-parse in `pr_sync.py` / `importer.py` — epic phase
  `perf`.
- Bumping the `sase-core-rs` floor in `pyproject.toml`.

## Handoff notes

- Changes land in two working trees: this repo, and the linked `sase-core` checkout
  resolved through `/sase_repo`. Do not commit either unless explicitly asked or a
  post-completion finalizer triggers; the commit finalizer already checks configured
  linked repos' `git status`. Report both diffs.
- Record any discovered follow-up work on the bead with
  `sase bead note sase-k2.1 'PROPOSED FOLLOW-UP: <one-line summary — detail>'` instead
  of filing task beads directly.
