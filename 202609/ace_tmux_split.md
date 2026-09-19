---
tier: tale
title: Split tmux launch support into focused modules
goal:
  Keep the ace tmux launch behavior intact while moving cohesive implementation areas
  into modules of at most 500 lines.
size: medium
proposed_by: bbugyi200.athena.toobig-5p.ace_tmux.0
create_time: 2026-09-19 14:25:38
status: wip
---

# Plan

1. Preserve `sase.main.ace_tmux` as the compatibility-facing entry point, including its
   public API and test seams, while moving shared tmux types, constants, command
   execution, and cleanup helpers to a support module.
2. Extract session bootstrap/resolution and window reservation/creation responsibilities
   into focused modules, keeping their imports acyclic and every resulting source file
   at or below 500 lines.
3. Update the facade and affected tests only as required for the new module boundaries;
   retain behavior for tmux races, claim cleanup, environment injection, fixed-size
   windows, and CLI error handling.
4. Run the focused tmux and screenshot tests, then the repository-required `just check`
   recipe; report the resulting file sizes and verification outcome.
