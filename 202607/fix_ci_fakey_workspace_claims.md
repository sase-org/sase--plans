---
create_time: 2026-07-11 11:41:28
status: done
prompt: 202607/prompts/fix_ci_fakey_workspace_claims.md
tier: tale
---
# Plan: Restore CI retry and agent-loader coverage

## Problem and root cause

GitHub Actions run `29155705709` fails in the visual suite and Python 3.14 suite. The Python 3.12 and 3.13 jobs are
cancellation fallout, not independent failures.

The visual failures and eight of the nine Python 3.14 failures share one cause. Real Fakey retry tests intentionally
produce an LLM provider error, after which prompt-step postprocessing tries to save central chat history. Anonymous
agent workflows already receive the current ChangeSpec/workspace label as `cl_name`, but the workflow prompt-step
executor does not forward that value to `invoke_agent` or its direct chat-history save. Chat filename generation
consequently shells out to the personal `branch_or_workspace_name` helper. That helper exists in the developer
environment but is not part of the CI installation. Its failure replaces the original retryable provider error,
preventing retry state, fallback, and handoff transitions. A CI-like local `PATH` reproduces all five retry-suite
failures, while the same suite passes with the personal helper available.

The remaining Python 3.14 failure is an independent test-isolation defect. The cross-snapshot PID de-dup test uses a
`gh-active` workspace-claim row. Workspace claim prefixes are intentionally obtained from registered providers; the
local environment has the GitHub plugin, while CI has only the built-in `git` provider. The test therefore exercises
different logic depending on external plugin installation. A controlled registry with `{"git", "gh"}` passes and
`{"git"}` fails.

## Implementation

1. Propagate the workflow execution context's non-empty string `cl_name` into both chat-history paths for prompt steps:
   - pass it as `branch_or_workspace` to `invoke_agent`, covering success and error postprocessing;
   - pass it to the prompt step's explicit `save_chat_history` call, keeping the TUI response-path record consistent.
     Standalone workflows without a usable `cl_name` will retain existing auto-detection behavior.

2. Make failed-invocation postprocessing best-effort, matching successful-invocation postprocessing: catch chat-history
   persistence errors, emit the existing warning style, and preserve the original provider exception. This prevents
   auxiliary history bookkeeping from masking retry classification or any other primary LLM error.

3. Add focused regression coverage:
   - verify a prompt-step workflow forwards `cl_name` to both invocation logging and direct chat persistence;
   - verify error postprocessing tolerates a chat-history save failure and reports a warning;
   - retain end-to-end proof by running the Fakey retry suite with a CI-like `PATH` that excludes the personal helper.

4. Remove the loader test's external plugin dependency by representing its generic workspace claim with the built-in
   `git` workflow prefix (or equivalently supplying an explicit registry fixture if surrounding test structure makes
   that clearer). Keep the assertion focused on same-PID cross-snapshot de-dup and metadata transfer, not GitHub plugin
   discovery.

## Verification

1. Run the focused workflow-executor, postprocessing, loader de-dup, and Fakey retry tests.
2. Run the three real-Fakey visual E2E tests under the CI-like `PATH`; inspect visual artifacts only if rendering
   changes unexpectedly. No golden image update is expected because the fix restores the intended states.
3. Run `just check` as required for repository changes, covering formatting, lint/type checks, the full test suite, and
   visual snapshots.
4. Review the final diff and repository status to ensure only the planned source/tests and this plan artifact changed.
