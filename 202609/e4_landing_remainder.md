---
tier: epic
title: Complete E4 receipt reporting and live acceptance
goal:
  Move receipt opportunity facts into the Rust core, require a released receipt-capable
  core before a fresh install can load the catalog, and prove prepared completion on
  athena.
parent_bead: sase-1ah
phases:
  - id: rust-opportunity-report
    title: Put opportunity facts in the Rust core
    depends_on: []
    size: large
    description:
      "rust-opportunity-report: implement the bounded, read-only receipt and
      content-equivalent repeat report in sase-core with a versioned binding and parity
      fixtures."
  - id: python-adapter-release
    title: Adopt the core report and released wheel
    depends_on:
      - rust-opportunity-report
    size: medium
    description:
      "python-adapter-release: replace the Python ledger and Git comparison with a thin
      Rust adapter, ratchet the core pin, and make fresh installs require a published
      receipt-capable wheel."
  - id: live-acceptance
    title: Demonstrate prepared completion and record the owner check
    depends_on:
      - python-adapter-release
    size: medium
    description:
      "live-acceptance: run focused and required check gates, demonstrate live athena
      pass and controlled no-new and drift outcomes, and record the 14-day owner check."
proposed_by: bbugyi200.athena.sase-1ah.land
create_time: 2026-09-26 13:37:25
status: wip
---

- **PROMPT:**
  [prompts/202609/e4_landing_remainder.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/e4_landing_remainder.md)
- **PARENT:**
  [202609/tool_e4_verified_completion.md](https://github.com/sase-org/sase--plans/blob/main/202609/tool_e4_verified_completion.md)

# Remaining E4 landing work

Parent epic `sase-1ah` has all seven original phases closed, but its approved plan
remains `status: wip`. This child covers only gaps found during the land audit. It does
not close the parent, retire its symbols, or change its plan status; the child
`parent_bead` link returns those decisions to the parent land agent.

## Evidence and constraints

- The approved E4 plan says `sase_core::tool_run` owns receipt policy, storage, lookup,
  and machine-local reporting facts. Python should be a thin adapter. Current
  `src/sase/tool/receipt_report.py` reads `tool_receipts` and `runs` with Python SQLite
  and computes content equivalence, Git blob comparison, grouping, and statistics in
  roughly 760 lines of Python. `sase-1ah.5` note #3 calls a Rust migration a possible
  later follow-up, but the approved boundary makes it remaining E4 work.
- On 2026-09-26, PyPI reports latest `sase-core-rs` as `0.34.73`, predating the receipt
  binding at core commit `9f86897` and no-new accept wire at `e654e7c`. `uv.lock`
  selects `0.34.71`, and `pyproject.toml` allows `>=0.34.71`. The source pin now names
  `e1179e65`, so a local rebuilt core works (50 focused Python receipt tests passed),
  but a fresh published-wheel install can reject `receipt:` in `sase/sase.yml`. Do not
  publish catalog policy with an old supported wheel. Follow the core repo's release-plz
  cadence and authorization; do not manually edit versions or changelogs.
- The original landing-proof note reported only a live `sase tool receipt check -j`
  `no_receipt` query and 217 focused tests, with six no-new tests skipped in that
  workspace. This audit reran 50 focused tests with no skips. The original plan also
  requires a live athena prepared-completion demonstration, including exact-tree commit
  or drift refusal and recovery, which is not evidenced by any phase note.
- `just symvision` on master `3065117500` fails on five `sase-19x.9` exemptions. The
  active card-block epic `sase-19x` already owns this and received a fresh
  `DISCOVERED ISSUE` note. Do not modify that epic's symbols here. The earlier
  `sase-19x.4` exemptions are gone. The `sase-18i` exemptions still name an open epic.
- `sase-1ah.2`'s gateway worker-pid flake was corroborated on existing task `sase-15e`.
  Apollo has too little independent KNOWN corpus and no mac machine was configured; keep
  no-new rollout claims athena-only. The Python-to-Rust report migration is an E4 gap,
  not a separate task. No `just check-full` is authorized.

## Phase 1: Rust opportunity facts

Open the linked `sase-core` repo via `/sase_repo` and follow its `AGENTS.md`. Add a
versioned `tool_run_receipts_report` request/result wire, Rust store query, and
content-addressed comparison. Preserve the current CLI JSON fields and semantics:
retained receipt counts and ages; windowed runs; equivalent groups, repeat duration and
hours, top tools; dirty-tree-to-committed-HEAD equality; missing Git objects/incomplete
fingerprints as uncomparable; bounded scan, Git-call, and output limits. Keep ledger
reads read-only and avoid storing argv secrets or output. Use existing Rust
fingerprint/proof helpers where possible. Cover parity against Python's existing
fixtures and add core fixtures for missing history and retention bounds. Expose and test
the PyO3 binding. This is domain behavior another frontend must share.

## Phase 2: Python adapter and released core

Replace `receipt_report.py`'s direct SQLite and Git comparison with a thin call to the
new binding, retaining CLI argument validation and Rich/JSON presentation. Add parity
tests and remove obsolete Python logic. Ratchet `sase-core-revision.txt` past the
binding commit and validate it. Wait for an official `sase-core-rs` wheel that contains
receipt policy, receipt lookup/settle, no-new accept, and the report binding; then raise
the minimum in `pyproject.toml` and update `uv.lock`. Verify a fresh wheel-based install
parses `sase/sase.yml` and runs both receipt commands. If the release has not happened,
leave this phase open and use the normal release cadence or an authorized urgent release
path; never claim the source build proves published-wheel compatibility.

## Phase 3: Live acceptance and owner check

Run relevant Rust and Python focused suites, `just fix`, and the required
`sase tool run check` in every changed repo. Use `just check` semantics; do not run
`just check-full`. Treat a failure caused by the active `sase-19x` symbol exemptions as
its owner issue, and recheck once that epic lands. On athena, demonstrate a prepared
`accept: pass` completion and a controlled `accept: no-new` verification fixture (or
real all-KNOWN/FLAKY red check if available). Inspect host receipts and commit state to
prove the exact verified tree was committed. Demonstrate a changed fingerprint or
another typed refusal launches recovery without committing. Record a dated 14-day owner
check for repeat opportunities, prepared-completion commits, and real fingerprint-change
refusals; future observations are not a landing prerequisite. Append concrete evidence
and any remaining limitation to the child and parent bead notes for the parent land
agent.
