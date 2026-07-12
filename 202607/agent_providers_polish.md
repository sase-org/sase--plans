---
create_time: 2026-07-08 14:27:54
status: wip
prompt: 202607/prompts/agent_providers_polish.md
tier: tale
---
# Plan: Tighten `docs/agent_providers.md` for concision & consistency

## Problem / product context

`docs/agent_providers.md` is a new onboarding page telling users how to install and authenticate each supported
coding-agent CLI (Claude Code, Codex CLI, OpenCode, Qwen Code, Antigravity CLI). A review confirmed the page is
**factually accurate**: install/auth commands are byte-identical to `_PROVIDER_SETUP_HINTS`
(`src/sase/doctor/checks_providers.py`), the per-provider API-key env vars match each plugin's `llm_auth_evidence`, and
the Qwen "OAuth free tier ended 2026-04-15" and "Antigravity replaces the retired consumer Gemini CLI" claims are
consistent with `docs/llms.md` and `CHANGELOG.md`.

The remaining weakness is **length and stylistic consistency** — the page is ~15–20% longer than it needs to be, repeats
itself, leaks SASE-internal phrasing, and presents the same concept (authentication) two different ways across
providers. A good short page is harder to write than a long one; this change closes that gap without touching any fact
or command.

**Scope: copyediting only.** No facts, commands, providers, links, or nav entries change. This is not a rewrite — it is
a targeted tightening of an already-correct page.

## Problems to fix (each confirmed in the current page)

1. **Verbatim triple repetition.** The sentence "…is distributed via npm, so it needs `node` / `npm` on your `PATH`."
   appears three times, essentially word-for-word, under Claude Code, Codex CLI, and Qwen Code. This is pure padding —
   the npm prerequisite is one fact, not three.
2. **Meta / internal-leaking OpenCode Install prose.** The OpenCode `### Install` section explains itself in terms of
   SASE's own doctor hint ("not distributed via npm through SASE's hint", "The `sase doctor` install hint is literally
   `install from https://opencode.ai/docs`"). A user installing OpenCode does not care how SASE stores its hint string;
   they need the one actionable line: install it via a method on its docs page.
3. **Inconsistent Authenticate presentation.** Codex and OpenCode render authentication as a clean fenced `bash` block
   (`codex login`, `opencode auth login`). Claude and Qwen instead leave a bare, lowercase, period-less sentence
   fragment ("run `claude` and complete the login flow") dangling directly under the `### Authenticate` heading. The
   reader sees two different visual treatments for the same step.
4. **(Optional) Slightly heavy API-key notes.** Each provider's API-key alternative is a 2–3 line paragraph. These can
   be trimmed to a single tight sentence each while preserving the "defer the full/current list to the vendor's
   canonical docs" guidance that keeps the page drift-resistant.

## Design / approach

Edit `docs/agent_providers.md` in place. Keep the existing structure (intro → one `##` per provider with `### Install` /
`### Authenticate` → `## Verify`); only tighten wording.

### 1. Consolidate the npm prerequisite into the intro

Remove the repeated per-provider npm sentence from the Claude, Codex, and Qwen `### Install` sections. Add **one**
sentence to the intro that states the distribution split, e.g.:

> Claude Code, Codex CLI, and Qwen Code are distributed via `npm` (so they need `node` / `npm` on your `PATH`); OpenCode
> and the Antigravity CLI ship their own installers.

The per-provider `### Install` sections then contain just the fenced install command (unchanged, byte-identical to
`_PROVIDER_SETUP_HINTS`), with no repeated prerequisite prose.

### 2. Reduce the OpenCode Install section to one actionable line

Replace the meta paragraph with a single user-facing sentence pointing at the canonical docs' install options, without
referencing SASE's internal hint string. Something like:

> OpenCode isn't distributed on npm — install it with any method on its docs page (npm, Homebrew, or its install
> script).

Keep the `Canonical docs:` link. Do not invent version-specific install commands; defer specifics to the linked page.

### 3. Normalize the Authenticate presentation

Pick one consistent treatment for the "run the CLI, then finish the login flow" providers (Claude, Qwen, and
Antigravity, which uses "run `agy` and complete the login/trust onboarding"). Recommended: a fenced `bash` block
containing the bare command the user runs, followed by a one-line sentence describing the interactive step — mirroring
how Codex/OpenCode already look. For example, Claude becomes:

> ```bash
> claude
> ```
>
> Run this and complete the interactive login flow.

Apply the same shape to Qwen (`qwen`) and Antigravity (`agy` … login/trust onboarding). The underlying command and the
described flow must stay faithful to `_PROVIDER_SETUP_HINTS` (the hint text `run \`claude\` and complete the login
flow`is preserved in meaning; the command inside the block is exactly the CLI name the hint tells the user to run). Alternatively, if fenced blocks feel wrong for a non-single-command step, standardize instead on a proper capitalized sentence ("Run`claude`
and complete the login flow.") for all such providers — the requirement is that all four providers use the **same**
style, not two.

### 4. (Optional) Tighten API-key notes

Collapse each API-key paragraph to a single sentence: name the representative env var(s) already documented (unchanged),
then defer to the canonical docs for the current full list. Keep the substance (e.g. Claude's `ANTHROPIC_API_KEY` /
`ANTHROPIC_AUTH_TOKEN` / `CLAUDE_CODE_OAUTH_TOKEN`, Codex's `OPENAI_API_KEY`, Antigravity's `GEMINI_API_KEY` /
`GOOGLE_API_KEY`, and the Qwen 2026-04-15 free-tier note). Do not add or remove any env var names.

## Hard constraints (must hold)

- Install and authenticate **commands stay byte-identical** to `_PROVIDER_SETUP_HINTS` in
  `src/sase/doctor/checks_providers.py`. Rewording surrounding prose is fine; changing a command is not.
- Preserve every **canonical vendor documentation link** and the **`SASE_<PROVIDER>_PATH` override** escape hatch in the
  `## Verify` section (names: `SASE_CLAUDE_PATH`, `SASE_CODEX_PATH`, `SASE_OPENCODE_PATH`, `SASE_QWEN_PATH`,
  `SASE_AGY_PATH`), plus the closing cross-link to `llms.md`.
- **No factual changes** and **no new providers** — in particular, do not add a separate "Gemini CLI" section; keep the
  note that `GEMINI.md` exists only because Antigravity reads it.
- Do **not** modify protected files: `AGENTS.md`, `memory/*.md`, or the generated provider shims (`CLAUDE.md`,
  `GEMINI.md`, `OPENCODE.md`, `QWEN.md`). Only `docs/agent_providers.md` is edited by this change.
- Prose wraps at 120 columns (enforced by prettier via `just fmt-md`).

## Files touched

- `docs/agent_providers.md` — the only file edited.

## Verification

- `just fmt-md` then `just fmt-md-check` — confirm 120-column wrapping passes.
- `just docs-check` (`mkdocs build --strict`) — confirm the page still builds under `strict: true` with all links and
  the nav entry intact (structure and links are unchanged, so this should stay green).
- `just check` after edits (docs markdown is not in the `sdd/research` exception). Note the full pytest phase can be
  killed by the sandbox (SIGTERM / exit 144); treat the static / lint / docs-build gates as the meaningful signal and do
  not misread a sandbox kill as a real failure.
- Manual diff review: confirm every install/auth command still matches `_PROVIDER_SETUP_HINTS` verbatim, all canonical
  links and `SASE_<PROVIDER>_PATH` names remain, and no fact was altered — the diff should be pure wording/formatting
  reduction.

## Out of scope

- Auto-generating the page from `_PROVIDER_SETUP_HINTS` (possible future follow-up; still hand-maintained here).
- Any change to `sase doctor` output, provider plugins, or other docs pages (`INSTALL.md`, `README.md`,
  `getting_started.md`, `llms.md`, `mkdocs.yml`) — their cross-links to this page already exist and are correct.
- Adding a summary/overview table: acceptable as a concision aid if the implementer finds it clearly improves scanning,
  but not required; if added it must not duplicate or drift from `_PROVIDER_SETUP_HINTS`.
