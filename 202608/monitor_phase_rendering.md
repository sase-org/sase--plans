---
tier: tale
title: Render monitor phases as MONITOR with their command and captured log
goal:
  A monitor family member reads as `MONITOR` everywhere in the ACE prompt panel, and
  selecting a family that owns monitors shows each monitor's command, recorded detail
  fields, and full captured output directly beneath that header.
size: medium
proposed_by: bbugyi200.athena.05z.f0.f0
create_time: 2026-08-18 10:41:50
status: wip
---

# Plan: Render monitor phases as `MONITOR` with their command and captured log

## Goal

Two things, one shape.

First, restore `MONITOR` as the phase header for a monitor family member. The recent
`AGENT (<role>)` unification swept `--mon` in with the agent roles, so a monitor now
reads `AGENT (monitor)`. That is wrong by SASE's own vocabulary: an **agent shell** is
one concrete LLM/provider run, while a monitor is a **proc shell** — a supervised OS
command. `AGENT (<role>)` is the naming rule for agent shells; a proc shell needs its
own header.

Second, make a monitor phase actually show its work. Today, selecting a family that owns
monitors renders a monitor phase as if it were an agent turn: no command, no state, no
exit code, and the raw command log pushed through the markdown lexer. Put the command
directly under the `MONITOR` header, follow it with the same detail fields the monitor's
own panel shows, and render the full captured output the way that panel already renders
it.

## Current Behavior

### Phase labels

`get_phase_label()` in `src/sase/ace/tui/widgets/prompt_panel/_agent_display_content.py`
maps a member's resolved family role to one header. Since the `AGENT (<role>)`
unification landed (`3df5c321b`), the `monitor` role returns `AGENT (monitor)`.

Five call sites consume that label, all display-only:

- `_agent_display_render.py` — AGENT REPLY phase dividers on a non-container agent that
  owns follow-ups.
- `_agent_display_hint_render.py` — the same dividers in hint mode.
- `_agent_display_family_render.py` — family-conversation phase dividers.
- `_agent_display_family.py` — `kind` for FAMILY MEMBERS roster rows.
- `_agent_display_neighbors.py` — `kind` for lane-neighbor roster rows.

Both roster adapters do `kind = "agent" if phase_label == "AGENT" else phase_label`, so
a monitor row currently reads `--mon · AGENT (monitor) · ⚙ MONITORING · shell · 15m`.

### A monitor phase inside a family

`concrete_family_member_rows()` includes monitor members, so a monitor is a phase in the
AGENT REPLY stream. `_update_family_display()` renders every phase identically: a purple
`render_phase_divider()` followed by
`render_agent_reply_content(phase, render_markdown)`. Rendering the family container
from the fixture in this plan's verification script produces exactly this today:

```text
AGENT REPLY · 2
─── AGENT (code) ─── 13:00:00 ────────────────────

Done implementing.

─── AGENT (monitor) ─── 13:01:00 ─────────────────

# not a heading
* not a bullet
FAILED tests/ace/tui/test_x.py::test_y
===== 1 failed, 33162 passed in 725.23s =====
```

Three defects are visible or latent there:

1. **The command is absent.** Nothing in the family view says what ran.
   `monitor_command`, `monitor_cwd`, `monitor_reason`, `monitor_next_action`,
   `monitor_state`, `monitor_exit_code`, the timeout budget, and the
   `sase monitor show <id> --follow` handle are all loaded on the row and all dropped.
2. **The log goes through the markdown lexer.** `render_agent_reply_content` calls
   `lazy_renderable(content, "markdown")`, i.e. Rich `Syntax` with a markdown lexer and
   the monokai theme. It does not restructure the text, but it paints a full-width dark
   code-block background over the whole log and colors `#`, `*`, backticks, and `**` as
   markdown tokens inside output that is not markdown.
3. **ANSI escapes are not decoded.** A monitor's `live_reply.md` holds raw merged
   stdout/stderr; most monitored commands (pytest, ruff, cargo, `just`) emit SGR codes.
   The monitor's own panel decodes them with `render_axe_output(..., "ansi")`; the
   family view passes the escape bytes straight through. Rendering
   `"\x1b[32mPASSED\x1b[0m tests/foo.py\n"` down each path confirms it: the ANSI path
   emits `PASSED` styled green, the markdown-lexer path emits the literal escape bytes
   wrapped in monokai styling.

The same three defects apply to the `agent.followup_agents` phase loop in
`_agent_display_render.py`, which is the path taken when the selected row is the
monitor's _starter_ rather than the family container.

### Hint mode on a monitor row is empty

`_update_display_with_hints_impl()` has no `agent.is_monitor` branch, so a monitor row
selected while file-path hint mode is active falls through to the ordinary prompt path,
finds no prompt file, and renders:

```text
AGENT PROMPT
No prompt file found.
```

The command, every field, and the entire captured log disappear until hint mode is
toggled off. This is a pre-existing bug, not a regression from the rename, but it is the
same behavior in a different render mode and is fixed by the same helpers.

## Decisions

**A monitor phase header is `MONITOR`, not `AGENT (monitor)`.** The glossary is the
argument: a _sase shell_ is either an _agent shell_ (one concrete LLM/provider run) or a
_proc shell_ (a named supervised proc). A monitor is a family-attached proc shell.
`AGENT (<role>)` says "this phase is an agent turn, and here is which role took it" —
true for `plan`, `code`, `q`, `epic`, `commit`, feedback rounds, and custom members;
false for a monitor. Keeping one header per _kind of shell_ is a cleaner rule than one
header for every family member, and it is the rule the panel had before the unification.

**`get_phase_label()` stays the single source of truth.** The FAMILY MEMBERS and
NEIGHBORS rosters keep calling it and pick up `MONITOR` for free. The phase-divider call
sites stop calling it directly for monitors only because the new monitor-phase renderer
calls it for them.

**A monitor phase renders the same fields its own panel renders.** Full parity: command,
`Cwd`, `Reason`, `Next action`, `State` (+ exit code), `Timeout`/`Elapsed`,
`Idle timeout`, `Monitor id` and its `sase monitor show … --follow` handle. The
alternative — showing only the command plus state — was considered and rejected. It
saves about six lines per monitor in a view that is already dominated by the log itself,
and it buys that with a second, divergent monitor layout that will drift from the first.
One field block, built once, rendered in both places, cannot disagree with itself.
`Reason` in particular is the most valuable line in a conversation view: it is the only
place that records _why_ this command ran.

**The monitor phase divider is amber and carries the `⚙` glyph.** A monitor phase now
renders structurally different content from every neighbouring phase — fields and a raw
log instead of prose. Giving it the same purple divider as an agent turn hides that
difference behind identical chrome. ACE already has exactly one monitor hue and one
monitor mark, both exported from `sase.monitor_state`: `MONITOR_GLYPH_COLOR` (`#FFAF5F`)
and `MONITOR_GLYPH` (`⚙`), used by the agent list, the roster, the procs pane, and the
top bar. Reusing them makes a monitor phase scannable in one glance and adds no new
visual vocabulary:

```text
─── ⚙ MONITOR ─── 13:01:00 ───────────────────────
```

This is a deliberate change from the pre-unification look (which was a purple
`MONITOR`). If the reviewer wants the header text restored without the new accent, drop
the `accent` and `glyph` arguments at the single call site in `build_monitor_phase()`;
nothing else in this plan depends on them.

**Inside a phase, the log is introduced by an `Output:` field, not an `OUTPUT` section
heading.** The monitor's own panel gives `OUTPUT` a real section heading because it is a
top-level section of that document, navigable with `Ctrl+J`. A family document already
spends its section anchors on `AGENT XPROMPT`, `AGENT PROMPT`, `AGENT REPLY`,
`FAMILY MEMBERS`, and `member:<name>`; minting a duplicate `output` anchor per monitor
would make section navigation ambiguous. Using `_field_label("Output:")` keeps one
visual language inside the phase, matches how `Command:` already labels a block that
starts on the next line, and registers no anchor.

**Hint mode annotates the command and the log, and nothing else.** Hint mode's contract
is that visible free-form content is plain text carrying `[N]` markers. The command and
the captured log are free-form and full of paths worth jumping to
(`pytest tests/foo.py`, a failing node id). `Cwd`, `Monitor id`, and the state fields
are structured metadata and stay verbatim. In hint mode the command renders as plain
`Text` rather than shell-highlighted `Syntax`, exactly as hint mode already degrades
every other syntax block.

**No feature flag.** Per the feature-flag rule in `CLAUDE.md`, a flag is for a disabled
beta, an early landed path, or a deprecation whose old branch must stay reachable. This
is a display correction that is right the moment it lands, with no old branch to keep
alive.

**No Rust core change.** Phase headers, dividers, accent colors, and field layout are
Textual presentation. `crates/sase_core` carries no phase-label or monitor-display
logic; per `sase/memory/rust_core_backend_boundary.md` this stays in the Python TUI.

## Target Behavior

### Labels

| Family role (canonical suffix) | Label today           | Label after |
| ------------------------------ | --------------------- | ----------- |
| `monitor` — `--mon`, `--mon-N` | `AGENT (monitor)`     | `MONITOR`   |
| every other role               | `AGENT (…)` / `AGENT` | unchanged   |

The FAMILY MEMBERS and NEIGHBORS rosters follow automatically:
`--mon · MONITOR · ⚙ MONITORING · shell · 15m`.

### A monitor phase in the AGENT REPLY stream

```text
─── ⚙ MONITOR ─── 13:01:00 ───────────────────────
  Command:
just check-full
  Cwd:         /home/bryan/sase
  Reason:      Full-suite verification before landing
  Next action: Report pass/fail to the user.
  State:       ✓ completed  (exit 0)
  Timeout:     15m0s of 45m0s budget
  Monitor id:  gh6fddk5v3g9  (gh6fdd)
               sase monitor show gh6fdd --follow
  Output:
✓ lint (ruff)
FAILED tests/ace/tui/test_x.py::test_y
===== 1 failed, 33162 passed in 725.23s =====
```

The divider, the `⚙`, and the label are amber; the field block reuses the exact styles
the monitor's own panel already uses; the command is shell-highlighted; the log is ANSI
decoded. A capped log keeps its `… output truncated (head + tail retained) …` notice
above the body. A monitor that has written nothing shows `No output yet.` under
`Output:`.

Fields are emitted exactly as the monitor's own panel emits them, including their
conditionals: absent `Cwd`, `Reason`, `Next action`, `Idle timeout`, and `Monitor id`
rows are skipped, and `Timeout` degrades to `Elapsed` when no budget was recorded. A
monitor row whose metadata has not been enriched yet shows the same sparse block its own
panel would show, rather than a special "not loaded" state.

### The monitor's own panel

Unchanged in normal mode: `▸ MONITOR` fold heading, the same fields, the `OUTPUT`
section heading, the same ANSI log. In hint mode it renders that same document as plain
text with `[N]` markers instead of collapsing to `No prompt file found.`

## Implementation

### 1. `src/sase/ace/tui/widgets/prompt_panel/_agent_display_content.py`

- Add `MONITOR_PHASE_LABEL = "MONITOR"` and return it from the `role == "monitor"`
  branch of `get_phase_label()`. Leave every other branch, the branch order, and the
  `is_promoted_root` guard exactly as they are.
- Update the `get_phase_label` docstring: agent shells map to `AGENT (<role>)`; a
  monitor is a proc shell and keeps `MONITOR`.
- Lift the hard-coded divider color into `PHASE_DIVIDER_ACCENT = "#AF87FF"` and widen
  the divider signature:

  ```python
  def render_phase_divider(
      label: str,
      start_time: datetime | None,
      *,
      accent: str = PHASE_DIVIDER_ACCENT,
      glyph: str | None = None,
  ) -> Text:
  ```

  Replace the four literal `#AF87FF` styles with `accent`. When `glyph` is given, append
  `f"{glyph} "` in `bold {accent}` immediately before the label and add `len(glyph) + 1`
  to the `used` width so the trailing rule stays the same length. Every existing caller
  keeps its current output.

`_PHASE_SUFFIX_TOKENS` needs no entry for `--mon`: `agent_family_role_for_suffix()`
already resolves both `--mon` and `--mon-N` to the `monitor` role, so that branch is
never reached for a monitor.

### 2. `src/sase/ace/tui/widgets/prompt_panel/_agent_monitor_section.py`

This module becomes the one place monitor detail is composed. Import `MONITOR_GLYPH` and
`MONITOR_GLYPH_COLOR` from `sase.monitor_state` (it already imports
`MONITOR_TIMEOUT_GLYPH` from there), `get_phase_label` and `render_phase_divider` from
`._agent_display_content`, and `render_axe_output` from `...util.axe_log_renderer`. None
of those modules import this one, so no cycle is introduced.

Define one shared annotation type —
`MonitorTextAnnotator = Callable[[str | Text], Text]` — used to thread hint-mode
decoration through the free-form blocks. When an annotator is passed, every part
returned by these builders is a `Text`.

- **`_monitor_field_parts(agent, *, annotate) -> list[object]`** (private): everything
  `build_monitor_section` currently emits _after_ its heading, unchanged. The command
  block renders as `lazy_renderable(agent.monitor_command, "bash")` when `annotate` is
  `None`, and as `annotate(agent.monitor_command + "\n")` otherwise.
- **`build_monitor_section(agent, *, panel_level, scale, annotate=None)`**: the existing
  `▸ MONITOR` fold heading followed by `_monitor_field_parts(...)`. With `annotate=None`
  its output is byte-for-byte what it produces today.
- **`build_monitor_output(agent, *, heading=True, annotate=None) -> list[object]`**: the
  output block, lifted verbatim out of `_update_monitor_display`. With `heading=True` it
  emits the blank line, the `─` rule, and the `OUTPUT` section heading; with
  `heading=False` it emits `_field_label("Output:") + "\n"` instead and registers no
  section anchor. Then the truncation notice when `agent.monitor_output_truncated`, then
  either the body —
  `render_axe_output(monitor_output_source_id(agent), output, "ansi")`, passed through
  `annotate` when given — or the `No output yet.` placeholder.
- **`monitor_output_source_id(agent) -> str`** (private): the existing
  `f"monitor:{agent.monitor_id or agent.identity}"`, so both panels share one
  `render_axe_output` cache slot.
- **`build_monitor_phase(agent, *, annotate=None) -> list[object]`**: the new
  family-facing renderer.

  ```python
  parts = [
      render_phase_divider(
          get_phase_label(agent),
          agent.run_start_time or agent.start_time,
          accent=MONITOR_GLYPH_COLOR,
          glyph=MONITOR_GLYPH,
      )
  ]
  parts.extend(_monitor_field_parts(agent, annotate=annotate))
  parts.extend(build_monitor_output(agent, heading=False, annotate=annotate))
  return parts
  ```

- **`monitor_phase_text(agent, *, annotate) -> Text`**: flattens
  `build_monitor_phase(agent, annotate=annotate)` into one `Text` with `append_text`,
  for the hint-mode call sites that append into an existing `Text` rather than a
  `Group`. Requiring `annotate` keeps the all-`Text` contract honest.

Update `__all__` (keep it sorted; `keep-sorted` lints it).

### 3. `src/sase/ace/tui/widgets/prompt_panel/_agent_display_step_render.py`

`_update_monitor_display` drops its inline output block and calls
`build_monitor_output(agent)` instead. Its rendered result is unchanged; this only moves
the code so the family path and the panel path cannot drift.

### 4. `src/sase/ace/tui/widgets/prompt_panel/_agent_display_family_render.py`

In the `for phase in phases:` loop of `_update_family_display`, branch once at the top:

```python
if phase.is_monitor:
    renderables.extend(
        build_monitor_phase(
            phase,
            annotate=(
                None
                if hint_state is None
                else self._monitor_phase_annotator(phase, hint_state, hint_budget)
            ),
        )
    )
    continue
```

Add the annotator factory to the same mixin. It reuses the existing per-phase workspace
guard so a monitor's paths resolve against the monitor's own workspace:

```python
def _monitor_phase_annotator(self, phase, hint_state, budget):
    def annotate(content: str | Text) -> Text:
        text = content if isinstance(content, Text) else Text(content)
        return self._family_text_with_hints(
            text,
            hint_state,
            workspace_dir=self._family_member_hint_workspace(phase, text.plain),
            budget=budget,
        )
    return annotate
```

The shared `HintContentBudget` keeps a family full of large monitor logs from blowing
the hint scan, exactly as it does for large agent replies today.

### 5. `src/sase/ace/tui/widgets/prompt_panel/_agent_display_render.py`

In the `for followup in agent.followup_agents:` loop, add the same guard using the
non-hint form:

```python
if followup.is_monitor:
    renderables.extend(build_monitor_phase(followup))
    continue
```

The main agent's own phase needs no guard: `_update_display_impl` already returns
through `_update_monitor_display` before this path when `agent.is_monitor`.

### 6. `src/sase/ace/tui/widgets/prompt_panel/_agent_display_hint_render.py`

Two changes.

- In the follow-up phase loop,
  `header_text.append_text(monitor_phase_text(followup, annotate=…))` for a monitor
  follow-up, where the annotator wraps `append_bounded_text_with_file_hints` against the
  already-resolved `workspace_dir`, `hint_counter`, and `hint_mappings` this path
  already threads.
- Add an `agent.is_monitor` branch beside the existing workflow-child branches, after
  `header_text` is built, mirroring `_update_monitor_display`:
  `build_monitor_section(agent, panel_level=…, annotate=…)` then
  `build_monitor_output(agent, annotate=…)`, appended into `header_text`, then the
  normal `AgentHintRender` return. This is the fix for the empty hint-mode monitor
  panel.

### 7. No other source changes

`_agent_display_parts.py` re-exports `get_phase_label` and `render_phase_divider`; both
keep working, and the divider's new arguments are keyword-only with defaults, so its
`__all__` needs no edit. `MONITOR_FAMILY_ROLE`, `is_monitor_member_role`, and the
`PLAN_CHAIN_MONITOR_SUFFIX` constants are unchanged.

## Explicit Non-Goals

- The monitor's own panel in **normal** mode. Its heading, fields, `OUTPUT` section, and
  ANSI log stay exactly as they are; only where that code lives changes.
- The compact attribution tokens in `src/sase/ace/tui/agent_context_members.py`. Those
  are lowercase context-lane tokens, not phase headers.
- Roster styling. Only the `kind` string changes to `MONITOR`; its style, the status
  glyph, and the roster layout are untouched.
- Hinting the structured monitor fields (`Cwd`, `Monitor id`, state rows). Only the
  command and the captured log are annotated.
- Fold-level suppression of the monitor field block inside a family phase. Family reply
  bodies are fully visible at every fold level today and stay that way.
- Any `sase/memory/*.md`, `AGENTS.md`, or generated provider shim edit. None of them
  mention these headers, and none may be edited without separate explicit user
  permission.

## Tests

- `tests/ace/tui/widgets/test_agent_display_phase_labels.py` — retarget `test_monitor`
  and `test_monitor_numbered_suffix` to `"MONITOR"`, and add a case for a member whose
  stored `agent_family_role="monitor"` carries an unrecognized suffix, proving the role
  branch (not the suffix table) owns the label. Every other case in the file must keep
  passing unchanged — that is the guard proving the carve-out is exactly one role wide.
- `tests/ace/tui/widgets/test_agent_display_family_member_roster.py:247` — the roster
  assertion becomes `("--mon", "MONITOR", "MONITORING")`.
- `tests/ace/tui/widgets/test_agent_display_phase_divider.py` — add coverage for the new
  keyword arguments: a default call still renders bold `#af87ff`, and an
  `accent`/`glyph` call renders the glyph before the label in the requested hue.
- `tests/ace/tui/widgets/test_agent_prompt_panel_monitor.py` — extend with a family
  fixture (a `--code` root plus a `--mon` follow-up carrying `monitor_command`,
  `monitor_cwd`, `monitor_reason`, `monitor_state`, `monitor_exit_code`, and a
  `live_reply.md`) and assert, on the rendered container:
  - the phase header is `MONITOR` and `AGENT (monitor)` appears nowhere;
  - the command, `Cwd`, `Reason`, `State`, exit code, and the
    `sase monitor show … --follow` handle are all present;
  - the full captured log is present;
  - ANSI input (`"\x1b[31mFAILED\x1b[0m tests/x.py\n"`) renders as styled `FAILED` with
    no escape bytes surviving into the rendered plain text — the regression guard for
    the markdown-lexer path;
  - a truncated monitor keeps its elision notice, and an output-less monitor shows
    `No output yet.`;
  - the same assertions hold when the selected row is the monitor's _starter_ (the
    `followup_agents` path) rather than the family container.
- Hint-mode coverage in the same file:
  - a family with a monitor rendered through `update_display_with_hints` shows the
    command and the log, and a path in the log receives a `[N]` marker recorded in
    `file_hints`;
  - a standalone monitor row rendered through `update_display_with_hints` shows
    `MONITOR`, the command, and `OUTPUT` — and never `No prompt file found.` This is the
    regression test for the pre-existing hint-mode gap.
- `tests/test_timezone_runtime_consistency.py:234` passes a literal label into
  `render_phase_divider` and is unaffected.

## Docs

- `docs/ace.md`, the AGENT REPLY bullet near line 3835: the `AGENT (<role>)` rule now
  covers agent-shell members only. Remove `--mon` from that list and add a sentence
  saying a monitor member is a proc shell, so its phase renders as an amber `⚙ MONITOR`
  divider followed by the monitor's command, its recorded detail fields, and its full
  captured output — the same block the monitor's own panel shows. Keep the file's prose
  wrapping.
- `docs/monitors.md`, the "In the ACE TUI" section: after the paragraph describing the
  `MONITOR` detail section and its `OUTPUT` block, add that the same block appears
  inline as a `MONITOR` phase when the monitor's family (or its starter) is selected,
  and that file-hint mode now renders the monitor document with `[N]` markers instead of
  falling back to the empty prompt view.

## Visual Snapshots

No existing PNG golden contains a monitor family member — the only visual fixtures that
set `monitor_command` are the Procs-tab helpers — so `just test-visual` is expected to
stay green with no golden churn. If any golden does move, stop and investigate rather
than accepting it.

Add one new golden that covers the new phase, since the whole point of the amber divider
and the field block is visual. In
`tests/ace/tui/visual/test_ace_png_snapshots_agents_family_panel.py`, give
`_family_agents` a `with_monitor: bool = False` parameter that appends a settled `--mon`
member (command, cwd, reason, next action, exit code, and a small `live_reply.md`) to
the family, and add one test that opens the family conversation with it and captures
`agents_family_conversation_monitor_120x40`. Defaulting the parameter to `False` keeps
every existing golden in that file byte-identical.

```bash
just test-visual                                     # confirm existing goldens are clean
just test-visual -- --sase-update-visual-snapshots   # accept the one new golden
just test-visual                                     # rerun clean
```

Inspect the new golden (or its `.pytest_cache/sase-visual/` artifact) and confirm the
divider, the shell-highlighted command, and the field alignment read the way this plan
describes.

## Verification

1. `just install` first — workspaces are ephemeral and dependencies may be stale.
2. `just check` while iterating.
3. `just test-visual` plus the golden step above.
4. `just check-full` before landing, run through `/sase_monitor` with a `--next` action
   (it routinely outruns a single agent turn). This change touches shared prompt-panel
   render helpers used by every agent document, so the full suite is required rather
   than the scoped lane.

Performance note for the implementer: this removes work rather than adding it. The
family view already reads `live_reply.md` for monitor phases, so there is no new disk
I/O, and `render_axe_output` is both tail-capped (`cap_ansi_output`) and cached per
`(source_id, source_type)`, where the markdown `Syntax` path it replaces is neither. No
render path gains a stat or a glob.

## Done When

- `get_phase_label()` returns `MONITOR` for the `monitor` role and is unchanged for
  every other role.
- The FAMILY MEMBERS and NEIGHBORS rosters show `MONITOR` as a monitor member's `kind`.
- Selecting a family container, or a monitor's starter, renders each monitor phase as an
  amber `⚙ MONITOR` divider followed by the command, the monitor's detail fields, and
  the full ANSI-decoded captured output.
- Hint mode renders the same content with `[N]` markers on the command and the log, and
  a standalone monitor row in hint mode no longer collapses to `No prompt file found.`
- The monitor's own panel is unchanged in normal mode.
- `docs/ace.md` and `docs/monitors.md` describe the new rule and the new phase block.
- The new PNG golden is captured and reviewed, existing goldens are unchanged, and
  `just check-full` passes.
