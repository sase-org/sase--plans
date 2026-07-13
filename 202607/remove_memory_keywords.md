---
create_time: 2026-07-13 07:01:21
status: done
prompt: 202607/prompts/remove_memory_keywords.md
tier: tale
---
# Remove the Memory `keywords` Property

## Goal

Retire the `keywords` property from SASE memory notes end to end. Canonical `memory/*.md` files, generated memory
documentation, proposal creation and review, CLI/configuration surfaces, and the packaged diagram should no longer
describe, accept, store, display, filter by, or emit memory keywords.

This change is intentionally scoped to the memory feature. Unrelated uses of “keyword” in Python call syntax, query
languages, xprompt directives, package metadata, and other domains remain untouched.

## Current Behavior and Compatibility

- Memory frontmatter officially uses `type`, `parent`, and `description`, but the canonical frontmatter renderer
  preserves arbitrary extra keys. That currently lets legacy `keywords` metadata survive `sase memory init` rewrites.
- `sase memory write --keyword` normalizes keyword values, records them in proposal ledger events and reduced state,
  exposes them in JSON and review output, includes them in TUI filtering, and promotes them into an approved note.
- The generated README template, public memory documentation, top-level README, configuration schema, and memory
  architecture diagram still advertise the retired concept.
- Proposal ledgers are append-only and may already contain keyword-bearing version-1 events. New code should ignore
  unknown fields in those historical JSON objects, preserving the existing general forward/backward-tolerant reader, but
  must not model, expose, or copy their retired keyword values. A schema-version bump is unnecessary and would make
  otherwise valid historical proposals disappear.

## Implementation Plan

1. **Make canonical memory frontmatter forget the retired property.**
   - Update the memory frontmatter rendering path so `keywords` is omitted when legacy frontmatter is normalized, while
     continuing to preserve unrelated extension metadata such as proposal provenance.
   - Replace the existing preservation assertion with coverage proving that the retired property is dropped and other
     extra metadata remains stable.
   - Update managed-memory initialization coverage so a legacy long note is migrated away from `keywords` during the
     normal frontmatter synchronization flow.

2. **Remove keyword data from the memory proposal domain.**
   - Delete keyword fields from proposal events and reduced proposal state, remove keyword normalization and its public
     exports, and stop accepting keyword inputs in proposal creation.
   - Stop serializing keywords into new ledger rows and stop requiring them when deserializing proposal events. Retain
     the ledger reader’s generic tolerance for additional JSON keys so pre-change rows still reduce normally, with the
     old value ignored.
   - Simplify approval rendering so promoted long-term notes contain only canonical metadata plus provenance, never
     keyword frontmatter.

3. **Remove user-facing proposal support.**
   - Delete `--keyword` from `sase memory write`, its handler plumbing, help text, examples, and parser tests.
   - Remove keywords from text/JSON proposal expectations, static review output, review TUI previews/details, and the
     TUI filter haystack. Keep filtering over proposal id, status, title, target, and author.
   - Adjust proposal, review, notification-adjacent, and TUI fixtures to use the keyword-free data model and assert that
     approvals and serialized payloads do not reintroduce the property.

4. **Migrate tracked memory files and generated artifacts.**
   - Remove `keywords` YAML properties from every tracked Markdown note under `memory/`, preserving all other
     frontmatter and note bodies exactly.
   - Remove the property from the packaged memory README template and refresh the generated `memory/README.md`,
     including any note statistics affected by deleted frontmatter lines.
   - Update the memory-directory-map source prompt and deterministic label recipe, edit/regenerate the packaged PNG to
     remove the `keywords:` row, copy it to `memory/assets/`, and verify the two committed PNGs are byte-identical.

5. **Clean up public documentation and stale schema surface.**
   - Rewrite the top-level Memory overview so it describes explicit audited reads rather than keyword-triggered context.
   - Remove keyword flags, examples, approval frontmatter, and filter claims from the memory and configuration docs.
   - Remove the orphaned `keywords` entry described as dynamic-memory matching from the xprompt configuration schema,
     ensuring configuration validation no longer advertises or accepts that legacy memory mechanism.
   - Remove or replace keyword-specific examples in generic frontmatter-stripping tests and any other memory-focused
     prose/assets so repository searches show no remaining memory-keyword contract.

6. **Verify the removal and guard against regressions.**
   - Run focused memory note, proposal, CLI, initialization, review, and review-TUI tests while iterating.
   - Search the scoped memory code, tests, docs, templates, schema, and tracked `memory/*.md` files for stale
     `keywords`/`--keyword` references, reviewing any remaining matches to distinguish unrelated terminology.
   - Run `sase memory init --check` to confirm generated memory files are current and compare the two diagram PNGs.
   - Because repository files are changing, run `just install` before validation and finish with the required
     `just check` suite.

## Expected Outcome

New proposals and approved notes have no keyword field; historical proposal rows still load but their old extra field is
ignored; initialization removes legacy note frontmatter instead of preserving it; all tracked memory notes and
generated/public documentation are keyword-free; and no CLI, schema, review UI, or diagram suggests the property is
supported.
