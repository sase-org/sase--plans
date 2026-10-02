---
tier: tale
title: Fix research_swarm handoff loops that never render at launch
goal:
  The research_swarm lead and linker prompts list real registered-report fields at
  launch, and plugin tests render deferred loops through the launch-time renderer so
  this regression cannot ship again.
size: small
proposed_by: bbugyi200.apollo.43
create_time: 2026-10-02 08:31:58
status: wip
---

# Plan: Fix `#research_swarm` handoff loops that never render at launch

## Problem

The `research.01.final` lead agent stopped at step 1 without writing a consolidated
report. Its "registered reports" list held five rows of unrendered placeholders, for
example ``- wait_name=`{{ a.wait_name }}` label=`{{ a.label }}` ...``. The prompt gave
it nothing to match against the five researcher reports (cdx, cld, grk, mus, gem), so it
correctly failed closed. The waiting `research.01.linker` would hit the same defect.

## Root cause

sase-research-artifacts commit `6859e88` ("render artifact fields as inline code so
prompt formatting preserves __ literals") wrapped every field of the raw-deferred
`wait.artifacts` loops in backticks:

```text
{% raw %}{% for a in wait.artifacts if ... %}
- wait_name=`{{ a.wait_name }}` label=`{{ a.label }}` ...
{% endfor %}{% endraw %}
```

`{% raw %}` defers the loop past xprompt expansion. The loop is then rendered by sase's
launch-time top-level pass (`preprocess_prompt_late` step 1 `protect_fenced_blocks` and
step 5 `render_toplevel_jinja2`). By design (commit `f5d718444a`), that pass treats
**inline code as well as fenced code** as literal. Each backticked `{{ a.* }}` span is
swapped for a placeholder before Jinja runs. The `{% for %}` outside the spans still
runs once per artifact, then every placeholder is restored verbatim. The result is N
rows of literal `{{ ... }}`. Declared-input substitution (`substitute_placeholders`)
renders inside inline code, but launch-time rendering does not, and the deferred loop
lands in the stricter pass.

Two things let the regression ship:

1. **Test-fidelity gap.** The plugin tests render the deferred loops with
   `sase.xprompt.workflow_executor_utils.render_template`. That workflow-step renderer
   protects only disabled regions, so it renders inside backticks and the tests passed.
   Reproduced on the real lead segment:
   - `render_template` gives ``wait_name=`research.m.cdx` ...``.
   - `sase.xprompt.render_toplevel_jinja2` gives ``wait_name=`{{ a.wait_name }}` ...``.
2. **The backticks were redundant.** sase commit `309551d92b`, landed three minutes
   earlier by the same agent, made `format_agent_prompt_markdown` (launch step 6)
   preserve `_` and `*` literals with private-use sentinels. Raw prettier still rewrites
   `t__cdx.md ... gh_sase-org__sase` to `t**cdx.md ... gh_sase-org**sase`.
   `format_agent_prompt_markdown` leaves it intact. The commit is after `v0.17.1`, so it
   ships in the 0.17.2 line. The plugin already requires `sase>=0.17.2`, so plain
   `key=value` fields are safe and no floor bump is needed.

## Changes

### 1. sase-research-artifacts: render handoff fields outside inline code

Open the linked plugin repo with `sase repo open sase-research-artifacts -r "<reason>"`,
then read its `AGENTS.md`.

In `src/sase_research_artifacts/xprompts/research_swarm.md`, restore plain field
rendering in all three raw-deferred loops:

- The lead segment's `wait.artifacts` markdown loop (the "researchers' registered
  reports" list).
- The linker segment's markdown loop (the "lead researcher's registered report" list).
- The linker segment's `kind == "image"` loop.

Each row becomes, for example:

```text
- wait_name={{ a.wait_name }} label={{ a.label }} source_path={{ a.source_path }} path={{ a.path }} ref={{ a.ref }}
```

The image row keeps its `vcs_relpath={{ a.vcs_relpath }}` field. Do not generate the
backticks from Jinja (for example `{{ "`" ~ x ~ "`" }}`). `__` preservation belongs to
sase's literal-preserving formatter, and the plain form keeps the template readable.
Leave everything else in the segments untouched. Step prose that mentions
`` `wait_name` `` in backticks is fine, because it contains no Jinja.

### 2. sase-research-artifacts: tests render through the launch-time renderer

In `tests/test_xprompt_loading.py`:

- Revert `_WAIT_ARTIFACTS_LOOP` and `_WAIT_IMAGE_ARTIFACTS_LOOP` to the plain field
  form, matching the template exactly.
- Add one helper (for example `_render_at_launch(segment)`) that renders a segment under
  the bound `wait` namespace with `sase.xprompt.render_toplevel_jinja2`. This is the
  public launch-time top-level renderer, which applies the same fenced and inline-code
  literal protection as `preprocess_prompt_late`. Do not call `preprocess_prompt_late`
  itself: its command-substitution step would execute the segment's
  `$(sase repo path research --ensure)` in tests.
- Switch every deferred-loop render from `render_template` to that helper:
  - `test_research_swarm_lead_renders_registered_reports_via_wait_artifacts`
  - `test_research_swarm_linker_renders_registered_lead_via_wait_artifacts`
  - `_render_swarm_with_dunder_artifacts`

  Then drop the now-unused `render_template` import.

- Update those tests' assertions to the plain form (for example
  `"wait_name=research.m.cdx label=research:202609/topic/topic__a.md" in rendered`).
  Keep the existing `"{{" not in rendered` and `"{%" not in rendered` assertions.
  Against the current backticked template they now fail, which is the regression signal
  the suite lacked.
- Replace `_assert_handoff_fields_backticked` and
  `test_research_swarm_handoff_fields_render_as_inline_code` with a launch-render test.
  It asserts that every handoff field in the rendered lead and linker loop lines carries
  its real value, with `__` intact (`t__cdx.md`, `gh_sase-org__sase`) and no `{{`.
- Keep `test_research_swarm_handoff_fields_survive_prompt_formatting` (still
  prettier-gated). Feed it the launch-rendered lead and assert the plain labels and
  paths survive `format_agent_prompt_markdown`, with no `**` rewrites.
- Add a static contract test over every plugin xprompt loaded via
  `load_xprompts_from_plugins()` whose name starts with `research`. For each
  `{% raw %}...{% endraw %}` region, assert that no backtick-delimited span on a line
  contains `{{` or `{%`. A regex over the raw region's lines is enough; do not import
  private `sase.xprompt._*` modules. On failure, the message should explain that
  deferred Jinja renders at launch, where inline code is literal.

### 3. sase-research-artifacts: docs

In `docs/xprompts.md`, replace the sentence added by `6859e88`, "Each field value
renders as inline code, so launch-time Markdown formatting cannot rewrite the `__` in
labels and paths.", with an accurate explanation:

- The loop's fields are plain `key=value` text, because the deferred loop renders in
  sase's launch-time top-level pass, where inline code is literal and would leave
  `{{ ... }}` unrendered.
- `__` in labels and paths survives because sase's agent-prompt formatter preserves `_`
  and `*` literals.

Do not hand-edit `CHANGELOG.md`; it is release-please managed.

### 4. sase: document the deferred-Jinja inline-code trap

In this repo's `docs/xprompt.md`, in the "Jinja2 Integration" section, add a short
paragraph right after the paragraph that introduces the `wait` namespace ("The `wait`
namespace is only populated while an agent run is rendering its executable prompt...").

It should say that an xprompt using runtime-only names such as `wait.artifacts` defers
them with `{% raw %}...{% endraw %}`, and that the deferred template renders in the
launch-time top-level pass, where fenced and inline code are literal. A `{{ ... }}`
inside backticks there is never substituted, and an enclosing `{% for %}` still repeats
the literal once per item. Deferred expressions must stay outside code spans. Rendered
values still go through literal-preserving prompt formatting, so `__` and `*` in paths
survive.

Keep it to a few sentences matching the surrounding prose. No sase code change is
needed. The launch-time literal semantics are intentional: user prompts may quote Jinja
in backticks.

## Verification

- Before the template fix, the switched lead and linker render tests fail with literal
  `{{ a.wait_name }}` rows. After the fix, they pass.
- Plugin: `sase tool run check` in the opened sase-research-artifacts checkout (per its
  `AGENTS.md`; bare `just check` is guarded).
- sase: `sase tool run check` in this repo for the docs change.
- Spot-check with a short script: expand `research_swarm` from the edited plugin file
  (`load_xprompt_from_file` + `expand_single_xprompt` with `image=true`). Bind
  `wait.artifacts` to a fake markdown artifact whose label and source path both contain
  `__`. Render the lead segment with `render_toplevel_jinja2`, then pass it through
  `format_agent_prompt_markdown`. Confirm that the row carries real values and that `__`
  survives.

## Out of scope

- Re-running the stalled `research.01` swarm. The five researcher reports are registered
  and intact. Once the fix is installed, the user can relaunch the lead consolidation.
- Changing sase's launch-time inline-code literal semantics, or adding a `sase doctor`
  lint for code-wrapped Jinja in raw regions. `src/sase/xprompts/skills/sase_var.md`
  intentionally uses inline-code-wrapped raw Jinja as literal documentation, so a
  generic lint would false-positive.
