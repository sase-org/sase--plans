---
tier: tale
size: medium
title: Finish linked-repository absorption safeguards and land sase-1kh
goal: Raw-commit absorption preserves baseline dirt and host or foreign commits, regression
  coverage proves the contract, and epic sase-1kh closes normally with its plan marked
  done.
proposed_by: bbugyi200.apollo.sase-1kh.land
bead: sase-1kh
status: done
---

- **PARENT:**
  [202610/protect_linked_repo_work.md](https://github.com/sase-org/sase--plans/blob/main/202610/protect_linked_repo_work.md)
- **BEAD:**
  [sase-1kh](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1kh/README.md)

# Finish linked-repository absorption safeguards and land sase-1kh

## Scope and existing evidence

This is the remaining work from the land audit of epic **sase-1kh**. All three phases
are closed and their commits are already on `origin/master`: 86a343de28 (raw commits),
64bd007204 (monitor CWD), and 1a638eac95 (clone rescue). The parent plan is
`plan:202610/protect_linked_repo_work.md`. Read it with `sase artifact read`, and read
the epic's landing-audit and follow-up-triage notes. The audit evidence is
`file:explicit:5a1c98b7060a59115e90bd29`; read it with `sase artifact read` for
reproduction details and verification results. Its approved decision **raw_commit_policy
= absorb** remains binding. Keep the execution-time backstop; do not introduce a new
Rust finalizer trigger enum.

The lander verified every phase note and implementation commit. The history after the
first epic commit's parent (010185b0c4), including fetched `origin/master`, contains
only the three epic commits. There was no unrelated post-start drift. 71
phase/baseline/monitor/live-finalizer tests and all 13 discarded-work guard tests
passed. There are no epic-symbol exemptions and no parent bead above sase-1kh as of this
audit. Rescue and monitor implementation require no further product changes.

Disposable Git reproductions proved two remaining absorption bugs in
`src/sase/llm_provider/commit_finalizer_raw_commits.py`:

- `_protected_paths` considers only paths still dirty. Raw-committing baseline dirt
  makes it clean, so the absorber stages and adopts that pre-existing work.
- `_is_linear` checks only for merges. If a raw commit is followed by a foreign agent
  commit or a host-recorded stitch commit, those later commits are excluded from
  findings but are still unwound by the soft reset. The resulting dirt includes other
  agents' work or work the host ledger already owns.

This tale must implement the repairs and finish the epic's landing itself. There is no
later land agent for a tale. No step may depend on the tale coder's eventual commit SHA,
push, or CI result, which exist only after its provider turn ends.

## 1. Make absorption eligibility prove ownership of the reset range

Keep Git observation and mutation in the existing Python host orchestration; these
repairs concern local Git operations, not a new shared domain API. Modify the raw-commit
module and its callers only as required.

Before pinning a recovery ref or resetting, prove that every commit from the parent of
the oldest finding through the observed HEAD is precisely the detected raw chain, in
order, with one parent per commit. Findings must form a contiguous suffix ending at
HEAD. A filtered foreign or host-recorded commit anywhere in that reset range makes it
ineligible; leave the repository unchanged so the existing non-retryable
`unpublished_local_commits` backstop handles it visibly. Do not silently absorb only a
smaller suffix and forget an earlier raw commit.

Preserve eligible behavior when a remote-reachable, foreign, or host-recorded commit
precedes the entire raw suffix: the reset target is the raw suffix's parent, preserving
that earlier history. Handle unavailable Git proof by declining absorption. Retain the
attached-branch, no-operation-in-progress, merge rejection, recovery-ref, and
idempotence rules. Revalidate observed HEAD before mutation so a changed tip cannot
invalidate the ownership proof.

Use all paths recorded dirty by the canonical baseline as the conservative protected set
for this history-rewriting eligibility check. Do not filter it through current dirt or
content equality: committing, editing, or restoring the path must not erase the
run-start/opened-repo ownership boundary. Collect the union of paths touched by each raw
commit, including both sides of renames; do not use the net `git diff baseline HEAD`,
which hides reverted changes. If any touched path overlaps the protected set, decline
before mutation. Keep the normal declaration dirt classifier's existing semantics
separate from this absorption eligibility proof.

Update `docs/commit_workflows.md` to describe the contiguous owned suffix and the
historical baseline-path protection. Keep `schema_version: 1` compatibility, SDD
exclusions, the foreign-agent detection exemption, and host-unpushed resume.

## 2. Prove the safety and prepared-completion paths

Extend the real-Git tests in `tests/llm_provider/test_commit_finalizer_raw_commits.py`
(split focused modules if useful) and the existing prepared-completion suite. Add
meaningful regressions:

- A tracked or untracked path dirty at baseline is raw-committed and now clean:
  absorption declines, HEAD/index/worktree are unchanged, and the backstop names the
  offending raw commits. Update the current protected-dirt test that wrongly expects
  absorption after committing the restore.
- A candidate touches a protected path and a later candidate restores it to the baseline
  HEAD contents: it still declines. Cover a rename of a protected path.
- A foreign-agent commit or a host-recorded `pushed: false` stitch commit after a raw
  commit, including a filtered commit between two raw commits: no reset or adoption
  occurs. The foreign/host commit remains at its original place in history and the raw
  chain remains visible to the backstop.
- A foreign/host/remote commit before a complete raw suffix survives successful
  absorption. Eligible staged changes contain only the raw suffix's work, the recovery
  ref retains the original tip, and a repeated call is idempotent.
- A changed HEAD between observation and mutation declines safely.
- A prepared host-completion intent made after eligible absorption observes the reset
  HEAD and staged index, binds successfully, and keeps the normal declaration
  obligations. `_prepare_completion.py` already publishes context before repository
  observation; change ordering only if the test exposes a gap.

Run the focused suites for baseline/raw commits, declaration/reconciliation, prepared
completion, host unpushed resume, discarded-work guard, clone rescue, and monitor CWD.
The live-finalizer suite passed completely at audit; keep it green. Run `just fmt` or
`just fix`, then **`sase tool run check`**, which executes `just check`. Do not run
`just check-full`. Use `/sase_monitor` if the check needs a handoff, with a follow-up
explicitly instructed to finish every closeout step below; a passing verification alone
must not skip the epic closeout. Keep known unrelated failures governed by the project
triage policy and the lander's recorded follow-up outcomes; fix any regression caused by
these repairs before closing.

## 3. Recheck drift and close the epic in this same work turn

Follow-up triage is complete in the lander's note on sase-1kh. Carry every outcome from
that note into the close note; do not duplicate tasks. Before closing, read
`sase bead read sase-1kh -r "Need landing scope, triage outcomes, and readiness"`, read
every child and note, and confirm all descendants and the linked plan's named phases are
complete. Refresh repository/base-branch history and integrate any new changes relevant
to rescue, absorption, declarations/prepared completion, or monitor CWD. No current task
depends on this tale's own commit.

The recorded outcomes to preserve are: corroboration on sase-1hy for duplicate
raw-prompt hints (including the phase1 cache proposal), corroboration on sase-1ia for
missing expanded-prompt bodies and session hints, corroboration on sase-1k9 and sase-19m
for the phase3 probe timeout reports, and new CI task **sase-1ki** for the unrelated
deterministic bottom-pin loss (related to neighboring sase-1b8). The phase2
eight-live-finalizer proposal was declined because all 14 live tests now pass. The
phase3 completion snapshot failures are resolved by the phase's checked-in regeneration.
The other currently passing historical nodes were declined as unconfirmed current
issues, with their identities and ToolRun evidence retained on the epic; no generic
triage umbrella was created.

Run **`sase bead epic-symbols sase-1kh`** immediately before close. Resolve every listed
symbol (wire it up, privatize it, add a valid real-consumer non-test pragma, or delete
it), or re-key its Justfile exemption only to a still-open later bead that demonstrably
needs it. Re-run relevant checks if this changes code.

Then run **`sase bead close sase-1kh --note "<verification>"`** normally. The note must
record: the three reviewed phase commits/notes, the approved absorb policy, the repaired
ownership and protected-path cases, targeted and just-check results, integration/drift
findings, every follow-up outcome from the lander's triage note, and that the
`sase_monitor` skill needs redeployment from a clean landed revision with
`sase skill init --force`. Do not deploy generated skills from this unlanded tale. If
close rejects leftover epic-symbol entries, finish cleanup and retry. If it rejects
incomplete phases, finish/reopen them; never force a successful nested landing or use
force merely to get a successful exit.

After close, run **`just symvision`** to confirm the whitelist is clean. Open the plan
sidecar with **`sase repo open plans -r "Mark verified sase-1kh epic plan done"`**. Use
`sase artifact read plan:202610/protect_linked_repo_work.md` for the audited read and
`sase artifact path plan:202610/protect_linked_repo_work.md` to resolve its path. Set
**`status: done`** in that plan's frontmatter, preserving every other field, decision,
and its body. The sidecar change is part of the host final declaration's commit
obligations.

Finally inspect **`sase bead read sase-1kh -r "Need the parent link after close"`**.
This epic currently has no parent; finish normally if still absent. If a parent was
introduced meanwhile, follow the landing prompt's normal parent readiness rules: verify
and close only a directly parented phase, or fully re-audit all descendants, plan and
drift before closing a parent plan; retire its symbols, run symvision, mark its plan
done, and repeat through fully ready plan ancestors. Stop at an ambiguous/incomplete
ancestor and leave a blocker note. Submit the root `/sase_final` declaration last with
all changed repositories committed by the host; do not create a manual Git commit.
