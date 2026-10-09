---
tier: tale
size: small
title:
  Fix the replay-golden cap and lock_wait_ms flake, then land sase-1h8.13.1.9,
  sase-1h8.13.1 and sase-1h8.13
goal:
  The epic's replay-golden test file is back under the 1,500-line cap, and its goldens
  no longer flake on wall-clock lock_wait_ms. Then epic sase-1h8.13.1.9, its parent plan
  sase-1h8.13.1 and phase sase-1h8.13 close with verified evidence, and their plan files
  are marked done.
proposed_by: bbugyi200.athena.sase-1h8.13.1.9.land
bead: sase-1h8.13.1.9
create_time: 2026-10-09 03:56:35
status: wip
---

- **PARENT:**
  [202610/unify_bead_mutation_algorithms.md](https://github.com/sase-org/sase--plans/blob/main/202610/unify_bead_mutation_algorithms.md)
- **BEAD:**
  [sase-1h8.13.1.9](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1h8/sase-1h8.13.1.9.md)

# Plan: finish landing epic `sase-1h8.13.1.9` and resume its parent landings

## Context

The land agent for epic `sase-1h8.13.1.9` ("One mutation algorithm per entry point, with
every suite in both modes", `plan:202610/unify_bead_mutation_algorithms.md`) checked all
eight phases against the source in the linked sase-core checkout (HEAD `2d009388` =
`origin/master`):

- Every ordinary mutation entry point runs one closure through `runner::run_mutation`.
- The `try_cached_*` copies are gone.
- `MutableStore::load` has exactly two non-test callers, `view.rs` `load_replay` and
  `store.rs` `export_jsonl`.
- `shared::mint_stream_event` is the only mint helper, and `view.rs` has no
  `#[allow(dead_code)]`.
- The dual-mode suites, the 155 goldens and the cached-equals-golden test exist.
- `read_model/store.rs` is 2,047 lines and `tail.rs` is 1,317.
- The `docs/beads.md` fix is in sase `9ed4b0fb93`.
- `sase tool run check` in sase-core is green (run `bdda1b99a55dad64036bb4e389e3817a`).
- `sase bead epic-symbols` reports no entries for `sase-1h8.13.1.9`, `sase-1h8.13.1` or
  `sase-1h8.13`.
- Follow-up triage is already done and recorded in the `LAND TRIAGE` note on
  `sase-1h8.13.1.9`. Do not re-triage.

The epic itself left two problems, and this tale fixes both:

1. **Over-cap new file.** `crates/sase_core/src/bead/mutation/tests/replay_goldens.rs`
   is 1,522 lines. The epic created it, and its rule is "New files stay at or under
   1,500 lines". The proof phase grew it from 1,399 when it added
   `cached_golden_bytes_match_replay`.
2. **Golden load flake.** The goldens serialize the whole `BeadMutationOutcomeWire`,
   including `lock_wait_ms`. That field is the measured `beads.db` flock wait, which
   `with_bead_mutation_lock` copies in from `lock.waited_ms()`.
   - Every committed golden pins `"lock_wait_ms":0`.
   - Under a loaded parallel lane, the uncontended flock wait reads 1-18 ms. Phases
     `.9.4` and `.9.8` saw `update_title_legacy` at 18 ms and `release_event` at 5 ms,
     and `bead::mutation` went red.
   - `crates/sase_core/tests/bead_read_model_mutation_proof.rs` (~line 86) already sets
     the precedent: it filters `lock_wait_ms` out as "contention telemetry, not mutation
     semantics".
   - `mutation/tests/links.rs` (~line 1093) still asserts `lock_wait_ms > 0` under real
     contention, so the field keeps its coverage.

All code paths below are relative to the linked sase-core checkout. Open it with
`sase repo open sase-core -r "<why>"` and read its `AGENTS.md`. Never run bare `cargo`.

## Step 1: Normalize `lock_wait_ms` in the golden comparison (sase-core)

In `crates/sase_core/src/bead/mutation/tests/replay_goldens.rs`:

- In `outcome_string`, which both `replay_golden_bytes_are_pinned` and
  `cached_golden_bytes_match_replay` use, set `outcome.lock_wait_ms = 0` before
  `serde_json::to_string`. Add a short comment that it is environmental flock-wait
  telemetry, citing the proof-test precedent.
- Add one sentence to the module doc next to the `remove_*` normalization paragraph, so
  "every other byte is exact" stays true.
- **Do not regenerate any golden.** Every committed golden already pins 0, so the files
  stay byte-identical. Confirm with
  `git status --short crates/sase_core/src/bead/mutation/tests/goldens/`, which must
  print nothing.

## Step 2: Bring `replay_goldens.rs` under 1,500 lines (sase-core)

Do a pure move with no behavior change:

- Move the scenario table into a new sibling module,
  `crates/sase_core/src/bead/mutation/tests/replay_golden_cases.rs`. The table is the
  `Case` struct, the `case`/`control_case`/`remove_case`/`mk` constructors, the act
  helpers `upd_title` … `snz_cancel`, and `cases()`.
- Register it in `mutation/tests/mod.rs` with `mod replay_golden_cases;`, keeping the
  list sorted.
- Widen only the items the other file needs to `pub(super)`.
- Keep in `replay_goldens.rs`:
  - the harness: constants, `SeedIds`, `Backing`/`Backings`, `SeedKind`, seed builders,
    snapshot and normalization, `diff_report` and the `run_case*` functions
  - both `#[test]` functions, so the documented regenerate filter
    `bead::mutation::tests::replay_goldens` still selects them
- Both files must end at or under 1,500 lines, and no other file over 1,500 may grow.
- No `macro_rules!`.

## Step 3: Verify (sase-core)

- `just fmt`, then `just fast` with zero warnings.
- `just test -p sase_core bead::mutation`. All tests pass, including
  `replay_golden_bytes_are_pinned` and `cached_golden_bytes_match_replay`, and goldens
  are unchanged.
- `just test -p sase_core --test bead_read_model_mutation_proof`.
- `sase tool run check` in the sase-core checkout. It takes about 10 minutes; give it a
  long explicit timeout.
  - A failure that passes alone is a load flake. Look for its flake bead with
    `sase bead search`; `sase-17n` and `sase-1ik` are known.
  - Never weaken an assertion, and never regenerate a golden.
- Run `wc -l` on both golden files and record the counts.

## Step 4: Close epic `sase-1h8.13.1.9`

1. Run `sase bead epic-symbols sase-1h8.13.1.9`. Resolve or re-key any entry per the
   Symvision epic-whitelist policy. The land agent saw none.
2. Close it. Never use `--force`.

   ```bash
   sase bead close sase-1h8.13.1.9 --note "<verification>"
   ```

   The note must cover:
   - **Single algorithm.** Every ordinary entry point goes through
     `runner::run_mutation` on both backings, and no `try_cached_*` remains.
   - **Audit.** `MutableStore::load` is reached only by `view.rs` `load_replay` and
     `export_jsonl`. There is one mint helper (`shared::mint_stream_event`) and no view
     `allow(dead_code)`.
   - **Suites and goldens.** 143 of 151 suite tests run in both modes, and 8 stay
     replay-only with stated reasons (phase `.9.1`). The 155 goldens are byte-identical,
     and cached-equals-golden holds (`.9.2`/`.9.8`).
   - **Caps and docs.** `read_model/store.rs` is 2,047 and `tail.rs` 1,317 (`.9.7`).
     `docs/beads.md` is fixed in sase `9ed4b0fb93`.
   - **Bench.** Medians match or improve against
     `file:explicit:cf5bda21694e0423eeb52bd1`; the after-artifact is
     `file:explicit:8ca5f30ead916b9acacf577e`. The full table is in `.9.8` note #2.
   - **This tale's fixes.** Cite the `lock_wait_ms` normalization and the golden-file
     split, with their line counts and sase-core commit context.
   - **Checks.** sase-core `sase tool run check` is green; give the run id.
   - **sase gate on clean master `bd830064ca`.** It failed only in places this epic
     never touched:
     - lint (test waits) is recorded on `sase-1hi.10.7.6`
     - the `sase/memory/README.md` token drift is recorded on `sase-1id`
     - three KNOWN tests are owned by `sase-1hr` and `sase-1i5.9.1.2.1.7`
   - **Triage.** Point to the `LAND TRIAGE` note: `sase-17n` and `sase-1hr` got +1s;
     `sase-1ik`, `sase-1il`, `sase-1im` and `sase-1in` were created; `lock_wait_ms` was
     handled here.
   - **Integration.** No post-start commit needs integration. The only non-epic
     sase-core commit is `4ea91b95` (triage remedies). The sase commits since the start
     are install, ACE, plan-decision and docs work that never touch bead mutation code.
     No sase pin move is needed, because no new binding is called.

3. Run `just symvision` in sase and confirm the whitelist is clean.
4. Set `status: done` in the frontmatter of the epic's plan file,
   `plan:202610/unify_bead_mutation_algorithms.md`. Its path is the PLAN line in
   `sase bead read sase-1h8.13.1.9 -r "<why>"`, under the plans sidecar repo.

## Step 5: Resume the landing of parent plan bead `sase-1h8.13.1`

`sase-1h8.13.1.9`'s `parent_bead` is plan bead `sase-1h8.13.1`
(`plan:202610/finish_read_model_mutations_child_epic.md`).

1. Re-check it:
   - Run `sase bead read sase-1h8.13.1 -r "<why>"` and review its two land notes.
   - Confirm all 8 phases and child epic `sase-1h8.13.1.9` are closed.
   - Re-read its plan's Landing section with `sase artifact read`.
   - Check post-child drift: `git log` in sase-core and sase since `2d009388` and
     `bd830064ca` for anything touching `crates/sase_core/src/bead/mutation/` or
     `read_model/`.
2. Run `sase bead epic-symbols sase-1h8.13.1` and retire any entries. None were present
   at planning time.
3. Close it:

   ```bash
   sase bead close sase-1h8.13.1 --note "<what you rechecked>"
   ```

   The note cites:
   - the previous landing notes #1 (follow-up triage) and #2 (unmet deliverables)
   - that `sase-1h8.13.1.9` delivered both unmet deliverables (a) and (b) and the
     store/tail/docs drift fix
   - the `sase-1h8.13.1.9.8` acceptance note #2

4. Run `just symvision`, then set `status: done` in
   `plan:202610/finish_read_model_mutations_child_epic.md`.

## Step 6: Close phase bead `sase-1h8.13`, the next parent

`sase-1h8.13.1`'s parent is phase bead `sase-1h8.13` of epic `sase-1h8`. Closing
`sase-1h8.13.1` may cascade-close it. Check with
`sase bead read sase-1h8.13 -f compact -r "<why>"`, and close it only if it is still
open.

1. Confirm the acceptance from the `read-model-mutations` section of
   `plan:202610/bead_store_history_independent_performance.md` (read it with
   `sase artifact read`). Map each item to evidence:

   | Acceptance item                                                    | Evidence                                                                                                                                                                                    |
   | ------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
   | Affected rows only on the cached path                              | `proof_bounded`, the `lifecycle_warm`/`claims_deps`/`links_evidence` warm suites, and `tests/bead_read_model_mutation_proof.rs`                                                             |
   | Write-through in the same critical section                         | `publish_direct` suite; `view.commit` → `commit_staged_write` runs inside `run_mutation`'s `with_bead_mutation_lock`                                                                        |
   | No-cache stores keep full replay through the view's replay backing | `MutationView::load_replay`, the replay goldens                                                                                                                                             |
   | Every existing mutation test passes in both modes                  | `sase-1h8.13.1.9.1`                                                                                                                                                                         |
   | Parity asserts cache equals replay after each mutation             | the dual-mode runner's per-test parity assert, plus `bead_read_model_parity`                                                                                                                |
   | Lock and contention tests unchanged                                | kept replay-only per `.9.1`                                                                                                                                                                 |
   | `store_io_stats` proves no full replay                             | the dual-mode and `proof_bounded` asserts                                                                                                                                                   |
   | Binding-level `append_note`/`update` costs at 1x and 8x recorded   | `file:explicit:c6e84b9abb0aa9ec8460fdf0` (before), `file:explicit:cf5bda21694e0423eeb52bd1` and `file:explicit:8ca5f30ead916b9acacf577e`; the bench table is in `sase-1h8.13.1.9.8` note #2 |

2. Close it if it is still open:

   ```bash
   sase bead close sase-1h8.13 --note "<summary plus those evidence refs>"
   ```

   The 1x→8x scaling target is not part of this phase. It belongs to perf gate
   `sase-1h8.14`.

3. Leave `sase-1h8` open for its waiting land agent. Add this note to `sase-1h8`:
   - "DISCOVERED ISSUE #3 (bead_read_parity event-store warning) needs no further work:
     e411a392 fixed the event-store expectation, and 64605819 added the legacy-store
     assertion."
   - "The 1x→8x perf miss is already recorded here as DISCOVERED ISSUE #5 for
     sase-1h8.14."

## Rules

- Do not edit `sase/memory/**`, provider shims or memory templates.
- Create no beads; triage is already recorded.
- Never use `--force` merely to make a close succeed.
- No `macro_rules!`, feature flags or config knobs.
- Never edit versions or changelogs.
- Do not run `just check-full`.
- If a close is rejected, read the reason, fix what it names, and close again.
- If a parent turns out to be incomplete or ambiguous, stop there and record a note on
  that parent that describes the blocker.
