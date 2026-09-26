---
tier: epic
title: Rename sase shell to sase turn
goal: 'The concept formerly called a sase shell is named a sase turn on every current
  surface in sase, sase-core, sase-telegram, sase-github, sase-research-artifacts,
  and chezmoi: code, wire contracts, persisted output, CLI, gate specs, config, the
  TUI, skills, docs, and memory. Agent, gate, and monitor shells become agent, gate,
  and monitor turns, and stand-alone proc shells become named procs. Pre-rename data
  still loads, retired user syntax keeps working behind a sunset flag, and unrelated
  meanings of "shell" (Unix shells, shell completion, UI chrome) are unchanged.'
phases:
- id: core-expand
  title: sase-core additive rename
  depends_on: []
  size: large
  description: 'core-expand: non-breaking sase-core change. Rename the Rust internals
    to turn and named-proc vocabulary and register the new pyo3 binding names next
    to the old ones. Inputs accept old and new spellings; serialized output stays
    byte-identical.'
- id: wire-cutover
  title: Python persistence and wire cutover
  depends_on:
  - core-expand
  size: large
  description: 'wire-cutover: bump the core pin and switch sase to the new binding
    names. Rename the Python wire mirrors and every durable key and value (agent meta,
    plan-gate meta and files, gate bundles, proc rows, runner-slot records, dismissed
    procs): new data is written only with turn/named-proc spellings, and readers accept
    both.'
- id: runtime-cutover
  title: Runtime, syntax, and CLI cutover
  depends_on:
  - wire-cutover
  size: large
  description: 'runtime-cutover: rename every non-TUI package, module, and identifier
    (gate_shell, shells, plan_shell, question_shell, ...). Make the turn CLI flags,
    gate-spec spellings, and config key canonical behind the legacy_sase_shell_syntax
    sunset flag, and update CLI help and JSON output, the scheduler job, telemetry,
    xprompts, and skill sources.'
- id: tui-cutover
  title: TUI turn surfaces
  depends_on:
  - runtime-cutover
  size: large
  description: 'tui-cutover: rename TUI modules, row kinds, section ids, and visible
    copy (SESSION TURNS, AGENT TURN, GATE TURN, MONITOR TURN, NAMED PROC, the 0-9
    turn footer, help legend, modals, notifications) and re-baseline the PNG goldens,
    with no change to the performance contract.'
- id: docs-memory
  title: Documentation and memory
  depends_on:
  - runtime-cutover
  size: medium
  description: 'docs-memory: redeploy the landed skill sources, rewrite every concept
    mention in docs/ (including headings and anchors), replace the Sase Shell, Agent
    Shell, Gate Shell, and Proc Shell glossary strands with Sase Turn, Agent Turn,
    Gate Turn, and Named Proc, update related strands and notes, then run sase memory
    init.'
- id: telegram
  title: sase-telegram cutover
  depends_on:
  - runtime-cutover
  size: small
  description: 'telegram: move sase-telegram tests and docstrings to the renamed sase
    gate-turn APIs through one named legacy-fallback import helper.'
- id: contract-flip
  title: sase-core contract flip
  depends_on:
  - tui-cutover
  - telegram
  size: medium
  description: 'contract-flip: breaking feat! sase-core change. Serialize the new
    key and value names, drop the legacy binding names, rename the gate_turn_id index
    column, bump the changed schema versions and the fleet protocol, and prove current
    sase master still passes against the new core before landing.'
- id: pin-bump
  title: Core pin bump and mirrors
  depends_on:
  - contract-flip
  size: medium
  description: 'pin-bump: move sase-core-revision.txt to the contract commit, update
    the Python schema-version mirrors and the fixtures and goldens that capture core
    output, and keep every durable legacy reader.'
- id: audit-deploy
  title: Cross-repo audit, guardrail, and deploy
  depends_on:
  - pin-bump
  - docs-memory
  size: medium
  description: 'audit-deploy: add the sase-turn terminology guard test, sweep and
    classify every remaining shell hit in all repos, fix tools/require_tool_run wording,
    note renamed identifiers on open beads, and redeploy chezmoi skills from the landed
    tree.'
proposed_by: bbugyi200.athena.0ss
create_time: 2026-09-26 00:15:02
status: wip
bead_id: sase-1ab
---

- **PROMPT:** [prompts/202609/sase_turn_rename.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/sase_turn_rename.md)
- **BEAD:** [sase-1ab](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ab/README.md)

# Plan: Rename sase shell to sase turn

## Context

A **sase turn** (formerly a **sase shell**) is one executing member of a sase agent: an
**agent turn** (one provider/LLM run), a **monitor turn** (a sase monitor running one
long command), or a **gate turn** (a durable command-backed user decision). A sase agent
is an ordered sequence of sase turns, named `<session>--<suffix>` inside an agent
session. Read the three kinds like a chat transcript: an agent turn is the assistant
speaking, a monitor turn is a tool result arriving, and a gate turn is the human's move.

"Turn" is already SASE's word for this unit: `decisions:single-turn-agents` says a SASE
agent run is one provider turn, the gate and monitor glossary entries say they "kill
that agent's turn", and the code has `turn_nonce` and `SASE_FINAL_TURN_NONCE`. "Shell"
collides with the Unix shell everywhere in a CLI product.

Scale of the old term (session-member meaning, not Unix meaning):

- sase: about 228 source files outside the TUI and 192 test files outside `tests/ace`,
  about 140 TUI source files and 104 `tests/ace` files, and about 150–200 Agents-tab PNG
  goldens that render `AGENT SHELL`, `SESSION SHELLS`, `N shells`, or `0-9 shell`.
- sase-core: about 650 occurrences in 68 `.rs` files, no file named after the concept.
- sase-telegram: 5 files. chezmoi: 35 generated skill copies (5 skills × 7 providers).
  sase-github and sase-research-artifacts: one `tools/require_tool_run` line each.
  sase-nvim: none.

Precedents and inputs:

- `plan:202609/agent_session_rename.md` (epic sase-17m, agent family → agent session) is
  the model. This plan reuses its expand/contract, legacy-reader, and sunset-flag
  machinery. Its landing notes show two traps to avoid: the contract flip left master
  red until hotfixes because `src/sase/core/health.py` and `tools/validate_sase_core_rs`
  hard-coded schema literals, and tests went stale when phases updated code without the
  matching expectations.
- `research:202609/sase_terminology_renames/sase_terminology_renames.md` ranks this as
  the best of the recent renames and supplies caveats adopted below: the named-proc
  meaning of "shell" gets proc vocabulary, not "turn"; use "monitor turn", never "proc
  turn"; never show a provider's `num_turns` as turns; grep with word boundaries. Its
  advice to fold turn spellings into sase-17m.8 came too late: `agent_session_shell` is
  now the canonical core key and `family_shell` its legacy alias, so that key's readers
  will accept three spellings.

The shared policy below binds every phase.

### Vocabulary

| Old                                                                                 | New                                                                            |
| ----------------------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| sase shell / shell (this concept), sase shells                                      | sase turn / turn, sase turns                                                   |
| agent shell                                                                         | agent turn                                                                     |
| gate shell; plan gate shell; question gate shell                                    | gate turn; plan gate turn; question gate turn                                  |
| session-attached proc shell; monitor shell                                          | monitor turn                                                                   |
| stand-alone / named proc shell (xprompt `%proc` unit, `sase proc run -N`)           | named proc                                                                     |
| session shell(s), `SESSION SHELLS`, "N shells", "one-shell agent", historical shell | session turn(s), `SESSION TURNS`, "N turns", "one-turn agent", historical turn |
| shared shell substrate (`sase.shells`)                                              | shared turn substrate (`sase.turns`)                                           |
| TUI kind headers `AGENT SHELL`, `GATE`, `MONITOR`, `PROC SHELL`                     | `AGENT TURN`, `GATE TURN`, `MONITOR TURN`, `NAMED PROC`                        |
| "Kill Proc Shell", footer `0-9 shell`, "No shell N"                                 | "Kill Named Proc", footer `0-9 turn`, "No turn N"                              |

### Identifier rules

- Session-member concept: replace `shell` with `turn`, always qualified:
  - `gate_shell` → `gate_turn`, `GateShell*` → `GateTurn*`, `GATE_SHELL_*` →
    `GATE_TURN_*` (for example `GateShellRecord`, `create_gate_shell`,
    `settle_gate_shell`, `find_gate_shell_by_gate_id`, `GATE_SHELL_NEXT_FORKS`)
  - `agent_session_shell` → `agent_session_turn`, `AgentSessionShell*Wire` →
    `AgentSessionTurn*Wire`, `ConcreteAgentSessionShellKind` →
    `ConcreteAgentSessionTurnKind`
  - `monitor_shell`, `done_shell`, `meta_shell` → `monitor_turn`, `done_turn`,
    `meta_turn`; `sase_agent_ref_for_shell` → `sase_agent_ref_for_turn`
  - shared substrate names: `ShellStatusPair`, `ShellSettlementConfig`,
    `ShellHandoffError`, `ShellIdSpec`, `allocate_shell_suffix`, `shell_state*`,
    `shell_status_*`, `touch_shell_refresh_pulse` → the `Turn`/`turn` forms
  - packages: `gate_shell` → `gate_turn`, `shells` → `turns`, `plan_shell` →
    `plan_gate_turn`, `question_shell` → `question_gate_turn`
- Never introduce a bare `turn`/`turns` identifier or JSON key where it could be read as
  a provider or conversation turn; qualify it (`sase_turn`, `agent_session_turn`,
  `gate_turn`, `turn_kind` on a member record). These existing names keep their meaning
  and are not renamed: `turn_nonce`, `SASE_FINAL_TURN_NONCE`,
  `SASE_FINALIZER_OWNED_TURN`, `finalizers/owned_turn.py`, provider `*Turn*` types,
  `num_turns` (never displayed as "turns"; call it "model round trips" if shown), and
  chat-history `parse_chat_turns`/`extract_previous_conversation_turns`.
- Named-proc meaning (the proc store's name for any named proc, including a monitor
  turn's proc, a gate turn's execution proc, and service procs):
  - proc record `shell_name` → `proc_name`; proc record `shell_kind` → `proc_role`
    (values `proc`, `gate`, `service` stay; `Proc.kind` already means
    command/tui/detached)
  - lifecycle and default origin `proc-shell` → `named-proc`; concurrency-key prefix
    `shell:` → `named-proc:`; `PROC_LIFECYCLE_PROC_SHELL` → `PROC_LIFECYCLE_NAMED_PROC`
  - `ProcShellNameError` → `NamedProcNameError`, `qualify_proc_shell_name` →
    `qualify_named_proc_name`, `named_proc_shell_concurrency_key` →
    `named_proc_concurrency_key`, `validate_standalone_proc_shell_name` →
    `validate_standalone_named_proc_name`
  - `AgentType.PROC_SHELL` (`"proc-shell"`) → `AgentType.NAMED_PROC` (`"named-proc"`),
    `is_proc_shell` → `is_named_proc`, hold candidate `proc_shell` → `named_proc`
- Member kind on agent metadata: `shell_kind` → `turn_kind`, with values `gate` and
  `monitor`. The legacy value `proc` reads as `monitor`.
- Gate specs: the `shell` block → `turn` block, `"fork": "shell"` → `"turn"`,
  `continuation_mode: "gate_shell"` → `"gate_turn"`, `shell_row_managed` →
  `turn_row_managed`, `shell_backed` → `turn_backed`.
- Where "agent shell" means the Unix environment of an agent process (for example
  `tools/require_tool_run`, `README.md`, `docs/init.md`, `docs/tool.md`,
  `main/parser_init.py`, `main/parser_service.py`, `service/platform.py`, and
  `sase/memory/lint_and_test.md` "agent shells export `CI=true`"), reword it to "SASE
  agent process". It becomes neither "shell" nor "turn".
- Rename comments, docstrings, log and error messages, test names, and fixtures along
  with the code. New commit scopes say `turn` or `gate-turn`, never `gate-shell`, and
  new phase-facing text says "TUI", not "ACE".

### Meanings of "shell" that must not change

- Unix shells and process execution: `sase.core.shell`/`run_shell_command`,
  `shell=True`, `$SHELL`, `DetectedShell`, `SUPPORTED_SHELLS`, `detect_shell`, login
  shells (`dispatch/ssh_login_shell.py`, `remote_login_shell_*`), `oneshot_shell_argv`,
  `create_subprocess_shell`, custom-mode `shell:` keys, the TUI's `!` background
  commands and "Enter shell command…", sudo `shell: bool` and the `"shell"` risk badge,
  `is_shell_whitespace`, `shell_join`, `long_shell_command`, `_SHELL_LIKE_TOOLS`, the
  Codex/Muse `shell` tool and `muse_synchronous_shell`, VHS `Set Shell`, and CI
  `shell: pwsh`.
- Shell completion: `sase completion`, `RefreshShellOutcome`, `ShellInstallStatus`,
  `CannotDetectShell`, `docs/completion.md`, and chezmoi
  `tests/bash/sase_completion_test.sh`.
- UI chrome called a shell: the Artifacts-pane frame (`ArtifactsShellState`,
  `widgets/artifacts/shell.py`, `build_shell_scope`, `PaneCapability.SHELL`,
  `.artifacts-shell-*`, `docs/artifacts_pane_*`), the `.gate-review-shell` modal frame,
  "loading shell" panel docstrings, and the Command Line "panel shell" tests.

### Deliberately unchanged history

- Generated CHANGELOGs (release-please in sase, release-plz in sase-core). Never
  hand-edit them; breaking-change footers produce the new entries.
- Accepted decision records under `sase/memory/decisions/`, including
  `gates-never-block.md`, whose aliases and body say "gate shell" and whose roster line
  in the generated instruction files keeps saying "a gate shell's follow-up". Records
  are immutable and a terminology rename is not a change of course; the Gate Turn
  strand's "formerly called a gate shell" clause bridges the old word.
- Sidecar archives (plans, beads, research reports, agent prompts and pages) and git
  history.

### Compatibility policy

1. **Durable data.** Readers prefer the new spelling and fall back to every legacy one.
   Writers emit only the new spelling, and a rewrite drops legacy keys. These readers
   are permanent and unconditional (not flag-gated), and they live in explicitly named
   helpers and `LEGACY_*` constants, not scattered literals. Durable surfaces:
   - `agent_meta.json` and `done.json`: `shell_kind` → `turn_kind` (value `proc` →
     `monitor`), the nested `agent_session_turn` object (legacy `agent_session_shell`,
     then `family_shell`), and the `gate_next_fork` value `shell` → `turn`
   - the plan gate's `plan_shell_*` meta keys (about 24, in `plan_shell/create.py`) →
     `plan_gate_turn_*`, and its artifact files `plan_shell_original_prompt.md`,
     `plan_shell_current_prompt.md`, and `plan_shell_qa_rounds.json` →
     `plan_gate_turn_*`; a plan gate can stay pending across the upgrade
   - gate bundles under `~/.sase/interaction_requests/`: the envelope `shell` block,
     `continuation_mode`, `fork`, cancel `source` values (`gate_shell`,
     `gate_shell_cancel`, `gate_shell_reclaim`), stored error codes (`invalid_shell`,
     `gate_shell_failed`, `gate_shell_handoff_failed`, `missing_gate_shell_row`), and
     the gate-intent marker `"source": "create_gate_shell"`. Never rewrite a stored
     bundle: the request hash covers the envelope (`notification_gates/service.py`), so
     verification hashes the stored form while new requests hash the canonical `turn`
     form.
   - `procs.jsonl`: `shell_name`, `shell_kind`, lifecycle and origin `proc-shell`, and
     `shell:` concurrency keys. The duplicate-name conflict check must treat the legacy
     and new keys for one name as equal, so an upgrade can never admit a second live
     proc with the same name.
   - `~/.sase/dismissed_proc_shells.json` → `~/.sase/dismissed_procs.json`. It is shared
     with the TUI's `!` background-command dismissals. Read the new file, fall back to
     the legacy file, write only the new file, and remove the legacy file after the
     first successful write.
   - any persisted `AgentType` value, TUI fold-section id, or fleet `follows.json`
     locator key that contains `proc-shell` or a `shell` component (verify each; add a
     legacy reader only where one is actually stored)
2. **Rebuildable caches.** The artifact-index SQLite `gate_shell_id` column and its
   index are renamed in `contract-flip` with an `AGENT_ARTIFACT_INDEX_SCHEMA_VERSION`
   bump through the existing migration path (pattern: `migrate_agent_session_column_v33`
   in `storage.rs`), which must never move an archive-sized rebuild onto TUI startup or
   the UI thread.
3. **User-authored syntax.** One `sunset` flag, `legacy_sase_shell_syntax`, keeps the
   retired spellings working as silent aliases while callers migrate:
   - `sase gate create --shell`, `--shell-status`, and `--shell-stop-status`
   - `sase gate create --next-fork shell`, gate spec `"fork": "shell"`, a gate spec's
     `"shell"` block, and `"continuation_mode": "gate_shell"`
   - `sase proc list --shell` and `sase proc run --shell`
   - the config key `gate.shell.reclaim_grace_seconds`

   With the flag off, each is rejected with an error that names its replacement (the
   config key is reported the way other unknown config keys are, naming the
   replacement). Help, completion, examples, and output never show a legacy spelling.
   Keep every alias in one module, `src/sase/agent/legacy_sase_shell_syntax.py`, modeled
   on `src/sase/agent/legacy_agent_family_syntax.py`. The existing
   `legacy_agent_family_syntax` aliases (`--next-fork family`, `"fork": "family"`) keep
   working unchanged.

4. **Two-step Rust contract (expand/contract).** Every sase workspace builds
   `sase_core_rs` from the linked sase-core checkout, so one breaking core commit would
   break every concurrent sase workspace. `core-expand` is additive and keeps output
   byte-identical; `contract-flip` flips the output only after every sase and telegram
   reader accepts both spellings, and only after current sase master passes against the
   new core.
5. **Versions.** Bump every versioned wire or persisted schema whose _emitted_ shape
   changes (in `contract-flip`) and the fleet protocol version, so a mixed-version fleet
   fails with the existing clean `incompatible_protocol` error. No Python code may
   hard-code a core schema literal; compare against the imported mirror constants.
6. **Breaking surfaces.** CLI JSON keys, CLI flags, the config key, the scheduler job
   name, and telemetry metric names change. Use a `feat!:` subject with a
   `BREAKING CHANGE:` footer that lists them.
7. **Process.** Open every linked repo with `/sase_repo` and read its `AGENTS.md` first.
   Verify with `sase tool run check` inside each repo you changed; run `just install`
   first in a fresh sase workspace; never run `just check-full` unless explicitly told.
   Read `sase/memory/tui.md` (and the notes it links) before TUI work,
   `generated_skills.md` before skill work, `cli_rules.md` before CLI changes,
   `sase_flags.md` before creating the flag, and `symvision.md` for symvision failures.
   Grep for the new spelling with `git grep -w` or `\bturns?\b`, because `turn` is a
   substring of `return`. Phase workers record `PROPOSED FOLLOW-UP:` notes on their own
   phase bead instead of creating beads.

## sase-core additive rename

Repo: sase-core, a non-breaking `feat:` change. After it lands, a sase tree pinned to
the previous core must still work.

- Rename the concept in the Rust internals, following the identifier rules:
  - agent scan: `AgentSessionShellWire`, `AgentSessionShellMonitorWire`,
    `AgentSessionShellGateWire`, the `agent_session_shell` fields on `AgentMetaWire` and
    `DoneMarkerWire`, `AgentMetaWire.shell_kind` → `turn_kind`, and
    `agent_session_shell_from_object` (`agent_scan/wire.rs`, `agent_scan/scanner.rs`)
  - index: `find_gate_shell_by_gate_id`, `gate_shell_id_from_record`,
    `RecordSummary.gate_shell_id`, and the `LAST_GATE_SHELL_LOOKUP_*` counters
    (`agent_scan/index/`). The SQLite column itself stays until `contract-flip`.
  - `fleet_agent_session.rs`: `ConcreteAgentSessionShellKind` and its classifier
    functions; `fleet_contract/`: `FleetRowKindWire::AgentShell`/`HistoricalShell`,
    `FleetAgentSessionRoleWire::HistoricalShell`, `AgentInstanceLocatorWire.shell_id`;
    `fleet_owner_facts.rs`: `apply_shell_facts` and `shell_start_status`/
    `shell_stop_status`; `fleet_catalog.rs` and `sase_gateway/src/fleet_reads/`
  - `runner_capacity`: `agent_session_shell_kind`/`_id`/`_state`; `agent_hold.rs`:
    `AgentHoldCandidateWire.proc_shell` and its `"proc_shell"` match kind;
    `agent_runtime.rs` member filters
  - procs: `PROC_SHELL_LIFECYCLE`, `ValidationMode::ProcShellWrite`, `is_proc_shell`,
    `ensure_proc_shell`, `default_shell_kind`, and every `shell_name`/`shell_kind` field
    (`procs/store.rs`, `procs/wire.rs`, `agent_launch/wires.rs`,
    `agent_launch/proc_runtime.rs`, `agent_launch/plan_resolution.rs`)
  - the root `pub use` list (`lib.rs`) and the `core_*` aliases in
    `sase_core_py/src/prelude.rs`
- Keep serialized output byte-identical. Every renamed serialized field or variant gets
  `#[serde(rename = "<legacy>", alias = "<new>")]`: it emits the legacy spelling and
  accepts both. `#[serde(deny_unknown_fields)]` wires (runner capacity, hold, fleet)
  need the alias so Python can send the new keys. Hand-read JSON reads the new key
  first, then each legacy key (`agent_session_turn`, `agent_session_shell`,
  `family_shell`; `turn_kind`, `shell_kind`).
- Parsing must accept the new values while emitted values stay legacy in this phase:
  lifecycle and origin `named-proc`, the `named-proc:` concurrency prefix (equal to
  `shell:` for conflict checks), `turn_kind` value `monitor` (equal to `proc`), fleet
  row kinds `agent_turn`/`historical_turn`, role `historical_turn`, a `turn` locator key
  component, and hold match kind `named_proc`.
- pyo3 bindings: register the new names and keep the legacy names registered for the
  same functions until `contract-flip`:

  | New name                              | Legacy name kept until `contract-flip` |
  | ------------------------------------- | -------------------------------------- |
  | `find_gate_turn_by_gate_id`           | `find_gate_shell_by_gate_id`           |
  | `validate_standalone_named_proc_name` | `validate_standalone_proc_shell_name`  |

- Editor and LSP text: the `%wait(proc=${1:proc-id-or-shell-name})` snippet and the
  "Wait for a proc ID or shell name" metadata (`editor/wire.rs`,
  `editor/directive/metadata.rs`) say proc name instead.
- Do not change any `*_SCHEMA_VERSION`, SQLite column, golden contract, or CHANGELOG.
- Tests: rename shell-named tests and helpers (for example
  `find_gate_shell_by_gate_id_*`, `write_gate_shell_artifact`); add tests that every
  renamed input accepts both spellings and that serialized output is unchanged
  (`python_wire_parity.rs` and `fleet_api_v1.json` stay as they are).
- Exit: `sase tool run check` in sase-core, and a sase workspace built against this core
  still passes `sase tool run check` with no sase changes.

## Python persistence and wire cutover

Repo: sase.

- Pin and bindings: bump `sase-core-revision.txt` to the landed `core-expand` commit
  with `just ratchet-core-revision`, rebuild the extension (`just install`), and call
  the new binding names. Update `tools/validate_sase_core_rs`,
  `tools/check_sase_core_rs_bindings` expectations, and any demo seed that uses the
  renamed fields. Do not touch the published `sase-core-rs` window in `pyproject.toml`;
  the release lane owns it.
- Canonical keys and legacy constants:
  - `src/sase/plan_chain.py` stays the home of agent-session member keys: add
    `AGENT_SESSION_TURN_KEY = "agent_session_turn"`, `TURN_KIND_KEY = "turn_kind"`, and
    `LEGACY_AGENT_SESSION_SHELL_KEY`/`LEGACY_SHELL_KIND_KEY` next to the existing
    `LEGACY_AGENT_FAMILY_SHELL_KEY`, each used only through one shared accessor.
  - Route every reader through those accessors, including `core/wire.py`,
    `core/agent_scan_wire_markers.py`, `core/wait_dependency_resolution/`, and the TUI
    loaders that read `meta.agent_session_shell` (`_meta_enrichment_wire.py`,
    `_workflow_snapshot_loaders.py`, `_done_snapshot_loaders.py`).
  - Writers emit only new spellings: `shells/member.py` writes `turn_kind` (`gate` or
    `monitor`), and `monitor/member.py`, `monitor/start.py`, and `gate_shell/member.py`
    pass the new kind.
- Durable Python-owned JSON, each with a legacy reader:
  - plan-gate meta keys and artifact files → `plan_gate_turn_*` (`plan_shell/create.py`,
    `plan_shell/followup.py`, and the inherited-prompt lookup of
    `plan_shell_original_prompt_path`)
  - gate envelopes: the `turn` block (`notification_gates/service.py`,
    `notification_gates/model_shell.py`), `continuation_mode: "gate_turn"`
    (`notification_gates/model_request.py`, `sudo/gate.py`), `gate_next_fork` value
    `turn`, cancel sources, stored error codes, and the gate-intent marker
    (`agent/gate_intent.py`), keeping the stored-bundle hash rule above
  - proc records: rename `Proc.shell_name`/`shell_kind` → `proc_name`/`proc_role`,
    `ProcSubmitRequest` and `ProcUnitWire` mirrors, the lifecycle constant, and the
    concurrency-key helper (`procs/models/`, `procs/request.py`, `procs/names.py`,
    `procs/submission.py`, `service/host_spawn.py`, `sudo/detach_approve.py`).
    `Proc.from_dict` reads both spellings; payloads sent to core use the new keys, which
    `core-expand` accepts.
  - runner-slot records: `agent_session_turn_{kind,id,state}`
    (`core/runner_slots/_admission_capacity_records.py`); hold candidates: `named_proc`
    (`agent/launch_admission_engine_holds.py`)
  - dismissed procs: rename `src/sase/ace/dismissed_proc_shells.py` →
    `src/sase/ace/dismissed_procs.py` and migrate the file as described in the policy;
    update the `bgcmd.py` and startup-prune callers mechanically
  - `AgentType.PROC_SHELL` → `AgentType.NAMED_PROC` (`src/sase/core/agent_types.py`),
    with a legacy value reader only if the value is persisted
- Python wire mirrors: rename `core/agent_scan_wire_agent_session_shell.py` →
  `core/agent_scan_wire_agent_session_turn.py`, the `AgentSessionShell*Wire` mirrors,
  `agent_session_shell_from_mapping`, and the fleet node and row mirrors (`row_kind`,
  `shell_id`, `historical_shell`, `shell_start_status`). Each mirror hydrates from
  either spelling, because core still emits legacy spellings until `contract-flip`.
- Contract-flip readiness: audit every Python comparison against a core schema version
  (including `src/sase/core/health.py` and `tools/validate_sase_core_rs`). A version
  bump must degrade through fallback or rebuild, never crash. Record anything that would
  still hard-fail as a note for `contract-flip`.
- Update `Agent` model references mechanically for the renamed core fields, but leave
  TUI-owned module, label, section, and row names to `tui-cutover`, and package/module
  names outside `core/` and `procs/` to `runtime-cutover`.
- Tests:
  - legacy-input tests for every durable surface above, each loading a realistic
    pre-rename file (plan gate pending across the upgrade, stored gate bundle with a
    `shell` block, legacy proc rows including a live `shell:` concurrency conflict,
    legacy dismissed file)
  - tests proving Python writers emit no legacy key
  - core round-trip tests through the new binding names
  - rename `tests/test_core_agent_scan_wire_agent_session_shells.py` and
    `tests/test_dismissed_proc_shells.py`; keep legacy-shaped fixtures, named as legacy,
    next to new-shape fixtures
- Exit: `sase tool run check`.

## Runtime, syntax, and CLI cutover

Repo: sase. Covers everything outside `src/sase/ace/tui/` and `tests/ace/`. Touch TUI
files only to follow renamed imports and functions.

- Rename packages and modules:
  - `src/sase/gate_shell/` → `src/sase/gate_turn/`, `src/sase/shells/` →
    `src/sase/turns/`, `src/sase/plan_shell/` → `src/sase/plan_gate_turn/`,
    `src/sase/question_shell/` → `src/sase/question_gate_turn/`
  - `main/gate_shell_handler.py` → `main/gate_turn_handler.py`,
    `main/gate_shell_render.py` → `main/gate_turn_render.py`,
    `notification_gates/model_shell.py` → `notification_gates/model_turn.py`
    (`GateShellSpec`, `GateShellNext`, `GateShellBranchSpec` → `GateTurn*`), and
    `scripts/sase_chop_gate_shell_reclaim.py` → the `gate_turn_reclaim` equivalent under
    the existing job-script naming convention
  - Rename every remaining concept identifier, comment, and message in `gate_turn`,
    `turns`, `monitor`, `procs`, `notification_gates`, `agent`, `axe`, `core`, `main`,
    `completion`, `sudo`, `service`, `telemetry`, `agents_sync`, `sase_agent.py`,
    `_plan_approval_response.py`, and the rest of `src/sase/` outside the TUI.
  - Do not keep internal alias modules just to shrink the diff; sase-telegram's imports
    are fixed in the `telegram` phase.
- Hazard: `gate_shell/agent_handoff.py` rewrites handoff errors with
  `str(exc).replace("shell", "gate shell")` and `monitor/handoff.py` with
  `.replace("shell", "monitor")`. Do not port this as a `"turn"` substring replacement
  (it would corrupt "return"); pass the member noun into `TurnHandoffError` instead.
- Canonical user syntax and the sunset flag:
  - `sase gate create`: `-G/--turn`, `-g/--turn-status`, `-E/--turn-stop-status`, and
    `-f/--next-fork {session,turn,none}` (`main/parser_gate.py`, including
    `_parse_next_fork`); re-sort the options by long name per `cli_rules.md`
  - gate specs: the `turn` block and `"fork": "turn"`
  - `sase proc list` and `sase proc run`: `-N/--name` (`main/parser_proc.py`)
  - config: `gate.turn.reclaim_grace_seconds` (`default_config.yml`,
    `config/sase.schema.json`, `config/_settings_system.py`)
  - Create the flag only with
    `sase flag new legacy_sase_shell_syntax -k sunset --when-enabled ... --when-disabled ... --remove-when ...`,
    using these sentences:
    - **when enabled:** SASE silently accepts the retired sase-shell spellings as
      aliases of their sase-turn replacements: `sase gate create --shell`,
      `--shell-status`, `--shell-stop-status`, and `--next-fork shell`; a gate spec's
      `"shell"` block, `"fork": "shell"`, and `"continuation_mode": "gate_shell"`;
      `sase proc list/run --shell`; and the `gate.shell.reclaim_grace_seconds` config
      key.
    - **when disabled:** SASE rejects those spellings with an error naming the sase-turn
      replacement; agent metadata, gate bundles, and proc rows written before the rename
      still load.
    - **remove when:** no maintained prompt, xprompt, skill, gate spec, script, or
      config in the sase-org repos or chezmoi still uses a retired sase-shell spelling,
      and a sase release with sase-turn syntax has shipped.
  - Legacy long options stay hidden (`argparse.SUPPRESS`) and route through
    `legacy_sase_shell_syntax.py`. Test both flag states. Add the flag's files to the
    allowlist of `tests/test_agent_session_terminology.py` only if that guard flags
    them.
- CLI help and output:
  - help and messages in `parser_gate.py`, `parser_proc.py`, `parser_agent_hold.py`
    ("named proc name"), `parser_sudo.py` ("gate turn ref"),
    `notifications/cli_wait.py`, and the `sase gate list` state choices
  - JSON: `sase gate list -j` `gate_shells` → `gate_turns`; `sase gate cancel -j` and
    `sase gate create -j`/launch requests `gate_shell` → `gate_turn`
    (`agent/launch_request_types.py`); `sase gate show -j` `shell`/`gate_shell` →
    `turn`/`gate_turn` (`notification_gates/cli_show.py`); `sase proc list/show -j`
    `shell_name`/`shell_kind`/`named_proc_shell` → `proc_name`/`proc_role`/`named_proc`
    (`main/proc_render.py`)
  - `sase proc` tables: the `SHELL` column → `NAME`, "Named proc shell" → "Named proc"
  - refresh `tests/completion/snapshots/cli_spec.json` with `tools/sync_completion_spec`
- Scheduler job: `gate_shell_reclaim` → `gate_turn_reclaim` in `default_config.yml`
  (including its description), the script module, and the `pyproject.toml` entry points
  (`sase_chop_gate_shell_reclaim`, `sase_job_gate_shell_reclaim`), following the
  existing chop/job alias convention. The job's prior run history under the old name is
  not migrated; say so in the breaking-change footer.
- Telemetry: `sase_gate_shell_lookup_duration_seconds` and
  `sase_gate_shell_lookup_fallbacks_total` → `sase_gate_turn_*`, and catalog entry
  `sase_gate_shell` → `sase_gate_turn` (`telemetry/metrics.py`, `telemetry/catalog.py`).
- Directives and xprompts: `%wait(proc=<proc id or named proc>)`
  (`xprompt/_directive_types.py`), `xprompts/fork.yml` (monitor turn or named proc), and
  `shell_routing_prefix` → `turn_routing_prefix`.
- Skill sources in `src/sase/xprompts/skills/`: the concept lines in `sase_gate.md`
  (heading "Declare The `turn` Block", "Create The Gate Turn, Then Stop", the JSON
  example, `--turn`, `--turn-status`, `--turn-stop-status`, `next.fork`
  `session|turn|none`), `sase_run.md` (the `"gate_turn"` descriptor key),
  `sase_questions.md`, `sase_plan.md`, and `sase_monitor.md`. Keep their Unix-shell
  lines. Do not deploy to chezmoi in this phase.
- Tests: rename `tests/gate_shell/`, `tests/shells/`, `tests/plan_shell/`,
  `tests/question_shell/`, `tests/gate_conformance/test_gate_shell_conformance.py`,
  `tests/main/test_gate_shell_handler_{cancel,list}.py`,
  `tests/telemetry/test_gate_shell_lookup.py`,
  `tests/test_agent_loader_pending_gate_shell.py`,
  `tests/test_agent_loader_status_override_gate_shell_agent_session.py`,
  `tests/test_axe_run_agent_exec_{plan,questions}_gate_shell.py`,
  `tests/test_config_schema_gate_shell.py` (and `tests/contract_manifest.txt`),
  `tests/test_shell_handoff_outcome_parity.py`, and
  `tests/test_shell_prompt_output_tail.py`. Update the test IDs recorded in the
  shard-timing and `tests/reproducible_flake_baseline.txt` files, and the module paths
  in the `tests/test_agent_session_terminology.py` allowlist.
- Commit with a `feat!:` subject and a `BREAKING CHANGE:` footer listing the CLI flags,
  JSON keys, config key, job name, and metric names.
- Exit: every remaining `shell` hit outside the TUI is an unrelated meaning, a named
  legacy reader or fixture, or the flag branch; `sase tool run check` passes.

## TUI turn surfaces

Repo: sase. Covers `src/sase/ace/tui/**`, `src/sase/ace/testing/**`, `tests/ace/**`,
`tests/perf/**`, and the TUI parts of `src/sase/default_config.yml` and
`src/sase/config/sase.schema.json`.

- Rename modules: `actions/agents/_proc_shell_dismiss.py`,
  `models/agent_proc_shells.py`, `models/_agent_session_shell_membership.py`,
  `widgets/prompt_panel/_agent_proc_shell_section.py`, and
  `widgets/prompt_panel/_agent_shell_section.py`. Rename the turn identifiers in
  `models/agent_session_members.py` (`ShellLaneCounts`, `NO_SHELL_LANES`,
  `row_is_agent_session_shell`, `shell_lane_counts`, `panel_shell_lane_counts`,
  `concrete_agent_session_shell_rows`, `current_agent_session_shell_row`), the lane
  types (`ShellLane`, `_AgentShellLane`, `_MonitorShellLane`, `_GateShellLane`),
  `ResponsiveShellSection`, `_is_untitled_sase_shell`, `is_shell`,
  `include_monitor_shells`, `is_proc_shell` → `is_named_proc`, and the style constants
  (`_PROC_SHELL_*`, `_SHELL_*`).
- Ids and strings that must move with the rename: `SHELL_SECTION_ID = "shells"` →
  `"turns"`, `PROC_SHELL_SECTION_ID = "proc-shell"` → `"named-proc"` (and the
  `"proc-shell:{id}"` output source prefix), worker group `"proc-shell-dismiss"`, and
  the producer-site method-name strings such as `"_do_kill_proc_shell"`
  (`_proc_producer_sites_actions.py`). If a section id is persisted in fold state, read
  the legacy id.
- Visible copy (see the vocabulary table):
  - `SESSION SHELLS` → `SESSION TURNS`, the hidden-rows tail, `"Shells: "`, the "… +N
    more shells (see SESSION SHELLS)" and "… also listed under SESSION SHELLS" rows, and
    `"{total} shells"`
  - kind headers in `_identity_header.py`, `_agent_display_header.py`, and the Node
    Finder kinds (`models/node_finder.py`): `AGENT TURN`, `GATE TURN`, `MONITOR TURN`,
    and `NAMED PROC`; the Node Finder preview `"TURNS"`/`"No loaded turns."`
  - `ConfirmKillProcShellModal` → a named-proc modal titled "Kill Named Proc"
  - notifications in `_monitor_stop_flow.py`, `_proc_shell_dismiss.py`,
    `proc_shell_count_phrase` and its callers, `_kill_flow.py`, `_wait_helpers.py`, and
    `_member_jump.py` ("No turn N", "Turn roster changed; jump cancelled"). Use "monitor
    turn" where the row is a session monitor and "named proc" where it is a stand-alone
    proc.
  - display-name fallbacks in `models/agent.py` ("named proc", "gate turn"), the footer
    `("0-9", "turn")` in `widgets/_keybinding_bindings_agents.py`, the help legend in
    `modals/help_modal/agents_bindings.py` ("Monitor turn, running", "Gate turn,
    settled", ...), and the descriptive text in `_artifact_tab_descriptions.py`,
    `modals/statistics_help_modal.py`, and `modals/statistics_pane_legends.py`
  - the "live shells" comment in `default_config.yml`
- No keymap or config value changes are expected. If one does change, update
  `src/sase/default_config.yml` per the keymap gotcha.
- Performance contract: renames only. Add no filesystem work, awaits, subprocesses, or
  full rebuilds to navigation or render handlers, and run the existing j/k navigation
  benchmark to confirm no regression.
- PNG goldens:
  - Rename the shell-named snapshot tests, fixtures, and the 7 shell-named goldens (for
    example `test_ace_png_snapshots_agents_proc_shells.py`,
    `_ace_agents_proc_shell_png_fixtures.py`, `agents_session_panel_shells_gate_*`),
    including golden window titles.
  - Re-baseline only goldens whose pixels change because the copy changed (expect
    roughly 150–200 Agents-tab goldens). Run `just fix-tui-screenshots` through
    `/sase_monitor` with `TESTING`/`TESTED`, inspect every creation, removal, and update
    group in the report, and remove stale goldens only after a full run.
- Exit: every remaining `shell` hit in scope is classified (Artifacts-pane chrome, modal
  frames, Unix shell, `!` commands, completion, or a named legacy reader), and
  `sase tool run check` passes.

## Documentation and memory

Repos: sase, plus a chezmoi skill redeploy. The user's request ("update ALL references")
authorizes the memory edits; follow `/sase_memory_write` for them.

- First, before editing anything: the `runtime-cutover` skill sources have landed, so
  from the clean landed tree run `sase skill init --force` (then `chezmoi apply` if it
  was skipped) so deployed skills stop teaching retired flags and keys. `audit-deploy`
  redeploys again at the end.
- Rewrite every concept mention in `docs/`, `README.md`, and the blog (no blog hits
  today):
  - heavy files: `ace.md`, `agent_sessions.md`, `monitors.md`, `configuration.md`,
    `xprompt.md`, `notifications.md`, `cli.md`, `architecture.md`, `telemetry.md`,
    `sudo.md`, and `axe.md`
  - also `troubleshooting/runner-slots.md`, `workflow_spec.md`, `development.md`,
    `beads.md`, `sdd.md`, `init.md`, `getting_started.md`, `commit_workflows.md`,
    `remote_dispatch.md`, `tool.md`, and whatever else the sweep turns up
  - bare "shell"/"shells" in the concept sense is common and a phrase grep misses it;
    read every `shell` hit and classify it
  - new names: `--turn`, `--turn-status`, `--turn-stop-status`, `--next-fork turn`,
    `sase proc -N/--name`, `gate.turn.reclaim_grace_seconds`, `gate_turn_reclaim`, the
    `sase_gate_turn_*` metrics, `continuation_mode: gate_turn`, lifecycle `named-proc`,
    the JSON keys, `SESSION TURNS`, and the TUI kind headers
  - where docs first introduce the concept (`agent_sessions.md` and `architecture.md`),
    add one sentence that turns were formerly called shells and the chat-transcript
    analogy
  - headings and anchors: `architecture.md` "## Agent, Monitor, and Gate Shells" → "##
    Agent, Monitor, and Gate Turns"; `notifications.md` "### Gate shells and
    continuation" → "### Gate turns and continuation", fixing its five inbound links
    (`axe.md`, `cli.md` twice, `configuration.md`, `sudo.md`)
  - reword "agent shell" meaning an agent's process environment per the identifier rules
- Memory:
  - Replace glossary strands:
    - `glossary/sase-shell.md` → `glossary/sase-turn.md`: keyword `Sase Turn`, alias
      `turn`; an agent turn, a monitor turn, or a gate turn; a sase agent is an ordered
      sequence of sase turns; the chat-transcript analogy; one clause that it was
      formerly called a sase shell; one clause separating it from a provider's internal
      round trips (`num_turns`) and chat-history conversation turns
    - `glossary/agent-shell.md` → `glossary/agent-turn.md` (keyword `Agent Turn`)
    - `glossary/gate-shell.md` → `glossary/gate-turn.md` (keyword `Gate Turn`, the
      `turn` block, `turn_kind: "gate"`, `<session>--gate` naming, formerly a gate
      shell)
    - `glossary/proc-shell.md` → `glossary/named-proc.md` (keyword `Named Proc`): a
      supervised proc with a stable `proc_name` and lifecycle `named-proc`; a monitor
      turn's command runs as a named proc named after its member; a stand-alone `%proc`
      xprompt unit is a named proc that belongs to no agent and shows as its own proc
      node on the Agents tab; a gate turn's execution proc (`proc_role: "gate"`) does
      not make the gate turn a monitor turn. This also resolves task bead `sase-sa`.
  - `glossary/sase-monitor.md`: add alias `monitor turn` and define a sase monitor as a
    monitor turn instead of a session-attached proc shell.
  - Update the strands that mention the concept: `sase-agent-session`, `sase-agent`
    ("one-turn agent"), `sase-node` (member sase turn nodes, stand-alone named proc
    node), `agent-node`, `agent-relation-jump-target` (`SESSION TURNS`),
    `agent-data-deck`, `sase-gate` (the `turn` block), and `proc` if it should point at
    Named Proc.
  - Flat note `lint_and_test.md`: "SASE agent shells export `CI=true`" → "SASE agent
    processes export `CI=true`".
  - Leave every decision record, including `gates-never-block.md`, unchanged.
  - Run `sase memory init`, then review the regenerated `glossary.md` roster,
    `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `QWEN.md`, `OPENCODE.md`, and the memory
    README. Confirm `sase memory read glossary:"sase turn" glossary:"monitor turn"`
    resolves.
- Close `sase-sa` with `sase bead close sase-sa --note "<what changed>"`.
- Exit: `sase tool run check`, including the docs lints.

## sase-telegram cutover

Repo: sase-telegram.

- `tests/test_gate_shell_settlement.py` → `tests/test_gate_turn_settlement.py`:
  `create_gate_turn_member`, `GateTurnSpec`, `spec["turn"]`, `turn_row_managed=True`,
  `_make_gate_turn_member`, `test_telegram_submits_a_turn_backed_gate`, and
  `"tg-turn-1"`-style ids; `tests/test_custom_gates.py`: `result.gate_turn_creation`.
- telegram's `sase>=` floor predates the rename. Import the renamed sase APIs through
  one clearly named test helper that falls back to the legacy module and names, so the
  suite passes against both the installed sase release and sase master.
- `src/sase_telegram/inbound.py`: the docstring naming
  `bind_gate_turn_execution_callbacks` → `settle_gate_turn` and "gate-turn settlement
  glue".
- If task bead `sase-16v` (the same test's rowless-block failure) is still open, fix it
  in this rewrite and close it with a note.
- Leave the CHANGELOG and the Unix-shell uses alone. `tools/require_tool_run` is handled
  in `audit-deploy`.
- Exit: `sase tool run check` in sase-telegram.

## sase-core contract flip

Repo: sase-core, a breaking `feat!:` change with a `BREAKING CHANGE:` footer. Start only
after all sase and telegram work has landed. sase changes are allowed only when they
work against both the old and the new core.

- Serialized output:
  - Remove the legacy `rename` pins so every renamed field and variant serializes its
    new name.
  - Keep `alias = "<legacy>"` only where durable pre-rename data or the sunset flag can
    still produce the old spelling, marking each with a `legacy sase-shell spelling`
    comment. Drop the aliases on purely in-process request wires (runner capacity, hold
    candidates, launch-plan and proc-dispatch requests).
- Remove the two legacy binding names and their `prelude.rs` aliases; assert them absent
  in tests.
- Emitted values: lifecycle and origin `named-proc`, concurrency prefix `named-proc:`,
  `turn_kind` value `monitor`, fleet row kinds `agent_turn`/`historical_turn`, role
  `historical_turn`, locator `turn_id` and its key component, hold match kind
  `named_proc`, and the launch-plan diagnostic code `invalid-named-proc-name`. Parsers
  keep accepting the legacy forms.
- Artifact index: rename column `gate_shell_id` → `gate_turn_id` and its index, and bump
  `AGENT_ARTIFACT_INDEX_SCHEMA_VERSION` (33 today) through the migration path.
- Schema versions: bump each whose emitted shape changed, for example
  `AGENT_SCAN_WIRE_SCHEMA_VERSION` (10), `PROC_WIRE_SCHEMA_VERSION` (3; keep the older
  versions supported), `FLEET_CONTRACT_SCHEMA_VERSION` (6),
  `RUNNER_CAPACITY_POLICY_SCHEMA_VERSION` (6), `AGENT_HOLD_WIRE_SCHEMA_VERSION` (2),
  `LAUNCH_PLAN_WIRE_SCHEMA_VERSION` (2), and `PROC_DISPATCH_WIRE_SCHEMA_VERSION` (1).
  Bump `FLEET_PROTOCOL_VERSION` (2) in both `fleet_contract/error.rs` and
  `sase_gateway/src/wire.rs`.
- Goldens and fixtures: regenerate `contracts/api_fleet_v1/fleet_api_v1.json` with
  `UPDATE_FLEET_CONTRACT=1`; update `tests/python_wire_parity.rs`; regenerate
  `crates/sase_core/tests/fixtures/command_line/sase_spec.json` from the landed sase CLI
  per that directory's `README.md` (if unrelated drift makes the resolver goldens churn,
  apply the rename to the fixture's concept flags and help strings in place instead);
  update the `src/sase/gate_shell/handoff.py` path fixture in
  `tool_run/triage/tests/mod.rs`.
- Before landing, build this core locally and run current sase master against it with
  `sase tool run check`. Python readers must accept the new spellings, and
  mirror-version mismatches must degrade through fallback or rebuild, not crash. Fix any
  sase breakage in sase in a way that works against both cores, and list every mirror
  constant `pin-bump` must move.
- Exit: `sase tool run check` in sase-core and the sase-master-against-new-core run.

## Core pin bump and mirrors

Repo: sase.

- Bump `sase-core-revision.txt` to the landed `contract-flip` commit and rebuild the
  extension.
- Update the Python `*_WIRE_SCHEMA_VERSION`, index-version, and fleet-protocol mirrors
  that `contract-flip` listed, plus `src/sase/core/health.py` and
  `tools/validate_sase_core_rs` expectations, so `sase core health` stays green.
- Update the fixtures and goldens that capture core output keys and values (`row_kind`,
  `turn_id`, `turn_kind`, `proc_name`, lifecycle `named-proc`). Keep every durable
  legacy reader and its legacy-input tests.
- Exit: `sase tool run check` and `sase core health`.

## Cross-repo audit, guardrail, and deploy

Repos: sase, sase-core, sase-telegram, sase-github, sase-research-artifacts, sase-nvim,
and chezmoi.

- Add `tests/test_sase_turn_terminology.py`, modeled on
  `tests/test_agent_session_terminology.py`:
  - It fails on shell-concept identifiers in `src/`: `gate_shell`, `GateShell`,
    `GATE_SHELL`, `agent_session_shell`, `AgentSessionShell`, `proc_shell`, `ProcShell`,
    `PROC_SHELL`, `plan_shell`, `question_shell`, `monitor_shell`, `shell_name`,
    `shell_kind`, and imports of `sase.shells`.
  - It also fails on stale phrases in current docs, skill sources, and memory: sase,
    agent, gate, proc, monitor, and session shell(s); `SESSION SHELLS`, `AGENT SHELL`,
    `PROC SHELL`; `--next-fork shell`, `"fork": "shell"`, `gate.shell.`, and
    `gate_shell_reclaim`.
  - A commented allowlist names the legacy-reader homes, the sunset-flag branch, the
    decision records, and the CHANGELOG.
- Sweep every listed repo with case-sensitive and case-insensitive `rg 'shell'` and
  classify each hit as an unrelated meaning, deliberately unchanged history, a named
  legacy reader, alias, or fixture, or the sunset flag. Fix small stragglers in place;
  record substantive leftovers as `PROPOSED FOLLOW-UP:` notes. Check with
  `rg -wi 'turns?'` that new "turn" wording means the new concept or an existing
  agent/provider turn, and that no surface shows a provider's `num_turns` as turns.
- `tools/require_tool_run` in sase, sase-telegram, sase-github, and
  sase-research-artifacts, and `scripts/require_tool_run` in sase-core: "in an agent
  shell" → "inside a SASE agent", keeping the copies byte-identical where they are
  today.
- Integrate: sweep commits that landed on sase, sase-core, and sase-telegram since this
  epic's first phase for newly added shell-concept wording from concurrent work, and fix
  it.
- Beads: append a `sase bead note` mapping old names to new on each still-open bead
  whose title or description names renamed identifiers (for example `sase-10p`,
  `sase-11x`, `sase-151`, `sase-161`, and `sase-16v`).
- chezmoi (via `/sase_repo`): from the landed sase tree, regenerate the provider skill
  copies per `generated_skills.md` (`sase skill init --force`, then `chezmoi apply` if
  it was skipped) and confirm `sase skill init --check` is clean.
- Confirm the `legacy_sase_shell_syntax` flag bead and registry entry exist with
  both-state tests.
- The epic is done when:
  - A fresh agent can create a gate turn with `sase gate create --turn` and
    `--next-fork turn`, start a monitor turn, run a named proc with
    `sase proc run -N <name>`, and see it all as `SESSION TURNS` with the new kind
    headers in the TUI, using only sase-turn contracts.
  - Pre-rename agent metadata, pending plan gates, gate bundles, proc rows, dismissed
    procs, and index databases still load.
  - Rust and Python agree, and `sase core health` is green.
  - `sase tool run check` passes in every changed repo.
