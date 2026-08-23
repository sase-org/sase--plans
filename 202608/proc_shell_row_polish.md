---
tier: tale
title: Beautiful stand-alone proc shell rows and details in the Agents tab
goal:
  Stand-alone `%proc` rows read as a gear plus the thing that actually ran, synthesized
  labels and meaningless runtime identifiers never reach the screen, and the proc detail
  panel leads with the command and its output instead of six lines of hashes.
size: medium
proposed_by: bbugyi200.athena.0by
---

- **AGENTS:**
  - [bbugyi200.athena.0by](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0by.md)
- **COMMITS:**
  - [e02106a](https://github.com/sase-org/sase/commit/e02106a29f12a8962b75678d4aa84d8f91e4dcb1)
    — feat(ace): polish proc shell rows

# Plan: Beautiful stand-alone proc shell rows and details

## Background

`sase-s6.7` shipped stand-alone `%proc` launch units as top-level Agents-tab rows backed
only by the proc store. The presentation that landed is functional but noisy. A finished
proc renders today as:

```
▣ unit-1 (DONE) · running [bash] 92vnpc            13:31:13 · 31s
```

Four separate problems in one row:

1. `▣` (U+25A3) is a generic blue box that says nothing about what a proc is.
2. `unit-1` is not a name the user chose. It is `LaunchUnitWire.logical_id`, a
   planner-internal ordinal that `src/sase/agent/launch_proc_runtime.py:262` uses as a
   last-resort label: `label=payload.label or payload.shell_name or unit.logical_id`.
3. `· running` is `Proc.phase`, the _dispatch_ phase. Nothing sets it back after
   `update_proc(proc.proc_id, ..., phase="running")` at
   `src/sase/agent/launch_proc_runtime.py:194`, so a `DONE (success)` proc permanently
   claims it is `running`. It is not merely noise, it is wrong.
4. `92vnpc` is `proc_id[:6]`, an opaque runtime handle. The addressing affordance the
   user actually needs (`sase proc show 92vnpc --follow`) already lives in the detail
   panel.

The right-hand detail panel has the mirror-image problem: `Digest`, `Origin`,
`Supervisor`, `Settlement`, and `Fingerprint` occupy five of the first nine lines,
`Command:` shows the internal `/bin/bash --noprofile --norc .../script.sh` wrapper
rather than what the user wrote, and the thing the user wrote is pushed below the fold
under the jargon heading `SAFE PREVIEW`.

Stand-alone procs are gated behind the `typed_launch_units` beta flag (`default=off`),
so this presentation change rides that existing flag. Do **not** create a new feature
flag for it.

## Design

Three requested changes, plus the two collisions they imply and the one thing that has
to replace the removed text.

### D1. The gear is ACE's mark for a supervised OS process

Set `_PROC_SHELL_GLYPH = "⚙"` (U+2699 GEAR).

This deliberately reuses the glyph `src/sase/monitor_state.py:9` already assigns to sase
monitors, and that reuse is the point rather than an accident to work around:

- A monitor and a stand-alone proc are the _same substrate_. Both are
  `PROC_LIFECYCLE_PROC_SHELL` rows in the proc store
  (`src/sase/monitor/proc_adapter.py:76`, `src/sase/agent/launch_proc_runtime.py:72`),
  both are supervisor-owned, and both appear on the Admin Center Procs tab. They differ
  only in _who asked_: a monitor is owned by an agent, a stand-alone proc is owned by
  nobody.
- That ownership difference is already carried structurally and cannot be confused. A
  monitor gear only ever renders on a tree child row (`tree_depth > 0`, or the
  `agent.is_monitor` branch of an indented row) in
  `src/sase/ace/tui/widgets/_agent_list_render_agent.py:236-243`. A stand-alone proc
  gear only ever renders on a top-level root row. The two branches are mutually
  exclusive today and must stay that way.
- A distinct gear codepoint is not an option. U+26ED GEAR WITHOUT HUB and U+26EE GEAR
  WITH HANDLES are carried by **none** of the four fonts under
  `tests/ace/tui/visual/fonts/`, so either would rasterize as a `.notdef` box in every
  PNG golden — exactly the failure `tests/ace/tui/visual/test_emoji_glyphs.py` exists to
  prevent. U+2699 is covered by `NotoEmoji-Regular.ttf` and `DejaVuSans.ttf` and is
  already audited, so this change needs no font work.

**Hue rule: a row's gear always matches the hue of the chip that counts that row.**
Monitors keep amber `#FFAF5F` when live and grey `#9E9E9E` when settled, because their
chips split by liveness. Stand-alone procs keep proc-cyan `#5FD7FF` in all states,
because they have exactly one cyan chip. Do not make the proc gear status-hued: it would
desync the row from its chip, and the status word one cell to the right already carries
liveness in full color.

### D2. Row identity: the explicit label, otherwise the command

Suppressing the synthesized `unit-1` leaves an unlabeled proc with no identity at all,
and a row that reads `⚙ (DONE) [bash]` is worse than the row we started with. The
identity of an unlabeled process is _what it runs_ — the same convention `htop`, `just`,
and every CI log already use. So:

- **Explicit label present** → render it as the row title, in proc-cyan, exactly as
  today.
- **No explicit label** → render `❯ <compact command>`, where the command is derived
  from `agent.proc_safe_preview`.

The `❯` prefix is `_STEP_RUN_GLYPH` from
`src/sase/ace/tui/widgets/_agent_list_styling.py:110`, already used for bash/python
workflow steps and already covered by `DejaVuSans.ttf`. Reusing it is semantically exact
("this is a command, not a name") and gives a **non-color** cue that separates "this
proc has no name" from "this proc is named `just check`". That distinction is a
reliability requirement, not decoration: without it the user cannot tell a derived title
from a chosen one.

Compaction rules for the derived title (pure function, unit-tested, no Textual):

- Take the first non-blank line of `proc_safe_preview`.
- Collapse runs of whitespace to a single space; strip a trailing `\` continuation.
- Truncate to a 48-cell budget with a trailing `…` (U+2026, covered by Fira Code).
- If the preview had more than one non-blank line, append `…` even when the first line
  fit, so a multi-line body never masquerades as a one-liner.
- If `proc_safe_preview` is empty or absent, render no title at all rather than
  inventing one. A pending proc whose preview has not been written yet gets
  `⚙ (STARTING) [bash]`, which is honest.

### D3. Row grammar

```
⚙ <title> (<STATUS>[ badges]) [<language>]              <right suffix unchanged>
```

Concretely, for the two cases:

```
⚙ verify (RUNNING) [bash]                               12:28:25 · 1m35s
⚙ ❯ echo hello && sleep 30 && echo world (DONE) [bash]  13:31:13 · 31s
```

- `· <phase>` is **removed** from the row entirely (see D5 for where phase survives).
- `proc_id[:6]` is **removed** from the trailing identity slot.
- `[<language>]` moves out of the middle of the sentence and into the trailing identity
  slot that `proc_id[:6]` used to own, so proc rows keep the same column rhythm as agent
  rows (which put their `%id` annotation there). Keep the square brackets — they
  visually fence the badge off from free-form command text, which now sits immediately
  to its left. Style stays `_PROC_SHELL_ID_STYLE` (dim cyan); retire or repoint
  `_PROC_SHELL_LANGUAGE_STYLE` and delete `_PROC_SHELL_PHASE_STYLE`, which becomes
  unused (symvision will flag it otherwise).

### D4. Explicit vs synthesized labels, decided at the source

Do not infer explicitness in the TUI by pattern-matching `unit-\d+`. Record it where the
truth is known.

In `_submit_unit` (`src/sase/agent/launch_proc_runtime.py`), keep
`ProcSubmitRequest.label` exactly as it is — `sase proc list` and the CLI still need a
non-empty display name — and additionally record the _provenance_ in the `xprompt_proc`
sidecar meta:

```python
"label": payload.label or None,
"shell_name": payload.shell_name or None,
```

**Trap:** `run_xprompt_proc` rebuilds `xprompt_meta` from scratch around
`src/sase/agent/launch_proc_runtime.py:176-190` and writes it back with
`update_proc(..., xprompt_proc=xprompt_meta)`. Both new keys must be carried forward
there or they silently vanish the moment the proc leaves the preparing phase, and every
real proc would fall through to the compatibility branch below. Add a regression test
that reads the meta _after_ the preparing-phase rewrite, not just after submit.

Then in `_observed_proc_to_agent` (`src/sase/ace/tui/models/agent_proc_shells.py:92`),
resolve `proc_label` to an explicit label or to `None`:

1. `meta["label"]` → explicit (`%proc(label="…")`).
2. else `row.shell_name` → explicit (a named `%proc` shell; the user chose that name).
3. else `row.display_name` when it differs from `meta["logical_id"]` → explicit. This is
   the compatibility branch for procs submitted before this change; keep it and comment
   it as such.
4. else `None`.

Everything that needs a guaranteed-non-empty name keeps working through the existing
fallback chain, because `agent_name` is still `shell_name or short_proc_id(proc_id)`:
`Agent.display_name` (`src/sase/ace/tui/models/agent.py:230-237`) degrades to the short
proc id, which is the right value for the panel `Name:` header and for the kill modal at
`src/sase/ace/tui/actions/agents/_monitor_stop_flow.py:41`.

That fallback is also why the row must **not** use `agent_tree_title` unchanged: it
returns `display_name`, which would put the short proc id back on the row we just
removed it from. Add a proc-shell branch to `agent_tree_title`
(`src/sase/ace/tui/models/_agent_tree.py:91`) that returns the explicit label, else the
derived command title, else `None`, so every caller stays consistent.

### D5. Detail panel: lead with what ran

`src/sase/ace/tui/widgets/prompt_panel/_agent_proc_shell_section.py`.

`PROC DETAILS` keeps the essentials at every fold level:

- `Status:` (unchanged, keep the exit-code badge)
- `Language:`
- `Phase:` — **only while `proc_status` is non-terminal** (`pending`, `running`,
  `settling`). A terminal proc must never display a stale `running` phase. This is the
  correctness half of removing phase from the row.
- `Timeout:` / `Elapsed:` / `Idle timeout:` (unchanged)
- `Waits:` / `Condition:` (unchanged — real information)
- `Proc id:` plus the `sase proc show <short> --follow` hint (unchanged; the row gave
  this up, so the panel must keep it)
- `Log path:` (unchanged)
- Drop `Label:` outright. When a label is explicit it is already the `Name:` in the
  header block; when it is not, there is nothing to show.

The diagnostics — `Origin`, `Digest`, `Fingerprint`, `Supervisor`, `Settlement`, and the
runtime argv — move behind the fold **inside the existing section**. The call site at
`src/sase/ace/tui/widgets/prompt_panel/_agent_display_hint_render.py:314` already
resolves a per-section fold level via
`lane_fold_overrides.get(PROC_SHELL_SECTION_ID, lane_fold_level)`, so gate them on
`effective_fold_level(panel_level, scale) >= FoldLevel.FULLY_EXPANDED`. No new section
id, no new fold machinery. When they are hidden, pass `summary="+N diagnostics"` to
`append_fold_section_heading` so the heading advertises what the fold is holding.

Rename the current `Command:` field to `Runtime argv:` and move it into that gated
group. It renders `/bin/bash --noprofile --norc .../script.sh`, which is the wrapper
this codebase spawned, not the command the user wrote — presenting it as `Command:`
directly above a separate `SAFE PREVIEW` block containing the real command is the single
most confusing thing in the current panel.

Finally, rename the `SAFE PREVIEW` heading to `COMMAND` and render it **above**
`PROC DETAILS`. Behavior is unchanged (still bounded by `_PROC_PREVIEW_MAX_CHARS`, still
redacted by `_SENSITIVE_LINE_RE`, still syntax-highlighted via `lazy_renderable`); only
the heading text and the section order change. `SAFE PREVIEW` describes the
implementation; `COMMAND` describes what the user is looking at. Leave `LOG TAIL` alone
— it is accurate and churning it buys nothing.

Resulting default panel order: header block → `COMMAND` → `PROC DETAILS` → `LOG TAIL`.

### D6. Count chips must agree with the row

`src/sase/ace/tui/widgets/_agent_list_styling.py` states the governing invariant in
`_monitor_glyph_style`'s docstring: a grey gear on a row and the grey count it feeds can
never disagree. Changing the row glyph therefore forces the chip.

Set `_PANEL_PROC_GLYPH = "⚙"` in
`src/sase/ace/tui/actions/agents/_display_panel_titles.py:52` and keep
`_PANEL_PROC_STYLE = "bold #5FD7FF"`. Keep it as its own chip — do **not** fold
stand-alone procs into the monitor lane counts. Proc rows already contribute to the
panel lane total and to `sase_agent_status_counts` (a proc is one of the `D15` in
`@default · 16 [R1 D15] ⚙1 ⊙10 ▣1`), so folding them into `⚙N` would double-count them
in the same title.

That leaves three gear chips that can co-occur: amber (live monitors), grey (settled
monitors), cyan (stand-alone procs). The two monitor chips already differ by hue alone
today, with an explicit rationale in `_MONITOR_SETTLED_COUNT_GLYPH_STYLE`, so this adds
no new _class_ of ambiguity — and the new pair is the one that separates best:
amber/grey against cyan is a warm-neutral-vs-cool split that survives deutan and protan
color vision deficiency, unlike a red/green pair. Render the chips in a fixed order —
procs, then running monitors, then settled monitors — and pin that composition in a test
so it cannot drift.

`agent_info_panel.py` needs no change: it renders the word `procs`, not a glyph.

## Files

| Path                                                                  | Change                                                                                                                           |
| --------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `src/sase/ace/tui/widgets/_agent_list_styling.py`                     | `_PROC_SHELL_GLYPH` → `⚙`; drop `_PROC_SHELL_PHASE_STYLE`; repoint the language badge style; keep `_TYPE_GLYPHS["proc"]` in sync |
| `src/sase/ace/tui/widgets/_agent_list_render_agent.py`                | Remove the phase and proc-id spans; move `[lang]` to the trailing identity slot; render the `❯` derived-title prefix             |
| `src/sase/ace/tui/widgets/_agent_list_render_cache.py`                | **Required:** add the derived-title inputs to `agent_render_key`                                                                 |
| `src/sase/ace/tui/models/_agent_tree.py`                              | Proc-shell branch in `agent_tree_title`                                                                                          |
| `src/sase/ace/tui/models/agent_proc_shells.py`                        | Explicit-label resolution; derived-command-title helper; keep `proc_shell_agent_signature` complete                              |
| `src/sase/agent/launch_proc_runtime.py`                               | Record `label`/`shell_name` provenance in `xprompt_proc`; carry both through the preparing-phase meta rewrite                    |
| `src/sase/ace/tui/widgets/prompt_panel/_agent_proc_shell_section.py`  | Field pruning, phase gating, fold-gated diagnostics, `Runtime argv:`, `COMMAND` heading                                          |
| `src/sase/ace/tui/widgets/prompt_panel/_agent_display_hint_render.py` | Emit `COMMAND` before `PROC DETAILS`                                                                                             |
| `src/sase/ace/tui/actions/agents/_display_panel_titles.py`            | `_PANEL_PROC_GLYPH` → `⚙`; fixed chip order                                                                                      |

Two cache/consistency traps to honor, both already documented in-tree:

- `agent_render_key` is deliberately explicit rather than `vars(agent)` "so adding a new
  visible field is a deliberate edit here rather than a silent cache desync". The row
  now displays content derived from `proc_safe_preview`; if that input is not in the
  key, rows render stale.
- `proc_shell_agent_signature` already lists `proc_safe_preview` and `proc_label`, so
  the projection side is covered — verify rather than assume, and extend it if the
  implementation introduces any new displayed field.

## Verification

Run `just install` first; these are ephemeral workspaces and dependencies drift.

New and updated tests:

- `tests/ace/tui/models/test_agent_proc_shells.py` — explicit label via `meta["label"]`,
  via `shell_name`, the pre-change compatibility branch, and a synthesized `logical_id`
  resolving to `proc_label is None`. Command-title derivation: whitespace collapse,
  first-line selection, 48-cell truncation, the multi-line `…` suffix, and empty-preview
  → no title.
- `tests/test_launch_proc_runtime.py` — `xprompt_proc` carries `label` and `shell_name`
  after submit **and** after the preparing-phase `update_proc` rewrite.
- A row-composition test over `format_agent_option` for a proc shell: gear present, no
  phase text, no `proc_id[:6]`, `[bash]` present in the trailing slot, explicit label
  rendered bare, synthesized label rendered as `❯ <command>`.
- A detail-section test: `Phase:` present while running and absent when terminal;
  `Label:` gone; diagnostics hidden at the default fold level and present when fully
  expanded; heading carries the `+N diagnostics` summary; `COMMAND` precedes
  `PROC DETAILS`.
- `tests/ace/tui/actions/` (or the existing panel-title test module) — chip order and
  the cyan gear chip.

Visual goldens (`tests/ace/tui/visual/`):

- Update the literal assertions in `test_ace_png_snapshots_agents_proc_shells.py`:
  `assert_page_svg_contains(page, "▣")` becomes `"⚙"`, and add an assertion for a
  derived `❯` title.
- `_proc_shell_index` matches on `display_name`; for an unlabeled fixture that is now
  the short proc id, so update the lookup accordingly.
- Extend `_ace_agents_proc_shell_png_fixtures.py` with the two cases the goldens do not
  cover today, since every current fixture has a human label: one proc whose
  `display_name` equals its `xprompt_proc["logical_id"]` (the `unit-1` case from the
  report), and one whose preview is long and multi-line (the truncation case). Bump the
  `assert len(proc_rows) == 5` count in `_assert_procs_are_top_level_rows`.
- Regenerate with `just test-visual --sase-update-visual-snapshots` and **look at the
  resulting PNGs** before accepting them. Inspect `.pytest_cache/sase-visual/` on any
  failure. This is a visual change; the goldens are the actual deliverable review
  surface.

Gates: `just check` while iterating. Because this touches the ACE render path and the
visual suite, finish with `just check-full` through `/sase_monitor`, never inline:

```bash
sase monitor start --command 'just check-full' \
  --start-status TESTING --stop-status TESTED --next '...'
```

Also run `just _lint-symvision` deliberately — this change deletes
`_PROC_SHELL_PHASE_STYLE` and may strand other private helpers; see
`sase/memory/symvision.md` via `/sase_memory_read` if it reports unused symbols.

## Out of scope

- No new feature flag. Stand-alone procs already ride `typed_launch_units` (beta,
  `default=off`).
- No memory-file edits. `sase/memory/*.md`, `AGENTS.md`, and the generated provider
  shims stay untouched unless the user asks in that conversation.
- The stale `Proc.phase` value itself is not fixed at the source. This plan stops
  _displaying_ a terminal proc's stale phase; making settlement write a terminal phase
  is a separate concern in `src/sase/procs/settlement.py` and should be filed as a task
  bead rather than folded in here.
- Monitor row and monitor chip presentation are unchanged.
- `LOG TAIL` is unchanged.
