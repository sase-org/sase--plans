---
tier: epic
title: Artifact links that survive, derive themselves, and pay for the turn
goal: "The artifact-link graph stops losing durable rows, grows to ~1,600 edges derived
  from facts SASE already owns without adding one token to the average agent's context,
  and becomes load-bearing: every audited read shows its neighborhood, ACE carries the
  relation type, and the Artifacts Agent pane filters on `relation:`, `linked:`, and
  `artifact:`.

  "
phases:
  - id: index-durability
    title: A rebuild may delete only what it can prove was deleted
    depends_on: []
    size: medium
    description: "index-durability: make `preview_aggregate` carry forward prior rows
      whose owning companion is not visible in this workspace, make `link rm` prune the
      aggregate unconditionally, add a cross-workspace reconciliation sweep gated on
      agent publication, and add doctor counters that make future index divergence loud.

      "
  - id: read-outbox
    title: Audited reads become durable and publish with the agent's commits
    depends_on:
      - index-durability
    size: medium
    description: "read-outbox: add a machine-local, replayable artifact-link outbox so
      `sase artifact read` stops leaving its row uncommitted, and drain it at stitch
      publication and from housekeeping, publishing an agent-endpoint row only once that
      agent resolves as published.

      "
  - id: bead-endpoints
    title: A bead in either endpoint position gets its event
    depends_on:
      - index-durability
    size: medium
    description: "bead-endpoints: teach the sase-core bead link wire a direction, fire
      `_upsert_bead` when the bead is the target as well as the source, backfill the
      endpoint events one-sided writes never produced, and make `sase artifact create
      --bead` write a typed link with `reference_added` kept as a legacy alias.

      "
  - id: rename-following
    title: Links follow renames instead of dangling
    depends_on:
      - index-durability
    size: medium
    description: "rename-following: consume git rename detection in the sidecar link
      refresh so a moved artifact carries its rows and its `links/` companion, then
      repair every dangling ref and orphaned companion the research workflow has already
      created.

      "
  - id: relation-registry
    title: Relation semantics, `derived` projection class, and a way to read them
    depends_on: []
    size: medium
    description: "relation-registry: settle `plan implements bead` in the registry with
      direction, worked examples, and recommended endpoint kinds; expose it through
      `sase artifact link relation list|show` and completions; and give `derived` a
      projection class so host-derived semantic edges render.

      "
  - id: derivation-core
    title: One derivation module behind one flag
    depends_on:
      - relation-registry
      - bead-endpoints
    size: medium
    description: "derivation-core: add the Textual-free derivation module that turns
      facts SASE already owns into `derived` rows -- research-swarm `__a`/`__b` lineage
      and plan `bead_id:` frontmatter -- behind a beta feature flag, with no call sites
      yet.

      "
  - id: derivation-hooks
    title: Derive at creation, on sidecar commit, and in the hourly sweep
    depends_on:
      - derivation-core
      - rename-following
      - read-outbox
    size: medium
    description: "derivation-hooks: call the derivation module from `sase plan propose`
      and `sase artifact create`, from the sidecar commit path for artifacts that land
      another way, and from a new `sase_chop_artifact_link_backfill` chop in the hourly
      housekeeping bucket that runs the retroactive sweep.

      "
  - id: citations
    title: The citation channel stops starving
    depends_on:
      - derivation-hooks
    size: medium
    description: "citations: derive `agent cites plan` from the archived prompt's own
      header block and resolve prose sidecar paths in published prompts into `cites`
      rows, while explicitly holding the plan `AGENTS`/`COMMITS` sections out of the
      graph.

      "
  - id: notes-migration
    title: "Run the RELATED: note backfill"
    depends_on:
      - bead-endpoints
    size: small
    description: "notes-migration: verify `sase artifact link migrate-notes --apply` on
      a scratch bead tree, correct its stale help text, and run it for real to convert
      the 269 auto-convertible `RELATED:` notes into typed related links.

      "
  - id: frontmatter-inlet
    title: Finish the `links:` frontmatter inlet as an authoring path
    depends_on:
      - derivation-core
    size: medium
    description: "frontmatter-inlet: give the orphaned Rust `links:` parser its
      production caller so an author can declare links in plan frontmatter, and make
      consumption remove the inlet so it never becomes a second editable copy of the
      row.

      "
  - id: read-payoff
    title: Make every audited read a discovery moment
    depends_on:
      - index-durability
      - relation-registry
    size: medium
    description: "read-payoff: print a one-line typed neighborhood footer from `sase
      artifact read`, warn when the artifact being read is superseded, and expand
      exactly one typed hop when a launch prompt cites an artifact.

      "
  - id: ace-fidelity
    title: ACE stops flattening the relation type
    depends_on:
      - relation-registry
    size: medium
    description: "ace-fidelity: carry relation slug, directional label, description,
      origin, and use count from the aggregate row into ACE's relation rail instead of
      collapsing every edge to `links`/`linked_by`, and add a typed link action with a
      required reason.

      "
  - id: agent-pane-filters
    title: "`relation:`, `linked:`, and `artifact:` on the Agent pane"
    depends_on:
      - index-durability
      - ace-fidelity
    size: medium
    description: "agent-pane-filters: join artifact-link facets onto the agent catalog
      query entry and declare the three link fields in the agents profile so both the
      Artifacts Agent pane and `sase agent search` can filter on them.

      "
  - id: suggest-and-close
    title: The judgment tier, the flag removal, and the two workflow lines
    depends_on:
      - citations
      - notes-migration
      - frontmatter-inlet
      - read-payoff
      - agent-pane-filters
    size: medium
    description:
      "suggest-and-close: add `sase artifact link suggest` over hard evidence only,
      remove the derivation feature flag, add the single fact line to `/sase_plan` and
      the research swarm lead step, and turn coverage into a doctor report rather than a
      gate."
proposed_by: bbugyi200.athena.sase-tj.land.w3
bead_id: sase-tw
create_time: 2026-09-09 19:49:51
status: wip
---

- **PROMPT:**
  [prompts/202608/artifact_link_durability_and_derivation.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/artifact_link_durability_and_derivation.md)
- **BEAD:**
  [sase-tw](https://github.com/sase-org/sase--beads/blob/main/pages/sase-tw/README.md)

# Plan: Artifact links that survive, derive themselves, and pay for the turn

## Problem

`research:202608/artifact_link_derivation/artifact_link_derivation.md` consolidates two
independent studies plus lead verification into one finding: SASE's link substrate is
well designed, its instruction channel is saturated and produces nothing, and the graph
it does hold silently evaporates. Every claim below was re-measured on this tree today
before this plan was written.

**The index is lossy.** `ArtifactLinkStore.preview_aggregate`
(`src/sase/sdd/_artifact_link_store_impl.py:233`) rebuilds the machine-global aggregate
from the _triggering workspace's_ sidecar clone plus bead events, carrying forward from
the prior aggregate only rows where `_is_aggregate_only(row)` is true — rows where
neither endpoint owns sidecar JSON anywhere. Every `plan:` and `research:` edge fails
that test, so any agent in any ephemeral workspace can shrink the machine-global graph
by writing one link. Measured in this workspace today:

| Store                                            | Rows |
| ------------------------------------------------ | ---: |
| Sidecar rows visible here (`_iter_sidecar_rows`) |  148 |
| Machine-global aggregate                         |  112 |
| What a rebuild from here would write             |  180 |

By relation, the aggregate holds `derives-from` **2** against 35 sidecar instances (20
unique). The graph loses rows on an unlucky rebuild, and the aggregate is the layer
every consumer reads.

**Audited reads never commit.** `handle_link_add` calls `store.upsert_row(...)` and then
`persist_artifact_link_graph_mutation(...)`. `_record_read_link`
(`src/sase/artifact_cli/read.py:243`) calls `upsert_row` and stops. Reproduced live
while writing this plan:
`sase artifact read research:202608/…/artifact_link_derivation.md` left
` M links/202608/artifact_link_derivation/artifact_link_derivation.md.json` uncommitted
in the research sidecar. Measured: 208 read events, 169 with `recorded_link: true`, 162
distinct `(agent, ref)` pairs, **100** surviving anywhere durable — 62 pairs lost
outright. Filed as `sase-t0`.

**A link whose target is a bead never reaches the bead.** `_upsert_bead`
(`_artifact_link_store_impl.py:373`) reads `bead_id_from_ref(incoming["source_ref"])`
and returns `None` when that is not a bead. `related` is registry-undirected, so a
one-sided write is half an edge, and every `plan implements bead` row this epic derives
would be invisible from the bead — the direction a person browsing beads actually wants.
Filed as `sase-t1`.

**Links do not survive renames, and the research workflow renames by design.**
`#research_swarm` step 3 always moves both source reports into the consolidation
directory. `sase/repos/research/links/202608/` currently holds orphaned companions for
`artifact_link_adoption.md`, `artifact_link_adoption_and_quality.md`,
`finalizer_bugs_and_capability_upgrades.md`, `sase_native_finalizer_use_cases.md`,
`generalized_finalizer_use_cases.md`, `stand_alone_proc_shells.md`,
`task_types_finalizer_migration.md`, `xprompt_if_directive.md`, and
`conditional_launch_if_directive/`, and `sase artifact doctor` reports the matching
dangling refs. Git already records these as pure renames; the refresh pass does not
consume that.

**Nothing consumes links, so nothing rewards writing them.** `_emit_link_edge`
(`src/sase/ace/tui/relations/artifact_links.py`) builds every `RelationEdge` from the
pane declaration's generic `links`/`linked_by` and discards the row's `relation`,
`description`, `origin`, and `uses` — the Markdown projection preserves more semantic
information than the interactive UI. `sase artifact read` strips the managed blocks and
the frontmatter, so the sanctioned read path shows _fewer_ links than `cat`. `derived`
is a valid origin with zero live rows and no projection class
(`_CURATED_ORIGINS = {"manual", "migrated"}`, `_artifact_link_projection.py:22`), so a
host-derived semantic edge would render nowhere.

**And the instruction channel is already saturated.** 893 Tier 1 characters in front of
every agent on every turn, plus 135 `sase memory read sase_artifacts.md` reads in five
days, produced five deliberate links across 749 non-owner runs — 0.7%. `implements` and
`supersedes` have each been written zero times. Next door, the bead dependency graph
holds ~3,097 live edges with no memory file evangelizing it, because a bead dependency
gates a launch. Edges get written when they are load-bearing, not when they are
documented.

Meanwhile the derivable population sits untouched: **602** plans carry `bead_id:`
frontmatter with zero `implements` rows, **58** research-swarm consolidations have their
`__a`/`__b` pair present on disk with one linked, **3,066** plans carry a `PROMPT:`
header, and `sase artifact link migrate-notes` reports **269** of 303 `RELATED:` bead
notes as auto-convertible with the apply path never once run.

## Decisions the owner already made

These were the research report's open questions. The answers are settled inputs to this
plan, not things a phase re-litigates.

1. **A plan implements a bead's requirements.** `plan:<relpath> implements bead:<id>`
   derived from `bead_id:` frontmatter is correct and directed that way.
2. **`sase artifact create --bead` writes a typed link.** `reference_added` is retained
   as a legacy alias, not a second vocabulary.
3. **The Agents sub-tab exists.** `agent:` refs have a real destination, so `agent:`
   edges stay and the prompt-header `cites` derivation is unblocked.
4. **Read-log derivation covers every read**, not a `plan:`/`research:`-filtered subset.
5. **Reconcile the graph across workspace clones, but publish an agent-connected row
   only once that agent has made commits** — and therefore has itself been published.

Point 5 has an exact mechanical test that already exists:
`resolve_cli_reference("agent:<name>")` returns `exact` for a published agent and
`missing` for one that never committed. Verified today: `agent:sase-tj.land` → `exact`;
`agent:0bc` → `missing`. Every phase that publishes an agent-endpoint row uses that
resolution as the gate, and none invents a second notion of "published".

## Principles this epic holds to

Taken from the research report's §6 and §10; a phase that wants to deviate must say so
in its bead notes rather than quietly widen scope.

- **Do not tell agents to link more.** Nothing is added to `AGENTS.md`. Nothing is added
  to `/sase_final`. The only agent-facing text this epic adds is one fact line in
  `/sase_plan` and one in the research swarm lead step (`suggest-and-close`), both read
  only by agents that produce a linkable artifact.
- **Do not infer semantic lineage from access.** A `read` is a candidate, not a
  `derives-from`. The observational plane (`read`, `cites`) stays separate from the
  semantic plane (`derives-from`, `implements`, `supersedes`, `related`).
- **Materialize from the native association.** Where SASE already owns the relationship
  — a plan's `bead_id:`, a swarm's `__a`/`__b` shape, a prompt's header block — the row
  is a projection of that fact, not an independent assertion that can drift.
- **No false completeness.** Coverage is a report, never a gate and never a quota.
- **Do not widen the relation registry.** Six slugs with two at zero usage is not a
  vocabulary problem. Revisit after the derived graph shows what is actually missing.

## Deviations from the research report

Two, both deliberate.

**The reactive derivation trigger is a host step, not a user-configured file hook.** The
report recommends `file_hooks:` because the machinery exists and costs nothing.
`src/sase/default_config.yml` ships `file_hooks: []`, so a shipped entry would be
user-visible configuration for host behavior, and a user who edits their chezmoi config
would silently turn the graph off. `derivation-hooks` instead calls the derivation
module from the sidecar commit path with the `artifact_links` cause excluded — the same
event, the same exclusion, host-owned. The `file_hooks` mechanism stays available for
anyone who wants an additional project-local hook.

**`sase artifact link suggest` ships in this epic rather than after it.** The report
puts it in a later Phase 5 on the grounds that a suggester with no derived baseline
re-proposes what derivation should have produced. That reasoning holds and is why
`suggest-and-close` depends on every derivation phase. But the owner's answer to Q4
("every read") only has a consumer if the read-log candidate surface exists, so it lands
here — writing nothing, hard evidence only.

## Sequencing note: the Agent pane is still landing

Epic `sase-tj` shipped the Artifacts Agent pane; its child epic `sase-tj.10` is in
progress and is fixing three landing gaps, one of which touches `sase agent search`
option parsing and one of which touches the pane's fast-startup inventory and PNG
goldens. `agent-pane-filters` edits both surfaces. That phase must not start until
`sase-tj.10` has landed on master; its agent should check `sase bead show sase-tj.10`
first and wait rather than rebase around in-flight pane work.

Tasks `sase-t0` (§3.2, audited reads never commit) and `sase-t1` (§3.3, bead in target
position) are `READY` task beads that this epic subsumes. `read-outbox` and
`bead-endpoints` respectively should close them with a resolution pointing at their
phase bead.

---

## A rebuild may delete only what it can prove was deleted

**Files.** `src/sase/sdd/_artifact_link_store_impl.py`,
`src/sase/sdd/_artifact_link_store_support.py`, `src/sase/artifact_cli/link_health.py`,
`src/sase/artifact_cli/doctor.py`.

The durable layer is sound; the cache-rebuild rule is the bug. Replace the current
"carry forward only rows that own no sidecar anywhere" rule with "carry forward every
prior row whose authoritative source was not consulted in this workspace".

Add a private classifier next to `_is_aggregate_only`:

- A prior row is **bead-authoritative** when `self.beads_dir is not None` and its
  `source_ref` is a bead. `_iter_bead_rows()` just re-read that truth, so a prior row
  missing from the collection was genuinely removed. Keep this test narrow to the source
  position in this phase — `bead-endpoints` widens it to either endpoint in the same
  change that starts writing the target-position event, and widening it earlier would
  drop `plan implements bead` rows whose plan companion is not cloned here.
- A prior row is **sidecar-authoritative** when at least one endpoint owns a sidecar
  root registered in `self.sidecar_roots` _and_
  `sidecar_index_path(root, ref).is_file()` in this workspace. `_iter_sidecar_rows()`
  read that exact file, so a prior row absent from the collection is proven deleted.
- Otherwise the row is carried forward unchanged.

`_remove_aggregate_rows` currently drops a matched row only when
`_is_aggregate_only(raw)` is true, which was correct only because a rebuild would drop
the rest anyway. With an additive rebuild that is now a leak: make it drop every row
matching `pair_matches(...)`, regardless of ownership, so `sase artifact link rm` still
takes effect in the aggregate.

Add `ArtifactLinkStore.reconcile_aggregate()` — the destructive, authoritative pass that
`preview_aggregate` deliberately is not. It unions the current aggregate with the
`links/**/*.json` rows of **every** workspace clone of this project, not just this one,
resolving sibling workspaces through the same registry `resolve_artifact_link_store`
already uses to find this one, and treating an unreadable or absent clone as
contributing nothing rather than as evidence of deletion. Rows whose `source_ref` or
`target_ref` is an `agent:` ref are admitted only when `resolve_cli_reference(ref)`
resolves (owner decision 5); an unpublished agent's rows stay out of the machine-global
index and are neither lost nor published. Wire it to `sase artifact doctor --fix`.

Add two counters to `ArtifactLinkHealthReport` and the doctor panel, as reports:

- durable sidecar row count (deduplicated) versus aggregate row count, with the delta;
- read events carrying `recorded_link: true` versus durable `read` rows, with the delta.

Both are exactly the measurements that made this defect visible; making them standing
output means the next divergence is loud instead of silent.

**Acceptance.** A test builds two sidecar clones of one project, writes a row from clone
A, rebuilds from clone B where the companion file does not exist, and asserts the row
survives; a second test deletes the row from clone A's companion, rebuilds from clone A,
and asserts it is gone. `sase artifact link rm` removes the row from the aggregate.
`sase artifact doctor` prints both counters, and on this tree the durable-versus-index
delta closes.

## Audited reads become durable and publish with the agent's commits

**Files.** `src/sase/artifact_cli/read.py`, `src/sase/sdd/_artifact_link_commit.py`,
`src/sase/workflows/commit/workflow_publication.py`, new
`src/sase/sdd/artifact_link_outbox.py`, `src/sase/artifact_cli/link_health.py`. Closes
`sase-t0`.

The naive fix — call `persist_artifact_link_graph_mutation` from `_record_read_link` the
way `link_add` does — is wrong here for two reasons. It puts a git commit and a push on
the hot path of a command agents run many times per turn, and
`persist_artifact_link_graph_mutation` raises `ArtifactLinkPersistError` on an
unpublished commit, which must never fail a read. It also publishes an agent-endpoint
row immediately, which owner decision 5 forbids for an agent that has not committed.

Add a machine-local, replayable outbox at
`~/.sase/projects/<key>/artifact-link-outbox.jsonl`. Each entry carries the full
validated row plus the recording agent's name. It is append-only, it survives eviction
of the workspace that recorded it, and because a row is content-identified by
`(source, relation, target)` any workspace can replay it into its own clone idempotently
through `upsert_row`. `_record_read_link` keeps its existing `upsert_row` call — the row
still lands in this workspace's companion and in the aggregate, so ACE and
`sase artifact link list` see it immediately — and appends one outbox entry. Nothing
else changes on the read path; it stays fast and it never raises.

Draining is one function used from two places:

- **`sase stitch create`**, in `workflows/commit/workflow_publication.py`, immediately
  after the `publish_agent_hood` step succeeds and before that step is checkpointed. At
  that instant the agent has demonstrably committed and been published, which is exactly
  owner decision 5's condition. Drain entries for the publishing agent, commit the
  touched `links/` indexes through the existing `commit_artifact_link_indexes` path with
  `push_after_commit="async"`, and record the step in `cp.completed_steps` so a
  `--resume` does not double-drain. A drain failure warns and leaves the entries queued;
  it never fails the commit that already succeeded.
- **`sase_chop_artifact_link_backfill`** (added in `derivation-hooks`), which drains
  entries for any agent that now resolves as published and leaves the rest queued.

An entry is dropped, with a doctor-visible count, only when its agent has been terminal
for longer than the retention window and still does not resolve — an agent that never
committed never publishes its reads, by decision.

Add the outbox depth and dropped count to the doctor link-health report.

**Acceptance.** A test records a read, asserts the outbox entry exists and the sidecar
is dirty but uncommitted, runs the drain for a published agent, and asserts a scoped
commit containing exactly the companion index; a second test runs the drain for an
unpublished agent and asserts the entry stays queued and nothing is committed. On this
tree, the 62 lost `(agent, ref)` pairs stop growing.

## A bead in either endpoint position gets its event

**Files.** `../sase-core` (`crates/sase_core/src/artifact_link/wire.rs`, the bead link
and event modules, bindings), `src/sase/bead/model.py`,
`src/sase/sdd/artifact_link_beads.py`, `src/sase/sdd/_artifact_link_store_impl.py`,
`src/sase/artifact_cli/create.py`. Closes `sase-t1`.

`BeadLinkWire` is `{target_ref, relation, description, origin}` and its doc comment says
"the bead itself is the source". That shape is why `_upsert_bead` can only ever handle
the source position: there is nowhere to record that the bead is the far end. Storing
the registry inverse slug instead is not an option — `implemented-by` and `read-by` are
labels, not registry slugs, and `validate_artifact_link_row` rejects them.

Per the Rust core boundary, this is core backend behavior: any frontend that reads a
bead needs the same answer. Add `direction: "out" | "in"` to `BeadLinkWire`, defaulting
to `"out"` so every stored event and every existing projection keeps its current
meaning, and thread it through the link-added and link-removed events, `add_link`,
`remove_link`, and the `IssueWire.links` projection. Release it the way the query
grammar widening was released for `sase-tj.1`, and let the release-branch reconciler
ratchet the pyproject window.

Then in Python:

- `_upsert_bead` resolves a bead id from `source_ref` **or** `target_ref` and writes one
  endpoint event per bead endpoint: `direction="out"` when the bead is the source,
  `direction="in"` when it is the target, storing the other endpoint's ref and the
  registry slug unchanged.
- `rows_from_bead_issues` projects a `direction="in"` link back into canonical order
  (`source_ref` = the stored `target_ref`, `target_ref` = `bead:<id>`), so an inbound
  edge and the sidecar row that also holds it collapse to one row under `unique_rows`
  instead of appearing twice.
- `_remove_bead_rows` already walks both endpoints; confirm it removes the inbound event
  as well.
- Widen `index-durability`'s bead-authoritative test from "source is a bead" to "either
  endpoint is a bead", now that both are re-derivable from the event store.

Backfill the endpoint events the one-sided writes never produced: for every existing
aggregate or sidecar row with a bead in the target position and no matching inbound
event, write the event. All 26 pre-existing `link_added` events have a bead as source,
so this is purely additive.

Finally, consolidate the two bead edge vocabularies (owner decision 2).
`_attach_reference_to_bead` in `artifact_cli/create.py` writes a `reference_added`
event; 94 rows across both concepts is cheap to unify now and expensive later. Make
`sase artifact create --bead` write a typed `related` link through the store, keep
`reference_added` readable as a legacy alias in every consumer that reads it today, and
stop writing new `reference_added` events.

**Acceptance.** `sase artifact link add plan:X implements bead:Y "why"` makes the edge
visible from `sase bead show Y` and on the generated bead page as well as from
`sase artifact link list plan:X`; the aggregate holds exactly one row for it. Both-state
tests cover a legacy `reference_added` event still rendering after the switch.

## Links follow renames instead of dangling

**Files.** `src/sase/sdd/_artifact_link_refresh.py`,
`src/sase/sdd/_artifact_link_files.py`, new rename-consuming helper in `src/sase/sdd/`,
`src/sase/artifact_cli/link_health.py`.

Git already computes the rename for free. Teach the sidecar link refresh to consume it:
for a commit touching a sidecar, ask git for rename pairs (`--find-renames`) restricted
to that sidecar's tracked artifacts, and for each `old -> new` pair rewrite every stored
row whose `source_ref` or `target_ref` names the old path, then move `links/<old>.json`
to `links/<new>.json` in the same commit. Both the row rewrite and the companion move
are one scoped mutation with the existing `artifact_links` cause, so the projection
commit does not re-trigger the pass.

This is the phase that makes derivation worth doing: `#research_swarm` step 3 moves both
source reports on every run, so a `derives-from` edge written before the move is
worthless without this.

Then repair what is already broken. Add a repair pass to `sase artifact doctor --fix`
that, for each dangling `plan:`/`research:` ref, looks for the rename in the owning
sidecar's history and rewrites the row and its companion; when no rename explains it,
the ref is reported, not deleted. Run it on this tree to clear the orphans listed in the
Problem section and the two dangling `research:` refs doctor reports today. Dangling
`agent:` refs are a different thing entirely — they are unpublished agents (owner
decision 5) — so classify them separately in the report instead of counting them as
breakage.

**Acceptance.** A test creates two linked research artifacts, `git mv`s one, runs the
refresh, and asserts the rows and the companion followed and nothing dangles. On this
tree `sase artifact doctor` reports zero dangling `plan:`/`research:` refs and zero
orphaned companions afterwards.

## Relation semantics, `derived` projection class, and a way to read them

**Files.** `../sase-core` relation registry,
`src/sase/sdd/_artifact_link_projection.py`, `src/sase/artifact_cli/link.py`, new
`src/sase/artifact_cli/link_relations.py`, completions,
`src/sase/artifact_cli/link_health.py`.

`implements` has zero live examples, which is why its direction was ambiguous enough to
split two researchers. Owner decision 1 settles it: **a plan implements a bead's
requirements**, so `plan:<relpath> implements bead:<id>`. Record that where it can be
read rather than in a plan nobody re-reads.

Extend each registry entry with the fields needed to disambiguate it without prose
elsewhere: a direction sentence, one positive and one negative worked example, and
recommended source and target kinds. `implements` gets `plan implements bead` as its
positive example and the inverted `bead implements plan` as its negative. Recommended
endpoint kinds are guidance surfaced in help and completions, not a validation gate —
narrowing what `link add` accepts is a separate decision with its own
backward-compatibility cost.

Add `sase artifact link relation list` and `sase artifact link relation show <slug>`
reading that registry, and feed the same data to shell completions for the `relation`
argument. Per the research report's §8 this is deliberately **not** added to core
memory: the registry is already in Tier 1 at 893 characters and produced nothing, and an
on-demand command costs zero tokens on turns that do not need it.

Give `derived` a projection class. `_CURATED_ORIGINS` is `{"manual", "migrated"}` and
`_AUTOMATIC_ORIGINS` is `{"prompt_ref", "read"}`, so a `derived` row renders in neither
managed block. A host-derived semantic edge is a curated assertion about meaning, not an
access record: add `derived` to the curated class so it renders in `## Links`, and make
`_curated_peer_keys` and `_rebuild_existing_projections` in `link_health.py` agree, or
doctor will report every derived row as a stale table. Keep the origin distinguishable
in `link list` output and in the row itself so provenance stays explainable.

**Acceptance.** `sase artifact link relation show implements` prints direction, both
examples, and recommended kinds. A row with `origin: derived` renders in a document's
`## Links` block and `sase artifact doctor` reports no stale table for it.

## One derivation module behind one flag

**Files.** New `src/sase/artifact_links/derive/` package, `src/sase/flags/` registry
entry, tests. No call sites yet.

Create the flag first with `sase flag new artifact_link_derivation -k beta`, authoring
`--when-enabled`, `--when-disabled`, and `--remove-when`. It is epic scaffolding: it
exists so a landed intermediate phase cannot half-populate the durable graph, and
`suggest-and-close` deletes the Off branch and closes the flag bead rather than waiting
for its thresholds.

The module is Textual-free and I/O-scoped to reading artifacts it is given. Its public
surface is a pure-ish function per rule plus one entry point that returns candidate rows
for a set of artifact refs; it writes nothing itself, so every call site controls its
own persistence and its own flag check.

Two rules ship in this phase, in the report's value order.

**Research-swarm lineage → `derives-from` (≈116 edges).** For
`research:<dir>/<name>.md`, when `<dir>/<name>__a.md` and/or `<dir>/<name>__b.md` exist
as siblings, emit `research:<lead> derives-from research:<source>` with
`origin: derived`. Measured on this tree: 58 `__a` files, 58 `__b` files, 58
consolidations with the lead present, zero orphans — the shape is rigid because
`#research_swarm` creates it. One consolidation currently carries the edge. Derive from
the on-disk shape rather than the rename commit, since `rename-following` already
guarantees the paths are current.

**Plan `bead_id:` frontmatter → `implements` (602 edges).** For a plan whose frontmatter
carries `bead_id: <id>`, emit `plan:<relpath> implements bead:<id>` with
`origin: derived` and a description naming the frontmatter as the source. Skip a bead id
that does not resolve in the bead store rather than writing a dangling row.

Explicitly **not** derived in this phase, per the research report: nothing is promoted
from a `read` to a `derives-from`, and the plan header's `AGENTS` and `COMMITS` sections
stay out (~2,300 edges with no consumer would bury the deliberate ones).

**Acceptance.** Unit tests over fixture trees for both rules including the skip cases;
with the flag off the module is never called by anything, which is trivially true here
because no call site exists yet.

## Derive at creation, on sidecar commit, and in the hourly sweep

**Files.** `src/sase/main/plan_propose_handler.py`, `src/sase/artifact_cli/create.py`,
the sidecar commit path in `src/sase/sdd/`, new
`src/sase/scripts/sase_chop_artifact_link_backfill.py`, `src/sase/default_config.yml`.

Three call sites, one module, zero agent instruction.

**At the lifecycle boundary.** `sase plan propose` and `sase artifact create` know they
just created a linkable artifact and hold the context to describe it. Derive there and
persist through the normal store path. This is also the only place a plan handoff can be
caught at all: proposing a plan terminates the runner mechanically, so finalizers never
run for that turn — which is the concrete reason the research report moved this decision
off `/sase_final` and onto the boundary.

**On sidecar commit.** For an artifact that lands another way — a chop, a repair pass, a
direct sidecar commit — run the same derivation from the sidecar commit path, excluding
the `artifact_links` cause so the projection commit does not re-trigger it. See
"Deviations" above for why this is a host step rather than a `file_hooks:` entry.

**In the hourly sweep.** Add `sase_chop_artifact_link_backfill` to the `housekeeping`
lumberjack bucket in `default_config.yml` (interval 3600, documented for "durable
maintenance that may scan substantial local state"). It owns four jobs, each bounded and
each reporting what it did: the retroactive derivation sweep over artifacts created
before this epic; the `read-outbox` drain for agents that have since published; the
cross-workspace `reconcile_aggregate()` sweep from `index-durability`; and the
dangling-ref repair from `rename-following`. It must be resumable and must log what it
skipped — a silently truncated sweep reads as "covered everything" when it did not.

Run the retroactive sweep once with the flag on and record the resulting row counts in
the phase bead so the epic's projected total can be checked against reality.

**Acceptance.** After the sweep, `sase artifact link list --origin derived --limit 0`
returns on the order of 700 rows, `sase artifact doctor`'s durable-versus-index delta is
zero, and a second sweep is a no-op.

## The citation channel stops starving

**Files.** `src/sase/sdd/plan_header_block.py` consumers,
`src/sase/agents_sync/prompt_archive/`, the derivation module, tests.

4,604 archived prompts contain 12 `@<kind>:` refs. The same prompts make 437 prose
mentions of plan and research paths across 219 prompts. Humans and launching agents cite
artifacts constantly; they type the path. The `cites` writer is correct and idle.

**`agent cites plan` from the header block (≈3,066 available).** A plan carries a
`PROMPT:` header pointing at its archived prompt, and the archived prompt carries its
own `AGENTS:` header listing the agents that consumed it — both are sections of the same
Rust-owned, machine-parsed header contract, not prose. Walk plan → `PROMPT:` → prompt →
`AGENTS:` and emit `agent:<name> cites plan:<relpath>` with `origin: derived`. Owner
decision 3 unblocks this: the Agent pane gives `agent:` refs a destination. Owner
decision 5 gates it: emit only for an agent whose ref resolves, since an agent with no
commits was never published.

Do **not** use the plan's own `AGENTS`/`COMMITS` sections for this; those stay held.

**Prose paths → `cites` (≈437 available).** On the prompt publication path, resolve
`(repos/)?(plans|research)/20\d{4}/<file>.md` occurrences against the sidecar and emit
`cites` rows for exact path matches only. No fuzzy title matching, no partial-stem
matching: a wrong `cites` row is worse than a missing one because the graph's only real
asset is trust. Mark the origin so a resolved prose citation stays distinguishable from
a real `@ref` citation.

**Read-log candidates (owner decision 4: every read).**
`~/.sase/projects/<key>/artifact_reads.jsonl` records, per agent, every artifact read
with the agent's own stated reason — the highest-quality inflow available anywhere in
the system, currently used for nothing. Emit these as **ranked candidates** carrying the
recorded reason as the candidate description, scoped to every read rather than a
`plan:`/`research:` subset. They are not rows: per PROV and OpenLineage, an activity
_using_ an entity is not the output being a _derivation_ of it, and promoting every read
would produce a dense, misleading graph. The candidate surface is
`sase artifact link suggest` in `suggest-and-close`; this phase produces the candidates
and stores nothing.

**Acceptance.** A test walks a fixture plan through its prompt to two agents and asserts
two `cites` rows for published agents and none for an unpublished one; a prose-citation
test asserts an exact path match produces a row and a near-miss produces none.

## Run the RELATED: note backfill

**Files.** `src/sase/artifact_cli/link_migrate.py`,
`src/sase/sdd/artifact_link_migrate_notes.py`.

`sase artifact link migrate-notes --apply` is fully wired —
`apply_related_note_migration` plus `bead_store_mutation.commit` — and has never been
run. Its own `--apply` help still says the mutation path "lands with the beads phase",
which is stale and is why nobody has trusted it.

Measured on this tree today: 4,306 beads scanned, 303 `RELATED:` notes, **269**
auto-convertible, 34 needing manual review.

Verify the apply path against a scratch bead tree first — a copy of the beads sidecar,
not the real one — and confirm it writes `related` events and `MIGRATED:` notes and is
idempotent on a second run. Correct the help text. Then run it for real, and record the
converted count and the 34-item worklist in the phase bead so the manual remainder is
visible rather than lost.

**Acceptance.** 269 new `related` rows exist and are visible from both endpoints (which
requires `bead-endpoints`, hence the dependency); a second `--apply` converts nothing;
the 34-item worklist is recorded.

## Finish the `links:` frontmatter inlet as an authoring path

**Files.** `../sase-core` inlet module (already built), `src/sase/sdd/` inlet consumer,
`src/sase/main/plan_propose_handler.py`, `src/sase/sdd/_artifact_link_refresh.py`.

The Rust `links:` frontmatter parser exists and is correct
(`crates/sase_core/src/artifact_link/inlet.rs`, tolerant, only ingests
`{ref, relation, description}` lists) and has **no production caller** — it appears only
in `tools/validate_sase_core_rs` and its test. Give it one.

An author declaring links in plan frontmatter is the one authoring affordance that fits
the workflow: it is written while the plan is being written, by someone who already
knows what the plan derives from. On archive, resolve the document's own canonical
source ref, validate each entry's relation and direction against the registry, upsert
the durable row with `origin: manual` — an author's assertion, not a host fact, so
`derived` stays reserved — then **remove the consumed inlet from the frontmatter** and
refresh the projection.

Removing it is the load-bearing part. An inlet that survives consumption is a second
editable representation of a row that the store also owns, and the two will diverge. A
malformed or unresolvable entry is reported and left in place rather than silently
dropped.

**Acceptance.** A plan with a valid `links:` block proposes, the rows appear in the
store and the rendered `## Links` block, and the frontmatter key is gone; a plan with an
unknown relation reports the error and keeps its frontmatter unchanged.

## Make every audited read a discovery moment

**Files.** `src/sase/artifact_cli/read.py`, prompt expansion in `src/sase/`, tests.

The sanctioned read path currently shows fewer links than `cat`: `_strip_managed_text`
applies `links_block_strip`, then `referenced_by_block_strip`, then drops frontmatter.
The two researchers disagreed about whether that is a feature or the highest-leverage
defect on the list. Both are right, and one change satisfies both.

**Keep stripping the multi-line managed blocks** — they would genuinely bloat context —
**and print a single-line typed footer** built from
`store.load_artifact_rows(canonical)`:

```
Links: implements bead:sase-r8 · derives-from research:202608/artifact_link_graph.md
```

Roughly 20 tokens. Prefer semantic relations over observational ones, cap the line, and
report the overflow count rather than wrapping. This is the smallest change in the epic
and probably the highest leverage, because it turns every audited read into a discovery
moment without touching any agent's instructions.

**Warn on a superseded artifact.** When the artifact being read is the target of a
`supersedes` row, print a one-line warning naming the superseding artifact. An agent
should not silently take direction from a superseded plan.

**Expand exactly one hop in launch context.** When a prompt cites `@plan:x`, include
`x`'s one-hop typed neighborhood, preferring `implements`, `derives-from`, and
`supersedes` and filtering observational edges. **Never expand transitively by
default**: a one-hop typed neighborhood is predictable and small; unconstrained
traversal is a context explosion.

**Acceptance.** Reading an artifact with links prints the footer and reading one without
prints nothing extra; reading a superseded artifact prints the warning; a prompt citing
a linked plan expands one hop and a two-hop neighbor does not appear.

## ACE stops flattening the relation type

**Files.** `src/sase/ace/tui/relations/artifact_links.py`,
`src/sase/core/artifact_relations.py`, the relation panel and rendering under
`src/sase/ace/tui/widgets/artifacts/`.

`_emit_link_edge` builds every `RelationEdge` from `decl.kind` / `decl.name` /
`decl.label` — the pane declaration's generic `links` / `linked_by` — and discards
`row["relation"]`, `row["description"]`, `row["origin"]`, and `row["uses"]`. The
relation slug survives only inside `_edge_key` for deduplication. The Markdown
projection preserves more semantic information than the interactive UI.

Carry the row's facts onto the edge: relation slug, the registry's directional label for
the direction actually being rendered (`implements` outbound, `implemented-by` inbound),
description, origin, and use count. Render the typed label instead of the generic one,
group the relation rail by relation, and show origin so a `derived` row is visibly a
host fact rather than a human assertion. Keep `_edge_key`'s undirected handling for
`related` unchanged.

Add a typed link action: link the marked artifact to the current one, choosing a
relation from the registry, with a **required** reason. Every deliberate edge in the
store today was typed at a shell; this is the first in-app authoring affordance, and
requiring the reason is what keeps it from becoming a rubber-stamp button.

This pane will carry the graph's largest edge population, so leave the seams the owner's
next project needs: every row exposes a stable `ArtifactEntryTarget`, a canonical
artifact reference, source project, available relations, open and copy locators, and
mutation capability facts kept separate from relation facts. Keep this generic — a later
"link this artifact to …" host action on any tab, and chops pointing at canonical agent
targets, should not need to import a per-pane widget.

**Acceptance.** The relation rail shows `implements` / `implemented-by` with the row's
description and origin rather than `links` / `linked_by`; the typed link action refuses
to write without a reason; existing relation snapshot tests are updated deliberately,
not force-accepted.

## `relation:`, `linked:`, and `artifact:` on the Agent pane

**Files.** `src/sase/ace/query_profile/profiles/_agents.py`,
`src/sase/agents/catalog/_query.py`, `src/sase/ace/tui/widgets/artifacts/query_rows.py`,
`src/sase/ace/tui/widgets/artifacts/agents_data.py`, `src/sase/agents/cli_search.py`.

The agent catalog pane research recommended shipping link _navigation_ and holding link
_filtering_ "until the derivation report's Phase 0 lands", because a `linked:false`
filter over an index that silently drops rows returns confidently wrong answers.
`index-durability` is that Phase 0, so this phase collects the deferred work.

**Do not start until `sase-tj.10` has landed** — see the sequencing note above.

Three fields on the agents profile:

- `relation:<slug>` — enum over the registry slugs; true when the row participates in at
  least one link row with that relation, in either direction.
- `artifact:<ref>` — exact-match string over the canonical refs the row is linked to, in
  either direction.
- `linked:<bool>` — true when the row has at least one link row.

The join is a facet map from agent name to `(relations, artifact_refs, count)` built
from `ArtifactLinksSnapshot`, normalizing the stored ref exactly the way
`_known_target_for_ref` already does — 44 of 45 live refs are bare local names and one
is owner-qualified, so match either through
`current_owner_agent_name_lookup_candidates`, and do not rewrite stored rows.

Thread the facets through as an optional argument so `sase.agents.catalog` stays
Textual-free and its query entry function keeps working without them.
`sase.ace.tui.relations.artifact_links` is currently Textual-free (verified), so
`sase agent search` can import the snapshot loader directly; if a Textual import ever
creeps into that module, extract the loader rather than duplicating it. Both frontends
must produce identical results for the same query — that parity is the point of the
Textual-free row model.

Add the fields to the filter bar's completion and value hints (they come from the
profile for free), and add a `linked:true` example to the pane's help.

**Acceptance.** `sase agent search 'relation:read linked:true'` and the same query typed
into the pane return the same rows; `artifact:plan:202608/<file>.md` returns exactly the
agents linked to it; `linked:false` returns agents with no rows and its count plus
`linked:true`'s equals the unfiltered total.

## The judgment tier, the flag removal, and the two workflow lines

**Files.** New `src/sase/artifact_cli/link_suggest.py`, `src/sase/artifact_cli/link.py`,
`src/sase/xprompts/skills/sase_plan.md`, the research swarm xprompt lead step,
`src/sase/artifact_cli/link_health.py`, the flag registry and its bead.

**`sase artifact link suggest`.** Proposes candidates with their evidence and **writes
nothing**. Evidence is restricted to hard signals only: shared bead, shared epic,
overlapping read sets (the read-log candidates from `citations`, carrying each agent's
own recorded reason), and filename lineage. No semantic-similarity suggester — it
generates plausible `related` edges faster than any human will audit them, and unaudited
`related` edges destroy the graph's only real asset.

Candidates stay **ephemeral until accepted**. Do not persist a speculative row and later
"confirm" it: the upsert key is `(source, relation, target)` and an upsert retains the
row's original origin, creator, and creation time while updating only the description,
so a persisted suggestion could never record its own promotion to a human-declared
assertion. Surface the output as a batched digest through a notification gate rather
than as work spread across hundreds of runs.

**Remove the flag.** Delete the Off branch, make derivation unconditional, remove the
registry entry, and close the flag bead in this change.

**Add the only two agent-facing lines in this epic.** One in `/sase_plan`, one in the
research swarm lead step, both stating a fact rather than adding an obligation:

> SASE derives your plan's links from the artifacts you read this turn; use
> `sase artifact read` for context you actually used.

Nothing is added to `AGENTS.md`. Nothing is added to `/sase_final`. The average agent's
context does not change by one token.

**Coverage as a report.** Add derived-coverage counters to `sase artifact doctor` —
linked fraction per population, rows by origin, rows by relation — as output only. The
denominator is "artifacts for which a workflow had a credible candidate", not all
Markdown, and it never becomes a gate or a quota.

**Acceptance.** `sase artifact link suggest` prints candidates with evidence and the
store is byte-identical afterwards; `tools/check_feature_flags` passes with the flag
gone; both xprompt lines render in the generated skills; doctor prints the coverage
report and exits zero regardless of coverage.

---

## What this epic is expected to produce

| Source                                                               |       Edges |
| -------------------------------------------------------------------- | ----------: |
| Aggregate today                                                      |         112 |
| Rows recovered by the non-lossy rebuild and the reconciliation sweep |         +50 |
| Read rows retained instead of lost                                   |         +62 |
| `plan implements bead`                                               |        +602 |
| Research-swarm `derives-from`                                        |        +116 |
| `RELATED:` migration                                                 |        +269 |
| Prompt-header and prose-path `cites`                                 | +437 and up |
| **Total**                                                            | **~1,600+** |

Agent instruction added: two lines, in two contexts read only by agents that produce a
linkable artifact.

## What this epic deliberately does not do

- No linking obligation in `AGENTS.md` and no link step in `/sase_final`. The experiment
  already ran and returned 0.7%.
- No promotion of `read` to `derives-from`.
- No `AGENTS` / `COMMITS` plan-header edges. Reopen when a consumer exists that would
  not be buried by ~2,300 rows.
- No new relation slugs.
- No fuzzy or semantic suggester.
- No coverage gate.
