---
tier: epic
title: Project artifact links from agents to the beads they wait on
goal: An agent launched with `%wait(bead=<id>)` gets a durable, visible artifact link
  to that bead, just as `%id(..., bead=<id>)` already yields `agent:<name> implements
  bead:<id>`.
phases:
- id: core-relation
  title: Awaits relation in the sase-core registry
  depends_on: []
  description: 'core-relation: in the linked sase-core repo, add the projection-only
    `awaits` / `awaited-by` builtin relation to the artifact-link registry, with Rust
    tests.'
  size: small
- id: wait-projection
  title: Publish bead waits and project awaits links
  description: 'wait-projection: publish `wait_for_beads` as portable agent metadata,
    add an `agent-wait-bead` projection rule emitting `agent:<name> awaits bead:<id>`,
    bump the sase-core pin, and update docs, TUI sigils, snapshots, and tests.'
  size: medium
  depends_on:
  - core-relation
proposed_by: bbugyi200.athena.0qk
create_time: 2026-09-24 08:23:06
status: wip
bead_id: sase-17o
---

- **PROMPT:** [prompts/202609/agent_wait_bead_links.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/agent_wait_bead_links.md)
- **BEAD:** [sase-17o](https://github.com/sase-org/sase--beads/blob/main/pages/sase-17o/README.md)

# Project artifact links from agents to the beads they wait on

## Findings (why this plan exists)

The two bead-bearing launch directives behave differently today:

- **`%id(..., bead=<id>)` — already linked, no change needed.** The launch writes
  `bead_id` into `agent_meta.json` (`build_agent_meta` in
  `src/sase/axe/run_agent_directive_metadata.py`). `bead_id` is in the portable
  published-metadata allowlist `V2_METADATA_FIELDS`
  (`src/sase/agents_sync/v2_validation.py`), so it reaches the agents sidecar's
  `agents/<global_name>/meta.json`. The `agent-bead` projection rule
  (`src/sase/artifact_links/projection/_agent_bead.py`) then projects
  `agent:<global_name> implements bead:<id>` for every non-empty `bead_id`,
  `epic_bead_id`, and `phase_bead_id`. Verified live:
  `sase artifact link list agent:bbugyi200.athena.sase-17m.3.1.2` shows
  `implements bead:sase-17m.3.1.2` (origin `projected`).
- **`%wait(bead=<id>)` — not linked at all.** The launch writes `wait_for_beads` into
  `agent_meta.json` (same builder; TUI wait edits patch the same key via
  `wait_meta_patch_for_token` in
  `src/sase/ace/tui/actions/agents/_directive_persistence.py`). But `wait_for_beads` is
  **not** in `V2_METADATA_FIELDS`, so it is dropped by `portable_metadata`
  (`src/sase/agents_sync/inventory_io.py`) and never published. No projection rule reads
  it, and the closed relation registry has no relation for "waited on". Verified live:
  agent `sase-17m.3--plan` has `wait_for_beads: [sase-17m.1, sase-17m.2]` locally, yet
  neither bead has any link back to it.

Why not reuse an existing relation: `depends-on`/`blocks` are reserved and refused (bead
scheduling belongs to `sase bead dep`); `related` is a CLI-written, undirected, manual
relation, so projecting it would blur provenance and lose direction; `implements` is
wrong because a waiting agent does not implement the bead it waits on (e.g. an epic land
agent waits on every phase bead but implements only the epic).

## Design

- New builtin relation **`awaits`**, inverse **`awaited-by`**, directed,
  `written_by: projection` (read-only like `produced-by` and `launched`;
  `link add`/`link rm` and the TUI remove action must not write or delete it). Source is
  the waiting agent, target is the bead: `agent:sase-tj.land awaits bead:sase-tj.2`.
  Recommended source kinds `["agent"]`, target kinds `["bead"]`.
- Publish `wait_for_beads` (a list of bead-id strings) as portable agent metadata,
  exactly like `bead_id`, so any machine that syncs the agents sidecar computes the same
  edges.
- A new projection rule `agent-wait-bead` reads published `meta.json` `wait_for_beads`
  and emits one `awaits` row per distinct, non-empty, whitespace-free bead id.
- Like the existing `implements` projection, rows exist only once the agent is published
  to the agents sidecar (an `agent:<global_name>` identity only resolves after
  publication). Links for still-running agents are out of scope.

### Compatibility note

`meta.json` and hood-snapshot readers reject unknown metadata keys
(`run_metadata_from_json` in `src/sase/agents_sync/v2_run_io.py` and the snapshot
validator in `src/sase/agents_sync/v2_snapshot_io.py`). An older SASE binary on another
tailnet machine that reads a newly published `meta.json` carrying `wait_for_beads` would
fail validation for that run. This follows the same precedent as the recent
`agent_session` metadata key (commit 29ca9ee46), which added a new portable key without
a flag. All SASE machines must upgrade together. Call this out in the stitch message. Do
not add a feature flag: this is additive metadata, not user-facing behavior that needs
an Off branch.

## Awaits relation in the sase-core registry

Work in the linked `sase-core` checkout (open it with `sase repo open sase-core`).

1. In `crates/sase_core/src/artifact_link/relation.rs`, append an
   `ArtifactRelationWire::builtin("awaits", "awaited-by", true, "projection", ...)`
   entry after `launched`, with:
   - direction note: "The waiting agent is the source; the bead it waited on is the
     target."
   - positive example: `agent:sase-tj.land awaits bead:sase-tj.2`
   - negative example: `bead:sase-tj.2 awaits agent:sase-tj.land`
   - recommended source kinds `["agent"]`, target kinds `["bead"]`.
2. Update the `builtins_cover_v1_table` test's expected slug list and add assertions for
   `awaits` lookup (directed, inverse `awaited-by`) and
   `relation_label_from_perspective("awaits", false) == "awaited-by"`.
3. Search sase-core for other hard-coded builtin relation lists or counts (Python
   binding tests in `crates/sase_core_py`, wire fixtures, doctor or reducer tests that
   enumerate relations, anything that treats `written_by == "projection"` specially) and
   extend them so `awaits` is handled like `produced-by`/`launched`. In particular,
   confirm that the CLI-writable check (only `cli` relations are manually writable)
   rejects `awaits` for manual `link add`.
4. Run the repo's standard Rust checks (per its `AGENTS.md`).

## Publish bead waits and project awaits links

Depends on `core-relation`. Read `sase/memory/lint_and_test.md` (through
`sase memory read`) before finishing.

1. **Pin.** Move `sase-core-revision.txt` to the `core-relation` sase-core commit (see
   `docs/rust_backend.md`), so the binding knows `awaits`.
2. **Python relation fallback.** Add an `awaits` entry to `_PROJECTION_RELATIONS` in
   `src/sase/sdd/_artifact_link_store_support.py`, mirroring the Rust wire exactly (it
   covers bindings older than the pin, like the existing `produced-by`/`launched`
   entries).
3. **Publish the metadata.** Add `"wait_for_beads"` to `V2_METADATA_FIELDS` in
   `src/sase/agents_sync/v2_validation.py`. Confirm `portable_metadata` publishes the
   list unchanged for live runs (`inventory_sources.py`, `_portable_metadata(meta)`).
   For dismissed-run bundles (`_portable_metadata(raw)` in the same file), check which
   key the dismissed bundle stores bead waits under. If it is not `wait_for_beads` (for
   example a TUI-side `waiting_for_beads`), normalize it into `wait_for_beads` before
   publication so dismissed agents also get their links. Normalize the published value
   to a deduplicated, order-preserving list of non-empty strings, and omit the key when
   that list is empty.
4. **Projection rule.** Add `src/sase/artifact_links/projection/_agent_wait_bead.py`
   (rule id `agent-wait-bead`), modeled on `_agent_bead.py`: the same scandir plus
   per-agent stat-signature cache via `read_rule_cache`/`write_rule_cache`, and the same
   best-effort handling of unparseable `meta.json`. For each published agent, emit
   `agent:<global_name> awaits bead:<id>` per distinct bead id in `wait_for_beads`, with
   a description such as "published meta.json's `wait_for_beads` names bead <id>". Wire
   it into `project_link_rows` in `src/sase/artifact_links/projection/_entry.py`. Prefer
   factoring the shared scan/cache loop out of `_agent_bead.py` rather than copying it.
   If you do, keep the two rules' cache keys and rule ids distinct so existing caches
   stay valid.
5. **Consumers of relation lists.** Add an `"awaits": "wait"` sigil to
   `_RELATION_SIGILS` in `src/sase/ace/tui/widgets/link_rail.py`. Grep for other
   hard-coded relation sets (link panel grouping, `origin`/`projected` read-only
   handling, bead and agent page rendering, doctor drift checks, the
   `artifact_link_backfill` job's projection recompute) and confirm `awaits` rows render
   and stay read-only like `produced-by`/`launched`. Do not add `awaits` to the
   prompt-expansion `(linked: …)` semantic neighbors (that set is `implements`,
   `derives-from`, `supersedes`), because a wait is scheduling context, not semantic
   lineage.
6. **Generated registry snapshot and memory.** The relation list in
   `sase/memory/sase_artifacts.md` and `sase/artifact_relations.json` is generated by
   `sase memory init` from `assembled_artifact_relations()`. Regenerate them, and use
   the `/sase_memory_write` skill before touching the memory note. Update the expected
   slug list in `tests/main/test_init_memory_task_types_snapshot.py`.
7. **Docs.** In `docs/artifact_links.md`, add the `awaits` / `awaited-by` row to the
   Relation Registry table ("projected from a published agent's `wait_for_beads`"). Add
   a bullet under "Projected relationships":
   `a published agent's wait_for_beads (from %wait(bead=<id>)) projects agent:<name> awaits bead:<id>`.
   Mention `awaits` in the Direction-matters sentence. Add a short note to the `%wait`
   directive docs (`docs/xprompt.md` or `docs/prompt.md`, wherever `%wait(bead=...)` is
   documented) that published agents are linked to the beads they waited on, and that
   `%id(..., bead=...)` yields `implements`.
8. **Tests.**
   - Projection rule unit tests, modeled on the existing agent-bead projection tests:
     emits one `awaits` row per distinct bead id; ignores empty, whitespace, and
     non-string entries; no rows when the key is absent; the cache hit path reuses rows
     and a changed `meta.json` recomputes; an unparseable `meta.json` yields no rows.
   - Publication tests: `wait_for_beads` survives `portable_metadata` and round-trips
     through `run_metadata_from_json` and the hood-snapshot validator without an
     "unsupported fields" error. Cover the dismissed-bundle path from step 3.
   - `project_link_rows` integration: an agent with both `bead_id` and `wait_for_beads`
     yields both the `implements` and the `awaits` rows.
   - Store and CLI: `sase artifact link add ... awaits ...` is refused as
     non-CLI-writable, and projected `awaits` rows cannot be removed with `link rm`
     (mirror the existing `produced-by`/`launched` tests in
     `tests/sdd/test_artifact_link_store_projected.py` and
     `tests/main/test_artifact_cli_link.py`).
9. **Verify end to end** after `just check`: with the rebuilt aggregate
   (`sase artifact doctor --fix` or the backfill path), confirm a published agent with
   `wait_for_beads` shows `awaits bead:<id>` in `sase artifact link list agent:<name>`,
   and the bead shows `awaited-by agent:<name>`. Agents published before this change do
   not carry the key, so historical waits are not backfilled. That is acceptable and
   should be stated in the stitch message.

## Out of scope

- Agent-to-agent waits (`%wait:<agent>`) are published as hood-snapshot `wait`
  relationships but also produce no artifact link. That is a candidate follow-up, and
  `awaits` is deliberately named so it could later accept `agent` targets.
- Links for unpublished (still-running) agents.
- Changing the existing `implements` projection for `epic_bead_id`.
