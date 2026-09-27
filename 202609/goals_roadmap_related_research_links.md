---
tier: tale
title: Link the SASE Goals epic roadmap to all related research
goal:
  Every agent planning a SASE Goals epic finds all related Goals research from
  sase_goals_epic_roadmap.md. That includes the design, the memory end state,
  persistence, why-not-beads, swarm goals, the first-round synthesis, and their drafts.
  Planners also get section-level reading pointers per epic, and typed artifact links
  make the same neighbors visible in `sase artifact read` and the TUI.
size: small
proposed_by: bbugyi200.athena.0te
create_time: 2026-09-27 18:54:40
status: wip
---

# Link the SASE Goals epic roadmap to every related research report

## Goal

Every agent that plans a SASE Goals epic (G1–G6) reads
`research:202609/sase_goals_epic_roadmap/sase_goals_epic_roadmap.md`. Today that roadmap
links only to its own five drafts and to the design. The other Goals reports are not
linked from it, including the memory end-state report that tells each G-plan which
memory changes it owns. A planner has to find them by luck.

Make the roadmap the index for the Goals research family:

1. Add a **Related research** section near the top of the roadmap. It links every
   related report by canonical ref, says what each adds, and maps which sections each
   epic's planner should read.
2. Add six **typed artifact links** (`sase artifact link add`). The same neighbors then
   show up in the `Links:` line that `sase artifact read` prints, in the TUI link rail,
   and in the one-hop expansion of any launch prompt that cites the roadmap.

## Where the work happens

- The roadmap lives in the `sase--research` sidecar. Open it with
  `sase repo open sase--research -r "<reason>"` and use only the path it prints. That
  checkout's root holds `202609/` and `README.md`. The repo has no lint or formatter
  config.
- Read reports only with `sase artifact read <ref> "<reason>"`. Canonical refs are
  `research:<path relative to the sidecar root>`.
- The primary `sase` repo is **not** changed. No `just check` is needed. The edited
  sidecar becomes a repository obligation that `/sase_final` commits.

## The Goals research family (verified 2026-09-27)

All of it is under `202609/` in the sidecar. A repo-wide grep for Goals terms finds
nothing outside these files.

| Report (lead)                                                                | Directory siblings                                                                                                                       |
| ---------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `sase_goals_epic_roadmap/sase_goals_epic_roadmap.md` (the file being edited) | drafts `__cdx/__cld/__grk/__mus/__gem.md` (already linked under **Sources**), `sase_goals_epic_roadmap_infographic.png` (not linked yet) |
| `sase_goals_design/sase_goals_design.md`                                     | drafts `__cdx/__cld/__grk/__mus/__gem.md`, `sase_goals_design_infographic.png`                                                           |
| `sase_goals_memory_end_state/sase_goals_memory_end_state.md`                 | drafts `__cdx/__cld/__grk/__mus/__gem.md` (no infographic)                                                                               |
| `goal_outcomes_and_verification/goal_outcomes_and_verification.md`           | drafts `__cdx/__grk/__mus/__gem.md` (no `__cld`), `goal_outcomes_and_verification_infographic.png`                                       |
| `sase_goals_persistence.md`                                                  | flat note                                                                                                                                |
| `sase_goals_why_not_beads.md`                                                | flat note                                                                                                                                |
| `xprompt_swarm_goals.md`                                                     | flat note                                                                                                                                |

Two adjacent reports outside the family are also worth one pointer:

- `sase_tool_epic_roadmap/sase_tool_epic_roadmap.md` is the `sase tool` sibling-epic
  split that the roadmap's §1.1 cites as its precedent.
- `sase_tool_e3_e4_landing_criteria/sase_tool_e3_e4_landing_criteria.md` holds the E4
  verification-receipt landing criteria. Those receipts are the `sase-1ah` contract
  whose snapshots G3's claims embed.

## Steps

### 1. Insert the Related research section

In `202609/sase_goals_epic_roadmap/sase_goals_epic_roadmap.md`, insert the section below
immediately after the paragraph
`§3 lists where individual reports were wrong or out of date.` and before the `---` rule
that precedes `## 0. Bottom line`. Leave one blank line on each side.

Constraints:

- Keep the heading **unnumbered** (`## Related research`). The body cross-references
  §0–§9 (for example §1.1, §3, §4, and §7), so nothing may be renumbered.
- Do not change any existing text in the roadmap.
- Wrap prose at 88 columns, as the rest of the file does. Link lines and table rows may
  run longer.
- **Verify every § pointer before committing.** Read each report with
  `sase artifact read` and check that each cited section says what the draft below
  claims. The pointers were checked while planning; fix any that have drifted rather
  than dropping them.

Section text to insert. Adjust only where verification demands it:

```markdown
## Related research

Read these with `sase artifact read <ref> "<reason>"` before planning a Goals epic. Each
link's text is its canonical ref. The design is the destination; this roadmap only
decides how to deliver it.

- [`research:202609/sase_goals_design/sase_goals_design.md`](../sase_goals_design/sase_goals_design.md)
  — **the design.** Lifecycle (§4.1), binding (§4.2–§4.4), the finalizer and evidence
  (§4.5), attention (§4.6), storage and sync (§4.7), provenance (§4.8), the `goal:` kind
  and relations (§4.9), the TUI (§4.10), the CLI (§4.11), the Rust/Python split (§4.12,
  every epic), and acceptance tests 1–12 (§5).
- [`research:202609/sase_goals_memory_end_state/sase_goals_memory_end_state.md`](../sase_goals_memory_end_state/sase_goals_memory_end_state.md)
  — **the memory changes each epic owns.** §4 has one row per epic listing the decision
  records, glossary strands, and memory edits that epic's plan should authorize, plus a
  definition-of-done line. §3 details each item.
- [`research:202609/sase_goals_persistence.md`](../sase_goals_persistence.md) —
  **storage and cross-machine sync:** when a goal is pushed and fetched, plus five sync
  gaps (§4) to close before the ledger contract freezes. It predates the G-numbering.
  Its "E1" is G1, and its "E2" is G2 plus G4's drafts.
- [`research:202609/sase_goals_why_not_beads.md`](../sase_goals_why_not_beads.md) —
  **why goals aren't beads.** Goals reuse the beads repository, hidden clone, and sync,
  but not the bead model (§3). §5 says what would reopen that choice.
- [`research:202609/xprompt_swarm_goals.md`](../xprompt_swarm_goals.md) — **swarm
  goals.** A swarm shares one draft goal, and only the lead claims. §5 maps swarm
  behavior onto G1–G6, and §6 lists three gaps for the G2–G4 planners.
- [`research:202609/goal_outcomes_and_verification/goal_outcomes_and_verification.md`](../goal_outcomes_and_verification/goal_outcomes_and_verification.md)
  — **the first-round synthesis** that the design builds on. It is background only;
  where the two disagree, the design wins.

**What to read for each epic,** in addition to this roadmap and that epic's §4 row in
the memory end-state report:

| Epic | Design sections                                                  | Other reports                                              |
| ---- | ---------------------------------------------------------------- | ---------------------------------------------------------- |
| G1   | §4.1, §4.7, §4.9 (the `goal:` kind), §4.11                       | persistence (all); why-not-beads §3                        |
| G2   | §4.2, §4.3 (the bound line), §4.4, §4.8 (unit-prompt digest)     | swarm §6.3                                                 |
| G3   | §4.5, §4.6                                                       | swarm §6.1; the receipt landing criteria below             |
| G4   | §4.2 (drafts), §4.3 (intake), §4.8 (naming publishes the prompt) | persistence §4 items 1, 4, and 5; swarm §2, §5, §6.1, §6.2 |
| G5   | §4.9 (jumps), §4.10                                              | —                                                          |
| G6   | §4.6, §5 (metrics)                                               | —                                                          |

**Adjacent research.**
[`research:202609/sase_tool_epic_roadmap/sase_tool_epic_roadmap.md`](../sase_tool_epic_roadmap/sase_tool_epic_roadmap.md)
is the `sase tool` split that §1.1 cites as the precedent.
[`research:202609/sase_tool_e3_e4_landing_criteria/sase_tool_e3_e4_landing_criteria.md`](../sase_tool_e3_e4_landing_criteria/sase_tool_e3_e4_landing_criteria.md)
sets the landing criteria for the E4 verification receipts (`sase-1ah`) whose snapshots
G3's claims embed.

**Per-researcher drafts** sit beside each lead report. Read one only for detail its lead
dropped or a position the lead overruled:

- design: [cdx](../sase_goals_design/sase_goals_design__cdx.md),
  [cld](../sase_goals_design/sase_goals_design__cld.md),
  [grk](../sase_goals_design/sase_goals_design__grk.md),
  [mus](../sase_goals_design/sase_goals_design__mus.md),
  [gem](../sase_goals_design/sase_goals_design__gem.md), and the
  [infographic](../sase_goals_design/sase_goals_design_infographic.png);
- memory end state:
  [cdx](../sase_goals_memory_end_state/sase_goals_memory_end_state__cdx.md),
  [cld](../sase_goals_memory_end_state/sase_goals_memory_end_state__cld.md),
  [grk](../sase_goals_memory_end_state/sase_goals_memory_end_state__grk.md),
  [mus](../sase_goals_memory_end_state/sase_goals_memory_end_state__mus.md),
  [gem](../sase_goals_memory_end_state/sase_goals_memory_end_state__gem.md);
- first round:
  [cdx](../goal_outcomes_and_verification/goal_outcomes_and_verification__cdx.md),
  [grk](../goal_outcomes_and_verification/goal_outcomes_and_verification__grk.md),
  [mus](../goal_outcomes_and_verification/goal_outcomes_and_verification__mus.md),
  [gem](../goal_outcomes_and_verification/goal_outcomes_and_verification__gem.md), and
  the
  [infographic](../goal_outcomes_and_verification/goal_outcomes_and_verification_infographic.png).

This roadmap's own drafts are linked under **Sources** above. Its infographic is
[`sase_goals_epic_roadmap_infographic.png`](sase_goals_epic_roadmap_infographic.png).
```

### 2. Check that every link resolves

From the sidecar root, confirm that every relative link target in the new section
exists. Resolve each path against `202609/sase_goals_epic_roadmap/`. For example,
extract every `](...)` target in the section with a short script and run `test -e` on
each. Also check that each canonical ref in link text equals `research:` plus the link's
resolved path relative to the sidecar root. Fix any mismatch before continuing.

### 3. Add six typed artifact links

Run these after the file edit. Use each exact source, relation, and target. The `why`
strings may be reworded, but each must stay one line of at most 240 characters:

```bash
sase artifact link add research:202609/sase_goals_epic_roadmap/sase_goals_epic_roadmap.md derives-from research:202609/sase_goals_design/sase_goals_design.md "splits the Goals design into the G1-G6 delivery epics"
sase artifact link add research:202609/sase_goals_memory_end_state/sase_goals_memory_end_state.md derives-from research:202609/sase_goals_epic_roadmap/sase_goals_epic_roadmap.md "lists the memory changes each roadmap epic G1-G6 should make"
sase artifact link add research:202609/xprompt_swarm_goals.md derives-from research:202609/sase_goals_epic_roadmap/sase_goals_epic_roadmap.md "maps swarm goal binding onto the roadmap epics; swarm goals land in G4"
sase artifact link add research:202609/sase_goals_epic_roadmap/sase_goals_epic_roadmap.md related research:202609/sase_goals_persistence.md "goal storage and cross-machine sync gaps the G1 ledger must settle"
sase artifact link add research:202609/sase_goals_epic_roadmap/sase_goals_epic_roadmap.md related research:202609/sase_goals_why_not_beads.md "why G1 co-hosts goals in the beads repo without reusing the bead model"
sase artifact link add research:202609/sase_goals_epic_roadmap/sase_goals_epic_roadmap.md related research:202609/goal_outcomes_and_verification/goal_outcomes_and_verification.md "first-round Goals synthesis that the design builds on"
```

Why these six, and why these relations:

- `derives-from` follows the registry's direction rule: the derived report is the
  source. The roadmap derives from the design. The memory end-state report and the swarm
  note are both keyed to the roadmap's G1–G6.
- `related` covers the reports that inform the roadmap without being derived from it or
  into it.
- The `Links:` line from `sase artifact read` shows at most five neighbors. Semantic
  relations sort first, then by label. From the roadmap's side, the five shown are
  derived-into memory end state, derived-into swarm goals, derives-from design, then
  related outcomes and related persistence. The two most important neighbors, the memory
  end state and the design, come first.
- **Do not** add typed links to the per-researcher drafts or to the adjacent `sase tool`
  reports. Drafts would crowd the five-item `Links:` line, and the adjacent reports are
  weaker relations. The in-file section already covers both.

`link add` publishes immutable link events and records its own
`chore(artifact-links): persist link events` commit in the sidecar. Never hand-edit
`link-events/` or `links/`. If `link add` fails, keep the file edit and do not try to
repair the SASE install. Failures include a publish or durability error and a stale
`sase_core_rs` binding error. The warning `managed tmp root registration failed …` that
every command prints today is not a failure. Report the failed command and its error in
the final response so the user can rerun it.

### 4. Verify

1. `sase artifact link list research:202609/sase_goals_epic_roadmap/sase_goals_epic_roadmap.md -o manual`
   shows the six new rows. The roadmap is the source or target of each one, exactly as
   listed in step 3.
2. `sase artifact read research:202609/sase_goals_epic_roadmap/sase_goals_epic_roadmap.md "<reason>"`
   prints a `Links:` line that starts with the semantic neighbors, not with `read-by`
   rows. Its body shows the new section before `## 0. Bottom line`.
3. `git diff` in the sidecar shows only the inserted section in the roadmap. It may also
   show link-event files if `link add` left them for the finalizer. No other report
   changes.

## Out of scope

- Editing any other report, for example adding backlinks in their bodies. The typed
  links are bidirectional, so each related report's `Links:` line already points back to
  the roadmap.
- Renumbering or rewording the roadmap's existing sections.
- `derives-from` links from each lead report to its own drafts.
- Any change to the `sase` repo, SASE memory, or skills.

## Acceptance

- The roadmap has a `## Related research` section between the opening
  **Sources**/verification paragraphs and `## 0. Bottom line`. It links, by canonical
  ref and working relative path, the design, memory end-state, persistence,
  why-not-beads, swarm-goals, and first-round reports; the two adjacent `sase tool`
  reports; every per-researcher draft and infographic of the design, memory end-state,
  and first-round reports; and the roadmap's own infographic.
- The section's per-epic table gives G1–G6 planners section-level reading pointers, and
  each pointer has been checked against the report it cites.
- The six typed links exist, or any failure is reported with its exact command.
- No other file content in the sidecar changed, apart from link-event publication.
