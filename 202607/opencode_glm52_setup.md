---
tier: tale
title: Install OpenCode and wire up GLM-5.2 behind a custom SASE gate
goal:
  OpenCode is installed and authenticated for GLM-5.2 on athena through a reusable custom SASE gate, and the supporting
  chezmoi-managed config files (OpenCode global config plus explicit GLM model aliases) exist and resolve correctly.
proposed_by: bbugyi200.athena.qt
create_time: 2026-07-31 18:05:03
status: done
---

- **PROMPT:** [202607/prompts/opencode_glm52_setup.md](prompts/opencode_glm52_setup.md)

# Plan: Install OpenCode and wire up GLM-5.2 behind a custom SASE gate

## 1. Background

The research sidecar's consolidated report `202607/glm_5_2_sase_rollout/glm_5_2_sase_rollout.md` (plus its two source
reports `__a.md` and `__b.md`) recommends running GLM-5.2 through SASE's **existing** `opencode` LLM provider rather
than building a new provider or redirecting Claude Code at Z.AI. No SASE code change is required. The only missing
pieces on athena are:

1. the OpenCode CLI itself (not installed),
2. an OpenCode credential for Z.AI,
3. SASE model aliases that name GLM-5.2 explicitly, and
4. one OpenCode global config file so OpenCode agents see the same home-level SASE memory every other runtime sees.

This tale delivers all four, with the install and authentication performed by a **custom SASE gate** so the user
approves the exact commands before they run and no agent ever handles the API key.

### Facts verified while planning (do not re-derive)

- **Provider/model IDs.** models.dev declares `zai` (metered, `https://api.z.ai/api/paas/v4`) and `zai-coding-plan`
  (subscription, `https://api.z.ai/api/coding/paas/v4`). Both declare env var `ZHIPU_API_KEY` and both carry model
  `glm-5.2` (1,000,000-token context, 131,072 output).
- **Effort variants are `high` and `max` only.** OpenCode generates GLM-5.2 variants from a hard-coded list
  (`["high", "max"]`) and rejects anything else with `VariantUnavailableError`. The effective user config sets
  `llm_provider.default_effort: xhigh`, and SASE's OpenCode adapter forwards **every** canonical effort level as
  `--variant <level>`. **Therefore every GLM alias must pin `@high` or `@max`**; an unpinned GLM model would be launched
  as `--variant xhigh` and fail at runtime. Alias effort beats the configured default (precedence is explicit
  `%effort` > alias effort > temporary override > configured default), so a pinned alias is sufficient — but an explicit
  `%effort:` on the launch prompt would still override it and break the run.
- **SASE already resolves the nested IDs.** `opencode/zai/glm-5.2@max` resolves to `('opencode', 'zai/glm-5.2', 'max')`
  and `opencode/zai-coding-plan/glm-5.2@high` to `('opencode', 'zai-coding-plan/glm-5.2', 'high')`.
- **Auth store format.** `opencode auth login` (an alias of `opencode providers login`) writes
  `$XDG_DATA_HOME/opencode/auth.json` (i.e. `~/.local/share/opencode/auth.json`, mode `0600`) as
  `{"<provider-id>": {"type": "api", "key": "<key>"}}`. There is **no** Z.AI-specific auth plugin, so no extra metadata
  is needed. SASE's `OpenCodeProvider.llm_auth_evidence` already lists that path and (as of commit `bc359cca6`) the
  `ZHIPU_API_KEY` env var.
- **`opencode auth login` cannot be automated.** The API-key step is an interactive `Prompt.password` TUI prompt; there
  is no `--api-key` flag (only `--provider`/`--method`). Gate commands run with no TTY, so this plan writes the same
  `auth.json` entry directly and then verifies it with the real CLI.
- **OpenCode's instruction discovery.** `session/instruction.ts` loads the first existing global file from
  `~/.config/opencode/AGENTS.md` then `~/.claude/CLAUDE.md` (neither exists on athena), plus project `AGENTS.md` found
  by walking **only** from cwd up to the worktree root, plus every path in the config's `instructions` array (`~/` and
  absolute paths supported). Claude Code picks up `~/AGENTS.md`-equivalent home memory by walking the whole tree to
  `$HOME`; OpenCode would **not**. That gap is what the new `opencode.json` closes.
- **Install method.** `npm install -g opencode-ai` (currently 1.18.10), matching `llm_install_metadata` and the other
  npm-managed CLIs on this machine. npm here is nvm-managed at `~/.config/nvm/versions/node/<version>/bin/npm`, so gate
  commands must resolve npm defensively.
- **Gate execution contract.** `sase gate create` hashes every command resource; the executor runs the selected commands
  with cwd set to the bundle, stdin = the gate input JSON, **no TTY**, no timeout, and requires **exactly one JSON value
  on stdout** that satisfies the option's `result_schema`. Anything human-readable must go to stderr (it is streamed
  into the ACE UI). A non-zero exit is recorded as `command_failed` with the command's stderr. Resource `source` paths
  are resolved relative to the cwd of `sase gate create`, so a gate directory with relative `source` entries is
  portable.

## 2. Scope

**In scope**

- One new OpenCode global config file (chezmoi-managed).
- Four explicit GLM-5.2 model aliases plus a Models-panel bucket (chezmoi-managed, host-scoped to athena).
- A reusable custom gate directory (chezmoi-managed) containing the gate request and its four command scripts.
- Creating that gate, waiting for the user's decision, and verifying the result.

**Out of scope (do not do these)**

- Changing `@default`, `@smart`, `@cheap`, `@smartest`, the coder/epic role aliases, or any built-in role pool.
- The five-task GLM pilot described in the research (that is follow-up work after this lands).
- Buying or switching to a GLM Coding Plan subscription on the user's behalf.
- Building a Claude-Code-backed `glm` provider (the research's explicit contingency, not a prerequisite).
- Any change to the `sase` repo itself. This tale only touches the chezmoi repo and machine state, so `just check` does
  not apply — but run it anyway if you end up editing anything under the sase checkout.

## 3. Deliverables

All four files are **new** except the aliases, which are an edit. Open the chezmoi repo with
`sase repo open chezmoi -r "<reason>"` and use the path it prints as the only path for these edits.

| Chezmoi source path                                                                                   | Applied path                                                                         |
| ----------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| `home/dot_config/opencode/opencode.json`                                                              | `~/.config/opencode/opencode.json`                                                   |
| `home/dot_config/sase/sase_athena.yml` (edit)                                                         | `~/.config/sase/sase_athena.yml`                                                     |
| `home/dot_config/sase/gates/opencode_glm52/request.json`                                              | `~/.config/sase/gates/opencode_glm52/request.json`                                   |
| `home/dot_config/sase/gates/opencode_glm52/commands/executable_{install,authenticate,verify,decline}` | `~/.config/sase/gates/opencode_glm52/commands/{install,authenticate,verify,decline}` |

## 4. Step 1 — OpenCode global config

Create `home/dot_config/opencode/opencode.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "instructions": ["~/AGENTS.md"]
}
```

Rationale: OpenCode only discovers `AGENTS.md` between cwd and the worktree root, so the home-project SASE memory at
`~/AGENTS.md` — which Claude Code loads automatically — would otherwise be invisible to OpenCode agents. Referencing the
file by path (instead of copying it) keeps it in sync with `sase memory init`. This is the whole file; do **not** add
`autoupdate`, provider blocks, or an `apiKey` here. (Auto-update only runs from OpenCode's TUI worker, never from
`opencode run`, which is all SASE invokes; and the API key belongs in the credential store, not in a dotfiles repo.)

## 5. Step 2 — Explicit GLM-5.2 model aliases

Edit `home/dot_config/sase/sase_athena.yml`. It already contains an `llm_provider:` block (`provider: codex`); SASE
deep-merges the host overlay onto the base `sase.yml`, so adding keys here does **not** clobber
`llm_provider.default_effort` or the base `model_aliases.custom` entries. Extend the existing block so it reads:

```yaml
llm_provider:
  provider: codex
  model_aliases:
    custom:
      # GLM-5.2 pilot aliases. Effort MUST stay pinned to @high or @max: OpenCode
      # only generates `high` and `max` variants for GLM-5.2 and rejects any other
      # --variant, and the base config's default_effort is xhigh.
      glm52_high:
        model: opencode/zai/glm-5.2@high
        description: "GLM-5.2 via OpenCode on the metered Z.AI API (high effort)."
        bucket: glm_pilot
      glm52_max:
        model: opencode/zai/glm-5.2@max
        description: "GLM-5.2 via OpenCode on the metered Z.AI API (max effort)."
        bucket: glm_pilot
      glm52cp_high:
        model: opencode/zai-coding-plan/glm-5.2@high
        description: "GLM-5.2 via OpenCode on the Z.AI Coding Plan subscription (high effort)."
        bucket: glm_pilot
      glm52cp_max:
        model: opencode/zai-coding-plan/glm-5.2@max
        description: "GLM-5.2 via OpenCode on the Z.AI Coding Plan subscription (max effort)."
        bucket: glm_pilot
    buckets:
      glm_pilot:
        description: "GLM-5.2 pilot models routed through OpenCode. Not part of any built-in role pool."
```

Two design points, both from the research:

- Host-scoped (`sase_athena.yml`, applied only on athena via `.chezmoiignore`) because OpenCode and the Z.AI credential
  are being installed only here. Promoting these to the shared `sase.yml` is a deliberate later decision.
- The metered (`zai`) and Coding Plan (`zai-coding-plan`) routes get **separate alias names** rather than one alias
  whose target gets rewritten. The two routes are different endpoints and different billing; naming them apart is what
  prevents an accidental cross-route launch.

## 6. Step 3 — The custom gate directory

Create `home/dot_config/sase/gates/opencode_glm52/`. Every `source` in the request is **relative**, so the gate can be
created from the chezmoi checkout now and re-created from `~/.config/sase/gates/opencode_glm52/` (or on another machine)
later — always by `cd`-ing into the directory first.

### 6.1 `commands/executable_install`

```sh
#!/bin/sh
# Install or upgrade the OpenCode CLI globally with npm.
# stdin: gate input JSON (ignored). stdout: exactly one JSON object. Logs -> stderr.
set -eu

log() { printf '%s\n' "$*" >&2; }
json_safe() { printf '%s' "$1" | tr -cd 'A-Za-z0-9._/+ -'; }

cat >/dev/null 2>&1 || true

find_npm() {
  if command -v npm >/dev/null 2>&1; then
    command -v npm
    return 0
  fi
  found=""
  for candidate in \
    "${NVM_DIR:-$HOME/.config/nvm}/versions/node"/*/bin/npm \
    "$HOME/.config/nvm/versions/node"/*/bin/npm \
    /usr/local/bin/npm \
    /opt/homebrew/bin/npm \
    /usr/bin/npm
  do
    [ -x "$candidate" ] && found="$candidate"
  done
  [ -n "$found" ] || return 1
  printf '%s\n' "$found"
}

if ! npm_bin="$(find_npm)"; then
  log "ERROR: npm was not found on PATH or in any known nvm/homebrew location."
  log "Install Node.js/npm (or start the gate from a shell where 'npm' resolves) and try again."
  exit 1
fi
log "Using npm: $npm_bin"

log "Running: $npm_bin install -g opencode-ai"
"$npm_bin" install -g opencode-ai --no-fund --no-audit >&2

opencode_bin=""
if prefix="$("$npm_bin" prefix -g 2>/dev/null)" && [ -x "$prefix/bin/opencode" ]; then
  opencode_bin="$prefix/bin/opencode"
elif command -v opencode >/dev/null 2>&1; then
  opencode_bin="$(command -v opencode)"
fi

if [ -z "$opencode_bin" ]; then
  log "ERROR: 'opencode' was not found after installation."
  exit 1
fi

version="$("$opencode_bin" --version 2>/dev/null | head -n 1 | tr -d '\r')"
[ -n "$version" ] || version="unknown"
log "Installed opencode $version at $opencode_bin"

printf '{"status":"installed","version":"%s","path":"%s"}\n' \
  "$(json_safe "$version")" "$(json_safe "$opencode_bin")"
```

### 6.2 `commands/executable_authenticate`

Invoked as `commands/authenticate <provider-id> <pass-entry>`. It never prints, logs, or passes the key through argv.

```sh
#!/bin/sh
# Store a Z.AI API key in OpenCode's credential store, exactly as
# `opencode auth login` would: ~/.local/share/opencode/auth.json, mode 0600.
# usage: authenticate <zai|zai-coding-plan> <pass-entry-name>
# stdout: exactly one JSON object. Logs -> stderr. The key is NEVER printed or
# passed through argv (it reaches jq through the environment only).
set -eu

log() { printf '%s\n' "$*" >&2; }

cat >/dev/null 2>&1 || true

provider="${1:-}"
pass_entry="${2:-}"

case "$provider" in
  zai|zai-coding-plan) ;;
  *) log "ERROR: provider must be 'zai' or 'zai-coding-plan' (got '${provider}')."; exit 1 ;;
esac

command -v jq >/dev/null 2>&1 || { log "ERROR: jq is required but was not found."; exit 1; }

store="${PASSWORD_STORE_DIR:-$HOME/.password-store}"
key=""
key_source=""

read_pass() {
  entry="$1"
  [ -n "$entry" ] || return 1
  [ -f "$store/$entry.gpg" ] || return 1
  command -v pass >/dev/null 2>&1 || return 1
  if command -v timeout >/dev/null 2>&1; then
    timeout 120 pass show "$entry" 2>/dev/null | head -n 1
  else
    pass show "$entry" 2>/dev/null | head -n 1
  fi
}

for entry in "$pass_entry" "zai_sase_api_key"; do
  [ -n "$key" ] && break
  candidate="$(read_pass "$entry" || true)"
  if [ -n "$candidate" ]; then
    key="$candidate"
    key_source="pass"
    log "Using key from pass entry: $entry"
  fi
done

if [ -z "$key" ] && [ -n "${ZHIPU_API_KEY:-}" ]; then
  key="$ZHIPU_API_KEY"
  key_source="env"
  log "Using key from \$ZHIPU_API_KEY"
fi

if [ -z "$key" ]; then
  log "ERROR: no Z.AI API key available."
  log "Store one first:  pass insert ${pass_entry:-zai_sase_api_key}"
  log "(or export ZHIPU_API_KEY before starting the gate)."
  log "If pass timed out, unlock gpg-agent once with: pass show ${pass_entry:-zai_sase_api_key} >/dev/null"
  exit 1
fi

data_dir="${XDG_DATA_HOME:-$HOME/.local/share}/opencode"
auth_file="$data_dir/auth.json"
mkdir -p "$data_dir"
chmod 700 "$data_dir"

umask 077
tmp="$auth_file.tmp.$$"
trap 'rm -f "$tmp"' EXIT

existing='{}'
if [ -s "$auth_file" ]; then
  existing="$(cat "$auth_file")"
fi

printf '%s' "$existing" \
  | ZAI_KEY="$key" jq --arg provider "$provider" \
      '.[$provider] = {"type": "api", "key": env.ZAI_KEY}' > "$tmp"
chmod 600 "$tmp"
mv "$tmp" "$auth_file"
trap - EXIT

jq -e --arg p "$provider" \
  '.[$p].type == "api" and ((.[$p].key // "") | length) > 0' "$auth_file" >/dev/null \
  || { log "ERROR: credential was not stored correctly."; exit 1; }

log "Stored '$provider' credential in $auth_file (mode 0600)."

printf '{"status":"authenticated","provider":"%s","credential_path":"%s","key_source":"%s"}\n' \
  "$provider" "$auth_file" "$key_source"
```

### 6.3 `commands/executable_verify`

Invoked as `commands/verify <provider-id>`. Two probes: a marker reply (auth + billing route + model + variant) and a
tool-backed file read (tool calling actually works), both in a throwaway directory.

```sh
#!/bin/sh
# Smoke-test GLM-5.2 through OpenCode: one marker reply and one tool-backed read.
# usage: verify <zai|zai-coding-plan>
# stdout: exactly one JSON object. All OpenCode output -> stderr.
set -eu

log() { printf '%s\n' "$*" >&2; }

cat >/dev/null 2>&1 || true

provider="${1:-}"
case "$provider" in
  zai|zai-coding-plan) ;;
  *) log "ERROR: provider must be 'zai' or 'zai-coding-plan' (got '${provider}')."; exit 1 ;;
esac
model="$provider/glm-5.2"

oc=""
if command -v opencode >/dev/null 2>&1; then
  oc="$(command -v opencode)"
elif command -v npm >/dev/null 2>&1 && prefix="$(npm prefix -g 2>/dev/null)" && [ -x "$prefix/bin/opencode" ]; then
  oc="$prefix/bin/opencode"
fi
[ -n "$oc" ] || { log "ERROR: 'opencode' not found; run the install step first."; exit 1; }

work="$(mktemp -d)"
trap 'rm -rf "$work"' EXIT

marker="GLM52_OK"
log "Probe 1/2: text reply from $model (--variant high)"
if ! "$oc" run --model "$model" --variant high --dangerously-skip-permissions \
      --dir "$work" "Reply with exactly: $marker" > "$work/probe1.txt" 2>&1; then
  cat "$work/probe1.txt" >&2
  log "ERROR: opencode run failed for $model."
  exit 1
fi
cat "$work/probe1.txt" >&2
grep -q "$marker" "$work/probe1.txt" || { log "ERROR: marker '$marker' not found in the reply."; exit 1; }

token="glm52probe$$"
printf '%s\n' "$token" > "$work/GLM_PROBE.txt"
log "Probe 2/2: tool-backed file read"
if ! "$oc" run --model "$model" --variant high --dangerously-skip-permissions --dir "$work" \
      "Read the file GLM_PROBE.txt in this directory and reply with exactly its contents." \
      > "$work/probe2.txt" 2>&1; then
  cat "$work/probe2.txt" >&2
  log "ERROR: tool-backed probe failed."
  exit 1
fi
cat "$work/probe2.txt" >&2
grep -q "$token" "$work/probe2.txt" || { log "ERROR: file contents were not echoed back; tool calling did not work."; exit 1; }

log "Both probes passed for $model."
printf '{"status":"verified","provider":"%s","model":"%s","variant":"high","marker":true,"tool_read":true}\n' \
  "$provider" "$model"
```

### 6.4 `commands/executable_decline`

```sh
#!/bin/sh
# No-op rejection path: change nothing, record the decision.
set -eu
cat >/dev/null 2>&1 || true
printf '{"status":"declined"}\n'
```

### 6.5 `request.json`

Two complete, mutually exclusive install paths (metered API vs Coding Plan) plus a rejection branch. The metered branch
is primary, per the research's recommendation to pilot on metered billing before subscribing. `producer.agent` should be
the implementing agent's own name.

```json
{
  "schema_version": 3,
  "kind": "custom",
  "producer": { "agent": "REPLACE_WITH_AGENT_NAME" },
  "payload": {
    "intent": "Install the OpenCode CLI and authenticate a Z.AI credential so SASE can run GLM-5.2",
    "runtime": "opencode",
    "package": "opencode-ai",
    "models": ["zai/glm-5.2", "zai-coding-plan/glm-5.2"],
    "credential_path": "~/.local/share/opencode/auth.json"
  },
  "presentation": {
    "icon": "🔌",
    "sender": "opencode-glm52-setup",
    "notes": [
      "Install OpenCode and authenticate GLM-5.2 for SASE.",
      "Pick ONE billing route: the metered Z.AI API (recommended for the pilot) or the Z.AI Coding Plan subscription.",
      "Prerequisite: the API key must already be in pass — `pass insert zai_sase_api_key` (metered) or `pass insert zai_coding_plan_sase_api_key` (Coding Plan); $ZHIPU_API_KEY is used as a fallback. No agent ever sees the key.",
      "Install runs `npm install -g opencode-ai`. Authenticate writes only the chosen provider's entry into ~/.local/share/opencode/auth.json (mode 0600). Verify makes two small paid GLM-5.2 calls in a temp directory.",
      "If gpg-agent is locked, run `pass show zai_sase_api_key >/dev/null` in a terminal first so the key read cannot block."
    ],
    "tags": ["opencode", "glm-5.2", "provider-setup"]
  },
  "query": "(install_api AND auth_api AND verify_api) OR (install_plan AND auth_plan AND verify_plan) OR decline",
  "primary_branch": ["install_api", "auth_api", "verify_api"],
  "options": [
    {
      "id": "install_api",
      "label": "Install OpenCode (npm -g opencode-ai)",
      "icon": "📦",
      "default_selected": true,
      "feedback": "optional",
      "command": { "argv": ["commands/install"] },
      "input_schema": { "type": "object" },
      "result_schema": {
        "type": "object",
        "required": ["status", "version"],
        "properties": {
          "status": { "const": "installed" },
          "version": { "type": "string" },
          "path": { "type": "string" }
        }
      }
    },
    {
      "id": "auth_api",
      "label": "Authenticate metered Z.AI API",
      "icon": "💳",
      "default_selected": true,
      "feedback": "optional",
      "command": { "argv": ["commands/authenticate", "zai", "zai_sase_api_key"] },
      "input_schema": { "type": "object" },
      "result_schema": {
        "type": "object",
        "required": ["status", "provider", "key_source"],
        "properties": {
          "status": { "const": "authenticated" },
          "provider": { "const": "zai" },
          "credential_path": { "type": "string" },
          "key_source": { "enum": ["pass", "env"] }
        }
      }
    },
    {
      "id": "verify_api",
      "label": "Verify GLM-5.2 on the metered API",
      "icon": "🧪",
      "default_selected": true,
      "feedback": "optional",
      "command": { "argv": ["commands/verify", "zai"] },
      "input_schema": { "type": "object" },
      "result_schema": {
        "type": "object",
        "required": ["status", "marker", "tool_read"],
        "properties": {
          "status": { "const": "verified" },
          "provider": { "const": "zai" },
          "model": { "type": "string" },
          "variant": { "type": "string" },
          "marker": { "const": true },
          "tool_read": { "const": true }
        }
      }
    },
    {
      "id": "install_plan",
      "label": "Install OpenCode (npm -g opencode-ai)",
      "icon": "📦",
      "default_selected": true,
      "feedback": "optional",
      "command": { "argv": ["commands/install"] },
      "input_schema": { "type": "object" },
      "result_schema": {
        "type": "object",
        "required": ["status", "version"],
        "properties": {
          "status": { "const": "installed" },
          "version": { "type": "string" },
          "path": { "type": "string" }
        }
      }
    },
    {
      "id": "auth_plan",
      "label": "Authenticate Z.AI Coding Plan",
      "icon": "🎟️",
      "default_selected": true,
      "feedback": "optional",
      "command": { "argv": ["commands/authenticate", "zai-coding-plan", "zai_coding_plan_sase_api_key"] },
      "input_schema": { "type": "object" },
      "result_schema": {
        "type": "object",
        "required": ["status", "provider", "key_source"],
        "properties": {
          "status": { "const": "authenticated" },
          "provider": { "const": "zai-coding-plan" },
          "credential_path": { "type": "string" },
          "key_source": { "enum": ["pass", "env"] }
        }
      }
    },
    {
      "id": "verify_plan",
      "label": "Verify GLM-5.2 on the Coding Plan",
      "icon": "🧪",
      "default_selected": true,
      "feedback": "optional",
      "command": { "argv": ["commands/verify", "zai-coding-plan"] },
      "input_schema": { "type": "object" },
      "result_schema": {
        "type": "object",
        "required": ["status", "marker", "tool_read"],
        "properties": {
          "status": { "const": "verified" },
          "provider": { "const": "zai-coding-plan" },
          "model": { "type": "string" },
          "variant": { "type": "string" },
          "marker": { "const": true },
          "tool_read": { "const": true }
        }
      }
    },
    {
      "id": "decline",
      "label": "Do not install or authenticate",
      "icon": "✋",
      "feedback": "required",
      "command": { "argv": ["commands/decline"] },
      "input_schema": { "type": "object" },
      "result_schema": {
        "type": "object",
        "required": ["status"],
        "properties": { "status": { "const": "declined" } }
      }
    }
  ],
  "groups": [
    {
      "options": ["install_api", "auth_api", "verify_api"],
      "label": "Install + metered Z.AI API",
      "icon": "💳"
    },
    {
      "options": ["install_plan", "auth_plan", "verify_plan"],
      "label": "Install + Z.AI Coding Plan",
      "icon": "🎟️"
    }
  ],
  "resources": [
    { "path": "commands/install", "role": "command", "source": "commands/install" },
    { "path": "commands/authenticate", "role": "command", "source": "commands/authenticate" },
    { "path": "commands/verify", "role": "command", "source": "commands/verify" },
    { "path": "commands/decline", "role": "command", "source": "commands/decline" }
  ],
  "gate_timeout_seconds": 86400,
  "auto": false
}
```

## 7. Step 4 — Static checks before creating the gate

From the gate directory in the chezmoi checkout:

```bash
jq empty request.json
for f in commands/*; do sh -n "$f" || echo "SYNTAX ERROR: $f"; done
sh commands/decline </dev/null | jq -e '.status == "declined"'
```

Do **not** run `install`, `authenticate`, or `verify` by hand — those are the gate's owned commands and must run through
the gate. Also confirm the key prerequisite without decrypting anything:

```bash
ls ~/.password-store/zai_sase_api_key.gpg ~/.password-store/zai_coding_plan_sase_api_key.gpg 2>&1
```

If neither exists, still create the gate (the notes already tell the user what to store), and say so plainly in your
final report — the `authenticate` command will fail with an actionable message rather than doing anything wrong.

## 8. Step 5 — Create the gate and wait

```bash
cd <chezmoi-checkout>/home/dot_config/sase/gates/opencode_glm52
sase gate create < request.json > /tmp/opencode_glm52_gate.json
cat /tmp/opencode_glm52_gate.json
sase gate wait --id "$(jq -r .request_id /tmp/opencode_glm52_gate.json)" --kind custom --json
```

Relative `source` paths make the `cd` mandatory. Handle the terminal result honestly:

- `answered` with `["install_api","auth_api","verify_api"]` (or the plan branch, or any non-empty subset of one branch)
  → continue to Step 6 for whatever ran.
- `answered` with `["decline"]` → nothing was installed. Report that, keep the config files (they are inert until a
  credential exists), and stop.
- `cancelled` or `timeout` → terminal. Do not install or authenticate through any other path.

If a command fails, the wait result and the bundle's recorded execution error carry the command's stderr; report it
verbatim rather than retrying blindly.

## 9. Step 6 — Post-gate verification

Only for the steps that actually ran:

```bash
sase agent-cli list -v                 # OpenCode: installed, with a version and a path
opencode --version
opencode auth list                     # lists the stored Z.AI credential (no key is printed)
opencode debug config | jq '.instructions'   # ["~/AGENTS.md"] once chezmoi has applied
stat -c '%a %n' ~/.local/share/opencode/auth.json   # expect 600
```

And on the SASE side, after the chezmoi changes are applied (see Step 7):

```bash
sase config show | grep -A 4 'glm52'
sase doctor -C config.model_aliases
.venv/bin/python -c 'from sase.llm_provider.model_alias_resolution import resolve_model_alias_with_effort as r; print(r("glm52_high")); print(r("glm52_max")); print(r("glm52cp_high")); print(r("glm52cp_max"))'
```

Expect `target='opencode/zai/glm-5.2'` with `effort='high'`/`'max'` and the `zai-coding-plan` equivalents, all
`valid=True`. (Run `just install` first if the workspace venv is stale.)

Do **not** launch a real SASE agent on GLM as part of this tale; that is the pilot, and it is follow-up work.

## 10. Step 7 — Commit and apply

The chezmoi repo's own `AGENTS.md` requires running `chezmoi update -a --force` after any commit to it, which is what
applies `opencode.json`, the aliases, and the durable gate directory into `~`. Let the normal SASE commit finalizer
commit and push the chezmoi changes, then run:

```bash
chezmoi update -a --force
```

Then re-run the SASE-side verification from Step 6 against the applied files, and confirm the durable gate directory
landed:

```bash
ls ~/.config/sase/gates/opencode_glm52 ~/.config/sase/gates/opencode_glm52/commands
```

Nothing in Step 5 depends on this apply — the gate is created from the checkout — so a failure here does not invalidate
an installed and verified OpenCode.

## 11. Risks and mitigations

- **`--variant xhigh` runtime failure.** Any GLM launch that does not pin `high`/`max` will fail. Mitigated by pinned
  aliases; call it out in the final report so the user knows not to add `%effort:` to a GLM launch.
- **`npm` not on the gate executor's PATH.** The install command searches nvm and Homebrew locations and fails with an
  explicit message instead of silently doing nothing.
- **gpg-agent locked / pinentry blocking.** `pass` reads are wrapped in `timeout 120`, and both the gate notes and the
  failure message tell the user how to prime the agent cache.
- **Writing `auth.json` directly instead of using `opencode auth login`.** Unavoidable (the login prompt needs a TTY).
  Mitigated by writing exactly the documented `{"type":"api","key":...}` shape at mode `0600`, merging with `jq` so
  other providers survive, and then verifying with the real CLI (`opencode auth list`) plus two live probes. If OpenCode
  ever rejects the entry, fall back to having the user run `opencode auth login` interactively once.
- **Wrong billing route.** Separate provider ids, separate alias names, separate pass entries, and separate gate
  branches; nothing rewrites one into the other.
- **Verification costs money.** Two very small metered GLM-5.2 calls, disclosed in the gate notes.

## 12. Rollback

```bash
npm uninstall -g opencode-ai
jq 'del(.zai, ."zai-coding-plan")' ~/.local/share/opencode/auth.json  # then write it back, mode 0600
```

plus reverting the chezmoi commit and re-running `chezmoi update -a --force`. The GLM aliases are inert on their own,
and no built-in role alias or pool is touched, so nothing else in SASE changes behavior.

## 13. Follow-up (not part of this tale)

- The research's five-task GLM pilot (architecture audit, bounded bug fix, cross-file refactor, long-context task, two
  concurrent agents), paired against the current preferred model.
- Only after that pilot: consider promoting the aliases to the shared `sase.yml`, adding an `m_glm`-style xprompt
  shortcut, or letting GLM into a narrow role alias. Observe a full quota cycle before considering `@cheap` or broader
  defaults.
