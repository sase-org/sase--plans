---
tier: tale
title: Add the schema-1 ToolRun receipt contract in sase-core
goal: "sase-core mints, supersedes, expires, and looks up fingerprint-bound verdict
  receipts in the existing ToolRun store, with typed refusals, retention, and PyO3
  bindings, while definition digests and schema version stay put.

  "
size: medium
proposed_by: bbugyi200.athena.sase-1ah.2
bead: sase-1ah.2
status: done
---

- **PARENT:**
  [202609/tool_e4_verified_completion.md](https://github.com/sase-org/sase--plans/blob/main/202609/tool_e4_verified_completion.md)
- **BEAD:**
  [sase-1ah.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ah/sase-1ah.2.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-1ah.2](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.sase-1ah.2.md)
- **COMMITS:**
  - [9f86897](https://github.com/sase-org/sase-core/commit/9f86897f834e9719c44f5e1669a4bd55d312b99c)
    — feat(tool-run): add schema-1 receipt contract and durable store

# Add the schema-1 ToolRun receipt contract in sase-core

## Goal

Phase `sase-1ah.2` (epic `sase-1ah`, design
`plan:202609/tool_e4_verified_completion.md`, section `rust-receipts`) lands the Rust
receipt contract. After this change, a settled named ToolRun can mint one self-contained
schema-1 receipt, a later same-fingerprint non-success can supersede it, and lookup
returns either `covered` or exactly one typed refusal. Receipt rows live in the ToolRun
SQLite file, expire by an injectable clock, and are bounded by the existing ToolRun
retention pass that `sase disk reap` already calls.

This is a tale because the table, policy wire, mint/lookup rules, retention, and PyO3
bindings are one contract. Splitting them would leave a store that later phases cannot
pin or call. Size is `medium`: the patterns already exist (additive triage tables,
`now_ts`, `telemetry` PyO3 neighbors, retention counts).

## Repository and boundaries

Open the linked core with `sase repo open sase-core` and edit only that checkout. Read
its `AGENTS.md` before editing. Domain code stays in `crates/sase_core` with no PyO3.
Bindings stay in `crates/sase_core_py`. Never run bare `cargo`; use `just`.

`sase-1ah.1` is closed and already chose the hermetic inputs this phase consumes. Bead
and flag state is not fingerprinted. `docs/tool.md` ("Fingerprinted toolchain and
external state") caps receipt TTL at 2 hours. Probes for ruff, mypy, symvision, and
prettier already make a fingerprint incomplete when they fail. Reuse that fingerprint;
do not add probes.

Later phases own the rest of E4. Leave them untouched:

- `sase-1ah.3` pins `sase-core-revision.txt`, creates the `tool_receipts` flag, and adds
  the catalog `receipt:` block.
- `sase-1ah.4` calls mint from the executor and adds `sase tool receipt`.
- `sase-1ah.5` measures repeat opportunities.
- `sase-1ah.6` gates prepared completion.
- `sase-1ah.7` documents, writes memory, and removes the flag.

Do not edit `sase/sase.yml`, `src/sase/config/tools.py`, the sase revision pin,
changelogs, or crate versions. Do not close `sase-1ah` or any ancestor. Python forwards
the whole tool mapping into `tool_run_normalize_definition`, so a `receipt:` key in a
catalog would hit today's pinned core and be rejected. The new wire field ships in
sase-core only.

## Policy wire, outside definition identity

Add an optional `receipt` field to `ToolDefinitionWire` in
`crates/sase_core/src/tool_run/wire.rs` (or a sibling receipt module that the wire
re-exports). `#[serde(default)]` and `skip_serializing_if` so catalogs without the key
still parse. `ToolDefinitionIdentity` in `catalog.rs` stays the seven fields it has
today: `name`, `argv`, `stages`, `inputs`, `env`, `args`, `fingerprint`. `description`,
`diagnostics`, and `receipt` stay out of `definition_digest`.

`ToolReceiptPolicyWire` (schema version 1, `deny_unknown_fields`):

- `accept`: list of strings. Normalize by trim, lowercase, reject unknown tokens, sort,
  and dedupe. The only tokens are `pass` and `no_new_failures`. `pass` is always
  inserted when a policy object is present, so an accept list of only `no_new_failures`
  becomes `[no_new_failures, pass]`.
- `ttl`: a string matching `^[1-9][0-9]*(s|m|h)$` whose duration is from 1 second
  through 7200 seconds inclusive. `2h` is the catalog literal later phases will write.
  Reject missing, zero, bare integers, and anything above 2 hours. There is no default
  TTL and no default policy. Omitting `receipt` leaves the field `None`.

`normalize_tool_definition` validates the policy when it is present and returns the
normalized object on the definition. A policy-only edit (accept set or TTL) must produce
the same digest as the same definition without `receipt`. The golden fixture
`crates/sase_core/src/tool_run/fixtures/definition_check.json` keeps its current digest.
Update every `ToolDefinitionWire` literal the new field breaks; existing tests stay on
`receipt: None`.

Keep `TOOL_RUN_WIRE_SCHEMA_VERSION` at 1. Add no variants to `ToolRunTriageVerdictWire`,
`ToolRunTriageFailureKindWire`, `ToolRunSourceWire`, `ToolRunSettledByWire`, or any
other existing enum.

## Receipt table

Add the table in `SCHEMA_SQL` inside `crates/sase_core/src/tool_run/store/connection.rs`
with `CREATE TABLE IF NOT EXISTS`. Do not bump the `meta.schema_version` value.
`open_write_store` already runs `SCHEMA_SQL`, so the first write creates the table on an
old file. Reads must not migrate: a missing table means zero receipts.

Columns:

- `receipt_id` TEXT PRIMARY KEY. Stable 64-hex `canonical_digest` of the source run id,
  so a retry names the same row.
- `source_run_id` TEXT NOT NULL, `UNIQUE`. Provenance only. No foreign key to `runs`, so
  summary deletion cannot cascade the proof away or fail on a dangling reference.
- Identity: `project`, `tool_name`, `definition_digest`, `extra_args_digest`,
  `fingerprint_digest`, all TEXT NOT NULL. `project` is `runs.project`, the catalog-repo
  identity.
- `verdict` TEXT NOT NULL, only `pass` or `no_new_failures`.
- `signature_refs_json` TEXT NOT NULL. JSON array of
  `{extractor, extractor_version, signature}` for KNOWN and FLAKY items. Signatures stay
  64-hex. No display text, locator paths, or output bytes.
- `proof_json` TEXT NOT NULL. Redacted fingerprint snapshot used for path diffs. See
  below. Never store argv, `private_argv`, environment values, log bytes, or
  diagnostics.
- `issue_ts`, `mint_ts`, `expiry_ts` INTEGER NOT NULL. `issue_ts` is the source run's
  `settled_ts`. `mint_ts` is the settle clock. `expiry_ts = mint_ts + ttl_seconds`.
- `policy_version` INTEGER NOT NULL. Constant `RECEIPT_POLICY_VERSION = 1` in the
  receipt module.
- `ttl_seconds` INTEGER NOT NULL and `accept_json` TEXT NOT NULL, the policy copied at
  mint.
- `status` TEXT NOT NULL: `active`, `superseded`, or `tombstone`.
- `superseded_by_run_id` TEXT, `superseded_ts` INTEGER, `explanation` TEXT, all
  nullable.

```sql
CREATE UNIQUE INDEX IF NOT EXISTS idx_tool_receipts_one_active
  ON tool_receipts(
    project, tool_name, definition_digest,
    extra_args_digest, fingerprint_digest
  )
  WHERE status = 'active';
```

Plus a non-unique lookup index on those five columns and `mint_ts`.

One active receipt per identity key. Supersede the previous active row in the same
transaction before inserting the new one.

### Redacted proof snapshot

`proof_json` holds only what a path diff needs:

- repos: `identity`, `head`, `index_tree`, and dirty entries of `path`, `status`,
  `kind`, `content_hash`
- inputs: `pattern` and matches of `path`, `content_hash`
- toolchain: probe name to `output` and `exit_code` (version text, no probe argv)
- env: probe key to the sha256 hex of the value, never the value

Drop absolute paths and any component that is empty, `.`, or `..` before writing.
`canonicalize_tool_fingerprint` already rejects physical dirty paths; reuse it on the
way in.

## Settle

New module `crates/sase_core/src/tool_run/receipt.rs` for the pure policy, eligibility,
and path-diff functions, and `crates/sase_core/src/tool_run/store/receipt.rs` for SQL.
`mod.rs` files only declare and re-export. Keep each new file at or under 1,500 lines.
No `macro_rules!`. Import `sase_core::tool_run::...` from the binding crate. Do not add
`core_*` aliases to `crates/sase_core_py/src/prelude.rs` or names to the `sase_core`
root re-export list.

Public function `receipt_settle(store_path, request, busy_timeout)`:

`ToolRunReceiptSettleRequestWire` (schema 1, `deny_unknown_fields`): `run_id`, optional
`policy` (`ToolReceiptPolicyWire`), `bypassed` (default false), optional `now_ts`. Clock
is `now_ts` when present and `unix_now()` otherwise, the same pattern as retention. No
clock trait.

One `BEGIN IMMEDIATE` transaction:

1. If a receipt row already exists for `source_run_id`, return it unchanged (`mint_ts`
   stable) and commit nothing new.
2. If this `run_id` is already `superseded_by_run_id` on the matching
   active-or-superseded row, return `superseded: true` and stop.
3. Load the run with the existing store loader. Load triage the same way `triage_show`
   does, and call `tool_run_triage_verdict` on those stored facts. Do not accept a
   caller-supplied verdict. Leave the show path's verdict inputs as they are, including
   its current `all_stages_complete` value. Forking a second verdict table is out of
   scope.
4. Apply eligibility below.
5. On an eligible mint, mark the previous `active` row for that identity key
   `superseded` (set `superseded_by_run_id` and `superseded_ts`), then insert the new
   `active` row.
6. On an ineligible terminal run that shares the full identity key with an `active`
   receipt, supersede that receipt when this run is later. Later means `settled_ts`
   greater than the receipt's `issue_ts`, or equal `settled_ts` and a lexicographically
   greater `run_id`. An earlier run must not displace a newer winner.
7. Commit. A unique-index conflict re-reads and returns the existing row for this
   `source_run_id`.

`ToolRunReceiptSettleResultWire`: `schema_version`, `run_id`, `minted`, `superseded`,
optional `receipt`, optional `reason` (why no row was written), `diagnostics`. `minted`
and a new row are false together. Ineligible runs write no receipt row.

### Eligible mint, all required

- `policy` is present (no policy, no row, including for a clean pass).
- `bypassed` is false. Raw `SASE_TOOL_BYPASS` never reaches this API; the flag is how a
  later wrapper attests a bypass. Phase 4 passes it.
- `runs.tool_name` and `runs.project` are non-empty. Empty tool name is the ad-hoc run
  `triage_settle` already calls `ad_hoc_run`.
- `definition_digest` and `extra_args_digest` are non-empty.
- `settled_by` is `wrapper`. `finish` writes that when `terminal_cause` is set.
  `reconcile` and `owner` do not mint.
- Run state is terminal.
- `mutated_input` is `Some(false)`. `None` and `true` do not mint.
- Before and after fingerprints are both present, both `completeness.complete`, and
  their `fingerprint_digest` values are equal.
- A `tool_triage_runs` row exists with `triaged_ts` set.
- Failure kind is `none` for verdict `pass`, or `verification` for verdict
  `no_new_failures`. Kind `control`, `infrastructure`, and `environment` do not mint.
- Verdict is `pass` or `no_new_failures`, and that verdict is in the normalized accept
  set. `no_new_failures` with a policy that only gained `pass` by the always-insert rule
  does not mint.
- `no_new_failures` also requires the stored verdict to be exactly that and triage to be
  present. Missing triage, `undetermined`, and `new_failures` write no row.

Store KNOWN/FLAKY signature refs from the triage items that produced the verdict. A
`pass` row stores an empty array.

A later eligible success on the same key mints a new active receipt and supersedes the
old one in that same transaction. That is the recovery-success case.

## Lookup

`receipt_lookup` is read-only. It uses `with_read_store` and does not create tables. A
missing database file or a missing `tool_receipts` table returns `no_receipt`. A locked,
unreadable, or corrupt database returns `ToolRunError` and never a covered result. A row
whose `proof_json` or signature JSON cannot be parsed is not covered.

`ToolRunReceiptLookupRequestWire` (schema 1, `deny_unknown_fields`): `project`,
`tool_name`, `definition_digest`, `extra_args_digest`, `fingerprint`
(`ToolFingerprintWire`), `accept` (the current policy's normalized list), optional
`now_ts`.

`ToolRunReceiptLookupResultWire`: `schema_version`, `outcome` (`covered` or `refused`),
optional `refusal`, `changed_paths`, `paths_truncated`, optional `receipt` (covered
only), `age_seconds` (covered only, `now_ts - mint_ts`), `reason`, `diagnostics`.

`refusal` is a snake_case enum with exactly these seven variants and no others:

1. `incomplete_fingerprint`
2. `definition_changed`
3. `fingerprint_changed`
4. `no_receipt`
5. `invalidated_by_later_run`
6. `expired`
7. `verdict_insufficient`

First match wins:

1. Canonicalize the supplied fingerprint. `complete: false` returns
   `incomplete_fingerprint` before any row is treated as proof. An absolute dirty path
   is `ToolRunError::Invalid`, the existing canonicalize error, not a refusal.
2. Receipts exist for `(project, tool_name, extra_args_digest)` and none share
   `definition_digest`: `definition_changed`.
3. Receipts exist for that definition identity and none share the current fingerprint
   digest: `fingerprint_changed`, with bounded paths from the newest such row's
   `proof_json`.
4. No row for the full key: `no_receipt`. A tombstone-only key returns `no_receipt` and
   copies `explanation` into `reason`.
5. Full key has no `active` row and has a `superseded` row: `invalidated_by_later_run`.
6. Active row with `expiry_ts <= now_ts`: `expired`. Expiry does not delete the row.
7. Active row whose `verdict` is absent from `accept`, or whose `policy_version` is not
   `1`: `verdict_insufficient`.
8. Otherwise `covered`. The public receipt carries id, source run id, identity digests,
   verdict, signature refs, issue/mint/expiry timestamps, policy version, ttl, and
   status. It omits `proof_json`.

### Changed paths

Diff the redacted snapshot against the canonical current fingerprint. Report only:

- repo-relative dirty or input paths that were added, removed, or whose `content_hash`
  or dirty `status` changed
- `toolchain:<name>` when probe output or exit code changed
- `env:<KEY>` when the value hash changed, with no value
- `head:<repo-identity>` or `index:<repo-identity>` when that field changed

Drop anything absolute, empty, `.`, or `..`. Sort, dedupe, and cap at 32. Set
`paths_truncated` when the cap cuts the list. Still return `fingerprint_changed`. Do not
spawn git or read the worktree. Lookup only diffs the snapshot the caller and the row
already hold.

## Retention and the disk surface

`sase disk reap` already calls `tool_run_retention_preview` / `tool_run_retention_apply`
and prints `summary_rows` and `detail_rows`
(`src/sase/core/disk_footprint_reap_tool_run.py` in the sase repo). Fold receipt
deletion into that count. Do not add a disk owner and do not edit the Python reaper in
this phase.

In `store/retention.rs`, same transaction as run deletion:

- Before deleting a run, if its receipt's `proof_json` is unreadable, set that receipt
  to `tombstone` with explanation `source run pruned; receipt proof unreadable`. A
  readable proof stays `active` and lookup still returns `covered` after the run row is
  gone. Lookup must not join `runs` to decide coverage.
- Delete receipt rows whose status is `superseded` or `tombstone`, or whose `expiry_ts`
  is at or before `now`, when that timestamp is older than `summary_cut`
  (`superseded_ts` for superseded rows, `expiry_ts` otherwise). An `active` row with
  `expiry_ts > now` stays, including when its source run is deleted in this pass.
- Add those deleted receipt rows to `detail_rows` in both preview and apply, using the
  same predicate. Expired rows remain until that horizon so lookup can still return
  `expired` rather than `no_receipt`. Deletion is the retention pass, not the lookup
  path.

`store_stats` adds `receipt_count` (`u64`, serde default 0). Missing table counts as 0.
The database file size already includes the table, so the existing ToolRun owner keeps
the bytes.

## PyO3

In `crates/sase_core_py/src/telemetry/mod.rs`, follow `py_tool_run_triage_settle`:

- `tool_run_receipt_settle(store_path, request, busy_timeout_ms=250)`
- `tool_run_receipt_lookup(store_path, request, busy_timeout_ms=250)`

Parse the dict into the request wire, call the core function, map `ToolRunError` to
`PyRuntimeError`, return `serialize_to_py`. Register both in `register_telemetry`. A
missed `add_function` only fails at runtime. Add a compact round-trip in
`telemetry/tests.rs` that mints through the binding and looks the receipt back up. Stay
under 1,500 lines in that file.

## Tests

Put store tests in `crates/sase_core/src/tool_run/store/tests/receipt.rs` and
`mod receipt;` next to the other store test modules. Drive `begin`, `finish`, and
`triage_settle` for the happy path. Pass `now_ts` everywhere so expiry does not depend
on the wall clock.

Cover:

- Policy normalization: unknown accept token, `2h` accepted, `3h` and `0s` rejected,
  omitted policy stays `None`, `pass` inserted, digest unchanged when only `receipt`
  differs, golden fixture digest unchanged.
- No row written for each ineligible case: empty tool name, `bypassed: true`,
  `mutated_input` true, `mutated_input` null, incomplete fingerprint, before digest
  different from after, `settled_by` of `reconcile` and of `owner`, kind `control` /
  `infrastructure` / `environment`, verdict `new_failures`, `undetermined`,
  `no_new_failures` without that token in `accept`, `no_new_failures` with no triage
  row, clean pass with `policy: None`.
- Eligible `pass` with a 2h policy writes one active row. Retry returns the same
  `receipt_id` and the same `mint_ts`. A second active insert for the key is impossible
  while the first is active.
- Eligible `no_new_failures` stores KNOWN/FLAKY signature refs and no output text.
- Expiry: lookup at `mint_ts + ttl - 1` is covered; at `mint_ts + ttl` is `expired`; the
  row is still there.
- Later same-key non-success returns `invalidated_by_later_run`. A still-later eligible
  success is covered by the new receipt id.
- An earlier run does not supersede a newer active receipt.
- Fingerprint drift on one repo-relative dirty path returns `fingerprint_changed`
  containing that path and no absolute path. A definition-digest change returns
  `definition_changed`.
- Incomplete current fingerprint returns `incomplete_fingerprint`.
- `policy_version` other than 1 on a row returns `verdict_insufficient`.
- Corrupt `proof_json` is not covered. Retention of its source run writes the tombstone
  explanation, and lookup returns `no_receipt` with that reason.
- Retention deletes an expired receipt past `summary_cut`, counts it in `detail_rows`,
  and leaves an unexpired receipt covered after its source run is summary-deleted.
- `store_stats.receipt_count` matches inserted rows and is 0 when the table is absent.
- Compatibility: build a store with the new code (`schema_version` meta still `"1"`, one
  receipt row). The historical column list in `store/tests/compat.rs` (`OLD_RUN_COLUMNS`
  / the pre-receipt `SELECT`) still reads the run. `show_run` on a store created from
  `OLD_SCHEMA_SQL` still works, and lookup there is `no_receipt`. This is the
  old-core/open-new-store demonstration available in-tree: replay the previous reader
  statements against a store that contains `tool_receipts`. Do not shell out to an older
  binary.
- Binding round-trip: settle then lookup through PyO3, schema version 1 on both results.

## Verification

From the sase-core checkout:

- `just fmt`
- `just test -p sase_core receipt`
- `just test -p sase_core_py tool_run_receipt`
- `sase tool run check` as the repo gate. `AGENTS.md` forbids a raw `just check` from an
  agent. Give that run at least 10 minutes. A targeted crate test does not replace it.

Then, on bead `sase-1ah.2` only:

- `sase bead epic-symbols sase-1ah.2`. This phase adds no `--epic-symbol` lines. If any
  symbol for this phase is still open, re-key that Justfile line to `sase-1ah` or to a
  later phase bead before closing.
- Close with
  `sase bead close sase-1ah.2 --note "<what the tests and the check run showed>"`.
- Record extra work as `sase bead note sase-1ah.2 'PROPOSED FOLLOW-UP: <one line>'`. Do
  not create beads. A `just check` failure that reproduces on the clean base tree is a
  follow-up note, and the phase still closes.

## Done when

- Schema meta stays 1 and an old-shaped reader still opens a store that contains
  `tool_receipts`.
- Definition digest ignores `receipt`. TTL above 2 hours is rejected.
- The no-mint matrix writes zero rows. Retry, expiry, supersession, recovery success,
  and fingerprint drift match the lookup order above.
- A pruned source run leaves either a self-contained active proof or an explicit
  tombstone, and lookup does not report `covered` for the tombstone.
- `detail_rows` includes deleted receipt rows, so the existing ToolRun disk owner bounds
  them.
- PyO3 settle and lookup round-trip.
- `sase-1ah.2` is the only bead closed.
