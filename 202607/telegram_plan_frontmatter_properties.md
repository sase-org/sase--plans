---
tier: tale
title: Beautiful Telegram plan properties
goal: Telegram plan approval messages present every frontmatter property clearly and
  remain safe, compact, and actionable for every valid plan.
create_time: 2026-07-16 12:35:22
status: wip
prompt: 202607/prompts/telegram_plan_frontmatter_properties.md
---

# Plan: Beautiful Telegram plan properties

## Context and product outcome

Telegram plan reviews currently render the plan body while `markdown_to_telegram_v2()` deliberately strips the leading
YAML frontmatter. That keeps raw YAML out of the prose, but it also hides decision-critical properties such as the
authored tier, title, goal, model, phases, dependencies, and future schema fields. ACE's Plans detail surface now treats
those values as a distinct properties panel; Telegram should offer the same information hierarchy in a chat-native form.

The implementation belongs in the linked `sase-telegram` repository. The coding agent must open that repository through
`/sase_repo` before reading or editing it. No SASE notification-schema or Rust-core change is expected: every
`PlanApproval` notification already carries the authoritative plan path, and the Telegram formatter already reads that
file to build the preview and attachment.

## Review experience

Keep the existing provider/model title, agent attribution, runtime, review note, attachment, and action keyboard.
Between that context and the Markdown body, add a visually distinct `Properties` card with compact label/value rows:

- Include every top-level frontmatter key; do not hard-code an allowlist that would silently hide optional or newly
  introduced plan fields.
- Use the same predictable semantic ordering as ACE: familiar identity and lifecycle fields first (`title`, `tier`,
  `kind`, `status`, creation timestamps, then `goal`), followed by every remaining key in stable case-insensitive
  alphabetical order.
- Humanize labels by turning underscores into spaces while preserving the full property value. Render scalar values
  compactly, use an explicit em dash for empty/null values, and render lists or mappings as indented multiline content
  so epic phases and `depends_on` relationships remain readable rather than becoming an ambiguous comma-separated blob.
- Treat labels and values as data, not Markdown. Escape every user-authored character through the existing MarkdownV2
  utilities, preserve Unicode, and use restrained typography/dividers so the card scans cleanly on a phone.
- Keep short property cards open. Wrap a long card in Telegram's existing expandable-blockquote treatment so detailed
  phase metadata is available without overwhelming the chat.

The plan body remains a separate rich Markdown preview and must not repeat the raw YAML. Plans without usable
frontmatter retain the current body-only presentation.

## Parsing and delivery design

Refactor the plan-approval formatter into a small read/parse/render pipeline in `src/sase_telegram/formatting.py`
(extract focused pure helpers if that keeps the module readable):

1. Read the attached plan once as UTF-8 and split it with SASE's safe frontmatter parser, retaining both the parsed
   mapping and the Markdown body. Do not evaluate YAML tags or interpolate values. Treat absent, unclosed, malformed,
   non-mapping, unreadable, and invalid-UTF-8 inputs as best-effort preview failures rather than notification failures.
2. Convert the parsed mapping into stable display rows with a recursive, deterministic value renderer. Preserve sequence
   order and mapping order within nested values, stringify booleans/numbers/dates predictably, and ensure empty
   containers are visibly distinct from missing values.
3. Assemble the header, notes, property card, and body under one explicit `MAX_MESSAGE_LENGTH` budget. Property names
   are never dropped. If unusually large values (especially `prompt`, `goal`, or `phases`) exceed the property budget,
   truncate only their displayed value with a clear `see attached plan` marker; then spend the remaining budget on the
   body and retain its existing truncation marker. This keeps the approval keyboard on one valid Telegram message and
   makes the attached plan/PDF the lossless escape hatch.
4. Continue attaching the original plan whenever it exists, even when parsing or preview rendering falls back. Preserve
   all callback payloads and button layout unchanged.

Keep this behavior future-compatible: Telegram should display unknown-but-parseable keys rather than needing a release
for each additive plan-schema field. The formatter may mirror ACE's ordering and label conventions, but the
chat-specific layout and length policy should remain owned by `sase-telegram` instead of importing TUI presentation
code.

## Verification

Extend `tests/test_formatting.py` with a representative exact rendering test plus focused edge cases:

- A short tale shows `title`, `tier`, and `goal` once, in the Properties card before the body, while preserving the plan
  attachment and existing two-row approval keyboard.
- A property-rich epic shows every scalar, empty value, Unicode/special-character value, nested phase mapping, and
  dependency list in stable order with valid MarkdownV2 escaping.
- An unfamiliar top-level key is still rendered, proving the display is not coupled to today's schema.
- Long goals/prompts/phases and a long body stay within `MAX_MESSAGE_LENGTH`, retain every property label, mark
  truncated values/body clearly, keep the keyboard actionable, and retain the full attachment.
- Missing frontmatter, malformed YAML, a non-mapping block, an unreadable/missing file, and invalid UTF-8 all degrade
  without raising; legacy frontmatter-free plan messages remain materially unchanged.

Update `docs/outbound.md` and the README's outbound overview to describe the Properties card, nested-value presentation,
expandable behavior, and the attachment fallback. Run `just install` followed by `just check` in `sase-telegram` and
manually inspect the dry-run MarkdownV2 output for one tale and one multi-phase epic to confirm that the hierarchy is
attractive and that frontmatter never appears twice.

## Acceptance criteria

- Every parseable top-level plan frontmatter property has a visible label in a Telegram plan approval message, with
  understandable values for both tale and epic shapes.
- Normal messages are compact and polished; metadata-heavy messages remain expandable, within Telegram's limit, and
  backed by the complete plan attachment.
- Frontmatter parsing or rendering can never prevent delivery of the review message or its approval controls.
- Existing plan actions, callbacks, attachment behavior, and frontmatter-free formatting continue to work unchanged.
