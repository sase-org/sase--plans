---
create_time: 2026-07-11 14:23:27
status: wip
prompt: 202607/prompts/research_swarm_consolidator_layout.md
tier: tale
---
# Research Swarm Consolidator Output Layout

## Goal

Update the third segment of the chezmoi-managed `research_swarm` xprompt so each research run preserves the two
independent source reports alongside the consolidated report in a dedicated subdirectory of the effective SDD research
directory.

For a consolidator-selected final filename `<name>.md`, the completed layout must be:

```text
<effective-research-directory>/
└── <name>/
    ├── <name>__a.md
    ├── <name>__b.md
    └── <name>.md
```

The `__a` file is the report produced by the first (`research_a` / `.cdx`) agent, and the `__b` file is the report
produced by the second (`research_b` / `.cld`) agent.

## Current Behavior

The consolidator prompt in `home/dot_xprompts/research_swarm.md` reads the two waiting-agent transcripts, identifies and
reads their flat markdown outputs in `$(sase sdd path research)`, writes one flat consolidated markdown file there, and
deletes the two source reports. The fourth image segment then forks the consolidator transcript.

## Implementation

1. Revise only the third (lead/consolidator) segment of `home/dot_xprompts/research_swarm.md`; retain the model aliases,
   waits, group, research request, dynamic `sase sdd path research` lookup, and the other swarm segments.
2. Preserve the existing requirement to inspect both chat transcripts and read both source reports before consolidating.
   Make the prompt associate each discovered report with its producing agent so the `__a` and `__b` assignments are
   deterministic rather than dependent on filesystem ordering.
3. Instruct the consolidator to choose the descriptive final markdown filename first and derive `<name>` by removing its
   `.md` suffix. It should then create `<effective-research-directory>/<name>/` before producing the consolidated
   report.
4. Replace the deletion instruction with explicit moves/renames:
   - first agent's report -> `<name>/<name>__a.md`
   - second agent's report -> `<name>/<name>__b.md` Require the consolidator to preserve both files and avoid silently
     overwriting any existing directory or destination file; if the chosen stem collides, it should select a distinct
     descriptive stem before moving anything.
5. Direct the consolidator to write the improved final report to `<name>/<name>.md`, after the two source reports have
   been safely relocated. Keep the existing quality requirements: verify the prior work, retain the strongest findings,
   resolve conflicts, add critical missing context, remove duplication, and avoid unnecessary length growth.
6. Leave the image-agent segment unchanged unless validation reveals that it relies on a flat path. Its fork of the
   consolidator transcript should naturally expose the new final report path.

## Validation

1. Review the rendered prompt text and segment separators to confirm this remains a four-agent markdown swarm and that
   Jinja/raw blocks plus the dynamic research-directory substitution are unchanged.
2. Check the prompt against representative source paths to confirm the exact deterministic mapping and final layout:
   `topic/topic__a.md`, `topic/topic__b.md`, and `topic/topic.md`.
3. Search the active xprompt for the obsolete instruction to delete intermediate reports and confirm it no longer
   appears.
4. Run the repository's Markdown formatting check for the edited xprompt (and the broader repository checks if required
   by the implementation workflow), then inspect the final diff to ensure no unrelated chezmoi configuration or legacy
   xprompt was modified.

## Non-Goals

- Do not change the independent researchers' prompts or require them to coordinate filenames.
- Do not change research model aliases or the downstream image model.
- Do not modify the legacy `old_research_swarm.md` xprompt.
- Do not add backend behavior: this is a prompt-contract change in the personal chezmoi source.
