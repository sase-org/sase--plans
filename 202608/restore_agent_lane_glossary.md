---
tier: tale
title: Restore Agent Lane to the project glossary
goal:
  Distinguish an agent lane's visible presentation from a sase agent's durable identity.
size: small
proposed_by: bbugyi200.athena.027
create_time: 2026-08-15 09:37:34
status: wip
---

# Restore Agent Lane to the Project Glossary

## Goal

Restore `Agent Lane` as a glossary term without conflating a lane's visible presentation
with the durable identity of a sase agent.

## Implementation

1. Add `Agent Lane` in alphabetical order to the project glossary source in
   `sase/sase.yml` with this concise definition:

   > An agent lane is the display container for a non-dismissed sase agent: its agent
   > family, or its agent shell when it has no family. Dismissal removes the lane, not
   > the sase agent's identity.

2. Run `sase memory init --no-commit` so the canonical glossary note, memory README,
   `AGENTS.md`, and existing provider instruction shims are regenerated from the source.
   Review the resulting diff to ensure `Agent Lane` appears in the generated glossary
   and Tier 2 glossary routing, and that no unrelated content changed.

## Validation

1. Run `sase memory init --check` and confirm there is no generated-memory drift.
2. Run `just install`, then `just check`, and resolve any failures caused by the change.
3. Re-read the generated glossary through
   `sase memory read glossary.md --reason "Verify the restored Agent Lane definition and its distinction from Sase Agent"`
   and confirm the final definition is concise, accurate, and present alongside the
   related agent terms.
