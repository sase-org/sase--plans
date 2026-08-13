---
tier: epic
title: Grok Build LLM provider
goal: 'SASE gains a first-class `grok` LLM provider driving xAI''s Grok Build CLI,
  supported everywhere the existing providers are — invocation, streaming text, reasoning
  pane, Tools panel, usage/cost accounting, doctor, agent-cli inventory and updates,
  model routing, skill deployment, TUI theming, and docs — with tool rows and reasoning
  that render as richly as Claude''s rather than degrading to opaque key lists.

  '
phases:
- id: wire
  title: Provider-neutral Messages-wire stream layer
  depends_on: []
  size: medium
  description: 'wire: generalize the Claude stream-json parser into a provider-parameterized
    Anthropic-Messages reader — runtime tagging, `errors[]` diagnostics, a thinking-block
    sink, and a pluggable tool-call writer seam — leaving Claude behavior byte-identical.'
- id: tools
  title: Grok tool-call normalizer
  depends_on:
  - wire
  size: medium
  description: 'tools: add `_tool_call_grok.py` mapping Grok''s snake_case tool names
    and JSON-string tool_result envelopes onto SASE''s canonical display names and
    structured summaries so ACE Tools rows show real commands and paths.'
- id: provider
  title: The grok provider module
  depends_on:
  - tools
  size: medium
  description: 'provider: add `src/sase/llm_provider/grok.py` and its `sase_llm` entry
    point — hooks, tier/model mapping, the verified four-level effort table, the invocation
    vector, the interrupt/continue loop, and unit tests over recorded fixtures.'
- id: identity
  title: Doctor, inventory, and binary-collision safety
  depends_on:
  - provider
  size: small
  description: 'identity: wire Grok into `sase doctor` and `sase agent-cli` install/update/version
    surfaces, and make the contested `grok` executable name fail loudly and actionably
    instead of silently launching an unrelated binary.'
- id: palette
  title: Badge, palette, and model-surface polish
  depends_on:
  - provider
  size: small
  description: 'palette: give Grok its emoji badge, TUI color palette, `default_config.yml`
    provider-list entry, and correct rendering in model pickers and provider-labeled
    rows.'
- id: skills
  title: Skill deployment and instruction files
  depends_on:
  - provider
  size: small
  description: 'skills: verify SASE skills deploy to and load from `~/.grok/skills/`,
    confirm the native-over-compat precedence and the no-shim AGENTS.md path, and
    record the CLAUDE.md double-load as a follow-up.'
- id: docs
  title: Documentation sweep
  depends_on:
  - identity
  - palette
  - skills
  size: medium
  description: 'docs: document Grok Build across the provider, configuration, install,
    and LLM reference docs, including the auto-approve/no-sandbox posture, the effort
    ceiling, and the best-effort usage caveat.'
- id: smoke
  title: Authenticated end-to-end smoke exercises
  depends_on:
  - docs
  size: xsmall
  description: 'smoke: launch real SASE agents on the grok provider to confirm text,
    reasoning, tool rows, usage, skills, interrupt/relaunch, and failure diagnostics
    behave in ACE.'
proposed_by: bbugyi200.athena.zu
create_time: 2026-08-13 14:40:32
status: wip
bead_id: sase-l3
---

- **PROMPT:** [prompts/202608/grok_provider.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/grok_provider.md)
- **BEAD:** [sase-l3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-l3/README.md)

# Plan: Grok Build LLM provider

## Context

A prior research pass (`202608/grok_build_provider/grok_build_provider.md` in the
research sidecar) recommended building a native in-tree `grok` provider around xAI's
**Grok Build** CLI (`@xai-official/grok`) in headless
`--output-format streaming-messages-json` mode. That recommendation stands and this plan
adopts it.

The research was written against an **unauthenticated** binary. Planning re-ran its open
questions against an authenticated Grok Build `1.0.3 (1a29d5bc12) [stable]`. Several
findings changed, and three integration surfaces the research did not consider turn out
to carry most of the quality risk. Everything below marked **[verified]** was measured
directly; where it contradicts the research doc, this plan wins.

### Verified corrections to the research

| Research said                                      | Verified reality                                                                                                                       | Consequence                                                                |
| -------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| Model is `grok-4.5`                                | **`grok-4.6`** is the only model in the authenticated catalog, and the default                                                         | Ship `grok-4.6`; `grok-4.5` does not exist for this account                |
| Effort maps 1:1; "pass all seven through"          | `grok-4.6` accepts **only** `low`, `medium`, `high`, `xhigh`. `none`, `minimal`, and `max` are rejected by the CLI with a nonzero exit | Declare exactly the four; SASE raises a clean error instead of a CLI crash |
| `total_cost_usd` likely blank on subscription runs | Populated on OAuth (`0.028946` on a trivial turn), with a per-model `modelUsage` ledger                                                | Cost accounting works on the subscription path                             |
| Reuse `_tool_call_claude` verbatim                 | It hardcodes `runtime: "claude"` and keys its rich summaries on Claude's PascalCase tool names                                         | Grok needs its own normalizer — see `tools`                                |
| Failure detail via `error`/`message`/`result`      | Failure frames carry **`errors[]`** only                                                                                               | Confirmed; the `errors[]` fold is required                                 |

### Verified live wire format

A no-tool turn emits three lines: `system`/`init`, one `assistant` message whose
`message.content[]` holds `thinking` and `text` blocks, and a terminal `result`. A
tool-using turn adds `assistant` messages with `tool_use` blocks and `user` messages
with `tool_result` blocks. `result.usage` carries exactly the four keys
`initial_usage_totals()` accumulates, plus a nested `server_tool_use` that SASE's
accumulator ignores harmlessly.

The deliberate-failure frame (bad `--model`, exit `1`) is:

```json
{
  "type": "result",
  "subtype": "error_during_execution",
  "is_error": true,
  "usage": {
    "input_tokens": 0,
    "output_tokens": 0,
    "cache_read_input_tokens": 0,
    "cache_creation_input_tokens": 0,
    "server_tool_use": { "web_search_requests": 0 }
  },
  "modelUsage": {},
  "errors": [
    "Couldn't set model 'definitely-not-a-model': Invalid params: \"unknown model id\"..."
  ],
  "session_id": "...",
  "uuid": "..."
}
```

`_subprocess_claude._process_json_line` reads
`event.get("error") or event.get("message") or event.get("result", "")` — none of which
are present — so today SASE would surface an empty detail.

### Verified flag surface

All of `--no-auto-update`, `--no-ask-user`, `--yolo`, `--no-plan`, `--no-subagents`,
`--no-memory`, `--no-leader`, and `--max-turns` parse successfully but are absent from
`--help`; a control `--bogus-flag-xyz` is rejected with `error: unexpected argument`.
`--prompt-file /dev/stdin` works end to end.

### Verified environment and config facts

- `grok inspect --json` reports **both** `CLAUDE.md` and `AGENTS.md` as project
  instructions (~2,930 tokens each, identical content). `[compat.claude] agents = false`
  does **not** suppress it — generic top-level `CLAUDE.md` stays recognized.
- `[compat.claude]` toggles _do_ work for skills, rules, and hooks: disabled entries are
  still listed by `inspect` but marked `compatibilityStatus: "disabled"`.
- A skill at `$GROK_HOME/skills/<name>/SKILL.md` is discovered, and a **native Grok
  skill shadows a same-named Claude-compat skill** — only one entry survives. SASE's
  default `.grok` deploy subpath is therefore correct and safe with compat left on.
- Grok exports `GROK_AGENT=1` to tool subprocesses but **no** `GROK_PROJECT_DIR`. SASE's
  existing `SASE_ACTIVE_PROJECT_DIR` covers workspace pinning; `env_contracts.py` needs
  no new entry.
- The native tool set is `run_terminal_command`, `read_file`, `write`, `search_replace`,
  `list_dir`, `grep`, `todo_write`, `spawn_subagent`, `web_search`, `web_fetch`,
  `ask_user_question`, and others.
- `ask_user_question` is confirmed present, settling the `provider_native_ask_tool`
  value.

### The three surfaces the research missed

1. **Tool-row fidelity.** `_summarize_tool_input` branches on Claude's `Bash`, `Read`,
   `Write`, `Edit`, `Grep`, `Glob`, `WebFetch`, `WebSearch`, `Task`. Grok's names are
   all different, so reusing the Claude writer drops every Grok row into the generic
   `{"input_keys": [...]}` fallback. Both `qwen` and `agy` already solve this by mapping
   native names onto canonical display names — Grok must too.
2. **Runtime mislabeling.** `_tool_call_claude._base_record` hardcodes
   `"runtime": "claude"`, and that value is both rendered to the user
   (`_report_render.py`: `**Runtime**: claude`) and used as part of the
   ToolUse/ToolResult pairing key. Grok rows would claim to be Claude runs.
3. **The reasoning pane.** Grok emits real `thinking` blocks with signatures. ACE has a
   whole thinking pane reading `codex_thinking.jsonl`; the Claude parser discards
   non-`text` blocks. Routing Grok's thinking into that pane is a small change and the
   single biggest visible quality win.

### Design decisions

**Effort ceiling is declared, not discovered.** `_effort_args.effort_cli_args` already
implements the right contract: an explicit `%effort` a provider cannot honor raises
`LLMInvocationError`, a config default is logged and skipped. Declaring exactly
`{low, medium, high, xhigh}` turns `%effort:max` into an immediate, readable SASE error
instead of a Grok process failure mid-launch. Note the consequence explicitly in docs:
the shipped `@smartest` alias is `claude/opus@max`, so a Grok run must name an effort
Grok supports.

**Subagents stay enabled.** The research flagged that subagent usage can set Grok's
internal `usage_is_incomplete` flag, which the Messages projection drops. That degrades
_telemetry_, not text or tool records, and SASE treats token counts as telemetry.
Passing `--no-subagents` would trade a real capability for an accounting nicety. Keep
them; state the caveat in docs.

**Native planning and asking are disabled.** `/sase_plan` owns planning handoffs and
`/sase_questions` owns asking, so pass `--no-plan` and `--no-ask-user`. Both are
undocumented in `--help`, so they get parse-probe tests.

**`--no-leader` is passed explicitly.** Leader mode is off by default but opt-in via
`[cli] use_leader = true` in a user's own config. SASE runs many agents concurrently
against one shared backend socket; a user's unrelated config setting must not be able to
reshape that. Explicit beats inherited.

**No sandbox profile is set.** SASE skills need controlled access to state outside the
checkout, matching what SASE already does for Codex, OpenCode, and Muse.

**Explicit-only registration.** `grok-dev` (a stale community CLI) installs a binary
named `grok` and also uses `~/.grok/`; Homebrew ships a deprecated unrelated `grok`
regex tool. Declare `llm_autodetect_cli_name` but no `llm_autodetect_priority`, exactly
as `muse` does, and make the collision a loud failure rather than a confusing one.

**No `GROK.md` shim.** Grok reads `AGENTS.md` natively **[verified]**, so
`PROVIDER_SHIM_FILES` in `src/sase/amd/constants.py` is deliberately unchanged.

**No Rust core changes.** `llm_provider` is carried as `Option<String>` throughout
`sase-core`; the whole change is Python-side, consistent with the Rust core boundary
rule.

---

## Provider-neutral Messages-wire stream layer

Generalize `src/sase/llm_provider/_subprocess_claude.py` so a second provider can share
the Anthropic-Messages reader without inheriting Claude's identity. Claude's observable
behavior must not change; prove it with regression tests over existing Claude fixtures.

Introduce a small parameter object (or explicit keyword arguments) threaded from
`stream_and_parse_json_output` down into `_process_json_line`, carrying:

- **`runtime`** — the provider name used for stdout JSON-decode diagnostics
  (`record_stdout_json_decode_diagnostic` currently hardcodes `"claude"`).
- **`tool_call_writer`** — the callable invoked per event, defaulting to
  `append_claude_tool_call_event`. This is the seam the `tools` phase implements
  against; its signature stays `(Mapping[str, Any]) -> None`.
- **`thinking_sink`** — an optional writer for `thinking` content blocks. When set, each
  `thinking` block in an `assistant` message is appended to the reasoning JSONL that
  `open_codex_thinking_file()` opens. Keep the `codex_thinking.jsonl` filename: ACE's
  `read_codex_thinking` reads that exact path and renaming it is a separate, larger
  change with no user-visible benefit. Write records in the shape
  `src/sase/ace/tui/thinking/parser.py` already parses, and verify against that parser
  rather than by inspection.

Extend the failure-detail extraction to fold `errors[]`:

```python
detail = event.get("error") or event.get("message") or event.get("result", "")
if not detail:
    errors = event.get("errors")
    if isinstance(errors, list):
        detail = "\n".join(str(item) for item in errors if item)
```

This is safe by construction for Claude: `append_error_events` in
`_subprocess_stream.py` returns early when `return_code == 0`, so a success-path
`result.result` is never mistaken for an error, and Claude never emits `errors[]` at
all.

Export the generalized entry point through `_subprocess.py` alongside the existing
compatibility aliases, following the naming already used for the other providers.

**Done when:** the full existing Claude provider test suite passes unchanged; new tests
cover the `errors[]` fold, the runtime tag reaching a decode diagnostic, and a thinking
block reaching the ACE thinking parser.

## Grok tool-call normalizer

Add `src/sase/llm_provider/_tool_call_grok.py`, modeled on the structure of
`_tool_call_qwen.py` and `_tool_call_agy.py`, exporting `append_grok_tool_call_event`
and `normalize_grok_tool_call_event`, and re-exported from `_tool_calls.py` with its
`__all__` entry.

It walks the same `assistant` → `tool_use` / `user` → `tool_result` shape the Claude
normalizer walks, but differs in three ways.

**1. Runtime.** Every record carries `"runtime": "grok"`.

**2. Display-name mapping.** Map Grok's native names onto SASE's canonical display names
so the shared summarizers in `_tool_call_common.py` produce rich previews:

| Grok tool              | Canonical display name |
| ---------------------- | ---------------------- |
| `run_terminal_command` | `Bash`                 |
| `read_file`            | `Read`                 |
| `write`                | `Write`                |
| `search_replace`       | `Edit`                 |
| `grep`                 | `Grep`                 |
| `list_dir`             | `Glob`                 |
| `web_fetch`            | `WebFetch`             |
| `web_search`           | `WebSearch`            |
| `spawn_subagent`       | `Task`                 |
| `todo_write`           | `TodoWrite`            |

Anything unmapped falls through under its own name — never drop a row for being unknown.
Normalize inputs into the field names the canonical summaries expect where they differ,
the way `_normalize_qwen_tool_input` and `_alias_first` already do.

**3. Result envelope decoding.** This is the part with no precedent. Grok's
`tool_result` blocks carry `content` as a **JSON-encoded string**, not text, and the
decoded object is a bespoke tagged shape:

```json
{"type":"Bash","output":[53,32,...],"output_for_prompt":"exit: 0\n5 hello.txt\n",
 "exit_code":0,"command":"wc -c hello.txt","truncated":false,"signal":null,
 "timed_out":false,"description":"...","current_dir":"/tmp/probe","total_bytes":12}
```

```json
{"type":"SearchReplace","EditsApplied":{"old_string":"","new_string":"HELLO",
 "tool_output_for_prompt":"The file /tmp/probe/hello.txt has been created.",
 "absolute_path":"/tmp/probe/hello.txt","edits":{"details":[...]}}}
```

Decode it and hand `summarize_tool_response` a structured envelope rather than an opaque
blob. Prefer `output_for_prompt` / `tool_output_for_prompt` for previews — they are the
human-readable projection Grok itself uses. Note that `output` is a **byte array**, not
a string; never preview it raw. Map `exit_code` through so `Bash` rows show exit status,
and `absolute_path` so edit rows show the file. A `content` string that is not valid
JSON must degrade to the existing plain-text preview path, not raise.

Grok's `user` messages carry no top-level `tool_use_result` envelope (verified: keys are
exactly `message`, `parent_tool_use_id`, `session_id`, `type`, `uuid`), so the decoded
`content` is the only structured source. Grok's tool-call ids are
`call-<uuid>-<n>`-shaped, which pair correctly through the existing id-based logic.

**Done when:** ACE reader tests analogous to `tests/ace/tui/tools/test_reader_qwen.py`
show a Grok `run_terminal_command` row rendering its command and exit code, a
`search_replace` row rendering its file path, rows labeled `runtime: grok`, an unmapped
tool name surviving as a row, and a non-JSON `content` degrading cleanly.

## The grok provider module

Add `src/sase/llm_provider/grok.py` (`GrokProvider`), closely modeled on `claude.py`,
and register it in `pyproject.toml` under `[project.entry-points."sase_llm"]` as
`grok = "sase.llm_provider.grok:GrokProvider"`.

**Hooks.**

| Hook                         | Value                                                                                                                                                                                                                              |
| ---------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `llm_provider_name`          | `"grok"`                                                                                                                                                                                                                           |
| `llm_provider_short_name`    | `"grk"`                                                                                                                                                                                                                            |
| `llm_resolve_model_name`     | both tiers → `grok-4.6`                                                                                                                                                                                                            |
| `llm_known_model_names`      | `["grok-4.6"]`                                                                                                                                                                                                                     |
| `llm_skill_template_context` | `provider_name: "Grok"`, `provider_tool_name: "Grok Build"`, `provider_native_ask_tool: "ask_user_question"`                                                                                                                       |
| `llm_autodetect_cli_name`    | `"grok"`                                                                                                                                                                                                                           |
| `llm_autodetect_priority`    | **omitted** — explicit-only                                                                                                                                                                                                        |
| `llm_skill_deploy_subpath`   | **omitted** — the default `.grok` is already correct                                                                                                                                                                               |
| `llm_cli_status_color`       | the primary from the `palette` phase                                                                                                                                                                                               |
| `llm_auth_evidence`          | `credential_paths: ["~/.grok/auth.json"]`, `api_key_env_vars: ["XAI_API_KEY"]`                                                                                                                                                     |
| `llm_install_metadata`       | `manager: "npm"`, `package: "@xai-official/grok"`, `scope: "global"`, `display_name: "Grok Build"`, `docs_url: "https://docs.x.ai/build/overview"`, `self_update_argv: ["update"]`, `latest_version_package: "@xai-official/grok"` |
| `llm_default_retry_config`   | xAI transport/rate-limit wording                                                                                                                                                                                                   |

Both tiers map to `grok-4.6` because it is the only model the authenticated catalog
offers. Inventing a distinct `small` mapping to a model that may not exist would make
ordinary `@cheap`/`@cheaper` routing fail. Revisit when the catalog grows. No
`llm_model_short_aliases` is needed — `grok-4.6` is already short — and no
`version_regex`: `grok --version` prints `grok 1.0.3 (1a29d5bc12) [stable]` and the
default `_SEMVER_RE` in `agent_clis/detect.py` extracts `1.0.3` correctly
**[verified]**.

Keep the retry `error_patterns` narrow and xAI-specific so they cannot collide with
Codex's ownership of generic `429` / `Too Many Requests` wording.

**Effort table.**

```python
_EFFORT_CLI_ARGS: dict[str, list[str]] = {
    level: ["--effort", level] for level in ("low", "medium", "high", "xhigh")
}
```

`none`, `minimal`, and `max` are deliberately absent: `grok-4.6` rejects all three by
name with a nonzero exit **[verified]**.

**Invocation vector.**

```python
[
    _resolve_grok_executable(),          # SASE_GROK_PATH, then PATH, then "grok"
    "--prompt-file", "/dev/stdin",
    "--output-format", "streaming-messages-json",
    "--permission-mode", "bypassPermissions",
    "--model", model,
    "--cwd", os.getcwd(),
    "--session-id", str(uuid.uuid4()),
    "--no-plan",
    "--no-ask-user",
    "--no-auto-update",
    "--no-leader",
    *effort_args,
]
```

The prompt goes to `process.stdin` exactly as `claude.py` already does, so there is no
temp file to leak or clean up on interrupt and no argv exposure of prompt text. Use
`--permission-mode bypassPermissions` rather than the undocumented `--yolo`.
`--no-auto-update` is not optional: without it Grok may replace its own 166 MB binary
mid-run, and SASE owns updates through `sase agent-cli`.

Carry over from `claude.py`: the `SASE_LLM_{LARGE,SMALL}_ARGS` →
`SASE_GROK_{LARGE,SMALL}_ARGS` passthrough, `provider_timer("Waiting for Grok")`,
`start_interrupt_monitor`, the interrupt/continue loop, `_log_interrupt`, and raising
`subprocess.CalledProcessError` on a nonzero exit with output and stderr attached. Add a
`_grok_executable_not_found_error` mirroring Muse's, naming `SASE_GROK_PATH` and
`sase agent-cli install grok`.

**Fixtures.** Record version-pinned NDJSON traces under `tests/fixtures/grok_stream/`
following the `tests/fixtures/qwen_stream/` precedent, named with the CLI version
(`grok_messages_notool_1.0.3.jsonl`, `grok_messages_tools_1.0.3.jsonl`,
`grok_messages_error_1.0.3.jsonl`). Capture them with
`grok --prompt-file /dev/stdin --output-format streaming-messages-json --permission-mode bypassPermissions --cwd <dir>`
for a no-tool prompt, a write-plus-shell prompt, and a deliberate `--model` failure.
Redact session ids and absolute home paths.

**Done when:** unit tests cover exact argv, tier and model-override mapping, each of the
four accepted effort levels, `LLMInvocationError` for an explicit `max`/`none`/`minimal`
and warn-and-skip for the same as a config default, the four usage keys accumulating
from a recorded trace, text split across arbitrary line boundaries, `errors[]` surfacing
in the raised diagnostics, missing-executable failure, and partial-output preservation
across an interrupt. Add parse-probe tests pinning `--no-plan`, `--no-ask-user`,
`--no-auto-update`, and `--no-leader` as accepted so a future Grok release that drops
one fails a test rather than an agent run.

## Doctor, inventory, and binary-collision safety

Add Grok to `_PROVIDER_HINTS` in `src/sase/doctor/checks_providers.py`:

- `tool`: `Grok Build`
- `install`: `npm install -g @xai-official/grok`
- `auth`:
  ``run `grok login` (or `grok login --device-code` on a headless host), or set `XAI_API_KEY` ``

Auth detection stays offline evidence only — a cached `~/.grok/auth.json` or a set
`XAI_API_KEY`. Never attempt a network login and never manage the credential.

**Collision safety.** Three distinct tools compete for the executable name `grok`, and
SASE's autodetect only checks PATH presence. Explicit-only registration prevents an
unrelated `grok` from _winning_ the default provider, but it does not stop an explicit
`llm_provider.provider: grok` from launching the wrong one. Add a doctor-level identity
advisory: when a `grok` is discoverable but its `--version` output does not look like
Grok Build, report it as a distinct, actionable finding naming `grok-dev` and Homebrew's
deprecated regex tool as the likely culprits and pointing at `SASE_GROK_PATH`. Keep this
in doctor and in the not-found/failure diagnostics, not on the per-invocation hot path.

Confirm Grok appears correctly in `sase agent-cli list`, that version detection reports
`1.0.3` from the real `--version` string, and that `sase agent-cli update grok` resolves
through the npm rail. Note in the code comment that the npm package is a trampoline and
the real binary lives under `~/.grok/bin/`, not `node_modules`.

**Done when:** doctor tests cover the install hint, the auth hint, both auth-evidence
paths, and the wrong-binary advisory; agent-cli tests cover version parsing and the npm
update path.

## Badge, palette, and model-surface polish

**Emoji badge** in `src/sase/integrations/provider_badges.py`: `🛰️` for `grok`, with
`xai` registered as an alias pointing at the same badge — matching how `anthropic`,
`openai`, and `meta` alias onto `claude`, `codex`, and `muse`. Deliberately not a mark
like `✖️`: a red X beside an agent row reads as failure at a glance, which is exactly
the wrong signal for a healthy run.

**Palette** in `src/sase/ace/tui/provider_styles.py`, with the same `xai` alias entry.
Cyan is the one clearly unclaimed hue among the existing providers (claude orange, codex
green, qwen purple, opencode yellow, agy indigo, muse blue, fakey pink):

```python
"grok": _ProviderStyle(
    name_style="bold #00C8D7",
    delimiter_style="#00A0AD",
    model_style="#5FE3EF",
    secondary_style="#00A0AD",
    dim_style="dim #7FD4DD",
),
```

Use `#00C8D7` as `llm_cli_status_color` so the plugin-declared primary and the fallback
palette agree. xAI's own branding is monochrome and would be unreadable in a dense TUI
row.

**Provider list** in `src/sase/default_config.yml`: add `grok` to the
`llm_provider.provider` comment and extend the existing explicit-only sentence —
currently written only about `muse` — to cover both providers that declare no autodetect
priority.

Check the model picker, agent list rows, and reasoning-effort metadata display render
Grok correctly, and add a Grok case to the provider-badge widget tests. Extend the ACE
PNG visual snapshot fixtures only if a snapshot actually exercises provider color; do
not regenerate goldens speculatively.

**Done when:** badge and style tests cover `grok` and `xai`, and `just test-visual` is
clean (or its goldens are deliberately re-accepted with the diff reviewed).

## Skill deployment and instruction files

`skill_deploy_subpaths()` in `src/sase/main/_init_skills_sources.py` defaults to
`f".{provider}"`, so `grok` yields `.grok` with no hook — **[verified]** that
`~/.grok/skills/<name>/SKILL.md` is discovered and that underscored directory names are
normalized to dashed skill names.

Confirm end to end that `sase init skills` writes Grok-rendered skills there and that
they render `Grok`, `Grok Build`, and `ask_user_question` from the provider's
`llm_skill_template_context` — `src/sase/xprompts/skills/sase_questions.md` branches on
`provider_native_ask_tool == "AskUserQuestion"`, so verify the non-Claude branch reads
correctly for Grok.

**Compat overlap.** Grok's `[compat.claude]` cells default to on, so a Grok run also
sees `~/.claude/skills/`. This is **[verified]** benign for SASE: a native
`~/.grok/skills/<name>` shadows a same-named Claude-compat skill entirely, leaving one
entry. Since `sase init skills` deploys every SASE skill to every registered provider's
subpath, SASE skills are always shadowed by their correctly-rendered Grok copies. Add a
test pinning that assumption so a future skill-deploy change that breaks parity is
caught. Do **not** write into the user's `~/.grok/config.toml`.

**Instruction files.** No `GROK.md` shim: `AGENTS.md` is native, so
`PROVIDER_SHIM_FILES` in `src/sase/amd/constants.py` and the memory inventory stay
unchanged. Grok does, however, load **both** `CLAUDE.md` and `AGENTS.md` as project
instructions — identical content, ~2,930 tokens each, injected twice — and
`[compat.claude] agents = false` does not suppress it **[verified]**. Accept it for now:
suppressing `CLAUDE.md` generation under a Grok provider would break any human running
`claude` in the same tree. Use `/sase_new_task` to file a follow-up task bead proposing
a narrower fix, and record the behavior in the `docs` phase.

**Done when:** a Grok-rendered skill is confirmed deployed and discovered, the shadowing
assumption has a test, and the follow-up bead exists.

## Documentation sweep

Add a Grok Build section to `docs/agent_providers.md` mirroring the Muse section's
shape: install (`npm install -g @xai-official/grok`), auth (`grok login`,
`grok login --device-code`, or `XAI_API_KEY`), updates via `sase agent-cli update grok`,
explicit-only selection (`llm_provider.provider: grok`, `%model:grok/grok-4.6`, or
`SASE_GROK_PATH`), and the executable-name collision with `grok-dev` and Homebrew's
deprecated regex tool.

Four things must be documented plainly rather than discovered by surprise:

1. **Execution posture.** SASE runs Grok with `--permission-mode bypassPermissions` and
   no sandbox profile. That is powerful local execution.
2. **Effort ceiling.** Grok supports `low`, `medium`, `high`, `xhigh` only.
   `%effort:max`, `%effort:none`, and `%effort:minimal` raise an error rather than
   silently downgrading — and the shipped `@smartest` alias uses `@max`, so a Grok run
   must name a supported level.
3. **Usage is best-effort.** Grok's `streaming-messages-json` output is a projection of
   its native usage ledger that drops the "usage incomplete" marker; subagent turns and
   interrupted turns can under-count or zero out. Text and tool records are unaffected.
   Cost reporting does work on the OAuth path.
4. **Instruction double-load.** Grok reads both `AGENTS.md` and SASE's generated
   `CLAUDE.md` shim, with the follow-up bead referenced.

Also link current xAI data and privacy terms without implying zero data retention. Then
sweep the sites that enumerate providers: `docs/configuration.md`, `docs/llms.md` (by
far the largest footprint), `docs/getting_started.md`, `docs/ace.md`, `docs/plugins.md`,
`docs/xprompt.md`, `INSTALL.md`, and `README.md`. Use the existing `muse` and `agy`
mentions in each file to locate the lists that need a new entry; the goal is that no
provider-enumerating list is left with a hole.

**Done when:** every provider list in the docs tree includes Grok, and the four caveats
above appear in `docs/agent_providers.md`.

## Authenticated end-to-end smoke exercises

Because this broadens the provider registry and touches subprocess, doctor, model, and
skill paths, run `just install` then `just check-full` through `/sase_monitor` — not
inline, and not only the provider unit tests.

Then launch real SASE agents on the grok provider and confirm each of these in ACE
rather than from test output alone:

1. A no-tool prompt: response text, live reply streaming, and nonzero usage.
2. A file edit and a shell command: Tools panel rows show the actual command, exit code,
   and file path, and are labeled `grok` — not `claude`.
3. The reasoning pane renders Grok's thinking blocks.
4. A SASE skill that writes state outside the checkout, confirming the no-sandbox
   posture works as documented.
5. An intentional model error: the `errors[]` detail reaches SASE's diagnostics.
6. An interrupt and relaunch: partial output is preserved and the continue loop resumes.
7. `sase doctor` reports Grok correctly, both with and without `grok` on PATH.

**Done when:** all seven behave correctly, or any that do not are captured as task beads
via `/sase_new_task` with the observed output attached.
