---
tier: epic
status: done
title: Task-section capture targeting with `@route+block-id#section`
goal:
  Typing `@foo+bar#requirements` in `bob capture` or the Bob Mac Capture panel files the
  captured note under the `REQUIREMENTS` bullet of the `^bar` task in `foo.md`, with
  `#`-triggered completion that lists that task's real sections.
phases:
  - id: sections_core
    title: Shared task-section scanner in bob-cli
    depends_on: []
    size: small
    description:
      "sections_core: add one authoritative bob-cli module that enumerates a task's
      ALL-CAPS child section bullets using the same title whitelist the
      bob-navigation-hotkeys Ctrl+Shift+Alt+N conversion uses, plus a canonical
      whitespace-free slug, selector matching, and insertion geometry, covered by unit
      tests and used by no CLI surface yet."
  - id: grammar
    title: Three-component `@route+block-id#section` marker grammar
    depends_on:
      - sections_core
    size: medium
    description:
      "grammar: extend bob-cli's capture grammar so a sub-bullet marker accepts a
      trailing `#<section>` component, with new span, need, and completion-field values,
      precise usage errors for every malformed shape, additive `capture-parse` output,
      updated help/README, and exhaustive lexical tests that keep the bare trailing `#`
      Pomodoro-note marker and `@route#section` note-bullet marker unchanged."
  - id: execution
    title: Task-section resolution and insertion in `bob capture`
    depends_on:
      - grammar
    size: medium
    description:
      "execution: teach `bob capture` to resolve the selected task section and append
      the captured bullet inside it, add the forced `-S/--task-section TITLE` picker
      option, report the matched section in human and JSON output, raise actionable
      errors with suggestions when no section matches, and cover insertion geometry,
      line-ending preservation, and dry-run parity with CLI tests."
  - id: discovery
    title: "`bob capture-task-sections` and `task_section` completion"
    depends_on:
      - grammar
    size: medium
    description:
      "discovery: add the read-only `bob capture-task-sections` subcommand and the
      `task_section` context in `bob capture-complete`, both backed by the shared
      scanner, with slug replacements, deterministic ranking, bounded warnings for an
      unresolvable parent task, registration in the command table and install smoke
      test, documentation, and integration tests."
  - id: mac_app
    title: "`#`-triggered task-section completion in Bob Mac Capture"
    depends_on:
      - discovery
    size: medium
    description:
      "mac_app: decode the additive `task_section` context and `sub_bullet_section` span
      in the linked bob-mac-capture app, open the completion popup as soon as `#`
      follows a resolved `@route+block-id`, render task-section rows with the parent
      task and a section-content badge, keep every existing context and the bare `#`
      Pomodoro-note marker unaffected, and document plus test the behavior."
proposed_by: bbugyi200.athena.085
create_time: 2026-09-09 19:59:56
---

- **PROMPT:**
  [prompts/202608/capture_task_sections.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/capture_task_sections.md)

# Task-section capture targeting with `@route+block-id#section`

## Outcome

`bob capture 'Postgres 17 minimum @foo+bar#requirements'` appends

```markdown
- Postgres 17 minimum
```

under the `REQUIREMENTS` bullet that lives beneath the `^bar` task in `~/bob/foo.md`,
instead of at the end of that task's block. In the Bob Mac Capture panel, typing `#`
immediately after a resolved `@foo+bar` opens the completion popup listing exactly the
sections that task actually has, so the section never has to be typed from memory.

This is an epic because it introduces a shared vault-scanning primitive, extends the
two-component marker grammar to three components, changes `bob capture` execution, adds
a new read-only subcommand plus a new completion context, and then consumes all of that
from a separate linked macOS repository. The architectural boundary is unchanged:

- `bob-cli` owns the marker grammar, the task-section definition, discovery, ranking,
  exact replacement text and byte ranges, and every vault mutation.
- `bob-mac-capture` owns caret plumbing, async orchestration, presentation, keyboard
  interaction, and accessibility.
- All wire changes are additive under schema version 1. Existing contexts (`route`,
  `section`, `pomodoro_block_id`, `task`, and the three wikilink contexts) and every
  existing capture behavior stay backward compatible.

`grammar` depends on `sections_core` so the two phases do not race on `capture.rs`: the
scanner promotes shared list helpers first, then the grammar adds the
`CaptureKind::SubBullet` section field and updates every constructor. `execution` and
`discovery` can then proceed in parallel. `mac_app` consumes the discovery contract.

## Product behavior and design decisions

### What a task section is

The definition is taken from the `Ctrl+Shift+Alt+N` project-note conversion in the
`bob-navigation-hotkeys` plugin (`bob-plugins` linked repo,
`plugins/bob-navigation-hotkeys/main.js`). Open that repo through `/sase_repo` and read
`PROJECT_SECTION_TITLE_RE`, `PROJECT_CHILD_LIST_ITEM_RE`, `PROJECT_LIST_ITEM_RE`,
`parseProjectSectionBulletTitle`, `parseProjectChildListItem`, and
`buildProjectSeedFromChildBullets` before implementing; do not re-derive the rule from
prose. Do not edit `bob-plugins` in this epic.

A list item `L` is a **task section** of task `T` in note `N` when all of the following
hold:

1. `L` is a direct child list item of `T`: it lies inside `T`'s `block_end`, its
   indentation is strictly greater than `T`'s, and `nearest_shallower_list_item_parent`
   returns `T`'s line. That is the same ancestry `capture.rs` already uses for managed
   Schedule/Work logs. It agrees with the plugin's shallowest-child-indent rule on
   consistently indented children and, unlike the plugin, still treats mixed-indent
   siblings as direct children of `T`. Typical vault notes are consistently indented; do
   not "fix" this back to the plugin's shallowest-only rule.
2. `L` has no task checkbox. The plugin's `PROJECT_CHILD_LIST_ITEM_RE` treats an
   optional single-character `[x]` plus following whitespace as a checkbox
   (`status !== null`). `capture.rs`'s `list_item_body` does **not** strip that prefix:
   `- [ ] REQUIREMENTS` yields body `[ ] REQUIREMENTS`. The scanner must parse the
   checkbox itself. A checkboxed ALL-CAPS child is never a section.
3. `L`'s body, trimmed, matches `^[A-Z0-9][A-Z0-9 \t&'(),./-]*$` and contains at least
   one `A-Z`. This is the plugin's deliberately narrow whitelist: wikilinks, tags, block
   IDs, inline code, and `snake_case` are all excluded, and an all-digit body is
   rejected by the at-least-one-letter rule. Internal whitespace is **not** collapsed
   before the regex (the plugin does not collapse it either); collapse only when
   computing the slug and the display title's comparison key.

The plugin additionally requires a section bullet to already have a nested nonblank list
item under it before the next direct child, because a childless bullet would seed an
empty `##` section. **Capture deliberately does not require that lookahead.** Requiring
it would make `@foo+bar#requirements` fail on a freshly authored, still-empty
`REQUIREMENTS` bullet — precisely the moment the feature is most useful — with an error
about a section the user can plainly see. Everything else about the definition is
identical, and `child_count` is exposed on every discovery result so a picker can badge
a section that is still empty. Record this divergence in a comment next to the predicate
so a later reader does not "fix" it back to parity.

Managed logs keep their existing meaning: a `🗓️ **SCHEDULE LOG**` or `🛠️ **WORK LOG**`
direct child is identified by `parse_managed_task_log_marker` and never offered as a
section, even if a lookalike title would pass the whitelist. A plain `- SCHEDULE LOG`
bullet with no bold/emoji wrapper **is** a valid section, matching the plugin.

Display `title` is the trimmed original body (`REQUIREMENTS`, `FUTURE WORK`), not the
plugin's title-cased project-header form (`Requirements`, `Future Work`). Capture is
filing into the vault bullet the user already sees.

### Marker grammar

The new form is `@<route>+<block-id>#<section-selector>`, a single whitespace-free token
in the existing leading-or-trailing marker position.

- `is_sub_bullet_marker_candidate` already claims a token whose `+` precedes any `:`,
  `#`, or `^`, so `@foo+bar#requirements` is routed to the sub-bullet family today and
  fails only because `bar#requirements` is not a valid block ID. The grammar change is
  therefore additive within one family and cannot steal a token from the note-bullet,
  Pomodoro, task-with-ID, or retired `::` families.
- Today `parse_sub_bullet_route_token` splits on the first `+` and validates the entire
  right-hand side as a block ID. The new split is `+` then a single `#`: route,
  block-id, optional section selector. A second `#`, or `#` before `+`, is not this
  family (`@foo#bar+baz` remains a note-bullet whose section prefix is `bar+baz`).
- An existing unit test in `capture.rs` currently asserts that `body @foo+bad#section`
  is a sub-bullet **usage error** ("plus must take precedence"). After this change that
  input is a **valid** sub-bullet targeting section `section` of task `bad`. Update that
  assertion. `body @foo^bad#section` stays a task-with-ID usage error (`^` family,
  `bad#section` is not a block ID). `body @foo+bad:id` stays a sub-bullet usage error
  (`:` is not a block-ID byte and is not a section separator).
- The **section selector** is compared as a slug. Define
  `slug(s) = trim, collapse internal whitespace to one space, ASCII-lowercase, then replace each space with '-'`.
  So `FUTURE WORK` and `FUTURE  WORK` both have slug `future-work`, `REQUIREMENTS` has
  slug `requirements`, `Q&A` has slug `q&a`, and `NON-GOALS` has slug `non-goals`. A
  typed selector is slugged the same way, which is what makes a multi-word section
  reachable from a token that cannot contain whitespace.
- A typed selector may contain ASCII alphanumerics (either case) and `& ' ( ) , . / -`.
  Anything else (notably `^`, `:`, `+`, a second `#`, `*`, or `_`) is a usage error
  naming the allowed set. `_` is excluded on purpose, matching the plugin's whitelist,
  so `snake_case` never reads as a section title.
- Selection among a task's sections, in document order: a whole-slug match wins;
  otherwise the first section whose slug starts with the selector's slug wins. `#future`
  therefore reaches `FUTURE WORK`, and an exact `#future-work` still wins over a
  hypothetical `FUTURE WORKFLOW` that appears earlier.
- `@foo+bar#` (empty selector) is not executable. It parses as mode `incomplete` with
  need `task_section`, which is exactly what opens the completion popup on `#`.
  Executing it is a usage error in the style of the existing empty-block-id message,
  pointing at `bob capture-task-sections -r foo -i bar`. This is **not** like `@notes#`,
  which means "any non-Tasks heading"; there is no "any task section" meaning.
- No section matches the selector: a hard error listing the task's actual section titles
  (bounded, in document order) plus a close-match suggestion when one exists, in the
  style of the existing missing-block-ID error. Capture must never silently fall back to
  the end of the task block, because that would hide a typo as a successful capture.

Every other capture rule is unchanged. The marker composes with `s:<N>`, `p:<N>`, and
`%...` clipboard markers in either terminal order exactly as `@route+block-id` does; the
rendered line is still the plain sub-bullet form with no `#task` tag, no
`[created::...]` stamp, and no block ID; authored sub-bullets in a multi-line draft
still nest one level under the new line; and a standalone trailing bare `#` remains the
Pomodoro-note marker, which still conflicts with any `@route` token on the same item.

### Insertion geometry

Once section `L` is selected under task `T`:

- The captured block is inserted at the end of `L`'s own block — that is, at the start
  of the next direct child of `T`, or at `T`'s `block_end` when `L` is the last one — so
  a later section is never pushed out of order.
- If a managed log sits directly under `L`, insertion goes before it, mirroring
  `first_direct_managed_log_start` as already applied to a task's own managed logs. A
  managed log that is a **sibling** of `L` (direct child of `T`) is simply the next
  direct child, so insertion is already before it.
- Indentation reuses `L`'s first existing child indentation (`first_child_indentation`
  re-rooted at `L`); otherwise `L`'s indentation plus the note's dominant
  tab-or-two-space unit; otherwise `L`'s indentation plus a tab. This is the existing
  `@route+block-id` fallback chain re-rooted at the section bullet.
- Line endings and every unrelated byte are preserved, and the note is replaced with one
  same-directory temporary-file rename, exactly as today.

The no-selector path (`@foo+bar` with no `#`) must remain byte-identical to today.

### Contracts

`bob capture-parse` gains no new JSON keys. `@foo+bar#req` reports mode `sub_bullet`,
route `foo`, `block_id` `bar`, and `section` `req` on the fields that already exist.
Today `MarkerParse` sets `section` XOR `block_id` based on `right_need`; the
three-component marker must populate **both**. It gains one new span kind,
`sub_bullet_section`, and one new need, `task_section` (add them to `SpanKind` / `Need`,
their `label()` methods, and the `capture-parse --help` vocabularies).

`bob capture-complete` gains the `task_section` context. Candidates reuse existing
optional keys — `replacement` (the slug), `title` (the original ALL-CAPS body), `route`,
`block_id`, `text` (the parent task description), `line`, and `child_count` — so
`CaptureCompletionCandidate` in Swift needs no new coding keys. `slug` may be included
as an additive optional field; the app can derive display from `title` and insert from
`replacement`.

`bob capture` JSON gains one optional `parent_section` field on a sub-bullet capture
(the matched original title), omitted for a plain `@route+block-id`. Human success
extends the existing `under [status] parent_text  ^block-id` line with a cyan ` · TITLE`
suffix when a section was selected; a no-section capture is unchanged.

`bob capture-task-sections` is new and read-only.

### Completion behavior

Typing `#` immediately after a resolved `@route+block-id` must open the popup with that
task's sections. That falls out of the contract rather than from a keystroke hook: the
parse of `@foo+bar#` reports `needs: ["task_section"]` and an `interactive_placeholder`
span over `#`, which is what the panel already watches. `@foo+bar#req` with the cursor
inside `req` reports a `sub_bullet_section` span and context `task_section`.

- Cursor inside the route component → `route` context, as today.
- Cursor inside the block-ID component → `task` context, as today, including the
  `--all-tasks` missing-ID flow. The `#` and selector are **not** part of that
  component's replacement range.
- Cursor past the `#` (including a zero-length selector) → `task_section` context,
  scoped to the route **and** block ID already typed in the same token.
- Block-ID component empty (`@foo+#req`): there is no parent task to scan, so return a
  successful empty result, matching how the authored right side of `@route^block-id` is
  handled.
- Block ID present but unresolvable (missing, duplicated, or not a task): return a
  successful empty result with one bounded `warnings` entry, so the panel can explain
  the empty list instead of showing nothing. Do not log the draft or the task
  description.
- Ranking mirrors `section_candidates` / `rank()`: slug-prefix matches first, then
  substring matches, preserving document order inside each tier. An empty query lists
  every section in document order.

`CompletionField` today is `{ context, route, query, replacement }` and
`completion_field_from_parts` models exactly two components. The third component needs
the already-typed block ID, so add `block_id: Option<String>` to `CompletionField`
(always `None` for existing contexts). Generalize the field builder to an optional third
component rather than cloning the two-component helper.

## Phase `sections_core`: shared task-section scanner in bob-cli

Work in the primary `bob-cli` repository.

1. Read the plugin source first:

   ```sh
   sase repo open bob-plugins -r "Read the Ctrl+Shift+Alt+N task-section definition before porting it to Rust"
   ```

   Study the functions named above. Do not edit that repo in this epic.

2. Add `src/native/capture_task_sections.rs` owning, and being the only place that owns:
   the title predicate, checkbox detection, the slug function, direct-child enumeration
   for a resolved `note_tasks::NoteTask`, selector matching (whole-slug then
   slug-prefix, document order), and the insertion geometry (section block end,
   managed-log guard, indentation fallback chain) described above. Model a discovered
   section as a small struct carrying at least the original title, the slug, the
   one-based line, the indentation, the block end, and `child_count`.
3. Reuse the existing helpers in `capture.rs` rather than reimplementing
   near-duplicates: `line_spans`, `list_item_body`, `list_marker_len`,
   `nearest_shallower_list_item_parent`, `leading_spaces_or_tabs_len`,
   `first_child_indentation`, `dominant_indent_unit`, `first_direct_managed_log_start`,
   and `parse_managed_task_log_marker`. Promote a helper's visibility (`pub(crate)`)
   where needed instead of copying it; a second, subtly different indentation or
   list-marker rule in the tree is a defect.
4. Register the module in `src/native.rs` (`mod capture_task_sections;` alphabetically
   after `capture_task_id`). Add no CLI surface, no `NativeCommand` variant, and no
   change to any existing command's behavior in this phase.
5. Unit tests covering: qualifying and non-qualifying titles at the whitelist edges
   (leading digit, `&'(),./-`, an `_`, lowercase, all digits, empty, a wikilink, a tag,
   a trailing block ID, inline code, two internal spaces); a checkboxed ALL-CAPS child
   (`- [ ] REQUIREMENTS` and `- [x] REQUIREMENTS`); a grandchild that must not be
   treated as a direct child; a section bullet with no children; ordered-list markers
   and `*`/`+` bullets; tab versus two-space versus four-space indentation and mixed
   indentation in one note; `SCHEDULE LOG` and `WORK LOG` children in every recognized
   marker spelling; whole-slug beating an earlier slug-prefix match; two sections with
   the same slug (first in document order wins); insertion offset for a middle section,
   the last section, a section followed by a blank line, and a section whose only child
   is a managed log; and CRLF input.

Validation for this phase:

```sh
just fmt
just lint
just test
git diff --check
```

## Phase `grammar`: three-component `@route+block-id#section` marker grammar

Work in the primary `bob-cli` repository. This phase is lexical: no vault mutation and
no new subcommand. It **does** extend `CaptureKind` so later phases compile against one
shape.

1. In `src/native/capture_language.rs`, extend the sub-bullet family to a
   three-component marker. Carry the selector on `CaptureKind::SubBullet` alongside its
   existing `target`:

   ```rust
   SubBullet {
       target: SubBulletTarget,
       section: Option<TaskSectionSelector>,
   }
   ```

   `TaskSectionSelector` holds the typed text and an `exact` flag, mirroring how
   `CaptureKind::Bullet` pairs `section_prefix` with `exact`. `exact` is only ever set
   by the forced picker option added in `execution`; a typed token is always
   prefix-capable. Update every constructor (`parse_sub_bullet_route_token`, the
   forced-`--task` overwrite in `plan_capture_item`, and tests in `capture.rs` /
   `capture_language.rs`) to `section: None`. Match sites that already use
   `CaptureKind::SubBullet { .. }` stay valid.

2. Update `parse_sub_bullet_route_token` to split a trailing `#<selector>` off the
   block-ID component **before** validating the block ID, and add distinct, actionable
   usage errors for: an empty selector, a selector with a disallowed character, an empty
   block ID with a selector present, and an empty route. Follow the existing message
   style, including the "run `bob capture-task-sections -r <route> -i <block-id>`" hint.
   `validate_special_terminal_markers_line` must reject the same shapes from the
   lone-marker position.
3. Add `SpanKind::SubBulletSection` (wire `sub_bullet_section`) and `Need::TaskSection`
   (wire `task_section`). Generalize `MarkerShape` / `marker_parse` to an optional third
   component rather than duplicating the two-component builder. Preserve the existing
   invariants: ordered, non-overlapping, `char`-boundary spans, and an
   `interactive_placeholder` span over the sigil or separator of a component that has
   not been typed yet. Needs are reported for every empty **required** component, in
   left-to-right order. `MarkerParse` must populate `block_id` from the middle component
   and `section` from the third; do not keep the current XOR.
4. `@foo+bar#req` reports mode `sub_bullet` with route, block ID, and section all
   populated. `@foo+bar#` and `@foo+#req` report mode `incomplete` with needs
   `["task_section"]` and `["task"]` respectively; `@foo+#` reports
   `["task", "task_section"]`. `@foo+bar` with no `#` is unchanged (complete
   `sub_bullet`, empty needs, `section: None`).
5. Extend `completion_field_at` / `marker_field_at_cursor` so a cursor past the `#`
   yields a `CompletionContext::TaskSection` field carrying the already-typed block ID
   and route. A cursor in the route or block-ID component keeps returning `Route` or
   `Task`, and those contexts' `CompletionField.block_id` stays `None`. The `#`
   separator itself is not part of either replacement range: a cursor after `#` on
   `@foo+bar#` is a zero-length `task_section` replacement at the insertion point, which
   is what makes typing `#` immediately completable.
6. Fail closed in `capture.rs` until `execution` lands: the
   `CaptureKind::SubBullet { target, section: None }` match arm keeps today's insertion;
   a `section: Some(_)` value must **not** fall into that arm (do not write `section: _`
   or `..`). A `Some` selector reaching `plan_sub_bullet_capture` in this phase is a
   programming error, not a silent append at the task end. `execution` replaces that
   guard with real insertion.
7. Update `bob capture-parse --help` (its documented mode and need vocabularies; add
   `@route+id#` to the incomplete-marker list in `long_about`), the `bob capture --help`
   marker description and examples, and the README's capture grammar, `capture-parse`
   contract, and Pomodoro-note interaction table (add a `@route+id#sec` row alongside
   the existing rejected markers).
8. Rust unit and CLI tests: every valid and malformed shape above; leading and trailing
   marker positions; the marker composed with `s:`, `p:`, and `%` in both terminal
   orders; multi-line drafts where only the first line may carry a leading marker;
   Unicode bodies and cursor positions on every component boundary; the flipped
   `@foo+bad#section` case; and explicit regressions proving a standalone trailing bare
   `#` is still the Pomodoro-note marker, `@route#Ideas` is still a note-bullet capture,
   `@route^id` and `@route:id` are unchanged, and `@route::id` still reports the
   retired-spelling diagnostic.

Validation for this phase:

```sh
just fmt
just lint
just test
git diff --check
```

## Phase `execution`: task-section resolution and insertion in `bob capture`

Work in the primary `bob-cli` repository. Read `sase/memory/cli_rules.md` through
`/sase_memory_read` before adding options.

1. In `src/native/capture.rs`, extend `plan_sub_bullet_capture` so that when the parsed
   marker carries a section selector it resolves the section through
   `capture_task_sections` after resolving the parent task, and computes the insertion
   offset and indentation from the section rather than the task. The `section: None`
   path must keep its current bytes-identical behavior. Authored children, clipboard
   children, and a `p:<N>` schedule log are still assembled into `capture_block`
   _before_ insertion; the section only changes where that whole block lands and which
   indent it is re-rooted at.
2. Apply `-S/--task-section` **after** the existing forced-`--task` overwrite in
   `plan_capture_item` (that overwrite currently replaces `kind` with
   `SubBullet { target, section: None }`). Errors are raised after the parent task
   resolves, so a bad block ID still reports the existing block-ID error. A selector
   that matches nothing errors with the task's actual section titles in document order
   (bounded) plus a close-match suggestion when one exists, and directs the caller to
   `bob capture-task-sections`. A task with no sections at all gets its own clearer
   message.
3. Add the forced `-S, --task-section TITLE` option to `bob capture`. It requires
   `--route` plus one of `--task`/`--task-ref`, conflicts with `--section`, and matches
   the whole section title exactly and case-insensitively — the picker counterpart to
   `--section`, and the reason the grammar carries an `exact` flag.
   `--task-section future-work` does **not** match `FUTURE WORK` (hyphen vs space);
   `--task-section "Future Work"` does. Keep the option list in `--help` alphabetically
   sorted (`task-section` after `task-ref`). Give the public long option its short alias
   `-S`. Update the Pomodoro-note conflict table so `--task-section` is rejected
   alongside `--task` / `--task-ref`.
4. Add optional `parent_section` (the matched original title) to
   `SubBulletCaptureDetails`, the `bob capture` JSON result
   (`skip_serializing_if = None`, next to `parent_text`), the human success `under`
   line, and the JSON-key-absence unit test that lists `parent_line` / `parent_text`. It
   is absent for a plain `@route+block-id` capture.
5. Update the README: the sub-bullet capture section (worked before/after for
   `@foo+bar#requirements`), the new marker form, the forced-option paragraph, the
   option list, and the picker-integration walkthrough (new step: after choosing a task,
   optionally run `capture-task-sections` and pass `--task-section TITLE`).
6. CLI tests in `tests/cli.rs`: capture into the first, middle, and last section of a
   task; into a section with existing children and into an empty one; prefix versus
   whole-slug selection; a multi-word section reached by slug and by prefix; a section
   whose only child is a managed log; a task whose managed log sits after its sections;
   tab, two-space, and four-space vaults; CRLF preservation; a multi-line draft with
   authored sub-bullets landing one level deeper inside the section; composition with
   `s:`, `p:`, `%`, and `--clip`; `--dry-run` planning the identical mutation without
   writing; `-S/--task-section` including its conflict and requirement errors; and every
   failure path (no match, no sections, missing task, duplicate block ID, non-task block
   ID) asserting the note is byte-identical afterwards.

Validation for this phase:

```sh
just fmt
just lint
just test
just install-smoke
git diff --check
```

## Phase `discovery`: `bob capture-task-sections` and `task_section` completion

Work in the primary `bob-cli` repository. Read `sase/memory/cli_rules.md` through
`/sase_memory_read` before adding the command.

1. Add a `run` entry point on the `sections_core` module (keep the scanner and the CLI
   shell in one file, matching `capture_sections.rs` / `capture_tasks.rs`) implementing:

   ```
   bob capture-task-sections --route NAME (-i|--block-id ID | -t|--task-ref REF)
                            [-b|--bob-dir DIR] [-f|--format human|json]
   ```

   Use `capture-task-id`'s option vocabulary — `-i` for the block ID, `-t` for the
   stale-safe ref — since both commands address one task inside one route. Exactly one
   of `-i`/`-t` is required. Keep the argument list, and the printed option list, sorted
   alphabetically by long name (`block-id`, `bob-dir`, `format`, `help`, `route`,
   `task-ref`), give every public long option a short alias, and follow the sibling
   commands' `--help`, `--about`, `--long-about`, `Examples:`, and `Environment:`
   structure.

2. The command is read-only. It reuses the exact parent-task lookup and error messages
   `bob capture` uses (not a task, duplicate ID, missing ID with suggestion). A resolved
   task with no sections returns a successful empty list, so a picker can skip the
   chooser. JSON is `{ok, schema_version: 1, route, block_id, ref, count, sections}`
   with each section carrying `title`, `slug`, `line`, `child_count`, and `depth`
   (always `1` for a direct child; include it so pickers can share the `capture-tasks`
   row shape). `block_id` is nullable when the parent was resolved by `--task-ref` and
   still has no ID. Human output uses the shared `Styler`, matching `capture-sections`'
   layout and colors (cyan titles, dim slugs or child counts, a "No task sections
   found." empty state, and a trailing `N sections` count).
3. Register the command in `src/native.rs` (`NativeCommand::CaptureTaskSections` next to
   `CaptureTaskId`, `mod` line already added in `sections_core`, and the `run()` match
   arm), `src/runner.rs` (command table, `about`, and the `bob --help` example block),
   the `install-smoke` recipe in `justfile` (after `capture-task-id --help`), and the
   alphabetical help-case tables in `tests/cli.rs`. All of these are alphabetically
   ordered; `capture-task-sections` sorts after `capture-task-id` and before
   `capture-tasks`.
4. In `src/native/capture_complete.rs`, add the `task_section` context: a
   `TaskSectionCandidate` with `replacement` (the slug), `title`, `slug`, `route`,
   `block_id`, `text` (the parent task description), `line`, and `child_count`; a
   `Candidates::TaskSection` variant wired through `len`, `candidate_lines`, and the
   JSON untagged enum; and the ranking described above. Extend every `CompletionContext`
   match (including the wikilink `unreachable!` arms) so the new variant compiles. An
   empty block-ID component returns a successful empty result; an unresolvable parent
   task returns a successful empty result plus one bounded `warnings` entry. Do not
   change any existing context's shape or ordering.
5. Update `bob capture-complete --help` (its documented context list and the paragraph
   describing which contexts are backed by which scan), the README command table, the
   `capture-complete` and picker-integration sections, and the new command's own README
   entry.
6. Tests: unit tests for candidate construction, ranking tiers, and the warning path;
   CLI integration tests for `capture-task-sections` in human and JSON form covering a
   task with several sections, one with none, a missing note, a missing task, a
   duplicate block ID, a non-task block ID, `--task-ref` resolution and staleness, the
   mutually-exclusive-argument errors, and stable JSON key order; and `capture-complete`
   tests for a cursor on every component of `@foo+bar#req`, a bare `@foo+bar#`,
   `@foo+#`, an unresolvable block ID, a multi-word section's slug replacement, and
   proof that `route`, `section`, `pomodoro_block_id`, `task`, and the wikilink contexts
   return byte-identical results to before.

Validation for this phase:

```sh
just fmt
just lint
just test
just install-smoke
git diff --check
```

## Phase `mac_app`: `#`-triggered task-section completion in Bob Mac Capture

Open the linked app with a specific audited reason before reading or editing it, and use
only the path that command prints:

```sh
sase repo open bob-mac-capture -r "Implement the approved task-section completion integration"
```

1. In `Sources/CaptureCore/CompletionRowContent.swift`: map the `sub_bullet_section`
   span kind to `.section` in `captureSemanticCategory`, and add `.taskSection` to
   `CaptureCompletionContext` for the raw value `task_section`. Unrecognized kinds must
   keep falling back to `.neutral` so an older `bob` stays usable.
2. Add the `.taskSection` case to `completionRowContent`: the section palette category,
   a distinct SF Symbol from the note-`Section` row's `list.bullet.indent` (use
   `list.bullet.rectangle`) so the two are not confused, the context label
   `Task Section`, the original ALL-CAPS `title` as primary text, the parent task's
   `text` as secondary text so the user can see which task they are filing under, and
   badges for `^block-id` plus either the section's item count (`N items`) or an
   explicit `Empty` badge derived from `child_count`. Query emphasis matches against
   `title`. Confirm against the CLI contract that `CaptureCompletionCandidate` needs no
   new coding keys (`title`, `text`, `blockID`, `route`, and `childCount` already
   decode). Accessibility hint: `Nests the capture under this task section.`
3. In `Sources/BobMacCapture/CapturePanelModel.swift`, add `task_section` to the
   `shouldRequestCompletion` needs set (alongside today's `route`, `section`,
   `pomodoro_id`, `task`) and `sub_bullet_section` to its span-kind set, so typing `#`
   after a resolved `@route+block-id` opens the popup on the next debounced analysis.
   Accepting a candidate keeps using the server's exact byte range and `cursor_after`;
   the block-ID prompt flow stays scoped to the `task` context only.
4. In `Sources/BobMacCapture/CapturePanelView.swift`, render task-section rows as a
   plain ungrouped list — the Ready-to-use / Needs-block-ID grouping stays specific to
   `context == "task"` — and check that the existing secondary-line rendering does not
   double up with the row content's secondary text for this context.
5. Update `Tests/Fixtures/fake-bob`'s `capture-complete` and `capture-parse` arms so a
   draft like `Postgres 17 minimum @foo+bar#` (cursor past `#`) returns context
   `task_section` with at least one candidate (`replacement: "requirements"`,
   `title: "REQUIREMENTS"`, parent `text`, `block_id`, `child_count`), and so
   `capture-parse` reports `needs: ["task_section"]` plus a `sub_bullet_section` or
   `interactive_placeholder` span. Keep every existing canned draft byte-identical.
6. Update the app README: the completion-context list, the row-presentation paragraph
   (which currently enumerates Destination/Section/Parent Task alongside
   Note/Heading/Block), the `bob` version requirement note (currently
   `capture-complete --all-tasks` and `capture-task-id`; add task-section completion),
   and the editor-syntax section that documents `@route+block-id`.
7. Tests: extend `Tests/CaptureCoreTests/CompletionRowContentTests.swift` for the new
   context and span mapping, including an empty section, a multi-word title, match
   emphasis on `REQUIREMENTS` against query `req`, and an unknown-context fallback;
   extend `Tests/BobMacCaptureTests/CapturePanelModelTests.swift` for the
   completion-request decision on `@foo+bar#`, on a bare trailing `#` (still a Pomodoro
   note, **not** a task-section popup), and on `@foo#` (still note-section completion),
   plus accepting a task-section candidate and landing the caret correctly.

Validation for this phase, on macOS:

```sh
just format-lint
just build
just test
git diff --check
```

`CaptureCore` and `CaptureCoreTests` also build on Linux for a fast type-check, but the
app target and `BobMacCaptureTests` need macOS; do not migrate tests off XCTest to work
around a mismatched toolchain.

## Acceptance criteria

- `bob capture 'Postgres 17 minimum @foo+bar#requirements'` appends a plain
  `- Postgres 17 minimum` bullet as the last child of the `REQUIREMENTS` bullet under
  the `^bar` task in `foo.md`, preserving line endings and every unrelated byte.
- `#future` and `#future-work` both reach a `FUTURE WORK` section; an exact slug beats
  an earlier prefix match; a section that is still empty is a valid target.
- A selector that matches nothing, a task with no sections, and a malformed selector
  each fail with an actionable message naming the real alternatives, and leave the note
  byte-identical.
- A checkboxed child, a grandchild, a `SCHEDULE LOG` or `WORK LOG` child, a lowercase
  bullet, and a `snake_case` bullet are never offered or matched as sections.
- `bob capture-task-sections -r foo -i bar` lists that task's sections in document order
  in human and JSON form, and returns a successful empty list for a task with none.
- `bob capture-parse` reports mode `sub_bullet` with route, block ID, and section for
  `@foo+bar#req`, and mode `incomplete` with the right ordered `needs` for every partial
  form.
- In the capture panel, typing `#` right after a resolved `@foo+bar` opens the popup
  listing that task's sections with the parent task shown; accepting one inserts the
  slug at the server's byte range and leaves the caret after it.
- A standalone trailing bare `#` still means a Pomodoro note, `@route#Ideas` still means
  a note-bullet capture, and `@route`, `@route^id`, `@route:id`, and `@route+id` behave
  exactly as before, including the `--all-tasks` Add-block-ID flow.
- Both repositories' full validation suites pass, and `bob --help`,
  `bob capture --help`, `bob capture-complete --help`,
  `bob capture-task-sections --help`, the command table, the install smoke test, and
  both READMEs agree with the shipped behavior.

## Cross-phase constraints

- Preserve unrelated worktree changes. Every access to `bob-mac-capture` and
  `bob-plugins` must go through `/sase_repo` and use the path that command prints; do
  not embed an ephemeral workspace path in any implementation artifact.
- Do not edit `bob-plugins` in this epic. It is read as the source of truth for the
  task-section definition only. If the plugin and this plan genuinely disagree about the
  definition, record it as follow-up work rather than changing either side unilaterally.
- The task-section predicate, slug, selector matching, and insertion geometry live in
  exactly one Rust module. No Swift reimplementation, no second copy inside
  `capture.rs`, and no duplicated list-marker or indentation rule.
- All wire changes stay additive under schema version 1. Do not renumber, rename, or
  reorder an existing JSON field, span kind, mode, need, or context.
- A parsed `section: Some(_)` selector must never be silently ignored by insertion.
  Until `execution` honors it, it is a programming error, not an append at the task end.
- Do not commit, push, deploy, install, run `bob plugins sync`, or change the active app
  unless separately authorized by the user or a post-completion finalizer.
- Treat draft text, task descriptions, and section titles as private: keep them out of
  logs, notifications, signposts, diagnostics history, and any test artifact built from
  the real vault.
- Each phase ends with focused tests, its repository's full available suite,
  `git diff --check`, and a diff review against this plan. Discovered unrelated work
  goes through the project's required SASE follow-up workflow instead of being folded
  into this feature.
