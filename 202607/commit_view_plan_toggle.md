---
tier: tale
title: View associated plans from the commit panel
goal: 'Every commit opened in the shared Commit panel can temporarily display its
  locally available SASE_PLAN document with p, then restore the same commit view with
  p again, with clear feedback when no usable plan is attached.

  '
create_time: 2026-07-21 11:10:57
status: wip
prompt: '[202607/prompts/commit_view_plan_toggle.md](prompts/commit_view_plan_toggle.md)'
---

# Plan: View associated plans from the commit panel

## Context and outcome

The Artifacts tab's Commits sub-tab and commit hints selected through `v` on the Agents tab already converge on
`CommitViewModal`. The Artifacts path builds view specs from `VcsLogResult` entries; the Agents path builds the same
specs from persisted `meta_commits` records. The modal owns multi-commit navigation, lazy diff loading, rendering,
caching, and its local footer, so the new behavior should be implemented once at that shared boundary rather than
separately in the two callers.

Add a modal-local `p` binding that treats the current commit's terminal `SASE_PLAN` footer as the source of truth. On
the first press, the modal should show the associated plan as bounded, syntax-highlighted Markdown. On the next press,
it should restore the exact commit and its already-loaded or still- loading diff rather than reopen the modal or rerun
the commit query. A commit without a `SASE_PLAN` tag, or one whose plan cannot be resolved or read locally, must leave
the commit view intact and produce a concise notification that names the commit and explains the problem.

## Commit-to-plan identity and resolution

Use the existing Rust-backed commit-footer parser rather than scanning message text. Only a structured trailing
`SASE_PLAN` tag should qualify; ordinary body text and legacy unprefixed tags should not accidentally activate the
feature. For a Markdown-linked tag, use its readable label as the local plan reference and retain the destination only
as metadata for diagnostics; pressing `p` must remain a local, prompt-free operation and must not fetch a remote file.
The parser's existing duplicate-tag behavior should continue to select the last value.

Preserve enough owning-project context on each `CommitViewSpec` to resolve a plan independently of the repository that
contains the commit. Agent-created specs can carry the selected agent's workspace identity. VCS-log specs need the one
or more project workspace roots that own each resolved primary, linked, or sidecar repository; extend the
frontend-neutral log-repository metadata and its current-project, explicit-project, all-project, and fallback resolution
paths so this context is computed during repository resolution, not by doing inventory work in a modal key handler. Keep
new fields optional so existing callers and serialized/render-only uses remain compatible. When a physical linked
repository belongs to more than one project, retain deterministic candidate owners instead of discarding all but one
during deduplication.

Factor the stable plan-reference rules already used by the Artifacts Plans document loader into a reusable read-only
resolver rather than introducing a second path dialect. Given the commit repository, candidate owning workspaces, and
their resolved SDD plans roots, cover every form emitted by commit handling:

- absolute and home-relative paths;
- paths relative to the repository or owning workspace, including arbitrary in-repository plan locations;
- `sdd/plans/...` and `.sase/sdd/plans/...` in-tree/local-store references;
- `plans/...` separate-store references; and
- flat `YYYYMM/...` sidecar-plan references.

Resolve candidates without materializing or synchronizing stores, accept only an existing regular file, and return a
typed result that distinguishes a missing tag, an invalid reference, a missing local file, an unreadable file, and
successful UTF-8 Markdown content. Include the displayed reference and the resolved local path in successful plan-view
metadata so the title and errors are informative. Keep the existing Artifacts Plans behavior covered while sharing the
resolver.

## Modal behavior and rendering

Introduce explicit commit/plan display state in `CommitViewModal`, separate from the current commit index and per-index
diff cache. Pressing `p` in commit mode should immediately switch the content area to a lightweight loading state and
start footer parsing, SDD resolution, stat/read work, and any other filesystem access in a thread worker. Tag the worker
result with the commit index/identity that launched it, cancel it on unmount or when it is superseded, and ignore stale
completions after a toggle or commit navigation. Cache a successful plan payload per commit for the modal lifetime so
returning to it does not add repeated disk work.

Pressing `p` while the plan (or its loading state) is visible should cancel any obsolete plan load, restore commit mode
for the same index, reset the scroll position, and reuse the existing diff state/cache. The diff worker may finish while
a plan is displayed; its result should still populate that commit's cache without replacing the visible plan. Moving
with `ctrl+n` or `ctrl+p` while a plan is visible should exit the temporary plan view before selecting and rendering the
next commit, so navigation never carries one commit's plan into another commit. Existing wrapping, SHA copying,
scrolling, and close bindings must remain unchanged.

Render plan mode through the existing bounded lazy Markdown renderer and syntax cache so large plans receive the same
truncation safeguards as other TUI document surfaces. Change the title from `COMMIT` to `PLAN` while showing a plan,
retain the repository, commit position, and SHA context, and show the plan reference/resolved path without requiring new
layout CSS unless visual verification demonstrates a real need. Make the local footer advertise `p plan` in commit mode
and `p commit` in plan mode. On load failure, return to the unchanged commit view and notify with warning/error severity
appropriate to “no SASE_PLAN tag” versus “tag exists but the local plan is unavailable.”

Document the modal control in the relevant Artifacts/Commits and Agents help content while respecting the help modal's
fixed box widths. Keep `p` local to `CommitViewModal`; it is not a global or conditionally advertised app-footer binding
and therefore should not be added to the configurable application keymap registry.

## Verification

Add focused resolver/footer tests for plain and Markdown-linked `SASE_PLAN` values, rejection of non-terminal or
unprefixed lookalikes, duplicate tags, all supported repository/store-relative path forms, multiple owner candidates,
and useful typed failures for absent, missing, non-file, non-UTF-8, and unreadable plans. Extend repository-resolution
and view-spec tests to prove that both Artifacts commits and Agents commit hints preserve the owning workspace context
and full tagged message needed by the worker.

Exercise the modal with Textual pilot tests that prove:

- `p` loads and renders the current plan, updates title/footer, and resets scroll without blocking the event loop;
- a second `p` restores the same commit and cached diff;
- missing-tag and unresolved/read-error cases notify clearly and retain the commit content;
- toggling back during a load and navigating during a load cannot apply a stale plan result;
- commit navigation from plan mode restores commit mode and preserves wrapping; and
- existing copy, scroll, close, diff-cache, and multi-commit behavior remains intact.

Update the existing Commit modal PNG golden for the new footer and add a plan- mode snapshot if it materially improves
coverage of the alternate title and Markdown body. Run the focused commit-modal, commit-pane, agent commit-hint,
repository-resolution, shared plan-document, help-modal, and visual tests, then run `just install` followed by the
mandatory `just check` for the full repository gate.
