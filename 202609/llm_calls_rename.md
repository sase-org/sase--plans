---
tier: tale
title: Rename the Agents provider-call panel to LLM Calls
goal:
  The Agents provider-call surface has an unambiguous LLM Calls identity without
  changing its behavior or artifact contracts.
size: medium
proposed_by: bbugyi200.apollo.0e.w0
create_time: 2026-09-18 03:08:02
status: wip
---

# Plan: Rename the Agents provider-call panel to LLM Calls

## Outcome and scope

Land the roadmap's standalone **LLM Calls rename** so `tool` is no longer an ambiguous
TUI panel/package name when the later `sase tool` control plane is designed. After this
change, the Agents detail picker, detail/zoom panels, help, footer, docs, and internal
panel namespace all say **LLM Calls**. The picker gesture remains `p` then `t`, and
every existing loading, caching, folding, zooming, exporting, and layout behavior
remains unchanged.

Implement this as one medium tale. The change is broad but mechanical and has one
indivisible user outcome; it does not need independently landed phases or multiple
agents. It is larger than a small patch because the panel identity crosses the TUI data
package, widget and zoom state machines, lazy exports, CSS selectors, tests, visual
goldens, documentation, and generated memory surfaces.

This tale is only the vocabulary migration. Do not add `sase tool`, ToolRun records,
managed-run cards, Bash-call linkage, or duplicate-suppression behavior. Those belong to
the later named-tools and surfaces epics after a ToolRun identity exists.

## Grounding and compatibility decisions

- The roadmap at `research:202609/sase_tool_epic_roadmap/sase_tool_epic_roadmap.md`
  makes this the third first move and explicitly includes the widget,
  `src/sase/ace/tui/tools/` package, keymap/default-config copy, docs, and a glossary
  strand.
- Agent `0e` landed commit `37d1b2592e` (`feat(agents): add detail view picker`). Its
  `p` modal now owns File / Tools / None selection plus the two existing layouts. This
  rename must update that picker coherently rather than introducing a parallel action or
  restoring the retired bracket-cycle paths.
- `src/sase/ace/tui/tools/` and `widgets/tools_panel.py` are presentation/read-model
  code for normalized provider tool-call artifacts. They are not managed SASE tools.
  Move the ambiguous package and panel namespaces to `llm_calls` / `llm_calls_panel`.
- Preserve names that accurately describe the provider artifact contract:
  `tool_calls.jsonl`, `tool_calls_writer_errors.jsonl`, `ToolCallEntry`, `SlowToolCall`,
  provider parser/writer APIs, and the public `ace.tool_calls` configuration block.
  Renaming those would create unrelated compatibility churn and would obscure that LLM
  Calls are still provider tool calls.
- Do not leave an old `sase.ace.tui.tools` package, `tools_panel` module, or old panel
  classes as compatibility aliases. These are internal TUI namespaces, and keeping a
  second vocabulary would defeat the roadmap goal. The on-disk provider artifacts and
  configuration above are the compatibility boundary.
- Panel and layout state is session-local presentation state. Rename enum members and
  selector ids directly; there is no persisted `"tools"` panel-mode value to migrate.
  Keep the picker key `t` and all other key ownership unchanged.
- This is presentation-only work in the Python/TUI repository. It needs no Rust-core
  API, wire-schema, store, feature flag, or compatibility branch.

## Naming contract

Use these names consistently for panel identity:

- user-facing label and rendered/exported heading: `LLM Calls` / `LLM CALLS`;
- package: `sase.ace.tui.llm_calls`;
- widget module and helper family: `llm_calls_panel.py` and `_llm_calls_panel_*.py`;
- primary/zoom widget names: `AgentLLMCallsPanel` and `ZoomLLMCallsPanel`;
- state/event names: `DetailPanelMode.LLM_CALLS`, `ZoomPanelTarget.LLM_CALLS`, and
  `LLMCallsVisibilityChanged`;
- widget selectors: `agent-llm-calls-*` and `zoom-llm-calls-*`;
- trace span: `widget.llm_calls_panel.update_display`;
- current-view chip value: `llm calls`.

Use `llm_calls_*` for panel capability/state fields and methods where `tools_*` meant
the panel as a whole. Keep `tool_call`, `slow_tool`, and related names where the value
is genuinely a provider tool-call record, source, threshold, or artifact. In particular,
do not mechanically rename every occurrence of the word `tool` across the repository.

## Implementation sequence

### 1. Move the provider-call read model and panel widget to their new namespaces

- Move `src/sase/ace/tui/tools/` to `src/sase/ace/tui/llm_calls/`, and move the
  corresponding focused reader tests from `tests/ace/tui/tools/` to
  `tests/ace/tui/llm_calls/`. Update all imports, monkeypatch strings, docs source-path
  references, and test fixtures that refer to the package path.
- Move `widgets/tools_panel.py` and its `_tools_panel_*` helpers to the
  `llm_calls_panel` naming family. Rename the panel class, visibility event, panel fetch
  result/cache aliases, and panel-scoped methods/fields so imports and trace labels no
  longer expose a generic Tools panel. Retain record-level types and functions such as
  `ToolCallEntry` when they still describe their data precisely.
- Update `widgets/__init__.py`, `widgets/__init__.pyi`, lazy export tables, dependent
  prompt/detail widgets, action mixins, and tests to export/import only the new panel
  names. Delete the old internal exports rather than aliasing them.
- Rename panel CSS ids/classes and every query selector in detail, zoom, helper, and
  test code. Preserve layout ratios, scroll restoration, background workers, mtime
  caching, refresh throttling, and message routing exactly; this rename must not add a
  render-path read, synchronous disk operation, polling loop, or list rebuild.

### 2. Rename detail, picker, zoom, and discovery state without changing behavior

- Rename the detail and zoom enum members and the related availability/state fields to
  `LLM_CALLS`. Update the Agents detail composition, show/hide transitions, visibility
  events, fold/detail-level routing, zoom seed/target cycling, search, copy/editor
  export, and refresh paths to use the renamed widgets and selectors.
- Update the new Agent view picker from commit `37d1b2592e`: the `t` row is **LLM
  Calls**, its subtitle explains that it shows provider tool calls/activity, layout rows
  dynamically say `LLM Calls larger`, and disabled guidance says
  `Choose File or LLM Calls first`. Preserve all current capability checks,
  stale-selection protection, same-choice no-op behavior, saved/current badges, `pp`
  swap behavior, and fixed modal-local key handling.
- Render `llm calls` in the top view chip and **LLM CALLS** in the timeline/export
  heading. Update loading and empty-state copy so it refers to LLM calls or provider
  tool-call artifacts, never a generic Tools panel. Update the metadata border
  indicator, zoom header/targets, footer labels, help, command descriptions/search
  terms, `src/sase/default_config.yml` comments, and keymap metadata. Keep `t` as the
  inner picker key and keep every action id/key binding otherwise unchanged.
- Update only panel-identity wording. Terms such as `slow tool calls`, provider tool
  names, tool-call thresholds, and `tool calls` as a searchable synonym remain correct
  and should not be rewritten into awkward terminology.

### 3. Update durable documentation and glossary terminology

- Rename the Agents documentation section and anchor to **Agents Tab LLM Calls Panel**,
  and update all intra-doc links from `docs/ace.md`, `docs/configuration.md`, and
  `docs/llms.md`. Update `docs/perf_runbook.md` source/trace references and any nearby
  TUI prose that still treats **Tools** as this panel's proper name. Do not rewrite
  unrelated uses of tools in other product areas.
- Through the authorized memory workflow, add `sase/memory/glossary/llm-calls.md` with
  the glossary term **LLM Calls**. Define it as the Agents detail/zoom view of
  normalized provider tool-call artifacts and explicitly distinguish it from named SASE
  tools/ToolRuns; it is a presentation/read-model term, not an execution record. Keep
  the definition compact so it helps future plans without adding unrelated policy.
- Run `sase memory init` after the canonical strand is added. Do not hand-edit generated
  `AGENTS.md`, provider shims, or the memory README; inspect the generated roster and
  instruction diff to ensure the new term appears once and no unrelated memory changed.

### 4. Rename and strengthen tests, then review the rendered result

- Move panel-identity test modules/helpers and visual test names from `tools_panel` to
  `llm_calls_panel`, while leaving tests whose subject is genuinely slow provider tool
  calls named accordingly. Update fake widgets, selector maps, type-name assertions,
  monkeypatch paths, and expected footer/help/export strings.
- Extend the picker tests added by `37d1b2592e` to assert the `t` row and dynamic layout
  labels say **LLM Calls**, `pt` still selects the same panel exactly once, unsupported
  and metadata-only entries retain their disabled behavior, and File / None / layout
  transitions are unchanged.
- Exercise the zoom modal's renamed target, availability fallback, detail-level keys,
  search, editor/copy export, and refresh event. Preserve the provider-neutral reader
  parity tests for Claude, Codex, Grok, Qwen, Agy, and hook records under the renamed
  package.
- Rename the three `agents_tools_panel_*` visual cases/goldens to
  `agents_llm_calls_panel_*`, regenerate only intentionally affected images, and inspect
  them. Also run the existing slow-tool/Agents visuals whose picker, chip, footer, or
  border copy changes. The images must visibly show **LLM Calls** without layout,
  wrapping, focus, color, or scroll regressions.

## Validation and acceptance

Run focused checks after the moves so import failures are easy to localize, then the
repository gates:

1. Run the renamed provider-reader suite, panel cache/event/timeline tests, picker and
   modal tests, detail transition/folding tests, zoom tests, keymap/help/footer tests,
   and relevant documentation/link checks.
2. Update the targeted PNG cases with the repository snapshot flag, inspect the
   actual/expected/diff artifacts, then run `just test-visual` to verify the complete
   visual suite.
3. Run a targeted stale-name audit. Outside historical prose or data that genuinely
   denotes provider tool calls, there must be no `sase.ace.tui.tools`, `tools_panel`,
   `AgentToolsPanel`, `ToolsVisibilityChanged`, `DetailPanelMode.TOOLS`,
   `ZoomPanelTarget.TOOLS`, `agent-tools-*`, `zoom-tools-*`, or user-facing **Tools
   panel** identity. Review every remaining `tool_calls`, `ToolCall`, and `slow tool`
   hit rather than globally replacing it.
4. Run `just fix` (or at minimum `just fmt`), `git diff --check`, and `just check`. Let
   `just check` complete any automatic full-suite escalation caused by source/test
   moves. Use `/sase_monitor` with the required TESTING/TESTED handoff if a check
   becomes long; run `just check-full` only if the documented broadening/landing rules
   require it, also through the monitor.

The tale is complete when a normal user opens the Agents view picker with `p`, presses
`t`, and sees **LLM Calls** consistently in the picker, view chip, detail timeline,
zoom/export surfaces, help, and docs; all prior panel behavior and provider artifact
compatibility remain intact; the ambiguous internal `tools`/`tools_panel` namespaces are
gone; and the glossary gives future `sase tool` plans one clear term for the old
provider-call surface.
