---
tier: tale
title: Refresh the user-facing SASE documentation
goal:
  SASE's user-facing guides accurately explain current repository behavior to newcomers.
size: medium
proposed_by: bbugyi200.athena.chop.refresh_docs.sase.8_383610.1
create_time: 2026-08-11 06:04:18
status: wip
---

# Refresh the user-facing SASE documentation

## Objective

Bring SASE's user-facing documentation up to date with repository behavior landed since
the most recent explicit documentation refresh (`64f9383f1`), while making the affected
workflows understandable to a newcomer.

## Scope and constraints

- Treat the current implementation, CLI help, configuration schema, and tests as the
  behavioral source of truth.
- Review changes from `64f9383f1..HEAD`, including later documentation commits, and
  avoid duplicating coverage that has already landed.
- Create, edit, or remove documentation files only. Do not change source code, tests,
  build configuration, generated provider instructions, or SASE memory files.
- If implementation behavior appears buggy, leave it unchanged and report it in the
  final handoff.

## Implementation

1. Audit post-refresh commits and their current implementation/help for user-visible
   changes, concentrating on the Stitch CLI rename and commit dispatch, ACE's Stitches
   view, folding controls, prompt snippets, glossary presentation, Patch PR origins,
   query properties, VCS merge filtering, tale/task sizing and lifecycle, and bead
   search/list metadata.
2. Cross-check the relevant READMEs and `docs/` guides for stale names, missing
   commands, incomplete option semantics, and explanations that assume prior SASE
   knowledge.
3. Update only the documentation pages that need correction or clearer onboarding. Keep
   command examples consistent with live CLI help and cross-link concepts instead of
   repeating long explanations.
4. Review the complete documentation diff for internal consistency, accurate anchors,
   and accidental edits outside the allowed file set.

## Verification

- Run the repository's generated-document and Markdown formatting checks.
- Run `just docs-check` for a strict MkDocs build.
- Run `just check` as required by the repository instructions after file changes.
- Confirm with `git status --short` that every changed file is documentation.
