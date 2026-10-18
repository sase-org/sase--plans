---
tier: tale
title: Fix stale Codex CLI shadowing so sase-1m3.7.1.5--5 succeeds on retry
goal:
  Agent runners launch the current Codex CLI, the scheduled sase-1m3.7.1.5--5 retry
  succeeds, and version skew is detectable.
size: medium
decisions:
  stale_installs:
    ask:
      How should the stale root-owned codex and claude installs under /usr/local be
      fixed?
    choices:
      remove:
        Delete the stale /usr/local copies; nvm installs become the only copies and
        cannot drift
      upgrade:
        Upgrade the /usr/local copies in place; two installs per CLI can drift again
        later
    default: remove
    why:
      One install per CLI removes the PATH-order hazard instead of resetting the drift
      clock
    answer: remove
  shadow_detection:
    ask:
      Should SASE detect shadowed agent CLI installs and name the binary when Codex
      rejects its version?
    choices:
      add:
        Report skew in agent-cli list and doctor, plus an actionable Codex
        version-rejection error
      skip:
        Fix only the machine and file a task bead; the failure can silently recur after
        updates
    default: add
    why:
      Shell and agent runners resolve different binaries, so tooling said current while
      agents ran stale
    answer: add
proposed_by: bbugyi200.apollo.92
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-18 16:24:28
status: wip
---

# Fix stale Codex CLI shadowing so `sase-1m3.7.1.5--5` succeeds on retry

## Root cause (verified)

Agent `sase-1m3.7.1.5--5` (codex / `gpt-6-astra`, artifacts dir
`~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202610/18/20261018160134`) failed
all three attempts with the same terminal error:

```
[turn.failed] {"type":"error","status":400,"error":{"type":"invalid_request_error",
  "message":"The 'gpt-6-astra' model requires a newer version of Codex. ..."}}
```

(The preceding `unknown variant 'max'` models-cache errors are a symptom of the same
stale client.)

- The machine has two Codex CLI installs:
  - `~/.config/nvm/versions/node/v24.15.0/bin/codex` → **0.163.0** (latest on npm)
  - `/usr/local/bin/codex` → `/usr/local/lib/node_modules/@openai/codex` → **0.128.0**
    (root-owned, installed 2026-05-03)
- `resolve_codex_executable()` in `src/sase/llm_provider/_codex_instructions.py` returns
  `SASE_CODEX_PATH`, else `shutil.which("codex")`, else `$NVM_BIN/codex`.
- The agent runner process (`run_agent_runner.py`, pid 2949828 at planning time; find it
  again with `sase agent list -j` → row `sase-1m3.7.1.5--5`) was spawned from inside a
  Codex session. Its inherited `PATH` puts `/usr/local/bin` **before** the nvm bin
  directory (the shell profile re-prepends user dirs after Codex injects its own
  entries). `shutil.which("codex")` under that exact PATH returns
  `/usr/local/bin/codex`, the stale 0.128.0 binary.
- `retry_state.json` shows `status: retrying`, `retry_count: 3/3`, `wait_seconds: 1800`,
  `next_retry_at_epoch` = **2026-10-18 20:41:42 UTC**. This is the final in-process
  retry. It runs in the same runner process with the same `os.environ`, so it resolves
  the same stale binary unless the file at `/usr/local/bin/codex` changes first. Codex
  retry config has no `fallback_model`, so if this attempt fails the agent ends FAILED.
- The same skew exists for Claude Code: `/usr/local/bin/claude` is 2.1.126 while the nvm
  copy is 2.1.302. Claude agents spawned from Codex sessions would launch the stale
  copy.

Changing SASE code cannot affect the already-running runner, because its executable
resolution and environment are fixed in-process. Only the machine-level fix (step 1) can
rescue the scheduled retry. Steps 3–4 prevent recurrence.

## Step 1 — URGENT: fix the stale root-owned installs (before 20:41:42 UTC)

Do this first, before reading any more code. It needs root, so request it through your
`/sase_sudo` skill (never raw `sudo`), with a reason that names the retry deadline.

> [!decision] stale_installs = remove

```bash
rm /usr/local/bin/codex /usr/local/bin/claude
rm -rf /usr/local/lib/node_modules/@openai/codex /usr/local/lib/node_modules/@anthropic-ai/claude-code
```

Leave `/usr/local/bin/mcp-hub` and every other `/usr/local/lib/node_modules` entry
alone.

> [!decision] stale_installs = upgrade

```bash
env "PATH=$HOME/.config/nvm/versions/node/v24.15.0/bin:/usr/bin:/bin" \
  npm install -g --prefix /usr/local @openai/codex@0.163.0 @anthropic-ai/claude-code@2.1.302
```

(Use the versions `codex --version` / `claude --version` report from the nvm bin dir at
implementation time if they have moved. There is no `/usr/local/bin/node`. The
`#!/usr/bin/env node` shebang resolves the nvm node from runner PATHs, so nvm's npm must
do the install.)

**Verify against the runner's real PATH**, not your own shell's:

```bash
RUNNER_PID=<pid of sase-1m3.7.1.5--5 from `sase agent list -j`>
RPATH="$(tr '\0' '\n' < /proc/$RUNNER_PID/environ | sed -n 's/^PATH=//p')"
PATH="$RPATH" sh -c 'command -v codex && codex --version'   # must print 0.163.0 (or newer)
PATH="$RPATH" sh -c 'command -v claude && claude --version' # nvm copy, current version
```

If the gate is not approved before 20:41:42 UTC, still apply the fix and then do step
2's relaunch branch.

## Step 2 — Confirm the retry (or relaunch it)

After 20:41:42 UTC, check `<artifacts_dir>/retry_state.json` and
`<artifacts_dir>/attempts/04/` (and `sase agent show sase-1m3.7.1.5--5`):

- **Retry started after the fix**: confirm the new attempt is not failing with "requires
  a newer version of Codex" (watch `retry_state.json` move to `running_retry` and the
  agent stay RUNNING past the first minute, when the old attempts died). Done.
- **Retry fired before the fix, or failed anyway**: once step 1 is verified, relaunch it
  through your `/sase_run` skill using the retry procedure from the `sase_agents_status`
  skill. Use `<artifacts_dir>/raw_prompt.md` as the prompt source, get a fresh name from
  `sase.agent.names.allocate_retry_name("sase-1m3.7.1.5--5")`, and rewrite the prompt
  with `sase.agent.retry_prompt.rewrite_retry_prompt_name(...)`. Do not force same-name
  reuse.

Do not touch bead `sase-1m3.7.1.5` or its notes. That phase belongs to the retried
agent.

## Step 3 — Detect shadowed agent CLI installs (durable)

> [!decision] shadow_detection = add

Today `sase agent-cli list` resolves the CLI from the caller's PATH only. From the
user's shell it sees nvm codex 0.163.0 and reports it current, while agent runners
launched from Codex sessions run the shadowed 0.128.0 copy. Make the skew visible:

1. `src/sase/agent_clis/detect.py`: after resolving `executable` in
   `detect_agent_cli_statuses`, enumerate every other executable for the same `binary`.
   Walk each `PATH` entry (`os.access(..., X_OK)`) and add `$NVM_BIN/<binary>`. That
   fallback is codex-specific today, so add it generically only when it exists.
   Deduplicate by `os.path.realpath` and drop the resolved executable itself. Probe each
   remaining candidate with the existing `probe_version` (same `version_argv` /
   `version_regex`), reusing the injected `run_fn` so tests stay offline.
2. `src/sase/agent_clis/models.py`: add a field on `AgentCliStatus`, for example
   `other_installs: tuple[ShadowInstall, ...] = ()`, with a small frozen dataclass of
   `executable`, `version`, `version_error`. Default it to empty so existing
   constructors and fixtures keep working.
3. `src/sase/agent_clis/cli_list.py` (and its JSON output, if it has one): when
   `other_installs` is non-empty, show each extra path and version. Flag it as version
   skew when any version differs from `installed_version`. Add a remedy hint: remove the
   stale copy, or pin `SASE_<PROVIDER>_PATH`, using
   `llm_registry.provider_path_env_var`.
4. `src/sase/doctor/checks_providers.py`: add (or extend an existing provider check
   with) a WARN result listing providers whose installs have version skew. Reuse
   `detect_agent_cli_statuses`, and keep doctor read-only and offline. Register any new
   check through `provider_check_specs`.

Keep `resolve_codex_executable()` precedence unchanged. Silently picking a different
binary than PATH would surprise users who intentionally shadow, and detection plus
`SASE_CODEX_PATH` is the explicit escape hatch.

## Step 4 — Actionable Codex version-rejection error (durable)

> [!decision] shadow_detection = add

In `src/sase/llm_provider/codex.py`, where a failed Codex run becomes the provider
error, detect the substring `requires a newer version of Codex` in the collected
error/stderr text. Append one line naming the executable SASE launched, using the
`resolve_codex_executable()` value already used to build the argv. Also suggest
`sase agent-cli list` and `SASE_CODEX_PATH`. Keep the original error text intact, and do
not add this phrase to retry `error_patterns`: retrying a stale binary cannot succeed
without a human fix.

> [!decision] shadow_detection = skip

Skip steps 3 and 4 entirely. Instead, file one task bead (via `/sase_new_task`)
describing the shadowed-install hazard and this root cause so it is not lost.

## Tests

- `tests/` coverage for `detect_agent_cli_statuses` with a fake PATH containing two
  executables for one binary. Cover: versions differ (skew reported), versions match
  (listed, not flagged), the same realpath through two symlinks (deduped), and the
  `NVM_BIN` candidate. Use the existing fake `run_fn` / tmp-path patterns in the
  agent-cli tests.
- `cli_list` rendering test for the skew line, plus a doctor check test (WARN with skew,
  pass otherwise).
- A codex provider test asserting that the version-rejection error gains the
  launched-executable hint and keeps the original message.

## Verification

Follow the `lint_and_test` reference memory (read it with your `/sase_memory_read`
skill) for the required repo checks. Use `just check` through `sase tool run` (not
`just check-full`). The machine fix in step 1 does not touch git-tracked files. Report
the step 1 verification output and the step 2 outcome in your final response.
