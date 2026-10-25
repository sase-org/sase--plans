---
tier: tale
title: Finish watcher error handling and land the published 0.18.1 release
goal: Correct and prove the remaining one-off watcher requirements, then close the
  verified release epic and every ready plan ancestor.
size: medium
proposed_by: bbugyi200.athena.sase-1nn.8.4.3.5.4.land
proposed_with: codex/gpt-6-astra
bead: sase-1nn.8.4.3.5.4
status: done
---

- **PARENT:**
  [202610/v0_18_1_timezone_and_watch.md](https://github.com/sase-org/sase--plans/blob/main/202610/v0_18_1_timezone_and_watch.md)
- **BEAD:**
  [sase-1nn.8.4.3.5.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1nn/sase-1nn.8.4.3.5.4.md)

# Finish the watcher contract and land the published 0.18.1 release

This is the remaining work of **sase-1nn.8.4.3.5.4**, whose parent is
**sase-1nn.8.4.3.5**. The release has already succeeded. This tale must finish the
watcher corrections below and **perform this epic's landing itself**. There is no later
land agent for the tale. Do not wait for this turn's eventual commit, push, or CI result
before closeout.

The original plan is `plan:202610/v0_18_1_timezone_and_watch.md`. Its timezone phase is
complete and requires no further changes. The published release and fresh installation
have been independently verified. The remaining coding scope is the one-off shell
watcher and its offline harness, not a new release service or backend.

## Verified starting point

The landing audit at fetched master `db31bf43dc5e84f704c5200cb28ec1aec506d4e5` read this
epic, both children and every note, the linked plan, actual source and commit
`abe1bd456e`, subsequent commits, the parent landing note and plan, and all three
sibling phases of that parent.

- All three question timestamp fallbacks use `sase.core.time.parse_local` and derive
  deadlines from the normalized epoch. The six affected question test files pass: **73
  under TZ=UTC and 73 under TZ=America/New_York**. The phase's recorded
  `sase tool run check` was `ed51ac320236fc64ab2175dc34cef60a`, pass. There is no
  remaining timestamp or screenshot work.
- Post-implementation drift is `9e5b384caa` (release metadata), `cc86e1cd37` (release
  branch core metadata), `7423fa8380` (master's exact core floor), and merge
  `db31bf43dc`. No new consumer bypasses the timestamp adapter. Version, dependency,
  lockfile, and source pin agree. No product integration edit is needed.
- PR321 merged at **2026-10-25T16:19:06Z**, commit/tag **db31bf43dc**. The `ci_watch`
  decision record **20261025T121825_980208** names PR321, release head
  `cc86e1cd37ef11ae5906b00988f3aab792348463`, `merged_pr=321`, `reason=merged`, and
  merged count 1. This proves the authorized merger independently of the GitHub account.
  GitHub v0.18.1 was published at 16:19:51Z.
- Master Gate **40188728401**, attempt 1, and Full CI **40189680772**, attempt 2,
  succeeded on qualifying master **7423fa8380**. Full CI completed at 16:15:03Z, before
  the merge. Publish **40211137125** succeeded on the release commit; all six jobs,
  including both install smoke jobs and publishing, succeeded.
- PyPI has unyanked sase **0.18.1** wheel and sdist, uploaded at 16:21:14Z and
  16:21:19Z. The dependency is exactly **sase-core-rs==0.42.6**. Core tag
  `7c0a7b8f25d0418047b48a2573b4b64145fcbde0` contains source pin
  `3156d11f33d5a5277ff806a78b92a9490f78d6be`; all four required platform wheel suffixes
  plus sdist are present and unyanked.
- Actual guarded core cut **40178573904** merged PR337 at 09:27:30Z; resulting
  build/publication **40180720930** succeeded at the core tag. This fulfills the
  parent's previously missed containing-core cut. Preserve its historical notes.
- A fresh Python **3.12.11** venv installed `sase==0.18.1` from PyPI with
  `UV_NO_CACHE=1`, temporary isolated HOME supplied only to child processes, and no
  global installation changes. Core **0.42.6** was installed; `sase core health --json`
  exited 0 with `status=ok`, `probe_ok=true`, and `rust_extension_loaded=true`.
- The original watcher finished RELEASED and closed phase **sase-1nn.8.4.3.5.4.2** done.
  Do not restart the operational watch, rerun remote CI, dispatch workflows, or
  republish anything for this tale.

Audit-read the readable report **file:explicit:fed0aeb43856ff96f2c10495** and the
accompanying archive **file:explicit:fc94b2bb95274482414ab2f3**. Both are recorded in
the newest note on **sase-1nn.8.4.3.5.4**. The archive contains PyPI metadata/hashes,
GitHub run/job facts, ci_watch's merge decision, clean-install and timezone logs, the
complete original watcher log/state/scripts, original test stubs, and five new
regression probes. Use `sase artifact path` only to supply the audited archive to
extraction; extract to a fresh temporary directory. Never reuse an old workspace path
from a log.

## Remaining defects and their evidence

The immutable original watcher is `file:explicit:779c018d76cca5d8d93d8d3f`, wrapper
`file:explicit:94ffb1bee494b28f971676cd`, and original 52-check result
`file:explicit:11e532db57f8414644dab3e5`. The audit independently reproduced:

1. **Pending checks accepted.** `eligible_core_pr` rejects only `bucket=fail`; a
   MERGEABLE/CLEAN release PR with a pending check caused a stub core dispatch.
2. **Fetch failures permit actions.** Both fetch failures explicitly continue with
   cached refs/tags. A failed-fetch scenario still issued a stub Master Gate rerun from
   the cached head.
3. **PyPI transport failure permits actions.** `tick` reads `$?` after an
   `if ...; then ...; fi` compound, losing `pypi_release_ok`'s return 2. A transport
   error still caused a stub core dispatch.
4. **Ambiguous dispatch repeats.** Cut/publish action tokens are saved only after a
   successful CLI response. Two ticks with a failed dispatch response issued two stub
   core dispatches for the same pin.
5. **Preflight consumes a rerun token.** A preflight seeing failed attempt 1 saved
   `gate_reruns=" 111"` while `act` suppressed the operation. The next watch tick never
   issued its permitted rerun. The live log similarly contains no Master Gate rerun
   despite its saved token for run 40174846552.

The same source audit found related unfinished requirements:

- Git fetch/ancestry, mutation calls, artifact/close calls, and wrapper status calls
  bypass the timeout wrapper. The promised subprocess bounds therefore do not hold.
- `seen_any` is not reset each tick; a prior successful observation can turn a later
  wholly unreadable core inventory into confirmed lag. Partial unknown newest-version
  evidence must not silently establish absence either.
- A requested Full CI rerun can be counted red again while GitHub still reports its old
  attempt. Only observed final attempts may vote; a requested rerun with no terminal new
  attempt is pending/unknown, not another red vote.
- The actual core dispatch returned run **40178573904**, but the immediately following
  run-list query saved older **40136208288** as `cut_run`. Preserve the real action
  identity, or explicitly record unknown; never pick an old run.
- Monitor-side `sase artifact create` failed because monitor processes lack
  `SASE_AGENT=1`; the script ignored it. The lander has now retained the full original
  log, so evidence is recovered. Do not spoof agent identity to bypass that guard.

These are unfinished requirements of this epic, not unrelated follow-up tasks.

## Implementation and proof

1. Re-read the current epic, both children, newest parent note, original plan, and this
   audit through the audited CLIs. Read `lint_and_test.md` and `sase_beads.md` with
   `/sase_memory_read`. Open other repos only through `/sase_repo`; use the returned
   paths. Inspect current master drift before editing. Keep the already verified
   timestamp code and release facts intact.

2. Make corrected copies of the one-off watcher, wrapper, and harness in a fresh
   temporary directory, then retain the corrected deliverables as explicit artifacts
   attached to this epic. Preserve the immutable originals as history. Make the harness
   resolve its own root; the archived original harness and audit probes contain
   temporary paths that must be relocated before use. This is bounded operational shell
   glue. Do not add a generic Python release policy engine, change the Rust core,
   promote these scripts to a new product service, or change the already published
   package.

   Correct the diagnosed logic together:
   - Capture return statuses directly; reject unknown/unreadable inventory and
     unsuccessful fetches before actions. Reset per-observation scratch state. Keep the
     overall persisted deadline enforceable on every error path; do not charge unknown
     observations as confirmed core/gate failure duration.
   - Require successful check reads and every check success/skipped. Pending, canceled,
     missing/malformed or failing evidence cannot authorize a cut. Validate exact
     expected wheel suffixes and release version, not substring coincidence or file
     counts.
   - Reserve action budgets durably before a real attempt; after an ambiguous response,
     retain the reservation and reconcile read-only observations. Preflight must not
     consume action budgets or alter existing watch state. Preserve maximum one core cut
     per pin, one rerun per run, two metadata refreshes at least an hour apart, and one
     permitted post-merge dispatch. Keep exact head/run identities and final-attempt
     Full CI voting.
   - Bound every external read and mutation, including git and close/status operations.
     Preserve bounded idempotent closure, markers only after confirmed closure, and
     visible failure/recovery instructions. Keep the original outer timeout ordering. A
     signal during child execution must still leave time for the wrapper's bounded close
     path.
   - Obtain a returned dispatch run identity when available; otherwise reconcile only
     uniquely matching runs created after the attempted action. Unknown identity never
     resets its action budget or claims an older run.
   - Separate monitor-local log retention from agent-only artifact indexing. State
     explicitly that a root lander registers the retained log with
     `sase artifact create`; return the path/status visibly. Do not claim an artifact
     was created when registration failed. For this completed watch, the audit's already
     retained log satisfies the retention requirement.

3. Run the original transition suite against the corrected copies and extend it with the
   five reproduced scenarios, pending/canceled/malformed check responses, failure of
   each fetch, unknown inventory after a prior good tick, ambiguous cut/publish response
   across restart, preflight-to-watch state, delayed rerun visibility, wrong/stale
   dispatch run candidates, bounded hanging reads/actions/close, and wrapper fallback
   after timeout or signal. Retain both the executable harness and results. Every
   GitHub, curl, git and bead mutation must be stubbed in these tests. Use a short test
   timeout rather than waiting on production budgets. Prove each regression fails
   against the original and passes against the corrected copy. Test all happy paths
   remain valid. Do not start another live watch to test a completed release.

   Read-only PyPI/GitHub observations may refresh the retained release facts. Reuse the
   already successful clean-install evidence unless new evidence raises a concrete
   concern. If any tracked sase file changes, run `just fix` followed by
   `sase tool run check` (the guarded `just check`). Do not run `just check-full`,
   weaken assertions, rebaseline unrelated images, or wait for this turn's future
   commit/CI.

4. Append a completion note explaining each correction and linking corrected
   scripts/proofs plus the original release evidence. The direct phase beads contain
   **zero PROPOSED FOLLOW-UP entries**. The prior parent's .5.1 note 2 is already routed
   to **sase-1od**, and note 3 to **sase-1ic**; retain those dispositions and do not
   create duplicates. Earlier ancestor proposals are disposed in **sase-1nn.8.4.3 note
   1**. Newly discovered unrelated work must use `/sase_new_task`; defects caused by
   this epic remain epic work.

## Final step: perform the complete landing in this turn

This final step is required even though the tale has no lander. It must happen before
the host makes this turn's eventual commit.

1. Recheck descendant and linked-plan readiness for **sase-1nn.8.4.3.5.4**, including
   every child note and the corrected watcher proof. Run
   `sase bead epic-symbols sase-1nn.8.4.3.5.4`. Resolve each entry through a real
   consumer, privatization, non-test pragma, or deletion per Symvision policy, or re-key
   only to a still-open later bead that actually needs it. Read `symvision.md` before
   any whitelist repair. There were no entries at the audit, but check again.
2. Close normally with
   `sase bead close sase-1nn.8.4.3.5.4 --note "<source/drift verification, corrected watcher evidence, actual publication and clean-install proof, follow-up dispositions>"`.
   Run `just symvision`. Through the plans checkout obtained from
   `sase repo open plans`, set **status: done** in
   **202610/v0_18_1_timezone_and_watch.md**. Read its content first with
   `sase artifact read plan:202610/v0_18_1_timezone_and_watch.md "Need final status update context"`.
3. Inspect its current parent link. The expected parent is the plan bead
   **sase-1nn.8.4.3.5**. Re-read its previous landing note, all descendants and every
   note, **plan:202610/v0_18_1_release_integration.md**, and post-child master drift.
   Rerun descendant/linked-plan readiness. Record that the missed containing-core cut
   was fulfilled by **40178573904**, followed by successful core publication
   **40180720930**, with original phase .2 notes preserved. If its canceled watch
   **sase-1nn.8.4.3.5.3** requires done for readiness, reopen it and close it normally
   with actual release evidence, preserving historical cancellation notes. Run
   `sase bead epic-symbols sase-1nn.8.4.3.5`, resolve/re-key every entry, and when
   complete close with
   `sase bead close sase-1nn.8.4.3.5 --note "<full readiness/drift recheck, release proof, prior follow-up dispositions>"`.
   Run `just symvision`; mark **202610/v0_18_1_release_integration.md** done through the
   opened plans checkout.
4. Repeat the full readiness audit, normal close, `just symvision`, and linked plan
   status update through directly parented plan ancestors in this order:

   | Plan bead      | Audited plan reference                 | Canceled watch to re-evaluate |
   | -------------- | -------------------------------------- | ----------------------------- |
   | sase-1nn.8.4.3 | plan:202610/v0_18_1_release_unblock.md | sase-1nn.8.4.3.4              |
   | sase-1nn.8.4   | plan:202610/v0_18_1_final_goldens.md   | sase-1nn.8.4.2                |
   | sase-1nn.8     | plan:202610/v0_18_1_release_landing.md | sase-1nn.8.3                  |
   | sase-1nn       | plan:202610/v0_18_1_release.md         | sase-1nn.7                    |

   For each, read previous landing notes, every descendant and note, its linked plan,
   actual implementation and post-child drift. Check `sase bead epic-symbols <id>`
   before closing; retire or legitimately re-key every entry. Reopen/close canceled
   watch phases normally only when actual completion is proved and readiness requires
   it. Use `sase bead close <id> --note "<what was rechecked and retained evidence>"`,
   then `just symvision`, then mark that linked plan done. Stop at the first incomplete
   or ambiguous ancestor, append its precise blocker, and report it. Do not interpret
   closed-canceled as done. If the actual parent link is instead a phase, verify/close
   only that phase and leave its containing epic to the waiting lander. **Never
   force-close to advance this successful nested landing or merely make a command
   succeed.**

All corrected scripts/proofs must be durable artifacts, and all edited plan repositories
must be included in `/sase_final`'s host-owned commit declaration. No manual commit,
push, merge, release dispatch, steward activation, ci_watch configuration change, or
global install is part of this tale.
