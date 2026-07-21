---
tier: tale
title: Move SDD cross-links into Markdown bullets
goal: 'New SDD plans and prompt snapshots expose their counterpart as a clickable
  top-of-body Markdown bullet on GitHub, while legacy frontmatter links remain readable
  and explicitly migratable without polluting artifact content or search.

  '
create_time: 2026-07-21 10:54:39
status: wip
prompt: '[202607/prompts/top_level_sdd_link_bullets.md](prompts/top_level_sdd_link_bullets.md)'
---

# Plan: Move SDD cross-links into Markdown bullets

## Context and outcome

GitHub renders YAML frontmatter as a property table but does not interpret Markdown link syntax inside property values,
so the recently introduced `prompt` and `plan` frontmatter links remain literal text. Move the canonical relationship
metadata into ordinary Markdown, where GitHub renders it as a link, without moving the rest of the plan or prompt
frontmatter.

For an artifact with YAML frontmatter, keep the opening delimiter at byte zero and place the link bullet immediately
after the closing delimiter as the first Markdown body element. For an artifact without frontmatter, make the bullet the
first line of the file. In both cases, emit exactly one blank line after the bullet before the existing content,
including an H1 when one exists. A flat plans-sidecar pair should therefore have these canonical shapes:

```markdown
---
tier: tale
title: Example
goal: Demonstrate the plan-side layout.
---

- **PROMPT:** [202607/prompts/example.md](prompts/example.md)

# Plan: Example
```

```markdown
---
create_time: 2026-07-21 12:00:00
---

- **PLAN:** [../202607/example.md](../example.md)

Original prompt text.
```

The type names the linked counterpart: plan artifacts use `PROMPT`, and prompt snapshots use `PLAN`. Preserve the
existing storage-layout-aware visible labels and compute each href relative to the physical source file, including the
requested `../` prefix in the prompt's visible plan label.

## Shared artifact-link contract

Replace the frontmatter-value-specific Rust contract with a document-level SDD artifact-link contract and expose the
required operations through the Python binding. The contract should parse and render the exact canonical bullet,
validate a non-empty safe label and relative POSIX href, identify whether the bullet is `PLAN` or `PROMPT`, and split it
from the remaining Markdown body. Rendering/upsert must preserve valid YAML frontmatter and unrelated body content while
normalizing the bullet's placement and the one blank separator line.

Treat duplicate link bullets, a bullet of the wrong type for the artifact, malformed Markdown, unsafe targets, or
multiple contradictory representations as invalid rather than guessing. A valid canonical bullet is the preferred
representation. Continue to recognize both historical frontmatter formats—the original plain path and the newer inline
Markdown value—as legacy read formats. If a canonical bullet and legacy frontmatter property coexist and resolve to the
same target, reads may tolerate the transition but repair should remove the redundant property; if they disagree,
validation and repair should report the conflict and avoid silently selecting a target.

Update Rust plan reads to project the parsed visible label through the existing counterpart-link model while excluding
the metadata bullet and its separator from the user-authored body. Titles, summaries, full-body output, and search must
therefore remain based on the real artifact content. Preserve link-label searchability and existing wire/CLI projection
semantics even though the value is no longer a frontmatter entry. Apply the same document parsing contract to prompt
snapshots so Python prompt search and prompt-kind plan search return the plan reference without exposing the bullet as
prompt text or deriving a title from `PLAN`.

## Canonical writes and explicit migration

Route both paired SDD writer entry points and the commit-time copied-plan/link-inference path through one source-aware
document updater. New plans must no longer receive a `prompt` YAML field, and new prompt snapshots must no longer
receive a `plan` YAML field. Existing plan metadata such as tier, title, goal, timestamps, status, and bead information
must remain in frontmatter and retain its ordering/meaning. Cover bodies with and without an H1, documents with and
without existing frontmatter, and repeated writes so each file contains exactly one correctly typed bullet in the
canonical position.

Refactor SDD discovery, list, validation, and reverse-link checks to consume the parsed artifact link rather than read
the relationship directly from the frontmatter map. Canonical hrefs continue to resolve from the containing file; legacy
frontmatter paths retain their current root/project fallback rules. Reverse-link validation should compare resolved
physical paths, allowing canonical/legacy pairs during gradual migration while diagnosing missing, malformed,
wrong-kind, duplicate, conflicting, and nonexistent-target links precisely.

Keep `sase plan links repair` as the only bulk migration path. Its dry run should report missing/stale bullets and both
legacy frontmatter encodings. `--write` should install the canonical bullet, remove only the corresponding legacy
`prompt` or `plan` YAML key, preserve every unrelated frontmatter field and body byte where practical, and be
idempotent. Ordinary reads, search, validation, initialization, and upgrades must not rewrite historical repositories.
Keep existing list/search text and JSON outputs stable by returning the visible path label rather than the raw bullet.

## Schema guidance, documentation, and rollout

Stop advertising `prompt` as a canonical plan-frontmatter field in validation/schema output and authoring guidance. The
strict plan validator should nevertheless accept it as a deprecated hidden system field so historical archived plans
remain valid until explicitly repaired. Prompt-side readers likewise retain compatibility with the legacy `plan`
property without encouraging new writes.

Update command help, the SDD guide, generated in-tree and sidecar README templates, and the beads/SDD narrative to call
these artifact links rather than frontmatter links. Document the exact `- **PROMPT:** ...` and `- **PLAN:** ...`
examples, byte-zero frontmatter rule, one-blank-line layout, relative-href behavior, legacy-read policy, conflict
handling, and explicit `sase plan links repair --write` migration. Regenerate the plans-sidecar README from its updated
template, but do not bulk-edit the archived plan corpus as part of initialization or documentation generation.

## Verification

Add focused Rust tests for document parsing/rendering/upsert with and without frontmatter and H1 headings; exact bullet
type and spacing; safe relative paths; malformed, duplicate, wrong-type, and conflicting representations; both legacy
frontmatter encodings; Python-binding parity; clean plan body/title/summary projection; preserved link searching; and
deprecated-field validation compatibility without presenting `prompt` in the canonical schema.

In SASE, cover exact serialization for flat sidecar, in-tree `sdd/`, and local `.sase/sdd` layouts; both writer entry
points; commit-time inferred links; preservation of unrelated YAML and body content; prompt Q&A updates; clean prompt
text/title/search projection; canonical, legacy, mixed, redundant, and conflicting validation; broken href and reverse
link diagnostics; stable list/search JSON; repair dry-run, property removal, canonical placement, and second-run
idempotence. Include fixtures proving GitHub-relative forward and reverse hrefs resolve to the intended files and that
the bullet precedes an H1 with exactly one intervening blank line.

Run the Rust repository's formatting, clippy-with-warnings-denied, workspace tests, binding tests, and doc tests. In the
SASE workspace, run `just install` before focused tests and the required `just check`, including generated README and
committed-SDD validation. Finish with diff and working-tree audits across SASE, `sase-core`, and the plans sidecar.
