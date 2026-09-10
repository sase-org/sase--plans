---
tier: tale
title: Normalize bullet-list paste into an empty capture bullet
goal:
  Command-V consumes a pasted list's duplicate first marker and rebases the list onto
  the capture placeholder's indentation without changing ordinary plain-text paste
  behavior.
size: medium
proposed_by: bbugyi200.athena.091
create_time: 2026-09-09 20:00:15
status: wip
---

# Normalize bullet-list paste into an empty capture bullet

## Goal

Make Command-V treat the `- ` row created by Ctrl-J as the first bullet of a pasted
Markdown list instead of producing `- - ...`. Preserve the pasted list's relative
indentation while rebasing its outer level onto the placeholder row's supported source
indentation (column zero or two ASCII spaces).

The implementation belongs entirely in `bob-mac-capture`'s native plain-text paste path.
It must continue to submit the resulting source text to `bob-cli` unchanged; there is no
capture grammar or JSON-contract change.

## Current behavior and constraints

- `CaptureBulletNewlineEditResolver` inserts a canonical `- ` row and copies the current
  authored bullet's supported indentation.
- `PlainTextPaste.insert` newline-normalizes an explicitly advertised plain-text
  clipboard flavor and inserts it at the `NSTextView` selection. Consequently, pasting
  `- first\n- second` after `- ` currently produces `- - first\n- second`.
- Bob recognizes authored child rows at column zero and nested child rows with exactly
  two ASCII spaces. A paste into ` -` therefore needs to shift the pasted list's outer
  level by two spaces; otherwise later siblings escape to column zero.
- The special case must remain a native `NSTextView` edit so undo, typing attributes,
  IME behavior, accessibility, completion dismissal, and the existing rich-paste
  avoidance remain intact.

## Implementation

1. Add a small pure paste-edit resolver alongside `PlainTextPaste` that accepts the
   draft string, the text view's UTF-16 `NSRange`, and normalized clipboard text. Have
   it return the replacement range/text needed for one native insertion. Keep ordinary
   paste as its default result, and activate bullet joining only when all of these are
   true:
   - the selection is collapsed at the end of a physical line;
   - that complete line contains only one supported authored bullet prefix (`-`, `*`, or
     `+`), its optional canonical indentation (zero or exactly two ASCII spaces), and
     trailing horizontal whitespace;
   - the clipboard's first physical line is a nonempty Markdown bullet with a supported
     marker and whitespace before its body.

   This deliberately leaves prose, marker-like tokens such as negative numbers,
   marker-only clipboard content, noncollapsed selections, mid-line carets, unsupported
   indentation, and non-placeholder rows on the existing exact plain-text splice path.

2. For the joined case, retain the placeholder's existing marker and canonicalize its
   marker/body separator to one space, remove the first pasted bullet's indentation,
   marker, and separator, and append its body. Rebase each subsequent nonblank pasted
   line from the first pasted bullet's leading-space baseline onto the placeholder's
   zero- or two-space indentation, preserving relative indentation and the pasted
   markers/bodies. Preserve blank lines and a trailing newline without introducing
   whitespace-only rows. Do not flatten, clamp, or otherwise reinterpret deeper pasted
   nesting: source preservation and `bob-cli`'s live parse/preview remain authoritative
   outside the representable two-level capture hierarchy.

3. Route `PlainTextPaste.insert` through the resolver and apply its range and text with
   `NSTextView.insertText`. Keep the current editable-responder and advertised `.string`
   guards, newline normalization, `willInsert` callback timing, signpost, and
   native-paste fallback behavior unchanged. Ensure the resulting caret is collapsed at
   the end of the inserted block and that the joined paste is a single undoable edit.

4. Update the README's Command-V behavior and bullet-editing narrative to explain that
   pasting a Markdown bullet list into an otherwise empty bullet row consumes the
   clipboard's first marker and rebases the list onto that row's indentation. Include a
   compact top-level example and note that all other clipboard content still follows
   ordinary plain-text insertion.

## Tests

Extend `Tests/BobMacCaptureTests/PlainTextPasteTests.swift` with focused pure-resolver
and `NSTextView` integration coverage:

- `Parent\n- ` plus `- first\n- second` becomes `Parent\n- first\n- second`, with the
  caret after `second`.
- A two-space placeholder plus a flat pasted list keeps every pasted sibling at two
  spaces, rather than allowing later rows to outdent to column zero.
- A clipboard list whose first bullet already has leading indentation is rebased from
  that baseline, while relative child indentation is preserved on later lines.
- Existing placeholder markers `*` and `+` are retained even when the clipboard uses a
  different supported marker; CRLF/lone-CR clipboard content is normalized before the
  contextual edit.
- Blank lines and a terminal newline remain blank/newline-only after rebasing.
- Prose, unsupported or marker-only first clipboard lines, invalid/multiline selections,
  mid-line carets, ordinary authored bullet rows, and unsupported placeholder
  indentation remain exact normalized plain-text insertions.
- The integration path still calls `willInsert` once, preserves uniform typing
  attributes/no imported links, and declines noneditable or unrelated responders just as
  it does today.

Run from the `bob-mac-capture` repository:

```bash
just format-lint
just build
just test
```

## Acceptance criteria

- The user's Ctrl-J then Command-V workflow never produces a duplicate first marker when
  a supported Markdown list is pasted into the empty bullet row.
- Top-level and nested placeholders yield correctly aligned pasted siblings, without
  damaging the pasted list's relative indentation.
- Contexts outside that narrow workflow retain today's exact plain-text paste and
  fallback semantics.
- Documentation, formatting, build, and the full Swift test suite pass.
