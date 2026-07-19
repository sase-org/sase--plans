---
tier: epic
title: Complete the agent tribe terminology migration
goal: 'Agent tribes are named "tribe" throughout current source, APIs, persisted output,
  Rust/Python wire contracts, ACE, tests, and documentation, while existing tag-shaped
  tribe state remains safely readable at explicit legacy migration boundaries and
  unrelated tag concepts remain unchanged.

  '
phases:
- id: contracts
  title: Canonical tribe persistence and wire contracts
  depends_on: []
  description: '''Canonical tribe persistence and wire contracts'' section: establish
    read-old/write-new tribe storage and versioned Python/Rust contracts.'
- id: runtime
  title: Runtime and integration cutover
  depends_on:
  - contracts
  description: '''Runtime and integration cutover'' section: move directive, launch,
    metadata, model, wait, CLI, and editor flows onto canonical tribe names.'
- id: ace
  title: ACE tribe surfaces and behavior
  depends_on:
  - runtime
  description: '''ACE tribe surfaces and behavior'' section: rename panel, query,
    action, cleanup, fold, archive, and visual interfaces without changing behavior
    or responsiveness.'
- id: audit
  title: Documentation, compatibility audit, and release validation
  depends_on:
  - ace
  description: '''Documentation, compatibility audit, and release validation'' section:
    update docs and memory, prove migrations, sweep residual terminology, and run
    both repositories'' full checks.'
create_time: 2026-07-19 13:59:12
status: wip
bead_id: sase-7j
---

# Plan: Complete the agent tribe terminology migration

## Context and intended result

The public concept is already an **agent tribe**: prompts use `%tribe` / `%t`, the CLI uses `sase agent tribe`, clan
declarations use `tribe=`, and ACE presents `@<tribe>` labels. The implementation still exposes the same concept as a
tag in several places: `PromptDirectives.tag`, `Agent.tag`, `agent_tags.json`, `agent_meta.json["tag"]`, scan/archive/
cleanup wires, panel and cleanup helpers, query syntax, keymap action names, modal copy, tests, and documentation. This
partial rename makes it too easy for new code to perpetuate the obsolete vocabulary.

This is a semantic migration, not a global word replacement. Keep **tag** where it accurately means a VCS/workspace tag,
xprompt semantic-role tag, prompt/frontmatter tag, notification/gate tag, ChangeSpec or commit-footer tag, Git release
tag, YAML/Jinja/serde tag, generic telemetry discriminator, or historical changelog text. Likewise, saved agent _groups_
remain groups: they are named saved selections, not tribes. Commit provenance fields such as `SASE_AGENT` and their
`agent_tag` parsing helpers also remain tags.

The final tree should use tribe terminology for every current agent-tribe symbol, filename, field, scope, action, query,
message, comment, test, and document. Old tag-shaped spellings may survive only inside clearly named legacy readers,
input aliases needed to load existing durable state, and fixtures that prove those migrations. New writes and new public
output must never emit the old shape.

## Compatibility and naming policy

- Make `tribe` the canonical scalar on directive results, runner results, artifact metadata, Python models, Rust models,
  cleanup requests, archive records, editor/list projections, modal results, and CLI JSON. Use `tribes` for collections
  and `clan_tribe` / `clan_tribes` only for the existing clan-declaration distinction.
- Replace tag-derived module, class, function, constant, argument, action, CSS/widget id, test, and fixture names with
  tribe-derived names. Avoid permanent internal aliases merely to reduce the rename diff.
- Preserve user data with read-old/write-new migrations. Canonical standalone assignments live in
  `~/.sase/agent_tribes.json` as records containing `"tribe"`. If that file exists it is authoritative; if it does not,
  load `agent_tags.json` and its scalar `"tag"` / older list `"tags"` shapes as legacy input. The first successful
  mutation writes the complete imported state to the canonical file. This precedence is essential so clearing a tribe
  cannot make a stale legacy assignment reappear. Do not delete the legacy file.
- New `agent_meta.json`, dismissed bundles, saved-group archives, fold state, and wire payloads write only tribe-shaped
  fields. Readers prefer the new field and fall back to the old field only when the new one is absent. An edit or
  rewrite removes obsolete aliases from that record, including on unset.
- Bump every versioned wire or persisted schema whose emitted shape changes, and retain explicit support for the prior
  version where durable files may already exist. For an in-process Python/Rust contract, fail over cleanly when an
  installed binding exposes the old schema instead of allowing a half-renamed payload to be accepted.
- Current user-facing vocabulary changes rather than advertising deprecated synonyms: query help and examples use
  `tribe:`, unassigned rows/panels say `(no tribe)` (or an equally clear tribe-based phrase), CLI JSON says `tribe`, and
  ACE commands say set/edit/clear tribe. Compatibility-only old literals are not shown in help or normal output.

## Canonical tribe persistence and wire contracts

Establish the migration foundation across the main `sase` repository and the linked `sase-core` repository. Open the
linked repository through `/sase_repo` before working in it, and keep the Rust and Python definitions synchronized.

- Turn the existing tribe helper and persistent assignment modules into a canonical tribe API (`validate_tribe_name`,
  tribe-specific errors/types, load/save/update/unset operations, and review/pinned tribe constants). Isolate legacy
  `agent_tags.json`, `"tag"`, and `"tags"` parsing behind explicitly named migration helpers and test canonical-file
  precedence, atomic updates, locking, malformed input, import-on-first-write, replacement, and unset behavior.
- Rename the agent scan metadata projection from `tag` to `tribe` in Rust and Python. Teach the scanner to read
  canonical artifact metadata first and legacy metadata second, bump both the scan wire and persistent artifact-index
  schema, and exercise the established stale-index fallback/rebuild path so the schema change never moves an archive-
  sized scan onto ACE startup or the UI thread.
- Rename tribe-bearing saved-group archive fields in the Rust wire, PyO3 conversion, Python wire/facade, capability
  probe, and persisted JSON. Bump the archive schema and accept/migrate existing version-1 records without changing the
  meaning of an actual saved group.
- Rename the cleanup backend contract end to end: target tribe, focused-panel tribe, tribe scope, request values, parent
  effective-tribe maps, Python reference planner, Rust planner, PyO3 payloads, and parity fixtures. Bump the cleanup
  wire schema and make stale-binding detection/fallback explicit. Preserve the planner's existing inheritance,
  selection, kill/dismiss, side-effect, and clan behavior.
- Add focused Rust and Python contract tests proving that legacy input hydrates the new fields, canonical serialization
  contains only tribe keys, schema mismatches are safe, and Python/Rust planners remain byte-for-byte equivalent for
  tribe, focused-panel, clan, marked, custom, and workflow-child cases.

## Runtime and integration cutover

Move producers and non-ACE consumers onto the new contract, then remove any temporary aliases used to keep the first
phase buildable.

- Change `%tribe` extraction and validation to populate `PromptDirectives.tribe`; propagate that through `AgentInfo`,
  launch validation, family/clan rules, runner metadata, automatic review assignment, and post-hoc persistence. New runs
  write `agent_meta.json["tribe"]` and the canonical tribe store. `%tribe` syntax and clan-wide `tribe=` semantics
  remain behaviorally identical.
- Rename the mutable agent state and related presentation data from `tag` to `tribe`, including the tuple used for clan
  labels. Keep direct standalone tribe assignment distinct from `clan_tribe`, and continue deriving a single effective
  panel tribe from the presentation anchor so clans and families are never split across panels.
- Update filesystem and scan-wire enrichment, review loaders, running/done deduplication, dismissed bundle round trips,
  revive metadata, and saved-record construction. Older artifacts and bundles with `tag` still load; every newly written
  or rewritten artifact/bundle uses `tribe`.
- Rename wait-dependency index state and parameters from agent tags to agent tribes. Resolve direct artifact metadata,
  post-hoc assignments, clan-wide declarations, `%wait:@tribe`, and `#fork` with the same newest-generation and
  inheritance rules as today.
- Rename the `sase agent tribe` implementation module and its internal helpers, and emit `tribe` in list JSON and error
  messages. Update presentation-neutral agent list models and the editor helper catalog so agent/family/clan/tribe
  completion is derived from tribe fields. Do not disturb `AgentVcsWorkflow.tag` or xprompt catalog tags, which are
  separate concepts.
- Cover directive parsing, runner writes, review auto-assignment, old/new metadata precedence, bundle revival, wait/fork
  resolution, CLI JSON/help, and editor completion with focused tests before the ACE-wide cutover.

## ACE tribe surfaces and behavior

Complete the rename through ACE while preserving the existing cached/off-thread architecture described by the TUI
performance contract.

- Rename agent tribe assignment actions, mixins, task records/dedup keys, modal/result classes, modules, widget ids,
  styles, imports, bindings, command-palette metadata, keymap types, and `default_config.yml` entries. Keep the `N`
  behavior (including marked-agent bulk edits and pinned default) while changing all copy to set/edit/clear **tribe**.
- Rename tag-panel terminology throughout panel indexing, collection, diffing, layout, titles, rendering, navigation,
  neighbor/revive selection, unread jumps, fold registries, caches, and tests. Keep panel keys generic, preserve stable
  ordering and focused-panel identity, and retain the selective patch/cache paths; the rename must not add filesystem
  work, full rebuilds, awaits, or subprocesses to navigation/render handlers.
- Change renderer inputs such as tag labels/panel tag to tribe labels/panel tribe, rename synthetic clan tag
  collections, and use tribe wording for the unassigned panel and all help/footer/document strings. Update only visual
  goldens whose pixels intentionally change because visible terminology changed.
- Replace the Agents query property `tag:` with `tribe:` in tokenization, evaluation, canonicalization, completion/help,
  saved-query coverage, and the `pinned` explanation. Keep exact case-insensitive matching and empty-value "any tribe"
  behavior.
- Rename cleanup chooser/result/state types, custom filters, user-facing choices, and action dispatch from tag to tribe;
  route them through the new Rust/Python cleanup wire. Rename fold-state panel discriminators to tribe/no-tribe, version
  the file, decode the old tag/untagged representation, and write only the new representation.
- Re-run the high-value panel/cache/diff/navigation, clan/family projection, assignment persistence, query, cleanup,
  revive/archive, and PNG snapshot suites. Include a responsiveness check or existing j/k benchmark to confirm that
  tribe panel navigation stays on its prior fast path.

## Documentation, compatibility audit, and release validation

Finish with a repository-wide semantic audit rather than assuming mechanical renames were complete.

- Update current documentation (`docs/ace.md`, `docs/agent_families.md`, configuration/query/help references, and any
  maintained image prompts) to describe canonical tribe storage and interfaces. Add a changelog entry for renamed JSON,
  query, keymap/config, persisted-file, and wire contracts, while leaving historical changelog entries untouched.
- Update the stale `sase agent tag` example in `sase/memory/cli_rules.md`. The user's repository-wide request provides
  the required memory-edit authorization; after editing the canonical note, run `sase memory init` and review the
  regenerated `AGENTS.md`, provider instruction shims, and memory README so only expected derived changes land.
- Run case-sensitive and case-insensitive `rg` sweeps for the known stale families (`agent_tag(s)`, `AgentTag`, `.tag`
  on agent models, tag panels/scope/query/action names, `untagged` agent copy, metadata/archive/cleanup `"tag"` keys).
  Classify every hit. The only accepted hits are unrelated tag domains, immutable history, or narrowly isolated legacy
  readers/fixtures whose names and comments make the compatibility purpose unmistakable. Add a focused terminology
  regression check if a stable scoped allowlist can prevent recurrence without flagging legitimate tags.
- Prove migration behavior from realistic old files: legacy standalone assignments, legacy artifact metadata, version-1
  saved archives/bundles/fold state, and an old artifact index. Verify canonical rewrites, precedence, clear/unset, and
  no data resurrection. Verify new-state round trips and mixed clan/direct tribe panel projection as well.
- In `sase-core`, run formatting, clippy, and the full Rust workspace tests (or the main repository's `just rust-check`
  wrapper) with the Python binding available. In `sase`, run `just install` first as required for an ephemeral
  workspace, then targeted migration/TUI/visual tests, `just test-visual`, and finally mandatory `just check`. Re-run
  the semantic sweep after formatters, generators, and snapshot updates.

The epic is complete when a fresh agent can be launched, listed, waited on/forked, grouped, queried, edited, cleaned up,
dismissed, and revived using only tribe-shaped current contracts; pre-migration state still loads safely; Rust and
Python agree; ACE behavior/performance is unchanged apart from corrected wording; and no unexplained agent-tribe use of
"tag" remains.
