---
tier: tale
title: Finish artifact-link index landing repairs
goal:
  Restore tolerant parity and worker-safe lazy queries, verify the integrated index, and
  close sase-1kj.8.1 and parent phase sase-1kj.8.
size: medium
proposed_by: bbugyi200.athena.sase-1kj.8.1.land
bead: sase-1kj.8.1
create_time: 2026-10-13 21:53:11
status: wip
---

- **PARENT:**
  [202610/artifact_link_inverted_index.md](https://github.com/sase-org/sase--plans/blob/main/202610/artifact_link_inverted_index.md)
- **BEAD:**
  [sase-1kj.8.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1kj/sase-1kj.8.1.md)

# Finish artifact-link index parity and worker integration, then land sase-1kj.8.1

Complete the remaining work in epic **sase-1kj.8.1** and close its parent phase
**sase-1kj.8** in this same coding turn. This is bounded repair of an already
implemented native index, not another migration. There is no tale land agent. Do not
wait for this turn's commit SHA, push, or CI before closing; the host commits after the
turn ends. Never close the containing epic **sase-1kj**.

## Context and verified work

Read `sase bead read sase-1kj.8.1 -r "Need landing audit and follow-up dispositions"`
and
`sase artifact read plan:202610/artifact_link_inverted_index.md "Need original index contracts"`.
The lander read every child and every note, native and Python source, and the epic
commits: sase `bad0d0ac36` and `16252682b2`; sase-core `daad8f26` and `d776d751`. At
audit, sase HEAD was `d69cd499dd`, with core pin
`d776d751046f26c212898a8520d9eccc9028e1a0`. All three child phases are closed. The
original epic's accepted decision remains **fallback_seconds = sec_45**. No memory,
feature-flag, release-version, or global-install changes are needed.

Already verified: native compilation consumes loaded rows, preserves separate chip and
raw-row views, queries indexed candidate ordinals, and uses a frozen Arc handle with GIL
release. LinkSubject and Record consumers share the native handle; hidden Patch requests
do not compile the graph. Sanitized parity and existing single-flight tests pass.
Real-data artifacts use the same 41,295-row input hash and report zero chip/edge
differences. The live input has no Patch refs, so its zero-edge Patch parity is
complemented by the nonempty fixture. Do not overstate that evidence as proof of
malformed-row coercion or UI scheduling.

Read evidence `file:explicit:adbc4a4bfd916ef633b7c1a7` (after timings/parity) and
`file:explicit:b40f51f8be770b1faebef99a` (baseline) through `sase artifact read`. The
landing audit artifact **file:explicit:405b9f024864d71654e1cb54** includes deterministic
parity and Event-based concurrency reproducers. It records 54 passing focused tests plus
the import-budget failure, and distinguishes the earlier five-failure phase check from
its later finalizer-contaminated check. Follow-up dispositions are recorded on the epic
before this plan is proposed; carry them into the close note.

Completed follow-up triage (do not file duplicates): phase `.1` note #2's stale
`sase-1kj.7` symbol is already removed and consumed by telemetry; phase `.3` note #1's
manifest mismatch is fixed by `96090a52cf` and passes. Those two proposals were declined
as resolved. Phase `.3` note #2's old baseline import overage was corroborated on
**sase-1ic**, while this epic's own +2 modules remain step 4 work. For phase `.3` note
#3, the registry flake corroborated **sase-1em** and the reporter abandoned-index-lock
flake corroborated **sase-1kr** (both caller paths use `run_with_git_lock_retry`; the
new log identifies permission denial while reading `/proc/<pid>/fd`). Its AcePage
missing `update-check` boundary was routed as a `DISCOVERED ISSUE` to active containing
epic **sase-1kj**, whose phase `.6` introduced the sequential startup maintenance chain.
This differs from **sase-1kf**'s leaked module override. All three flaky nodes passed
focused reruns. The later finalizer-environment refusal cluster additionally
corroborated **sase-19g**. These are recorded +1/parent-note outcomes; no new tasks were
needed.

Before editing, read required memory via `/sase_memory_read`: `tui.md`, `tui_perf.md`,
`lint_and_test.md`, `symvision.md`, and `sase_beads.md`. Open the linked Rust checkout
with `/sase_repo` and read its AGENTS.md. Do not access another repo by a guessed path.
Keep all shared parsing/domain fixes in `sase_core`; Python owns scheduling, caches,
styling, and dataclass conversion.

## 1. Restore tolerant read-model parity in Rust

In `crates/sase_core/src/artifact_link/index/compile.rs`, `parse_uses` and
`string_field` currently differ from the old Python walk. The lander loaded the legacy
module from sase `bad0d0ac36` using `git show` and compared it to the current native
binding in the same interpreter:

| Input                  | Legacy     | Current native |
| ---------------------- | ---------- | -------------- |
| `uses=True`            | 1          | 0              |
| `uses=2.9`             | 2          | 0              |
| `uses="1_000"`         | 1000       | 0              |
| `description=False`    | empty      | `false`        |
| `description=True`     | `True`     | `true`         |
| `description={"a": 1}` | `{'a': 1}` | `{"a":1}`      |
| `origin=0`             | empty      | `0`            |

Recover the exact prior aggregate-reader semantics for JSON-representable inputs,
including truthiness/string conversion, count truncation and tolerated numeric strings.
Examine large/negative counts, nulls, lists/maps, boolean fields, malformed refs and
project hints rather than merely special-casing these examples. Keep strict mutation
validation unchanged. Use the old committed walk only to generate small additional
expected cases; do not retain a Python backend fallback or a second production parser.
Add core and PyO3/Python regression cases that pin visible chip fields, duplicate winner
selection, and projected edge metadata. Preserve the intentional difference between
max-uses chip winners and first-row pane edges. Keep ordinary fixture outputs unchanged.

Use the existing Rust wire/domain helpers wherever possible. If an additive wire change
is needed, coordinate callers and binding tests. Rebuild the checkout-local extension
through supported `just` recipes, never bare cargo or a global install. Follow
`docs/rust_backend.md`: declare both changed repos so the host commits core first and
ratchets the primary pin. Never invent an uncommitted SHA, change release versions, or
raise the published wheel dependency floor for these unpublished fixes.

## 2. Keep lazy chip queries off the UI thread and reuse their cache

`src/sase/ace/tui/actions/link_subject.py:link_edges_for_selection` and
`link_follow_available_for_selection` still call `index.chips_for(subject.ref)`
synchronously. `_schedule_link_index_refresh` only builds the lazy wrapper on its
worker. Thus the new native output conversion, Python chip creation and styling reach
the rail/footer UI path on the first query. Fix first selection, subsequent navigation,
cache eviction, and aggregate refresh, not just initial prefetch.

Use the existing pump-free scheduling/coalescing approach. Capture the requested ref on
the UI thread, perform the potentially large query/styling on a worker, then re-read
selection and generation after the await before publishing. UI accessors must return
cached results or schedule the missing work without waiting or displaying the previous
subject's chips. Preserve link-follow/rail behavior, absence vs non-selectable
membership, startup/activity gates, cancellation and failure recovery. Do not
materialize all refs or add a full-graph walk.

`link_index_for_snapshot` currently constructs new `_ChipMapping` wrappers on every
native-handle hit, throwing away the 128-ref cache. Reuse the bounded lazy view for the
same handle/context so repeated rail and Record requests retain requested chips. Keep
scope/context invalidation correct and memory bounded. Capture presentation/config
context on the worker; lazy lookup must not initiate filesystem/config reads through
document descriptor/style resolution. The Record path
(`widgets/decks/record/relations.py:load_relations_extras`) already runs chip queries on
its worker; preserve it and test simultaneous rail/Record access.

Add deterministic tests recording thread identity and using Event/barrier control: no
cold chip materialization on the UI thread; moving from A to B during a query never
applies A's result to B; high-degree refs, cache miss after eviction, refresh races,
spawn/query failure, and teardown release their guards. Assert same-handle repeated
requests reuse the lazy cache and shared native compilation still happens once. Do not
use wall-clock timing thresholds for these guarantees.

## 3. Fix generation fencing and real-build accounting

In `relations/_native_link_index.py`, publication compares sequence numbers only against
the currently cached entry. An Event-based reproduction with cache limit 1 holds
`p/signature1`, completes `p/signature2`, evicts p by requesting q, then releases
signature1: the older p generation is published again. Clear also loses publication
history. Preserve generation/clear-epoch fencing for pending builds even when their
cache slot is evicted or cleared, without accumulating permanent unbounded per-scope
tombstones. Pending callers must still receive their owned result/error and be released;
clear/replacement must not let old work repopulate a newer generation. Keep registry
locks out of native calls/waits and different scopes independent. Add explicit eviction,
clear, replacement and failure races.

In `actions/patch/_relation_index.py`, `_REAL_BUILD_COUNT` increments only when a
completed build wins memo publication. Two distinct successful builds completing
newer-before-older report one. Count each actual successful full build once, including
superseded results, but never count waiters, hits, or targets-only requests. Test the
out-of-order case and preserve full-over-targets-only semantics, strong Patch object
identity, hidden deferral and reveal. Audit clear while a build is pending for
consistent counter/memo lifetime semantics.

## 4. Integrate current startup imports and reverify

The lander's `.venv/bin/python tools/tui_import_closure --diff 16252682b2^` measured
3519 modules before the integration commit and 3521 now. The two added imports are
`sase.ace.tui.relations._native_link_index` and `sase.core.artifact_link_index_facade`.
The current strict cap is `<3520`. Defer these worker-only imports at their use sites,
including transitive top-level edges, instead of raising the cap. Test the real
fresh-interpreter closure.

The reviewed non-epic commits since the baseline are `8581d2313f` (rescue tags),
`9271ed1ed9` (single SQLite owner), `96090a52cf` (manifest-v2), `6da4076b8e` (plural
Artifacts labels), `f7458363b1` (Patch loading split), and `d69cd499dd` (instruction
delivery). No origin/master commits were ahead at audit. Core's intervening `973017b6`
manifest-v2 change is included by the pin. Retain the split
`_loading_apply.py`/`_loading_refresh.py` locations, `ARTIFACTS_LABELS`, single SQLite
ownership and sec_45 choice. Review subsequent drift before closing.

Run focused tests covering the repaired behavior and existing link index/parity, all
pane projection goldens, Patch memo, Record, LinkSubject/link-follow and import-budget
paths. Extend the sanitized compatibility fixture without replacing its old
expectations. Recompare the saved real input with the baseline using the existing
helper, through audited artifact reads, and retain parity evidence. Report remaining
materialization cost honestly; timing repetition is only needed for a changed
performance path. No rendered output change is intended; follow the screenshot
memory/workflow if output or screenshot coverage changes.

Run `just fmt` (or `just fix`) then `sase tool run check` in each changed repo. This is
the wrapped `just check` gate. **Do not run `just check-full`.** Use `/sase_monitor` if
a gate cannot fit the turn, with continuation explicitly carrying this entire tale's
closeout obligations. Never end with a live unmonitored command. Treat unexplained
failures as yours until reproduced/classified. Existing sase-19g tracks finalizer-env
contamination of tests; do not misreport the prior 267-failure finalizer run as a
product parity pass. Record new gate results and handle truly unrelated failures under
the normal duplicate/causal triage policy.

## 5. Close this epic and only its parent phase in this turn

This is mandatory final work of the tale, before the host commits its code:

1. Re-read `sase-1kj.8.1` and every descendant/notes, confirm no open descendants or
   unresolved issues within this epic remain, and recheck its linked plan's completion
   criteria. Carry forward the lander's follow-up triage outcomes and add the repair
   verification and any post-audit integration changes to the note.
2. Run `sase bead epic-symbols sase-1kj.8.1`. Resolve every entry by wiring,
   privatizing, deleting, or a justified non-test pragma per Symvision policy; re-key
   only to a still-open later bead that actually needs the exemption. Then run
   `sase bead close sase-1kj.8.1 --note "<concrete verified contracts, checks, parity/timing limitations, drift review, and every follow-up disposition>"`.
   Never force merely to succeed or to advance successful nested landing. If close
   rejects stale symbols, finish them and retry. If phases are incomplete, finish them
   or deliberately resolve cancellation/supersession with evidence; do not force a
   successful completion.
3. Run `just symvision` after closing. Open the plans repo with
   `sase repo open plans -r "Mark verified artifact-link epic plan done"`, audit-read
   `plan:202610/artifact_link_inverted_index.md`, and set **`status: done`** in the
   frontmatter of **`202610/artifact_link_inverted_index.md`** under that printed plans
   checkout. Preserve other frontmatter and managed content. Include this changed plans
   repo in finalization.
4. Re-read `sase-1kj.8.1` for its parent link and `sase-1kj.8` for required work. The
   concrete parent is the phase **sase-1kj.8**, whose scope is native link indexing,
   shared single-flight adapters, and pin advancement. Verify this child has completed
   it, resolve/re-key any `sase bead epic-symbols sase-1kj.8` entries, and close only
   that phase normally with
   `sase bead close sase-1kj.8 --note "<native index, worker/lazy integration, single-flight/deferral, parity, checks, core pin and limitations verified>"`.
   Leave containing epic **sase-1kj** open for its waiting land agent, including the
   startup-test follow-up routed to it. If the phase is unexpectedly incomplete or its
   parent relationship has changed, record the blocker and stop upward closure rather
   than forcing it.
5. Use `/sase_final` as the last action of normal completion. Declare every changed repo
   (sase, sase-core, plans) with host-owned commits. No manual commits, SHA/push waits,
   or CI-before-close dependency. Report what closed and any remaining parent-epic
   blocker honestly.
