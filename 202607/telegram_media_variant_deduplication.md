---
tier: tale
title: Deduplicate Telegram completion media variants
goal: 'Telegram agent-completion messages attach one representative for each logical
  motion-media artifact while SASE continues to retain every discovered file.

  '
create_time: 2026-07-17 13:56:44
status: done
prompt: 202607/prompts/telegram_media_variant_deduplication.md
---

# Plan: Deduplicate Telegram Completion Media Variants

## Context and root cause

The completed `sase-6l.1` run provides a concrete reproduction. Its durable `done.json` and completion notification
contain five modified GIF paths and five modified MP4 paths. Each GIF/MP4 pair has the same parent directory and stem,
and the notification contains no repeated path. This rules out repeated render checkpoints or duplicate manifest
insertion as the cause.

SASE is behaving correctly as the artifact producer: both encodings were modified, both remain useful in the Agents UI
and artifact-file inventory, and the completion notification preserves that lossless list. The duplication is introduced
in the linked `sase-telegram` repository. Workflow-completion formatting currently filters attachments only by whether
each path exists, and outbound delivery then sends every non-diff path independently. Consequently a GIF is sent as an
animation and its same-stem MP4 is sent again as a video, making one logical demo appear twice in Telegram.

This behavior became visible after completion media gained dedicated GIF and video routing. The inline-media-to-document
fallback is not the source of the reported case: it operates on one path after a send failure, whereas the recorded
notification already contains two successful candidates for every logical demo.

## Telegram-side selection policy

Implement the fix only in `sase-telegram`'s workflow-completion attachment formatting. Do not discard or rewrite
`done.json`, SASE notifications, or the default artifact-file inventory.

Before returning completion attachments, coalesce alternate motion-media encodings that have the same normalized parent
directory and exact filename stem. Limit this equivalence rule to formats that outbound already treats as animation or
video (`.gif`, `.mp4`, `.m4v`, `.mov`, and `.webm`, with case-insensitive suffix handling). Prefer GIF when a group
contains one because it is the authored animation artifact in the motivating case; otherwise prefer MP4 and then the
remaining supported video formats in a documented, deterministic order. Preserve the logical group's original position
so attachment ordering stays stable.

Keep unrelated paths lossless: same-stem media in different directories, static images such as a poster PNG beside a
GIF, documents, diffs, and media with different stems must all remain separate. Apply the policy only to
workflow-completion notifications so approval gates, custom gates, generated images, and other explicit attachment
surfaces continue to honor their exact file lists. The chosen attachment must still use the existing photo, animation,
video, PDF, conversion, and document-fallback delivery paths.

## Implementation and documentation

Open the linked `sase-telegram` repository through `/sase_repo` before making changes. Add a small, pure
attachment-selection helper near the notification formatting logic and call it from `_format_workflow_complete` after
path expansion/existence filtering. Keep outbound delivery focused on transport so normal sends, dry runs, and tests all
observe the same selected attachment list.

Update `docs/outbound.md` to explain that workflow completions send one preferred motion-media representation per
same-directory stem, while other artifacts and the underlying SASE inventory remain available. If a concise user-facing
attachment note exists in the repository README, keep it consistent with the detailed outbound documentation.

## Regression coverage

Add focused formatter and outbound tests that model the actual failure:

1. A completion containing multiple same-directory `{name}.gif` and `{name}.mp4` pairs yields only one attachment per
   stem, preferring each GIF, and the outbound loop calls animation delivery without also calling video delivery for
   those pairs.
2. A video-only artifact and distinct-stem GIF/video artifacts retain their existing routing and order.
3. Same-stem motion media in different directories are not coalesced.
4. A same-stem static image and animation are both retained.
5. Suffix matching is case-insensitive and selection remains deterministic even when a lower-priority video appears
   before its GIF.
6. Existing missing-file filtering and inline-media document fallback continue to work for the selected representative.
7. Non-completion notification types that carry explicit attachment lists are unchanged.

Run the focused formatting/outbound tests first, then run the linked repository's full validation (`just install` if
needed, followed by `just check`). The change is complete when the `sase-6l.1`-shaped fixture produces five motion-media
sends rather than ten, all existing attachment routing tests pass, and no SASE core or primary-repository source changes
are needed.

## Risks and guardrails

Basename-only grouping would incorrectly merge files from separate directories, and grouping static images with videos
could hide intentionally distinct poster art. Use the normalized parent plus exact stem and the narrow motion-media
extension set to avoid both errors. Keep the selection helper pure and covered by order-sensitive tests so future
transport additions cannot silently restore duplicate completion uploads.
