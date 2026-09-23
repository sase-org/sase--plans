---
tier: tale
title: Add an Agent Relation Jump Target glossary strand
goal:
  The glossary defines "agent relation jump target" (alias "agent jump target") as the
  numbered Agents-tab roster rows that numeric keymaps jump to, clearly separated from
  every other SASE use of "jump target", and the regenerated instructions list it.
size: small
proposed_by: bbugyi200.athena.0qa.f0--plan-0
create_time: 2026-09-23 15:54:08
status: wip
---

# Plan: Add an `Agent Relation Jump Target` glossary strand

## Context

On 2026-09-23 the user told agent `0qa--plan` that "the agent metadata panel sections
that currently contain 'jump targets' (i.e. the listings of the nodes that are targeted
by the numeric keymaps) were supposed to be MOVED to the new sticky footer" (epic
`sase-16y`). The epic's implementer had misread that request. The glossary has no term
for it.

The first version of this plan (`jump_target_glossary_strand.md`) proposed the keyword
`Jump Target`. The reviewer rejected it with this feedback: "can we go with the term
'agent relation jump targets' (aka 'agent jump targets') instead to avoid the ambiguity
with the other uses of the term 'jump target'?" This plan keeps that plan's research and
changes the term. The keyword is `Agent Relation Jump Target`, and its only alias is
`agent jump target`. It deliberately has no bare `jump target` alias.

### What the user means, from research

**The user's own prompts:**

- `dz` (2026-07-18) introduced numbered roster entries: "a single number starting at 0
  and ending at 9", `00`–`99` past ten, "no more than 100". Each number's keymap
  "navigates to the corresponding Agent/Agent Family, expanding the containing clan or
  family if necessary".
- `sase-6w.land.w4` extended this to `TRIBE MEMBERS`.
- `kj` (2026-07-25) added `NEIGHBORS`, each with "a numeric keymap listed to the left of
  it".
- `0ap` (2026-08-22) renamed `FAMILY MEMBERS` to `FAMILY SHELLS` and numbered monitor
  shells too.
- `0q0` (2026-09-23), which created `sase-16y`, speaks of "all of the sections that
  contain references associated with numeric keymaps".
- `0qa` is the only prompt where the user writes "jump targets".

**Why "agent relation":** each of the four numbered rosters lists the selection's
relatives through one agent relation that already has a glossary term:

| Roster          | Relation                     |
| --------------- | ---------------------------- |
| `FAMILY SHELLS` | agent family                 |
| `NEIGHBORS`     | agent hood (agent neighbors) |
| `CLAN MEMBERS`  | agent clan                   |
| `TRIBE MEMBERS` | agent tribe                  |

**How the code behaves** (verified at `f456a8b3b`):

- **Rosters.** `append_member_roster()` in
  `src/sase/ace/tui/widgets/prompt_panel/_member_roster.py` has exactly four callers:
  `_agent_display_family.py` (`FAMILY SHELLS`), `_agent_display_neighbors.py`
  (`NEIGHBORS`), `_agent_display_clan.py` (`CLAN MEMBERS`), and
  `_agent_display_tribe.py` (`TRIBE MEMBERS`). Each numbered row publishes one
  `_MemberJumpTarget` into a `MemberJumpMap`. Target roles are `member`, `neighbor`, and
  `dismissed`.
- **Numbering.** `MemberJumpNumbering` is the "digit allocator shared by every numbered
  roster in one panel document". Its width is one digit for up to 10 targets and two
  digits (`00`–`99`) beyond that. `MEMBER_ROSTER_LIMIT = 100` caps the total.
- **Rows that are not targets.** These rows get no number:
  - rows folded behind a `… +N more` tail;
  - unnumbered child rows;
  - `… also listed under FAMILY SHELLS` duplicates in `NEIGHBORS`.

  Tribe `CLAN SUMMARIES`/`PROMPTS` chips reuse a member's digit as a label. They are not
  new targets.

- **Jumping.** `src/sase/ace/tui/actions/navigation/_member_jump.py` handles a digit
  press:
  - It selects the target and reveals it through any folds.
  - It revives a `dismissed` target instead of selecting it
    (`_activate_member_jump_target`).
- **Jump panel.** `AgentJumpPanel` (`src/sase/ace/tui/widgets/agent_jump_panel.py`) is
  the sticky footer that lists every live numbered target. The approved plan
  `jump_panel_roster_move.md` (being coded by `0qa--code`) moves the full roster
  sections into that panel. The strand below is worded so that it is true both before
  and after that move.

**Other uses of "jump target" that the new term must exclude:**

- `'` entry hints (`_entry_jump_mode.py`). Their target type in
  `src/sase/ace/tui/actions/navigation/jump_hints.py` is literally named
  `AgentJumpTarget`, which collides with the new alias.
- `Ctrl+J`/`Ctrl+K` metadata section stops (archived plan
  `metadata_section_jump_targets.md`).
- `,j`/`,J` unread and stopped-agent jumps (`TimedAgentJumpCandidate`, a "stable jump
  target").
- `Ctrl+]` prompt-definition jumps (`docs/ace.md`: "several jump targets").
- The Admin Center previous-section footer and the `,L` error-log jump
  (`docs/configuration.md`).
- Separately, the Artifacts tab's `RelationPanel` `<`/`>` keys follow artifact
  relations. The word "relation" in the new term could suggest that panel.

## Implementation

1. Create `sase/memory/glossary/agent-relation-jump-target.md` with this content:

   ```markdown
   ---
   keyword: Agent Relation Jump Target
   aliases:
     - agent jump target
   ---

   An agent relation jump target is what an Agents-tab numeric keymap jumps to: a sase
   node, or a dismissed sase agent that the keymap revives, reached through one of the
   selection's agent relations. Each one is a numbered row, its number chip at the left,
   in that relation's roster section: `FAMILY SHELLS` (an agent family's sase shells,
   monitor and gate shells included), `NEIGHBORS` (the selected agent's agent neighbors,
   dismissed ones included), `CLAN MEMBERS` (an agent clan's direct members), or
   `TRIBE MEMBERS` (an agent tribe's top-level clans, families, workflows, and agents).
   The rosters shown for one selection share a single number sequence: `0`–`9`, or
   `00`–`99` past ten targets, capped at 100. Typing a number selects its target and
   reveals it through any folds; a dismissed target is revived instead. The jump panel,
   the sticky footer below the Agents detail panels, lists every live agent relation
   jump target.

   Only numbered rows are agent relation jump targets. Rows behind a `… +N more` tail,
   unnumbered child rows, `… also listed under FAMILY SHELLS` duplicates, and tribe
   `CLAN SUMMARIES` / `PROMPTS` chips that reuse a member's number are not. Plain "jump
   target" is ambiguous, and none of its other uses is an agent relation jump target:
   `'` entry hints (including the `AgentJumpTarget` type in `jump_hints.py`),
   `Ctrl+J`/`Ctrl+K` metadata section stops, `,j`/`,J` unread and stopped-agent jumps,
   `Ctrl+]` prompt-definition jumps, and Admin Center section and `,L` error-log jumps.
   The Artifacts tab's relation panel `<`/`>` keys follow artifact relations, not agent
   relations.
   ```

   - Keep the single alias `agent jump target`. Do not add a bare `jump target` alias,
     because avoiding that ambiguity is the reason for this term. Lookups and implicit
     mentions already match plurals: `glossary:"agent hoods"` resolves to `Agent Hood`.
   - Add no `[[...]]` links. The glossary web uses `link_reference: implicit`, so the
     body's term mentions become related strands on their own. Those mentions are sase
     node, sase agent, agent family, sase shells, monitor, gate shells, agent neighbors,
     agent clan, and agent tribe.
   - `just fmt` may rewrap the prose. That is fine as long as the wording stays the
     same.

2. Run `sase memory init`. It regenerates the roster in `sase/memory/glossary.md`, which
   must now list `Agent Relation Jump Target (agent jump target)` between `Agent Node`
   and `Agent Shell`. It also regenerates `sase/memory/README.md`, `AGENTS.md`, and the
   provider shims (`CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`). Never hand-edit
   those generated files.

3. Check the result:
   - `sase memory web show glossary jump` lists the new strand with its alias.
   - `sase memory show glossary:"agent relation jump targets"` and
     `sase memory show glossary:"agent jump targets"` both print the new strand. Its
     related strands include Agent Family, Agent Neighbor, Agent Clan, Agent Tribe, Sase
     Node, Sase Agent, and Sase Shell.
   - `sase memory show glossary:"jump target"` does **not** resolve to the new strand.
   - `sase memory init` and `sase doctor` report no new memory-web warnings or
     catalog-ambiguity errors.

## Verification

Run `just fix`, then `sase tool run check`. Do not run `just check-full`.

## Out of scope

- A separate glossary strand for the jump panel itself.
- Renaming code identifiers (such as `AgentJumpTarget` or `MemberJumpMap`), or rewording
  other "jump target" uses in docs, docstrings, or plans.
- The roster move in `jump_panel_roster_move.md`.
