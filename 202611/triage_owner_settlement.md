---
tier: epic
title: Attach triage owners after stage classification and bound candidate loading
goal:
  Complete sase-1tc.5 by preserving verdicts while attaching task owners to staged
  failures, loading candidates efficiently, and proving the persisted CLI output.
phases:
  - id: owner_annotations
    title: Attach owners at settlement without rewriting stage verdicts
    depends_on: []
    description:
      "owner_annotations: fix the reproduced stage-before-settle loss in sase-core,
      preserving classification and decision evidence while filling empty owner
      annotations; add store and binding regressions."
    size: medium
  - id: candidate_projection
    title: Project candidate fields in Rust and prove the full CLI path
    depends_on:
      - owner_annotations
    description:
      "candidate_projection: replace full Python task hydration with a narrow Rust
      candidate query, retain existing budgets and diagnostics, and prove exact node and
      file owners through a real isolated check, show, and failures command sequence."
    size: medium
proposed_by: bbugyi200.athena.sase-1tc.5
proposed_with: codex/gpt-6-astra
parent_bead: sase-1tc.5
create_time: 2026-11-19 09:15:27
status: wip
---

- **PROMPT:**
  [prompts/202611/triage_owner_settlement.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/triage_owner_settlement.md)
- **PARENT:**
  [202611/backlog_epic_cmdsub_retries_launches.md](https://github.com/sase-org/sase--plans/blob/main/202611/backlog_epic_cmdsub_retries_launches.md)

# Failure triage that retains the owning task bead

## Scope and evidence

This is a child implementation epic for **sase-1tc.5**, the `triage_owners` phase of
`plan:202611/backlog_epic_cmdsub_retries_launches.md`. It resolves the behavior tracked
by **sase-1ha**. The assigned phase had no previous implementation notes when read on
2026-11-19. Its dependency, sase-1tc.4, is closed. Its accepted parent decision
`mirror_fix = retire_mirror` stands: do not reintroduce Python SQLite ownership of the
bead store. No memory changes or new reviewer decisions are needed.

Investigation used sase **c4cb77b3b8** and linked sase-core
**d8681dbbff2b435cbb83d7052db7c118afcd9b67**, which also matches the primary core pin.
The working trees were clean. Existing core commit **9d049aac** already implements
file-identity owner matching; this phase must preserve it, not redo that fix.

The live query `sase tool failures -d 7 -n 500 -t check -j` returned **500 groups, zero
with nonempty `newest_owners`**. A direct call to `gather_owner_candidates(Path.cwd())`
returned **309 candidates in 0.355 seconds** with no diagnostics. This single warm
measurement is not a scalability guarantee. Run **6a06e0823754c1c7dc7499d26219cc77**
still records `triage owner candidates timed out`; it has stage labels and a recorded
stop decision. The historical task IDs from sase-1ha are no longer in the current
candidate set, so acceptance must use an eligible current owner or an isolated seeded
fixture.

The ownership loss was reproduced through the installed bindings using two disposable
stores and otherwise identical inputs:

| Sequence                                                                         | `possible_owners` | `newest_owners` | Class     |
| -------------------------------------------------------------------------------- | ----------------- | --------------- | --------- |
| Begin, observe, finish, settle with exact-node candidate                         | `sase-probe`      | `sase-probe`    | `unknown` |
| Begin, observe, stage with no candidates, finish, settle with the same candidate | empty             | empty           | `unknown` |

The fixture's output was `FAILED tests/triage_probe.py::test_owner - AssertionError`.
Its candidate was
`{node_id: sase-probe, location: tests/triage_probe.py::test_owner, status: open}`. The
extractor produced `locator_paths: [tests/triage_probe.py]`. Both requests reported
`triaged: true`. The child exit code was 9. These stores were discarded; the live ledger
was not edited.

The cause is concrete:

1. `src/sase/tool/triage_stage.py` deliberately passes `owner_candidates: []` and
   persists stage classifications. Its module contract says owners attach at settle.
2. `src/sase/tool/executor_triage.py` gathers candidates at final settlement.
3. In sase-core, `tool_run/store/triage.rs` preserves existing classifications with
   `ON CONFLICT ... DO NOTHING` and `WHERE item_id = ?1 AND class IS NULL`.
4. `tool_run/store/triage_stage.rs::triage_settle` only fills missing classifications.
   It has no independent owner-annotation pass for already-classified items.

The existing Python regression `test_triage_settle_stores_location_matched_owner_only`
exercises fresh settlement, so it cannot catch this stage-before-settle failure.

Two medium phases keep the ToolRun persistence change separate from bead-query and
Python integration work. Both have concrete implementation boundaries and tests; they
run strictly in sequence. The child lander owns final combined acceptance.

## Rules for both phases

- Read the assigned phase and this plan through audited SASE commands. Before edits,
  repeat `sase agent list -j` and inspect active plan scopes with
  `sase bead list -t plan --status in_progress -n 0`. The active work checked during
  planning was held-workspace settlement and disk attribution, not triage ownership. If
  a live agent now owns a required file, coordinate through bead notes and record a
  `PROPOSED FOLLOW-UP:` rather than editing its scope.
- Open sase-core with `/sase_repo`, then read its `AGENTS.md`; use only the path that
  command prints. Shared matching, selection policy, and persistence belong in Rust.
  Python resolves the store location, calls a binding, and transports diagnostics.
- Preserve the decision `triage-annotates-does-not-change-exit-codes`. Owners are
  routing suggestions, never witnesses for KNOWN or FLAKY. Do not change the item class,
  verdict, continuation decision, or process outcome to add an owner.
- Preserve the 1-second gatherer slice and 5-second settlement budget. Do not increase
  timeouts, weaken assertions, add skips/xfails, or add suppressions to obtain a pass.
- Do not add a CLI option, automatic historical re-triage, a broad matcher rewrite,
  global reinstall, or a new Python database reader. No memory files are authorized for
  editing. Source docstrings and tests can document the locator limitation below.
- Read `lint_and_test` before finishing. Use focused tests, then `sase tool run check`
  in every changed repository. In sase-core use its `just test` wrapper, never bare
  cargo. Do not run `check-full`. Use `/sase_monitor` before long commands; finish a
  staged monitor command and then declare through `/sase_final`.
- Let host completion move `sase-core-revision.txt` to a containing core commit, or use
  the supported ratchet after that commit exists. Never invent a SHA. Validate Python
  against the local rebuilt binding, then verify the committed pin contains it.
- Phase workers close only their own child phase. They never close sase-1tc.5, sase-1tc,
  or sase-1ha. Run `sase bead epic-symbols <own-phase>` before closing and resolve or
  re-key every entry. Record discovered work as `PROPOSED FOLLOW-UP:` on the assigned
  phase; do not create task beads.

## Phase owner_annotations

### Implementation

1. Add the stage-before-settle regression to sase-core's
   `crates/sase_core/src/tool_run/store/tests/triage_stage.rs` or a focused sibling. The
   reproduction must first show an empty stage owner list and then fail because settle
   cannot attach the supplied exact-node candidate. Retain a fresh-settle control.
   Assert `triage_show` and `tool_run_failures` both expose the annotation.
2. Reuse the existing matcher in `crates/sase_core/src/tool_run/triage/classify.rs`
   through a crate-internal seam. Matching already strips pytest node suffixes and
   understands file paths, directory boundaries, lists, title fallbacks, status
   ordering, and the two-owner cap. Preserve these semantics and the existing
   false-positive regressions.
3. Add an explicit owner-enrichment step to `triage_settle`, after normal item and label
   persistence and before the final read/verdict response. It must examine stored
   classified items even if `request.stages` and `run_output` are empty or retained
   output is gone. Use stored locator paths and full item identity, including run,
   stage, extractor version, and signature.
4. Persist only `possible_owners_json`, in a guarded transaction, when it is empty
   (`[]`, or legacy NULL where applicable) and matching yields a nonempty result.
   Preserve a preexisting nonempty list. A settle with no candidates is a no-op, so a
   later successful attempt can enrich an empty list. Replaying after enrichment is
   idempotent. Do not relax the generic record API's first-classification-wins rule.
5. Keep all classification fields byte-for-byte stable: class, touched, rule version,
   knobs, evidence, and classified timestamp. Preserve stage decisions, run outcome, and
   receipt/verdict semantics. Suppressed control/infrastructure runs must not gain item
   classifications or owner annotations through this path.
6. Extend the Python binding/store regression in `tests/core/_tool_run_store_triage.py`,
   keeping its collection facade in `tests/core/test_tool_run_store.py` wired correctly.
   This proves the new behavior survives the Rust/Python wire and database round trip.

### Required checks

- Exact pytest node and plain test-file candidates attach via `matched_on: location`.
- Stored-only settlement, missing output, and fresh settlement all work.
- All four classes retain their classification evidence while owners attach.
- Replay preserves nonempty owners; a failed/empty gather does not erase them; item and
  stage identities do not bleed into another item, run, or project.
- Existing nonmatching-token and directory-boundary cases still reject false owners.
- Before/after show and failure aggregation agree, and the run exit code stays 9.
- Focused Rust store/classifier tests, the binding regression, and both repository
  checks pass or have an identical clean-base failure recorded with evidence.

Record the before/after result and containing revisions on the phase bead. This phase
fixes persistence only; leave the parent phase open for candidate loading and CLI proof.

## Phase candidate_projection

### Implementation

1. Replace `gather_owner_candidates`'s full `list_issues(..., TASK)` hydration with a
   focused Rust candidate projection exposed through a required binding, for example
   `bead_triage_owner_candidates`. Reuse the current store-location and freshness
   mechanisms. The seam returns only candidate fields and diagnostics, not full issue
   bodies, notes, relationships, or Python `Issue` instances.
2. Put the projection/filtering policy in the bead read domain. Include only CI, flake,
   and bug tasks. Include open, ready, claimed, in-progress, and snoozed tasks as wire
   status `open`; include recently closed tasks within the existing seven-day lookback
   as `closed`. Exclude closed tasks with unknown or expired close time. Prefer task
   field `node_id` over `location`, and retain the title fallback and existing wire
   meanings: candidate `node_id` is the bead ID, while `location` contains the failing
   pytest node or source path. Honor an explicit `now_ts`.
3. Implement the warm read against the existing Rust read model. Push task-type, status,
   and recency filtering before hydrating full issues or crossing the Python boundary.
   Use the existing indexed columns plus a narrow JSON projection from `issues.row` for
   fields the cache does not materialize. Avoid a cache-schema redesign: the task is to
   eliminate full-row transport and Python hydration, not to rebuild the bead storage
   architecture. The ordinary Rust replay path must return equivalent candidate values
   when the cache is unavailable.
4. Do not cap candidates by newest N before matching: that can drop an older open bead
   that owns the exact failing test. Keep the existing outer bounded gatherer; cold
   refresh, contention, or unavailable stores must fail open with diagnostics. Release
   the GIL during the Rust read as the existing bead-list binding does. The hot path
   must have enough margin under the unchanged one-second slice on a large warm store;
   record the cold-path limit honestly rather than promising every overloaded read can
   succeed.
5. Add the thin Python facade and switch `src/sase/tool/triage_inputs.py` to it. Remove
   Python task filtering/recency duplication from this gatherer. Do not add a fallback
   implementation. Register the binding, add its round-trip test, and update
   required-binding validation if that project's mechanism requires it. Ensure the core
   pin contains the new binding before the primary changes are landed.
6. Document in the gatherer contract that a bare stage-label locator, such as
   `lint (symvision)`, is unsupported: owners need a file/directory location or a title
   naming a supported file locator. Add a negative regression. Do not infer an owner
   from generic stage words or bead-ID tokens.

### Required checks and CLI acceptance

- Candidate projection matches the old eligible values for representative stores,
  including every supported status, recent/expired/unknown close dates, `now_ts`,
  node-versus-location precedence, and ignored non-task/nonfailure task types.
- Cached and replay reads agree. A large fixture with irrelevant historical tasks and
  large notes does not hydrate full Python issues or serialize their notes. Use
  structural assertions and recorded timings rather than a fragile wall-clock unit test.
  Record before/after gather timings on the real warm store and a large fixture.
- Retain timeout and unavailable-store regressions in
  `tests/tool/test_triage_inputs_executor.py`: diagnostic persists, process exit code is
  unchanged, and any stored owner annotation survives an empty gather.
- Add a deterministic subprocess integration fixture with isolated ToolRun and bead
  stores, an open CI/flake task whose node matches, and a disposable named `check` tool
  using the real stage helper. Run a real `sase tool run check` that executes a failing
  pytest node, then `sase tool show -j <run>` and `sase tool failures -t check -j` in
  the same fixture environment. Do not mock the owner matcher, candidate gatherer,
  binding, or persistence in this proof. Cover normal stage-first mode and the
  fresh-settle path where practical.
- Record the failing node, fixture owner ID, run ID, command exit code, nonempty
  `label.possible_owners`, and matching `newest_owners`. The intentional failure is
  acceptance evidence, not the final verification gate. Keep it out of the live
  production ledger, do not create a real task bead for it, and remove disposable
  fixture files after retaining the evidence.
- Run the focused suites and the real repository `sase tool run check` in both changed
  checkouts. The final gate runs on the product tree without the intentional failing
  fixture. Never force a production test to fail just to obtain a screenshot or owner
  record.

## Child epic landing

The child lander verifies both child phases and their containing core pin, reruns only
checks justified by integration changes or unresolved evidence, and collects the CLI
proof plus candidate timing evidence. No owner list alone is evidence that a failure is
KNOWN. A gate failure reproduced identically on the clean base is recorded as a
`PROPOSED FOLLOW-UP:` with any existing tracking bead; it does not strand completed
phase work.

Record
`TASK RESOLVED: sase-1ha — <before/after owner evidence, run ID, and check results>` on
**sase-1tc.5**. Once this child epic and its descendants are complete, its land agent
closes **sase-1tc.5** with the verified result, first checking
`sase bead epic-symbols sase-1tc.5` and resolving every leftover. This is the explicit
completion target of this child plan; individual child phase workers only prepare that
evidence. Leave **sase-1tc** and **sase-1ha** for the outer epic's land agent. Do not
continue the outer epic's backlog-selection chain from this child epic.
