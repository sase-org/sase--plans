---
tier: tale
title: Glanceable Telegram epic phase counts
goal: 'Telegram epic approval messages show an accurate, polished phase count at a
  glance while preserving phase details, approval controls, fallbacks, and the message-size
  budget.

  '
create_time: 2026-07-18 10:53:54
status: wip
prompt: 202607/prompts/telegram_epic_phase_count.md
---

# Plan: Glanceable Telegram epic phase counts

## Context and product decision

`EpicApproval` messages are rendered by the linked `sase-telegram` repository. The formatter already reads the attached
plan once with SASE's safe frontmatter parser and renders the authored `phases` list in an ordered **Properties** card.
That card can become an expandable blockquote or have large values truncated to stay within Telegram's 4,096-character
limit, so a count placed only beside the full list would not always be visible at a glance.

Add the count to the bold review heading while leaving the detailed property unchanged:

```text
📋 Epic Review · 3 phases
```

The same compact suffix should compose with the existing provider/model and agent labels, use the singular `1 phase`,
and remain absent from tale reviews. This location makes scope immediately scannable, follows ACE's established
`epic · N phases` summary idiom, and avoids presenting a synthetic `Phase count` row as if it were authored plan
metadata.

## Telegram formatting and failure semantics

In `sase-telegram/src/sase_telegram/formatting.py`, derive the count from the already-parsed top-level `phases` value
before finalizing the review heading. Only an `EpicApproval` with a genuine non-string sequence should receive the
suffix; count the sequence entries themselves rather than rendered lines, phase body headings, dependencies, or
notification fields. Treat an empty sequence as `0 phases` defensively, even though validated epics are expected to
contain at least one phase.

Keep the extraction small and side-effect-free, with explicit singular/plural formatting. Missing files, unreadable or
malformed frontmatter, absent `phases`, and scalar or mapping-shaped values must omit the suffix instead of displaying
an invented count or failing notification delivery. A failure later in generic property rendering may still fall back to
the existing body-only preview while retaining an already verified count.

Preserve the current phase list exactly in the **Properties** card. Do not add a gate schema field, notification
`action_data` field, second plan read, or SASE core/backend behavior: this is presentation derived from the same
document the Telegram frontend already owns. The expanded heading must continue to participate in the existing shared
message-budget calculation so notes, properties, and body truncation still guarantee `MAX_MESSAGE_LENGTH`. Approval
keyboards, attachments, callbacks, and legacy no-bundle behavior remain unchanged.

## Coverage and visual contract

Extend `sase-telegram/tests/test_formatting.py` with exact, readable assertions for plural and singular headings,
including composition with provider/model, agent, and runtime metadata. Cover tale isolation and defensive epic inputs:
missing/invalid frontmatter and non-sequence `phases` values should render the ordinary `Epic Review` heading without a
count and without disrupting the existing fallback or attachment behavior.

Exercise a metadata-heavy epic whose phase details force the Properties card to expand or truncate. Assert that the
count remains in the heading, the complete phase property is still represented according to the established truncation
rules, and the full message stays within 4,096 characters. Update the command-backed epic gate coverage in
`sase-telegram/tests/test_custom_gates.py` to prove that a real validated gate displays the count while retaining its
keyboard and attachment.

Document the glanceable heading count and its best-effort omission behavior in `sase-telegram/README.md` and
`sase-telegram/docs/outbound.md`, alongside the existing Properties-card and budget description. Keep examples focused
on the visible result rather than internal helper names.

## Validation

From the linked `sase-telegram` checkout, install the current editable dependencies if needed, run the focused formatter
and command-backed gate tests, then run the repository's complete required check:

```bash
just install
just test tests/test_formatting.py tests/test_custom_gates.py
just check
```

Re-read the exact rendered-heading assertions after formatting to confirm the MarkdownV2 escaping, separator spacing,
singular/plural wording, and visual hierarchy remain intentional rather than merely syntactically valid.
