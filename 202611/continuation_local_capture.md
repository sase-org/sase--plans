---
tier: tale
title: Capture local requests before fork expansion
size: medium
goal:
  Keep continuation captures local to each turn so fork ancestry cannot overflow the 256
  KiB authored-request limit, while preserving existing replay and actionable recovery.
proposed_by: bbugyi200.athena.sase-1sd.4
proposed_with: codex/gpt-6-astra
bead: sase-1sd.4
create_time: 2026-11-12 09:53:45
status: wip
---

- **PARENT:**
  [202611/next_seven_salvage_and_unstrand.md](https://github.com/sase-org/sase--plans/blob/main/202611/next_seven_salvage_and_unstrand.md)
- **BEAD:**
  [sase-1sd.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1sd/sase-1sd.4.md)

# Capture Local Requests Before Fork Expansion

Implement assigned phase `sase-1sd.4` of
`plan:202611/next_seven_salvage_and_unstrand.md`, resolving the defect tracked by
`sase-1o1`. Read both beads with `sase bead read` and the parent plan with
`sase artifact read` before coding. Keep the parent epic's accepted decisions. No memory
changes or additional task beads are authorized by this plan.

## Scope and sizing

This is one medium tale: preserve request provenance across the existing launch and
capture pipeline, retain the existing parent-chain replay implementation, and cover the
actual launch boundary in tests. The monitor diagnostic propagation and manual
checkpoint recovery mechanisms already exist. Extend those mechanisms; do not add a new
graph walker, historical Markdown parser, migration command, or automatic rewrite of
immutable session records.

There are no previous implementation notes on `sase-1sd.4`. Planning inspected clean
sase `8eef093fa9` and sase-core `afc64b10`; the capture bug remains in these trees. The
earlier monitor-ownership phase's core fix is present in that core revision.

## Findings that constrain the implementation

1. `src/sase/axe/run_agent_runner.py::_admit_and_launch` replaces `state.prompt` with
   `expand_deferred_launch_macros(...)` before executing the workflow.
   `run_agent_runner_setup_prompt.py` expands deferred fork workflows after dependency
   admission. Thus the anonymous workflow's `step.agent` can already contain the entire
   rendered parent history.
2. `src/sase/macro/workflow_executor_steps_prompt_prepare.py::_prepare_prompt_step`
   calls that string `authored_local_request=step_prompt`. `preprocess_prompt_early`
   also tags the input as a `local_authored` segment. Fixing only the final delta writer
   or this final assignment would leave another ancestry-bearing local copy, or lose the
   original request entirely.
3. `src/sase/continuation_capture/agent_delta.py::persist_agent_delta` prefers
   `prepared_prompt.json`'s authored field, then falls back to an execution-loop prompt
   that can also be expanded. `ensure_prepared_prompt` retains an existing prepared
   capture. Both successful and interrupted/retry capture paths matter.
4. `segments.py` already distinguishes `local_authored`, `local_materialized`, and
   `injected_parent`; canonical deltas exclude injected-parent segments. Preserve
   ordinary macro, gate, and artifact material without retagging an ancestry dump as
   local material.
5. Python replay already hydrates exact parents in
   `history/chat_fork/continuation/_source.py` and `replay.py`. Rust
   `crates/sase_core/src/continuation/replay.rs` orders and deduplicates those nodes.
   `_render.py` renders each captured authored request verbatim within its user block.
   Existing immutable records must keep their current interpretation.
6. `tests/history/test_continuation_replay_hydration_basic.py` already has a 100-handoff
   growth test. Its helper directly supplies clean local requests to capture, bypassing
   the launch expansion that causes this bug. Retain that coverage and add the missing
   production-boundary regression.
7. Rust `continuation/schema.rs::validate_agent_delta` enforces
   `MAX_TEXT_BYTES = 256 * 1024`. Keep this limit and control-character checks. Python's
   existing terminal-text normalization and raw debug archive must remain intact.
8. `continuation_capture/_monitor_parents.py::starter_capture_error_summary` and
   `monitor/_settlement_followup.py::_blocked_followup_error` already propagate the
   starter error and print resume guidance. The current generic model-route suggestion
   does not repair an oversized authored field; this failure needs checkpoint-specific
   guidance, verified through the existing resume path.

Planning could not run the validator reproducer because the checkout's `.venv` lacks
`sase_core_rs`; both boundary probes failed at import, before validation. Establish the
development environment before claiming a red/green result. The parent report's cited
research artifact did not resolve, and a scan of currently listed local agents found no
accessible prepared request above 200 KiB. The parent's November 11 measurement of
708,391 bytes is historical evidence, not a measurement reproduced by this planner.

## Implementation

### 1. Establish the failing production-boundary case

Use `/sase_repo` to open `sase-core`, read its `AGENTS.md`, and use the printed path.
Recheck `sase agent list -j` before editing. Planning found no live `sase-1ra` member;
the `sase-1n6.11` steward was waiting on a GitHub-settings monitor. Record overlap and
coordinate on the phase bead if that changes. Keep this epic's phases sequential.

Set up the checkout's isolated environment as required by `lint_and_test`; hand known
long setup commands to `/sase_monitor`. Do not install globally. Build a regression that
expands a fork at the runner boundary, feeds its result through anonymous workflow
preparation, and persists the resulting agent delta with real Rust validation. Stub
provider execution and external publication, not provenance selection or validation. Use
enough deterministic parent content to cross 262,144 UTF-8 bytes while the local request
remains small. Record the failing field and byte count on `sase-1sd.4`.

### 2. Carry request provenance through launch and capture

Retain the turn's local request before launch-deferred fork expansion, alongside the
materialized prompt and provenance-tagged segments. The correct boundary is immediately
before the deferred fork substitution, after ordinary local macro expansion as needed;
the alias-resolved raw prompt remains useful audit evidence. Preserve a fork reference
as authored text if present, but never its rendered ancestry as local instructions.

Carry this value explicitly through runner state, execution-loop state, anonymous
workflow construction, prompt preparation, and the invocation's existing
`authored_local_request`/`continuation_segments` arguments. Use a small typed carrier or
optional fields appropriate to the existing structures. Scope any override to the exact
turn and anonymous step it belongs to. A named workflow's independent agent steps must
keep their own requests. Do not read a global `raw_prompt.md` indiscriminately from
every step or assume `LoopState.original_prompt` is unexpanded.

Account for retries, restored staged handoffs, plan/code or question phase transitions,
and artifacts-directory changes: each new logical turn gets its own authored request,
while retries of the same turn retain it. Do not carry a predecessor's request into a
different phase. Preserve newly appended local feedback and question answers.

At prepared capture, make the authored field and `local_authored` segment agree. Keep
fork material exclusively in injected-parent/debug records and retain genuinely local
materialized instructions. Make the missing-prepared-record fallback obey the same
provenance rule. Do not recover structure by splitting on `# Previous Continuation`,
`### User`, fences, or another user-authorable delimiter. Strings containing those
headings are legitimate local content.

This transport is Python glue. Any new shared request-selection or normalization policy
must live in sase-core with a thin Python adapter; do not introduce a second backend
policy in Python. Prefer carrying known provenance over inferring it.

### 3. Preserve replay and make over-cap failures actionable

Continue to reconstruct ancestry using exact parent IDs and the existing hydration and
Rust ordering. Correct a missing edge at its capture boundary if the new regression
reveals one. Do not require nested authored text for ancestry, and do not use broad
session history as a substitute for the exact chain.

Keep replay byte-for-byte identical for a fixed set of existing records, including
legacy records whose authored text already contains nested history. New local-only
records must render exactly like equivalent canonical local records using today's
renderer; they should not reproduce the defective nested text. Compare stable block
bytes when extending a chain, since the overall header's node count changes. Preserve
shared-parent deduplication, missing-parent refusal, evidence policy, and retry-attempt
splicing. No wire or projection version bump is expected merely to transport the correct
existing field.

In sase-core, enhance the authored-request over-limit diagnostic with the observed UTF-8
byte count, allowed limit, and field identity, retaining the existing recognizable
`authored_local_request is too large` wording. Add real core boundary tests for exactly
the limit, one byte over, and multibyte text, plus binding coverage as appropriate. Keep
all other validation unchanged. Do not raise the limit or truncate instructions.

For an old prepared capture already over the cap, use the parent's explicitly allowed
diagnostic-and-recovery option. Leave its immutable content untouched, report the
actionable failure in capture errors and monitor settlement, and direct the operator to
inspect retained request/response/evidence and provide a bounded authored checkpoint
with `sase monitor resume <id> -k FILE`. Verify the exact supported command and
checkpoint shape against the implementation. A route with more model context alone
cannot fix this capture error. Do not automatically rerun the monitored command, resume
live sessions, fabricate the missing parent, or silently treat an absent capture as
complete.

Read `docs_writing` and the docs authoring contract before updating the existing monitor
Reference page with this specific failure and recovery procedure. Include a valid small
checkpoint example or link to its existing schema. Preserve the current recovery guards
for stopped/lost monitors and conditional host finalization. Test checkpoint resume
through successor launch eligibility with the provider stubbed, so the documented remedy
is known to work for the over-cap missing-parent case.

## Verification and completion

- Add a chain regression with at least 20 agent/monitor cycles that goes through
  deferred fork expansion and prompt preparation. Size fixtures so the old
  implementation exceeds the authored cap. Assert every new authored field equals that
  turn's local request, its UTF-8 size stays below the cap, canonical local segments
  exclude inherited content, exact parent links remain intact, and original
  constraints/final responses remain reachable once each. Assert stable replay bytes for
  equivalent record sets and automatic eligibility of the final monitor follow-up.
- Cover plain prompts, locally expanded forks, a fork introduced by another macro,
  independent workflow steps, retry and phase-transition provenance, and literal
  history-looking headings. Use focused cases at the relevant boundaries rather than
  duplicating the entire launch suite.
- Cover old nested records without rewriting them, a legacy over-cap prepared record, a
  genuinely over-cap local request, and actionable checkpoint recovery. Keep existing
  hostile-text, portable hydration, missing-parent, and output-policy tests green.
- Locate the parent's reported athena session through supported agent/artifact lookup if
  still retained. Inspect `continuation/capture_errors.jsonl` and field sizes before and
  after using a copied fixture, without mutating the live session. If unavailable,
  record that limit explicitly and use the synthetic 708,391-byte fixture; never claim
  live repair or live before/after verification without evidence.
- Run focused Python continuation, fork replay, runner/preparation, retry, and monitor
  recovery tests; run focused core continuation and binding tests through the repo's
  `just test` recipes, never bare Cargo. Then `sase tool run check` in each changed
  repo. Route long checks through `/sase_monitor` and follow `/sase_final`
  prepared-completion rules where suitable. Do not run `just check-full`.
- Preserve checks and budgets. A failure reproduced identically on the clean base is
  recorded as `PROPOSED FOLLOW-UP:` on `sase-1sd.4` with evidence and any existing task
  reference; it does not keep this phase open. Do not create task beads.
- Let host completion commit sase-core and move `sase-core-revision.txt` past its actual
  commit; do not invent a pin or commit manually. Verify the resulting pin includes the
  required core changes as part of completion.
- Record `TASK RESOLVED: sase-1o1 — <before/after evidence and limits>` on `sase-1sd.4`.
  Run `sase bead epic-symbols sase-1sd.4` again (none were present during planning),
  resolve or re-key any entries, and close only `sase-1sd.4` with a verification note
  after its full scope is complete. Keep it in progress during planning and partial
  implementation. The parent lander owns `sase-1o1` and ancestor closure.
