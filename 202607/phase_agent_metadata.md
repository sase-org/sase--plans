---
tier: tale
goal: 'ACE shows the full SASE PLAN section only for the agent that authored an epic
  and the agent that lands it, while a selected epic phase worker shows only its own
  Bead metadata derived from the epic plan frontmatter without a bead-store read on
  the modern path.

  '
create_time: 2026-07-16 06:40:23
status: wip
prompt: 202607/prompts/phase_agent_metadata.md
---

# Plan: Make epic phase-agent metadata role-aware

## Context

ACE currently resolves an associated plan for any agent whose name or metadata points at an epic or phase bead. That
shared association is then rendered as the complete `SASE PLAN` section, so selecting a phase worker incorrectly shows
the same goal, path, and full phase roadmap as the epic's author. The `Bead:` row is enriched independently through the
bead store, even though approved epic creation already materializes phase beads deterministically from the validated,
ordered `phases` frontmatter in the linked plan file.

The intended metadata contract is:

- The epic-authoring planner keeps the full `SASE PLAN` section, including the complete authored phase roadmap.
- The epic lander keeps the same full `SASE PLAN` section.
- A phase worker never renders `SASE PLAN`; it renders only `Bead: <phase bead id> - <that phase's display text>`.
- The phase display text follows the existing bead-display semantics using the matching validated frontmatter phase:
  normalize and prefer its authored description, and preserve the deterministic fallback used when no description was
  authored. No other epic phases leak into the selected phase worker's metadata.
- Existing/historical phase-agent records remain readable through a compatibility fallback, but newly launched phase
  agents have enough explicit metadata to resolve the plan and phase without querying bead storage.

## Implementation

1. Extend the epic-work launch metadata contract at the point where `sase bead work` already has the epic issue, linked
   plan reference, ordered phase assignments, and per-segment environment. Persist the epic plan reference and explicit
   epic/phase bead identities into each phase agent's `agent_meta.json`; persist the epic identity and plan reference
   for the lander as well. Reuse the existing `AgentMetaWire`/ACE model fields where their semantics fit, keep
   filesystem and snapshot enrichment equivalent, and add narrowly named launch environment fields only where needed to
   carry this information into the child before its metadata marker is written. Do not change `SASE_PLAN` commit
   attribution as an accidental side effect.

2. Refactor the associated-plan/frontmatter enrichment into a role-aware result. Use the existing mtime/size-bounded
   plan cache and authoritative epic validator to read a directly associated plan once off the Textual event loop. For a
   phase worker, match its phase bead identity to the corresponding frontmatter phase using the deterministic creation
   order, construct the normalized `Bead:` value from that one phase, and deliberately return no renderable
   associated-plan section. For the authoring planner and lander, retain the current complete plan summary and
   responsive roadmap. Make the classification explicit for current metadata and supported legacy exact/`.land` naming
   so a dotted phase name is not mistaken for an epic lander.

3. Integrate the phase result with the detail-header cache and asynchronous refresh flow. The immediate selection path
   must remain memory-only; plan stat/read/validation stays in the existing debounced worker. Explicitly identified
   phase agents should bypass the generic bead-confirmation lookup/worker (and visible-row bead warmup where
   applicable), while the enriched result updates only the selected agent after the usual generation/identity staleness
   checks. Keep a bounded legacy fallback for old records that lack the new plan/phase metadata, but suppress
   `SASE PLAN` for those phase rows too. Ensure cache keys include every plan path and phase identity input needed to
   prevent one phase's display from being reused for another.

4. Add regression coverage across the launch and ACE boundaries:
   - epic work emits distinct author/phase/land metadata and persists it through both filesystem and wire-backed agent
     enrichment;
   - a modern phase worker selects the correct first, middle, or nested-ID phase from validated frontmatter without any
     bead lookup, normalizes multiline descriptions, and uses the same no-description fallback as deterministic bead
     creation;
   - phase workers render one `Bead:` row and no `SASE PLAN`, no goal/path/full roadmap, and do not schedule the generic
     bead resolver; stale async results cannot overwrite a newly selected phase;
   - epic author/planner and modern or legacy lander rows still render the complete `SASE PLAN` roadmap, while ordinary
     tale-plan behavior and unrelated confirmed-bead behavior remain unchanged;
   - missing, damaged, or out-of-range phase metadata fails quietly to the safest bead-only representation and never
     exposes a full epic plan on a phase row.

5. Update the ACE documentation to describe the role-specific display contract and the plan-frontmatter data source for
   phase workers. Run focused bead-work, metadata-enrichment, associated-plan, header-rendering, async-worker, and
   visual tests while iterating, then run `just install` followed by the required `just check` for the repository-wide
   gate.

## Risks and guardrails

- Phase-to-frontmatter matching relies on the existing invariant that deterministic epic creation allocates phase beads
  in frontmatter order. Keep that invariant documented and tested at both creation and display boundaries rather than
  inferring from mutable bead descriptions.
- Do not perform plan or bead I/O in header construction, row rendering, selection handlers, or after an `await` without
  rechecking the selected identity. Reuse the existing deferred worker and mtime-keyed cache.
- Do not remove the general bead-display path for ordinary bead agents or legacy records; scope the fast path to agents
  positively identified as epic phase workers.
