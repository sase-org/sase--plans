---
tier: tale
title: Artifact-reference completion in the ACE prompt bar
goal:
  ACE completes kind-tagged artifact references from warm providers without regressing ordinary @path completion or
  performing keystroke-path I/O.
bead: sase-av.6
create_time: 2026-07-29 14:39:11
status: done
---

- **PROMPT:** [prompts/202607/artifact_ref_prompt_completion.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/artifact_ref_prompt_completion.md)
- **PARENT:** [202607/artifact_refs_and_prompt_bar.md](https://github.com/sase-org/sase--plans/blob/main/202607/artifact_refs_and_prompt_bar.md)
- **BEAD:** [sase-av.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-av/sase-av.6.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-av.6--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-av.6.md#member-code)
  - [bbugyi200.athena.sase-av.6--plan](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-av.6.md#member-plan)
- **COMMITS:**
  - [e55aab9](https://github.com/sase-org/sase/commit/e55aab9c92f73f5f902fa58ee39641da6a78686a) — feat(ace): add artifact reference prompt completion

# Plan: Complete kind-tagged artifact references in the ACE prompt bar

## Goal

Add two-stage `@<kind>:<payload>` completion to the ACE prompt input without changing ordinary `@path` completion.
Completion must identify incomplete artifact references before the generic `:`-delimited tokenizer, offer dynamic
document-role and builtin kinds, serve payloads entirely from warm memory, reopen across the kind/payload boundary, and
surface recorded artifact references in the existing recent-file history menu.

## Current architecture and constraints

- `PromptTextArea` owns one shared completion state machine spread across `_file_completion_context.py`,
  `_file_completion_open.py`, `_file_completion_tab.py`, `_file_completion_refresh.py`, `_file_completion_accept.py`,
  and the prompt-bar panel renderer. Artifact completion must participate in all of these paths instead of creating a
  parallel popup.
- Generic token extraction deliberately treats `:` as a delimiter. Artifact-reference detection therefore needs a
  dedicated cursor-context pass before generic token/path handling, analogous to the VCS-ref detectors.
- The existing artifact-reference highlighter already resolves the prompt's target project, warms its known-kind set
  off-thread, and treats a cold cache as indeterminate. Extend or share that project-scoped warm state so completion
  does not repeat synchronous role discovery.
- Prompt keystroke handlers and refresh paths are read-only, subprocess-free, and I/O-free. Plan/document discovery,
  artifact-index reads, chat scans, stats, and any other filesystem work must occur in a thread worker. Commit and bug
  candidates may only be projected from already-loaded Artifacts-pane snapshots.
- A bare `@` and ordinary `@~/...`, `@/...`, and similar tokens retain the current file-path behavior. Automatic
  artifact menus start only after two kind characters or once a syntactically valid kind is followed by `:`.
- The prior launch-processing phase already records well-formed artifact references verbatim in file-reference history,
  so history integration should reuse the existing source and deletion behavior rather than add another store.
- ACE option changes require matching defaults, JSON schema, help-popup, and user documentation updates.

## Implementation

1. Add a pure artifact-completion model and detector under `src/sase/ace/tui/widgets/`.
   - Define an `artifact_ref` completion-kind constant, a cursor context containing the absolute replacement span, stage
     (`kind` or `payload`), partial kind/payload, resolved known kind when applicable, and a stage-specific panel title.
   - Detect `@`, `@<partial-kind>`, `@<kind>:`, and `@<kind>:<partial-payload>` around the cursor before generic token
     extraction, including cursor positions in the middle of a payload. Enforce the artifact-ref left-context and kind
     grammar, stop at prompt/literal delimiters, and decline contexts that are clearly ordinary file paths.
   - Define typed metadata for kind and payload rows. Build candidates with stable case-insensitive prefix filtering,
     deterministic ranking, canonical insertions, and shared-prefix extension. Kind rows include the four builtins plus
     the target project's dynamic document roles; payload rows retain the full `@kind:` prefix.

2. Build and own a bounded, project-scoped warm payload catalog outside the keystroke path.
   - Reuse the highlighter's project/workspace resolution and known-kind warm lifecycle, expanding the worker result to
     carry immutable document, artifact-file, and chat candidates. Document rows come from the plan-search facade using
     the context's role/root pairs, are newest-first, and show relpath plus title metadata. Artifact-file rows come from
     the explicit artifact index and carry id, label, kind, and age metadata, with reuse keyed by index mtime/size. Chat
     rows come from a bounded `iter_chat_files` scan, are recency-sorted, and insert paths relative to the chats root.
   - Cache successful snapshots by target project, coalesce duplicate warms, and publish worker results on the UI
     thread. On a miss, completion returns no candidates and schedules a warm; it never falls through to a synchronous
     provider.
   - At candidate-build time, merge optional commit and bug rows from the mounted Artifacts panes' already-warm,
     immutable snapshots. Commits insert `<repo>@<full-sha>`; bugs insert `<display-project>#<number>`. Never launch
     git, contact a provider, or trigger pane loading from prompt completion.
   - Warm the active project's catalog when a prompt pane mounts/rebuilds beside the existing xprompt, known-kind, VCS,
     model, history-word, and placeholder warmers. Refresh only an active artifact menu when a matching catalog worker
     result lands.

3. Integrate the new context through the shared completion state machine.
   - Give artifact detection priority in manual `Ctrl+T`, automatic menu opening, active-menu refresh, token-range
     lookup, and acceptance. Keep VCS/directive/xprompt precedence intact where their grammars already claim a cursor.
   - Add `auto_artifact_menu` (default `true`) to `PromptCompletionSettings`, parsing, `default_config.yml`, and
     `sase.schema.json`. Auto-open only for `@` plus at least two kind characters or any valid `@kind:` context, and
     continue respecting insert mode, the completion debounce, and existing navigation/activity guards.
   - Accepting a kind replaces the detected range with `@<kind>:` and immediately rebuilds/reopens the artifact menu at
     the payload stage. Accepting a payload replaces the full artifact context and closes the menu. Retyping or moving
     within the context recomputes from warm candidates while preserving the selected insertion when possible.
   - Render artifact rows with useful stage-specific metadata and panel titles (`artifact kinds`, `<kind>: documents`,
     `file: artifacts`, `chat: chats`, `commit: commits`, or `bug: bugs`) using the existing shared panel. Artifact
     history rows remain `@`-form recent-file candidates with the existing `Ctrl+D` affordance.

4. Document and verify the complete behavior.
   - Update the prompt help popup to mention artifact-reference completion and update `docs/ace.md`'s completion-kind,
     automatic-menu, history, and settings descriptions.
   - Add pure tests for detection at every cursor position, invalid/ordinary `@path` exclusions, kind/payload filtering,
     ranking, metadata, and shared prefixes, using a fixture dynamic role named `designs`.
   - Add widget tests for manual and automatic opening, cold-cache empty/schedule behavior, all five payload providers,
     stage-boundary accept/reopen, active refresh, panel titles/rows, mid-payload replacement, and the bare-`@` path
     regression.
   - Add history round-trip/deletion coverage for recorded `@`-form artifact refs and settings/schema/help
     synchronization tests.
   - Add a warm-cache performance guard that patches disk/index/corpus/chat/git/tracker entry points to fail if the
     detector, candidate builder, auto-open, refresh, or accept keystroke paths touch them.

## Validation

1. Run focused artifact-completion, prompt-history, prompt-settings/schema, and existing `@`-file completion tests while
   iterating.
2. Run `just install` before repository validation, then run `just check`.
3. Re-run the focused warm-cache performance guard and bare-`@` regression after any fixes from the full check.

## Non-goals

- Do not change the Rust artifact-reference grammar, launch expansion, or LSP completion.
- Do not add network-backed commit or bug collection, a new history store, CLI commands, keybindings, or document-role
  names hardcoded in Python.
- Do not close or otherwise mutate the parent epic.
