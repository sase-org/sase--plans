---
tier: epic
title: Vars-driven commit finalization with exclusion-based staging
goal: 'Agents record commit intent (message, exclusions) as sase variables via `sase
  commit --vars`, the finalizer executes the commit deterministically before agent
  completion is recorded, and file staging becomes exclusion-based so changes the
  agent did not make surface as anomalies.

  '
phases:
- id: list-vars-rust
  title: List-valued output variables in the sase-core scan wire
  depends_on: []
  size: medium
  description: 'list-vars-rust: extend the Rust agent-scan wire and scanner coercion
    in the sase-core repo so output variables carry string-or-list-of-string values,
    with parity tests and a released version bump.'
- id: list-vars-python
  title: List-valued sase variables in Python storage, CLI, and consumers
  depends_on:
  - list-vars-rust
  size: medium
  description: 'list-vars-python: teach agent_output_variables storage, sase var set,
    publication validation, wire loaders, TUI/sidecar rendering, and the Jinja context
    to accept list values, and bump the sase-core-rs pin.'
- id: commit-exclusions
  title: Exclusion-based file selection for sase commit
  depends_on: []
  size: medium
  description: 'commit-exclusions: replace the public -f include flags with -x exclusion
    flags, stage via git pathspec excludes, keep a hidden internal include flag for
    tooling, surface exclusion warnings, and update wrapper, skills, prompts, callers,
    docs, and tests.'
- id: commit-vars-option
  title: sase commit --vars intent recording
  depends_on:
  - list-vars-python
  - commit-exclusions
  size: medium
  description: 'commit-vars-option: add a --vars flag that records commit intent as
    reserved commit_* sase variables (multiline message, exclusion list) instead of
    committing, deleting the -M message file on success, with wrapper, skill, and
    test updates.'
- id: finalizer-vars-commit
  title: Finalizer executes recorded commit intent before completion
  depends_on:
  - commit-vars-option
  size: medium
  description: 'finalizer-vars-commit: have the commit finalizer prompt for --vars,
    execute recorded intent deterministically via a sase commit subprocess, tolerate
    exclusion-only residual dirt with warnings, clear consumed intent vars, and add
    ordering regression tests.'
create_time: 2026-07-30 16:05:54
status: wip
bead_id: sase-be
---

- **PROMPT:** [202607/prompts/commit_vars_finalizer.md](prompts/commit_vars_finalizer.md)
- **BEAD:** [sase-be](https://github.com/sase-org/sase--beads/blob/main/pages/sase-be/README.md)

# Plan: Vars-driven commit finalization with exclusion-based staging

## Motivation

Today the post-completion commit finalizer asks the agent to run `/sase_git_commit` → `sase commit`, and the agent
chooses which files to stage with repeated `-f/--file` flags (omitting `-f` stages everything). This inverts the trust
model we actually want:

- Agent file changes are isolated to the agent's own workspace clone, so **everything dirty in the workspace should be
  the agent's work**. Making the agent enumerate its own files invites omissions; instead the agent should only name
  files it wants **excluded** — and any exclusion is a signal that something touched the workspace that the agent did
  not author, which we want surfaced loudly, not silently skipped.
- The actual `git commit` is performed by the LLM shelling out mid-conversation. Moving the mechanical commit into the
  finalizer (driven by values the agent records with a new `sase commit --vars` mode) makes commits deterministic,
  observable, and guarantees they happen strictly **before** the agent's completion notification and terminal status.

## Current state (verified anchors; line numbers as of current master)

- **CLI**: `src/sase/main/parser_commit.py` (`register_commit_parser`, all options, no positionals; `-f/--file` append
  at :23-30; `-s/--status` at :54-60). Handler `src/sase/main/commit_handler.py:20-132` builds
  `payload = {"message", "files", ...}`; **latent bug**: `args.status` is parsed but never placed in the payload, so
  `-s` is a no-op and only `SASE_PR_STATUS` reaches `src/sase/workflows/commit/commit_tracking.py:578-582`
  (docs/commit_workflows.md claims otherwise). Message files (`-M`) are read, `.rstrip()`-ed, and deleted only on
  success (`commit_handler.py:106-110`); preserved with a retry hint on failure (:118-123).
- **Staging**: `src/sase/vcs_provider/plugins/_git_commit_dispatch.py` — `vcs_create_commit` :332-374 and
  `vcs_create_pull_request` :390-433 run `git add -- <files>` when `payload["files"]` is non-empty, else `git add -A`.
  `vcs_create_proposal` :377-387 saves the whole diff and cleans the workspace (no staging). Pure Python; no
  `sase_core_rs` in the commit path.
- **Workflow**: `src/sase/workflows/commit/workflow.py` (`CommitWorkflow.run()` at :85 uses `os.getcwd()` at :113;
  conflict → `RunResult.CONFLICT` → exit code 2; checkpoint + `--resume` via `workflow_resume.py`). Result marker
  `commit_result.json` written by `commit_tracking.write_result_marker`.
- **Wrapper**: `src/sase/scripts/sase_git_commit` (bash) writes a `commit_skill_invoked.json` marker (its embedded
  `_parse_args` understands `-f/--file`, `-t/--type`, `-r/--resume`) then `exec`s `sase commit "$@"`.
- **In-repo callers**: `src/sase/axe/run_agent_exec_plan_sdd.py:82-84` invokes `sase commit -M <path> -f <file>...` for
  SDD plan-status commits. `src/sase/ace/restore.py:219-223` invokes `sase commit <base_name>` with a positional
  argument **the parser does not define** (already broken; masked by mocks in `tests/test_restore.py:211-235`).
- **Skill sources** (generated, per `sase skill init`): `src/sase/xprompts/skills/sase_git_commit.md` (documents
  `-M`/`-m`/`-f`, the wrapper, exit codes, `--resume` recovery) and `src/sase/xprompts/skills/sase_hg_commit.md` (raw
  `sase commit` form). Contract-pinned by
  `tests/main/test_init_skills_sources.py::test_git_commit_skill_invokes_observable_wrapper` (:389) which asserts exact
  phrases including the `-f` guidance.
- **Finalizer**: `src/sase/llm_provider/commit_finalizer.py::run_commit_finalizer` (:128-318) runs inline in
  `src/sase/llm_provider/_invoke.py:308-317` immediately after a successful provider turn — i.e. **before** `done.json`
  (`src/sase/axe/run_agent_exec_finalize.py:619`), before status/claim release
  (`src/sase/axe/run_agent_runner_lifecycle.py:157-269`), and before the completion notification
  (`run_agent_runner_lifecycle.py:271-297` → `src/sase/axe/run_agent_runner_finalize.py::send_completion_notification`
  :221-377). If the repo is still dirty after `max_passes` (default 2, `src/sase/default_config.yml:792-794`) it raises
  `CommitFinalizerError`, converting the run to failed. Prompt text lives in `src/sase/commit_instructions.py:106-153`
  (currently: "use one `-f` flag for each listed file") and `src/sase/llm_provider/commit_finalizer_prompting.py`
  (non-primary repos at :75-94). Family members each run their own runner process and their own finalizer, so per-member
  ordering extends to families automatically.
- **sase variables**: `src/sase/core/agent_output_variables.py` stores a flat `dict[str, str]` under `output_variables`
  in the agent's `agent_meta.json` (flock + atomic write; CRLF→LF normalization; NUL rejection; 8 KiB per value —
  `MAX_OUTPUT_VARIABLE_VALUE_BYTES` :20). **Multiline strings: supported. List values: NOT supported** —
  `_string_output_variables` (:102-109) silently drops non-string values, publication validation rejects them
  (`src/sase/agents_sync/v2_validation.py:95-124`), the sidecar sanitizer drops them
  (`src/sase/agents_sync/inventory_io.py:33-66`), and the Rust scanner in the sase-core repo coerces to
  `BTreeMap<String, String>` (`crates/sase_core/src/agent_scan/wire.rs:242`, `coerce_str_str_map` in
  `crates/sase_core/src/agent_scan/scanner.rs:846-864`, used at :958; parity test
  `crates/sase_core/tests/agent_scan_parity.rs:1045-1080` asserts non-strings are dropped). CLI:
  `sase var set KEY=VALUE ...` or `sase var set KEY -v TEXT | -f PATH|-` (`src/sase/main/parser_var.py`,
  `src/sase/main/var_handler.py`; requires `SASE_AGENT=1` + `SASE_ARTIFACTS_DIR`). Downstream consumers: Jinja `agents`
  namespace (`src/sase/agent/output_variable_context.py`), `%repeat` STOP (`src/sase/axe/run_agent_repeat_stop.py`),
  completion notifications (`run_agent_runner_finalize.py::_completion_output_variables` :191-205, which filters
  `STOP`), ACE TUI rendering (`src/sase/ace/tui/widgets/prompt_panel/_agent_output_variables.py`, loaders in
  `src/sase/ace/tui/models/_loaders/`), clan/tribe aggregation
  (`src/sase/ace/tui/models/_agent_clan_sections.py:337-354`), and agents-sidecar publication
  (`src/sase/agents_sync/rendering_variables.py`).

## Design decisions

1. **Exclusion-based staging replaces include-based staging on the public CLI.** `-f/--file` is removed; new
   `-x/--exclude` (repeatable) names paths to leave uncommitted. Default (no `-x`) stages everything, as omitting `-f`
   does today. The `payload["files"]` include path survives **only** for internal tooling via a hidden
   `--internal-stage-file` flag (`help=argparse.SUPPRESS`; the CLI-rules short-alias requirement exempts internal
   subprocess arguments). This is a breaking CLI change and must be marked as such (`feat(commit)!:` / BREAKING CHANGE
   footer) when committed.
2. **Exclusions are anomalies, not conveniences.** Skills and finalizer prompts instruct agents to exclude only files
   they did not author. Non-empty exclusions are echoed as a colored CLI warning, persisted in the commit checkpoint and
   `commit_result.json`, recorded in `commit_finalizer_result.json`, and surfaced as a warning marker in the agent's
   completion notification.
3. **`sase commit --vars` records intent; it never commits.** It validates like a real commit (message required,
   method-conflict check), writes reserved `commit_*` variables through the existing `set_agent_output_variables` API,
   deletes the `-M` message file on success (preserving it on failure with the existing retry hint), and exits 0.
   Reserved keys: `commit_message` (multiline string), `commit_exclude_files` (list), `commit_repo_root` (repo root of
   the invoking cwd), plus passthroughs recorded only when the corresponding flag was explicitly given: `commit_name`,
   `commit_parent`, `commit_bug_id`, `commit_status`, `commit_method` (canonicalized).
4. **The finalizer executes recorded intent via a `sase commit` subprocess** run in the project dir (not in-process:
   `CommitWorkflow.run()` binds to `os.getcwd()` and the CLI layer owns run-log events, conflict exit codes, and message
   handling). Exit 0 → intent vars cleared and success recorded; exit 2 → next finalizer pass prompts the existing
   conflict-resolution + `--resume` flow; other failures → next pass prompts with captured stderr; still-unresolved
   after `max_passes` → existing `CommitFinalizerError` path (run recorded failed, no completion notification). Because
   the finalizer already runs strictly before `done.json`/status/notification, executing commits inside it satisfies
   "completion only after commits" for single agents and family members alike — this phase adds regression tests pinning
   that ordering.
5. **Exclusion-aware clean check.** After a successful intent commit, residual primary-repo dirt that is a subset of the
   recorded exclusions counts as finalized (new result reason `finalized_with_exclusions`, with warnings) instead of
   failing after max passes. Dirt outside the exclusion set keeps today's behavior.
6. **v1 scope: the vars flow targets the primary workspace repo.** Sibling/linked/external/SDD repo commits keep the
   existing direct `/sase_git_commit` instructions (`commit_finalizer_prompting.py:75-94`). `commit_repo_root` is
   recorded so the finalizer executes an intent only when it matches the primary project dir; a mismatch produces a
   warning and falls back to direct instructions on the next pass. Extending vars-driven commits to non-primary repos is
   deliberate future work (it needs per-repo intent keying).
7. **List values become first-class variable values (`str | list[str]`).** Normalization (CRLF→LF, NUL rejection, 8 KiB
   UTF-8 cap) applies per element; a new list-length cap (`MAX_OUTPUT_VARIABLE_LIST_ITEMS = 512`) bounds pathological
   lists; merge semantics stay whole-value replace (a list replaces a scalar and vice versa). On the CLI, repeating
   `-v/--value` builds a list (a single `-v` remains a scalar string — setting a one-element list is API-only in v1,
   which is fine for this feature since `--vars` writes programmatically). `KEY=VALUE` positional assignments stay
   scalar. The `STOP` variable remains scalar-only (list-valued STOP is ignored by `detect_repeat_stop`).
8. **The 8 KiB per-value cap stays.** It is generous for commit messages; `--vars` exits 1 with a clear "shorten the
   commit message" error when exceeded rather than growing the cap.

## Phase details

### list-vars-rust — sase-core scan-wire list support

Work happens in the **sase-core linked repo**. Open it with the `/sase_repo` skill
(`sase repo open sase-core -r "<reason>"`) and use only the printed path.

1. `crates/sase_core/src/agent_scan/wire.rs:242` — change `output_variables: BTreeMap<String, String>` to
   `BTreeMap<String, OutputVariableValue>` where `OutputVariableValue` is a new `#[serde(untagged)]` enum
   `{ Text(String), List(Vec<String>) }` (derive the same traits the wire structs already use).
2. `crates/sase_core/src/agent_scan/scanner.rs` — add a dedicated coercion (e.g. `coerce_output_variable_map`) used at
   :958 that accepts `Value::String` and `Value::Array` whose elements are all strings; entries with any other shape
   (numbers, nested objects, mixed arrays) are dropped, matching the Python-side semantics defined in list-vars-python.
   Leave `coerce_str_str_map` untouched for `output_types` (:1093, :1167).
3. Confirm the pyo3 crate (`crates/sase_core_py`) needs no code change: scan snapshots cross the boundary as JSON-shaped
   dicts, so list values flow through; update its doc comments if they spell out the `output_variables` shape.
4. Tests: update `crates/sase_core/tests/agent_scan_parity.rs::running_record_carries_string_output_variables`
   (:1045-1080) and add cases: `["a", "b"]` preserved in order; `[1, "b"]`, nested arrays, and objects dropped; empty
   list preserved as empty list.
5. Bump crate/package versions and land through the sase-core repo's normal release flow so a new `sase-core-rs` version
   is publishable (the sase repo currently pins 0.14.2 — see sase commit 5a8dc1cba for the pin-bump pattern the next
   phase follows).
6. Verification: sase-core's own test/lint flow (cargo test for `sase_core`, plus that repo's just/CI conventions).

### list-vars-python — Python-side list support and pin bump

1. `pyproject.toml`: bump the `sase-core-rs` requirement to the version released by list-vars-rust (pattern: commit
   5a8dc1cba).
2. `src/sase/core/agent_output_variables.py`:
   - Introduce `OutputVariableValue = str | list[str]` and thread it through `set_agent_output_variables` /
     `read_agent_output_variables`.
   - Normalize per element; add `MAX_OUTPUT_VARIABLE_LIST_ITEMS = 512`; reject non-string elements with `ValueError` on
     write; replace `_string_output_variables` with a portable reader that keeps `str` values and lists whose elements
     are all `str`, dropping others.
   - Add `clear_agent_output_variables(artifacts_dir, keys)` (locked, atomic, reindexing — mirror
     `set_agent_output_variables`) for finalizer consumption in finalizer-vars-commit.
3. CLI (`src/sase/main/parser_var.py`, `src/sase/main/var_handler.py`): make `-v/--value` repeatable; two or more
   occurrences produce a list value; update help text (keep options alphabetized, colored output conventions).
4. Publication/validation: `src/sase/agents_sync/v2_validation.py:95-124` and
   `src/sase/agents_sync/inventory_io.py:33-66` accept `str | list[str]` with element-wise limits (same 8 KiB/item and
   list-length caps; 256-key cap unchanged).
5. Wire/read consumers: `src/sase/core/agent_scan_wire_markers.py:138` type update;
   `src/sase/ace/tui/models/_loaders/_meta_enrichment_wire.py:80`, `_meta_enrichment_filesystem.py:110-111`, and
   `_meta_enrichment_common.py::string_output_variables` (:158-165) accept both shapes (the loaders must tolerate old
   string-only and new list payloads regardless of source).
6. Rendering: `src/sase/agents_sync/rendering_variables.py` and
   `src/sase/ace/tui/widgets/prompt_panel/_agent_output_variables.py` render list values one element per line, reusing
   the existing multiline-value presentation and truncation rules. Caution: `aggregate_agent_output_variables`
   (`src/sase/ace/tui/models/_agent_clan_sections.py:337-354`) and tribe summaries group by value — lists are
   unhashable, so group on a canonical form (tuple or rendered string).
7. Jinja context (`src/sase/agent/output_variable_context.py`): lists pass through so templates can iterate
   `{{ agents["x"].some_list }}`; update type hints. `src/sase/axe/run_agent_repeat_stop.py`: type-guard so a
   list-valued `STOP` is ignored.
8. Skill + docs: `src/sase/xprompts/skills/sase_var.md` documents list values (repeated `-v`) and the reserved
   `commit_*` names landing later in this epic can be omitted here; `docs/configuration.md:3082-3117`, `docs/xprompt.md`
   (cross-agent output variables section), `docs/agents_sidecar.md`, `docs/workflow_spec.md:696-719` as applicable.
9. Tests: extend `tests/main/test_var_handler.py` (repeated `-v`, list round-trip, caps),
   `tests/test_agent_output_variable_context.py`, `tests/test_core_agent_scan_wire.py:166-247` (list wire cases),
   agents_sync validation/sanitizer tests, `tests/ace/tui/widgets/test_agent_display_output_variables.py` (list
   rendering), STOP guard test in `tests/test_run_agent_repeat_stop.py`.
10. Verification: `just install`, targeted pytest for every touched suite, then `just check`.

### commit-exclusions — exclusion-based staging

1. Parser (`src/sase/main/parser_commit.py`): remove `-f/--file`; add `-x/--exclude` (append; help: excluded paths stay
   uncommitted and signal changes the agent did not make); add hidden `--internal-stage-file` (append,
   `help=argparse.SUPPRESS`); keep listed options alphabetized and help text excellent per CLI rules. Wire `-s/--status`
   into the payload (`payload["status"]`) so the flag matches its documented behavior (`commit_tracking.py:578-582`
   already prefers `payload["status"]` over `SASE_PR_STATUS`).
2. Handler (`src/sase/main/commit_handler.py`): payload gains `exclude_files` from `-x`; `files` is populated only from
   `--internal-stage-file`; passing both `-x` and `--internal-stage-file` is an error.
3. Staging (`src/sase/vcs_provider/plugins/_git_commit_dispatch.py`):
   - `vcs_create_commit` (:332) and `vcs_create_pull_request` (:390): when `exclude_files` is present run
     `git add -A -- . ':(exclude)<path>'` (one pathspec per excluded path, executed from the repo root) so modified,
     deleted, and untracked files are handled uniformly; empty `exclude_files` keeps plain `git add -A`; the internal
     `files` include branch stays as-is. `_validate_staged` unchanged (a commit that excludes everything fails with "no
     staged changes", which is correct).
   - `vcs_create_proposal` (:377): apply exclusions to the saved diff and leave excluded paths in place during workspace
     cleanup if straightforward; if that proves invasive, error clearly when `exclude_files` is combined with
     `create_proposal` (decide in-phase; erroring is acceptable for v1 — document whichever lands).
4. Warnings/records: non-empty exclusions print a colored warning listing the paths ("excluded paths remain uncommitted;
   changes not authored by this agent may indicate a problem"); persist `excluded_files` in the checkpoint
   (`src/sase/workflows/commit/checkpoint.py`) and in `commit_result.json` (`commit_tracking.write_result_marker`) so
   downstream phases and `--resume` see them.
5. Callers:
   - `src/sase/axe/run_agent_exec_plan_sdd.py:82-84`: switch `-f` → `--internal-stage-file`.
   - `src/sase/ace/restore.py:219-223`: fix the broken positional invocation to a valid `sase commit` form (message via
     `-m`/`-M`; staging semantics the restore flow actually needs) and replace the mock-only coverage in
     `tests/test_restore.py:211-235` with an assertion on the real argv.
   - Wrapper `src/sase/scripts/sase_git_commit`: update the embedded `_parse_args` marker parsing (`-f` →
     `-x`/`--exclude`); argv forwarding stays verbatim.
6. Skills + prompts (same-change contract per the generated-skills memory):
   - `src/sase/xprompts/skills/sase_git_commit.md` and `sase_hg_commit.md`: replace the `-f` include
     instructions/examples with the exclusion contract — default commits every change including untracked files; use
     `-x` only for files the agent did not author; excluding a file is an anomaly worth mentioning in the agent's reply.
     Treat both runtimes uniformly.
   - `src/sase/commit_instructions.py:106-153`: replace the "-f per listed file" guidance with exclusion guidance
     ("commit everything; if a listed change is not yours, exclude it with -x and say why").
7. Docs: `docs/commit_workflows.md` (args table, payload notes, `-s` fix note), `docs/vcs.md:239-261` plus the stale
   `sase commit my_feature` positional forms (:80, :601, :698), `docs/cli.md:242`, `docs/change_spec.md:203`,
   `docs/configuration.md:2321, 2516`.
8. Tests: rework `tests/test_commit_cli.py` (exclusion matrix, internal include flag, `-x` + internal conflict, `-s`
   payload), staging behavior in `tests/workflows/test_commit_add.py` (pathspec excludes cover
   modified/deleted/untracked), `tests/test_sase_git_commit_wrapper.py` marker shape,
   `tests/main/test_init_skills_sources.py:389` phrase asserts, `tests/test_commit_instructions.py`,
   `tests/test_sdd_commit_plan_accept.py` argv shape, proposal-exclusion behavior tests per the in-phase decision.
9. Verification: `just install`, targeted pytest, `just check`. Commit message must carry the breaking-change marker for
   the `-f` removal.

### commit-vars-option — record intent as variables

1. Parser: add `-V/--vars` (`store_true`; short alias per CLI rules; alphabetized). Invalid combinations: `--vars` with
   `--resume` or `--internal-stage-file` → exit 1.
2. Handler (`src/sase/main/commit_handler.py`): a `--vars` branch after message resolution and method-conflict
   validation but before workflow construction:
   - Require agent context like `sase var set` does (`SASE_AGENT=1` + `SASE_ARTIFACTS_DIR`; clear error otherwise).
     Require a message (`-m` or `-M`).
   - Write via `set_agent_output_variables`: `commit_message`, `commit_exclude_files` (only when `-x` given),
     `commit_repo_root` (repo root of cwd), and explicit-flag passthroughs `commit_name`, `commit_parent`,
     `commit_bug_id`, `commit_status`, `commit_method` (canonicalized through `METHOD_ALIASES`).
   - On success: delete the `-M` file (same semantics as a successful commit today), print a summary of the recorded
     keys, exit 0. On failure (e.g. oversized message → "shorten the commit message"): preserve the `-M` file with the
     existing retry hint, exit 1.
3. Wrapper `src/sase/scripts/sase_git_commit`: `_parse_args` records `vars: true` in the invocation marker.
4. Skills: `sase_git_commit.md` and `sase_hg_commit.md` gain a brief `--vars` explanation — it records commit intent
   (message + exclusions) as sase variables for the post-completion finalizer to execute, and is the expected mode when
   the finalizer asks for it; the full finalizer flow text lands in finalizer-vars-commit.
5. Docs: flag rows in `docs/commit_workflows.md` and `docs/cli.md`; reserved `commit_*` variable names in
   `docs/configuration.md`'s variable section and `sase_var.md`'s rules list (reserved like `STOP`).
6. Tests: `tests/test_commit_cli.py` — `--vars` writes the expected variables (including the list-valued exclusions),
   deletes the message file, constructs no `CommitWorkflow`, errors outside agent context, rejects invalid flag
   combinations, records passthroughs only when flagged; wrapper marker test; skill-source phrase asserts.
7. Verification: `just install`, targeted pytest, `just check`.

### finalizer-vars-commit — deterministic finalizer commits and ordering

1. Prompt changes (`src/sase/commit_instructions.py`, `src/sase/llm_provider/commit_finalizer_prompting.py`):
   primary-repo instructions ask the agent to write the commit message file and run the commit skill with `--vars` (plus
   `-x` for any listed change it did not author) instead of committing directly. Non-primary repos
   (siblings/linked/external/SDD stores) keep the existing direct-commit instructions (design decision 6).
2. Intent execution (`src/sase/llm_provider/commit_finalizer.py`): add
   `_execute_pending_commit_intent(project_dir, artifact_root)` called (a) right after the initial dirty-state
   collection — the agent may have recorded intent during its main turn — and (b) after each pass's dirty-state
   re-collection:
   - Read `commit_*` variables from the agent artifacts dir. Skip unless `commit_message` is present and the primary
     repo is dirty.
   - If `commit_repo_root` does not resolve to the primary project dir: warn, record the mismatch in the result
     artifact, do not execute, fall back to direct instructions.
   - Run `sase commit` as a subprocess in `project_dir`: `-m <commit_message>`, one `-x` per `commit_exclude_files`
     element, passthrough flags for recorded `commit_name` / `commit_parent` / `commit_bug_id` / `commit_status`; pass
     `-t <commit_method>` (with `SASE_COMMIT_METHOD_ALLOW_OVERRIDE=1`) only when `commit_method` was recorded — it was
     validated against the env at record time.
   - Exit 0 → `clear_agent_output_variables` for all consumed `commit_*` keys; record success + exclusions in
     `commit_finalizer_result.json`. Exit 2 → keep the vars; the next pass prompt is the existing conflict-resolution +
     `sase_git_commit --resume` guidance. Other exits → keep the vars; include captured stderr in the next pass prompt.
     Exhausted passes → existing `CommitFinalizerError` (run fails; no completion notification).
3. Exclusion-aware clean evaluation: after successful intent execution, if the remaining primary-repo dirty files are a
   subset of the recorded exclusions, treat the primary repo as finalized (result status `finalized`, reason
   `finalized_with_exclusions`, excluded files listed in the result artifact) instead of iterating to failure; dirt
   outside the exclusion set keeps today's loop/failure behavior.
4. Anomaly surfacing at completion (`src/sase/axe/run_agent_runner_finalize.py`): `send_completion_notification`
   (:221-377) appends a visible warning to the notification notes when `commit_finalizer_result.json` records exclusions
   (e.g. "N files excluded from commit — review workspace"); `_completion_output_variables` (:191-205) filters
   `commit_*` intent keys the same way it filters `STOP`, as defense in depth for uncleared vars.
5. Ordering regression tests: the finalizer already runs before `done.json` and the completion notification
   (`_invoke.py:308` → `run_agent_exec_finalize.py:619` → `run_agent_runner_lifecycle.py:271-297`), and family members
   run per-member finalizers. Add tests pinning that (a) the completion notification fires only after successful intent
   execution, (b) intent-execution failure after max passes yields a failed run with no completed outcome/notification,
   and (c) a family-member run behaves identically.
6. Docs: `docs/commit_workflows.md` finalizer flow (:452-478), `docs/llms.md:145-161`, `docs/notifications.md` (warning
   marker), and the stale "Stop hook" naming notes in `docs/images/commit-workflow-infographic.*.md` if trivially
   editable (no image regen).
7. Tests: new finalizer suites for intent execution (success/conflict/failure/mismatched repo-root/pre-pass-1
   execution), var clearing, exclusion-tolerated residual dirt, prompt text; update `tests/test_commit_instructions.py`
   and `tests/llm_provider/test_commit_finalizer_*.py`; ordering tests alongside the existing runner finalize coverage.
8. Verification: `just install`, targeted pytest, `just check`.

## Cross-cutting rules for every phase

- Phases touching this repo must run `just install` before other just commands (ephemeral workspaces) and `just check`
  before finishing. The sase-core phase uses that repo's own test/lint flow.
- Any `sase commit` CLI argument change ships in the same change as its in-repo callers/wrappers, the skill `SKILL.md`
  sources that document it, and the tests validating both (CLI/Skill Contract Synchronization rule).
- Skill sources under `src/sase/xprompts/skills/` are generated artifacts' sources: iterate with
  `sase skill init --diff`/`--dry-run` only; deployment happens after the change lands on the canonical branch (commit
  first, then deploy) — do not deploy from a dirty tree.
- No edits to `sase/memory/*.md`, `AGENTS.md`, or provider instruction shims are in scope.
- Show project names, never ProjectSpec keys, in any user-facing text added.

## Risks and notes

- **Breaking CLI change**: `-f/--file` removal affects any out-of-repo muscle memory; the exclusion model is the point
  of the change. Mark commits breaking; skills/docs updated in the same change keep agents on the new contract.
- **Pathspec staging**: `git add -A -- . ':(exclude)…'` must be exercised against modified, deleted, and untracked files
  (tests in commit-exclusions); behavior from the repo root vs subdirectories should be pinned by running the command
  from the repo root.
- **8 KiB message cap** (design decision 8): `--vars` fails fast with actionable guidance.
- **List values downstream**: clan/tribe aggregation hashing, sidecar consumers, and Telegram notification payloads must
  tolerate `str | list[str]`; publication validation changes are additive so already-published string-only inventories
  remain valid.
- **Transition compatibility**: Python readers accept both wire shapes, so a version skew between sase-core-rs and this
  repo during rollout degrades to today's behavior (lists dropped) rather than crashing.
- **Proposal exclusions** are an in-phase decision (apply or reject clearly); either outcome is documented and tested.
