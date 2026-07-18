---
tier: tale
title: Make newly saved snippets live in open ACE prompts
goal: 'A snippet saved from the ACE prompt panel is immediately expandable from every
  already-mounted prompt input, including when use_chezmoi writes only the source
  tree first, while catalog refreshes remain non-blocking and eventually converge
  on the applied config.

  '
create_time: 2026-07-18 08:56:30
status: done
prompt: 202607/prompts/live_saved_snippets.md
---

# Plan: Make newly saved snippets live in open ACE prompts

## Context and root cause

The prompt widget itself is already live: every `Tab` expansion asks `AceApp.get_snippets()` for the current app-owned
registry, so an existing `PromptTextArea` does not retain a construction-time copy. The stale behavior occurs before
that lookup, in the save-to-catalog publication pipeline.

There are two distinct freshness gaps to close:

- A direct YAML write is followed immediately by `load_merged_config()`. Config tokens now use stale-while-revalidate
  semantics, so that call can return the pre-write merged object. `_refresh_snippet_caches()` then republishes the old
  `ace.snippets` mapping and asks a forced prompt-catalog build to read through the same stale token.
- With `use_chezmoi: true`, the save panel intentionally writes the chezmoi source file, while the running TUI reads the
  applied home config. The optional commit/push task applies chezmoi later, and its completion currently does not
  refresh the prompt catalog. A cache clear at initial save time therefore cannot make this case live, and applying
  chezmoi automatically would improperly broaden the save action.

Preserve the existing app-owned catalog, generation guards, worker coalescing, snippet-reference semantics, and
source-precedence behavior. Do not remount prompt bars or introduce a per-widget snippet cache.

## Immediate known-write publication

Add a small app-owned mapping of pending snippet saves, keyed by trigger and containing the raw template that was
successfully written by the unified save panel. Record an entry only after the YAML write succeeds. A failed or
cancelled save must not change the running registry.

After a successful write, build a provisional effective snippet registry off the Textual event loop from the last good
in-memory registry plus all pending saves, then run the normal `resolve_snippet_references()` composition over that
merged mapping. Atomically publish the finished mapping to `_snippets_cache` before showing the success toast or opening
the commit confirmation. This keeps existing xprompt-derived and user snippets available, makes overwriting a trigger
take effect in the same session, and lets a newly saved snippet reference an existing snippet without waiting for a disk
rescan.

Keep pending saves separate from `_user_snippets`: the latter represents the effective merged config, whereas a chezmoi
source write may deliberately be ahead of the applied config. The pending mapping is session state only; the saved YAML
remains the durable source of truth.

## Catalog rebuild and convergence

Thread a snapshot of the pending saves through the existing coalesced prompt-catalog rebuild. The off-thread builder
must overlay them after xprompt-derived snippets and effective `ace.snippets` have been assembled, but before snippet
references are resolved. This prevents a watcher event, startup warm, project-catalog expansion, or older in-flight
build from erasing a just-saved snippet. Continue to bump the catalog generation for each successful save so a build
captured before that save cannot publish over it.

Extend the catalog snapshot with the raw effective user-snippet mapping needed for convergence. When a fresh snapshot
shows that an effective configured trigger has the same raw template as a pending save, retire that pending entry; the
normal catalog now owns it. Pending entries that are not yet present in live config remain overlaid for the rest of the
session, which makes a chezmoi source save usable even if the user skips or has not yet completed commit/push/apply.
Saving the same trigger again replaces the pending value deterministically.

After a successful commit/push/chezmoi-apply task for a snippet, request an explicit config-backed catalog refresh so
the applied file is observed promptly and the matching pending entry can converge. Keep the callback scoped to the
snippet save flow; xprompt save wording and unrelated commit tasks should retain their behavior. A failed commit or
apply reports its existing error and leaves the successfully written snippet available through the session overlay.

## Fresh config events without UI-thread I/O

Make known config writes and config-file watcher events bypass the stale config-token window. Carry a `config_dirty`
signal through the prompt-source debounce and rebuild coalescing state, including a pending dirty request while another
build is in flight. The catalog worker—not the watcher callback, render path, or key handler—must explicitly invalidate
and freshly load merged config before computing/publishing the new snapshot. Xprompt-file-only events should retain the
cheaper existing path.

This restores the original external-edit auto-reload contract as well as the save-panel flow: a single inotify burst
must not be lost merely because `current_config_token()` was still serving its previous value. Keep all filesystem
stats, YAML parsing, directory walks, config reloads, and snippet-reference recomputation off the Textual event loop; UI
work is limited to flags, generation changes, snapshot swaps, and selective refresh of already-visible surfaces.

## User-facing behavior and documentation

The availability boundary is successful snippet persistence: once the TUI reports `Created` or `Saved`, typing that
trigger into any prompt input that was already open and pressing `Tab` expands the saved body. This applies to direct
user/project config destinations and to chezmoi source destinations before optional deployment completes. Existing
tabstop handling, trigger validation, collision behavior, prompt drafts, xprompt completion, and the
`Ctrl+G Ctrl+X Ctrl+X` chord stay unchanged.

Update the prompt/ACE snippet documentation to state that panel-saved snippets become available immediately in the
current TUI and to explain the session behavior for chezmoi-backed saves without implying that SASE automatically
applies unconfirmed chezmoi changes. No keymap, modal layout, visual snapshot, CLI, or Rust-core change is required.

## Regression coverage

Add focused coverage at the save, catalog, watcher, and real widget boundaries:

- Prime the real merged-config cache, save a snippet to a direct config file, and prove the fresh effective mapping is
  published despite a warm stale token.
- Separate a chezmoi source config from the applied config, save a snippet, and verify the same already-mounted
  `PromptTextArea` instance expands it before apply. Force an intervening catalog rebuild and verify it cannot erase the
  pending snippet.
- Simulate successful apply, refresh the catalog, and verify the configured snapshot retains the snippet while the
  matching pending entry is retired. Cover skipped/failed commit behavior and a second save of the same trigger.
- Cover snippet-reference resolution and preservation of pre-existing xprompt-derived snippets in the provisional and
  rebuilt registries.
- Exercise generation/coalescing races so an older build cannot overwrite a newer pending save and a queued
  `config_dirty` request is not dropped.
- Simulate one config watcher burst while the config token is warm and assert the external edit reaches the catalog
  without a restart.
- Hold config/catalog loading in a worker while an event-loop heartbeat continues, guarding the no-blocking-input-path
  requirement.

Run `just install` before checks. Iterate with the targeted prompt-save, prompt-catalog, watcher, and snippet-expansion
tests, then run the required full `just check`. Visual goldens need no update unless implementation unexpectedly changes
rendered UI, in which case treat that as a regression rather than accepting new snapshots.
