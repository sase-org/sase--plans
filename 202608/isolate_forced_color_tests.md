---
tier: tale
title: Isolate captured-output tests from ambient forced color
goal:
  Ordinary CLI and TUI assertions are deterministic under forced-color agent and CI
  environments while explicit color coverage remains intact.
size: small
proposed_by: bbugyi200.athena.sase-m7
bead: sase-m7
create_time: 2026-08-15 16:18:11
status: wip
---

- **PROMPT:**
  [prompts/202608/isolate_forced_color_tests.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/isolate_forced_color_tests.md)
- **BEAD:**
  [sase-m7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-m7/README.md)

# Make captured-output tests independent of ambient forced color

## Context

The agent and CI environment can export `FORCE_COLOR=1` (often alongside
`CLICOLOR_FORCE`, `CLICOLOR`, and `NO_COLOR`). Rich treats a nonempty `FORCE_COLOR`
value as proof that even pytest's non-TTY capture stream is a terminal. As a result,
ordinary CLI and Textual render tests receive ANSI styling inside words and fail
otherwise-correct plain-substring assertions. Representative failures have been
reproduced in `tests/test_output.py`, `tests/test_plan_validate.py`,
`tests/test_bead/test_cli_work_from_plan_preview.py`, and
`tests/ace/tui/test_plugins_browser_pane_detail.py`.

Tests that intentionally verify colored output already opt in explicitly through APIs
such as `--color always` or a forced terminal. The PNG visual suite separately owns and
pins its renderer environment. Ambient caller state should therefore not choose color
behavior for the ordinary suite.

## Implementation

1. Add a root autouse environment-isolation fixture in `tests/_conftest_environment.py`
   that removes ambient console force-color overrides for each test, using pytest's
   `monkeypatch` lifecycle so tests remain free to set an override intentionally. Expose
   the fixture through `tests/conftest.py` alongside the other shared environment
   fixtures.
2. Keep the fixture narrowly scoped to variables that force color on non-TTY capture; do
   not strip ANSI in assertions or change production rendering. Preserve the visual
   suite's pinned renderer fixture and explicit CLI color-mode coverage.
3. Re-run the representative failure matrix under the same hostile ambient environment
   (`FORCE_COLOR=1`, `CLICOLOR_FORCE=1`, `CLICOLOR=1`, `NO_COLOR=1`, and `CI=true`) and
   run explicit colored-rendering tests to prove their ANSI expectations still pass.
4. Because a root-conftest helper is in the test-selection broadening set, run
   `just check-full` through the SASE monitor workflow after the focused checks. Inspect
   the completed result and resolve any failures caused by this change before closing
   `sase-m7` with a note naming the hostile-environment reproduction, explicit-color
   coverage, and full-suite verification.

## Acceptance criteria

- Ordinary captured CLI/TUI output remains plain and its substring assertions pass even
  when the surrounding agent or CI process exports forced-color variables.
- Individual tests can still opt into and assert ANSI rendering explicitly.
- No production rendering code or one-off substring assertion is weakened.
- Focused hostile-environment checks and the repository's full verification suite pass.
