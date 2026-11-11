---
tier: epic
title: Security and robustness audit fixes
goal: Verified injection, traversal, privilege-gap, and state-corruption bugs from
  the codebase audit are fixed with regression tests
decisions:
  agent_cmdsub:
    ask: Should $(cmd) substitution be provenance-gated so agent-authored prompt text
      never executes it?
    choices:
      provenance: Thread an authorship flag from launch paths; agent prompts skip
        $(cmd) but macros still expand
      wrap: Wrap agent-authored prompt text in disabled regions, which also disables
        macro expansion there
    default: provenance
    why: Handoff and launch prompts legitimately carry macros; wholesale wrapping
      would break that
    answer: provenance
  local_config_commands:
    ask: How should command-bearing keys in project-local sase/sase.yml be trust-gated?
    choices:
      digest_trust: Execute local-layer commands only after a per-project, digest-pinned
        user acknowledgment
      user_layers_only: Ignore command-bearing keys from the project-local layer;
        hooks move to user/overlay config
    default: digest_trust
    why: Preserves per-project hooks while blocking repo-supplied command execution
    answer: digest_trust
  bash_autoquote:
    ask: Should Jinja interpolations in workflow bash steps be shell-quoted by default?
    choices:
      autoquote: shlex.quote every interpolation with an explicit raw opt-out filter;
        update shipped macros
      tighten_only: Only tighten macro input-type charsets; bash templates keep raw
        interpolation
    default: autoquote
    why: Charset validation alone cannot protect line/text inputs interpolated into
      shell
    answer: autoquote
phases:
- id: cmdsub-provenance
  title: Stop $(cmd) execution in agent-authored prompt text
  depends_on: []
  size: medium
  description: "`preprocess_prompt_late` (src/sase/llm_provider/preprocessing.py:205)\
    \ runs\n`process_command_substitution` (sink: src/sase/file_references.py:393,\n\
    `_execute_cmd_cached`, `subprocess.run(cmd, shell=True)`) on every prompt after\n\
    protecting only disabled regions and fenced blocks. Monitor follow-up prompts\n\
    already defend against this: src/sase/monitor/followup_prompt.py:337 wraps\nuntrusted\
    \ command output in `wrap_disabled_region`. But two agent-authored prompt\npaths\
    \ get no such protection, so LLM-written text is executed host-side as shell,\n\
    outside every permission gate:\n\n1. Handoff successor prompts: persisted verbatim\
    \ by\n   src/sase/agent/_staged_handoff_successor.py (~lines 65-88), launched\
    \ without\n   approval via src/sase/axe/run_agent_successor_delivery.py (~lines\
    \ 383-439),\n   then macro-expanded and late-preprocessed. Any unfenced `$(...)`\
    \ the previous\n   agent wrote executes in the host process. Agent-authored `sase\
    \ run` launch\n   prompts flow the same way.\n2. Workflow-interpolated LLM output:\
    \ `preprocess_prompt_early` Jinja-renders\n   workflow context into prompt steps\
    \ (preprocessing.py ~lines 100-110, via\n   src/sase/macro/workflow_executor_steps_prompt_prepare.py),\
    \ and context includes\n   raw prior agent responses (`context[step.name] = {\"\
    _raw\": response_text}` in\n   src/sase/main/query_handler/_standalone_steps.py\
    \ ~line 192). `$(...)` inside a\n   model response then executes during the late\
    \ phase.\n\nFirst re-verify both traces (line numbers may have drifted). Then\
    \ implement per\nthe epic's `agent_cmdsub` decision. For `provenance` (default):\
    \ add an explicit\nprompt-authorship parameter threaded from the launch entry\
    \ points into\n`preprocess_prompt_late`, and skip `process_command_substitution`\
    \ (and any other\nhost-side execution steps, but NOT macro/artifact expansion)\
    \ when the prompt text\nis agent-authored. Agent-authored entry points to cover:\
    \ staged-handoff successor\ndelivery, agent-initiated `sase run`/LaunchApproval\
    \ launches, and any other path\nwhere the prompt author is an agent rather than\
    \ a human. Human-typed TUI/CLI\nprompts keep current behavior. Independently of\
    \ the decision, wrap workflow\ncontext values that carry model output (`_raw`,\
    \ `_response`) in\n`wrap_disabled_region` the way monitor output already is, so\
    \ Jinja interpolation\ncannot smuggle `$(...)` back into an executable region.\
    \ For `wrap`: instead wrap\nthe entire agent-authored prompt body in a disabled\
    \ region at persist/launch time.\n\nTests: regression tests proving (a) a handoff/launch\
    \ prompt containing `$(touch\n/tmp/pwn)` does not execute and the literal text\
    \ survives, (b) a workflow agent\nstep whose response contains `$(...)` does not\
    \ execute it when a later prompt step\ninterpolates `{{ step._raw }}`, (c) human-authored\
    \ prompts still perform command\nsubstitution, (d) macros still expand in handoff\
    \ prompts under the `provenance`\nchoice. Place tests near existing preprocessing\
    \ tests (tests/ mirrors src/\nlayout).\n"
- id: local-config-trust
  title: Trust-gate command-bearing project-local config
  depends_on: []
  size: large
  description: "SASE merges project-local config (`sase/sase.yml` found by walking\
    \ up from cwd to\nthe nearest VCS root: src/sase/config/core.py `load_merged_config`,\
    \ ~lines\n562-604, local layer via `get_local_config_path` /\n`discover_project_root`\
    \ in src/sase/content_layout.py) with the same authority as\nthe user's own ~/.config/sase/sase.yml.\
    \ Command-bearing keys from that local layer\nare executed with `shell=True` by\
    \ host-side (not permission-gated) processes:\n\n- commit hooks: src/sase/workflows/commit/command_hooks.py:176\
    \ (`Popen(cmd,\n  shell=True)`), fed from `load_merged_config()[\"commit_hooks\"\
    ][\"before\"/\"after\"]`\n- rebase gate: src/sase/workflows/commit/rebase_gate_process.py:58,\
    \ fed from\n  `commit_hooks.after_rebase` via `load_merged_config_for_local_path`\n\
    - file hooks: src/sase/file_hooks/runner.py:188, commands from\n  `file_hooks[].command`\
    \ across `load_config_layers()` including the local layer\n  (src/sase/config/file_hooks.py)\n\
    - workflows/macros are also loadable from the local layer (config/core.py\n  `load_macros_by_source`),\
    \ and macro workflow bash steps execute at expansion\n  time.\n\nConsequence:\
    \ a hostile repo adopted as a project, or an agent editing its own\nworkspace's\
    \ sase/sase.yml, injects commands that the host-owned commit finalizer\nruns with\
    \ no gate — defeating the \"Completion Is Host-Owned\" boundary (agents must\n\
    not be able to escalate a plain file write into host-side execution).\n\nImplement\
    \ per the epic's `local_config_commands` decision. For `digest_trust`\n(default):\
    \ execute command-bearing local-layer config (commit_hooks,\nfile_hooks[].command,\
    \ local-layer macro/workflow definitions that contain bash or\npython steps) only\
    \ when the user has acknowledged that project's current\ncommand-bearing content,\
    \ pinned to a content digest; any change requires\nre-acknowledgment (direnv-style\
    \ allow model). Untrusted state must not block\nnon-command config (display names,\
    \ aliases, etc.). Design the trust store\nlocation, the acknowledgment UX (a CLI\
    \ command and/or gate prompt — never\nagent-answerable), and the refusal diagnostics\
    \ as part of this phase's planning.\nFor `user_layers_only`: stop honoring command-bearing\
    \ keys from the local layer\nentirely and document the migration (hooks move to\
    \ user config/overlays).\n\nThis deprecates user-reaching behavior, so before\
    \ implementing read the memory\nnotes sase/memory/sase_flags.md (a `sunset` flag\
    \ is mandatory: old\nrun-local-commands-untrusted branch stays reachable while\
    \ migrating; create it\nwith `sase flag new`, never by hand) and the gotchas core\
    \ note (update\nsrc/sase/default_config.yml if config keys are added). Tests must\
    \ cover both flag\nstates, trusted/untrusted/digest-changed projects, and that\
    \ an untrusted local\nlayer still merges its non-command keys.\n"
- id: path-containment
  title: Path-component validation for repo clone dirs and project log paths
  depends_on: []
  size: medium
  description: "Two confirmed containment gaps where repo-resident (hence agent/third-party\n\
    writable) strings reach filesystem path joins without the component validation\n\
    their sibling helpers already have:\n\n1. src/sase/_linked_repo_paths.py: `linked_repo_clone_dir`\
    \ (~line 28) and\n   `sidecar_repo_clone_dir` (~line 58) do `Path(host_checkout).joinpath(*SUBDIR,\n\
    \   name)` then `normalize_path()` (which resolves), so a `name`/`dirname` like\n\
    \   `../../../../home/user/.config/x` escapes `<checkout>/sase/repos/`. The same\n\
    \   file's `external_repo_clone_dir` (~line 36) and `hidden_sidecar_clone_dir`\n\
    \   (~line 66) validate components (\"must be a safe path component\"), proving\
    \ the\n   invariant is intended. Origins: `sase.yml` `repos.sidecar.custom.<role>`\
    \ keys\n   and `repos.linked[].name` (src/sase/_linked_repo_config_entries.py\
    \ only strips\n   empties), and `.sase/sdd-store.json` role keys / `repo` basenames\n\
    \   (`_repo_basename(\"x/..\") == \"..\"`, _linked_repo_paths.py ~line 85;\n \
    \  src/sase/sdd/_store_records.py accepts any non-empty role). Reached with no\n\
    \   user interaction by sidecar auto-connect/auto-sync\n   (src/sase/_sidecar_auto_connect.py,\
    \ src/sase/_sidecar_auto_sync.py →\n   src/sase/workspace_provider/_ownership_paths.py\
    \ `kind_root_for_context`) and by\n   clone/materialize paths (src/sase/_linked_repo_workspaces.py\
    \ →\n   `ensure_git_clone_at` in src/sase/workspace_provider/_utils_checkout_clone.py,\n\
    \   which has no containment check). Fix: apply the `hidden_sidecar_clone_dir`\n\
    \   safe-component check (non-empty, not `.`/`..`, no `/`, `\\`, NUL, no leading\n\
    \   `.`, no surrounding whitespace) to `name`/`dirname` in `linked_repo_clone_dir`\n\
    \   and `sidecar_repo_clone_dir`; reject non-component role keys where config\
    \ and\n   sdd-store records are parsed (`_sidecar_bucket_entries` /\n   `_sidecars_from_raw`);\
    \ and add a belt-and-suspenders containment check in\n   `ensure_git_clone_at`\
    \ (refuse targets not under the expected `sase/repos/`\n   root when invoked for\
    \ linked/sidecar clones).\n\n2. Project-name joins under ~/.sase/projects/: `validate_sase_project_name`\n\
    \   (src/sase/core/paths.py ~line 175, docstring: rejecting separators/dots \"\
    is\n   what enforces containment under sase_projects_dir()\") is called at somesinks\n\
    \   (src/sase/_plan_approval_artifacts.py,\n   src/sase/agents_sync/prompt_archive/protection.py)\
    \ but NOT in five log-path\n   helpers that mkdir+append at `sase_projects_dir()\
    \ / project_name / ...`:\n   src/sase/artifact_read_log.py `artifact_read_log_path`\
    \ (~line 54),\n   src/sase/memory/_read_log_events.py (~line 160), src/sase/skills/use_log.py\n\
    \   (~line 130), src/sase/repo_open_log.py (~line 115),\n   src/sase/memory/legacy_glossary_read_log.py\
    \ (~line 56). The name can originate\n   from a committed `.sase/checkout.json`\
    \ `project_name`\n   (src/sase/workspace_provider/marker.py `from_dict` accepts\
    \ any string;\n   preferred by `project_memory_name` in src/sase/main/init_memory/config.py)\
    \ or\n   from a git-URL basename, so a hostile checkout makes audited reads mkdir\
    \ and\n   append outside ~/.sase/projects/. Fix: route all five through one shared\n\
    \   helper that calls `validate_sase_project_name` (decide and document the\n\
    \   failure behavior: raise, or fall back to a sanitized name — prefer raising\
    \ to\n   match the validated sinks), and additionally validate\n   `CheckoutMarker.from_dict`'s\
    \ `project_name`/`project_key` at parse time.\n\nTests: traversal attempts (`..`-bearing\
    \ and absolute names) for each fixed helper\nmust raise/refuse and must not create\
    \ directories; valid names keep working.\nCheck tests/ for existing coverage of\
    \ these modules and extend it.\n"
- id: shell-arg-hardening
  title: Shell-quote workflow interpolations, validate git refs, fix agent-guard parity
  depends_on: []
  size: medium
  description: "Three bounded hardening fixes, all independently verified:\n\n1. Workflow\
    \ bash/python step injection (per the epic's `bash_autoquote`\n   decision). Sinks:\
    \ src/sase/macro/workflow_executor_steps_script.py (~line 135)\n   and src/sase/main/query_handler/_standalone_steps.py\
    \ (~line 85) run\n   Jinja-rendered templates with `subprocess.run(..., shell=True)`;\
    \ the Jinja env\n   (src/sase/macro/workflow_executor_utils.py) has no shell-quoting\
    \ filter.\n   Interpolated values include macro inputs from prompt text and prior\
    \ step\n   outputs (including model responses). The `word` input type\n   (src/sase/macro/models.py\
    \ ~line 168) only rejects whitespace, so\n   `$(touch${IFS}/tmp/pwn)`, `;`, and\
    \ quotes all pass. Shipped injectable\n   examples: src/sase/macros/file.yml (`{{\
    \ name }}` inside a bash step path) and\n   src/sase/macros/json.yml (`'{{ schema\
    \ }}'` — a single quote breaks out). For\n   `autoquote` (default): auto-apply\
    \ shlex.quote to every Jinja interpolation\n   inside `bash:` steps with an explicit\
    \ opt-out filter (e.g. `| raw`) for\n   intentional raw interpolation; apply equivalent\
    \ escaping or argument-passing\n   for `python:` steps (prefer passing values\
    \ via argv or env rather than\n   splicing into source); update the shipped macros\
    \ under src/sase/macros/ and\n   src/sase/default_macros/ to stay correct under\
    \ quoting. For `tighten_only`:\n   instead restrict `word` to a charset allowlist\
    \ `[A-Za-z0-9._-]+` and audit\n   shipped templates. In BOTH cases tighten `word`\
    \ validation.\n\n2. Git argument injection in VCS plugins:\n   src/sase/vcs_provider/plugins/_git_core_ops.py\
    \ `vcs_checkout` (~line 29)\n   passes `revision` positionally to `git checkout`,\
    \ and\n   src/sase/vcs_provider/plugins/bare_git.py `vcs_mail` (~line 33) to `git\
    \ push\n   -u origin <revision>`. `revision` derives from Patch NAME / branch-map\
    \ values\n   in agent-writable `.sase` files (src/sase/vcs_provider/plugins/\n\
    \   _git_revision_ops.py), so values like `--receive-pack=...` (push; executes\
    \ a\n   local command for filesystem remotes) or `-B`/`--pathspec-from-file=`\n\
    \   (checkout) smuggle flags. Fix: validate candidate revisions where they are\n\
    \   read (reject names starting with `-` and names failing `git check-ref-format\n\
    \   --branch` or an equivalent charset check), and prefer flag-safe invocation\n\
    \   forms where git's syntax allows.\n\n3. Agent-guard detection parity: tools/_sase_install_env.py\
    \ `AGENT_ENV_VARS`\n   (~line 22) lists only `SASE_AGENT` and `SASE_MONITOR_ID`,\
    \ but the canonical\n   detector `agent_marker_present` (src/sase/agent/identity.py\
    \ ~lines 34-48) also\n   recognizes `SASE_AGENT_NAME` and discovered agent metadata.\
    \ A process carrying\n   only `SASE_AGENT_NAME` bypasses the \"Global Install\
    \ Is Human-Only\" refusal in\n   `agent_guard`. Fix: add `SASE_AGENT_NAME` to\
    \ `AGENT_ENV_VARS` (keep the module\n   stdlib-only; do not import sase) and add/extend\
    \ the parity test the module's\n   comments promise so the two sets cannot silently\
    \ drift again. Read the memory\n   note sase/memory/tools.md before changing anything\
    \ under tools/.\n\nTests: injection attempts through macro args and workflow context\
    \ must execute\nnothing (assert via marker files/mocked subprocess); hostile revision\
    \ names must\nbe refused before any git invocation; agent_guard must refuse under\
    \ each marker\nenv var.\n"
- id: lock-lifecycle
  title: Fix patch-lock fd double close and edit-lock acquisition races
  depends_on: []
  size: small
  description: "Two verified defects in src/sase/ace/patch/locking.py:\n\n1. `patch_lock`\
    \ (~lines 166-184): on lock timeout the `except BlockingIOError`\n   branch runs\
    \ `os.close(fd)` and raises `LockTimeoutError`, but the function's\n   `finally`\
    \ then runs `fcntl.flock(fd, LOCK_UN)` + `os.close(fd)` on the\n   already-closed\
    \ fd. Single-threaded this raises OSError(EBADF) from the finally,\n   replacing\
    \ LockTimeoutError so handlers keying on it (e.g.\n   src/sase/ace/patch/duplicate_repair.py,\
    \ src/sase/ace/hooks/persistence.py)\n   misclassify the failure; multi-threaded\
    \ (ACE runs these in worker threads) the\n   fd number can be reused by another\
    \ thread between the close and the finally,\n   and the finally then unlocks/closes\
    \ an unrelated descriptor. Fix: restructure\n   so acquisition happens before\
    \ the try whose finally owns cleanup (or drop the\n   close from the timeout branch\
    \ and make the finally the single owner); the\n   timeout path must raise LockTimeoutError\
    \ exactly once with no second close.\n\n2. `acquire_edit_lock` / `is_edit_locked`\
    \ (~lines 37-91): the lock file is\n   created with a plain overwrite (no O_CREAT|O_EXCL),\
    \ so two processes can both\n   believe they own the edit lock; liveness uses\
    \ `os.kill(pid, 0)`, so a recycled\n   PID makes a stale lock look held and `wait_for_edit_lock_release`\
    \ can poll for\n   hours. Fix: create with O_CREAT|O_EXCL (retry after stale-lock\
    \ cleanup),\n   record PID plus process start time (or an equivalent boot-relative\
    \ token), and\n   treat a PID whose start time does not match as stale.\n\nTests:\
    \ timeout path raises LockTimeoutError (not OSError) and leaves no dangling\n\
    fd (assert via os.fstat raising on the fd or fd-count accounting); concurrent\n\
    edit-lock acquisition admits exactly one owner; a stale lock with a recycled-PID\n\
    token is reclaimed.\n"
- id: atomic-state-writes
  title: Atomic writes and locking for MRU, history, and axe error state
  depends_on: []
  size: medium
  description: "Durable state files written with truncate-then-write and unlocked\n\
    read-modify-write cycles, all verified:\n\n1. src/sase/history/vcs_macro_mru.py\
    \ `_save_vcs_macro_mru` (~line 263): plain\n   `open(mru_file, \"w\")` + json.dump.\
    \ This file is the source of truth for the\n   current project (see src/sase/current_project.py\
    \ module docstring) and is\n   written on every launch (`record_vcs_macro_usage`)\
    \ and by the prune path in\n   `load_launchable_vcs_macro_mru_pairs`, from multiple\
    \ processes (TUI, headless\n   launches, `set_current_project`). A reader racing\
    \ the truncate gets invalid\n   JSON → treated as empty → the next writer permanently\
    \ wipes the MRU (and with\n   it the current-project selection). Fix: write via\
    \ unique mkstemp +\n   `os.replace` (follow the pattern in src/sase/history/prompt_store.py),\
    \ and\n   serialize the read-modify-write in `record_vcs_macro_usage` and the\
    \ prune path\n   with an flock sidecar lock.\n2. Same truncate-write pattern in\
    \ src/sase/history/hook.py `_save_hooks` (~line\n   70) and src/sase/history/command.py\
    \ (~line 78): convert both to temp+replace\n   (locking optional but preferred;\
    \ at minimum eliminate the\n   corruption-on-crash/race window).\n3. src/sase/axe/state.py\
    \ `atomic_write_json` (~lines 45-66) uses one\n   deterministic temp name (`path.with_suffix(\"\
    .tmp\")`), so concurrent writers\n   (orchestrator + each lumberjack process call\
    \ `append_error` in\n   src/sase/axe/_state_scheduler.py ~line 175 on the shared\
    \ recent_errors.json)\n   interleave writes in the same temp file and can install\
    \ garbage JSON; the\n   loser's rename failure is swallowed. Fix: use `tempfile.mkstemp(dir=...,\n\
    \   prefix=...)` like src/sase/notification_gates/durability.py already does,\
    \ and\n   protect `append_error`'s read-modify-write with an flock so concurrent\
    \ errors\n   are not lost.\n\nKeep behavior on read unchanged (corrupt/missing\
    \ files still degrade to\nNone/empty). Tests: concurrent-writer test (threads\
    \ or processes) showing no\ncorrupt JSON and no lost entries for the MRU and recent_errors\
    \ paths; a\ncrash-window test (kill between open and write is hard — instead assert\
    \ the\nimplementation never truncates the target in place, e.g. by checking the\
    \ target's\ncontent is always one of the two valid states while a writer loops).\n"
- id: sqlite-robustness
  title: Chat-catalog cache lock handling and bead migration idempotency
  depends_on: []
  size: small
  description: "Two verified sqlite defects:\n\n1. src/sase/history/chat_catalog_provenance/cache.py\
    \ `open_catalog_cache`\n   (~lines 30-48): the rebuild path catches `sqlite3.DatabaseError`\
    \ —\n   but `sqlite3.OperationalError` (\"database is locked\") is a subclass,\
    \ and\n   `_connect` uses a short busy timeout, so plain cross-process contention\
    \ is\n   misdiagnosed as corruption and `_discard_broken_cache` unlinks a database\n\
    \   another process has open mid-transaction (its commit lands on the orphaned\n\
    \   inode and is lost; both processes then pay a full head rescan). Fix: treat\n\
    \   lock contention as transient — re-raise or retry `sqlite3.OperationalError`\n\
    \   lock errors (match the \"database is locked\" family, or check\n   `sqlite3.DatabaseError\
    \ and not OperationalError` plus the\n   malformed/not-a-database messages) and\
    \ only discard on genuine corruption;\n   consider raising the busy timeout. Add\
    \ a regression test: hold a write\n   transaction from one connection, open the\
    \ cache from another, and assert the\n   db file is NOT unlinked and the second\
    \ open either waits or fails without\n   destroying state.\n\n2. src/sase/bead/_db_migrations.py\
    \ `_migrate_issue_types` (~lines 224-270): in\n   sqlite3 legacy autocommit mode\
    \ the `CREATE TABLE _issues_new` DDL commits\n   immediately while the following\
    \ INSERT...SELECT opens the implicit\n   transaction. If the INSERT fails (legacy\
    \ row violating the new CHECKs) or the\n   process dies before `conn.commit()`,\
    \ `_issues_new` persists while the guard\n   (`\"'plan'\" in create_table_sql`\
    \ of `issues`) still reports \"not migrated\" —\n   so every subsequent `init_db`\
    \ raises \"table _issues_new already exists\" and\n   the bead mirror is wedged\
    \ until manual repair. Fix: `DROP TABLE IF EXISTS\n   _issues_new` before the\
    \ CREATE so the rebuild is restartable; audit the other\n   table-rebuild migrations\
    \ in the same file for the same pattern and apply the\n   same guard where missing.\
    \ Also make `init_db` (src/sase/bead/db.py ~lines\n   28-40) close its connection\
    \ when `run_migrations` raises (try/finally or\n   closing contextmanager). Tests:\
    \ simulate the half-migrated state (create\n   `_issues_new`, keep old-schema\
    \ `issues`) and assert init_db now completes the\n   migration; assert no connection\
    \ leak on a failing migration (e.g. monkeypatch\n   a migration to raise and assert\
    \ the connection is closed)."
proposed_by: bbugyi200.athena.1bk
proposed_with: claude/claude-fable-5
decided_by: reviewer
decided_via: tui
create_time: 2026-11-11 17:38:50
status: wip
bead_id: sase-1sb
---

- **PROMPT:** [prompts/202611/security_bug_audit_fixes.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/security_bug_audit_fixes.md)
- **BEAD:** [sase-1sb](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1sb/README.md)

# Plan

## Context

A security and correctness audit of this codebase (requested: "audit this codebase for
any bugs or security issues and fix any that you find") fanned out over the four
highest-risk areas — shell/command execution, path and file handling, privileged
execution gates, and persistent-state/concurrency code — and each finding below was then
re-verified by reading the implicated code. Grep-level hygiene is good (YAML is
safe-loaded everywhere via `sase/_yaml_safe.py`, bead/catalog SQL is parameterized, no
`tempfile.mktemp`, the one wheel-extraction site reads a single named member), so the
surviving findings are data-flow issues and concurrency defects.

Threat model used: the user's own config and typed input are trusted; LLM-agent-written
text (prompts, plan files, `.sase` fields, workspace file edits), and the contents of
external/GitHub repos (committed `sase/sase.yml`, `.sase/checkout.json`,
`sdd-store.json`, branch names) are untrusted or semi-trusted. SASE's own decision
records ("Completion Is Host-Owned", "Global Install Is Human-Only") define the
privilege boundaries the security phases restore.

Audit areas that held up (no phase needed): the sudo gate request→approval→execution
binding (manifest digest re-verified with constant-time compare before argv-exec, env
cleared, no shell), notification-gate response handling (atomic exclusive writes,
authorship fails closed to `agent`), `sase tool run -H` fail-closed reservation,
gate-bundle path safety (O_NOFOLLOW, containment, symlink rejection), memory
flat-note/strand read containment, external-repo clone path validation, footprint and
prompt-store atomic writes, git_lock_retry stale-lock gating, and tz-aware time handling
in the schedulers.

## Phases

All seven phases are independent (`depends_on: []` everywhere) and can run in parallel.
Every phase adds regression tests beside the existing tests for the files it touches and
runs `just check` before finishing.

### Security phases

**cmdsub-provenance** closes the highest-impact gap: host-side `$(cmd)` execution of
agent-authored prompt text. The codebase already treats this as a real risk for monitor
output (`wrap_disabled_region` in `monitor/followup_prompt.py`); the phase extends the
same protection to handoff/launch prompts and workflow-interpolated model output.

> [!decision] agent_cmdsub = provenance Thread an authorship flag from the launch entry
> points into late preprocessing and skip command substitution for agent-authored
> prompts; macros and artifact refs still expand, so handoff prompts keep working.

> [!decision] agent_cmdsub = wrap Wrap the whole agent-authored prompt in a disabled
> region at persist/launch time, matching the monitor precedent exactly; simpler, but
> macros inside handoff/launch prompts stop expanding.

**local-config-trust** restores the host-owned-completion boundary: today a plain file
write to a repo's `sase/sase.yml` (by an agent, or by whoever authored a repo the user
adopts) becomes host-side shell execution via commit hooks, the rebase gate, and file
hooks.

> [!decision] local_config_commands = digest_trust Keep per-project hooks working but
> execute local-layer commands only after a one-time, digest-pinned user acknowledgment;
> content changes require re-acknowledgment (direnv-style).

> [!decision] local_config_commands = user_layers_only Simpler and strictly safer:
> command-bearing keys are honored only from user/overlay layers; per-project hooks
> migrate into user config.

This phase is `large` (it plans before implementing): the trust-store shape,
acknowledgment UX, and the mandatory `sunset` feature flag (see
`sase/memory/sase_flags.md`) need their own design pass.

**path-containment** adds the missing safe-path-component checks in
`_linked_repo_paths.py` (its sibling helpers already validate, so this is an omission,
not a policy) and routes the five project-named audit-log writers through
`validate_sase_project_name`.

**shell-arg-hardening** covers workflow bash-step quoting, git ref validation in the VCS
plugins, and the `tools/_sase_install_env.py` agent-detection parity gap.

> [!decision] bash_autoquote = autoquote Interpolations in `bash:` steps are
> shlex-quoted by default with an explicit raw opt-out; shipped macros under
> `src/sase/macros/` are updated to stay correct.

> [!decision] bash_autoquote = tighten_only Lower-churn alternative: only the macro
> input types are tightened (word → charset allowlist); templates keep raw interpolation
> and remain injectable through `line`/`text` inputs.

### Robustness phases

**lock-lifecycle** fixes the `patch_lock` timeout double-close (which can close another
thread's fd and masks `LockTimeoutError` with `EBADF`) and the non-atomic,
PID-reuse-prone edit lock.

**atomic-state-writes** converts the truncate-written MRU/current-project store, hook
and command history, and the axe `recent_errors.json` writer (fixed shared temp name) to
unique-temp + rename with locked read-modify-write where multiple processes write.

**sqlite-robustness** stops the chat-catalog cache from unlinking a live database on
simple lock contention, and makes the bead `_migrate_issue_types` rebuild restartable
instead of permanently wedging `init_db` after a mid-migration crash.

## Out of scope / follow-ups

Two audit observations live outside this repo and are not planned here: the unused
`_run_shell` primitive in `src/sase/vcs_provider/_command_runner.py` is exposed to VCS
plugins (its only plausible callers are in the linked `sase-github` repo), and
`enforce_sidecar_remote_policy` in `src/sase/_git_remote.py` passes non-HTTP remote
strings (e.g. `ext::`) through unchanged for plugin-side consumers. These are being
filed as a separate task bead to audit the `sase-github` plugin's use of both surfaces.
