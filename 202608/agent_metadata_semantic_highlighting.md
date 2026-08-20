---
tier: tale
title: Highlight glossary terms and repositories in agent prompts
goal:
  Agent xprompts and prompts use reliable, beautiful, project-aware semantic
  highlighting consistent with the prompt input.
size: medium
proposed_by: bbugyi200.athena.08k
create_time: 2026-08-20 10:48:41
status: wip
---

# Semantic glossary and repository highlighting in agent prompts

## Goal

Make glossary terms and configured repository names immediately recognizable in the
Agents-tab metadata panel's `AGENT XPROMPT` and `AGENT PROMPT` sections. Match the
prompt input's semantic language: glossary terms use a bold, underlined tint derived
from the active theme's primary color, while repository mentions use a distinct bold,
underlined tint derived from the accent color. Preserve the panel's existing Markdown,
xprompt, project-name humanization, file-hint, folding, and section-navigation behavior.

The finished behavior should feel like the read-only continuation of the prompt input:
the same project context recognizes the same terms and repos, the same concepts have the
same visual roles, and cold or invalid catalogs never delay or damage the panel.

## Design

### One semantic overlay contract

Introduce a small, presentation-only semantic-overlay utility for Rich `Text` regions.
It should accept already-warm `EditorGlossaryCatalog` and `EditorRepoMentionCatalog`
values, scan through the existing Rust-backed glossary and repo matchers, convert every
UTF-16 segment range to Python text offsets, and apply all styles only after scanning
succeeds. Extract the duplicated editor-range conversion helpers currently private to
the prompt glossary and repo-mention mixins so the prompt input and metadata panel share
one Unicode- and wrapped-term-safe conversion path.

Keep the two semantic roles intentionally distinct and theme-adaptive. Factor the prompt
input's existing style derivation into a shared TUI helper rather than copying color
formulas: glossary terms derive from `primary`; repo mentions derive from `accent`; both
are bold and underlined. Include a compact style signature for rendered cache keys so
switching between dark and light themes cannot reuse stale colors.

The overlay must be bounded by the existing prompt/Markdown byte and line caps and must
fail open. A missing, cold, malformed, or failed catalog leaves the original text and
all existing styles visible. Match the prompt input's precedence contract:

1. Markdown supplies the base presentation.
2. Glossary and repo roles annotate natural-language text.
3. Xprompt/directive/skill syntax wins inside structural xprompt tokens.
4. Inline and fenced code retain code styling without semantic underlines.
5. Numbered file hints are restored last and remain the strongest local affordance.

Use the existing repo catalog rather than matching `agent.linked_repos` directly. That
preserves the prompt input's case handling, glossary-name exclusions, sidecar/external
repo support, and rejection of path-adjacent text such as `../sase-core`.

### Agent-scoped, memory-only catalog context

Expand the prompt-panel highlighting context helper so one agent produces a stable
semantic context and cache fingerprint. Resolve the project first from the raw VCS
xprompt tag, then from the agent's project file, with the agent workspace as the
resolver fallback. Read catalogs only through the app's existing in-memory prompt
catalog getters. A render may request an asynchronous warm, but it must never read,
stat, glob, parse config, acquire an unbounded store lock, or run a subprocess.

When a glossary or repo warm/invalidation completes, extend the existing prompt-catalog
surface refresh to schedule the selected Agents detail through its established
`DetailPanelDebouncer`. This should repaint a cold panel when semantics arrive, remove
stale semantics after invalidation, coalesce the independent glossary/repo completions,
and preserve active numbered-hint mode. Do not add an alternate detail-refresh path.

Put catalog/style fingerprints into both the small prompt-body render caches and the
annotated hint-document cache key. This makes cold-to-warm publication, config
invalidation, project changes, and theme changes deterministic without clearing global
caches or rescanning unchanged documents on every paint.

### Apply semantics only to authored prompt sections

Give the panel separate render helpers for authored prompts and generic Markdown.
`AGENT XPROMPT` should keep its existing Markdown and xprompt token highlighting, with
glossary/repo semantics layered underneath structural tokens. `AGENT PROMPT` should gain
glossary/repo semantics while remaining ordinary Markdown. Replies, chats, tracebacks,
tool output, monitor output, and unrelated metadata must continue through the generic
Markdown/plain-text path and must not acquire prompt semantics merely because they
repeat a term or repo name.

Route every direct authored-prompt presentation through these helpers: the normal agent
view, numbered file-hint view, family-container prompt view, a pinned attempt's
invariant `AGENT PROMPT`, and the top-level workflow's attempted `AGENT PROMPT`.
Preserve the current section bodies, line breaks, folding, project display-name
humanization, hint numbers/mappings, and large-content fallback. Aggregate clan
`PROMPTS` previews are not direct `AGENT XPROMPT`/`AGENT PROMPT` documents and remain
outside this change.

For normal prompt bodies, keep the existing capped/lazy path when no semantic catalog is
warm or content exceeds the semantic highlight cap. Once catalogs are warm, cache the
bounded highlighted Rich text by content, catalogs, and style signature. For hint mode,
apply semantic spans to the already-humanized, hint-annotated visible region, then
restore structural xprompt and numbered-hint spans in precedence order.

## Implementation map

- Refactor the prompt glossary/repo range conversion and theme-derived semantic roles
  into shared TUI utilities, retaining the prompt input's current visible behavior and
  tests.
- Extend the Rich prompt-syntax utility with a fail-open glossary/repo overlay that can
  target a complete `Text` or an offset region, coexist with Markdown/xprompt syntax,
  skip code-literal interiors, and enforce existing size caps.
- Expand the agent prompt highlighting context to resolve an agent project/workspace,
  obtain warm catalogs through the app cache API, schedule cold warms, and expose a
  conservative fingerprint including known skills, both catalog identities/signatures,
  and theme-derived styles.
- Add cached `_render_agent_prompt`/semantic xprompt paths to the prompt-panel renderer;
  thread the agent context through normal, family, pinned-attempt, and workflow prompt
  sections while leaving all response renderers generic.
- Apply the same overlay and precedence rules to regular and family numbered-hint
  renderers, and include semantic context in the hint-render cache digest.
- Extend glossary/repo catalog publication and invalidation refreshes to the existing
  debounced Agents detail route, including active hint mode and theme-driven cache
  refresh.

## Acceptance criteria

- In both `AGENT XPROMPT` and `AGENT PROMPT`, configured glossary terms and their
  effective aliases receive the glossary role, while configured linked, sidecar, and
  external repository identifiers receive the visually distinct repo role.
- Matching behavior agrees with the prompt input for case, punctuation, wrapped
  multi-line terms, non-BMP Unicode offsets, glossary/repo name collisions, and
  path-adjacent repo-like text.
- Glossary and repo roles are distinguishable, readable, and consistent in dark and
  light themes; a theme switch cannot retain the previous theme's cached colors.
- Markdown, inline/fenced code, xprompt invocations/directives/skills, project display
  names, and numbered file hints retain their existing styling and precedence.
- Normal, numbered-hint, family, pinned-attempt, and top-level workflow prompt sections
  behave consistently. Replies/chats and other metadata do not gain these overlays.
- A cold catalog renders immediately with the current presentation, schedules at most
  the app's coalesced warm, and repaints the still-selected agent through the detail
  debouncer when the result arrives. Failed/empty catalogs remain plain and error-free.
- Rendering performs no filesystem or subprocess work, does not scan beyond the existing
  caps, reuses bounded caches for unchanged content, and preserves the large prompt
  fallback.

## Verification

Add focused unit and pilot coverage for:

- shared UTF-16/wrapped-segment conversion and semantic overlay ordering, including code
  literals, xprompt overlap, non-BMP text, malformed spans, and oversized input;
- agent project/workspace resolution, cold/warm getter behavior, catalog/style cache
  fingerprints, cold-to-warm repaint, invalidation, and coalesced debounced refresh;
- normal and numbered-hint `AGENT XPROMPT` plus `AGENT PROMPT` rendering, ensuring file
  hints and xprompt styles still win and generic replies remain unstyled;
- family, pinned-attempt, and workflow authored-prompt paths;
- cache reuse for unchanged text and misses after agent, catalog, config, or theme
  changes.

Update the existing Agents xprompt PNG fixture so both sections visibly contain a
glossary term and a linked-repo mention, retain the xprompt palette and inline code, and
add a light-theme companion snapshot. Inspect the actual/expected/diff artifacts before
accepting intentional goldens.

Before handoff, run `just install`, the focused non-visual tests, the dedicated visual
snapshot test with intentional golden updates, and `just check`. If scoped selection
escalates or reports unusual selection, follow project guidance and run
`just check-full` through `/sase_monitor`.
