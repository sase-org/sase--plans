---
tier: tale
title: Fix new-snippet prefix collision handling
goal:
  Let an unused snippet trigger create a new snippet even when it is a prefix of an
  existing trigger.
size: small
proposed_by: bbugyi200.athena.0ai
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0ai](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0ai.md)
- **COMMITS:**
  - [95295fe](https://github.com/sase-org/sase/commit/95295fea07682dc7cce6f09dc654b6d3b6595cb6)
    — fix(snippets): allow unused prefix triggers

# Plan: Fix new-snippet prefix collision handling

## Diagnosis

The `New snippet` modal deliberately builds a prefix-match list so users can inspect and
Tab-complete existing triggers. In `src/sase/ace/tui/modals/snippet_name_modal.py`,
`_build_analysis()` currently infers `destination_exists` with
`any(match.is_destination for match in matches)`. Because `matches` contains prefix
matches, typing an unused trigger such as `r` marks the destination as existing whenever
that config file contains a longer trigger such as `rchat`. The verdict and Enter path
then treat `r` as an exact definition and `_result_for_analysis()` calls
`load_snippet_template(..., "r")`, producing the reported `KeyError` instead of opening
a new snippet pane.

The exact collision data is already available from `snippet_collision()`, independently
of the display-only prefix list. The bug is confined to the modal's classification of
that data; keymap dispatch, snippet persistence, and the shared Rust core do not need to
change.

## Implementation

1. In `src/sase/ace/tui/modals/snippet_name_modal.py`, compute `destination_exists` from
   the exact-name collision matches rather than the truncated prefix-preview matches.
   Preserve the existing fallback probe for an explicitly configured destination that is
   absent from the discovered location list. This keeps prefix suggestions available for
   display and Tab completion while reserving the edit-existing path for an exact
   trigger in the selected destination.
2. In `tests/ace/tui/modals/test_snippet_name_modal.py`, add a focused modal regression
   test whose destination defines a longer trigger (for example `rchat`) while the user
   submits its unused prefix (`r`). Assert that the longer trigger remains visible as a
   match, the verdict offers to create the exact typed trigger, Enter dismisses with a
   non-existing `SnippetNameResult`, and the starting body is empty. Retain the existing
   exact destination, cross-file collision, derived collision, and Tab-completion tests
   as coverage for the neighboring branches.

## Verification

1. Run `just test tests/ace/tui/modals/test_snippet_name_modal.py` to exercise the new
   regression and all adjacent modal behaviors.
2. Run `just check` as the required whole-repository lint and diff-scoped test gate. If
   the workspace reports stale dependencies first, run `just install` and repeat the
   check.
