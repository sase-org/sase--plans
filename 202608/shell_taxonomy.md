---
tier: epic
title: Sase agent and shell taxonomy migration
goal:
  Replace the agent-lane term with the canonical sase-agent and sase-shell model,
  preserve serialized and Python compatibility where required, and migrate monitor CLI
  language without changing runtime behavior.
phases:
  - id: sase-agent-projection
    title: Canonical sase-agent projection and compatibility aliases
    depends_on: []
    size: medium
    description:
      "sase-agent-projection: introduce SaseAgentRef and sase-agent projection helpers
      as the canonical vocabulary, retain narrow AgentLaneRef and lane_*
      import/serialization aliases for compatibility, clarify concrete-shell versus
      family/container provenance, and update focused unit and integration tests without
      changing identity resolution or sidecar paths."
  - id: shell-glossary-surfaces
    title: Shell glossary and generated terminology surfaces
    depends_on:
      - sase-agent-projection
    size: medium
    description:
      "shell-glossary-surfaces: replace the canonical Agent Lane definition with Sase
      Agent, Sase Shell, Agent Shell, and Proc Shell entries; revise Agent Family and
      Proc; migrate only genuine agent-lane presentation in ACE, docs, errors, statuses,
      and tests; run sase memory init and validate generated instruction and memory
      surfaces while leaving unrelated AXE, test, display, and launch-routing lanes
      unchanged."
  - id: monitor-agent-cli
    title: Monitor agent CLI language and compatibility
    depends_on:
      - sase-agent-projection
    size: medium
    description:
      "monitor-agent-cli: rename monitor lane-facing language and filters to agent, use
      -a/--agent for start, retain -a/--all plus -l/--agent for list, accept deprecated
      --lane compatibility aliases without advertising them, and update handlers,
      completions, skill source, docs, errors, JSON scope compatibility, and focused
      tests while preserving monitor behavior."
proposed_by: bbugyi200.athena.sase-m9.1
parent_bead: sase-m9.1
status: done
bead_id: sase-m9.1.1
create_time: 2026-09-09 19:51:31
---

- **PROMPT:**
  [prompts/202608/shell_taxonomy.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/shell_taxonomy.md)
- **PARENT:** [202608/supervised_proc_shells.md](supervised_proc_shells.md)
- **BEAD:**
  [sase-m9.1.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-m9/sase-m9.1.1.md)

# Plan: Sase agent and shell taxonomy migration

## Outcome

Make “sase agent” the stable projection currently called an agent lane and define a sase
agent as an ordered sequence of sase shells. An agent shell is one concrete LLM/provider
run; a proc shell is a named supervised proc attached to a sase agent. This child epic
is terminology- and compatibility-focused: it must not implement the proc-shell
lifecycle architecture assigned to later phases of the parent epic.

The migration must preserve family parsing, ownership normalization, sidecar paths,
commit attribution, monitor records, monitor execution, ACE interaction, and existing
serialized data. Compatibility names may remain at explicit boundaries, but new code,
help, and documentation use the canonical vocabulary.

## Scope rules

- Rename “lane” only where it means an agent family or a standalone agent. Do not rename
  AXE scheduling lanes, scoped-test lanes, ACE display/layout lanes, or launch-routing
  lanes.
- `SASE_AGENT_NAME` continues to identify the concrete agent shell. The family
  projection and the `SASE_AGENT=` commit footer identify the sase agent.
- `sase agent list` remains a listing of concrete agent shells. Its presentation and
  tests should make that meaning clear. `sase agent kill` help must say that it kills a
  live agent shell, not a family container.
- Do not add a top-level `sase shell` command. Reserve “interpreter” for a command
  interpreter and do not overload existing shell-related CLI flags.
- Historical fields, imports, monitor records, and externally consumed JSON must stay
  readable. Prefer deprecation aliases and dual-read/single-write transitions over a
  flag day; test every retained alias.
- The canonical glossary memory edit is explicitly authorized by the parent design.
  After editing `sase/memory/glossary.md`, run `sase memory init` and include all
  intended generated memory/agent-instruction updates.
- Generated skill destinations are not edited directly. Change
  `src/sase/xprompts/skills/sase_monitor.md` and use read-only generation preview in
  this epic; deployment occurs only from a clean landed commit.

## Phase 1: Canonical sase-agent projection and compatibility aliases

Inventory `src/sase/agent_lanes.py` and all provenance, publication, association,
history, attachment, and commit callers. Introduce the canonical `SaseAgentRef` model
and helper names (including projections from a concrete agent shell and from an
already-projected sase-agent name). Either rename the module with a compatibility shim
or expose both vocabularies from a canonical module; choose the least disruptive shape
after checking import-cycle and static-lint constraints.

Keep the underlying Rust-owned family-name parsing and globalization behavior exactly
as-is. Preserve the four pieces of projection data: local sase-agent name, global
provenance name, family/container knowledge, and optional concrete member name. Retain
temporary `AgentLaneRef` and `lane_*` aliases for internal/external callers where a hard
cutover risks serialization or import breakage, clearly marking which are compatibility
surface. Do not rename unrelated generic lane concepts.

Migrate callers that express provenance or ownership to canonical names, including agent
association/publication, prompt archive, hosted links, artifact providers, commit
finalization, and image attachment boundaries. Update docstrings and errors so they
distinguish a concrete agent shell from its sase-agent projection. Add focused tests
proving solo-agent, family-member, reserved-family, local/global, legacy-alias,
page-path, and missing-registry behavior is unchanged. Update provenance tests to prove
`SASE_AGENT_NAME` projects a concrete shell while `SASE_AGENT=` records the sase agent.

## Phase 2: Shell glossary and generated terminology surfaces

Edit the canonical glossary to replace Agent Lane with these definitions, polished for
house style but not semantically changed:

- **Sase Agent** (alias: agent): an agent family or a single agent belonging to no
  family; owns an ordered sequence of sase shells; its name never ends in `--<suffix>`;
  a one-shell agent may share its shell name, while a family uses the bare name for its
  container.
- **Sase Shell** (alias: shell): one executing member of a sase agent, either an agent
  shell or proc shell; a sase agent is a sequence of sase shells.
- **Agent Shell**: one concrete LLM/provider run; it holds a workspace claim while
  active and is the execution row shown by `sase agent list`.
- **Proc Shell**: a named supervised proc belonging to a sase agent, with durable output
  and lifecycle state; a family-attached proc shell is a monitor and may carry timeout,
  workspace-claim, and follow-up policy.

Revise Agent Family to describe the sequential shell chain. Revise Proc so it does not
present `command`, `tui`, and `detached` as permanent semantic kinds, while retaining
their historical compatibility meaning. Run `sase memory init` and inspect the generated
README and provider instruction shims for consistent output.

Migrate genuine user-facing agent-lane labels in ACE help, counts, confirmation text,
neighbor/folding presentation, docs, and tests to sase-agent or agent-shell wording as
appropriate. Internal presentation helpers may be renamed when doing so improves the
model, but avoid broad mechanical churn that obscures behavior. Explicitly audit every
remaining `agent lane`, `AgentLane`, and `agent_lane` occurrence and document why any
compatibility occurrence remains. Update glossary-link/render tests and relevant ACE
tests. Run the visual snapshot suite only if rendered ACE text or layout changes;
inspect diffs and accept goldens solely for intentional terminology changes.

## Phase 3: Monitor agent CLI language and compatibility

Change monitor ownership terminology from lane to agent without changing the record
model or execution lifecycle. For `sase monitor start`, expose `-a/--agent NAME`. For
`sase monitor list`, retain `-a/--all` and expose `-l/--agent NAME`; clearly document
the intentional short-option difference. Keep `--lane` as a suppressed compatibility
alias during this migration and test both old and new forms. Parser destinations may
remain compatible internally or normalize once in the handler, but new help, examples,
errors, statuses, completion candidates, and JSON-facing documentation use “agent.”

Update `src/sase/xprompts/skills/sase_monitor.md`, monitor documentation, CLI docs,
parser and handler tests, completion tests, and any snapshots. Use
`sase skill init --diff` or `--dry-run` to validate generated skill output without
deploying dirty workspace content. Preserve list filtering, current-agent defaults,
cwd/workspace resolution, conflict detection, start/stop/show behavior, and historical
monitor rows.

The existing monitor start command has required options that conflict with current CLI
rules. Do not expand this terminology phase into the proc-platform redesign; record the
required-command/reason/timeout correction as an explicit handoff to the dependent
proc-shell phase unless a minimal compatibility-safe parser correction is demonstrably
necessary for the alias work.

## Verification and landing

Each phase runs its focused unit/integration tests and audits terminology with `rg`.
Before landing the combined child epic, run `just install`, then run `just check-full`
through `/sase_monitor` with a concrete `--next` action. If any phase changes rendered
ACE strings or layout, also run `just test-visual`, inspect actual/expected/diff
artifacts, and update snapshots only when the new terminology is intentional.

Final review must confirm:

1. Projection and provenance behavior matches the pre-migration implementation,
   including every documented compatibility alias.
2. The canonical glossary, generated memory surfaces, CLI help, skills, docs, errors,
   and statuses consistently describe sase agents and sase shells.
3. Monitor `start` and `list` advertise the specified `--agent` forms, old `--lane`
   invocations still work during the compatibility window, and `-a` retains its
   command-specific meanings.
4. `sase agent list` and `kill` describe concrete agent shells accurately.
5. Remaining lane terminology belongs either to a named compatibility boundary or to an
   explicitly out-of-scope lane concept.
6. No proc supervisor, proc schema, ACE proc ownership, or top-level shell command was
   introduced in this child epic.
