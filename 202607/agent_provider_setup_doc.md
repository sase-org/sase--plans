---
create_time: 2026-07-08 13:50:27
status: wip
prompt: 202607/prompts/agent_provider_setup_doc.md
tier: tale
---
# Plan: Add an "Agent Provider Setup" docs page (install + authenticate each supported CLI)

## Problem / Product context

SASE orchestrates an existing coding-agent CLI; it does not ship or replace that CLI's install and authentication flow.
Today the requirement "install and authenticate at least one provider CLI" is stated in several places (`INSTALL.md`,
`README.md`, `docs/getting_started.md`, blog posts, and `sase doctor` output) but **no single page tells a user how to
actually do it** — there are no per-provider install commands, auth commands, or links to each vendor's canonical docs.
A new user who runs `sase doctor` and sees "provider not ready" has to hunt across five different vendors to fix it.

Goal: add one focused docs page that, for every provider SASE currently supports, gives (1) the install command, (2) the
authentication command/flow, and (3) a link to that vendor's **canonical** install documentation — then link to this
page from the places that currently only state the requirement.

## Supported providers (authoritative)

SASE discovers providers via `sase_llm` entry points (`pyproject.toml`), and `sase doctor` prints per-provider setup
hints from the single source of truth `_PROVIDER_SETUP_HINTS` in `src/sase/doctor/checks_providers.py` (lines 19-45).
The new doc's install/auth commands MUST mirror that dict verbatim so the page and `sase doctor` never disagree. The
five supported providers are:

| Provider key | Tool name       | Install command (from `_PROVIDER_SETUP_HINTS`)                 | Auth (from `_PROVIDER_SETUP_HINTS`)               |
| ------------ | --------------- | -------------------------------------------------------------- | ------------------------------------------------- |
| `claude`     | Claude Code     | `npm install -g @anthropic-ai/claude-code`                     | run `claude` and complete the login flow          |
| `codex`      | Codex CLI       | `npm install -g @openai/codex`                                 | run `codex login`                                 |
| `opencode`   | OpenCode        | install from https://opencode.ai/docs                          | run `opencode auth login`                         |
| `qwen`       | Qwen Code       | `npm install -g @qwen-code/qwen-code`                          | run `qwen` and complete the login flow            |
| `agy`        | Antigravity CLI | `curl -fsSL https://antigravity.google/cli/install.sh \| bash` | run `agy` and complete the login/trust onboarding |

Notes that must be reflected in the doc so it stays accurate:

- **There is no separate "Gemini CLI" provider.** `GEMINI.md` exists only because Antigravity (`agy`) reads it for
  workspace context. The doc must NOT invent a Gemini CLI section.
- `claude`, `codex`, `qwen` are npm-distributed, so they need `node`/`npm` (already noted in `INSTALL.md` optional
  table). `opencode` and `agy` are installed via their own installers, not npm.
- Each provider also honors a `SASE_<PROVIDER>_PATH` override env var (e.g. `SASE_CLAUDE_PATH`) for a non-standard
  executable location; the doc should mention this once as an escape hatch, deriving names from
  `provider_path_env_var()` in `src/sase/llm_provider/registry.py`.
- API-key env vars are an alternative to interactive login for some providers (e.g. `ANTHROPIC_API_KEY` /
  `ANTHROPIC_AUTH_TOKEN` for claude, `OPENAI_API_KEY` for codex, `GEMINI_API_KEY`/`GOOGLE_API_KEY` for agy). Keep these
  brief and defer the full list of accepted env vars to each vendor's canonical docs to avoid drift; the exact
  per-provider env var lists live in each `src/sase/llm_provider/<name>.py` plugin if we want to spot-check.

## Canonical vendor documentation to link (verify each resolves at implementation time)

The doc must "point back to the canonical documentation for installing the given agent CLI." Candidate canonical URLs
below — the implementer must confirm each with a quick `WebFetch`/`WebSearch` and correct any that have moved, since
these are vendor-owned and change over time:

- Claude Code: `https://docs.claude.com/en/docs/claude-code` (Anthropic Claude Code docs)
- Codex CLI: `https://developers.openai.com/codex/cli` (OpenAI Codex CLI docs; fall back to
  `https://github.com/openai/codex`)
- OpenCode: `https://opencode.ai/docs` (already the canonical URL baked into `_PROVIDER_SETUP_HINTS`)
- Qwen Code: `https://github.com/QwenLM/qwen-code` (Qwen Code repo / docs)
- Antigravity CLI: `https://antigravity.google` (Google Antigravity; the install script is hosted there)

## Deliverables

### 1. New docs page: `docs/agent_providers.md`

- snake_case filename to match sibling docs (`getting_started.md`, `change_spec.md`). Title H1:
  `# Installing & Authenticating Agent Providers`. No YAML front-matter is required (most reference docs omit it), but a
  one-line intro paragraph is.
- Structure:
  1. Short intro: SASE orchestrates an existing provider CLI; you need **at least one** installed and authenticated;
     `sase doctor` (specifically `sase doctor -C llm.auth -v`) is the authoritative readiness check and prints the same
     install/auth hints this page documents.
  2. One `##` section per provider (Claude Code, Codex CLI, OpenCode, Qwen Code, Antigravity CLI), each containing:
     - a one-liner on what it is,
     - `### Install` with a fenced `bash` block using the exact command from the table above,
     - `### Authenticate` with the exact auth command/flow,
     - a "Canonical docs:" link to the vendor page,
     - optional note on the API-key env var alternative where applicable.
  3. A short "Verify" section: run `sase doctor` and expect the provider to report ready; mention `SASE_<PROVIDER>_PATH`
     override for custom install paths.
- 120-column prose wrapping (prettier `--prose-wrap=always --print-width=120` will enforce this via `just fmt-md`).

### 2. Register the page in `mkdocs.yml` nav

`strict: true` (mkdocs.yml:9) fails the build if a page is unlinked or a link is broken, so this step is mandatory. Add
under the **Getting Started** group (best fit for an onboarding how-to) right after Initialization:

```yaml
- Getting Started:
    - Your First 15 Minutes: getting_started.md
    - Initialization: init.md
    - Agent Providers: agent_providers.md
```

(Alternative placement under Integrations next to `LLM Providers: llms.md` is acceptable if we prefer to keep it beside
the provider-integration reference; Getting Started is recommended because this is setup-oriented, not integration
internals.)

### 3. Link to the new page from installation docs

- **`INSTALL.md`** (primary): the `### Required` table row "One coding-agent CLI" (line 126) states the requirement but
  gives no how-to. Add a sentence immediately after the `### Required` table (after line 127) such as: "For per-provider
  install and authentication commands, see [Installing & Authenticating Agent Providers](docs/agent_providers.md)." Use
  a root-relative `docs/agent_providers.md` link (matches how INSTALL.md/README.md reference docs pages).
- **`README.md`**: (a) add a link near the Prerequisites/`sase doctor` guidance (around lines 22-24 / 51-52) pointing to
  the new page; (b) add the page to the "local docs" link list (README lines ~221-253) following the existing
  `https://sase.sh/... ([local](docs/....md))` pattern used for every other docs page.
- **`docs/getting_started.md`**: Step 2 "Check Provider Readiness" currently points to `llms.md`; add/point a link to
  `agent_providers.md` for the actual install/auth steps (relative link `agent_providers.md`).
- **`docs/llms.md`**: this is the provider _integration_ reference. Add a short cross-link near its top (or in the per-
  provider auth subsections) to `agent_providers.md` for install/auth how-to, keeping the two docs' scopes distinct
  (llms.md = how SASE integrates the provider; agent_providers.md = how you install/authenticate the CLI).

## Constraints / guardrails

- Do NOT modify `AGENTS.md`, `memory/*.md`, or the generated provider shims (`CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`,
  `QWEN.md`). `INSTALL.md`, `README.md`, and `docs/*.md` are all fair game and are NOT protected.
- Keep install/auth commands byte-identical to `_PROVIDER_SETUP_HINTS`. If we ever want the doc to be generated from
  that dict, note it as a future improvement but do NOT build codegen in this change — a hand-written page is in scope.
- No new provider sections beyond the five entry points; explicitly no "Gemini CLI" section.

## Verification

- `just fmt-md` (prettier) then `just fmt-md-check` to confirm 120-col wrapping passes.
- Build the docs to satisfy `strict: true`: `mkdocs build --strict` (or the repo's documented docs-build just recipe if
  one exists) — confirms the new nav entry and all new cross-links resolve.
- `just check` after edits (per repo policy for non-exempt file changes; docs/ markdown is not in the sdd/research
  exception). Run `just install` first if the workspace venv is stale. Note: the full pytest phase can be killed by the
  sandbox (env SIGTERM / exit 144) — treat static/lint/docs-build gates as the meaningful signal and don't misread a
  sandbox kill as a real failure.
- Manual spot check: each canonical vendor URL resolves; each install/auth command matches `_PROVIDER_SETUP_HINTS`.

## Out of scope

- Auto-generating the page from `_PROVIDER_SETUP_HINTS` (possible future follow-up).
- Documenting per-provider model mapping / SASE integration internals (already covered by `docs/llms.md`).
- Any change to how `sase doctor` reports readiness.
