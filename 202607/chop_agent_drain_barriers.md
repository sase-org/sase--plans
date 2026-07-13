---
create_time: 2026-07-13 07:42:13
status: done
prompt: 202607/prompts/chop_agent_drain_barriers.md
tier: tale
---
# Plan: Add `%w(runners=0)` Drain Barriers to Chop-Launched Agents

## Goal

Ensure every root user agent launched by an Athena axe chop carries the exact `%w(runners=0)` directive, so chop work
uses the `sase-5u` runner-slot admission system as a FIFO drain barrier: the agent starts only after earlier root agents
have stopped, and later barrier agents cannot jump ahead of it.

This is a prompt/configuration change, not a new admission-control feature. Preserve all existing chop cadences,
ChangeSpec guards, project/workspace selection, names, `%group:chop` tagging, auto/proposal behavior, conditional launch
logic, and multi-prompt dependency ordering.

## Launch-Surface Inventory and Scope

The implementation must cover every current Athena path that can create a root user agent from a chop, including both
the chop's initial workflow agent and any detached root agents that workflow later launches:

1. **Eight configured agent-backed chops in the chezmoi Athena overlay** (`home/dot_config/sase/sase_athena.yml`):
   - `sase_toobig_split`
   - `sase_recent_bug_audit`
   - `sase_recent_improvement_audit`
   - `sase_refresh_docs`
   - `sase_core_refresh_docs`
   - `sase_github_refresh_docs`
   - `sase_nvim_refresh_docs`
   - `sase_telegram_refresh_docs`
2. **Two configured executable chops that call `sase run -d` themselves** in chezmoi:
   - `home/bin/executable_sase_chop_sase_fix_just` via `FIX_JUST_PROMPT`
   - `home/bin/executable_sase_chop_gh_actions_fix` via `build_prompt()` (one possible fixer per configured repo)
3. **Detached root-agent prompts constructed by bundled SASE workflows**:
   - one split-file prompt per offender in `xprompts/toobig_split.yml`
   - the bug-audit prompt in `xprompts/audit_recent_bugs.yml`
   - the improvement-audit prompt in `xprompts/audit_recent_improvements.yml`
   - the documentation update and polish prompts in `xprompts/refresh_docs.yml`

The initial and detached prompts both need the directive. Gating only the initial short-lived workflow would not cover
the detached agents after that workflow exits; gating only the detached prompts would allow the chop workflow itself to
start amid unrelated agent work.

Explicitly exclude these non-root/non-user-agent paths:

- Script-only chops that never launch a user agent (including Telegram and maintenance chops).
- Axe hook, mentor, CRS, and other ChangeSpec runners governed by `axe.max_hook_runners` / `axe.max_agent_runners`; the
  `sase-5u` epic deliberately excludes them from the root-user-agent gate.
- The `fix_just.yml` `agent:` steps (`fix_linters` and `fix_tests`). They invoke the provider synchronously inside the
  already-gated `sase_fix_just` workflow root and do not pass through the root-agent runner-slot gate themselves. Adding
  `%w(runners=0)` to those step prompts would be parsed and stripped but would not enforce admission. The outer
  `FIX_JUST_PROMPT` barrier owns the runner slot for the whole workflow instead.
- Unrelated `sase run` examples/skills and ordinary prompts that merely mention chops.

## Implementation

### 1. Add barriers to every SASE workflow prompt that launches detached agents

Update the four bundled launcher workflows so each prompt passed to `launch_agent_from_cwd(...)` contains exactly one
`%w(runners=0)` directive:

- Add it to every generated split-file segment in `toobig_split.yml`.
- Add it to the generated bug- and improvement-audit prompts.
- Add it independently to both generated refresh-docs segments.

Keep the existing multi-prompt dependency semantics intact. In particular, later `toobig_split` segments must still
carry the existing bare `%wait` dependency on the preceding named agent, and the refresh-docs polish segment must still
wait for the update segment. A segment may therefore contain both the dependency wait and `%w(runners=0)`, but only one
`runners=` value.

### 2. Add barriers to every Athena chop entry point in chezmoi

Update all eight `agent:` values in `sase_athena.yml` to contain the exact directive while retaining the existing
workflow arguments and folded-YAML values.

Update both executable chop prompt sources:

- Add the directive to `FIX_JUST_PROMPT`.
- Add the directive to the first/directive line returned by the GitHub Actions fixer's `build_prompt()` for every repo.

Do not change scheduling, dedupe state formats, ChangeSpec guards, prompt bodies, or chop code beyond the prompt text.

### 3. Pin the complete inventory with focused regression coverage

In the SASE tests:

- Extend the executable `toobig_split` workflow test to parse every captured multi-prompt segment and assert
  `wait_runners == 0`.
- Add focused coverage for the generated audit and refresh-docs prompts, asserting that each detached launch parses to
  `wait_runners == 0` and that directives are stripped cleanly before model-visible prompt content.
- Assert the existing dependency chain remains present on later split/docs segments while the first segment has no
  unintended named dependency.
- Continue loading and validating all changed workflow YAML through the production loader/validator.

In the chezmoi tests:

- Update the existing `sase_fix_just_chop_test.sh` and `gh_actions_fix_chop_test.sh` expectations to require the exact
  barrier directive in the prompt captured from the fake `sase run -d` executable.
- Add a focused Athena-config invariant test that parses every `agent:`-backed chop in the overlay and requires exactly
  one `%w(runners=0)` occurrence. Pin the expected eight chop names so adding, removing, or silently omitting a
  configured agent chop fails loudly.

## Validation

1. In SASE, run `just install`, the focused workflow/directive tests, and the required full `just check`.
2. In chezmoi, run the focused bashunit chop/config tests and the full `just check`.
3. Perform a final two-repository launch-source audit:
   - enumerate every effective `agent:` chop;
   - enumerate executable chop scripts that call `sase run`;
   - enumerate bundled workflow steps that call `launch_agent_from_cwd`;
   - verify every in-scope constructed root prompt contains exactly one `%w(runners=0)` and still parses successfully.
4. Render the chezmoi-managed Athena config and inspect `sase axe chop list --json --verbose` after deployment to
   confirm all eight agent-backed prompts expose the directive. Do not manually execute the chops as a validation
   technique; tests and read-only config inspection avoid launching real agents or mutating external repositories.

## Rollout

Land/publish the SASE workflow changes before applying the chezmoi overlay, so a newly launched configured workflow can
only resolve to bundled leaf-prompt definitions that already carry their barriers.

Changing an agent-backed chop prompt changes its recorded `chop_prompt_hash`. Before applying/restarting the Athena
configuration, pause axe (or otherwise prevent new ticks) and confirm no affected old-hash agent chop is still live;
otherwise hash-filtered deduplication could treat the updated prompt as a distinct launch and enqueue a duplicate. Apply
the chezmoi change only once the affected chops are quiescent, then resume axe and confirm the effective prompt
inventory without forcing a chop run.

## Risks and Mitigations

- **Accidental duplicate `runners=` directives:** assert exactly one barrier per root prompt and parse prompts through
  the production directive parser; multiple `runners=` values are invalid.
- **Broken serial chains:** separately assert named dependencies and the runner threshold on multi-prompt segments.
- **Only the wrapper or only the leaf is gated:** the explicit two-layer inventory covers both short-lived workflow
  roots and detached agents.
- **Misleading barriers on in-process workflow steps:** keep those steps out of scope because they do not traverse the
  root-agent gate; rely on the outer workflow's slot instead.
- **Prompt-hash rollout duplication:** quiesce affected chops before applying the changed overlay.
- **Inventory drift:** pin the current configured chop names and audit all three launch mechanisms (`agent:`,
  `sase run`, and `launch_agent_from_cwd`).

## Out of Scope

- Automatically injecting runner directives in axe or changing general chop-launch semantics.
- Changing `max_running_agents`, FIFO admission, wait-marker behavior, or TUI display.
- Gating axe ChangeSpec runners or script-only chops.
- Redesigning the `fix_just` workflow or converting its in-process agent steps to detached agents.
- Broad documentation changes; `%w(runners=0)` and drain-barrier semantics are already documented by the completed
  `sase-5u` epic.
