---
tier: tale
size: medium
title: Land Footprint export/import on the sase side and close epic sase-1p1
goal: "Finish the last missing piece of the Footprint epic sase-1p1: phase sase-1p1.12
  (identity-free export packs and multi-machine merge) closed, but only its sase-core
  half landed. Add the sase facade, the `sase footprint export` and `import` commands,
  the merged-figure note, tests, docs, and the CI pin move. Retire the epic's last two
  Symvision exemptions, fix the epic's flaky catalog-match test, and close the epic.

  "
proposed_by: bbugyi200.athena.sase-1p1.land
proposed_with: claude/opus
bead: sase-1p1
create_time: 2026-10-28 12:15:50
status: done
---

- **PARENT:**
  [202610/footprint_usage_census.md](https://github.com/sase-org/sase--plans/blob/main/202610/footprint_usage_census.md)
- **BEAD:**
  [sase-1p1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1p1/README.md)

# Land Footprint export/import (sase side) and close epic sase-1p1

## Why this work exists

The land agent for epic `sase-1p1` (Footprint: a local usage census) verified every
phase against the tree. Twelve of the thirteen phases are really in master. Phase
`sase-1p1.12` (`export_import`) is closed, but only its Rust half landed:

- sase-core commit `65546fd2` ("feat(footprint): add identity-free weekly export and
  replace-all import") is on sase-core `master`. It adds `footprint/export.rs`,
  `import.rs`, `week.rs`, `query/merge.rs`, `transfer_tests.rs`, the wire types, and the
  PyO3 bindings `footprint_export` and `footprint_import`.
- **None** of the sase-side work exists on sase `master`:
  - no facade functions
  - no `sase footprint export` or `import` subcommands
  - no `cli:footprint.export` or `cli:footprint.import` catalog IDs
  - no `tests/footprint/test_export.py`
  - no "Merging machines" or "Export governance" docs
  - no pin move: `sase-core-revision.txt` is still `49870f00…`, an ancestor of
    `65546fd2`
- The phase's catalog-test isolation fix (its note #6) never landed either.

The closeout phase `sase-1p1.13` recorded the gap as its PROPOSED FOLLOW-UP #1. The
epic's plan is `plan:202610/footprint_usage_census.md`. Read its "Shared contracts",
"CLI surface", and "### export_import" sections with
`sase artifact read plan:202610/footprint_usage_census.md "<why>"` before starting. That
section is the spec. This tale only adds the concrete decisions below.

The land agent already triaged every `PROPOSED FOLLOW-UP:` and recorded the outcomes on
`sase-1p1` in a "LAND TRIAGE" note. Do **not** re-triage them or file more tasks for
them.

Before editing, read these memory notes with `/sase_memory_read`: `cli_rules.md` (new
subcommands and options), `docs_writing.md` (docs pages), `symvision.md`, and
`lint_and_test.md`. The Rust core rule applies: `sase_core` already owns export, import,
and merge, so the Python side is a thin adapter and needs no Rust change.

## What the Rust side already provides (no sase-core change needed)

Open sase-core with `sase repo open sase-core -r "<why>"` only to read these:
`crates/sase_core/src/footprint/wire.rs` and `crates/sase_core_py/src/footprint/mod.rs`.

- `footprint_export(request: dict, busy_timeout_ms: int) -> list[dict]`
  - The request is `ExportRequestWire`:
    `{store_path, today, weeks: int|None, sase: "<major>.<minor>", os: "linux"|"darwin"}`.
  - It returns one `PackWire` per complete ISO week, with exactly these keys:
    `{schema: "sase.footprint.export/1", nonce, week: "YYYY-Www", sase, os, observed: {surface: days}, features: [{id, actor, days, n}], not_measured: [surface]}`.
  - `n` is bucketed as `1 | 2-5 | 6-20 | 21+`.
  - Packs hold built-in IDs only: no `~custom`, config leaves, option IDs, or plugin
    names.
  - Nonces are random for every call, by design.
- `footprint_import(store_path: str, packs: list[dict], today: str, dry_run: bool, busy_timeout_ms: int) -> dict`
  - It returns
    `ImportResultWire {schema_version, packs_imported, packs_deduped, feature_rows, dry_run}`.
  - Import is replace-all: it deletes the previously imported tables, then loads these
    packs, deduped by nonce.
  - `PackWire` uses `deny_unknown_fields`, so a malformed pack is rejected with
    `ValueError`.
- The report header (`footprint_report(...)["header"]`) already carries
  `imported_pack_count` and `imported_day`. The query engine already merges imported
  weekly data into the map, cold, fading, agent-only, and hot lenses and into card
  figures:
  - usage days per week use `min(7, local + Σ imported)`
  - observed days use `max(local, imported)`
  - dead paths, friction, and coverage stay local
- `CardWire` has no imported fields. The card note comes from the report header (see
  step 5).

## Steps

### 1. Move the CI pin and confirm the binding

- Move `sase-core-revision.txt` to sase-core `master`, which includes `65546fd2`, with
  `just ratchet-core-revision`. If the ratchet cannot pick it, set the file to
  `65546fd244ec43ee023369fdafb5b5edb6197061` or a later sase-core master SHA. See
  `docs/rust_backend.md`.
- Check the venv with
  `.venv/bin/python -c "import sase_core_rs as m; assert hasattr(m, 'footprint_export') and hasattr(m, 'footprint_import')"`.
  If the check fails, run `just install-venv`, which is long; hand it to `/sase_monitor`
  if needed.

### 2. Facade, plus retiring the two leftover epic symbols

Edit `src/sase/core/footprint_facade.py`:

- Add
  `footprint_export(request: Mapping[str, Any], *, busy_timeout_ms: int = 250) -> list[dict[str, Any]]`.
  - Call `_require_wire_schema()` first.
  - Stringify `store_path`.
  - Return `[dict(p) for p in payload]`.
- Add
  `footprint_import(store_path, packs, today, *, dry_run: bool = False, busy_timeout_ms: int = 250) -> dict[str, Any]`.
- **Delete** the public `footprint_wire_schema_version()` and
  `footprint_validate_overlay()` facade functions. They have no non-test consumer.
  `_require_wire_schema()` already guards the version, and Rust validates overlay shape
  inside `footprint_report` and `footprint_card` (`query/overlay.rs` calls
  `validate_overlay_shape`). Keep the Rust bindings themselves.
- In `tests/footprint/test_binding_smoke.py`, call
  `require_rust_binding("footprint_wire_schema_version")()` and
  `require_rust_binding("footprint_validate_overlay")(...)` directly instead of the
  deleted facade functions.
- Add smoke assertions for `footprint_export` and `footprint_import`.
- In the `Justfile` `_lint-symvision` recipe:
  - delete `--epic-symbol 'sase-1p1(footprint_wire_schema_version)'` and
    `--epic-symbol 'sase-1p1(footprint_validate_overlay)'`
  - delete the `# sase-1p1 (Footprint usage census) owns unused Rust facade exports …`
    comment paragraph above the recipe, which ends "Drop each remaining row when its
    caller lands."
  - Never put a comment line inside the continued command.
- Add `"footprint_export"` and `"footprint_import"` to the footprint binding list in
  `tools/validate_sase_core_rs`, next to `footprint_validate_overlay`. Add them to the
  expected set in
  `tests/test_validate_sase_core_rs_runtime_tool.py::test_validate_sase_core_rs_requires_footprint_bindings`.
- Add `footprint_export` and `footprint_import` to the Footprint binding list in
  `docs/contribute/rust-backend.md`, around lines 123–128.

### 3. Export and import logic (`src/sase/footprint/transfer.py`, new)

Keep the logic off the hot path; only the handler imports it. Use one module for both
directions:

- `build_export_request(weeks: int | None) -> dict`
  - `store_path` is `paths.ledger_path()`.
  - `today` is the local ISO date, the same helper the other footprint commands use.
  - `sase` is the major.minor of `sase.__version__`.
  - `os` is `"linux"` when `sys.platform` starts with `linux`, and `"darwin"` for
    `darwin`.
  - On any other platform, raise
    `ValueError("Footprint export supports linux and darwin only")`.
- `render_packs(packs) -> bytes`
  - JSON Lines: one pack per line,
    `json.dumps(pack, sort_keys=True, separators=(",", ":")) + "\n"`.
  - This one function produces both the preview bytes and the file bytes, so they are
    byte-identical for the same packs.
- `read_pack_files(paths) -> list[dict]`
  - Parse every non-blank line of every file as a JSON object.
  - On bad JSON or a non-object line, raise `ValueError` naming `file:line`.
  - Nothing is imported unless every file parses.

### 4. CLI: `sase footprint export` and `import`

Per the epic's CLI surface table, follow `cli_rules.md`.

- In `src/sase/main/parser_footprint.py`, add the subparsers alphabetically, and add
  `export` and `import` to the subcommand `metavar` choices string at about line 55:
  - `export`: `-o/--output FILE` and `-p/--preview` in a required mutually exclusive
    group, plus `-w/--weeks N` (int ≥ 1, default all complete weeks).
    - Help: the pack is weekly and bucketed and carries no identity.
    - `--preview` prints exactly the bytes `--output` would write. Nonces are fresh on
      every run.
  - `import`: positional `FILE` (`nargs="+"`, path completion like backfill's
    `-p/--path`) and `-n/--dry-run`.
    - Help: import is replace-all, so re-importing never double counts.
  - Give both a description and an examples epilog like the sibling subcommands.
- Dispatch from `handle_footprint_command` in `src/sase/main/footprint_handler.py`. It
  is already 652 lines, so put the handlers in a new
  `src/sase/main/footprint_transfer_handler.py`.
  - **export**:
    - Run `fold_if_stale()` first, guarded like `_handle_show`.
    - Call `footprint_export`.
    - With zero packs, print "No complete ISO weeks to export yet." and exit 0 without
      creating a file.
    - With `-p`, write `render_packs(...)` to `sys.stdout.buffer`.
    - With `-o FILE`, write the bytes atomically with mode 0600 and print a summary
      (`N packs · weeks W1–Wn · B bytes → FILE`).
    - Catch `OSError`, `RuntimeError`, `TimeoutError`, and `ValueError`, then print
      `sase footprint export: <msg>` to stderr and exit 1.
  - **import**:
    - Call `read_pack_files`.
    - Call `footprint_import(ledger_path(), packs, today, dry_run=args.dry_run)`.
    - Print `Imported N pack(s) (D deduped, R feature rows)`, or `Would import …` with
      `-n`.
    - Note that re-importing replaces the previous imports.
    - Errors follow the export rule: stderr and exit 1.
- Run `just fix`. It runs `docs-gen` (CLI reference `docs/reference/cli/footprint.md`)
  and `sync-footprint-catalog`, which adds `cli:footprint.export`,
  `cli:footprint.import`, and their option IDs to `src/sase/footprint/catalog.jsonl`.
- If `tests/completion/test_snapshot.py` drifts, regenerate the snapshot with
  `tools/sync_completion_spec` (see its `--help`).

### 5. Merged-figure note ("+N imported packs")

Add the note in the shared render layer, so the CLI and the TUI pick it up together:

- `src/sase/footprint/render/_report.py`: the header trust line.
- `src/sase/footprint/render/_markdown.py`: the header line and `card_markdown`.
- The card renderers: `_print_card_rich(card, header)` in `footprint_handler.py`, and
  the TUI evidence card panel.

When `header["imported_pack_count"] > 0`, append
`· +N imported pack(s) (imported YYYY-MM-DD)` using `imported_day`, in the muted style.
When the count is zero or missing, output must stay byte-for-byte unchanged, so no PNG
goldens churn.

CLI `show` already fetches a map-report header; reuse it. For the TUI card, pass the
view's current report header into the card render if the panel does not already have it.
If any rendered TUI output changes for the zero-import case, run
`just fix-tui-screenshots` through `/sase_monitor` and inspect the report per
`lint_and_test.md`.

### 6. Make the catalog-match test immune to in-process pollution

Under xdist or the scoped lane,
`tests/footprint/test_catalog.py::test_checked_in_catalog_matches_generator` has failed
with one extra generated leaf (3596 vs 3597 and 3652 vs 3653) that a clean run does not
produce. In-process parser state leaks from other tests.

Make this test generate in a fresh subprocess. Run
`subprocess.run([sys.executable, str(REPO_ROOT / "tools" / "sync_footprint_catalog"), "--check"], capture_output=True, text=True)`
and assert `returncode == 0`, with stderr in the failure message. This also covers
header count and rev. Keep the other catalog tests in-process. Once nothing else uses
the shared `generated` fixture, drop it.

### 7. Tests

- **`tests/footprint/test_export.py` (new):**
  - **Preview bytes:** monkeypatch the facade's `footprint_export` to return fixed
    packs. Then `export -p` stdout bytes equal the bytes `export -o FILE` writes. The
    file mode is 0600.
  - **Real-binding round trip:**
    - Build a sandboxed `SASE_HOME` ledger through the facade with at least one complete
      past ISO week of spool lines (follow `tests/footprint/test_report.py` and
      `test_closeout_smoke.py` fixtures). Include a `~custom` macro, a config override,
      an option `#dest`, and a plugin hit.
    - Export and assert that every pack's keys are exactly the `PackWire` set, every `n`
      is in the bucket set, and every `os` and `sase` matches.
    - Assert that no feature ID contains `~`, `#`, `config:`, or `plugin:`.
    - Assert that the bytes contain no sandbox path, username, or hostname.
  - **Import:**
    - Import the file into a second sandbox. The report header then shows
      `imported_pack_count == N` and a non-null `imported_day`.
    - Re-importing the same file leaves the count at N (replace-all).
    - `-n` changes nothing.
    - A file with a bad JSON line exits 1 naming `file:line` and imports nothing.
  - **Note rendering:** the header and card note appears only when N > 0, in rich and
    markdown.
- **`tests/footprint/test_closeout_smoke.py`:** extend the end-to-end smoke with
  `export --preview` and `import`, as the epic plan's closeout step 2 requires. Assert
  the preview bytes contain no forbidden token.
- **`tests/footprint/test_binding_smoke.py`:** the updates from step 2.

### 8. Docs

Respect the docs budgets; `just fix` runs `fix-docs-budgets`.

- `docs/reference/footprint.md`: add a "Merging machines" section covering:
  - `sase footprint export -o week.jsonl`, copying the file, then
    `sase footprint import week.jsonl` on the other machine
  - `--preview` shows the exact bytes
  - packs are weekly, bucketed, built-in IDs only, with no identity
  - import is replace-all, so re-importing never double counts
  - the merge bounds: usage days `min(7, local + imported)` per week (errs toward
    keeping) and observed days `max(local, imported)` (keeps the gate conservative)
  - dead paths, friction, and coverage stay local
  - the "+N imported packs" note
- `docs/contribute/footprint.md`: add an "Export governance" section covering:
  - the exact exported field list and the excluded categories
  - any change to exported fields requires a new decision record
  - a server can never enable fields or modes

### 9. Verify

- Run `just fix`, then `sase tool run check`. Never run `just check-full`.
- Two KNOWN unrelated master reds are out of scope:
  - the Symvision `_RecoveryAssessment` private import in
    `src/sase/finalizers/_failed_close_recovery_assess.py`, tracked by `sase-1p3`
  - `tests/test_agent_artifact_directory_operation_audit.py::test_artifact_directory_operation_sites_are_reviewed`,
    tracked by `sase-1p5`
- Anything else red that this change touched is yours.
- Bead and artifact commands on this host can take over 2 minutes. Give them generous
  timeouts and never retry a mutation blindly; read the bead first.

### 10. Close out epic sase-1p1 (final step, same turn as the code)

Close the epic in this turn. Do not wait for this work's commit, push, or CI.

1. Run `sase bead epic-symbols sase-1p1`. After step 2 it must list no entries. If any
   entry remains, resolve it with the `symvision.md` hierarchy: wire it up, privatize
   it, add a non-test pragma, or delete it. No open later bead of this epic exists, so
   re-keying is not an option.
2. Close the epic with a note that summarizes the land agent's verification and this
   tale's work:

   ```bash
   sase bead close sase-1p1 --note "Landed: phases .1-.11 and .13 verified in tree by the land agent (catalog/ids/overlay/drift guard, Rust ledger+fold, CLI recorder incl. post-split entry.py/_dispatch_* fast paths, TUI seams+flusher, fold/status/clear/AXE job/disclosure, query engine, derived sources+zsh backfill, render+list/show/decide, view 09, evidence card+finder, decide/finish-review+#footprint_review, docs+closeout smoke); integration reviewed for the 24 non-epic commits since a5207375e7 (entry split 3d52c6ce74 keeps recording; statistics/catalog_gen splits are footprint-internal). Remaining .12 sase side finished here: facade footprint_export/import, sase footprint export/import, catalog IDs, +N imported packs note, test_export.py + smoke, Merging machines/Export governance docs, sase-core pin moved past 65546fd2; retired epic symbols footprint_wire_schema_version/footprint_validate_overlay (deleted, Rust validates overlay in queries); catalog-match test runs in a subprocess. Follow-up triage recorded in the LAND TRIAGE note (sase-1ic note, +1 sase-1p3/sase-1oy/sase-xo, new sase-1p5/1p6/1p7/1p8). sase tool run check: <verdict and run id>."
   ```

   Fill in the real check verdict and run ID. Never use `--force` merely to make the
   close succeed. If the close is refused for leftover `--epic-symbol` entries, finish
   that cleanup and close again.

3. Run `just symvision`. It must report no `--epic-symbol` errors. The only acceptable
   remaining error is the unrelated `_RecoveryAssessment` private import (`sase-1p3`);
   mention it in your final response.
4. Set `status: done` in the frontmatter of the epic's plan file. Use the PLAN path that
   `sase bead read sase-1p1 -r "Need the plan path for closeout"` prints
   (`plan:202610/footprint_usage_census.md`, under the plans sidecar's `202610/`
   directory). Change only `status: wip` → `status: done`.
5. `sase-1p1` has no `parent_bead`, so no ancestor needs handling. Finish normally.
