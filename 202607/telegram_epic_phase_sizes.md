---
tier: tale
title: Glanceable Telegram epic phase sizes
goal: 'Telegram epic approval messages show a compact, trustworthy phase-size breakdown
  while retaining their existing phase count, complete properties, attachments, controls,
  MarkdownV2 safety, and delivery fallbacks.

  '
create_time: 2026-07-20 14:47:00
status: wip
prompt: 202607/prompts/telegram_epic_phase_sizes.md
---

# Plan: Glanceable Telegram epic phase sizes

## Context and contract

Epic approval formatting in the linked `sase-telegram` repository already reads the plan once, parses its frontmatter
for a lossless Properties card, and derives a best-effort raw phase count for the review heading. The main SASE package
now exposes normalized Rust-backed plan validation through
`sase.sdd.plan_validate.validate_plan(content, "epic", mode="launch")`; successful results provide validated `small`,
`medium`, and `large` values, while legacy missing sizes normalize to `small` and explicit invalid values remain errors.

Keep the raw phase-count contract independent from semantic validation so malformed modern plans can still retain any
safe sequence count the formatter already knows. Treat the size breakdown as derived presentation only: it must come
from a successful epic validation result, use fixed `small`, `medium`, `large` ordering, omit zero buckets and empty
epics, and disappear quietly when validation, imports, bindings, or preview work are unavailable. Never infer values
from arbitrary frontmatter or weaken message delivery to show the summary.

## Formatting pipeline

Extend the single-read plan-document pipeline to pass the already-read source text into a small best-effort helper that
invokes the normalized validator in launch-consumption mode. Convert only a successful nonempty normalized phase list
into readable MarkdownV2-safe text such as `Phase sizes: 2 small · 1 medium · 1 large`; return no line for tales,
zero-phase epics, invalid plans, unavailable validator capability, or any preview exception. Keep imports defensive so
installations paired with an older SASE version degrade by omission rather than failing notifications; do not raise the
dependency floor to an unreleased SASE version.

Place the nonempty line directly below the bold review heading and optional agent attribution, composing predictably
with provider/model and runtime lines. Make it part of `header_text` before the existing shared 4,096-character
allocation so notes, the complete-label Properties card, and the body continue to truncate within their established
priorities. Preserve the detailed nested `phases[].size` properties, raw phase-count suffix, expandable blockquotes,
truncation markers, attachment selection, gate loading, callback payloads, and inline keyboard layout. Property-card
formatting failure must retain both independently computed heading metadata and continue through the legacy preview.

## Verification

Expand formatter tests with valid single-size, mixed-size, repeated-size, and legacy missing-size epics, asserting exact
line ordering and MarkdownV2-readable separators alongside provider/model, agent, and runtime metadata. Cover empty,
nonmapping, and malformed phases; explicit invalid sizes and unrelated invalid epic structure; missing/unreadable files;
validator import/capability/runtime failures; and Properties-card fallback. In each degradation case assert that no
confident size line is emitted while the raw phase-count behavior, attachment list, controls, and preview remain intact.

Extend metadata-heavy tests to ensure the summary participates in the 4,096-character cap without dropping property
labels or truncation pointers. Update command-backed epic gate coverage so a real validated gate demonstrates legacy
missing sizes normalizing to `small` while its attachment and keyboard remain unchanged. Run focused formatter and gate
tests during iteration, then install current dependencies and run the linked repository's mandatory `just check`.

## Documentation and scope

Update the Telegram README and outbound documentation to describe the new line, its fixed ordering, its relationship to
the detailed Properties card, legacy `small` normalization, and best-effort omission on validation/capability failure.
Keep this phase confined to `sase-telegram`: the authoritative validator and normalized phase-size wire already exist in
SASE, and no duplicate enum, core validation rule, keyboard behavior, or parent-epic lifecycle change belongs here.
