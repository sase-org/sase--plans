---
tier: tale
title: Give the Agents-header runner limit a distinct semantic color
goal: 'The configured maximum in the Agents-tab N/M running count uses the established
  gold runner-limit color, is visually distinct from the cyan done-agent count, and
  preserves all existing count, queue, and render-cache behavior.

  '
create_time: 2026-07-24 18:31:14
status: done
---

- **PROMPT:** [prompts/202607/agent_runner_limit_color.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/agent_runner_limit_color.md)
- **AGENTS:**
  - [bbugyi200.athena.jq.f0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.jq.f0/README.md)
  - [bbugyi200.athena.jq.f0--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.jq.f0.md#member-code)
- **COMMITS:**
  - [f943218](https://github.com/sase-org/sase/commit/f943218ef38a0eb9a42f7630ddec8ff3f4624394) — fix(tui): highlight agent runner limit

# Plan: Give the Agents-header runner limit a distinct semantic color

## Context

The consolidated Agents-tab status strip renders the visible running count and configured maximum as `N/M running`. The
maximum `M` currently uses bold `#87D7FF`, while the `done` count uses bold `#5FD7FF`. Although the hex values differ,
both render as bright cyan in the dense one-line header, so the capacity denominator reads like another completed-agent
metric instead of a separate configured limit.

The Statistics pane already gives the current global runner limit a gold (`#FFD700`) treatment in its summary, timeline
legend, and occupancy comparisons. Reusing that established meaning in the Agents header makes `M` immediately distinct
from the cyan `done` metric without inventing another palette entry. Gold also remains coherent when the running
numerator turns gold at exact capacity: `N/M` then reads as a deliberately emphasized full-capacity pair. The red
over-limit numerator behavior remains unchanged.

## Implementation

1. In `src/sase/ace/tui/widgets/agent_info_panel.py`, give the runner-limit denominator an explicit, named bold-gold
   style and use it only for `M` in `_append_status_strip()`.
   - Keep the slash, `running` label, separators, and brackets dim.
   - Keep the running numerator's below-limit green, at-limit gold, and over-limit red behavior unchanged.
   - Keep queued and all other status-count colors unchanged, especially the cyan `done` count.
   - Do not alter the underlying runner limit/count values, status-strip wording, queue visibility, stable-state cache,
     countdown fast path, or any loading/refresh flow; this is a presentation-only change.

2. Update `tests/ace/tui/widgets/test_agent_info_panel.py` so the Rich-style coverage asserts that the denominator is
   bold `#FFD700` in below-, at-, and over-limit states and is distinct from the bold `#5FD7FF` style used by the `done`
   count. Preserve the existing assertions for numerator threshold colors, dim chrome, positive queues, and zero-queue
   omission.

3. Regenerate only the ACE PNG goldens whose rendered Agents header contains the runner-limit denominator. Inspect the
   representative ordinary Agents-list and runner-queue snapshots, then audit the changed-image diff bounds/colors to
   confirm that every accepted pixel change is confined to the denominator glyphs in the Agents header. Reject and
   investigate any list, detail-panel, modal-body, footer, layout, or text changes.

## Validation

- Run the focused `AgentInfoPanel` widget tests.
- Run the representative Agents-list and runner-slot-wait PNG snapshot tests before accepting goldens, then regenerate
  the intentional snapshots and run the complete visual suite (`just test-visual`) against the accepted files.
- Run `just install` first as required for an ephemeral SASE workspace, then finish with the mandatory `just check`. If
  an external/global generated-file drift gate fails for reasons unrelated to this change, report it precisely and
  independently run the remaining in-repository formatting, lint, unit, and visual gates.

## Non-goals

- Do not change how running, waiting, queued, failed, unread, done, or lane totals are calculated.
- Do not change the configured maximum, runner-slot scheduling, queue eligibility, capacity thresholds, header text, or
  layout.
- Do not recolor other uses of cyan or gold elsewhere in ACE.
