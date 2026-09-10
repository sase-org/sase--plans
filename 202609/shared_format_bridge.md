---
tier: epic
title: Canonical shared formats and coordinated wire contracts
goal:
  Every shared SASE format that still has two spellings has a tested, parser-aware,
  reversible conversion and a canonical wire contract that host, Rust core, gateway,
  LSP, and the installed plugins agree on, staged as one verified undeployed cohort with
  a per-host conversion manifest, so shared-data-cutover can converge the fleet and
  canonical-contracts can later delete the old readers against certified data.
phases:
  - id: bridge-inventory
    title: Fix each shared-format contract's disposition and refresh its fleet corpus
    depends_on: []
    description:
      "bridge-inventory: Close census gaps G5 and G2 and finding F6, refresh the
      per-host corpus for every shared format, and assign each of the seven contract
      families exactly one disposition (convert, coordinated-wire, or code-only) in a
      versioned bridge ledger. Report the append-only bead-wire field conflict for
      review instead of deciding it. No code or data changes."
    size: medium
  - id: bridge-core-contracts
    title: Land every Rust core contract change as one release
    depends_on:
      - bridge-inventory
    description:
      "bridge-core-contracts: Make the canonical Patch wire native in sase_core's
      parser, add canonical discriminator support to the gateway and xprompt LSP, add
      the two migration conversion contracts the kit will call, update the PyO3 bindings
      and parity tests, then publish one core release and ratchet the host's
      sase-core-rs floor and pinned revision."
    size: medium
  - id: residual-format-proofs
    title: Prove the no-live-data formats and correct the few mutable records
    depends_on:
      - bridge-inventory
    description:
      "residual-format-proofs: Produce per-symbol receipts proving no live producer or
      stored record uses the legacy plan prefixes, plan-chain suffixes, chat-link
      timestamps, memory note types, or legacy task-types heading on any host, and
      correct the handful of mutable bead records that lack explicit sizes through
      supported mutations only."
    size: medium
  - id: host-wire-adoption
    title: Move host and plugin callers onto the canonical wire
    depends_on:
      - bridge-core-contracts
    description:
      "host-wire-adoption: Retire every live host call to the Python patch-to-changespec
      wire translation, emit the canonical completion-catalog discriminator, update the
      mobile helper catalog and the four installed plugins, and add a sunset flag only
      where the inventory proved a real mixed-format interval."
    size: medium
  - id: patch-record-converter
    title: Build and rehearse the project-spec Patch record conversion
    depends_on:
      - bridge-core-contracts
    description:
      "patch-record-converter: Add the parser-aware patch-records operation to the
      migration kit catalog, converting legacy Patch headings, stitch section headers,
      review URL labels, and project spec extensions losslessly, and rehearse it on
      protected copies of the real mixed-spelling corpora with refusal, resume, and
      restore proofs."
    size: medium
  - id: gate-bundle-converter
    title: Build and rehearse the gate request bundle conversion
    depends_on:
      - bridge-core-contracts
    description:
      "gate-bundle-converter: Add the gate-bundles operation converting settled v2 gate
      request envelopes to v3 with recomputed hashes and unchanged identities,
      responses, and cancellations, refuse in-flight bundles and the older envelopes the
      current validator never accepted, and rehearse on protected copies."
    size: medium
  - id: bridge-cohort
    title: Stage, verify, and publish the undeployed bridge cohort
    depends_on:
      - residual-format-proofs
      - host-wire-adoption
      - patch-record-converter
      - gate-bundle-converter
    description:
      "bridge-cohort: Build the exact host, core, and plugin artifacts as one cohort,
      prove them with isolated wheel smoke and the full verification lanes, and publish
      the per-host conversion manifests, deployment note, and acceptance receipt that
      shared-data-cutover consumes. Deploy nothing and mutate no production data."
    size: medium
proposed_by: bbugyi200.athena.sase-x7.5
parent_bead: sase-x7.5
bead_id: sase-x7.5.1
create_time: 2026-09-10 12:53:58
status: wip
---

- **PROMPT:**
  [prompts/202609/shared_format_bridge.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/shared_format_bridge.md)
- **PARENT:**
  [202609/canonical_only_fleet_cutover.md](https://github.com/sase-org/sase--plans/blob/main/202609/canonical_only_fleet_cutover.md)
- **BEAD:**
  [sase-x7.5.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-x7/sase-x7.5.1.md)

# Canonical shared formats and coordinated wire contracts

This is the focused plan for phase `shared-format-bridge` of the canonical-only fleet
cutover. It is landing one of that epic's two-landing rule: **stage canonical writers
and conversions while the old readers are still available**. `shared-data-cutover`
converges the fleet against what this epic produces, and `canonical-contracts` deletes
the old readers afterwards. Nothing here deploys to a host or mutates production data.

## Why this is an epic rather than a tale

Three hard barriers make this multi-agent work:

1. **Two repositories and a mandatory release barrier.** The canonical Patch wire, the
   completion-catalog discriminator, and the conversion semantics all belong in
   `sase-core`. The host cannot call a new `sase_core_rs` binding until a core release
   exposing it is published, `pyproject.toml`'s `sase-core-rs` floor is raised (today
   `>=0.32.61,<0.33.0`) and `sase-core-revision.txt` is ratcheted; CI's
   `release-core-floor-smoke` job runs `tools/check_sase_core_rs_bindings` against the
   declared floor. The parent epic forbids collapsing two landings into one.
2. **Conversion code must not exist before its disposition is decided.** The census left
   two gaps (`G5` wire-contract inventory, `G2` plugin legacy-symbol audit) and one
   unreproduced citation (`F6`) squarely on this phase. Writing a converter for a format
   that turns out to have zero live records is exactly the permanent framework the
   parent epic forbids, and skipping a format that does have records would leave
   `canonical-contracts` deleting a reader that is still needed.
3. **Five separate verification surfaces.** A coherent cohort means the host repo,
   `sase-core` (including PyO3), and up to four installed plugins pass together, plus
   isolated wheel smoke on both Python 3.12 and the athena 3.14 runtime. That routinely
   outruns one agent turn.

All seven phases are `medium` and directly implementable. `residual-format-proofs` runs
in parallel with `bridge-core-contracts`; the three implementation phases run in
parallel after the single core landing. Every Rust core change is deliberately
concentrated in one phase so no two agents edit shared core exports, parsers, or
bindings at the same time — the parent epic's explicit ordering rule.

## Context and evidence

Read through the audited artifact interface this turn:

- `file:explicit:5d205f8bf16e8de09e033937` — the `fleet-census` ledger: Tier A–F
  contracts, dispositions, per-host data locations, findings F1–F6, gaps G1–G5.

Prior-phase evidence this plan depends on: `migration-kit` shipped exactly four
operations and explicitly reserved the registration point for this phase ("Patch
`COMMITS:`/`STITCHES:` headings, plan path prefixes, gate schema, plan chain suffixes,
and chat-link timestamps are Tier E. `shared-format-bridge` owns those conversions and
registers them into this catalog"). `canonical-producers` already deployed canonical
sources fleet-wide and reconciled mac's `type: long` memory note. `telegram-bridge`'s
recovered host commit is on master, so this phase starts from a tree that has the shared
pending-action transport API.

### Read-only measurements taken while planning

All on athena at master `59c4d1186`, core `63bb275`. Each is a starting point for
`bridge-inventory` to refresh on all three hosts, not a substitute for it.

| Contract family                  | Measurement                                                                                                                                                                                                                                                                          | Live legacy data? |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------- |
| Patch records                    | 13 `.sase` files under the projects root, two of them `.tmp_*` scratch files; 2 use `COMMITS:` — `sase-core.sase` (5 sections) and `gh_sase-org__sase-archive.sase` (532 KB, 57 `COMMITS:` **and** 11 `STITCHES:`, genuinely mixed). No `## ChangeSpec`, `CL:`, or `.gp` spec files. | **yes**           |
| Gate requests                    | 3,261 bundles: 3,187 v3, 21 v2, 53 v1. Every non-v3 bundle is from 2026-07-16..18; 72 of the 74 are settled with a response or cancellation, 2 are abandoned-unsettled (one v1, one v2), and `load_and_verify_bundle` accepts only v2/v3.                                            | **yes** (v2)      |
| Completion / editor catalog      | Materialized `vcs_project_catalog.json` (schema 4) still writes `kind: "changespec"` for patch rows beside the canonical `entry_kind: "patch"`; the Rust LSP matches on `"patch" \| "changespec"`.                                                                                   | wire only         |
| Plugin / gateway wire            | Host `parse_project_bytes` → `changespec_wire_from_dict` still rehydrates the legacy struct; `sase_core::parse_patch_project_bytes` is a `.into()` over `ChangeSpecWire`; the gateway keeps `/api/v1/changespec-tags` and `MobileChangeSpecTag*`.                                    | wire only         |
| Plans and artifact references    | 0 of 123 stored bead refs and 0 plan-header link targets in the plans sidecar use `.sase/sdd/plans/`, `sase/repos/plans/`, `sdd/plans/`, or bare `plans/`.                                                                                                                           | **no**            |
| Config, content, workflow syntax | No `type: short\|long` frontmatter in project or home memory (only prose describing the alias); no positional `{N}` placeholders in live xprompts.                                                                                                                                   | **no**            |
| Beads and task metadata          | 230 live beads; 34 sizeless are `plan` beads (correct by design) and exactly 2 are sizeless `task` beads (`sase-bw`, `sase-cx`, both `ready`).                                                                                                                                       | 2 records         |

### Finding F6 is resolved

The census could not reproduce the research's gate schema citation. It is reproducible
on the current tree: `src/sase/notification_gates/model_validation.py:11-12` defines
`LEGACY_GATE_REQUEST_SCHEMA_VERSION = 2` and `GATE_REQUEST_SCHEMA_VERSION = 3`. The live
readers are `notification_gates/hashing.py:36` (accepts 2 or 3) and `hashing.py:139`
(derives `primary_branch` from `branches[0]` for v2), plus
`notification_gates/debug_rendering.py:163`. `model_request.py:174-183` already refuses
anything but v3 on the creation path, so **v2 is read-only compatibility over settled
records**, which is what makes the conversion in `gate-bundle-converter` safe.
`bridge-inventory` records this closure in the ledger.

### One conflict this plan surfaces rather than decides

`changespec_name` and `changespec_bug_id` are `IssueWire` fields in
`sase-core/crates/sase_core/src/bead/events.rs`. They appear in 9,807 and 9,803 of the
29,930 append-only bead events respectively, though only 11 events carry a non-empty
value, and they are also keys in the `issues.jsonl` projection and in the gateway's
mobile bead summary wire. Renaming them on the wire would require the historical replay
path to accept the old keys, which **broadens the epic's named historical exception**.
The parent epic requires that concrete conflict be reported for plan review before the
exception is widened. `bridge-inventory` therefore records it with evidence and an
explicit recommendation, and no phase here renames those two fields. The default
recommendation to carry into review is: keep the stored event key names, because they
are immutable history, and confine any canonical rename to regenerated outward surfaces.

## Scope

**In scope.** Parser-aware conversions and canonical wire contracts for the parent
epic's seven seed families, staged as one tested cohort with per-host manifests:

- the canonical Patch wire across Rust core, PyO3, host, gateway, and LSP;
- project-spec Patch record conversion registered in the migration kit;
- gate request bundle conversion registered in the migration kit;
- the completion-catalog and mobile discriminator set;
- receipts proving which families have no live records at all;
- temporary sunset flags where — and only where — a real mixed-format interval exists.

**Out of scope, by owner.** Deploying anything to any host, converting production data,
restarting any writer, or observing a cutover window (`shared-data-cutover`, sase-x7.8).
Deleting any old reader, normalizer, alias, or the sunset flags' Off branches
(`canonical-contracts`, sase-x7.9, and its successors). Telegram's second store
(`telegram-cutover`, sase-x7.7). The bead note blob codec (`historical-codec`,
sase-x7.13). Deleting the migration kit itself (`enforce-and-verify`, sase-x7.14).

## Rules every phase inherits

- **Two landings, never one.** Old readers stay reachable for the whole of this epic. A
  phase that removes a reader is out of scope, full stop.
- **Rust owns shared semantics.** Parsing, conversion, storage contracts, and validation
  belong in `sase-core/crates/sase_core`; update `crates/sase_core_py` and the parity
  tests with them. Python is thin orchestration and presentation. Never add a Python
  fallback for an absent binding — fail clearly instead.
- **Open other repositories with `/sase_repo`** and use only the printed path; read
  artifacts with `sase artifact read`. This includes `sase-core` and every plugin.
- **Conversions are parser-aware and lossless.** Preserve identities, timestamps,
  ordering, references, comments, and unknown extension fields. Never use a global
  textual replacement for `task`, `COMMITS`, `changespec`, a path, or prompt syntax. Use
  atomic same-filesystem writes, bounded locks, compare-before-replace, and do not
  follow symlinks outside inventoried roots. A destination that already exists, or a
  migration marker, is not proof of completion.
- **Refuse rather than guess.** An irreconcilable record is a reported blocker, not a
  dropped row and not an invented value.
- **Sunset flags only for real intervals.** Create them only with `sase flag new`, never
  by hand-adding a registry member. Each needs enabled canonical behavior, an explicit
  disabled old branch, both-state tests, and a per-host zero-use removal condition. Do
  not touch the six existing unrelated flags.
- **Verification.** Changed host source needs `just install` when the workspace
  environment is stale, then `just check`; `just check-full` runs only through
  `/sase_monitor` with the `TESTING`/`TESTED` pair. `sase-core` needs its root
  `just check` including PyO3 — cargo-only tests are insufficient. Run each plugin's
  prescribed checks.
- **Bead discipline.** Each phase closes only its own bead, runs
  `sase bead epic-symbols <id>` before closing, and records discovered work as
  `PROPOSED FOLLOW-UP:` notes on its own bead. No phase closes an ancestor.

## 1. bridge-inventory

Produce a versioned machine-readable bridge ledger plus a short narrative report, both
registered as explicit artifacts on the phase bead. The ledger extends the census
ledger; it does not replace it.

Close the three inherited items:

- **G5** — the full LSP, gateway, and PyO3 wire-contract inventory. Cover every
  `sase_core_rs` export the host calls, `crates/sase_gateway/contracts/api_v1/`, the
  `crates/sase_xprompt_lsp` catalog readers, and the host's `src/sase/core/wire.py`,
  `core/wire_conversion.py`, `core/parser_facade.py`, and
  `xprompt/vcs_project_completion.py`. Record every field, discriminator, schema
  version, and route that has both a canonical and a legacy spelling, with its producers
  and consumers.
- **G2** — an end-to-end legacy-symbol audit of `sase-nvim`, `sase-github`,
  `sase-telegram`, and `sase-research-artifacts`, opened with `/sase_repo`. Go beyond
  the symbols research already named; follow actual call sites.
- **F6** — record the gate discriminator closure documented above.

Then assign every one of the seven contract families exactly one disposition:

- `convert` — live legacy records exist; name the owning converter phase, the operation,
  the affected roots per host, and the semantic fingerprint used to prove equality;
- `coordinated-wire` — no persisted legacy records, but producer and consumer must land
  and deploy together; name both sides and whether a sunset flag is required;
- `code-only` — no live producer and no stored record; name the receipt that
  `canonical-contracts` will cite when it deletes the reader.

Refresh the per-host corpus on athena, mac, and apollo through audited reads and
read-only SSH, recording exact counts and digests for: project spec files and their
heading, stitch-section, review-label, and extension spellings; the gate bundle
`schema_version` histogram with per-bundle settlement state; memory note `type:` values;
the materialized completion catalog's discriminators; live bead records lacking an
explicit size; and stored plan-reference spellings. Every row needs an exact read-only
reproduction command.

Record the `changespec_name` / `changespec_bug_id` conflict as described above, with
event counts, the affected Rust and host symbols, and a recommendation — not a decision.

Acceptance: all three hosts measured fresh; every family has exactly one disposition
with its reproduction command; unreachable hosts, unknown records, and unclassified
active writers are explicit blockers; no code changed and no data mutated.

## 2. bridge-core-contracts

One `sase-core` landing, one core release. Concentrating all core work here is what
keeps the three later implementation phases from editing shared exports concurrently.

- **Canonical Patch wire is native.** Today `sase_core::parse_patch_project_bytes` is
  `parse_project_bytes(...).map(PatchWire::from)`, so the canonical API is derived from
  the legacy struct. Invert that: the parser builds `PatchWire` directly, and
  `ChangeSpecWire` becomes an explicitly derived legacy projection retained for this
  interval only. Both PyO3 exports stay; canonical output must not route through the
  legacy struct. Byte-for-byte JSON output of both APIs must be unchanged for both
  canonical and legacy input — prove it with the existing golden and parity fixtures
  before changing any field.
- **Canonical discriminator support.** Teach the `crates/sase_xprompt_lsp` catalog
  reader to prefer `entry_kind` and to accept an entry with no legacy `kind` at all,
  keeping the current `"patch" | "changespec"` match working. Add the canonical
  counterparts to the `crates/sase_gateway` mobile wire and its contract document
  alongside — not instead of — the existing `/api/v1/changespec-tags` route and
  `MobileChangeSpecTag*` types.
- **Conversion contracts.** Extend `crates/sase_core/src/migration/` with the two
  contracts the kit will call: project-spec Patch record conversion and gate request
  v2→v3. Each needs plan, apply, and verify wire types, semantic fingerprints, and
  structured conflict records, following the existing `procs.rs` and `residue.rs` shape.
  Nothing may run automatically during startup, import, completion, or an ordinary read.
- **Bindings and release.** Update `crates/sase_core_py` and every parity test, land,
  publish a core release exposing the new bindings, then raise the host's `sase-core-rs`
  floor and ratchet `sase-core-revision.txt`, and confirm
  `tools/check_sase_core_rs_bindings`, `tools/validate_sase_core_rs`, and
  `tools/validate_sase_core_rs_version` all pass against the declared floor.

Acceptance: core root `just check` including PyO3 passes; host `just check` passes on
the raised floor; the golden and parity suites show identical output for both parse APIs
on canonical and legacy input; the new bindings validate; the release is published and
the floor and revision are ratcheted in one host-owned change.

## 3. residual-format-proofs

The families the measurements expect to have no live records. The deliverable is a
receipt table, one row per symbol, that `canonical-contracts` cites when it deletes the
reader — not a deletion.

- **Plan references.** Prove no stored plan reference on any host uses
  `.sase/sdd/plans/`, `sase/repos/plans/`, `sdd/plans/`, or bare `plans/`: bead refs,
  plan header sections, plan frontmatter, and the associations store. Covers
  `sdd/associations/_normalization.py::_LEGACY_PLAN_PREFIXES` and
  `sdd/plan_header_writes.py::_LEGACY_PLAN_MARKERS`. Prose inside a plan body that
  merely mentions an old path is not a stored reference; classify it as such explicitly
  rather than rewriting it.
- **Plan-chain suffixes.** `plan_chain.py`'s `_LEGACY_DOTTED_SUFFIX_MAP` and
  `_LEGACY_DASH_SUFFIX_MAP`: prove no live agent record, launch path, or stored family
  name uses a dotted or single-dash role suffix. Archived chats and prompts are
  historical text and are not rewritten.
- **Chat-link timestamps.** `history/chat_links.py::_LEGACY_TIMESTAMP_LINE_RE` and the
  block metadata regexes: prove no current writer emits the legacy spelling. Raw
  transcripts keep their original bytes.
- **Memory note types and the legacy task-types heading.** `memory/notes.py`'s
  `_LEGACY_NOTE_TYPES` and
  `main/init_memory/root_rendering_task_types.py::_LEGACY_TASK_TYPES_NOTE_TYPES_HEADING`:
  confirm no managed note on any host declares `type: short|long` and no generated note
  still carries the legacy heading. Any real edit goes through `/sase_memory_write`.
- **Mutable bead metadata.** Re-derive the list of live task beads with no explicit size
  (two at planning time, `sase-bw` and `sase-cx`, both `ready`) and give each an
  explicit size through supported mutations only. Derive each size from that bead's own
  description and evidence; if it is ambiguous, surface it for resolution rather than
  applying a default — the parent epic forbids silently choosing a worker model for
  ambiguous historical work. Never rewrite an event file. Plan beads carry `tier` and no
  `size` by design; leave them alone. Also distinguish current `## Types` uses from the
  obsolete parsing alias, per the epic's seed table.

Acceptance: one receipt row per symbol with its reproduction command and its result on
each of the three hosts; any family that turns out to have live records is handed back
to `bridge-inventory`'s ledger as a `convert` row with a named owner, not silently
converted here.

## 4. host-wire-adoption

Move the host and the installed plugins onto the canonical contract landed in phase 2.
The legacy exports stay; what changes is that no live path calls them.

- **Retire the host's wire translation from live paths.** `core/parser_facade.py`'s
  `parse_project_bytes` and `core/wire_conversion.py`'s `changespec_wire_from_dict`
  remain exported for the interval, but every live caller moves to
  `parse_patch_project_bytes` / `parse_patch_project_file` and `PatchWire`. That
  includes `sase/core/__init__.py`'s documented surface and the golden and compatibility
  test corpora that still call `parser_facade.parse_project_file`. This is the epic's
  explicit requirement to "eliminate Python patch-to-changespec wire translation by
  changing the actual Rust API".
- **Completion catalog.** `xprompt/vcs_project_completion.py` emits `entry_kind`
  unconditionally today and still writes `kind: "changespec"` for patch rows for old
  readers. Keep the legacy key only for as long as an un-upgraded LSP binary can be
  running, behind the sunset flag below, and bump `VCS_PROJECT_CATALOG_SCHEMA_VERSION`
  if the emitted shape changes.
- **Mobile and plugin callers.** Move `integrations/_mobile_helper_catalog.py` and the
  gateway client path to the canonical field set, and update whatever the `G2` audit
  found in `sase-nvim`, `sase-github`, `sase-telegram`, and `sase-research-artifacts`.
  Run each plugin's prescribed checks. A plugin change that needs an API the released
  core does not have belongs back in phase 2, not in a prematurely deployed consumer.
- **Sunset flags.** Create one only where `bridge-inventory` proved a real mixed-format
  interval. The expected case is the completion-catalog discriminator, where a
  long-lived editor session can still be running an LSP binary that only understands
  `kind: "changespec"` after the host is upgraded. Use `sase flag new` with kind
  `sunset`, both-state tests, and a per-host zero-use removal condition; record the flag
  bead so `canonical-contracts` can delete the Off branch and close it in one change.

Acceptance: `just check` passes; no live host path calls the legacy wire translation,
and a test asserts that; the completion catalog is consumable by both an old and a new
LSP binary for the flagged interval; every touched plugin passes its own checks;
`tools/check_feature_flags` agrees with any new flag bead.

## 5. patch-record-converter

Register a `patch-records` operation in `src/sase/migration_kit/catalog.py` and
`src/sase/migration_kit/operations/`, calling the Rust conversion contract from phase 2.
This is the registration point `migration-kit` deliberately left open.

Per project spec file, convert `## ChangeSpec` → `## Patch`, `COMMITS:` → `STITCHES:`,
`CL:` → `PR:`, and the `.gp` extension → `.sase`, parser-aware, never textually.
Preserve record order and mixed-section ordering, stitch identities and numbering,
timestamps, `DELTAS`, `HOOKS`, `MENTORS`, `COMMENTS`, `TIMESTAMPS` bodies, unknown
fields, and the byte content of everything outside a converted line. The archive file
measured above mixes 57 `COMMITS:` sections with 11 `STITCHES:` sections in one
document, so partial conversion and section interleaving are the primary correctness
cases, not edge cases.

Refuse — do not repair — when: the canonical parse differs semantically before and
after; one record mixes both spellings in a conflicting way; a destination file already
exists; the source digest changed since the dry run; or a path resolves through a
symlink outside the inventoried roots.

The dry run emits host identity, root and repo revision, per-file source digest, record
counts, semantic fingerprints, conflicts, estimated space, backup location, and intended
action. Apply is digest-gated, idempotent on re-run, journal-resumable without skipping
an unconverted record, and writes atomically on the same filesystem.

Rehearse on protected copies only: the two real athena files measured above, plus
synthetic cases for mixed ordering, an empty stitch section, a record with a legacy
review label, corruption, an interrupted write, a concurrent writer, and a symlinked
path. Prove restoration through `sase migrate restore`. Refresh the mac and apollo
corpora from the ledger and rehearse whatever they contain.

Acceptance: exact conversion on legacy fixtures; a genuine no-op on canonical fixtures;
every refusal case refuses; interrupted-run resume loses nothing; a restore rehearsal
succeeds; parse equality against the pre-conversion `PatchWire` records holds; no
production data mutated.

## 6. gate-bundle-converter

Register a `gate-bundles` operation the same way.

Convert settled v2 gate request envelopes to v3: materialize `primary_branch` exactly as
`notification_gates/hashing.py:139` derives it today from `branches[0]`, recompute
`hashes.request`, and preserve `request_id`, `kind`, the bundle path, `response.json`,
`cancellation.json`, the creation journal, and every timestamp. **Identities never
change**, because the epic requires migrating complete bundles and indexes without
changing IDs or losing responses, cancellations, or approval state.

Two refusals are mandatory:

- **In-flight bundles.** A bundle with no response, no cancellation, and a lifecycle
  deadline that has not passed is refused, so no live approval is rewritten underneath a
  running gate shell.
- **Envelopes the current validator never accepted.** The 53 v1 bundles measured above
  are already unloadable by `load_and_verify_bundle`. Synthesizing a v3 record from a
  shape current code refuses would invent history. Classify them as archive-only
  historical residue, report them with their settlement state, and let the ledger carry
  the disposition; do not guess a decision for the one unsettled v1 bundle.

Rehearse on protected copies of the athena corpus (3,187 v3, 21 v2, 53 v1 at planning
time; refresh before acting) and on whatever mac and apollo hold. Verify that
`load_and_verify_bundle` accepts every converted bundle at v3 with matching hashes, that
`sase gate show` renders a converted bundle identically before and after, and that
`debug_rendering.py`'s v2 branch is no longer reached by any converted record.

Acceptance: conversion is exact and reversible from the backup; both refusal classes
refuse with a report; resume and restore are proven; no production data mutated.

## 7. bridge-cohort

Assemble the cohort and hand it to `shared-data-cutover`. This phase deploys nothing.

- **Build one coherent set of artifacts:** the core wheel from the exact released
  revision, the host wheel, and a wheel for every plugin `host-wire-adoption` changed.
  Record exact Git SHAs, wheel hashes, the `sase-core-rs` floor, and the pinned core
  revision. Package version equality is not evidence of a cohort.
- **Prove it:** isolated smoke in fresh virtualenvs on Python 3.12 and the athena 3.14
  runtime, asserting imports resolve inside the venv; host `just check-full` through
  `/sase_monitor`; `sase-core` root `just check` including PyO3; each plugin's own
  checks. If a lane fails for a cause this epic did not introduce, corroborate it with
  evidence and record it — do not relax a budget, a tolerance, or a baseline to obtain a
  green.
- **Publish:** the per-host dry-run manifests for `patch-records` and `gate-bundles`;
  the deployment note naming every artifact identity and the exact order
  `shared-data-cutover` must follow; the sunset-flag inventory with each flag's zero-use
  removal condition; and the acceptance receipt sase-x7.8 consumes. Register each as an
  explicit artifact on the phase bead.
- **State the barrier explicitly** in the deployment note: freezing writers, converting
  authoritative records, synchronizing replicas, and issuing the all-host certificate
  are sase-x7.8's work. Old dormant clones must either receive canonical content or be
  barred from writing until their recorded update step completes.

Acceptance: every check passes or its failure is corroborated as pre-existing with
evidence; the cohort's artifact identities are recorded and reproducible; the manifests
are dry-run only; no host was deployed to, no writer restarted, and no production record
converted.

## Rollback

Before any reader deletion, rollback is the retained bridge cohort: the old readers are
still present in every phase of this epic, so reverting the host, core, and plugin
revisions as one unit restores prior behavior. Because this epic converts no production
data, there is no shared-data rollback to coordinate here — the migration kit's verified
backups plus the per-operation archive copies cover the rehearsal corpora, and
`shared-data-cutover` owns the coordinated shared-store procedure. Keep backup locations
and their restore commands usable without SASE itself.
