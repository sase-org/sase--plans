---
tier: tale
goal:
  Bare clipboard captures create unlabeled child bullets, while explicit clipboard
  headers retain their current formatting and behavior across every capture mode.
create_time: 2026-09-09 19:53:14
status: wip
---

# Plan: Make the clipboard capture header truly optional

## Product context

`bob capture` currently recognizes a bare terminal `%` marker and a bare `--clip` option
as clipboard-capture requests, but internally turns the omitted `<bullet_header>` into
the synthetic header `clip`. Every default capture therefore writes `**CLIP:**` before
the clipboard text or reference. An explicit marker such as `%build_log`, or an explicit
option such as `--clip=build_log`, formats the supplied header in uppercase, replaces
underscores with spaces, and writes it in bold before the captured content.

Change the omitted-header behavior so omission means absence: bare `%` and bare `--clip`
still request clipboard capture, but add no label, punctuation, bold span, or empty
placeholder bullet. Explicit headers retain the existing grammar, validation,
formatting, classification, file handling, atomicity, dry-run behavior, and output
confirmations.

## Behavioral contract

- A bare `%` terminal marker requests clipboard capture with no header. It remains
  composable with schedule markers, ordinary routes, bullet routes, forced
  routes/sections, and Pomodoro routes in all currently supported terminal positions.
- Bare `-c` / `--clip` also requests clipboard capture with no header and continues to
  keep `%...` tokens in the capture text literal. Preserve the current `--clip[=HEADER]`
  and `require_equals` interface so an optional header never consumes the following text
  argument.
- `%<header>` and `--clip=<header>` keep the current `[A-Za-z0-9_-]+` validation and
  rendered-header transformation. Invalid terminal `%...` tokens remain literal, invalid
  explicit option values remain usage errors, and `--no-clip` continues to disable
  marker parsing and conflict with `--clip`.
- The exact Markdown rendering depends on both content shape and header presence:
  - Headerless inline text, a single attachment, or a snippet link is one direct child
    bullet, such as `  - clipboard text`, `  - ![[img/example.png|400]]`, or
    `  - [[file/clip-...]]`.
  - Headerless flat multi-line text and multiple attachments are direct sibling child
    bullets at two-space indentation, such as `  - first` followed by `  - second`. Do
    not emit an empty `  -` container or retain four-space nesting when there is no
    header to own that nested list.
  - Explicitly headed captures retain the existing layouts: inline/reference content
    follows `**HEADER:**` on the same child bullet, while multi-line/multi-attachment
    content nests beneath a `  - **HEADER:**` container.
- Parent task, ordinary bullet, and Pomodoro lines remain byte-for-byte unchanged; only
  the clipboard child lines differ when the header was omitted. Content classification,
  attachment copies/reuse, snippet naming, note placement, and rollback semantics remain
  unchanged.

## Representation and flow

- Refactor the capture grammar so “clipboard capture requested” is distinct from “an
  explicit header was supplied.” Use a small request/marker value carrying an optional
  header (or an equivalently explicit representation) rather than overloading
  `Option<String>` such that `None` means both no capture and no header.
- Make both marker parsing and command-line parsing produce that same semantic value.
  Remove the `clip` default header constant and replace the current
  `default_missing_value("clip")` behavior with presence-aware handling for bare
  `--clip`; preserve clap's no-argument-vs-following-text behavior and explicit header
  validation.
- Pass the optional header into clipboard planning. Render and store the
  uppercase/underscore-normalized header only when one was explicitly supplied, and
  centralize the header-aware line layout so inline, lines, attachments, and snippet
  modes follow the same rules.
- Keep the JSON `clip.header` key present for clipboard captures, but make it nullable:
  headerless capture reports `"header": null`, while an explicit `%build_log` continues
  to report `"header": "BUILD LOG"`. This represents semantic absence without an
  ambiguous empty-string sentinel and preserves a predictable `clip` object shape. The
  `clip.lines` array remains the exact Markdown written to the note.

## Help and documentation

- Update `bob capture --help` to say that `%` and bare `--clip` capture without a
  header, removing every claim that a bare request renders `**CLIP:**`. Keep
  explicit-header examples so the formatting transformation remains discoverable.
- Update the README marker, option, rendering/classification, and JSON sections with
  headerless examples for single and multiple items, explicit-header examples, and the
  nullable `clip.header` contract.
- Retain the documented `%20` accepted quirk, clipboard source resolution, file naming,
  failure behavior, and all other unrelated clipboard guidance.

## Verification

- Update parser unit tests to distinguish a recognized bare clip request with
  `header: None` from no clip request, while retaining coverage for explicit headers,
  schedule/route composition, duplicate-marker behavior, literal invalid or mid-text `%`
  tokens, forced route/section parsing, and `--no-clip`.
- Add focused rendering tests for headerless and explicitly headed variants of all four
  modes: inline, flat lines, single/multiple attachments, and snippet. Assert exact
  indentation and ensure headerless collection modes have no blank container bullet.
- Update CLI integration tests so bare `%` is exercised under task/bullet/Pomodoro
  placement and bare `--clip` is exercised independently; assert there is no `CLIP`,
  bold marker, or header punctuation in the resulting child lines. Preserve explicit
  `%header` and `--clip=header` assertions as regressions.
- Assert JSON uses `header: null` for omitted headers, retains a rendered string for
  explicit headers, and reports exact headerless `lines`. Keep attachment reuse, snippet
  creation, dry-run, failure atomicity, help ordering, and non-clip JSON behavior
  covered.
- Run `just all` and require formatting, Clippy, unit tests, and CLI integration tests
  to pass.

## Risks and boundaries

- A headerless multi-item capture intentionally changes the outline shape from one
  labeled child with grandchildren to multiple direct children. This is the only layout
  that avoids a visually empty bullet while keeping every captured item represented as a
  sub-bullet of the new parent.
- Consumers that assumed `clip.header` was always a string must accept `null` for the
  newly meaningful omitted-header case; explicitly headed captures remain compatible.
- Do not add a configuration option for a default header, reinterpret an explicitly
  empty header value as valid, change clipboard classification thresholds, or migrate
  existing notes.
