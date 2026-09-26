---
tier: epic
title: 'E4: Verified completion with fingerprint-bound verdict receipts'
goal: 'A named verification run can mint a short-lived receipt for its exact fingerprint
  and trustworthy E3 verdict; users and host-owned prepared completion can query that
  proof, while every tool invocation still executes and any drift or uncertain failure
  recovers.

  '
phases:
- id: hermetic-baseline
  title: Recheck E3 precision and verification inputs
  depends_on: []
  size: medium
  description: 'hermetic-baseline: re-run the KNOWN precision gate, inventory changing
    lint and bead inputs, and make check fingerprints complete within bounded probe
    cost.'
- id: rust-receipts
  title: Add the Rust receipt contract and durable store
  depends_on:
  - hermetic-baseline
  size: large
  description: 'rust-receipts: add additive schema-1 receipt storage, strict mint
    and invalidation rules, typed lookup refusals, retention, bindings, and compatibility
    tests.'
- id: core-pin-catalog
  title: Pin the released core before catalog adoption
  depends_on:
  - rust-receipts
  size: medium
  description: 'core-pin-catalog: ratchet sase to a published receipt-capable core,
    create the beta flag, then declare per-tool accept and TTL policy without moving
    definition identity.'
- id: receipt-execution-cli
  title: Mint and query receipts on both execution paths
  depends_on:
  - core-pin-catalog
  size: medium
  description: 'receipt-execution-cli: wire foreground and handed-off settlement to
    Rust minting and expose a truthful receipt query with versioned JSON and exact
    exit codes.'
- id: opportunity-report
  title: Measure content-equivalent verification repeats
  depends_on:
  - receipt-execution-cli
  size: medium
  description: 'opportunity-report: list receipts and report content-addressed repeat
    opportunities, including dirty-tree-to-commit equivalence, without changing run
    behavior.'
- id: verdict-completion
  title: Gate prepared completion on a covering receipt
  depends_on:
  - receipt-execution-cli
  size: large
  description: 'verdict-completion: add explicit no-new intent policy, recheck the
    verified tree at commit time, recover with typed refusals, and record provenance
    in host actions.'
- id: landing-proof
  title: Prove acceptance and remove the beta flag
  depends_on:
  - opportunity-report
  - verdict-completion
  size: medium
  description: 'landing-proof: exercise all nine landing gates, demonstrate athena
    completion, document the contract, publish authorized memory and skill sources,
    and remove the flag.'
proposed_by: bbugyi200.athena.0st
create_time: 2026-09-26 07:26:21
status: wip
bead_id: sase-1ah
---

- **PROMPT:** [prompts/202609/tool_e4_verified_completion.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/tool_e4_verified_completion.md)
- **BEAD:** [sase-1ah](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ah/README.md)

# Plan: E4 verified completion

## Outcome and authority

On a just-verified checkout, `sase tool receipt check` reports the covering ToolRun,
verdict, and age. A change to the fingerprinted checkout makes the query refuse with a
typed reason. An explicitly opted-in prepared completion can commit a checkout with
`no_new_failures` despite the child's nonzero exit, but only when the same frozen
ToolRun has an eligible, unexpired receipt and the host verifies the tree again
immediately before commit. NEW, UNKNOWN, infrastructure, drift, expiry, and invalidation
launch the ordinary recovery follow-up. Default prepared completion remains pass-only.
Ordinary `/sase_final` completion remains available; it records receipt provenance when
present and says `unverified` otherwise. **No E4 command skips a child execution.**

Approval of this plan explicitly approves the opt-in `accept: no-new` host-completion
policy, which §7 Q1 of the landing-criteria report reserved for Bryan. It does not
silently change the default, make a KNOWN label override an exit code, or authorize
reuse. The same approval names these SASE memory changes for `/sase_memory_write`: add a
short `glossary:receipt` strand and a decision record, _Receipts prove before they skip;
reuse waits for measured opportunity_; update existing reference guidance only where its
completion examples need the new opt-in form. If the approval declines the policy, do
not start this epic: the report §5.9 recommends deferring E4 altogether.

Inputs read with `sase artifact read`: the roadmap
`research:202609/sase_tool_epic_roadmap/sase_tool_epic_roadmap.md`, its newer and
controlling E3/E4 landing criteria
`research:202609/sase_tool_e3_e4_landing_criteria/sase_tool_e3_e4_landing_criteria.md`,
and the E2 hand-off plan `plan:202609/tool_e2_durable_handoff.md`. The newer report
replaces the roadmap's E4 reuse promise: 0 already-passing `check` repeats across 527
fingerprinted runs, including a content-addressed recount; 9 of 11 identical-fingerprint
sase-core retries after a failure passed. Fail-fast lint-to-test time was 247 s median,
so a cheap-stage shortcut is not established. Receipts here are proof for host
completion; E4b reuse reopens only after at least 2 h/week of content-equivalent covered
repeats for one tool on one machine for two consecutive weeks and a hermeticity proof.

Planning snapshot (recheck at implementation): sase HEAD `5f676a28e`, core pin
`9d049aac62ff173e41c9fec59734fcfc4af982d8`; E3 `sase-18j` and its remediation child are
closed. E3's landing audit reported 60/60 KNOWN examples correctly labeled, zero KNOWN
on added or untracked paths, and a live athena pass with durable triage. `sase-182`
(catalog-repo identity) and `sase-114` (nested stage leak) are closed. The current
`check`/`check-full` catalogs probe Python, just, and sase-core-rs, but omit ruff, mypy,
symvision, and prettier; `sase-vr` tracks the broader install provenance bug. The
ToolRun store is schema 1; E3's durable triage lives in additive tables. E2 stores
`settled_by` and both fingerprints. `src/sase/monitor/settlement.py` currently chooses
host completion only for a success branch, then
`src/sase/monitor/host_completion_state.py` evaluates the prepared intent. That branch
selection must be changed deliberately for `no-new` while keeping the pass-only golden.

## Contract and boundaries

- Rust `sase_core::tool_run` owns receipt policy normalization, eligibility, storage,
  invalidation, lookup, and the machine-local reporting facts. Python CLI/executor and
  monitor code are thin adapters. Open `sase-core` via `/sase_repo` and follow its own
  `AGENTS.md`. No second ledger, supervisor, or log copy.
- The receipt identity is catalog-repo identity + named tool + definition digest +
  extra-args digest + complete fingerprint digest. The row carries receipt id, source
  run id, verdict, KNOWN/FLAKY signature references, issue/mint/expiry timestamps, and
  policy version. It never stores argv secrets, environment values, or output bytes. An
  ad-hoc run cannot opt in.
- Mint only after authoritative wrapper settlement and E3 triage settlement, for a
  named, non-bypassed run whose before/after fingerprints are complete and equal, with
  `mutated_input = false`, `settled_by = wrapper`, failure kind `verification` (or a
  clean pass), and verdict in that tool's `receipt.accept` set. `pass` is always
  allowed; `no_new_failures` requires explicit catalog policy. Never mint from control,
  infrastructure, environment, new-failures, or undetermined outcomes. Incomplete or
  missing E3 evidence refuses no-new. A later same-identity, same-fingerprint
  non-success supersedes an earlier receipt until a later eligible success. Make the
  settle/mint/supersession update atomic or idempotently recoverable after a crash;
  retries cannot create duplicate or stale winners.
- Rust lookup returns `covered(receipt)` or exactly one typed refusal: `no_receipt`,
  `fingerprint_changed` (bounded changed paths), `expired`, `incomplete_fingerprint`,
  `verdict_insufficient`, `definition_changed`, or `invalidated_by_later_run`.
  Distinguish missing history from changed definition and report only paths from safe
  repo-local diff computation. Receipt expiry is a miss, not row deletion. A persisted
  but corrupt or unavailable store never proves coverage.
- Add `receipt: {accept: [pass, no_new_failures], ttl: 2h}` to the catalog wire as a
  per-tool opt-in with a literal bounded TTL and no global default. This _policy_ is
  excluded from `ToolDefinitionIdentity`, so editing it does not move historical
  duration or E6 corpus identity. Adding toolchain probes or fingerprint inputs _does_
  move the digest once, as intended. Since `ToolDefinitionWire` denies unknown fields,
  publish and pin the new core before adding any catalog `receipt:` block. Preserve wire
  `schema_version: 1`; use only new tables or nullable fields and no new values in old
  enums. Demonstrate an older core can open the store.
- Use the existing E3 verdict and its independent witness rule. Re-run its chronological
  backtest before enabling no-new. If the hand-audited KNOWN precision is below 95%,
  disable no-new and repair E3 classification first. Do not infer KNOWN from a bead
  alone. The research's apollo/mac corpus caveat remains a rollout limit: inspect those
  machine ledgers when available before enabling no-new there; unavailable/offline
  machines are not asserted proven by athena's audit.
- Keep one beta flag `tool_receipts` while partial user-facing behavior lands. Use
  `sase flag new` and test both branches; the Off branch preserves today's query and
  completion behavior. Remove it and close its flag bead in the landing phase.

## Phase implementation

### 1. `hermetic-baseline`

Run `tools/tool_triage_backtest` again on current athena data using E3's documented
sample/audit process; retain its artifact report and confirm >=95% KNOWN precision and
zero added/untracked KNOWN. Assess apollo/mac ledgers by permitted read-only host access
when online, or record why they cannot yet be enabled. Coordinate with `sase-vr` so E4
does not silently claim an install-provenance fix. Add and test bounded version probes
for ruff, mypy, symvision, and prettier to both `check` and `check-full`; ensure an
unavailable probe makes the fingerprint incomplete instead of yielding an old-looking
successful identity. Choose and document either a bounded digest of bead statuses read
by epic-symbol/flag lint or a conservative `check` TTL <=2h with an explicit external
state limitation. The probe budget is <=1 s each and <=2 s total. Show the chosen bead
change or TTL expiry causes lookup refusal. Keep raw `just check` behavior intact.

### 2. `rust-receipts`

In `sase-core`, add the receipt policy wire outside definition identity; normalize its
accepted verdicts and TTL. Add a new receipt table/index to the ToolRun SQLite store,
version 1 request/result wires, typed refusal results, mint/invalidate/lookup, and
retention integration. Use an injectable clock in tests. Retain a receipt's proof or an
explicit tombstone/explanation if its source run is pruned; neither a dangling ref nor
an unqualified covered result is acceptable. Add PyO3 bindings in the tool-run domain,
register and round-trip them, and test old-core/open-new-store compatibility and digest
stability for policy-only changes. Core tests cover every no-mint case, expiry,
invalidating run, recovery success, crash/retry idempotence, and fingerprint drift.
Record the new disk owner/retention footprint through the existing `sase disk` surface
rather than leaving receipt rows unbounded.

### 3. `core-pin-catalog`

After the core change lands and an installable core revision includes the new binding,
ratchet `sase-core-revision.txt` and binding validation in sase in this phase (see
`docs/rust_backend.md`). Do not deploy `receipt:` with an older pin. Add the opt-in
`tool_receipts` beta flag with `sase flag new` before the first user-facing behavior.
Add the opt-in catalog block for `check` only when its hermeticity choice and precision
gate pass; use `check-full` only if separately proven. Keep install and ad-hoc
receipt-less. Verify that policy-only edits preserve the existing definition digest and
that the phase-1 probe change intentionally changes it. Add explicit compatibility tests
and rollout notes for mixed installed core versions. If the release/pin is unavailable,
this phase waits; it must not bypass binding validation.

### 4. `receipt-execution-cli`

Call the Rust mint/supersession API after the E3 verdict has settled in both foreground
and claimed hand-off worker paths. The adapter may fail open for tool execution but
records a diagnostic; receipt lookup remains fail closed. Ensure `sase tool run` always
spawns the child even if an eligible receipt exists. Add
`sase tool receipt TOOL [-a/--accept {pass,no-new}] [-j/--json]`: exit 0 for covered, 1
for typed refusal, 2 for usage; show receipt id, run id, verdict, age, and bounded
reason, with versioned JSON. No string says the landing gate is satisfied, because that
is host policy. Keep options sorted, short aliases for public long forms, and clear help
per `cli_rules.md`. Test parity between foreground and `-H`/verify-monitor runs, and
show behavior under partial ledger writes and unavailable core binding.

### 5. `opportunity-report`

Add `sase tool receipts [-d/--days N] [-j/--json]` to list retained receipts and
content-equivalent repeat opportunities: counts, summed duration/hours, top tools, and
observation window, all versioned. Compute the comparison from path content at each HEAD
plus dirty/untracked hashes, with toolchain, environment, extra arguments, and catalog
identity kept in the key; compare Git blobs by content, not only HEAD or the existing
fingerprint digest. A verified dirty tree later committed and rechecked at a new HEAD
must count as a potential reuse opportunity. Distinguish that measurement from an
actually applicable current receipt, and mark missing Git objects or incomplete
fingerprints uncomparable. The report informs a later E4b decision; it never changes
what `run` executes. Add the cross-commit fixture and bounded history/retention cases.

### 6. `verdict-completion`

Extend the host-sealed prepared intent with explicit `accept` (default `pass`, optional
`no-new`) and bind it to the verify command, repository obligations, and policy version.
Only a verify monitor whose settled ToolRun is the bound named verification may use
no-new; a raw command, unrelated ToolRun, lost monitor, or missing owner link cannot. At
monitor settlement, select the completion branch for `no_new_failures` even though its
child exited nonzero, then re-evaluate the sealed intent. For the opt-in no-new path, at
the last host-owned precommit boundary, re-observe each obligated checkout and
Rust-lookup its covering receipt; compare the verified fingerprint and source run,
expiry, invalidation, verdict, policy, and current tree. Every obligated repo must
appear in the receipt's fingerprint, or the host refuses the no-new path. If any check
fails, withhold all commits and launch ordinary recovery with the typed reason. Handle
resumed host completion and multi-repo obligations without using a stale once-checked
result. The default pass path keeps its existing eligibility and does not acquire a new
receipt requirement. Provenance for commits and bead closes records receipt
id/run/verdict/KNOWN list, or `unverified` for the ordinary `/sase_final` path without a
covering receipt; this is nonblocking on that ordinary path. Keep default `accept: pass`
output and behavior byte-identical to its golden. Do not alter E3's child exit code or
let no-new mask NEW/UNKNOWN items.

### 7. `landing-proof`

Prove every DoD below with fixtures and one live athena prepared-completion demo,
including a red, all-KNOWN/FLAKY verification if available; if master is green, use a
controlled red fixture for the no-new branch and keep the live pass/drift demonstration
truthful. Verify the host committed exactly the fingerprinted tree, or refused and
launched recovery. Update `docs/tool.md`, `docs/monitors.md`, and the CLI/JSON docs;
update the `sase_final` and `sase_monitor` _source templates_ in
`src/sase/xprompts/skills/`, preview with `sase skill init --diff`, and deploy only from
landed canonical source per `generated_skills.md`. Use `/sase_memory_write` for
`glossary:receipt` and the named decision record; republish with `sase memory init`.
Remove `tool_receipts` and close its flag bead after testing On and Off behavior and
retiring the Off branch. Run focused Rust/Python fixtures and each changed repo's
required `sase tool run check` gate; do not run `check-full` absent an explicit request.
The epic land agent checks the full evidence, handles discovered follow-ups, and closes
the parent only when all gates hold.

## Landing gates (all required; green-master independent)

1. Hermeticity: changing a lint-tool version refuses an old receipt; changing relevant
   bead state refuses it by bounded probe or after the documented <=2h TTL. Probe cost
   stays within budget and failures are incomplete, never falsely covered.
2. Negative mint matrix: ad-hoc, bypassed, mutated, incomplete, non-wrapper-settled,
   control, infrastructure, environment, NEW, undetermined, and unaccepted no-new runs
   leave no receipt row. Both execution paths pass the same matrix.
3. Invalidation and expiry: later same-identity/fingerprint non-success blocks an old
   receipt until later eligible success; injectable-clock expiry and crash retry are
   deterministic.
4. Query: a verified tree exits 0 with receipt/run/verdict/age; touching a tracked file
   exits 1 with `fingerprint_changed: <path>`; JSON is versioned; query makes no landing
   sufficiency claim.
5. Completion: explicit no-new commits the exact verified tree with only KNOWN/FLAKY;
   NEW/UNKNOWN, drift, expiry, invalidation, wrong run, and stale policy all refuse and
   launch recovery with typed reasons. Default pass remains byte-identical. Demonstrate
   on athena.
6. No skip: every `sase tool run` invokes its child despite an existing receipt.
7. Compatibility: schema stays 1 and additive; old core opens store; `receipt:` leaves
   definition digest unchanged; the receipt-capable core is pinned before catalog use;
   retention has no unexplained dangling reference.
8. Opportunity report: content-addressed equivalent runs, hours, and top tools are
   correct, including dirty tree committed to a new HEAD; unavailable history is
   explicitly uncomparable.
9. Governance: precision >=95%, one beta flag removed and flag bead closed, docs,
   generated skill _sources_, `glossary:receipt`, decision record, and every memory
   write through `/sase_memory_write` are complete. Record a 14-day owner check for
   repeat opportunities, prepared-completion commits (baseline zero), and real
   fingerprint-change refusals; do not pretend those future observations are an E4
   landing prerequisite.

## Explicit exclusions and failure handling

No `-R/--force`, install reuse, instant second check, per-stage receipts, cheap-stage
inline hand-off, unchanged-since-failure refusal, cross-machine receipt sharing, or raw
`just check` redirect. A failed store write never changes the child's result, and a
missing receipt never authorizes host completion. Unknown triage remains unknown. A red
master is a fixture, not a blocker. The plan is an epic because the Rust release and
pin, executor paths, CLI measurement, host policy, and live acceptance are distinct
contracts with separate landing evidence; dependencies above make their order explicit.
