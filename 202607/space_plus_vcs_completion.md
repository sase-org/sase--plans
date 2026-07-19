---
tier: epic
title: Space-triggered VCS project completion
goal: 'Project and active ChangeSpec completion opens from a bare plus token at the
  start of a prompt or immediately after a literal space, with hash-plus removed as
  a trigger and ACE kept in parity with the Rust editor and LSP.

  '
phases:
- id: core
  title: Rust core and LSP trigger contract
  depends_on: []
  description: '''Rust core and LSP trigger contract'' section: replace hash-plus
    and BOF-only detection with the shared bare-plus-at-BOF-or-after-space contract,
    then update Rust completion and LSP coverage.'
- id: ace
  title: ACE integration, parity, and documentation
  depends_on:
  - core
  description: '''ACE integration, parity, and documentation'' section: align the
    Python helper and prompt widget with the Rust contract, replace regression vectors,
    update user guidance, and run integrated checks.'
create_time: 2026-07-19 08:38:34
status: wip
---

# Plan: Space-triggered VCS project completion

## Context and behavior contract

ACE currently opens its enabled-project/active-ChangeSpec picker for `+query` only when the plus begins the entire
prompt; elsewhere it requires `#+query`. The Python trigger helper is also a stated parity contract for the Rust editor
core, whose xprompt LSP exposes the same completion catalog and tag rewrite. This change should therefore update both
implementations instead of making the prompt widget diverge from editor completion.

Adopt this trigger contract:

| Prompt text at the cursor                  | Project/ChangeSpec completion                                                                 |
| ------------------------------------------ | --------------------------------------------------------------------------------------------- |
| `+` or `+sa` at absolute offset zero       | Open and filter, preserving the existing prompt-start shortcut                                |
| `Fix +` or `Fix +sa`                       | Open and filter because the plus is immediately preceded by the literal ASCII space character |
| `line\n +`                                 | Open because the plus itself follows a literal space                                          |
| `line\n+`, `\t+`, `word+`, `a+b`, or `c++` | Do not claim the token                                                                        |
| `#+`, `#+sa`, or `Fix #+sa`                | Do not open project/ChangeSpec completion; hash-plus is ordinary prompt text for this feature |

For a valid trigger, the token begins at `+`, continues to the next whitespace boundary, and uses the text between `+`
and the cursor as its case-insensitive project, alias, or ChangeSpec-name query. Acceptance continues to remove the
entire trigger token and apply the existing canonical expansion: replace any line-start VCS workflow tag, or prepend the
selected tag after leading frontmatter and directives. Candidate ordering, active ChangeSpec statuses, enabled-project
filtering, empty-catalog presentation, completion navigation, and catalog caching are out of scope and must remain
unchanged.

The literal-space rule intentionally means prose such as `2 + 2` can open the picker when the plus is typed. That is the
direct-trigger tradeoff requested; the existing dismissal behavior should remain cheap, and the keystroke path must
continue using the pre-warmed/cached catalog without new I/O or subprocess work.

## Rust core and LSP trigger contract

Open the linked `sase-core` repository through `sase repo open` and make the Rust editor implementation authoritative
for the new cross-frontend behavior.

- Update VCS-project token classification and context detection so only a `+query` token at document offset zero or
  immediately after literal ASCII space is recognized. Remove `#+query` classification and the variable
  hash-plus/bare-plus prefix handling; valid queries now always begin after one plus byte.
- Keep completion edit generation and canonical VCS-tag expansion intact, but replace the shared golden vectors with
  bare-plus-at-BOF and space-delimited-plus cases, including existing-tag replacement, frontmatter/directive placement,
  a cursor inside a longer query token, and non-overlapping edit assertions.
- Align the xprompt LSP response path with the single `+` spelling so client-side `filter_text`, completion
  documentation, and direct LSP tests no longer advertise or accept hash-plus. Retain `+` as an LSP trigger character.
- Add negative Rust coverage for hash-plus at BOF and after space, plus after a newline or tab without an intervening
  literal space, and glued/operator uses; add positive coverage for BOF and mid-prompt space-delimited automatic/manual
  completion, query filtering, ChangeSpec rows, and canonical edits.
- Re-sweep nonhistorical Rust comments and docs for the obsolete syntax while leaving released changelog history
  untouched. Validate with `cargo fmt --all -- --check`, `cargo clippy --workspace --all-targets -- -D warnings`, and
  `cargo test --workspace`.

## ACE integration, parity, and documentation

After the core phase, apply the same contract to the Python helper consumed by every ACE prompt pane and update the
prompt widget regressions around it.

- Change `find_vcs_project_trigger` and its data-model documentation to detect only `+query` at prompt offset zero or
  immediately after literal ASCII space. Preserve whole-token spans, cursor-local queries, filtering, catalog caching,
  and `apply_vcs_project_selection` behavior.
- Keep the prompt widget's existing post-keypress auto-open, refresh, manual `Ctrl+T`, acceptance, dismissal, and
  off-thread catalog warm-up paths; update their terminology and precedence comments from hash-plus to the new plus
  token without adding work to the keystroke path.
- Rewrite the Python/Rust parity vectors and focused ACE tests to exercise BOF and mid-prompt `+` auto-open, narrowing,
  navigation, selection, cursor placement, existing-tag replacement, frontmatter/directive insertion, placeholders,
  dismissal, and `Ctrl+T`. Explicitly assert that `#+` no longer routes to the project picker and that newline-start,
  tab-delimited, and glued plus forms remain unclaimed. Update any visual test setup to use the supported spelling; the
  panel layout itself should not require a golden-image change.
- Update the ACE, xprompt, editor/LSP, and configuration documentation to show BOF `+query` and literal-space-delimited
  `+query` only. Describe the syntax once consistently for both ACE and the xprompt LSP, and sweep source/test
  docstrings for stale `#+` guidance while preserving historical changelog entries.
- Run the focused headless and widget completion tests first, then the relevant visual snapshot suite. Because this
  phase changes files in the SASE repo, run `just install` before checks and finish with `just check`; use the locally
  opened core checkout so the final verification exercises the paired Rust and Python changes together.

## Completion criteria and risks

The epic is complete when ACE and the xprompt LSP agree on every positive and negative trigger example above, selecting
a row still produces byte-identical canonical prompt text in Rust and Python, and no current help text tells users to
type `#+`. The principal regression risks are accidentally treating all Unicode whitespace as the requested literal
space, allowing `#+` to fall through as a project trigger, breaking replacement spans when the plus appears mid-prompt,
or adding catalog work to the typing path; the phase-specific tests and full repository checks must cover those
boundaries.
