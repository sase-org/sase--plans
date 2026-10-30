---
tier: tale
title: Repair monitor adoption and finish the monitor-staging landing
goal:
  Staged monitors preserve starter identity, response, and checkpoint content; epic
  sase-1pj.5.1 and parent phase sase-1pj.5 close with integrated verification.
size: medium
proposed_by: bbugyi200.athena.sase-1pj.5.1.land
proposed_with: codex/gpt-6-astra
bead: sase-1pj.5.1
create_time: 2026-10-30 15:00:10
status: wip
---

- **PARENT:**
  [202610/monitor_staging.md](https://github.com/sase-org/sase--plans/blob/main/202610/monitor_staging.md)
- **BEAD:**
  [sase-1pj.5.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1pj/sase-1pj.5.1.md)

# Repair monitor adoption and finish the monitor-staging landing

## Scope and binding context

Finish the remaining work of epic **sase-1pj.5.1**, whose parent is phase **sase-1pj.5**
in outer epic **sase-1pj**. This is one bounded coding turn, followed by the same
coder's landing. A tale has no land agent that will finish this later.

Read `sase bead read sase-1pj.5.1 -r "Need landing audit and follow-up dispositions"`
and its four closed phases. Read `plan:202610/monitor_staging.md` and
`plan:202610/staged_handoffs.md` using `sase artifact read`. The inherited DECISIONS are
final; this repair needs no new memory, flag, or user-doc changes. Read
`lint_and_test.md` and `symvision.md` using `/sase_memory_read` before verification.

The lander reviewed the actual Python and Rust contracts and all epic commits:
`b175b63758`, `8f982c8ae9`, `f7eb80c7a8`, `7da649b562`, and `c9a65d1a58`. The core
contract commit `2a625131` is an ancestor of the current core revision pin
`17e79c8cf5323d2ef4abcadef3bf95aa4e81c24b`. The existing transaction is implemented:
durable staging, pending owner liveness, prepared reservation/bind/release, atomic join
reservation/activation, post-finalizer dispatch, and held/discard cleanup. Preserve
these APIs and the existing `.go`/ack/claim barrier.

The three remaining defects were reproduced on `c9a65d1a58` with isolated fixtures:

1. Real `stage_monitor` promotes a bare `acme` starter to `acme--0`, but
   `_adopt_monitor` calls `_runner_ctx_from_metadata`, which rejects that legitimate
   persisted name against the original live runner name. Adoption raises
   `StagedHandoffError: persisted agent identity differs from the active runner identity`.
   A second real probe reserved a ToolRun for bare `acme` and then staged `-J`:
   promotion to `acme--0` made the join reservation refuse with `agent_mismatch`.
2. `_adopt_monitor` deletes `response` and returns `monitored` without saving a chat or
   capturing the starter delta. With its returned outcome and an otherwise real
   `finalize_loop`, `saved_chat_paths` is empty, the saved response is `''`, and
   `persist_agent_delta_best_effort` is called zero times despite a real response in the
   workflow result. The legacy monitor handler did that work before returning
   `monitored`. Missing exact starter ancestry can block or mis-parent the follow-up.
3. Staging a request with both `checkpoint_ref` and `checkpoint_document`, then
   reconstructing it through `launch_from_persisted_payload`, retains the reference but
   sets `request.checkpoint_document` to `None`. The CLI loads the authored checkpoint
   into memory; its canonical content is never durably frozen by staging.
   `_monitor_intent.persist_monitor_start_intent` needs that content to publish it.

The existing 53-test staging/contracts/activation/completion-selection run passed, but
the activation tests stub staging or launching. Their successful ordering branch also
replaces `finalizer_settlement_refusal`, so it does not prove the requested real
finalizer/check ordering or these identity/continuation boundaries.

The lander's symbol cleanup also passed its 34 affected tests, 16 prepared/host
completion tests, `just fix`, `just symvision`, and governed `just check` (ToolRun
`811e6a0b4501e7f85e6a30855f17afe0`, exit 0). The Justfile edit made that check select
the full nonvisual test lane. No `check-full` was run. Verify the actual repairs again
as required below; this green suite does not exercise the reproduced gaps.

## 1. Preserve legitimate promotion and exact run identity

Exercise the real staging path for both a bare root and an existing session member.
Update the monitor adopter's context rebinding to accept the verified promotion of the
same starter artifact/run. Reuse existing session-promotion and canonical identity
helpers; keep refusal for unrelated or substituted metadata. Do not weaken the shared
adopter check into accepting any matching prefix or any persisted name.

Use the original runner identity for the already-reserved ToolRun where required, and
the verified persisted member identity for chat, metadata, and continuation attribution.
Check `-J` on a bare starter too: staging promotes metadata before
`_reserve_staged_join`, whose Rust starter-agent check is exact. Preserve the original
run's starter ownership instead of changing the Rust join guard to accept strangers.
Derive facts from persisted run identity, not whichever `SASE_AGENT_NAME` happens to be
in the adopter environment. New shared identity policy, if needed, belongs in
`sase-core`; open it via `/sase_repo`, read its instructions, add bindings/tests, and
ratchet the core revision pin through the supported workflow.

## 2. Restore the staged monitor's transcript and continuation

Extract the reusable capture/metadata work from `src/sase/axe/run_agent_exec_monitor.py`
into a helper used by the staged path and, where practical, the legacy path. Keep legacy
kill normalization separate. The staged adopter must save the actual provider response,
include monitor details as extra context, append the saved chat to
`LoopState.saved_chat_paths`, update chat/member metadata and role suffix after
promotion, and publish the starter continuation delta and monitor-handoff checkpoint
using the existing capture APIs. Bind the monitor's continuation parents to that exact
starter node. Do not manufacture an interrupted response or reset kill state on normal
adoption.

Review `run_agent_exec_finalize.py` as part of this change: returning `monitored`
currently assumes the adopter has already saved the chat and continuation. Keep a single
owner for this work and prove finalization neither overwrites the real response nor
duplicates capture. Keep the terminal marker ordering that monitor completion waits on,
the moved workspace claim, and the post-done publication doorbell introduced by
`e69360f49d`. Preserve the recent private-module splits of staged dispatch and handoff
storage; do not restore duplicate implementations in the old facades.

## 3. Freeze authored checkpoint content across CLI exit

Persist the canonical authored checkpoint content during staging in the monitor's
durable artifacts and reconstruct it at adoption, using the existing checkpoint
canonicalization/hash validation. Prefer a durable content reference in the existing
claim-context payload over reopening the user's mutable original path. Deleting or
changing that original file after the stage must not change what adoption publishes.
Missing or corrupt staged content must refuse explicitly, not silently degrade to a
reference with no content. Keep historical payloads without authored checkpoints
readable. Do not publish the monitor start intent or launch a command before ack; reuse
the existing post-ack intent publication.

## 4. Prove the repaired boundaries and integrate later changes

Add meaningful integration coverage using the real stage/reconstruct/adopt paths and
existing finalizer/monitor harnesses. Stub the external process/provider boundaries, not
the behavior under test. Required observations:

- A bare starter and an existing session member stage and adopt successfully; a foreign
  persisted starter is refused before proc launch. Bare-starter `-J` reserves and
  observes the same run without rerunning its command.
- The final stored chat contains the real response and monitor details; the done marker
  points to it; the starter delta is captured once; the follow-up receives the exact
  starter parent after the starter's terminal marker.
- An authored checkpoint survives CLI exit and original-file mutation/removal, and
  corrupted frozen checkpoint content refuses safely.
- An event/barrier through the actual finalizer controller proves the monitor command
  starts only after `just fix` and all remaining finalizer passes settle. A failure
  produces zero launches and retains held evidence with released reservations.
- Exercise prepared starter coverage and host completion with the staged descriptor:
  green commits exactly once without another model; red/stale commits none of the
  prepared work and takes recovery. Reuse existing host-completion tests instead of
  inventing another completion engine, especially
  `tests/test_finalizers_prepared_bridge.py` and the existing host-completion no-model
  and eligibility suites. Preserve early-settled join consumption, cleanup, explicit
  commit-before-wait, and legacy flag-off/non-agent behavior.

Check the narrow monitor skill wording introduced by the activation phase against the
real controller policy. `src/sase/macros/skills/sase_monitor.md` currently says that a
staged monitor "never synthesizes carry for you", although unprepared staged monitors
set `carry_allowed` and the host's `ensure_current_declaration` can synthesize carry
when no explicit declaration exists. Clarify the host's role and retain the explicit
commit-before-CI-wait instruction and prepared commit-on-green behavior. Cover that
unprepared branch with the existing controller harness. Read `generated_skills.md` and
`macros.md` through `/sase_memory_read` first; edit only the in-repo source template and
preview without deployment. Deploying generated skills requires a landed source revision
and is outside this tale's same-turn closeout.

Inspect commits landed after this audit, including the base branch if applicable, for
interactions with these files. The original completion snapshot/kind drift and
suite-gate helper-copy issue were already fixed by `86bec79ca4`; both passed in the
lander run and need no duplicate repair. The lander also fetched and reviewed base
commits through `deb8bb60b6`: `3f9fe9c107` stabilizes CLI test assertions, `d69741b370`
changes stopped-agent presentation, and `deb8bb60b6` fixes release wheel matching. None
adds a monitor-staging integration requirement. Outer crash reconciliation and re-adopt
belong to `sase-1pj.6`, carried-work workspace holds to `sase-1pj.7`, and flag removal
and broad memory/docs changes to later outer phases. Keep this tale bounded to the three
defects, their missing acceptance proof, and that source-template correction.

Run relevant targeted tests, `just fix`, and **`sase tool run check`** (the governed
`just check`). Do not run `just check-full`. A check-full-only failure would be test
infrastructure, not remaining product work. If verification needs a monitor handoff, its
successor must retain all of the closeout obligations below; do not use a prepared
completion that would skip them. No closeout step depends on this tale's own commit SHA,
push, or CI run; host finalization commits after the coding turn ends.

## 5. Finish this epic and its parent phase in the coding turn

The lander has already triaged every proposed follow-up and recorded all dispositions on
`sase-1pj.5.1`. Include them in the final close note:

- `sase-1pj.5.1.1` note #2: corroborated existing IPC listener flake `sase-1gz`; no
  duplicate task.
- `sase-1pj.5.1.2` note #1: created ready small CI task `sase-1q3` for the missing
  ambient `SASE_MODEL_OVERRIDE` test scrub; override gives 7 failures/44 passes,
  unsetting only that variable gives 51 passes. This is separate from monitor staging.
- `sase-1pj.5.1.3` note #1: declined new work because `86bec79ca4` repaired the
  suite-gate scratch import closure; the exact host-budget node now passes.

The lander's accompanying commit retires all four monitor epic-symbol rows:
`StagedMonitorStart`, `StagedMonitorCleanupResult`, and the payload loader become
private in-file helpers; the unused public prepared-reservation loader is deleted, with
tests observing the existing store reader. Do not reintroduce those exports merely for
tests. The live stage/launch/cleanup entry points remain public.

After all repairs and verification, perform this concrete closeout:

1. Re-read `sase-1pj.5.1`, every descendant and note, and its linked plan. Confirm all
   descendants are closed and every remaining-work item is complete. The tale
   implementation and closeout occur in the same turn; there is no later land agent.
2. Run `sase bead epic-symbols sase-1pj.5.1`. Resolve every listed entry by wiring it
   up, privatizing it, deleting it, or adding a justified non-test pragma under
   Symvision policy. Re-key only if a concrete still-open later bead needs it. The
   lander removed the current upstream rows before this tale so closeout does not depend
   on the coder's own future commit.
3. Run
   `sase bead close sase-1pj.5.1 --note "<verified stage/finalizer ordering, promotion, real response and exact continuation, checkpoint persistence, prepared outcomes, joins, cleanup, both flag states, post-start integration, check results, and the three follow-up dispositions above>"`.
   Then run `just symvision`.
4. Resolve the PLAN path from `sase bead read` / `sase artifact path` for
   `plan:202610/monitor_staging.md`, open the plans repository via `/sase_repo`, and set
   **`status: done`** in that plan's frontmatter. Include the sidecar change in the
   final declaration.
5. Re-read `sase-1pj.5.1` for its parent link. Its parent is the **phase `sase-1pj.5`**.
   Verify the completed child plan fulfills that phase's monitor-staging/just-fix-race
   scope. Run `sase bead epic-symbols sase-1pj.5`, resolve any entries, and close only
   that parent phase normally with
   `sase bead close sase-1pj.5 --note "<verified ordering, prepared outcomes, joins, cleanup, promotion/continuation/checkpoint fixes, flag parity, and checks>"`.
   Leave containing epic `sase-1pj` and its plan status to its waiting land agent.

Never force merely to make a close succeed, and never use `--force` to advance this
successful nested landing. If a guard reports real leftover symbols or unfinished
phases, finish the missing work and retry. If the parent is genuinely incomplete or
ambiguous, record the exact blocker on it and report that state honestly. Finish through
`/sase_final`, keeping host-owned commit behavior for every changed repository.
