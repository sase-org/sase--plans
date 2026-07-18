---
tier: epic
title: Split agent clans, sequential families, and tribes
goal: 'Parallel agent groups become named "clans" (never agents themselves, with reserved
  names, hood-scoped members, correct wall-clock runtimes, and a three-level Agents-tab
  tree), sequential plan-chain groups remain "families" (created by rename-on-attach,
  never single-member), and groups/tags become "tribes" (%tribe, @-prefixed). Epics
  and the research_swarm xprompt migrate to the new model.

  '
phases:
- id: core-runtime
  title: Clan wire model and runtime aggregation in sase-core
  depends_on: []
  description: '''Phase core-runtime'' section: add the agent_clan wire field with
    legacy mapping and implement interval-union clan runtime (excluding human-wait)
    in sase_core plus Python facade mirrors.'
- id: clan-launch
  title: '%clan directive, clan registry, and launch validation'
  depends_on:
  - core-runtime
  description: '''Phase clan-launch'' section: replace %f|%family with %c|%clan (no
    roles, no root agent), persist clan membership, reserve clan names against all
    agent names, and enforce the hood constraint with clear errors.'
- id: tribe
  title: '%tribe directive, CLI, and @-prefix display'
  depends_on: []
  description: '''Phase tribe'' section: replace %g|%group with %t|%tribe, rename
    the sase agent tag CLI to tribe, and display tribes with an @ prefix across the
    TUI.'
- id: family-sequential
  title: Sequential families via rename-on-attach
  depends_on:
  - clan-launch
  description: '''Phase family-sequential'' section: make %n(parent, suffix) rename
    the original agent with its own --<role> suffix so the bare family name becomes
    a pure container; keep wait-on-family and sequential-only execution.'
- id: epic-clan
  title: Epic migration to clans and the @epic tribe
  depends_on:
  - clan-launch
  - tribe
  description: '''Phase epic-clan'' section: rename the epic lander to <epic_id>.land
    in sase-core, flip the legacy cleanup direction, and emit %clan:<epic_id> plus
    %tribe:epic on every epic segment.'
- id: tui-tree
  title: Three-level Agents-tab tree and clan runtime column
  depends_on:
  - core-runtime
  - clan-launch
  description: '''Phase tui-tree'' section: add synthetic clan rows, clan-aware h/l
    fold levels with a third indentation level, the corrected wall-clock clan runtime
    column, and clan-aware kill/dismiss cascades.'
- id: clan-panel
  title: Aggregate clan metadata panel
  depends_on:
  - tui-tree
  description: '''Phase clan-panel'' section: render a complete aggregate detail-panel
    view (header stats plus a navigable members section) when a clan row is selected.'
- id: docs-memory
  title: Glossary, docs, and chezmoi migration
  depends_on:
  - epic-clan
  - family-sequential
  - clan-panel
  description: '''Phase docs-memory'' section: rewrite the glossary definitions (user-authorized),
    refresh docs and help surfaces, and migrate research_swarm plus %g usages in the
    chezmoi repo.'
- id: smoke
  title: End-to-end clan exercises
  depends_on:
  - docs-memory
  description: '''Phase smoke'' section: exercise an epic clan, a swarm clan, family
    attach, and the TUI tree end-to-end; refresh PNG snapshots and check j/k latency.'
create_time: 2026-07-17 17:34:42
status: done
bead_id: sase-6n
---

# Plan: Split agent clans, sequential families, and tribes

## Context & Motivation

SASE recently migrated epic phase workers + the epic lander, and the `research_swarm` xprompt, onto "parallel agent
families" (`%family` directive). This overloaded the existing sequential "agent family" concept (`--`-suffixed
plan-chain follow-ups) and produced real bugs and awkward semantics:

- The parallel group is represented by a **root agent** (the epic lander is literally named `<epic_id>`; the swarm lead
  is `research.@.final`). The group's identity is entangled with a real agent, which blocks name hygiene and makes
  "root"/"member" terminology misleading.
- The aggregate runtime shown at the far right of Agents-tab rows is wrong for parallel groups:
  `src/sase/ace/tui/models/agent_time.py::_aggregate_runtime` **sums** each member's elapsed seconds instead of taking
  the wall-clock union of their run intervals, double-counting concurrent members.
- Members that are themselves families cannot be expanded: the fold filter (`models/_fold_filter.py`) and row rendering
  only model a single parent→child level.

This epic separates three concepts cleanly:

| Concept          | Replaces                           | Directive            | Key property                                                          |
| ---------------- | ---------------------------------- | -------------------- | --------------------------------------------------------------------- |
| **Agent Clan**   | parallel agent family              | `%c` / `%clan`       | Named container, never an agent; members live in its hood             |
| **Agent Family** | (unchanged name) sequential family | `%n(parent, suffix)` | Strictly sequential; created by rename-on-attach; never single-member |
| **Agent Tribe**  | agent groups/tags                  | `%t` / `%tribe`      | User label, displayed `@name`                                         |

There are no existing `clan`/`tribe` identifiers anywhere in sase or sase-core (verified), so the vocabulary is free.
The only real-world users of `%family`/`%group` are the epic launch generator (`src/sase/bead/work.py:343`), the chezmoi
`research_swarm.md` xprompt, and `%g:chop`/`%g:research` tags in chezmoi's `sase_athena.yml` routines — all migrated by
this epic. No backwards-compatible directive aliases are kept; `%f`, `%family`, `%g`, and `%group` are removed outright.

## Target Concepts (the contract)

**Agent Clan.** A named container for agents launched to work in parallel.

- A clan is **never an agent**. Its name is reserved: it must not equal the name of any existing or archived agent, and
  no future agent may claim it. Both directions are validated with clear errors.
- Members are agents and/or agent families. Every member's name must be inside the clan's hood: `<clan>.<suffix>`
  (`foo.bar` can join clan `foo`; `baz.bar` cannot). Violations fail launch planning with an error notification that
  names the offending agent, the clan, and the required prefix.
- Declared with `%clan:<name>` on **every** member segment (no root segment, no roles — a member's identity is its
  hood-relative suffix like `.land`, `.3`, `.cdx`). Template names are allowed (`%clan:research.@`); segments with an
  identical raw `%clan` value in one batch form one clan, and a later launch may join an existing clan by resolved name
  (mirroring today's `resolve_existing_family_membership`).
- Execution-neutral: no implied waits or ordering; `%wait` stays explicit. `%wait:<clan>` (and `#fork` fallback
  resolution) waits on the entire clan — all members complete.
- Killing/dismissing the clan cascades to its live members; killing one member leaves the rest alone (preserves today's
  root-cascade behavior, re-keyed to the clan).
- A clan member that grows into a family stays a member as a family; agents attached to that family inherit clan
  membership.

**Agent Family.** A strictly sequential chain of follow-ups sharing `<family>--<suffix>` names.

- Created **only** when a new launch targets an existing agent via two-argument `%n(<parent>, <suffix>)` (including
  plan-chain auto follow-ups). At first attach, the original agent is **renamed** with its own `--<role>` suffix —
  persisting what today is only the display slot from `assign_bare_family_root_zero_suffix` (`--plan-0` for a plan
  proposer, `--0` for a generic root). `<parent>` becomes the family name: a pure container, never an agent.
  Consequently no family ever has a single member.
- Members never run concurrently: the existing queue-behind-exact-parent-artifact machinery is the enforcement; `%clan`
  and `%n(parent, suffix)` remain mutually exclusive in one segment.

**Agent Tribe.** A user-facing label grouping related agents across clans and families (`@epic`, `@chop`, `@research`).
Assigned with `%tribe:<name>` (alias `%t`), managed with `sase agent tribe`, and always displayed with an `@` prefix
(never `#`). The persisted store (`~/.sase/agent_tags.json`) and the internal/wire field spelling `tag` are
intentionally unchanged — this epic renames the user-facing surface only.

## Design Decisions

**Data model (wire).** Add `agent_clan: Option<String>` to `AgentMetaWire` (sase-core `agent_scan/wire.rs`) and to
`agent_meta.json`, with matching Python mirrors in `src/sase/core/agent_scan_wire.py` and the SQLite artifact index (new
column + index; bump the scan-wire and artifact-index schema versions). `agent_family`/`agent_family_role` are reserved
for sequential families; `agent_family_parallel` is no longer written. Legacy mapping at the projection layer: records
with `agent_family_parallel == true` surface as `agent_clan = agent_family` with the family fields cleared, so archived
agents keep rendering correctly without a data migration.

**Clan identity without a root.** `FamilyMembershipPlan` / `SASE_AGENT_FAMILY_MEMBERSHIP` (root-pinned) becomes a
`ClanMembershipPlan` env payload carrying the clan name plus a per-launch clan generation stamp (the batch launch
timestamp) instead of `root_timestamp`/`is_root`. Wait/identity pinning that previously keyed on the root agent's
artifact keys on the clan generation. The TUI groups members by `agent_clan` (+ generation) instead of
`parent_timestamp`-to-root.

**Name reservation.** The agent-name registry (`src/sase/agent/names/`) gains clan container records: creating a clan
claims the name under the existing allocation lock (rejecting names of existing/archived agents or other clans), and
`validate_launch_name_requests` rejects any `%name` equal to a reserved clan name with a message suggesting a name
inside the clan's hood. Family bases created by rename-on-attach are reserved the same way. The dotted-namespace prefix
machinery (`_dotted_namespace_prefixes`) already prevents template-named collisions; this extends the same guarantee to
explicit names.

**Runtime aggregation (the bug fix) lives in sase-core.** All timestamps needed already exist on `AgentMetaWire`
(`run_started_at`, `stopped_at`, `plan_submitted_at[]`, `questions_submitted_at[]`, waiting/pending-question markers,
`DoneMarkerWire.finished_at`). Add a pure core function + wire type (e.g.
`ClanRuntimeWire { wall_clock_seconds, active }`) that computes the measure of the union of member active intervals with
human-wait windows excised (PLAN submission → resolution, QUESTION submission → answered, open pending-question windows
extend to "now" excluded). This matches the telemetry/vcs-log precedent for cross-agent math in core, and gives every
frontend identical numbers. The same function serves sequential families (disjoint intervals ⇒ union == sum), so one
correct implementation replaces the buggy Python sum.

**TUI tree.** Reuse the existing three-state `FoldLevel` machine for clans: COLLAPSED (clan row only) → `l` → EXPANDED
(members: agents, family rows, workflow steps) → `l` → FULLY_EXPANDED (hidden steps plus family members at a third
indentation level); `h` reverses, matching the current "press `l` twice" idiom. Clan rows are synthetic (not agents),
styled distinctly but consistently: a clan glyph + name, compact member status counts (reusing
`parallel_family_member_counts` visuals), `@tribe` labels, and the wall-clock runtime right-aligned. Indentation becomes
depth-aware (the flat `_CHILD_INDENT` gains a depth multiplier; `models/agent_panel_summary._row_depth` is the in-repo
precedent) with tier-guide gutters kept visually consistent with group banners.

## Phase core-runtime

Open the `sase-core` repo with the `/sase_repo` skill; `just install` in the sase workspace builds `sase_core_rs` from
the local checkout so Python changes can be exercised against the new core.

- Add `agent_clan` to `AgentMetaWire` + `RecordSummary` + the SQLite `agent_artifacts` schema (column + index; bump
  `AGENT_SCAN_WIRE_SCHEMA_VERSION` and `AGENT_ARTIFACT_INDEX_SCHEMA_VERSION`), with the legacy `agent_family_parallel` →
  `agent_clan` projection mapping described above and Rust tests for both shapes.
- Implement the clan runtime interval-union aggregation as a pure `sase_core` function with a wire type, an explicit
  human-wait exclusion policy, and exhaustive unit tests (overlapping members, gaps, open intervals, plan/question
  windows, empty input). Bind it in `sase_core_py` and expose it through a thin facade in `src/sase/core/`.
- Mirror all wire changes in `src/sase/core/agent_scan_wire.py` (field order must match) and update the wire schema
  parity tests in both repos.

## Phase clan-launch

- Directive surface: add `clan` to `_KNOWN_DIRECTIVES` with alias `c`; remove `family`/`f`. Grammar is colon-only
  (`%clan:<name>`); the paren/role form is dropped. Update `_directive_collect/_extract/_values/_types`,
  `directive_completion.py` hints, `directive_edit.py`, and the docstring directive lists.
- Launch planning (`multi_prompt_launcher.py`): replace `_ParallelFamilyPrepass` root-matching with clan grouping by
  identical raw `%clan` value; resolve template clan names once per batch; support joining an existing clan from a
  separate launch. Replace `FamilyMembershipPlan` with the rootless `ClanMembershipPlan` (new env var name), and persist
  `agent_clan` in `run_agent_directives.py` instead of `agent_family`+`agent_family_parallel`.
- Validation with excellent errors: hood-membership check per segment (resolved `%name` must be `<clan>.<suffix>`),
  clan-name uniqueness in both directions via the name registry, and self/duplicate-clan guards. Errors surface both at
  launch planning and as notifications when planning happens detached.
- Update kill/dismiss cascade, wait resolution (`%wait:<clan>` = whole clan; member names still target members), and the
  TUI model loaders (`_meta_enrichment_wire/_filesystem`) plus `_agent_parallel_family.py` aggregation to key on
  `agent_clan` (rename the module vocabulary to clans as part of this phase). Rewrite the affected tests
  (`tests/test_directives_family.py`, `tests/test_parallel_agent_family_*`, etc.) under the new names.

## Phase tribe

- Directive: add `tribe` with alias `t`; remove `group`/`g`. `PromptDirectives.tag` stays the internal field; update
  `parse_group_tag` → tribe naming, `directive_edit.py`, completion hints, and error text
  (`"'%tribe' directive requires a tribe name argument (e.g., %tribe:review)"`).
- CLI: rename `sase agent tag` → `sase agent tribe` (`parser_agent.py`, `cli_tag.py`, `agent_handler.py`) following the
  CLI rules memory (alphabetical listing, short aliases, excellent colored help). No hidden legacy alias.
- Display: replace every `#`-prefix tribe rendering with `@` — row label (`_agent_list_render_agent.py:189`), panel
  titles (`_display_panel_titles.py:99`), collapsed panel summary (`models/agent_panel_summary.py:81`), and agent
  completion candidates (`agent_completion.py:239`). Leave the unrelated `#{embedded_workflow_name}` annotation
  untouched. The render-cache key already includes the tag fields, so this is display-only.

## Phase family-sequential

- Implement rename-on-attach: when `%n(parent, suffix)` (or a plan-chain follow-up) attaches the **first** member to an
  agent, transactionally rename the original to `<parent>--<role>` (reusing the existing display-slot derivation: plan
  proposers get `--plan-0`, generic roots `--0`), update its `agent_meta.json` name, re-index the artifact record, and
  convert the registry claim on `<parent>` into a family container reservation. The rename must be safe when the parent
  is still running (name is metadata; the artifacts directory is timestamp-keyed and unchanged).
- Preserve semantics that already exist and must keep working after the rename: wait-on-family (`%wait:<family>` waits
  for the whole chain), `#fork` resolution, bead badge derivation through `--`-suffix stripping, and sequential queueing
  of members. Remove the now-redundant display-only `assign_bare_family_root_zero_suffix` path in favor of the persisted
  rename.
- Enforce "no single-member families" as an invariant (a family exists iff ≥2 members) and keep the
  `%clan`-vs-`%n(parent, suffix)` mutual exclusion with updated wording.

## Phase epic-clan

- sase-core (via `/sase_repo`): change `land_agent_name` in `crates/sase_core/src/bead/work.rs` to
  `format!("{epic_id}.land")`; update the Rust tests and the Python parity test
  (`tests/test_bead/test_work_epic_plan.py`).
- Flip the legacy-name reconciliation in `src/sase/bead/cli_work_plan.py`: the plain `<epic_id>` becomes the legacy wipe
  target and `<epic_id>.land` the live name; update collision tests.
- `render_multi_prompt` (`src/sase/bead/work.py`): every segment (phases and lander) emits `%clan:<epic_id>` and
  `%tribe:epic` instead of `%family(<lander>, role=phase)`; delete `_family_directive`. Update the human launch preview
  and `docs`-visible examples. Verify the `.land` display paths (`bead_display.py`, `agent_associated_plan.py`
  "land"/"ambiguous" branches) with tests — the `.land` suffix handling already exists.

## Phase tui-tree

Respect the TUI perf memory throughout (selective updates, cache-key completeness, no event-loop work).

- Row model: introduce synthetic clan row entries built from `agent_clan` grouping; extend `_fold_filter.py` to resolve
  visibility recursively (clan → member → family member) and `compute_fold_annotation` / `_compute_visible_parents` to
  be level-aware. Wire `h`/`l` through the existing `FoldStateManager` three-state machine per the design above; keymap
  names in `default_config.yml` are unchanged.
- Rendering: depth-aware indentation with tier guides for the third level; clan row styling (glyph, name, compact status
  counts, `@tribes`, right-aligned wall-clock runtime). Extend `agent_render_key` and `_runtime_signature` in
  `_agent_list_render_cache.py` with every new visible input (depth, clan identity, clan runtime), and extend the
  `patch_row` / `try_remove_rows` gates so grandchildren never strand or silently force full rebuilds.
- Runtime column: replace `_aggregate_runtime`'s sum with the core interval-union aggregation (via the phase
  core-runtime facade) for clans and families; keep tick predicates in sync so live clans update once per second.
- Update status projection (`agent_summary_status_counts`), kill/dismiss cascade targets, headline counts, the
  keybinding footer conditions, and the help modal (`?`) — including the fold legend — for the new tree. PNG visual
  snapshots for collapsed/expanded/fully-expanded clan states.

## Phase clan-panel

- When a clan row is selected, the detail panel renders an aggregate view: a CLAN header (name, `@tribes`, rolled-up
  status with counts, wall-clock runtime, "N agents · M families"), followed by a navigable MEMBERS section built on the
  existing `_append_parallel_family_members_section` machinery — one row per member in launch order showing hood suffix,
  kind (agent/family/step), status glyph, model, and duration; family members render one row each, indented under their
  family with the family's own aggregate line. Selecting a family member row navigates to it, matching current MEMBERS
  behavior.
- Route panel updates through `DetailPanelDebouncer`; add PNG snapshots for the clan panel (epic-shaped and swarm-shaped
  clans).

## Phase docs-memory

- **Glossary (`sase/memory/glossary.md`).** The user explicitly authorized this memory edit in the originating request:
  "You should add a better set of glossary definitions to the memory/glossary.md file (using the definition-list format
  that is already used in that file)". Replace the current **Agent Family** entry and add **Agent Clan** and **Agent
  Tribe** entries per the "Target Concepts" section, keeping each entry to a few concise, accurate sentences. Remove the
  misleading **Root Agent/Workflow Entry** and **Child Agent/Workflow Step Entry** entries, replacing them with a single
  **Agents-Tab Nesting** entry: rows nest clan → members (agents, families, workflow python/bash steps) → family
  members; `l` expands one level (twice reveals hidden steps and family members), `h` collapses. Do not edit generated
  shims by hand; `sase init` regenerates them post-commit.
- **Docs.** Rewrite `docs/agent_families.md` around the clan/family/tribe split (renaming or splitting the page as
  appropriate), update the directive table and examples in `docs/xprompt.md`, and sweep `docs/ace.md`,
  `docs/query_language.md`, `docs/beads.md`, and the `sase_run` skill source for `%family`/`%group`/`#tag` references.
  Update `sase/memory/xprompts.md`'s directive table (same explicit user authorization applies to this cleanup of stale
  `%g` rows only if needed — otherwise leave untouched).
- **Chezmoi (via `/sase_repo`).** Migrate `home/sase/xprompts/research_swarm.md`: each of the four segments gets
  `%clan:research.@` (dropping `%family(research.@.final, role=...)`) and `%t:research` replaces `%g:research`;
  waits/models/forks are unchanged. Migrate `old_research_swarm.md` `%g` usages and the `%g:chop` occurrences in
  `home/dot_config/sase/sase_athena.yml` to `%t`. Check `sase-nvim` (via `/sase_repo`) for `%family`/`%group`
  syntax/completion support and update it if present. Follow the chezmoi repo's post-commit instruction
  (`chezmoi update -a --force`).

## Phase smoke

Exercise the finished feature end-to-end in a scratch project; fix what breaks and refresh goldens:

- Launch a small real epic: verify phase agents + `<epic_id>.land` land in clan `<epic_id>` with tribe `@epic`, the clan
  row aggregates correctly, `l`/`l`/`h` walk all three levels, and the clan runtime reflects wall-clock (two overlapping
  phases must not double-count; a PLAN gate pause must not accrue).
- Launch the migrated `research_swarm` (or a local copy of its prompt) and verify clan grouping and the clan panel.
- Attach a follow-up via `%n(parent, suffix)` to confirm rename-on-attach, the container family name, wait-on-family,
  and clan inheritance for a family inside a clan hood.
- Verify clan-name collision and hood-violation errors read well (both CLI and notification surfaces).
- Run `just check`, the visual snapshot suite, and a `SASE_TUI_PERF=1` j/k spot-check on a populated Agents tab (p95 <
  16 ms per the perf memory).

## Testing Strategy

Each phase lands with its own unit coverage: Rust interval-union and wire-mapping tests (core-runtime), directive
parse/validation and launch-planning tests (clan-launch, tribe, family-sequential), epic prompt-render tests
(epic-clan), and fold/render/cache/panel tests plus PNG snapshots (tui-tree, clan-panel). The smoke phase is the
cross-cutting integration pass. Legacy-shape fixtures (an `agent_family_parallel` archive record) must keep loading
throughout.

## Risks

- **Wire/schema bumps** (scan wire, artifact index) invalidate cached indexes; rely on the existing
  stale-schema-detection startup path (perf memory rule 9) rather than blocking first paint.
- **Rename-on-attach of a running parent** touches live metadata; it must stay transactional under the name allocation
  lock, and anything holding the old name (pending waits, unread markers) must resolve through the family container
  semantics. This is the riskiest change; family-sequential should land behind thorough tests.
- **sase-core coupling:** phases core-runtime and epic-clan change sase-core; `just install` rebuilds the local wheel,
  but the pinned `sase-core-rs` version window in `pyproject.toml` must be bumped in step with a core release before the
  sase changes can ship independently of a sibling checkout.
- **Memory-file edits** (glossary) are explicitly user-authorized in the originating conversation, as quoted in the
  docs-memory phase; no other memory files may be modified.
