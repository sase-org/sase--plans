---
tier: epic
status: done
title: Free the @@ global destination from the header line and teach it to absorb
goal: "A `@@route` / `@@route+block-id` global destination declaration can be typed on
  any line of any capture item instead of only on a header line, and typing a bare `@@`
  inside an item that already carries a `@file` reference moves that reference onto the
  `@@` and deletes the original, so a draft can never end up with a shadowed local
  marker or two competing declarations.

  "
phases:
  - id: grammar
    title: Position-free @@ declarations in the capture grammar
    depends_on: []
    size: medium
    description: "grammar: make any `@@…` token a global destination declaration
      anywhere in a draft, drop the header-line-only restriction, add
      duplicate-declaration and shadowed-local diagnostics, and update capture,
      capture-parse, capture-complete, help text, docs, and tests.

      "
  - id: rewrite
    title: bob capture-rewrite and the bare @@ absorption rule
    depends_on:
      - grammar
    size: medium
    description: "rewrite: add the `bob capture-rewrite` subcommand that turns a bare
      `@@` into `@@<payload>` by absorbing the item's local destination marker (or the
      draft's other declaration), deleting the source token and returning edits, cursor,
      and a human summary.

      "
  - id: macapp
    title: Mac capture panel absorbs @@ as you type
    depends_on:
      - rewrite
    size: medium
    description:
      "macapp: wire the bob-mac-capture panel to `bob capture-rewrite` so typing a bare
      `@@` rewrites the draft in place with an announced summary, plus CaptureCore
      models, process-client lane, tests, and README updates."
proposed_by: bbugyi200.athena.0cv
create_time: 2026-09-09 19:59:49
---

- **PROMPT:**
  [prompts/202608/capture_global_destination_anywhere.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/capture_global_destination_anywhere.md)

# Plan: Free the `@@` global destination from the header line and teach it to absorb

## Background

`bob capture` recently gained a draft-wide destination declaration: `@@<route>` or
`@@<route>+<block-id>`. Today it is only recognized as **exactly one whitespace-free
token on the first nonblank physical line of the draft**. `split_capture_draft` in
`src/native/capture_language.rs` peels that line off as a `GlobalHeaderLine`, and every
other `@@…` token anywhere in the draft is rejected by
`reject_embedded_global_declarations` with `MISPLACED_GLOBAL_DESTINATION_ERROR`.

Two problems with that design:

1. **Placement is rigid.** The declaration is a property of the draft, but the user has
   to compose it before writing anything, and has to scroll back to the top to change
   it. It should be typeable at the end of whichever note or task the user is on.
2. **It silently loses to local markers.** An item with its own `@route` marker ignores
   the declaration entirely, so `Buy milk @dev @@groceries` declares a destination that
   the very item declaring it does not use. Nothing tells the user.

This epic fixes both, and adds the convenience the second problem really calls for:
typing a bare `@@` inside an item that already has a `@file` reference **moves** that
reference onto the `@@` and deletes the original.

## Design

### Vocabulary

- **Declaration token** — a whitespace-delimited token whose text starts with `@@`.
- **Local destination marker** — an item-local `@route`, `@route+block-id`,
  `@route#Section`, `@route^block-id`, `@route:block-id`, or a trailing bare `#`
  Pomodoro-note marker. These are what `select_marker_token` / `resolve_line` already
  resolve today.
- **Bare `@@`** — a declaration token whose text is exactly `@@`.

### Rule G1 — a declaration token is recognized anywhere

Any declaration token, on any physical line of any capture item — parent line or
authored child line, leading, trailing, or mid-line — declares the draft's global
destination. It is never body text.

Recognizing it _anywhere on the line_ rather than only in the terminal region is
deliberate: it is a strictly simpler rule to state and to implement, it matches how the
absorption rule is described ("typed anywhere inside a note/task"), and it removes a
class of positional gotchas. End-of-line remains the documented convention.

`@@` stays reserved, exactly as it is today, so this is not a regression for literal
text: every `@@…` token is currently a hard error outside the header. The existing
escape hatch is unchanged — wrap the text in inline code (`` `@@foo` ``), which makes
the token start with a backtick rather than `@@`.

### Rule G2 — a declaration-only line is not a capture line

A physical line whose tokens are _all_ declaration tokens carries no capture text. It is
removed from the physical-line stream **before** blank-line item splitting, so it is
neither a capture item nor an authored child, and it does not act as an item separator.

This is what preserves today's documented header spellings with no special case:

```text
@@foo
First task

Second task
```

becomes the two items `First task` and `Second task`, exactly as it does today, because
the `@@foo` line is stripped before splitting. So does the blank-line-separated form.

Known consequence, to be documented: a declaration-only line wedged between two
unseparated body lines (`First task` / `@@foo` / `Second task`) leaves the two body
lines in one item, and `Second task` is then an invalid authored child. That is an
honest error; blank-line separation or an end-of-line `@@foo` is the fix.

### Rule G3 — declarations are extracted before anything else on the line

Declaration tokens are pulled out of a line's token vector **before**
`extract_terminal_markers` and before local route resolution. This matters: those
routines are positional (`is the last token a route marker`,
`walk back over trailing s:/p:/% markers`), and a trailing `@@groceries` would otherwise
hide the real markers. With the declarations removed first, every existing rule applies
unchanged:

| Input                       | Body       | Local | Global                            |
| --------------------------- | ---------- | ----- | --------------------------------- |
| `Buy milk @@groceries`      | `Buy milk` | —     | `groceries`                       |
| `Buy milk @@groceries s:2`  | `Buy milk` | —     | `groceries` (`s:2` still applies) |
| `Buy milk s:2 @@groceries`  | `Buy milk` | —     | `groceries`                       |
| `Buy milk @dev @@groceries` | `Buy milk` | `dev` | `groceries` (+ G7 warning)        |
| `@dev Buy milk @@groceries` | `Buy milk` | `dev` | `groceries` (+ G7 warning)        |

### Rule G4 — at most one declaration per draft

A draft may declare at most one global destination.

- `bob capture` fails with a new `duplicate global destination declaration` usage error
  that names the line numbers of both tokens.
- `bob capture-parse` emits an error diagnostic with code `duplicate_global_destination`
  on every declaration token after the first, and keeps the first as the effective
  declaration so the editor still has a usable parse.

### Rule G5 — declaration shape is unchanged and still strict

A declaration token must be exactly `@@<route>` or `@@<route>+<block-id>`. `#`, `^`, and
`:` forms, an empty route, an invalid route, and an empty or invalid block ID keep
today's error text (`GLOBAL_DESTINATION_SHAPE_ERROR`, `GLOBAL_DESTINATION_ROUTE_ERROR`,
`GLOBAL_DESTINATION_BLOCK_ID_ERROR`, `unsupported_global_destination_error`).

`GLOBAL_DESTINATION_EXTRA_TEXT_ERROR` and `MISPLACED_GLOBAL_DESTINATION_ERROR` are
retired along with their diagnostics codes `invalid_global_destination` (extra-text
variant only) and `misplaced_global_destination`.

`@@route#Section` support is **deliberately out of scope** for this epic. A draft-wide
bullet destination would have to flip every unrouted item from task mode to bullet mode,
which is a real behavioral expansion with its own execution semantics in `capture.rs`;
it stays an error here, and the absorption rule below explains itself rather than
silently dropping the section (Rule A5).

### Rule G6 — inheritance is unchanged

An item with no local route/mode marker inherits the declaration; an item with one keeps
its own. `inherit_global_destination` and `inherit_editor_global_destination` keep
today's behavior. The declaration is still metadata: absent from item counts, semantic
capture text, preview bodies, and note contents. A draft that declares a destination and
contains no other capture item still fails with `missing_capture_item_error`.

### Rule G7 — a shadowed declaration is a warning, not an error

When an item carries **both** a local destination marker and a declaration token on one
of its own lines, that item ignores the destination it declares. That is legal and
unambiguous, so it is a warning rather than a failure:

- `bob capture-parse` emits `Severity::Warning`, code `global_destination_shadowed`, on
  the declaration token:
  `this item's @dev marker overrides the @@groceries destination it declares; move @@groceries to an item without a local marker, or delete @dev`.
- `bob capture` prints the same text to stderr as a warning and still succeeds, and
  reports it in JSON through a new additive optional `warnings: Vec<String>` field on
  `CaptureResult` (`skip_serializing_if = "Vec::is_empty"`, so existing consumers are
  unaffected).

This is the safety net for text that arrives already in that shape (paste, scripts,
`bob capture` from a shell). The absorption rule below is the safety net for text being
typed.

### Rule A1 — a bare `@@` is an absorb request

A bare `@@` claims the draft's destination declaration, taking its payload from the item
it was typed in.

**Which bare `@@`:** the one that contains or ends at `--cursor` when a cursor is given;
otherwise the last bare `@@` in source order.

**Payload source, in order:**

1. The bare `@@`'s **own item's** single local destination marker, when the item has
   exactly one and it is an absorbable form (`@<route>` or `@<route>+<block-id>`). Rule
   name: `absorb_local_marker`.
2. Otherwise the draft's **other declaration token**, when exactly one exists and it
   carries a payload. Rule name: `absorb_declaration`.
3. Otherwise nothing — the bare `@@` stays bare and remains a normal incomplete marker
   awaiting route completion.

**Result:** the bare `@@` becomes `@@<payload>`, the payload source token is deleted,
and every _other_ declaration token in the draft is deleted too, so a rewritten draft
always ends with exactly one declaration and no shadowed local marker on the declaring
item.

Source (1) before source (2) is what makes the common case right: the user typed `@@` on
the item that already has `@dev`, and that is the reference they mean. Source (2) exists
because there can only ever be one declaration, so a stale one must go rather than
becoming a duplicate error the user has to scroll back and fix by hand — which is the
exact annoyance this epic removes.

**Whitespace:** deleting a token also consumes one adjacent whitespace run — the
preceding run when the token ends its line, otherwise the following run — so no double
spaces are left behind. Deleting the only token of a line deletes the whole line
including its terminator. Whitespace runs are never merged across a line terminator.

**Cursor:** the returned cursor is the offset just past the rewritten `@@<payload>`
token, so the user can keep typing (`+block-id`, more prose) without repositioning.

**Idempotence:** after the rewrite the claiming token is no longer bare, so running the
rewrite on its own output is a no-op. This must be covered by a test.

### Rule A5 — non-absorbable local markers explain themselves

When the item's single local marker is `@route#Section`, `@route^block-id`,
`@route:block-id`, or a bare `#`, **no rewrite happens**. `@@` cannot express those
destinations (Rule G5), and grafting only the route would silently drop the section,
block ID, or Pomodoro link. `bob capture-rewrite` instead returns `changed: false` plus
a `notices` entry naming the marker and why, e.g.
`@@ cannot take a section: leave @notes#Ideas on this item, or delete it and declare @@notes`.

### Rule A6 — ambiguity is left alone

When the item has more than one local destination marker the draft is already in a
duplicate-marker error state that `capture-parse` reports. No rewrite happens.

### Where absorption lives

Absorption is an **editor typing assist**, not a grammar rule. `bob capture` executes
exactly the text it is given: a bare `@@` there is still an incomplete declaration and
still fails with `GLOBAL_DESTINATION_SHAPE_ERROR`. The rewrite mutates the user's
visible text, so after it runs the CLI and the panel agree on what the draft says. This
keeps execution semantics honest and keeps `bob capture-parse` purely read-only.

The rewrite logic lives in `bob-cli` — the mac app must not reimplement capture grammar
— and is exposed as its own subcommand so it is scriptable and directly testable.

---

## Phase `grammar`: Position-free `@@` declarations in the capture grammar

**Repository:** `bob-cli` (this project's primary repo).

Implement rules G1–G7.

### `src/native/capture_language.rs`

1. **Replace the header model.** `CaptureDraft`'s `header: Option<GlobalHeaderLine>`
   becomes a list of declaration sites collected from the whole draft. Suggested shape:

   ```rust
   /// One `@@…` declaration token plus where it came from.
   pub(crate) struct GlobalDeclarationToken<'a> {
       pub(crate) token: Token<'a>,
       pub(crate) line_number: usize,
   }
   ```

   `split_capture_draft` should:
   - split physical lines as today;
   - classify each line: a line whose every token starts with `@@` is a declaration-only
     line (Rule G2) — record its declaration tokens and drop the line from the stream;
   - split the remaining lines into items exactly as today via
     `split_items_from_physical_lines`, preserving original byte offsets and 1-based
     line numbers (they must still index the complete original draft);
   - return `CaptureDraft { declarations: Vec<GlobalDeclarationToken<'a>>, items }`,
     where `declarations` at this point holds only the declaration-only-line tokens.

   Declaration tokens that share a line with body text are found later, per line, so
   that they can be removed from the same token vector the body is built from.

2. **Strip declarations per line (Rule G3).** Add a helper used by every per-line entry
   point:

   ```rust
   /// Remove every `@@…` token from `tokens`, returning them in source order.
   /// Called before `extract_terminal_markers` and before any route resolution.
   fn take_global_declarations<T: ParseToken>(tokens: &mut Vec<T>) -> Vec<T>;
   ```

   Call it at the top of `resolve_line` (execution path) and `parse_editor_line` (editor
   path), and in `completion_field_at` before `extract_terminal_markers`. Delete
   `reject_embedded_global_declarations` and the `misplaced_global_destination`
   diagnostic block in `parse_editor_line`.

   The execution path needs the stripped tokens to travel back up to the draft level.
   `LineOutcome` and `AggregateMarkers` already carry per-line results to
   `parse_capture_item`; extend them with the line's declaration tokens (text plus span
   plus line number) and have `parse_capture_draft_with_clip_control` merge them with
   the declaration-only-line tokens from `split_capture_draft`. Because `resolve_line`
   receives `Vec<&str>` today, it will need position-carrying tokens for spans; the
   simplest change that keeps one grammar is to have `parse_capture_item` tokenize with
   `tokenize_line_with_spans` and pass `Vec<Token<'_>>` through `resolve_line` (it is
   already generic over `ParseToken`). Do whatever keeps a single tokenizer and a single
   classification path — do not fork the grammar for the editor.

3. **Resolve the draft declaration.**
   - Zero declarations: `global` is `None`, as today.
   - One declaration: parse it with the existing `parse_global_destination_token`
     (execution) / `classify_global_token` (editor). Keep both `start`/`end` and add
     `line: usize` to `ParsedGlobalDestination` and `EditorGlobalDestination`.
   - Two or more (Rule G4): execution fails with
     `duplicate_global_destination_error(first_line, second_line)`; the editor keeps the
     first as effective and pushes a `duplicate_global_destination` error diagnostic on
     each later token.

4. **Shadow warning (Rule G7).** In `parse_editor_item` (or right after items are built
   in `parse_for_editor`, whichever keeps the per-item state available), when an item
   has `has_local_destination == true` _and_ owns a declaration token, push a
   `Severity::Warning` diagnostic with code `global_destination_shadowed` ranged on the
   declaration token. The execution path needs the same condition so `bob capture` can
   surface it — return it from `parse_capture_draft_with_clip_control` as a
   `Vec<String>` of warnings on `ParsedCaptureDraft`.

5. **Completion (Rule G9).** `completion_field_at` currently special-cases
   `draft.header`. Replace that with: locate the line containing the cursor as today,
   tokenize it, and if any token starts with `@@` and the cursor sits inside it,
   delegate to the existing global-token field logic (rename
   `global_completion_field_at` to take a `Token` instead of a `GlobalHeaderLine`).
   Otherwise continue with the existing marker path over the declaration-stripped
   tokens.

6. **Retire the dead constants and helpers:** `GLOBAL_DESTINATION_EXTRA_TEXT_ERROR`,
   `MISPLACED_GLOBAL_DESTINATION_ERROR`, `GlobalHeaderLine`,
   `parse_global_header_strict`, `reject_embedded_global_declarations`. Keep
   `MISSING_CAPTURE_ITEM_ERROR` (Rule G6 still needs it) but reword it away from the
   word "header":
   `global destination declaration has no capture item; add a capture item to this draft`.

7. **Span kinds are unchanged.** `SpanKind::GlobalRoute`, `GlobalSubBulletRoute`, and
   `GlobalSubBulletBlockId` keep their names and meanings and may now appear anywhere.
   This is what lets the mac app's existing highlighting keep working untouched.

### `src/native/capture_parse.rs`

- Add an additive `line: usize` to `GlobalDestinationParse`. `SCHEMA_VERSION` stays `1`
  (additive fields keep version 1, per the comment on that constant).
- Update the `--help` long text: the declaration is no longer described as a leading
  header; describe it as "a `@@<route>` or `@@<route>+<block-id>` token anywhere in the
  draft", and document the `duplicate_global_destination` and
  `global_destination_shadowed` diagnostic codes.
- Human output: keep the `global` field, and print the shadow warning through the
  existing diagnostics rendering.

### `src/native/capture.rs`

- Add `warnings: Vec<String>` to `CaptureResult` with
  `#[serde(skip_serializing_if = "Vec::is_empty")]`, fed from the parse warnings.
- Print each warning to stderr in human mode, styled like other bob warnings.
- Update the `--help` long text and examples: drop "on the first nonblank physical
  line", add an end-of-line example such as `bob capture 'Buy milk @@groceries'` and
  `printf 'First task\nSecond task @@foo\n' | bob capture`.
- Keep `competing_destination_error` (a textual declaration still cannot be combined
  with `--route` / `--section` / `--task` / `--task-section`), but reword "declaration"
  wording away from "header".

### `src/native/capture_complete.rs`

- Update the `--help` long text: replace "a leading `@@`, `@@fragment`, or `@@route+…`
  header" with "a `@@`, `@@fragment`, or `@@route+…` declaration anywhere in the draft",
  and add an example completing a trailing declaration, e.g.
  `bob capture-complete -c 20 -- 'Buy milk @@gro'`.

### Docs

- `docs/capture.md`: rename the "Global destination header" section to "Global
  destination declaration". Rewrite it around rules G1–G7 with worked examples for
  end-of-line placement, a child-line declaration, the declaration-only line, the
  duplicate error, and the shadow warning. Update the "Grammar at a glance" table rows
  for `@@route` / `@@route+block-id` to say "anywhere in the draft". Update the
  paragraph that currently reads "Leading `@route text` is accepted only on an item's
  first physical line" to note that `@@…` has no such restriction. Keep the "Wrap
  `@@...` in inline code to keep it literal" sentence.
- `README.md`: update the `@@` paragraph and the two grammar table rows, and add an
  end-of-line example next to the existing `printf` one.

### Tests (`tests/cli.rs` plus in-module unit tests)

Update the existing `@@` tests (search `@@` in `tests/cli.rs` — the header-only ones
around the "extra text", "misplaced", and "first nonblank line" assertions must be
replaced, not merely edited) and add coverage for:

- `Buy milk @@groceries` — routed to `groceries.md`, body has no `@@` token.
- `printf 'First task\nSecond task @@foo\n'` — declaration on a _later_ item's line
  applies to the earlier item too.
- Declaration on an authored child line applies draft-wide.
- `Buy milk @@groceries s:2` and `Buy milk s:2 @@groceries` — both markers survive.
- `@@foo\nFirst task` and `@@foo\n\nFirst task\n\nSecond task` still behave exactly as
  they do today (regression guard for Rule G2).
- Two declarations → `duplicate_global_destination` error naming both line numbers, and
  the matching `capture-parse` diagnostics.
- `Buy milk @dev @@groceries` → item routes to `dev.md`, other items inherit
  `groceries`, and both `capture-parse` (warning diagnostic) and `bob capture`
  (`warnings` field + stderr) surface the shadow.
- `@@foo` alone in a draft still fails with the missing-capture-item error.
- `` `@@foo` `` (inline code) stays literal body text.
- `bob capture-complete` completing `@@gro` at the end of a line and `@@foo+ta` at the
  end of a child line.
- `capture-parse -f json` reports `global_destination.line` correctly for a trailing
  declaration.

Run `just` (or the project's usual `cargo test` / `cargo clippy` / `cargo fmt` recipes —
check `justfile`) and leave the tree clean.

---

## Phase `rewrite`: `bob capture-rewrite` and the bare `@@` absorption rule

**Repository:** `bob-cli`. **Depends on:** `grammar`.

Implement rules A1–A6 and the new subcommand.

### Grammar-side function

Add the absorption computation to `src/native/capture_language.rs` (it is grammar
knowledge, and it must reuse the same tokenizer, the same declaration scan, and the same
local-marker classification the rest of the module uses):

```rust
pub(crate) struct DraftRewrite {
    pub(crate) rule: Option<RewriteRule>,      // None when nothing changed
    pub(crate) edits: Vec<TextEdit>,           // sorted by start, non-overlapping
    pub(crate) text: String,                   // input with edits applied
    pub(crate) cursor: Option<usize>,          // Some when a cursor was supplied
    pub(crate) summary: Option<String>,
    pub(crate) notices: Vec<String>,
}

pub(crate) struct TextEdit { pub(crate) start: usize, pub(crate) end: usize, pub(crate) replacement: String }

pub(crate) enum RewriteRule { AbsorbLocalMarker, AbsorbDeclaration }

pub(crate) fn rewrite_draft(raw_text: &str, cursor: Option<usize>) -> DraftRewrite;
```

Implementation notes:

- Find every declaration token in the draft (reuse the phase-`grammar` scan) and every
  item's local destination marker (reuse `parse_for_editor`, whose `EditorItemParse`
  already reports `has_local_destination`, `mode`, `route`, `section`, and `block_id`;
  you will additionally need the marker token's _span_, so return it from
  `parse_editor_item` — e.g. a `local_destination_span: Option<(usize, usize)>` field —
  rather than re-deriving it).
- Select the bare `@@` per Rule A1. A cursor "at" a token means
  `cursor >= token.start && cursor <= token.end`.
- Choose the payload source per Rule A1's ordered list; produce no rewrite when there is
  none.
- Build edits: one replacement on the bare `@@` token span with `@@<payload>`, plus one
  deletion per removed token. Extend each deletion to swallow one adjacent whitespace
  run per the whitespace rule, and to swallow the whole line (terminator included) when
  the deleted token was the line's only token. Sort by `start` and assert
  non-overlapping.
- Apply the edits to produce `text`, and map the input cursor through them; when a
  rewrite applied, override the result cursor to the offset just past the rewritten
  `@@<payload>`.
- Rule A5 notices: when the item's single local marker exists but is a non-absorbable
  form, emit no edits and one notice naming the marker text and the reason.
- `summary` is a short human sentence for the panel to announce, e.g.
  `Moved @dev into @@dev` or `Moved the @@foo declaration here`.

### `src/native/capture_rewrite.rs` (new)

Model it directly on `src/native/capture_parse.rs` — same argument handling (`TEXT`
joined with spaces, full piped stdin when omitted), same "purely lexical, read-only,
never opens the vault, takes no `--bob-dir`" contract, same `Styler` usage, same error
handling shape.

Options (alphabetical in help, every long option gets a short alias, per the project's
CLI rules memory `sase/memory/cli_rules.md`):

```
-c, --cursor <N>       UTF-8 byte offset of the editor's insertion point
-f, --format <FORMAT>  Output format: human or json  [default: human]
-h, --help             Show help
```

JSON output, `schema_version: 1`:

```json
{
  "ok": true,
  "schema_version": 1,
  "input": "Buy milk @dev @@",
  "text": "Buy milk @@dev",
  "changed": true,
  "cursor": 14,
  "rule": "absorb_local_marker",
  "edits": [
    { "range": { "start": 9, "end": 14 }, "replacement": "" },
    { "range": { "start": 14, "end": 16 }, "replacement": "@@dev" }
  ],
  "summary": "Moved @dev into @@dev",
  "notices": []
}
```

- `rule` is `absorb_local_marker`, `absorb_declaration`, or omitted when nothing
  changed.
- `cursor` is present only when `--cursor` was supplied.
- `summary` is omitted when nothing changed; `notices` is omitted when empty.
- `edits` index `input`, are sorted by `start`, never overlap, and applying them
  left-to-right yields `text`. `text` always equals `input` when `changed` is `false`.

Human output uses `Styler` and should be genuinely pleasant to read: a header line
naming the rule, then the before/after draft with the touched spans colored, then any
notices. When nothing changed, print a dim `no rewrite` line plus the notices.

Exit codes mirror `capture-parse`: only a bad flag or missing TEXT is an error; every
other input succeeds with `ok: true`.

### Wiring

- `src/native/mod.rs`: add `capture_rewrite` and a `NativeCommand::CaptureRewrite`
  variant.
- `src/runner.rs`: add the `capture-rewrite` entry to `SUBCOMMANDS`, **between**
  `capture-parse` and `capture-sections` (the table is alphabetically sorted and
  `subcommands_are_sorted` guards it). About text:
  `Apply the capture grammar's automatic draft rewrites`.

### Docs

- `docs/capture.md`: new `bob capture-rewrite` section next to `bob capture-parse`,
  added to the Contents list. Document rules A1–A6 with worked examples, including the
  A5 notice cases and the idempotence guarantee.
- `README.md`: one-line mention in the capture command list.

### Tests

In-module unit tests for `rewrite_draft` plus `tests/cli.rs` coverage for:

- `Buy milk @dev @@` with the cursor at the end → `Buy milk @@dev`, cursor `14`.
- `@dev Buy milk` + `@@` typed at the end → the leading marker is absorbed.
- A local marker on the parent line absorbed by a `@@` typed on an authored child line
  of the same item (the "don't make me scroll back" case).
- `@cash+goog-exit` absorbed to `@@cash+goog-exit`.
- Declaration-only line `@@foo` absorbed by a bare `@@` on a later item — the whole
  `@@foo` line is deleted, terminator included.
- Item with `@notes#Ideas` → `changed: false` plus the Rule A5 notice; same for
  `@dev^id`, `@dev:id`, and a trailing bare `#`.
- Item with two local markers → no rewrite (Rule A6).
- No bare `@@` anywhere → no rewrite, `text == input`.
- Two bare `@@` tokens with a cursor on the first → the first one claims; with no cursor
  → the last one claims.
- Idempotence: feeding the output back in produces no further change.
- Whitespace: no double space is left behind, and the resulting draft parses cleanly
  under `bob capture-parse` with no diagnostics.
- Multi-byte input (e.g. an emoji in the body) keeps every offset on a character
  boundary.

---

## Phase `macapp`: Mac capture panel absorbs `@@` as you type

**Repository:** `bob-mac-capture` (a linked repo). Open it with the `/sase_repo` skill
before reading or writing any of its files, and use only the path that command prints.

**Depends on:** `rewrite`.

**Build constraint to be aware of:** on a Linux host only the `CaptureCore` target
builds; the `BobMacCapture` app target and its tests are covered by macOS CI. Verify
what you can locally (`swift build --target CaptureCore` and the `CaptureCoreTests`),
keep the app-target changes tight and reviewable, and say plainly in the phase report
what could not be executed locally.

### `Sources/CaptureCore`

- New `CaptureRewriteResponse` (and `CaptureRewriteEdit`) `Decodable` models in
  `CaptureModels.swift` (or a small new file alongside it, matching the file's existing
  conventions), decoding the phase-`rewrite` JSON contract with `schema_version` 1,
  snake_case `CodingKeys`, and optional `cursor` / `rule` / `summary` / `notices`.
- `BobProcessClient.captureRewrite(_ draft: String, cursor: Int) async throws -> CaptureRewriteResponse`,
  arguments
  `["capture-rewrite", "--cursor", String(cursor), "--format", "json", "--", draft]`,
  `expectedSchema: 1`, on its own lane (`lane: "rewrite"`) so it never cancels the
  parse, completion, or preview lanes.

### `Sources/BobMacCapture/CapturePanelModel.swift`

- In `editorTextDidChange(cursorUTF8Offset:)`, after the existing
  `isApplyingProgrammaticDraft` and `suppressedCompletionAcceptanceDraft` guards and
  after the empty-draft early return, detect the trigger: the two UTF-8 bytes ending at
  the insertion offset are `@@` **and** the byte before them (if any) is not `@`. Keep
  this check tiny and purely positional — it is a trigger, not grammar; `bob` decides
  whether anything is actually absorbed.
- On trigger, fire the rewrite immediately (no debounce), cancelling any in-flight
  rewrite task, alongside — not instead of — the normal `scheduleAnalysis` call.
- On response, apply only when `plainDraft` is still byte-identical to the draft that
  was sent (same staleness discipline `applyParse` and the completion path already use).
  Then:
  - `changed == true` → set `suppressedCompletionAcceptanceDraft`, call
    `setPlainDraft(response.text, cursorUTF8Offset: response.cursor, suppressSelectionCallbacks: true)`,
    `announceStatus(response.summary)`, and
    `scheduleAnalysis(cursorUTF8Offset:requestCompletion:)` with the new cursor.
  - `changed == false` with a non-empty `notices` → do not touch the draft; announce the
    first notice so the Rule A5 safety net is visible.
  - `changed == false` with no notices → do nothing at all.
- A failed or cancelled rewrite must never disturb the draft or the status line; log or
  swallow it exactly like other best-effort lanes do.

### Tests

- `Tests/Fixtures/fake-bob`: answer the `capture-rewrite` command, driven by env vars in
  the same style the fixture already uses for the other capture commands.
- `Tests/BobMacCaptureTests/CapturePanelModelTests.swift`: the trigger fires and
  rewrites the draft with the announced summary; a stale response is dropped; a
  `changed: false` response with a notice announces without mutating the draft; typing a
  single `@` does not trigger; typing the second `@` of a `@@@` sequence does not
  trigger.
- `Tests/CaptureCoreTests/CaptureModelTests.swift`: decode a full and a minimal
  `capture-rewrite` payload, and reject a wrong `schema_version`.

### README

Update `README.md`'s global-destination section (currently around the "first nonblank
physical line" wording, the `@@foo` example block, and the completion-context bullets):

- The declaration is now a token anywhere in the draft.
- Typing a bare `@@` inside an item that already has a `@file` reference moves that
  reference onto the `@@`.
- The A5 notice cases and the shadow warning.
- Bump the "requires a `bob` build that supports…" note to mention
  `bob capture-rewrite`.

---

## Acceptance

- `@@route` / `@@route+block-id` works at the end of any line of any item, on parent and
  authored child lines, and the existing header spellings still behave identically.
- A second declaration is a clear error naming both lines.
- An item that declares a destination it will not use produces a visible warning from
  both `bob capture-parse` and `bob capture`.
- `bob capture-rewrite` turns `Buy milk @dev @@` into `Buy milk @@dev` with the cursor
  after the token, is idempotent, and explains itself when it cannot help.
- In the mac panel, typing `@@` on a note or task that already has a `@file` reference
  rewrites the line in place and says what it did.
- `docs/capture.md`, both `README.md` files, and every `--help` long text describe the
  new grammar with no leftover "header" or "first nonblank physical line" wording.
