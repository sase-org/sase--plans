---
tier: tale
size: medium
goal: 'sase-core records every failed stage''s output as durable, versioned failure
  items: pure extractors with normalized, workspace-independent signatures (golden-tested
  on real athena logs), additive cascade-safe triage tables with early fingerprint_before
  persistence and retention, and the tool_run_triage_extract/record/show bindings,
  with the wire schema still 1 and the sase repo untouched.'
title: 'E3 core-failure-items: triage tables, extractors, normalization, and extract/record/show
  bindings in sase-core'
proposed_by: bbugyi200.athena.sase-18j.2
bead: sase-18j.2
status: done
---

- **PARENT:**
  [202609/tool_e3_failure_triage.md](https://github.com/sase-org/sase--plans/blob/main/202609/tool_e3_failure_triage.md)
- **BEAD:**
  [sase-18j.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-18j/sase-18j.2.md)

# Plan: sase-18j.2 — durable failure items, extractors, and normalization (sase-core only)

This is phase `core-failure-items` of epic `sase-18j`
(`plan:202609/tool_e3_failure_triage.md`, sections "Binding contracts", "Durable
additions", "Extractors and normalization", "Rust bindings", and "2.
core-failure-items"). Everything below lands in the linked **`sase-core`** repo only.
Open it with `sase repo open sase-core -r "<why>"`, read its `AGENTS.md`, and work only
in the printed path. **Touch nothing in the sase repo** (epic decision 10): no pin move,
no adapters, no validator entries.

Classification (NEW/KNOWN/FLAKY/UNKNOWN), `failure_kind`, verdict, REPEAT, owners,
`tool_run_triage_stage`/`settle`/`failures`, and the show owner selector all belong to
the next phase (`sase-18j.3`). This phase only produces and stores **facts**, plus
nullable label columns that the next phase fills.

## Verified starting facts

- `sase-core` `master` = `origin/master` = `20ac645`. The ToolRun ledger lives in
  `crates/sase_core/src/tool_run/` (`wire.rs`, `handoff_wire.rs`,
  `store/{connection, lifecycle,query,retention,handoff,reconcile}.rs`,
  `store/tests/{compat,handoff, reconcile_owner}.rs`).
- `SCHEMA_SQL` in `store/connection.rs` runs on every write open and starts with
  `PRAGMA foreign_keys = ON`; read opens also set `foreign_keys = ON` and never create
  tables. `TOOL_RUN_WIRE_SCHEMA_VERSION` = 1 and must stay 1.
- `store/*` helpers (`with_write_store`, `with_read_store`, `validate_schema`,
  `touch_write_meta`, `unix_now`, `load_run`) are `pub(super)` inside `store/`, so the
  store-backed triage operations go in `store/triage.rs`, not under `triage/`.
- `ToolRunObserveRequestWire` derives `Eq`; `ToolFingerprintWire` is only `PartialEq`.
- Python bindings for `tool_run_*` live in `crates/sase_core_py/src/telemetry/mod.rs`
  (616 lines) with the round-trip test in `telemetry/tests.rs`
  (`tool_run_bindings_round_trip_python_dicts`).
- sase's Python reaper (`src/sase/core/disk_footprint_reap_tool_run.py`) deletes any
  `file_candidates[].path` that is a regular file under the tools root, regardless of
  `kind`, and skips directories. A new candidate kind `stage_output` is therefore safe.
- sase's `tools/check_sase_core_rs_bindings` only checks _required_ names, so adding
  bindings to core cannot redden sase.
- `regex`, `sha2`, `hex`, `serde_json`, `tempfile` are already `sase_core` deps; regexes
  are compiled once with `std::sync::OnceLock` (see `suffix.rs`). No `macro_rules!`.
- Real failure shapes, sampled from `~/.sase/tools/logs/<run>/stdout.log` on athena
  (`✗ <stage>` markers): `lint (symvision)` 229, `lint (mypy)` 106, `test (scoped)` 44,
  `fmt (python)` 19, `lint (pyscripts)` 15, `lint (feature flags)` 12, `lint (toobig)`
  5, `fmt (markdown)` 5, `SASE validation` 4, `lint (test waits)` 2, `lint (ruff)` 1.
  sase-core `check` runs (no stage markers) hold cargo-test failures. No retained log
  has a failing keep-sorted stage, so its fixture comes from running the pinned
  `keep-sorted --mode lint` on a crafted out-of-order YAML file (the real tool's
  output).

## 1. Module layout

New pure module `crates/sase_core/src/tool_run/triage/`:

| File                         | Owns                                                                                       |
| ---------------------------- | ------------------------------------------------------------------------------------------ |
| `mod.rs`                     | facade: `mod` + `pub use` lines only                                                       |
| `wire.rs`                    | all triage wires and enums (below)                                                         |
| `normalize.rs`               | line normalization, path-root stripping, display redaction and bound                       |
| `extract.rs`                 | registry, signature digest, per-stage collapse, `extract()`, `compare_triage_signatures()` |
| `extractors/mod.rs`          | facade                                                                                     |
| `extractors/lint.rs`         | symvision, mypy, ruff, ruff_format, prettier, keep_sorted, toobig                          |
| `extractors/tests_output.rs` | pytest, cargo_test                                                                         |
| `extractors/environment.rs`  | `_setup` environment markers                                                               |
| `extractors/generic.rs`      | generic fallback                                                                           |
| `tests/` (`mod.rs`, …)       | unit + golden tests                                                                        |
| `fixtures/<case>/`           | golden cases (see §6)                                                                      |

Store-backed operations: new `crates/sase_core/src/tool_run/store/triage.rs` (record,
show, table probe) with tests in `store/tests/triage.rs`; schema in `connection.rs`;
retention in `retention.rs`. Every file stays ≤ 1,500 lines. In `tool_run/mod.rs` add
`mod triage;`, `pub use triage::*;` (wires, `extract_triage_items`,
`compare_triage_signatures`), and extend the `pub use store::{…}` line with
`triage_record, triage_show`. Do not touch `lib.rs`'s root `pub use` list or
`sase_core_py/src/prelude.rs`.

## 2. Wires (`triage/wire.rs`)

Request wires are `#[serde(deny_unknown_fields)]`; result wires are lenient. Every
request/result/item/label/stage/run-facts object carries `schema_version: u32` (serde
default = 1, validated = 1 on requests via the existing `validate_schema`). Constants:
`TOOL_RUN_TRIAGE_DISPLAY_MAX_CHARS = 512`, `TOOL_RUN_TRIAGE_STAGE_KEY_RUN_OUTPUT = "*"`.

Enums (serde `snake_case`, each with `as_str`/`from_db` like `ToolRunStateWire`):

- `ToolRunTriageExtractionStatusWire`: `parsed`, `generic`, `output_missing`,
  `output_truncated`.
- `ToolRunTriageClassWire`: `new`, `known`, `flaky`, `unknown`.
- `ToolRunTriageContinuationModeWire`: `never`, `always`, `known`.
- `ToolRunTriageDecisionKindWire`: `continue`, `stop`.

Structs:

- `ToolRunTriageItemWire` — `schema_version`, `item_id: Option<String>` (None from
  extract; set by the store), `stage_key`, `stage_id: Option<String>`, `extractor`,
  `extractor_version: u32`, `signature` (64 lowercase hex), `display`,
  `locator_paths: Vec<String>` (repo-relative, sorted, deduped), `occurrences: u32`,
  `label: Option<ToolRunTriageLabelWire>`.
- `ToolRunTriageLabelWire` — `class`, `touched: Option<bool>`, `rule_version: u32`,
  `knobs: serde_json::Value`, `evidence: serde_json::Value`,
  `possible_owners: serde_json::Value` (default `[]`), `classified_ts: i64`. The
  JSON-valued fields stay opaque here; phase `core-classification` defines their shapes.
- `ToolRunTriageDecisionWire` — `mode`, `decision`, `reason: String`,
  `elapsed_ms: Option<i64>`, `decided_ts: i64`.
- `ToolRunTriageStageFactsWire` — `stage_key`, `stage_id: Option<String>`,
  `extraction_status`, `output_path: Option<String>`,
  `decision: Option<ToolRunTriageDecisionWire>`, `created_ts: Option<i64>` (set by the
  store on show).
- `ToolRunTriageRunFactsWire` — `continuation_mode: Option<mode>`, `recipe_finished_ts`,
  `first_continued_exit_code: Option<i32>`, `continuation_extra_ms`,
  `repeat_of_run_id: Option<String>`, `triaged_ts`, `diagnostics: Vec<String>` (all
  optional/defaulted).
- `ToolRunTriageExtractRequestWire` — `schema_version`, `stage_key: String`,
  `stage_id: Option<String>`, `output: Option<String>`, `truncated: bool` (default
  false), `project_root: Option<String>` (catalog project root to strip),
  `workspace_roots: Vec<String>` (default empty; extra roots to strip).
- `ToolRunTriageExtractResultWire` — `schema_version`, `stage_key`, `stage_id`,
  `status`, `items: Vec<ToolRunTriageItemWire>`, `diagnostics`.
- `ToolRunTriageRecordRequestWire` — `schema_version`, `run_id`,
  `stages: Vec<ToolRunTriageStageRecordWire>`,
  `run_facts: Option<ToolRunTriageRunFactsWire>`, `now_ts: Option<i64>`.
  `ToolRunTriageStageRecordWire` = stage facts (`stage_key`, `stage_id`,
  `extraction_status`, `output_path`, `decision`) + `items: Vec<ToolRunTriageItemWire>`.
- `ToolRunTriageRecordResultWire` — `schema_version`, `run_id`,
  `refused: Option<ToolRunTriageRefusalWire>`, `items_inserted`, `items_existing`,
  `labels_written`, `labels_kept`, `stages_inserted`, `decisions_written`,
  `diagnostics`. `ToolRunTriageRefusalWire` (snake_case): `run_not_found`, `ad_hoc_run`
  — typed values, not errors.
- `ToolRunTriageShowRequestWire` — `schema_version`, `run_id`.
- `ToolRunTriageShowResultWire` — `schema_version`, `run_id`, `run_found: bool`,
  `triaged: bool` (a `tool_triage_runs` row with `triaged_ts` exists),
  `run_facts: Option<…>`, `stages: Vec<ToolRunTriageStageFactsWire>`,
  `items: Vec<ToolRunTriageItemWire>`, `diagnostics`.

Add JSON fixtures `tool_run/fixtures/triage_extract_request.json`,
`triage_record_request.json`, `triage_show_result.json` with round-trip tests (parse →
serialize → equal JSON), plus a test that each request wire rejects an unknown field.

## 3. Normalization (`triage/normalize.rs`)

`normalize_line(line, roots) -> String`, applied to every output line before any
extractor sees it, in this order:

1. Strip ANSI/terminal control: CSI (`\x1b\[[0-9;?]*[ -/]*[@-~]`), OSC
   (`\x1b\][^\x07\x1b]*(\x07|\x1b\\)`), other `\x1b.` pairs, and C0 controls except tab;
   keep only the text after the last `\r` in a line.
2. Strip roots (longest first): each request root (`project_root`, `workspace_roots`,
   trailing `/` ensured); `\S*/sase/repos/linked/[^/\s]+/`;
   `\S*/workspaces/[^/\s]+/[^/\s]+/[^/\s]+_\d+/`; `\S*/sase_\d+/`. All replace with ``
   so paths become repo-relative.
3. Cargo target dirs:
   `\S*?/(?:cargo-targets/[^/\s]+|target)/(?:[^/\s]+/)*?(debug|release)/` →
   `<target>/$1/` (keeps `deps/<crate>-…` for the cargo extractor).
4. `/tmp/\S*` → `<tmp>`.
5. Timestamps: ISO-8601 date-times → `<ts>`; `\b\d{2}:\d{2}:\d{2}(\.\d+)?\b` → `<time>`.
6. Durations: `\b\d+(\.\d+)?(ns|µs|us|ms|s|m|h)\b` → `<dur>`; pytest `(H:MM:SS)` →
   `(<dur>)`.
7. PIDs: `thread '(.*)' \(\d+\)` → `thread '$1' (<pid>)`; `\bpid[ =:]\d+` → `pid=<pid>`.
8. Hex ids: `\b0x[0-9a-fA-F]+\b` and `\b[0-9a-f]{12,}\b` containing a digit → `<hex>`.

Line and column numbers are **not** stripped globally; each non-positional extractor
drops them when building its key (all extractors in this phase are non-positional).

`display_text(raw) -> String`: normalize, then redact and bound:

- secret-shaped tokens → `<redacted>`:
  `(?i)(token|secret|password|passwd|api[_-]?key|authorization|bearer)\s*[:=]\s*\S+`,
  `ghp_|gho_|ghs_|github_pat_[A-Za-z0-9_]{20,}`, `sk-[A-Za-z0-9_-]{20,}`,
  `xox[abprs]-\S+`, `AKIA[0-9A-Z]{16}`;
- environment assignments `\b[A-Z][A-Z0-9_]{2,}=\S+` → `NAME=<redacted>`;
- any remaining absolute path token (`/` + ≥ 2 components) → `<path>/<last component>`;
- collapse whitespace; bound to 512 **chars** on a char boundary, ending in `…` when
  cut.

Locator paths: strip a leading `./`; drop any that is still absolute (diagnostic).

## 4. Extractors and signatures (`triage/extract.rs`, `triage/extractors/`)

Registry: a `static` slice of
`ExtractorSpec { name: &'static str, version: u32, specific: bool, extract: fn(&[String]) -> Vec<RawItem> }`
(normalized lines in, raw items out; `RawItem { key, display, locator_paths }`). All
versions start at 1.

`signature = sha256(canonical_json([extractor, version, key]))` via
`canonical::canonical_digest`. Stage-independent by construction.

`extract_triage_items(request) -> Result<ToolRunTriageExtractResultWire, ToolRunError>`:

1. `validate_schema`; empty `stage_key` → `Invalid`.
2. `output` None or whitespace-only → status `output_missing`, no items.
3. Normalize lines; run every specific extractor; if none produced items, run `generic`.
   Collapse identical `(extractor, signature)` within the stage into one item: first
   display wins, `occurrences` summed, locator paths unioned/sorted.
4. Status: `output_missing` > `output_truncated` (when `truncated`; items still
   extracted, diagnostic `output truncated; items may be incomplete` — the next phase
   must treat it as not-proven) > `parsed` (a specific extractor matched) > `generic`.
5. Items keep first-appearance order (registry order, then line order).

`compare_triage_signatures(a, b) -> Result<bool, ToolRunError>`: same extractor with
different `extractor_version` →
`Err(Invalid("… cross-version signatures are not comparable"))`; different extractor →
`Ok(false)`; else `Ok(a.signature == b.signature)`.

Extractors (key; locator paths):

| name          | Detection (normalized lines)                                                                                                                                                                                                                                                                                                                                                                                                                               | Key                                                                            | Locators  |
| ------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------ | --------- |
| `symvision`   | a non-indented heading ending in `:` containing `functions/classes`, followed by `^\s{2,}(\S+) in (\S+)$` lines; category slug `unused_public` / `private_imported` / `private_unused`, else heading text lowercased with digits → `N`. Also `^Error: --epic-symbol '([^'(]+)\(([^)]+)\)': (.*)$` → category `epic_symbol_closed` ("is closed"), `epic_symbol_unneeded` ("already properly used"), else `epic_symbol`; symbol `bead(sym)`, path `Justfile` | `category\|symbol\|path`                                                       | path      |
| `mypy`        | `^(\S+\.pyi?):\d+(?::\d+)?: error: (.*?)\s+\[([a-z0-9-]+)\]$` (notes ignored)                                                                                                                                                                                                                                                                                                                                                                              | `path\|code\|message` with `"…"`→`"_"` and `'…'`→`'_'`                         | path      |
| `ruff`        | full format: `^([A-Z]{1,4}\d{3,4}) .+` followed within 3 lines by `^\s*--> (\S+?):\d+:\d+$`; concise: `^(\S+\.pyi?):\d+:\d+: ([A-Z]{1,4}\d{3,4}) `                                                                                                                                                                                                                                                                                                         | `rule\|path`                                                                   | path      |
| `ruff_format` | `^unformatted: File would be reformatted$` followed within 3 lines by `--> path:l:c`; or `^Would reformat: (\S+)$`                                                                                                                                                                                                                                                                                                                                         | `format\|path`                                                                 | path      |
| `prettier`    | `^\[warn\] (\S+)$` excluding `Code style issues…`                                                                                                                                                                                                                                                                                                                                                                                                          | `prettier\|path`                                                               | path      |
| `keep_sorted` | a JSON array block (line `[` … line `]`) that parses as objects with `path` and `message`                                                                                                                                                                                                                                                                                                                                                                  | `message with digits → N\|path`                                                | path      |
| `toobig`      | `^ERROR: VIOLATION: (\S+) has \d+ lines`                                                                                                                                                                                                                                                                                                                                                                                                                   | `path`                                                                         | path      |
| `pytest`      | `^(FAILED\|ERROR) (\S+\.py(?:::\S+)?)(?: - .*)?$` (so `ERROR    sase.x:app.py:375 …` log lines never match)                                                                                                                                                                                                                                                                                                                                                | node id with a trailing `[…]` parametrization removed                          | test file |
| `cargo_test`  | `^test (\S+) \.\.\. FAILED$`, crate from the latest `Running … \(.*deps/([A-Za-z0-9_]+)-` line (fallback `-p <crate>` from `error: test failed, to rerun pass`, else `unknown`)                                                                                                                                                                                                                                                                            | `crate::test::path`                                                            | none      |
| `environment` | `missing_binding` (`missing required binding(s)`, `does not expose binding`), `core_import` (`cannot import sase_core_rs`, `No module named 'sase_core_rs'`), `stale_core` (`[setup] ERROR: the sase-core checkout is behind`), `core_wheel` (`SASE_CORE_WHEEL does not name a wheel file`), `required_plugins` (`[setup] error: could not read plugins.required`), `keep_sorted_missing` (`error: keep-sorted is required`)                               | marker kind                                                                    | none      |
| `generic`     | only when no specific extractor matched                                                                                                                                                                                                                                                                                                                                                                                                                    | `stage_key` + sha256 of the last 40 non-empty lines with every digit run → `N` | none      |

Environment display is `environment: <kind> — run <remedy>` + the marker line (remedy
`just install` for `missing_binding`/`core_import`/`core_wheel`/`keep_sorted_missing`,
`sase update` for `stale_core`/`required_plugins`), so the remedy hint survives in the
stored display without a new column.

Stages the epic table does not cover (`lint (pyscripts)`, `lint (feature flags)`,
`lint (test waits)`, `SASE validation`, changelog/terminology) intentionally fall to
`generic` in v1; record that as a `PROPOSED FOLLOW-UP:` note (§9).

## 5. Store (`connection.rs`, `store/triage.rs`)

Append to `SCHEMA_SQL` (all `IF NOT EXISTS`):

```sql
CREATE TABLE IF NOT EXISTS tool_triage_items (
    item_id TEXT PRIMARY KEY,
    run_id TEXT NOT NULL REFERENCES runs(run_id) ON DELETE CASCADE,
    stage_id TEXT, stage_key TEXT NOT NULL,
    extractor TEXT NOT NULL, extractor_version INTEGER NOT NULL,
    signature TEXT NOT NULL, display TEXT NOT NULL,
    locator_paths_json TEXT NOT NULL, occurrences INTEGER NOT NULL,
    created_ts INTEGER NOT NULL,
    class TEXT, touched INTEGER, rule_version INTEGER, knobs_json TEXT,
    evidence_json TEXT, possible_owners_json TEXT, classified_ts INTEGER);
CREATE TABLE IF NOT EXISTS tool_triage_stages (
    run_id TEXT NOT NULL REFERENCES runs(run_id) ON DELETE CASCADE,
    stage_key TEXT NOT NULL, stage_id TEXT,
    extraction_status TEXT NOT NULL, output_path TEXT, created_ts INTEGER NOT NULL,
    mode TEXT, decision TEXT, reason TEXT, elapsed_ms INTEGER, decided_ts INTEGER,
    PRIMARY KEY (run_id, stage_key));
CREATE TABLE IF NOT EXISTS tool_triage_runs (
    run_id TEXT PRIMARY KEY REFERENCES runs(run_id) ON DELETE CASCADE,
    continuation_mode TEXT, recipe_finished_ts INTEGER,
    first_continued_exit_code INTEGER, continuation_extra_ms INTEGER,
    repeat_of_run_id TEXT, triaged_ts INTEGER, created_ts INTEGER NOT NULL,
    diagnostics_json TEXT NOT NULL);
CREATE INDEX IF NOT EXISTS idx_tool_triage_items_run ON tool_triage_items(run_id, stage_key);
CREATE INDEX IF NOT EXISTS idx_tool_triage_items_signature ON tool_triage_items(signature, extractor_version);
```

`triage_tables_present(conn) -> Result<bool>` probes `sqlite_master` for all three;
every read path (show, retention preview, compat) treats missing tables as empty.

`triage_record(store_path, request, busy_timeout)` (write, immediate transaction):

- Validate: schema; non-empty `run_id`; each stage's `stage_key` non-empty and unique
  within the request; each item's `stage_key` equals its stage's; `signature` is 64
  lowercase hex; `extractor` non-empty; `extractor_version ≥ 1`; `occurrences ≥ 1`; no
  absolute locator path; `display` is re-bounded to 512 chars (not an error). Invalid
  input → `ToolRunError::Invalid`.
- Run missing → `refused: run_not_found`; run with `tool_name` NULL →
  `refused: ad_hoc_run`; nothing written in either case.
- `item_id = sha256(canonical_json([run_id, stage_key, extractor, extractor_version, signature]))`,
  computed by core (caller-supplied `item_id` must be None or equal, else `Invalid`).
- Stages: `INSERT … ON CONFLICT DO NOTHING`; `output_path`/`stage_id` filled only when
  NULL; decision columns written only when `decision IS NULL` (first decision wins →
  `decisions_written`).
- Items: `INSERT … ON CONFLICT(item_id) DO NOTHING` (`items_inserted` /
  `items_existing`); a supplied label is written with `UPDATE … WHERE class IS NULL`
  (`labels_written` / `labels_kept`). Stored rows always win.
- Run facts: insert the row if missing; each nullable column set with
  `COALESCE(stored, new)`; diagnostics merged as an order-preserving de-duplicated
  union.
- `touch_write_meta`; commit.

`triage_show(store_path, request, busy_timeout)` (read-only; never creates tables):
missing store → `run_found: false` + diagnostic `tool run store does not exist`; missing
tables → empty with `run_found` from `runs`; else stages ordered by
`created_ts, stage_key`, items by `stage_key, created_ts, extractor, signature`, labels
rebuilt when `class` is non-NULL (unparseable class → `label: None` + diagnostic).

**Observe `fingerprint_before`.** Add `fingerprint_before: Option<ToolFingerprintWire>`
(serde default, skip if None) to `ToolRunObserveRequestWire` and drop its `Eq` derive
(fix any `Eq`-dependent use). In `lifecycle::observe`, canonicalize
(`canonicalize_tool_fingerprint`) and persist it when the stored column is NULL; when
stored and equal, no-op; when stored and different, keep the stored value and add the
diagnostic
`fingerprint_before differs from the recorded value; kept the recorded value`.
`replayed` is true only when the child fields match **and** no new fingerprint was
written. In `finish`, apply the same keep-the-recorded-value rule when the column
already holds a value (an equal resend is a no-op; old flows that never observed a
fingerprint still write it at finish exactly as today). Existing finish/observe tests
must keep passing unchanged.

## 6. Retention (`store/retention.rs`)

- **Detail cut** (`detail_days`): for settled runs, delete `tool_triage_items`,
  `tool_triage_stages`, and `tool_triage_runs` rows with `created_ts < detail_cut`;
  count them in `detail_rows`.
- **Summary cut**: before deleting `runs`, explicitly delete every triage row whose run
  is being deleted (the cascade is only the older-core safety net); count those not
  already counted by the detail cut in `detail_rows`.
- Both the counts and deletes are skipped when `triage_tables_present` is false (a
  read-only preview of an old store must not fail).
- **Stage output files**: new helper `stage_output_files(events_path) -> Vec<String>`
  lists regular files (via `symlink_metadata`; the `stage_output` directory itself must
  be a real directory, not a symlink) in `<events_path parent>/stage_output/`, sorted.
  Add them as candidates with kind `stage_output` at the `log_days` cut, and include
  them in `select_aggregate_log_candidates`'s per-run bytes/files (so the aggregate cap
  and `protected_bytes` for unsettled runs account for them).

## 7. Tests

Golden extractor fixtures under `crates/sase_core/src/tool_run/triage/fixtures/<case>/`
with `request.json` (extract request minus `output`), `output.txt`, `expected.json`. A
test walks the directory (`env!("CARGO_MANIFEST_DIR")`), runs `extract_triage_items`,
and compares pretty JSON; `UPDATE_TRIAGE_GOLDENS=1` rewrites `expected.json`. Build
cases from real retained athena logs, split at `✗ <stage>` markers (the next `✓`/`✗` or
EOF ends a section), trimmed to a representative excerpt; before committing, replace
every absolute user path with a synthetic root (for example `/srv/ws/sase_34/…`) and
check the files contain nothing secret-shaped
(`grep -iE 'token|secret|password|api.?key|ghp_|sk-'`) and no `/home/`. At least one
case per extractor:

- `symvision_unused_public`, `symvision_private_imported`, `symvision_epic_symbol`,
  `mypy_attr_defined` (with repeats that collapse), `ruff_check`, `ruff_format`,
  `prettier_markdown`, `keep_sorted` (from the real tool on a crafted file), `toobig`,
  `pytest_failed` (with parametrized ids and `ERROR    logger:` lines that must not
  match), `cargo_test_failed` (sase-core run), `environment_missing_binding`,
  `generic_feature_flags` (a `lint (feature flags)` output → generic item), and
  `output_missing`, `output_truncated`.

Unit tests (`triage/tests/`):

- **Cross-workspace equality**: the same mypy, symvision, pytest, and cargo failures
  from `/srv/ws/sase_3/…`, `/srv/ws/sase_41/…`, a `…/sase/repos/linked/sase-core/…`
  root, and a caller-supplied `project_root` yield identical signatures; differing line
  numbers, durations, PIDs, timestamps, hex ids, ANSI codes, and `/tmp` paths do not
  change signatures.
- **Collisions**: different pytest node ids, symvision symbols or categories, mypy error
  codes, ruff rules, and paths yield different signatures; pytest parametrizations of
  one test collapse into one item with `occurrences` = count.
- **Display**: a 2,000-char line bounds to ≤ 512 chars ending in `…` (multi-byte safe);
  secret shapes, `NAME=value` assignments, and absolute paths are redacted.
- **Version refusal**: `compare_triage_signatures` errors on
  same-extractor/different-version and returns `false` across extractors.
- Generic runs only when no specific extractor matched; status precedence.

Store tests (`store/tests/triage.rs`): record → show round trip; idempotent replay
(second record with changed display/decision/label changes nothing; counters reflect
existing rows); label first-writer-wins; refusals `run_not_found` and `ad_hoc_run`;
validation errors; observe persists `fingerprint_before`, equal resend is a no-op, a
different value is kept-with-diagnostic at observe and at finish; show on a missing
store and on a store without triage tables.

Compat tests (extend `store/tests/compat.rs`):

- an `OLD_SCHEMA_SQL` store without triage tables: `triage_show` returns empty and the
  tables are still absent from `sqlite_master` afterwards; a `triage_record` then
  creates them and writes;
- the pre-change `OLD_RUN_COLUMNS` query, `show_run`, and `list_runs` still load a store
  that holds triage rows;
- the current (pre-change) retention `DELETE` statements, copied verbatim into the test
  as `OLD_RETENTION_DELETE_SQL`, run with `foreign_keys = ON` against a store with
  triage rows: the runs delete succeeds and the triage rows cascade away;
- `retention_preview`/`retention_apply` report and delete triage rows at the detail and
  summary cuts, list `stage_output` files as candidates at the log cut and under the
  aggregate cap, and never select an unsettled run's stage output.

## 8. Bindings (`crates/sase_core_py/src/telemetry/mod.rs`)

Model on `py_tool_run_show`:

- `tool_run_triage_extract(request)` — pure, no store path, `allow_threads`.
- `tool_run_triage_record(store_path, request, busy_timeout_ms=250)`.
- `tool_run_triage_show(store_path, request, busy_timeout_ms=250)`.

Register all three in `register_telemetry`. Extend
`tool_run_bindings_round_trip_python_dicts` (or add a sibling test in `tests.rs`):
extract a symvision output, record its items on a begun+finished named run, show them
back, and assert a replay reports `items_existing`; also pass `fingerprint_before`
through `py_tool_run_observe`.

## 9. Verification, notes, and hand-off

- Iterate with `just fast` and `just test -p sase_core tool_run` /
  `just test -p sase_core_py telemetry`; then `just fmt`.
- From inside the `sase-core` checkout, run `sase tool run check` (≈5 min; give the
  command a timeout of at least 15 minutes). If it fails, fix anything this change
  caused; a failure that reproduces identically on clean `origin/master` goes in a
  `PROPOSED FOLLOW-UP:` note instead.
- Confirm `git -C <sase checkout> status` shows no sase-repo changes from this phase.
- Run `sase bead epic-symbols sase-18j.2`; resolve any leftovers.
- Bead notes on `sase-18j.2` (`sase bead note`):
  - the pin expectation: "sase phase `bindings-and-backtest` must move
    `sase-core-revision.txt` to a pushed sase-core commit containing this phase's commit
    (host finalizer creates it) before calling `tool_run_triage_extract`,
    `tool_run_triage_record`, `tool_run_triage_show`, or sending observe
    `fingerprint_before`";
  - the wire summary for `sase-18j.3` (label JSON fields are opaque; `output_truncated`
    must never yield a continue/no_new_failures; stage key `*` constant);
  - `PROPOSED FOLLOW-UP:` specific extractors for `lint (pyscripts)`,
    `lint (feature flags)`, `lint (test waits)`, and `SASE validation` (all generic →
    UNKNOWN in v1, 33 of 442 failing stage sections on athena).
- Close with `sase bead close sase-18j.2 --note "<verified evidence>"`. Do not close the
  epic or any other bead; do not create beads.
