---
tier: tale
title: Add the Work Log glossary entry to bob-cli
goal:
  Define Work Log precisely in bob-cli's Tier 2 glossary and synchronize the generated
  memory instructions without adding unnecessary always-loaded context.
size: small
proposed_by: bbugyi200.athena.028.f0.f0.f0
create_time: 2026-08-15 11:25:38
status: done
---

# Plan: Add the Work Log glossary entry to bob-cli

## Goal

Document `Work Log` as a concise, durable bob-cli domain term without expanding the
always-loaded agent context unnecessarily. The canonical definition should explain the
log's task-local Markdown structure, newest-first ordering, and the exact
In-Progress-to-Open behavior that creates an entry, including the blank-summary case.

## Scope

- Work in the registered `bob-cli` repository, opened through `/sase_repo`.
- Edit only the canonical project memory source at `sase/memory/glossary.md` by adding
  this entry after the existing task-related terms:

  ```markdown
  ## Work Log

  A task-local history of work performed, stored as a `🛠️ **WORK LOG**` direct child
  with newest-first summary sub-bullets. When `<ctrl+shift+enter>` changes an In
  Progress (`[/]`) task to Open, its nonblank work summary is prepended; a blank summary
  leaves the log untouched.
  ```

- Run `sase memory init --no-commit` after the canonical edit, as required for bob-cli
  memory changes. Accept the generated `AGENTS.md`, provider-shim, and memory-index
  updates produced by that command; do not hand-edit those generated files.
- Keep the full definition in Tier 2. The generated glossary routing list should add
  only `Work Log`, so agents know when to load the note without placing the definition
  in every prompt.

## Validation

- Inspect the complete diff and confirm that the only semantic source change is the new
  `Work Log` entry and that generated files contain only the corresponding routing or
  index updates.
- Run `sase memory init --check` and require a clean result, proving the canonical
  memory and generated files are synchronized.
- Run `git diff --check` to reject malformed Markdown whitespace.

## Acceptance criteria

- `sase/memory/glossary.md` contains exactly one `Work Log` definition with the agreed
  concise wording and no redundant alias or implementation detail.
- The definition accurately states the canonical direct-child marker, newest-first
  entries, nonblank-summary insertion, and blank-summary no-op behavior.
- Generated agent instruction shims route `Work Log` to `glossary.md`, while the full
  definition remains Tier 2.
- Memory initialization reports no drift and the final diff is whitespace-clean and
  contains no unrelated changes.
