---
tier: tale
title: Slash-skill preview and definition navigation
goal:
  Prompt-input K preview and Ctrl+] go-to-definition recognize xprompt skills written as
  slash invocations, while preserving file-path and hash-xprompt behavior without adding
  synchronous work to the TUI keypress path.
create_time: 2026-09-09 19:53:28
status: wip
---

# Plan: Slash-skill preview and definition navigation

## Context and desired behavior

The prompt input's NORMAL-mode `K` and `Ctrl+]` commands already share the preview/jump
target machinery for `#xprompt` references and file paths. They label a resolved
`XPrompt` with `skill: true` as a skill, but their lexical detector does not recognize
the slash form that users actually write when invoking an agent skill (for example,
`/sase_plan`). Since the file-path matcher accepts absolute single-component paths,
`/sase_plan` currently falls through as a file target instead of reaching the xprompt
resolver.

Extend both commands so that, with the cursor anywhere on a known slash-skill token:

- `K` opens the existing preview modal for that skill's definition/content.
- `Ctrl+]` opens the existing definition-navigation choices at the skill's actual source
  file and location.
- The displayed reference uses the slash spelling where appropriate and the payload
  remains labeled as a `skill`.
- Unknown slash words, genuine absolute/relative file paths, `#xprompt` references,
  workflow references, and existing file line/column suffixes keep their current
  behavior.
- Slash-like text inside fenced code, inline code, or disabled xprompt regions is not
  treated as an actionable skill, matching the existing xprompt syntax-inspection
  affordance.

This is a presentation-layer editor affordance. Slash skills remain literal prompt text
rather than launch-time xprompt syntax, so no Rust-core or launch-parser change is
needed.

## Design

### Recognize known slash skills before file paths

Teach the shared preview-target detector to accept the current set of known skill names
and reuse `sase.xprompt.xprompt_inspect`'s slash-skill spans. That scanner already owns
the strict token boundaries and protected region rules used for slash-skill
highlighting, including rejection of unknown, greedy, and path-like matches. Convert the
span under the cursor into the existing xprompt-shaped target (`target` is the name
without `/`, while the raw or display reference retains `/name`) before running file
detection. This ordering resolves the current `/sase_plan` versus absolute-file
ambiguity without weakening file recognition.

Thread the optional known-skill set through jump-target detection, which deliberately
builds on preview detection, so `K` and `Ctrl+]` cannot drift. Keep default arguments
compatible with the pure helper tests and any callers that do not have a catalog
context.

### Use the app-owned warm catalog on the keypress path

Have the prompt actions obtain skill names from the existing memory-only, project-aware
prompt catalog helper used by slash-skill highlighting. The selected catalog must follow
the leading VCS reference in the active prompt, with the existing prompt context as the
resolution fallback, so detection and `get_xprompt_or_workflow` lookup see the same
source precedence.

Do not build catalogs, read definitions, stat/glob files, or otherwise touch disk
synchronously while handling `K` or `Ctrl+]`. Preserve the existing worker-based
resolution and request-id stale-result guards. If the relevant catalog is cold, request
its existing background warm and fail/defer cleanly rather than performing a synchronous
catalog scan or silently treating a plausible skill token as the wrong kind; the normal
startup/prompt-mount warm remains the fast path. Revalidate during background resolution
that a slash-form target still resolves to an xprompt exposed as a skill so a stale
catalog entry cannot navigate to an unrelated non-skill definition.

### Preserve slash identity through resolution and presentation

Carry enough token metadata for preview and jump payloads, titles, and user-facing
failures to distinguish `/name` from `#name` while continuing to use the same underlying
xprompt/workflow source resolver. Avoid duplicating source classification,
editable/loadable Markdown handling, YAML definition-line lookup, modal construction, or
editor/tmux launch behavior. A slash skill should therefore gain all existing preview
and go-to-definition actions automatically, including source-file preview, config/plugin
source resolution, and read-only handling.

The current help modal already documents both keymaps as operating on
“xprompt/skill/file,” and the bindings are hard-coded NORMAL-mode commands, so neither
help copy nor `default_config.yml` needs a new entry unless implementation changes the
visible interaction contract.

## Verification

Add focused coverage at each existing seam:

- Preview-target unit tests recognize `/sase_plan` only when it is a known skill, honor
  inclusive/exclusive cursor boundaries, give it precedence over the absolute-file
  matcher, preserve its slash display identity, and reject
  unknown/path-like/protected-region text. Retain explicit regression cases for real
  absolute files and `#xprompt` references.
- Jump-target unit tests pass the known-skill context through shared detection and
  resolve a slash skill to the same source path, body line, loadability, and editability
  as its hash-form xprompt definition.
- NORMAL-mode widget tests seed the warm catalog and prove both `K` on `/sase_plan` and
  `Ctrl+]` on `/sase_plan` reach their existing modal/action flows with the expected
  skill token, project, and base-directory context. Keep the counted-command and
  dot-repeat protections intact.
- Cold-catalog tests prove neither command performs a synchronous catalog build and that
  the warm/defer behavior does not misclassify unrelated slash paths. Include
  stale-entry/error coverage so no modal/editor action occurs after the skill ceases to
  resolve as a skill.
- Run the focused preview/jump/xprompt-inspection test modules, then `just install` and
  the required `just check` suite. Existing visual snapshots should remain valid because
  this reuses the current modals; update them only if an intentional visible
  slash-reference change affects a covered snapshot.

## Expected implementation surface

The primary changes should remain localized to the prompt preview/jump target helpers
and their two action mixins, plus their existing unit/widget test modules. Reuse the
prompt text area's warm-skill-name cache rather than adding a second catalog cache. If a
shared detector caller such as prompt `g`-prefix availability needs the new optional
catalog argument to preserve current hints, update that caller for consistency, but do
not broaden this tale into unrelated prompt navigation or xprompt launch semantics.
