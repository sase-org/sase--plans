---
tier: epic
title: Corroborated SASE task beads and disciplined task creation
goal: 'Agents can record independently evidenced +1s on existing task beads, every
  new task has an intentional size and size-derived launch route, and concise generated
  guidance prevents duplicate tasks or tasks caused by active epics.

  '
phases:
- id: core-contract
  title: Atomic task +1 domain and persistence contract
  depends_on: []
  size: medium
  description: 'core-contract: add the Rust-backed structured +1 evidence operation,
    compatibility codecs, and task-size creation invariant.'
- id: cli-routing
  title: Public CLI, task sizing, and model routing
  depends_on:
  - core-contract
  size: medium
  description: 'cli-routing: expose the +1 workflow, require sizes on every task creation
    path, and replace the sizeless task model alias with phase-size routing.'
- id: surface-polish
  title: Task +1 presentation across every user surface
  depends_on:
  - core-contract
  size: medium
  description: 'surface-polish: render +1 counts and evidence beautifully in CLI,
    TUI, triage, mobile, and published bead views.'
- id: agent-guidance
  title: Concise sase_new_task skill and agent policy
  depends_on:
  - cli-routing
  size: medium
  description: 'agent-guidance: add the generated task-creation skill, route agent
    prompts and memories through it, and document duplicate and active-epic decisions.'
- id: integration
  title: Cross-repository verification and contract cleanup
  depends_on:
  - cli-routing
  - surface-polish
  - agent-guidance
  size: small
  description: 'integration: exercise the complete workflow, reconcile generated contracts
    and snapshots, and prove legacy compatibility and removal of task_worker routing.'
proposed_by: bbugyi200.athena.rl
create_time: 2026-08-01 13:10:27
status: done
bead_id: sase-dr
---

- **PROMPT:**
  [prompts/202608/task_bead_plus_one.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/task_bead_plus_one.md)
- **BEAD:** [sase-dr](https://github.com/sase-org/sase--beads/blob/main/pages/sase-dr/README.md)

# Plan: Corroborated SASE task beads and disciplined task creation

## Product contract and design decisions

Treat a task `+1` as one additional, independently attributed report of the same actionable issue—not as a generic vote.
Expose it as:

```bash
sase bead +1 <task-id> -n "<reproduction and evidence>" [-R <artifact-ref> ...]
```

`-n/--note` is mandatory, `-R/--ref` is repeatable, and `-a/--author` is available for the same explicit-attribution
cases as `sase bead note` while normally resolving to the current SASE agent. The backend records a structured +1
evidence entry containing timestamp, reporter, note, and normalized artifact references. The visible counter is derived
from these immutable entries instead of being an independently mutable scalar, so the count and supporting evidence
cannot drift apart or be erased by replacing ordinary bead notes. Machine output exposes both `plus_one_count` and the
structured evidence entries.

Count at most one +1 per reporter, and do not count the task's original creator as an additional reporter. Retrying the
same reporter is an idempotent no-op with guidance to use `sase bead note` for later supplementary evidence. A +1 on an
`open` draft or `closed` task promotes it to `ready` in the same atomic operation (clearing stale close metadata for a
closed task); `claimed`, `ready`, and `in_progress` tasks retain their status. This makes duplicate suppression
actionable instead of leaving corroboration stranded on a draft or terminal task. Reject +1s on plan and phase beads.

Optional evidence refs are attached to the bead's normal reference set as part of the same event. The event reducer must
be deterministic under concurrent stream merges: independent reporters survive and increment the derived count, while
duplicate events from one reporter collapse to one evidence entry. Legacy issues and event streams default to no +1
evidence and remain readable without migration.

Do not add automatic fuzzy duplicate detection. `/sase_new_task` gives an agent enough compact context to make the
semantic decision: two reports are duplicates when they describe the same underlying defect/root cause or desired
remediation, not merely the same subsystem or a superficially similar symptom.

Require `size: xsmall | small | medium | large | xlarge` when creating every new task bead through the core API, CLI, or
ACE. Continue reading and launching legacy sizeless tasks; normalize those only at consumption time to the existing
small phase behavior. A task without an explicit model uses the same size-to-phase-worker mapping as an epic phase.
Remove only the implicit `@task_worker` model alias and its configuration/documentation—not the ordinary term “task
worker” used for background execution. Align task planning behavior with phase sizing as well: large and xlarge task
launches receive the plan-first handoff, while xsmall, small, and medium tasks implement directly.

## Phase 1: Atomic task +1 domain and persistence contract

Work through the linked `sase-core` checkout using `sase repo open sase-core` before reading or editing it. Extend the
shared Rust bead domain, event reducer, mutation engine, and PyO3 boundary rather than implementing backend semantics in
Python.

- Add a structured task +1 evidence wire record and a backwards-compatible, default-empty collection on `IssueWire`.
  Validate nonblank reporters/notes, normalized refs, uniqueness by reporter, and that only task beads can carry the
  collection. Keep old JSONL projections, event fixtures, and missing-field inputs valid.
- Add a dedicated append-only event operation/payload and an atomic mutation. It must resolve shorthand IDs, reject
  non-tasks, make repeat calls by the creator or an existing reporter unchanged, append evidence and refs for a new
  reporter, perform the draft/closed-to-ready transition, update timestamps, and save the canonical event store plus
  projection under the existing mutation lock.
- Teach event validation, deterministic reduction, three-way stream merging, history diffs, search indexing, stats,
  JSONL import/export, and storage parity about +1 evidence. Add concurrency coverage proving two agents are preserved
  and duplicate events from one agent cannot inflate the count.
- Enforce “new task requires size” in the core create request without tightening `IssueWire.validate()` in a way that
  rejects legacy sizeless task records. Preserve raw phase creation compatibility unless separately constrained by an
  epic plan.
- Export the mutation through `sase_core`, `sase_core_py`, and the binding registration/docs. Extend the Python issue
  model, Rust/Python wire conversion, compatibility JSONL and SQLite mirror/migration, mutation facade, and
  `BeadProject` API so all frontends receive the same structured result.
- Add focused Rust and Python facade tests for validation, idempotency, status promotion, close-metadata clearing,
  artifact-ref normalization/deduplication, serialization defaults, event/history replay, and unchanged bytes on every
  rejected mutation.

## Phase 2: Public CLI, task sizing, and model routing

Build the agent-facing command and creation guarantees on the shared operation.

- Register `sase bead +1` in sorted bead help/dispatch with excellent examples and `-n/--note`, `-R/--ref`, and
  `-a/--author`. Route attribution through current-agent discovery, use the standard mutation lock/commit/push path,
  emit a concise colored success/no-op message including the total, and classify the command correctly in the early Rust
  fast-path guard. Add the canonical `chore(beads): +1 <id>` commit message.
- Make `sase bead create -T task` fail clearly unless `-z/--size` is present, while retaining conditional validity for
  plan and phase types. Update every product call site that creates tasks; update tests that are exercising creation to
  choose a size, while retaining explicit fixtures that prove legacy sizeless records still load and launch.
- Remove `TASK_WORKER_MODEL_ALIAS_NAME`, the `task_worker` shipped alias/default-config surface, model completion and
  picker entries, schema/policy references, and their tests. Route every task launch without an explicit model through
  the existing phase-size map; legacy missing size selects the small phase alias. Preserve explicit model precedence.
- Reuse the phase plan-first predicate when rendering task prompts so large/xlarge tasks add `#plan`. Update launch
  previews and tests across all five sizes, explicit models, legacy sizeless tasks, and deterministic single-segment
  task prompts.
- Add CLI golden/parser/auto-commit/push/error tests for required evidence, wrong bead types, missing size, shorthand
  IDs, refs, reporter idempotency, and ready promotion.

## Phase 3: Task +1 presentation across every user surface

Create one small Python presentation layer for +1 labels, count badges, colors, and evidence formatting so surfaces
share language and palette rather than reproducing ad hoc strings.

- In `sase bead show`, full list/search output, compact list/search/ready/blocked rows, stats, and JSON envelopes,
  expose a `+N` badge (when nonzero), a visually distinct `+1 EVIDENCE` section with reporter/time/ref context,
  aggregate +1 totals where appropriate, and structured machine fields. Include evidence text and refs in search.
- In published bead pages, add the count to task identity/roster facts and render each evidence entry as a bounded,
  structurally safe callout with resolvable artifact links. Keep page output deterministic and byte stable.
- In ACE's Artifacts/Beads pane, show a count badge on task rows, a `+1 reports` property and distinct evidence cards in
  detail/preview, make the evidence searchable, and add a `has:+1` filter token. Require an actual size in the create
  modal. Also carry the count/evidence through the Agents tab's associated task-bead summary and BEAD lane rather than
  showing it only in the Artifacts tab.
- Add the count and structured evidence to task-triage notification presentation/preview and the mobile bead payload.
  Replace the triage reconciler's contract-only freshness check with a stable presentation fingerprint (including title,
  description, notes, size, refs, and +1 evidence), so a pending gate is regenerated when later corroboration changes
  the task instead of displaying a stale snapshot.
- Extend unit, JSON parity, page golden, TUI rendering, filtering, notification, mobile, and PNG visual snapshot tests.
  Preserve cached/off-thread bead loading and avoid storage reads in render or keystroke paths.

## Phase 4: Concise sase_new_task skill and agent policy

Add `src/sase/xprompts/skills/sase_new_task.md` as the generated skill source with a trigger description strong enough
that every agent about to create a task uses it. Keep the body terse and operational:

1. First run
   `sase memory read sase_beads.md --reason "Need task-bead lifecycle and duplicate policy before recording discovered work"`.
2. Form a candidate title/scope and gather reproducible evidence. Register a durable artifact when a file materially
   supports the report, retaining its canonical reference.
3. Run `sase bead list` with task type, full/unlimited output, and all statuses; inspect plausible matches with
   `sase bead show`. For a semantic duplicate, run `sase bead +1 ...` with independent reproduction/impact evidence and
   artifact refs when available. Do not create a task.
4. Independently list in-progress epic plan beads and inspect plausible epics and children. If an epic has a credible
   causal link—not merely topical overlap—append a clearly labeled `DISCOVERED ISSUE:` note to the epic bead with
   reproduction, impact, and artifact refs. If both duplicate and epic cases apply, record both. Do not create a task.
5. Only when neither case applies, choose a required size, create an `open` task with an evidence-rich description and
   refs, refine dependencies/scope, then mark it `ready`. Explain sizes compactly: xsmall is nearly mechanical; small is
   focused and straightforward; medium is substantial but bounded with a known design; large needs its own planning
   handoff; xlarge is rare and likely needs epic decomposition or deliberately deferred planning.

Add source-generation tests that pin the essential commands, branch rules, “no task” outcomes, evidence contract, and
size guidance without forcing redundant prose into the skill.

Update the canonical `sase/memory/sase.md` and `sase/memory/sase_beads.md` notes, their default templates, and all raw
agent-facing task-creation examples to require `/sase_new_task` and a size. This request explicitly authorizes those
memory edits; after editing, run `sase memory init` so `AGENTS.md` and every provider instruction shim are regenerated
as required. Do not hand-edit generated instruction files.

Update the built-in task-worker and epic-lander xprompts to invoke `/sase_new_task` for genuinely distinct follow-ups.
Retain the phase-worker prohibition on task creation. Strengthen the land prompt's existing verification step to say
explicitly that the lander must review the epic bead's own notes and every child bead's notes before integration,
triage, and close; unresolved issues caused by that epic remain epic work rather than new tasks.

Update living bead, ACE, xprompt, model-routing, configuration, and onboarding documentation to describe +1 semantics,
required size, legacy fallback, and the new skill. Remove current-behavior references to `@task_worker`, including
examples and blog material that claims sizeless task creation is supported. Preview generated skill output with
`sase skill init --diff`; do not deploy shared chezmoi skill targets from a dirty or unlanded tree. Record the clean,
post-land `sase skill init --force` deployment step in the final handoff if it cannot safely run during the epic.

## Phase 5: Cross-repository verification and contract cleanup

Integrate the linked-core and main-repository work, resolve any phase-seam drift, and verify the public workflow as one
feature.

- In `sase-core`, run formatting and the focused bead/event/storage tests, then the appropriate workspace-wide Rust test
  suite. Confirm old fixture projections still parse and new +1 streams reduce identically after merge/export.
- In the SASE repository, run `just install` before other checks, then `just check`. Run focused CLI, task-work,
  task-triage, page, mobile, generated-skill/memory, model-alias, and ACE tests while iterating; run `just test-visual`
  and inspect diffs before accepting only intentional +1/required-size snapshot changes.
- Exercise an end-to-end temporary bead store: reject sizeless task creation; create a sized task; +1 it with evidence
  and an artifact ref; prove creator/same-agent retries do not increment; prove a second agent does; show/list/search
  and ACE/triage projections agree; and prove +1 promotes draft and closed tasks to ready without losing history.
- Re-run `sase memory init` if any canonical memory text changed during integration, validate generated shims are in
  sync, and use `sase skill init --diff` to verify `/sase_new_task` generation for every supported runtime.
- Sweep for stale direct agent instructions that call `sase bead create -T task` without `/sase_new_task` or `--size`,
  and for remaining model-role references to `task_worker`. Ordinary implementation terms such as Textual task workers
  are not part of that alias cleanup.

The final handoff should summarize the command contract, compatibility behavior, status-promotion rule, visual surfaces,
tests run in both repositories, and any clean-tree generated-skill deployment still required after landing.
