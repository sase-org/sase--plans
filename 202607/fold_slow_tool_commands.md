---
tier: tale
title: Fold Slow Tool Call Commands Behind a Positional Detail Ladder
goal: The SLOW TOOL CALLS section keeps a compact one-line digest per call at fold
  level 1 and reveals the full, indented, line-wrapped command plus richer call facts
  from level 2 up, on both ordinary agent lanes and agent family containers.
create_time: 2026-07-28 08:01:53
status: done
---

- **PROMPT:** [202607/prompts/fold_slow_tool_commands.md](prompts/fold_slow_tool_commands.md)

# Fold Slow Tool Call Commands Behind a Positional Detail Ladder

## Goal

Give the Agents-tab `SLOW TOOL CALLS` section a real fold ladder on every agent lane (ordinary agents and family
containers alike):

- **Level 1** stays a compact, perfectly aligned triage table: one line per slow call showing when it started, its
  status glyph, its tool name, a short target digest, its duration, and its running / did-not-complete state. The full
  command is folded away and a dim tail says so.
- **Level 2 and above** keep those rows and add an indented, properly line-wrapped detail block under each row: the
  complete command or target, when the call actually ended, its exit/outcome facts, and any error. The last level of the
  lane's scale adds output previews, subagent statistics, and a slowest-call share line.

Today the section is fold-inert on ordinary agent lanes and only has a heading-or-everything fold on family containers,
and its single target column crams a long command into 44 cells with an ugly `...[38 more]` tail. After this change the
section is a first-class fold owner with a genuinely useful level-2 view.

## Design Decisions and Assumptions

**Screenshot reference.** The prompt cited `.sase/home/tmp/screenshots/20260728_072534.png` as the target look for
level 1. That capture shows the metadata panel scrolled to `AGENT XPROMPT` / `AGENT PROMPT` with `[view: collapsed]`;
the `SLOW TOOL CALLS` section is not visible in it. This plan therefore treats the screenshot as evidence of the desired
_level-1 aesthetic_ — a clean, quiet, compact panel — and designs the section's level-1 form from that principle. If the
reviewer intended a specific rendering, the level-1 row grammar in [Level 1](#level-1--compact-triage-table) is the
piece to correct.

**Level 1 keeps a short digest instead of dropping the target column.** A bare `time · glyph · tool · duration` table
renders eight indistinguishable `Bash` rows and destroys the section's triage value. The user-visible contract is that
the _full command_ is folded, not that the call becomes anonymous. Level 1 therefore shows a deliberately short, cleanly
elided digest (agent-authored description, path tail, pattern, or command head), and level 2 shows the complete command.

**Positional, not level-name, semantics.** `NEIGHBORS` already resolves its content "by position, not by level name"
(`neighbor_entry_limit()` in `_agent_display_neighbors.py`, documented in `docs/ace.md`). The slow-call ladder follows
exactly the same rule, which is what makes "only show the command from level 2" true for both a two-position family
scale and a three-position agent scale:

| Lane kind                                 | Position 1 | Position 2 | Position 3 |
| ----------------------------------------- | ---------- | ---------- | ---------- |
| Ordinary agent (`AGENT_FOLD_SCALE`, 3)    | `COMPACT`  | `DETAIL`   | `FULL`     |
| Family container (`FAMILY_FOLD_SCALE`, 2) | `COMPACT`  | `FULL`     | —          |

**The section never collapses to a heading only.** Position 1 keeps its rows. The section is already capped at
`MAX_VISIBLE_SLOW_TOOL_CALLS = 8`, so a compact table is cheap, and hiding the rows entirely at the default panel level
(`FoldLevel.COLLAPSED`) would make slow calls invisible by default. This intentionally drops the current family-only
"collapsed shows heading only" behavior.

**Visual language is borrowed from the tools panel.** The expanded tools timeline (`_tools_panel_details.py`) already
renders per-call detail with a `   │` gutter, a dim italic label, and a six-space value indent. Reusing that grammar
makes the two ACE surfaces read as one system.

**Reflow happens in a Rich renderable, not by hard-wrapping logical text.** The metadata header already supports
width-aware sections through `AgentHeaderRenderable` plus `responsive_ranges` (`ResponsiveBeadSection`,
`ResponsivePlanSection`). The slow-call detail block joins that mechanism so long commands fold with a real hanging
indent at any panel width and in metadata zoom, while logical text keeps one unwrapped line per source line for search,
copy, and `E`.

**Out of scope.** The clan aggregate renderer (`_agent_display_clan_sections.py`), the tribe aggregate renderer
(`_agent_display_tribe.py`), and the tools panel itself keep their current slow-call presentation. No new file hints or
hint numbers are introduced. No new disk reads: every field used already lives on the `SlowToolSource` / `ToolCallEntry`
data the section receives.

## Rendering Contract

### Heading

Every lane renders one fold-glyph heading carrying the existing summary, with the stable `slow-tool-calls` section id:

```
▸ SLOW TOOL CALLS · ≥20s · 7 calls · 2 running · 3 agents
```

The glyph reflects the section's effective level within the lane's own scale. Family containers lose their current
`▾ SLOW TOOL CALLS · 7` count-only heading in favor of this richer shared form.

### Level 1 — compact triage table

One line per visible call, using the existing column grammar (timestamp, status glyph, optional hint marker, optional
source chip, `_TOOL_NAME_WIDTH` tool name, target column, right-aligned duration, running / did-not-complete state). The
only change is what fills the target column: the **digest** instead of `entry.compact_target`, elided at a token or path
boundary with a single `…` and never with `...[N more]`.

```
────────────────────────────────────────────────────
▸ SLOW TOOL CALLS · ≥20s · 4 calls · 1 running
  07:21:33  ✔  Bash          install dev dependencies              1m 04s
  07:23:02  ✔  Bash          just check                           12m 03s
  07:35:41  ⏳ Bash          pytest tests/ace/tui                   4m 12s ● running
  07:36:02  ✘  Read          prompt_panel/_agent_slow_tools.py         22s
  … full commands hidden · zz / za to show
```

The dim tail appears only when at least one visible row actually has hidden detail. The existing
`+ N more · press ] for the full tools timeline` overflow tail is unchanged and appears at every level.

### Digest rules

`_digest_target(entry)` is a pure function over `tool_input_summary`, resolved in this order and then whitespace-
collapsed and elided to the target column budget:

1. `file_path` / `path` → the last two path components, prefixed `…/` when shortened.
2. `url` → host plus the first path segment.
3. `pattern`, then `query` → the value.
4. `description` → the value (this is the agent's own summary of the call).
5. `command` → the first line, cut at the last token boundary that fits.
6. `subagent_type` → the value.
7. Otherwise → the existing `entry.compact_target`, with any inherited `...[N more]` marker stripped before elision.

The digest reads raw input values rather than `compact_target` precisely so it never inherits that marker, and the
elision helper is the single place that appends `…`.

### Levels 2+ — detail block

Each row is followed by an indented block. `DETAIL` emits the primary value block, the `ran` line, and `error` when
present. `FULL` adds `output`, `subagent`, and the share line.

```
▾ SLOW TOOL CALLS · ≥20s · 4 calls · 1 running
  07:23:02  ✔  Bash          just check                           12m 03s
    │ command
      just check --keep-going --dir /home/bryan/.local/state/sase/workspaces/sase-org/
      sase/sase_17 && echo done
    │ ran 07:23:02 → 07:35:05 · exit 0 · timeout 600s
    │ output  All checks passed · 22,871 passed, 7 skipped
    │ #1 slowest · 61% of slow time
  07:35:41  ⏳ Bash          pytest tests/ace/tui                   4m 12s ● running
    │ command
      pytest tests/ace/tui -x -q
    │ started 07:35:41 · running for 4m 12s
```

Block rules:

- The value-block label is the primary input key (`command`, `file path`, `url`, `query`, `pattern`, `prompt`), matching
  the tools panel's `_INPUT_PRIMARY_KEYS` ordering. A call with no primary input emits no value block.
- Multi-line commands keep their own line breaks; each source line is a separate logical line that folds independently.
- The value block is capped at six logical lines with a dim `… (+N more lines)` tail, matching
  `_append_multiline_detail(max_lines=6)`.
- `ran <start> → <end>` for completed calls; `started <start> · running for <duration>` for running ones;
  `started <start> · did not complete` for the did-not-complete case. Outcome facts (`exit N`, `failed`, `interrupted`)
  and `timeout Ns` (when the input recorded one) join that line with `·` separators.
- `error` renders one bounded line in the existing error red, from `entry.error` or the response summary's error.
- `output` (FULL only) renders the first meaningful line of the first present preview key, using the tools panel's
  preview key order.
- `subagent` (FULL only) renders agent type, tool-use count, and token count for subagent calls.
- The share line (FULL only) renders `#N slowest · P% of slow time`, computed only across the selected calls already in
  hand.
- Every wrapped continuation line is indented under the value column; no continuation line ever starts at column 0.

## Implementation

1. **Add the shared tool-detail visual language.**
   - Create `src/sase/ace/tui/widgets/_tool_detail_language.py` exporting the gutter and wrap-indent constants plus the
     small label/value line builders currently private to `_tools_panel_details.py` (`_EXPANDED_GUTTER`,
     `_EXPANDED_WRAP_INDENT`, and the single-line and multi-line detail line shapes).
   - Update `src/sase/ace/tui/widgets/_tools_panel_details.py` to consume them so both surfaces share one definition and
     Symvision sees two real non-test consumers. Do not change any tools-panel output.

2. **Add the positional detail ladder and the digest.**
   - Create `src/sase/ace/tui/widgets/prompt_panel/_agent_slow_tools_detail.py` with:
     - a `SlowToolDetail` enum (`COMPACT`, `DETAIL`, `FULL`);
     - `slow_tool_detail_level(level, scale)` resolving position 1 → `COMPACT`, last position → `FULL`, any middle
       position → `DETAIL`, mirroring `neighbor_entry_limit()` and using `fold_scale_position()`;
     - the digest function and its elision helper;
     - a prepared per-row detail model (primary label and value lines, `ran`/outcome line, error, preview, subagent,
       share) built once from data already loaded.
   - Keep every function pure and free of disk access; the render path must not stat, glob, or parse files.

3. **Add the responsive section renderable.**
   - In the same module, add `ResponsiveSlowToolCallsSection`, following `ResponsiveBeadSection`: a `logical_text`
     property that reproduces exactly the unwrapped text spliced into the header, and a `__rich_console__` that reflows
     the detail blocks to `options.max_width` with the hanging indent contract above.
   - Widen the `AgentHeaderRenderable` section union in
     `src/sase/ace/tui/widgets/prompt_panel/_agent_display_header_renderable.py` to include it.
   - Only register a responsive range when the resolved detail level is above `COMPACT`, so the default level adds no
     renderable and no measurement cost.

4. **Rework the section renderer.**
   - In `src/sase/ace/tui/widgets/prompt_panel/_agent_slow_tools.py`, replace the `fold_level: FoldLevel | None`
     parameter with the neighbors-style trio (`panel_level`, `scale`, `section_fold_overrides`) plus a
     `responsive_ranges` output mapping, and resolve the section's own effective level from the `slow-tool-calls`
     override against the lane scale.
   - Render the shared fold-glyph heading, the compact rows with digests, the level-1 hidden-detail tail, the detail
     blocks for levels above `COMPACT`, and the unchanged overflow tail.
   - Preserve selection, ordering, the 8-row cap, source chips, hint-marker registration and widths, and tool-call
     report specs exactly as they are today; hint numbering must not change across detail levels.
   - Keep a no-fold-owner entry point for `src/sase/ace/tui/widgets/prompt_panel/_workflow_render.py`, whose aggregate
     rows own no fold scale: those render the `DETAIL` tier so the command stays visible where it cannot be unfolded.

5. **Generalize the fold heading helper.**
   - Add a scale-aware `append_fold_section_heading()` to `src/sase/ace/tui/widgets/prompt_panel/_fold_language.py`, and
     make `append_family_fold_heading()` in `_agent_display_family.py` delegate to it with `FAMILY_FOLD_SCALE`. No other
     family heading behavior changes.

6. **Give ordinary agent lanes a real lane fold level.**
   - In `src/sase/ace/tui/widgets/prompt_panel/_agent_display_header.py`, stop gating the slow-call section on
     `family_fold_enabled`; always pass the lane's resolved level, `lane_fold_scale(agent)`, and the lane overrides, and
     collect its responsive range alongside `BEAD` and `PLAN`.
   - In `_agent_display_render.py`, `_agent_display_hints.py`, and `_agent_display.py`, pass `lane_fold_level` and
     `lane_section_fold_overrides` unconditionally rather than only when `lane_summary_enabled`. Before doing so, verify
     that nothing else keys off `lane_fold_level is None`: `family_fold_enabled` already requires
     `is_family_container_row`, the family roster and header-owned sections are gated on it, and `lane_neighbors` is
     gated on `agent_owns_lane()`. Keep `member_jump_map_publisher` gated exactly as it is today.

7. **Make fold-scope feedback honest.**
   - `_notify_lane_fold_scope()` in `src/sase/ace/tui/actions/navigation/_fold.py` currently tells a neighborless
     single-agent lane that fold levels only shape clan, family, and neighbor summaries. Suppress that notice when the
     selected lane has qualifying slow calls, and update its wording to include slow-call summaries.
   - `za` / `zA` already reach `slow-tool-calls` through `_current_agent_metadata_section_id()`; no change is needed
     there beyond confirming the section id stays stable and stays out of `_FOLD_INERT_AGENT_SECTION_IDS`.

8. **Update user documentation.**
   - `docs/ace.md`: rewrite the **Slow tool calls** bullet to describe the positional ladder, the level-1 digest and
     hidden-command tail, the wrapped level-2 command block, and the extra facts. Correct the fold section's claim that
     "Every other section on a regular-agent panel stays fold-inert" — a regular-agent lane now owns `NEIGHBORS` and
     `SLOW TOOL CALLS`. Keep the `z*` key table unchanged.
   - `docs/agent_families.md`: note that a family container's slow-call section shows digests at level 1 and full call
     detail at level 2.

## Validation

Run from the repository workspace after `just install`.

1. Focused functional tests:

   ```bash
   pytest \
     tests/ace/tui/widgets/test_agent_slow_tools.py \
     tests/ace/tui/widgets/test_summary_fold_contracts.py \
     tests/ace/tui/test_agents_panel_fold_mode.py \
     tests/ace/tui/widgets/test_prompt_panel_header.py \
     tests/ace/tui/widgets/test_agent_display_workflow_async.py \
     tests/ace/tui/widgets/test_agent_display_clan_sections.py \
     tests/ace/tui/widgets/test_agent_display_tribe.py \
     tests/ace/tui/widgets/test_tools_panel_timeline.py
   ```

   Extend `tests/ace/tui/widgets/test_agent_slow_tools.py` to cover: digest resolution per input-key kind and its
   token-boundary elision; `slow_tool_detail_level()` at every position of both `AGENT_FOLD_SCALE` and
   `FAMILY_FOLD_SCALE`; the command block absent at position 1 and present from position 2; the level-1 hidden-detail
   tail appearing only when detail exists; multi-line commands preserved and capped with the `+N more lines` tail; `ran`
   / `started` / did-not-complete variants; error, preview, subagent, and share lines gated to their tiers; hint markers
   and report specs identical across all three tiers; per-section `slow-tool-calls` overrides beating the panel level.
   Keep the existing `assert_logical_section_is_compact` / `assert_rendered_section_is_compact` assertions green, and
   add a rendered-width assertion at widths 60 and 120 proving no wrapped continuation line starts at column 0.

   In `tests/ace/tui/widgets/test_summary_fold_contracts.py`, add the slow-call ladder to the agent and family cases so
   the cross-kind contract covers a section that now differs between adjacent positions on both scales.

   In `tests/ace/tui/test_agents_panel_fold_mode.py`, prove `za` / `zA` on `slow-tool-calls` change an _ordinary agent_
   lane's rendering, that a panel-level cycle clears that override, and that the neighborless-lane notice is suppressed
   when the lane has slow calls.

2. Dedicated visual coverage. Add `tests/ace/tui/visual/test_ace_png_snapshots_agents_slow_tools.py` with a fixture
   whose agent has a long multi-line `Bash` command, a running call, and a failed call, and capture
   `agents_slow_tool_calls_level_1_120x40.png` and `agents_slow_tool_calls_level_2_120x40.png`. Inspect the generated
   actual/expected/diff artifacts under `.pytest_cache/sase-visual/` before accepting any golden, and accept only
   goldens caused by this change:

   ```bash
   just test-visual
   ```

3. TUI performance floor — the section now builds a renderable and more per-row text, so confirm the hint scan cap and
   the off-keypress hint bar still hold and that no committed baseline is refreshed:

   ```bash
   just view-hints-perf-check
   ```

4. Full required repository gate:

   ```bash
   just check
   ```

   `just _lint-symvision` must pass without new pragmas: the promoted tool-detail language needs both the tools panel
   and the slow-call section as real consumers, and any helper that ends up used only inside its own module must be
   private.

Manual acceptance in `sase ace`:

- Select an ordinary agent with several slow calls. At level 1 the rows are aligned, no full command is visible, and the
  hidden-detail tail is present. Press `zz` (or `z2`): every row gains its wrapped command block with a hanging indent.
  Press `zz` again for level 3 and confirm previews, subagent stats, and the share line appear.
- Repeat on a real family container: level 1 is the compact table and level 2 shows the full detail block.
- With one heading at the top of the metadata viewport, `za` / `zA` cycle only the slow-call section, and a panel-level
  chord clears that override.
- Narrow the terminal and zoom the metadata panel (`Z`): commands reflow at both widths with no continuation line at
  column 0 and no horizontal overflow.
- Enter hint mode at both levels: the numbered tool-call report markers are unchanged and open the same reports.
- Press `]` from the section and confirm the overflow pointer to the tools timeline still resolves.
