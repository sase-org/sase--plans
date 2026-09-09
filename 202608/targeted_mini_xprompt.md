---
tier: epic
title: Targeted mini-xprompt authoring in the ACE prompt stack
goal: "Users can create or edit one non-swarm xprompt in a focused, pane-scoped ACE
  workflow, while whole-stack saves, frontmatter-local helpers, and existing
  targeted-xprompt behavior remain clear, safe, and visually distinct.

  "
phases:
  - id: target_catalog
    title: Mini-xprompt target catalog and name panel
    depends_on: []
    size: medium
    description:
      "target_catalog: build the cached target-resolution model and live-completing
      new-or-existing xprompt name panel."
  - id: pane_mode
    title: Pane-scoped mini-xprompt editing lifecycle
    depends_on:
      - target_catalog
    size: medium
    description:
      "pane_mode: add the dedicated mini-xprompt pane role, scoped frontmatter editing,
      focus restoration, and dirty-draft guards."
  - id: persistence
    title: Conflict-safe mini-xprompt saves and live publication
    depends_on:
      - pane_mode
    size: medium
    description:
      "persistence: implement create, overwrite, reload, and failure-safe save flows
      through existing atomic xprompt writers."
  - id: keymap_polish
    title: Keymap migration, visual polish, documentation, and regression audit
    depends_on:
      - persistence
    size: medium
    description:
      "keymap_polish: complete the g-prefix migration, theme-safe presentation,
      documentation, snapshots, and end-to-end verification."
proposed_by: bbugyi200.athena.08v
bead_id: sase-rl
create_time: 2026-09-09 19:51:42
status: wip
---

- **PROMPT:**
  [prompts/202608/targeted_mini_xprompt.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/targeted_mini_xprompt.md)
- **BEAD:**
  [sase-rl](https://github.com/sase-org/sase--beads/blob/main/pages/sase-rl/README.md)

# Plan: Targeted mini-xprompt authoring in the ACE prompt stack

## Outcome and design contract

Add a third prompt-bar target state: a **targeted mini-xprompt** is one dedicated
prompt-input pane that edits exactly one simple xprompt definition. It is deliberately
different from both existing states:

- A targeted xprompt continues to bind the entire agent-prompt stack and may represent
  an xprompt swarm.
- A targeted snippet continues to bind one auxiliary pane to one snippet trigger.
- A targeted mini-xprompt binds one auxiliary pane to one simple, non-swarm xprompt. It
  never becomes an agent segment and is excluded from launch, stash, whole-stack
  save-as, pane counts, reorder, and prompt history payloads.

The mini-xprompt pane is bottom-pinned and visually part of the prompt stack, following
the proven targeted-snippet interaction. Opening it captures the originating agent pane,
cursor, and Vim mode. Closing it after save or discard restores that exact state, with
cursor clamping if the source pane changed meanwhile. The original agent pane is never
overwritten implicitly.

For a new, unused name, the mini-xprompt starts with a copy of the originating pane's
body so `gx` is a fast extraction workflow. It does **not** copy shared prompt-stack
frontmatter: `gX` remains the explicit whole-draft operation for saving all panes plus
shared frontmatter. Mini mode does apply the established raw-placeholder conversion to
its copied body and seeds only the inferred xprompt inputs. For an existing name, it
loads that definition's body and frontmatter instead. Opening or retargeting an existing
mini pane preserves the user's current mini draft, matching snippet rename/retarget
semantics; selecting a definition from a fresh agent-pane request loads that definition.

Only one auxiliary authoring target may be active at a time. Opening a mini-xprompt
while a snippet target is open, or vice versa, must reuse the same generic dirty-draft
guard: a clean auxiliary pane can be replaced directly, while a dirty one requires an
explicit discard confirmation. This keeps stack operations and focus restoration
deterministic.

Mini mode accepts Markdown- or config-backed simple xprompts. YAML workflow graphs,
skills, memory definitions, and definitions whose body contains a real top-level `---`
swarm separator are shown truthfully in name resolution but cannot be opened as mini
targets; the UI points users to the XPrompt Browser / external editor or the whole-stack
targeted flow. Fenced separators remain literal. The same single-prompt invariant is
revalidated before every save so a mini draft cannot silently become a swarm.

## User interaction

The final default prompt-local keymap is:

| NORMAL | INSERT     | Result                                                            |
| ------ | ---------- | ----------------------------------------------------------------- |
| `gx`   | `Ctrl+G x` | Open or retarget one mini-xprompt pane.                           |
| `gX`   | `Ctrl+G X` | Open the existing unified whole-stack xprompt/snippet save panel. |
| `gL`   | `Ctrl+G L` | Convert the active agent pane into a frontmatter-local xprompt.   |
| `gt`   | `Ctrl+G t` | Open or retarget one snippet pane (unchanged).                    |
| `gw`   | `Ctrl+G w` | Write the existing whole-stack targeted xprompt (unchanged).      |

Retain `Ctrl+G Ctrl+X` as the compatibility alias for the unified whole-stack save panel
unless the implementation discovers a concrete terminal conflict. It must no longer be
displayed as an alias of lowercase `x`; help and hint metadata should group it with
uppercase `X`.

`gx` opens a keyboard-first **Open mini-xprompt** panel:

1. The name field validates with `validate_xprompt_name()` as the user types.
2. A cached list shows up to six prefix matches from the effective xprompt catalog. Rows
   distinguish editable simple xprompts, read-only/override candidates, swarms,
   workflows, skills, and memory definitions. `Tab` completes the highlighted match;
   `Up`/`Down` and `Ctrl+P`/`Ctrl+N` navigate without stealing typing focus.
3. A destination panel shows the resolved file/config entry and discovery precedence. A
   fresh name defaults to the last-used writable xprompt destination, then the first
   writable standard destination. An exact editable match defaults to its actual source
   so Enter edits in place. A read-only match defaults to a writable override target.
   Users may cycle writable destinations for a deliberate fork/override.
4. One verdict line says exactly what Enter will do: create, edit, fork/override,
   shadow/be-shadowed, or refuse an incompatible target. All path and definition reads
   happen before the modal opens or in debounced off-thread analysis; keystroke handlers
   operate only on cached data and discard stale results.

Entering the mini pane uses INSERT mode. Its separator reads left-to-right as a
`#<name>` chip, middle-elided destination, and truthful `new`, dim `✓`, gold `●`, or
`⚠ changed on disk` state. Its subtitle advertises Enter to review/save, `Ctrl+C` to
discard, `gx` / `Ctrl+G x` to rename or retarget, and `g=` / `Ctrl+G =` for properties.
The mini pane gets a theme-derived accent related to targeted xprompts but distinct from
the snippet pane's `⇥` visual language; no hard-coded background fill may reduce
contrast in light themes.

The existing Frontmatter Panel becomes scope-aware. On an agent pane it continues to
edit shared prompt-stack frontmatter. On the mini pane it edits that mini target's own
frontmatter and labels the scope with `#<name>`. Switching panes first persists the
current panel model, then swaps the panel view without leaking mini inputs or local
helpers into the launch stack. Mini-pane completion may use the mini definition's local
xprompts; agent panes continue to use only stack frontmatter.

Enter in a mini pane opens a focused save review. New definitions show Draft; existing
definitions default to a unified Diff and can cycle Draft / Existing / Diff. Empty
bodies, invalid frontmatter, incompatible skill placement, or a real swarm separator
block saving with actionable feedback. A byte-identical existing definition closes as
`No changes` without writing. `Ctrl+C` discards through the dirty guard; Escape only
returns the editor to NORMAL mode.

## Phase `target_catalog`: Mini-xprompt target catalog and name panel

Build a reusable, typed target index rather than teaching the modal to inspect the
filesystem on every change. Reuse `UnifiedSaveLocation`, xprompt discovery order,
`markdown_save_plan()`, `resolution_after_save()`, source classification, and the shared
definition loader where their contracts already fit. Extract shared target-resolution
helpers from the unified save panel instead of duplicating namespace, config-entry,
chezmoi, or collision rules.

The index should retain, per callable name and physical definition, the workflow kind,
source/display path, storage format and entry name, editability, effective-precedence
status, compatibility with mini mode, and enough metadata to build a binding or writable
override target. Load it off the Textual event loop. Names from a warm app catalog may
be used as the immediate view, but opening the panel must schedule a fresh off-thread
snapshot so newly written xprompts and project context are correct.

Implement a dedicated name modal modeled on `SnippetNameModal`, with pure helpers for
prefix ranking, exact-match selection, destination cycling, validation, and verdict
construction. Guard programmatic `OptionList.highlighted` updates synchronously so
highlight echoes cannot move the cursor. Cache analysis by `(name, destination)` and
apply only the latest requested identity.

Tests cover name validation, namespace/storage-name mapping, directory and config
targets, duplicate definitions across precedence layers, last-used destination fallback,
chezmoi write-target resolution, exact editable/read-only/incompatible matches, prefix
ordering and Tab completion, destination navigation, and stale async analysis
suppression. Modal tests also prove typing and navigation do no synchronous disk I/O.

## Phase `pane_mode`: Pane-scoped mini-xprompt editing lifecycle

Generalize the prompt-stack model's auxiliary target role so an item is exactly one of
agent, snippet target, or mini-xprompt target. Preserve compatibility-facing snippet
properties where useful, but centralize `agent_items` filtering, bottom pinning,
selection, removal, dirty detection, and the single-auxiliary invariant. The mini target
model owns its name/reference, read/write/apply paths, storage format and config entry,
frontmatter, loaded body/document, source fingerprint, collision warning, and clean
hash. Keep filesystem methods off the model and event loop.

Add prompt-bar messages and app handlers that capture the exact origin bar and
generation-scoped pane id before loading the target catalog. After modal dismissal,
reject stale/unmounted origins. A fresh-name result copies only the origin pane body,
performs raw-placeholder-to-input conversion using the existing configuration flag, and
opens a bottom-pinned mini pane. An existing compatible result loads body and
frontmatter off-thread and opens it. Retargeting an already-open mini pane changes its
destination/comparison metadata while preserving its draft.

Route active-pane behavior by pane role:

- Enter requests mini save review instead of launching.
- `Ctrl+C` requests guarded discard instead of recording cancelled history.
- add, remove, reorder, stash, launch, local-xprompt conversion, search, editor, and
  whole-stack save capture continue to act only on agent items.
- Any action that would dismiss or replace the bar first guards a dirty auxiliary
  target. Keeping the draft refocuses it; confirming discard proceeds exactly once.
- Focus, cursor, and Vim mode restore exactly after close. Stack rebuilds clear stale
  completion and frontmatter-panel state.

Refactor the Frontmatter Panel integration around an explicit scope accessor rather than
scattering mini-mode checks. Persist panel changes to the selected scope, swap the
rendered model when focus crosses the auxiliary boundary, and preserve raw invalid YAML
without accidental loss. Verify prompt-local xprompt completion remains isolated by
scope.

Model and widget tests cover new/existing opens, active-body copying, placeholder input
inference, incompatible target refusal, retarget-without-body-loss, auxiliary
replacement guards, exclusion from every agent payload, single- and multi-agent stacks,
frontmatter scope switching, completion isolation, stale origin results, cursor/mode
restoration with clamping, and interaction with a clean or dirty snippet target.

## Phase `persistence`: Conflict-safe mini-xprompt saves and live publication

Add an xprompt-specific pane save review modeled on the snippet confirmation but backed
by the existing xprompt serializers and atomic writers. Render the complete Markdown or
config-entry preview so frontmatter changes are visible. For existing sources, compare
the captured fingerprint immediately before write. A changed source must never be
silently overwritten: offer reload, explicit overwrite, or retarget/save-as. Reload
refreshes body, frontmatter, binding, clean hash, and fingerprint without closing the
pane. A failed read, validation, serialization, write, fingerprint refresh, or chezmoi
apply leaves the draft mounted and retryable.

Use the same write primitives and post-write chooser as unified/whole-stack xprompt
saves for Markdown files, config entries, canonical names, atomic replacement, chezmoi
source/apply targets, and optional commit-and-push follow-ups. Preserve unchanged
Markdown source bytes/comments whenever the existing bound-write contract can do so.
Never hold the app pump during definition reads, hashing, diff generation,
serialization, writes, config refresh, or follow-up preparation.

After success, invalidate the names/definition index, publish or schedule a fresh app
prompt catalog so `#` completion, `#@`, the Browser, and the next `gx` see the new
definition in the same session, record the last-used xprompt destination, notify with
create/write/no-change wording, close the mini pane, and restore the origin. Do not bind
the surrounding agent stack to the saved source.

Tests cover new Markdown and config xprompts, namespaced filenames/frontmatter,
overwrites, no-change, read-only override, precedence warnings, comments/body
preservation, external changes with reload/overwrite/retarget, destination deletion,
chezmoi redirects, write and refresh failures, post-write actions, immediate catalog
visibility, successful close/restore, and failed-write draft retention. Include a
heartbeat/thread assertion for slow I/O.

## Phase `keymap_polish`: Keymap migration, visual polish, documentation, and regression audit

Update the single declarative prompt `g`-prefix table so dispatch and hints migrate
together:

- lowercase `x` calls the mini-xprompt target request;
- uppercase `X` calls the existing unified whole-stack save request and owns the
  `Ctrl+G Ctrl+X` compatibility alias;
- uppercase `L` calls active-pane frontmatter-local conversion and keeps the caller's
  INSERT/NORMAL target mode.

Update availability and contextual labels for agent, snippet, mini, empty,
frontmatter-only, targeted whole-stack, and multi-pane states. Update static help,
subtitles, docs, and every old `gx`/`gX` reference, including the raw-placeholder
documentation. Do not introduce configurable keymap fields for these hard-coded
prompt-local continuations; if implementation does add or change configuration, update
`src/sase/default_config.yml`, schema/default tests, and migration guidance in the same
phase.

Add theme-safe CSS and width-aware rendering for the name panel, mini separator,
frontmatter scope chip, state markers, save review, narrow terminals, and parked/active
states. Add PNG snapshot coverage for at least fresh-name completion, editable-existing
selection, incompatible/swarm verdict, new/clean/dirty/stale mini panes, scoped
frontmatter, and save diff; include light-theme or palette-safety coverage where color
contrast is material. Accept intentional golden updates only after inspecting actual,
expected, and diff artifacts.

Finish with focused unit/widget/modal suites, keymap/help/documentation assertions,
visual snapshots, and repository verification. Run `just install` before verification,
then `just check`; because this touches prompt-stack behavior, g-prefix routing,
frontmatter, save targets, and visual snapshots, run the dedicated visual suite and use
`/sase_monitor` for `just check-full` before landing the combined epic.

## Compatibility and non-goals

- Existing Browser, `gd`, and other surfaces continue to use whole-stack targeted
  xprompt editing; this plan adds a faster prompt-local loop rather than changing those
  semantics.
- Existing `gw` remains whole-stack write-back. Mini panes use Enter and their own
  confirmation flow, avoiding an ambiguous `gw` target when both a stack binding and an
  auxiliary pane are visible.
- No xprompt loader or swarm execution semantics change. Mini-mode single-prompt checks
  use the canonical parser and are a UI authoring constraint only.
- No feature flag is needed: this is an atomic keymap migration delivered with its
  complete behavior, documentation, and tests. If phases must land separately in a
  user-visible branch, introduce the flag with `sase flag new` after reading the flag
  memory, and remove it in the final phase.
