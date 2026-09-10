---
tier: epic
title: Obsidian-aware completion and highlighting for Bob Mac Capture
goal:
  Typing an Obsidian wikilink in the Bob Mac Capture popup produces fast, accurate,
  keyboard-first suggestions and polished semantic highlighting without moving capture
  grammar or vault authority into Swift.
phases:
  - id: link_protocol
    title: Authoritative Obsidian link protocol in bob-cli
    depends_on: []
    size: medium
    description:
      "link_protocol: extend bob-cli's editor-facing parse and completion contracts with
      a shared, vault-aware Obsidian wikilink scanner, note/alias/heading/block
      discovery, byte-exact replacements, deterministic ranking, additive JSON metadata,
      documentation, and exhaustive Rust coverage while preserving existing capture
      behavior and the lexical no-I/O guarantee of capture-parse."
  - id: caret_integration
    title: Caret-correct link intelligence in Bob Mac Capture
    depends_on:
      - link_protocol
    size: medium
    description:
      "caret_integration: update the linked bob-mac-capture app to decode the additive
      contract, drive completion from the real AttributedTextSelection insertion point,
      preserve selection and IME state while applying highlights, accept server-provided
      replacements with a restored caret, and harden cancellation, stale-response,
      Unicode, and failure behavior with fixture-backed Swift tests."
  - id: visual_polish
    title: Beautiful, accessible completion presentation and release gate
    depends_on:
      - caret_integration
    size: medium
    description:
      "visual_polish: refine the popup's wikilink palette and completion rows into an
      adaptive native macOS experience with clear note/path/alias/subpath hierarchy,
      matched-result emphasis, keyboard and VoiceOver feedback, responsive sizing,
      documentation, automated regression coverage, and a focused
      light/dark/high-contrast/IME owner-assisted validation gate."
proposed_by: bbugyi200.athena.00w.f0.f0.w0.w0
status: done
create_time: 2026-09-09 20:00:16
---

- **PROMPT:**
  [prompts/202608/obsidian_link_completion.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/obsidian_link_completion.md)

# Obsidian-aware completion and highlighting for Bob Mac Capture

## Outcome

Make Obsidian linking feel native inside the Bob Mac Capture popup. Typing `[[sas`
should immediately offer `sase.md`; accepting the suggestion should produce a valid,
closed `[[sase]]` link and leave the caret in the expected place. The same experience
should understand nested paths, aliases, embeds, headings, and block references, remain
correct when editing in the middle of a Unicode draft, and look intentional in every
macOS appearance and accessibility mode.

The implementation keeps the existing architectural boundary:

- `bob-cli` owns Obsidian-link syntax recognition, vault discovery, ranking, exact
  replacement text/ranges, and all capture semantics.
- `bob-mac-capture` owns caret/selection plumbing, async orchestration, presentation,
  keyboard/pointer interaction, accessibility, and macOS visual treatment.
- The wire changes are additive under schema version 1. Existing capture-marker contexts
  (`route`, `section`, `pomodoro_block_id`, and `task`) and every existing capture
  behavior remain backward compatible.

This is an epic because it introduces a protocol in `bob-cli` and then consumes that
protocol in a distinct linked macOS repository. The app phases deliberately depend on
the CLI phase so no worker has to guess candidate syntax or duplicate a parser.

## Product behavior and design decisions

### Supported wikilinks

Recognize and semantically highlight complete and in-progress forms anywhere in the
capture body, including:

- notes and paths: `[[sase]]`, `[[Projects/Alpha]]`, and an unfinished `[[sas`;
- embeds: `![[sase]]`;
- display text and frontmatter aliases: `[[Artificial Intelligence|AI]]`;
- headings: `[[sase#Design]]`, including same-destination `[[#Design]]`;
- blocks: `[[sase#^block-id]]`, including same-destination `[[#^block-id]]`;
- Obsidian's vault-wide heading/block searches, `[[##query]]` and `[[^^query]]`.

The scanner must not activate on escaped brackets or inside inline/fenced code, must not
cross a newline looking for an unfinished link, and must recover predictably from nested
or repeated `[[` while the user is still typing. Unsupported attachment links remain
untouched and can still be highlighted lexically, but this feature's discovery index is
intentionally limited to Markdown notes; binary attachment indexing is a separate
product decision.

The standard Obsidian grammar is the source of truth: vault-root paths use `/`, aliases
are emitted as `[[canonical target|alias]]`, headings follow `#`, and block identifiers
follow `#^`. The relevant stable references are
[Internal links](https://obsidian.md/help/links) and
[Aliases](https://obsidian.md/help/aliases).

### Completion behavior

The cursor, not the end of the draft, determines the active component. Completion is off
for a non-collapsed selection, ordinary text, a closed link when the caret is outside
its editable components, escaped/code-literal syntax, and malformed contexts that cannot
be replaced safely.

Contexts are additive values in `capture-complete`:

- `wikilink_note` searches Markdown note paths, stems, and aliases.
- `wikilink_heading` searches headings in a resolved target, the eventual capture
  destination for `[[#...]]`, or the whole vault for `[[##...]]`.
- `wikilink_block` searches named block IDs in the analogous target or vault-wide
  `[[^^...]]` scope.

`capture-complete` returns the exact candidate `replacement` and half-open UTF-8 byte
range. It owns delimiter completion: selecting from `[[sas` inserts the missing `]]`,
while selecting inside `[[sas]]` reuses the existing close rather than duplicating it.
It also owns alias expansion, subpath separators, and the final post-accept caret byte
offset. Add an optional `cursor_after` field to the response/candidate contract rather
than making Swift infer whether a suffix was synthesized. A stale or invalid range is
rejected without modifying the draft.

For note candidates, insert a vault-relative path without `.md`; top-level `sase.md`
therefore remains the compact `[[sase]]`, while `Projects/Alpha.md` becomes
`[[Projects/Alpha]]`. Paths are explicit for nested notes so duplicate stems never
silently resolve to the wrong file. Selecting an alias emits the interoperable
`[[path|Alias]]` form Obsidian itself uses.

Ranking is case-insensitive, deterministic, and capped to a documented result budget:

1. exact alias, stem, or path matches;
2. prefix matches, with basename/stem before deeper path matches;
3. word-boundary and acronym matches;
4. substring matches;
5. stable normalized path order as the final tie-breaker.

An empty query offers a bounded, stable list rather than serializing the entire vault.
Candidate metadata describes why an item matched, but the app does not rerank it.

### Vault discovery and reliability

Build a small dedicated Markdown-note index rather than reusing the full Dataview task
index or recursively invoking another `bob` process. It should recursively enumerate
regular `.md` files, never follow directory symlinks, normalize separators to `/`, and
reuse Bob's always-excluded note-directory policy for `.git`, `.obsidian`, `_generated`,
and `_templates`; hidden directories and `.trash` must not become suggestions. Read only
the strictly closed frontmatter block needed for `aliases`, accepting the documented
YAML list form and tolerating a legacy scalar defensively.

Root discovery failure is actionable. A single unreadable note or malformed alias
property must not erase every otherwise valid path candidate: keep path-only metadata
when possible, return bounded additive warnings, and never create or modify the vault.
Resolve headings and blocks only after a target is known; vault-wide search may scan all
eligible files but must remain bounded and cancellable at the app process boundary.

### Semantic highlighting

`capture-parse` stays purely lexical and read-only. A shared wikilink scanner returns
ordered, non-overlapping byte spans that are merged with existing capture-marker spans
and validated as one set. Use semantic kinds rather than one flat link color:

- `wikilink_delimiter` for `![[`, `]]`, `#`, `#^`, and `|` punctuation;
- `wikilink_target` for the note/path;
- `wikilink_heading` for heading text;
- `wikilink_block_id` for a block identifier;
- `wikilink_alias` for display text.

Highlighting is syntax-only: `capture-parse` must not touch the vault merely to color a
target and must not paint an unresolved note as an error. Existing capture diagnostics
remain the only diagnostic source unless the lexical link structure itself is unsafe.

In the app, punctuation uses a quiet secondary tone, the target uses the system accent,
headings use adaptive purple, block IDs use adaptive indigo, and aliases use adaptive
teal. Preserve sufficient contrast in light, dark, increased-contrast, and non-default
accent configurations; do not encode meaning by color alone. Apply attributes without
replacing the entire attributed value in a way that resets selection, typing attributes,
undo, or marked-text composition.

### Completion presentation

Keep the completion surface in normal layout flow below the editor so it never covers
the draft. Reuse the panel's material, rounded geometry, measured content sizing, and
five-row scroll budget, but give wikilink results a clearer information hierarchy:

- a compact context label and SF Symbol (`doc.text`, heading, or block) identifies what
  will be inserted;
- the primary line shows the matched note/alias/heading with restrained match emphasis;
- the secondary line shows the canonical vault-relative path and, where applicable, an
  `alias`, heading-level, or block badge;
- the selected row uses an accent-tinted rounded fill that works in light/dark and
  increased-contrast modes;
- long values truncate from the middle where preserving the basename is more useful than
  preserving the path prefix.

No raster artwork or bespoke icon asset is needed. Use system materials, SF Symbols,
native fonts, and semantic colors. Existing Return/Tab acceptance, arrow and
Control-N/Control-P navigation, Escape dismissal, scrolling, pointer selection, and
panel autosizing remain consistent across capture-marker and wikilink contexts.

VoiceOver must announce the context, candidate count, selected result, canonical path,
alias/subpath metadata, and the effect of acceptance without reading decorative
punctuation as noise. Status messages distinguish no matches, unavailable vault data,
and a transport failure without logging private draft text.

## Phase `link_protocol`: authoritative Obsidian link protocol in bob-cli

Work in the primary `bob-cli` repository.

1. Add a focused shared module (for example `src/native/capture_links.rs`) that scans
   complete/incomplete wikilinks, models their semantic components, and identifies the
   active completion field at any valid UTF-8 cursor boundary. Keep the existing
   capture-marker tokenizer/classifier authoritative for capture semantics; merge link
   spans into its editor result instead of teaching link syntax to execution parsing.
   When the cursor is inside a wikilink, link completion takes precedence over marker
   completion so an `@` or `%` inside link text can never be mistaken for capture
   syntax.
2. Add a dedicated read-only note index (factored separately if clearer) for eligible
   paths, frontmatter aliases, ATX headings, and named block IDs. Reuse existing
   Markdown fence/frontmatter/heading helpers and directory-exclusion policy instead of
   copying subtly different rules. Resolve same-destination subpaths from the editor
   parse's explicit route, falling back to `mac_inbox.md` exactly as `bob capture` does.
3. Extend `CaptureCompleteResult` with the three wikilink contexts and an untagged
   candidate shape carrying exact replacement text plus optional `path`, `name`,
   `alias`, `match_kind`, `heading`, `level`, `block_id`, and short block/heading
   display metadata. Add `cursor_after` and bounded `warnings` additively. Preserve all
   existing candidate fields and schema version 1.
4. Merge the new semantic spans into `CaptureParseResult`, preserving ordered,
   non-overlapping scalar-boundary invariants. Update human rendering, `--help`, command
   tables, README contracts/examples, and privacy/no-mutation guarantees for the
   expanded existing commands; no new public subcommand or option is necessary.
5. Add Rust unit and CLI integration tests for complete/incomplete syntax, embeds,
   aliases, same-note/target-qualified/vault-wide subpaths, escaped/code-literal input,
   delimiter recovery, cursor positions before/inside/after links, closing-bracket
   deduplication, Unicode/combining characters/CRLF, span merging, duplicate stems,
   nested paths, malformed YAML, unreadable entries, exclusions, symlink loops, result
   limits, ranking tiers, empty/no-match results, stable JSON, and unchanged legacy
   marker contexts. Assert `capture-parse` performs no I/O and link completion performs
   no writes.

Validation for this phase:

```sh
just fmt
just lint
just test
just install-smoke
git diff --check
```

Also exercise a generated large temporary vault to record warm/cold completion timing
and response bounds. Treat a material regression as a design problem to fix, not as a
reason to add a persistent vault cache or daemon in this feature.

## Phase `caret_integration`: caret-correct link intelligence in Bob Mac Capture

Open the linked app with a specific audited reason before reading or editing it:

```sh
sase repo open bob-mac-capture -r "Implement the approved Obsidian link completion integration"
```

1. Extend `CaptureCore`'s additive response models for wikilink metadata, warnings, and
   `cursor_after`; keep unknown future keys decodable. Update the fake `bob`, process
   client tests, and schema model tests with realistic note, alias, heading, and block
   responses.
2. Replace the end-of-draft approximation in `AutosizingCaptureEditor`. Observe both
   text and `AttributedTextSelection` changes, call `selection.indices(in:)`, and derive
   the actual UTF-8 insertion offset from a collapsed insertion point. Suppress
   completion for range/multi-selection while continuing parse/highlight/preview work.
   Cover the beginning, middle, and end of ASCII and multi-scalar text.
3. Keep text and selection synchronized during highlighting using the macOS 26
   attributed-string transformation APIs rather than reconstructing
   `AttributedString(draft)` on every parse. Preserve the caret, selected range, typing
   attributes, undo registration, and IME marked text. If the native rich `TextEditor`
   still cannot satisfy those invariants under the Mac gate, use the already-approved
   narrow fallback of an AppKit `NSTextView` adapter inside the existing editor
   boundary; do not move parsing or completion into AppKit.
4. Request `capture-complete` whenever the authoritative parse reports a wikilink span
   containing the insertion point, not only when capture `needs` or marker spans match.
   Keep one cancellable completion lane and generation checks; a superseded process,
   late response, or warning must never replace newer results. Run live preview without
   waiting on a vault-wide completion scan once parse has established that the two
   operations are independent.
5. Apply the server byte range and `cursor_after` atomically. Restore a collapsed
   selection at that offset, suppress only the programmatic callback for that accepted
   revision, refresh parse/highlighting/preview, and keep completion dismissed until the
   next user edit or caret movement. Stale ranges or non-boundary cursors leave the
   draft unchanged and produce a concise status.
6. Extend model/controller tests for real-caret requests, caret-only requery, Unicode
   conversion, mid-draft acceptance, synthesized/deduplicated closes, alias expansion,
   selection suppression, selection survival through highlighting, stale-generation
   rejection, cancellation, transport/index warnings, failure recovery, and unchanged
   capture-marker completion behavior.

Validation for this phase, from the linked repository, uses its Xcode-resolved wrapper:

```sh
just format-lint
just build
just test
git diff --check
```

## Phase `visual_polish`: beautiful, accessible completion presentation and release gate

Work in the linked `bob-mac-capture` repository after `caret_integration`.

1. Centralize a semantic editor palette and apply the five new span kinds with adaptive
   colors and restrained emphasis. Ensure incomplete delimiters are legible but quiet,
   ordinary prose remains primary, and existing capture route/section/schedule/priority/
   clipboard colors still form one coherent palette.
2. Refactor completion-row presentation around a small context-specific view model so
   route, section, task, note, heading, and block results each render useful metadata
   without option-heavy branching in SwiftUI. Add the wikilink context label/icon,
   primary match emphasis, canonical path, semantic badges, selection treatment,
   truncation policy, hover/pointer behavior, and accessibility label/hint.
3. Keep the list's measured in-flow placement, five-row viewport, selected-row
   auto-scroll, panel screen clamp, editor/footer priority, and unanimated window
   sizing. Check narrow widths, long nested paths, large Dynamic Type, and auxiliary
   preview or error content together so beauty does not come at the cost of hidden
   actions.
4. Update the app README's runtime contract, keyboard/completion guidance,
   troubleshooting, and privacy language. Describe supported link forms, alias
   insertion, exact caret behavior, and the fact that note metadata is read locally by
   `bob` and never logged by the app.
5. Add focused presentation/view-model/layout/accessibility tests where behavior is
   testable without screenshots. Run the full format/build/test/bundle checks, then
   validate the installed popup manually on macOS 26 in light, dark, increased contrast,
   Reduce Motion, default and alternate accent colors, keyboard-only use, VoiceOver,
   long paths, empty/no-match/error states, Unicode, and an IME composition session.

Final automated gate:

```sh
just format-lint
just build
just test
just bundle
git diff --check
```

The phase handoff must record which physical Mac checks ran. It must not claim an
installed-app, signing-identity, VoiceOver, or IME result that was not actually
observed.

## Acceptance criteria

- Typing `[[sas` anywhere in a draft shows `sase.md` promptly; acceptance yields one
  valid `[[sase]]`, restores the caret exactly, and never submits the capture.
- Nested note paths and duplicate stems are unambiguous; alias selection produces
  canonical `[[path|Alias]]`; heading and block completion work for target-qualified,
  same-destination, and vault-wide forms described above.
- Complete and incomplete links receive semantic delimiter/target/subpath/alias
  highlighting without vault I/O in `capture-parse`, overlap corruption, selection
  jumps, undo loss, or IME breakage.
- Completion follows the real collapsed caret, works in the middle of Unicode text, and
  rejects stale or malformed byte ranges without changing the draft.
- Results are deterministic, bounded, exclude private/generated configuration areas,
  tolerate individual bad notes, and surface true root/transport failures without
  silently falling back or logging the draft.
- The list remains fully keyboard and pointer operable, readable with VoiceOver and
  increased contrast, visually coherent in light/dark and alternate accent colors, and
  never overlaps the editor, preview, errors, or footer actions.
- Existing capture parsing, route/section/task completion, preview, submission,
  autosizing, shortcuts, privacy, and wire compatibility remain intact; the complete
  validation suites pass in both repositories.

## Cross-phase constraints

- Preserve unrelated worktree changes. Every access to `bob-mac-capture` must go through
  `/sase_repo` and use the path that command prints; do not embed an ephemeral workspace
  path in implementation artifacts.
- Use `apply_patch` for source edits. Do not commit, push, deploy, install, or change
  the active app unless separately authorized by the user or a post-completion
  finalizer.
- Do not add a resident daemon, persistent vault index, direct vault mutation in Swift,
  copied Swift parser/ranker, WebView, custom raster asset, or new public CLI command.
- Treat draft text, aliases, paths, heading text, and block previews as private. Keep
  them out of logs, notifications, signposts, diagnostics history, and test artifacts
  built from the real vault.
- Each phase ends with focused tests, its repository's full available suite,
  `git diff --check`, and a diff review against this plan. Discovered unrelated work is
  handled through the project's required SASE follow-up workflow rather than folded into
  this feature.
