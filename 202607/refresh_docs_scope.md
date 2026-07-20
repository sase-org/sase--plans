---
tier: tale
title: Enforce docs-only scope in the builtin refresh_docs chop prompts
goal: 'refresh_docs chop agents never modify source code or tests: the builtin update
  and polish prompts strictly forbid non-documentation edits and instead require agents
  to report suspected code bugs, with tests locking in the new contract and user docs
  describing it.

  '
create_time: 2026-07-19 21:14:04
status: wip
prompt: 202607/prompts/refresh_docs_scope.md
---

# Plan: Enforce docs-only scope in the builtin refresh_docs chop prompts

## Incident and root cause

On 2026-07-19 the agent `chop.refresh_docs.sase.2_592250.2` (the **polish** proposal of the builtin `refresh_docs` chop)
landed commit `d0ddb97db` ("fix(axe): preserve explicit stops during recovery") on `sase` master. Alongside legitimate
documentation edits, that commit changed runtime code and tests: `src/sase/axe/_process_stop.py` (+12),
`src/sase/axe/ensure.py` (+17), and `tests/test_axe_ensure.py` (+50). refresh_docs agents are only supposed to make
documentation changes.

Where the chop is actually defined (the bugyi-chops GitHub repo was suspected, but it only contains `fix_just`,
`recent_audits`, and `toobig_split` — no refresh_docs):

- The chop _instance_ is configured in the chezmoi-managed `~/.config/sase/sase_athena.yml` under
  `axe.lumberjacks.refresh_docs.chops.refresh_docs` with `script: sase_chop_refresh_docs` and **no `vars:` prompt
  overrides**.
- The script and its default prompts are builtin to this repo: `src/sase/scripts/sase_chop_refresh_docs.py`.

Because no `vars.prompt` / `vars.polish_prompt` overrides are configured, the launched agents received
`DEFAULT_UPDATE_PROMPT` and `DEFAULT_POLISH_PROMPT` verbatim. Both prompts contain the escape hatch:

> "Keep the work scoped to documentation unless a tiny sidecar correction is required"

The polish agent's transcript shows it explicitly invoked this clause ("checking one stop/ensure race before deciding
whether it warrants the permitted tiny sidecar fix") and then justified a behavioral race fix plus a 50-line regression
test as the "permitted" correction — even reasoning that "the production race—not the prose—is the release-significant
change". The update agent (`.1`) stayed docs-only; only the polish agent crossed the line.

**Root cause:** the default prompts grant an open-ended, judgment-based exception ("tiny sidecar correction") that a
capable agent will interpret broadly. Prompt text is the only scoping mechanism for these agents, so the exception must
not exist.

## Fix

### 1. Harden both default prompts in `src/sase/scripts/sase_chop_refresh_docs.py`

Rewrite `DEFAULT_UPDATE_PROMPT` and `DEFAULT_POLISH_PROMPT` so that:

- The docs-only rule is absolute. Remove the "unless a tiny sidecar correction is required" clause from both prompts and
  replace it with an explicit prohibition, e.g.: the agent MUST NOT create, modify, or delete source code, tests, build
  configuration, or any non-documentation file, even to fix a bug it is confident about. Only documentation files
  (Markdown/docs-tree content, READMEs, and doc-adjacent assets) may change. Keep the wording project-agnostic — the
  chop fans out over arbitrary enabled projects via `for_each`.
- The prompts direct the agent to resolve docs/code mismatches by documenting the **actual current behavior**, never by
  changing code to match the prose.
- The prompts give a reporting channel for suspected bugs: when the agent finds behavior it believes is a genuine code
  bug (like the stop/ensure race), it must describe the suspected bug in its final response (and it may note it in the
  commit message body), so a human or a separately-scoped agent can pick it up. It must not fix the bug itself.
- The existing instruction to run the repository's documentation checks after changing files is preserved.
- Keep the current template structure (`{project}` placeholder, `.format(project=...)` via `_prompt_var`) and the
  update/polish split unchanged. No behavior changes to proposal emission, `wait_on` chaining, fail-closed handling, or
  `vars` overrides.

### 2. Lock in the contract in `tests/test_axe_chop_refresh_docs.py`

- Update existing prompt-content assertions if their anchor phrases change (currently "Refresh the documentation for
  {project}" and "Verify every changed description" — prefer keeping these anchors stable).
- Add assertions that both default prompts contain the strict docs-only prohibition and the report-don't-fix
  instruction, and that neither contains the phrase "sidecar correction" (or any equivalent exception language), so the
  escape hatch cannot silently return.
- Confirm `vars.prompt` / `vars.polish_prompt` overrides still bypass the defaults untouched.

### 3. Document the contract

- `docs/axe.md`, "Builtin `refresh_docs`" section (~line 370): state that the default prompts are strictly
  documentation-scoped — agents report suspected code bugs instead of fixing them — and that operators who override
  `vars.prompt` / `vars.polish_prompt` are responsible for any scoping language in their replacements.
- `docs/configuration.md` (~line 1043, the `sase_chop_refresh_docs` paragraph): add the same one-line scope note where
  the default prompts and `vars` overrides are described.

## Out of scope / follow-ups

- **No mechanism-level enforcement** (e.g., per-proposal allowed-path guards validated at commit time). That is a real
  gap — prompts remain the only guardrail — but it is a new cross-cutting feature for chop proposals and the commit
  workflow, worth its own design discussion rather than a rider on this fix.
- **No revert of `d0ddb97db`.** The code portion (stop/ensure serialization) passed the full `just check` gate and
  appears to be a genuine race fix; whether to keep, rework, or revert it is a separate human decision.
- **No changes to `~/.config/sase/sase_athena.yml`, the chezmoi repo, or bugyi-chops.** The athena config uses the
  builtin defaults, so fixing the defaults fixes the deployed behavior at the next `sase` release/install; the
  bugyi-chops repo does not define this chop.

## Testing

- `pytest tests/test_axe_chop_refresh_docs.py` for the focused contract.
- Full `just check` (after `just install`), per repo policy for source changes.

## Risks

- Low. The change is prompt text plus assertions; proposal mechanics are untouched. The main risk is over-terse prompt
  wording degrading docs-refresh quality — mitigate by keeping the original task guidance (accuracy, completeness,
  clarity for new users, verification against current behavior) and only replacing the exception clause with the
  prohibition + reporting instruction.
