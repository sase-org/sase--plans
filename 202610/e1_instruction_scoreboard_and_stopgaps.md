---
tier: epic
title: 'E1: Instruction scoreboard and stopgaps (memory-built instruction migration)'
goal: 'One read-only command, `sase instructions verify`, shows what each provider''s
  SASE runs and native helpers actually loaded, read from the providers'' own session
  records. It reproduces today''s delivery bugs: Claude and Codex load the contract
  twice, Muse gets no home layer, and Grok gets nothing. The two measured harms stop.
  Grok root runs get the SASE single-turn directive and the project `AGENTS.md` exactly
  once through `--rules`. Claude native helpers get a packaged helper template, and
  a per-run PreToolUse guard denies their root-only operations (`sase final …`, turn-ending
  skills). Both stopgaps have sunset kill switches. Baseline and after scoreboard
  JSON plus an acceptance record are attached to this epic, and bead `sase-1gj` is
  closed as superseded.'
phases:
- id: cli-group
  title: The `sase instructions` command group absorbs `sase memory agent-docs`
  depends_on: []
  size: small
  description: 'cli-group: add the top-level `sase instructions` group. Its `list`
    subcommand, also the bare-group default, is today''s `sase memory agent-docs list`
    inventory, unchanged. Delete the `agent-docs` subcommand, then update the parser
    registries, entry dispatch, completion snapshot, tests, and docs (cli.md, configuration.md,
    init.md).'
- id: scoreboard
  title: '`sase instructions verify`: observed-mode scoreboard and doctor group'
  depends_on:
  - cli-group
  size: medium
  description: 'scoreboard: build `sase instructions verify`. Pure Python parsers
    turn Claude transcripts and subagent transcripts, Codex rollouts, Grok `prompt_context.json`
    and `system_prompt.txt`, and Muse `session.jsonl` into per-session observations.
    agy is reported as unverifiable. Locate sessions from the newest SASE runs per
    provider (bounded index query, run cwd and time window, capped reads). Show contract,
    home, project, directive, native-full, foreign, and helper columns with `-j` JSON,
    and add the deep-only doctor group `instructions`. Fixture tests must reproduce
    the baseline table. Attach the live baseline JSON to the epic.'
- id: grok-root
  title: Grok root runs receive the directive and project AGENTS.md once via --rules
  depends_on: []
  size: small
  description: 'grok-root: absorbs sase-1gj with a corrected fix. On every invocation
    cycle the Grok adapter passes `--rules` with a new Grok single-turn directive,
    plus the project root `AGENTS.md` text exactly once in SASE-managed projects (the
    directive alone elsewhere). Never pass the home layer or `--trust`, and never
    set GROK_CLAUDE_AGENTS_ENABLED. Add a 120 KiB argv guard, the sunset flag `grok_rules_delivery`,
    argv tests for both flag states, a live parse probe, a local canary, and Grok
    docs.'
- id: claude-helpers
  title: Claude native helpers get a helper template and a root-only PreToolUse guard
  depends_on: []
  size: medium
  description: 'claude-helpers: on every Claude invocation cycle, pass a packaged
    static helper template through the hidden `--append-subagent-system-prompt-file`,
    gated by a cached no-API capability probe. Also pass inline `--settings` JSON
    with a stdlib-only PreToolUse guard on Bash|Skill. When the hook input carries
    `agent_id`, the guard denies `sase final context|defer|prepare|submit`, turn-ending
    CLI commands, and root-only skills. Add the sunset flag `claude_helper_channel`,
    the doctor deep check `providers.claude_helper_channel`, tests, and raw mechanism
    probes: deny under bypass mode, the Explore and general-purpose markers, and whether
    forks carry `agent_id`.'
- id: record
  title: Root-only contract sentence, decision record, capability docs, ownership
    inventory
  depends_on:
  - scoreboard
  - grok-root
  - claude-helpers
  size: medium
  description: 'record: add the actor-qualified root-only sentence to `memory-sase.template.md`
    and the `sase_final` skill source, then regenerate the project''s generated instruction
    files. Write the decision record `helpers-return-roots-declare` ("Native Helpers
    Return; Only Roots Declare") via /sase_memory_write. Add marker-consistency tests
    that tie the scoreboard fingerprints to the shipped constants. Document root/helper
    limits by provider capability, and write `docs/instruction_inventory.md` with
    a disposition for every instruction surface.'
- id: acceptance
  title: Live probes, after-scoreboard, acceptance record, close sase-1gj
  depends_on:
  - record
  size: small
  description: 'acceptance: once the host runs the landed code, request one Grok probe
    and one Claude probe through /sase_run LaunchApproval (resume_requester). The
    Claude probe spawns a general-purpose helper and an Explore helper, and each helper
    attempts `sase final submit`. Verify both probes with `sase instructions verify`,
    attach the after JSON and an acceptance record to the epic, and close sase-1gj
    as superseded.'
proposed_by: bbugyi200.athena.0x2
create_time: 2026-10-05 15:52:16
status: wip
bead_id: sase-1gu
---

- **PROMPT:** [prompts/202610/e1_instruction_scoreboard_and_stopgaps.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/e1_instruction_scoreboard_and_stopgaps.md)
- **BEAD:** [sase-1gu](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1gu/README.md)

# Plan: E1 — Instruction scoreboard and stopgaps

## Context

### Where this comes from

- **Research (primary input):**
  `research:202610/memory_built_instruction_migration_epics/memory_built_instruction_migration_epics.md`
  (sections "E1 Instruction scoreboard and stopgaps", "The yardstick and what it should
  show after each epic", "Issues in the three prior reports", "Checklist for every epic
  plan"). Read it with `sase artifact read`, not by opening files.
  - It splits the migration to memory-built, per-invocation instructions into five
    ordered epics (E1–E5), plus optional E6 (TUI) and conditional E7 (audiences). **This
    plan is E1 only.**
  - Its researcher reports sit in the same directory (`__cld`, `__cdx`, `__grk`,
    `__mus`, `__gem`). `__cld` §5–§6 has the most E1 detail.
  - The Claude helper-channel evidence and the helper overlay draft come from
    `research:202610/fresh_clone_bootstrap_and_native_subagent_instructions/fresh_clone_bootstrap_and_native_subagent_instructions.md`
    ("One shared core and two lifecycle overlays" and "Root-only operations enforced
    mechanically").
- **Bead `sase-1gj` (READY, bug, size large):** "Grok SASE runs load zero instruction
  files since ~2026-09-10". This epic absorbs it in `grok-root` and closes it as
  superseded in `acceptance`. **Do not launch its suggested fix as written.** That fix
  pipes the home layer into `--rules`, and both the home and project files carry the
  contract, so Grok would get it twice. It also adds no directive, and its
  `GROK_CLAUDE_AGENTS_ENABLED=0` does nothing.
- **Why E1 is its own epic:** E1 changes only Grok root runs and Claude helpers. Its
  effects need at least 3 days of real runs before E3 switches delivery for every
  provider. An epic cannot pause for days, so this soak is an epic boundary. E2 (shadow
  bundles) may start as soon as E1 lands.

### The two measured harms

1. **Grok roots get nothing.**
   - From 2026-09-11 to 2026-10-05, 568 of 568 Grok sessions in SASE workspaces loaded 0
     instruction files. Since 2026-10-03, every Grok `prompt_context.json` has
     `agents_md_files: []`.
   - The workspaces are untrusted, and `grok.py` passes no `--rules`, no trust flag, and
     no directive.
   - The non-research final-declaration rate fell from 60% to 26%.
2. **Claude helpers act as roots.**
   - General-purpose helpers load `~/CLAUDE.md` plus the workspace `CLAUDE.md`, so they
     see the contract twice and are told to "use `/sase_final`".
   - Over 30 days, 13–15 of them invoked `sase final`, and 2 submitted accepted
     declarations for their parent's turn.
   - Explore helpers get no instructions at all.
   - `sase final` identifies the turn from environment variables only
     (`SASE_ARTIFACTS_DIR`, `SASE_AGENT_TIMESTAMP`, `SASE_FINAL_TURN_NONCE`). Claude
     helpers inherit that environment, so the host cannot tell a helper's submit from
     the root's.

### Baseline the scoreboard must reproduce

```
provider contract home project directive native-SASE-full foreign      helpers
claude      2×     ✓     ✓        ✓          2           auto-memory  gp: root contract, 13–15 `sase final`/30d, 2 accepted; Explore: nothing
codex       2×     ✓     ✓        ✓          2           —            0 spawns/30d
muse        1×     ✗     ✓        ✓          1           —            0 spawns
grok        0      ✗     ✗        ✗          0           memory-v2½   few children: nothing
agy         ◌      ✗     ◌        ✓          ◌ (likely 2) —           —
```

**If the scoreboard shows all green on day one, the scoreboard is wrong.**

### Facts verified while planning (2026-10-05; Claude Code 2.1.289, codex-cli 0.160.0, grok 1.0.46)

- **Grok `--rules` is observable.**
  - `grok --help` lists `--rules <RULES>`, "Extra rules to append to the system prompt".
  - A canary run (`--rules` with a unique heading and body marker, untrusted `/tmp` cwd)
    put the text verbatim into the session's `system_prompt.txt`, inside
    `<human_rules>…</human_rules>`. It also appeared in `chat_history.jsonl`.
  - `prompt_context.json` still showed `agents_md_files: []`.
  - Grok session layout:
    `~/.grok/sessions/<url-encoded cwd>/<session-id>/{prompt_context.json, system_prompt.txt, chat_history.jsonl, …}`.
    `prompt_context.json` has `agents_md_files`, `audience` (`primary` or `subagent`),
    `memory_enabled`, and `working_directory`.
- **The Grok adapter** (`src/sase/llm_provider/grok.py`, `_invoke_loop`) builds a new
  argv, with a fresh `--session-id <uuid4>` and `--cwd os.getcwd()`, on every
  continuation cycle. There is no `--rules`, no `--trust`, and no directive constant.
- **The Claude adapter** (`src/sase/llm_provider/claude.py`, `_invoke_loop`) passes
  `--append-system-prompt _SINGLE_TURN_DIRECTIVE` and `--disallowedTools ScheduleWakeup`
  on every cycle. Wait-continuations rerun with `--resume <uuid>`. There is no
  `--settings` and no subagent flag. `tests/llm_provider/test_claude_hooks.py` forbids
  creating or modifying `<workspace>/.claude/`, so a per-run hook must travel as
  **inline** `--settings` JSON.
- **The hidden Claude flag can be probed without an API call.**
  - `claude --version` and `claude --help` short-circuit and exit 0 even with a bogus
    flag, so neither can test whether a flag parses.
  - `claude -p --append-subagent-system-prompt-file /nonexistent/x.md </dev/null` prints
    `Error: Append subagent system prompt file not found: /nonexistent/x.md` when the
    flag is supported.
  - `claude -p --some-unknown-flag </dev/null` prints
    `error: unknown option '--some-unknown-flag'`.
  - Neither call contacts the API.
- **Claude transcripts.**
  - Root transcripts are at
    `~/.claude/projects/<cwd with every non-alphanumeric char replaced by "-">/<session>.jsonl`.
  - Records with `type: "attachment"` include:
    - `attachment.type == "instructions"`, with `files: [{path, type, content}]` for
      natively loaded `CLAUDE.md` files;
    - `prompt_snapshot`, with `systemPrompt: [...]`, which contains the
      `--append-system-prompt` text and the auto-memory section when it is on;
    - `nested_memory`.
  - Helpers are at `<session>/subagents/agent-<id>.jsonl`, next to
    `agent-<id>.meta.json` (`agentType`, `description`, `spawnDepth`). Helper records
    carry `agentId` and `isSidechain: true`.
  - The existing resolver, `src/sase/ace/tui/thinking/session_resolver.py`, encodes the
    cwd and matches by time. It looks at top-level files only and is used only by tests.
- **Codex rollouts** are at `~/.codex/sessions/YYYY/MM/DD/rollout-*.jsonl`. The per-run
  shadow `CODEX_HOME` symlinks `sessions/` back to the real home
  (`codex._real_codex_home`).
  - The first record is `session_meta`, with `cwd`, `timestamp`, `cli_version`, and
    `source`.
  - A developer message carries the directive, which starts
    `SASE single-turn instructions for Codex:`.
  - A user message starting `# AGENTS.md instructions for <dir>` carries the natively
    loaded `AGENTS.md` text.
- **Muse sessions** are at
  `$XDG_DATA_HOME/muse/sessions/*/*/*/<session-id>/session.jsonl`. SASE stores
  `muse_session_id` in `<artifacts>/run_metadata.json`, and
  `src/sase/llm_provider/_muse_session_usage.py` already finds the file. Native rules
  appear as `source: "rules_file"` records whose text contains
  `<rules-file scope="project" path="AGENTS.md">`.
- **agy** stores SQLite conversations under `~/.gemini/antigravity-cli/conversations`
  (`_subprocess_agy.py`) and records no instruction loads.
- **Run-to-session mapping.**
  - SASE records no provider session id in `agent_meta.json`; Muse's is the only one, in
    `run_metadata.json`. A run maps to its provider sessions by `workspace_dir` (the
    provider cwd, via `enter_agent_workspace`) and by the run's time window.
  - `sase.agent.listing_snapshot.listing_snapshot(project=…, requested_limit=N)` is the
    bounded newest-N listing. It queries the artifact index and accepts a `provider`
    candidate filter.
  - **Never use `include_full_history=True` or any O(history) walk.** Those caused
    athena's resource spikes.
- **`sase final` subcommands** that act for the turn are `context`, `defer`, `prepare`,
  and `submit`. A successful submit prints `Accepted final declaration …`.
- **The contract text** comes from
  `src/sase/main/init_memory/templates/memory-sase.template.md`, section
  `## SASE Final Declaration`.
  - `sase memory init` renders `sase/memory/sase.md`, then the root `AGENTS.md` and its
    four byte-identical shims (`CLAUDE.md`, `GEMINI.md`, `QWEN.md`, `OPENCODE.md`).
  - When chezmoi is in use, it also renders the home layer into the chezmoi source.
  - `tests/main/test_init_memory_handler_outputs.py` pins the contract markers.
- **`sase doctor` has no subcommands.**
  - Checks are `CheckSpec(id, group, title, runner, deep=…)`, registered by
    `*_check_specs(context)` functions in `src/sase/doctor/runner.py`.
  - `providers` and `terminal` are existing deep-only groups.
  - `tests/main/test_doctor_command.py` asserts the deep ID set.
- **`sase memory agent-docs list`** is `src/sase/amd/inventory.py::run_amd_list`, parsed
  in `src/sase/main/parser_memory.py` and dispatched in
  `src/sase/main/memory_handler.py`. Only docs call it; nothing invokes it from code.
- **Helpers this plan reuses:**
  - `project_is_sase_managed(cwd)` in `src/sase/feature_flags/managed.py`;
  - `discover_project_root` in `src/sase/content_layout.py`;
  - packaged-text loading in `src/sase/mdtemplates.py`;
  - the cached CLI capability pattern in
    `src/sase/llm_provider/usage/_claude_preflight.py`;
  - the live parse-probe test pattern (`_require_grok_build`) in
    `tests/llm_provider/test_grok_provider_core.py`.
- **The host runs agents from an editable install of the primary checkout.** A phase's
  adapter change therefore reaches new SASE runs once the primary checkout has synced
  the landed commit. That is why `acceptance` runs last.

## Design decisions (owned by this plan; phase workers do not re-litigate them)

1. **Observe first, from the providers' own records.** The scoreboard reads provider
   session records only. It never treats `agent_meta.json["instruction_snapshot"]` or
   anything else SASE wrote as evidence of a load. A column it cannot observe shows `◌`.
2. **CLI shape (per `cli_rules`).**
   - One group, `sase instructions`. `list` (moved from `sase memory agent-docs`) is the
     bare default; `verify` is the scoreboard.
   - Doctor gets a deep-only **check group** named `instructions`, never a
     `sase doctor instructions` subcommand.
   - `sase memory agent-docs` is deleted outright. It has no programmatic callers, and a
     compatibility alias would need a sunset flag that isn't worth carrying for a
     read-only inspection command. The commit is a breaking `feat(cli)!`.
3. **The Grok stopgap delivers once.**
   - `--rules` = the new Grok single-turn directive, then the project root `AGENTS.md`
     text **exactly once**, in SASE-managed projects only. Elsewhere it is the directive
     alone.
   - No home layer (it arrives with E3's bundle), no `CLAUDE.md` (a byte-identical
     shim), never `--trust`, and no `GROK_CLAUDE_AGENTS_ENABLED`.
4. **The Claude helper channel.**
   - A static packaged template goes through `--append-subagent-system-prompt-file`. It
     reaches Explore and general-purpose helpers but not the root.
   - A PreToolUse guard goes through per-run inline `--settings`. It is keyed on
     `agent_id`, which Claude sets only for calls made inside a subagent.
   - Native `CLAUDE.md` loading stays unchanged; E3 replaces it.
5. **Shared literal markers.** Phases running in parallel must use these exact strings:

   | Marker                      | Exact text                                | Owner                        |
   | --------------------------- | ----------------------------------------- | ---------------------------- |
   | Contract fingerprint        | the heading text `SASE Final Declaration` | existing template            |
   | Grok directive opening      | `SASE single-turn instructions for Grok:` | `grok-root`                  |
   | Helper template first line  | `# SASE Helper Instructions`              | `claude-helpers`             |
   | Guard deny-reason prefix    | `SASE helper guard:`                      | `claude-helpers`             |
   | Accepted declaration output | `Accepted final declaration`              | existing `sase final submit` |

   Existing directive markers: Claude `SASE single-turn print mode`, Codex
   `SASE single-turn instructions for Codex`, agy
   `SASE Antigravity print-mode instructions`. For Muse, `scoreboard` picks a stable
   substring that both of `_muse_single_turn_directive`'s modes share.

6. **Root-only operations** are one list, owned by `claude-helpers` as constants:
   - **Skills:** `sase_final`, `sase_gate`, `sase_git_commit`, `sase_handoff`,
     `sase_monitor`, `sase_plan`, `sase_questions`, `sase_run`, `sase_sudo`.
   - **Bash forms:** `sase final (context|defer|prepare|submit)`, plus the turn-ending
     CLI forms those skills run. Derive the latter from the skill sources in
     `src/sase/macros/skills/` (for example `sase plan propose`, `sase monitor start`,
     `sase launch request`). A helper running one of them would SIGTERM its parent's
     runner.
   - Matching tolerates path prefixes (`.venv/bin/sase`) and leading `VAR=value`
     assignments.
   - Read-only forms (`sase final list|show|status|doctor`) stay allowed.
7. **Failure posture.**
   - The guard never denies a call without `agent_id`, and it exits 0 silently on
     malformed input.
   - If the capability probe says the hidden flag is unsupported or unknown, the adapter
     omits only that flag, logs one warning, and keeps the guard. The doctor check turns
     ERROR, so a Claude upgrade cannot break every SASE Claude run.
   - Grok rules over the argv limit raise an actionable error. A silent omission is how
     the Grok regression went unnoticed.
8. **Flags.** There are two `sunset` kill switches, default on, created with
   `sase flag new … -k sunset`:
   - `grok_rules_delivery`: Off = today's Grok argv, with no `--rules`.
   - `claude_helper_channel`: Off = today's Claude argv, with no `--settings` and no
     subagent flag.
   - Each needs tests for both states.
   - Remove-when: the E1 watch readout (at least 3 days) shows no regression
     attributable to the channel, or E3's delivery cutover supersedes the channel.
   - Rollback is one `sase flag disable <key>` per execution host (athena, apollo, mac).
   - No `beta` flags.
9. **Rust/Python split: Python only.**
   - No `sase-core` change and no `sase-core-revision.txt` move.
   - The observed-load parsers read provider-owned formats that change with provider CLI
     releases. They sit next to the Python adapters that produce those runs, where every
     existing transcript reader also lives. Their only consumers in E1 are the CLI and
     doctor; there is no TUI or other frontend yet.
   - Keep the parsers pure (record bytes in, typed observation out) with no CLI
     coupling. Version the JSON with `schema_version: 1`, and make the fixture corpus
     usable as a cross-language conformance suite.
   - **Reopen:** when E6 builds the Rust `ledger_view` that needs observed loads, port
     the parsers there and keep the fixtures as the conformance suite.
10. **The home layer is out of bounds.** No phase delivers, edits, commits, or publishes
    `~/AGENTS.md`, `~/CLAUDE.md`, or the chezmoi source; E5 owns them.

## Phase details

### cli-group (small)

- Add `src/sase/main/parser_instructions.py`, with `register_instructions_parser` and
  `RawDescriptionHelpFormatter` help that includes examples.
  - Register it in `_COMMAND_REGISTRARS` (`src/sase/main/parser_registry.py`) and
    `COMMAND_REGISTRARS_BY_NAME` (`src/sase/main/parser_full_registrars.py`).
  - Add an alphabetical `# --- instructions ---` dispatch block in
    `src/sase/main/entry.py`.
  - Add the handler
    `src/sase/main/instructions_handler.py::handle_instructions_command`.
- `list` behaves exactly like today's `sase memory agent-docs list` (`run_amd_list`).
  - A bare `sase instructions` delegates to `list` through the central default-list
    convention, which prints the delegation notice.
  - The group description documents the bare default, as `sase plan` does.
- Delete `agent-docs` from `parser_memory.py` and `memory_handler.py`.
  - Move the `tests/main/test_memory_agent_docs*.py` coverage to instructions-group
    tests.
  - Update `tests/main/test_parser_command_help.py` (the memory subcommand set), the
    completion snapshot (`just sync-completion-spec`), and any completion kinds.
- Docs:
  - `docs/init.md`: the inventory section and the command table rows.
  - `docs/cli.md`: memory rows, plus a new instructions row.
  - `docs/configuration.md`: the CLI flag tables.
  - The `src/sase/amd/inventory.py` docstrings.
- If any memory note mentions `sase memory agent-docs`, do not edit it; this plan does
  not authorize that. Record `PROPOSED FOLLOW-UP:` on the phase bead instead.
- **Done when:** `sase instructions` and `sase instructions list` print the Agent
  Documents panels, `sase memory agent-docs` is an argparse error, and `just check`
  passes.

### scoreboard (medium; depends on cli-group)

- **Package.** Create `src/sase/instructions/`.
  - Parsers: one module per provider (Claude, Codex, Grok, Muse, agy) plus a shared
    fingerprint module.
  - `_runs.py` finds sessions; a model module holds frozen dataclasses for the
    observation and the provider row; plus a renderer and a JSON encoder.
  - The CLI handler and the doctor checks stay thin.
- **Run enumeration (bounded).**
  - Take the newest SASE runs per provider from the artifact index through
    `listing_snapshot` or its bounded query. Use `-n` (default 20, maximum 200), the
    `--since`/`--until` window, and an optional `project` or `agent` filter.
  - Read only those runs' `agent_meta.json`: `llm_provider`, `workspace_dir`,
    `run_started_at`, and the end time from the run's completion markers.
- **Session location by provider.** Never glob all of `~/.claude/projects` or all of
  `~/.codex/sessions`.

  | Provider | How to find the run's sessions                                                                                                                                                                                                                      |
  | -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
  | Claude   | Encoded-cwd project dir, then top-level `*.jsonl` whose first timestamp falls in the run window. Extract a shared encoder from `session_resolver` rather than duplicating it. Helpers are `<session-id>/subagents/agent-*.jsonl` plus `.meta.json`. |
  | Codex    | Only the `YYYY/MM/DD` directories the run window covers, then `session_meta.cwd == workspace_dir` and a timestamp in the window.                                                                                                                    |
  | Grok     | `<GROK sessions root>/<url-encoded cwd>/*/` within the window. `audience` splits root sessions from subagent sessions.                                                                                                                              |
  | Muse     | `muse_session_id` from `run_metadata.json`, then reuse the `_muse_session_usage` finder.                                                                                                                                                            |
  | agy      | Every instruction column is `◌`. Report directive presence only if the conversation record exposes the prompt.                                                                                                                                      |
  - Resolve every root through overridable environment variables and helpers, such as
    `CLAUDE_CONFIG_DIR`, the real Codex home, a Grok home, and `XDG_DATA_HOME`, so tests
    point them at tmp dirs.

- **Bounded reads.**
  - Root Claude transcripts: stop after the initial attachment block, or after a byte
    cap.
  - Helper transcripts: stream with a 16 MiB cap.
  - Any capped observation is marked `partial`.
  - A run with several provider sessions (wait-continuations, interrupts, declaration
    recovery, commit-repair) contributes one observation per session.
- **Fingerprints** (decision 5).
  - **Contract:** count the loaded sources whose whitespace-flattened text contains the
    `SASE Final Declaration` heading. The count is per source, not per mention.
  - **Home:** the first H1 of the current `~/AGENTS.md`.
  - **Project:** the first H1 of the run's workspace-root `AGENTS.md`.
  - **Directive:** the per-provider markers.
  - **Helpers:**
    - template marker `# SASE Helper Instructions`;
    - guard marker `SASE helper guard:`;
    - accepted marker `Accepted final declaration` in a `tool_result`;
    - an attempt is a Bash `tool_use` whose command matches
      `\bsase\s+final\s+(context|defer|prepare|submit)\b`, or a Skill `tool_use` of
      `sase_final`.
  - Tests assert that the Claude, Codex, Muse, and agy markers occur in the adapters'
    current directive constants. `record` adds the Grok and helper cross-checks.
- **Channels per source.** Count the native and explicit channels separately, so that
  Grok's `<human_rules>` is explicit and not native:
  - natively loaded files: Claude `instructions` attachments, the Codex
    `# AGENTS.md instructions` block, Muse `rules_file`, Grok `agents_md_files`;
  - explicit channels: the Claude `prompt_snapshot` append text, the Codex developer
    message, the Grok `<human_rules>` block in `system_prompt.txt`, and prompt prefixes.
- **Columns per provider row:**
  - provider, runs, and sessions;
  - contract: the modal `N×`, with the spread when it is mixed;
  - home, project, and directive: `✓`, `✗`, or `◌`;
  - native-SASE-full: natively loaded files that carry the contract;
  - foreign: Claude auto-memory present in `prompt_snapshot`, or Grok's memory flag as
    "k/N";
  - helpers: sessions by type, template marker presence, `sase final` attempts, denied,
    and accepted;
  - root guard denials: an alarm column that should always be 0.
- **CLI `sase instructions verify`** (options alphabetical, each with a short alias; per
  `cli_rules`):
  - `-a/--agent NAME`: one agent's runs, with per-session detail.
  - `-H/--helpers`: per-helper rows.
  - `-j/--json`: JSON output.
  - `-n/--limit N`: newest runs per provider.
  - `-p/--provider NAME`: repeatable.
  - `-s/--since WHEN`: a duration such as `24h` or `7d`, or an ISO timestamp; default
    `7d`.
  - `-u/--until WHEN`: same forms as `--since`.
  - Output is a Rich `Panel` and `Table`, with an injectable `console` for tests.
  - `-j` prints `schema_version: 1` JSON: `generated_at`, filters, provider rows, and
    observations when `-a` or `-H` is given.
  - Add completion kinds or value hints for every value-taking option.
  - Exit 0 on success, even when rows show bugs; the command reports, it does not gate.
- **Doctor.** Add `src/sase/doctor/checks_instructions.py`, a deep-only group named
  `instructions`, registered in `src/sase/doctor/runner.py`:
  - `instructions.delivery` uses a small fixed window: newest 10 runs per provider over
    7 days. It WARNs and names every row that breaks contract 1×, project ✓, directive
    ✓, or native-full ≤ 1. It SKIPs when there are no runs.
  - `instructions.helpers` WARNs on any accepted helper declaration or any root guard
    denial.
  - Add both IDs to the deep set in `tests/main/test_doctor_command.py`.
  - `claude-helpers` edits the same set in parallel, so rebase carefully.
- **Fixtures.** Put them in `tests/instructions/fixtures/<provider>/`. They are
  **synthetic, minimal** records that mirror the real shapes, with no real transcript
  text, emails, or tokens.
  - One set per baseline row.
  - Claude helper sets: a general-purpose helper with two contract files and an attempt
    whose result says `Accepted final declaration`, and an Explore helper with no
    instructions.
  - **After-state shapes:** a Grok `<human_rules>` block with the directive and the
    project H1, and a Claude helper with the template marker plus a guard denial.
- **Tests:**
  - The fixtures reproduce the baseline table exactly and the expected after-state rows.
  - The JSON schema.
  - The window excludes a later run's sessions in the same cwd.
  - Caps produce `partial`.
  - Parser and help.
  - The doctor checks.
- **Docs:**
  - `docs/agent_providers.md`: a "Verifying instruction delivery" section that explains
    the columns, symbols, and channels.
  - `docs/cli.md`: the instructions rows and the doctor group.
  - `docs/configuration.md`: the CLI flags.
  - Refresh the completion snapshot.
- **Baseline artifact.** After `just check` passes:
  1. Run `sase instructions verify -j -n 50 --until <this epic bead's created_at>` (and
     the table form). Read the bead with `sase bead read`.
  2. Confirm that it shows Claude 2×, Codex 2×, Muse home ✗, and Grok 0. A row that
     differs must be explained or fixed.
  3. Attach the JSON with
     `sase bead attach <epic-id> <file> -N instructions_baseline.json -n "<summary>"`.

### grok-root (small)

- **Directive.** In `grok.py`, add `_GROK_SINGLE_TURN_DIRECTIVE`. It must begin with
  `SASE single-turn instructions for Grok:`.
  - Model it on Claude's `_SINGLE_TURN_DIRECTIVE` and Codex's directive: one turn,
    nothing wakes you, run commands in the foreground, and handoff commands can take up
    to a minute.
  - Name Grok's actual background and subagent-wait primitives. Check them in a recent
    `system_prompt.txt` tool list.
- **Rules text.** Add `_grok_rules_text(cwd)`.
  - The result is the directive, a blank line, and the project root's `AGENTS.md` text,
    exactly once. This applies when `project_is_sase_managed(cwd)` is true and
    `<discover_project_root(cwd)>/AGENTS.md` exists. Otherwise the result is the
    directive alone.
  - Read the file on every call. Never include `~/AGENTS.md`, other home files, or
    `CLAUDE.md`.
- **Argv.** Append `--rules <text>` inside `_invoke_loop`, before the
  `SASE_LLM_*`/`SASE_GROK_*` extra args, so every continuation cycle carries it. Never
  add `--trust`, and never set `GROK_CLAUDE_AGENTS_ENABLED`.
- **Size guard.** If the UTF-8 rules exceed 120 KiB, raise an actionable error that
  names the file and its size. The Linux per-argument limit is 128 KiB; the precedent is
  agy's `_AGY_PRINT_PROMPT_ARGV_BYTE_LIMIT`.
- **Flag.** Create `grok_rules_delivery` with `sase flag new -k sunset`. Paste its
  registry entry. The Off branch is today's argv exactly.
  - `claude-helpers` also adds a flag in parallel, so expect a trivial registry rebase.
- **Tests** in `tests/llm_provider/test_grok_provider_core.py`:
  - `--rules` appears exactly once;
  - its value is the directive plus the `AGENTS.md` bytes once, with no home H1;
  - `--trust` is absent;
  - a non-managed project gets the directive only;
  - every continuation cycle carries the rules;
  - an over-limit file raises;
  - with the flag off, there is no `--rules`;
  - a live parse probe, gated by `_require_grok_build`, pins that `--rules` parses.
- **Local canary.** Run the adapter once from the workspace root, with a tiny prompt and
  the cheapest Grok model, through the provider registry.
  - Confirm that the new session's `system_prompt.txt` holds `<human_rules>` with the
    directive marker and the project H1 exactly once, and that `prompt_context.json`
    still has `agents_md_files: []`.
  - Record the session path and the result as a note on this phase bead, and add a note
    on `sase-1gj` pointing to it. Do not close `sase-1gj` here.
- **Docs.**
  - Rewrite `docs/agent_providers.md` "Instruction double-load" into "Instruction
    delivery". It should say that workspaces stay untrusted so Grok loads no native
    files, that SASE delivers the directive and the project `AGENTS.md` through
    `--rules`, that the home layer is not delivered until E3, and how to use the kill
    switch.
  - Update `docs/llms.md` Grok "Command Construction" and "Skills and Instruction File".

### claude-helpers (medium)

- **Template.** Add `src/sase/llm_provider/templates/claude_helper_instructions.md`.
  - Static, about 2 KB or less, and the first line is exactly
    `# SASE Helper Instructions`.
  - Base the content on R-fresh's helper overlay:
    - You are a helper spawned by a SASE agent, and your parent owns the turn. Your
      final message is your result.
    - Never run `sase final …` or the root-only skills. Never commit, create beads, or
      launch agents. Text elsewhere that tells "the agent" to do these things is
      addressed to your parent.
    - Run commands in the foreground. Report changed paths, verification, and blockers.
    - Read memory with `sase memory read <note> -r "<why>"`, never by opening
      `sase/memory/` files. Open other repos with `sase repo open`, and read sidecar
      artifacts with `sase artifact read`.
  - No project-specific text.
  - Pass the installed file's path, resolved through `importlib.resources`. There is no
    per-run temp file.
- **Guard.** Add `src/sase/llm_provider/_claude_helper_guard.py`.
  - It is a **stdlib-only** script with no `sase` imports, so the hook can run
    `<sys.executable> -I <path>` quickly.
  - It reads the PreToolUse JSON on stdin. Deny only when `agent_id` is a non-empty
    string and one of these holds:
    - the Bash `tool_input.command` matches the root-only patterns;
    - the Skill tool's input names a root-only skill. Confirm the exact field name from
      a real hook payload.
  - A deny prints
    `{"hookSpecificOutput": {"hookEventName": "PreToolUse", "permissionDecision": "deny", "permissionDecisionReason": "SASE helper guard: <what was blocked and why; return your result to the parent>"}}`
    and exits 0.
  - Every other input, including a missing `agent_id`, other tools, and malformed stdin,
    exits 0 silently.
  - Keep the root-only lists (decision 6) as module constants.
- **Adapter.** In `claude.py` `_invoke_loop`, on every cycle including `--resume`
  cycles, when the flag is on:
  - append `--settings <inline JSON>`:
    `{"hooks":{"PreToolUse":[{"matcher":"Bash|Skill","hooks":[{"type":"command","command":"<shlex-quoted sys.executable> -I <shlex-quoted guard path>","timeout":10}]}]}}`;
  - append `--append-subagent-system-prompt-file <template path>`, only when the
    capability probe reports the flag as supported;
  - never write under the workspace.
- **Capability probe and version guard.**
  - Probe with `claude -p --append-subagent-system-prompt-file <nonexistent> </dev/null`
    under a short timeout. "file not found" means supported; "unknown option" means
    unsupported; anything else means unknown.
  - Cache the result by resolved executable path plus its mtime and size, following the
    usage-preflight cache pattern.
  - Unsupported or unknown: omit the subagent flag and log one warning.
  - Add a doctor deep check `providers.claude_helper_channel` in
    `src/sase/doctor/checks_deep_providers.py` that runs the probe uncached and reports
    OK, or ERROR with next steps.
  - Add a live parse-probe test that skips when `claude` is absent.
- **Flag.** Create `claude_helper_channel` with `sase flag new -k sunset`. The Off
  branch is today's argv exactly.
- **Tests:**
  - Argv on the first cycle and on a `--resume` cycle.
  - The `--settings` value parses as JSON and names the guard path.
  - The template exists and starts with its marker.
  - Flag off.
  - Probe outcomes for supported, unsupported, and unknown, with a fake executable.
  - Guard unit tests:
    - root input is never denied;
    - helper Bash forms `sase final submit x.json`, `.venv/bin/sase final prepare x`,
      and `FOO=1 sase final submit` are denied;
    - each turn-ending CLI form is denied;
    - `sase final status` and `sase final list` are allowed;
    - helper Skill `sase_final` is denied;
    - other tools are allowed;
    - malformed stdin exits 0.
  - `test_claude_hooks.py` stays green.
  - Measure the guard's latency and record its p95 in the phase note; the target is
    under 100 ms.
- **Raw mechanism probes.** Run direct `claude -p` calls with the cheapest model, in a
  scratch directory outside the workspace, using the adapter's argv. Check:
  - (a) a JSON deny blocks the call under `--dangerously-skip-permissions`; if it does
    not, switch to exit code 2 with a stderr reason and retest;
  - (b) a general-purpose helper and an Explore helper both carry `agent_id` and both
    see the template marker;
  - (c) whether a forked helper carries `agent_id`.
  - Record the results as a phase bead note; `record` cites them.
- **Docs.** In `docs/llms.md`, update Claude "Command Construction" and add a "Native
  helpers" subsection covering the template, guard, probe, kill switch, and fork result.
  Add a brief note to the Claude section of `docs/agent_providers.md`.

### record (medium; depends on scoreboard, grok-root, claude-helpers)

- **Template sentence.** Make this the first sentence of `## SASE Final Declaration` in
  `memory-sase.template.md`: "Only the root SASE agent — the provider turn SASE launched
  — submits the final declaration; if you were spawned or forked as a helper (a native
  subagent), return your result to your parent instead."
  - Update the markers in `tests/main/test_init_memory_handler_outputs.py`.
  - Regenerate with the workspace's own build: `.venv/bin/sase memory init --no-commit`.
  - Commit only the sase repo's generated files: `sase/memory/sase.md`, root `AGENTS.md`
    plus its four shims, and `sase/memory/README.md` if it changed.
  - **Do not commit, push, or hand-edit the home layer or the chezmoi source.** If the
    regeneration rewrote home files, list them in a phase bead note.
- **Skill.** Add the same actor qualification ("root agent only; helpers return their
  result") to the description and opening of `src/sase/macros/skills/sase_final.md`.
  - Update the skill-content tests.
  - Preview with `sase skill init --diff`. **Do not deploy**; skills deploy only from
    landed source.
- **Decision record.** Use `/sase_memory_write` (authorized here by name).
  - Create `sase/memory/decisions/helpers-return-roots-declare.md`, titled "Native
    Helpers Return; Only Roots Declare", with `metadata.status: accepted` and
    `decided: <date>`.
  - **Claim:** only the host-launched root submits declarations and runs turn-ending
    operations, while native helpers and forks return their results.
  - **Why:** helpers inherit the env identity; there were accepted helper declarations;
    rejected alternatives were env-based identity, disabling native subagents, and a
    host-side caller check that has no reliable identity.
  - **Cost:** reliance on a hidden flag, the fork result from the probes, and text-only
    coverage for other providers.
  - **Reopens when:** a host-verifiable helper identity or a provider-native helper
    identity appears.
  - Link `[[decisions/host-owned-completion]]`, `[[decisions/single-turn-agents]]`, and
    `[[decisions/adapters-normalize-harnesses]]`.
  - Regenerate so the roster in `sase/memory/decisions.md` and `AGENTS.md` picks it up.
- **Marker-consistency tests.** The scoreboard's Grok directive marker, helper template
  marker, and guard marker must equal what `grok-root` and `claude-helpers` shipped.
  Import both sides.
- **Capability docs.** Add a "Root and helper agents" section to
  `docs/agent_providers.md`, as a per-provider table:

  | Provider  | Helper coverage                                                                            |
  | --------- | ------------------------------------------------------------------------------------------ |
  | Claude    | template plus guard; the fork result                                                       |
  | Codex     | v1 children inherit `developer_instructions`, so the sentence covers them; 0 spawns so far |
  | Grok      | subagents get nothing; sentence only, with the child channel deferred                      |
  | Muse, agy | none                                                                                       |

  This satisfies "limits listed by capability, not hidden".

- **Ownership inventory.** Write `docs/instruction_inventory.md`. Add it to the
  `mkdocs.yml` nav, and link it from `docs/agent_providers.md` and `docs/init.md`.
  - One row per surface: project or repo, paths, generated or hand-written, count,
    disposition (`migrate` | `projection` | `repo-owned` | `deferred`), reason, and the
    owning epic.
  - **Generated, memory-backed (35):** `sase` 20 (root plus three nested scopes), home
    5, `bob-cli` 5, `actstat` 5.
  - **Hand-written:** `sase-core` 3, the plugin repos 2 each, `sase-github` 1. Verify
    each count.
  - **Config-based discovery surfaces:** Codex's shadow-home `AGENTS.md` link and any
    provider config that points at `~/AGENTS.md`.
  - Gather read-only, with `git ls-files`, `ls ~`, and
    `sase repo open <repo> -r "<why>"`. **Modify none of those repos.**

### acceptance (small; depends on record)

- **Precondition.** The host must run the landed code. Find the primary checkout with
  `sase repo list`, then confirm with `git merge-base --is-ancestor` that it contains
  the `grok-root`, `claude-helpers`, and `record` commits. If it does not, record that
  and stop.
- **Probes.** Use `/sase_run` to submit **one** LaunchApproval request with two
  segments, `max_slots: 2`, and the default `resume_requester` continuation. The
  checkpoint is "verify both probes with the scoreboard, attach the acceptance record,
  close sase-1gj". Preflight with `sase macro expand`.
  - **Grok probe:** `+sase`, pinned to a Grok model, xsmall. Without reading any file,
    it quotes the H1 and the `## SASE Final Declaration` sentence from its own
    instructions, then finishes normally with `/sase_final`.
  - **Claude probe:** `+sase`, pinned to the cheapest Claude model. It spawns one
    general-purpose helper and one Explore helper. Each helper runs
    `sase final submit /dev/null` through Bash and reports the output verbatim. The root
    then finishes normally with `/sase_final`.
- **On resume, run and save:**
  - `sase instructions verify -a <grok-probe> -j`
  - `sase instructions verify -a <claude-probe> -H -j`
  - `sase instructions verify --since <first E1 landing> -j`
  - `sase final status <claude-probe>`
- **Check against the exit criteria below.** If the scoreboard itself is wrong, fix it
  in this phase. Record any other failure as `DISCOVERED ISSUE:` on this phase bead, and
  leave the bead open, naming the failed criterion.
- **Acceptance record.** Write a short Markdown record with:
  - source revisions;
  - `claude`, `grok`, and `codex` versions;
  - the commands that were run;
  - references to the baseline and after JSON;
  - expected versus observed results;
  - declared gaps: forks, and helpers of other providers.
  - Attach the record and the after JSON to the epic bead with `sase bead attach`.
- **Close `sase-1gj`** only when the Grok probe passes: `sase bead close sase-1gj` with
  resolution `superseded` and a reason that points to this epic. Check the flags with
  `-h`.
- If the approval is rejected or times out, note on the epic that the probes did not
  run, attach what exists, and leave both this phase bead and `sase-1gj` open.

## Exit criteria (checked at landing)

- [ ] `just check` passes on the landed tree.
- [ ] Scoreboard fixture tests reproduce the baseline table and the after-state rows.
- [ ] The attached baseline JSON (`--until` the epic's creation) shows Claude 2×, Codex
      2×, Muse home ✗, and Grok 0.
- [ ] The Grok probe shows contract 1×, project ✓, directive ✓, and native-full 0, and
      its `prompt_context.json` keeps `agents_md_files: []`. Adapter tests prove the
      argv carries `--rules` once and never `--trust`.
- [ ] The Claude probe shows both helpers (general-purpose and Explore):
  - each carries the `# SASE Helper Instructions` marker;
  - each `sase final submit` attempt is denied with `SASE helper guard:`;
  - zero helper declarations are accepted;
  - the root's own declaration is accepted (`sase final status`);
  - root guard denials are 0.
- [ ] Fork and other-provider helper limits are listed by capability in
      `docs/agent_providers.md`.
- [ ] `docs/instruction_inventory.md` gives a disposition for every surface.
- [ ] `sase flag list` shows the `grok_rules_delivery` and `claude_helper_channel`
      sunset flags, and both-state tests exist for each.
- [ ] `sase doctor -D -C instructions` runs and reports, and
      `sase doctor -D -C providers.claude_helper_channel` is OK on athena.
- [ ] `sase memory init --check` is clean for the project. The root `AGENTS.md` and its
      shims contain the actor-qualified sentence, and
      `sase memory read decisions:helpers-return-roots-declare` works.
- [ ] `sase instructions` replaces `sase memory agent-docs`, and the docs say so.
- [ ] The baseline JSON, after JSON, and acceptance record are attached to the epic
      bead, and `sase-1gj` is closed as superseded.

## Watch metrics (at least 3 days after landing; not landing criteria)

- Grok's non-research final-declaration rate. It was 60% before the regression and 26%
  after.
- Claude helper `sase final` attempts, which should all be denied, and accepted helper
  declarations, which should be 0.
- Root guard denials, which must stay 0. Any non-zero value means
  `sase flag disable claude_helper_channel` on every host.
- Grok run failures caused by `--rules`, such as argv errors.

## Confirm it yourself (about 10 minutes, after deployment)

```bash
sase instructions verify --since 24h            # Grok row flips 0 → 1×, project ✓, directive ✓
sase instructions verify -p claude -H           # helper template ✓, attempts denied, 0 accepted
sase doctor -D -C instructions                  # delivery + helper checks
sase flag list | grep -E 'grok_rules_delivery|claude_helper_channel'
```

## Expected scoreboard after this epic

```
provider contract home project directive native-SASE-full foreign      helpers
claude      2×     ✓     ✓        ✓          2           auto-memory  gp+Explore: helper template ✓; sase final attempts denied; 0 accepted
codex       2×     ✓     ✓        ✓          2           —            0 spawns
muse        1×     ✗     ✓        ✓          1           —            0 spawns
grok        1×     ✗     ✓        ✓          0           memory-v2½   children: nothing (deferred spike)
agy         ◌      ✗     ◌        ✓          ◌           —            —
```

## Rollback

`sase flag disable grok_rules_delivery` and/or `sase flag disable claude_helper_channel`
on each execution host (athena, apollo, mac), or revert the adapter commits. No
committed instruction file is deleted. The template sentence and the decision record are
inert text.

## Memory edits (complete list; no phase may touch any other memory file)

1. `src/sase/main/init_memory/templates/memory-sase.template.md`: the actor-qualified
   sentence (code template; `record`).
2. Files regenerated by `.venv/bin/sase memory init --no-commit` in `record`:
   `sase/memory/sase.md`, `sase/memory/decisions.md` (roster), `sase/memory/README.md`
   if it changes, and the root `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `QWEN.md`, and
   `OPENCODE.md`.
3. New strand `sase/memory/decisions/helpers-return-roots-declare.md` (`record`, via
   `/sase_memory_write`).

The skill source `src/sase/macros/skills/sase_final.md` is not memory, but `record`
edits it and must not deploy it.

## Out of scope

- Delivering the home layer to any provider, suppressing native files, and switching
  Claude auto-memory or Grok memory v2 off (E3).
- Bundles and manifests (E2).
- Deleting any committed instruction file, including `QWEN.md` and `OPENCODE.md` (E4).
- Home and other-project files (E5).
- Any TUI work (E6).
- Codex, Muse, and agy argv changes.
- Child channels for Grok, Codex, and Muse (deferred spike tasks).
