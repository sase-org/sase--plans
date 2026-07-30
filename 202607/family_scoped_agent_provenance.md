---
tier: epic
title: Anchor agent commit provenance on the agent lane instead of the family member
goal: 'Commits made by an agent family member are published, tagged, and linked as
  their family (`foo`, not `foo--code`), solo agents keep their current behavior exactly,
  and no commit provenance is lost anywhere that previously relied on the member-name
  `SASE_AGENT=` tag.

  '
phases:
- id: lanes
  title: Shared agent-lane vocabulary
  depends_on: []
  size: small
  description: 'lanes: add the Python lane projection (lane name, lane kind, lane
    page path) composed from existing Rust core primitives, so every later phase resolves
    member -> lane identically.'
- id: tag
  title: Lane-scoped SASE_AGENT commit tag
  depends_on:
  - lanes
  size: small
  description: 'tag: make the runtime commit-footer tag carry the lane label and the
    lane page destination, dropping the member anchor, for real commits, PRs, and
    SDD auto-commits.'
- id: publish
  title: Lane-anchored sidecar publication requests
  depends_on:
  - lanes
  size: small
  description: 'publish: record the lane as the publication identity in the outbox
    and make hood publication accept a lane name deliberately rather than by accidental
    fallback.'
- id: snapshot
  title: Family containers carry their lane commits
  depends_on:
  - lanes
  size: medium
  description: 'snapshot: extend the v2 hood snapshot so family containers own lane-attributed
    commits, keep the strict decoder and import path in agreement, and render them
    on the family page.'
- id: inventory
  title: Lane-keyed commit history in the sidecar inventory
  depends_on:
  - tag
  - snapshot
  size: medium
  description: 'inventory: read lane-tagged commit history, keep solo attribution
    unchanged, route family-lane commits to the family container, stop fabricating
    phantom runs, and keep import evidence matching.'
- id: assoc
  title: Lane-based plan and bead agent associations
  depends_on:
  - tag
  size: medium
  description: 'assoc: make plan-header and bead-page AGENTS rows label the lane once,
    resolve their link from the artifact member or the footer''s recorded destination,
    and stop listing a member and its lane as two agents.'
- id: consumers
  title: Remaining SASE_AGENT tag readers and back-compat
  depends_on:
  - tag
  size: small
  description: 'consumers: make image-attachment scanning, revert discovery, and the
    PR body footer lane-aware, and lock in that every reader still accepts legacy
    member-name tags.'
- id: docs
  title: Documentation refresh
  depends_on:
  - tag
  - snapshot
  - inventory
  - assoc
  - consumers
  size: small
  description: 'docs: update the commit-workflow, agents-sidecar, and SDD docs to
    describe lane-scoped provenance and the family page as the durable home of a family''s
    commits.'
create_time: 2026-07-30 10:32:31
status: wip
---

# Plan: Anchor agent commit provenance on the agent lane instead of the family member

## Background

Every SASE-authored commit gets a runtime provenance tag. `_resolve_runtime_commit_tags()`
(`src/sase/workflows/commit/runtime_tags.py:50`) resolves the _concrete_ agent that made the commit via
`resolve_local_agent_name()` — which deliberately prefers the member name recorded in
`SASE_ARTIFACTS_DIR/agent_meta.json` over the lane-valued `SASE_AGENT_NAME` env var (`runtime_tags.py:63-81`) — and
hands it to `resolve_agent_commit_tag()` (`src/sase/agents_sync/links.py:23`).

That produces two things today, for a member named `pc--code`:

```
SASE_AGENT=[bbugyi200.athena.pc--code][2]

[2]: https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.pc.md#member-code
```

So the _destination page_ is already the family page; it is the **label** and every downstream identity derived from it
that still names the member. The same member name is also what `CommitWorkflow` stores as `publication_agent`
(`src/sase/workflows/commit/workflow.py:206`) and what `publish_committed_agent_hood()` records in the publication
outbox (`src/sase/agents_sync/commit_publication.py:109-119`).

Solo agents already behave the way we want: `resolve_agent_commit_tag("sase-b5.land")` yields the label
`bbugyi200.athena.sase-b5.land` linked to `agents/<global>/README.md`.

## Goal and non-goals

**Goal.** For a family member, the published identity, the `SASE_AGENT=` label, and the link destination all become the
**agent lane**: `foo` for a member of family `foo`, and the agent's own name for a solo agent. In SASE's existing
vocabulary this is precisely the _lane_ — the glossary already defines a lane as "either an agent family or a single
agent that does not belong to a family", and `src/sase/agents_sync/rendering_kinship.py` already renders sidecar
neighbor sections in lane terms.

**Explicit non-goals.**

- Solo agents change in no observable way. Every phase must carry a regression test proving the solo label, destination,
  publication identity, and attribution are byte-identical to today.
- No historical rewriting. Commits already tagged with a member name keep working; every reader must accept both
  spellings forever.
- `agent_link_target()` (`src/sase/core/agent_identity_facade.py:394`) keeps its current member-anchored behavior. It is
  still the right answer for `src/sase/agents_sync/rendering_index_pages.py:156` and
  `src/sase/agents_sync/rendering_kinship.py:136`, which link a _specific run_ from an index page and genuinely want
  `families/<family>.md#member-<role>`.
- No change to `../sase-core`. The design below is deliberately composed from primitives the Rust core already exposes
  (`parse_agent_family_name`, `agent_link_target`, `globalize_owned_agent_name`, `agent_local_hood`), so no wheel bump
  is needed. If an implementing phase concludes it genuinely needs a new naming primitive in Rust, stop and raise it
  rather than open-coding name semantics: that would require opening `sase-core` through `/sase_repo`, a release, and a
  pin bump of `sase-core-rs` in `pyproject.toml:46`.

## The one hard problem: `foo` is lexically ambiguous

A family container named `foo` and a solo agent named `foo` are indistinguishable as strings.
`parse_agent_family_name("foo")` returns `kind=solo`. So any consumer that receives only the lane label
`bbugyi200.athena.pc` cannot tell whether the page is `families/bbugyi200.athena.pc.md` or
`agents/bbugyi200.athena.pc/README.md`.

The plan resolves this without inventing a marker in the tag text, using three sources in precedence order:

1. **Full information at write time.** Everything that _writes_ a lane identity (the commit tag, the publication
   request) starts from the concrete member name, so it knows the kind for free. Nothing needs to be inferred at write
   time.
2. **The footer's own recorded destination.** `parse_commit_footer()` returns a `LinkedCommitTagValue` whose
   `.destination` already encodes `/families/` vs `/agents/`. Readers that re-derive a link from a commit should reuse
   that URL instead of re-resolving the name. This is history-proof: it works for commits made before and after this
   change.
3. **The local name registry.** `get_reserved_family_names()` (`src/sase/agent/names/_registry.py:88`) returns every
   name owned by a family container. This is authoritative for owner-local names and is available before the first
   publication of a brand-new family — which matters because `refresh_committed_plan_header()` runs _before_
   `publish_committed_agent_hood()` in `run_agent_publication_step()`
   (`src/sase/workflows/commit/workflow_publication.py:39-87`), so the family page may not exist on disk yet on a
   family's very first commit.

A fourth fallback — checking `target.sidecar_path / "families" / f"{global}.md"` on the local agents clone — is
available to both `src/sase/agents_sync/links.py` and `src/sase/sdd/hosted_links.py:243-272` and may be used as a last
resort for imported/foreign names. When nothing resolves, degrade to an unlinked label; never guess a URL. That is the
existing contract of both link resolvers.

---

## Phase `lanes`: Shared agent-lane vocabulary

Add one small module, `src/sase/agent_lanes.py`, that every later phase imports. It is a thin projection over existing
core primitives — it must not re-implement name parsing.

```python
@dataclass(frozen=True, slots=True)
class AgentLaneRef:
    local_name: str      # "pc"        (lane, not member)
    global_name: str     # "bbugyi200.athena.pc"
    is_family: bool      # True when the lane is a family container
    member_local_name: str | None   # "pc--code" when known, else None
```

Functions:

- `lane_ref_for_agent(name, identity) -> AgentLaneRef` — the write-time path. Normalizes with
  `normalize_owned_agent_name`, parses with `parse_agent_family_name`, and returns `local_name=parsed.family_name`,
  `is_family=(parsed.kind is AgentFamilyNameKind.MEMBER)`, `member_local_name=` the input when it is a member. For a
  solo this is the identity function on the name, with `is_family=False`.
- `lane_ref_for_lane_name(name, identity) -> AgentLaneRef` — the read-time path, for callers that only have a lane
  label. `is_family` comes from `get_reserved_family_names()` (guarded: registry access must never raise into a caller),
  falling back to `False`.
- `lane_page_path(ref, owner) -> str` — `f"families/{ref.global_name}.md"` when `ref.is_family`, else
  `agent_link_target(ref.local_name, owner).path`. When `ref.member_local_name` is known, prefer
  `agent_link_target(member).path` (which already returns the family page path) so the sidecar layout stays owned by one
  core function.
- `lane_name(name) -> str` — the bare string projection, for callers that only need the label.

Requirements:

- Pure and side-effect free apart from the guarded registry read; no git, no filesystem.
- Every function must be total for legacy spellings (`athena.sase-7r.land--code`) and must raise nothing on malformed
  input beyond what the core parser already raises — callers in this codebase are all best-effort boundaries.
- Add `tests/test_agent_lanes.py` covering: solo → itself/not-family; member → family/is-family; nested family
  (`foo.bar--code` → `foo.bar`); a lane name that the registry reports as a family; a lane name that is a real solo
  agent; and the legacy machine-qualified spelling.

## Phase `tag`: Lane-scoped `SASE_AGENT` commit tag

Change `resolve_agent_commit_tag()` (`src/sase/agents_sync/links.py:23`) so that, after resolving the hosted sidecar
exactly as it does now:

- the **label** is `lane_ref_for_agent(agent_name, snapshot).global_name`;
- the **destination** is built from `lane_page_path(...)` and **carries no fragment** — the `#member-<role>` anchor is
  dropped;
- every early return (`owner is None`, no project, target resolution failure, more/less than one target, primary-root
  mismatch, no `.git`, unresolvable branch, non-hosted remote) returns the **lane** global label instead of the member
  global label. These are the `return global_name` paths at `links.py:48,52,55,61,63,66,72`; each must now return the
  lane spelling.

Nothing else in the writer changes: `_resolve_runtime_commit_tags()` still resolves the concrete member first (that
information is required to compute the lane), `apply_runtime_commit_tags()`, `apply_commit_tags()`, and
`apply_auto_commit_tags_with_runtime()` are untouched, so SDD auto-commits pick up lane labels automatically.

Update the docstring at `runtime_tags.py:136-146` and the comment at `runtime_tags.py:76-79` to say that the member name
is resolved so the _lane_ can be derived.

Tests:

- `tests/agents_sync/test_links.py` — rewrite `test_family_member_commit_tag_links_to_stable_member_anchor` into a lane
  test asserting `LinkedCommitTagValue("alice.athena.foo.bar", ".../families/alice.athena.foo.bar.md")` with no
  fragment. Keep and extend the solo cases unchanged — they are the regression guard.
- Add an unhosted-sidecar member case asserting the bare lane label `alice.athena.foo.bar`.
- `tests/test_commit_runtime_tags.py` — assert the rendered footer for a member-run process.

## Phase `publish`: Lane-anchored sidecar publication requests

`publish_committed_agent_hood()` (`src/sase/agents_sync/commit_publication.py:74`) currently normalizes the committing
member and stores it as `local_agent`/`global_agent` on the `AgentPublicationOutboxItem`.

- Derive the lane with `lane_ref_for_agent()` and store the **lane** in `local_agent` and `global_agent`. `local_hood`
  is unchanged: `agent_local_hood("pc--code")` and `agent_local_hood("pc")` both yield `pc`, and for a nested family
  `foo.bar--code` and `foo.bar` both yield `foo`. **Publication scope therefore does not change** — it was already
  whole-hood. What changes is the identity recorded on the request, which flows into `logical_key`
  (`publication_outbox.py:48`) and the notification subject (`publication_outbox.py:330`).
- Keep `CommitCheckpoint.publication_agent` (`src/sase/workflows/commit/checkpoint.py:34`) holding the concrete member
  as written by `workflow.py:206`, so a `sase commit --resume` still knows exactly which run committed. The member→lane
  projection happens at the publication boundary.
- `publish_agent_hood()` (`src/sase/agents_sync/publication.py:68`) currently only tolerates a family name by accident:
  `any(run.local_name == local_name ...)` fails for a container name and falls through to the `hood_runs(hood)` check.
  Make this deliberate — accept a lane, document that a family lane never matches a run, and keep the
  `hood {hood!r} has no publishable runs` error for the genuinely empty case (that string is load-bearing:
  `_prepare_publications()` matches on it at `commit_publication.py:483-486` to mark a request terminal).

Tests: a member-run publication enqueues the family lane; the drained hood payload is unchanged versus a member-anchored
request; a solo publication is byte-identical to today.

## Phase `snapshot`: Family containers carry their lane commits

Today a commit's association to a run is reconstructed two ways: from the artifact's `commit_results.json` markers
(`src/sase/agents_sync/inventory.py:243-252`, written by `_upsert_commit_results_marker()` at
`src/sase/workflows/commit/commit_tracking.py:353`) and from `SASE_AGENT=`-tagged git history (`inventory.py:359-395`).
Once the tag names a lane, family-lane history can no longer name a member — so the family, not any one member, becomes
the durable owner of those commits. The v2 hood snapshot needs somewhere to put them.

Extend `V2ContainerRecord` (`src/sase/agents_sync/v2_models.py:135`) with `commits: tuple[CommitRecord, ...] = ()`,
serialized as a `commits` key in `to_json_dict()`.

Requirements:

- The strict decoder in `src/sase/agents_sync/v2_snapshot_io.py` uses `exact_object()` field sets, so `_containers()`
  must be updated to accept — and require, for self-consistency — the new key. Decide and document the
  `V2_SCHEMA_VERSION` (`v2_models.py:11`) question explicitly in the commit message: this is a payload shape other
  machines import, so either bump the version or make the key optional-on-read and always-present-on-write. Prefer
  optional-on-read/always-on-write so existing published shards stay importable without a migration.
- Commits are sorted by `(committed_at, sha)` and deduplicated by sha, matching `_published_run()`.
- Clan containers keep an empty tuple; only families accumulate lane commits.
- `_hood_file_set()` (`publication.py:576`) is unaffected — container commits live inside `snapshot.json`, whose digest
  changes naturally.
- Check parity in `src/sase/agents_sync/v2_import_package.py` (container reconstruction around line 423) and
  `src/sase/agents_sync/incoming_cache_metadata.py` so imports round-trip.
- `src/sase/agents_sync/rendering_family_page.py:100-116` currently builds `family_commits` purely from `run.commits`
  with a per-member role. Union in the container's own commits with role `—`, deduplicated by sha against the
  member-attributed rows (a member row always wins), keeping the existing `render_family_commits()` sort and
  `MAX_RENDERED_COMMITS` cap (`src/sase/agents_sync/rendering_commits.py`).

Tests: snapshot round-trip with and without container commits; digest stability when the field is empty; family page
rendering with a mix of member-attributed and lane-only commits.

## Phase `inventory`: Lane-keyed commit history in the sidecar inventory

`_historical_associations()` (`src/sase/agents_sync/inventory.py:359`) builds `{local_name: (CommitRecord, ...)}` by
reading `tags.get("AGENT")` from primary git history. After the `tag` phase that key is a lane, so:

- Key the history map by **lane** and record, per lane, whether the tag's footer destination indicates a family page
  (see "The one hard problem", source 2), falling back to the registry.
- **Solo lanes: unchanged.** The lane equals the run's `local_name`, so `history.get(local_name, ())` at
  `inventory.py:242` and `inventory.py:341` keeps working exactly as today. This is the single most important regression
  to hold.
- **Family lanes:** do not attach lane history to any member run. Member runs still get their exact commits from
  `commit_results.json` markers, which is the authoritative per-run source. Route the lane's commits to the family
  container built in `_build_containers()` (`src/sase/agents_sync/publication.py:344`), which already groups members by
  `parse_agent_family_name(run.local_name).family_name`. Threading requires passing the lane history from
  `ProjectHoodInventory` into `_build_hood_snapshot()`/`_build_containers()`; add the lane→commits map to
  `ProjectHoodInventory` (`src/sase/agents_sync/inventory_models.py`) rather than re-reading git.
- **Never fabricate a phantom run for a family lane.** `_add_commit_only_runs()` (`inventory.py:530`) exists so a commit
  whose artifact was cleaned still appears in the sidecar. For a family lane it would synthesize a _run_ named `pc`,
  producing a bogus `agents/bbugyi200.athena.pc/README.md` page next to the real `families/bbugyi200.athena.pc.md`. Skip
  family lanes here; their commits reach the sidecar through the container instead. Preserve the existing behavior for
  solo lanes verbatim.
- If a family lane has commits but the hood has no runs at all, still publish the family container with its commits
  rather than dropping the history; if that is not representable, emit an inventory diagnostic (the `diagnostics` list
  already exists for exactly this) — never drop silently.
- `_matching_primary_commit_evidence()` (`src/sase/agents_sync/v2_import_history.py:226-246`) proves an imported foreign
  run really committed locally by matching `globalize(normalize(raw_name))` against the expected run global name. A lane
  tag will never equal a member's global name, so compare **lanes on both sides**: project both the tag value and each
  expected run's `global_name` through the lane projection before comparing. Keep accepting an exact member-name match
  so legacy commits still count as evidence.

Tests: a member-tagged legacy commit still attaches to its run; a lane-tagged commit attaches to the family container
and not to a member; a lane-tagged commit for a solo behaves identically to today; `_add_commit_only_runs` produces a
run for a solo lane and nothing for a family lane; import evidence matches for both legacy and lane spellings.

## Phase `assoc`: Lane-based plan and bead agent associations

The plan-header `AGENTS` section and bead-page agent rows are built from two merged sources:

- git history tags — `src/sase/sdd/associations/_history.py:72` and `src/sase/bead_pages/associations/_history.py:123`,
  both of which currently discard the `LinkedCommitTagValue` destination and keep only the label;
- live artifact metadata — `src/sase/sdd/associations/_artifacts.py:59-94`, which contributes **member** names.

Without this phase those two sources disagree after the `tag` change: a plan would list both `pc` (from history) and
`pc--code` (from artifacts) as two separate agents, and `resolver.agent_url("bbugyi200.athena.pc")`
(`src/sase/sdd/associations/_build.py:100`, `src/sase/bead_pages/associations/_build.py:250`) would resolve to a
nonexistent `agents/bbugyi200.athena.pc/README.md`.

Changes:

- Normalize **both** sources to the lane label so each lane appears exactly once. Artifact-derived names go through
  `lane_ref_for_agent()`, which retains `member_local_name`.
- Carry a resolvable link hint alongside the label rather than re-deriving it from an ambiguous string. Preference order
  per lane: (1) `agent_link_target(member).path` when any source knew a concrete member — note this already yields the
  family page path; (2) the `LinkedCommitTagValue` destination recorded in the commit footer; (3) `lane_page_path()` via
  the registry; (4) `None` (unlinked label). Widening the internal `agents: set[str]` to a small record is expected;
  keep `HostedLinkResolver.agent_url()`'s public signature intact.
- Make `HostedLinkResolver.agent_url()` (`src/sase/sdd/hosted_links.py:122`) family-aware for the case where a bare lane
  name is all a caller has: resolve `is_family` via the registry, and fall back to checking `families/<global>.md` in
  the local agents clone reachable from `_resolve_agents_remote()` (`hosted_links.py:243`). Keep returning `None` rather
  than guessing.
- Bead-page agent rows carry a `commit_count` (`bead_pages/associations/_build.py:252`); after lane normalization this
  counts the lane's commits. Make sure `agent_commits` is re-keyed by lane so the count does not silently become zero.
- `agent_name_bead_id()` (`src/sase/bead_pages/associations/_agent_names.py:38`) derives an agent bead ID from a name;
  confirm lane spellings still derive the intended bead and add a test.

Tests: a plan touched by `pc--code` and `pc--plan` lists exactly one `pc` row linked to the family page; a plan touched
by a solo lists that solo unchanged; a legacy member-tagged commit with no live artifact still produces a working row
using the footer's recorded destination.

## Phase `consumers`: Remaining `SASE_AGENT` tag readers and back-compat

- `src/sase/axe/image_attachments.py:286-293` matches `tagged_agent == agent_name` or the prefixes `f"{agent_name}."` /
  `f"{agent_name}--"`. With a lane tag and a member `agent_name` the match now fails, so a family member would lose the
  images it attached via its own commits. Compare lanes: project both sides through `lane_name()` before matching, while
  keeping the existing exact and prefix matches so legacy member tags still match.
- `src/sase/ace/revert_agent_discovery.py:117-127` already tolerates a lane tag through `family_base`
  (`agent_family_base("pc")` is `None`, so `tag_base` falls back to `"pc"`, which equals the member's `family_base`).
  Verify with a test rather than changing it, and confirm `discover_bulk_commits()`'s dedupe-by-sha still behaves when a
  family parent and one of its members are both selected as revert targets.
- `build_pr_body()` (`src/sase/workflows/commit/pr_operations.py:110-149`) renders `**Agent:** [label](destination)`
  from the footer, so it follows the tag automatically. Add a test pinning the lane rendering, and check the
  `meta.get("name")` fallback at line 139 — that path still yields a member name and should be projected to a lane for
  consistency.
- `src/sase/vcs_log/_tag_style.py:112` only styles the chip; no change, but add a rendering test so the shorter lane
  label is covered.
- Add a dedicated back-compat test module asserting that a fixture commit message carrying the legacy
  `SASE_AGENT=[bbugyi200.athena.pc--code][2]` footer (with the `#member-code` anchor) is still read correctly by every
  reader touched in this epic.

## Phase `docs`: Documentation refresh

- `docs/commit_workflows.md:195-197` and `:211` — describe the tag as `SASE_AGENT=<username>.<machine>.<lane>`, state
  that a family member is tagged as its family, that solo agents are unchanged, and that the destination is the family
  page without a member anchor.
- `docs/agents_sidecar.md:89` — the sentence "Family member links use stable `member-<role>` anchors" is now true only
  for sidecar index/neighbor pages, not for commit footers; split the two cases.
- `docs/agents_sidecar.md:145` and `:187` — the synthesized-run and import-evidence paragraphs both describe
  member-level `SASE_AGENT` behavior that the `inventory` phase changes.
- `docs/sdd.md:207` — association sections are now re-derived at lane granularity.
- Optionally add a short "Lanes, families, and commit provenance" subsection to `docs/agent_families.md` tying the
  glossary's lane definition to what the commit tag means.

Do not edit `sase/memory/*.md`, `AGENTS.md`, or the generated provider instruction shims in any phase of this epic; the
user has not granted permission for memory edits here.

## Verification

Every phase runs `just install` then `just check` before finishing (the workspace is ephemeral, so the install step is
not optional). Beyond each phase's own tests, the epic is complete when:

1. A family member's commit in this repo produces `SASE_AGENT=[<user>.<machine>.<family>][n]` linked to
   `families/<global-family>.md` with no fragment.
2. A solo agent's commit footer, publication request, sidecar pages, plan-header rows, and bead-page rows are unchanged
   from before the epic.
3. `sase agents sync` reconciles the whole project without new diagnostics, and the resulting family page shows both
   member-attributed and lane-level commits.
4. A repository whose history contains only legacy member-name tags still publishes and renders exactly as it does
   today.
