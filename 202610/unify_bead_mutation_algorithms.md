---
tier: epic
title: One mutation algorithm per entry point, with every suite in both modes
goal: "Every ordinary bead mutation runs one algorithm over MutationView, on both the
  cached and the replay backing. The parallel MutableStore replay copies are deleted.
  The nine existing mutation suites run in cached and replay modes. Legacy and no-git
  stores keep their bytes, proven by goldens. The over-cap read-model files return to
  their pre-epic sizes. The landing of sase-1h8.13.1, and then of sase-1h8.13, can then
  resume.

  "
parent_bead: sase-1h8.13.1
phases:
  - id: suite-modes
    title: Run the nine existing mutation suites in cached and replay modes
    depends_on: []
    size: medium
    description:
      "suite-modes: make every test in the nine existing mutation suites run against
      both a git-backed cached store and a plain replay store, without macro_rules;
      assert cache equals replay after each cached mutating test; fix every parity
      defect this exposes in the try_cached_* algorithms with a regression test."
  - id: replay-goldens
    title: Byte goldens for replay-backed mutations before any replay code is deleted
    depends_on: []
    size: medium
    description:
      "replay-goldens: generate committed byte-level goldens from the current replay
      code for every ordinary mutation entry point, covering success and rejection on a
      no-git event store and on a legacy issues.jsonl store, with an UPDATE_ env
      regenerator. Later phases must keep them byte-identical."
  - id: view-commit
    title: View-owned staging and one commit on both backings; create and notes unified
    depends_on:
      - suite-modes
      - replay-goldens
    size: medium
    description:
      "view-commit: give MutationView config and event staging, lazy stream loading and
      a single commit on both backings (the replay commit keeps MutableStore::save
      semantics exactly), plus one runner that retries the same algorithm on the replay
      backing when the cached path declines before any append; run create and the notes
      family as single algorithms through it and delete their replay copies."
  - id: unify-lifecycle
    title: Open, close and remove as single view algorithms
    depends_on:
      - view-commit
    size: medium
    description:
      "unify-lifecycle: run open, close, close-with-note and remove through the
      view-commit runner on both backings, and delete the close_remove.rs replay copies
      and its family stream-slot helper; suites and goldens stay green."
  - id: unify-claims-deps
    title: Claims, ready marking and dependencies as single view algorithms
    depends_on:
      - view-commit
    size: medium
    description:
      "unify-claims-deps: run launch/wait claims, release, epic preclaim, mark/unmark
      ready, and dependency and reference add/remove through the view-commit runner on
      both backings, and delete their replay copies; suites and goldens stay green."
  - id: unify-links-evidence
    title: Links, +1 and snooze as single view algorithms
    depends_on:
      - view-commit
    size: medium
    description:
      "unify-links-evidence: run link add/projections/remove, +1, snooze and cancel
      through the view-commit runner on both backings, and delete their replay copies,
      including apply_prepared_link_projections; suites and goldens stay green."
  - id: caps-docs
    title: Shrink the over-cap read-model files and fix the stale phase-approval docs
    depends_on: []
    size: small
    description:
      "caps-docs: move the read-model functions that publish-direct changed out of
      read_model/store.rs (2,174 lines, 2,099 before the epic) and read_model/tail.rs
      (1,554, 1,519 before) into new files, so tail.rs is at or under 1,500 and store.rs
      at or under 2,099; in sase, fix the docs/beads.md sentence that still says phase
      agents auto-approve epic-tier plans."
  - id: proof
    title:
      Cleanup, single-algorithm audit, cached-equals-golden bytes, and acceptance
      evidence
    depends_on:
      - unify-lifecycle
      - unify-claims-deps
      - unify-links-evidence
      - caps-docs
    size: medium
    description:
      "proof: delete what the unification left unused, including the view's
      allow(dead_code); prove by search that MutableStore::load is reached only by the
      view's replay backing and export_jsonl; assert that cached-mode golden scenarios
      produce the golden bytes; rerun the parity and proof suites and the matched 1x/8x
      bench; record acceptance evidence for the sase-1h8.13.1 and sase-1h8.13 landings."
proposed_by: bbugyi200.athena.sase-1h8.13.1.land
create_time: 2026-10-08 21:24:11
status: wip
---

- **PROMPT:**
  [prompts/202610/unify_bead_mutation_algorithms.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/unify_bead_mutation_algorithms.md)
- **PARENT:**
  [202610/finish_read_model_mutations_child_epic.md](https://github.com/sase-org/sase--plans/blob/main/202610/finish_read_model_mutations_child_epic.md)

# Plan: One mutation algorithm per entry point, with every suite in both modes

## Why this epic exists

Child epic `sase-1h8.13.1` ("Finish read-model mutations so sase-1h8.13 can close") has
all eight phases closed. Its land agent verified the source and found two required
deliverables missing:

1. **No single algorithm.**
   - The epic's goal starts "Every ordinary bead mutation runs one shared algorithm on
     an indexed mutation view".
   - Its `view-core` phase required the following:
     - a view-owned staging area and a single `commit`, where "on the replay backing,
       `commit` keeps today's `MutableStore::save` semantics exactly"
     - "single algorithms" for create and the notes family
   - Its three port phases required each family to be rewritten "as single algorithms
     over the view, so both backings run the same code and no ordinary mutation in the
     family calls `MutableStore::load` directly".
   - What landed is a second implementation of every entry point:
     - A `try_cached_*` function runs on the cached view and returns `Ok(None)` to
       decline.
     - The untouched `MutableStore::load` replay code then runs.
     - The doc comments say the cached code "mirrors the replay path exactly".
   - The view's replay backing is only used in tests: `MutationView::load` and
     `with_replay` are called only from `mutation/tests`, under `#[allow(dead_code)]`.
   - Event minting still exists twice (`shared::mint_stream_event` and
     `MutableStore::append_issue_event`). Each family also has its own stream-slot
     helper.
   - The mutation modules grew from about 3,200 to about 6,850 lines.
2. **The existing suites never run cached.**
   - The `dual-mode-tests` phase was to run "each existing suite" in both modes. It
     added 10 new representative tests (`mutation/tests/dual_mode.rs`) instead.
   - The 151 tests in `mutation/tests/{claims,close,create,delegation_remove,`
     `dependencies,links,notes_update,snooze_plus_one,store}.rs` still never create a
     `.git`, so they exercise only replay.
   - Parent phase `sase-1h8.13`'s own test requirement is "Every existing mutation test
     passes in both cached and uncached modes". It is therefore not yet met.

The landing also found two smaller drifts this epic fixes:

- **Over-cap files grew.** `publish-direct` grew `read_model/store.rs` from 2,099 to
  2,174 lines and `read_model/tail.rs` from 1,519 to 1,554. The epic's rule was "Do not
  grow over-cap files".
- **Stale docs.** `planner-guard` added its sentence to a `docs/beads.md` paragraph that
  starts "When a phase agent auto-approves an epic-tier implementation plan". Since
  `e6adb110af` (sase-1id.4), phase agents run under `%auto:tale`, so a nested epic plan
  parks for human review.

Everything else the previous epic reported is real and stays:

- direct write-through publication, the content-generation CAS and the forced-sweep fix
- affected-row-only cached paths for every family
- the `publish_direct`, `view_core`, `lifecycle_warm`, `claims_deps`, `links_evidence`
  and `proof_bounded` suites, and `tests/bead_read_model_mutation_proof.rs`
- the matched benchmark artifacts

Do not redo that work. Collapse the duplication onto it.

**Reviewer note.** Every other acceptance item of `sase-1h8.13` is met. The alternative
to this epic is to reject it: close `sase-1h8.13.1` now with the duplication and
single-mode suites recorded as a task bead. That lets the perf gate `sase-1h8.14` start
sooner, but it keeps two implementations of about 25 entry points in sync by hand. It
also closes `sase-1h8.13` with one of its stated test requirements unmet. This plan
takes the faithful path.

## Authoritative context

Read before starting any phase:

- `sase bead read sase-1h8.13.1 -r "<why>"`: the previous epic and its landing notes.
  Its child phases' notes describe what each port did.
- `sase bead read sase-1h8.13 -r "<why>"`: the phase whose acceptance this work
  completes.
- `sase artifact read plan:202610/finish_read_model_mutations_child_epic.md "<why>"`:
  the previous epic's plan. Its "Rules for every phase" still apply unless restated
  below.
- `sase artifact read plan:202610/bead_store_history_independent_performance.md "<why>"`:
  section "Phase `read-model-mutations`" is the parent acceptance.

## Verified starting point (sase-core `3e459248` = `origin/master`, clean, at planning time)

Paths are relative to the linked sase-core checkout
(`sase repo open sase-core -r "<why>"`). Re-verify `HEAD` first. Line numbers are
approximate.

- **Entry points.** Each has the shape
  `with_bead_mutation_lock(.., || { if let Some(view) = MutationView::load_cached(..)? { if let Some(outcome) = try_cached_X(..)? { return Ok(outcome) } } let mut store = MutableStore::load(..)?; <replay copy> })`:
  - `create.rs`: `create_issue`
  - `notes_update.rs`: `update_issue(s)`, `append_issue_note`, `edit_issue_note`,
    `remove_issue_note`
  - `close_remove.rs`: `open_issue`, `close_issues(_with_note)`, `remove_issue(s)`
  - `claims.rs`: `claim_for_agent_launch`, `claim_for_agent_wait`,
    `release_agent_claim`, `preclaim_epic_work_plan`
  - `store.rs`: `set_ready_to_work` (mark/unmark)
  - `dependencies.rs`: `add_dependency`, `remove_dependencies`, `add_bead_references`,
    `remove_bead_references`
  - `links.rs`: `add_bead_link`, `set_bead_link_projection(s)`, `remove_bead_link`.
    `apply_prepared_link_projections` (~334) is replay-only.
  - `plus_one_snooze.rs`: `add_task_plus_one`, `snooze_task`, `cancel_task_snooze`
- **Cached-side plumbing.** Each `try_cached_*`:
  - loads config with `load_config`
  - builds `streams`/`base_lens`/`stream_index` through a per-family slot helper (for
    example `close_remove.rs` `lifecycle_stream_slot`) over
    `shared::load_mutation_stream`
  - mints with `shared::mint_stream_event`
  - commits with `shared::commit_staged_write` (`shared.rs` ~132)

  `commit_staged_write` returns `Ok(None)` only before any durable write (a missing
  manifest). The other `Ok(None)` returns are missing stream files, also before the
  write.

- **Replay side.**
  - `MutableStore::load` (`store.rs` ~382): for an event store it prunes, reads, reduces
    and tracks streams. For a legacy store it imports `issues.jsonl` and marks every
    stream changed.
  - `MutableStore::save` (~429): validates every issue and external-ref uniqueness,
    writes the changed streams, saves config, and rewrites `issues.jsonl` only for a
    legacy store.
  - `append_issue_event` (~553) mints with ordinal `len + 1`.
  - `TrackedEventStreams::stream_mut` creates missing streams.
- **The view** (`view.rs`, 969 lines):
  - Backings are `Cached { cache_path, witness }` and `Replay` over a borrowed
    `&[IssueWire]`.
  - Lookups, all overlay-aware: `resolve`, `get`, `children`, `descendants`,
    `ancestors`, `dependency_targets`, `reverse_dependents`, `external_ref_owner`,
    `next_top_level_counter`, `next_child_id`, `stream_id_for_issue`,
    `projection_receipt_seen`.
  - Staging is `stage_issue`/`stage_removal` only. There is no event staging, config or
    `commit`.
  - `#[allow(dead_code)]` sits at ~37, ~58, ~67 and ~966.
- **Tests.**
  - `tests/support.rs` has `StoreMode`, `mode_store`, `assert_cache_equals_replay`,
    `assert_cached_path_used` and `assert_no_full_replay`.
  - `store_io_stats` (`store.rs` ~900+) counts loads, full replays, hydrated rows,
    stream reads, full sweeps, snapshot loads, signature rows and published rows.
- **File sizes.**
  - Already over 1,500: `read_model/store.rs` 2,174 and `read_model/tail.rs` 1,554.
    `mutation/tests/notes_update.rs` is 1,501 and was unchanged by the epic.
  - Near 1,500: `mutation/links.rs` 1,469, `mutation/close_remove.rs` 1,425,
    `mutation/tests/links.rs` 1,450, `read_model/queries.rs` 1,403.
- **Baseline evidence.** Matched bench, seed 20261006, 20 runs:
  - before: `file:explicit:c6e84b9abb0aa9ec8460fdf0`
  - after the previous epic: `file:explicit:cf5bda21694e0423eeb52bd1`
  - after-numbers at p95: 1x note 114.5 ms, update 156.5 ms; 8x note 321.3 ms, update
    455.5 ms

## Rules for every phase

- Close only your own phase bead. Never close this epic, `sase-1h8.13.1`, `sase-1h8.13`,
  `sase-1h8` or any ancestor. Create no beads. Record discovered work as
  `sase bead note <your-phase-bead> 'PROPOSED FOLLOW-UP: <summary — evidence>'`.
- Do not edit `sase/memory/**`, provider shims or memory templates.
- Keep all shared behavior in sase-core, and read its `AGENTS.md`. Keep every binding
  and wire signature. No sase pin move is needed: no new binding is called.
- No feature flag, daemon, CLI option or config knob. No Python fallback.
- Never run bare `cargo`. Never edit versions or changelogs. `beads.db` stays the
  mutation flock. Never weaken an assertion or regenerate a golden to get a green run.
- No `macro_rules!`. `mod.rs` files stay facades. New files stay at or under 1,500
  lines, and no file over 1,500 may grow. Move changed functions into new files instead.
- **The replay oracle fixes semantics:**
  - outcomes, before-images, ordering, deduplication, error kinds and error text
  - event minting, ordinals and receipt checks
  - physical stream routing: a plan owns its stream, and a non-plan follows its parent's
    stream when the parent exists
  - legacy-store materialization and the `issues.jsonl` rewrite
  - byte-for-byte output on legacy, no-git and unreadable-cache stores
- **Warm-path guarantees stay:** one admission sweep, zero full replays and snapshot
  loads, and hydration bounded by the affected set. The existing warm-path suites must
  stay green with unchanged bounds.
- **Fail-open stays:** after a durable append, a cache problem never fails, retries or
  double-appends a mutation. A cached-path decline is legal only before the first
  durable write.
- Never benchmark against the live bead store, its hidden clone or a sidecar. Use
  generated corpora under `/tmp`.
- **Verify** with the linked-core recipes:
  - `just fmt` and `just fast`
  - targeted runs: `just test -p sase_core bead::mutation`, `bead::read_model`,
    `--test bead_read_parity`, `--test bead_event_parity`,
    `--test bead_read_model_parity`, `--test bead_read_model_mutation_proof`, and
    `just test -p sase_core_py`
  - then `sase tool run check` in every repository you changed. Hand long commands to
    `/sase_monitor`. Run no `check-full`.
- **Load flakes and pre-existing failures:**
  - Treat a check failure that passes alone as a load flake: look for its flake bead
    (known ones: sase-15h for sudo_runner and sase-yn for provider_priority) and cite it
    in a `PROPOSED FOLLOW-UP:`.
  - A failure that reproduces identically on the clean base never blocks your close.
- Before closing, run `sase bead epic-symbols <your-phase-bead>` and resolve any
  entries.

## Phase `suite-modes`: the nine suites in both modes

1. **Fixture support.** In `mutation/tests/support.rs` (or a new sibling if it nears
   1,500), add a mechanism without `macro_rules!` that runs every test in these suites
   once against a git-backed store and once against a plain store: `claims.rs`,
   `close.rs`, `create.rs`, `delegation_remove.rs`, `dependencies.rs`, `links.rs`,
   `notes_update.rs`, `snooze_plus_one.rs`, `store.rs`.
   - The git-backed store has a `.git` dir, so `read_model_cache_path_for_store` finds a
     cache location. Fixtures that commit set a local `user.name`/`user.email`.
   - The plain store has no `.git`, so it runs full replay.
   - Candidate mechanisms:
     - include each suite module twice with `#[path]` under mode-specific parent modules
       that set the mode the fixtures read
     - a mode-parameterized store constructor that every suite fixture calls
   - Choose the least invasive one. Route every store the suites create through it,
     including the `support.rs` fixtures.
   - Lock and contention tests may run once. Tests that hand-build a legacy or corrupt
     store stay replay-only, with a one-line comment saying why.
2. **Parity after each test.** In cached mode, assert after each mutating test that the
   read model equals a full replay (`assert_cache_equals_replay`). A fixture guard whose
   `Drop` skips the check while panicking is fine.
3. **Cached path used.** For tests whose operation has a cached path (every family now),
   assert the cached path ran. Do not add per-test bound assertions; the warm-path
   suites already own them.
4. **Fix what this exposes.**
   - Any divergence between `try_cached_*` and replay is a defect in the cached
     algorithm. The replay copy is the oracle until it is deleted.
   - Fix each one with a regression test. Never weaken an assertion.
   - Keep the fix inside the family file. A shared lookup fix in `view.rs` is allowed if
     it is minimal.
5. **Split oversized files.** Split `mutation/tests/notes_update.rs` (1,501) if you
   touch it, and keep every test file at or under 1,500 lines.
6. **Bead note.** Record the mechanism, how many tests now run per mode, every defect
   found and fixed, and the verify results.

## Phase `replay-goldens`: byte goldens before deletion

1. **New test file and fixture directory** (for example
   `mutation/tests/replay_goldens.rs` plus a `goldens/` fixture directory). It covers
   every entry point listed in the starting point, plus `init_store`/`export_jsonl` as
   controls. For each one, run deterministic scenarios with explicit `now` values and a
   fixed owner and actor:
   - a **success** case, and each **rejection class** the replay code distinguishes (not
     found, ambiguous shorthand, validation, descendant guard, closed target, receipt
     replay and no-op), choosing representative cases rather than every message
   - on a **no-git event store**
   - on a **legacy store** (an `issues.jsonl` with no `events/`), for every entry point
     that is valid there. Legacy stores materialize the event store on first save.
2. **What a golden records:**
   - the serialized outcome or error (kind plus message)
   - a sorted map of every file under the beads dir with its exact bytes or a strong
     digest, excluding `beads.db` and lock and holder files
3. **Regenerator.** An `UPDATE_MUTATION_GOLDENS=1` environment variable rewrites the
   goldens, following the gateway's `UPDATE_MOBILE_CONTRACT` precedent. Without it, any
   mismatch fails with a readable diff naming the scenario.
4. **Generate from the current, untouched replay code.** This phase changes no
   `src/bead/mutation/*.rs` production code.
5. **Bead note.** Record the scenario count per family and the regeneration command.
   State that later phases must not regenerate the goldens; any regeneration needs
   explicit justification in that phase's note.

## Phase `view-commit`: one staging area, one commit, one runner

1. **View owns the store state.**
   - The replay backing owns its `MutableStore`, or an equivalent holding the full issue
     list, tracked streams and config. Build it only when the cached path declines or no
     cache is admitted.
   - The cached backing holds config loaded once, plus lazily loaded streams.
   - Both expose `config()`/`config_mut()`.
2. **Event staging.**
   - Add a single overlay-aware
     `stage_event(issue_id, operation, payload, timestamp, actor)`:
     - Route the stream through `stream_id_for_issue`, using the oracle's parent-exists
       rule against staged state.
     - Load the stream lazily: through `load_mutation_stream` on cached, and from
       tracked streams on replay, where missing streams are created like
       `TrackedEventStreams::stream_mut`.
     - Mint through one helper.
   - Delete the second minting path (`append_issue_event` or `mint_stream_event`) once
     nothing calls it.
3. **One `commit`.**
   - **Cached backing:** today's `commit_staged_write` behavior, including the manifest
     total, writer signatures, config, publish and reducer-truth corrections.
   - **Replay backing:**
     - Apply the overlay to the full issue list in the oracle's order: new issues
       appended in creation order, removals removed.
     - Then run exactly `MutableStore::save`: validate every issue and unique external
       refs, write the changed streams, save config, and rewrite `issues.jsonl` only for
       a legacy store.
   - Commit returns the committed rows: corrected rows on cached, and the staged rows on
     replay.
4. **One runner.**
   - A helper such as
     `run_mutation(beads_dir, name, |view| -> Result<Step<Outcome>, BeadError>)`:
     - takes the flock
     - admits the cached view
     - runs the algorithm
     - on an explicit "needs replay" decline before any durable write, builds the replay
       backing and runs the **same** closure again
   - On the replay backing a decline is a bug: return an io error.
5. **Port create and the whole notes family** onto the runner, as single algorithms:
   `create_issue`, `update_issue(s)`, `append_issue_note`, `edit_issue_note` and
   `remove_issue_note`.
   - Delete their `MutableStore` replay copies.
   - Delete helpers that become unused, except ones other families still call.
   - Leave the other families' `try_cached_*` + replay structure working, so the
     parallel phases start from green.
6. **Tests.**
   - `replay_goldens`, `suite-modes`' dual-mode suites, `view_core`, `publish_direct`,
     `read_model_mutations` and the parity suites all stay green unchanged.
   - Add unit tests for replay-backed `commit` on a legacy store and for the runner's
     decline-then-replay path, asserting one durable write.
7. **Bead note.** Document the runner and staging API in a few lines. Name the functions
   the three unify phases must use, and any family-specific caveats you noticed.

## Phases `unify-lifecycle`, `unify-claims-deps`, `unify-links-evidence`

These run in parallel after `view-commit`. Each moves its family onto the runner, so one
closure serves both backings. Each deletes its `MutableStore::load` replay copies, its
per-family stream-slot helper and its own config loading, and keeps each entry point's
public signature.

To stay merge-safe:

- Keep family helpers in your family's files.
- Do not edit `view.rs`, `shared.rs`, the publish modules, `tests/support.rs` or the
  goldens, unless a genuinely shared piece is missing; then make one minimal addition.
- Put new tests in new family-named files when an existing file is near 1,500 lines.

Every phase must keep these green, unchanged:

- its family's dual-mode suites from `suite-modes`
- `replay_goldens`
- its family's warm-path suite (`lifecycle_warm`, `claims_deps` or `links_evidence`)
  with unchanged bounds
- `proof_bounded` and `tests/bead_read_model_mutation_proof.rs`

**`unify-lifecycle`** (`close_remove.rs`):

- Entry points: `open_issue`, `close_issues`, `close_issues_with_note`, `remove_issues`
  and `remove_issue`.
- Keep the replay-identical `cascade_removed_issue_ids` order, delegated-parent
  completion through the children lookup, and ancestor reopening through the view.
- `reject_unclosed_descendants_in_batch` and `reopen_closed_ancestors` survive only if
  another family still calls them.

**`unify-claims-deps`** (`claims.rs`, `set_ready_to_work` in `store.rs`,
`dependencies.rs`):

- Entry points: launch/wait claims, release, all-or-nothing epic preclaim with its
  rollback data, mark/unmark ready, and dependency and reference add/remove.
- Claims keep raw-ID lookups with no shorthand resolution.
- Blocker status stays on dependency-target point lookups.

**`unify-links-evidence`** (`links.rs`, `plus_one_snooze.rs`):

- Entry points: `add_bead_link`, `set_bead_link_projection(s)`, `remove_bead_link`,
  `add_task_plus_one`, `snooze_task` and `cancel_task_snooze`.
- Fold `apply_prepared_link_projections` into the single projection algorithm.
- Keep canonical `bead:` targets, undirected holder selection, receipt idempotency,
  provenance, +1 deduplication, promotions, the observation window and wake behavior.

## Phase `caps-docs`: file caps and docs drift

1. **sase-core file caps.**
   - Move the functions `publish-direct` (`64605819`) changed or exposed out of
     `read_model/store.rs` and `read_model/tail.rs` into new files under `read_model/`.
     Add `mod` lines only to the facade.
   - Candidates:
     - from `store.rs`: the meta-key helpers (`tail_meta_defaults`,
       `backfill_meta_keys`, `set_token_in_txn`) and the refresh decision
       (`RefreshFinish`, `refresh_changed_store`)
     - from `tail.rs`: the resume-load helpers that publication now shares
       (`TailLoadPlan`, `tail_load_plan`, `IssueIndex`, `load_issue_index`, `load_rows`,
       `query_dependents`)
   - Targets: `tail.rs` at or under 1,500 lines, and `store.rs` at or under 2,099.
     Further splitting of `store.rs` is out of scope.
   - This is a pure move: no behavior change, and all read-model and parity suites
     green.
2. **sase docs.** In `docs/beads.md`, the paragraph beginning "When a phase agent
   auto-approves an epic-tier implementation plan" (~line 2818) must describe the
   current behavior. A phase agent's epic-tier plan parks for human review under
   `%auto:tale`. Once approved, the child epic is created beneath the phase, and the
   phase stays open. Keep planner-guard's sentence, and keep every sentence that
   `tests/test_bead_macro_tags.py` or other doc tests pin.
3. **Verify.**
   - Run `sase tool run check` in sase-core.
   - In sase, read the `lint_and_test.md` reference memory with `/sase_memory_read`,
     then run `sase tool run check`. A failure that reproduces identically on the clean
     base is a `PROPOSED FOLLOW-UP:`, not a blocker.

## Phase `proof`: cleanup, audit, bytes, acceptance

1. **Cleanup on merged master.**
   - Run `sase tool run check` in sase-core first, and fix drift from the parallel
     phases.
   - Delete everything the unification left unused, and remove every
     `#[allow(dead_code)]` in `view.rs`.
   - Rename leftover `try_cached_*`/"cached" names where they now describe the single
     algorithm.
   - Update module docs that still describe a cached-plus-replay pair.
2. **Single-algorithm audit.** By search, confirm and record in the acceptance note:
   - `MutableStore::load` is reached only by the view's replay-backing constructor and
     `export_jsonl`.
   - No entry point holds a second copy of its algorithm.
   - Exactly one event-minting helper exists.
3. **Cached bytes equal golden bytes.**
   - Run each `replay_goldens` scenario on a git-backed store as well.
   - Assert identical outcomes and identical bytes for every non-cache file under the
     beads dir (events, manifest, config).
   - Any mismatch is a defect to fix, not a golden to regenerate. If a difference is
     genuinely legitimate, document why in the test and the bead note.
4. **Rerun the proof suites:**
   - the dual-mode suites, `replay_goldens`, `proof_bounded`, `publish_direct` and the
     warm-path suites
   - `tests/bead_read_model_mutation_proof.rs`, including its slow-gated
     `bench_corpus_sampled_mutations` run locally once on a generated 1x corpus
   - the parity suites, then the full `sase tool run check`
5. **Matched bench, as a no-regression check.**
   - Rebuild the core into the sase workspace `.venv` with
     `SASE_ALLOW_STALE_CORE=1 just rust-install "$PWD/.venv"`, run from the sase
     workspace root.
   - Through `/sase_monitor`, run
     `.venv/bin/python tests/perf/bench_bead_scale.py --scale 1 --scale 8 --runs 20 --only note_append,update --output /tmp/bead-mutations-unify-after.json`.
   - Register the result as an artifact; read the `sase_artifacts.md` reference memory
     first.
   - Compare with `file:explicit:cf5bda21694e0423eeb52bd1`. A regression beyond noise is
     a defect for this phase. The 1x→8x target itself belongs to perf gate
     `sase-1h8.14`; do not chase it here.
6. **Acceptance note** on your phase bead. The resumed landings copy it into the close
   notes of `sase-1h8.13.1` and `sase-1h8.13`. Cover:
   - every family on one algorithm with both backings, plus the audit results
   - suites per mode, the golden and byte-equality results, and the check outcomes
   - the bench table and artifact references

## Landing (this epic's land agent)

1. Verify every phase closed with its evidence. Read the actual source for the
   single-algorithm claim; do not take phase notes on trust.
2. Re-run `sase bead epic-symbols` for this epic. Confirm `sase tool run check` is green
   in sase-core and sase, or that each failure is a documented clean-base reproduction
   or known flake.
3. Triage the phases' `PROPOSED FOLLOW-UP:` notes with `/sase_new_task`.
4. Close this epic. Its `parent_bead` is plan bead `sase-1h8.13.1`. Resume that landing
   with the standard plan-parent checks:
   - re-verify its phases and this epic
   - retire its `--epic-symbol` entries
   - close it with a note that cites this epic's acceptance note and the previous
     landing note (follow-up triage is already recorded on `sase-1h8.13.1`)
   - run `just symvision`
   - set `status: done` in `plan:202610/finish_read_model_mutations_child_epic.md`
5. **Handle `sase-1h8.13`, the next parent: a phase bead of `sase-1h8`.**
   - Confirm its acceptance from the `read-model-mutations` section of the parent plan:
     - affected rows only on the cached path
     - write-through in the same critical section
     - no-cache stores keep full replay, through the view's replay backing
     - every existing mutation test passes in both modes
     - parity asserts cache equals replay after each mutation
     - lock and contention tests are unchanged
     - `store_io_stats` proves no full replay
     - binding-level `append_note`/`update` costs at 1x and 8x are recorded
   - Close it with `sase bead close sase-1h8.13 --note "<summary plus evidence refs>"`.
   - Leave `sase-1h8` to its waiting land agent.
   - Note on `sase-1h8`:
     - DISCOVERED ISSUE #3 needs no further work. `e411a392` fixed the event-store
       expectation, and `64605819` added the legacy-store assertion.
     - The 1x→8x perf miss is already recorded there as a DISCOVERED ISSUE for
       `sase-1h8.14`.
