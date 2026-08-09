---
tier: tale
title: Use regex alternation for new-task duplicate searches
goal:
  The new-task skill finds semantic duplicates efficiently with one concise regex search
  while preserving active-epic checks.
proposed_by: bbugyi200.athena.wb.f1
create_time: 2026-08-09 09:11:13
status: wip
---

# Plan: Use regex alternation for `/sase_new_task` duplicate searches

## Goal

Keep duplicate detection broad but inexpensive by teaching `/sase_new_task` to combine
several distinctive identifiers into one opt-in `sase bead search --regex` query. The
default literal-search semantics remain useful for a single sufficient substring, and
the independent `sase bead list` check for related in-progress epics remains unchanged.

## Scope

- Update `src/sase/xprompts/skills/sase_new_task.md` so its task-duplicate example uses
  one case-insensitive regex alternation with `--regex --type task`.
- Concisely tell agents to choose a few short, distinctive terms (such as a symbol,
  filename, command, or error fragment), escape metacharacters intended literally, and
  fall back to the default literal search when one substring is sufficient.
- Preserve the existing all-status coverage, semantic-duplicate criteria, prohibition on
  listing every task bead, and separate in-progress epic-list workflow.
- Update `tests/main/test_init_skills_sources.py` to pin the regex-search example and
  retain the regression guards against task-wide listing and removal of the epic list.

No CLI behavior, SASE memory file, rendered global skill, or documentation outside the
skill source is changed. Generated-skill deployment remains a post-commit operation and
is not part of this implementation.

## Implementation

1. Replace the literal placeholder command in step 3 of the skill with a compact regex
   alternation example, keeping the task type filter.
2. Rewrite only the adjacent search guidance needed to explain why and when to use the
   regex form, including literal metacharacter escaping and the single-literal fallback.
3. Adjust the source-content assertions so tests require the regex command/flag while
   continuing to reject `sase bead list --type task` and require the existing
   in-progress epic `sase bead list` command.

## Validation

1. Run `just install` to refresh the workspace environment.
2. Run `sase skill init --diff` and confirm the generated preview contains only the
   intended `/sase_new_task` instruction change.
3. Run the focused skill-source tests:
   `.venv/bin/python -m pytest tests/main/test_init_skills_sources.py -q -p no:randomly`.
4. Run `just check` for the required repository-wide lint gates and diff-scoped tests.

## Acceptance criteria

- The skill directs agents to perform one regex-alternation task search instead of a few
  separate literal searches when several clues are available.
- The guidance remains concise, accurately describes opt-in regex versus default literal
  behavior, and searches task beads across every status.
- Duplicate classification/corroboration and active-epic routing behavior are unchanged.
- Focused tests and `just check` pass, and the generated-skill preview is as expected.
