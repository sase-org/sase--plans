---
tier: tale
title: Fix bead show crash on attachment span offsets
goal:
  sase bead show and the pager bead resolver render every attachment-bearing bead in
  rich color mode, with attachment chips bound to the exact chip text.
size: small
proposed_by: bbugyi200.athena.0ub
create_time: 2026-09-30 06:59:36
status: wip
---

# Plan: Fix `sase bead show` crashing on beads with note attachments

## Goal

`sase bead show <id>` in a color terminal, and the pager's bead-link resolver, must
render every bead whose notes or +1 evidence carry attachments. Today every such bead
crashes with:

```
ValueError: attached target span 5472:5493 exceeds body length 4821 for section bead:sase-1d5.1
```

The bead data is valid; nothing needs rewriting in the bead store. The defect is purely
in how the show renderer computes the clickable attachment spans.

## Context: did `sase-1d6.2` use bead attachments correctly?

Yes. The salvage agent posted each patch using the documented inline grammar:
`sase bead note <id> "... @/path/to/file ..."`. That grammar stores an
`@attachment:<name>` token plus a content-addressed manifest entry, and the bead renders
it as a `[<name>]` chip and an ATTACHMENTS descriptor row. Every `attach` reported
`stayed local on this machine`, and `sase bead attachment push` reported that no shared
attachment store is configured. The agent recorded that caveat and kept second copies
outside any checkout, which was the right call. That is a machine setup gap, not misuse,
and it is out of scope here. The agent's notes simply exercised a renderer bug that
every attachment-bearing bead already had.

## Diagnosis (already confirmed; do not re-investigate)

- `_show_batch_sections` in `src/sase/bead/cli_show_batch.py` renders each bead with
  `render_issue_detail(...)`. With `style=DetailStyle.RICH` the result is a `str` full
  of ANSI SGR escapes.
- It then calls `attachment_targets_for_body(issue.id, body, names)`
  (`src/sase/bead/show_images.py`), which locates `[name]` chips and ATTACHMENTS
  descriptor rows with `str.find`, so the offsets it returns are in _ANSI-string_
  coordinates.
- `PagerSection.__post_init__` (`src/sase/pager/document.py`) converts the body with
  `Text.from_ansi` and validates every `AttachedTarget` against `len(plain_text)`. The
  escape bytes shift the offsets, so they exceed the plain length (crash). When they
  happen to fit, they silently cover the wrong text.
- `DetailStyle.PLAIN` has no escapes, which is why `sase bead read` (piped, plain) and
  `--style plain` work while `--color always --style rich` crashes.
- Introduced by
  `56d5cd277e feat(bead): view note attachments from bead show with image previews`. The
  only unit test, `test_pager_targets_and_media_order` in
  `tests/test_bead/test_show_images.py`, feeds `attachment_targets_for_body` a
  hand-written plain string. It never runs the real rich path.
- Blast radius: every attachment-bearing bead in `sase bead show` on a TTY (color is the
  default), plus `src/sase/pager/beads.py`, which always builds bead documents with
  `DetailStyle.RICH`. That path catches the error and reports the bead as "could not be
  resolved". The five beads that surfaced this (`sase-1d5.1`, `sase-1cx.1`,
  `sase-1cj.12.1`, `sase-1co`, `sase-1ck`) all fail the same way.
- A prototype that feeds `attachment_targets_for_body` the `Text.from_ansi` plain text
  made all five beads render. Each resulting span covers exactly `[<name>]` or `<name>`.

## Changes

### 1. Compute attachment spans in the section's own plain-text coordinates

In `_show_batch_sections` (`src/sase/bead/cli_show_batch.py`):

- Build the `PagerSection` first, with no attachment targets (all other fields as
  today).
- Then, inside the existing best-effort `try: ... except Exception:` block, collect
  attachment names and call
  `attachment_targets_for_body(issue.id, section.plain_text, names)`. When that returns
  targets, replace the section with `dataclasses.replace(section, targets=targets)`.
  `replace` re-runs `__post_init__`, which rebuilds `_body_text` and re-validates the
  spans. This is already confirmed to work on this slotted frozen dataclass.
- Keep the `replace` call inside the `try` block. Attachment spans are optional
  enrichment, so a future span bug can at worst drop the clickable chips and never make
  a bead unreadable again. On failure, append the untargeted section.
- `PagerSection.plain_text` is the exact coordinate space the validator uses, and it
  also covers the trailing-newline normalization. Do not re-derive plain text with a
  private helper or a separate `Text.from_ansi` call.
- `cli_show_batch.py` is already at 696 lines and toobig's lowest threshold is 700. Keep
  the file net-neutral or smaller. Move the note plus +1-evidence attachment-name
  collection loop into a small public helper in `src/sase/bead/show_images.py` (for
  example `attachment_names_for_issue(issue) -> list[str]`, added to `__all__`) and call
  it from `_show_batch_sections`.

### 2. Document the coordinate contract

In `attachment_targets_for_body` (`src/sase/bead/show_images.py`), expand the docstring
to say that `body_plain` must be ANSI-free: the owning `PagerSection.plain_text`. The
returned offsets are validated against that text. Do not add ANSI stripping inside the
function. The caller owns the coordinate space.

Leave `PagerSection`'s strict span validation unchanged. It correctly caught this bug.

### 3. Regression tests

Add tests next to the existing show-image tests (`tests/test_bead/test_show_images.py`,
or a new focused module under `tests/test_bead/` if that file would grow awkwardly).
Build the bead in memory, with no CLI subprocess and no real bead store:

- Build an `Issue` (from `sase.bead.model`) with a long styled prefix (a multi-paragraph
  description and several notes) so that ANSI offsets would clearly exceed the plain
  length. Add one note whose text contains an `@attachment:<name>` token, with a
  matching `BeadNoteAttachment(name=..., sha256=..., size_bytes=..., mime_type=...)`
  manifest entry (see `_note_issue` / `_descriptor` in
  `tests/test_bead/test_attachment_lifecycle.py` for the shape). Use neutral file names
  such as `trace.log` and `shot.png`. Avoid the word "patch" in fixtures because the
  repo's terminology audit lint polices it.
- Resolve it through `resolve_show_batch` with an in-memory view and a no-network render
  context. `tests/test_bead/test_bead_show_pager.py` has `_view(...)` and
  `_plain_render_context` to copy. Then call `build_show_batch_document(...)` with
  `style=DetailStyle.RICH` (the crashing case) and again with `DetailStyle.PLAIN`.
- Monkeypatch `SASE_HOME` to a tmp dir so the attachment status lines never touch the
  real attachment cache.
- For both styles, assert that construction does not raise, that the section has at
  least one `artifact_ref` target per attachment, and that every target's
  `section.plain_text[t.start:t.end]` equals `[<name>]` or `<name>`. Also assert that
  `str(t.target) == f"attachment:{issue_id}/{name}"`.
- Add a unit test for the new name-collection helper that covers note attachments and
  +1-evidence attachments.
- Keep `test_pager_targets_and_media_order` as-is. It still correctly unit-tests the
  plain-text function.

## Verification

1. Run the new and neighboring tests:
   `.venv/bin/pytest tests/test_bead/test_show_images.py tests/test_bead/test_bead_show_pager.py tests/pager/test_document.py -q`.
   Also run the new module, if you created one.
2. Real-bead smoke test (read-only), using this checkout's own binary so the new code
   runs. Each command must exit 0 and print the bead:

   ```bash
   for b in sase-1d5.1 sase-1cx.1 sase-1cj.12.1 sase-1co sase-1ck; do
     .venv/bin/sase bead read "$b" --color always --style rich \
       -r "Verify attachment span fix renders colored bead detail" >/dev/null || echo "FAIL $b"
   done
   ```

3. Run `sase tool run check` and fix anything it reports.

## Non-goals

- Do not edit, re-note, or rewrite any bead or note. The stored data is correct.
- Do not change attachment storage, shared-store configuration, or the
  `@<path>`/`@attachment:` grammar.
- Do not relax `PagerSection` target validation.
- No `sase-core` change: this is presentation-only span mapping in the Python pager
  adapter.
