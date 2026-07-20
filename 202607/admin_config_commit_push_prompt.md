---
tier: tale
title: Prompt to commit Admin Center config edits
goal: 'Successful Admin Center Config-tab edits offer the established y/n commit-and-push
  flow for the file actually written, including chezmoi-backed config sources, without
  blocking the TUI.

  '
create_time: 2026-07-20 10:04:55
status: done
prompt: 202607/prompts/admin_config_commit_push_prompt.md
---

# Plan: Prompt to commit Admin Center config edits

## Context

The Config tab already sends `e` edits through `ConfigEditModal`, writes the source-preserving YAML change in a worker,
and applies the corresponding home target when `use_chezmoi` is enabled. After the modal dismisses, `ConfigPane`
currently only reports success and refreshes its inventory, so the flow ends before offering to persist the dirty
chezmoi repository change to git.

The Models panel's persistent alias editor is the behavioral reference for this same situation. It inspects
`AppliedResult.path` (the file actually written, which is the chezmoi source path when remapping is active), skips files
that are clean or outside git, presents the canonical `ConfirmActionModal` with y/n bindings and `Commit & push` /
`Skip` choices, and runs the existing commit/pull/push helper through the app's tracked task queue. The Config tab
should use that same contract rather than inventing another prompt or git sequence.

## Shared config commit workflow

Extract or generalize the alias-neutral parts of the Models panel's post-write workflow into a shared Config-TUI helper
while retaining thin alias-specific message construction and compatibility at the Models panel boundary. The shared
offer should contain the git root, actual written file, repository-relative display path, and a `config`-tagged commit
message supplied or formatted for the caller.

Offer discovery must use the actual `AppliedResult.path`, not the original applied home target. Resolve its git root and
verify that this specific file has pending changes; return no offer for a clean file, a non-git path, a
missing/unresolvable repository, or git inspection failure. This path-based policy naturally selects the chezmoi source
repository while preserving the existing Models behavior for any directly edited config file that is itself under git.

Keep subprocess work off Textual's event loop. Run git-root/status discovery in a thread-backed worker, and submit
confirmed commit/pull/push work through the existing tracked task queue using the established synchronous git helper.
Preserve the existing `config-commit` task identity and root/path-based deduplication, completion notifications, failure
reporting, and stale `index.lock` retry warning. The Models alias editor should continue to show and execute the same
workflow after the shared extraction, with no user-visible regression.

## Config-tab integration

Extend the successful `ConfigPane` edit-dismissal path so its existing success toast and inventory refresh still happen
immediately, then asynchronously prepare the commit offer for the completed write. Track this worker separately from the
pane's inventory-loading worker and ignore/cancel stale results when the pane is no longer mounted or a newer relevant
operation supersedes them.

When discovery returns an offer, push the existing `ConfirmActionModal` configured consistently with the Models
config-edit use case: `Commit & Push` title, upward-arrow icon, the repository-relative file as subject, `Commit & push`
confirmation, `Skip` cancellation, and confirm as the default. Tailor only the explanatory sentence to the general
config field change. A `y` answer submits the shared tracked commit task; `n`, escape, or dismissal leaves the
already-written/applied change uncommitted. Do not show a prompt when the edit was cancelled, chezmoi apply failed, the
target is clean, or the written path has no git repository.

Ensure the lifecycle remains understandable if the user switches Admin Center tabs while discovery or commit work is
running: discovery must not mutate an unmounted pane, while confirmed git work remains visible in the existing task
indicator/queue and reports its terminal result through the normal notification path.

## Verification

Add focused helper coverage for building a general config commit offer from a dirty file, including a chezmoi-style
source nested in a git repository, and for skipping clean/non-git targets. Preserve and adapt the existing Models alias
helper and panel tests to prove the shared extraction does not change its prompt, commit message tagging, tracked-task
metadata, or completion behavior.

Add Config-pane widget regression tests that exercise a successful `AppliedResult` and verify:

- the write toast and refresh still occur before offer handling;
- a dirty chezmoi source produces the canonical `ConfirmActionModal` with the expected relative subject and y/n labels;
- declining or dismissing the prompt submits no task;
- confirming submits one deduplicated `config-commit` tracked task for the actual written source file;
- clean/non-git results and cancelled/failed edits produce no prompt;
- success, git failure, and stale-lock outcomes surface the established notifications without blocking navigation.

Reuse the existing confirmation-modal visual styling and snapshots unless the integration exposes a genuinely new visual
state; the regression should primarily assert that the Config tab pushes the canonical modal rather than a bespoke
dialog. Run the focused Config-pane, Config-edit, Models-panel, and commit-helper test modules, then run `just install`
followed by the repository-required `just check` before handoff.

## Acceptance criteria

A user who edits a Config-tab field with `e` and writes into a dirty chezmoi source repository is asked, via the
standard y/n popup, whether to commit and push that exact source file. Confirming performs the established tracked
commit/pull/push flow and reports the result; skipping leaves the successful local write and chezmoi apply intact. No
prompt appears when there is nothing eligible to commit, existing Models alias editing behaves identically, and neither
offer discovery nor git execution blocks the TUI event loop.
