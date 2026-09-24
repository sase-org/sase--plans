---
tier: epic
title: Rename agent family to sase agent session
goal: 'The concept formerly called an agent family is named a sase agent session (agent
  session) on every current surface in sase, sase-core, sase-telegram, and chezmoi:
  code, wire contracts, persisted output, CLI, prompt syntax, ACE, skills, docs, and
  memory. Pre-rename data still loads, retired user syntax keeps working behind a sunset
  flag, and unrelated meanings of "family" are unchanged.

  '
phases:
  - id: free-name
    title: Free the agent session name
    depends_on: []
    size: small
    description:
      'free-name: rename the existing identifiers and prose that already use "agent
      session" for other things (provider transcripts, the ACE tmux session, a single
      agent run, a workflow lifetime, fold scope), so the phrase is free for the new
      concept.'
  - id: core-expand
    title: sase-core additive rename
    depends_on: []
    size: large
    description:
      "core-expand: non-breaking sase-core change. Rename the Rust internals to
      agent-session vocabulary and add the new pyo3 binding names alongside the old
      ones. Inputs accept both old and new spellings; serialized output stays
      byte-identical."
  - id: wire-cutover
    title: Python persistence and wire cutover
    depends_on:
      - free-name
      - core-expand
    size: large
    description:
      "wire-cutover: bump the core pin and switch sase to the new binding names. Rename
      the Python wire mirrors and durable JSON fields: new data is written only with
      agent_session keys, and readers accept both key spellings. Rename the Agent model
      fields and the rebuildable caches."
  - id: runtime-cutover
    title: Runtime, syntax, and CLI cutover
    depends_on:
      - wire-cutover
    size: large
    description:
      "runtime-cutover: rename every non-ACE module and identifier. Make session= /
      session: / --next-fork session / SASE_AGENT_SESSION_ATTACH canonical and keep the
      old spellings working behind the legacy_agent_family_syntax sunset flag. Update
      CLI help and JSON output, the editor bridge, and the skill templates."
  - id: ace-cutover
    title: ACE agent session surfaces
    depends_on:
      - runtime-cutover
    size: large
    description:
      "ace-cutover: rename ACE modules, row kinds, the grouping mode, and visible copy
      (SESSION SHELLS, SESSION). Update keymap and help text, default_config.yml and the
      schema, perf baselines, and the PNG goldens, with no change to the performance
      contract."
  - id: docs-memory
    title: Documentation and memory
    depends_on:
      - runtime-cutover
    size: medium
    description:
      "docs-memory: rename docs/agent_families.md to docs/agent_sessions.md and rewrite
      every concept mention in docs/ and the blog. Replace the Agent Family glossary
      strand with Sase Agent Session, update the related strands and notes, then run
      sase memory init."
  - id: telegram
    title: sase-telegram cutover
    depends_on:
      - runtime-cutover
    size: small
    description:
      "telegram: move the /show session kind, formatting, help, docs, and tests to the
      renamed sase APIs, and stop a missing import from silently disabling the lookup."
  - id: core-contract
    title: sase-core contract flip
    depends_on:
      - ace-cutover
      - docs-memory
      - telegram
    size: medium
    description:
      "core-contract: breaking feat! sase-core change. Serialize the new key and value
      names, drop the legacy binding names, emit sessions/ link paths and session: fleet
      keys, and bump the changed schema versions and the fleet protocol. Keep aliases so
      legacy durable data still reads."
  - id: session-pages
    title: Pin bump and agents sidecar session pages
    depends_on:
      - core-contract
    size: medium
    description:
      "session-pages: bump the core pin and the Python schema mirrors. Publish
      agents-sidecar pages under sessions/, keep permanent redirect stubs at the old
      families/ paths for historical commit-footer links, and update the sidecar docs,
      templates, and goldens."
  - id: audit
    title: Cross-repo audit, guardrail, and deploy
    depends_on:
      - session-pages
    size: medium
    description:
      'audit: add a terminology regression test and sweep every repo, classifying each
      remaining "family" hit. Regenerate the chezmoi skill copies from the landed tree
      and update the chezmoi ACE snippet.'
proposed_by: bbugyi200.athena.0qh
create_time: 2026-09-23 22:46:31
status: wip
---

- **PROMPT:**
  [prompts/202609/agent_session_rename.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/agent_session_rename.md)

# Plan: Rename agent family to sase agent session

## Context

A **sase agent session** (short form: **agent session**; formerly **agent family**) is a
sase agent whose agent shells run as a strict sequence named `<session>--<suffix>`. The
first `%id(parent, suffix)` attachment reserves the bare name as the session container.
The old term shows up in about 14.4k case-insensitive hits across roughly 1,265 sase
files, about 1.9k hits in 141 sase-core files, and 13 sase-telegram files. It also
appears in 49 generated chezmoi skill copies and one chezmoi ACE snippet. sase-github,
sase-nvim, and sase-research-artifacts have no agent-family references. The tag→tribe
migration (`plan:202607/agent_tribe_terminology.md`, epic sase-7j) is the precedent:
this is a semantic migration, not a global word swap. The shared policy below binds
every phase.

### Vocabulary

| Old                                             | New                                               |
| ----------------------------------------------- | ------------------------------------------------- |
| agent family / family (this concept)            | sase agent session / agent session / session      |
| agent families                                  | agent sessions                                    |
| family member, family root, family container    | session member, session root, session container   |
| family shell(s), `FAMILY SHELLS`, `FAMILY`      | session shell(s), `SESSION SHELLS`, `SESSION`     |
| family-attached shell; promote into a family    | session-attached shell; promote into a session    |
| `<family>--<suffix>`, `<family>--mon`, `--gate` | `<session>--<suffix>`, `<session>--mon`, `--gate` |
| agents sidecar `families/<global>.md`           | `sessions/<global>.md`                            |

### Identifier rules

- `agent_family` → `agent_session`, `AgentFamily` → `AgentSession`, `AGENT_FAMILY` →
  `AGENT_SESSION` (for example `AGENT_FAMILY_SEPARATOR` → `AGENT_SESSION_SEPARATOR`,
  `find_agent_family` → `find_agent_session`, `parse_agent_family_name` →
  `parse_agent_session_name`, `SASE_AGENT_FAMILY_ATTACH` → `SASE_AGENT_SESSION_ATTACH`).
- Where the concept appears as a bare `family`, `families`, `Family`, or `FAMILY` inside
  a longer identifier, JSON key, module, or file name, replace it with `agent_session`,
  `agent_sessions`, `AgentSession`, or `AGENT_SESSION`. Never use bare `session` there,
  because that word already means provider transcript sessions (`session_id`), TUI
  sessions (`sase.sessions`, `sase proc --session`), question sessions, proc sessions,
  snippet sessions, and usage windows. Examples:
  - `family_shell` → `agent_session_shell` and `FamilyShellWire` →
    `AgentSessionShellWire`
  - `family_id`, `family_name`, `family_role`, `family_label` → `agent_session_id`,
    `agent_session_name`, `agent_session_role`, `agent_session_label`
  - `family_attach_parent` → `agent_session_attach_parent` and `family_root_suffix` →
    `agent_session_root_suffix`
  - `canonical_global_family` → `canonical_global_agent_session` and
    `dismissed_family_*` → `dismissed_agent_session_*`
  - modules: `agent/_family_attach_*.py` → `agent/_agent_session_attach_*.py`,
    `history/chat_fork/family.py` → `history/chat_fork/agent_session.py`,
    `fleet_family.rs` → `fleet_agent_session.rs`
- Enum and string values, and the short tokens users type, use bare `session`. There the
  surrounding context already removes any ambiguity:
  - kind, role, scope, container, reservation, link-target, and stats group-by values:
    `"family"` → `"session"`
  - `"convert_family"` → `"convert_session"` and `"serial_family"` → `"serial_session"`
  - `%id(<suffix>, family=<parent>)` → `%id(<suffix>, session=<parent>)`
  - agent query `family:` → `session:` and `kind:family` → `kind:session`
  - `sase gate create --next-fork family` → `session`, and gate spec `"fork": "family"`
    → `"session"`
  - ACE grouping mode `by_family` → `by_session`
  - fleet logical-key segment `family:` → `session:`, and fallback ids `family-<hex>` →
    `session-<hex>`
- The retired "parallel family" marker (`agent_family_parallel`, which already loads as
  a clan) follows the same rules in code. If no current writer still emits it, keep
  `agent_family_parallel` as a legacy input key only and rename the in-memory and wire
  fields. Prose describes it as the legacy parallel marker that loads as a clan.
- Rename comments, docstrings, log messages, error messages, and test names along with
  the code.

### Meanings of "family" that must not change

- `model_family`, provider-usage `family:` scopes (`family:3p`, `family:gemini`), and
  `_PROVIDER_FAMILY_COLORS`.
- `vcs_family`, `detect_vcs_family`, and `get_display_name_by_vcs_family`. This includes
  sase-github's only hits.
- The Patch revert/sibling family: `RelationKind.FAMILY`, `RelationRole.FAMILY`, the
  query `.family` operator, the ChangeSpec `__N` family in sase-core `query/`, and
  "reverted-family" in the `sase_patches` skill. The Agents-pane relation _named_
  `family` (source `agent_family_container`) is this concept and is renamed; the Patch
  _kind_ stays.
- CSS `font-family`, VHS `FontFamily`, and fontconfig. OS family (`target_family`,
  `LINUX_FAMILY_OS`).
- Metric, token, command, hue, merge-subject, and language families.
  "familiar"/"unfamiliar". `SASE_ML_FILE_FAMILIES`.
- The glossary pluralization test ("Family"→"Families") in sase-core.

### Deliberately unchanged history

- Generated CHANGELOGs: sase uses release-please and sase-core uses release-plz. Never
  hand-edit them. Breaking-change footers produce the new entries.
- Accepted decision records under `sase/memory/decisions/`. Records are immutable, and a
  terminology rename is not a change of course.
- Sidecar archives: plans, beads, research reports, and agent prompts. Also git history.
- Published blog posts are _not_ treated as history. They are updated like the rest of
  `docs/`.

### Compatibility policy

1. **Durable data:** readers prefer the new spelling and fall back to the legacy one.
   Writers emit only the new spelling, and any rewrite drops the legacy key. These
   legacy readers are permanent, like the existing parallel-family → clan and tag →
   tribe readers. Put them in explicitly named helpers and constants (for example
   `LEGACY_AGENT_FAMILY_KEY`), not scattered literals.

   Durable surfaces:
   - `agent_meta.json` keys and the nested shell object in `done.json`
   - dismissed agent bundles, saved dismissed groups, and fleet `follows.json` logical
     keys
   - `wait_for_fork_sources`, chat-fork sources, gate descriptors and `gate_next_fork`,
     and notification `action_data`
   - ops revert requests, the Rust-owned hold store, held launch wires, and runner-slot
     records
   - the agents-sidecar manifest and snapshot, and commit-footer links

2. **Rebuildable caches:** the artifact-index SQLite (`agent_family` column and index)
   and `agent_name_registry.json` are renamed and their schema versions bumped. Rebuilds
   use the existing stale-cache fallback path, which must never move an archive-sized
   rebuild onto ACE startup or the UI thread.
3. **User-authored syntax:** a single `sunset` flag, `legacy_agent_family_syntax`, keeps
   the retired spellings working as silent aliases while callers migrate:
   - `family=`
   - `family:` and `kind:family`
   - `--next-fork family` and `"fork": "family"`
   - the `SASE_AGENT_FAMILY_ATTACH` handoff variable across an upgrade

   With the flag off, each is rejected with an error that names its replacement. Help,
   completion, examples, and output never show the legacy spellings.

4. **Two-step Rust contract (expand/contract):** every sase workspace builds
   `sase_core_rs` from the linked sase-core checkout. A single breaking core commit
   would therefore break every concurrent sase workspace until the Python cutover
   landed.
   - `core-expand` is additive and keeps serialized output unchanged.
   - `core-contract` flips the output and removes the old binding names only after all
     sase and telegram work has landed and every sase reader accepts both spellings.
5. **Versions:** bump every versioned wire or persisted schema whose _emitted_ shape
   changes (in `core-contract`). Keep the Python mirror constants in sync (in
   `session-pages`). Bump the fleet protocol version so a mixed-version fleet fails with
   the existing clean `incompatible_protocol` error instead of a `deny_unknown_fields`
   decode failure.
6. Open every linked repo with `/sase_repo` and read its `AGENTS.md` first. Verify with
   `sase tool run check` inside each repo you changed. Run `just install` first in a
   fresh sase workspace, and never run `just check-full` unless explicitly told to. Read
   `sase/memory/tui.md` and `tui_perf.md` before ACE work, `generated_skills.md` before
   skill work, `cli_rules.md` before CLI changes, and `sase_flags.md` before creating
   the flag.

## Free the agent session name

Repo: sase. Frees the phrase before it gains its new meaning.

- Transcript lookup:
  - What changes: in `src/sase/ace/tui/thinking/session_resolver.py`,
    `resolve_agent_session` → `resolve_agent_transcript` and `resolve_agent_sessions` →
    `resolve_agent_transcripts`.
  - Where else: the `thinking/__init__.py` exports, all callers, and
    `tests/test_session_resolver.py`.
  - Why: these functions resolve a provider's JSONL transcript.
- ACE tmux helpers:
  - What changes: in `src/sase/main/ace_tmux_session.py`,
    `resolve_or_create_agent_session` → `resolve_or_create_agents_tmux_session` and
    `_create_agent_session_with_bootstrap` →
    `_create_agents_tmux_session_with_bootstrap`. Also rename `_AGENTS_SESSION` →
    `_AGENTS_TMUX_SESSION` in `ace_tmux_support.py`.
  - Where else: `ace_tmux.py` and the tests under `tests/main/`.
- Prose that uses "agent session" for one agent run: change "SASE-launched agent
  session(s)" to "SASE-launched agent run(s)" in these files:
  - `docs/commit_workflows.md`, `docs/configuration.md`, `docs/llms.md`,
    `docs/workspace.md`
  - `docs/blog/posts/commit-workflows-plugins.md` and
    `docs/images/commit-workflow-infographic.critique.md`
  - `docs/acknowledgements.md`: change "a single agent session's context window" to "a
    single agent's context window".
- Prose that uses it for a workflow lifetime: "persist for the entire agent session"
  becomes "the entire workflow run" in `src/sase/xprompt/workflow_models.py`,
  `src/sase/xprompts/workflow.schema.json`, and `docs/workflow_spec.md`.
- Fold-scope wording that says "session":
  - `docs/ace.md`: the `z1`–`z3` "clan or regular-agent session scope" row and "the same
    session scope carries over"
  - `docs/configuration.md`: `# family 1-2; clan/session 1-3; tribe 1-4` and the
    matching prose near its `z1` docs
  - the same comment in `src/sase/default_config.yml`

  Use fold-scope wording such as "clan or single-agent scope" and "clan/agent 1-3".
  Later phases rename "family" itself.

- Leave alone: TUI sessions (`src/sase/sessions/`, `sase proc --session`), provider
  `session_id`, and the tmux session _name_ `sase_ace_agents`.
- Exit: `git grep -inE 'agent[ _-]session|AgentSession'` finds nothing in sase, and
  `sase tool run check` passes.

## sase-core additive rename

Repo: sase-core, a non-breaking `feat:` change. After it lands, a sase tree pinned to
the previous core must still work.

- Rename the concept in the Rust internals:
  - modules: `agent_family.rs` → `agent_session.rs` and `fleet_family.rs` →
    `fleet_agent_session.rs`
  - every family-concept type, fn, const, enum variant, and internal helper, following
    the identifier rules. That covers:
    - the family-resolution wires and `AGENT_FAMILY_RESOLUTION_WIRE_SCHEMA_VERSION`
    - `AgentFamilyNameWire`, `InvalidFamilyName`, `historical_family_scope`,
      `AgentContainerKind::Family`, and `FamilyContainerMemberMismatch`
    - the `FamilyShell*Wire` types, dismissal lineage, the fleet family-role and
      promotion wires, and `AgentStatsRuntimeGroupByWire::Family`
    - `DirectiveValueRole::Family`, `ConvertFamily`, `RESERVATION_KIND_FAMILY`,
      `CONTAINER_KIND_FAMILY`, and the runner-slot, hold, and ownership helpers
  - the root `pub use` list and the `core_*` aliases in `sase_core_py/src/prelude.rs`
- Keep serialized output byte-identical. Every renamed serialized field or variant gets
  `#[serde(rename = "<legacy>", alias = "<new>")]`: it emits the legacy spelling and
  accepts both. Hand-read JSON (scanner `data.get("agent_family"…)`,
  `family_shell_from_object`, and hold normalization) reads the new key first and the
  legacy key second.
- pyo3 bindings: register the new names and keep the four legacy names registered for
  the same functions until `core-contract`. Returned dict keys stay legacy in this
  phase.

  | New name                                                         | Legacy name kept until `core-contract`                    |
  | ---------------------------------------------------------------- | --------------------------------------------------------- |
  | `parse_agent_session_name`                                       | `parse_agent_family_name`                                 |
  | `resolve_agent_session_parent`                                   | `resolve_agent_family_parent`                             |
  | `reconcile_agent_artifact_index_dismissed_agent_session_members` | `reconcile_agent_artifact_index_dismissed_family_members` |
  | `fleet_followed_batch_agent_session_promotions`                  | `fleet_followed_batch_family_promotions`                  |

- Directives and LSP: `%id` parsing (`agent_launch/identity.rs`) accepts both `session=`
  and `family=`. `directive_contract()` adds `session` as an additional keyword,
  additively, so an older sase still reads the contract. Completion, snippets, and hover
  offer only `session=`. The existing test that `%family`/`%f` are unknown directives
  stays.
- Parsing that must accept the new spellings; emitted values stay legacy in this phase:
  - agent-query `session:`/`kind:session`, if the Rust query engine owns those fields
  - artifact-link paths under `sessions/`
  - fleet logical keys with a `session:` segment and `session-<hex>` fallback ids
- Reserve the agent names `session` and `sessions` as well as `family` and `families`,
  which stay reserved for legacy link paths. Check first that no existing agent uses
  them; if one does, block only new names.
- Do not change any `*_SCHEMA_VERSION`, SQLite column, or golden contract here. Those
  belong to `core-contract`.
- Tests:
  - rename family-named tests and helpers
  - add tests that every renamed input accepts both spellings and that serialized output
    is unchanged (the `python_wire_parity.rs` key order and `fleet_api_v1.json` stay as
    they are)
- Do not edit versions or CHANGELOGs.
- Verify:
  - `sase tool run check` in sase-core
  - a sase workspace built against this core still passes `sase tool run check` with no
    sase changes

## Python persistence and wire cutover

Repo: sase.

- Pin and bindings:
  - Bump `sase-core-revision.txt` to the landed `core-expand` commit with
    `just ratchet-core-revision`.
  - Call the new binding names.
  - Update `tools/validate_sase_core_rs` (the `agent_family*`, `family_shell_*`, and
    `family_id` fields) and `demos/scripts/seed_sase_ace_demo`.
- Canonical keys:
  - `src/sase/plan_chain.py` owns `AGENT_SESSION_KEY = "agent_session"`,
    `AGENT_SESSION_ROLE_KEY`, the parallel marker, and `AGENT_SESSION_SEPARATOR = "--"`.
    It also owns `LEGACY_AGENT_FAMILY_*` constants, used only by one shared accessor
    that reads the new key, then the legacy key.
  - Route every reader of these keys through that accessor. There are about 40,
    including:
    - names lookup, `wait_watch`, and monitor `start_lane`
    - `gate_shell/transaction.py`, `plan_shell/create.py`, and `bead/epic_launch.py`
    - `core/agent_hold_facade.py`, `_restart_planning.py`, and
      `agents_sync/inventory.py`
  - Writers emit only the new keys and drop legacy keys when they rewrite a record:
    - `axe/run_agent_directive_metadata.py` and `axe/run_agent_helpers_artifacts.py`
    - `agent/_family_promotion.py`
    - the launch-request planning, continuation, and follow-up paths
- Python wire mirrors:
  - rename fields in `core/agent_scan_wire_markers.py` and
    `agent_scan_wire_conversion.py`
  - rename `agent_scan_wire_family_shell.py` → `agent_scan_wire_agent_session_shell.py`
  - also: the launch, cleanup, group-archive, and runner-slot records, hold identity,
    gate hand-off evidence, monitor follow-up kwargs, and the fleet nodes, rows,
    promotion, and follow store

  Each mirror hydrates from either spelling, because core still emits legacy spellings
  until `core-contract`. Payloads sent to core use the new spellings, which
  `core-expand` accepts.

- The `Agent` dataclass fields (`src/sase/ace/tui/models/_agent_state.py`) become
  `agent_session*`. Update every reference mechanically, but leave ACE-owned module,
  label, and row names to `ace-cutover`. Dismissed bundles load the old field names.
- Durable Python-owned JSON:
  - saved dismissed groups (`canonical_global_agent_session`)
  - `wait_for_fork_sources` and chat-fork source kinds (`session`)
  - gate descriptors and `gate_next_fork` (`session`; a stored `family` still loads)
  - notification `action_data`: add `agent_session_root_suffix` to the existing
    multi-key lookup
  - ops revert requests (`agent_session_base`, scope `session`)
  - launch-request `agent_session_type` and its context keys
  - stats `runtime_group_by`
  - `follows.json` logical keys: parse both segments
- Name registry: `reservation_kind`/`container_kind` becomes `session`. Bump its schema
  from 2 to 3 so old caches rebuild.
- Tests:
  - legacy-input tests for every durable surface above, loading a realistic pre-rename
    file
  - write tests proving no legacy key is emitted
  - core round-trip tests through the new binding names
  - legacy-shaped fixtures that prove migration are kept and named as legacy; new-shape
    fixtures are added next to them
- Exit: `sase tool run check`.

## Runtime, syntax, and CLI cutover

Repo: sase. Covers everything outside `src/sase/ace/` and the matching tests, plus the
shared agent query dialect. Touch ACE files only to follow renamed imports.

- Rename these src modules:
  - `agent/`: `_family_attach_{candidates,directives,launch,resolution,types}.py`,
    `_family_promotion.py`, and `family_attach.py`
  - `agent_family_plan_preview.py`, `agents/catalog/_family.py`,
    `history/chat_fork/family.py`, and `scripts/_agent_chat_from_name_family.py`

  Rename every remaining family-concept identifier, comment, and message in these
  packages:
  - `agent`, `axe`, `names`, `monitor`, `gate_shell`, `plan_shell`, `question_shell`,
    `bead`, `wait_watch`
  - `runner_slots`, `dispatch`, `integrations`, `agents`, `stats`, `ops`, `sdd`,
    `llm_provider`, `xprompt`, `notification_gates`, `sudo`
  - `relaunch_prompt.py`, `sase_agent.py` (except the sidecar `families/` URL, which
    `session-pages` owns), and `config/_settings_runner.py`

  Do not keep internal aliases just to shrink the diff. sase-telegram's import is fixed
  in the `telegram` phase.

- Canonical user syntax:
  - `%id(<suffix>, session=<parent>)`: directive collection and editing, completion
    tokens, and the error text in `names/_registry.py` and `relaunch_prompt.py`
  - `sase gate create -f/--next-fork {session,shell,none}` and gate spec
    `"fork": "session"`
  - agent query `session:`/`kind:session`: `query_profile/profiles/_agents_shared.py`,
    `agents/catalog/_query.py`, `agent_live_query*.py`, and the `parser_agent_search.py`
    example
  - `SASE_AGENT_SESSION_ATTACH`, whose JSON payload keys `agent_session_role`,
    `parent_agent_session_member_name`, and `parent_agent_session_role_suffix` are read
    with the existing legacy defaults
- Sunset flag: create it only with
  `sase flag new legacy_agent_family_syntax -k sunset --when-enabled ... --when-disabled ... --remove-when ...`,
  using these three sentences:
  - **when enabled:** SASE silently accepts the retired agent-family spellings as
    aliases of their agent-session replacements: `%id(..., family=...)`, the
    `family:`/`kind:family` agent queries, `--next-fork family`, gate
    `"fork": "family"`, and `SASE_AGENT_FAMILY_ATTACH`.
  - **when disabled:** SASE rejects those spellings with an error naming the
    agent-session replacement and ignores the legacy environment variable.
  - **remove when:** no maintained prompt, xprompt, skill, saved query, gate spec, or
    config in the sase-org repos or chezmoi still uses a retired agent-family spelling,
    and a sase release with agent-session syntax has shipped.

  The Rust `family` directive alias from `core-expand` stays. Python enforces the flag
  after parsing. Test both flag states. Removing the flag later also removes the Rust
  alias; say so in the registry entry.

- CLI:
  - Help text in `parser_agent_lifecycle.py`, `parser_agent_hold.py`,
    `parser_monitor.py`, `parser_gate.py`, `parser_pipe.py`, `pipe_handler.py`, and
    `agents/_restart_render.py`.
  - JSON output: `sase agent list -j` (`agent_session`, `agent_session_role`),
    `sase agent search -j` (`agent_session`), `sase agent index`
    (`dismissed_agent_session_*`), `sase agent wait -j` (target kind `session`), and the
    `sase editor` bridge (`kind: "session"`, `agent_session*`).
  - Refresh `tests/completion/snapshots/cli_spec.json`.
  - Confirm sase-nvim does not branch on the old editor kinds.
  - These JSON changes are breaking: use a `feat!:` subject with a `BREAKING CHANGE:`
    footer that lists them.
- Skill sources and xprompts:
  - `src/sase/xprompts/skills/` `sase_run`, `sase_gate`, `sase_pipe` (including its
    frontmatter description), `sase_questions`, `sase_monitor`, and
    `sase_agents_status`, plus `src/sase/xprompts/with_feedback.yml`. Use the new
    vocabulary and syntax.
  - Leave the Patch meaning in `sase_patches`.
  - Do not deploy to chezmoi in this phase.
- Tests: rename the family-named test files in scope, for example:
  - `tests/test_dynamic_agent_family_attach_*.py` and
    `tests/test_agent_family_plan_preview.py`
  - `tests/test_agent_chat_from_name_family.py` and
    `tests/test_editor_helper_family_catalog.py`
  - `tests/test_agent_loader_status_override_*family*.py`

  Update the test IDs recorded in the shard-timing and reproducible-flake baseline
  files.

- Exit:
  - Every remaining `famil` hit outside ACE falls into one of four groups: an unrelated
    meaning, a named legacy reader, the flag branch, or the sidecar path left for
    `session-pages`.
  - `sase tool run check` passes.

## ACE agent session surfaces

Repo: sase. Covers `src/sase/ace/**`, `tests/ace/**`, `tests/perf/**`,
`src/sase/default_config.yml`, and `src/sase/config/sase.schema.json`.

- Rename modules:
  - `actions/agents/_loading_family_previews.py`
  - `models/`: `_agent_imported_family`, `_agent_parallel_family`,
    `_agent_status_family{,_core,_planner,_policy}`, `_family_shell_membership`,
    `agent_family_members`, and `agent_family_preview_cache`
  - `widgets/prompt_panel/_agent_display_family{,_render}`

  Also rename their classes, widget/CSS ids, and row kinds (`agents_list.py`,
  `agents_navigation.py`, `agents_revival.py`, `query_rows.py`), the grouping mode
  `by_family` → `by_session`, and the Agents-pane relation
  `family`/`agent_family_container` → `session`/`agent_session_container`. Keep the
  Patch `RelationKind.FAMILY`.

- Visible copy:
  - `FAMILY SHELLS` → `SESSION SHELLS`, including the "… also listed under" rows
  - the identity header `FAMILY` → `SESSION`
  - the command-palette alias "collapse family" → "collapse session"
  - `bindings.py`, `keymaps/metadata.py`, the `help_modal/agents_bindings.py` text, the
    family fold-scope labels, and the family wording in the `default_config.yml`
    comments and `sase.schema.json` descriptions
- Performance contract: renames only. Add no filesystem work, awaits, subprocesses, or
  full rebuilds to navigation or render handlers.
  - Trace names: `agents.family_plan_preview_warmup` →
    `agents.agent_session_plan_preview_warmup`, and task `sase-agents-family-previews` →
    `sase-agents-session-previews`.
  - Perf scenarios: `family_container_press` → `session_container_press` and
    `family_container_unfolded_press` → `session_container_unfolded_press`, in
    `tests/perf/tui_trace/view_hints.py` and
    `tests/perf/baselines/view_hints_baseline.json`.
  - Run the existing j/k navigation benchmark to confirm no regression.
- PNG snapshots:
  - Rename the family-named snapshot tests, fixtures, and goldens (26 goldens), for
    example `test_ace_png_snapshots_agents_families.py` and
    `_ace_agents_png_snapshot_family_fixtures.py`.
  - Re-baseline only goldens whose pixels change because the copy changed. Run
    `just fix-tui-screenshots` through `/sase_monitor`, inspect every creation, removal,
    and update group, and remove stale goldens only after a full run.
- Leave the model-family usage indicator, the Patch relation family, and generic
  "family" wording (colour, palette, marker) alone.
- Exit: every remaining `famil` hit in ACE is classified, and `sase tool run check`
  passes.

## Documentation and memory

Repo: sase. The user's request authorizes the memory edits. Follow `/sase_memory_write`
for them.

- Rename `docs/agent_families.md` → `docs/agent_sessions.md`:
  - Retitle it "Agent Clans, Sessions, and Tribes", with one sentence saying agent
    sessions were formerly called agent families.
  - Rename anchors: `#sequential-agent-families` → `#sequential-agent-sessions` and
    `#agent-initiated-family-launches` → `#agent-initiated-session-launches`.
  - Change the `mkdocs.yml` nav entry to "Agent Sessions".
  - Fix inbound links in `ace.md`, `monitors.md`, and `xprompt.md`.
  - Update the doc path in `tests/test_agent_tribe_terminology.py`.
- Rewrite every concept mention in `docs/`, including all blog posts:
  - heavy files: `ace.md`, `xprompt.md`, `monitors.md`, `agents_sidecar.md` (concept
    text only), `configuration.md`, `perf_runbook.md`, `cli.md`, `notifications.md`,
    `editor.md`
  - also the medium and small files the sweep turns up
  - new syntax and names: `session=`, `session:`/`kind:session`, `--next-fork session`,
    the JSON keys, `SESSION SHELLS`, and `<session>--<suffix>`

  Leave the sidecar `families/` layout and footer-link text to `session-pages`. Keep the
  unrelated meanings listed above.

- Memory:
  - Replace glossary strand `glossary/agent-family.md` with
    `glossary/sase-agent-session.md`:
    - keyword `Sase Agent Session`, alias `agent session`
    - definition in session vocabulary, including `<session>--<suffix>` and `session=`
    - one clause saying it was formerly called an agent family
    - one clause separating it from TUI sessions, provider transcript sessions
      (`session_id`), and question sessions
  - Update these strands: `sase-agent` ("an agent session or a single agent that does
    not belong to a session"), `agent-node`, `agent-relation-jump-target`
    (`SESSION SHELLS`), `agent-tribe`, `gate-shell` (`<session>--gate`,
    session-attached), `sase-monitor` (`<session>--mon`), `proc-shell`, `sase-gate`, and
    any clan, hood, or neighbor strand that mentions families.
  - Update the `glossary.md` roster (if the generator does not own it) and the `%id` row
    in `xprompts.md`.
  - Leave the decision records and the unrelated `symvision.md` and
    `corpus-before-mechanism` wording alone.
  - Run `sase memory init`, then review the regenerated `AGENTS.md`, `CLAUDE.md`,
    `GEMINI.md`, `QWEN.md`, `OPENCODE.md`, and memory README.
- Exit: `sase tool run check`, including the docs lints.

## sase-telegram cutover

Repo: sase-telegram.

- Rename `/show <agent|clan|family|@tribe>` → `/show <agent|clan|session|@tribe>`. That
  covers:
  - `ShowKind` `"family"` → `"session"`
  - `format_family_show` → `format_agent_session_show`
  - the "Families" index and "Family" row wording
  - the help text in `inbound_handlers/commands.py` and `agent_show.py`
  - `README.md` and `docs/inbound.md`
- Import `find_agent_session` and read the `agent_session*` attributes. telegram's
  `sase>=` floor predates the rename, so fall back to the legacy names only through one
  clearly named helper. Add a test that proves the lookup works against the installed
  sase, so an `ImportError` can no longer silently disable `/show`.
- Update these tests:
  - `test_show_entities.py` and `test_show_format.py`
  - `test_inbound.py`: callback `show:familykey:open` → `show:sessionkey:open`
  - `test_gate_shell_settlement.py`: the metadata key
  - `test_custom_gates.py`: `family=reviewer` → `session=reviewer`
- Leave `pdf_style.css` font-family, "unfamiliar", and the CHANGELOG alone.
- Exit: `sase tool run check` in sase-telegram.

## sase-core contract flip

Repo: sase-core. A breaking `feat!:` change with a `BREAKING CHANGE:` footer. Start only
after all sase and telegram work has landed.

- Serialized output:
  - Remove the legacy `rename` pins so every renamed field and variant serializes its
    new name.
  - Keep `alias = "<legacy>"` only where durable pre-rename data or the sunset flag can
    still produce the old spelling. Mark each one with a `legacy agent-family spelling`
    comment.
  - Drop aliases on purely in-process request wires.
- Remove the four legacy binding names.
- Directive contract: `session` becomes the canonical `%id` keyword. `family` stays an
  accepted, hidden alias until the `legacy_agent_family_syntax` flag is removed.
- Emitted values:
  - link paths: `sessions/<global>.md#member-<role>`
  - fleet: `session:` logical-key segments and `session-<hex>` fallback ids
  - runner claim keys, stats group-by, container, and link-target kinds: `session`
  - parsers keep accepting the legacy forms
- Artifact index: rename the column and index. Bump
  `AGENT_ARTIFACT_INDEX_SCHEMA_VERSION` from 31 to 32 through the existing rebuild path.
- Schema versions: bump each version whose emitted shape changed, for example the agent
  scan (9 → 10), fleet contract, runner capacity, hold, gate follow-up, launch,
  ownership batch, cleanup, saved-group archive, stats, editor, relationship, and
  session resolution. Also bump the fleet protocol version.
- Goldens and parity: regenerate `fleet_api_v1.json` with `UPDATE_FLEET_CONTRACT=1`, and
  update the `python_wire_parity.rs` key order.
- Before landing, run current sase master against a locally built copy of this core:
  - Python readers must accept both spellings.
  - Mirror-version mismatches must degrade through fallback or rebuild, not crash.
  - List every mirror constant that `session-pages` must bump.
- Exit: `sase tool run check` in sase-core.

## Pin bump and agents sidecar session pages

Repo: sase.

- Core pin and mirrors:
  - Bump `sase-core-revision.txt` to the `core-contract` commit.
  - Update the Python `*_WIRE_SCHEMA_VERSION` and index-version mirrors that
    `core-contract` listed.
  - Update fixtures and goldens that capture core output keys.
  - Keep the durable-data legacy readers.
- Agents sidecar publishing (`src/sase/agents_sync/`):
  - Publish session pages at `sessions/<global>.md`. Rename `rendering_family_page.py` →
    `rendering_agent_session_page.py`, and update publication planning, rendering,
    kinship, validation, and git sync (`sessions/.gitkeep`).
  - v2 manifest and snapshot: container kind `session`, `agent_session_count`, allowed
    metadata fields, and `agent_sessions_published`. Readers accept the legacy values.
  - The index page gets a "Sessions" column and count.
  - Replace each existing `families/<global>.md` page with a permanent redirect stub
    that links to `../sessions/<global>.md`; never delete these pages. Commit footers in
    immutable git history link to them.
- Link generation:
  - `sase_agent.py` and `sdd/hosted_links.py` generate `sessions/` URLs.
  - `inventory_history.py` parses both the `families` and `sessions` path segments.
  - `sdd/_init_files.py` scaffolds `sessions/.gitkeep`.
- Templates and docs:
  - `src/sase/sdd/templates/sidecar-agents-README.md` and
    `src/sase/sdd/assets/agents-directory-map.png.prompt.md` describe the new layout and
    the legacy stubs.
  - Also update `docs/agents_sidecar.md` and the footer-link paragraph in
    `docs/commit_workflows.md`.
- Goldens:
  - Rename `tests/agents_sync/goldens/{deep-family,rootless-family}.md` →
    `{deep-session,rootless-session}.md`, and refresh the other goldens that contain
    `families/` links.
  - Keep a legacy `tests/sdd/fixtures/referenced_by_v1` fixture that proves old footers
    still parse.
- Exit: `sase tool run check`.

## Cross-repo audit, guardrail, and deploy

Repos: sase, sase-core, sase-telegram, sase-github, sase-nvim, sase-research-artifacts,
and chezmoi.

- Add `tests/test_agent_session_terminology.py`, modeled on
  `tests/test_agent_tribe_terminology.py`.
  - It fails on agent-family identifiers in `src/`: `agent_family`, `AgentFamily`,
    `AGENT_FAMILY`, `family_shell`, `FamilyShell`, `family_attach`, `family_id`,
    `family_name`, `family_role`, and `find_agent_family`.
  - It also fails on stale phrases in current docs, skill sources, and memory: "agent
    family", "agent families", `FAMILY SHELLS`, `family=`, `kind:family`,
    `--next-fork family`, `<family>--`, and `families/`.
  - A commented allowlist names the legacy-reader modules, the flag branch, and each
    unrelated meaning.
- Run case-sensitive and case-insensitive `rg 'famil'` sweeps in every repo listed
  above. Classify every hit as one of: an unrelated meaning, deliberately unchanged
  history, a named legacy reader, alias, or fixture, or the sunset flag.
  - Fix small stragglers in place.
  - File substantive leftovers through `/sase_new_task`.
  - Check that `rg -i 'agent session'` hits mean only the new concept.
- chezmoi (via `/sase_repo`):
  - From the landed sase tree, regenerate the provider skill copies by following
    `generated_skills.md` (`sase skill init --force`, then `chezmoi apply` if it was
    skipped).
  - In `home/dot_config/sase/sase.yml`, replace the ACE snippet `af: agent family` with
    `as: agent session`, keeping the keep-sorted order.
  - Leave the CSS font-family snippets, `syslog.vim`, and the newsboat cache alone.
- Confirm the `legacy_agent_family_syntax` flag bead and registry entry exist with
  both-state tests.
- The epic is done when:
  - A fresh agent can be launched with `%id(x, session=parent)`, then listed, queried
    with `session:`, gated with `--next-fork session`, monitored, forked, waited on,
    dismissed, revived, and published using only agent-session contracts.
  - Pre-rename agents, bundles, groups, follows, and sidecar links still load.
  - Rust and Python agree.
  - `sase tool run check` passes in every changed repo.
