---
tier: tale
title: Complete core failure classification and aggregation
goal:
  Rust core deterministically classifies durable ToolRun failure items, computes
  verdicts, and exposes stage, settle, show, and failures operations.
size: medium
proposed_by: bbugyi200.athena.sase-18j.3
bead: sase-18j.3
status: done
---

- **PARENT:**
  [202609/tool_e3_failure_triage.md](https://github.com/sase-org/sase--plans/blob/main/202609/tool_e3_failure_triage.md)
- **BEAD:**
  [sase-18j.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-18j/sase-18j.3.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-18j.3](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-18j.3.md)
- **COMMITS:**
  - [321e7b4](https://github.com/sase-org/sase-core/commit/321e7b4762ff461f189ce18281c8306e0fb9c0eb)
    — feat(triage): pure classification, verdict, stage/settle, and failures aggregation

# Complete phase sase-18j.3: core failure classification

## Goal and boundary

Implement the `core-classification` phase of the E3 failure-triage epic in the linked
`sase-core` repository only. This phase consumes phase 2's immutable extracted items,
triage tables, and record/show bindings. Do not change the `sase` repository, move its
core revision pin, alter the ToolRun schema version, or close the parent epic. Open
`sase-core` with `sase repo open sase-core`, read its `AGENTS.md`, and edit only the
printed checkout.

## Implementation

1. Extend `crates/sase_core/src/tool_run/triage/` with versioned, `schema_version: 1`
   request/result wires and pure functions for `tool_run_triage_classify` and
   `tool_run_triage_verdict`. Requests reject unknown fields; results remain lenient.
   Model subjects, bounded same-identity evidence runs and selection-health records,
   first-parent ancestry, active flake-baseline entries, owner candidates, and the two
   knobs explicitly. Canonicalize and sort every input and output so permutations
   produce byte-identical results. Refuse cross-version signatures, incomplete
   fingerprints, ad-hoc or cross-identity evidence. Treat absent evidence as UNKNOWN.
   Never use a bead or title match to establish KNOWN.
2. Implement the epic's classification contract exactly: FLAKY from baseline or
   same-complete-fingerprint disagreement for test extractors; KNOWN from
   distinct-workspace valid witnesses, after clearing-run and clean-tree-knob checks;
   NEW from touched paths or an eligible pass witness; UNKNOWN otherwise, including
   generic and environment items. Record ordered rejection reasons and the evidence
   refs, distinct agent/workspace counts, first-seen time, touched state, rule version,
   and knob values in each label. Match at most two possible owners by item locator
   tokens against supplied candidate node ids, locations, and titles; distinguish open
   candidates from recently closed possible fixes. Compute REPEAT only for a prior
   failed evidence run with the same complete fingerprint and signature set.
3. Implement pure failure-kind and verdict functions, including every `terminal_cause`,
   the conservative legacy mapping, environment `_setup` remedy, control/infrastructure
   suppression, and the ordered verdict table. Exit 0 means pass; no-new-failures
   requires all KNOWN/FLAKY items, all stages complete, a `recipe_finished` fact, and a
   stageful tool. Missing labels or triage data yield undetermined.
4. Add store-backed `tool_run_triage_stage` and `tool_run_triage_settle`: extract and
   record missing output, query only bounded prior settled runs of the same
   project/tool/args and machine within the seven-day lookback, classify unlabeled
   items, persist labels once, attach supplied owner candidates, and store repeat/run
   facts. Preserve first-writer-wins behavior on retries. Extend `tool_run_triage_show`
   with computed kind and verdict and the newest-run lookup by `(owner_kind, owner_id)`.
   Add read-only `tool_run_failures` grouped by project/tool, stage key, extractor
   version, and signature. Implement project/all-project, tool, class, days, and limit
   filters; include run/agent/workspace counts, first/last seen, last run, newest class
   and owners; sort deterministically by agent count then recency. Probe for absent
   triage tables on read.
5. Export and register all new bindings in the telemetry PyO3 domain and extend its real
   binding round-trip test. Add golden JSON request/result fixtures for classify, stage,
   settle, show, and failures. Keep files below the repository's 1,500-line limit and
   the triage module facade limited to declarations and exports.

## Verification

- Test DoD-3: all terminal causes and legacy mappings; `_setup` environment output and
  remedy; no labels for control/infrastructure.
- Test DoD-4's ledger cases (a)–(j), both tightening knobs, owner matching, REPEAT,
  cross-project/machine exclusion, and a permutation property for byte-identical
  classification output.
- Test verdict table boundaries: no `no_new_failures` without recipe finish, with
  UNKNOWN, for `stages: none`, or with untriaged rows. Test store idempotence,
  stage/settle/show round trips, aggregation and filters, and linked-project isolation.
- Run targeted tests during implementation, format through `just fmt`, then run the full
  required `sase tool run check` from the `sase-core` checkout with a generous timeout.
  If the check fails identically on a clean base, record a `PROPOSED FOLLOW-UP:` note on
  sase-18j.3 and close the phase anyway; otherwise repair regressions.
- Before close, run `sase bead epic-symbols sase-18j.3` and resolve any remaining
  symbols or re-key them to an open bead. Close only `sase-18j.3` with
  `sase bead close sase-18j.3 --note "<verified evidence>"`.
