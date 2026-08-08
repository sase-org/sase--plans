---
tier: epic
title: Meta Muse Code as a first-class SASE LLM provider
goal: 'SASE can run agents on Meta''s Muse Code CLI as a native provider — selected
  by config, `%model:muse/...`, or `SASE_MUSE_PATH` — with reply, usage, and tool-call
  artifacts; correctly-rendered Muse-native skills; a data-sharing advisory that makes
  the Contributor model''s training terms impossible to miss; and `sase agent-cli`
  install/update support that works for a channel-versioned, self-updating CLI.

  '
phases:
- id: cli_meta
  title: Channel-versioned agent-CLI detection and update
  depends_on: []
  size: medium
  description: 'cli_meta: teach agent-CLI detection, latest-version resolution, and
    update planning about CLIs distributed by a version channel instead of npm — a
    JSON-endpoint latest oracle, an exact version comparator, env-carrying self-update
    commands, and the `script` install manager.'
- id: provider
  title: The Muse provider and its JSONL stream parser
  depends_on: []
  size: medium
  description: 'provider: add `MuseProvider` and `_subprocess_muse`, register the
    `muse` entry point, map tiers and the full canonical reasoning-effort vocabulary,
    deploy skills to Muse''s native root, add the doctor setup fallback, and test
    both against recorded release-keyed fixtures.'
- id: cli_install
  title: sase agent-cli install
  depends_on:
  - cli_meta
  size: medium
  description: 'cli_install: add a confirmed, shell-free `sase agent-cli install`
    subcommand that fetches a provider-declared installer over HTTPS, shows its digest
    before running it, and never edits the user''s shell rc files.'
- id: artifacts
  title: Usage, tool-call, and model-identity artifacts
  depends_on:
  - provider
  size: medium
  description: 'artifacts: extract tool calls from the Muse event stream, recover
    token usage from the session log SASE owns via `--session-id`, and record the
    model Muse actually configured.'
- id: advisory
  title: Model advisories and the Contributor data-sharing guard
  depends_on:
  - provider
  size: medium
  description: 'advisory: add a provider-neutral model-advisory hook, surface it in
    the model picker, `%model` completion, and model labels, and add a doctor check
    that reports when a resolved default routes SASE traffic to an advisory-flagged
    model.'
- id: polish
  title: ACE styling and provider badges
  depends_on:
  - provider
  size: small
  description: 'polish: give Muse a Meta-blue provider palette, an emoji badge, and
    a family color so agent rows, model labels, and integrations render it as a known
    provider instead of the neutral fallback.'
- id: docs
  title: Documentation sweep
  depends_on:
  - cli_install
  - artifacts
  - advisory
  - polish
  size: medium
  description: 'docs: add the Muse provider section and update every provider enumeration
    across the docs set and `default_config.yml` comments, including the new install/update
    and advisory behavior.'
- id: verify
  title: Live end-to-end verification
  depends_on:
  - docs
  size: small
  description: 'verify: run real SASE agents on Muse, confirm the artifacts and skill
    rendering on disk, exercise `sase agent-cli` install/update against the live channel,
    and land the tree green under `just check-full`.'
proposed_by: bbugyi200.athena.ve
create_time: 2026-08-07 20:45:30
status: done
bead_id: sase-ha
---

- **PROMPT:** [prompts/202608/muse_provider.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/muse_provider.md)
- **BEAD:** [sase-ha](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ha/README.md)

# Plan: Meta Muse Code as a first-class SASE LLM provider

## Context and evidence

The consolidated research report is the starting point. Read it before implementing any
phase — open the research sidecar with `/sase_repo` and read
`202608/muse_code_harness_provider/muse_code_harness_provider.md`.

This plan supersedes that report wherever they disagree. The report was written without
an authenticated Meta account and deliberately deferred six questions to "§8 What still
needs an authenticated Meta account". Those questions have now been answered by running
the authenticated CLI directly, and the answers changed several load-bearing design
decisions. Three sanitized captures from release `0.1.0-R708.1` are stored as durable
artifacts and are the seed fixtures for this work:

- `/home/bryan/.sase/artifacts/agents/gh_sase-org__sase/20260807203045/muse_exec_read_tool_R708.1-8cfd0e83ddae.jsonl`
  — 60-event `muse exec --json` stdout stream for a single `read_file` tool call.
- `/home/bryan/.sase/artifacts/agents/gh_sase-org__sase/20260807203045/muse_exec_write_bash_tools_R708.1-b149daaeea8c.jsonl`
  — 86-event stdout stream covering `write_file` (with `edit_facts`) and `bash`.
- `/home/bryan/.sase/artifacts/agents/gh_sase-org__sase/20260807203045/muse_session_log_usage_R708.1-75577524ee74.jsonl`
  — the matching on-disk session log, which is where token usage actually lives.

### What the authenticated runs changed

| Question           | Report's position                          | Verified answer                                                                                                                                                                                                                         |
| ------------------ | ------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Token usage        | "Return `usage=None` until captured"       | **Not in the stdout stream at all.** It is in the session log as `runtime.session` / `model_completed` / `usage`. SASE can have real usage, but only by reading the session it named.                                                   |
| Tool-call shape    | Unknown; assumed Codex-like                | `tool.result` carries `call_id`, `correlation_facts.{tool_name,outcome}`, and `edit_facts.{path,added}` for edits. **Tool arguments are never in the stream** — file paths come from `edit_facts`, shell commands from the result JSON. |
| Model catalog      | "Do not ship `muse-spark-1.2-contributor`" | It exists and is real. It is the same model at ~95% lower cost, and **Meta trains on its inputs and outputs.** That makes it worth shipping _and_ worth guarding.                                                                       |
| Update mechanics   | `self_update_argv`                         | No update subcommand exists. The update is env-driven (`MUSE_SYNC_UPDATE=1`), which SASE's update planner cannot express today.                                                                                                         |
| Latest version     | `latest_version_package`                   | npm-only. Muse's version lives at a JSON channel endpoint.                                                                                                                                                                              |
| Version comparison | Not raised                                 | **`is_newer` silently returns `False` for every Muse version.** It uses PEP 440, and `0.1.0-R708.1` is not a valid PEP 440 version. Without a fix, Muse would report "no known updates" forever — a silent failure, not a visible one.  |

That last row is the reason `cli_meta` exists as its own phase and is not folded into
`provider`. Verified directly:

```text
'0.1.0-R709'   newer than '0.1.0-R708.1' = False
'0.1.1-R1'     newer than '0.1.0-R708.1' = False
```

### Verified facts this plan depends on

Run against `Muse Code 0.1.0 (0.1.0-R708.1)` on this machine:

- `muse --version` prints `Muse Code 0.1.0 (0.1.0-R708.1)`. The default semver regex
  extracts only `0.1.0` and discards the release id.
- `GET https://api.meta.ai/muse-code/channels/muse-stable` returns
  `{"channel":"muse-stable","version":"0.1.0-R708.1","urgency":"none","min_version":null,...}`.
- `MUSE_SYNC_UPDATE=1 muse --version` performs a synchronous launcher + binary update
  and then execs the binary, printing the resulting version. It exits `0` and takes well
  under a second when already current. `MUSE_NO_AUTO_UPDATE=1` suppresses the update
  entirely; the two must never be set together.
- The installer at `https://dev.meta.ai/install.sh` writes the launcher to
  `${MUSE_INSTALL_DIR:-~/.local/bin}/muse`, verifies an advertised `x-content-sha256`,
  runs `MUSE_LAUNCHER_INSTALL=1` on it, and then — only when `MUSE_UPGRADE_MODE` is not
  `1` — appends `export PATH=...` lines to the user's shell rc files.
  **`MUSE_UPGRADE_MODE=1` suppresses those rc edits.**
- `--session-id <UUID>` determines the session directory name:
  `~/.local/share/muse/sessions/YYYY/MM/DD/<UUID>/session.jsonl`.
- `muse skills list --json` on this machine currently reports **27 skills loaded from
  `$HOME/.claude/skills/`** — SASE's own generated skills, rendered with
  `provider_name: "Claude"` Jinja context. The skill-bleed problem is not theoretical;
  it is live right now.
- `sase-core` has no provider enum (`grep -rn "enum Provider"` over `crates/` is empty).
  Provider identity is untyped strings. **No Rust core change is required by any phase
  of this plan**, which satisfies the core-backend boundary rule rather than
  side-stepping it.

## Design

### Identity

| Field            | Value                                                                                           |
| ---------------- | ----------------------------------------------------------------------------------------------- |
| Provider name    | `muse`                                                                                          |
| Short name       | `mus` (unique against `cld`, `cdx`, `qwn`, `opc`, `agy`, `fky`; enables `foo.mus` agent naming) |
| Display name     | `Muse Code`                                                                                     |
| CLI name         | `muse`                                                                                          |
| Autodetect       | `llm_autodetect_cli_name` only — **no `llm_autodetect_priority`**                               |
| Skills root      | `.config/muse` → `~/.config/muse/skills/<name>/SKILL.md`                                        |
| Instruction file | `AGENTS.md`, natively. **No `MUSE.md` shim.**                                                   |

`muse` is a generic executable name and SASE's autodetect only checks PATH presence.
Omitting the priority keeps the provider out of `autodetect_candidates` while
`provider_cli_available()` still uses `cli_name` for doctor and the `sase agent-cli`
inventory. Muse is opt-in: `llm_provider.provider: muse`, `%model:muse/<model>`, or
`SASE_MUSE_PATH`.

### Model catalog

Taken from the Meta Model API console, which is authoritative and supersedes both
research reports:

| Model                        | Context | In / Cached / Out (per 1M) | Notes                                                                                                                                              |
| ---------------------------- | ------- | -------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `muse-spark-1.2`             | 1M      | $1.25 / $0.15 / $4.25      | Coding-optimized, purpose-built for agentic workflows.                                                                                             |
| `muse-spark-1.2-contributor` | 1M      | $0.10 / $0.002 / $0.20     | Same model, same capabilities. **Inputs and outputs are used to train and improve Meta's AI models.** Rate limited. Available in select countries. |
| `muse-spark-1.1`             | 1M      | $1.25 / $0.15 / $4.25      | Agentic + multimodal (text, images, video, documents).                                                                                             |

Tier mapping — **both tiers map to `muse-spark-1.2`**:

```python
_TIER_TO_MODEL: dict[ModelTier, str] = {
    "large": "muse-spark-1.2",
    "small": "muse-spark-1.2",
}
```

Mapping `small` to the Contributor model would be the obvious cost play and is the wrong
call. `small` is what `@cheap` and `@cheaper` reach for automatically, so xsmall and
small phase workers would silently ship the user's proprietary source code into Meta's
training corpus. SASE must not make that decision on a user's behalf. The Contributor
model stays fully available — it is in `llm_known_model_names`, it has the short alias
`spark12c`, and `%model:muse/muse-spark-1.2-contributor` works — but reaching it
requires typing its name, and the `advisory` phase makes sure that when someone does,
they see what they are agreeing to.

Short aliases: `spark12` → `muse-spark-1.2`, `spark12c` → `muse-spark-1.2-contributor`,
`spark11` → `muse-spark-1.1`.

### Reasoning effort

Muse accepts `none|minimal|low|medium|high|xhigh|ultra` and rejects `max` by name.
SASE's canonical vocabulary is `none|minimal|low|medium|high|xhigh|max`:

```python
_EFFORT_CLI_ARGS: dict[str, list[str]] = {
    level: ["--reasoning-effort", level]
    for level in ("none", "minimal", "low", "medium", "high", "xhigh")
} | {"max": ["--reasoning-effort", "ultra"]}
```

Muse becomes the first provider to cover all seven canonical levels. Note that Muse
defaults to `high` internally, so a run with no resolved effort shows blank in SASE
while Muse actually used `high`; the `artifacts` phase closes that gap by recording what
Muse reported.

### Invocation

```bash
MUSE_NO_AUTO_UPDATE=1 muse exec \
  --json \
  --workspace "$CWD" \
  --model <resolved-model> \
  [--reasoning-effort <level>] \
  --trust-workspace --disable-approval --disable-sandbox \
  --user-input-auto-resolve --no-foreign-personal-context \
  --session-id "$SESSION_UUID" \
  --prompt-file "$SECURE_TEMP_PROMPT" \
  [$SASE_LLM_{LARGE,SMALL}_ARGS or $SASE_MUSE_{LARGE,SMALL}_ARGS]
```

Decisions inside that command:

- **`--prompt-file`, not stdin and not a positional argument.** `exec` reserves stdin
  for `--api-key-stdin`, and SASE prompts routinely exceed comfortable argv limits.
  Write the prompt to a `0o600` file under `get_sase_managed_tmpdir()` and remove it in
  a `finally`.
- **`MUSE_NO_AUTO_UPDATE=1`.** The launcher otherwise checks for and swaps in a new
  binary hourly. A multi-hour agent run must not have its binary replaced mid-flight.
  Users update Muse through `sase agent-cli update muse` instead.
- **`--session-id` is generated by SASE**, not left to Muse, because it is the handle
  the `artifacts` phase uses to find the session log.
- **Sandbox off by default.** Under Muse's sandbox, `.git`, `.muse`, and `.agents` are
  read-only inside the workspace root, which breaks any in-run `sase commit` the agent
  performs through the `sase_git_commit` skill. (SASE's own post-invoke commit finalizer
  runs in SASE's Python process and is unaffected either way.) Disabling it matches what
  SASE already does for Codex and OpenCode and satisfies the uniform-runtime rule.
  Approvals must go regardless — a headless run cannot answer them.
- **A hardened opt-in**, gated on `SASE_MUSE_SANDBOX=on`: keep the sandbox and pass
  `--sandbox-network enabled` instead of `--disable-sandbox`. This is containment SASE
  has with no other provider, and it is genuinely useful for read-only research agents.
  It is an env var rather than config because `llm_provider:` has no per-provider config
  namespace today and inventing one for a single switch is not worth the schema.
  Document that in-run commits will fail under it.
- **No `-w/--worktree`.** It already defaults to `off`; SASE's workspace is the
  workspace. `--subagent-worktree-isolation` is a documented no-op and must not be
  passed.
- **No `llm_default_retry_config` in v1.** Muse retries its own model stream up to 10
  times — the captures show `attempt 1/10` with `error_kind` in `task.lifecycle.status`.
  A nonzero exit falls into SASE's generic retry path. Add patterns only once real
  failures are characterized, and do not reuse the bare `429` / `Too Many Requests`
  strings Codex already owns.

### The event stream

Every event is an envelope:

```json
{
  "schema_version": 1, "id": "...", "sequence": 45,
  "stream": {"kind": "session", "id": "..."},
  "recorded_at": 1786149261442292,
  "record_type": "event | status | reconciliation",
  "durability": "durable | ephemeral",
  "causation_id": null,
  "payload_type": "run.output.delta",
  "payload_schema_version": 1,
  "payload": {"kind": "run_output_delta", "text": "...", "run_stream": {...}}
}
```

stdout is pure JSONL; human diagnostics go to stderr. Parser rules, in priority order:

1. **`run.terminal.completed` → `payload.text` is the authoritative reply.**
   `payload.terminal` is the outcome (`"completed"`) and `payload.reason` the detail.
   Parse those; never pattern-match text.
2. **`run.output.delta` is for live display only.** It is `ephemeral`, and in both
   captures it carries the same text the terminal event later repeats. Stream it into
   `live_reply.md` and the timestamps file; never append it to the returned content, or
   every reply doubles.
3. **A failed, rejected, or cancelled task is not a failed run.** The captures show
   `task.lifecycle.rejected` with `reason: "skip_if_running"` and
   `task.lifecycle.cancelled` with `reason: "main run completed"` on runs that exited
   `0`. A Codex-style `append_error_events` that treats any failure event as an error
   will manufacture spurious failures. Gate success on `run.terminal.*` plus the exit
   code, and record task-level failures as diagnostics only.
4. **Tolerate the unknown.** Unrecognized `payload_type` values and higher
   `schema_version` / `payload_schema_version` numbers must not raise. On a parse
   failure, surface the observed versions through `record_stdout_json_decode_diagnostic`
   rather than returning an empty success.
5. **Exit code 2 is a usage error**, distinct from a run failure. Say so in the raised
   `CalledProcessError` diagnostics so a bad flag does not read as a model failure.

Keep every flag string and payload-type string in one module-level constant block, and
record the observed `muse --version` in run metadata so a fixture can be keyed to the
release that produced it.

### Install and update

Muse breaks four assumptions in `sase/agent_clis/` at once, and each break is addressed
generically rather than with a `if name == "muse"` branch:

| Assumption                    | Reality for Muse              | Fix                                                           |
| ----------------------------- | ----------------------------- | ------------------------------------------------------------- |
| Latest versions come from npm | A JSON channel endpoint       | `install.latest_version_url` + a JSON oracle in `latest.py`   |
| Versions compare with PEP 440 | `0.1.0-R708.1` is not PEP 440 | `install.version_compare: "exact"`                            |
| Update is `<exe> <argv>`      | Update is env-driven          | `install.self_update_env` threaded through `run_command`      |
| Install is npm or a docs link | A remote install script       | `manager: "script"` + `install_script_url` + a new subcommand |

Resulting install metadata:

```python
{
    "manager": "script",
    "display_name": "Muse Code",
    "docs_url": "https://developer.meta.com/ai/resources/blog/build-with-muse-code/",
    "version_argv": ["--version"],
    "version_regex": r"\((?P<version>[^)]+)\)",   # 0.1.0-R708.1, not 0.1.0
    "latest_version_url": "https://api.meta.ai/muse-code/channels/muse-stable",
    "latest_version_json_field": "version",
    "version_compare": "exact",
    "self_update_argv": ["--version"],
    "self_update_env": {"MUSE_SYNC_UPDATE": "1"},
    "install_script_url": "https://dev.meta.ai/install.sh",
    "install_env": {"MUSE_UPGRADE_MODE": "1"},
}
```

`self_update_argv: ["--version"]` is not a trick: with `MUSE_SYNC_UPDATE=1` the launcher
updates itself and the binary and then execs the binary, so the update command and the
version probe are literally the same command, and the existing post-update re-probe in
`execute_agent_cli_updates` reports the new version with no extra machinery.
`MUSE_NO_AUTO_UPDATE` must never be set for this command.

`install_env: {"MUSE_UPGRADE_MODE": "1"}` is the detail that makes SASE-driven
installation safe to offer: without it, the installer appends `export PATH=...` lines to
`~/.zshrc` and friends — which on this machine are chezmoi-managed. SASE installs the
binary and then _reports_ whether the install directory is on `PATH`, leaving the
dotfiles alone.

### Model advisories

The Contributor model is a real feature and a real hazard, and the honest way to ship it
is to make the trade visible everywhere the model is visible. A provider-neutral hook:

```python
@hookspec(firstresult=True)
def llm_model_advisories(self) -> dict[str, dict[str, str]] | None:
    """Per-model advisories surfaced in model-selection UI.

    Maps a model id to ``{"severity": "warn"|"info", "label": <short>,
    "detail": <sentence>}``. Omitting the hook means no advisories, so
    third-party providers stay compatible.
    """
```

Muse returns:

```python
{
    "muse-spark-1.2-contributor": {
        "severity": "warn",
        "label": "trains on your data",
        "detail": (
            "Meta uses this model's inputs and outputs to train and improve "
            "its AI models. Same capabilities as muse-spark-1.2 at roughly "
            "95% lower cost. Rate limited; available in select countries."
        ),
    }
}
```

This is a hook and not a hardcoded Muse special case because the pattern recurs — free
tiers, preview models, and discounted tiers across vendors all carry terms a user should
see at selection time, and none of them should need a new render site.

### Explicit non-goals

Do not wire Muse's hook system; SASE's Claude provider deliberately moved away from
tool-call hooks toward stream parsing. Do not add a `MUSE.md` shim or touch
`PROVIDER_SHIM_FILES`. Do not give Muse an autodetect priority. Do not put `muse resume`
in the retry loop — it is interactive-only; keep the accumulated continuation-prompt
restart the other providers use. Do not project Muse subagents into ACE (see the
follow-up note in `artifacts`). Do not touch `../sase-core`.

---

## cli_meta: Channel-versioned agent-CLI detection and update

Make `sase agent-cli` correct for CLIs distributed by a version channel and updated
through their own launcher. Nothing here is Muse-specific; Muse is the first consumer.

**`src/sase/agent_clis/latest.py`** — generalize the oracle:

- Keep `_fetch_npm_latest_version` exactly as-is.
- Add `_fetch_url_latest_version(url, *, field="version")` using the same
  `urllib.request` pattern, timeout, and `User-Agent`, returning the stripped string at
  `field` or `None`. Reject non-`https://` URLs outright.
- Introduce a small request record — `LatestQuery(key, kind, target, field)` — so
  `get_latest_versions` accepts npm packages and URL endpoints in one call and keeps one
  TTL cache keyed by `key` (`npm:<package>` / `url:<url>`). The existing cache file
  gains no schema change beyond new key shapes; bump `SCHEMA_VERSION` to 2 and let
  unknown-version envelopes fall back to `{}` as they already do.
- Preserve every current behavior: offline never fetches, a stale cached value is
  returned with `offline_stale_cache`, and fetch errors degrade to
  `registry_unavailable` rather than a false "no update".

**`src/sase/agent_clis/models.py`**:

- `InstallMethod` gains `SCRIPT = "script"` for CLIs whose install manager is a remote
  script. Detection precedence: `bundled` → not-installed → npm → homebrew →
  `self_update_argv` ⇒ `SELF_MANAGED` → `UNKNOWN`. A `script`-manager CLI that declares
  `self_update_argv` still classifies as `SELF_MANAGED` (it is self-updating), and
  `SCRIPT` describes only how it is _installed_; carry it on
  `AgentCliStatus.install_manager` rather than overloading `install_method`. Add
  `install_script_url`, `install_env`, `self_update_env`, `latest_version_url`, and
  `version_compare` fields to `AgentCliStatus`, all optional with today's behavior as
  the default.

**`src/sase/agent_clis/runner.py`** — `run_command` gains an optional
`env_overlay: Mapping[str, str] | None` applied over `os.environ.copy()` after `TMPDIR`.
It stays shell-free. Overlay keys and values are logged in the plan preview so
`--dry-run` shows exactly what will run, env included.

**`src/sase/agent_clis/detect.py`**:

- Read the new install-metadata keys through the existing `_optional_str` /
  `_string_tuple` helpers, plus a `_string_map` for env dicts.
- `_install_hint` gains a `script` branch returning
  ``f"run `sase agent-cli install {name}`"`` — the intuitive answer, and better than
  today's fallback of "install from &lt;docs_url&gt;".

**`src/sase/agent_clis/operations.py`**:

- `collect_agent_cli_statuses` builds `LatestQuery` values from both
  `latest_version_package` and `latest_version_url`.
- Replace the bare `is_newer(latest, installed)` call with a comparator selected by
  `status.version_compare`: `"pep440"` (default, today's `is_newer`) or `"exact"`
  (`latest is not None and installed is not None and latest != installed`). Exact
  comparison is right for a channel-pinned distribution: the channel serves one release
  id, and "different from what I have" is precisely "there is an update".
- `plan_agent_cli_status` passes `self_update_env` into the entry so
  `execute_agent_cli_updates` can hand it to `run_fn`.

**`src/sase/doctor/checks_providers.py`** — `setup_hint` gains a `manager == "script"`
branch before the `docs_url` branch, so a script-installed provider's `install:` line
reads ``run `sase agent-cli install <name>` `` instead of being overridden by its docs
URL.

**Tests.** Cover: URL oracle success, malformed JSON, missing field, non-HTTPS
rejection, offline and stale-cache paths; exact-vs-PEP-440 comparator selection,
including the regression that `is_newer("0.1.0-R709", "0.1.0-R708.1")` is `False` and
the exact comparator gets it right; env-overlay threading through plan → execute →
re-probe; and the `script` install hint. Do not hit the network in tests.

## provider: The Muse provider and its JSONL stream parser

**`src/sase/llm_provider/muse.py`**, modeled on `codex.py` and `opencode.py`. Implements
the identity, model, effort, and invocation design above: `llm_provider_name`,
`llm_provider_short_name` (`mus`), `llm_resolve_model_name`, `llm_known_model_names`,
`llm_model_short_aliases`, `llm_skill_template_context` (`provider_name: "Muse Code"`,
`provider_tool_name: "Muse Code"`, `provider_native_ask_tool: "request_user_input"`),
`llm_skill_deploy_subpath` (`.config/muse`), `llm_cli_status_color` (`#0064E0`),
`llm_autodetect_cli_name` (no priority), `llm_auth_evidence`, and
`llm_install_metadata`.

Auth evidence: `credential_paths` `["$MUSE_AUTH_PATH", "~/.config/muse/auth.json"]`,
`api_key_env_vars` `["META_API_KEY"]`.

Executable resolution honors `SASE_MUSE_PATH` (derived for free by
`provider_path_env_var`) and falls back to `shutil.which("muse")`, with a
`FileNotFoundError` that names the env var, the PATH expectation, and
`sase agent-cli install muse`.

Interrupt handling reuses the Qwen/OpenCode/Codex accumulated-context restart:
`start_interrupt_monitor`, `_log_interrupt` into `SASE_ARTIFACTS_DIR`, and a
reconstructed continuation prompt. Muse has no headless resume, so this is the only
option; keep its session log for manual recovery.

**`src/sase/llm_provider/_subprocess_muse.py`** — `stream_and_parse_muse_json_output`
built on the shared `stream_json_lines`, `append_stream_text`, `open_live_reply_file`,
`open_live_reply_timestamps_file`, and `record_stdout_json_decode_diagnostic` helpers,
implementing the five parser rules. Returns
`(content, stderr_content, return_code, usage_totals)`; `usage_totals` is the zeroed
`initial_usage_totals()` in this phase and is filled in by `artifacts`. Re-export it
from `_subprocess.py` alongside the other providers.

**`pyproject.toml`** — `muse = "sase.llm_provider.muse:MuseProvider"` under
`[project.entry-points."sase_llm"]`.

**`src/sase/doctor/checks_providers.py`** — add `_PROVIDER_SETUP_FALLBACKS["muse"]`.
This is required, not cosmetic: `setup_hint` sources the `auth:` line only from that
dict, and no hook publishes it. Use `tool: "Muse Code"`, and `auth: "run `muse
login`, or set META_API_KEY"`.

**Fixtures and tests.** Copy the three captured artifacts into
`tests/llm_provider/fixtures/` under release-keyed names
(`muse_exec_read_tool_R708.1.jsonl`, `muse_exec_write_bash_tools_R708.1.jsonl`,
`muse_session_log_usage_R708.1.jsonl`) and add
`tests/llm_provider/test_muse_provider_core.py` covering: subclassing and registration;
tier and override model resolution; the exact argv, including that `--model` and
`--reasoning-effort` appear only when resolved; all seven effort levels and the
`max → ultra` mapping; `SASE_LLM_*_ARGS` taking precedence over `SASE_MUSE_*_ARGS`;
missing executable; nonzero exit; exit code 2 reported as a usage error;
interrupt-resume prompt reconstruction; no shell interpolation; the prompt file being
written `0o600` and removed; `MUSE_NO_AUTO_UPDATE=1` present in the child env;
`SASE_MUSE_SANDBOX=on` swapping `--disable-sandbox` for `--sandbox-network enabled`; and
— the regression that matters most — a fixture run containing `task.lifecycle.rejected`
and `task.lifecycle.cancelled` with exit `0` parsing as a clean success whose content is
exactly the terminal text, with no duplication from `run.output.delta`.

## cli_install: sase agent-cli install

Add the missing verb. Today SASE can list and update agent CLIs but can only _suggest_
an install, and for a script-installed CLI that suggestion is a bare URL.

**`src/sase/agent_clis/install.py`** (new) — plan and execute an install for a provider
whose metadata declares `install_script_url`:

1. Refuse if already installed unless `--force`.
2. Fetch the script over HTTPS into a `0o600` file under
   `get_sase_managed_tmpdir("agent-clis")`, with a timeout and a size cap. Reject
   non-`https://` URLs and any redirect to a non-HTTPS scheme.
3. Compute and display the SHA-256 of exactly what was downloaded, alongside the URL,
   the interpreter argv (`bash <tmpfile>`), the env overlay, and the resolved install
   directory.
4. Execute through `run_command` with the declared `install_env`. Never `curl | bash`;
   never a shell.
5. Re-probe the version, then report where the binary landed and whether that directory
   is on `PATH`. If it is not, print the exact line to add — and say plainly that SASE
   did not edit any shell rc file. Remove the temp script in a `finally`.

**Confirmation.** Executing a remote script is not something to do implicitly.
`sase agent-cli install <name>` prints the full plan and requires `--yes` to proceed
non-interactively, or an interactive confirmation. `--dry-run` prints the plan —
including the digest of the fetched script — and executes nothing.

**`src/sase/agent_clis/cli_install.py`** — Rich and JSON presentation matching the
existing `cli_list` / `cli_update` shape: a versioned JSON envelope, a bordered panel,
explicit skip reasons, exit `2` for usage errors and unknown names (reusing
`AgentCliUnknownName`), exit `1` for failures.

**`src/sase/main/parser_agent_cli.py`** and **`src/sase/main/agent_cli_handler.py`** —
register the `install` subcommand, update the `metavar="{list,update}"` to
`{list,update,install}`, and add examples. Providers with no `install_script_url` get an
explicit, actionable skip ("Muse Code installs from a script; Codex installs with
`npm install -g @openai/codex`").

**`src/sase/agent_clis/history.py`** — record install runs in the same durable history
as updates, with the trigger and the script digest, so `sase agent-cli` history explains
where a binary came from.

**Tests.** A fake HTTP fetcher and a fake runner cover: HTTPS enforcement, size cap,
digest computation and display, `--dry-run` executing nothing, missing `--yes` refusing
to execute, `install_env` reaching the runner, already-installed handling with and
without `--force`, temp-file cleanup on failure, PATH reporting, and the JSON envelope
shape. No network in tests.

## artifacts: Usage, tool-call, and model-identity artifacts

**Tool calls — `src/sase/llm_provider/_tool_call_muse.py`.** Build records purely from
the stdout stream; no on-disk coupling. The state machine, verified against both
captures:

| Event                                             | Carries                                                                                        | Use                                                                                                                                                                |
| ------------------------------------------------- | ---------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `task.lifecycle.proposed`                         | `task_kind: "tool.<name>"`, `task_id`                                                          | Open a pending call. **Only `task_kind` values starting with `tool.`** — `model.meta.response` and `reminder.agent.plugin:*` tasks must never become tool records. |
| `task.lifecycle.scheduled` / `side_effect_intent` | `idempotency_key: "tool:<call_id>"`, `operation: "tool:<name>"`, `policy_decision`             | Bind `task_id` → `call_id`.                                                                                                                                        |
| `task.lifecycle.output`                           | `event.chunk`                                                                                  | Streamed tool output.                                                                                                                                              |
| `tool.result`                                     | `call_id`, `correlation_facts.{tool_name,outcome}`, optional `edit_facts.{path,added}`, `text` | Close the call with its outcome and result.                                                                                                                        |

Tool _arguments_ are not in the stream. Derive the record's target honestly and in this
order: `edit_facts.path` when present; for `bash`, the `command` and `description`
fields of the result JSON; otherwise a truncated preview of `payload.text` via the
shared `_summarize_tool_input` preview limit. Emit through the same normalized artifact
writers the other providers use, and re-export `append_muse_tool_call_events` from
`_tool_calls.py`. Finalize any call still pending at stream end through
`finalize_pending_tool_calls`.

**Usage — from the session log SASE named.** Usage is absent from stdout and present on
disk. Because the provider passes `--session-id`, the location is deterministic: glob
`~/.local/share/muse/sessions/*/*/*/<session-id>/session.jsonl` (the glob rather than
today's date, so a run spanning midnight still resolves), and honor `XDG_DATA_HOME`.
After the subprocess exits, sum `usage` across `runtime.session` events whose
`payload.event.kind == "model_completed"`:

```json
{
  "kind": "model_completed",
  "model": "muse-spark-1.2-contributor",
  "finish_reason": "tool_calls",
  "duration_ms": 1238,
  "usage": {
    "input_tokens": 15722,
    "output_tokens": 124,
    "cached_tokens": 0,
    "cache_read_tokens": 0,
    "cache_write_tokens": 0,
    "reasoning_tokens": 23
  }
}
```

Map to SASE's totals: `input_tokens` → `input_tokens`, `output_tokens` →
`output_tokens`, `cache_read_tokens or cached_tokens` → `cache_read_input_tokens`,
`cache_write_tokens` → `cache_creation_input_tokens`. Use `model_completed` **only** —
`goal_usage_attribution` events repeat the same numbers for the same call and would
double-count. A missing or unreadable session log is not an error: log a diagnostic and
return zeroed usage, exactly as a provider with no usage support would. Do not shell out
to `muse export`; it costs a subprocess, and `--redacted` strips the `call_id`s while
unredacted output contains verbatim encrypted reasoning SASE has no reason to retain.

**Model identity.** `run.model.configured` carries
`{"model_id": ..., "display_label": ..., "provider_id": "meta", "profile_id": ...}`.
Record `model_id` in run metadata. This closes the observability gap where a run with no
resolved effort or model shows blank in SASE while Muse actually used its own defaults.

**Follow-up, not scope.** Muse's session-lifetime subagents fan out inside one SASE
agent slot, and SASE counts the whole thing as one agent. Accept that for v1 — the model
was co-trained with the harness, and suppressing fan-out trades away the reason to adopt
Muse. Work stays in SASE's claimed workspace (worktree off, subagent isolation shared),
so the commit finalizer sees everything. The captures show the hooks for doing better
later: `task.stream.linked` / `task.lifecycle.*` events, per-subagent
`subagent/<uuid>/session.jsonl` directories, and
`goal_usage_attribution.owner.{owner_type,requester_kind}` distinguishing `main_root`
from subagent usage. File that as a task bead through `/sase_new_task`, along with a
second bead to check whether Muse's `cron_*` tools and per-session `cron.db` can
schedule work that outlives a SASE invocation.

**Tests** run entirely off the recorded fixtures: tool records for `read_file`,
`write_file` (with `edit_facts.path`), and `bash` (with the parsed command); non-tool
tasks producing no records; a pending call at stream end being finalized; usage summed
from the session-log fixture and matching known totals; double-count avoidance; a
missing session log degrading to zeroed usage; and `model_id` capture.

## advisory: Model advisories and the Contributor data-sharing guard

Add `llm_model_advisories` to `_hookspec.py` with the contract above, normalize it in
`_registry_metadata.provider_metadata` as `model_advisories` (defaulting to `{}`, with
non-conforming values dropped rather than raising, so third-party providers cannot break
the registry), and expose it through the metadata payload the same way
`model_short_aliases` is exposed.

Implement it on `MuseProvider` for `muse-spark-1.2-contributor`.

Render it in three places, all reading from the registry so no site hardcodes a model
id:

- **`src/sase/ace/tui/modals/model_picker_rows.py`** — a warning-styled suffix
  (`⚠ trains on your data`) on the row, with the `detail` available in the row's
  secondary text. This is where somebody actually chooses the model, and it is the one
  place the warning must be impossible to miss.
- **`src/sase/xprompt/model_completion.py`** — the advisory `label` in the completion
  detail for `%model:muse/muse-spark-1.2-contributor`.
- **`src/sase/llm_provider/model_label.py`** — mark an active advisory model in the
  resolved label so agent rows and status surfaces show it for the run's whole life, not
  just at selection.

**The guard.** Add a doctor check `llm.model_advisory` that resolves the configured
default and every configured model alias and reports — as a warning, not a failure — any
that land on an advisory-flagged model, naming the alias and quoting the advisory
`detail`. Opting in globally is the user's call; doing it without being told is not.
Keep `_TIER_TO_MODEL` free of advisory models, and assert that in a test so a future
cost optimization cannot quietly reintroduce the problem.

**Tests**: hook normalization including malformed input; a provider without the hook
staying valid; picker row, completion, and label rendering; the doctor check firing for
a configured alias and staying silent otherwise; and the tier-mapping assertion.

## polish: ACE styling and provider badges

Muse currently falls back to the neutral purple provider style and a `None` badge. Give
it an identity:

- **`src/sase/ace/tui/provider_styles.py`** — a `_ProviderStyle` for `muse` and `meta`
  in Meta blue: `name_style="bold #0064E0"`, `delimiter_style="#0064E0"`,
  `model_style="#4A9DFF"`, `secondary_style="#1877F2"`, `dim_style="dim #4A9DFF"`. Check
  the result against the ACE PNG snapshot suite and against a light terminal background
  before settling on the exact shades; the pure brand blue can read too dark on dark
  themes, so lighten `model_style` if needed.
- **`src/sase/integrations/provider_badges.py`** — `"muse": "♾️"` and `"meta": "♾️"`.
  Meta's mark is the infinity loop, it is instantly readable at badge size, and it is
  unused by the existing set (🎭 🤖 🐼 🐙 🪐 🧪).
- **`src/sase/llm_provider/registry.py`** —
  `_PROVIDER_FAMILY_COLORS["meta"] = "#0064E0"`.
- **`src/sase/ace/tui/modals/custom_model_input_modal.py`** — leave the placeholder
  alone unless a nested Muse path reads better; Muse model ids are flat.

Run `just test-visual` and inspect `.pytest_cache/sase-visual/` if any PNG snapshot
shifts; use `--sase-update-visual-snapshots` only for intentional changes.

## docs: Documentation sweep

`docs/llms.md` carries ~23 provider enumeration sites plus per-provider sections;
`docs/configuration.md` has 6, `docs/agent_providers.md` 4, `docs/xprompt.md` 2,
`docs/ace.md` and `docs/plugins.md` 1 each. Update every one, and add:

- **`docs/llms.md`** — a Muse Code section matching the depth of the Codex and OpenCode
  sections: the exact invocation, the model catalog with pricing and the Contributor
  caveat, the full effort map including `max → ultra`, the event model and the five
  parser rules, where usage comes from and why, the tool-argument limitation stated
  plainly, the sandbox default and the `SASE_MUSE_SANDBOX=on` opt-in, `SASE_MUSE_PATH` /
  `SASE_MUSE_LARGE_ARGS` / `SASE_MUSE_SMALL_ARGS`, and the explicit-only selection model
  with the reason autodetect is omitted.
- **`docs/agent_providers.md`** — install and auth, including
  `sase agent-cli install muse` and the fact that SASE passes `MUSE_UPGRADE_MODE=1` so
  the installer does not edit shell rc files.
- **`docs/configuration.md`** — `llm_provider.provider: muse`, the new env vars, and the
  model-advisory behavior.
- **`docs/ace.md`** — the provider badge, palette, and the advisory marker in the model
  picker.
- **`docs/plugins.md`** — the new `llm_model_advisories` hook and the extended
  install-metadata keys (`latest_version_url`, `latest_version_json_field`,
  `version_compare`, `self_update_env`, `install_script_url`, `install_env`), documented
  as optional so third-party providers stay compatible.
- **`docs/xprompt.md`** — `%model:muse/...` and `foo.mus` agent naming.
- **`src/sase/default_config.yml`** — comment examples listing `muse` alongside the
  other providers.

Do not edit `sase/memory/*.md`, `AGENTS.md`, or any generated provider instruction shim
in this epic. If a memory update looks warranted, file a task bead through
`/sase_new_task` and let the user decide.

## verify: Live end-to-end verification

Muse is authenticated on this machine, so verify against the real thing rather than only
fixtures. Prefer `muse-spark-1.2-contributor` for these throwaway checks — it is roughly
95% cheaper and no proprietary code needs to leave the box for a smoke test — but
confirm the default path resolves to `muse-spark-1.2`.

1. Launch a real SASE agent with `%model:muse/muse-spark-1.2` on a trivial task and
   confirm the reply, `live_reply.md` streaming, the tool-call artifact, the usage
   artifact, and the recorded model id.
2. Confirm the skill fix: after `sase init`, check that
   `~/.config/muse/skills/<name>/SKILL.md` exists and is rendered with
   `provider_name: "Muse Code"`, and that `muse skills list --json` picks the
   `.config/muse` copies as winners with the `$HOME/.claude/skills` copies reported as
   shadowed. This is the concrete fix for the 27 Claude-rendered skills Muse loads
   today.
3. Confirm an in-run commit works — a Muse agent invoking `sase_git_commit` must succeed
   with the default sandbox-off invocation, and is expected to fail under
   `SASE_MUSE_SANDBOX=on`. Verify both, and make sure the failure mode under the
   hardened mode is a clear message rather than a confusing permissions error.
4. `sase agent-cli list` shows Muse with its release id and the correct update marker
   against the live channel; `sase agent-cli update muse --dry-run` shows the
   env-carrying command; `sase agent-cli update muse` succeeds (a no-op when current);
   `sase agent-cli install muse --dry-run` prints the URL, digest, and target directory
   without executing.
5. Verify the advisory renders in the ACE model picker and that
   `sase doctor -C llm.model_advisory -v` is silent for the shipped defaults.
6. `just check-full`, not `just check` — this epic touches the broadening set. Fix any
   Symvision findings through `/sase_memory_read` on `sase/memory/symvision.md` rather
   than by adding pragmas.

Record the observed `muse --version` in the final commit message so the fixtures are
traceable to a release. If Muse has shipped a new release by then and any fixture no
longer matches, re-capture rather than loosening an assertion — the fixtures exist
precisely so a beta rename is a one-line fix.
