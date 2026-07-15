---
plan: 202607/stabilize_xprompt_skill_highlight_test.md
---
 The `just test` command just failed (see the output below). Can you help me diagnose the root cause of this issue and fix it? Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.

```
============================================================================= short test summary info ==============================================================================
FAILED tests/ace/tui/widgets/test_prompt_xprompt_highlight.py::test_xprompt_highlight_overlay_marks_spans_and_registers_styles - AssertionError: assert 'xprompt.skill' in ['heading', 'xprompt.invocation', 'xprompt.invocation_arg', 'xprompt.directive', 'xprompt.invocation', 'xprompt.invocation_arg', ...]
======================================================= 1 failed, 17213 passed, 7 skipped, 46 warnings in 138.84s (0:02:18) ========================================================
error: recipe `test` failed on line 212 with exit code 1
```