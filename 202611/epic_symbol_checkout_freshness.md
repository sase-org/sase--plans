---
tier: tale
title: Diagnose stale epic-symbol rows before expensive verification
goal:
  Distinguish obsolete checkout rows from live whitelist defects and provide an early
  actionable refusal while preserving close and Symvision enforcement.
size: medium
proposed_by: bbugyi200.athena.sase-1qn.1
proposed_with: codex/gpt-6-astra
bead: sase-1qn.1
status: done
---

- **PARENT:**
  [202611/task_backlog_epic_4.md](https://github.com/sase-org/sase--plans/blob/main/202611/task_backlog_epic_4.md)
- **BEAD:**
  [sase-1qn.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1qn/sase-1qn.1.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-1qn.1](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.sase-1qn.1.md)
- **COMMITS:**
  - [e60f3e1](https://github.com/sase-org/sase-core/commit/e60f3e1658d34f1345bc4bef8bf112a396749c87)
    — feat(bead): classify epic symbol freshness in Rust

# Diagnose closed epic-symbol rows against the checkout being verified

Implement the remaining work for phase **sase-1qn.1**, under parent epic **sase-1qn**,
whose seed is **sase-o7**. Honor the final parent decision
`optional_scope = keep_five_seeds`: this plan includes only the whitelist lifecycle bug,
its evidence, and its regression tests. Do not edit memory, change Symvision's
closed-bead refusal, or change the epic-whitelist policy.

## Finding that changes the implementation

The three October 30 reports named in the parent plan do **not** establish that stale
rows reached master after their beads closed. The cleanup commits precede the closes. An
unrelated agent retained an older source checkout while consulting the live bead store.
Its verification subsequently rejected the obsolete rows.

Read the parent design with
`sase artifact read plan:202611/task_backlog_epic_4.md "Need parent phase scope"` and
read `sase-o7` and `sase-1qn.1` through `sase bead read` before coding. Preserve this
distinction in the completion evidence; a failed check on an old clean base does not
prove the contemporary default branch was broken.

Evidence gathered at sase `078e3da1d6`:

| Bead named by the rows | Introduction and authoring agent                                                                                    | Cleanup commit and commit time (UTC) | Bead close time (UTC) |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------- | ------------------------------------ | --------------------- |
| `sase-1pj.4.1`         | `c85ceb0c3b`, agent `sase-1pj.4.1.6`, re-keyed three workflow rows to the open parent                               | `0060c20fd7`, October 30 09:10:20    | October 30 09:15:20   |
| `sase-1pj.5.1.3`       | `b175b63758`, agent `sase-1pj.5.1.1`, added three prepared-intent rows for the later phase                          | `f7eb80c7a8`, October 30 09:48:21    | October 30 09:53:33   |
| `sase-1pj.5.1.4`       | `b175b63758`, `8f982c8ae9`, and `f7eb80c7a8`, agents `sase-1pj.5.1.1`, `.2`, and `.3`, added or re-keyed eight rows | `7da649b562`, October 30 16:19:49    | October 30 16:57:24   |

The last sequence has particularly strong evidence:

- `sase bead history sase-1pj.5.1.4 --format full` records the initial close refusal at
  10:51:12 UTC: its local cleanup was ready, but the upstream rows still existed. The
  worker explicitly left the bead open for publication. Its later successful close cites
  the published cleanup. The close guard worked.
- ToolRun `3ca0ba01382eb58f81f4b5063766ebe1` started at 16:59:15 UTC for agent `16a--1`.
  Its before/after fingerprints name source HEAD `db37d68096`, committed at 14:14:05
  UTC. `Justfile` is absent from its dirty paths.
- `git merge-base --is-ancestor db37d68096 7da649b562` succeeds. The old HEAD's Justfile
  contains all eight rows; the cleanup drops four and re-keys four to `sase-1pj.5.1`.
  Thus this is an old-checkout failure after successful cleanup and close, not evidence
  of a late stale-row publication.
- Audited artifact `file:explicit:550fb2439c9eec33e4392004` contains that run's eight
  closed-bead errors. Artifact `file:explicit:0dbeaffa58185dbdc6374ece` records the
  other two stale-row groups and the subsequent unused-symbol errors when a worker
  removed only the rows from the old tree. Importing just a newer Justfile would
  therefore be insufficient; the corresponding source changes matter too.
- ToolRun `add90bf79f4ca6a533f3b4d5497f7d2d`, from the reclaim agent's earlier
  verification, records HEAD `302f58e9ad` (08:38:21 UTC), before both earlier cleanup
  commits. Do not mislabel this as the timestamp of the later standalone Symvision run;
  it is supporting checkout evidence only.

Use ToolRun JSON and commit footers to retain original run/author provenance in a phase
note. Do not inspect another agent's current checkout to reconstruct history. Commit
timestamps establish ordering but are not precise remote push timestamps; the successful
close's upstream scan and recorded close notes provide the additional publication
evidence.

## Resulting behavior

Detect a locally retained closed-bead whitelist row at the start of verification, before
the expensive lint/test sequence. When fresh upstream evidence proves that the row was
already removed, refuse with a **stale checkout** diagnostic naming the bead, symbol,
local HEAD, observed upstream revision, and safe recovery: preserve local work,
integrate the upstream cleanup through the normal workspace workflow, and rerun
verification. Do not instruct the agent to delete the whitelist row alone or open a
duplicate master-red task.

Distinguish that case from a newly authored invalid row or a row that actually survives
upstream. Those still need the existing symbol cleanup or legitimate re-keying remedy.
Unknown evidence must remain unknown. A cached or failed upstream read must not be
presented as proof that master is clean.

This is an early, actionable refusal, **not** a successful verification and not an
automatic checkout mutation. The existing Symvision invocation continues to enforce its
full policy, including statuses that change after preflight. This plan does not claim to
prevent every possible concurrent publication/close race, nor treat the historical
reports as evidence for a new publication guard.

## Implementation

### 1. Reproduce the observed sequence before changing production code

Extend the isolated Git fixture approach in
`tests/test_bead/test_cli_close_epic_symbols.py`. Use a local test remote and two
checkouts plus an isolated bead store; do not touch the real bead lifecycle.

1. Publish a row for an open phase; leave an observer checkout at that revision.
2. Remove or legitimately re-key the row in the producer, and publish its cleanup.
3. Close the phase through the real guarded CLI path. Assert that upstream is clean and
   that the close succeeds only after the cleanup is visible upstream.
4. Consult the closed status from the old observer tree. Demonstrate the stale-row
   failure without claiming the remote has the row.
5. After implementation, assert the observer gets the specific freshness refusal before
   expensive verification starts, and that integrating the cleanup removes the refusal.
   Reproduce the distinction between source cleanup and a Justfile-only change using a
   small synthetic symbol fixture where useful.

Keep the historical SHAs in explanatory evidence, not as runtime dependencies of the
regression suite.

### 2. Put parsing and classification in the Rust core

Open `sase-core` using `/sase_repo` and follow its `AGENTS.md`. Add a small domain
module and wire binding for epic-symbol parsing and verification assessment. The Rust
function should operate on supplied texts/rows and observed facts; Python owns bounded
Git and store I/O. Do not add a Python fallback.

Inputs need the local rows, HEAD rows, live status observations, and optional upstream
rows with their revision/freshness and ancestry evidence. Classify each closed local row
into:

- **Stale checkout:** the row is inherited from local HEAD, a successfully fetched
  upstream revision descends from that HEAD, and that revision lacks the row.
- **Local invalid row:** a closed row was introduced locally rather than inherited from
  the old HEAD.
- **Upstream unresolved:** the fresh upstream tree still contains the closed row.
- **Uncertain:** missing status, unavailable Git history, divergent histories, or
  unavailable/stale upstream evidence prevents the stronger conclusion.

Return structured evidence and reasons; Python renders the user-facing diagnostic. For a
known closed local row, uncertainty about upstream does not turn it into an allowed row.
For unavailable bead status, preserve the existing linter's lookup behavior rather than
inventing an open or closed state.

Keep `_parse_epic_symbol_entries` in `src/sase/bead/epic_symbols.py` as a thin binding
adapter so the current close/list paths and the new preflight share parsing. Preserve
its supported quoted, bare, and `=` forms, public row representation, source labels, and
malformed-token behavior. Keep existing close ownership, descendant matching,
upstream-union, offline fallback, and flag-close semantics. Do not attempt a wider bead
lifecycle migration.

Register and round-trip-test the binding. The current source has no landed epic-symbol
Rust module despite sase-o7 note #23's earlier completion claim; verify the
implementation rather than assuming that note describes code present in HEAD. Let host
completion publish any core changes and advance `sase-core-revision.txt`; do not invent
a commit SHA or globally reinstall SASE.

### 3. Add a narrow verification preflight

Use a thin Python adapter and an ordinary repository tool such as
`tools/check_epic_symbol_freshness`. Read the `tools` reference memory before adding it.
The tool checks the current checkout and routes the bead IDs through the existing
project/store resolution APIs. Batch status reads using the existing `statuses_for_ids`
facade where appropriate; do not read every bead or spawn one CLI process per symbol.

- If there are no local epic-symbol rows, return immediately. If every referenced status
  is known non-closed, avoid fetching upstream.
- For closed local rows, observe local HEAD and the default-branch Justfile using the
  existing bounded Git conventions. Deduplicate the fetch and status queries. Record the
  exact upstream SHA and whether the fetch succeeded. Do not reinterpret the
  local/upstream union as a single tree for this classification.
- Report every affected row once, with the appropriate remedy and evidence quality. Do
  not mutate files, refs beyond a normal fetch, bead status, or the index. No stash,
  reset, automatic rebase, or copying individual upstream files.
- Add a named `tools/run_silent` preflight stage near the start of `check`, before
  formatting and lint work. Keep the existing `_require-tool-run` dependency and ToolRun
  record-before-execution behavior. Ensure a preflight refusal retains a nonzero command
  result; triage must not convert it to success.
- Run the same lightweight adapter at the `_lint-symvision` boundary for standalone
  lint/symvision invocations and for a status change occurring later in a long check.
  Preserve the actual Symvision arguments and its enforcement. Extra flags supplied
  directly by a user remain subject to Symvision's own checks.
- Avoid a new public `sase` CLI command, configuration setting, feature flag, or
  frontend UI. The changes belong to the existing verification workflow.

### 4. Regression coverage and verification

Add focused Rust classifier/binding tests and Python integration tests for:

- The two-checkout cleanup/publish/close/stale-observer sequence above.
- A closed row still present upstream, and a locally added row naming a closed bead.
- An open row, an empty Justfile, multiple symbols sharing one bead, and a re-keyed row
  whose new owner remains open.
- Fetch failure, missing objects, divergent history, and unknown bead status; none may
  falsely establish that the upstream cleanup landed.
- A dirty checkout with unrelated changes; diagnostics must preserve all bytes and index
  state. Cleanup requires integrating corresponding code, not deleting rows.
- A closed status appearing between initial preflight and Symvision invocation.
- Ordinary close refusals for local and upstream-only rows, descendant coverage,
  update-to-closed, force behavior, and already-closed no-ops remain unchanged.

Use existing recipe/tool test patterns to assert the preflight runs before the expensive
stages and preserves its failure status. Do not require a live GitHub remote or
hard-code the developer's filesystem in tests.

Read `lint_and_test` and `symvision` reference memory. Run focused tests and then
`sase tool run check` in each changed repository; do not run `just check-full`. Use
`/sase_monitor` when needed, without starting a second copy of a running check. Record
before/after results, revisions, and limitations.

## Completion and scope controls

Record the corrected historical finding and the implemented early refusal on
`sase-1qn.1`, with the ToolRun and commit evidence above. Explain that the existing
guard already protected master in the cited sequence; the remaining defect was the
verification path discovering an old checkout only at the expensive Symvision gate.

Append `TASK RESOLVED: sase-o7 — <verified behavior and historical distinction>` only
after the regression and required checks support that claim. If investigation reveals an
independently reproduced late-publication or unguarded-close path, record its exact
sequence as `PROPOSED FOLLOW-UP:` on this phase; do not silently substitute it for this
proven stale-checkout mechanism or claim this plan fixed it.

Before closure run `sase bead epic-symbols sase-1qn.1`, resolve any entries belonging to
this phase, and close **only** `sase-1qn.1` with the verification note. Do not close
`sase-o7`, `sase-1qn`, or another ancestor; the parent lander owns those. Do not create
beads. Unrelated check failures reproduced identically on a clean base go into a
`PROPOSED FOLLOW-UP:` phase note and do not keep this phase open.
